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
