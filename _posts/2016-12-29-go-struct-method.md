---
layout: post
title: go方法
date:   2016-12-29 14:57:06
categories: Go
image: /assets/images/post.jpg
---

##### methods on value or pointer

```go
func (s *MyStruct) pointerMethod(){} // method on pointer
func (s MyStruct) valueMethod(){} // method on value
```

##### 想起曾经的值传递和地址传递

在C语言程序中，有值传递和地址传递的概念。

```go
func (s *MyStruct) pointerMethod() ⇒ pointerMethod(s *MyStruct)
func func (s MyStruct) valueMethod() ⇒ valueMethod(s MyStruct)
```
method on pointer 实际上传入的是类型值的指针，传入指针就不需要拷贝这个类型值，
并且通过在函数内对该类型值的修改是影响原值或外部变量的。为什么呢？因为是指针(请脑补指针的概念知识)

method on value 传入的是类型值的拷贝，如果对象比较大，拷贝会比较耗时，在函数中对该拷贝值所做的改变也不会影响到原值或外部变量。

```go
var p *Mystruct
p.pointerMethod() //实际上是 pointerMethod(p)
p.valueMethod() // 实际上是 valueMethod(*p)
var p Mystruct
p.pointerMethod() //实际上是 pointerMethod(&p)
p.valueMethod() // 实际上是 valueMethod(p)
```

##### 如何选择

1.如果方法改变调用者的话，那么就用 method on pointer

2.类型比较大，复制一份该类型的值耗时，那么使用 method on pointer

3.如果有的方法必须是 method on pointer 那么该类型所有的方法都应该是 method on pointer

4.如果是比较小的类型或者 基础类型, 分片等使用method on value会比较清晰。

5.如果你不知道该用什么，那么就用method on pointer

##### 在项目中使用的例子

我在model层定义了一个method on pointer 的方法

```go
type App struct {
	Name      string `gorm:"size:255"`
}
func (app *App) WhereName(name string) bool {
	result := db.Where("name = ?", name).First(&app).RecordNotFound()
	return result
}
```

在handler中，则不用暴露db，而直接导入model相关代码，直接定义app struct之后，使用该封装的方法。

```go
app ：= App{}
if app.WhereName(name) {
  log.Println("App not found")
} else {
  log.Println(app.Name)
}
```

##### method set
看下这段代码

```go
package main
type MyStruct struct {
}
func (s *MyStruct) pointerMethod() {
}
func (s MyStruct) valueMethod() {
}
type IMyStruct interface {
    pointerMethod()
    valueMethod()
}
func main() {
    p := &MyStruct{}
    v := MyStruct{}
    var i IMyStruct
    i = p //没有问题
    i = v  //error
       //cannot use v (type MyStruct) as type IMyStruct in assignment:
    // MyStruct does not implement IMyStruct (pointerMethod method has pointer receiver)
}
```
为什么把一个MyStruct值的指针可以赋值给 IMyStruct接口，而一个MyStruct的值赋值给IMyStruct接口就会报错？

```go
var p Mystruct
p.pointerMethod() //实际上是 pointerMethod(&p)
p.valueMethod() // 实际上是 valueMethod(p)
```

原因：
问题的关键在于 var p Mystruct 定义的这个变量是addressable的，所以能够调用method on pointer类型的函数
而将该值赋值给一个interface后，interface会拷贝一份该值，拷贝的值不是addressable的，也就不能再调用pointerMethod了
除了interface中的值不是addressable的map中的值也是不能addressable的

```
Method Sets: A type may have a method set associated with it.
The method set of an interface type is its interface.
The method set of any other named type T consists of all methods
with receiver type T. The method set of the corresponding
pointer type *T is the set of all methods with receiver *T or T
(that is, it also contains the method set of T). Any other type
has an empty method set. In a method set, each method must have
a unique name.

Calls: A method call x.m() is valid if the method set of (the
type of) x contains m and the argument list can be assigned to
the parameter list of m. If x is addressable and &x’s method set
contains m, x.m() is shorthand for (&x).m().
```

##### 嵌套类型
go语言没有继承的概念，但是go语言中的一个结构体可以通过匿名嵌套另外一个结构体来获得该结构体的函数

```go
type Human struct {
}
func (h Human) Age() {
}
func (h *Human) Name() {
}
type Student  struct {
    Human
}
/*
Human 的method set为 (Age)
*Human 的method set 为(Age Name)
Student的method set (Age)
Student *的method set 为(Age Name)
*/
type Student  struct {
    *Human
}
/*
Student的method set (Age Name)
Student *的method set 为(Age Name)
*/
//总结如下
type C struct {
    T
    *S
}
类型C的Method Set = T的Method Set + *S的Method Set
类型*C的Method Set = *T的Method Set + *S的Method Set
```
参考文献

[FAQ](https://golang.org/doc/faq#methods_on_values_or_pointers)

[golang类型方法](http://shanks.leanote.com/post/Untitled-55ca439338f41148cd000759-17)

[MethodSets](https://github.com/golang/go/wiki/MethodSets)
