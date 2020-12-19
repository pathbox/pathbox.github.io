---
layout: post
title: 最近工作总结(40)
date:  2020-08-04 17:30:06
categories: Work
image: /assets/images/post.jpg
---

### elasticsearch match,term,match_phrase简记
https://blog.csdn.net/sinat_29581293/article/details/81486761

### 同步数据跑的脚本要具有幂等性

### golang channel runtime.chansend
如果当前 Channel 的 recvq 上存在已经被阻塞的 Goroutine，那么会直接将数据发送给当前的 Goroutine 并将其设置成下一个运行的 Goroutine；
如果 Channel 存在缓冲区并且其中还有空闲的容量，我们就会直接将数据直接存储到当前缓冲区 sendx 所在的位置上；
如果不满足上面的两种情况，就会创建一个 runtime.sudog 结构并将其加入 Channel 的 sendq 队列中，当前 Goroutine 也会陷入阻塞等待其他的协程从 Channel 接收数据；
发送数据的过程中包含几个会触发 Goroutine 调度的时机：

发送数据时发现 Channel 上存在等待接收数据的 Goroutine，立刻设置处理器的 runnext 属性，但是并不会立刻触发调度；
发送数据时并没有找到接收方并且缓冲区已经满了，这时就会将自己加入 Channel 的 sendq 队列并调用 runtime.goparkunlock 触发 Goroutine 的调度让出处理器的使用权

https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/

### 非对称加密传输密钥，对称加密传输内容

### SQL查询得到的struct建议所有字段都有对应真实值
例子：
```go
type Person struct {
  Name string
  UserID int
  CompanyID int
}
```
一个SQL的查询方法中，对上述的struct实例实际只取出了Name，UserID两个字段，但实际使用中，有可能会根据IDE的提示，取CompanyID字段，这个字段是0，则导致了后面诡异的问题的出现。所以，建议Person struct的所有Field都取真实的值。
如果CompanyID字段没有用到，则新建一个

```go
type PersonB struct {
  Name string
  UserID int
}
```

### 协程co-routine的优缺点
协程是在用户态的轻量线程。在语言框架层面实现调度，在用户态(协程调度器)进行切换开销比在内核态小

N个协程绑定1个线程，优点就是协程在用户态线程即完成切换，不会陷入到内核态，这种切换非常的轻量快速。但也有很大的缺点，1个进程的所有协程都绑定在1个线程上

缺点：
某个程序用不了硬件的多核加速能力
一旦某协程阻塞，造成线程阻塞，本进程的其他协程都无法执行了，根本就没有并发的能力了。所以协程调度器的调度算法很关键，需要平衡调度策略，避免某一个协程阻塞太久，导致其他协程饥饿。对于IO密集型的场景，协程是能够提升并发性能的。但是对于CPU密集型的场景，则没有太大的作用，只能靠增加CPU核心才能真正提高性能

为什么要让m3和m4自旋，自旋本质是在运行，线程在运行却没有执行G，就变成了浪费CPU. 为什么不销毁现场，来节约CPU资源。因为创建和销毁CPU也会浪费时间，我们希望当有新goroutine创建时，立刻能有M运行它，如果销毁再新建就增加了时延，降低了效率。当然也考虑了过多的自旋线程是浪费CPU，所以系统中最多有GOMAXPROCS个自旋的线程(当前例子中的GOMAXPROCS=4，所以一共4个P)，多余的没事做线程会让他们休眠。

### Golang接口注意点
https://www.jianshu.com/p/07fc95df7e5c

### epoll本质超好的文章系列
https://zhuanlan.zhihu.com/p/64138532

### select有三弊，epoll有三优

>select底层采用数组来管理套接字描述符，同时管理的数量有上限，一般不超过几千个，epoll使用树和链表来管理，同时管理数量可以很大。
>
>select不会告诉你到底哪个套接字来了消息，你需要一个个去询问。epoll直接告诉你谁来了消息，不用轮询。
>
>select进行系统调用时还需要把套接字列表在用户空间和内核空间来回拷贝，循环中调用select时简直浪费。epoll统一在内核管理套接字描述符，无需来回拷贝

### 简记Redis和MongoDB区别
1. Redis 支持的数据结构丰富，包括hash、set、list等。MongoDB 数据结构比较单一，但是支持丰富的数据表达，索引，最类似关系型数据库，支持的查询语言非常丰富
2. 当物理内存够用的时候，性能，redis>mongodb>mysql。mongodb可以存储文件，适合存放大量的小文件，内置了GirdFS 的分布式文件系统
3. MongoDB从1.8版本后，采用binlog方式（MySQL同样采用该方式）支持持久化，增加可靠性；Redis依赖快照进行持久化；AOF增强可靠性；增强可靠性的同时，影响访问性能。可靠性上MongoDB优于Redis
4. Redis适合做缓存，MongoDB比Redis更适合承担持久化数据库的角色。
5. 写操作MongoDB会写写入内存再写入磁盘，MySQL是直接写入磁盘，所以在单词写操作上，MySQL不如MongoDB

### Golang中的struct能不能比较
struct字段属性中有不可比较的类型，则不可以比较，没有则可以
- 可排序的数据类型有三种，Integer，Floating-point，和String
- 可比较的数据类型除了上述三种外，还有Boolean，Complex，Pointer，Channel，Interface和Array
- 不可比较的数据类型包括，Slice, Map, 和Function

### 一个Pod生成经历了什么
https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?
https://github.com/jamiehannaford/what-happens-when-k8s/tree/master/zh-cn

### defer after panic
https://mp.weixin.qq.com/s/XsCTQJWry2UUtAkM4OFbwA
如果代码panic了，之后的代码不会操作。如果加上defer，即使panic，defer也会操作执行

### Redis Cluster集群的扩容和收缩
1.集群扩容
当一个 Redis 新节点运行并加入现有集群后，我们需要为其迁移槽和数据。首先要为新节点指定槽的迁移计划，确保迁移后每个节点负责相似数量的槽，从而保证这些节点的数据均匀。
首先启动一个 Redis 节点，记为 M4。
使用 cluster meet 命令，让新 Redis 节点加入到集群中。新节点刚开始都是主节点状态，由于没有负责的>槽，所以不能接受任何读写操作，后续我们就给他迁移槽和填充数据。
对 M4 节点发送 cluster setslot { slot } importing { sourceNodeId } 命令，让目标节点准备导入槽的数据。 >4) 对源节点，也就是 M1，M2，M3 节点发送 cluster setslot { slot } migrating { targetNodeId } 命令，让源节>点准备迁出槽的数据。
源节点执行 cluster getkeysinslot { slot } { count } 命令，获取 count 个属于槽 { slot } 的键，然后执行步骤>六的操作进行迁移键值数据。
在源节点上执行 migrate { targetNodeIp} " " 0 { timeout } keys { key... } 命令，把获取的键通过 pipeline 机制>批量迁移到目标节点，批量迁移版本的 migrate 命令在 Redis 3.0.6 以上版本提供。
重复执行步骤 5 和步骤 6 直到槽下所有的键值数据迁移到目标节点。
向集群内所有主节点发送 cluster setslot { slot } node { targetNodeId } 命令，通知槽分配给目标节点。为了>保证槽节点映射变更及时传播，需要遍历发送给所有主节点更新被迁移的槽执行新节点。
2.集群收缩
收缩节点就是将 Redis 节点下线，整个流程需要如下操作流程。
首先需要确认下线节点是否有负责的槽，如果是，需要把槽迁移到其他节点，保证节点下线后整个集群槽节点映射的完整性。
当下线节点不再负责槽或者本身是从节点时，就可以通知集群内其他节点忘记下线节点，当所有的节点忘记改节点后可以正常关闭。
下线节点需要将节点自己负责的槽迁移到其他节点，原理与之前节点扩容的迁移槽过程一致。
迁移完槽后，还需要通知集群内所有节点忘记下线的节点，也就是说让其他节点不再与要下线的节点进行 Gossip 消息交换。
Redis 集群使用 cluster forget { downNodeId } 命令来讲指定的节点加入到禁用列表中，在禁用列表内的节点不再发送 Gossip 消息。

### 为什么RedisCluster会设计成16384个槽
1.如果槽位为65536，发送心跳信息的消息头达8k，发送的心跳包过于庞大。

如上所述，在消息头中，最占空间的是 myslots[CLUSTER_SLOTS/8]。 当槽位为65536时，这块的大小是: 65536÷8÷1024=8kb因为每秒钟，redis节点需要发送一定数量的ping消息作为心跳包，如果槽位为65536，这个ping消息的消息头太大了，浪费带宽。

2.redis的集群主节点数量基本不可能超过1000个。

如上所述，集群节点越多，心跳包的消息体内携带的数据越多。如果节点过1000个，也会导致网络拥堵。因此redis作者，不建议redis cluster节点数量超过1000个。 那么，对于节点数在1000以内的redis cluster集群，16384个槽位够用了。没有必要拓展到65536个。

3.槽位越小，节点少的情况下，压缩率高

Redis主节点的配置信息中，它所负责的哈希槽是通过一张bitmap的形式来保存的，在传输过程中，会对bitmap进行压缩，但是如果bitmap的填充率slots / N很高的话(N表示节点数)，bitmap的压缩率就很低。 如果节点数很少，而哈希槽数量很多的话，bitmap的压缩率就很低。

而16384÷8÷1024=2kb
