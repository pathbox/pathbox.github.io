---
layout: post
title: 最近工作总结(四)
date:   2017-05-04 14:57:06
categories: Work
image: /assets/images/post.jpg
---

##### 在数据库层面建立唯一索引，是最好的防止数据重复的方法，即使是在高并发的情况下
在Rails model层使用validates　方法进行唯一性的验证逻辑，比如:

```ruby
validates :content, uniqueness: { scope: [:company_id], message: "%{value}已经使用" }
```
然而，当高并发时,两个请求相差0.4s，这个在model层的验证并不是真正原子性的，是的这层验证失效。
如果能在数据库建立唯一索引[company_id, content]，这样，对高并发情况，也能支持验证，防止数据库产生脏数据。

##### union 出现在request body中时,会被阿里高防识别为可疑攻击而返回405。
比如 MySQL 的union注入攻击
