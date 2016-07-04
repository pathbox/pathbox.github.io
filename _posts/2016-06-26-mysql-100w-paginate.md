---
layout: post
title: MySQL百万以上记录表分页优化
date:   2016-06-26 11:02:06
categories: algorithm
image: /assets/images/post.jpg
---

### MySQL百万以上记录表分页优化

我们知道，当MySQL表的数据达到百万以上甚至千万的时候，整个表的查询性能是会下降的。如果遇到更大的数据时，有一些解决方案，比如分表分库。
看了一些文章有讲“关于百万以上记录表的分页优化”，所以今天实际操作总结一下。

##### 构造有100w条测试记录数据

首先，要先构造有100w条测试数据记录的表。网上有不少解决方案。这里我选择在rails环境下构造。
创建一张'test_products'表

```ruby
class CreateTestProducts < ActiveRecord::Migration
  def change
    create_table :test_products do |t|
      t.string :name
      t.string :number
      t.string :desc
      t.string :category
      t.string :field1
      t.string :field2
      t.string :field3
      t.string :field4
      t.string :field5
      t.string :field6
      t.string :field7
      t.string :field8
      t.string :field9
      t.text :field10
      t.timestamps null:false
    end
  end
end
```

用ruby创建一个100w个id数的txt文件

```
ruby -e "(1..1000000).each{|i| puts i.to_s+',good name,1000000,nice product,food,field1,field2,field3,field4,field5,field6,field7,field8,field9,field10,'+Time.now.strftime('%Y-%m-%d %H:%M:%S')+','+Time.now.strftime('%Y-%m-%d %H:%M:%S')}" > 100w.txt
```
用了7秒创建了100w.txt文件

进入mysql客户端进行导入操作。

```
 mysql> load data infile 'd.txt' into table tt
        -> fields terminated by','
        -> lines terminated by'\n'
 Query OK, 1000000 rows affected (52.86 sec)
```
用了53秒完成导入。

mysql提供了多种导入txt文件的条件,如果txt文件是这样的
文件d.txt的内容示例:
 1
 2
 3

```
    mysql> load data infile '/Users/path/code/say_morning/1000.txt' into table test_products
        -> lines terminated by'\n'
        -> (id);
```
导出到txt文件示例：

```
mysql> select * from tt into outfile 'd.txt'
    -> fields terminated by','
    -> lines terminated by'\n'
```

注意 mac os系统下这里的换行符是 '\n'；如果是windows系统是 '\r\n'

这样，在很短的时间内，就可以构造一张简单的有100w条数据记录，十几个字段的表了。实际产生数据和导入数据在60秒内可以完成。
如果，用批量插入的方法去构造100w条测试数据记录的话，显然所需的时间远远多于60s。

关于批量导入，mongo的批量插入更为惊人。你可以向下面这样(在用的mongoid):

```ruby
batch = []
100_00_00.times.each{batch << {name: 'name', age: '25', sex: 1}}
Person.collection.insert(batch)
```
速度是非常快的。

##### 分页测试
发现产生的数据其实不够复杂。表才140M+。所以，我又构造了一个更为复杂的表，方法和上面描述的一样。当然所需要的时间也更多了一些.

有了测试数据，我们开始分页测试。

1.   直接用limit start, count分页语句， 也是一般程序中用的方法：

```
mysql> select * from test_products limit 100, 30;  0.01sec
mysql> select * from test_products limit 10000, 30;  0.07sec
mysql> select * from test_products limit 100000, 30;  23.23sec
mysql> select * from test_products limit 900000, 30;  120.06sec
```

总结出两件事情：
1）limit语句的查询时间与起始记录的位置成正比
2）mysql的limit语句是很方便，但是对记录很多的表并不适合直接使用。后面的分页速度直接就到60秒以上了


##### 分页优化

利用表的覆盖索引来加速分页查询

因为利用索引查找有优化算法，且数据就在查询索引上面，不用再去找相关的数据地址了，这样节省了很多时间。另外Mysql中也有相关的索引缓存，在并发高的时候利用缓存就效果更好了。
我们知道id字段是主键，自然就包含了默认的主键索引

那么如果我们也要查询所有列，有两种方法，一种是id>=的形式，另一种就是利用join，看下实际情况：

```
SELECT * FROM test_products WHERE ID >= (select id from test_products limit 900000, 1) limit 30
30 rows in set (0.21 sec)
```

另一种写法：

```
SELECT * FROM test_products a JOIN (select id from test_products limit 900000, 30) b ON a.ID = b.id
30 rows in set (0.24 sec)
```

rails中一种简单的分页优化：

```ruby
@postids = Post.paginate(:page => params[:page]).order('created_at DESC').pluck(:id)
@posts = Post.where('posts.id' => @postids).order('posts.created_at DESC').eager_load [:user]
```

总结：

利用索引id，快速的找到需要条件的记录id，再通过这些id进行下一步的查找，得到分页所需要的数据。
虽然这里用到了嵌套查询，但是嵌套查询只是为了找到满足条件的id字段，加上id是主键索引，查找的速度非常快。所以说，嵌套查询也不是一定就会很慢哦。

本次实际优化操作收获：合理使用索引，特别是id主键索引进行数据库的优化，会有奇效。不要害怕嵌套查询或是其他的复杂查询，他们其实不慢，慢的是选择的查询条件或方法错了。
具体的优化，和表有关系。


本文完。

参考文章: https://ruby-china.org/topics/27004
         http://blog.jobbole.com/102520/







