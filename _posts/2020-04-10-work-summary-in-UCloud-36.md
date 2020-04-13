---
layout: post
title: 最近工作总结(35)
date:  2020-03-06 16:25:06
categories: Work
image: /assets/images/post.jpg
---

### 修改Go Module相关环境变量
```
export GOPROXY=https://goproxy.cn,direct
export GOSUMDB=sum.golang.google.cn  (此地址未被墙)
```

### tcpkali 进行websocket的压力测试
```
tcpkali --ws -c 100 -m 'hello world!!13212312!' -r 10k localhost:8081
```

### map的并发崩溃示例,不要随便打印输出context
>https://www.xargin.com/map-concurrent-throw/

```go
package main

import (
	"sync"
	"time"
)

type resp struct {
	k string
	v string
}

func main() {
	res := fetchData()
    log.Print(res)
}

func rpcwork() resp {
	// do some rpc work
	return resp{}
}

func fetchData() (map[string]string, error) {
	var result = map[string]string{} // result is k -> v
	var keys = []string{"a", "b", "c"}
	var wg sync.WaitGroup
	var m sync.Mutex
	for i := 0; i < len(keys); i++ {
		wg.Add(1)

		go func() {
			m.Lock()
			defer m.Unlock()
			defer wg.Done()

			// do some rpc
			resp := rpcwork()

			result[resp.k] = resp.v
		}()
	}

	waitTimeout(&wg, time.Second)
	return result, nil
}

func waitTimeout(wg *sync.WaitGroup, timeout time.Duration) bool {
	c := make(chan struct{})
	go func() {
		defer close(c)
		wg.Wait()
	}()
	select {
	case <-c:
		return false // completed normally
	case <-time.After(timeout):
		return true // timed out
	}
}
```
线上会偶现崩溃(concurrent write and iteration)
这里的 waitgroup 使用不恰当，若下游系统发生超时时，该 waitgroup 其实并没有完成，这也就意味着，其子任务也并没有全部完成。虽然在 fetchData 内部对 map 的修改加了写锁，但若下游超时，在 fetchData 返回后，fetchData 内部启动的 goroutine 仍然可能对返回的 map 进行修改。
当 map 对象同时进行加锁的 write 和不加锁的读取时，也会发生崩溃。不加锁的读取发生在什么地方呢？其实就是这里例子的 log.Print。如果你做个 json.Marshal 之类的，效果也差不多。
至于为什么是偶发，超时本来也不是经常发生的，看起来这个 bug 就变成了一个偶现 bug。
和这个 bug 类似的还有在打印 context 对象的时候，参考这里。
我们再顺便控诉一下 Go 本身，这种 map 并发崩溃的 bug 对很多人造成了困扰，按说崩溃的时候会打印导致崩溃的 goroutine 栈，但为什么还是一个值得总结的问题呢？
是因为 Go 在崩溃时，其实并不能完整地打印导致崩溃的因果关系，参考这里。
这个 issue 中同时也给了下面这段代码，只有在 go run -race 时，才能看到导致 throw 的真正原因

不要随便打印输出context，因为context底层是map存储，go的map读写不加锁有并发问题，而在外层用fmt 或没有锁的log打印和无锁read一样，会报并发读写崩溃
