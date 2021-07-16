---
layout: post
title: 最近工作总结(50)
date:  2021-07-16 21:00:00
categories: Work
image: /assets/images/post.jpg


---

​    

### 使用hash大幅度提高Redis value内存利用率

如果把要使用的 redis 数据都集中到一起，集中存放，则 value 的大小会远大于 key 和其他内存结构的大小，从而使内存利用率达到 50%~99%。然而此方案也有弊端：如果只想取某个子模块的数据也必须把整体数据都拉下来，无状态化的情况下本来就会频繁读写数据，此方案将显著增加 redis 的CPU压力。
redis 的 hash 类型既可以把数据集中存放，也支持 key 分开读写
