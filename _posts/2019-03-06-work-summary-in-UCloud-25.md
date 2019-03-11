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
