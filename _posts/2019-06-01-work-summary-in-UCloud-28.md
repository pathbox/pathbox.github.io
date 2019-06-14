---
layout: post
title: 最近工作总结(28)
date:  2019-06-01 17:00:06
categories: Work
image: /assets/images/post.jpg
---

### 在一个SaaS或PaaS系统中,唯一属性的是company_id(company)

在一个SaaS或PaaS系统中，具有唯一性的应该是company_id，而不是身份证、手机号或邮箱。一个手机号可以属于不同的company。 所以，在设置表的主键时，往往和company_id一起设置为联合主键，这样才是合理的

### 数据库集群同步对业务的一个影响

一个写操作在node2节点， 写入操作之后会调用mq推送给另一个业务，该业务调接口请求得到这个数据，请求的是总库，结果没拿到该数据。是因为，node2节点的数据库同步到总库需要时间，且时间大于业务逻辑请求总库时获取数据的时间，数据还没同步到总库。

解决方案: 请求从库的接口、 使用sleep方法+补充措施
