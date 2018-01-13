---
layout: post
title: 最近工作总结(十二)
date:   2018-01-06 16:55:06
categories: Work
image: /assets/images/post.jpg
---

##### SLB(LVS)探测
阿里云SLB(LVS)对代理的服务端口进行的探测是`HEAD`方法请求。所以，你要定义一个`HEAD`根目录域名的接口

##### net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_recycle = 1

表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭；

当某个时段有大量请求来的时候，会产生很多的TIME-WAIT socket。如果这时候，被快速回收了，会导致socket 没有完全通讯完，结果就是这次调用失败了。
从而会出现大量的错误。所以，这个配置，思考清楚了再选择是否开启配置。

##### select，poll，epoll
select，poll，epoll都是IO多路复用的机制。I/O多路复用就通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。

select 的缺点：

1. 每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多的时候会很大
2. 同时每次调用select，都需要在内核轮询遍历传递进来的所有fd，这个开销在fd很多的时候也很大
3. select支持的文件描述符数量太小了，默认是1024

epoll是对select和poll的改进，就应该能避免上述的三个缺点。

epoll既然是对select和poll的改进，就应该能避免上述的三个缺点。那epoll都是怎么解决的呢？在此之前，我们先看一下epoll和select和poll的调用接口上的不同，select和poll都只提供了一个函数——select或者poll函数。而epoll提供了三个函数，epoll_create,epoll_ctl和epoll_wait，epoll_create是创建一个epoll句柄；epoll_ctl是注册要监听的事件类型；epoll_wait则是等待事件的产生。

　　对于第一个缺点，epoll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝一次。

　　对于第二个缺点，epoll的解决方案不像select或poll一样每次都把current轮流加入fd对应的设备等待队列中，而只在epoll_ctl时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表）。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用schedule_timeout()实现睡一会，判断一会的效果，和select实现中的第7步是类似的）。（事件驱动的方式）

　　对于第三个缺点，epoll没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。

总结：

（1）select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升。

（2）select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，并且要把current往设备等待队列中挂一次，而epoll只要一次拷贝，而且把current往等待队列上挂也只挂一次（在epoll_wait的开始，注意这里的等待队列并不是设备等待队列，只是一个epoll内部定义的等待队列）。这也能节省不少的开销

https://www.cnblogs.com/Anker/p/3265058.html

##### RPC、IPC、进程内通信

RPC（Remote Procedure Call） 远程程序调用。

IPC（Inter-Process Communication） 进程间通信。

进程内通信：比如多线程之间通信，goroutine 使用channel通信。

直接共享内存地址空间的多线程编程相比，IPC的缺点：

+ 采用了某种形式的内核开销，降低了性能;

+ 几乎大部分IPC都不是程序设计的自然扩展，往往会大大地增加程序的复杂度


IPC实现方式：

+ 管道

```
ps -ef | grep java | xargs echo
```

+ 共享内存
+ 信号量
+ Socket套接字

```
Socket一般情况下是用在不同的两台机器的不同进程之间通信的，当Socket创建时的类型为 AF_LOCAL或AF_UNIX 时，则是本地进程通信了(当然你也可以直接使用网络套接字，如果你觉得走下网络更酷，或者以后便于服务分离)
```
有两种类型的IPC：

1.本地过程调用(LPC)LPC用在多任务操作系统中，使得同时运行的任务能互相会话。这些任务共享内存空间使任务同步和互相发送信息。

2.远程过程调用(RPC)RPC类似于LPC，只是在网上工作。RPC开始是出现在Sun微系统公司和HP公司的运行UNIX操作系统的计算机中。

RPC:

RMI、gRPC、Thrift基于IDL跨语言。

RPC组件包括一些模块：
```
1.serviceClient：这个模块主要是封装服务端对外提供的API，让客户端像使用本地API接口一样调用远程服务。一般，我们使用动态代理机制，当客户端调用api的方法时，serviceClient会走代理逻辑，去远程服务器请求真正的执行方法，然后将响应结果作为本地的api方法执行结果返回给客户端应用。类似RMI的stub模块。

2.processor：在服务端存在很多方法，当客户端请求过来，服务端需要定位到具体对象的具体方法，然后执行该方法，这个功能就由processor模块来完成。一般这个操作需要使用反射机制来获取用来执行真实处理逻辑的方法，当然，有的RPC直接在server初始化的时候，将一定规则写进Map映射中，这样直接获取对象即可。类似RMI的skeleton模块。

3.protocol：协议层，这是每个RPC组件的核心技术所在。一般，协议层包括编码/解码，或者说序列化和反序列化工作；当然，有的时候编解码不仅仅是对象序列化的工作，还有一些通信相关的字节流的额外解析部分。序列化工具有：hessian，protobuf，avro,thrift，json系，xml系等等。在RMI中这块是直接使用JDK自身的序列化组件。

4.transport：传输层，主要是服务端和客户端网络通信相关的功能。这里和下面的IO层区分开，主要是因为传输层处理server/client的网络通信交互，而不涉及具体底层处理连接请求和响应相关的逻辑。

5.I/O：这个模块主要是为了提高性能可能采用不同的IO模型和线程模型，当然，一般我们可能和上面的transport层联系的比较紧密，统一称为remote模块
```

RPC 生态：

服务发现、服务注册、服务治理、服务负载均衡、服务监控追踪(opentracing)

RPC中间件代表：

阿里的Dubbo、当当二次开发的DubboX、新浪Motan、Facebook的Thrift、Google的gRPC

##### rails设置接口可跨域请求

```ruby
class MyController < ActionController::Base
  after_action :setup_response_headers

  def setup_response_headers
    host = request.headers['Origin'] || '*'
    response.headers['Access-Control-Allow-Origin'] = host
    response.headers['Access-Control-Allow-Headers'] = 'Origin, X-Requested-With, Content-Type, Accept,Authorization'
    response.headers['Access-Control-Allow-Methods'] = 'POST, PUT, DELETE, GET, OPTIONS'
    response.headers['Access-Control-Request-Method'] = '*'
  end
end
```

##### ActiveRecord mysql_utf8mb4

```ruby
# string_native_type_for_mysql_utf8mb4.rb

require 'active_record/connection_adapters/abstract_mysql_adapter'

module ActiveRecord
  module ConnectionAdapters
    class AbstractMysqlAdapter
      NATIVE_DATABASE_TYPES[:string] = { name: "varchar", limit: 191}
    end
  end
end
```
