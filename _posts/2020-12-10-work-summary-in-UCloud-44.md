---
layout: post
title: 最近工作总结(44)
date:  2020-12-10 20:00:00
categories: Work
image: /assets/images/post.jpg


---

 

### automaxprocs解决容器获取真实配置的CPU核心数量,解决golang调度G-M平衡

K8s使用时，pod的服务通常都对 CPU 资源做了限制，例如默认的 4C。但是在容器里通过 `lscpu` 仍然能看到宿主机的所有 CPU 核心。这就导致 golang 服务默认会拿宿主机的 CPU 核心数来调用 `runtime.GOMAXPROCS()`，导致 P 数量远远大于可用的 CPU 核心(M)，在高并发，产生大量goroutine时，引起频繁上下文切换，影响高负载情况下的服务性能。

automaxprocs

```go
package main

import (
    "fmt"
    _ "go.uber.org/automaxprocs"
    "runtime"
)

func main() {
    // Your application logic here.
    fmt.Println("real GOMAXPROCS", runtime.GOMAXPROCS(-1))
}
```



https://kingsamchen.github.io/2019/11/09/src-study-uber-automaxprocs/



