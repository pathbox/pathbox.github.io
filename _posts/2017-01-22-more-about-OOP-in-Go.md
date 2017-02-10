---
layout: post
title: More about OOP in Go
date:   2017-01-22 15:39:06
categories: Go
image: /assets/images/post.jpg
---

面向对象程序设计（英语：Object-oriented programming，缩写：OOP）是种具有对象概念的程序编程范型，同时也是一种程序开发的方法。它可能包含数据、属性、代码与方法。对象则指的是类的实例。它将对象作为程序的基本单元，将程序和数据封装其中，以提高软件的重用性、灵活性和扩展性，对象里的程序可以访问及经常修改对象相关连的数据。Go语言也是一种面向对象编程的语言，但是和传统的编程语言有一些不同。但是Go也有很多传统OOP语言的特点：

+ Methods on any defined type
+ Polymorphism
+ Namespacing
+ Message Passing/Delegation

Go不支持Objects 和Classes，虽然Go没有明确的对象类型，但是Go有typs和methods来替代Objects 和Classes。Go提供的结构体就是把使用各种数据类型定义的不同变量组合起来的高级数据类型。然后我们可以为这些数据类型声明方法。一个方法就是一个包含了接受者的函数，接受者可以是命名类型或者结构体类型的一个值或者是一个指针。所有给定类型的方法属于该类型的方法集。这看起来和传统的OOP语言的习惯差不多。然而Go却是不支持继承的。在Go中也没有继承类型。Go通过组合复用方式来实现继承

##### 组合复用

```go
type Person struct {
  Name string
  jobTitle string
  ShoeSize float32
}

func (p Person) SayHello() {
  fmt.Println("Hello, I'm ", p.Name)
}

func (p Person) GetJobTitle() {
 fmt.Println(“I’m a “, p.JobTitle)
}

type Doctor struct {
  Person
  Degree string
}

func (d Doctor) SayHello() {
 fmt.Println(“Hello, I’m Dr.”, d.Name, “, “, d.Degree)
}
func (d Doctor) Diagnose() {
 fmt.Println(“It’s Lupus.”)
}
```

让我想起了Ruby Rails的Mix-in

##### 接口

Go 语言里有非常灵活的 接口 概念，通过它可以实现很多面向对象的特性。接口提供了一种方式来 说明 对象的行为：如果谁能搞定这件事，它就可以用在这儿。
接口定义了一组方法（方法集），但是这些方法不包含（实现）代码：它们没有被实现（它们是抽象的）。接口里也不能包含变量。

```go
type Namer interface {
    Method1(param_list) return_type
    Method2(param_list) return_type
    ...
}
```

接口实现多态:

```go
package main

import "fmt"
type Animal interface {
 Type() string
 Swim() string
 IsFurry() bool
}
type Dog struct {
 Name string
 Breed string
}
type Frog struct {
 Name string
 Color string
}

func main(){
  f := new(Frog)
  d := new(Dog)
  zoo := [...]Animal{f, d}
  for _, a := range zoo {
    fmt.Println(a.Type(), " can ", a.Swim())
  }
}

func (f *Frog) Type() string {
  return "Frog"
}
func (f *Frog) Swim() string {
  return "Kick"
}
func (d *Dog) Type() string {
  return "Dog"
}
func (d *Dog) Swim() string {
  return "Doggie"
}

```

总结:

+ type struct 要属于这个接口

+ type struct 要定义实现接口定义的方法代码

+ type struct 这样就能执行各自方法的代码逻辑
