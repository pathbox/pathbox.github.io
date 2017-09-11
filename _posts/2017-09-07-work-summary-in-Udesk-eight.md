---
layout: post
title: 最近工作总结(八)
date:   2017-09-07 10:25:06
categories: Work
image: /assets/images/post.jpg
---

##### ActiveRecord reload
reload方法：数据库更新不可能反馈到变更前创建的对象上。通过reload方法让对象重新加载数据库最新的变更。

##### SaaS PaaS IaaS 开设一家披萨店
阮一峰关于 SaaS PaaS IaaS 的解释文章
http://www.ruanyifeng.com/blog/2017/07/iaas-paas-saas.html

##### 传current_user 还是传current_user_id

一个方法的参数，是选择 传current_user 还是 current_user_id

如果选择传 current_user， 在方法使用中往往还需要考虑，current_user 会不会为nil。为nil的时候，代码逻辑会不会报错，需要做什么处理。
而当传current_user_id的时候，如果方法中只是需要current_user.id，并不需要更多的current_user的属性，我会选择传current_user_id
因为我发现，传current_user会产生更复杂的情况
