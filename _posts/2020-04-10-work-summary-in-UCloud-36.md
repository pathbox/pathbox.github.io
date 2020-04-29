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

### 二叉搜索树的特性：BST 的中序遍历是升序序列

### grpc超时传递
>客户端客户端发起 RPC 调用时传入了带 timeout 的 ctx
gRPC 框架底层通过 HTTP2 协议发送 RPC 请求时，将 timeout 值写入到 grpc-timeout HEADERS Frame 中
服务端接收 RPC 请求时，gRPC 框架底层解析 HTTP2 HEADERS 帧，读取 grpc-timeout 值，并覆盖透传到实际处理 RPC 请求的业务 gPRC Handle 中
如果此时服务端又发起对其他 gRPC 服务的调用，且使用的是透传的 ctx，这个 timeout 会减去在本进程中耗时，从而导致这个 timeout 传递到下一个 gRPC 服务端时变短，这样即实现了所谓的 超时传递  

### grpc keepalive.ServerParameters 连接设置解释
```go
type ServerParameters struct {
	// MaxConnectionIdle is a duration for the amount of time after which an
	// idle connection would be closed by sending a GoAway. Idleness duration is
	// defined since the most recent time the number of outstanding RPCs became
	// zero or the connection establishment.
  // 例子:链接如果在5分钟都没有请求使用，则会关闭
	MaxConnectionIdle time.Duration // The current default value is infinity.
	// MaxConnectionAge is a duration for the maximum amount of time a
	// connection may exist before it will be closed by sending a GoAway. A
	// random jitter of +/-10% will be added to MaxConnectionAge to spread out
	// connection storms.
  // 例子: 链接无论有在工作还是没有，在1小时后都会关闭
	MaxConnectionAge time.Duration // The current default value is infinity.
	// MaxConnectionAgeGrace is an additive period after MaxConnectionAge after
	// which the connection will be forcibly closed.
  // 在MaxConnectionAge到期后，增加的时间段，在这时间之后链接被强制关闭
	MaxConnectionAgeGrace time.Duration // The current default value is infinity.
	// After a duration of this time if the server doesn't see any activity it
	// pings the client to see if the transport is still alive.
	// If set below 1s, a minimum value of 1s will be used instead.
	Time time.Duration // The current default value is 2 hours.
	// After having pinged for keepalive check, the server waits for a duration
	// of Timeout and if no activity is seen even after that the connection is
	// closed.
	Timeout time.Duration // The current default value is 20 seconds.
}
```

进行测试:

```go
keepalive.ServerParameters{
		MaxConnectionIdle:     5 * time.Second,
		MaxConnectionAge:      1 * time.Second,
		MaxConnectionAgeGrace: 5 * time.Second,
		Time:                  5 * time.Second,
		Timeout:               1 * time.Second,
	}
```
并且在代码中sleep(30)秒，客户端没设超时时间，客户端得到了响应:
```
Response Data
Unavailable (14)
transport is closing
```
可以看到，`transport is closing` 链接关闭了，没有得到响应的数据。这个符合预测
一个注意的事情是，链接关闭了，不代表代码不在执继续行了，通过监听日志输出，可以看到sleep之后，代码是继续执行的。
grpc client 设置超时时间，我觉得是很有必要的

### golang 对数组断言操作
```go
func valueToStringSlice(value interface{}) []string {
	strSlice := make([]string, 0)
	switch v := value.(type) {
	case string:
		strSlice = append(strSlice, v)
	case []interface{}: // 数组中的每个元素也是interface, []string 这种是无效的
		for _, val := range v {
			strSlice = append(strSlice, val.(string))
		}
	}
	return strSlice
}
```

### 一个管理系统的基本系统功能
- 用户管理：提供用户的相关配置，新增用户后，默认密码为123456
- 角色管理：对权限与菜单进行分配，可根据部门设置角色的数据权限
- 菜单管理：已实现菜单动态路由，后端可配置化，支持多级菜单
- 部门管理：可配置系统组织架构，树形表格展示
- 岗位管理：配置各个部门的职位
- 字典管理：可维护常用一些固定的数据，如：状态，性别等
- 操作日志：记录用户操作的日志
- 异常日志：记录异常日志，方便开发人员定位错误
- SQL监控：采用druid 监控数据库访问性能，默认用户名admin，密码123456
- 定时任务：整合Quartz做定时任务，加入任务日志，任务运行情况一目了然
- 邮件工具：配合富文本，发送html格式的邮件
- 表单提交：支持HTML和markdown的富文本表单
- 免费图床：使用sm.ms图床，用作公共图片上传使用，该图床不怎么稳定，不太建议使用
- 七牛云存储：可同步七牛云存储的数据到系统，无需登录七牛云直接操作云数据
- 支付宝支付：整合了支付宝支付并且提供了测试账号，可自行测试
