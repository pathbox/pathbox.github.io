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

原因：没有从GOPATH/src进入到该项目进行build，从GOPATH/src 目录进去到该项目文件夹后，再进行build，就会都以vendor的包为加载优先了

### vendor和GOPATH谁优先使用，GOPATH优先使用

如果一个包在vendor和GOPATH下面都存在那么谁会优先使用呢。
结论是：

优先使用vendor目录下面的包。
如果vendor下面没有搜索到，再搜索GOPATH下面的包。
要么完整使用vendor下面的包，要么完整使用GOPATH下面的包，不会混合使用：
假如一个函数定义再GOPATH下面的包里，而没有定义在vendor路径下的同名包里，那么调用者就会报函数未定义错误，因为调用者如果找到有vendor路径下面的包，就不会去找GOPATH下面的包了。

### 高效能人士的七个习惯

1、积极主动--个人愿景的原则，即采取主动，为自己过去、现在及未来的行为负责，并依据原则及价值观，(而非情绪或外在环境)来下决定，创造改变，积极面对一切；

2、以终为始--自我领导的原则，个人、团队和组织在做任何计划时，均先拟出愿景和目标，并据此塑造未来，全心投注于自己最重视的原则、价值观、关系及目标之上；

3、要事第一--自我管理的原则，即实质的创造，是你的目标、愿景、价值观及要事的组织与实践；

4、双赢思维--人际领导的原则，双赢思维是一种基于互敬、寻求互惠的思考框架与心意，鼓励我们解决问题，并协助个人找到互惠的解决办法，是一种信息、力量、认可及报酬的分享；

5、知彼解己--移情沟通的原则，当我们改变以回答的心态，而以了解对方的心态去聆听别人，便能开启真正的沟通，增进彼此关系，知彼需要仁慈心，解己需要勇气，能平衡两者，则可大幅提升沟通的效率；

6、统合综效--创造性合作的原则，统合综效是创造第三种选择，既非按照我的方式，亦非你的方式，而是第三种远胜过个人之见的办法，它是互相尊重的成果，使整体获得一加一大于二的成效。

7、不断更新--平衡的自我更新的原则，在四个基本生活方面，即生理、社会、情感、心智中，不断更新自己，这个习惯提升了其它六个习惯的实施效率。

### curl cip.cc 得到服务器的外网IP地址信息

### 重复两次注册了 driver MySQL 会报错

```
panic: sql: Register called twice for driver mysql
goroutine 1 [running]:
database/sql.Register(0x14ba2ae, 0x5, 0x1549280, 0x189a258)
	/usr/local/go/src/database/sql/sql.go:51 +0x189
github.com/go-sql-driver/mysql.init.0()
	/github.com/go-sql-driver/mysql/driver.go:161 +0x5c
```

### 验证码的作用
能够在一定程度上防止循环暴力匹配,得到信息

### 一个错误的SQL语句可能不报错且导致全表查询
```sql
select * from person where  name = "xxx" like '%xxx%' limit 10;
```

### 快速记大小端的区别

一段连续的内存:起始地址start和结束地址end

- 大端模式: 位权大的的进制数值在start地址开始，位权大的排列在位权小的字节之前
- 小端模式: 位权小的的进制数值在start地址开始，位权大的排列在位权小的字节之后

### Redis Cluster Config 配置例子
https://juejin.im/post/5b625b9be51d4519956759d0

主从复制功能的详细步骤可以分为7个步骤：

- 设置主服务器的地址和端口
- 建立套接字连接
- 发送PING命令
- 身份验证
- 发送端口信息
- 同步
- 命令传播

全量重同步的步骤如下
- 主节点收到从服务器的全量重同步请求时，主服务器便开始执行bgsave命令，同时用一个缓冲区记录从现在开始执行的所有写命令。
- 当主服务器的bgsave命令执行完毕后，会将生成的RDB文件发送给从服务器。从服务器接收到RDB文件时，会将数据文件保存到硬盘，然后加载到内存中。
- 主服务器将缓冲区所有缓存的命令发送到从服务器，从服务器接收并执行这些命令，将从服务器同步至主服务器相同的状态。
```
info replication 可用查看集群Info

192.168.17.103:6379> slaveof 192.168.17.101 6379
OK  角色已经变成从服务器
```

#### config
```
################################# REPLICATION #################################

# slaveof <主服务器ip> <主服务器端口>
# slaveof <masterip> <masterport>

# masterauth <主服务器Redis密码>
# masterauth <master-password>

# 当slave丢失master或者同步正在进行时，如果发生对slave的服务请求
# yes则slave依然正常提供服务
# no则slave返回client错误："SYNC with master in progress"
slave-serve-stale-data yes

# 指定slave是否只读
slave-read-only yes

# 无硬盘复制功能
repl-diskless-sync no

# 无硬盘复制功能间隔时间
repl-diskless-sync-delay 5

# 从服务器发送PING命令给主服务器的周期
# repl-ping-slave-period 10

# 超时时间
# repl-timeout 60

# 是否禁用socket的NO_DELAY选项
repl-disable-tcp-nodelay no

# 设置主从复制容量大小，这个backlog 是一个用来在 slaves 被断开连接时存放 slave 数据的 buffer
# repl-backlog-size 1mb

# master 不再连接 slave时backlog的存活时间。
# repl-backlog-ttl 3600

# slave的优先级
slave-priority 100

# 未达到下面两个条件时，写操作就不会被执行
# 最少包含的从服务器
# min-slaves-to-write 3
# 延迟值
# min-slaves-max-lag 10

```
