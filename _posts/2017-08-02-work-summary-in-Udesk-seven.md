---
layout: post
title: 最近工作总结(七)
date:   2017-08-02 16:55:06
categories: Work
image: /assets/images/post.jpg
---

##### 使用正则快速获取括号内的字符串
result = file_name.match(/\A.*\((\S+)\).*\Z/)
result1 = file_name.match(/\A.*\[(\S+)\].*\Z/)
 p $1
 p result.to_a
