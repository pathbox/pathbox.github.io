---
layout: post
title: 最近工作总结(十五)
date:   2018-05-01 11:24:06
categories: Work
image: /assets/images/post.jpg
---


### 理解互斥锁、读写锁

读写锁
- 只要没有线程持有某个给定的读写锁用于写，那么任意数目的线程可以持有该读写锁用于读
- 仅当没有线程持有某个给定的读写锁用于读或写时，才能分配该读写锁用于写。

>只要没有线程在修改某个给定的数据，那么任意数目的线程都可以拥有该数据的读访问权。仅当没有其他线程在读或修改某给给定的数据时，当前线程才可以修改它

规律：临界区没有加写锁，可以被多个线程获取读锁读和一个线程获取写锁写；资源加了读锁，只能被其他线程获取读锁读。

而互斥锁不分读写，保护临界区某个时段只有一个线程对临界区进行读写操作。相比读写锁，损失了读的并发性能。

### Restarting without closing the socket

Graceful Restart in Golang

- Fork a new process which inherits the listening socket
- The child performs initialization and starts accepting connections on the socket
- Immediately after, child sends a signal to the parent causing the parent to stop accepting connections and terminate

- Fork 一个新的进程，这个进程继承正在监听的socket的所有属性信息
- 子进程执行初始化，之后开始接受新的socket连接
- 之后，子进程马上发送signal信号通知父进程停止接受新的连接然后终止

> https://grisha.org/blog/2014/06/03/graceful-restart-in-golang/

### 数据库四种食物隔离级别先行小记

+ 未提交读(Read Uncommitted): 允许脏读，也就是可能读取到其他会话中未提交事物修改的数据
+ 提交读(Read Committted): 只能读取到已提交的数据。Oracle等多数数据库默认是这个级别
+ 可重复读(Repeated Read): 可重复读。在同一个事物内的查询都是事物开始时刻一致的，InnoDB默认级别。该级别消除了不可重复读，但是还存在幻读
+ 串行读(Serializable): 完全串行化的读，每次读都需要获得表级共享锁，读写互相都会阻塞
---
- 脏读：是指当一个事务A正在访问数据，并且对数据进行了修改，而这个修改还没有提交到数据库中，这时，另一个事务也访问了这个数据，访问到的是A修改前的数据，然后使用了这个数据
- 不可重复读：是指在一个事务内，多次相同条件读取同一数据。在这个事务还没有结束时，另一个事务也访问了同一数据。在第一个事务中的两次读取数据之间，由于第二个事务的修改，那么第一个事务两次读到的数据可能是不一样的。这样就发生了在一个事务内两次读到的数据是不一样的，因此称为是不可重复读
- 可重复读：消除了不可重复读的的情况。在同一个事务内，多次相同条件读取同一数据，不受其他事务对这个数据的修改的影响，都是读取到同样的数据，但不能消除幻读的影响
- 幻读：第一个事务对一个表中的数据进行了修改，这种修改涉及到表中的全部数据行。同时，第二个事务也修改这个表中的数据，这种修改是向表中插入一行或多行新数据。那么，以后就会发生操作第一个事务的用户发现表中还有没有修改的数据行（发现这新增的数据记录），就好象发生了幻觉一样
