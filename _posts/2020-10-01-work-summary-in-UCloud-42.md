---
layout: post
title: 最近工作总结(42)
date:  2020-10-01 18:15:06
categories: Work
image: /assets/images/post.jpg

---

 

### Elasticsearch+HBase的存储方案

Elasticsearch中存储需要的索引字段，完整的数据存储在HBase。HBase适合海量数据的在线存储，不适合复杂的搜索，但是简单的根据id或者范围查询性能上没有问题。从Elasticsearch中进行复杂搜索得到对应的doc id，然后再到HBase中查询完整的数据返回给前端。这样，Elasticsearch中只存储必要的索引字段，能够更节省内存，使得Elasticsearch能更高效的使用内存(filesystem cache)，从而获得更高的查询性能

### scroll 的分页方式

类似于微博中，下拉刷微博，刷出来一页一页的分页数据。性能会比上面说的那种分页性能要高很多很 多，基本上都是毫秒级的。缺点是：不能随意跳到任何一页的场景

### Redis持久化 AOF和RDB相结合

Redis 支持同时开启开启两种持久化方式，我们可以综合使用 AOF 和 RDB 两种持久化机 制，用 AOF 来保证数据不丢失，作为数据恢复的第一选择; 用 RDB 来做不同程度的冷备， 在 AOF 文件都丢失或损坏不可用的时候，还可以使用 RDB 来进行快速的数据恢复

