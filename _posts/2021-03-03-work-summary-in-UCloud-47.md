---
layout: post
title: 最近工作总结(47)
date:  2021-03-03 20:00:00
categories: Work
image: /assets/images/post.jpg


---

​    

### etcd集群启动方式

```

$ etcd -name infra0 -initial-advertise-peer-urls http://10.0.1.10:2380 \
 -listen-peer-urls http://10.0.1.10:2380 \
 -initial-cluster-token etcd-cluster-1 \
 -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
 -initial-cluster-state new

$ etcd -name infra1 -initial-advertise-peer-urls http://10.0.1.11:2380 \
 -listen-peer-urls http://10.0.1.11:2380 \
 -initial-cluster-token etcd-cluster-1 \
 -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
 -initial-cluster-state new

$ etcd -name infra2 -initial-advertise-peer-urls http://10.0.1.12:2380 \
 -listen-peer-urls http://10.0.1.12:2380 \
 -initial-cluster-token etcd-cluster-1 \
 -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
 -initial-cluster-state new

```



### **SQL JOIN 中 on 与 where 的区别**

- **left join** : 左连接，返回左表中所有的记录以及右表中连接字段相等的记录。
- **right join** : 右连接，返回右表中所有的记录以及左表中连接字段相等的记录。
- **inner join** : 内连接，又叫等值连接，只返回两个表中连接字段相等的行。
- **full join** : 外连接，返回两个表中的行：left join + right join。
- **cross join** : 结果是笛卡尔积，就是第一个表的行数乘以第二个表的行数。

**关键字 on**
数据库在通过连接两张或多张表来返回记录时，都会生成一张中间的临时表，然后再将这张临时表返回给用户。
在使用 **left jion** 时，**on** 和 **where** 条件的区别如下：

- 1、 **on** 条件是在生成临时表时使用的条件，它不管 **on** 中的条件是否为真，都会返回左边表中的记录。
- 2、**where** 条件是在临时表生成好后，再对临时表进行过滤的条件。这时已经没有 **left join** 的含义（必须返回左边表的记录）了，条件不为真的就全部过滤掉。

假设有两张表：
**表1：tab2**
idsize110220330
**表2：tab2**
sizename10AAA20BBB20CCC
两条 SQL:
select * form tab1 left join tab2 on (tab1.size = tab2.size) where tab2.name='AAA'
select * form tab1 left join tab2 on (tab1.size = tab2.size and tab2.name='AAA')

第一条SQL的过程：
1、中间表
on条件:
tab1.size = tab2.sizetab1.idtab1.sizetab2.sizetab2.name11010AAA22020BBB22020CCC330(null)(null)
2、再对中间表过滤
where 条件：
tab2.name='AAA'tab1.idtab1.sizetab2.sizetab2.name11010AAA
第二条SQL的过程：
1、中间表
on条件:
tab1.size = tab2.size and tab2.name='AAA'
(条件不为真也会返回左表中的录)

tab1.idtab1.sizetab2.sizetab2.name11010AAA220(null)(null)330(null)(null)
其实以上结果的关键原因就是 **left join、right join、full join** 的特殊性，不管 **on** 上的条件是否为真都会返回 **left** 或 **right** 表中的记录，**full** 则具有 **left** 和 **right** 的特性的并集。 而 **inner jion**没这个特殊性，则条件放在 **on** 中和 **where** 中，返回的结果集是相同的



###MySQL中通过EXPLAIN如何分析SQL的执行计划详解

**1、type=ALL，全表扫描，MySQL遍历全表来找到匹配行**

一般是没有where条件或者where条件没有使用索引的查询语句

```text
EXPLAIN SELECT * FROM customer WHERE active=0;
```

![img](https://pic3.zhimg.com/80/v2-bff660f59d12171048251740021d73e2_1440w.png)

**2、type=index，索引全扫描，MySQL遍历整个索引来查询匹配行，并不会扫描表**

一般是查询的字段都有索引的查询语句

```
EXPLAIN SELECT store_id FROM customer;
```

![img](https://pic1.zhimg.com/80/v2-07e410c4dcf2c13020145255b4646468_1440w.png)

**3、type=range，索引范围扫描，常用于<、<=、>、>=、between等操作**

```
EXPLAIN SELECT* FROM customer WHEREcustomer_id>=10 ANDcustomer_id<=20;
```

![img](https://pic1.zhimg.com/80/v2-d4ecd00d262eed0fdde2f319b1a1b5a4_1440w.png)

**注意**这种情况下比较的字段是需要加索引的，如果没有索引，则MySQL会进行全表扫描，如下面这种情况，create_date字段没有加索引：

EXPLAIN SELECT * FROM customer WHERE create_date>='2006-02-13' ;

![img](https://pic2.zhimg.com/80/v2-0038deb6db359392311eb0fd5ce2a9ad_1440w.png)

**4、type=ref，使用非唯一索引或唯一索引的前缀扫描，返回匹配某个单独值的记录行**

`store_id`字段存在普通索引（非唯一索引）

```
EXPLAIN SELECT* FROMcustomer WHEREstore_id=10;
```

![img](https://pic4.zhimg.com/80/v2-28704c3ccb414d8fda75940abb5b753f_1440w.png)

ref类型还经常会出现在join操作中：

customer、payment表关联查询，关联字段`customer.customer_id`（主键），`payment.customer_id`（非唯一索引）。表关联查询时必定会有一张表进行全表扫描，此表一定是几张表中记录行数最少的表，然后再通过非唯一索引寻找其他关联表中的匹配行，以此达到表关联时扫描行数最少。

![img](https://pic2.zhimg.com/80/v2-21e872d4ee49067d02d47133760ab2ad_1440w.jpg)

因为customer、payment两表中customer表的记录行数最少，所以customer表进行全表扫描，payment表通过非唯一索引寻找匹配行。

```text
EXPLAIN SELECT * FROM customer customer INNER JOIN payment payment ON customer.customer_id = payment.customer_id;
```

![img](https://pic2.zhimg.com/80/v2-87a82a6070843ce025ead199d9b03f39_1440w.png)

**6、type=const/system，单表中最多有一条匹配行，查询起来非常迅速，所以这个匹配行的其他列的值可以被优化器在当前查询中当作常量来处理**

const/system出现在根据主键primary key或者 唯一索引 unique index 进行的查询

根据主键primary key进行的查询：

```
EXPLAIN SELECT* FROMcustomer WHEREcustomer_id =10;
```

![img](https://pic2.zhimg.com/80/v2-33782e55c986dc5b6bca0e89cde49531_1440w.png)

根据唯一索引unique index进行的查询：

```text
EXPLAIN SELECT * FROM customer WHERE email ='MARY.SMITH@sakilacustomer.org';
```

![img](https://pic4.zhimg.com/80/v2-71e65de46caf22a78c7bc5cae2ce6e6f_1440w.jpg)

**7、type=NULL，MySQL不用访问表或者索引，直接就能够得到结果**

![img](https://pic3.zhimg.com/80/v2-9a358a3c7b052adee7d2e84d21187f56_1440w.png)

**.possible_keys:** 表示查询可能使用的索引

**.key:** 实际使用的索引

**.key_len:** 使用索引字段的长度

**.ref:** 使用哪个列或常数与key一起从表中选择行。

**.rows:** 扫描行的数量

**.filtered:** 存储引擎返回的数据在server层过滤后，剩下多少满足查询的记录数量的比例(百分比)

**.Extra:** 执行情况的说明和描述，包含不适合在其他列中显示但是对执行计划非常重要的额外信息

最主要的有一下三种：

| Using Index           | 表示索引覆盖，不会回表查询                            |
| --------------------- | ----------------------------------------------------- |
| Using Where           | 表示进行了回表查询                                    |
| Using Index Condition | 表示进行了ICP优化                                     |
| Using Flesort         | 表示MySQL需额外排序操作, 不能通过索引顺序达到排序效果 |

