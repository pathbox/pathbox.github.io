---
layout: post
title: 用喜欢和舒服的方式在Golang中使用锁、使用channel自定义锁
date:   2017-08-12 17:05:06
categories: Golang
image: /assets/images/post.jpg
---

众所周知，我们能使用Golang轻松编写并发程序。Golang利用goroutine，让我们编写并发程序变得容易。并发程序中重要的问题之一就是如何正确的处理“竞争资源”或“共享资源”。Golang为我们提供了锁的机制。这篇文章，就简单介绍Golang中锁的使用方法。并且进行错误的使用方法和正确的使用方法的代码示例对比。文章的所以代码示例在：https://github.com/pathbox/learning-go/tree/master/src/lock

我们看第一个栗子：

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct {
	Value int
}

var wg sync.WaitGroup
var mutex sync.Mutex // 声明了一个全局锁
func main() {

	wg.Add(1000)
	counter := &Counter{Value: 0}

	for i := 0; i < 1000; i++ {
		go Count(counter, mutex)
	}
	wg.Wait()
	fmt.Println("Count Value: ", counter.Value)
}

func Count(counter *Counter, mutex sync.Mutex) {
	mutex.Lock()
	defer mutex.Unlock()
	counter.Value++
	wg.Done()
}

/*
输出结果：
Count Value:  982
*/
```

这里声明了一个全局锁 sync.Mutex，然后将这个全局锁以参数的方式代入到方法中，这样并没有真正起到加锁的作用。

正确的方式是：

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct {
	Value int
}

var wg sync.WaitGroup
var mutex sync.Mutex // 声明了一个全局锁
func main() {

	wg.Add(1000)
	counter := &Counter{Value: 0}

	for i := 0; i < 1000; i++ {
		go Count(counter)
	}
	wg.Wait()
	fmt.Println("Count Value: ", counter.Value)
}

func Count(counter *Counter) {
	mutex.Lock()
	defer mutex.Unlock()
	counter.Value++
	wg.Done()
}

/*
输出结果：
Count Value:  1000
*/
```
声明了一个全局锁后，其作用范围是全局。直接使用，而不是将其作为参数传递到方法中。


下一个栗子

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct {
	Value int
}

var wg sync.WaitGroup

func main() {
	var mutex sync.Mutex // 声明了一个非全局锁
	wg.Add(1000)
	counter := &Counter{Value: 0}

	for i := 0; i < 1000; i++ {
		go Count(counter, mutex)
	}
	wg.Wait()
	fmt.Println("Count Value: ", counter.Value)
}

func Count(counter *Counter, mutex sync.Mutex) {
	mutex.Lock()
	defer mutex.Unlock()
	counter.Value++
	wg.Done()
}

/*
输出结果：
Count Value:  954
*/
```

上面栗子中，声明的不是全局锁。然后将这个锁作为参数传入到Count()方法中，这样并没有真正起到加锁的作用。

正确的方式：

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct {
	Value int
}

var wg sync.WaitGroup

func main() {
	mutex := &sync.Mutex{} // 定义了一个锁 mutex,赋值给mutex
	wg.Add(1000)
	counter := &Counter{Value: 0}

	for i := 0; i < 1000; i++ {
		go Count(counter, mutex)
	}
	wg.Wait()
	fmt.Println("Count Value: ", counter.Value)
}

func Count(counter *Counter, mutex *sync.Mutex) {
	mutex.Lock()
	defer mutex.Unlock()
	counter.Value++
	wg.Done()
}

/*
输出结果：
Count Value:  1000
*/
```
这次通过 mutex := &sync.Mutex{}，定义了mutex，然后作为参数传递到方法中，正确实现了加锁功能。

简单的说，在全局声明全局锁，之后这个全局锁就能在代码中的作用域范围内都能使用了。但是，也许你需要的不是全局锁。这和锁的粒度有关。
所以，你可以声明一个锁，在其作用域范围内使用，并且这个作用域范围是有并发执行的，别将锁当成参数传递。如果，需要将锁当成参数传递，那么你传的不是一个锁的声明，而是这个锁的指针。

下面，我们讨论一种更好的使用方式。通过阅读过很多”牛人“写的Go的程序或源码库，在锁的使用中。常常将锁放入对应的 struct 中定义，我觉得这是一种不错的方法。

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct {
	Value int
	sync.Mutex
}

var wg sync.WaitGroup

func main() {

	wg.Add(1000)
	counter := &Counter{Value: 0}

	for i := 0; i < 1000; i++ {
		go Count(counter)
	}
	wg.Wait()
	fmt.Println("Count Value: ", counter.Value)
}

func Count(counter *Counter) {
	counter.Lock()
	defer counter.Unlock()
	counter.Value++
	wg.Done()
}

/*
输出结果：
Count Value:  1000
*/
```

这样，我们声明的不是全局锁，并且这个需要加锁的竞争资源也正是 struct Counter 本身的Value属性，反映了这个锁的粒度。我觉得这是一种很舒服的使用方式（暂不知道这种方式会带来什么负面影响，如果有踩过坑的朋友，欢迎聊一聊这个坑），当然，如果你需要全局锁，那么请定义全局锁。

还可以有更多的使用方式：

```go
// 1.
type Counter struct {
   Value int
   Mutex sync.Mutex
}

counter := &Counter{Value: 0}
counter.Mutex.Lock()
defer counter.Mutex.Unlock()

//2.
type Counter struct {
   Value int
   Mutex *sync.Mutex
}

counter := &Counter{Value: 0, Mutex: &sync.Mutex{}}
counter.Mutex.Lock()
defer counter.Mutex.Unlock()
```

Choose the way you like~

接下来，我们自己尝试创建一个互斥锁。

简单的说，简单的互斥锁锁的原理是：一个线程(进程)拿到了这个互斥锁，在这个时刻，只有这个线程(进程)能够进行互斥锁锁的范围中的"共享资源"的操作，主要是写操作。我们这里不讨论读锁的实现。锁的种类很多，有不同的实现场景和功能。这里我们讨论的是最简单的互斥锁。

我们能够利用Golang 的channel所具有特性，创建一个简单的互斥锁。

/locker/locker.go

```go
package locker

// Mutext struct
type Mutex struct {
	lock chan struct{}
}

// 创建一个互斥锁
func NewMutex() *Mutex {
	return &Mutex{lock: make(chan struct{}, 1)}
}

// 锁操作
func (m *Mutex) Lock() {
	m.lock <- struct{}{}
}

// 解锁操作
func (m *Mutex) Unlock() {
	<-m.lock
}
```

main.go
```go
package main

import (
	"./locker"
	"fmt"
	"time"
)

type record struct {
	lock          *locker.Mutex
	lock_count    int
	no_lock_count int
}

func newRecord() *record {
	return &record{
		lock:          locker.NewMutex(),
		lock_count:    0,
		no_lock_count: 0,
	}
}

func main() {
	r := newRecord()

	for i := 0; i < 1000; i++ {
		go CountWithoutLock(r)
		go CountWithLock(r)
	}
	time.Sleep(2 * time.Second)
	fmt.Println("Record no_lock_count: ", r.no_lock_count)
	fmt.Println("Record lock_count: ", r.lock_count)
}

func CountWithLock(r *record) {
	r.lock.Lock()
	defer r.lock.Unlock()
	r.lock_count++
}

func CountWithoutLock(r *record) {
	r.no_lock_count++
}

/* 输出结果
Record no_lock_count:  995
Record lock_count:  1000
*/
```

locker 就是通过使用channel的读操作和写操作会互相阻塞等待的这个同步性质。
可以简单的理解为，channel中传递的就是互斥锁。一个线程(进程)申请了一个互斥锁(struct{}{})，将这个互斥锁存放在channel中，
其他线程(进程)就没法申请互斥锁放入channel，而处于阻塞状态，等待channel恢复空闲空间。该线程(进程)进行操作”共享资源“，然后释放这个互斥锁(从channel中取走)，channel这时候恢复了空闲的空间，其他线程(进程)
就能申请互斥锁并且放入channel。这样，在某一时刻，只会有一个线程(进程)拥有互斥锁，在操作"共享资源"。
