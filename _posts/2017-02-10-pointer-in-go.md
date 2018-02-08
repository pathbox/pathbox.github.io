---
layout: post
title: Pointer in Go
date:   2017-02-10 10:58:06
categories: Golang
image: /assets/images/post.jpg
---


Go有指针类型含义和作用和C语言的指针是一样的，指向类型的地址，而且用途很广，但是你不能对指针进行运算。
比如C语言中的对指针的++ 运算或移位运算在Go中是没有的。

+ * 表示指针
+ & 表示取地址符

example.1 go

```go

package main

import (
	"fmt"
)

func val(ival int) {
	ival = 0
}

func ptr(iptr *int) {
	*iptr = 0
}

func main() {
	i := 1
	fmt.Println("initial:", i) // 1
	zeroval(i)
	fmt.Println("zeroval:", i) // 1

	zeroptr(&i)                // 参数为int指针，&i 就是传了这个变量参数的地址(指针指向的地址,也就是这个指针的值)
	fmt.Println("zeroptr:", i) // 0
	// Pointers can be printed too.
	fmt.Println("pointer:", &i) // pointer

}

```

example1.go 的例子展示了老生常谈的函数参数为值传递和地址传递区别。
值传递时，函数会copy一个参数值，函数中操作的是这个copy的参数值
地址传递，函数中操作的就是外部传递过来的这个参数，如果改变了，外部的这个参数也就被改变了

example2.go

```go

package main

import (
	"fmt"
)

type Test struct {
	Name string
}

func change2(t *Test) {
	t.Name = "2"
}

func change3(t *Test) {
	//注意这里括号
	//如果直接*t.Name=3 编译不通过 报错 invalid indirect of t.Name (type string)
	//其实在go里面*可以省掉，直接类似change2函数里这样使用。
	(*t).Name = "3"
}

func change4(t Test) {
	t.Name = "4"
}

func main() {
	// t 是一个地址
	t := &Test{Name: "1"}
	fmt.Println(t.Name)

	change2(t)
	fmt.Println(t.Name)

	change3(t)
	fmt.Println(t.Name)

	// 这里传递变量用了*
	change4(*t)
	fmt.Println(t.Name)
}

```

example3.go

```go

package main

import (
	"fmt"
)

type abc struct {
	v int
}

func (a abc) aaaa() { // 传入的是值，而不是引用
	a.v = 1
	fmt.Printf("1:%d\n", a.v)
}

func (a *abc) bbbb() { //传入的是引用，而不是值
	fmt.Printf("2:%d\n", a.v)
	a.v = 2
	fmt.Printf("3:%d\n", a.v)
}

func (a *abc) cccc() { //传入的是引用，而不是值
	a.v = 3
	fmt.Printf("4:%d\n", a.v)
}

func main() {
	// aobj := abc{} // new(abc)  aobj 是一个值
	aobj := new(abc)
	fmt.Println("aobj:", aobj)

	aobj.aaaa()
	fmt.Println("aobj:", aobj)

	aobj.bbbb()
	fmt.Println("aobj:", aobj)

	aobj.cccc()
	fmt.Println("aobj:", aobj)

}


```

对于struct 来说, call method时, 无论struct是值还是指针去call这个方法,都是可行的。
但是，如果方法的调用对象是值，比如func (a abc) aaaa()方法，则在方法中对a的修改，其实修改的是abc的一个copy值，
不会影响到外部的struct abc。 而bbbb()和cccc()方法调用对象是传的指针，所以，在方法中的修改，就是该struct abc。
会影响到外部的struct abc。

指针其实不难，关键是理解值传递和地址传递，以及指针指向的是类型的地址，指针存储的值就是其所指向的地址的值。
