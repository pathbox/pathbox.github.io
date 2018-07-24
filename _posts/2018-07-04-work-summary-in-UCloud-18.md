---
layout: post
title: 最近工作总结(十七)
date:   2018-06-26 20:45:06
categories: Work
image: /assets/images/post.jpg
---

### 运营开发思维

Leader介绍公司组织架构和产品，可以感受到Leader对产品设计的深入性。提到了一点`运营开发思维`，第一次听到这个概念，简单总结就是： 开发的产品上线后可能会遇到什么问题，在开发前就能有所思考，然后对其进行优化。

### struct{}{} has a width of zero, it occupies zero bytes of storage

```go
func main() {
  finish := make(chan struct{})
  var done sync.WaitGroup
  done.Add(1)
  go func() {
          select {
          case <-time.After(1 * time.Hour):
          case <-finish:
          }
          done.Done()
  }()
  t0 := time.Now()
  close(finish) // finish <- struct{}{}
  done.Wait()
  fmt.Printf("Waited %v for goroutine to stop\n", time.Since(t0))
}
```

>As the behaviour of the close(finish) relies on signalling the close of the channel, not the value sent or received, declaring finish to be of type chan struct{} says that the channel contains no value; we’re only interested in its closed property.

The channel contains no value, it save the memory.

```go
var s struct{}
fmt.Println(unsafe.Sizeof(s)) // prints 0
```

>it should be evident that the empty struct has a width of zero. It occupies zero bytes of storage

```go
type S struct {
        A struct{}
        B struct{}
}
var s S
fmt.Println(unsafe.Sizeof(s)) // prints 0
```

>Because the empty struct consumes zero bytes, it follows that it needs no padding. Thus a struct comprised of empty structs also consumes no storage.

```
https://dave.cheney.net/2013/04/30/curious-channels
https://dave.cheney.net/2014/03/25/the-empty-struct
```

### 初识CPU底层的pipeline

这里的`pipeline`翻译为：流水线，管线。不是网络和I/O读写的pipeline，暂时未了解二者之间的关系。

>流水线，亦称管线，是现代计算机处理器中必不可少的部分，是指将计算机指令处理过程拆分为多个步骤，并通过多个硬件处理单元并行执行来加快指令执行速度。其具体执行过程类似工厂中的流水线，并因此得名。如果作出类比，则计算机指令就是流水线传送带上的产品，各个硬件处理单元就是流水线旁的工人。在使用流水线的处理器中一个指令不是在处理器的一个定时器讯号中完成的，而是被分到多个讯号中去完成，但是与此同时多个指令的分任务被同时处理。由于这些分任务比整个指令要简单，因此可以通过使用流水线提高定时器频率。虽然每个指令需要多个讯号后才能完成，但是通过多个指令的并行运算每个讯号内一个指令可以完成，因此通过这个方法整个速度可以提高。一条流水线的每个分步骤被称为流水线级。它们被流水线寄存器分开。除指令流水线外在现代系统中还有其它流水线，比如用来计算浮点数的算术流水线。

关键词: 计算机处理器、计算机指令、硬件处理单位、多个定时讯号器

No Pipeline: A=>B=>C A=>B=>C A=>B=>C

Pipeline: A=>B=>C
             A=>B=>C
                A=>B=>C

计算机指令被拆分为多个步骤，并通过多个硬件处理单元并行执行来加快指令的执行速度。这句话是关键，这样，CPU只要进行流水线的调度，将代码执行分配到硬件处理单元处理，实现并行处理。这样自然比完成一个一个完整的指令的串行化快多了,不过这是一种更微观的并行，这种并行已经不是在CPU层面，而是更下面的指令硬件处理单元层面。

`That's amazing world.`

### RabbitMQ Quality of Service (QOS)

三个参数：prefetchCount, prefetchSize int, global bool

如果没有配置，默认情况：消息是一个一个出队列，然后被消费者消费，再Ack。

So let’s increase our QOS value to “5”. Now our Consumer reads from the Queue at a rate of 5 messages at a time. Our Queue is now empty. But what’s happening at the Consumer? The Consumer manages its own Shared Queue in memory. This Shared Queue is a local representation of an AMQP Queue. When a Consumer de-queues a message, that message is cached in the Consumer’s Shared Queue, and processed accordingly.

配置了，消费者可以一次取多个消息(根据配置情况)，放到消费者本地的共享缓存队列中，再一个一个操作消息。这样在一定程度上可以减少消费者取队列的操作，减少network traffic involve time。

prefetchCount和prefetchSize配置不是越大越好。

1. 太大会增加消费者共享缓存队列的大小，增加了内存消耗
2. 在消费者共享缓存中取消息操作，同样需要循环的时间，消费单个消息的时间并没有减少
3. 当有多个消费者的时候，A消费者取了100个消费，B消费者取了剩下的10个消息，这时候A消费者的工作负载就更大，B消费者消费完10个消息后，一段时间没有新的消息，则这段时间B消费者处于闲置idle状态，浪费的处理资源。也就是配置数量越大，越容易导致多消费者消费负载不够均衡(not adequately balanced)
4. 如果是对实时性要求比较高的，增加集群数量和消费者数量，提高并发性才是正确的方式，而进行QOS配置会降低实时性的并发性

`适合处理实时性不那么高的场景`，合适的配置可以加快整体的消费时间

```
https://insidethecpu.com/2014/11/11/rabbitmq-qos-vs-competing-consumers/
https://stackoverflow.com/questions/21652517/amqp-acknowledgement-and-prefetching
```

### Go语言的传参都是值传递
Go语言中所有的传参都是值传递（传值），都是一个副本，一个拷贝。因为拷贝的内容有时候是`非引用类型`（int、string、struct等这些），这样就在函数中就无法修改原内容数据；有的是`引用类型`（指针、map、slice、chan等这些），这样就可以修改原内容数据。

一句话总结就是：Go语言的传参都是值传递，分为`非引用类型`不可修改原内容数据还是`引用类型`可以修改原内容数据

### keep-alive and keepalive

- HTTP的keep-alive是为了让TCP连接存活久一点以便复用连接完成更多HTTP请求，提高连接利用率。

- TCP的Keepalive是一种TCP连接的探测机制，使其一直保持可用

http://blog.51cto.com/littledevil/2062101

### 关于Java Thread and Goroutine

Java JVM默认为每个线程1MB堆栈。所以，在一般机器，比如16G内存的服务器，可能最多也只要1w+的thread，实际情况可能还要少

Golang 新的goroutine只有大约4KB的堆栈。每个堆栈4KB，你可以在一个GB的RAM中放置250万个goroutine - 这比Java每个线程的1MB有了很大的改进

由于JVM使用操作系统线程，因此它依赖于操作系统内核来安排它们

Golang实现了自己的调度程序，goroutine是在用户态中，允许许多Goroutines在同一个OS线程上运行

Golang首先开始使用分段堆栈模型，其中堆栈实际上会扩展到单独的内存区域，使用一些聪明的标记来跟踪。后来的实现改进了特定情况下的性能，而不是使用连续堆栈而不是拆分堆栈，就像调整散列表大小，分配新的大堆栈并通过一些非常棘手的指针操作，所有内容都被小心地复制到新的，更大，堆栈

https://rcoh.me/posts/why-you-can-have-a-million-go-routines-but-only-1000-java-threads/

### TCP四次挥手再简记

- 主动关闭连接的一方，调用close()；协议层发送FIN包
- 被动关闭的一方收到FIN包后，协议层回复ACK；然后被动关闭的一方，进入CLOSE_WAIT状态，主动关闭的一方等待对方关闭，则进入FIN_WAIT_2状态；此时，主动关闭的一方 等待 被动关闭一方的应用程序，调用close操作
- 被动关闭的一方在完成所有数据发送后，调用close()操作；此时，协议层发送FIN包给主动关闭的一方，等待对方的ACK，被动关闭的一方进入LAST_ACK状态；
- 主动关闭的一方收到FIN包，协议层回复ACK；此时，主动关闭连接的一方，进入TIME_WAIT状态；而被动关闭的一方，进入CLOSED状态
- 等待2MSL时间，主动关闭的一方，结束TIME_WAIT，进入CLOSED状态

FIN_WAIT_2：主动关闭方
CLOSE_WAIT：被动关闭方
LAST_ACK：被动关闭方
TIME_WAIT：主动关闭方
CLOSED：被动关闭方
CLOSED：主动关闭方

TIME_WAIT: 属于tcp正常的一个状态，是为了解决网络的丢包和网络不稳定锁存在的一个状态

### Go is a value-oriented language

>Go is a value-oriented language in the tradition of C-like systems languages rather than reference-oriented language in the tradition of most managed runtime languages

### 了解一个项目代码逻辑的最好的方式是‘将项目重构一遍’
Yeah, this is a joke.

### makefile中常见的错误—missing separator. Stop.—原因命令行缺少tab键
出现问题的原因是：在makefile中，命令行要以tab键开头, 而不是两个空格

### 新人培训小结

1. 云是什么？云是服务器集群
2. 为什么有云？为了解决服务资源贡献问题
3. 云的未来。大数据、AI、区块链、物联网

Google和AWS在几乎同一年开始云服务产品，为什么AWS完全碾压了Google？

Google早期是走PaaS路线，AWS是走IaaS路线。由于将IDC模式转到IaaS比PaaS更容易，兼容性更好，成功率高。所以，早期较多的用户选择了AWS。之后AWS不断完善产品，使得他们成为了世界云计算市场的最大赢家。现在，Google也开始发力IaaS，但已经较难追赶上AWS了。

Google的三篇著名分布式计算、存储相关论文：

- Google FS
- MapReduce
- BigTable
