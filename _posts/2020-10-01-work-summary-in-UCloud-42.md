---
layout: post
title: 最近工作总结(42)
date:  2020-10-01 18:15:06
categories: Work
image: /assets/images/post.jpg

---

 

### Elasticsearch+HBase的存储方案

Elasticsearch中存储需要的索引字段，完整的数据存储在HBase。HBase适合海量数据的在线存储，不适合复杂的搜索，但是简单的根据id或者范围查询性能上没有问题。从Elasticsearch中进行复杂搜索得到对应的doc id，然后再到HBase中查询完整的数据返回给前端。这样，Elasticsearch中只存储必要的索引字段，能够更节省内存，使得Elasticsearch能更高效的使用内存(filesystem cache)，从而获得更高的查询性能

