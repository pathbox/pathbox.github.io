---
layout: post
title: 最近工作总结(33)
date:  2019-11-05 11:12:06
categories: Work
image: /assets/images/post.jpg
---

### Golang原始 IN SQL语句构造

```
fmt.Sprintf("SELECT user_id,user_email FROM user WHERE company_id = %d AND user_email IN (%s);", companyID, "'11','22','33'")
```

### 优秀的独立博客收集列表

`https://github.com/timqian/chinese-independent-blogs`

### Go项目中强烈建议使用数据库ORM库来进行SQL操作

- Go项目中强烈建议使用数据库ORM库来进行SQL操作
- 用Go原生SQL库纯写SQL的方式实在太低效且非常难维护
- 不要自己拼接SQL查询，低效，且会很多坑

### 滑动窗口方法进行接口rate limit
>滑动窗口方法，因为它可以灵活地缩放比例速率并具有良好的性能。速率窗口是她将速率限制数据呈现给API使用者的一种直观方法。它还避免了漏斗的饥饿问题和固定窗口实现的爆裂问题

### 优雅的重启http server服务

目的:
不关闭现有连接（正在运行中的程序）
新的进程启动并替代旧进程
新的进程接管新的连接
连接要随时响应用户的请求，当用户仍在请求旧进程时要保持连接，新用户应请求新进程，不可以出现拒绝请求的情况
流程:
1、替换可执行文件或修改配置文件
2、发送信号量 SIGHUP
3、拒绝新连接请求旧进程，但要保证已有连接正常执行结束
4、启动新的子进程
5、新的子进程开始 Accet
6、系统将新的请求转交新的子进程
7、旧进程处理完所有旧连接后正常结束

### TiDB工作小结
- 加新字段不锁表
- 加索引不锁表,时间根据数据记录大小
- 将字符串长度改大可以，改小不行
- 修改字段默认值不锁表
- 当一个字段NOT Null 进行INSERT操作时，如果SQL语句没有该字段，会报错 `Field 'admin' doesn't have a default value`,而MySQL会自动帮你加入默认值而不报错
- TiDB没有锁表机制 所以DDL操作都是不缩表的

### 数据库用户权限限制IP地址访问段，增强安全性
将数据库用户权限设置为内网地址的网段，能够增强数据库访问的安全性
