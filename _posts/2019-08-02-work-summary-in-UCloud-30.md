---
layout: post
title: 最近工作总结(30)
date:  2019-08-02 11:55:06
categories: Work
image: /assets/images/post.jpg
---

### 简单理解行数据库和列数据库

>如果使用的是行式数据库，你正好需要对一行数据进行操作时，数据库的性能是最好的，因为仅一个页面被放到了内存中（这只是个示例，实际上操作系统会带来不止一个页面的数据）。但是，如果你只是想对表中所有数据的某一列的数据做一些操作，这意味着你将花费时间去访问每一行，可你用到的仅是一行中的小部分数据。此时使用的若是列式数据库，就可以方便快捷的访问数据，因为每一列的信息都是存储在一起的。蛋疼的时候来了，假如现在使用的是列式存储，你又想获取 Alice 的所有信息，那你又必须去读取大量的列（页面）来获取所有的数据。正因为如此，才有了行式存储和列式存储的使用场景的区别。当你的核心业务是 OLTP 时，一个行式数据库，再加上优化操作，可能是个最好的选择。当你的核心业务是 OLAP 时，一个列式数据库，绝对是更好的选择

### fsync同步数据到存储设备上
fsync 在 Linux 中的意义在于同步数据到存储设备上。大多数块设备的数据都是通过缓存进行的，将数据写到文件上通常将该
数据由内核复制到缓存中，如果缓存尚未写满，则不将其排入输出队列上，而是等待其写满或者当内核需要重用该缓存时，再 将该缓存排入输出队列，进而同步到设备上 。 这种策略的好处是减少了磁盘读写次数，不足的地方是降低了文件内容的更新速 度，使其不能时刻同步到存储设备上， 当系统发生故障时，这种机制很有可能导致了文件内容的丢失。因此，内核提供了 fsync
接口，用户可以根据自己的需要通过此接口更新数据到存储设备上.

### Rabbitmq 官方conf example

https://github.com/rabbitmq/rabbitmq-server/blob/master/docs/rabbitmq.conf.example

### RabbitMQ 只要求在集群中至少有一个磁盘节点
RabbitMQ 只要求在集群中至少有一个磁盘节点，所有其他节点可以是内存节点。当节点加入或者离开集群时，它们必须将变更通知到至少一个磁盘节点。如果只有一个磁盘节点，而且不凑巧的是它刚好崩溃了，`那么集群可以继续发送或者接收消息，但是不能执行创建队列、交换器、绑定关系、用户，以及更改权限、添加或删除集群节点的操作了`。也就是说，如果集群 中唯一的磁盘节点崩溃，集群仍然可以保持运行，但是直到将该节点恢复到集群前，你无法更改任何东西。所以在建立集群的时候应该保证有两个或者多个磁盘节点的存在。

### Namespace的作用
>“在同一个名称空间中，同一类型资源对象的名称必须具有唯一性。名称空间通常用于实现租户或项目的资源隔离，从而形成逻辑分组”

### Nagle算法规则
 `Nagle算法就是为了尽可能发送大块数据，尽可能的利用网络带宽，避免网络中充斥着许多小数据块，而导致网络拥塞`
- 如果包长度达到MSS，则允许发送
- 如果该包含有FIN，则允许发送
- 设置了TCP_NODELAY选项，则允许发送
- 为设置TCP_CORK选项时，若所有发出去的小数据吧（包长度小于MSS）均被确认，则允许发送
- 上述条件都未满足，但发生了超时（一般为200ms），则立即发送

### 一次k8s服务访问网关接口异常问题
pod的环境是IPv6，之前没有发生过，该接口一直正常使用，这周开始，该接口执行10次才有1次成功
使用postman调用该接口，一切正常

在pod中执行curl -v得到异常的结果:
```
Trying ipv6_ip_a...
* Connected to api.myadmin.com (ipv6_ip_a) port 80 (#0)  http 正常的

About to connect() to api.myadmin.com port 80 (#0)       
*   Trying ipv6_ip_b...            http  不正常的
* Connected to api.myadmin.com (ipv6_ip_b) port 80 (#0)
```

发现当调用失败时，Trying的是`ipv6_ip_b`这个ipv6的地址。
最终排查到的结果: pod服务访问 api.myadmin.com会转到广州网关,  这个是根据我们内网dns返回的ipv4地址转换的
但这台网关机器IPv6没有配置监听443。所以从pod访问时走IPv6会报错，将这台网关机器IPv6配置监听443即可

### INSERT INTO A SELECT * FROM B
`INSERT INTO A SELECT * FROM B`
如果需要这样简便的操作，同步记录需要 `A B两表的字段数量和类型完全一致`

### golang xor

```go
// xorToken XORs tokens ([]byte) to provide unique-per-request CSRF tokens. It
// will return a masked token if the base token is XOR'ed with a one-time-pad.
// An unmasked token will be returned if a masked token is XOR'ed with the
// one-time-pad used to mask it.

func xor(a, b []byte) []byte {
	n := len(a)
	if len(b) < n {
		n = len(b)
	}

	res := make([]byte, n)

	for i := 0; i < n; i++ {
		res[i] = a[i] ^ b[i]
	}

	return res
}
```
