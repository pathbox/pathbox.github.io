---
layout: post
title: Write Cool Codes In Golang Project
date:   2018-09-29 14:13:06
categories: Golang
image: /assets/images/post.jpg
---

### It is happy to use sync.RWMutex with map

```go
var ConnectionPool = struct {
  sync.RWMutex
  pool map[string]Conn
}{pool: make(map[string]Conn{})}
```

### Regard func() as the object, regard interface as the object

what is the `func()`? It is a type, regard it as the object.

what is the `func()`? It is a object

```go

package task

type TaskHandler interface{
  ProcessAction(params map[string]interface{}) (result interface{})
}

type SQLTaskFunc func(params map[string]interface{}) (result interface{})

func (fn SQLTaskFunc) ProcessAction(params map[string]interface{}){
  return fn(params)
}

var HandlerPool = struct {
  sync.RWMutex
  pool map[string]Conn
}{pool: make(map[string]TaskHandler{})}

func RegisterTaskHandle(key string, handle TaskHandler) {
  HandlerPool.Lock()
  defer HandlerPool.Unlock()
  HandlerPool[key] = TaskHandler
}

package action

import "task"

func init() {
  params := map[string]string{
    "name": "foo"
  }
  task.RegisterTaskHandle("key", task.SQLTaskFunc(updateFunc(params)))
}

// updateFunc is the same type as SQLTaskFunc(相同的传入参数和返回值，所以他们是同种类型type的func)
func updateFunc(params map[string]interface{}) interface{} {
  // do something with params
  return "Well Done"
}

// you can wrapFunc, it is a good way
func wrapFunc(f func() error) func() {
	return func() {
		f()
	}
}
```
