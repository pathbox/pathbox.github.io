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

### 4A（统一安全管理平台解决方案）
1. 集中帐号（account）管理
4A功能结构图
4A功能结构图
为用户提供统一集中的帐号管理，支持管理的资源包括主流的操作系统、网络设备和应用系统；不仅能够实现被管理资源帐号的创建、删除及同步等帐号管理生命周期所包含的基本功能，而且也可以通过平台进行帐号密码策略，密码强度、生存周期的设定。

2. 集中认证(authentication)管理
可以根据用户应用的实际需要，为用户提供不同强度的认证方式，既可以保持原有的静态口令方式，又可以提供具有双因子认证方式的高强度认证（一次性口令、数字证书、动态口令），而且还能够集成现有其它如生物特征等新型的认证方式。不仅可以实现用户认证的统一管理，并且能够为用户提供统一的认证门户，实现企业信息资源访问的单点登录。

3. 集中权限(authorization)管理
可以对用户的资源访问权限进行集中控制。它既可以实现对B/S、C/S应用系统资源的访问权限控制，也可以实现对数据库、主机及网络设备的操作的权限控制，资源控制类型既包括B/S的URL、C/S的功能模块，也包括数据库的数据、记录及主机、网络设备的操作命令、IP地址及端口。

4. 集中审计(audit)管理
将用户所有的操作日志集中记录管理和分析，不仅可以对用户行为进行监控，并且可以通过集中的审计数据进行数据挖掘，以便于事后的安全事故责任的认定。
