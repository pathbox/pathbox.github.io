---
layout: post
title: 记一次Go websocket项目内存泄露排查 + 使用Go pprof定位内存泄露
date:   2017-05-26 20:20:06
categories: Go
image: /assets/images/post.jpg
---

从同事接手了一个项目。这个项目是用了`<go-socket.io>`这个库，仿照socket.io的功能，和前端进行交互。因为之前是用socket.io做的后端，但是
觉得性能不够用了，就改用Go来进行重构。重构之后，发生内存泄露的现象。这个项目峰值大概维护1w-1.5w个socket长连接,一开始内存消耗量大概到2g-3g。
然而，当连接数没有增长的情况下，随着运行时间的增长，内存也在增长。并且大概一个多小时的时间，内存就涨到了10g，而且连接数才1w多，说好的支撑"百万连接呢"。如果不进行重启服务操作，内存会涨到14g(16g的内存)。所以，确定是有内存泄露。我需要做的就是找到内存泄露的原因，修复这个问题。

Go提供了一个已经足够强大的pprof库。这样，我直接使用这个库，来查看内存的消耗情况。

在main.go　中加入 `<import _ "net/http/pprof">`,因为项目中有http请求，所以，当有http请求的时候，pprof会帮我们分析资源的使用情况。我只针对我的操作，描述如何使用`<pprof>`，具体的操作，可以搜相关文档或文章。

然后，build。把build后的二进制文件传到服务器，重新启动项目。在本地可以用websocketbench　这样的测试工具。因为在本地，内存逐渐增长的环境没能被复现出来，所以，决定用线上环境来进行排查。

让程序跑１个小时，这时内存涨到了3.9g。socket 连接数为6500+。在浏览器输入:　`<host_name:9090/debug/pprof>`，显示pprof 的主页。
网页会显示几个选项，有　heap、goroutine等。这时候的`<goroutine>`数量是24497。我点击heap，进入到　`<host_name:9090/debug/pprof/heap?debug=1>` 移到最下面：

```
# runtime.MemStats
# Alloc = 3537545936
# TotalAlloc = 217546115776
# Sys = 4223095608
# Lookups = 1321203
# Mallocs = 1558757559
# Frees = 1533787828
# HeapAlloc = 3537545936　　　　　　　　　　　　分配给堆的内存使用量
# HeapSys = 3784015872　　　　　　　　　　　　　堆占用系统的内存使用量
# HeapIdle = 167993344　　　　　　　　　　　　　堆中空闲但没有释放还给系统的内存使用量
# HeapInuse = 3616022528　　　　　　　　　　　　堆中正在使用的内存使用量
# HeapReleased = 0
# HeapObjects = 24969731
# Stack = 221347840 / 221347840
# MSpan = 50597304 / 51511296
# MCache = 9600 / 16384
# BuckHashSys = 3240312
# GCSys = 148043776
# OtherSys = 14920128
# NextGC = 3824913056　　　　　　　　　　　　　下一次内存使用量达到多少值时触发GC

```

Then，在我本地build项目的目录，执行：

```
go tool pprof ./bin/my-go-app-20170426103223 http://xxx.xxx.xxx.xxx:9090/debug/pprof/heap
```

```
(pprof) top
1517.34MB of 1870.97MB total (81.10%)
Dropped 760 nodes (cum <= 9.35MB)
Showing top 10 nodes out of 83 (cum >= 43.01MB)
      flat  flat%   sum%        cum   cum%
  332.58MB 17.78% 17.78%   332.58MB 17.78%  runtime.rawstringtmp
  312.09MB 16.68% 34.46%   339.85MB 18.16%  runtime.mapassign
  274.15MB 14.65% 49.11%   274.15MB 14.65%  runtime.makemap
  165.06MB  8.82% 57.93%   165.06MB  8.82%  gopkg.in/mgo%2ev2.copySession
  121.17MB  6.48% 64.41%   121.17MB  6.48%  github.com/gorilla/websocket.newConn
   96.51MB  5.16% 69.57%    97.51MB  5.21%  context.WithCancel
   82.52MB  4.41% 73.98%   703.77MB 37.62%  net/http.readRequest
   45.51MB  2.43% 76.41%   454.68MB 24.30%  net/textproto.(*Reader).ReadMIMEHeader
   44.74MB  2.39% 78.80%    55.25MB  2.95%  _/home/user/Documents/udesk_vistor_go/app/controllers.SocketConnection
   43.01MB  2.30% 81.10%    43.01MB  2.30%  net/url.parse
```
上面列出了现在系统哪些包和库的函数的内存使用量，可以使用 list lib_name　来定位到具体代码的位置。

```
1517.34MB of 1870.97MB total (81.10%)
```
这一行统计了heap内存的使用量。和浏览器中统计的会有误差。

+ flat 表示函数自身执行所用内存

+ cum 表示执行函数自身和其调用的函数所用的内存和

我们观察占用内存量最多的前三项目：

```
332.58MB 17.78% 17.78%   332.58MB 17.78%  runtime.rawstringtmp
312.09MB 16.68% 34.46%   339.85MB 18.16%  runtime.mapassign
274.15MB 14.65% 49.11%   274.15MB 14.65%  runtime.makemap
```
这是Go的原生库的操作，是对字符串和新建map的操作。之前系统用来一个全局map的库`<github.com/streamrail/concurrent-map>`，作为全局map，存储相关数据。
起初怀疑是它的问题，经过修改，去掉了这个全局map，而是使用mongoDB来存储数据。之后观察发现，内存的上涨速度变慢了，但是，内存随着时间仍然上涨，
内存泄露问题仍然存在。

在看过整个项目的代码后，排除代码使用的问题。该close()的地方，比如连接或是MongoDB的操作，都应该没问题。

然后，把关注的移到了使用的第三方库上。在线上进行pprof的时候，观察发现goroutine的数量，当连接数量减少时，一段时间，goroutine的数量也会减少。
Google　`<go-socket.io>` 这个库，也并没搜到其内存泄露的问题。看了`<go-socket.io>`的源码，它的实现机制其实和真正的`<socket.io>`不是一样的。`<socket.io>`是`<nodejs>`的一个库，原理基于`<nodejs>`的事件驱动模式。而`<go-scoket.io>`这个库，当有连接来的时候会起多个goroutine，在goroutine中进行处理，类似多线程的方式，基于Go对goroutine的调度进行处理。

在本地对 `<github.com/pschlump/jsonp>` `<gopkg.in/mgo.v2/bson>` 进行单独测试，没有内存泄露的情况发生。
对使用到　`<github.com/kidstuff/mongostore>` 和　`<github.com/gorilla/sessions>` 的接口进行测试，本地也没有内存上涨的情况发生。(以上测试发生在两周前，并且用了一周的时间进行了逐步排查)

大概两个半小时后的数据：
```
28600 groutines 4800 connections 8.6g Memory  MongoDB connection 464
```

```
# runtime.MemStats
# Alloc = 4523698072
# TotalAlloc = 402074198032
# Sys = 9180321800
# Lookups = 2255805
# Mallocs = 2692470963
# Frees = 2656425569
# HeapAlloc = 4523698072
# HeapSys = 7991754752
# HeapIdle = 2413297664
# HeapInuse = 5578457088
# HeapReleased = 0
# HeapObjects = 36045394
# Stack = 729907200 / 729907200
# MSpan = 100263152 / 108544000
# MCache = 9600 / 16384
# BuckHashSys = 3443088
# GCSys = 320460800
# OtherSys = 26195576
# NextGC = 7747394592
```

```
(pprof) top
3016.30MB of 3658.47MB total (82.45%)
Dropped 774 nodes (cum <= 18.29MB)
Showing top 10 nodes out of 86 (cum >= 313.05MB)
      flat  flat%   sum%        cum   cum%
  655.15MB 17.91% 17.91%   655.15MB 17.91%  runtime.rawstringtmp
  643.69MB 17.59% 35.50%   693.39MB 18.95%  runtime.mapassign
  561.81MB 15.36% 50.86%   561.81MB 15.36%  runtime.makemap
  324.61MB  8.87% 59.73%   324.61MB  8.87%  gopkg.in/mgo%2ev2.copySession
  200.96MB  5.49% 65.22%   200.96MB  5.49%  github.com/gorilla/websocket.newConn
  195.02MB  5.33% 70.55%   195.52MB  5.34%  context.WithCancel
  167.04MB  4.57% 75.12%  1397.52MB 38.20%  net/http.readRequest
   92.51MB  2.53% 77.65%   918.37MB 25.10%  net/textproto.(*Reader).ReadMIMEHeader
   89.01MB  2.43% 80.08%    89.01MB  2.43%  github.com/gorilla/securecookie.New
   86.51MB  2.36% 82.45%   313.05MB  8.56%  github.com/kidstuff/mongostore.(*MongoStore).New
```
可以看到，heap的内存使用量确实上涨了，但是，实际的socket连接数是减少的。

第二天，将数据和同事讨论。关注到了这两行：

```
167.04MB  4.57% 75.12%  1397.52MB 38.20%  net/http.readRequest
 92.51MB  2.53% 77.65%   918.37MB 25.10%  net/textproto.(*Reader).ReadMIMEHeader
```
他们的cum很高。这是之前的观察测试中没有注意到的。cum表示了执行函数自身和其调用的函数所用的内存和。说明这两个地方的整个调用函数中占用了很多内存。(还原真实的环境，数据才最有效)
然后Google　net/http.readRequest　memory leak 和　net/textproto.(*Reader).ReadMIMEHeader　memory leak。其中有一个人也遇到相似的内存泄露的问题，也是使用了go-socket.io这个库。他解决了这个内存泄露问题，不是因为go-socket.io的原因，而是因为他使用的一个库用到了context，但是context的值没有清除。直觉告诉我们，很有可能我们也是在使用类似context的时候，数据没有清空。

然后我全局搜了 context。发现项目中使用了 `<github.com/gorilla/context>`这个库。并不是我们直接使用，而是在使用`<github.com/gorilla/sessions>`的时候，使用了`<github.com/gorilla/context>`。

有可能是`<github.com/gorilla/sessions>`的问题。来到`<github.com/gorilla/sessions>`的主页查看文档，在网页搜索"leak"发现了下面这句:

> Important Note: If you aren't using gorilla/mux, you need to wrap your handlers with context.ClearHandler as or else you will leak memory! An easy way to do this is to wrap the top-level mux when calling http.ListenAndServe:
http.ListenAndServe(":8080", context.ClearHandler(http.DefaultServeMux))

意思是，如果不用我们自家的　`<gorilla/mux>`  `<gorilla/sessions>` 包会有内存泄露。或者是　

> http.ListenAndServe(":8080", context.ClearHandler(http.DefaultServeMux))

这样操作。

What the hell......我震惊了!

我继续搜索到这个方法　`<context.ClearHandler>`　相关的代码如下:

```go
var (
	mutex sync.RWMutex
	data  = make(map[*http.Request]map[interface{}]interface{})
	datat = make(map[*http.Request]int64)
)

func ClearHandler(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer Clear(r)
		h.ServeHTTP(w, r)
	})
}

// Clear removes all values stored for a given request.
//
// This is usually called by a handler wrapper to clean up request
// variables at the end of a request lifetime. See ClearHandler().
func Clear(r *http.Request) {
	mutex.Lock()
	clear(r)
	mutex.Unlock()
}

// clear is Clear without the lock.
func clear(r *http.Request) {
	delete(data, r)
	delete(datat, r)
}

```
顿时明白了。`<gorilla/sessions>` 这个库使用了 `<gorilla/context>`。

`<gorilla/context>`这个库定义了两个全局变量的map。
当每个request来的时候，就将往这两个全局map中以http.Request对象为key存值。如果没有调用`<context.ClearHandler>`方法。即使这个request是已经关闭了，但是，由于这个request作为了全局变量map的key,使得Go GC没法回收已经结束的request对象。内存就很快就被消耗完了。(比较奇怪的是，没在本地复现)

问题找到了，马上修复。上线代码，观察了３个多小时。

The world has returned calm.

1w+ 的socket　连接没有超过2g内存使用量。这才是Go应有的表现(百万连接不是梦)


总结:

+ Go pprof is awesome

+ Global variable is devil

+ go-socket.io is more powerful than socket.io

参考链接：

[那位遇到同样问题的人的提问](https://stackoverflow.com/questions/39404625/golang-websocket-memory-leak)

[gorilla/sessions](http://www.gorillatoolkit.org/pkg/sessions)

[pprof的使用](http://gpdb.rocks/golang/profiling/2016/07/31/golang-profiling.html)
