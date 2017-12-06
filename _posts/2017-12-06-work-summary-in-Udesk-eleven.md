---
layout: post
title: 最近工作总结(十一)
date:   2017-12-06 16:55:06
categories: Work
image: /assets/images/post.jpg
---

##### Go chan 类型需要初始化操作

如果你使用Go chan 类型,需要进行初始化操作.
例子:

```go
type Socket struct {
  message chan []byte
  close chan struct{}
  conn *websocket.Conn
}

socket := &Scoket{}

for {
  select {
  case msg := <- socket.message:
  case <-socket.close:
  default:
  }
}

这个`select`不会起任何作用.因为 socket没有为chan初始化值,内存里是没有这个指针值的

正确的姿势:


socket := &Scoket{
  message: make(chan []byte),
  close: make(chan struct{}),
}

for {
  select {
  case msg := <- socket.message:
  case <-socket.close:
  default:
  }
}

这样就能正确使用了

```
