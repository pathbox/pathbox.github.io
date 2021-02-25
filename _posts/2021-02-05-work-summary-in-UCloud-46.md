---
layout: post
title: 最近工作总结(46)
date:  2021-02-05 20:00:00
categories: Work
image: /assets/images/post.jpg


---

   

### Kong+go plugin server 对上传文件接口处理的bug

kong的网关接口出现了内存一直上升不释放，导致Pod配置的内存被耗尽的情况

![WeChatWorkScreenshot_ba0fa94b-85a6-458f-928a-d9afb34a53e3](/Users/pathbox/Desktop/WXWork Files/Image/2021-02/WeChatWorkScreenshot_ba0fa94b-85a6-458f-928a-d9afb34a53e3.png)

服务的日志中打印了大量该日志，从日志上看是mmap的读写操作。

```lua
while not ngx.worker.exiting() do 
  kong.log.notice("Starting"..server_def.name or "")
  server_def.proc = assert(ngx_pipe.spawn(server_def.start_command, {
        merge_stderr = true
      }))
end

while not ngx.worker.exiting() do 
  kong.log.notice("Starting"..server_def.name or "")
  server_def.proc = assert(ngx_pipe.spawn(server_def.start_command, {
        merge_stderr = true,
        buffer_size = 40960 
      }))
end
```

解决方法：buffer_size默认是4096byte，这里将其重置扩大了10倍

openresty这里buffer_size用的默认值。 导致读取go plugin server返回的内容时，由于上传的文件可能是几M，会不断尝试申请更大的内存，直到申请到足够大的内存。 但是由于lua gc的释放内存逻辑，之前申请的内存也不会及时释放，导致短时间内存上升，将Pod的内存耗尽

### 数据库多地，缓存非多地导致的查询问题

A数据库会同步到B数据库，但写操作只操作A数据库。且在A区域有一个缓存集群，所有数据库都共用该缓存集群。

问题流程: 一个写操作 => 将缓存删除 => A的数据还未同步到B数据库 => B地域有读请求,从B数据库读取到了旧的数据 => 此时没有缓存，则B的读操作会更新缓存 => 旧的数据又更新为了缓存 => A的数据同步到了B数据库,但是B数据库不会删除缓存，将旧的缓存数据库删除

快速的解决方案: A数据同步到B数据库的时候也将对应的缓存删除。但这样其实B地区的用户第一个请求时候，还是可能读取到的是B数据库的旧数据。此方案并非完全一致性，是最终一致性，有实时性问题。



### Redis超时大于网关接口超时而导致的诡异情况

redis缓存操作超时，该操作并非异步处理，而超时时间达到了2分钟。超时报错之后程序会继续执行，但实际网关的超时时间是1分钟，已经超时返回给了前端。所以出现了，从日志上看后端逻辑都执行了，只是中间延迟了2分钟，而前端操作接受到了接口超时的返回而没有继续进行业务下一步的接口调用



### 提高ElasticSearch写入性能

- 增大刷盘时间(refresh_interval):默认是 1s，我们时间过程中调到了 5s。调大之后写入性能上升还是比较明显的，带来的问题是日志写入 5s 之后才能被查询到，不过 5s 延迟延迟业务上是完全可以接受的。

- 0 备份并且关掉事务日志（“durability”: “async”）：这个对写入性能的提高是大幅度的，几乎是两倍的提升，我们的集群最高可以写到 15W+。但是问题是无法保证可靠性，万一挂了怎么办？  我们的解决方式是 kafka 保存 12 小时的数据+低峰期（晚上）备份。  首先 kafka 保存 12 小时的数据保证了即使 flink 挂了或者 ES 挂了，都可以通过重置消费位点把数据找回来。晚上备份的话，保证了十二小时之前的数据就不会丢了

- 提前创建索引：业务日志每到晚上零点的时候，都会堆积数据。这是因为这个时候在大量的创建索引，写入速度自然受影响。解决思路就是提前把索引创建好

- 减少集群副本分片数，过多副本会导致 ES 内部写扩大。ES 集群主用于构建热门 Trace 索引用于定位问题，业务特性是写入量大而数据敏感度不高。所以我们可以采用经济实惠的配置，去掉过多副本，维护单副本保证数据冗余已经足够，另外对于部分超大索引，我们也会采用 0 副本的策略。
  索引设计方面，id 自动生成（舍弃幂等），去掉打分机制，去掉 DocValues 策略，嵌套对象类型调整为 Object 对象类型。此处优化的目的是通过减少索引字段，降低 Indexing Thread 线程的 IO 压力，经过多次调整选择了最佳参数。
  根据 ES 官方提供的优化手段进行调整，包括 Refresh，Flush 时间，Index_buffer_size 等。
  上述优化，其实是对 ES 集群一种性能的取舍，牺牲数据可靠性以及搜索实时性来换取极致的写入性能。但其实 ES 只是存储热门数据，天机阁有专门的 Hbase 集群对全量数据进行备份，详细记录上报日志流水，保证数据的可靠性。


   


 