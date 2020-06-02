---
layout: post
title: 最近工作总结(二十)
date:   2018-10-12 17:32:06
categories: Work
image: /assets/images/post.jpg
---


### Linux系统下，查询当前目录下大文件夹,大文件数据

Linux系统下，查询当前目录下大文件夹数据，文件夹深度设为了2，当服务器磁盘报警的时候，可以用于查询是哪个文件夹下的数据占用磁盘最多，不是数据库服务器的话，一般都是日志啦

`sudo du -hm --max-depth=2 -h | sort -nr | head -20`

find . -type f -size +100M #查找100M以上的文件

### 如果是命令脚本文件，分为生产环境和开发环境有参数不同，比如是数据库连接参数，则改成终端传入的方式

如果是命令脚本文件，分为生产环境和开发环境有参数不同，比如是数据库连接参数，则改成外部传入的方式比如是数据库连接参数，则改成终端传入的方式。Golang很方便的编译成命令行工具，使用flag包来协助传入外部参数

例子:

```go
var account = flag.String("account", "", "DB account")
var password = flag.String("password", "", "DB password")
var host = flag.String("host", "", "DB host")
var port = flag.String("port", "", "DB port")

dbUrl := fmt.Sprintf("%s:%s@tcp(%s:%s)/uaccount?charset=utf8", *account, *password, *host, *port)
fmt.Println("dbUrl:", dbUrl)

db, err := sql.Open("mysql", dbUrl)
if err != nil {
	fmt.Println(err)
	return
}
```

```sh
./my_sql_cmd -account=xxx -password=xxx -host=127.0.0.1 -port=3306
```

### 跑完脚本还是要进行严格验证呀

### 图片相似度算法介绍

https://github.com/nivance/image-similarity

### HashMap

HashMap = hash function + array of buckets

### PID和pid文件

PID: 进程唯一id号，是一个整数,标识进程唯一性

- pid文件的内容：pid文件为文本文件，内容只有一行, 记录了该进程的ID。
用cat命令可以看到。
- pid文件的作用：防止进程启动多个副本。只有获得pid文件(固定路径固定文件名)写入权限(F_WRLCK)的进程才能正常启动并把自身的PID写入该文件中。其它同一个程序的多余进程则自动退出

### 辉哥奇谈

### 灵感来源于组装挂衣架

搬到了新的住所，一早起床开始组装可以晒衣服的衣架。突然发现(之前没在意)所有的螺丝🔩都是同一样的规格，螺帽也是同样的规格，组装的时候，任意一个螺丝都能匹配任意一个螺口。一个拧螺丝的小工具就可以用于所有螺丝，几乎是无脑的就把衣架组装好了。思考：软甲开发中，无论是架构设计还是，代码编程，是否也可以借鉴于此呢？这样，程序变得更简单，然而却依然"扎实"。并且，只要很小的成本，其他人就能理解和使用

### MySQL的binlog复制

主库有一个log dump线程，将binlog传给从库
从库有两个线程，一个I/O线程，一个SQL线程，I/O线程读取主库传过来的binlog内容并写入到relay log,
SQL线程从relay log里面读取内容，写入从库的数据库。
