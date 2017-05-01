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

// send in the main goroutine
// 如果读取操作在子goroutine， 写操作在main goroutine， 读取操作会需要先执行，并且阻塞main goroutine
// 如果 channel 读取操作和写操作 都在子goroutine 则谁先就谁先ready
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

// c <- in sub goroutine, <- c in main goroutine, c <- block main goroutine
// no deadlock. 子goroutine中的 c<- 100 阻塞了main goroutine
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

// send is in the sub goroutine
// send is be ready first, then received
// receive is not blocking. This is a nice way.
// but main goroutine don't wait for two of them,
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

// send is in the sub goroutine
// receive is in the sub goroutine

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
// main goroutine will not wait for two sub goroutine, they will be killed when main goroutine is over
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

example7

```go

package main

import (
	"fmt"
)

func main() {
	naturals := make(chan int)
	squares := make(chan int)

	go func() {
		for x := 0; x < 100; x++ {
			naturals <- x
		}
		close(naturals)
	}()

	go func() {
		for x := range naturals {
			squares <- x * x
		}
		close(squares)
	}()

	for x := range squares {
		fmt.Println(x)
	}
}

// 发送完成后，可以关闭channel，关闭后所有对这个channel的写操作都会panic,而读操作依旧可以进行，当所有值都读完后，继续读该channel会得到zero value

```

deadlock example8

```go

package main

import (
	"fmt"
	"time"
)

func main() {
	c := make(chan int)
	go func() {
		time.Sleep(3 * time.Second)
		fmt.Println("before received")

	}()
	fmt.Println("before send")
	c <- 1
	fmt.Println("after received")
}

// before send
// before received
// fatal error: all goroutines are asleep - deadlock!

// goroutine 1 [chan send]:
// main.main()
//   /Users/pathbox/code/learning-go/src/channel/example/example_lock1.go:16 +0x10c
// exit status 2

// send in main goroutine
// no receive, send the data to channel, deadlock!

```

no deadlock example9

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
		fmt.Println("receive", <-c)
		fmt.Println("after received")

	}()

	time.Sleep(1 * time.Second)
	fmt.Println("done")
}

// before received
// done

// no send, receive will get nil, receive <-c in sub goroutine
// fmt.Println("receive", <-c) and fmt.Println("after received") don't run
// no deadlock

```

example10

```go

package main

import (
	"fmt"
	"time"
)

func main() {
	c := make(chan int)
	go func() {
		fmt.Println("before send")
		c <- 100
		fmt.Println("after send")

	}()

	time.Sleep(1 * time.Second)
	fmt.Println("done")
}

// before send
// done

// no receive, send c <- 100 in sub goroutine
// fmt.Println("after send") don't run
// no deadlock

```

deadlock example11

```go

package main

import (
	"fmt"
	"time"
)

func main() {
	c := make(chan int)
	go func() {
		fmt.Println("before send")

	}()

	<-c

	time.Sleep(1 * time.Second)
	fmt.Println("done")
}

// before send
// fatal error: all goroutines are asleep - deadlock!

// goroutine 1 [chan receive]:
// main.main()
//   /Users/pathbox/code/learning-go/src/channel/example/example_lock4.go:15 +0x7f
// exit status 2

// receive <-c in main goroutine
// no send
// deadlock!

```

如果 main goroutine中有channel操作，但是没有子goroutine对channel操作，deadlock发生死锁，因为main goroutine
被channel操作阻塞了。
如果 子 goroutine中有channel操作，但是没有其他goroutine队channel操作，no deadlock。main goroutine
没有被channel操作阻塞。 子 goroutine会自动消亡
