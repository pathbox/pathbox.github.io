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

配置了，消费者可以一次取多个消息(根据配置情况)，放到消费者本地的共享缓存队列中，再一个一个操作消息。这样在一定程度上可以减少消费者取队列的操作。

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
