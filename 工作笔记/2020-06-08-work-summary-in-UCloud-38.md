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
