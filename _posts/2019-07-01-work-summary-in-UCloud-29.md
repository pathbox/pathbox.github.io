---
layout: post
title: 最近工作总结(29)
date:  2019-07-01 14:19:06
categories: Work
image: /assets/images/post.jpg
---

### WaitGroup 不能被拷贝传递

在主 goroutine 中 Add(delta int) 索要等待goroutine 的数量。 在每一个 goroutine 完成后 Done() 表示这一个goroutine 已经完成，当所有的 goroutine 都完成后，在主 goroutine 中 WaitGroup 返回返回。

```go
func main(){
    var wg sync.WaitGroup
    var urls = []string{
        "http://www.golang.org/",
        "http://www.google.com/",
    }
    for _, url := range urls {
        wg.Add(1)
        go func(url string) {
            defer wg.Done()
            http.Get(url)
        }(url)
    }
    wg.Wait()
}
```
在Golang官网中对于WaitGroup介绍是A WaitGroup must not be copied after first use,在 WaitGroup 第一次使用后，不能被拷贝

应用示例:
```go
func main(){
 wg := sync.WaitGroup{}
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(wg sync.WaitGroup, i int) {
            fmt.Printf("i:%d", i)
            wg.Done()
        }(wg, i)
    }
    wg.Wait()
    fmt.Println("exit")
}
```
运行:
```
i:1i:3i:2i:0i:4fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
sync.runtime_Semacquire(0xc000094018)
        /home/keke/soft/go/src/runtime/sema.go:56 +0x39
sync.(*WaitGroup).Wait(0xc000094010)
        /home/keke/soft/go/src/sync/waitgroup.go:130 +0x64
main.main()
        /home/keke/go/Test/wait.go:17 +0xab
exit status 2
```
它提示所有的 goroutine 都已经睡眠了，出现了死锁。这是因为 wg 给拷贝传递到了 goroutine 中，导致只有 Add 操作，其实 Done操作是在 wg 的副本执行的。

### 网络发包阻塞的原因
> 来于得到吴军《信息论40讲》：
发送方于是马上把那些包重新发送，结果原来的包还没有发完，现在又要多发很多包，网络就变得更加拥堵，最后无论是发送方还是接收方都会锁死在那里。
你有时打开一个网页，刚刚显示了头上10%的内容就再也打不开下面的内容了。你就在想，即便是网速很慢，只有56K的带宽，等待时间长一点也该传完了吧。
其实不是，因为在网络的某一处信道的容量难以满足传输率的要求后，你的计算机作为接收方很长时间没有收到某个包，就无法发出接收完成的信息，传送信息的服务器就不断重新传输那些没有得到接收确认的数据包。传输就永远无法完成了。

思考如何能够增加个人的信息带宽容量?无论是学习过程中，生活上，与人交往上等等

### docker 容器服务的自动重启
- 查看容器的重启次数: docker inspect -f "{{ .RestartCount }}" my-container
- 获取上一次容器重启时间: docker inspect -f "{{ .State.StartedAt }}" my-container
- 设置容器服务always自动重启: docker run --restart=always my-container-server
- 设置容器服务自动重启策略: docker run --restart=on-failure:10 my-container-server

### redis配置登入密码
配置文件redis.conf增加: `requirepass 123456`

redis-cli: `config set requirepass 123456` 这样不用重启也能增加密码，并且redis重启可以读取redis.conf中的配置内容

```
127.0.0.1:6379> keys *
(error) NOAUTH Authentication required.
127.0.0.1:6379> auth 123456
OK
```

### GOPATH 还是vendor
一个加载配置文件内容的模块包， 有两个方法使用，一个是load方法，只需要在初始的时候调用一次，将配置文件内容加载到全局变量中，另一个是getkey方法，获取对应的配置值。有两个包中用到了配置文件，一个是main，一个是另一个包。main的init方法调用了load，在main处正常获得了配置值，而在另一个包获得的是nil值。代码逻辑上没有错，最后发现了，main出调用的config模块是加载自GOPATH的，而另一个包中调用的config模块加载自vendor，由于另一个包没有调用load方法，所以全局变量中的map值是空的，所以导致了获得的配置值是nil
