---
layout: post
title: 最近工作总结(45)
date:  2021-01-05 20:00:00
categories: Work
image: /assets/images/post.jpg


---

 

### Docker 服务异常导致所有容器失效

由于主机服务突然断电或重启，导致Docker服务所有容器缺失了state.json文件。`docker info` `docker ps`等命`令会卡住，无法继续执行。到/var/log/docker 中查看输出日志,得到: 

```sh
evel=fatal msg="open /var/run/docker/libcontainerd/containerd/054f92393f757e0418b014ed1fa35673fbce2293de43e42153f4e10ec4910c77/state.json: no such file or directory
```

 解决方式:

```
1.service docker stop
2.Go to /var/run/docker and delete any directory related to the container id
3.Go to /var/lib/docker and delete any directory related to the container id
4.service docker start
```

有可能需要将目录下的所有 `container id`的文件目录都删了，因为他们都失效了。

docker 重启后，会恢复正常。但是容器都没有了，需要重新创建过。

如果是有CI/CD的，重新发布过服务即可，如果没有，那么所有失效容器重新创建会是比较繁琐了过程



### 在事务中的查询操作

在事务中的查询操作，如果查询的数据是事务中前面修改的数据，也要使用事务tx实例进行查询操作，如果用别的DB实例，得到的是没有修改或插入的数据，会导致诡异的不一致问题出现

### 如果是将没有值的缓存key的值存为null，则相关数据新增时，要将对应的缓存删除，否则取到的缓存有可能还是null



### mysql错误Every derived table must have its own alias解决

把MySQL语句改成：select * from (select * from ……) as 别名

### 注意redis key 前后不要有空格

### MySQL5.7 or查询使用索引情况

如果只是一个字段  

```
where name = 'foo' or name = 'bar' 
```

这样可以使用到索引

两个字段以上即使满足最左前缀索引也会很可能为全表扫描

```
where name = 'foo' and age = 10 or age = 20 
```



### MySQL explain type的简明意思

```
表示MySQL在表中找到所需行的方式，又称“访问类型”。

常用的类型有： ALL, index,  range, ref, eq_ref, const, system, NULL（从左到右，性能从差到好）

ALL：Full Table Scan， MySQL将遍历全表以找到匹配的行

index: Full Index Scan，index与ALL区别为index类型只遍历索引树

range:只检索给定范围的行，使用一个索引来选择行

ref: 表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值

eq_ref: 类似ref，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配，简单来说，就是多表连接中使用primary key或者 unique key作为关联条件

const、system: 当MySQL对查询某部分进行优化，并转换为一个常量时，使用这些类型访问。如将主键置于where列表中，MySQL就能将该查询转换为一个常量,system是const类型的特例，当查询的表只有一行的情况下，使用system

NULL: MySQL在优化过程中分解语句，执行时甚至不用访问表或索引，例如从一个索引列里选取最小值可以通过单独索引查找完成。

ref

表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值

 

rows

 表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数

 
Extra

该列包含MySQL解决查询的详细信息,有以下几种情况：

Using where:列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的，这发生在对表的全部的请求列都是同一个索引的部分的时候，表示mysql服务器将在存储引擎检索行后再进行过滤

Using temporary：表示MySQL需要使用临时表来存储结果集，常见于排序和分组查询

Using filesort：MySQL中无法利用索引完成的排序操作称为“文件排序”

Using join buffer：改值强调了在获取连接条件时没有使用索引，并且需要连接缓冲区来存储中间结果。如果出现了这个值，那应该注意，根据查询的具体情况可能需要添加索引来改进能。

Impossible where：这个值强调了where语句会导致没有符合条件的行。

Select tables optimized away：这个值意味着仅通过使用索引，优化器可能仅从聚合函数结果中返回一行

```

