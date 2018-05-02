---
layout: post
title: TCP三次握手四次挥手总结
date: 2018-04-26 21:00:06
categories: Summary
image: /assets/images/post.jpg
---

### 三次握手建立连接

1. 服务器必须准备好接受外来的连接。这通常通过调用socket、bind和listen这三个函数来完成，我们称为被动打开（passive open）
2. 客户通过调用connect发起主动打开（active open）。客户TCP发送一个SYN（同步）分节，它告诉服务器将在待建立的连接中发送的数据的初始序列号。通常SYN分节不懈怠数据，其所在IP数据报只含有一个IP首部、一个TCP首部及可能有的TCP选项。
3. 服务器必须确认（ACK）客户的SYN，同时自己也发送一个SYN分节，它含有服务器将在同一连接中发送的数据的初始序列号。服务器在单个分节中发送SYN和对客户SYN的ACK确认。
4. 客户收到服务器发送的SYN之后，必须向服务器发送ACK分节，确认服务器的SYN

这种交换至少需要3个分组，因此称之为TCP的三路握手（three-way handshake）

![tcp]( /assets/images/TCP/tcp1.png "Optional title")

### 四次挥手断开连接

1. 某个应用进程首先调用close，我们称该端执行主动关闭（active close）。该端的TCP发送一个FIN分节，表示数据发送完毕。
2. 接收到这个FIN的对端执行被动关闭（passive close）。这个FIN由TCP确认。它的接收作为一个文件结束符（end-of-file）传递给接收端应用进程（放在已排队等候该应用进程接收的任何其他数据之后），因为FIN的接收意味着接收应用进程在相应连接上再无额外数据可接收。
3. 一端时间后，接收到这个文件结束符的应用进程将调用close关闭它的套接字。这导致它的TCP也发送一个FIN。
4. 接收这个最终FIN的原发送端TCP（即执行主动关闭的那一端）确认这个FIN。

既然每个方向都需要一个FIN和一个ACK，因此通常需要4个分节。某些情况下：步骤1步骤1的FIN随数据一起发送；另外，步骤2和步骤3发送的分节都出自执行被动关闭那一端，有可能合并成一个分节。

类似SYN，一个FIN也占据1个字节的序列号空间。因此，每个FIN的ACK确认号就是这个FIN的序列号加1。

在步骤2和步骤3之间，从执行被动关闭一端到执行主动关闭一端流动数据是可能的。这称为半关闭（half-close）

![tcp]( /assets/images/TCP/tcp2.png "Optional title")

### 11种TCP状态图

![tcp]( /assets/images/TCP/tcp3.png "Optional title")

![tcp]( /assets/images/TCP/tcp4.png "Optional title")

```
状态：     描述
CLOSED：无连接是活动的或正在进行
LISTEN：服务器在等待进入呼叫
SYN_RECV：一个连接请求已经到达，等待确认
SYN_SENT：应用已经开始，打开一个连接
ESTABLISHED：正常数据传输状态
FIN_WAIT1：应用说它已经完成
FIN_WAIT2：另一边已同意释放
ITMED_WAIT：等待所有分组死掉
CLOSING：两边同时尝试关闭
TIME_WAIT：另一边已初始化一个释放
LAST_ACK：等待所有分组死掉
```

### TIME_WAIT状态

执行主动关闭的那一端经历了这个状态。该端点停留在这个状态的持续时间是最长分节生命期（maximum segment lifetime, MSL）的两倍，2MSL。任何TCP实现都必须为MSL选择一个值，一般是30s或2分钟。所以2MSL有可能是1分钟和4分钟，这是一个经验值。

TIME_WAIT状态有两个存在的理由：

- 可靠地实现TCP全双工连接的终止

在主动关闭方发送的最后一个 ack(fin) ，有可能丢失，这时被动方会重新发
fin, 如果这时主动方处于 CLOSED 状态 ，就会响应 rst 而不是 ack。所以
主动方要处于 TIME_WAIT 状态，而不能是 CLOSED

- 允许老的重复分节在网络中消逝

防止上一次连接中的包，迷路后重新出现，影响新连接
（经过2MSL，上一次连接中所有的重复包都会消失）

当系统并发量突然增大，很有可能是受到了攻击，通常会导致大量TCP连接处于`TIME_WAIT`状态。会有以下问题发生：

+ 三次握手建立连接、四次握手断开连接都会对性能有损耗；
+ 断开的连接断开不会立刻释放，会等待2MSL的时间，大概是1分钟；
+ 大量TIME_WAIT会占用内存，一个连接实测是3.155KB

解决方案：

- 所以合理设置 keepAlive，能够降低短连接的频繁创建
- 为服务设计连接池(例如为MySQL配置连接池)
- 开启快速回收`TIME_WAIT` TCP连接设置

```
sudo vim /etc/sysctl.conf

编辑文件，加入以下内容：
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30

然后执行/sbin/sysctl -p让参数生效。

net.ipv4.tcp_syncookies = 1表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；

net.ipv4.tcp_tw_reuse = 1表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；

net.ipv4.tcp_tw_recycle = 1表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。

net.ipv4.tcp_fin_timeout修改系統默认的TIMEOUT时间
```

开启快速回收`TIME_WAIT` TCP连接设置，自然就违背了原来`TIME_WAIT` 存在的两个理由，特别是在网络繁忙的情况下有可能会出现一些问题
