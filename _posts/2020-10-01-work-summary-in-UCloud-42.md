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

一个订单可以分为：下单和付款两个部分。对于每个请求必须有一个唯一的标识，举个栗子：订单支付请求，肯定得包含订单 id，一 个订单 id 最多支付一次，对吧。 每次处理完请求之后，必须有一个记录标识这个请求处理过了。常见的方案是在 mysql 中 记录个状态啥的，比如支付之前记录一条这个订单的支付流水。 每次接收请求需要进行判断，判断之前是否处理过。比如说，如果有一个订单已经支付 了，就已经有了一条支付流水，那么如果重复发送这个请求，则此时先插入支付流水， orderId 已经存在了，唯一键约束生效，报错插入不进去的。然后你就不用再扣款了。

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