---
layout: post
title: 最近工作总结(六)
date:   2017-07-03 20:16:06
categories: Work
image: /assets/images/post.jpg
---

##### 如果调用第三方接口，请别忘了超时机制
周末的时候，线上出了问题。是由于IPIP服务商的服务器出了问题，我们调的接口迟迟没有返回值。
导致我们项目中，开了很多goroutine，每个goroutine中会请求IPIP的接口分析ip地址，由于IPIP的服务器没能及时相应，
导致大量的goroutine都阻塞在了那里。所以， 不要“信任”第三方服务，调用他们服务接口的时候，应该使用超时机制。

优化方案：使用了本地化IP地址的方案，使用了这个

> https://github.com/lionsoul2014/ip2region

减少了调用IPIP的次数。这样， 内存也一下少了200+M。 Perfect!

##### 什么才是更新操作
更新操作是更新什么某个字段或某个值，则前端传那个值，然后更新这个值

如果前端把没有被更新的值也传向后端了，这也许不是合理的选择。这样，这些值之间就会互相影响。

比如，某个状态字段需要一定的权限才能更新，当更新name字段值，把所有字段值到放入参数传向后端，后端直接使用这个参数，
这样相当于更新了多个字段值。此时原本没有权限更新状态字段，但是，前端其实只是想更新name字段值，则受到影响，不会成功

##### 有趣的ES前缀搜索优化
随着数据量的增大，customer索引出现了很多ES的慢日志查询。一种查询是由于前缀搜索导致。

慢查询的语句
```ruby
{filtered: {filter: {bool: {must: [{term: {company_id: company_id}}], should: [{prefix: {"nick_name.raw"=>"1"}}, {prefix: {"cellphone.raw"=>"1"}}, {prefix: {"emails.raw"=>"1"}}]}}}}
```

观察发现，查询的参数都是 1 或者 13 这样的值。然后搜索了资料，查到了这个内容。

> prefix查询是一个工作在词条级别的低级查询。它不会在搜索前对查询字符串进行解析。它假设用户会传入一个需要查询的精确前缀。默认情况下，prefix查询不会计算相关度分值。它只是进行文档匹配，匹配的文档的分值为1。其实，相比查询它更像一个过滤器。prefix查询和prefix过滤器的唯一区别在于过滤器可以被缓存。

```
为了支持前缀匹配，查询会执行以下的步骤：

遍历词条列表并找到以W1开头的词条。
收集对应的文档ID。
移动到下一个词条。
如果该词条也以W1开头，那么重复步骤2；否则结束操作。
尽管以上的步骤对于我们的小例子而言能很好地工作，想象一下当倒排索引含有一百万个以W1开头的邮政编码时的情景，prefix查询需要访问一百万个词条来得到结果。
```

当搜索值是 1 的时候，这个可以遍历匹配大量的词条，导致速度很慢。

优化方案： 使用 match_phrase_prefix  

```
{filtered: {filter: {bool: {must: [{term: {company_id: company_id}}], should: [{query: {match_phrase_prefix: {"nick_name.tokenized"=>{query: "1", max_expansions: 30}}}}, {query: {match_phrase_prefix: {"emails.tokenized"=>{query: "1", max_expansions: 30}}}}]}}}}
```

关于这里max_expansions的使用，有下面这断解释：

In the example above, ES will examine all terms in the "message" field
to look for those that begin with the letters "test". It will combine
all of those terms into a query (eg "test","tester","testing","tests"
etc), working in alphabetical (lexicographic) order, up to
max_expansions.

Set max_expansions to limit the number of terms that will be collected.
Eg, if the prefix is "a" you may end up with thousands of terms, which
is pretty meaningless. Set max_expansions to (eg) 30 so that the user
gets some response, but your server is not overwhelmed by too many
clauses.

但我发现一个有趣的事情，就是当搜索字符串的长度大于5的时候， 原来的前缀查询的速度反而更快。
这里可以理解，当搜索的字符串长度变长，prefix的方法遍历的匹配词条数量变少，速度变快了。
而 match_phrase_prefix 方法对搜索值会进行分词，然后再进行匹配。搜索值长度越长，分词匹配的时间
也会增加。所以，我用了5这个分界（不同的项目会有不同情况），当搜索字符串长度小于5时，我使用 match_phrase_prefix，
大于等于5时，我使用prefix。

##### 对多语言的翻译考虑
语言选择，从客户角度、从客服角度、从公司全局角度

三个不同的角度，范围不同、粒度不同

##### 关于数据库优化的记录
1.索引优化

+ 最左前缀原则： MySQL会一直向右匹配直到遇到范围查询（>,<,between,like）就停止匹配，比如a=1 and b=2 and c>3 and d=4. 如果建立（a,b,c,d）顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引，则都可以用到，abd的顺序可以任意调整

+ 避免重复索引：idx_abc多列索引，相当于创建了a单列索引，a，b组合索引以及a，b，c组合索引。不在索引列使用函数 如max(id) > 10, id+1>3 等

+ 尽量选择区分度高德列作为前缀索引：区分度的公式是 count(distinct col) / count(*)， 表示字段不重复的比例，比例越大我们扫描的记录数越少

2. SQL开发优化

+ 不使用存储过程、触发器、自定义函数

+ 不使用全文索引

+ 不使用分区表

+ 针对OTLP业务尽量避免使用多表join和子查询

+ SELECT使用具体的列名：在发生列的增删时，发生列名修改时，最大限度避免程序逻辑中没有修改导致的BUG,IN的元素个数300-500

+ 避免使用大事务，使用短小的事务没减少锁等待和竞争

+ 禁止使用 % 前缀模糊查询 where like '%xxx'

+ 禁止使用子查询，遇到使用子查询的情况，尽量使用join代替

+ 遇到分页查询，使用延迟关联解决。分页如果有大offset， 可以先取id，然后用主键关联表会提高效率

+ 禁止并发执行count(*)， 并发导致CPU飚高

+ 禁止使用order by rand()

+ 不使用负向查询，如 not in/like， 使用in 反向代替

+ 不要一次更新大量（大于 30000条）数据，使用循环批量更新和删除

SQL中使用到OR的改写为用IN() or的效率没有in的效率高

##### 开发API设计思考
开发API设计，严格对参数进行校验，对参数必传、类型、有效性、合法性进行校验。
不能让A客户调用的API访问到B客户的数据

对response的设计考虑。对不同情况的报错值得设计考虑，要简单明了的返回错误信息，让客户在接口返回值
层面，就知道自己调用该接口的问题所在。

##### 默认第三方服务都是不可靠的
默认第三方服务都是不可靠的，因为他们是不可控的。当他们服务崩溃时，你不知道什么时候可以修复。
你也不知道他们什么时候会崩溃。

所以，调用这些第三方服务的接口都要使用超时机制

当第三方服务崩溃时，要有崩溃处理机制，不要由于第三方服务的崩溃导致，本地服务不可用或造成重大的影响
比如使用云问机器人，当调用他们的接口不是返回200，则显示一个页面，页面中有转人工按钮，客户还能通过转人工进入
IM通讯界面，要不，云问机器人变为了唯一的IM入口，导致IM整个功能不可用

再比如，调用IPIP，IPIP服务崩溃时，可以选择取为空的ip分析值，返回空值，而不是因为IPIP的崩溃，导致代码
逻辑异常而中断服务

永远保持不相信第三方服务不会崩溃的态度

##### jwt解析失败
jwt 调用接口失败。

注意点：

+ 参数的组成
+ token是否正确
+ encode 和 deceode 是否对应

##### 使用 capistrano-sidekiq
使用 capistrano-sidekiq 可以在一台服务器上部署多个sidekiq。

set :sidekiq_processes, 4

表示开启4个sidekiq 进程

这样的好处是，能够一定程度提高sidekiq的并发能力，以及进行了一定的容灾处理

##### hash 的赋值操作，无处不是坑
hash的merge操作其实是很危险的操作，会把原来的整个值被新的值，相同的key是会被覆盖。如果原本的hash值的层级比较多，当新的值的层级比较少，
则很有可能原有值中的不想被覆盖的值也被覆盖了。

例子：
```ruby
ha = {name: 'Tom', age: {old: 22, new: 21}}
hb = {name: 'Jerry', age: {old: 20}}

ha.merge!(hb)

```

这时候，ha 就变为了{name: 'Jerry', age: {old: 20}} , 实际上是只想更改old 这个key的值。

对 hash字段值，别用 等号。

```ruby
user.fields = {name: 'Tom'}
user.save
```

上面的代码会把原有的fields的值完全覆盖。这时候应该做的是，

```ruby
result = user.fields[:name] = 'Tom'
user.fields = result
user.save
```

hash 的赋值操作，无处不是坑

##### Sidekiq执行

MyWorker.perform_async(notice) 异步执行

MyWorker.new.perform(notice) 同步执行会阻塞，可以在rails c中快速执行测试

MyWorker.perform_in(3.seconds, notice)  一定时间后执行

Sidekiq::Client.push('class' => MyWorker, 'args' => [1, 2, 3])  # Lower-level generic APISidekiq::Client.push('class' => MyWorker, 'args' => [1, 2, 3])  # Lower-level generic API

##### serialize fields 中的changes、 pervious_changes
如果某个字段 fields 是被序列化为数组或hash

```ruby
serialize fields, Array

serialize fields, Hash
```

这时想要得到对象的 changes、 pervious_changes值。 需要做的是 构造一个fields_deep_dup 的值，然后修改这个值，再覆盖这个fields字段。 如果，直接修改fields字段中的值，外层对象其实是不变的。changes、 pervious_changes 就得不到这个fields 的change

##### crontab的尝试
crontab -e

打开crontab的编辑界面，进行定时任务的vim编辑页面

```
*/30 * * * * /home/user/Documents/server.sh
```

然后保存。crontab会自动读取新的设置

crontab -l 可以列出你的设置内容

crontab -r 删除crontab文件

可以使用这种方法在$HOME目录中对crontab文件做一备份:

crontab -l > $HOME/mycron

如果没有使用vim对crontab进行编辑，可以编辑$HOME目录下的. profile文件，在其中加入这样一行:

```
EDITOR=vi; export EDITOR  
```

##### 三种类型的Elasticsearch node节点

配置文件中给出了三种配置高性能集群拓扑结构的模式,如下：

```
1. 如果你想让节点从不选举为主节点,只用来存储数据,可作为负载器
node.master: false
node.data: true
2. 如果想让节点成为主节点,且不存储任何数据,并保有空闲资源,可作为协调器
node.master: true
node.data: false
3. 如果想让节点既不称为主节点,又不成为数据节点,那么可将他作为搜索器,从节点中获取数据,生成搜索结果等
node.master: false
node.data: false
```

##### ElasticSearch各个目录说明

| type| description| location|
|------|------|-------|------|
|home	|Home of elasticsearch installation|	/usr/share/elasticsearch|
|bin|	Binary scripts including elasticsearch to start a node	|/usr/share/elasticsearch/bin|
|conf	|Configuration files elasticsearch.yml and logging.yml	|/etc/elasticsearch|
|conf	|Environment variables including heap size,file descriptors	|/etc/default/elasticsearch|
|data	|The location of the data files	|/var/lib/elasticsearch/|
|logs	|Log files location	|/var/log/elasticsearch|
|plugins| Plugin files location	|/usr/share/elasticsearch/plugins|

##### 使用 Elasticsearch 帮助你快速的count表数据
当一个表的数据到达千万甚至亿级别的时候，MySQL的 count操作的耗时已经惨不忍睹。如果，你的Elasticsearch中也有一份和MySQL几乎一样的数据库，可以用Elasticsearch帮你实现这种查询，和MySQL的count相比，真的是飞速。

在rails c中执行：

```ruby
time = "1909-07-27 16:42:04 +0800"
pa = {filtered: {filter: {bool: {must:[{range: {created_at: {gte: time}}}]}}}}
cs=Customer.__elasticsearch__.search(query: pa)
cs.results.total
```
这是一种简单的方法，当然还有别的方法。如果需要增加条件，则可以修改对应的pa hash的值就可以了

##### 用elasticsearch Edge NGram Tokenizer分词，优化前缀搜索

Edge NGram Tokenizer https://www.elastic.co/guide/en/elasticsearch/reference/1.4/analysis-edgengram-tokenizer.html

当长度等于3时，可以使用这个分词来进行前缀搜索匹配，而当长度大于3的时候，使用prefix 性能会更佳好。

##### 分布式简记
访问层 <-> 逻辑层 <-> 数据存储层
