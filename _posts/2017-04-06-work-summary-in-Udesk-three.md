---
layout: post
title: 最近工作总结(三)
date:   2017-04-06 17:32:06
categories: Work
image: /assets/images/post.jpg
---

### !!符号
!! 符号可以将nil转为true之后，再转为false。这样可以将false或nil都以false结果进行判断

```ruby

!!(Integer(id) rescue nil)
```
