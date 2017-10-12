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
