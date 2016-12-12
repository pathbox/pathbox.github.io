---
layout: post
title: Elasticsearch 聚合搜索
date:   2016-12-12 17:50:06
categories: elasticsearch
image: /assets/images/post.jpg
---

这次掉了这个坑里。

原来的聚合操作：

```
curl -XPOST "localhost:9200/tickets/ticket/_search?pretty=1" -d '{"query":{"filtered": {"filter": {"bool": {"must": [{"term": {"company_id": 25007}}, {"terms": {"status_id":[1], "execution": "and"}}, {"terms": {"platform_id":[8], "execution": "and"}}], "must_not": [], "should": []}}}}, "size": 0,"aggs": {"group_by": {"terms": {"field": "SelectField_3523"}}}}'

```

这样的结果，只会默认进行十个条件的group by 聚合操作。

修改操作：

```
curl -XPOST "10.26.34.165:9200/tickets/ticket/_search?pretty=1" -d '{"query":{"filtered": {"filter": {"bool": {"must": [{"term": {"company_id": 25007}}, {"terms": {"status_id":[1], "execution": "and"}}, {"terms": {"platform_id":[8], "execution": "and"}}], "must_not": [], "should": []}}}}, "size": 0,"aggs": {"group_by": {"terms": {"field": "SelectField_3523","size":0}}}}'
```

```
"aggs": {"group_by": {"terms": {"field": "SelectField_3523","size":0}}}
```

这里增加

```
"size":0
```

整个查询有两个

```
"size":0
```

第一个表示不返回查找得到的记录结果，只是放回聚合的结果。主要就是buckets。

第二个表示聚合操作不使用默认的配置，进行十个条件的聚合，而是将字段的所有值都进行group by 并且放回。

不知其所以然，结果只能自己默默的填坑。。。。。。谨记
