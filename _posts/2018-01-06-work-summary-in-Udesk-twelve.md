---
layout: post
title: 最近工作总结(十二)
date:   2017-12-06 16:55:06
categories: Work
image: /assets/images/post.jpg
---

##### SLB(LVS)探测
阿里云SLB(LVS)对代理的服务端口进行的探测是`HEAD`方法请求。所以，你要定义一个`HEAD`根目录域名的接口

##### net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_recycle = 1

表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭；

当某个时段有大量请求来的时候，会产生很多的TIME-WAIT socket。如果这时候，被快速回收了，会导致socket 没有完全通讯完，结果就是这次调用失败了。
从而会出现大量的错误。所以，这个配置，思考清楚了再选择是否开启配置。
