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
