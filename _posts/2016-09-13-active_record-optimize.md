---
layout: post
title: Rails 项目的数据库慢查询优化
date:   2016-09-13 21:40:06
categories: MySQL
image: /assets/images/post.jpg
---

最近项目中出现了很多MySQL的慢查询，平均耗时10s钟。运维小哥把慢查询找了出来，交给我们进行优化。本文就是本次MySQL慢查询优化的总结。

优化的表:　某表 1000W+条记录,　70个字段。对，这是一个有70个字段的表。

##### N+1 问题
很多问题都是N+1问题导致的，这种问题使用includes　一般都能有效的解决。在使用includes时，有两种情况产生。一种是产生连接表的sql查询，
一种是执行两次简单的sql语句查询。第一次会找到满足的对象，第二句再根据外键将第一句的查询结果进行in条件查询。示例：

```
1. SELECT `users`.* FROM `users` WHERE `users`.`c_id` = 1 AND `users`.`r_id` = 2
2. SELECT `ys`.* FROM `ys` WHERE `ys`.`k` = 'f' AND `ys`.`user_id` IN (2, 3, 4, 5, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100, 101, 102, 103, 104, 105, 106, 107, 108, 109, 110, 111, 112, 113, 114, 115, 116, 117, 118, 119)
```

使用连接表有可能由于表的数据和结构造成执行效率低，而转换为两次查询，执行效率会有很大的提升。但是，第二种情况的查询方式不是万能的，有一个很明显的缺陷，就是IN条件的数组的大小。
在我优化的过程中，如果IN条件的数组大小在5000以内，执行速度是很快的，如果达到了1w甚至更多，这时候的查询往往会非常慢，即使对IN的字段做了索引。有一种说法是，IN条件的数组大小超过
一定数量时，会导致索引失效而进行全表扫描，所以就慢了。
我就遇到了这个IN条件的数组是2w大小的情况，结果查询直接到了120多秒。这时候，可以选择可以选择left join连接
表的方式，或者union、子查询等比较执行速度，得到最好的优化方法。

##### 正确的姿势创建和使用索引

如果数据库有重复的索引，请删除一个。在优化时，遇到一个表中两个一模一样的索引，只是索引名字不同，在删除其中一个索引后，查询速度立马正常了。

尽量使用联合索引，删除多余的单个索引。对大小于比较(> <)的where查询，对这个字段建立索引或联合索引，执行速度明显加快。
我在优化的时候，建立2-3个字段的联合索引，效果最好。

##### first 也许不是最好的选择
在某个查询想取第一个记录时, 使用take 或 limit(1)　代替first，速度会变快。
看下面的两个查询：

```
email = c.cs.where("email like ?", "#{email}\_%\@temp.com").try(:first).try(:email)
SELECT `cs`.* FROM `cs` WHERE  `cs`.`c_id` = 1 AND (email like "xxxxx_%@temp.com") ORDER BY `cs`.`id` ASC LIMIT 1;

email = c.cs.where("email like ?", "#{email}\_%\@temp.com").take.try(:email)
SELECT `cs`.* FROM `cs` WHERE  `cs`.`c_id` = 1 AND (email like "xxxxx_%@temp.com") LIMIT 1;
```

对c_id和email创建联合索引。使用first　需要16s, 使用take　用了0.01s。

first其实用到了order by　id　没用到company_id,email的联合索引,而是用了primary使得在like查询上耗费了大量时间。
这让我知道了order by 不是可以乱用了，假如没有给order by 的字段加合理的索引，也许查询瓶颈就是出在了order by上。

##### 使用　explain　进行分析,　能给予你一个好的解决思路
如果还不知道explain的使用，请务必好好的研究，它为你分析sql查询带来了极大的帮助。

##### 关于count
count(*) 通常比count(id) 的效果在各种情况下更好。 count(*) from tables 效率最高， count(*) from tables where 会进行全表扫描，
所以最好给where 的字段建立索引。在本次优化中就遇到这种情况，加上联合索引后，原本需要6秒的count操作只需要0.1秒了

##### count group by 的复杂查询
C.where(o_g_id: o_g_ids,
         c_id: c.id,
         deleted: 0)
  .group(:o_g_id)
  .count
SELECT COUNT( * ) AS count_all, o_g_id AS o_g_id FROM `cs` WHERE `c`.`o_g_id` IN (1, 3, 1731, 9860, 9929, 15805, 15806, 15810, 15937, 15940, 15941, 15942, 15946, 15997, 16007, 19453, 19519, 19732, 20603, 20946, 20956, 22279, 23697, 25003, 25009, 25018, 25170, 26086, 26289, 26655, 26656, 27507, 28301) AND `cs`.`c_id` = 26 AND `cs`.`deleted` = 0 GROUP BY o_g_id

为 c_id, deleted, o_g_id 加上联合索引，这样优化就非常明显了。直接原来需要5秒的查询变为了0.6秒

##### 另一种思路的比较，尽量让MySQL做更多事情，而不是转为数组

> 有一个查询是这样的 if　cs.pluck(:email).uniq.include? email

有时候这句查询需要6秒的时间，一般情况是3秒。通过explain分析，这个查询的row达到了300+w

> 优化成 if cs.where(email: email).plesent?

email原本有索引。结果只需要0.05秒，row = 1。而且不耗内存，第一种方式产生的数组有可能消耗一定内存
present? 方法还可以用　exists?代替

##### 复杂的MySQL查询让索引失效，拆成多个简单的查询。
如果遇到了复杂的MySQL查询，有可能让索引失效，这时候要么重新找能合理使用上索引的方式，要么将复杂的查询拆分成简单的查询，也往往可以得到很好的效果。
一个查询得到xx.id数组集合的sql使用了or查询方式，结果使得联合索引失效。我拆成了两个简单的where查询，在用pluck得到id数组，最后将两个id数组一起返回。原本需要10几秒的查询，现在只需要0.8秒。
分析了下得到的数组可能的最大值是2w，多消耗的内存没有影响。

这次优化过程，我受益匪浅。在早期数据量不大的情况下，一切都很正常。只有当数据量大的时候，才能暴露更多的问题。真的别说ActiveRecord慢，你的项目真的到了需要完全不用ORM而手写sql语句才能有好的性能的时候，我想，那时候你的项目需要的是架构上的重构了吧，比如使用更多的集群，使用哪种集群的架构，使用哪种数据库等等。我不能说ActiveRecord真是快，ActiveRecord存在的意义也不是为了快，而是为了便捷。
但是，ActiveRecord不慢，而是你的使用姿势不正确罢了(说白了，就是你功底差，写的查询都是xxx)

##### 关于联合索引
where条件的顺序是不重要的，可以认为是无序的。mysql会寻找最优的索引进行查询。关键的是联合索引的顺序要满足创建索引条件的规则。举个例子:

```
where("a = 1 and c > 2 and b = 3 ")
```

index(a,b,c) 和index(a,c,b) 这两个索引，能够使用到index(a,b,c) 而索引 index(a,c,b) 其实相当于使用到了index(a,c) b条件并不会在索引中使用到。因为 > 条件会终止查找后面条件的索引。

##### order(id: :asc) vs order(created_at: :asc)
一个500w+ 数据的表A

```
A.where(aa_id: #{aa_id}).where(created_at: start_time..end_time).order(id: :asc)
```
建立联合索引: create index idx_a_aa_id_created_at
使用explain 该sql，发现没有使用联合索引，而是使用的主键，需要2s左右的时间。
和预测的不一样，容易发现，原因在于order(id: :asc)，导致MySQL选择使用了主键作为索引。
更改代码:

```
A.where(aa_id: #{aa_id}).where(created_at: start_time..end_time).order(created_at: :asc)
```

这样联合索引就使用到了，只用了0.5ms查询出结果，nice! 优化成功
所以，在order by的时候，不一样选择id，当数据量大且查询sql条件变多变复杂的时候，就需要实际测试了。

##### or is not good， it make a slow slow sql

在model B中有一个scope

```
scope :b_show, ->(a_id) do
  user_ids = B.users.pluck(:id)
  where("user_id  in (?) or a_id  = ?", user_ids, a_id)
end
```

查询的瓶颈在于or查询。

放弃or 查询语句，拆成多个查询语句

```
scope :b_show, ->(a_id) do
  user_ids = B.users.pluck(:id)
  ids_1 = B.where(user_id: user_ids).pluck(:id)
  ids_2 = B.where(a_id: a_id).pluck(:id)
  ids = (ids_1 + ids_2).uniq
  where(id: ids)
end
```
速度一下由s级别变为几十ms。是的，这样多了几个查询，但是，速度快了100倍。
但是，这里有个注意点，就是ids元素个数。如果得到的ids元素个数是100w，那也会产生性能问题。
因为最后一句 where(id: ids) 是sql的 IN 查询语句。亲自测试，当ids元素个数为10w一下时，
where(id: ids) 速度都可以很快。 这得益于id主键。 如果你IN 的字段不是主键，且没有加索引，
也会产生性能问题。可以简单的概况为下面的性能比较式子:

```
IN 主键 >= IN 唯一索引 >= IN 不唯一索引 > IN 无索引字段
```

所以，在一定程度，可以放心的使用IN。其实，Rails include语句，在解决N+1 问题的时候，
最后一条sql就是使用IN id主键的查询

优化仍在继续，将继续积累总结
