---
layout: post
title: 最近工作总结(48)
date:  2021-04-08 20:00:00
categories: Work
image: /assets/images/post.jpg


---

​    

### 接口耗时暴增原因排查

起因: 调用方部门反馈某个内部接口耗时突然增加了，导致原本超时时间设置太小，而接口超时失败。

快速处理：调用方将接口超时时间增大到合适值

排查：通过监控查看该接口一周的调用情况，发现确实从2天前开始，接口耗时从100ms左右飙升到了3-4s。但是，接口调用请求量并没有暴涨，和之前差不多

- 先从代码入手，查看2天前的那个时间点后，代码上是否有变更导致。 结果：代码没有问题
- 构造一个测试数据，发起测试请求，得到的请求耗时正常100ms内。从日志中过找出一个耗时长的请求，重新测试，请求耗时3s
- 将接口中涉及到的所有SQL操作列出，搜索数据库慢日志，看是否有对应慢日志。结果：没有对应慢日志
- 根据耗时3s请求的参数，手动在数据库上执行SQL，SQL耗时正常，均在20ms完成。结果：SQL，数据库性能应该正常
- 接口业务中是否有调第三方API。结果：没有
- 接口业务中发现有用tcp方式非http方式调用第三方的服务。通过对比测试调用和不调用该tcp第三方服务，是该服务有问题。但由于这块老代码中，没有超时或者报错提示，导致日志中没有报错信息，问题被隐藏

解决方式：

- 通知该tcp服务负责人员
- 将该调用加上相应超时和报错信息日志
- 异步方式调用该tcp服务

还有哪些方向可以查：

- 局域网网络是否有问题
- DNS解析是否耗时过长
- 对应前置内部网关是否有问题
- 调用方部门的服务是否部署到了别的地域导致不再同一个局域网内



### go context cancel不执行会怎样

If you fail to cancel the context, the [goroutine that WithCancel or WithTimeout created](https://golang.org/src/context/context.go?s=9162:9288) will be retained in memory indefinitely (until the program shuts down), causing a memory leak. If you do this a lot, your memory will balloon significantly. It's best practice to use a `defer cancel()` immediately after calling `WithCancel()` or `WithTimeout()`

很有可能会导致内存泄漏(goroutine没有关闭,goroutine泄漏)

