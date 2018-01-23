---
layout: post
title: 安装配置elasticsearch-5.5.2+IK中文分词器
date:   2017-09-13 17:26:06
categories: Work
image: /assets/images/elasticsearch.png
---

最近的任务是将elasticsearch集群由低版本的1.4.2升级到5.5.2版本。由于跨的版本比较多，查看了一下elasticsearch文档，Breaking changes 非常多，所以这会是一次改动很大的升级。

第一次调研完毕后，开始在本机安装elasticsearch环境。

##### 下载tar安装包
本机系统是 Ubuntu 14.04，直接在elasticsearch官方网站下载5.5.2的安装包。elasticsearch很人性化的提供了不同的安装包文件，我选择了tar压缩包。不到40M，很快下载完。

我直接解压到了home目录

vim config/elasticsearch.yml

下面是我的配置

```yml
cluster.name: es_develompent
node.name: es_1
node.master: true
node.data: true
bootstrap.memory_lock: true
path.data: /home/user/path/data
path.logs: /home/user/path/logs
path.plugins: /path/to/plugins
network.host: 127.0.0.1
http.port: 9200
transport.tcp.port: 9300
discovery.zen.ping_timeout: 3s
discovery.zen.ping.unicast.hosts: ["127.0.0.1"]
indices.fielddata.cache.size: 30%
action.auto_create_index: false # 禁止自动创建索引
```

如果没有指定scripts的目录，需要在config目录下新建一个scripts目录

配置比较简单，其他的配置使用了默认值

建议将`data`、`plugins`和`logs`目录不和`elasticsearch`安装目录放在一起，而是另外放在别的目录，这样当再进行小版本升级的时候，就很方便了，几乎不会影响之前的数据。

在本地 network.host: 127.0.0.1 这样配置，你就仍然可以使用 curl localhost:9200 来调用elasticsearch接口

启动 elasticsearch 别使用sudo

```
./elasticsearch
```

报
```
max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
max number of threads [1024] for user [tommy] is too low, increase to at least [2048]
```

解决方法

```
在 /etc/security/limits.conf 文件最后添加

* hard nofile 65536
* soft nofile 65536

* - memlock unlimited  # 打开内存lock
#user soft memlock unlimited
#user hard memlock unlimited
```
然后重新登入，比如ssh重新登入

你可以通过 ulimit -n 命令看是否修改成功

vim /etc/sysctl.conf

```
vm.max_map_count=262144
```
之后运行 sudo /sbin/syctl -p 可以立即让修改生效

vim config/jvm.options

```
# Xms represents the initial size of total heap space
# Xmx represents the maximum size of total heap space
# 配置的内存大小需要为整数，单位用g或m

-Xms1g
-Xmx1g
```

配置启动elasticsearch时，jvm可以锁定这些内存,别超过32G。elasticsearch准备了jvm.options 文件，不像低版本时期，还要手动做额外的配置，这样就方便多了

##### 以守护进程方式启动elasticsearch

```
./bin/elasticsearch -d
```

##### 快速测试连接
```
curl localhost:9200

{
  "name" : "es_1",
  "cluster_name" : "es_develompent",
  "cluster_uuid" : "0g6WTHYzSBuqsZhi9H0vEg",
  "version" : {
    "number" : "5.5.2",
    "build_hash" : "b2f0c09",
    "build_date" : "2017-08-14T12:33:14.154Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.0"
  },
  "tagline" : "You Know, for Search"
}
```

##### 对集群模式重要的几个配置
```
discovery.zen.minimum_master_nodes: (N/2+1) N是集群节点数量

gateway.recover_after_nodes: n
这个设置主要防止不必要的数据处理，比如一个集群全部重启，然后有一个机器起的比较慢，然后机器就会组成集群，选出master，然后从备份中恢复出故障集群的数据。集群此时恢复正常。然后起的慢的机器又重启成功了，又会把数据同步到这台机器上并删除冗余数据。这项配置决定直到第n个节点可用才可以进行恢复操作

gateway.recover_after_time: 5m
gateway.expected_nodes: 10

这三项要求首先等待n个节点恢复，然后等待5分钟或者10个节点已经加入了集群就开始数据恢复
```

#####设置副本分配 replicas number 为 0
```
curl -XPUT es_ip:9200/_all/_settings -d '
{
  "index" : {
    "number_of_replicas" : 0
  }
}'
```
等一台ES服务数据导入完毕后，再打开为1.然后打开分片移动
```
curl -XPUT es_ip:9200/_cluster/settings -d '{
  "transient": {
    "cluster.routing.allocation.enable": "all"
    }
  }'
```

让ES集群自动创建replicas分片的数据。这样速度会非常快

##### 发现的问题
之前在1.4版本的时候有下面的配置

```
"index.analysis.analyzer.default.type" : "ik"
```

在5.5.2版本，我修改成
```
"index.analysis.analyzer.default.type" : "ik_smart"
```

启动时报错

```
Found index level settings on node level configuration.

Since elasticsearch 5.x index level settings can NOT be set on the nodes
configuration like the elasticsearch.yaml, in system properties or command line
arguments.In order to upgrade all indices the settings must be updated via the
/${index}/_settings API. Unless all settings are dynamic all indices must be closed
in order to apply the upgradeIndices created in the future should use index templates
to set default values.

Please ensure all required values are updated on all indices by executing:

curl -XPUT 'http://localhost:9200/_all/_settings?preserve_existing=true' -d '{
  "index.analysis.analyzer.default.type" : "ik_smart"
}'
```

英文提示：5.x以上的版本，elasticsearch.yml中不再支持配置index的设置。而使用API的方法进行配置。我觉得这个改进很不错，将节点固定配置放在elasticsearch.yml中，将index配置这种集群可以动态配置的选项都使用API接口的方式配置。因为elasticsearch集群搭建好之后，尽量不要重新启动，重启对线上的服务影响很大。

```
discovery.zen.ping.timeout =>(更新为) discovery.zen.ping_timeout
```

在config目录，手动mkdir scripts， 文件夹中可以没有任何脚本文件

config 目录中的文件权限，都修改为644. 因为新版本的elasticsearch 不建议使用root 或sudo的方式启动了，所以，对应的权限，也需要给予

##### ES mapping 的field的数量默认限制是1000，如果需要使用更大的field数量，需要手动修改

```ruby
client.indices.put_settings(index: index_name, body: {'index.mapping.total_fields.limit'=> 100000})
```

##### ignore_above 256
为 keyword 类型的字段配置 ignore_above: 256，保留256 byte 这样就不会报超过 ` bytes can be at most 32766 in length`的错误
lucene 对keyword 类型字段bytes长度限制

##### 安装IK中文分析器

```
Analyzer: ik_smart , ik_max_word , Tokenizer: ik_smart , ik_max_word
```

已经不再使用ik这个名称了。

ik_max_word: 会将文本做最细粒度的拆分

ik_smart: 会做最粗粒度的拆分

我安装了对应版本的IK分析器，只要一条命令，真心方便

```
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.2/elasticsearch-analysis-ik-5.5.2.zip
```
现在貌似对5.5.2版本不支持这种命令方式安装了，你需要使用第二种安装方式。

下载对应版本analysis-ik的zip，然后自行在config和plugins目录解压analysi-ik文件夹

下载地址在： https://github.com/medcl/elasticsearch-analysis-ik/releases

IK目录
```
/home/user/elasticsearch-5.5.2/config/analysis-ik

├── extra_main.dic
├── extra_single_word.dic
├── extra_single_word_full.dic
├── extra_single_word_low_freq.dic
├── extra_stopword.dic
├── IKAnalyzer.cfg.xml
├── main.dic
├── preposition.dic
├── quantifier.dic
├── stopword.dic
├── suffix.dic
└── surname.dic

/home/user/elasticsearch-5.5.2/plugins/analysis-ik

.
├── commons-codec-1.9.jar
├── commons-logging-1.2.jar
├── elasticsearch-analysis-ik-5.5.2.jar
├── httpclient-4.5.2.jar
├── httpcore-4.4.4.jar
└── plugin-descriptor.properties

```
也不再需要在 elasticsearch.yml 中对IK进行任何配置了，以前还需要

```
index:
  analysis:
    analyzer:
      ik:
          alias: [news_analyzer_ik,ik_analyzer]
          type: org.elasticsearch.index.analysis.IkAnalyzerProvider
```

现在直接使用就可以

在启动elasticsearch的时候，可以在 log中看见这一句

```
2017-09-14T16:43:32,614][INFO ][o.w.a.d.Monitor          ] try load config from /home/user/elasticsearch-5.5.2/config/analysis-ik/IKAnalyzer.cfg.xml
```

说明IK加载成功了。

环境搭建好了，就可以愉快的进行升级改造了～
