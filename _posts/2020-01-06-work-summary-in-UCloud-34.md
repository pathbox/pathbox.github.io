---
layout: post
title: 最近工作总结(34)
date:  2020-01-06 11:25:06
categories: Work
image: /assets/images/post.jpg
---

### 用struct{}{}来表示空或无用值能节省空间,struct{}{}的size是0

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	am := make(map[string]bool)
	bm := make(map[string]struct{})
	am["1"] = true
	am["2"] = true
	am["3"] = true
	am["4"] = true
	bm["1"] = struct{}{}
	bm["2"] = struct{}{}
	bm["3"] = struct{}{}

	fmt.Printf("bool size: %d\n", unsafe.Sizeof(true))
	fmt.Printf("struct size: %d\n", unsafe.Sizeof(struct{}{}))
	fmt.Printf("am size: %d\n", unsafe.Sizeof(am))
	fmt.Printf("bm size: %d\n", unsafe.Sizeof(bm))
}

/*
bool size: 1
struct size: 0
am size: 4
bm size: 4
*/
```

### 孤儿进程被PID 1进程接收
子进程的结束和父进程的运行是一个异步的过程， 即父进程永远不知道子进程到底什么时候结束。如果创建子进程的父进程退出，那么这个子进 程就成了没人管的孩子，俗称孤儿进程。为了避免孤儿进程退出时无法释放所占用的资源而僵 死，进程号为 1 的进程 init 就会接受这些孤儿进程。

### iptables学习

http://www.zsythink.net/archives/tag/iptables/page/2/

### 图论:哥尼斯堡桥问题
“能够一笔画成的通路(通路问题)，所有顶点都是偶点(度数为偶数)或有两个奇点(度数为奇数)”

### 二分查找是一种利用2的指数爆炸💥性质进行的查找，效率极其高

### 一个好的接口报错，当参数必填性、参数类型报错都要给予提示

### fasthttp存在的并发问题
https://cloud.tencent.com/developer/news/462918
>fasthttp使用了一个ctxPool来维护RequestCtx，每次请求都先去ctxPool中获取。如果能获取到就用池中已经存在的，如果获取不到，new出一个新的RequestCtx。这也就是fasthttp性能高的一个主要原因，复用RequestCtx可以减少创建对象所有的时间以及减少内存使用率。但是随之而来的问题是：如果在高并发的场景下，如果整个请求链路中有另起的goroutine，前一个RequestCtx处理完成业务逻辑以后(另起的协程还没有完成)，立刻被第二个请求使用，那就会发生前文所述的错乱的request body。

通过复制一个新的request来解决这个问题
```go
newRequest := &fasthttp.Request{}
ctx.Request.CopyTo(newRequest)

body := newRequest.Body()
```
