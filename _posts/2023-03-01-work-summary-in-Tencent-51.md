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