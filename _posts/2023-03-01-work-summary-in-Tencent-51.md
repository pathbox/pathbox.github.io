---
layout: post
title: 最近工作总结(51)
date:  2023-03-01 21:00:00
categories: Work
image: /assets/images/post.jpg


---

### ​elasticsearch 小记

1. 有20个节点，将副本分片设置为19，这样每个节点都有20个分片(加上主分片)的数据，请求到该节点后，直接查询数据就可以返回了，省去了路由到别的节点带来的网络消耗。不过这样会增加磁盘空间的使用，毕竟数据多存了
2. 副本数默认是1，别忘了要进行调大，否则只会有2个节点有数据，其他节点没有数据，所有请求都转发到有数据的这两个节点上，承担了极大的请求压力
3. 建议有2个ES集群，实现物理隔离
4. 准备重启整个ES集群的脚本，可以快速重启集群
5. 使用别名方式切换所有是100%切换流量，可以使用Nacos，配置流量分配比例来实现灰度切换流量


### golang 的map选择 sync.Map 还是 concurrent-map

```
通过以上的代码分析，我们看出sync.Map的这个机制，是一个想追求无锁读写的结构，它最好的运行方式是读永远都命中read，写只命中dirty，这用能不用任何锁机制就能做到map读写。而它最差的运行状态是read和dirty不断做替换和清理动作，性能就无法达到预期。而什么时候可能出现最差运行状态呢？- 大量的写操作和大量的读操作。大量读写会导致“map的miss标记大于dirty的个数”。 这个时候sync.Map中第一层屏障会失效，dirty就会频繁变动。 而current-map就相当于是一个比较中等中规中矩的方案。它的每次读写都会用到锁，只是这个锁的粒度比较小。它的最优运行方式是我们的所有并发读写都是分散在不同的hash切片中。它的最差运行方式就是我们所有的并发读写都集中在一个hash切片。但是按照实际运行逻辑，这两种极端情况都不会发生。 所以总结下来，concurrent-map 的这段话确实没有骗我们： sync.Map在读多写少性能比较好，而concurrent-map 在key的hash度高的情况下性能比较好。 在无法确定读写比的情况下，建议使用 concurrent-map。
```


### 学习业界难题-“跨库分页”的四种方案

https://cloud.tencent.com/developer/article/1048654

```
方法一：全局视野法

（1）将order by time offset X limit Y，改写成order by time offset 0 limit X+Y

（2）服务层对得到的N*(X+Y)条数据进行内存排序，内存排序后再取偏移量X后的Y条记录

这种方法随着翻页的进行，性能越来越低。

方法二：业务折衷法-禁止跳页查询

（1）用正常的方法取得第一页数据，并得到第一页记录的time_max

（2）每次翻页，将order by time offset X limit Y，改写成order by time where time>$time_max limit Y

以保证每次只返回一页数据，性能为常量。

方法三：业务折衷法-允许模糊数据

（1）将order by time offset X limit Y，改写成order by time offset X/N limit Y/N

方法四：二次查询法
（1）将order by time offset X limit Y，改写成order by time offset X/N limit Y
（2）找到最小值time_min
（3）between二次查询，order by time between $time_min and $time_i_max
（4）设置虚拟time_min，找到time_min在各个分库的offset，从而得到time_min在全局的offset
（5）得到了time_min在全局的offset，自然得到了全局的offset X limit Y
将第二次得到的所有数据排序，知道time_min的offset，然后能够取到 offset X 的数据是哪一个，再往后取Y个数据
缺点，需要两次查询
```

### 一次SQL查询优化的场景：对于大范围查询，可以将范围进行适当缩小，但增加语句的并发，CPU反而是可以降低
背景：一个脚本需要遍历几千万的数据，使用的是`SELECT WHERE id >= 1 AND id <= 4000000 AND status = 0 LIMIT 250` 语句，遍历过的记录会把status更改为1。
开了5个进程并发执行。此时观测MySQL机器的CPU升到了10%，思考是否可以优化，因为语句已经是直接查主键了，所以，语句上没有优化空间。原来是id查询的范围是400w，将其缩小为200w，并且增加到10个进程并发执行，CPU反而降到了5%，而脚本执行速度是快了一倍。查询每个脚本的执行情况，得到id的最小值，将id的查询范围缩小到100w，任然保持10个进行，CPU将低到了3%左右。
总结：对于大范围查询，可以将范围进行适当缩小，但增加语句的并发，CPU反而是可以降低。我理解是范围减少了，底层查和过滤的条数是减少的(即使已经有很多条数是不满足条件不需要了)，所以CPU降低

### 红包预拆分方案

比如现将红包或现金券  拆分好多个token池子(红包池)，对每个用户取模，对应到一个池子，然后加锁，这样能分担缩的压力。对锁粒度细化

### 需要传输或查询大数据时考虑对数据进行压缩和解压处理


### Direct Memory Access直接存储器访问
操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核（kernel），保证内核的安全，操心系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。针对linux操作系统而言，将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF），供内核使用，称为内核空间，而将较低的3G字节（从虚拟地址0x00000000到0xBFFFFFFF），供各个进程使用，称为用户空间，如图2所示。这里还有一个比较重要的概念，叫DMA（Direct Memory Access直接存储器访问），它的作用是处理各种I/O，包括网络I/O和磁盘I/O。CPU是不会直接处理I/O的，这是因为CPU非常宝贵，而I/O是比较耗时的，如果CPU一直等待某一次I/O事件完成，会带来极大的浪费，且性能会急剧下降，因此需要一种机制能够完成I/O，并通知CPU，DMA即是这个角色

边缘触发（edge triggered ET）
对于边缘触发，epoll_wait()只返回一次，即只在该读写事件发生时返回，也就是说如果事件处理函数只读取了该文件描述缓冲区的部分内容时返回，再次调用epoll_wait()，虽然此时该描述符对应缓冲区中还有数据，但epoll_wait()函数不会返回。
水平触发（level triggered LT）
对于水平触发，它不管是否有事件反生，只要文件描述符对应的缓冲区中有数据可读写，epoll_wait()就会返回。

聚簇索引是索引和数据在一起的，不用回表

### 了解MySQL MRR(5.6以上版本)
```
二、Multi-Range Read (MRR)

MRR 的全称是 Multi-Range Read Optimization，是优化器将随机 IO 转化为顺序 IO 以降低查询过程中 IO 开销的一种手段，这对IO-bound类型的SQL语句性能带来极大的提升，适用于range ref eq_ref类型的查询

MRR优化的几个好处

使数据访问有随机变为顺序，查询辅助索引是，首先把查询结果按照主键进行排序，按照主键的顺序进行书签查找

减少缓冲池中页被替换的次数

批量处理对键值的操作

在没有使用MRR特性时

第一步 先根据where条件中的辅助索引获取辅助索引与主键的集合，结果集为rest

1select key_column, pk_column from tb where key_column=x order by key_column

第二步 通过第一步获取的主键来获取对应的值


for each pk_column value in rest do:

select non_key_column from tb where pk_column=val

使用MRR特性时

第一步 先根据where条件中的辅助索引获取辅助索引与主键的集合，结果集为rest

1select key_column, pk_column from tb where key_column = x order by key_column

第二步 将结果集rest放在buffer里面(read_rnd_buffer_size 大小直到buffer满了)，然后对结果集rest按照pk_column排序，得到结果集是rest_sort

第三步 利用已经排序过的结果集，访问表中的数据，此时是顺序IO.

1select non_key_column fromtb where pk_column in (rest_sort)

在不使用 MRR 时，优化器需要根据二级索引返回的记录来进行“回表”，这个过程一般会有较多的随机IO, 使用MRR时，SQL语句的执行过程是这样的：

优化器将二级索引查询到的记录放到一块缓冲区中

如果二级索引扫描到文件的末尾或者缓冲区已满，则使用快速排序对缓冲区中的内容按照主键进行排序

用户线程调用MRR接口取cluster index，然后根据cluster index 取行数据

当根据缓冲区中的 cluster index取完数据，则继续调用过程 2) 3)，直至扫描结束

通过上述过程，优化器将二级索引随机的 IO 进行排序，转化为主键的有序排列，从而实现了随机 IO 到顺序 IO 的转化，提升性能

此外MRR还可以将某些范围查询，拆分为键值对，来进行批量的数据查询，如下：

SELECT * FROM t WHEREkey_part1 >= 1000 ANDkey_part1 < 2000ANDkey_part2 = 10000;

表t上有二级索引(key_part1, key_part2)，索引根据key_part1,key_part2的顺序排序。

若不使用MRR：此时查询的类型为Range，sql优化器会先将key_part1大于1000小于2000的数据取出，即使key_part2不等于10000，带取出之后再进行过滤，会导致很多无用的数据被取出

若使用MRR：如果索引中key_part2不为10000的元组越多，最终MRR的效果越好。优化器会将查询条件拆分为（1000,1000），（1001,1000），... （1999,1000）最终会根据这些条件进行过滤
```