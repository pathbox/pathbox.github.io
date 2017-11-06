---
layout: post
title: 最近工作总结(十)
date:   2017-11-03 16:28:06
categories: Work
image: /assets/images/post.jpg
---

##### rails find_by 别忘了查询条件
线上出现了一个诡异的bug。查了很久终于发现是find_by查询没有加条件。举个例子：

```ruby
user = User.find_by(user_id)
```

结果导致user获取错误。(我的锅)

查了rails源码：

find_by源码

```ruby
def find_by(arg, *args)
  where(arg, *args).take
rescue ::RangeError
  nil
end
```

find_by就是where查询，加上take

在rails console中执行：

```ruby
User.find_by(1)
# => User Load (3.4ms)  SELECT `users`.* FROM `users` WHERE (1) LIMIT 1
```

得到的是表中的第一个user记录，所以导致了传的｀user_id｀　没有起到查询的作用，SQL语句默认是true而查询了进行了全表查询，之后再LIMIT　1　得到第一条记录

正确的代码是

```ruby
user = User.find_by(id: user_id)
```

Don't do it again

##### redis服务挂了
周末redis服务器挂了，持续了20分钟。影响到了登入后的初始界面，使得初始界面无法访问。初始界面会调的接口，应该没有redis请求的相关逻辑，避免客户连登入都登入不了。
而是哪个模块需要redis，在那个模块的接口再请求redis。如果不可避免需要请求redis，也许要思考redis集群方案，或redis挂了后及时的报警机制，以及时定位问题
单点故障就是像地雷一样
