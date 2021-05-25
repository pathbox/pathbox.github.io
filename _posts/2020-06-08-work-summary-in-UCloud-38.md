---
layout: post
title: 最近工作总结(38)
date:  2020-06-08 11:12:06
categories: Work
image: /assets/images/post.jpg
---

### TiDB binlog 同步到MySQL时，TiDB会把SQL语句进行重新组装，比如UPADATE 拆分成DELETE 然后REPLACE INTO，这样通过binlog恢复或同步数据会有更高的性能吧。但是如果需要binlog原始操作逻辑进行区分的，这种重组方式就不合适了

### K8s变更远程配置中心内容后，需要重新打tag部署
因为老的tag已经生成了老的镜像，不打tag生成新的镜像，重新部署用的还是老的镜像，而老的镜像中的配置文件是没有修改的。所以一定要重新打tag后，生成新的镜像，配置变更才同步

### kafka sarama client OffsetNewest 和 OffsetOldest区别

By default, sarama's Config.Consumer.Offsets.Initial is set to sarama.OffsetNewest. This means that in the event that a brand new consumer is created, and it has never committed any offsets to kafka, it will only receive messages starting from the message after the current one that was written.

If you wish to receive all messages (from the start of all messages in the topic) in the event that a consumer does not have any offsets committed to kafka, you need to set Config.Consumer.Offsets.Initial to sarama.OffsetOldest

### 关于分布式事务简记

在三阶段提交中，如果在第三阶段协调者发送提交请求之后挂掉，并且唯一的接受的参与者执行提交操作之后也挂掉了，这时协调者通过选举协议产生了新的协调者。在二阶段提交时存在的问题就是新的协调者不确定已经执行过事务的参与者是**执行的提交事务还是中断事务**。但是在三阶段提交时，肯定得到了第二阶段的再次确认，那么第二阶段必然是已经正确的执行了事务操作，只等待提交事务了。所以新的协调者可以从第二阶段中分析出应该执行的操作，**进行提交或者中断事务操作**，这样即使挂掉的参与者恢复过来，所有参与者的操作是能够一致的，数据也是一致的。

所以，三阶段提交解决了二阶段提交中存在的由于协调者和参与者同时挂掉可能导致的数据一致性问题和单点故障问题，并减少阻塞。因为一旦参与者无法及时收到来自协调者的信息之后，他会默认执行提交事务，而不会一直持有事务资源并处于阻塞状态

https://mp.weixin.qq.com/s?__biz=MzU0MzQ5MDA0Mw==&mid=2247489849&idx=1&sn=cbac2a6ad99ac466f2ba8d69507fd2fe&chksm=fb0bf3adcc7c7abb565a9865e14b357888f7b7b78874b74c18bfdc5a4278ec2503b258c27730&scene=21#wechat_redirect

### Redlock redis分布式锁实现原理
https://www.cnblogs.com/rgcLOVEyaya/p/RGC_LOVE_YAYA_1003days.html
