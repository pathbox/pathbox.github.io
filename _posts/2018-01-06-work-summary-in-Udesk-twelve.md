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

##### Golang 本地文档搭建
godoc -http=:6060

访问localhost:6060

##### 枚举字段别存在数据库中
我觉得将枚举字段，枚举hash 枚举map存到数据库中是不合适的，应该hard code 在代码常量中。
在数据库中，增加了数据库的查询
在数据库中，当有新的枚举增加的时候，需要将生产环境、本地开发环境、测试环境的数据库都进行同步。假设你有20台测试服务器，每台测试服务器用本地的数据库，那么需要修改20次数据库。并且要让所有开发人员知道这个新增，以完善本地开发环境。如果没有一个好的”同步流程”，出现报错的时候，就又需要花不少时间进行排查。而且，这种问题是小问题，但是排查难度却不小。

##### RabbitMQ vs Kafka

>分布式消息中间件Kafka和RabbitMQ在行业认可、服务支持、可靠性、可维护性、兼容性、易用性等方面各有特色。Kafka在开源许可证、产品活跃度、性能、安全性、可扩展性等方面优于RabbitMQ，Kafka采用的许可证更宽松，活跃度更高，性能远高于RabbitMQ，在安全性和可扩展性方面能够提供更好的保障。Kafka仅在功能上略少于RabbitMQ，但是已经具备了主要的功能。

##### Redis RPOPLPUSH实现安全队列

使用 RPOPLPUSH 获取消息时，RPOPLPUSH 会把消息返给客户端，同时把该消息放入一个备份消息列表，并且这个过程是原子的，可以保证消息的安全。当客户端成功的处理了消息后，就可以把此消息从备份列表中移除了。如果客户端因为崩溃的原因没有处理某个消息，那么就可以从备份列表destination中重新获取并处理这个消息。

redis pub/sub 目前的问题是redis某个实例宕机（此时可能已经发布了消息但并没有订阅者消费）之后再恢复时，并不会重新发送为消费的消息。为了解决这个问题，我们约定生产者和消费者直接，通过redis的数据结构List，作为双方的通讯中介，数据传送也是通过这个List，比如生产者从list左边push数据lpush mylist message，消费者从右边pop取走数据，rpop mylist 得到message。而pub/sub消息机制是一个不可靠的方式，应该使用List + RPOPLPUSH 来实现简单安全的消息队列

##### tmux 正确姿势
在服务器上进行tmux命令，在执行想要的命令。

如果是在本地tmux，然后ssh到服务器上。当本地机子睡眠，网络断了，会导致服务器上的tmux服务也关了，你执行的任务也就关了。

##### elasticsearch 对nil的处理
elasticsearch 对 nil的field会忽略，不会存到document中


##### Elasticsearch 安装

```
Elasticsearch 安装
版本: 5.6.4
系统: Ubuntu 16.04
Java: jdk-8u161-linux-x64.tar.gz
1. 安装 JRE
# 检查版本
$ java -version
$ echo $JAVA_HOME
# 删除openJDK
$ sudo apt-get purge openjdk-\*
# 如果oracle-java8-installer报404
$ cd /var/lib/dpkg/info
$ sudo cp oracle-java8-installer.postinst bak.oracle-java8-installer.postinst
$ sudo sed -i 's|JAVA_VERSION=8u151|JAVA_VERSION=8u161|' oracle-java8-installer.postinst
$ sudo sed -i 's|PARTNER_URL=http://download.oracle.com/otn-pub/java/jdk/8u151-b12/e758a0de34e24606bca991d704f6dcbf/|PARTNER_URL=http://download.oracle.com/otn-pub/java/jdk/8u161-b12/2f38c3b165be4555a1fa6e98c45e0808/|' oracle-java8-installer.postinst
$ sudo sed -i 's|SHA256SUM_TGZ="c78200ce409367b296ec39be4427f020e2c585470c4eed01021feada576f027f"|SHA256SUM_TGZ="6dbc56a0e3310b69e91bb64db63a485bd7b6a8083f08e48047276380a0e2021e"|' oracle-java8-installer.postinst
$ sudo sed -i 's|J_DIR=jdk1.8.0_151|J_DIR=jdk1.8.0_161|' oracle-java8-installer.postinst
# 安装
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository -y ppa:webupd8team/java
$ sudo apt-get update
$ sudo apt-get -y install oracle-java8-installer
$ sudo apt install oracle-java8-set-default
2. 安装 Elasticsearch
# 安装
$ sudo mkdir /usr/local/services
$ curl -l https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.4.tar.gz -o elasticsearch-5.6.4.tar.gz
$ sudo tar -xvf elasticsearch-5.6.4.tar.gz -C /usr/local/services/
# 修改文件权限
$ cd /usr/local/services/elasticsearch-5.6.4
$ sudo chmod 644 config/elasticsearch.yml config/jvm.options config/log4j2.properties
$ sudo mkdir config/scripts
$ sudo chmod 755 config/scripts
# 创建日志和数据目录
$ sudo mkdir /mnt/elasticsearch-5.6.4
$ sudo chown webuser:webuser /mnt/elasticsearch-5.6.4
$ mkdir -p /mnt/elasticsearch-5.6.4/data
$ mkdir -p /mnt/elasticsearch-5.6.4/logs
$ sudo ln -sfn /usr/local/services/elasticsearch-5.6.4 /usr/local/services/elasticsearch
# !注意: 修改 elasticsearch.yml 中的 cluster.name, 避免和现有集群冲突
3. 安装 ik 分词
$ sudo ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.6.4/elasticsearch-analysis-ik-5.6.4.zip
# 修改权限
$ sudo chmod 755 config/analysis-ik
$ sudo chmod 644 config/analysis-ik/*
4. 修改配置
Elasticsearch生产环境部署_配置
config/elasticsearch.yml
config/jvm.options
# config/elasticsearch.yml
# ======================== Elasticsearch Configuration =========================
#
# NOTE: Elasticsearch comes with reasonable defaults for most settings.
# Before you set out to tweak and tune the configuration, make sure you
# understand what are you trying to accomplish and the consequences.
#
# The primary way of configuring a node is via this file. This template lists
# the most important settings you may want to configure for a production cluster.
#
# Please consult the documentation for further information on configuration options:
# https://www.elastic.co/guide/en/elasticsearch/reference/index.html
#
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: udesk_proj_production
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: udesk_proj_es01
node.master: true
node.data: true
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
path.data: /mnt/elasticsearch-5.6.4/data
#
# Path to log files:
#
path.logs: /mnt/elasticsearch-5.6.4/logs
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
network.host: 10.1.251.187
#
# Set a custom port for HTTP:
#
http.port: 9200
transport.tcp.port: 9300
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when new node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
discovery.zen.ping.unicast.hosts: ["10.1.251.187", "10.1.251.192", "10.1.251.193", "10.1.251.190", "10.1.251.191"]
#
# Prevent the "split brain" by configuring the majority of nodes (total number of master-eligible nodes / 2 + 1):
#
discovery.zen.minimum_master_nodes: 3
#
# For more information, consult the zen discovery module documentation.
#
# ---------------------------------- Gateway -----------------------------------
#
# Block initial recovery after a full cluster restart until N nodes are started:
#
gateway.recover_after_nodes: 3
gateway.recover_after_time: 5m
gateway.expected_nodes: 5
#
# For more information, consult the gateway module documentation.
#
# ---------------------------------- Various -----------------------------------
#
# Require explicit names when deleting indices:
#
#action.destructive_requires_name: true
node.max_local_storage_nodes: 1
indices.fielddata.cache.size: 30%
action.auto_create_index: false
# config/jvm.options
## JVM configuration
################################################################
## IMPORTANT: JVM heap size
################################################################
##
## You should always set the min and max JVM heap
## size to the same value. For example, to set
## the heap to 4 GB, set:
##
## -Xms4g
## -Xmx4g
##
## See https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html
## for more information
##
################################################################
# Xms represents the initial size of total heap space
# Xmx represents the maximum size of total heap space
-Xms16g
-Xmx16g
################################################################
## Expert settings
################################################################
##
## All settings below this section are considered
## expert settings. Don't tamper with them unless
## you understand what you are doing
##
################################################################
## GC configuration
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=75
-XX:+UseCMSInitiatingOccupancyOnly
## optimizations
# pre-touch memory pages used by the JVM during initialization
-XX:+AlwaysPreTouch
## basic
# force the server VM (remove on 32-bit client JVMs)
-server
# explicitly set the stack size (reduce to 320k on 32-bit client JVMs)
-Xss1m
# set to headless, just in case
-Djava.awt.headless=true
# ensure UTF-8 encoding by default (e.g. filenames)
-Dfile.encoding=UTF-8
# use our provided JNA always versus the system one
-Djna.nosys=true
# use old-style file permissions on JDK9
-Djdk.io.permissionsUseCanonicalPath=true
# flags to configure Netty
-Dio.netty.noUnsafe=true
-Dio.netty.noKeySetOptimization=true
-Dio.netty.recycler.maxCapacityPerThread=0
# log4j 2
-Dlog4j.shutdownHookEnabled=false
-Dlog4j2.disable.jmx=true
-Dlog4j.skipJansi=true
## heap dumps
# generate a heap dump when an allocation from the Java heap fails
# heap dumps are created in the working directory of the JVM
-XX:+HeapDumpOnOutOfMemoryError
# specify an alternative path for heap dumps
# ensure the directory exists and has sufficient space
-XX:HeapDumpPath=/mnt/elasticsearch-5.6.4/logs/
## GC logging
-XX:+PrintGCDetails
#-XX:+PrintGCTimeStamps
-XX:+PrintGCDateStamps
-XX:+PrintClassHistogram
-XX:+PrintTenuringDistribution
-XX:+PrintGCApplicationStoppedTime
# log GC status to a file with time stamps
# ensure the directory exists
-Xloggc:/mnt/elasticsearch-5.6.4/logs/gc.log
# By default, the GC log file will not rotate.
# By uncommenting the lines below, the GC log file
# will be rotated every 128MB at most 32 times.
#-XX:+UseGCLogFileRotation
#-XX:NumberOfGCLogFiles=32
#-XX:GCLogFileSize=128M
# Elasticsearch 5.0.0 will throw an exception on unquoted field names in JSON.
# If documents were already indexed with unquoted fields in a previous version
# of Elasticsearch, some operations may throw errors.
#
# WARNING: This option will be removed in Elasticsearch 6.0.0 and is provided
# only for migration purposes.
#-Delasticsearch.json.allow_unquoted_field_names=true
环境配置
# /etc/security/limits.conf
# For Elasticsearch
webuser soft nofile 65536
webuser hard nofile 65536
webuser soft memlock unlimited
webuser hard memlock unlimited
# /etc/sysctl.conf
# For Elasticsearch
vm.max_map_count=262144
# 生效
$ sudo /sbin/syctl -p
5. 动态配置
# 设置日志阀值
$ curl -XPUT 'http://localhost:9200/_all/_settings?preserve_existing=true' -d '{
"index.indexing.slowlog.threshold.index.debug" : "2s",
"index.indexing.slowlog.threshold.index.info" : "5s",
"index.indexing.slowlog.threshold.index.trace" : "500ms",
"index.indexing.slowlog.threshold.index.warn" : "10s",
"index.search.slowlog.threshold.fetch.debug" : "500ms",
"index.search.slowlog.threshold.fetch.info" : "800ms",
"index.search.slowlog.threshold.fetch.trace" : "200ms",
"index.search.slowlog.threshold.fetch.warn" : "1s",
"index.search.slowlog.threshold.query.debug" : "2s",
"index.search.slowlog.threshold.query.info" : "5s",
"index.search.slowlog.threshold.query.trace" : "500ms",
"index.search.slowlog.threshold.query.warn" : "10s"
}'
6. 开机启动和自动拉起
# /lib/systemd/system/elasticsearch.service
[Unit]
Description=Elasticsearch
Documentation=http://www.elastic.co
Wants=network-online.target
After=network-online.target
[Service]
User=webuser
Group=webuser
Environment=CONF_DIR=/usr/local/services/elasticsearch/config
WorkingDirectory=/usr/local/services/elasticsearch
ExecStartPre=/usr/local/services/elasticsearch/bin/elasticsearch-systemd-pre-exec
ExecStart=/usr/local/services/elasticsearch/bin/elasticsearch --quiet -Edefault.path.conf=${CONF_DIR}
LimitNOFILE=65536
LimitNPROC=2048
LimitAS=infinity
LimitFSIZE=infinity
LimitMEMLOCK=infinity
KillSignal=SIGTERM
KillMode=process
SendSIGKILL=no
SuccessExitStatus=143
Restart=on-failure
RestartSec=45s
TimeoutSec=20s
TimeoutStopSec=0
[Install]
WantedBy=multi-user.target
生效
$ sudo systemctl enable elasticsearch.service
```


##### .gitignore规则不生效的解决办法
```
git rm -r --cached .
git add .
git commit -m 'update .gitignore'
```

##### go抢占式调度+非抢占式调度

go有抢占式调度:如果一个Goroutine一直占用CPU，长时间没有被调度过，就会被runtime抢占掉

##### 你只有 10 只小白鼠和一星期的时间,有1000瓶药水,其中有一瓶有毒药，如何检验出哪个瓶子里有毒药？

这个问题和 有3只小白鼠,有10瓶药水,其中一瓶有毒药,如何检测出哪瓶药水是毒药问题是类似的.

解题思路:

```
2^10=1024 > 1000 可以检测1024瓶药水

000=0
001=1
010=2
011=3
100=4
101=5
110=6
111=7
一位表示一个老鼠，0-7表示8个瓶子。也就是分别将1、3、5、7号瓶子的药混起来给老鼠1吃，2、3、6、7号瓶子的药混起来给老鼠2吃，4、5、6、7号瓶子的药混起来给老鼠3吃，哪个老鼠死了，相应的位标为1。如老鼠1死了、老鼠2没死、老鼠3死了，那么就是101=5号瓶子有毒。同样道理10个老鼠可以确定1000个瓶子
```

这也可以看成是组合问题. 一只老鼠喝了药水有两种置位: 活着(0) 死了(1). 有10只老鼠,可以组合出 2^10=1024 种喝药的状态, 每种喝药状态对应一个瓶子,表示这瓶药,有哪些老鼠喝了哪些老鼠没喝,可以看成就是这瓶药的喝药状态.对于老鼠来说,它不仅喝一种药,而是多种药混着一起喝. 这样,最后会有一种喝药的状态出现,就能确定这种喝药状态对应哪一瓶药,则可以确定是这瓶药是毒药
