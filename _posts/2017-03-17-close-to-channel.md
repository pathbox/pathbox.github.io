---
layout: post
title: simple examples make you be close to channel in Go
date:   2017-03-17 17:03:06
categories: Go
image: /assets/images/post.jpg
---

example1

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	c := make(chan int)
	go func() {
    fmt.Println("in")
		time.Sleep(3 * time.Second)
		fmt.Println("before received")
		fmt.Println(<-c) // 这里在阻塞， 这里会先执行 ready
	}()

	c <- 1
	fmt.Println("after received")
}

// in
// before received
// fatal error: all goroutines are asleep - deadlock!

// send in the main goroutinue
// 如果读取操作在子goroutinue， 写操作在main goroutinue， 读取操作会需要先执行，并且阻塞main goroutinue
// 如果 channel 读取操作和写操作 都在子goroutinue 则谁先就谁先ready
```

example2

```go

package main

import (
	"fmt"
	"time"
)

func main() {
	c := make(chan int)
	go func() {
		time.Sleep(1 * time.Second)
		fmt.Println("before send")
		c <- 100
	}()

	fmt.Println(<-c) // here is waiting for the c <- 100

	fmt.Println("after received")
}

// before send
// 100
// after received

// c <- in sub goroutinue, <- c in main goroutinue, c <- block main goroutinue
// no deadlock. 子goroutinue中的 c<- 100 阻塞了main goroutinue
```

example3

```go

package main

import (
	"fmt"
	"time"
)

func main() {
	c := make(chan int)
	go func() {
		time.Sleep(1 * time.Second)
		fmt.Println("before received")
		fmt.Println(<-c)
	}()

	go func() {
		time.Sleep(2 * time.Second)
		fmt.Println("before send")
		c <- 1
	}()

	time.Sleep(2 * time.Second)
	fmt.Println("after received")
}

// before received
// before send
// 1
// after received

// send is in the sub goroutinue
// send is be ready first, then received
// receive is not blocking. This is a nice way.
// but main goroutinue don't wait for two of them,
// c <- 1 block <-c, send first then received
// <-c wait for c<-1

```

example4

```go

ackage main

import (
	"fmt"
	"time"
)

func main() {
	c := make(chan int)
	go func() {
		time.Sleep(1 * time.Second)
		fmt.Println("before received")
		fmt.Println("print channel", <-c) // 这里在阻塞， 这里会先执行 ready
	}()

	go func() {
		time.Sleep(2 * time.Second)
		fmt.Println("before send")
		c <- 1
	}()

	time.Sleep(3 * time.Second)

	fmt.Println("after received")
}

// before received
// before send
// print channel 1
// after received

// send is in the sub goroutinue
// receive is in the sub goroutinue

// receive is be ready first, then send
// receive is blocking until send the data, it is not good
// so if send is done before receive, it is a good way
```

example5

```go

package main

import (
	"fmt"
	"time"
)

func main() {
	c := make(chan int)
	go func() {
		time.Sleep(2 * time.Second)
		fmt.Println("before received")
		fmt.Println("print channel", <-c) // 这里在阻塞， 这里会先执行 ready
	}()

	go func() {
		time.Sleep(3 * time.Second)
		fmt.Println("before send")
		c <- 1
	}()

	time.Sleep(1 * time.Second)

	fmt.Println("after received")
}

// after received
// main goroutinue will not wait for two sub goroutinue, they will be killed when main goroutinue is over
```

example6

```go

package main

import (
	"fmt"
	"time"
)

func main() {
	c := make(chan int)
	go func() {

		fmt.Println("before received")
		fmt.Println(<-c) // 这里在阻塞，直到 数据过来
	}()
	time.Sleep(1 * time.Second)
	fmt.Println("before send")
	c <- 1
	time.Sleep(2 * time.Second)
	fmt.Println("after received")
}

// before received
// before send
// 1
// after received


// fmt.Println(<-c) <-c is blocking, until c <- 1
// 新开的goroutine首先去读channel,可是由于channel中没有值，所以它被阻塞了，直到main中向channel发送值，goroutine才拿到它想要的值并继续运行。
```
