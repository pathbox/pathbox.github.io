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
