---
layout: post
title: 升级Elasticsearch集群数量实战记录
date:   2017-07-30 21:00:06
categories: Work
image: /assets/images/elasticsearch.png
---

现在线上有一个elasticsearch集群搜索服务有三台elasticsearch实例（es1、es2、es3），打算将其升级为5台（增加es4、es5）。这篇文章主要是对整个操作的过程记录，以及出现的问题总结，包括移动数据量所需要的时间。因为，一开始由于不知道线上数据量全部分配完需要多少时间，如果从凌晨开始操作，到早上8点都还没有同步完，这样会影响到白天线上业务的正常使用。

#### 准备阶段
线上es集群使用的是阿里云服务器，copy其中一个镜像。然后更改其elasticsearch.yml配置文件，检查IK插件是否安装成功。按照这个流程，准备两台新的服务器放入阿里云的隔离组，并安装好elasticsearch，测试elasticsearch实例可以正确启动。也做了将这两台服务器构建一个集群的测试。开始升级操作前30分钟，再次检查elasticsearch.yml 配置。主要的修改是：

```
discovery.zen.minimum_master_nodes:3
discovery.zen.ping.unicast.hosts: ["es1_ip", "es2_ip","es3_ip","es4_ip","es5_ip"]
```

#### 升级操作
关闭es集群shard分配功能。对es1执行：

```
curl -XPUT es1_ip:9200/_cluster/settings -d '{
  "transient": {
   "cluster.routing.allocation.enable": "none"
   }
}'
```
然后检查：

```
curl es1_ip:9200/_cluster/settings
curl es2_ip:9200/_cluster/settings
curl es3_ip:9200/_cluster/settings
```

得到的结果是：

```
{"transient":{"cluster":{"routing":{"allocation":{"enable":"none"}}}}}
```
说明es集群已经关闭shard分配功能

#### 关闭es1、es2、es3上的monit

```
sudo service monit stop
```

手动控制elasticsearch进程的启动，避免monit自动拉起elasticsearch进程导致意外问题

#### 这时，新的两台服务器es4、es5还在隔离组。把隔离取消，然后启动这两台服务器的elasticsearch实例

使用目录下面写好的启动脚本，这个脚本可以在启动时，为elasticsearch获取所设定的内存(控制分配给es进程的最大内存、最小内存)，JVM的设置等

```
./elasticsearch.sh start
```

执行
```
curl es1_ip:9200/_cat/nodes
curl es1_ip:9200/_cat/shards
curl es1_ip:9200/_cat/health
curl es1_ip:9200/_cat/indices
```

验证nodes节点信息是否变为五个，以及shard现在的分布情况，应该只分布在es1、es2、es3上。es4和es5还没有shard

然后记录下当前的indices信息

例子：
```
health status index    pri rep docs.count docs.deleted store.size pri.store.size
green  open   users     5   1   15036978      4262221     13.2gb          6.6gb
green  open   posts     5   1   15036978      4262221     13.2gb          6.6gb
```

我们可以看到索引的健康度、状态、文档数量、数据大小和主分片数量、副分片的倍数设置。记录下这些信息，在最后升级完的时候
进行比对，看是否有偏差。如果一切正常，这些信息是不会变的。

#### 启动es集群shard分配功能
这是集群操作，所以只要对其中一台es实例操作即可
```
curl -XPUT es1_ip:9200/_cluster/settings -d '{
  "transient": {
   "cluster.routing.allocation.enable": "all"
   }
}'
```

#### shard分配开始
执行：

```
curl es1_ip:9200/_cat/shards

curl es1_ip:9200/_cat/health

curl es1_ip:9200/_cat/indices

curl es1_ip:9200/_cat/recovery
```

+ 观察shards分布情况，是否向es4和es5分配shards。以及分配的百分比进度
+ 监控es4 和 es5 服务器的elasticsearch的log
+ 登入 es4和es5， 查看挂载的SSD数据盘数据量是否在增长
+ 测试进行es搜索的测试，快速搜索，工单筛选，客户筛选，观察对线上业务的影响

这段时间，主要使用：

```
curl es1_ip:9200/_cat/shards

curl es1_ip:9200/_cat/health

curl es1_ip:9200/_cat/recovery

```

观测shard分配进度和情况，以及线上系统健康度。


curl es1_ip:9200/_cat/shards 可以看见具体哪个索引的，哪个es服务器在移动shard到另一台es服务器

```
正在shard移动时的状态
posts r RELOCATING  3628884   5.4gb es_ip1   elasticsearch1 -> es_ip5 elasticsearch5
```
可以看到哪个索引，哪个shard分片，从哪台服务器上移到另一台服务器上

这个过程发现，elasticsearch会选择将reproduction shard(副本分片)移到新的elasticsearch服务器上，而避免移到主分片。

这样可以避免移到主分片时候发生数据丢失的情况，而移动副本分配不用担心这个问题，并且线上的health一直是green。尽可能不影响线上正常搜索业务。

移动shard默认是两个并发操作。一开始误解了，以为每个es实例会进行两个并发的shard移动，会有6个shard在并发移动，实际情况是，整个集群只有2个shard在并发移动。下次可以将这个值调大一些，加快shard的移动速度，幸好shard数据移动的速度比想象的要快。

通过htop观察服务器的负载，在进行shard分配的服务器，CPU使用率一般在20%-40%之间。服务器是16核，也就是一个elasticsearch进程shard的移动，一个核心都没有跑满，服务器负载在0.2。可见elasticsearch的分片移动还是很保守的，对服务器几乎没有很大的压力。并且观察发现，elasticsearch不会一次把某个索引该分配的shard都分配完再选择下一个索引，而是轮询的分配索引的shard。A 索引分配了一个shard，就分配B索引的shard，一圈之后，又回到A 索引继续分配shard。直到最后所有索引shard数量在集群中平衡。

#### 收尾操作
大约一个小时时间，shard分配结束。一共将88G左右的数据分配到了两台新的服务器上。这是在默认shard并发分配数2的情况下的时间记录。大家以后可以根据这个记录，预估自己的elasticsearch shard分配会需要多少时间。

动态修改 master选举数量。根据elasticsearch文档推荐的公式： (N(master)+1)/2

```
curl -XPUT es1_ip:9200/_cluster/settings -d '{
      "persistent" : {
          "discovery.zen.minimum_master_nodes" : 3
      }
    }'
```
检查配置：

```
{"persistent":{"discovery":{"zen":{"minimum_master_nodes":"3"}}},"transient":{"cluster":{"routing":{"allocation":{"enable":"all"}}}}}
```

再检测API命令
```
curl es1_ip:9200/_cat/health

curl es1_ip:9200/_cat/indices

curl es1_ip:9200/_cat/shards
```

这时候，观察到shard分配的结果是，每个索引有10个shard，每个es服务器拥有一个索引的两个shard。新加入的es4、es5上都是副本shard，原有集群中的es1、es2、es3拥有主shard和副本shard。

例子：

```
index              shard   prirep   state     docs     store  ip    node
posts               2       r       STARTED   993718   3.5gb es1_ip es1
posts               2       p       STARTED   993718   3.5gb es1_ip es1
posts               0       p       STARTED   993428   3.7gb es2_ip es2
posts               0       p       STARTED   993428   3.7gb es2_ip es2
posts               3       p       STARTED   993653   3.6gb es3_ip es3
posts               3       p       STARTED   993653   3.6gb es3_ip es3
posts               1       r       STARTED   994063   3.5gb es4_ip es4
posts               1       r       STARTED   994063   3.5gb es4_ip es4
posts               4       r       STARTED   993938   3.5gb es5_ip es5
posts               4       r       STARTED   993938   3.5gb es5_ip es5
```

posts索引的十个shard分片在每台elasticsearch服务器上有两个分片。perfect!(实际结果是乱序的)

#### 后悔的操作选择
我们修改了es1的配置文件，想要重启es1 elasticsearch实例。重启后，发生了意想不到的事情，索引的shard又开始分配移动，等了快40分钟分配移动结束，但是，这时候不是每个es服务器平均拥有一个索引的两个shard，而是有的es服务器有该索引的三个shard。

```
posts               2 r STARTED   993718   3.5gb es4_ip es4
posts               2 p STARTED   993718   3.5gb es2_ip es2
posts               0 p STARTED   993428   3.7gb es4_ip es4
posts               0 r STARTED   993428   3.7gb es1_ip es1
posts               3 r STARTED   993653   3.6gb es1_ip es1
posts               3 p STARTED   993653   3.6gb es2_ip es2
posts               1 r STARTED   994063   3.5gb es5_ip es5
posts               1 p STARTED   994063   3.5gb es1_ip es1
posts               4 r STARTED   993938   3.5gb es5_ip es5
posts               4 p STARTED   993938   3.5gb es3_ip es3
```

posts在elasticsearch1 上出现了三个shard

实际上，原本集群中的三台服务器是不用重启的，你可以修改他们elasticsearch.yml 配置中的单拨数组设置：discovery.zen.ping.unicast.hosts。加上新加的两台服务器的ip，和 discovery.zen.minimum_master_nodes:3。下次重启的时候，就会读取这个新的配置，而不需要马上重新。因为，k可以通过调用API的方式，动态配置discovery.zen.minimum_master_nodes，而discovery.zen.ping.unicast.hosts的配置，在新的elasticsearch服务器上配置五台服务器的ip地址就可以了。

打开所有es服务器的monit，测试线上elasticsearch搜索功能

修改项目代码，加上新加的两台elasticsearch服务器的IP

```
Elasticsearch::Model.client = Elasticsearch::Client.new(
  host: [es_ip1, es_ip2, es_ip3, es_ip4, es_ip5],
  retry_on_failure: 0,
  log: true,
  transport_options: { request: { timeout: 10 } }
)
```
