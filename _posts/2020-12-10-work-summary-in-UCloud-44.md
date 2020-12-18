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

https://kingsamchen.github.io/2019/11/09/src-study-uber-automaxprocs/



### TiDB对join 连表加order by的查询方式优化不好，经常导致索引失效而产生慢查询



### KMP算法的nextArr数组

nextArr[i]的含义是在match[i]之前的字符串match[0..i-1]中，必须以match[i-1]结尾的后缀子串(不能包含match[0])与以match[0]开头的前缀子串(不包含match[i-1])最大匹配长度是多少。这个长度就是nextArr的值

### 短网址的一种方式

![image_1dqogdl4i1e3vv2118our8jf8a9.png-9.7kB](http://static.zybuluo.com/woshiaotian/9cqjhmsoky64qpf5ese92h8u/image_1dqogdl4i1e3vv2118our8jf8a9.png)

##### 1) 将长地址与一个整数建立映射(一对多)

这里整数使用int64，保存映射关系。笔者为了简单使用是MySQL数据库，如果为了更好的并发存储，还可以NoSQL数据库或者数据分分库分表。

```
"https://github.com/vearne/tinyurl" -> 10363
```

这里主键id就是整数值
长地址存储在url字段中

```
+-------+---------------------------------+---------------------+
| id    | url                             | created_at          |
+-------+---------------------------------+---------------------+
| 10000 | http://vearne.cc/archives/39217 | 2019-11-28 14:02:56 |
+-------+---------------------------------+---------------------+
```

**提示** 这里不能用哈希的原因是，哈希后的值如果太短则容易出现碰撞，如果太长则压缩的效率太低



### 修改完hosts文件之后，浏览器需要重启才能在浏览器生效

### 数据库(Mysql)连接无效(invalid connection)解决方案

一般连接数据库的代码库都有实现连接池，如golang语言database/sql库，其中SetConnMaxLifetime(d time.Duration)是用来设置连接池里每条连接关闭的时间，当d <= 0时，连接池里的连接永久重用,即永远都在连接池里，拿来就用，不管此连接是否真的有效(这里有问题，下面讲)。当d > 0时，到了时间d才会关闭连接，把连接移出连接池，但这并不是时间一到就关闭，因为当连接还在使用时会等连接完成之后，等下一个清理连接周期(周期为d)时会关闭连接，移出连接池。

   Mysql为了防止空闲连接过多，超过了参数mysql_connection之后会拒绝新连接，mysql会自动关闭空闭连接超过wait_timeout参数的时间，会关闭使用中超过interactive_timeout参数的连接。

   由于mysql会自动关闭超时连接，所以database/sql的SetConnMaxLifetime()不能设置为永久有效，要不然连接已经被mysql关闭了，但还是拿着失效的连接使用就会报invalid connection。

   解决的方案就是SetConnMaxLifetime()设置的时间小于wait_timeout就行，一般建议wait_timeout/2