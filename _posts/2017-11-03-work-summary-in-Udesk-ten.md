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
