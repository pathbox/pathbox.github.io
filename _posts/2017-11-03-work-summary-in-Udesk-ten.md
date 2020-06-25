---
layout: post
title: 最近工作总结(十)
date:   2017-11-03 16:28:06
categories: Work
image: /assets/images/post.jpg
---

##### rails find_by 别忘了查询条件
线上出现了一个诡异的bug。查了很久终于发现是find_by查询没有加条件。举个例子：

```ruby
user = User.find_by(user_id)
```

结果导致user获取错误。(我的锅)

查了rails源码：

find_by源码

```ruby
def find_by(arg, *args)
  where(arg, *args).take
rescue ::RangeError
  nil
end
```

find_by就是where查询，加上take

在rails console中执行：

```ruby
User.find_by(1)
# => User Load (3.4ms)  SELECT `users`.* FROM `users` WHERE (1) LIMIT 1
```

得到的是表中的第一个user记录，所以导致了传的｀user_id｀　没有起到查询的作用，SQL语句默认是true而查询了进行了全表查询，之后再LIMIT　1　得到第一条记录

正确的代码是

```ruby
user = User.find_by(id: user_id)
```

Don't do it again

##### redis服务挂了
周末redis服务器挂了，持续了20分钟。影响到了登入后的初始界面，使得初始界面无法访问。初始界面会调的接口，应该没有redis请求的相关逻辑，避免客户连登入都登入不了。
而是哪个模块需要redis，在那个模块的接口再请求redis。如果不可避免需要请求redis，也许要思考redis集群方案，或redis挂了后及时的报警机制，以及时定位问题
单点故障就是像地雷一样

##### 从conn 读取数据时，慎用bufio

good read

```go
func read(conn net.Conn) (string, error) {
  readBytes := make([]byte, 1)
  var buffer bytes.Buffer
  for {
    _, err := conn.Read(readBytes) // conn => readBytes
    if err != nil {
      return "", err
    }
    readByte := readBytes[0]
    if readByte == DELIMITER{
      break
    }
    buffer.WriteByte(readByte) // readByte => buffer
  }
  return buffer.String(), nil
}
```

bad read

```go
func read(conn net.Conn) (string, error) {
  reader := bufio.NewReader(conn)
  readBytes, err := bufio.NewReader(conn)
  if err != nil {
    return "", err
  }
  return string(readBytes[:len(readBytes) - 1]), nil
}
```

在这里，我们队read函数的每一次调用都会导致一个新的针对当前连接的缓冲读取器被创建，我们实际上是在使用不同的缓冲读取器试图从同一个连接上读取数据。这显然会造成一些问题，因为没有任何机制来协调它们读取操作，本应该留给后面的缓冲读取器读取的数据却被提前读取到了前面的缓冲读取器的缓冲区。这不但会导致一些数据块的不完整，甚至还可能会使一些数据块被漏掉。
解决方法也很简单，直接在for代码块之前初始化一个缓冲读取器，并且保证在for循环中总是使用同一个缓冲读取器来读取护具。

##### bufio 4KB缓冲区

buf := bufio.NewReader(c.conn)

这时的buf占用 4KB(4096B)大小.在不计算其中存储数据的情况下.

##### 模拟高并发socket测试时的报错

+ Cannot assign requested address

由于客户端频繁的连服务器，由于每次连接都在很短的时间内结束，导致很多的TIME_WAIT，以至于用光了可用的端 口号，所以新的连接没办法绑定端口，即“Cannot assign requested address”。是客户端的问题不是服务器端的问题。通过netstat，的确看到很多TIME_WAIT状态的连接。

client端频繁建立连接，而端口释放较慢，导致建立新连接时无可用端口。

网上的解决方法：

执行命令修改如下2个内核参数 （需要root权限）
sysctl -w net.ipv4.tcp_timestamps=1  开启对于TCP时间戳的支持,若该项设置为0，则下面一项设置不起作用
sysctl -w net.ipv4.tcp_tw_recycle=1  表示开启TCP连接中TIME-WAIT sockets的快速回收

+ Connection reset

服务器关闭了Connection[调用了Socket.close()方法]。大家可能有疑问了：服务器关闭了Connection为什么会返回“RST”而不是返回“FIN”标志。原因在于Socket.close()方法的语义和TCP的“FIN”标志语义不一样：发送TCP的“FIN”标志表示我不再发送数据了，而Socket.close()表示我不在发送也不接受数据了。问题就出在“我不接受数据” 上，如果此时客户端还往服务器发送数据，服务器内核接收到数据，但是发现此时Socket已经close了，则会返回“RST”标志给客户端。

+ 服务器返回了“RST”时，如果此时客户端正在从Socket套接字的输出流中读数据则会提示Connection reset”；

+ 服务器返回了“RST”时，如果此时客户端正在往Socket套接字的输入流中写数据则会提示“Connection reset by peer”。

##### 理解Server and client （65535）

```
如果是作为server，被其它客户端client调用，那很简单，100w/s的调用还算不上密集，服务器集群简单接口每秒上亿的调用次数实现都很轻松。如果是作为客户端，那很多回答都提到了，单机最多只能开6万多个连接，因为系统端口上限只有65535个，系统本身服务还有些预留和占用。每开一个外部网络连接，就需要占用一个端口，和是否多线程无关。要超过这个限制，需要一个分布式的集群。每秒并发去调用外部100万个接口。。。我见过最豪迈的爬虫系统也没这么高。这是奔着底层开始就设计一个百度 google之类的搜索引擎用巨型分布式爬虫系统去的.
```

##### (转)TCP服务器端口数，最大连接数以及MaxUserPort的关系

> 关于TCP服务器最大并发连接数有一种误解就是“因为端口号上限为65535,所以TCP服务器理论上的可承载的最大并发连接数也是65535”。

> 先说结论：对于TCP服务端进程来说，他可以同时连接的客户端数量并不受限于可用端口号。并发连接数受限于linux可打开文件数，这个数是可以配置的，可以非常大，所以实际上受限于系统性能。

> 从理论上说，端口号的作用是在网络连接中标识应用层的进程，服务端一般使用众所周知的端口号进行监听，客户端连接时系统自动分配端口号。一个服务端进程服务于n个客户远程进程，只需要能通过ip地址+端口号的组合把他们区分开即可，没有必要占用本机的其他端口号，客户端连接数增加并不会占用服务器端口号，因此端口号并不能限制并发连接数。当然一台机器上端口号数量的上限确实是65536个，因为tcp首部中使用16bit去存储端口号。所以如果说65536影响了连接数，只有一种可能，就是同一台客户端机子上开n个进程去连同一个服务端进程，因为客户端ip是同一个，为了区分出这些连接，只能使用客户端连接的端口号，那么服务端和一个客户端主机之间的tcp连接数理论上线确实是65536。但是，服务端可以连接n多客户端机子呢。
实际上，确实有个限制端口号的配置，就是MaxUserPort，这实际上是一台主机向外连接使用端口数的限制，这个数也可以配置的，可能默认值才5000，实际上对于正常的服务器主机是够用的，因为你是等别人连接进来的，不是要去连接很多不同的其他主机的。当然你的服务器上可能跑了一些转发的服务，这样你就需要对外连接了，如果被限制在这个配置这儿了确实需要改。但是这个MaxUserPort确实和服务器可以承载的来自客户端的并发连接数没有关系。

> 伴随这个误解的还有另外一个误解，就是accept之后产生的已连接套接字占用了新的端口。这个绝对是错误的，linux内核不会这么写的，因为完全没必要嘛。客户端连接上来之后产生的这个socket fd就是用来区分客户端的，里面会填上客户端的ip和端口作为发包用，来自客户端的包也会使用这个fd去读取。可以试试netstat -ano，然后起一个服务器看下，客户端连上来这后产生的套接字的服务端端口还是监听的端口。

##### 一个服务端进程服务于n个客户远程进程，只需要能通过ip地址+端口号的组合把他们区分开即可，没有必要占用本机的其他端口号，客户端连接数增加并不会占用服务器端口号，因此端口号并不能限制并发连接数。

http://www.jianshu.com/p/4a58761d758f

这篇文章讲得挺好的，记录下来

##### RSA 加密解密 签名验证 公钥私钥

既然是加密，那肯定是不希望别人知道我的消息，所以只有我才能解密，所以可得出公钥负责加密，私钥负责解密；同理，既然是签名，那肯定是不希望有人冒充我发消息，只有我才能发布这个签名，所以可得出私钥负责签名，公钥负责验证。

##### 不要相信前端的输入

一次事故：

由于前端输入的电话号码多了两个回车符号，导致后端逻辑中出现异常而报错，引起一次不小的事故

再次验证了，不要相信前端的输入。后端逻辑需要更加严谨的验证

##### 关于sql count 的优化思考

在不进行分表分库的情况下，是否可以优化count查询呢？

+ 新建一个表或者count 字段（count_cache的设计），专门存储数量。增加记录的时候给对应记录的表示count的字段加一，删除记录时减一。
+ 缺点： 瓶颈转移到了写操作。当同一时间大量并发写操作的时候（批量新增或删除），会导致写操作的性能问题。特别是如果用悲观锁。

+ 借助redis，存储count值。 能发挥redis读写的性能
+ 缺点：引进了新的服务，需要进行维护。防止redis挂掉的影响

+ 仍然存RDS，使用redis作为count数值的缓存。
+ 缺点：对于插入或删除操作频繁特点的表没有很大优化效果。警惕缓存雪崩触发

##### 传输层设计小结

最近的项目涉及到了传输层的设计。在项目中使用了websocket作为传输层。将这层抽取为一个模块，不要设计太多复杂的逻辑。当需要使用别的传输方式的时候，在代码层面和设计层面，都能轻松的去处这个模块，而使用新的传输模块。或者是，多种不同的传输方法共存。
传输层，尽量减少写数据的操作。

设计思路：将真正的方法调用抽象一个模块A，每一个websocket连接是一个实体，每个websocket实体再设计对应的调用方法，去调用对应的模块A方法

现在，我们增加http的传输方式。则可以不需要更改原有代码的主要逻辑，而是增加http的传输模块，调用模块A。同理，当你想用别的传输方式：Tcp、Udp、Socket等。

##### 将错误信息返回给前端 VS 打印错误日志
在项目中遇到了这个问题。什么时候将错误返回给前端，什么时候只需要打印错误日志，将什么错误返回前端，进行了纠结的思考。

将错误返回前端
+ 前端明确需要报错信息
+ 前端关系返回值的正确与否
+ 前端只需要响应的错误信息
+ 返回错误信息利于前端调试
+ 返回错误信息会暴露一定信息给前端

打印错误日志，不返回给前端
+ 该操作不是响应操作，而是比如写数据库或redis操作
+ 前端不关心该操作的返回值
+ 对前端调试依赖小，主要考日志信息

对后端来说，打印日志比返回错误消息给前端更重要

##### macOS 系统在rails项目中对文件的大小写不敏感

zh-cn.yml zh-CN.yml 两个文件共存，在macOS系统中，会出现覆盖问题，只能识别出一个。

这个锅我不背啊！

##### Why do many websocket libraries implement their own application-level heartbeats?

+ Websocket `ping` is initiated by the server only
+ The browser Websocket API isn't able to send `ping` frames and the incoming `ping` from the server are not exposed in any way
+ These pings arer about keepalive, not presence
+ Therefore if the server goes away without a proper TCP teardown(network lost/crash etc), the client doesn't know if connection is still open
+ Adding a heartbeat at application level is a way for the client to establish the servers presence, or lack thereof. These must be sent as normal data messages because that's all the Websocket API (browser) is capable of.
