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
