---
layout: post
title: 最近工作总结(35)
date:  2020-03-06 16:25:06
categories: Work
image: /assets/images/post.jpg
---

### 修改Go Module相关环境变量
```
export GOPROXY=https://goproxy.cn,direct
export GOSUMDB=sum.golang.google.cn  (此地址未被墙)
```

### tcpkali 进行websocket的压力测试
```
tcpkali --ws -c 100 -m 'hello world!!13212312!' -r 10k localhost:8081
```
