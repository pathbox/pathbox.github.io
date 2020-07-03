---
layout: post
title: 使用canal+Kafka进行数据库同步实践
date:  2020-07-02 20:00:06
categories: System Design
image: /assets/images/post.jpg
---

最近的工作中，建了新的数据库。新数据库的数据来源于已有的TiDB数据库。TiDB数据库要将数据同步到新的数据库中。
一种方案在代码逻辑中，有相关写操作时，以调用接口的方式，调用接口服务，接口服务再将数据写到新的数据库中。这种方式看似简单，但其实“坑”很多。在原有代码逻辑中会增加大量这种调用接口同步的代码，增加了项目代码的复杂度，以后会越来越难维护。想到这点，就将这个方案排除了。

第二种方案，通过数据库的binlog进行同步。
这也是一套成熟的生产解决方案。也有不少binlog同步的中间件工具，所以我们关注的就是哪个工具能够更好的构建稳定、性能满足且易于高可用部署的方案。经过调研，我们选择了canal[https://github.com/alibaba/canal]。`canal`是阿里巴巴 MySQL binlog 增量订阅&消费组件，已经有在生产上实践的例子，并且方便的支持和其他常用的中间件组件组合，比如kafka，elasticsearch等，并且也有了`canal-go` go语言的client库，其他具体内容参阅canal的github主页。

![canal]( /assets/images/canal/canal.png "canal")

### 工作原理
![canal]( /assets/images/canal/binlog.jpeg "canal")

OK，开始干!根据wiki很快就用docker跑起了一台`canal-server`服务，直接用`canal-go`写`canal-client`代码逻辑。用`canal-go`直接连`canal-server`，`canal-server`和`canal-client`之间是Socket来进行通信的，传输协议是TCP，交互协议采用的是 Google Protocol Buffer 3.0。

### 官方工作流程
1.Canal连接到mysql数据库，模拟slave

2.canal-go与Canal建立连接，并订阅对应的数据库或表

2.数据库发生变更写入到binlog

5.Canal向数据库发送dump请求，获取binlog并解析

4.canal-go向Canal请求数据库变更

4.Canal发送解析后的数据给canal-go

5.canal-go收到数据，消费成功，同步到新的数据库，发送回执（可选）

6.Canal记录消费位置

Protocol Buffer的序列化速度还是很快的。反序列化后得到的数据，是每一行的数据，按照字段名和字段的值的结构，放到一个数组中

代码简单示例:
```go
func Handler(entry protocol.Entry)  {
	var keys []string
	rowChange := &protocol.RowChange{}
	proto.Unmarshal(entry.GetStoreValue(), rowChange)
	if rowChange != nil {
		eventType := rowChange.GetEventType()
		for _, rowData := range rowChange.GetRowDatas() { // 遍历每一行数据
			if eventType == protocol.EventType_DELETE || eventType == protocol.EventType_UPDATE {
				 columns := rowData.GetBeforeColumns() // 得到更改前的所有字段属性
			} else if eventType == protocol.EventType_INSERT {
				 columns := rowData.GetAfterColumns() // 得到更后前的所有字段属性
			}
			......
		}
	}
}
```

为了高可用和更高的性能，我们会创建多个`canal-client`构成一个集群，来进行解析并同步到新的数据库。这里就出现了一个比较重要的问题，如何保证`canal-client`集群解析消费binlog的顺序性呢？

我们使用的binlog是row模式。每一个写操作都会产生一条binlog日志。
举个简单的例子：插入了一条a记录，并且立马修改a记录。这样会有两个消息发送给`canal-client`，如果由于网络等原因，更新的消息早于插入的消息被处理了，还没有插入记录，更新操作的最后效果是失败的。

怎么办呢？canal可以和消息队列组合呀!而且支持kafka，rabbitmq，rocketmq多种选择，如此优秀。我们在消息队列这层来实现消息的顺序性。(后面会说怎么做)

我们选择了消息队列的业界标杆:kafka
UCloud提供了kafka和rocketMQ消息队列产品服务，使用它们能够快速便捷的搭建起一套消息队列系统。加速开发，方便运维。让我们一探究竟。

选择kafka消息队列产品，并申请开通
![kafka]( /assets/images/canal/kafka1.png "kafka")

开通完成后，在管理界面，创建kafka集群，根据自身需求，选择相应的硬件配置
![kafka]( /assets/images/canal/kafka2.png "kafka")

一个kafka+ZooKeeper集群就搭建出来了，给力！
![kafka]( /assets/images/canal/kafka3.png "kafka")

并且包含了节点管理、Topic管理、Consumer Group管理，能够非常方便的直接在控制台对配置进行修改

监控视图方面，监控的数据包括kafka生成和消费QPS，集群监控，ZooKeeper的监控。能够比较完善的提供监控指标。
![kafka]( /assets/images/canal/kafka4.png "kafka")
![kafka]( /assets/images/canal/kafka5.png "kafka")
![kafka]( /assets/images/canal/kafka6.png "kafka")

canal配上kafka也非常的简单。
vi /usr/local/canal/conf/canal.properties
```
# ...
# 可选项: tcp(默认), kafka, RocketMQ
canal.serverMode = kafka
# ...
# kafka/rocketmq 集群配置: 192.168.1.117:9092,192.168.1.118:9092,192.168.1.119:9092
canal.mq.servers = 127.0.0.1:9002
canal.mq.retries = 0
# flagMessage模式下可以调大该值, 但不要超过MQ消息体大小上限
canal.mq.batchSize = 16384
canal.mq.maxRequestSize = 1048576
# flatMessage模式下请将该值改大, 建议50-200
canal.mq.lingerMs = 1
canal.mq.bufferMemory = 33554432
# Canal的batch size, 默认50K, 由于kafka最大消息体限制请勿超过1M(900K以下)
canal.mq.canalBatchSize = 50
# Canal get数据的超时时间, 单位: 毫秒, 空为不限超时
canal.mq.canalGetTimeout = 100
# 是否为flat json格式对象
canal.mq.flatMessage = false
canal.mq.compressionType = none
canal.mq.acks = all
# kafka消息投递是否使用事务
canal.mq.transaction = false

# mq config
canal.mq.topic=default
# dynamic topic route by schema or table regex
#canal.mq.dynamicTopic=mytest1.user,mytest2\\..*,.*\\..*
canal.mq.dynamicTopic=mydatabase.mytable
canal.mq.partition=0
# hash partition config
canal.mq.partitionsNum=3
canal.mq.partitionHash=mydatabase.mytable
```
具体见：https://github.com/alibaba/canal/wiki/Canal-Kafka-RocketMQ-QuickStart

看到下面这一行配置
```
canal.mq.partitionHash=mydatabase.mytable
```
我们配置了kafka的partitionHash，并且我们一个Topic就是一个表。这样的效果就是，一个表的数据只会推到一个固定的partition中，然后再推给consumer进行消费处理，同步到新的数据库。通过这种方式，解决了之前碰到的binlog日志顺序处理的问题。这样即使我们部署了多个kafka consumer端，构成一个集群，这样consumer从一个partition消费消息，就是消费处理同一个表的数据。这样对于一个表来说，牺牲掉了并行处理，不过个人觉得，凭借kafka的性能强大的处理架构，我们的业务在kafka这个节点产生瓶颈并不容易。并且我们的业务目的不是实时一致性，在一定延迟下，两个数据库保证最终一致性。

下图是最终的同步架构

![kafka]( /assets/images/canal/kafka7.png "kafka")

并且，我们还用这套架构进行缓存失效的同步。我们使用的缓存模式是:`Cache-Aside`。同样的，如果在代码中数据更改的地方进行缓存失效操作，会将代码变得复杂。所以，在上述架构的基础上，将复杂的触发缓存失效的逻辑放到kafaka client的服务端统一处理，达到一定解耦的目的。

目前这套同步架构正常运行中，后续有遇到问题再继续更新。
