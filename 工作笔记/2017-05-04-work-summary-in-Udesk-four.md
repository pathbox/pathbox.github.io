---
layout: post
title: 最近工作总结(四)
date:   2017-05-04 14:57:06
categories: Work
image: /assets/images/post.jpg
---

##### 在数据库层面建立唯一索引，是最好的防止数据重复的方法，即使是在高并发的情况下
在Rails model层使用validates　方法进行唯一性的验证逻辑，比如:

```ruby
validates :content, uniqueness: { scope: [:company_id], message: "%{value}已经使用" }
```
然而，当高并发时,两个请求相差0.4s，这个在model层的验证并不是真正原子性的，是的这层验证失效。
如果能在数据库建立唯一索引[company_id, content]，这样，对高并发情况，也能支持验证，防止数据库产生脏数据。

##### union 出现在request body中时,会被阿里高防识别为可疑攻击而返回405。
比如 MySQL 的union注入攻击

##### https域名中有http的请求
https的域名下发起http的请求，浏览器会认为发起了不安全的请求而报错。
将该请求也使用https
一般https域名的地址，当使用http协议访问时，Nginx只要做了http到https的重定向，就能重定向到https协议下

##### Don't forget Sidekiq get the json not the hash params
当传递hash 参数到Sidekiq 中的时候，Sidekiq 中间会转为json.　再转回hash
所以，hash = {name: 'cary'} 到Sidekiq 中就变为　hash = {"name" => 'cary'}
hash[:name] => nil
hash["name"] => cary

##### 在同一个方法中，平级的操作，将复杂的更可能报错的操作放在后面

```ruby

def operation_data

  process_easy

  process

  process_hard

end

```
在同一个方法中，平级的操作，将复杂的更可能报错的操作放在后面,　这样可以避免当process_hard操作报错中止代码往下运行，而导致process_easy的操作没有执行。如果这些都是数据库的操作，则会导致有数据没有被操作成功。这里并没有想要用事务操作，因为其中的报错并不需要回滚，可以容忍报错，但希望见得数据库操作不要被这报错影响。

#####　binding.pry 别放在方法的最后，这样，如果方法有返回值，则不会正常返回，而返回的是nil

##### RestClient timeout
RestClient 其实有两种timeout。　read_timeout　和　open_timeout。　如果传的参数是timeout，则这两个timeout的值都是传递过来的参数
timeout的值。也可以分开进行传递，只传read_timeout　或　open_timeout

##### Go http 代码的执行流程

通过对http包的分析之后，现在让我们来梳理一下整个的代码执行过程。

首先调用Http.HandleFunc

按顺序做了几件事：

1 调用了DefaultServeMux的HandleFunc

2 调用了DefaultServeMux的Handle

3 往DefaultServeMux的map[string]muxEntry中增加对应的handler和路由规则

其次调用http.ListenAndServe(":9090", nil)

按顺序做了几件事情：

1 实例化Server

2 调用Server的ListenAndServe()

3 调用net.Listen("tcp", addr)监听端口

4 启动一个for循环，在循环体中Accept请求

5 对每个请求实例化一个Conn，并且开启一个goroutine为这个请求进行服务go c.serve()

6 读取每个请求的内容w, err := c.readRequest()

7 判断handler是否为空，如果没有设置handler（这个例子就没有设置handler），handler就设置为DefaultServeMux

8 调用handler的ServeHttp

9 在这个例子中，下面就进入到DefaultServeMux.ServeHttp

10 根据request选择handler，并且进入到这个handler的ServeHTTP

  mux.handler(r).ServeHTTP(w, r)
11 选择handler：

A 判断是否有路由能满足这个request（循环遍历ServerMux的muxEntry）

B 如果有路由满足，调用这个路由handler的ServeHttp

C 如果没有路由满足，调用NotFoundHandler的ServeHttp

##### MongoDB 权威指南 小记录

范式化和反范式化。范式化是将数据分散到多个不同的集合，不同的集合之间可以互相引用数据。如果需要修改这块数据，只需要修改保存这块数据的那一个文档就行了。但是，MongoDB没有提供连接(join)工具，所以在不同集合之间执行连接查询要进行多次查询。
反范式化与范式化相反：将每个文档所需要的数据都嵌入在文档内部。每个文档都拥有自己的数据副本，而不是所有文档共同引用同一个数据副本。这意味着，如果信息发生了变化，那么所有相关文档都需要进行更新，但是在执行查询时，只需要一次查询，就可以得到所有数据。
范式化利于更新，而需要做更多的查询，比如join或多次查询。反范式化利于查询，一次查询就得到所有数据，但是不利于更新。数据同步更困难。

许多人混淆复制和分片的概念，复制是让多台服务器都有统一的数据副本，每一台服务器都是其他服务器的镜像，而每一个分片都有其他分片拥有不同的数据子集。 进行分片一般需要有定义分片键，分片键需要建立索引。需要根据分片键定制一个路由机制。路由机制可以简单的说，就是根据一定的规则，将这个文档存入某个分片中。但一个查询来时，按照路由机制，去对应的分片中得到这个文档。这个和ES的分片机制是相似的。简单的路由机制，比如取ID这种唯一字段值得hash值。MongoDB中需要建立配置服务器，配置服务器和普通的MongoDB服务器一样，但是它相当于集群的大脑，保存着集群和分片的元数据，即各个分片包含哪些数据的信息。一般建立三台独立的物理机器。配置服务器中存的是数据的分布表，而不是真实的文档数据，所以，相对来说数据量不会很大。可以认为，配置服务器就是存储着“路由机制”。MongoDB中，对分片还进行“块（chunk）机制”的存储方案。一个块只存在于一个分片上，所以MongoDB用一个比较小的表就能够维护块和分片的映射。当一个块增长到特定大小时，MongoDB会自动将其拆分为两个较小的块。一个文档属于且只属于一个块。

##### Global Variable is "devil"
不要轻易使用全局变量,　如果有用到全局变量，一定小心内存泄露。

如果用的别人的Gem或Go 的库有用了全局变量，要注意是否会造成内存泄露。如何识别？ 当你的application 内存持续增长，但是负载其实不多的时候，很有可能是这个原因。特别是Go的库。因为在Go的中，很多库都会使用全局变量，用来作为缓存等操作。特别是全局变量`<map>`。有可能只是往里面加key的值，但是当这个key如果不再使用了，要将它删除。第二种情况，比如gorilla/sessions　这个库，它是每个request都会新建两个全局变量map，如果没有使用`<gorilla/mux>`或者，没按照文档提示：`<http.ListenAndServe(":8080", context.ClearHandler(http.DefaultServeMux))>`则会造成内存泄露。

全局变量确实很方便的帮我们解决了很多问题，但是，一不小心，也许就会掉进陷阱中。

##### Effective ruby tip
所有的变量代表的值都有可能是nil

浅拷贝只拷贝了对象，对于像Array一样的集合，这意味着仅仅复制了容器本身而没有复制其中的元素。可以在不影响原始集合的情况下增添或移除其中的元素，但修改元素本身则是有影响的。修改任一元素都会影响原始集合本身，同样也会影响到任何引用这一元素的对象。

##### Golang Slices And The Case Of The Missing Memory
[文章链接](http://openmymind.net/Go-Slices-And-The-Case-Of-The-Missing-Memory/)

```go
body, _ := ioutil.ReadAll(resp.Body)
```

slice 的设计是当bytes数达到现在的长度值时，slice会去申请现在两倍的容量，所以，有时候会出现cap 是 len两倍的大小，而浪费了一定的内存

+ 2x growth algorithm

If we know the size of the response, the above code is rather inefficient. The buffer starts at 512 bytes and uses 2x growth algorithm. The result is a number of unnecessary allocation and high likelihood of wasted memory (again, this last part isn't immediately obviously until you compare the length of body with its capacity). The solution?:

buffer := bytes.NewBuffer(make([]byte, 0, resp.ContentLength)
buffer.ReadFrom(res.Body)
body := buffer.Bytes()

Given a ContentLength of 70KB, what would you imagine len(body) and cap(body) to equal? The length will be the expected 70KB, but the capacity will be 143872 bytes. For a reason that isn't clear to me, the bytes.Buffer class requires bytes.MinRead extra space. It has a value of 512. So unless we create a buffer of res.ContentLength + bytes.MinRead, we'll end up with an array which is 70K*2 + 512 of actual space.（2x + 512）

When you know the size of the data, like in the above case with a ContentLength, you should use the `<io.ReadFull>`:

```go
body := make([]byte, 0, resp.ContentLength)
_, err := io.ReadFull(res.Body, body)
```

Otherwise, you'll have to first read it into a growing buffer and then copy it back into a fixed-length array. You'll need to decide if the saved memory is worth the extra CPU. We use something like:

```go
//ioutil.ReadAll starts at a very small 512
//it really should let you specify an initial size
buffer := bytes.NewBuffer(make([]byte, 0, 65536))
io.Copy(buffer, r.Body)
temp := buffer.Bytes()
length := len(temp)
var body []byte
//are we wasting more than 10% space?
if cap(temp) > (length + length / 10) {
  body = make([]byte, length)
  copy(body, temp)
} else {
  body = temp
}
```
但是上面的方案能够减少不必要的内存空间的创建，但是，增加了一定的cpu使用

##### 服务集群的目的
服务集群的目的不仅是为了提高服务的性能，也为了备份、容灾
