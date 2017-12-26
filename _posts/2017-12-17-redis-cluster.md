---
layout: post
title: Redis 集群搭建及使用Golang示例
date:   2017-12-17 15:35:06
categories: redis
image: /assets/images/post.jpg
---

Redis 在3.x版本之后，自身支持了集群模式。Redis的集群主要是 master-slave的形式。集群定义了
16384个`hash slot`。这些`hash slot`分布在所有master上。`we simply take the CRC16 of the key modulo 16384` 将key计算得到对应的`hash slot`的值，然后看这个`hash slot`在哪个redis服务上，这个key就会保存在对应的这个redis服务。和Elasticsearch、mongoDB不一样，redis通过这种方式进行sharding。不能不说这是一种简单，但对redis来说是有效的一种集群方式。能够最大限度的保留原有redis的属性。
[[Document]](https://redis.io/topics/cluster-tutorial)

下面是部署过程的例子。

有三台服务器 A、B、C。外网ip分别为 A_IP,B_IP,C_IP。

三台服务器先卸载原有redis版本，安装最新的redis版本

```
$ wget http://download.redis.io/releases/redis-4.0.6.tar.gz
$ tar xzf redis-4.0.6.tar.gz
$ cd redis-4.0.6
$ make
```

```
cd /src
cp redis-server /usr/bin/
cp redis-cli /usr/bin/
cp redis-trib.rb /usr/bin/

mkdir redis-cluster
cd redis-cluster
mkdir 7000
mkdir 7100
```

进入7000，新建redis.conf 文件，编写配置文件
```
cd 7000
vim redis.conf
```

配置内容为
```ruby
port 7000                      # 绑定的端口
bind A_IP                      # 绑定的IP，如果你要开放出去，需要使用外网IP
cluster-enabled yes            # 是否启用集群模式
cluster-config-file nodes.conf # 节点配置文件
cluster-node-timeout 5000      # 超时设置
appendonly yes                 # appendonly 配置 持久化设置
cluster-require-full-coverage no  
```

`cluster-require-full-coverage`默认是yes。如果某个redis挂了，没有对应的slave升级为master，这时候，整个redis集群不可用。

所以建议设为no， 这样某个redis挂了，只是影响这一部分的`hash slot`查询有问题，不影响集群的其他redis的读写。

同理完成 7100的redis.conf配置文件内容。将三台机器都安装上述进行安装配置。

在A、B、C机器上

```
cd 7000
redis-server redis.conf
cd 7001
redis-server redis.conf  
```

到A机器上， 执行

```
redis-trib.rb create --replicas 1 A_IP:7000 B_IP:7000 \
C_IP:7000 A_IP:7100 B_IP:7100 C_IP:7100
```

之后会提示输入yes，执行完后。

```
redis-cli -p 7000 -h A_IP cluster nodes
```

查看集群状态，会得到类似这样的信息。

```
aedabb4bd830978905d68ab8d88a94a031b515fe A_IP:7000@17000 master - 0 1513840545000 2 connected 5461-10922
861ff3beac7a6a3023424a64446f1101f8193d9d A_IP:7100@17100 slave 3d9e37599f6547387503e165d21838556435bac4 0 1513840544011 4 connected
3d9e37599f6547387503e165d21838556435bac4 B_IP:7000@17000 myself,master - 0 1513840545000 1 connected 0-5460
3ef82f387086931dbdedbd24b379671b8bd03289 C_IP:7000@17000 master - 0 1513840544411 3 connected 10923-16383
27764a4a521e4e5bd8e2e108c472ef6ed51645cd B_IP:7100@17100 slave aedabb4bd830978905d68ab8d88a94a031b515fe 0 1513840543912 5 connected
7f6ef6f05968ee869359c0771d42d768243e4ca1 C_IP:7100@17100 slave 3ef82f387086931dbdedbd24b379671b8bd03289 0 1513840545414 6 connected
```

Bingo～ redis集群服务就跑起来了。 三台机器，每台机器一个master，一个slave

redis集群增加节点或Resharding操作是使用redis-trib脚本命令，这个脚本是Ruby写的。每个node有一个node ID，比如： `aedabb4bd830978905d68ab8d88a94a031b515fe`

Golang 代码示例

```go
package redis

import (
	"time"

	"github.com/go-redis/redis"
)

// redis数据超时时间
var Timeout = 5 * time.Hour

var Client = GetClusterClient()

func GetClusterClient() *redis.ClusterClient {
	var client *redis.ClusterClient
	client = redis.NewClusterClient(&redis.ClusterOptions{
		Addrs: []string{"A_IP:7000", "B_IP:7000", "C_IP:7000"},
	})
	err := client.Ping().Err()
	if err == nil {
		log.Info("Redis cluster OK")
	} else {
		log.Error("Redis cluster wrong")
	}
	return client
}

redis.Client.Set("word", "Hello World", redis.Timeout)
str, _ := rediscluster.Client.Get("word").Result()
fmt.Println(str) // => "Hello World"
```
