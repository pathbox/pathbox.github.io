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
