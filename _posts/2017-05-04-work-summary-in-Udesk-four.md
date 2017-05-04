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

##### https域名中有http的请求
https的域名下发起http的请求，浏览器会认为发起了不安全的请求而报错。
将该请求也使用https
一般https域名的地址，当使用http协议访问时，Nginx只要做了http到https的重定向，就能重定向到https协议下
