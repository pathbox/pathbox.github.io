---
layout: post
title: 最近工作总结(39)
date:  2020-07-01 15:45:06
categories: Work
image: /assets/images/post.jpg
---

### 尽量不要使用SELECT *
1. 不需要的列会增加数据传输时间和网络开销。需要解析更多的对象、字段、权限、属性等内容，在SQL语句复杂，硬解析较多的情况下，会对数据库造成沉重的负担，大的文本会增加网络开销
2. 对无用的打字单会增加io操作。长度超过728字节的时候，会先把超出的数据序列化到另一个地方，因此读取这条记录会增加一次io操作(MySQL InnoDB)
3. 失去MySQL优化器"覆盖索引"策略优化的可能性。首先要通过辅助索引过滤数据，然后再通过聚集索引获取所有的列，这就多了一次b+树查询。原本可能只通过辅助索引即可拿到所需要的字段数据

### Go http库 重用底层 TCP 连接需要注意读取完body并关闭

在结合实际的场景之后，我发现其实有的时候问题出在我们并不总是会去读取完整个http.Response 的 Body。为什么这么说呢？
在常见的 API 开发的业务逻辑中，我们会定义一个 JSON 的对象来反序列化 http.Response 的 Body，但是通常在反序列化这个回复之前，我们会做一些 http 的 StatusCode 检查，比如当 StatusCode 为 200 的时候，我们才去读取 http.Response 的 Body，如果不是 200，我们就直接返回一个包装好的错误。比如下面的模式：
```go
resp, err := http.Get("http://www.example.com")
if err != nil {
    return err
}
defer resp.Body.Close()

if resp.StatusCode == http.StatusOK {
    var apiRet APIRet
    decoder := json.NewDecoder(resp.Body)
    err := decoder.Decode(&apiRet)
    // ...
}
```
如果代码是按照上面的这种方式写的话，那么在请求异常的时候，会导致大量的底层 TCP 无法重用，所以我们稍微改进下就可以了。
```go
resp, err := http.Get("http://www.example.com")
if err != nil {
    return err
}
defer resp.Body.Close()

if resp.StatusCode == http.StatusOK {
    var apiRet APIRet
    decoder := json.NewDecoder(resp.Body)
    err := decoder.Decode(&apiRet)
    // ...
}else{
    io.Copy(ioutil.Discard, resp.Body)  // 关键的一步，帮我们读取完body
    // ...
}
```
我们通过直接将 http.Response 的 Body 丢弃掉就可以了。

### Kafka 的consumer offset数据过大容易导致的启动加载问题
Kafka内部会有`topic: __consumer_offsets`, 这个topic存储offset信息。当__consumer_offsets分区数据巨大，且分布不均，Kafka启动或重启时，加载__consumer_offsets数据就会非常的慢，很有可能导致启动超时，从而Kafka服务没法启动。

修改__consumer_offsets的cleanup.policy=delete，保留时间为15天，减少topic保存的数据量，减少Kafka加载压力

Kafka在合ZooKeeper连接时，如果由于网络等原因，可能会导致没法连接上ZooKeeper而发生重启。

### 使用canal和kafka进行数据库同步
为每个表配置分区key，每个表对应一个partition，以保证按照binlog数据顺序进行同步。是的这样会牺牲并发性。

### 如何为Kafka集群选择合适的Partitions数量
>https://blog.csdn.net/oDaiLiDong/article/details/52571901?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase

### epoll存在惊群效应
>惊群效应会影响：多进程/线程的唤醒，涉及到的一个问题是上下文切换问题。频繁的上下文切换带来的一个问题是数据将频繁的在寄存器与运行队列中流转。极端情况下，时间更多的消耗在进程/线程的调度上，而不是执行。
accept是队列方式处理，解决了惊群效应，accept确实应该只能被一个进程调用成功，但是epoll的情况就比较复杂，epoll监听的文件描述符，
除了可能后续被accept调用外，还可能是其他网络IO事件的监听对象，那其他网络IO是否只能由一个进程处理我们是不得知的。
所以linux对epoll并没有就惊群效应做修复，而是放之，让用户层自己做处理。比如：Nginx accept mutex锁，在请求来的时候只有获得accept mutex锁的worker才会唤醒去处理这个请求
Nginx 主体的思想是通过锁的形式来处理这样问题。我们每个进程在监听 FD 事件之前，我们先要通过 ngx_trylock_accept_mutex 去获取一个全局的锁。如果拿锁成功，那么则开始通过 ngx_process_events 尝试去处理事件。如果拿锁失败，则放弃本次操作。所以从某种意义上来讲，对于某一个 FD ，Nginx 同时只有一个 Worker 来处理 FD 上的事件。从而避免惊群。

epoll惊群问题在内核版本要在 Linux Kernel 4.5通过增加一个 EPOLLEXCLUSIVE 标志位，进行了优化，但是只保证唤醒的进程数小于等于我们开启的进程数，而不是直接唤醒所有进程，也不是只保证唤醒一个进程。很多时候我们还是要依靠应用层自身的设计来解决

https://zhuanlan.zhihu.com/p/60966989

### 经典云计算架构 IaaS、PaaS、SaaS
IaaS基础设施即服务:IaaS层为基础设施运维人员服务，提供计算、存储、网络及其其他基础资源，云平台使用者可以在上面部署和运行包括操作系统和应用程序在内的任意软件，无需为基础设施的管理而分心
Paas平台即服务:PaaS层为应用开发人员服务，提供支撑应用运行所需要的软件运行时环境、相关工具服务，入数据库服务、日志服务、监控服务等，让应用开发者可以专注于核心业务的开发
SaaS软件即服务:SaaS层为一般用户服务，提供了一套完整可用的软件系统，让一般用户无需关注技术细节，只需要通过浏览器、应用客户端等方式就能使用部署在云上的应用服务
