---
layout: post
title: 最近工作总结(十二)
date:   2018-03-11 20:00:06
categories: Work
image: /assets/images/post.jpg
---

##### redis AOF 重写机制-BGREWRITEAOF
AOF重写并不需要对原有AOF文件进行任何写入和读取，它针对的是数据库中键的当前值。

```
RPUSH list 1 2 3 4    //[1,2,3,4]
RPOP list                    //[1,2,3]
LPOP list            //[2,3]
```

AOF重写，会将上面的三个操作，用一个操作重写到AOF文件中

```
RPUSH list 2 3
```

减少AOF文件中的内容，以达到压缩AOF文件的目的。根据键的类型，使用适当的写入命令来重现键的当前值，这就是AOF重写的实现原理。

redis会开启一个后台子进程进行AOF重写程序。

+ 子进程进行AOF重写时，主进程可以继续处理命令请求；
+ 子进程携带有主进程的数据副本，使用子进程而不是线程，可以避免在锁的情况下，保证数据的安全性。

子进程在进行AOF重写期间，主进程还要继续处理命令请求，而新的命令可能对现有的数据进行修改，这会让当前数据库的数据和重写后的AOF文件中的数据不一致。

为了解决这个问题，redis增加了AOF重写缓存。这个缓存在fork出子进程之后开始启用，Redis主进程在接到新的写命令之后，除了会将这个写命令的内容追加到现有的AOF文件之外，还会追加到这个缓存中：

1、处理命令请求；

2、将写命令追加到现有的AOF文件中；

3、将写命令追加到AOF重写缓存中。

当子进程完成对AOF文件重写之后，它会向父进程发送一个完成信号，父进程接到该完成信号之后，会调用一个信号处理函数，该函数完成以下工作：

+ 将AOF重写缓存中的内容全部写入到新的AOF文件中；”执行完毕后，现有AOF文件、新的AOF文件和数据库三者的状态就完全一致了

+ 对新的AOF文件进行改名，覆盖原有的AOF文件。”执行完毕后，程序就完成了新旧两个AOF文件的替换

在整个AOF后台重写过程中，只有最后的“主进程写入命令到AOF缓存”和“对新的AOF文件进行改名，覆盖原有的AOF文件。”这两个步骤会造成主进程阻塞，在其他时候，AOF后台重写都不会对主进程造成阻塞，这将AOF重写对性能造成的影响降到最低。

每次当serverCron（服务器常规操作）函数执行时，它会检查以下条件是否全部满足，如果全部满足的话，就触发自动的AOF重写操作：

1）、没有BGSAVE命令（RDB持久化）/AOF持久化在执行；

2）、没有BGREWRITEAOF在进行；

3）、当前AOF文件大小要大于server.aof_rewrite_min_size（默认为1MB），或者在redis.conf配置了auto-aof-rewrite-min-size大小；

4）、当前AOF文件大小和最后一次重写后的大小之间的比率等于或者等于指定的增长百分比（在配置文件设置了auto-aof-rewrite-percentage参数，不设置默认为100%）

如果前面三个条件都满足，并且当前AOF文件大小比最后一次AOF重写时的大小要大于指定的百分比，那么触发自动AOF重写。

##### RPOPLPUSH 不支持 redis cluster 模式

RPOPLPUSH 不支持 redis cluster 模式, Sidekiq底层队列实现是使用的redis POPLPUSH, 所以 redis cluster 不支持Sidekiq

##### Ubuntu 14.04 tmp 文件夹
Ubuntu 14.04 tmp 文件夹下,手动创建的文件夹,在重启系统之后,刚才手动创建的文件夹已被删除

##### 线上的操作,放在访问量少的时候,要考虑对线上的影响

##### find . -name "*.go" | xargs wc -l

统计你的目录下 go代码行数

##### 为什么`TCP采用四次挥手关闭连接`

TCP连接是全双工连接，可以同时发送和接收数据。建立连接的时候，LISTEN状态下的SOCKET当收到SYN报文的建连请求后，它可以把ACK和SYN（ACK起应答作用，而SYN起同步作用）放在一个报文里来发送。在关闭连接的时候，A收到了B的FIN报文通知，它仅仅表示B不再发送数据给A了，但B仍然可以接收此刻从A发出的还没发完的数据（此时B是处于半双工的状态，即只接收数据，不再发送数据）；也就是此时A知道B不会再发送数据给我，但是我仍然可以把数据发送给B，B仍然可以接收，等这些数据发完了，A再发送FIN报文给B表示我知道现在要关闭连接了，而且我的数据也发完了，可以关闭连接了，所以它这里的ACK报文和FIN报文多数情况下都是分开发送的。

简单描述：

第一次挥手：A发送通知告诉B我要关闭连接了

第二次挥手：B收到了通知回复A，B发一个ACK通知给A：好的我知道了，我不会再接收你的数据，但我还有数据没有发完

第三次挥手：B的数据发完了（A作为接收数据方，是不知道发送方什么时候停止发送正常数据），B再发送一个通知给A告诉A我的数据也发完了，我不会再接收和发送数据了，你可以关闭连接了

第四次挥手： A收到了B的第二个通知，A最后再发一个通知给B：好的我不会再接收和发送数据了，你也不会再接收了发送数据了，关闭连接吧

连接关闭

个人认为四次挥手的原因是AB两方由全双工变为半双工再继续关闭连接，需要四次通信才能稳定的停止互相发送和接收数据，然后关闭连接，和三次握手创建TCP连接不同，因为创建连接的时候，只有一方在发送数据，另一方在接收数据。而不像连接建立后，双方都可以在发送和接收数据。

##### ab 压测工具结果描述简记

命令：

```
ab -n 100 -c 100 -H “Cookie: Key1=Value1; Key2=Value2” http://test.com/
```

```ruby
Server Software:        nginx                                               #表示访问的是nginx服务器
Server Hostname:        127.0.0.1
Server Port:            9011

Document Path:          /
Document Length:        2 bytes                                            #http相应的内容长度
Concurrency Level:      10000                                              #并发请求数 -c的参数
Time taken for tests:   0.772 seconds                                      #整个测试持续时间
Complete requests:      10000                                              #完成的请求数
Failed requests:        0                                                  #失败的请求数
Total transferred:      1180000 bytes                                      #整个场景的网络传输量
HTML transferred:       20000 bytes                                        #整个场景HTML内容传输量
Requests per second:    12954.54 [#/sec] (mean)                            #1.吞吐率， 每秒请求数
Time per request:       771.930 [ms] (mean)                                #2.用户平均请求等待时间，相当于 LR 中的平均事务响应时间，后面括号中的 mean 表示这是一个平均值
Time per request:       0.077 [ms] (mean, across all concurrent requests)  #3.务器平均请求处理时间
Transfer rate:          1492.81 [Kbytes/sec] received                      #平均每秒网络上的流量，可以帮助排除是否存在网络流量过大导致响应时间延长的问题

Connection Times (ms)                                                     #网络上消耗的时间的分解
              min  mean[+/-sd] median   max
Connect:      186  234  24.7    233     292
Processing:   141  244  46.5    234     377
Waiting:       98  218  60.5    209     372
Total:        376  478  37.9    468     613

#个请求处理时间的分布情况，50%的处理时间在4930ms内，66%的处理时间在5008ms内...，重要的是看90%的处理时间
Percentage of the requests served within a certain time (ms)      
  50%    468
  66%    487
  75%    507
  80%    524
  90%    532
  95%    537
  98%    547
  99%    569
 100%    613 (longest request)

```

##### systemd 文件目录

把编写好的 xxx.service 文件放入 `/lib/systemd/system`

在 `/usr/lib/systemd` 有systemd相关的其他文件

##### 不同机器之间的时间同步问题

在系统中,不同服务直接调用,比如微服务直接的调用,服务直接如果是在不同的机器上,不同的机器涉及到一个问题: 时间同步的问题.

比如,使用时间戳进行追踪或日志

使用时间戳作为接口鉴权过期判断

使用时间进行缓存的过期判断等

如果不同机器上的时间差距较大,则会出现异常问题,我遇到了A机器时间比B机器快3分钟,A机器进行访问B机器的时候,总是报时间戳过期的错误,就是因为A机器用"未来时间戳"对B访问,而B判断时间戳过期的代码逻辑不允许未来时间

使用`ntpdate`快速同步机器时间为标准时间

```
sudo ntpdate time.windows.com
```

##### Understand Nginx with 65535

A 代表客户端们

B代表服务器

B的前面挂着一个Nginx做负载均衡和代理转发

对A来说访问B服务器，其实先访问的是Nginx，Nginx作为服务器，可以接受65535个来自A的请求，或者是A发起的长连接，Websocket连接或gRPC连接

此时Nginx是作为反向代理`服务端`，对A来说，是不受IP端口的限制，Nginx可以建立或处理的连接数不受65535的限制，而是根据内存资源、文件描述符资源和Nginx配置的值来确定可以最多接受多少请求或长连接

简单描述一下Nginx的处理请求的原理： 首先是master worker，会监听普通worker（数量一般设为CPU核心进程数）。每个普通worker有连接池，每个连接池的大小是设置的 worker_connections。Nginx通过多路复用+事件驱动的模式，来处理这些连接请求。

worker_connections 的总数是可以超过65535的

然后，Nginx作为Client的形式，将请求代理转发给后面的B服务器。在 `upstream`中，Nginx理论上最多可以配 65535个 server，对，这里收到了65535端口数的限制。

在某个时刻，B服务器收到来自某一个Nginx的请求或连接处理理论上最多为 65535个。因为此时Nginx是作为客户端进行代理转发，转发给B服务器的时候，就收到了IP端口数量的限制。

A => (no 65535 limit) => Nginx (65535 limie) => B

所以，从A来看，某个时刻，A可以发起了超过65535的长连接或请求到Nginx， Nginx某个时刻代理转发最多65535个请求或长连接到B服务器

如果，Nginx upstream中的server不是代理的端口，而是B服务的sock文件，则Nginx转发给B的时候，不再受65535的限制（只是听Leader说的）

接下来，准备搭建测试环境，证明上述的观点。至少需要四台机器

##### socket.io 的连接断开
当浏览器页面关闭时，客户端 socket.io 不需要实现disconnection，只需要服务端实现disconnection的监听方法就可以，服务端会监听到客户端连接关闭了，然后再进行操作。

```go
so.On("disconnection", func(data map[string]string) {
})
```

##### 一段代码，理解泛型

经常遇到两个模块的功能非常相似，只是一个是处理int型数据，另一个是处理String类型数据，或者其它自定义类型数据，但是我们没有办法，只能分别写多个方法处理每种数据类型，因为方法的参数类型不同。有没有一种办法，在方法中传入通用的数据类型，这样不就可以合并代码了吗？泛型的出现就是专门解决这个问题的。

```java
使用泛型
下面是用泛型来重写上面的栈，用一个通用的数据类型T来作为一个占位符，等待在实例化时用一个实际的类型来代替。让我们来看看泛型的威力：

public class Stack<T>

{

    private T[] m_item;

    public T Pop(){...}

    public void Push(T item){...}

    public Stack(int i){

         this.m_item = new T[i];

    }

}

类的写法不变，只是引入了通用数据类型T就可以适用于任何数据类型，并且类型安全的。这个类的调用方法：

//实例化只能保存int类型的类

Stack<int> a = new Stack<int>(100);

   a.Push(10);

   a.Push("8888");//这行编译不通过，因为类a只接收int类型的数据

   int x = a.Pop();

Stack<String> b = new Stack<String>(100);

    b.Push(10);//这行编译不通过，因为类b只接收String类型的数据

   String y = b.Pop();



这个类和Object实现的类有截然不同的区别：

1.它是类型安全的。实例化了int类型的栈，就不能处理String类型的数据，其他的数据类型也一样。

2.无需装箱和拆箱。这个类在实例化时，按照所传入的数据类型生成本地代码，本地代码数据类型已确定，所以无需装箱和拆箱。

3.无需类型转换。
```

##### Golang对参数的处理报错或null pointer 会导致整个服务崩溃
Golang对参数的处理报错或null pointer 会导致服务崩溃，所以如果不确定具体参数类型时，可以进行接口话，再通过 val.(type)来处理不同的值类型。 在从map中取值的时候，善用ok， 如果key不存在，ok是false，就可以return而不再继续执行，继续执行会报错导致整个服务崩溃

使用 defer recover()， 让服务不会因为这次处理而崩溃

##### Golang 程序的优化向导

> https://stackimpact.com/docs/go-performance-tuning/
