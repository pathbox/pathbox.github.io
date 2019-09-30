---
layout: post
title: 最近工作总结(31)
date:  2019-09-02 19:55:06
categories: Work
image: /assets/images/post.jpg
---

### 只需五步，自己动手写一个静态博客
1. 收集markdown列表
2. 解析markdown源
3. 生成博客文章
4. 生成博客首页索引
5. 开始编译
http://muxueqz.top/a-small-static-site-generator.html
https://blog.thea.codes/a-small-static-site-generator/

### 领导者的三种模式

模式一："这就是我想要的，你按照我说的做。"

模式二："这就是我想要的，你自己想如何去做。"

模式三："让我们一起弄清楚我们能做些什么

### 单元测试如此重要,让你的代码更有信心

### 账号系统中，优惠券，赠金必然会存在媷羊毛漏洞问题

### Golang项目中使用db(将db设置为全局变量模式)

1. 将db设置为全局变量模式，整个项目只有一个db实例
2. 如果项目需要连接多个不同的数据库源，可以用map存储，一次初始化好所有db实例，放到全局的map中保存
3. 如果每个请求过来都新建一个connect db实例，这样浪费资源，如果并发稍微大一点，则会创建非常多的db实例连接，导致db服务器连接被耗尽而出现问题。其实我们希望的是，只有一个实例，最多使用该实例设置的连接池的连接数，这样才不会导致将db服务的连接耗尽的情况

### Golang map 内存的释放
Golang map delete 操作不会释放占用内存,会等到GC的时候,由GC选择释放。想要立即释放内存，需要将map设置为nil，即可立即释放所占内存

### mock in test
mock 在测试中的作用是解除外部依赖(数据库，文件依赖，第三方API调用)等，而不是为了测试依赖


### Golang slice map不是线程安全
Golang slice map 不是线程安全，如果定义了全局的slice和map，在读写slice和map数据时，请使用锁

### 四个抽象

- 文件是对I/O设备的抽象
- 虚拟内存是对程序存储器的抽象
- 进程是对一个正在运行的程序的抽象
- 虚拟机,它提供整个计算机的抽象，包括操作系统、处理器和程序

### Go 1.13 GO module 相关设置
在Go 1.13中，我们可以通过GOPROXY来控制代理，以及通过GOPRIVATE控制私有库不走代理。

设置GOPROXY代理：

go env -w GOPROXY=https://goproxy.cn,direct
设置GOPRIVATE来跳过私有库，比如常用的Gitlab或Gitee，中间使用逗号分隔：

go env -w GOPRIVATE=*.gitlab.com,*.gitee.com

如果在运行go mod vendor时，提示Get https://sum.golang.org/lookup/xxxxxx: dial tcp 216.58.200.49:443: i/o timeout，或者go mod init时会报13设置了默认的GOSUMDB的错误，则是因为Go 1.13设置了默认的GOSUMDB=sum.golang.org，这个网站是被墙了的，用于验证包的有效性，可以通过如下命令关闭：

go env -w GOSUMDB=off

### Binlog 的三种模式

1. Statement Level模式
每一条修改数据的sql都会记录到master的bin_log中，slave在复制的时候sql进程会解析成master端执行过的相同的sql在slave库上再次执行。

优点：statement level下的优点首先就是解决了row level下的缺点，不需要记录每一行的变化，较少bin-log日志量，节约IO，提高性能。因为它只需要记录在master上所执行的语句的细节，以及执行语句时候的上下文信息。

缺点：由于它是记录执行语句，所以，为了让这些语句在slave端也能正确执行，那么它还必须记录每条语句在执行的时候的一些相关信息，也就是上下文信息，来保证所有语句在slave端能够得到和在master端相同的执行结果。由于mysql更新较快，使mysql的赋值遇到了不小的挑战，自然赋值的时候就会涉及到越复杂的内容，bug也就容易出现。在statement level下，目前就已经发现了不少情况会造成mysql的复制出现问题，主要是修改数据的时候使用了某些特定的函数或者功能的时候会出现。比如：sleep（）函数在有些版本中就不能正确赋值，在存储过程中使用了last_insert_id（）函数，可能会使slave和master上得到不一致的id等等。由于row level是基于每一行记录的裱花，所以不会出现类似的问题
1、解决了row level的缺点，不需要记录每一行的变化。
2、日志量少，节约IO，从库应用日志块。
Statement level缺点：一些新功能同步可能会有障碍,比如函数、触发器等

2. Row Level模式
日志中会记录成每一行数据修改的形式，然后在slave端再对相同的数据进行修改。

优点：在row level的模式下，bin_log中可以不记录执行的sql语句的上下文信息，仅仅只需要记录哪一条记录被修改，修改成什么样。所以row level的日志内容会非常清楚的记录每一行数据修改的细节，非常容易理解。而且不会出现某些特定情况下的存储过程，或fuction，以及trigger的调用或处罚无法被正确复制的问题。

缺点：row level模式下，所有的执行语句都会记录到日志中，同时都会以每行记录修改的来记录，这样可能会产生大量的日志内容

1、记录详细，解析简单。是5.7之后推荐的方式
2、解决statement level模式无法解决的复制问题。
row level的缺点：日志量大，因为是按行来拆分

3. Mixed模式（混合模式）
前两种模式的结合，在mixed模式下，mysql会根据执行的每一条具体的sql语句来区分对待记录的日志形式，也是在statement和row之间选择一种

### 修改Binlog模式
```
mysql> show variables like "%binlog_format%";

+---------------+-----------+

| Variable_name | Value     |

+---------------+-----------+

| binlog_format | STATEMENT |

+---------------+-----------+

1 row in set (0.00 sec) 

方法一：在线修改立即生效

mysql> set global binlog_format='MIXED';

Query OK, 0 rows affected (0.00 sec)

退出mysql，查看当前mysql日志模式

mysql> show variables like "%binlog_format%";

+---------------+-------+

| Variable_name | Value |

+---------------+-------+

| binlog_format | MIXED |

+---------------+-------+

1 row in set (0.00 sec)

方法二：在配置文件中参数如下：

[mysqld]

log-bin=/var/lib/mysql/mysql-bin

#binlog_format="ROW"

binlog_format="MIXED"   #开启MIXED模式

#binlog_format="STATEMENT"

修改后重启mysql服务日志模式：

mysql> show variables like "%binlog_format%";

+---------------+-------+

| Variable_name | Value |

+---------------+-------+

| binlog_format | MIXED |

+---------------+-------+

1 row in set (0.00 sec)

三、在日志模式当前为row的模式下，记录日志的形式内容。

mysql> show variables like "%binlog_format%";

+---------------+-------+

| Variable_name | Value |

+---------------+-------+

| binlog_format | ROW   |

+---------------+-------+
1 row in set (0.00 sec)

```
