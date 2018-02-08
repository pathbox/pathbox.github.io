---
layout: post
title: 在Linux系统中监控系统TCP数量
date:   2017-08-25 15:21:06
categories: Tool
image: /assets/images/post.jpg
---

### netstat

+ 查看哪些IP连接本机
netstat -nat

+ 查看某个端口
netstat -nat | grep 80

```
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN     
tcp        1      0 192.168.1.225:44342     218.11.0.240:80         CLOSE_WAIT
tcp        0      0 192.168.1.225:38028     101.37.135.82:5880      ESTABLISHED
tcp        0      0 127.0.0.1:42916         127.0.0.1:80            ESTABLISHED

```

+ 统计某个端口访问连接数
netstat -nat | grep 80 | wc -l

+ 统计已连接上的，状态为 ESTABLISHED
netstat -na|grep ESTABLISHED|wc -l

+ 对各种状态的连接数分组统计结果
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'

```
FIN_WAIT2 46         从远程TCP等待连接中断请求
CLOSING 7            没有任何连接状态
SYN_RECV 3           表示正在等待处理的请求数
CLOSE_WAIT 154       等待从本地用户发来的连接中断请求
TIME_WAIT 3268       表示处理完毕，等待超时结束的请求数
ESTABLISHED 38449    成功建立的连接，表示正常数据传输状态
LAST_ACK 267         等待原来的发向远程TCP的连接中断请求的确认
FIN_WAIT1 71         等待远程TCP连接中断请求，或先前的连接中断请求的确认

TCP连接状态详解
LISTEN： 侦听来自远方的TCP端口的连接请求
SYN-SENT： 再发送连接请求后等待匹配的连接请求
SYN-RECEIVED：再收到和发送一个连接请求后等待对方对连接请求的确认
ESTABLISHED： 代表一个打开的连接
FIN-WAIT-1： 等待远程TCP连接中断请求，或先前的连接中断请求的确认
FIN-WAIT-2： 从远程TCP等待连接中断请求
CLOSE-WAIT： 等待从本地用户发来的连接中断请求
CLOSING： 等待远程TCP对连接中断的确认
LAST-ACK： 等待原来的发向远程TCP的连接中断请求的确认
TIME-WAIT： 等待足够的时间以确保远程TCP接收到连接中断请求的确认
CLOSED： 没有任何连接状态
```

```
如发现系统存在大量TIME_WAIT状态的连接，通过调整内核参数解决，
vim /etc/sysctl.conf
编辑文件，加入以下内容：
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
然后执行 /sbin/sysctl -p 让参数生效。

net.ipv4.tcp_syncookies = 1 表示开启SYN cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；
net.ipv4.tcp_tw_reuse = 1 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
net.ipv4.tcp_tw_recycle = 1 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
net.ipv4.tcp_fin_timeout 修改系統默认的 TIMEOUT 时间
```
