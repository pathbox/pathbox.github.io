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

##### Many RUN is a shit in Dockerfile

`shit`
用了N层，有几个RUN就有几层

```
FROM	debian:jessie
RUN	apt-get	update
RUN	apt-get	install	-y	gcc	libc6-dev	make
RUN	wget	-O	redis.tar.gz	"http://download.redis.io/releases/redi
s-3.2.5.tar.gz"
RUN	mkdir	-p	/usr/src/redis
RUN	tar	-xzf	redis.tar.gz	-C	/usr/src/redis	--strip-components=1
RUN	make	-C	/usr/src/redis
RUN	make	-C	/usr/src/redis	install
```

`nice`

这样才用了一层
```
FROM	debian:jessie
RUN	buildDeps='gcc	libc6-dev	make'	\
				&&	apt-get	update	\
				&&	apt-get	install	-y	$buildDeps	\
				&&	wget	-O	redis.tar.gz	"http://download.redis.io/releases/r
edis-3.2.5.tar.gz"	\
				&&	mkdir	-p	/usr/src/redis	\
				&&	tar	-xzf	redis.tar.gz	-C	/usr/src/redis	--strip-component
s=1	\
				&&	make	-C	/usr/src/redis	\
				&&	make	-C	/usr/src/redis	install	\
				&&	rm	-rf	/var/lib/apt/lists/*	\
				&&	rm	redis.tar.gz	\
				&&	rm	-r	/usr/src/redis	\
				&&	apt-get	purge	-y	--auto-remove	$buildDeps
```

##### 变量取名思考

根据变量类型取名

根据变量功能取名
