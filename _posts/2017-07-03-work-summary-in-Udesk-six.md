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
