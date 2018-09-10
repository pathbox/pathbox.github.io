---
layout: post
title: 最近工作总结(十九)
date:   2018-09-03 11:12:06
categories: Work
image: /assets/images/post.jpg
---

### F in 0x 16进制
在16进制中(0x)，F表示为二进制是： `1111`，整数值为：15。16进制中的每个元素为4个bit位，一个字节是8个bit位

### SQL error in Golang
在Golang项目中使用SQL语句，也许语句错误，但是没法通过编译报错，只能通过某个请求或执行操作触发了这个SQL查询才能报错

### Redis常见性能问题和解决方案

1. Master最好不要做任何持久化工作，如RDB内存快照和AOF日志文件
2. 如果数据比较重要，某个Slave开启AOF备份数据，策略设置为每秒同步一次
3. 为了主从赋值的速度和连接稳定性，Master和Slave最好在同一个局域网内
4. 尽量避免在压力很大的主库上增加从库(也许你需要深夜操作)
5. 主从复制不需要用图状结构，全从Master复制，Master的压力会增加。可以用单向链表结构更为稳定：Master <- Slave1 <- Slave2 <- Slave3...

这样的结构方便解决单点故障问题，实现Slave对Master的替换。如果Master挂了，可以立刻启用Slave1做Master，其他不变。但这样时效性高的Slave只有一台Slave1，当Master和Slave1同时挂掉了，才有可能出现较多的数据丢失的情况
