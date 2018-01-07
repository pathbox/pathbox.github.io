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
