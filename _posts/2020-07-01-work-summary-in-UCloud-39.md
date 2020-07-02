---
layout: post
title: 最近工作总结(39)
date:  2020-07-01 15:45:06
categories: Work
image: /assets/images/post.jpg
---

### 尽量不要使用SELECT *
1. 不需要的列会增加数据传输时间和网络开销。需要解析更多的对象、字段、权限、属性等内容，在SQL语句复杂，硬解析较多的情况下，会对数据库造成沉重的负担，大的文本会增加网络开销
2. 对无用的打字单会增加io操作。长度超过728字节的时候，会先把超出的数据序列化到另一个地方，因此读取这条记录会增加一次io操作(MySQL InnoDB)
3. 失去MySQL优化器"覆盖索引"策略优化的可能性。首先要通过辅助索引过滤数据，然后再通过聚集索引获取所有的列，这就多了一次b+树查询。原本可能只通过辅助索引即可拿到所需要的字段数据

### Go http库 重用底层 TCP 连接需要注意读取完body并关闭

在结合实际的场景之后，我发现其实有的时候问题出在我们并不总是会去读取完整个http.Response 的 Body。为什么这么说呢？
在常见的 API 开发的业务逻辑中，我们会定义一个 JSON 的对象来反序列化 http.Response 的 Body，但是通常在反序列化这个回复之前，我们会做一些 http 的 StatusCode 检查，比如当 StatusCode 为 200 的时候，我们才去读取 http.Response 的 Body，如果不是 200，我们就直接返回一个包装好的错误。比如下面的模式：
```go
resp, err := http.Get("http://www.example.com")
if err != nil {
    return err
}
defer resp.Body.Close()

if resp.StatusCode == http.StatusOK {
    var apiRet APIRet
    decoder := json.NewDecoder(resp.Body)
    err := decoder.Decode(&apiRet)
    // ...
}
```
如果代码是按照上面的这种方式写的话，那么在请求异常的时候，会导致大量的底层 TCP 无法重用，所以我们稍微改进下就可以了。
```go
resp, err := http.Get("http://www.example.com")
if err != nil {
    return err
}
defer resp.Body.Close()

if resp.StatusCode == http.StatusOK {
    var apiRet APIRet
    decoder := json.NewDecoder(resp.Body)
    err := decoder.Decode(&apiRet)
    // ...
}else{
    io.Copy(ioutil.Discard, resp.Body)  // 关键的一步，帮我们读取完body
    // ...
}
```
我们通过直接将 http.Response 的 Body 丢弃掉就可以了。

### Kafka 的consumer offset数据过大容易导致的启动加载问题
Kafka内部会有`topic: __consumer_offsets`, 这个topic存储offset信息。当__consumer_offsets分区数据巨大，且分布不均，Kafka启动或重启时，加载__consumer_offsets数据就会非常的慢，很有可能导致启动超时，从而Kafka服务没法启动。

修改__consumer_offsets的cleanup.policy=delete，保留时间为15天，减少topic保存的数据量，减少Kafka加载压力

Kafka在合ZooKeeper连接时，如果由于网络等原因，可能会导致没法连接上ZooKeeper而发生重启。

### 使用canal和kafka进行数据库同步
为每个表配置分区key，每个表对应一个partition，以保证按照binlog数据顺序进行同步。是的这样会牺牲并发性。

### 如何为Kafka集群选择合适的Partitions数量
>https://blog.csdn.net/oDaiLiDong/article/details/52571901?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase
