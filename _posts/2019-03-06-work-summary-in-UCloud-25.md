---
layout: post
title: 最近工作总结(25)
date:  2019-03-06 10:18:06
categories: Work
image: /assets/images/post.jpg
---

### sendFile in Kafka
Kafka 在数据写入及数据同步采用了零拷贝(zero-copy)技术，采用sendFile()函数调用，
sendFile() 函数是在两个文件描述符之间直接传递数据，完全在内核中操作，
从而`避免了内核缓冲区与用户缓冲区之间数据的拷贝`，操作效率极高
