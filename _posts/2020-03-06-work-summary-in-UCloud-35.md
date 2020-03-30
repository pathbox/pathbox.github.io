---
layout: post
title: 最近工作总结(35)
date:  2020-03-06 16:25:06
categories: Work
image: /assets/images/post.jpg
---

### Casbin API 简单了解

```go
//全局变量 e是执行者实例
e := NewEnforcer("examples/rbac_model.conf", "examples/rbac_policy.csv")

//获取当前策略中显示的主题列表
allSubjects := e.GetAllSubjects()

//获取当前命名策略中显示的主题列表
allNamedSubjects := e.GetAllNamedSubjects("p")

//获取当前策略中显示的对象列表
allObjects := e.GetAllObjects()

//获取当前命名策略中显示的对象列表
allNamedObjects := e.GetAllNamedObjects("p")

//获取当前策略中显示的操作列表
allActions := e.GetAllActions()

//获取当前命名策略中显示的操作列表
allNamedActions := e.GetAllNamedActions("p")

//获取当前策略中显示的角色列表
allRoles = e.GetAllRoles()

//获取当前命名策略中显示的角色列表
allNamedRoles := e.GetAllNamedRoles("g")

//获取策略中的所有授权规则
policy = e.GetPolicy()

//获取策略中的所有授权规则，可以指定字段筛选器
filteredPolicy := e.GetFilteredPolicy(0, "alice")

//获取命名策略中的所有授权规则
namedPolicy := e.GetNamedPolicy("p")

//获取命名策略中的所有授权规则，可以指定字段过滤器
filteredNamedPolicy = e.GetFilteredNamedPolicy("p", 0, "bob")

//获取策略中的所有角色继承规则
groupingPolicy := e.GetGroupingPolicy()

//获取策略中的所有角色继承规则，可以指定字段筛选器
filteredGroupingPolicy := e.GetFilteredGroupingPolicy(0, "alice")

//获取策略中的所有角色继承规则
namedGroupingPolicy := e.GetNamedGroupingPolicy("g")

//获取策略中的所有角色继承规则
namedGroupingPolicy := e.GetFilteredNamedGroupingPolicy("g", 0, "alice")

// 确定是否存在授权规则
hasPolicy := e.HasPolicy("data2_admin", "data2", "read")

//确定是否存在命名授权规则
hasNamedPolicy := e.HasNamedPolicy("p", "data2_admin", "data2", "read")

//向当前策略添加授权规则。 如果规则已经存在，函数返回false，并且不会添加规则。 否则，函数通过添加新规则并返回true
added := e.AddPolicy("eve", "data3", "read")

// 向当前命名策略添加授权规则。 如果规则已经存在，函数返回false，并且不会添加规则。 否则，函数通过添加新规则并返回true
added := e.AddNamedPolicy("p", "eve", "data3", "read")

// 从当前策略中删除授权规则
removed := e.RemovePolicy("alice", "data1", "read")

// 移除当前策略中的授权规则，可以指定字段筛选器。 RemovePolicy 从当前策略中删除授权规则
removed := e.RemoveFilteredPolicy(0, "alice", "data1", "read")

//从当前命名策略中删除授权规则
removed := e.RemoveNamedPolicy("p", "alice", "data1", "read")

//从当前命名策略中移除授权规则，可以指定字段筛选器
removed := e.RemoveFilteredNamedPolicy("p", 0, "alice", "data1", "read")

//确定是否存在角色继承规则
has := e.HasGroupingPolicy("alice", "data2_admin")

//确定是否存在命名角色继承规则
has := e.HasNamedGroupingPolicy("g", "alice", "data2_admin")

// 向当前策略添加角色继承规则。 如果规则已经存在，函数返回false，并且不会添加规则。 如果规则已经存在，函数返回false，并且不会添加规则
added := e.AddGroupingPolicy("group1", "data2_admin")

//将命名角色继承规则添加到当前策略。 如果规则已经存在，函数返回false，并且不会添加规则。 否则，函数通过添加新规则并返回true
added := e.AddNamedGroupingPolicy("g", "group1", "data2_admin")

// 从当前策略中删除角色继承规则
removed := e.RemoveGroupingPolicy("alice", "data2_admin")

//从当前策略中移除角色继承规则，可以指定字段筛选器
removed := e.RemoveFilteredGroupingPolicy(0, "alice")

//从当前命名策略中移除角色继承规则
removed := e.RemoveNamedGroupingPolicy("g", "alice")

//当前命名策略中移除角色继承规则，可以指定字段筛选器
removed := e.RemoveFilteredNamedGroupingPolicy("g", 0, "alice")

//添加自定义函数
func CustomFunction(key1 string, key2 string) bool {
    if key1 == "/alice_data2/myid/using/res_id" && key2 == "/alice_data/:resource" {
        return true
    } else if key1 == "/alice_data2/myid/using/res_id" && key2 == "/alice_data2/:id/using/:resId" {
        return true
    } else {
        return false
    }
}

func CustomFunctionWrapper(args ...interface{}) (interface{}, error) {
    key1 := args[0].(string)
    key2 := args[1].(string)

    return bool(CustomFunction(key1, key2)), nil
}

e.AddFunction("keyMatchCustom", CustomFunctionWrapper)
```
>https://blog.csdn.net/lk2684753/article/details/99680892?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task

### Rabbitmq的producer和consumer
producer产生message，将message 推送给rabbitmq，rabbitmq再将message推给consumer进行消费处理，那么producer和consumer之间是如何连接的呢？rabbitmq这里有几种选择方式，下面是简单描述常用的方式。

producer 与rabbitmq指定的exchange绑定，并且指定Type类型:
```go
producer = uamqp.NewExchange(getMQNodes(), uamqp.ExchangeOptions{
		Name:    "uaccount",
		Type:    "topic",
		Durable: true,
	}).CreateProducer()
```

producer 定义routingKey:
```go
routingKey := fmt.Sprintf("key_pattern.%s", name)
pubData := amqp.Publishing{
	ContentType:  "application/json",
	DeliveryMode: 2,
	Body:         body,
}

err := producer.Publish(routingKey, pubData, false, false)
```
这样，当前定义的producer会往`Exchange:uaccount`推送消息，并且定义的routing key 是key_pattern.xxx这样的模式。

consumer同样需要去绑定`Exchange:uaccount`,并且Type也是`topic`。
consumer定义队列的名称，并且consumer绑定的是routing key 是key_pattern.xxx，这样，consumer定义的队列就和producer绑定到一起了。

所以producer和consumer的连接关系是: exchange、type、routing key

### B树和B+树简记
B树：
1. 树内的每个节点都存储数据
2. 叶子节点之间无指针相邻
B+树：
1. 数据只出现在叶子节点
2. 所有叶子节点增加一个链指针

B树的树内存储数据，因此查询单条数据的时候，B树的查询效率不固定，最好的情况是O(1)。我们可以认为在做单一数据查询的时候，使用B树平均性能更好。但是，由于B树中各节点之间没有指针相邻，因此B树不适合做一些数据遍历操作。

(2)B+树的数据只出现在叶子节点上，因此在查询单条数据的时候，查询速度非常稳定。因此，在做单一数据的查询上，其平均性能并不如B树。但是，B+树的叶子节点上有指针进行相连，因此在做数据遍历的时候，只需要对叶子节点进行遍历即可，这个特性使得B+树非常适合做范围查询。

因此，我们可以做一个推论:没准是Mysql中数据遍历操作比较多，所以用B+树作为索引结构。而Mongodb是做单一查询比较多，数据遍历操作比较少，所以用B树作为索引结构。

那么为什么Mysql做数据遍历操作多？而Mongodb做数据遍历操作少呢？

因为Mysql是关系型数据库，而Mongodb是非关系型数据。

### SQL四种语言：DDL,DML,DCL,TCL

```
1.DDL（Data Definition Language）数据库定义语言statements are used to define the database structure or schema.

DDL是SQL语言的四大功能之一。
用于定义数据库的三级结构，包括外模式、概念模式、内模式及其相互之间的映像，定义数据的完整性、安全控制等约束
DDL不需要commit.
CREATE
ALTER
DROP
TRUNCATE
COMMENT
RENAME

2.DML（Data Manipulation Language）数据操纵语言statements are used for managing data within schema objects.

由DBMS提供，用于让用户或程序员使用，实现对数据库中数据的操作。
DML分成交互型DML和嵌入型DML两类。
依据语言的级别，DML又可分成过程性DML和非过程性DML两种。
需要commit.
SELECT
INSERT
UPDATE
DELETE
MERGE
CALL
EXPLAIN PLAN
LOCK TABLE

3.DCL（Data Control Language）数据库控制语言  授权，角色控制等
GRANT 授权
REVOKE 取消授权

4.TCL（Transaction Control Language）事务控制语言
SAVEPOINT 设置保存点
ROLLBACK  回滚
SET TRANSACTION

SQL主要分成四部分：
（1）数据定义。（SQL DDL）用于定义SQL模式、基本表、视图和索引的创建和撤消操作。
（2）数据操纵。（SQL DML）数据操纵分成数据查询和数据更新两类。数据更新又分成插入、删除、和修改三种操作。
（3）数据控制。包括对基本表和视图的授权，完整性规则的描述，事务控制等内容。
（4）嵌入式SQL的使用规定。涉及到SQL语句嵌入在宿主语言程序中使用的规则。
```

### 为什么要使用bitset
例如有N个需要存储，int resource=[1,2,4]
由于一个int 占用4个字节，那么resource数组占用12个字节。
如果使用bitset来储存long[0]=【0,1,1,0,1,...0】(总共64位,...全部用0填充)，原来一个整数占用内存是4个字节，现在用一个位来存储，比例32:1，简单的说用bitset按位存储可以省32倍的内存空间。
效率优势：1(M)=1X1024X1024x8=8388608(bit)，用1m的空间可以存储839万个整数，1G大约可以存86亿个整数

### MySQL Binlog 两种格式比较
##### Statement 优点
历史悠久，技术成熟；
产生的 binlog 文件较小；
binlog 中包含了所有数据库修改信息，可以据此来审核数据库的安全等情况；
binlog 可以用于实时的还原，而不仅仅用于复制；
主从版本可以不一样，从服务器版本可以比主服务器版本高；

##### Statement 缺点：
不是所有的 UPDATE 语句都能被复制，尤其是包含不确定操作的时候，DELETE 和 UPDATE 语句如果使用了 LIMIT 但是没有使用 ORDER BY，那么结果也是不确定的，也不能使用 Statement-Based 方式记录日志。
调用具有不确定因素的 UDF 时复制也可能出现问题；
运用以下函数的语句也不能被复制：
使用了下面方法的语句不能使用 Statement-Based 方式记录日志：
LOAD_FILE()
UUID(), UUID_SHORT()
USER()
FOUND_ROWS()
SYSDATE() (除非Master 和 Slave 在启动时都添加了 --sysdate-is-now 选项)
GET_LOCK()
IS_FREE_LOCK()
IS_USER_LOCK()
MASTER_POS_WAIT()
RAND()
RELEASE_LOCK()
SLEEP()
VERSION()
INSERT … SELECT 会产生比 RBR 更多的行级锁；
复制须要执行全表扫描 (WHERE 语句中没有运用到索引) 的 UPDATE 时，须要比 row 请求更多的行级锁；
对于有 AUTO_INCREMENT 字段的 InnoDB 表而言，INSERT 语句会阻塞其他 INSERT 语句；
对于一些复杂的语句，在从服务器上的耗资源情况会更严重，而 row 模式下，只会对那个发生变化的记录产生影响；
存储函数(不是存储流程 )在被调用的同时也会执行一次 NOW() 函数，这个可以说是坏事也可能是好事；
确定了的 UDF 也须要在从服务器上执行；
Master 和 Slave 上的表结构必须完全一致。数据表必须几乎和主服务器保持一致才行，否则可能会导致复制出错；
执行复杂语句如果出错的话，会消耗更多资源；

>Note:
更新数据库信息的语句，比如 GRANT,REVOKE 和对触发器、视图、存储程序（包括存储过程）的操作，都是使用 Statement-Based 方式写日志

##### Row 优点
任何情况都可以被复制，这对复制来说是最安全可靠的；
和其他大多数数据库系统的复制技能一样；
多数情况下，从服务器上的表如果有主键的话，复制就会快了很多；
复制以下几种语句时的行锁更少：
* INSERT … SELECT
* 包含 AUTO_INCREMENT 字段的 INSERT
* 没有附带条件或者并没有修改很多记录的 UPDATE 或 DELETE 语句
执行 INSERT，UPDATE，DELETE 语句时锁更少；
从服务器上采用多线程来执行复制成为可能；

##### Row 缺点
生成的 binlog 日志体积大了很多；
复杂的回滚时 binlog 中会包含大量的数据；
主服务器上执行 UPDATE 语句时，所有发生变化的记录都会写到 binlog 中，而 statement 只会写一次，这会导致频繁发生 binlog 的写并发请求；
UDF 产生的大 BLOB 值会导致复制变慢；
不能从 binlog 中看到都复制了写什么语句(加密过的)；
当在非事务表上执行一段堆积的 SQL 语句时，最好采用 statement 模式，否则很容易导致主从服务器的数据不一致情况发生；
另外，针对系统库 MySQL 里面的表发生变化时的处理准则如下：
如果是采用 INSERT，UPDATE，DELETE 直接操作表的情况，则日志格式根据 binlog_format 的设定而记录；
如果是采用 GRANT，REVOKE，SET PASSWORD 等管理语句来做的话，那么无论如何都要使用 statement 模式记录；
使用 statement 模式后，能处理很多原先出现的主键重复问题


### 关于递归代码的编写的思考总结！

关于递归代码的编写，以树的遍历为例子，为什么我一定要纠结代码是如何写的呢？实际上递归就是不断的入栈，达到返回条件，则出栈，返回到上一层，继续递归或返回。如果能够明白树是如何递归遍历的，纠结于思考代码是怎么递归的也许是一个死胡同。而从实际中明白起递归遍历的方式，才是更好的方式

### Kubernetes pod状态出现ImagePullBackOff的原因
kubectl describe pod pod-6bcbb6d5c7-bpmhs -nxxx

发现报错信息:
```
Events:
  Type     Reason          Age                    From                   Message
  ----     ------          ----                   ----                   -------
  Normal   Scheduled       3m41s                  default-scheduler      Successfully assigned prj-pod-xxx-pre-6bcbb6d5c7-bpmhs to 172.22.10.66
  Normal   Pulling         3m27s (x2 over 3m39s)  kubelet, 172.22.10.66  Pulling image "xxx.com/pod-xxx:pre_v0.0.37"
  Warning  Failed          3m27s (x2 over 3m39s)  kubelet, 172.22.10.66  Failed to pull image "xxx.com/pod-xxx:pre_v0.0.37": rpc error: code = Unknown desc = Error response from daemon: manifest for xxx.com/pod-xxx:pre_v0.0.37 not found
  Warning  Failed          3m27s (x2 over 3m39s)  kubelet, 172.22.10.66  Error: ErrImagePull
  Normal   SandboxChanged  3m20s (x7 over 3m38s)  kubelet, 172.22.10.66  Pod sandbox changed, it will be killed and re-created.
  Normal   BackOff         3m17s (x6 over 3m36s)  kubelet, 172.22.10.66  Back-off pulling image "xxx.com/pod-xxx:pre_v0.0.37"
  Warning  Failed          3m17s (x6 over 3m36s)  kubelet, 172.22.10.66  Error: ImagePullBackOff
```

image not found。可能是在build image的时候因为网络或别的原因，这个imagebuild失败了，或者是一些未知原因被删除释放了。
修复:重新tag，构造image，然后再部署过

### tcp KeepAlive
```sh
sysctl -a  |grep tcp_keepalive
net.ipv4.tcp_keepalive_time = 1200  #tcp建立链接后1200 秒如果无数据传输，则会发出探活数据包
net.ipv4.tcp_keepalive_probes = 9   #共发送9次
net.ipv4.tcp_keepalive_intvl = 75   #每次间隔75秒
```

KeepAlive并不是默认开启的，在Linux系统上没有一个全局的选项去开启TCP的KeepAlive。需要开启KeepAlive的应用必须在TCP的socket中单独开启

Nginx 开启tcp KeepAlive
```nginx
server {
    listen 127.0.0.1:3306 so_keepalive=on;
    proxy_pass 172.17.0.3:3306;

    #建立连接时间
    proxy_connect_timeout 5s;
    #保持连接时间
    proxy_timeout 3600s;
    error_log /data/logs/my.log info;
```

so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]

on: 开启，探测参数更加系统默认值
off: 关闭
keepidle: 等待时间
keepintvl: 发送探测报文间隔
keepcent: 探测报文发送次数
proxy_timeout：建连后无数据时与后端服务器连接保持时间，默认75s
proxy_connect_timeout：与后端服务器建立连接的超时, 不超过75s
proxy_send_timeout：向后端服务器组发出write请求后，等待响应的超时间，默认为60秒
proxy_read_timeout：向后端服务器组发出read请求后，等待响应的超时间，默认为60秒

### LVS为什么不反向代理
LVS为什么不反向代理？因为响应数据太多。影响性能。所以响应数据就直接到客户端
