---
layout: post
title: 最近工作总结(50)
date:  2021-07-16 21:00:00
categories: Work
image: /assets/images/post.jpg


---

​    

### 使用hash大幅度提高Redis value内存利用率

如果把要使用的 redis 数据都集中到一起，集中存放，则 value 的大小会远大于 key 和其他内存结构的大小，从而使内存利用率达到 50%~99%。然而此方案也有弊端：如果只想取某个子模块的数据也必须把整体数据都拉下来，无状态化的情况下本来就会频繁读写数据，此方案将显著增加 redis 的CPU压力。
redis 的 hash 类型既可以把数据集中存放，也支持 key 分开读写

### 返回结构体 还是结构体指针

> 1MiB字节以下，返回结构体都更有优势。那返回指针的方式是不是没用了呢？也不是，如果你最终的结构体，就是要存放到堆里，比如要存放到全局的map里，那返回指针优势就更大些，因为其省去了返回结构体时的拷贝操作

返回结构体指针性能会较差的原因是：结构体指针会分配在堆上，分配堆的函数比分配在栈上更复杂，所以更耗时。分配在堆上需要GC来回收

### rz 命令上传文件到跳板机
- 登入跳板机
- 输入 rz命令
- 弹出选择文件框，选择要上传的文件
- 极速上传ing
- It is so COOL!

### 旧版的MySQL字段是字符串类型，传入整数，不会自动转换，能得到数据，但索引会失效，是全表扫描

### 使用redis连接池处理链接是应对高并发的有效方式

当QPS很高的时候，比如10w QPS，如果不使用连接池，会导致大量的短连接请求，对于http请求，会有大量的三次握手和四次挥手，由于在挥手的时候，tcp有time wait 机制，会在1min-2min(根据系统而定)才会完全释放端口给下一个请求使用，所以在time wait时间，会导致端口耗尽，没有课使用的端口，使得短连接请求失败。
解决方法之一，就是加机器，比如加到100台甚至更多，这样能够有足够的端口使用，但这种方式会造成浪费CPU和内存的情况
第二种方式：使用连接池+合适的机器数量，让资源充分合理的使用

### Linux的时间

内核有多种时间：
- RTC 精度低，精确到毫秒。
- wall time(xtime) 记录UTC 1970年1月24日到当前时刻所经历的纳秒，大部分时间函数或命令是从这里获取
- monotonic time 单调递增，不会累加系统休眠时间，受到ntp adjtimex影响
- raw monotonic time 不受到ntp影响
- boot time

### PHP & 取值符号，会升级临时变量的作用域
PHP & 取值符号，会升级临时变量的作用域，临时变量变为方法中的全局变量，如果之后的代码中有用相同的变量名称，是操作相同的变量地址

### 避免时间千年虫发方式
使用时间戳做时间的比较

### raft的详细中文论文翻译 
https://github.com/OneSizeFitsQuorum/raft-thesis-zh_cn/blob/master/raft-thesis-zh_cn.md

### 前端与后端
```
前端的问题不是难，而是它面对最终用户。只要用户的喜好和口味发生变化，前端就必须跟上。这导致前端不得不快速变化，因为用户的口味正在越来越快地改变。

后端不需要面对最终用户，需要解决的都是一些经典的计算科学问题，比如算法和数据结构。这些问题很少变化，可以利用以前的研究成果，所以变化速度慢得多。

前端的特征是混乱、嘈杂、易变，因为这些都是最终用户的特征，前端需要匹配用户。如果你不适应混乱、嘈杂、易变的开发，你就很难适应前端。
后端涉及到计算科学、语音设计、编译原理等高深内容，想要搞懂这些东西，绝非易事。
```

### 封装DAO层进行数据操作，避免在业务逻辑中写SQL
这样也能方便mock测试

### 简单的概率抽奖算法PHP

```php
/**
    * 概率抽奖算法
    // TODO 测试50%概率
    // $proArr = [
    //     1 => 5000,
    //     2 => 5000,
    // ];
    */
    function run_get_rand($proArr) 
    {
        $prize = '';
        $proSum = array_sum($proArr);
        foreach($proArr as $key => $proCur) {
            $randNum = mt_rand(1, $proSum);
            if($randNum <= $proCur) {
                $prize = $key;
                break;
            } else {
                $proSum -= $proCur;
            }
        }
        return $prize;
    }
```

```php
$a = 0;
$b = 0;
$p = [
  1 => 30,
  2 => 60
];

for($i=1;$i<=1000;$i++){
    if (run_get_rand($p)==1){
            $a++;
    } else {
            $b++;
    }
}

echo $a; // 得到的$a的值大概是300-350之间，占1000的三分之一左右


function run_get_rand($proArr)
    {
        $prize = '';
        $proSum = array_sum($proArr);
        foreach($proArr as $key => $proCur) {
            $randNum = mt_rand(1, $proSum);
            if($randNum <= $proCur) {
                $prize = $key;
                break;
            } else {
                $proSum -= $proCur;
            }
        }
        return $prize;
    }
```

### InnoDB 的 MVCC 是如何实现的
InnoDB 是如何存储记录多个版本的？这些数据是 事务版本号，行记录中的隐藏列和Undo Log。

事务版本号
每开启一个日志，都会从数据库中获得一个事务ID（也称为事务版本号），这个事务 ID 是自增的，通过 ID 大小，可以判断事务的时间顺序。

行记录的隐藏列
row_id :隐藏的行 ID ,用来生成默认的聚集索引。如果创建数据表时没指定聚集索引，这时 InnoDB 就会用这个隐藏 ID 来创建聚集索引。采用聚集索引的方式可以提升数据的查找效率。
trx_id: 操作这个数据事务 ID ，也就是最后一个对数据插入或者更新的事务 ID 。
roll_ptr:回滚指针，指向这个记录的 Undo Log 信息。

Undo Log 事务前的备份记录，用于事务回滚
InnoDB 将行记录快照保存在 Undo Log 里。

数据行通过快照记录都通过链表的结构的串联了起来，每个快照都保存了 trx_id 事务ID，如果要找到历史快照，就可以通过遍历回滚指针的方式进行查找。

Read View 是啥？
如果一个事务要查询行记录，需要读取哪个版本的行记录呢？ Read View 就是来解决这个问题的。Read View 可以帮助我们解决可见性问题。 Read View 保存了当前事务开启时所有活跃的事务列表。换个角度，可以理解为: Read View 保存了不应该让这个事务看到的其他事务 ID 列表。

trx_ids 系统当前正在活跃的事务ID集合。
low_limit_id ,活跃事务的最大的事务 ID。
up_limit_id 活跃的事务中最小的事务 ID。
creator_trx_id，创建这个 ReadView 的事务ID。
ReadView

如果当前事务的 creator_trx_id 想要读取某个行记录，这个行记录ID 的trx_id ，这样会有以下的情况：

如果 trx_id < 活跃的最小事务ID（up_limit_id）,也就是说这个行记录在这些活跃的事务创建前就已经提交了，那么这个行记录对当前事务是可见的。
如果trx_id > 活跃的最大事务ID（low_limit_id），这个说明行记录在这些活跃的事务之后才创建，说明这个行记录对当前事务是不可见的。
如果 up_limit_id < trx_id <low_limit_id,说明该记录需要在 trx_ids 集合中，可能还处于活跃状态，因此我们需要在 trx_ids 集合中遍历 ，如果trx_id 存在于 trx_ids 集合中，证明这个事务 trx_id 还处于活跃状态，不可见，否则 ，trx_id 不存在于 trx_ids 集合中，说明事务trx_id 已经提交了，这行记录是可见的。
如何查询一条记录
获取事务自己的版本号，即 事务ID
获取 Read View
查询得到的数据，然后 Read View 中的事务版本号进行比较。
如果不符合 ReadView 规则， 那么就需要 UndoLog 中历史快照；
最后返回符合规则的数据
InnoDB 实现多版本控制 （MVCC）是通过 ReadView+ UndoLog 实现的，UndoLog 保存了历史快照，ReadView 规则帮助判断当前版本的数据是否可见。

总结
如果事务隔离级别是 ReadCommit ，一个事务的每一次 Select 都会去查一次ReadView ，每次查询的Read View 不同，就可能会造成不可重复读或者幻读的情况。
如果事务的隔离级别是可重读，为了避免不可重读读，一个事务只在第一次 Select 的时候会获取一次Read View ，然后后面索引的Select 会复用这个 ReadView.

https://zhuanlan.zhihu.com/p/147372839