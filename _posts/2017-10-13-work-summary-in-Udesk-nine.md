---
layout: post
title: 最近工作总结(九)
date:   2017-10-12 17:37:06
categories: Work
image: /assets/images/post.jpg
---

##### net/http in Golang

1.实现TCP Server监听，为每一新来的TCP link建立一个goroutine, 并在goroutine与客户端交互（不用担心单机C10K问题，因为goroutine是用户态线程，很轻量级，可以很随意就创建成千上万个）。

2.在每个goroutine中将TCP link中的请求数据按HTTP协议格式解析出来(可以将数据解析出成Request对象，以后的访问提供方便)，并根据其URL找到相应的处理程序Handler, 因此你还需要提前建立好 URL：Handler映射表。

3.当处理程序处理结束后，你还需要将处理结果数据按HTTP协议格式返回给客户端。代码大致如下：

```go
//建立 URL:Handler映射表
//注意：由于table会在不同goroutine中使用，因此真正环境中需要锁保护
var table = map[string]Handler{
  "/": rootHandler,
  "/login": loginHandler,
  ...
}

// Server 监听
ln, _ := net.Listen("tcp", ":8080")
for {
  conn, _ := ln.Accept()
  go handleConn(conn)
}

// 请求处理
func handleConn(conn net.Conn) {
    //1. 将请求成封装Request对象
    //2. 从table中查找相应处理程序
    //3. 将处理结果封装HTTP格式数据返回给客户端
}

net/http包中几个重要的类型:
http.ServeMux: 建立URL:Handler映射表
http.Server: 运行HTTP Server
http.Request: 封装客户端HTTP请求数据
http.ResponseWriter: 用来构造服务器端HTTP响应数据
http.Handler: URL处理程序必须实现的接口
```

##### 不要手贱卸载homebrew
重要的事说三遍：

不要手贱卸载homebrew

不要手贱卸载homebrew

不要手贱卸载homebrew

##### C10K 问题
简单描述：假设这里有10000个并发连接。一般来说，只有少量的文件描述符被使用，比如10个已经读就绪。
那么每次pool()/select()被调用，就有9900个文件描述符被毫无意义的拷贝和扫描。

select/poll模型，这些技术都有一定的缺点：如selelct最大不能超过1024、poll没有限制，但每次收到数据需要遍历每一个连接查看哪个连接有数据请求

创建的进程线程多了，数据拷贝频繁（缓存I/O、内核将数据拷贝到用户进程空间、阻塞）， 进程/线程上下文切换消耗大， 导致操作系统崩溃，这就是C10K问题的本质！

解决C10K问题的关键就是尽可能减少这些CPU等核心计算资源消耗，从而榨干单台服务器的性能，突破C10K问题所描述的瓶颈。

解决思路：　每个进程/线程同时处理多个连接（IO多路复用）

当文件句柄数目达到 10K 的时候，epoll 已经超过 select 和 poll 两个数量级

Epoll is C10K killer

lib libevent

##### index_exists?

```ruby
add_index :agent_extras, :agent_id	unless index_exists?(:agent_extras, :agent_id)
```

用该方法可以在创建索引的时候,判断索引是否存在,索引不存在才创建,避免创建相同索引而报错

##### 在接口调用中使用 proxy(中心处理) and adapter(分类处理)
最近总结了项目中对云问机器人的调用接口。代码使用了proxy模式+adapter模式

proxy中，对接口进行统一的处理。对request params 和response 进行处理

adapter 根据传的识别参数不同， 调用微信机器人的接口，或client端的机器人接口

想起另一个项目中，也是类似的使用了这两种模式结合的方式设计代码

益处是:  共同的方法或内容统一处理，分类的方法再细分处理。避免重复性代码，并且可维护性提高。

不需要同事维护多个地方，一般只要维护proxy一个地方。 因为adapter处的维护较少。

当需要增加一个类别adapter的时候，可以很方便的增加。只要增加一个新的adapter，使用共同的proxy。

而不用写一个新的proxy
