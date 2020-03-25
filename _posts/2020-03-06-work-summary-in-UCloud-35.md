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
