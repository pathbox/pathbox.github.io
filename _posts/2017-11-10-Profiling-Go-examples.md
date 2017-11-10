---
layout: post
title: Profling Go Example
date:   2017-11-10 15:16:06
categories: Go
image: /assets/images/post.jpg
---

##### Tools Matrix

| name     | Pros    | Cons |
| -------- | ------- | ---- | ------ | ------------ |
| ReadMemStats     | Simple, quick and easy.  Only details memory usage. |  Requires code change   |
| pprof | Details CPU and Memory.Remote analysis possible.Image generation. |  Requires code change. More complicated API.|
|trace| Helps analyse data over time.Powerful debugging UI.Visualise problem area easily.|Requires code change. UI is complex. Takes time to understand. |


##### Basic Example

```go
package main

import (
  "log"
)

// bigBytes allocates 100 megfabytes
func bigBytes() *[]byte{
  s := make([]byte, 100000000)
  return &s
}

func main() {
  for i := 0; i < 10; i++{
    s := bigBytes()
    if s == nil {
      log.Println("oh noes")
    }
  }
}

```

##### ReadMemStats

```go
package main

import (
  "log"
  "runtime"
)

func bigBytes() *[]byte{
  s := make([]byte, 100000000)
  return &s
}

func main() {
  var mem runtime.MemStats

  log.Println("memory baseline...")

  runtime.ReadMemStats(&mem)
  log.Println(mem.Alloc)
  log.Println(mem.TotalAlloc)
  log.Println(mem.HeapAlloc)
  log.Println(mem.HeapSys)

  for i := 0; i < 10; i++ {
       s := bigBytes()
       if s == nil {
           log.Println("oh noes")
       }
   }

   log.Println("memory comparison...")

   runtime.ReadMemStats(&mem)
   log.Println(mem.Alloc)
   log.Println(mem.TotalAlloc)
   log.Println(mem.HeapAlloc)
   log.Println(mem.HeapSys)
}

```

result show:

```
2017/11/10 15:41:49 memory baseline...
2017/11/10 15:41:49 55792
2017/11/10 15:41:49 55792
2017/11/10 15:41:49 55792
2017/11/10 15:41:49 819200
2017/11/10 15:41:49 memory comparison...
2017/11/10 15:41:49 200070136
2017/11/10 15:41:49 1000139976
2017/11/10 15:41:49 200070136
2017/11/10 15:41:49 300810240

```

##### CPU Analysis

```go
package main

import (
	"log"
	"os"
	"runtime/pprof"
)

// bigBytes allocates 10 sets of 100 megabytes
func bigBytes() *[]byte {
	s := make([]byte, 100000000)
	return &s
}

func main() {
	pprof.StartCPUProfile(os.Stdout)
	defer pprof.StopCPUProfile()

	for i := 0; i < 10; i++ {
		s := bigBytes()
		if s == nil {
			log.Println("oh noes")
		}
	}
}
```
command:

```
go build -o app && time ./app > cpu.profile
go tool pprof cpu.profile
```

##### Memory Analysis

```go
package main

import (
	"log"
	"os"
	"runtime/pprof"
)

// bigBytes allocates 10 sets of 100 megabytes
func bigBytes() *[]byte {
	s := make([]byte, 100000000)
	return &s
}

func main() {
	for i := 0; i < 10; i++ {
		s := bigBytes()
		if s == nil {
			log.Println("oh noes")
		}
	}

	pprof.WriteHeapProfile(os.Stdout)
}
```

command:

```
go build -o app && time ./app > memory.profile
go tool pprof memory.profile
```

##### Remotely analyse via web server

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	_ "net/http/pprof"
	"sync"
)

// bigBytes allocates 10 sets of 100 megabytes
func bigBytes() *[]byte {
	s := make([]byte, 100000000)
	return &s
}

func main() {
	var wg sync.WaitGroup

	go func() {
		log.Println(http.ListenAndServe("localhost:6060", nil))
	}()

	for i := 0; i < 10; i++ {
		s := bigBytes()
		if s == nil {
			log.Println("oh noes")
		}
	}

	wg.Add(1)
	wg.Wait() // this is for the benefit of the pprof server analysis
}
```

command:

```
http://localhost:6060/debug/pprof/
```

```
profiles:
0	block
4	goroutine
5	heap
0	mutex
7	threadcreate

full goroutine stack dump
```

+ block: stack traces that led to blocking on synchronization primitives
+ goroutine: stack traces of all current goroutines
+ heap: a sampling of all heap allocations
+ mutex: stack traces of holders of contended mutexes
+ threadcreate: stack traces that led to the creation of new OS threads

```go
package main

import (
    "net/http"
    "net/http/pprof"
)

func message(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello World"))
}

func main() {
    r := http.NewServeMux()
    r.HandleFunc("/", message)

    r.HandleFunc("/debug/pprof/", pprof.Index)
    r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
    r.HandleFunc("/debug/pprof/profile", pprof.Profile)
    r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
    r.HandleFunc("/debug/pprof/trace", pprof.Trace)

    http.ListenAndServe(":8080", r)
}
```

command：

```
go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap
go tool pprof -inuse_space http://localhost:6060/debug/pprof/heap
```


##### Trace

```go
package main

import(
  "runtime/trace"
  "sync"
  "log"
)
func main() {
	trace.Start(os.Stdout)
	defer trace.Stop()

	for i := 0; i < 10; i++ {
		s := bigBytes()
		if s == nil {
			log.Println("oh noes")
		}
	}

	var wg sync.WaitGroup
	wg.Add(1)

	var result []byte
	go func() {
		result = make([]byte, 500000000)
		log.Println("done here")
		wg.Done()
	}()

	wg.Wait()
	log.Printf("%T", result)
}
```

command:

```
go build -o app
time ./app > app.trace
go tool trace app.trace
```

[Profiling Go](http://www.integralist.co.uk/posts/profiling-go/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
