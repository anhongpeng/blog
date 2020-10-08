# 缓存

> 本文笔记摘自 [一文读懂前端缓存 | 知乎](https://zhuanlan.zhihu.com/p/44789005)

cache 的发音是 `[kæʃ]`（同 cash）。

## 前端缓存 VS 后端缓存

![请求的3个步骤](https://pic3.zhimg.com/80/v2-05f4e6a5aaac9fce4d2a7a2fcc652c9e_1440w.jpg)

基本的网络请求就是3个步骤：请求、处理、响应

* 后端缓存：主要集中在“处理”步骤，通过「保留数据库连接」、「存储处理结果」等方式缩短处理时间，尽快进入“响应”步骤
* 前端缓存：
  * 在“请求”步骤中，浏览器可以通过存储结果的方式直接使用资源，从而省去了发送请求；
  * 在“响应”步骤中，需要浏览器和服务器共同配合，通过减少响应内容来缩短传输时间

## 按缓存所处的位置分类

### memory cache

memory cache 是「内存中的缓存」，与之相对的是 disk cache，也就是硬盘中的缓存。按操作系统的常理：先读内存，再读硬盘。

几乎所有的网络请求资源都会被浏览器自动加入到 memory cache 中，但由于这些资源数量很大，并且浏览器内存也是有限的，所以 memory cache 注定只是一个「短期存储」。常规情况下，浏览器标签页关闭后，该次浏览的 memory cache 便会失效。

### disk cache

disk cache 也叫 HTTP cache，顾名思义是「存储在硬盘上的缓存」，因此它是持久存储的，是实际存在于文件系统中的。

disk cache 会严格根据 HTTP 头信息中的各类字段来判断要缓存哪些资源。

当命中缓存后，浏览器会从硬盘中读取资源，虽然比起从内存中读取慢了一些，但比网络请求还是快了很多。

凡是持久性存储都会面临容量增长的问题，disk cache 也不例外。在浏览器自动清理时，会按算法把“最老的”或者“最可能过时的”资源删除，不同浏览器算法的实现是不同的。

### Service Worker

memory cache 和 disk cache 的缓存策略以及缓存/读取/失效的动作都是由浏览器内部判断进行的。我们只能设置响应头的的某些字段来告诉浏览器，而不能自己操作。

举个生活中去银行存/取钱的例子来说，你只能告诉银行职员，我要存/取多少钱，然后把由他们经过一系列的记录和手续之后，把钱放到金库中去，或者从金库中取出钱来交给你。

但 Service Worker 的出现，给予了我们另外一种更加灵活，更加直接的操作方式。我们可以绕开银行职员，自己走到金库前（当然是有别于上述金库的一个单独的小金库），自己把钱放进去或取出来。因此我们可以选择放哪些钱（缓存哪些文件），什么情况把钱取出来（路由匹配规则），取哪些钱出来（缓存匹配并返回）。

我们可以在 Chrome 浏览器控制台中，Application --> Cache Storage 找到这个单独的“小金库”。这个缓存是永久性的，即便关闭标签页或者浏览器，下次打开依然还在。

有 2 种情况会将缓存中的资源清除：

* 手动调用 API `cache.delete(resource)`
* 缓存容量超过限制，被浏览器清空

综上，请求时从缓存获取资源的优先级是：

1. Service Worker
2. Memory Cache
3. Disk Cache
4. 网络请求

由上到下寻找，找到即返回，找不到则继续。

## 按失效策略分类

memory cache 是浏览器为了加快读取缓存速度而进行的自身优化行为，不受开发者控制，也不受 HTTP 协议头的约束，算是一个黑盒。

Service Worker 是由开发者编写的额外的脚本，且缓存位置独立，出现也较晚，使用还不算太广泛。

所以我们平时最为熟悉的其实是 disk cache，也叫 HTTP cache（因为不像 memory cache，它遵守 HTTP 协议头中的字段）。平时所说的「强制缓存」、「对比缓存」，以及「Cache-Control」等，也都归于此类。

### 强制缓存（也叫强缓存）

强制缓存的含义是，当客户端请求后，会先访问缓存数据库看缓存是否存在。如果存在则直接返回；不存在则请求真的服务器，响应后再写入缓存数据库。

**强制缓存直接减少请求数，是提升最大的缓存策略。**它的优化覆盖了之前提到过的请求数据的三个步骤。如果考虑使用缓存来优化网页性能的话，强制缓存应该是首先被考虑的。

可以造成强制缓存的字段是 `Cache-control` 和 `Expires`

#### Expires

这是 HTTP 1.0 的字段，表示缓存到期时间，是一个绝对的时间 (当前时间+缓存时间)，如：

`Expires: Thu, 10 Nov 2017 08:45:11 GMT`

在响应消息头中，设置这个字段之后，就可以告诉浏览器，在未过期之前不需要再次请求。

但是，这个字段设置时有两个缺点：

* 由于是绝对时间，用户可能会将客户端本地的时间进行修改，而导致浏览器判断缓存失效，重新请求该资源。此外，即使不考虑自行修改，时差或者误差等因素也可能造成客户端与服务端的时间不一致，致使缓存失效
* 写法太复杂了。表示时间的字符串多个空格，少个字母，都会导致非法属性从而设置失效。

#### Cache-control

已知 `Expires` 的缺点之后，在 HTTP/1.1中，增加了一个字段 `Cache-control`，该字段表示「资源缓存的最大有效时间」，在该时间内，客户端不需要向服务器发送请求。

这两者的区别就是前者是绝对时间，而后者是相对时间。如下：

`Cache-control: max-age=2592000`

下面列举一些 `Cache-control` 字段常用的值（完整的列表可以查看 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Cache-Control)）：

* `max-age`：最大有效时间
* `must-revalidate`：如果超过了 `max-age` 的时间，浏览器必须向服务器发送请求，验证资源是否还有效
* `no-cache`：虽然字面意思是“不要缓存”，但实际上还是要求客户端缓存内容的，只是是否使用这个内容由后续的对比来决定
* `no-store`: 真正意义上的“不要缓存”。所有内容都不走缓存，包括强制缓存和对比缓存
* `public`：所有的内容都可以被缓存（包括客户端和代理服务器，如 CDN）
* `private`：所有的内容只有客户端才可以缓存，代理服务器不能缓存。默认值

这些值可以混合使用，例如 `Cache-control:public, max-age=2592000`。在混合使用时，它们的优先级如下图：

![Cache-control 值的优先级](https://pic1.zhimg.com/v2-9af573cd1971b2e0260ec9f38ef96650_r.jpg)

自从 HTTP/1.1 开始，`Expires` 逐渐被 `Cache-control` 取代。Cache-control 是一个相对时间，即使客户端时间发生改变，相对时间也不会随之改变，这样可以保持服务器和客户端的时间一致性。而且 `Cache-control` 的可配置性比较强大。

**`Cache-control` 的优先级高于 `Expires`**，为了兼容 HTTP/1.0 和 HTTP/1.1，实际项目中两个字段都会设置。

### 对比缓存（也叫协商缓存）

**当强制缓存失效（超过规定时间）时，就需要使用对比缓存，由服务器决定缓存内容是否失效。**

流程：

1. 浏览器先请求缓存数据库，返回一个缓存标识。之后浏览器拿这个标识和服务器通讯
2. 如果缓存未失效，则返回 HTTP 状态码 304 表示继续使用，于是客户端继续使用缓存
3. 如果失效，则返回新的数据和缓存规则，浏览器响应数据后，再把规则写入到缓存数据库

**对比缓存在请求数上和没有缓存是一致的，**但如果是 304 的话，返回的仅仅是一个状态码而已，并没有实际的文件内容，因此**在响应体体积上的节省是它的优化点。**

它的优化覆盖了之前提到的请求三步中的最后一个：“响应”。**通过减少响应体体积，来缩短网络传输时间。所以和强制缓存相比提升幅度较小，**但总比没有缓存好。

对比缓存是可以和强制缓存一起使用的，作为在强制缓存失效后的一种后备方案。实际项目中他们也的确经常一同出现。

对比缓存有 2 组字段（不是两个）：

#### Last-Modified && If-Modified-Since

1. 服务器通过 `Last-Modified` 字段告知客户端，资源最后一次被修改的时间，例如 `Last-Modified: Mon, 10 Nov 2018 09:10:11 GMT`
2. 浏览器将这个值和资源内容一起记录在缓存数据库中
3. 下一次请求相同资源时，浏览器从自己的缓存中找出“不确定是否过期的”缓存。在请求头中将上次的 `Last-Modified` 的值写入到请求头的 `If-Modified-Since` 字段
4. 服务器会将 `If-Modified-Since` 的值与 `Last-Modified` 字段进行对比。如果相等，则表示未修改，响应 `304`；反之，则表示修改了，响应 `200` 状态码，并返回数据

但是他还是有一定缺陷的：

* 如果资源更新的速度是秒以下单位，那么该缓存是不能被使用的，因为它的时间单位最低是秒
* 如果请求的文件是通过服务器动态生成的，那么该方法的更新时间永远是生成的时间，尽管文件可能没有变化，所以起不到缓存的作用。

#### Etag && If-None-Match

为了解决上述问题，出现了一组新的字段 `Etag` 和 `If-None-Match`

`Etag` 存储的是文件的特殊标识（一般都是 hash 生成的），服务器存储着文件的 `Etag` 字段。之后的流程和 `Last-Modified` 一致，只是 `Last-Modified` 字段和它所表示的更新时间改变成了 `Etag` 字段和它所表示的文件 `hash`，把 `If-Modified-Since` 变成了 `If-None-Match`。

服务器同样进行比较，命中返回 304, 不命中返回新资源和 200。

**`Etag` 的优先级高于 `Last-Modified`**

## 缓存小结

当浏览器请求资源时：

1. 查看 `Service Worker`
2. 查看 `memory cache`
3. 查看 `disk cache`：
  1. 如果有强制缓存且未失效，则使用强制缓存，不请求服务器。这时的状态码全部是 200
  2. 如果有强制缓存但已失效，使用对比缓存（也叫协商缓存），比较后确定 304 还是 200
4. 都找不到缓存则发送请求

发送请求时：

1. 发送网络请求，等待服务器响应
2. 把响应内容存入 disk cache（如果 HTTP 头信息配置可以存的话）
3. 把「响应内容的引用」存入 memory cache（无视 HTTP 头信息的配置）
4. 把响应内容存入 Service Worker 的 Cache Storage（如果 Service Worker 的脚本调用了 `cache.put()`）