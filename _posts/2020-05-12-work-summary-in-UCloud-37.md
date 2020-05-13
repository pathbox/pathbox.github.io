---
layout: post
title: 最近工作总结(37)
date:  2020-05-12 19:45:06
categories: Work
image: /assets/images/post.jpg
---

### 分布式ID生成的几种方案选择
https://www.cnblogs.com/cider/p/11776088.html

snowflake 会遇到时间回拨的问题，一种解决思路：https://juejin.im/post/5a7f9176f265da4e721c73a8

### 关于高可用服务的简单几点
- 服务无状态
- 幂等性
- 服务超时设置避免阻塞
- 异步-消息队列
- 高并发、高可用: 网关、限流、降级、缓存、容灾
- 数据: 备份、一致性(最终一致性)
- 集群，服务发现和服务注册

### Redis::CommandError: CROSSSLOT Keys in request don’t hash to the same slot
mset（Multi-key）del keys... 等命令，报错：
Redis::CommandError: CROSSSLOT Keys in request don’t hash to the same slot
原因： Redis cluster对多key操作有限，要求命令中所有的key都属于一个slot，才可以被执行。客户端可以对multi-key命令进行拆分，再发给redis。
另外一个局限是，在slot迁移过程中，multi-key命令特别容易报错(CROSSSLOT Keys in request don’t hash to the same slot)。建议不用multi-key命令。

解决： 在key名中增加{XXXX}，这样redis将仅使用XXXX来计算slot的位置
