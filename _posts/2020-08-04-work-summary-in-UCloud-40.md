---
layout: post
title: 最近工作总结(40)
date:  2020-08-04 17:30:06
categories: Work
image: /assets/images/post.jpg
---

### elasticsearch match,term,match_phrase简记
https://blog.csdn.net/sinat_29581293/article/details/81486761

### 同步数据跑的脚本要具有幂等性

### golang channel runtime.chansend
- 当存在等待的接收者时，通过 runtime.send 直接将数据发送给阻塞的接收者；
- 当缓冲区存在空余空间时，将发送的数据写入 Channel 的缓冲区；
- 当不存在缓冲区或者缓冲区已满时，等待其他 Goroutine 从 Channel 接收数据
