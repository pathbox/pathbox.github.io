---
layout: post
title: 百万并发连接、65536和Linux TCP/IP 性能优化
date:   2017-02-05 17:57:06
categories: concurrency
image: /assets/images/post.jpg
---
##### 65536

2^16 = 65536

端口号在TCP协议字段里，一共两个字节，二进制的16位，意味着有2^16= 65536个端口号可用，但0-1023系统通常保留为知名服务端口，所以最多有64512个端口做为端口池资源。

作为客户端，最多可以 使用64512个端口，也就是同一时刻对同一个服务器同一个端口可以创建64512个TCP连接

而对于服务器来说，它没有所谓的“65536”端口数量的限制。可以有多少客户端连接服务器，取决于CPU、内存等资源的大小。

同一台客户端机子上开n个进程去连同一个服务端进程，因为客户端ip是同一个，为了区分出这些连接，只能使用客户端连接的端口号，那么服务端和一个客户端主机之间的tcp连接数理论上线确实是65536。但是，服务端可以连接n多客户端机子

另一个误解：

accept之后产生的已连接套接字占用了新的端口。这个绝对是错误的，客户端连接上来之后产生的这个socket fd就是用来区分客户端的，里面会填上客户端的ip和端口作为发包用，来自客户端的包也会使用这个fd去读取。

一个TCP连接唯一性确定：

ServerIP => ServerPort <= TCP => ClientIP => ClientPort

##### 百万并发连接

```
/etc/sysctl.conf
sysctl -w fs.file-max=10485760 #系统允许的文件描述符数量10m
sysctl -w net.ipv4.tcp_rmem=1024 #每个tcp连接的读取缓冲区1k，一个连接1k
sysctl -w net.ipv4.tcp_wmem=1024 #每个tcp连接的写入缓冲区1k
#修改默认的本地端口范围
sysctl -w net.ipv4.ip_local_port_range='1024 65535'
sysctl -w net.ipv4.tcp_tw_recycle=1  #快速回收time_wait的连接
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_timestamps=1
#用户单进程的最大文件数，用户登录时生效
echo '* soft nofile 1048576' >> /etc/security/limits.conf
echo '* hard nofile 1048576' >> /etc/security/limits.conf
ulimit -n 1048576 #用户单进程的最大文件描述符数 当前会话生效
```

瓶颈主要是在系统的内存和网卡IO

#####

```ruby
net.core.netdev_max_backlog = 400000
#该参数决定了，网络设备接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目。

net.core.optmem_max = 10000000
#该参数指定了每个套接字所允许的最大缓冲区的大小

net.core.rmem_default = 10000000
#指定了接收套接字缓冲区大小的缺省值（以字节为单位）。

net.core.rmem_max = 10000000
#指定了接收套接字缓冲区大小的最大值（以字节为单位）。

net.core.somaxconn = 100000
#Linux kernel参数，表示socket监听的backlog(监听队列)上限

net.core.wmem_default = 11059200
#定义默认的发送窗口大小；对于更大的 BDP 来说，这个大小也应该更大。

net.core.wmem_max = 11059200
#定义发送窗口的最大大小；对于更大的 BDP 来说，这个大小也应该更大。

net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
#严谨模式 1 (推荐)
#松散模式 0

net.ipv4.tcp_congestion_control = bic
#默认推荐设置是 htcp

net.ipv4.tcp_window_scaling = 0
#关闭tcp_window_scaling
#启用 RFC 1323 定义的 window scaling；要支持超过 64KB 的窗口，必须启用该值。

net.ipv4.tcp_ecn = 0
#把TCP的直接拥塞通告(tcp_ecn)关掉

net.ipv4.tcp_sack = 1
#关闭tcp_sack
#启用有选择的应答（Selective Acknowledgment），
#这可以通过有选择地应答乱序接收到的报文来提高性能（这样可以让发送者只发送丢失的报文段）；
#（对于广域网通信来说）这个选项应该启用，但是这会增加对 CPU 的占用。

net.ipv4.tcp_max_tw_buckets = 10000
#表示系统同时保持TIME_WAIT套接字的最大数量

net.ipv4.tcp_max_syn_backlog = 8192
#表示SYN队列长度，默认1024，改成8192，可以容纳更多等待连接的网络连接数。

net.ipv4.tcp_syncookies = 1
#表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；

net.ipv4.tcp_timestamps = 1
#开启TCP时间戳
#以一种比重发超时更精确的方法（请参阅 RFC 1323）来启用对 RTT 的计算；为了实现更好的性能应该启用这个选项。

net.ipv4.tcp_tw_reuse = 1
#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；

net.ipv4.tcp_tw_recycle = 1
#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。

net.ipv4.tcp_fin_timeout = 10
#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间。

net.ipv4.tcp_keepalive_time = 1800
#表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为30分钟。

net.ipv4.tcp_keepalive_probes = 3
#如果对方不予应答，探测包的发送次数

net.ipv4.tcp_keepalive_intvl = 15
#keepalive探测包的发送间隔

net.ipv4.tcp_mem
#确定 TCP 栈应该如何反映内存使用；每个值的单位都是内存页（通常是 4KB）。
#第一个值是内存使用的下限。
#第二个值是内存压力模式开始对缓冲区使用应用压力的上限。
#第三个值是内存上限。在这个层次上可以将报文丢弃，从而减少对内存的使用。对于较大的 BDP 可以增大这些值（但是要记住，其单位是内存页，而不是字节）。

net.ipv4.tcp_rmem
#与 tcp_wmem 类似，不过它表示的是为自动调优所使用的接收缓冲区的值。

net.ipv4.tcp_wmem = 30000000 30000000 30000000
#为自动调优定义每个 socket 使用的内存。
#第一个值是为 socket 的发送缓冲区分配的最少字节数。
#第二个值是默认值（该值会被 wmem_default 覆盖），缓冲区在系统负载不重的情况下可以增长到这个值。
#第三个值是发送缓冲区空间的最大字节数（该值会被 wmem_max 覆盖）。

net.ipv4.ip_local_port_range = 1024 65000
#表示用于向外连接的端口范围。缺省情况下很小：32768到61000，改为1024到65000。

net.ipv4.netfilter.ip_conntrack_max=204800
#设置系统对最大跟踪的TCP连接数的限制

net.ipv4.tcp_slow_start_after_idle = 0
#关闭tcp的连接传输的慢启动，即先休止一段时间，再初始化拥塞窗口。

net.ipv4.route.gc_timeout = 100
#路由缓存刷新频率，当一个路由失败后多长时间跳到另一个路由，默认是300。

net.ipv4.tcp_syn_retries = 1
#在内核放弃建立连接之前发送SYN包的数量。

net.ipv4.icmp_echo_ignore_broadcasts = 1
# 避免放大攻击

net.ipv4.icmp_ignore_bogus_error_responses = 1
# 开启恶意icmp错误消息保护

net.inet.udp.checksum=1
#防止不正确的udp包的攻击

net.ipv4.conf.default.accept_source_route = 0
#是否接受含有源路由信息的ip包。参数值为布尔值，1表示接受，0表示不接受。
#在充当网关的linux主机上缺省值为1，在一般的linux主机上缺省值为0。
#从安全性角度出发，建议你关闭该功能
```

修改了系统默认的配置，一定要进行严格的压测。并监控线上的业务影响波动。不一定别人给的配置就是适合你的系统的配置。如果有异常情况，宁可使用默认配置。


参考链接

https://www.zhihu.com/question/20831000

https://www.jianshu.com/p/4a58761d758f

http://www.blogjava.net/yongboy/archive/2013/04/11/397677.html

https://zhuanlan.zhihu.com/p/21378825

http://colobu.com/2014/09/18/linux-tcpip-tuning/
