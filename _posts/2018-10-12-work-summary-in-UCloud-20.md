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
