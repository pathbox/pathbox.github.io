---
layout: post
title: RabbitMQ practice example in Golang with some thoughts
date:   2018-07-21 20:58:06
categories: Server
image: /assets/images/post.jpg
---

### 环境
Mac OSX rabbitmq-server -detached 启动一个rabbitmq服务

### Publisher
https://github.com/pathbox/learning-go/blob/master/src/amqp/examples/simple_producer.go

### Consumer

建立连接，绑定exchange和queue，consume 等待deliveries

https://github.com/pathbox/learning-go/blob/master/src/amqp/examples/simple_consumer.go


### 优化版本的Consumer

- 对集群node的连接尝试，如果一个node崩溃了，会连接别的node
- 连接3s超时
- 不断的重连尝试机制，这样当rabbitmq集群都挂了，但consumer服务不会崩溃，只要将rabbitmq集群重启好，consumer服务即可重连成功，不需要重新启动

```go
func Run() {
	for {
		bindConsumer() // bind AMQP exchange and queue, then consumer is OK, it is waitting for the message from publisher
	}
}

// bind AMQP exchange and queue, then consumer is OK, it is waitting for the message from publisher
func bindConsumer() {
	defer func() { // 如果当前rabbitmq node崩溃了，会不断重试连接
		if err := recover(); err != nil {
			ulog.Errorf("No AMQP Node OK Now,recover: %s", err)
		}
	}()

	for { // 如果当前rabbitmq node崩溃了，会不断重试连接
		conn := newAMQPConn()

		exchangeVal, _ := ufcommon.GetConfigByKey(code.OplogExchange)
		queueVal, _ := ufcommon.GetConfigByKey(code.OplogQueue)
		oplogsExchange := exchangeVal.(string)
		oplogsQueue := queueVal.(string)
		ulog.Infof("oplog Exchange: %s Queue: %s", oplogsExchange, oplogsQueue)

		mqChan, err := conn.Channel()
		if err != nil {
			ulog.Errorf("conn.Channel Error:%s", err)
			return
		}
		defer mqChan.Close()

		if err = mqChan.ExchangeDeclare(
			oplogsExchange,
			amqp.ExchangeTopic,
			true,
			true,
			false,
			false,
			nil,
		); err != nil {
			ulog.Errorf("mqChan.ExchangeDeclare Error:%s", err)
			return
		}

		queue, err := mqChan.QueueDeclare(
			oplogsQueue,
			false,
			true,
			false,
			false,
			nil,
		)
		if err != nil {
			ulog.Errorf("mqChan.QueueDeclare Error: %s", err)
			return
		}

		if err = mqChan.QueueBind(
			queue.Name,
			code.TopicKey,
			oplogsExchange,
			false,
			nil,
		); err != nil {
			ulog.Errorf("ch.QueueBind Error:%s", err)
			return
		}

		if err = mqChan.Qos(code.PrefetchCount, 0, false); err != nil {
			ulog.Errorf("mqChan.Qos Error:%s", err)
			return
		}

		// consumer 实际上和 exchange和queue不一定要在同一个代码或服务器中定义，完全可以将consumer单独作为一个服务，建立conn，取一个chan，然后消费对应的队列
		deliveries, err := mqChan.Consume( // consume read message from queue
			oplogsQueue,
			code.OplogConsumer,
			false, // if autoAck true, prefetchcount doesn't work
			false,
			false,
			false,
			nil,
		)
		if err != nil {
			ulog.Errorf("mqChan.Consume Error:%s", err)
			return
		}

		logic.ConsumeLoop(deliveries) // loop consume handle
		conn.Close()
	}
}

func newAMQPConn() *amqp.Connection {
	rabbitmq, _ := ufcommon.GetConfigByKey("rabbitmq")
	mqStr := rabbitmq.(string)
	mqURLs := strings.Split(mqStr, ",")
	l := len(mqURLs)

	var conn *amqp.Connection
	var err error

	if l == 0 {
		ulog.Errorf("config rabbitmq blank")
		return conn
	}

	for i := 0; i < l; i++ {
		conn, err = amqp.DialConfig("amqp:///", amqp.Config{
			Dial: func(network, addr string) (net.Conn, error) {
				return net.DialTimeout("tcp", mqURLs[i], 3*time.Second) // 每次3s超时
			},
		})

		if err != nil {
			ulog.Errorf("amqp.Dial Error: %s", err)
			if i == l {
				return conn // 所有addr地址都尝试过失败则返回
			}
			continue
		}

		goto BIND // 没有报错，则跳到bind部分
	}

BIND:
	ulog.Infof("MQ Conn: %v OK...vhost: %s", conn.LocalAddr(), conn.Config.Vhost)
	return conn
}
```

### RabbitMQ Quality of Service (QOS)

三个参数：prefetchCount, prefetchSize int, global bool

如果没有配置，默认情况：消息是一个一个出队列，然后被消费者消费，再Ack。

So let’s increase our QOS value to “5”. Now our Consumer reads from the Queue at a rate of 5 messages at a time. Our Queue is now empty. But what’s happening at the Consumer? The Consumer manages its own Shared Queue in memory. This Shared Queue is a local representation of an AMQP Queue. When a Consumer de-queues a message, that message is cached in the Consumer’s Shared Queue, and processed accordingly.

配置了，消费者可以一次取多个消息(根据配置情况)，放到消费者本地的共享缓存队列中，再一个一个操作消息。这样在一定程度上可以减少消费者取队列的操作，减少network traffic involve time。

prefetchCount和prefetchSize配置不是越大越好。

1. 太大会增加消费者共享缓存队列的大小，增加了内存消耗
2. 在消费者共享缓存中取消息操作，同样需要循环的时间，消费单个消息的时间并没有减少
3. 当有多个消费者的时候，A消费者取了100个消费，B消费者取了剩下的10个消息，这时候A消费者的工作负载就更大，B消费者消费完10个消息后，一段时间没有新的消息，则这段时间B消费者处于闲置idle状态，浪费的处理资源。也就是配置数量越大，越容易导致多消费者消费负载不够均衡(not adequately balanced)
4. 如果是对实时性要求比较高的，增加集群数量和消费者数量，提高并发性才是正确的方式，而进行QOS配置会降低实时性的并发性

`适合处理实时性不那么高的场景`，合适的配置可以加快整体的消费时间,不让生产者感到queue的阻塞

### 启动多个消费者Server

通过配置 `QOS prefetchCount`,可以加快消费队列消息的速度，批量将消息从队列取到消费者服务本地缓存，然后处理，从宏观上看，批量从消息队列fetch消息可以减少取消息的次数，从而减少网络请求，但是上面也说了，过大的prefetchCount会导致consumer本地内存被消耗过多。

- 启动多个消费者Server
- Golang是二进制服务，可以非常容易的部署成集群服务
- 多个Consumer服务可以，避免单点故障，使整个服务变为高可用
- 增强并发消费能力，充分利用CPU核心
- 不宜过多，理论上不要超过CPU核心数
- 如果把多个Consumer服务部署在一台服务器上，应该注意`QOS prefetchCount`造成的内存消耗

### 需要非常高的消息实时性场景

应该设autoAck为true，一个一个的处理消息。这样对于每个消息来说，是能够最快的被消费完，而从整体上看，消费性能也许并不高。

### RabbitMQ 报错问题

```
inequivalent arg 'durable' for exchange 'csExchange' in vhost '/': received
```

使用不同的MQ客户端时，常常会出现以上错误信息。

比如用node作为product,使用java, Go, ruby, python作为consume.

最常见的原因是: durable,auto_delete,passive参数不一致，保持参数一致性就ok了

例子：使用node代码第一次创建了 exchange 和 queue，它们都有配置相关参数，然后又使用另一套node代码或者其他语言如Go代码的客户端也使用创建好的exchange和queue

这时候要保证相关参数一致，否则会报错，而失败