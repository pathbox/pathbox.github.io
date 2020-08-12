---
layout: post
title: 最近工作总结(40)
date:  2020-08-04 17:30:06
categories: Work
image: /assets/images/post.jpg
---

### elasticsearch match,term,match_phrase简记
https://blog.csdn.net/sinat_29581293/article/details/81486761

### 同步数据跑的脚本要具有幂等性

### golang channel runtime.chansend
如果当前 Channel 的 recvq 上存在已经被阻塞的 Goroutine，那么会直接将数据发送给当前的 Goroutine 并将其设置成下一个运行的 Goroutine；
如果 Channel 存在缓冲区并且其中还有空闲的容量，我们就会直接将数据直接存储到当前缓冲区 sendx 所在的位置上；
如果不满足上面的两种情况，就会创建一个 runtime.sudog 结构并将其加入 Channel 的 sendq 队列中，当前 Goroutine 也会陷入阻塞等待其他的协程从 Channel 接收数据；
发送数据的过程中包含几个会触发 Goroutine 调度的时机：

发送数据时发现 Channel 上存在等待接收数据的 Goroutine，立刻设置处理器的 runnext 属性，但是并不会立刻触发调度；
发送数据时并没有找到接收方并且缓冲区已经满了，这时就会将自己加入 Channel 的 sendq 队列并调用 runtime.goparkunlock 触发 Goroutine 的调度让出处理器的使用权

https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/

### 非对称加密传输密钥，对称加密传输内容

### SQL查询得到的struct建议所有字段都有对应真实值
例子：
```go
type Person struct {
  Name string
  UserID int
  CompanyID int
}
```
一个SQL的查询方法中，对上述的struct实例实际只取出了Name，UserID两个字段，但实际使用中，有可能会根据IDE的提示，取CompanyID字段，这个字段是0，则导致了后面诡异的问题的出现。所以，建议Person struct的所有Field都取真实的值。
如果CompanyID字段没有用到，则新建一个

```go
type PersonB struct {
  Name string
  UserID int
}
```
