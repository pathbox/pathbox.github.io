---
layout: post
title: 最近工作总结(十七)
date:   2018-06-26 20:45:06
categories: Work
image: /assets/images/post.jpg
---

---
layout: post
title: 最近工作总结(十八)
date:   2018-07-04 16:45:06
categories: Work
image: /assets/images/post.jpg
---

### 运营开发思维

Leader介绍公司组织架构和产品，可以感受到Leader对产品设计的深入性。提到了一点`运营开发思维`，第一次听到这个概念，简单总结就是： 开发的产品上线后可能会遇到什么问题，在开发前就能有所思考，然后对其进行优化。

### struct{}{} has a width of zero, it occupies zero bytes of storage

```go
func main() {
  finish := make(chan struct{})
  var done sync.WaitGroup
  done.Add(1)
  go func() {
          select {
          case <-time.After(1 * time.Hour):
          case <-finish:
          }
          done.Done()
  }()
  t0 := time.Now()
  close(finish) // finish <- struct{}{}
  done.Wait()
  fmt.Printf("Waited %v for goroutine to stop\n", time.Since(t0))
}
```

>As the behaviour of the close(finish) relies on signalling the close of the channel, not the value sent or received, declaring finish to be of type chan struct{} says that the channel contains no value; we’re only interested in its closed property.

The channel contains no value, it save the memory.

```go
var s struct{}
fmt.Println(unsafe.Sizeof(s)) // prints 0
```

>it should be evident that the empty struct has a width of zero. It occupies zero bytes of storage

```go
type S struct {
        A struct{}
        B struct{}
}
var s S
fmt.Println(unsafe.Sizeof(s)) // prints 0
```

>Because the empty struct consumes zero bytes, it follows that it needs no padding. Thus a struct comprised of empty structs also consumes no storage.

```
https://dave.cheney.net/2013/04/30/curious-channels
https://dave.cheney.net/2014/03/25/the-empty-struct
```




