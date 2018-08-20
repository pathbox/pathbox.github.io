---
layout: post
title: RabbitMQ Cluster Thoughts
date:   2018-08-05 20:31:06
categories: Server
image: /assets/images/post.jpg
---

### RabbitMQ  集群安装

1. 三台rabbitmq服务node，组成内网并且互相ping通
2. 在三台node上安装`erlang`、`rabbitmq-server`
3. 读取其中一台节点上的erlang cookie，并复制到其他节点（节点之间通过cookie确定相互是否可通信）
4. 逐个启动节点：rabbitmq-server -detached（后台启动模式）
5. 查看各节点的状态： rabbitmqctl status, rabbitmqctl cluster_status
6. 将node加入集群

##### host 和 hostname

```
// 为每个node永久设置hostname,host,之后重启
vim /etc/hostname
rabbitmq-node1
vim /etc/sysconfig/network
HOSTNAME=rabbitmq-node1

//在/etc/hosts中添加
101.10.128.67    rabbitmq-node1-disk
101.10.128.68    rabbitmq-node2-mem
101.10.128.69    rabbitmq-node3-mem
```

##### 安装erlang

```
yum -y install make gcc gcc-c++ kernel-devel m4 ncurses-devel openssl-devel    ncurses-devel
yum -y install xz perl unixODBC unixODBC-devel

mkdir -p /apache/RabbitMQ/erlang/
cd /apache/RabbitMQ/erlang/
wget http://erlang.org/download/otp_src_19.2.tar.gz
tar xvf otp_src_19.2.tar.gz
cd otp_src_19.2
./configure
make
make install
//输入erl出现如下界面即表示安装完成
[root@rabbitmq-node1 otp_src_19.2]# erl
Erlang/OTP 19 [erts-8.2] [source] [64-bit] [smp:8:8] [async-threads:10] [hipe] [kernel-poll:false]

Eshell V8.2  (abort with ^G)
1>
```

##### 安装RabbitMQ
```
// 下载、解压、设置路径PATH环境,将rabbitmq-server和rabbitctl 放到 /usr/bin
cd /apache/RabbitMQ
wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.6/rabbitmq-server-generic-unix-3.6.6.tar.xz
xz -d rabbitmq-server-generic-unix-3.6.6.tar.xz
tar xvf rabbitmq-server-generic-unix-3.6.6.tar

vi /etc/profile
export PATH=$PATH:/apache/RabbitMQ/rabbitmq_server-3.6.6/sbin
source /etc/profile
```

##### erlang cookie

>Rabbitmq的集群是依赖于erlang的集群来工作的，所以必须先构建起erlang的集群环境。Erlang的集群中各节点是通过一个magic cookie来实现的，这个cookie存放在 /root/.erlang.cookie 中，文件是400的权限。所以必须保证各节点cookie保持一致，否则节点之间就无法通信。删除其中三台的/root/.erlang.cookie，然后将另一台的/root/.erlang.cookie拷贝到这三台上。文件权限是 400

##### 组成集群，在node2 3 分别运行：
```
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@rabbitmq-node1 --ram
//默认是磁盘节点，如果是内存节点的话，需要加--ram参数
rabbitmqctl start_app

rabbitmqctl cluster_status -n rabbit // 可以看到集群信息
```

一般情况，只要有一个disk磁盘节点，其他为内存节点，这样能够提高集群的性能

##### 常用命令
```
rabbitmq-server -deched  --后台启动服务
rabbitmqctl start_app  --启动服务
rabbitmqctl stop_app  --关闭服务
rabbitmq-plugins enable rabbitmq_management --启动web管理插件
rabbitmqctl add_user zlh zlh  --添加用户，密码 // 在客户端连接的时候，就需要在url上配置用户和密码
rabbitmqctl set_user_tags zlh administrator --设置zlh为administrator权限
```
