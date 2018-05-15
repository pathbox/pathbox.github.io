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


### 数据库三星索引原则

##### 索引的优点

1 索引大大减少了服务器需要扫描的数据量

2 索引可以帮助服务器避免排序和临时表

3 索引可以将随机I/O变成顺序I/O

##### 索引的缺点

1.降低了带索引的数据列插入/修改/删除的速度

2.索引越多，占据的磁盘空间越大

##### 三星索引

索引将相关的记录放到一起则获得一星；如果索引的数据顺序和查找的排列顺序一致则获得二星；如果索引中的列包含了查询中需要的全部列则获得三星。

##### 索引选择性

索引的选择性指不重复的索引值（基数，cardinality）和数据表的记录总数（#T）的比值，范围在1/#T和1之间。索引的选择性越接近1，查询效率就越高。唯一性索引的选择性为1，因此性能也是最好的。

##### 前缀索引

对于很长的字符列，创建索引可能索引变得很大且慢，一种方法就是使用哈希索引，另一种方法就是对列的前缀进行索引，这样可大大节约索引空间，提供索引效率，但会降低索引的选择性。

对于Blob，Text或者很长的varchar类型的列，必须使用前缀索引，因为mysql不允许索引这些列的完整长度。

建立前缀索引的要点：选择足够长的前缀以保证较高的选择性，使得前缀索引选择性尽量接近于索引整个列

### type is func() in Golang

```go
package main

import "fmt"

// 定义一个func类型，定义好参数和返回值，具体这个selfDoHandler的逻辑代码是怎样的，在初始化这个类型具体实例的时候定义
type selfDoHandler func(param string) string

func main() {

	var sdh selfDoHandler //定义一个变量sdh 为 selfDoHandler类型

	sdh = func(p string) string { // 这个func 的参数和返回值和 selfDoHandler 类型一致就可以
		s := "Hello" + p // 具体实现selfDoHandler 方法逻辑
		return s
	}

	fmt.Println(sdh(" World"))
}

/*
可以用在哪？ 定义一个map
map[string]selfDoHandler

根据key，可以设置不同的selfDoHandler，但是他们的参数和返回值都和selfDoHandler一致，只是具体实现的时候的代码逻辑可以不一样

这样不同的key相当于进行了不同的逻辑处理。具体的逻辑实现还可以用定义一个func的方式
*/
```

##### 理解回调函数

- 第一步：先注册函数，可以注册一个或多个函数，但不执行它们
- 第二步：当程序执行到某处时，才调用其中一个或多个函数
