---
layout: post
title: 最近工作总结(31)
date:  2019-09-02 19:55:06
categories: Work
image: /assets/images/post.jpg
---

### 只需五步，自己动手写一个静态博客
1. 收集markdown列表
2. 解析markdown源
3. 生成博客文章
4. 生成博客首页索引
5. 开始编译
http://muxueqz.top/a-small-static-site-generator.html
https://blog.thea.codes/a-small-static-site-generator/

### 领导者的三种模式

模式一："这就是我想要的，你按照我说的做。"

模式二："这就是我想要的，你自己想如何去做。"

模式三："让我们一起弄清楚我们能做些什么

### 单元测试如此重要,让你的代码更有信心

### 账号系统中，优惠券，赠金必然会存在媷羊毛漏洞问题

### Golang项目中使用db(将db设置为全局变量模式)

1. 将db设置为全局变量模式，整个项目只有一个db实例
2. 如果项目需要连接多个不同的数据库源，可以用map存储，一次初始化好所有db实例，放到全局的map中保存
3. 如果每个请求过来都新建一个connect db实例，这样浪费资源，如果并发稍微大一点，则会创建非常多的db实例连接，导致db服务器连接被耗尽而出现问题。其实我们希望的是，只有一个实例，最多使用该实例设置的连接池的连接数，这样才不会导致将db服务的连接耗尽的情况

### Golang map 内存的释放
Golang map delete 操作不会释放占用内存,会等到GC的时候,由GC选择释放。想要立即释放内存，需要将map设置为nil，即可立即释放所占内存

### mock in test
mock 在测试中的作用是解除外部依赖(数据库，文件依赖，第三方API调用)等，而不是为了测试依赖


### Golang slice map不是线程安全
Golang slice map 不是线程安全，如果定义了全局的slice和map，在读写slice和map数据时，请使用锁

### 四个抽象

- 文件是对I/O设备的抽象
- 虚拟内存是对程序存储器的抽象
- 进程是对一个正在运行的程序的抽象
- 虚拟机,它提供整个计算机的抽象，包括操作系统、处理器和程序

### Go 1.13 GO module 相关设置
在Go 1.13中，我们可以通过GOPROXY来控制代理，以及通过GOPRIVATE控制私有库不走代理。

设置GOPROXY代理：

go env -w GOPROXY=https://goproxy.cn,direct
设置GOPRIVATE来跳过私有库，比如常用的Gitlab或Gitee，中间使用逗号分隔：

go env -w GOPRIVATE=*.gitlab.com,*.gitee.com

如果在运行go mod vendor时，提示Get https://sum.golang.org/lookup/xxxxxx: dial tcp 216.58.200.49:443: i/o timeout，或者go mod init时会报13设置了默认的GOSUMDB的错误，则是因为Go 1.13设置了默认的GOSUMDB=sum.golang.org，这个网站是被墙了的，用于验证包的有效性，可以通过如下命令关闭：

go env -w GOSUMDB=off