---
layout: post
title: 最近工作总结(十二)
date:   2017-12-06 16:55:06
categories: Work
image: /assets/images/post.jpg
---

##### SLB(LVS)探测
阿里云SLB(LVS)对代理的服务端口进行的探测是`HEAD`方法请求。所以，你要定义一个`HEAD`根目录域名的接口
