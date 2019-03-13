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
