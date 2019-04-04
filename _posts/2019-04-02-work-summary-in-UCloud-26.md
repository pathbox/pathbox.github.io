---
layout: post
title: 最近工作总结(26)
date:  2019-04-02 17:00:06
categories: Work
image: /assets/images/post.jpg
---

### Microservices Concerns

- Config Management                  配置中心
- Auto Scaling & Self Healing        自动扩展和自我修复
- Scheduling & Deployment            调度和部署
- Distributed Tracing                分布式追踪
- Centralized Metrics                集中式度量中心
- Centralized Logging                集中式日志中心
- Service Security                   服务安全
- API Management                     API管理
- Resilience & Fault Tolerance       服务弹力性和失败容忍，节流
- Service Discovery & LB             服务发现与负载均衡

### slice的扩容规则
当原 slice 容量小于 1024 的时候，新 slice 容量变成原来的 2 倍；原 slice 容量超过 1024，新 slice 容量变成原来的>=1.25倍，因为进行了内存对齐操作

### 上游超时必须始终大于下游总超时
上游超时必须始终大于下游总超时（包括重试次数

上游超时应设置在“边缘”服务器上并始终级联

例子：两个OpenResty， A，B，A在B的前面。A的请求连接超时 < B的请求连接超时。 一种情况：当A因超时和B之间的请求连接断开了，B的返回就无法到达A了，从而出现诡异的报错
