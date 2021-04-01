---
layout: post
title: 最近工作总结(42)
date:  2020-10-01 18:15:06
categories: Work
image: /assets/images/post.jpg

---

 

### Elasticsearch+HBase的存储方案

Elasticsearch中存储需要的索引字段，完整的数据存储在HBase。HBase适合海量数据的在线存储，不适合复杂的搜索，但是简单的根据id或者范围查询性能上没有问题。从Elasticsearch中进行复杂搜索得到对应的doc id，然后再到HBase中查询完整的数据返回给前端。这样，Elasticsearch中只存储必要的索引字段，能够更节省内存，使得Elasticsearch能更高效的使用内存(filesystem cache)，从而获得更高的查询性能

### scroll 的分页方式

类似于微博中，下拉刷微博，刷出来一页一页的分页数据。性能会比上面说的那种分页性能要高很多很 多，基本上都是毫秒级的。缺点是：不能随意跳到任何一页的场景

### Redis持久化 AOF和RDB相结合

Redis 支持同时开启开启两种持久化方式，我们可以综合使用 AOF 和 RDB 两种持久化机 制，用 AOF 来保证数据不丢失，作为数据恢复的第一选择; 用 RDB 来做不同程度的冷备， 在 AOF 文件都丢失或损坏不可用的时候，还可以使用 RDB 来进行快速的数据恢复

### 限流降级的作用

防止系统服务或数据库完全被打挂而僵死，无法处理任何新请求。数据库绝对不会死，限流组件确保了每秒只有多少个请求能通过。 只要数据库不死，就是说，对用户来说，2/5 的请求都是可以被处理的。 只要有 2/5 的请求可以被处理，就意味着你的系统没死，对用户来说，可能就是点击几次刷 不出来页面，但是多点几次，就可以刷出来了

### 用MQ缓解数据库并发写操作

可能你还是会出现高并发写的场景，比如说一个业务操作里要频繁搞数据 库几十次，增删改增删改，那高并发绝对搞挂你的系统，你要是用 redis 来承载写那肯 定不行，人家是缓存，数据随时就被 LRU 了，数据格式还无比简单，没有事务支持。所以该用 mysql 还得用 mysql 啊。用 MQ 吧，大量的写请求灌入 MQ 里，排队慢慢玩儿， ，控制在 mysql 承载范围之内。所以你得考虑考虑你的项目里，那些承 载复杂写业务逻辑的场景里，如何用 MQ 来异步写，提升并发性。MQ 单机抗几万并发也是 ok 的

### 从大量数据中查找重复数据(字符串or整数)

1. hash取余分治到小一些的文件
2. 使用hashmap查重过滤
3. 对于大量整数查重，使用bitmap方式，或者布隆过滤器



### ProtoBuf的优点

其实 PB 之所以性能如此好，主要得益于两个：它使用 proto 编译器，自动进行序列化 和反序列化，速度非常快，应该比 XML 和 JSON 快上了 20~100 倍； 它的数据压缩 效果好，就是说它序列化后的数据量体积小。因为体积小，传输起来带宽和速度上会有优化。



### 对订单付款幂等性的设计

一个订单可以分为：下单和付款两个部分。对于每个请求必须有一个唯一的标识，举个栗子：订单支付请求，肯定得包含订单 id，一 个订单 id 最多支付一次，对吧。 每次处理完请求之后，必须有一个记录标识这个请求处理过了。常见的方案是在 mysql 中 记录个状态，比如支付之前记录一条这个订单的支付流水。 每次接收请求需要进行判断，判断之前是否处理过。比如说，如果有一个订单已经支付 了，就已经有了一条支付流水，那么如果重复发送这个请求，则此时先插入支付流水， orderId 已经存在了，唯一键约束生效，报错插入不进去的。然后你就不用再扣款了。

实际运作过程中，你要结合自己的业务来，比如说利用 Redis，用 orderId 作为唯一键。只有成 功插入这个支付流水，才可以执行实际的支付扣款。

要求是支付一个订单，必须插入一条支付流水，order_id 建一个唯一键 unique key 。你在支 付一个订单之前，先插入一条支付流水，order_id 就已经进去了。你就可以写一个标识到 Redis 里面去， set order_id payed ，下一次重复请求过来了，先查 Redis 的 order_id 对应的 value，如果是 payed 就说明已经支付过了，你就别重复支付了



###TiDB 没有间隙锁

TiDB和MySQL悲观锁中的差异之一:

当无法保证符合过滤条件的数据唯一时：

- MySQL 会锁住过滤条件能涵盖到的所有行：范围锁，全表锁。
- TiDB 只会对读取到的行加锁



### JWT 续期策略

由于JWT token是不存储在存储层的，所以在过期时间之后的JWT就无法使用。但是此时客户端可能并未下线，或者登出操作，客户端希望对JWT的过期无感，所以需要对JWT进行续期操作。

对JWT的续期方式：

后端对每次请求校验JWT现在的过期时间如果还有10分钟就过期了，告知网关(或者在后端进行处理)，网关重新分配新的JWT添加到header中，返回给客户端。客户端将新的JWT存入header中



### 由于物理设备电缆不稳定导致的grpc长连接关闭问题

近来服务时不时忽然大量报错: `code = Unavailable desc = transport is closing`。导致grpc的请求失败。如果failfast 设置为false的话，应该会重试的；并且 连接关闭之后grpc.clientConn也会维护这个状态，所以不应该出现这个问题才对。

网上的一个解决方案：

```go
服务端
grpc.KeepaliveParams(keepalive.ServerParameters{
  MaxConnectionIdle: 5 * time.Minute, //这个连接最大的空闲时间，超过就释放，解决proxy等到网络问题（不通知grpc的client和server）
}

grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
  MinTime:             5 * time.Second,
  PermitWithoutStream: true,
})
     
客户端
grpc.WithKeepaliveParams(keepalive.ClientParameters{
  // send pings every 10 seconds if there is no activity
  Time: 10 * time.Second,
  // wait 1 second for ping ack before considering the connection dead
  Timeout: time.Second,
  // send pings even without active streams
  PermitWithoutStream: true,
})
```

问题没有解决。最后运维部门排查出是由于k8s服务集群机房的某根电缆不够稳定，导致网络传输上出现异常问题。换了这根电缆，观察两周时间，报错不再发生



### kubelet启动失败

(k8s v.1.19)执行kubectl的各个命令报错：

```sh
The connection to the server 10.23.192.27:6443 was refused - did you specify the right host or port?
```

之后查资料排查下来，是kubelet启动失败。

systemctl status kubelet 得到PID 23993

journalctl _PID=23993 | vim - 命令查看报错日志

```
kubelet cgroup driver: “cgroupfs” is different from docker cgroup driver: “systemd”
```

vim /etc/docker/daemon.json

```json
{
  "registry-mirrors": ["https://registry.docker-cn.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
```



kubelet和docker二者的cgroup driver不一致。kubelet使用的是cgroupfs，而docker中配置的是systemd。

##### cgroupfs 和systemd的区别

>cgroups 的全称是 Linux Control Groups，主要作用是限制、记录和隔离进程组（process groups）使用的物理资源（cpu、memory、IO 等）cgroup 内核功能没有提供任何的系统调用接口，而是对 linux vfs 的一个实现，因此可以用类似文件系统的方式进行操作systemd、lxc、docker 这些封装了 cgroups 的软件也能让你通过它们定义的接口控制 cgroups 的内容

 新版的k8s建议使用systemd的配置

修改kubelet 的cgroup driver为：systemd

1. **vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf** 

   在KUBELET_KUBECONFIG_ARGS 后面追加 --cgroup-driver=systemd

2. **vim /var/lib/kubelet/config.yaml**

   配置cgroupDriver: systemd

3. **vim /var/lib/kubelet/kubeadm-flags.env(这一步不操作也可以)**

KUBELET_KUBEADM_ARGS="--cgroup-driver=systemd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.2"

https://my.oschina.net/comics/blog/3158924

https://www.thegeekdiary.com/troubleshooting-kubectl-error-the-connection-to-the-server-x-x-x-x6443-was-refused-did-you-specify-the-right-host-or-port/

https://blog.csdn.net/Andriy_dangli/article/details/85062983



### 保证幂等性的思考

从发送端和消费端两方面思考

1. 同一个请求避免并发重复发送
2. 同一个消息体避免因为服务重启等原因重复发送
3. 在消息体中设置唯一ID表示，这样在消费端可根据该标识来识别是否处理过该消息
4. 消费端的代码逻辑尽量做到能够兼容幂等性
5. 对并发的消息可能以加锁的方式进行处理



### 使用自增id做主键还是uuid

自增的主键的值是顺序的,所以Innodb把每一条记录都存储在一条记录的后面。当达到页面的最大填充因子时候(innodb默认的最大填充因子是页大小的15/16,会留出1/16的空间留作以后的   修改)：

1.下一条记录就会写入新的页中，一旦数据按照这种顺序的方式加载，主键页就会近乎于顺序的记录填满，提升了页面的最大填充率，不会有页的浪费

2.新插入的行一定会在原有的最大数据行下一行,mysql定位和寻址很快，不会为计算新行的位置而做出额外的消耗

3.减少了页分裂和碎片的产生

4.顺序的方式存储记录速度会更快。并且在SELECT范围查询时，有更大几率用更少的IO操作就把所需要的数据从磁盘中取到放入内存

因为uuid相对顺序的自增id来说是毫无规律可言的,新行的值不一定要比之前的主键的值要大,所以innodb无法做到总是把新行插入到索引的最后,而是需要为新行寻找新的合适的位置从而来分配新的空间。

这个过程需要做很多额外的操作，数据的毫无顺序会导致数据分布散乱，将会导致以下的问题：

1. 写入的目标页很可能已经刷新到磁盘上并且从缓存上移除，或者还没有被加载到缓存中，innodb在插入之前不得不先找到并从磁盘读取目标页到内存中，这将导致大量的随机IO

2. 因为写入是乱序的,innodb不得不频繁的做页分裂操作,以便为新的行分配空间,页分裂导致移动大量的数据，一次插入最少需要修改三个页以上

3. 由于频繁的页分裂，页会变得稀疏并被不规则的填充，最终会导致数据会有碎片

在把随机值（uuid和雪花id）载入到聚簇索引(innodb默认的索引类型)以后,有时候会需要做一次OPTIMEIZE TABLE来重建表并优化页的填充，这将又需要一定的时间消耗。

结论：使用innodb应该尽可能的按主键的自增顺序插入，并且尽可能使用单调的增加的聚簇键的值来插入新行



自增id也会存在以下几点问题：

1. 可以根据数据库的自增id获取到你的业务增长信息，很容易分析出你的数据情况。业务应该避免暴露主键id信息

2. 对于高并发的负载，innodb在按主键进行插入的时候会造成明显的锁争用，主键的上界会成为争抢的热点，因为所有的插入都发生在这里，并发插入会导致间隙锁竞争

3. Auto_Increment锁机制会造成自增锁的抢夺,有一定的性能损失



### 倒排索引就是关键词到文档的映射

 每个关键词都对应着一系列的文档，这些中都出现了关键词



### 磁盘顺序IO与随机IO性能区别原因简记

磁盘的读取时间主要由三部分组成：

1. 寻道时间，表示磁头在不同磁道之间移动的时间

2. 旋转延迟，表示在磁道找到时，中轴带动盘面旋转合适的扇区开头处

3. 传输时间，表示表示盘面继续转动，实际读取数据的时间。

   7200转/min，旋转一周需要8.33ms，寻道约10ms，所以整个磁盘读取时间在一个磁道上是10ms级的

####顺序读写和随机读写对于机械硬盘来说为什么性能差异巨大？

顺序读写=读取一个大文件

随机读写=读取多个小文件

顺序读写比随机读写快的原因

1. 顺序读写，主要时间花费在了传输时间，而这个时间两种读写可以认为是一样的。

   随机读写，需要多次寻道和旋转延迟。而这个时间可能是传输时间的许多倍。

   随机写会导致磁头不停地换磁道，造成效率的极大降低；顺序写磁头几乎不用换磁道，或者换道的时间很短

2. 顺序读写，磁盘会预读，预读即在读取的起始地址连续读取多个页面

（本次不需要的页面也读取了，这样以后用时就不用再读取，当一个页面用到时，大多数情况下，它周围的页面也会被用到） 

而随机读写，因为数据没有在一起，将预读浪费掉了(或是预读起到的效果很小)。

3. 另一个原因是文件系统的overhead。

读写一个文件之前，得一层层目录找到这个文件，以及做一堆属性、权限之类的检查。

写新文件时还要加上寻找磁盘可用空间的耗时。

对于小文件，这些时间消耗的占比就非常大

>固态硬盘顺序比随机快的原因:
>
>理论上来说，它不应该存在明显的随机写与顺序写的速度差异，因为它就是一块支持随机寻址的存储芯片，没有寻道和旋转盘片的开销，但是随机写实际上还是比顺序写要慢。这是由于其存储介质**闪存**的一些特性导致的，简单来说：
>1、闪存不支持in-place update：你更新一个数据，不可以直接在原有数据上改，而要写到新的空白的地方，并把原有数据标记为失效。
>2、标记失效的数据不是浪费空间么？可以将其清除。但是闪存上清除操作的最小单位是一个大块，大约128K-256K的大小。一次清除会影响到还未标记失效的有用的数据，要先把它们移走。
>这种感觉就如同你在网格纸上写一篇文章，一格一格往下写，只能写在空白的格子里；但是你若要清除之前写的内容，只能整行擦除。非常难受而且浪费空间对吧？所以固态硬盘里实现了垃圾回收算法，用来更好地利用存储空间，同时减少数据迁移，保护闪存寿命。
>那么随机写显然比顺序写带来更大的碎片化，从而带来更多的垃圾回收开销、数据迁移开销，自然就比顺序写要慢了

https://zhuanlan.zhihu.com/p/68750796

https://zhuanlan.zhihu.com/p/67192901

### 限流算法：漏桶 vs 令牌桶

> 漏桶算法和令牌桶算法本质上都是为了做流量整形（Traffic Shaping）或速率限制（Rate Limiting），避免系统因为大流量而被打崩，但两者核心差异在于限流的方向是相反的。
>
> 令牌桶限制的是流量的平均流入速率，并且允许一定程度的突然性流量，最大速率为桶的容量和生成 token 的速率。而漏桶限制的是流量的流出速率，是相对固定的。
>
> 在某些场景中，漏桶算法并不能有效的使用网络资源，因为漏桶的漏出速率是相对固定的，所以在网络情况比较好，没有拥塞的状态下，漏桶依然是限制住的，并没有办法放开量。
>
> 而令牌桶算法则不同，其能够是限制平均速率的同时支持一定程度的突发流量。
>
> 
>
> 两者主要区别在于“漏桶算法”能够强行限制数据的传输速率，而“令牌桶算法”在能够限制数据的平均传输速率外，还允许某种程度的突发传输。在“令牌桶算法”中，只要令牌桶中存在令牌，那么就允许突发地传输数据直到达到用户配置的门限，所以它适合于具有突发特性的流量
>
> https://gocn.vip/topics/11108

### TiDB Int类型限制超过，使用BIGINT

```
线上报错：Error 1690: constant 2147483647 overflows int
是因为数据超过了TiDB int类型的限制。
INTEGER` 类型，别名 `INT`。占4个字节1字节8位，2^(4*8)=4294967296，有符号数的范围是 `[-2147483648, 2147483647]`。无符号数的范围是 `[0, 4294967295]，并且即使修改int的长度也没有效果。
需要修改为BIGINT类型，占8个字节2^(8*8)=18446744073709551616：alter table t_user_opt_log modify column mycloumn BIGINT(20); 可以online DDL 不会锁表，耗时1s
```

### 告警排查简记

1. 是什么告警，资源类告警：查看CPU、内存、存储、网络、IO等指标。服务类告警：查看接口异常，请求量、延迟等是否出现高峰，定位对应接口
2. 查看是否有服务挂了，或是在重启
3. 搜索日志：搜索是否有很多Error报错，定位具体报错信息；搜索对应接口的延迟情况，分析是什么操作有大量延迟，比如：慢SQL、调用第三方接口服务耗时多、死锁
4. 在微服务集群服务中搜索日志信息，需要借助链式搜索方式，比如opentracing。每个链式调用应该要有一个request_uuid，以此来查找整个调用链