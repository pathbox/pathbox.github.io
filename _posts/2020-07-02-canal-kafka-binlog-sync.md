---
layout: post
title: 使用canal+Kafka进行数据库同步实践
date:  2020-07-02 20:00:06
categories: System Design
image: /assets/images/post.jpg
---

在微服务拆分的架构中，各服务拥有自己的数据库，常常会遇到服务之间数据通信的问题，B服务数据库的数据来源于A服务的数据库。A服务的数据有变更操作时，需要同步到B服务中。

第一种解决方案: 在代码逻辑中，有相关A服务数据写操作时，以调用接口的方式，调用B服务接口，B服务再将数据写到新的数据库中。这种方式看似简单，但其实“坑”很多。在A服务代码逻辑中会增加大量这种调用接口同步的代码，增加了项目代码的复杂度，以后会越来越难维护。并且，接口调用的方式并不是一个稳定的方式，没有重试机制，没有同步位置记录，接口调用失败了怎么处理，突然的大量接口调用会产生的问题等，这些都要考虑并且在业务中处理。这里会有不少工作量。想到这里，就将这个方案排除了。

第二种解决方案，通过数据库的binlog进行同步。这种解决方案，与A服务是独立的，不会和A服务有代码上的耦合。可以直接TCP连接进行传输数据，优于接口调用的方式。
这是一套成熟的生产解决方案，也有不少binlog同步的中间件工具，所以我们关注的就是哪个工具能够更好的构建稳定、性能满足且易于高可用部署的方案。经过调研，我们选择了`canal`[https://github.com/alibaba/canal]。`canal`是阿里巴巴 MySQL binlog 增量订阅&消费组件，已经有在生产上实践的例子，并且方便的支持和其他常用的中间件组件组合，比如kafka，elasticsearch等，也有了`canal-go` go语言的client库，满足我们在go上的需求，其他具体内容参阅canal的github主页。

### 原理简图
![canal]( /assets/images/canal/canal.png "canal")

![canal]( /assets/images/canal/binlog.jpeg "canal")

OK，开始干！现在要将A数据库的数据变更同步到B数据库。根据wiki很快就用docker跑起了一台`canal-server`服务，直接用`canal-go`写`canal-client`代码逻辑。用`canal-go`直接连`canal-server`，`canal-server`和`canal-client`之间是Socket来进行通信的，传输协议是TCP，交互协议采用的是 Google Protocol Buffer 3.0。

### 工作流程
1.Canal连接到A数据库，模拟slave

2.canal-client与Canal建立连接，并订阅对应的数据库表

3.A数据库发生变更写入到binlog，Canal向数据库发送dump请求，获取binlog并解析，发送解析后的数据给canal-client

4.canal-client收到数据，将数据同步到新的数据库

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

### 遇到的问题
为了高可用和更高的性能，我们会创建多个`canal-client`构成一个集群，来进行解析并同步到新的数据库。这里就出现了一个比较重要的问题，如何保证`canal-client`集群解析消费binlog的顺序性呢？

我们使用的binlog是row模式。每一个写操作都会产生一条binlog日志。
举个简单的例子：插入了一条a记录，并且立马修改a记录。这样会有两个消息发送给`canal-client`，如果由于网络等原因，更新的消息早于插入的消息被处理了，还没有插入记录，更新操作的最后效果是失败的。

怎么办呢？canal可以和消息队列组合呀!而且支持kafka，rabbitmq，rocketmq多种选择，如此优秀。我们在消息队列这层来实现消息的顺序性。(后面会说怎么做)

### 选择了canal+kafka方案
我们选择了消息队列的业界标杆: kafka
UCloud提供了kafka和rocketMQ消息队列产品服务，使用它们能够快速便捷的搭建起一套消息队列系统。加速开发，方便运维。

### canal的kafka配置
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

### 解决顺序消费问题

看到下面这一行配置
```
canal.mq.partitionHash=mydatabase.mytable
```
我们配置了kafka的partitionHash，并且我们一个Topic就是一个表。这样的效果就是，一个表的数据只会推到一个固定的partition中，然后再推给consumer进行消费处理，同步到新的数据库。通过这种方式，解决了之前碰到的binlog日志顺序处理的问题。这样即使我们部署了多个kafka consumer端，构成一个集群，这样consumer从一个partition消费消息，就是消费处理同一个表的数据。这样对于一个表来说，牺牲掉了并行处理，不过个人觉得，凭借kafka的性能强大的处理架构，我们的业务在kafka这个节点产生瓶颈并不容易。并且我们的业务目的不是实时一致性，在一定延迟下，两个数据库保证最终一致性。

下图是最终的同步架构，我们在每一个服务节点都实现了集群化。全都跑在UCloud的UK8s服务上，保证了服务节点的高可用性。canal也是集群换，但是某一时刻只会有一台canal在处理binlog，其他都是冗余服务。当这台canal服务挂了，其中一台冗余服务就会切换到工作状态。同样的，也是因为要保证binlog的顺序读取，所以只能有一台canal在工作。

![kafka]( /assets/images/canal/kafka7.png "kafka")

并且，我们还用这套架构进行缓存失效的同步。我们使用的缓存模式是:`Cache-Aside`。同样的，如果在代码中数据更改的地方进行缓存失效操作，会将代码变得复杂。所以，在上述架构的基础上，将复杂的触发缓存失效的逻辑放到`kafka-client`端统一处理，达到一定解耦的目的。

目前这套同步架构正常运行中，后续有遇到问题再继续更新。
