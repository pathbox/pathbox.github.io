---
layout: post
title: 最近工作总结(28)
date:  2019-06-01 17:00:06
categories: Work
image: /assets/images/post.jpg
---

### 在一个SaaS或PaaS系统中,唯一属性的是company_id(company)

在一个SaaS或PaaS系统中，具有唯一性的应该是company_id，而不是身份证、手机号或邮箱。一个手机号可以属于不同的company。 所以，在设置表的主键时，往往和company_id一起设置为联合主键，这样才是合理的
