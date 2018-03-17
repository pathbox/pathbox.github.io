---
layout: post
title:  Golang 后台守护进程队列处理方式小结
date:   2018-03-17 14:33:06
categories: Golang
image: /assets/images/post.jpg
---

在写一个 `Golang Server`的时候，比如 http接口，最简单的就是使用net/http 包，每个请求就会起一个goroutine来进行操作。很方便，但是，当并发量大的时候，就会起了成千上万的goroutine，当goroutine的量达到一个很大的数量，服务性能也就出现了瓶颈。

我们可以手动构建简单的goroutine池，再借助channel队列，后台起守护goroutine，来处理channel中的数据。模型大概是这样的：

```go
var myChan = make(chan int, 1000) // 全局channel队列

for i := 0; i < poolSize; i++{
  go initWorker()
}

func ProcessWorker() {
  for {
    select {
    case value := <-myChan:
      fmt.Println(value)
      // ... do something with value
    case done ：= <-done: // 关闭机制
      break
    default: // 这里不用超时处理，就是希望goroutine一直在后台执行
    }
  }
}

func indexHandler() {
  //...
  myChan <- value
}
```

这样我们就创建了poolSize数量的goroutine，在不断的从myChan中获取数据，Golang的调度会帮我们做这一切。

很容易就可以发现，这里的一个瓶颈就是myChan的size大小。当myChan很快就满了，后续的请求就会阻塞了。在我的项目的尝试中，我将size设为 `10000`， 进行ab测试，发现测试结果比用普通的多goroutine处理方式要好。我加到了10w，性能反而下降了。所以，myChan Size的代销不是越多越好，而是根据你实际情况来测试出一个合适的值。

在我的凌一个项目中，是对文件的格式转换，对核心代码的执行，希望不要支持高并发，在高并发下核心的命令代码会执行报错。所以，我只创建了一个守护goroutine，并且make myChan的Size 为200.这样，用ab测试时，能支持200并发，并且几乎没有执行失败的报错。

我似乎喜欢上了使用上述的方式来处理请求，Golang select和channel的设计让我可以很愉悦的使用上面的方式，并且达到预期的效果。

我并没有发现程序执行的异常，直到今天，我忽然发现服务在没有请求的时候，CPU占用率达到了100%（4核心的机器）。我十分诧异，第一时间想到了可能是 `ProcessWorker` 的问题。

这里涉及到了select调度的方法：

当某个case的channel数据可以取到了就执行它；

当多个case同时取到数据了，会随机执行一个；

当没有case取到数据，都阻塞时，会执行 default。

当服务没有接收请求的时候，ProcessWorker 方法中会执行的应该是default，这个没错。而在外层我们用了 for 的死循环，以保证goroutine一直执行。这个问题就来了，ProcessWorker会一直执行default，而不会阻塞在case。如果你在default打印日志

```go
func ProcessWorker() {
  for {
    select {
    case value := <-myChan:
      fmt.Println(value)
      // ... do something with value
    case done ：= <-done: // 关闭机制
      break
    default: // 这里不用超时处理，就是希望goroutine一直在后台执行
      log.Println("default")
    }
  }
}
```

你会发现后台在不断的输出日志。

##### 一个简单的代码例子

```go
// example.go

package main

import (
	"fmt"
	"log"
	"net/http"
)
var c = make(chan int, 100)

func main() {
	go worker()
	addr := ":9090"

	http.HandleFunc("/", index)
	log.Fatal(http.ListenAndServe(addr, nil))
}

func worker() {
	for {
		select {
		case d := <-c:
			fmt.Println(d)
			default:
		}
	}
}

func index(w http.ResponseWriter, r *http.Request) {
	c <- 1
	w.Write([]byte("OK"))
}
```

当你 `go run example.go` 的时候，打开htop或top，查看这个服务，在我的Ubuntu 14.04环境下，CPU占用到了100%.

当把 default 注释了变成

```go
func worker() {
	for {
		select {
		case d := <-c:
			fmt.Println(d)
			// default:
		}
	}
}
```

CPU占用率就恢复了正常。这时候，`worker()`是阻塞在了 `case d：= <-c: ` 上，这种阻塞并不会占用CPU的调度处理，CPU会闲置或去处理别的任务。直到channel中有数据了，会唤醒该channel的数据结构中对应的goroutine，设置为runnable的状态，该goroutine的调度才会继续进行。

##### 结论： 将default: 删除即可

##### default并非是罪恶之人

只是在上述的使用情况下，default 变成了不必要的了。 当你的使用方式并非如此时，比如下面的方法：

```go
func TryRun() {
  select {
  case a := <-c:
    //...
  // case t := time.After(10 * time.Second):
  //   return
  default:
    return
  }
}
```

外层没有for死循环，当 `<-c` 阻塞时，default 会里执行return，而起到立即释放goroutine的效果，或者你可以加一定的超时机制。要不然，在大量使用goroutine的时候，极有可能造成goroutine泄露。

Golang 设计了便捷的goroutine的使用方式: "go 一下"，方便的进行并发处理。并且设计了使用select 方法来进行调度处理。一定程度上节省了并发编程的难度。

不过，我们还是需要对其原理和底层结构有所掌握，这样才能写出合适的代码。

参考:

```
https://about.sourcegraph.com/go/understanding-channels-kavya-joshi/
https://www.youtube.com/watch?v=KBZlN0izeiY
```
