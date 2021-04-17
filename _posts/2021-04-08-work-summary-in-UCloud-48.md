---
layout: post
title: 最近工作总结(48)
date:  2021-04-08 20:00:00
categories: Work
image: /assets/images/post.jpg


---

​    

### 接口耗时暴增原因排查

起因: 调用方部门反馈某个内部接口耗时突然增加了，导致原本超时时间设置太小，而接口超时失败。

快速处理：调用方将接口超时时间增大到合适值

排查：通过监控查看该接口一周的调用情况，发现确实从2天前开始，接口耗时从100ms左右飙升到了3-4s。但是，接口调用请求量并没有暴涨，和之前差不多

- 先从代码入手，查看2天前的那个时间点后，代码上是否有变更导致。 结果：代码没有问题
- 构造一个测试数据，发起测试请求，得到的请求耗时正常100ms内。从日志中过找出一个耗时长的请求，重新测试，请求耗时3s
- 将接口中涉及到的所有SQL操作列出，搜索数据库慢日志，看是否有对应慢日志。结果：没有对应慢日志
- 根据耗时3s请求的参数，手动在数据库上执行SQL，SQL耗时正常，均在20ms完成。结果：SQL，数据库性能应该正常
- 接口业务中是否有调第三方API。结果：没有
- 接口业务中发现有用tcp方式非http方式调用第三方的服务。通过对比测试调用和不调用该tcp第三方服务，是该服务有问题。但由于这块老代码中，没有超时或者报错提示，导致日志中没有报错信息，问题被隐藏

解决方式：

- 通知该tcp服务负责人员
- 将该调用加上相应超时和报错信息日志
- 异步方式调用该tcp服务

还有哪些方向可以查：

- 局域网网络是否有问题
- DNS解析是否耗时过长
- 对应前置内部网关是否有问题
- 调用方部门的服务是否部署到了别的地域导致不再同一个局域网内



### go context cancel不执行会怎样

If you fail to cancel the context, the [goroutine that WithCancel or WithTimeout created](https://golang.org/src/context/context.go?s=9162:9288) will be retained in memory indefinitely (until the program shuts down), causing a memory leak. If you do this a lot, your memory will balloon significantly. It's best practice to use a `defer cancel()` immediately after calling `WithCancel()` or `WithTimeout()`

很有可能会导致内存泄漏

在goroutine中往channel写入数据，很可能由于读取channel的逻辑错误而没法执行到读取channel而导致写入channel的goroutine一直阻塞,造成goroutine泄漏，GC也不会将其回收。该阻塞的goroutine实际上被认为还在使用

### go 内存逃逸示例

`golang程序变量`会携带有一组校验数据，用来证明它的整个生命周期是否在运行时完全可知。如果变量通过了这些校验，它就可以在`栈上`分配。否则就说它 `逃逸` 了，必须在`堆上分配`。

能引起变量逃逸到堆上的**典型情况**：

- **在方法内把局部变量指针返回** 局部变量原本应该在栈中分配，在栈中回收。但是由于返回时被外部引用，因此其生命周期大于栈，则溢出。
- **发送指针或带有指针的值到 channel 中。** 在编译时，是没有办法知道哪个 goroutine 会在 channel 上接收数据。所以编译器没法知道变量什么时候才会被释放。
- **在一个切片上存储指针或带指针的值。** 一个典型的例子就是 []*string 。这会导致切片的内容逃逸。尽管其后面的数组可能是在栈上分配的，但其引用的值一定是在堆上。
- **slice 的背后数组被重新分配了，因为 append 时可能会超出其容量( cap )。** slice 初始化的地方在编译时是可以知道的，它最开始会在栈上分配。如果切片背后的存储要基于运行时的数据进行扩充，就会在堆上分配。
- **在 interface 类型上调用方法。** 在 interface 类型上调用方法都是动态调度的 —— 方法的真正实现只能在运行时知道。想像一个 io.Reader 类型的变量 r , 调用 r.Read(b) 会使得 r 的值和切片b 的背后存储都逃逸掉，所以会在堆上分配

```go
package main
import "fmt"
type A struct {
 s string
}
// 这是上面提到的 "在方法内把局部变量指针返回" 的情况
func foo(s string) *A {
 a := new(A) 
 a.s = s
 return a //返回局部变量a,在C语言中妥妥野指针，但在go则ok，但a会逃逸到堆
}
func main() {
 a := foo("hello")
 b := a.s + " world"
 c := b + "!"
 fmt.Println(c)
}

// go build -gcflags=-m main.go
```



### 如何处理大量的写请求

一些具体的场景

- 如果相应的数据能够完全放入缓存中，可以考虑利用redis集群当数据库使用
- 如果写请求不需要立刻知道结果，业务逻辑上可以有一定的延迟。将写请求消息发到消息队列中(RocketMQ)，通过消息队列消费者写入到数据库。消息队列有重试机制，能尽可能的提高成功率
- 对写请求即时性比较高，考虑先将写操作在缓存中进行，然后再通过消息队列持久化到数据库

写请求是量大，并非并发多

- 是否能够将写操作合为批量处理

 