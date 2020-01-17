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
