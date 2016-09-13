---
layout: post
title: Rails 项目的数据库慢查询优化
date:   2016-09-13 21:40:06
categories: MySQL
image: /assets/images/post.jpg
---

最近项目中出现了很多MySQL的慢查询，平均耗时10s钟。运维小哥把慢查询找了出来，交给我们进行优化。本文就是本次MySQL慢查询优化的总结。

优化的表:　1500W+条记录,　65个字段。对，这是一个有65个字段的表。
