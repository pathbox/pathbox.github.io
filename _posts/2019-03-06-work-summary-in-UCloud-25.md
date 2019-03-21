---
layout: post
title: 最近工作总结(25)
date:  2019-03-06 10:18:06
categories: Work
image: /assets/images/post.jpg
---

### sendFile in Kafka
Kafka 在数据写入及数据同步采用了零拷贝(zero-copy)技术，采用sendFile()函数调用，
sendFile() 函数是在两个文件描述符之间直接传递数据，完全在内核中操作，
从而`避免了内核缓冲区与用户缓冲区之间数据的拷贝`，操作效率极高

### Golang struct filed 用大写开头，避免遇坑

```go
type Person struct {
  name string `json:"name"`
  age int `json:"age"`
  city string `json:"city"`
}

person := Person{
  name: "Cary",
  age: 28,
  city: "Beijing"
}

data, err := json.Marshal(person)
if err != nil {
  panic(err)
}

fmt.Println(string(data))
```

输出的结果 却是 Person{}， 字段Field值没有序列化进去

原因就是 struct 的Field 是小写开头的，是非导出的

将Person 改为

```go
type Person struct {
  Name string `json:"name"`
  Age int `json:"age"`
  City string `json:"city"`
}
```

要用大写开头的Field

### etcd：从应用场景到实现原理的全方位解读

https://infoq.cn/article/etcd-interpretation-application-scenario-implement-principle

### 计算机科学只是一种工具

计算机科学只是一种工具：包括硬件、程序、编程语言等。

如果认为计算机科学是中心，是未来，是能解决所有问题，也许这是一种无知。

人工智能领域: `Math is the King, Probability Statistics is the queue, Computer Science is just the soldier`

好好掌握计算机科学这个工具，在工作中不断解决问题，帮助自己和他人解决问题，才是正确的方式

能解决问题和会写代码，就是计算机工程师和码农的根本区别

### Understand Worker Queue Job

```go
type Queue struct {
	Jobs    chan interface{} // 生产者角色job 需要处理的job数量
	done    chan bool
	workers chan chan int // 消费者角色worker  处理job的多线程能力， worker的目的是多线程处理 job
	mux     sync.Mutex
}
```

- Queue 是一个队列work 队列
  - Jobs 接收每个job需要的参数入队列, 对外界来说是Queue的入口, Queue和`生产者的连接层`
  - workers 工人，职蚁。 从Jobs不断取出job进行处理和消费，workers的数量,表示Queue起了多少线程(goroutinue)在并发处理Jobs队列,表示Queue的并发处理能力。每个worker应该是在不同的线程(goroutinue)中执行。worker 应该是最终的`消费者`

- 一个Queue就是消费者和生产者的模型

### 数学计算表达式利用程序实现的原理

一个数学计算表达式如何用程序实现(最简单的计算器):

将一个数学计算表达式(infix:中缀表达式) => 后缀表达式(postfix 逆波兰表达式) => 利用堆栈stack结构存储计算，如果是数字，不断的入栈，当遇到一个操作符时，不入栈而是取出最top的两个数字和这个操作符进行eval，得到新的数值入栈

```
sk = new stack
cur = head of expression list
while(cur != null) {
  if cur.element is a base expression number {
    s.push cur.element
  } else if cur.element is an operator {
    operand2 = s.pop()
    operand1 = s.pop()
    operator = cur.element
    s.push(evaluate(operand1, operator, operand2))
  }
  cur = cur.next
}
```

所以现在的关键是: infix => postfix

更简单的情况：

如果我们有一个表达式树，则可以直接利用后续遍历postorder traversal 进行计算

```
evalExpTree(root) {
  if root is a leaf
    return root.value
  else
    firstOperand = evelExpTree(root.leftChild)
    secondOperand = evelExpTree(root.rightChild)
    return evaluate(firstOperand, root, secondOperand)
}
```

### Shunting-yard algorithm
Shunting-yard algorithm 调度场算法将中序表达式转为后续表达式(逆波兰表达式)

### 计算机互联网软件Logo搜索网站

https://www.vectorlogo.zone/

### 应该努力做好一位"前人"
在工作中应该尽力做好一位"前人"，为后人栽树，而不是挖沟、挖坑

### 权限系统 RBAC、ABAC

RBAC: Role-Based Access Control。用户关联角色，角色关联权限

ABAC: Attribute-Based Access Control。属性通常来说分为四类：用户属性（如用户年龄），环境属性（如当前时间），操作属性（如读取）和对象属性（如一篇文章，又称资源属性）。“允许所有班主任在上课时间自由进出校门”这条规则，其中，“班主任”是用户的角色属性，“上课时间”是环境属性，“进出”是操作属性，而“校门”就是对象属性了。为了实现便捷的规则设置和规则判断执行，ABAC通常有配置文件（XML、YAML等）或DSL配合规则解析引擎使用。XACML（eXtensible Access Control Markup Language）是ABAC的一个实现

RBAC比ABAC流行，因为对于大多数系统，RBAC简单但足够使用，ABAC对于他们来说往往过于复杂

### 在互联网信息时代应该学习和受教的方向

在互联网信息时代，获取信息和知识很方便、很容易，而要把知识讲解清楚、理解清楚，用通俗易懂的方式解释，很难

### 分布式系统的主从架构

1. MySQL的主从架构，一个Master 对应多个Slave,这种架构在分布式系统中有很大的劣势，即:并发处理能力不强，是一种backup的模式

2. 主=>复制分片模式(Master => Replication)

将数据根据一定的分片原则(如hash、取模等),分成了多个分片，每组分片又分为主分片和多个复制分片。主分片主要负责写入(也可以读取)，复制分片从主分片中同步数据，负责读取操作。这样同时提高了写和读的并发性能，当主分片挂了，复制分片能够升级为主分片。

3. 主服务器只有一台，掌控全局；从服务器很多台，负责具体的事情。
- Resource Node: 掌控全局,存储这个集群系统的元信息、状态信息、分片信息，进行资源调度，相当于集群的大脑
- Data Node: 负责具体的事情。

4. Leader => Follower

每个node保存的数据和信息是一致的

进行写操作前，会进行选举出Leader Node，由Leader Node进行写数据操作，再同步给Follower Node，Follower Node 可以负责读操作

Leader Node挂了，选举的时候就会从其他Follower中选出新的Leader Node

### 在测试环境，没有把线上环境所具有的条件情况都测过，拒绝上线
测试环境和线上环境有几个不同也是坑

### 生产环境的服务器不能访问外网，测试环境却可以，就是一个坑
有些生产环境的服务器由于安全性原因，不能访问外网，会有很多不便的问题，这样第三方外网服务API就无法直接使用

在使用这种第三方外网服务的API的时候，也要思考服务器是否能访问外网

现在的坑是，测试环境是可以访问的，生产环境无法访问外网，结果导致测试测过了，以为通过了，到发布生产的时候出了问题

### K8s Docker 时区问题解决tips

先看看没有设置前，容器的情况

```
docker run -it --rm centos
date
cat /etc/localtime
```

1. 通过环境变量的方式来改变容器的时区
```
docker run -it --rm -e "TZ=Asia/Shanghai" centos
date
cat /etc/localtime
```

2. 挂载主机的时区文件到容器中
```
docker run -it --rm -v /etc/localtime:/etc/localtime  centos
date
cat /etc/localtime
```

3. Kubernetes的时区设置方式,直接设置env环境变量
```yml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env-tz
spec:
  containers:
  - name: ngx
    image: nginx:latest
    imagePullPolicy: IfNotPresent
    env:
      - name: TZ
        value: Asia/Shanghai
```

4. 通过挂载主机时区文件设置
```yml
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-tz
spec:
  containers:
  - name: ngx
    image: nginx:latest
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: tz-config
      mountPath: /etc/localtime
      readOnly: true
  volumes:
  - name: tz-config
    hostPath:
      path: /etc/localtime
```

5. 通过Pod Preset预设置时区环境变量
激活Pod Preset

编辑/etc/kubernetes/manifests/kube-apiserver.yaml，

- 在-runtime-config增加settings.k8s.io/v1alpha1=true
- 在--admission-control增加PodPreset`

```yml
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-tz-env
spec:
  selector:
    matchLabels:
  env:
    - name: TZ
      value: Asia/Shanghai
```
一定需要写selector...matchLabels，但是matchLabels为空，标示应用于所有容器，这个正式我们所期望的
