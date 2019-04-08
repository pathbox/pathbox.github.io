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

### Golang的goroutinue调度器
Golang老的goroutinue调度器模型是：`M → G` 模型

缺点

1. 只有一个G全局队列，取g时需要加锁，导致锁竞争激烈

2. M转移G会造成延迟和额外的系统负载。比如当G中包含创建新协程的时候，M创建了G’，为了继续执行G，需要把G’交给M’执行，也造成了很差的局部性，因为G’和G是相关的，最好放在M上执行，而不是其他M'。

2. M中的mcache是用来存放小对象的，mcache和栈都和M关联造成了大量的内存开销和差的局部性。

3. 系统调用导致频繁的线程阻塞和取消阻塞操作增加了系统开销

Golang新的调度器模型: `M → P → G`
在用户态引入了`P`

两层调度关系

优点

1. 有全局的G队列，P也有本地G队列，这样将G分到多个G队列中，极大的减小了锁的竞争
2. work stealing，当本线程无可运行的G时，尝试从其他线程绑定的P偷取G，而不是销毁线程
3. hand off，当本线程因为G进行系统调用阻塞时，线程释放绑定的P，把P转移给其他空闲的线程执行
4. 抢占式调度: 在coroutine中要等待一个协程主动让出CPU才执行下一个协程，在Go中，一个goroutine最多占用CPU 10ms，防止其他goroutine被饿死，这就是goroutine不同于coroutine的一个地方
