---
layout: post
title:  从io.Writer进一步理解interface
date:   2018-02-15 10:00:06
categories: Golang
image: /assets/images/s1.jpeg
---

最近在阅读Golang源码时，有个地方没有看的很明白。认真思考了一下，本文作为记录总结。

看的源代码是

`net/http/internal/chunked.go`

```go

// Writing to chunkedWriter translates to writing in HTTP chunked Transfer
// Encoding wire format to the underlying Wire chunkedWriter.
type chunkedWriter struct {
	Wire io.Writer
}

// Write the contents of data as one chunk to Wire.
// NOTE: Note that the corresponding chunk-writing procedure in Conn.Write has
// a bug since it does not check for success of io.WriteString
func (cw *chunkedWriter) Write(data []byte) (n int, err error) {

	// Don't send 0-length data. It looks like EOF for chunked encoding.
	if len(data) == 0 {
		return 0, nil
	}

	if _, err = fmt.Fprintf(cw.Wire, "%x\r\n", len(data)); err != nil {
		return 0, err
	}
	if n, err = cw.Wire.Write(data); err != nil {  // 这一行...
		return
	}
	if n != len(data) {
		err = io.ErrShortWrite
		return
	}
	if _, err = io.WriteString(cw.Wire, "\r\n"); err != nil {
		return
	}
	if bw, ok := cw.Wire.(*FlushAfterChunkWriter); ok {
		err = bw.Flush()
	}
	return
}

```

chunkedWriter.Wire 是一个io.Writer， io.Writer是一个 interface.

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

卡住的点是， `cw.Wire.Write(data)` 的Write源代码在哪里呢? 我通过跳转并没有找到对应的代码位置。今天恰好看了goCN的推荐文章，连接在参考链接中。其中的代码实现是这样的：

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
)

type Person struct {
	Id   int
	Name string
	Age  int
}

func (p *Person) Write(w io.Writer) {
	b, _ := json.Marshal(*p)
	w.Write(b)
}

func main() {
	p := &Person{Id: 1, Name: "Joe", Age: 27}
	var b bytes.Buffer

	p.Write(&b)

	fmt.Println(b.String())

}

```

看完这个例子我一下就明白了， `cw.Wire` 是io.Writer interface类型，只要某个类型或struct实现了Write方法,就实现了这个interface.

回到 `chunkedWriter`, 初始化`chunkedWriter`的时候,将 `Wire`初始化为 bytes.buffer 一类的实现了io.Writer interface的类型,这样就能使用到他们定义的Write方法了.Golang源码中有大量的这种运用.现在对interface又有更深的理解.

参考连接
```
https://medium.com/@as27/a-simple-beginners-tutorial-to-io-writer-in-golang-2a13bfefea02
```
