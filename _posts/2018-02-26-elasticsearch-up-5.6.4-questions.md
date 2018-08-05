---
layout: post
title:  Elasticsearch升级到5.x之后，线上遇到的问题小结
date:   2018-02-26 17:43:06
categories: Elasticsearch
image: /assets/images/s1.jpeg
---

##### 请求耗时由5ms增长到了15ms左右

切到新版本的Elasticsearch集群后,通过观察Newrelic和GrayLog的Elasticsearch日志,发现Elasticsearch的请求时间由原来的平均5ms内,增长到了20ms. 一开始以为是冷数据问题,新的集群没有做过"预热处理",是直接切换过去.但是发现几天之后，请求耗时降到了13-15ms左右的范围。这请求耗时翻了一倍多，令人诧异，费了那么大劲把Elasticsearch升级上去了，而没有达到预期的效果，这就让人难以接受。之后查找问题，原来是 `index.translog`的问题。 5.x版本之后，默认 `index.translog` 是：

```
"index.translog.durability": "request"
```

表示每次Elasticsearch请求(e.g. index, delete, update, bulk)，translog fsync 刷新到硬盘，这样 translog 能够保证每个操作不会丢失。在每次请求后都执行一个 fsync 会带来一些性能损失，尽管实践表明这种损失相对较小（特别是bulk导入，它在一次请求中平摊了大量文档的开销）。

但是对于一些大容量的偶尔丢失几秒数据问题也并不严重的集群，使用异步的 fsync 还是比较有益的。比如，写入的数据被缓存到内存中，再每5秒执行一次 fsync 。

所以，我们对每个索引进行了修改：

```
PUT /my_index/_settings
{
    "index.translog.durability": "async",
    "index.translog.sync_interval": "5s"
}
```

因为我们的真正数据在MySQL上还有一份，所以，真的遇到有Elasticsearch数据丢失，可以从MySQL同步恢复。

配置之后，Elasticsearch的请求耗时降到了7-8ms左右。

##### 对elasticsearch.yml 配置的优化

```
gateway.recover_after_nodes: 3
gateway.recover_after_time: 5m
gateway.expected_nodes: 5
```

当集群中有3个node，过5分钟才开始重新分片(sharding)，或者有5个node的时候，立即开始分片

Elasticsearch的每个集群是5个node。这样配置，可以在重启某个node节点时，不会马上开始分片，因为常常这不是想要的结果。之前就遇到过这样的问题，每次重新分片会根据数据量耗一定时间。

##### thread_pool 的配置

线上遇到Elasticsearch报错，

```
es_rejected_execution_exception
```

经过排查，应该是有bulk操作失败了。这是由于这些bulk操作的Elasticsearch thread_pool在那个时刻满了，再来bulk操作，就会被 Elasticsearch rejected 处理。

```
curl -XGET 'http://localhost:9200/_nodes/stats?pretty'
```

通过上面的命令，得到信息，查看发现有一个node
```
"thread_pool" : {
    "bulk" : {
      "threads" : 16,
      "queue" : 0,
      "active" : 0,
      "rejected" : 68,
      "largest" : 16,
      "completed" : 84695441
    },
```

说明有68个bulk操作失败了。

适当的修改bulk thread_pool 大小。为什么说适当的呢？因为Elasticsearch官方建议是不要随便修改，原因也很简单，当thread_pool 值越大时，不一定性能就会越好， 第一 线程越多是要耗一定资源的，第二 线程池中线程的切换是要消耗CPU的。这也是使用线程池会遇到的问题，所以默认 bulk thread_pool是CPU核心数， search是 `(n(CPU)*3/2)+1`。

暂时设置：

```
curl -XPUT 'localhost:9200/_cluster/settings' -d '{
    "transient": {
        "threadpool.bulk.type": "fixed",    批量请求线程池类型 固定线程池大小
        "threadpool.bulk.size": 32,         线程池大小
        "threadpool.bulk.queue_size": 1000  队列大小
    }
}'


threadpool.bulk.queue_size: 1000
```

##### 对空串的查询会报语法错误

content = params[:query] => ""

前端传了空字符串

```ruby
{
  bool:{
    must: [
      multi_match: {
        type: "phrase",
        query: content,
        slop: 0,
        fields: ["content.tokenized", "subject.tokenized"]
      }
    ],
    filter:{
      term: { company_id: company_id }
    }
  }
}

# 会报语法错误
```

优化为：

```ruby
query = {bool: {must: [], filter: {}}}

if content.present?
  match = { multi_match: {
              type: "phrase",
              query: content,
              slop: 0,
              fields: ["content.tokenized", "subject.tokenized"]
            }
          }
  query[:bool][:must] << match
end

query[:filter] = { term: {company_id: company_id}}
```

当 content 不存在的时候，只是进行匹配查询

##### 线上集群关闭分片自动均衡

分片的自动均衡主要目的防止更新造成各个分片数据分布不均匀。但是如果线上一个节点挂掉了，很容易触发自动均衡，此时集群内部的数据移动占用所有带宽。建议采用闲时定时均衡策略来保证数据的均匀。

##### 尽可能延长refresh时间间隔

为了确保实时索引ES索引刷新时间间隔默认1s，索引刷新会导致查询性能受影响，在确保业务时效性的基础上可以适当延长refresh时间间隔保证查询的性能

##### 除非有必要，把all字段去掉

索引默认除了索引每个字段外，还有额外创建一个all的字段，保存所有文本，去掉这个字段可以把索引大小降低50%

##### 创建索引时，尽可能把查询比较慢的索引和快的索引物理分离


参考链接：
```
https://www.elastic.co/guide/cn/elasticsearch/guide/current/translog.html
https://www.elastic.co/guide/en/elasticsearch/guide/current/_don_8217_t_touch_these_settings.html#_threadpools
http://blog.csdn.net/opensure/article/details/51491815
http://blog.51cto.com/jfzhang/1685530
```
