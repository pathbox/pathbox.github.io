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
