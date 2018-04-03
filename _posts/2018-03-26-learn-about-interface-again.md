---
layout: post
title:  Learn about interface in Go again
date:   2018-03-26 20:30:06
categories: Golang
image: /assets/images/post.jpg
---

##### 接口组合

最普遍的例子之一就是Reader和Writer了

```go
type Reader interface{
  Read()
}

type Writer interface{
  Write()
}

type ReadWriter interface{
  Reader
  Writer
}
```

```go
package main

import "fmt"

type Reader interface{
  Read() string
}

type Writer interface{
  Write(name string)
}

type ReadWriter interface{
  Reader
  Writer
}

type RWshow struct {
  Name string
}

func (rw *RWshow) Read() string {
  return rw.Name
}

func (rw *RWshow) Write(name string) {
  rw.Name = name
}

func main() {
  var rw ReadWriter = &RWshow{Name: "Joe"}
  var r Reader = rw
  var w Writer = rw
  fmt.Println(rw, r, w)
}

```

+ 对于要实现 `ReadWriter`接口，就是要去分别实现`Reader`和`Writer`接口，也就是想要实现的struct，定义`Read()`和`Write()` 方法，在参数和返回值也要一致。

+ rw 是`ReadWriter`接口类型，那么，rw同时也是`Reader`和`Writer`接口类型。这个非常好理解，因为rw实现了`Read()`和`Write()`方法，自然也就实现了`Reader`和`Writer`接口类型。

##### 开放interface, 隐藏具体实现的struct属性和方法

```go
package socketio

import (
	"fmt"
	"net/http"
	"sync"
)

// Socket is the socket object of socket.io.
type Socket interface {
	Id() string
	Mark() string
	Namespace() string
	SetNamespace(nsp string) string
	Rooms() []string
	Request() *http.Request
	On(event string, f interface{}) error
	Emit(event string, args ...interface{}) error
	Close()
	Join(room string) error
	Leave(room string) error
	BroadcastTo(room, event string, args ...interface{}) error

	GetUserID() string
	InitUserID(userID string)
}

type socket struct {
	*socketHandler
	conn      engineio.Conn
	namespace string
	id        int
	mu        sync.Mutex
	ConnNum   int
	UserID    string
}
func newSocket(conn engineio.Conn, base *baseHandler) *socket {
	ret := &socket{
		conn: conn,
	}
	ret.socketHandler = newSocketHandler(ret, base)
	return ret
}
func (s *socket) Id() string {
	return s.conn.Id()
}
func (s *socket) Mark() string {
	return fmt.Sprintf("%s-%d", s.Id(), s.ConnNum)
}
func (s *socket) Namespace() string {
	return s.namespace
}
func (s *socket) SetNamespace(nsp string) string {
	s.namespace = nsp
	return s.namespace
}
func (s *socket) Request() *http.Request {
	return s.conn.Request()
}
func (s *socket) Emit(event string, args ...interface{}) error {
	if err := s.socketHandler.Emit(event, args...); err != nil {
		return err
	}
	if event == "disconnect" {
		s.conn.Close()
	}
	return nil
}
func (s *socket) Close() {
	s.conn.Close()
}
func (s *socket) GetUserID() string {
	return s.UserID
}
func (s *socket) InitUserID(userID string) {
	s.UserID = userID
}
func (s *socket) send(args []interface{}) error {
	//...
}
func (s *socket) sendConnect() error {
	//...
}
func (s *socket) sendId(args []interface{}) (int, error) {
	//...
}
func (s *socket) loop() error {
//...	 
}
```
上面的代码来自`go-socket.io`库。代码真正的实现是使用 socket struct，而开放出去给外部使用的是`Socket interface`， 这样，在外部只能使用`Socket interface`的方法，而不能使用socket的方法，这样很有效的保护了socket。最外层只能看到接口层，使用接口层的方法，避免使用更深的方法，特别是写方法，在外部改变内部socket的属性值。

我刚好需要一个需求，希望给socket增加UserID的属性，并且能在外部通过`Socket`包操作这个属性的读写。我在给socket 增加完这个属性后，在Socket接口上声明两个方法：

```go
GetUserID() string
InitUserID(userID string)
```

然后在内部为 `socket struct` 实现这两个方法，这样就能在外部通过Socket包调用接口，来操作内部`socket` UserID 这个属性了。

通过这个实践和阅读这段源码，我知道今后在代码设计中，通过设计合理的开放interface，将需要的方法暴露给外部，除此之外，外部不需要关心或入侵内部的具体实现方法逻辑或struct的属性。这样能够很好的保护内部的实现。

用interface时,可以将这个interface赋值为nil,然后在判断的时候,就可以直接判断是否nil,这是具体struct无法做到的

```go
package main

import "fmt"

type Man interface {
	MyName() string
}

type Boy struct {
	Name string
}

func (b *Boy) MyName() string {
	return b.Name
}

func main() {
	var m Man
	m = NewMan()

	if m == nil {
		fmt.Println("No Man")
	} else {
		fmt.Println(m.MyName())
	}
}

func NewMan() Man {
	return nil
  // return &Boy{Name: "Joe"}
}
```

参考链接：

```
https://www.ardanlabs.com/blog/2018/03/interface-values-are-valueless.html
```
