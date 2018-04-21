---
layout: post
title: 最近工作总结(十四)
date:   2018-04-03 20:00:06
categories: Work
image: /assets/images/post.jpg
---

### go tool

go build

+ -o 指定输出的文件名，可以带上路径，例如 go build -o a/b/c
+ -i 安装相应的包，编译+go install
+ -a 更新全部已经是最新的包的，但是对标准包不适用
+ -n 把需要执行的编译命令打印出来，但是不执行，这样就可以很容易的知道底层是如何运行的
+ -p n 指定可以并行可运行的编译数目，默认是CPU数目
+ -race 开启编译的时候自动检测数据竞争的情况，目前只支持64位的机器
+ -v 打印出来我们正在编译的包名
+ -work 打印出来编译时候的临时文件夹名称，并且如果已经存在的话就不要删除
+ -x 打印出来执行的命令，其实就是和-n的结果类似，只是这个会执行
+ -ccflags 'arg list' 传递参数给5c, 6c, 8c 调用
+ -compiler name 指定相应的编译器，gccgo还是gc
+ -gccgoflags 'arg list' 传递参数给gccgo编译连接调用
+ -gcflags 'arg list' 传递参数给5g, 6g, 8g 调用
+ -installsuffix suffix 为了和默认的安装包区别开来，采用这个前缀来重新安装那些依赖的包，-race的时候默认已经是-installsuffix race,大家可以通过-n命令来验证
+ -ldflags 'flag list' 传递参数给5l, 6l, 8l 调用
+  -tags 'tag list' 设置在编译的时候可以适配的那些tag，详细的tag限制参考里面的 [Build Constraints](https://golang.org/pkg/go/build/)

go vet / go tool vet

```
检查代码的正确性

– 错误的printf格式
– 错误的构建tag
– 在闭包中使用错误的range循环变量
– 无用的赋值操作
– 无法到达的代码
– 错误使用mutex
    等等。

使用方法：
    go vet [package|filename]
```

go test -v -cover

用 `github.com/gorilla/mux` 包做例子

进入到包中,执行 `go test -v -cover`

```
=== RUN   TestNativeContextMiddleware
--- PASS: TestNativeContextMiddleware (0.00s)
=== RUN   TestHost
......
PASS
coverage: 85.4% of statements
ok  	github.com/gorilla/mux	0.020s
```

可以看到测试覆盖率为 85.4%

执行 `go test -coverprofile=cover.out`
```
PASS
coverage: 85.4% of statements
ok  	github.com/gorilla/mux	0.016s
```
会生成 cover.out 文件

执行 `go tool cover -func=cover.out`
```
github.com/gorilla/mux/context_native.go:10:	contextGet		100.0%
github.com/gorilla/mux/context_native.go:14:	contextSet		66.7%
github.com/gorilla/mux/context_native.go:22:	contextClear		100.0%
github.com/gorilla/mux/mux.go:17:		NewRouter		100.0%
github.com/gorilla/mux/mux.go:61:		Match			100.0%
github.com/gorilla/mux/mux.go:80:		ServeHTTP		95.5%
github.com/gorilla/mux/mux.go:118:		Get			100.0%
github.com/gorilla/mux/mux.go:124:		GetRoute		0.0%
github.com/gorilla/mux/mux.go:142:		StrictSlash		100.0%
github.com/gorilla/mux/mux.go:155:		SkipClean		100.0%
github.com/gorilla/mux/mux.go:170:		UseEncodedPath		100.0%
github.com/gorilla/mux/mux.go:180:		getNamedRoutes		80.0%
github.com/gorilla/mux/mux.go:192:		getRegexpGroup		100.0%
github.com/gorilla/mux/mux.go:199:		buildVars		100.0%
github.com/gorilla/mux/mux.go:211:		NewRoute		100.0%
github.com/gorilla/mux/mux.go:219:		Handle			100.0%
github.com/gorilla/mux/mux.go:225:		HandleFunc		100.0%
github.com/gorilla/mux/mux.go:232:		Headers			100.0%
github.com/gorilla/mux/mux.go:238:		Host			0.0%
github.com/gorilla/mux/mux.go:244:		MatcherFunc		0.0%
github.com/gorilla/mux/mux.go:250:		Methods			0.0%
github.com/gorilla/mux/mux.go:256:		Path			100.0%
github.com/gorilla/mux/mux.go:262:		PathPrefix		100.0%
github.com/gorilla/mux/mux.go:268:		Queries			0.0%
github.com/gorilla/mux/mux.go:274:		Schemes			0.0%
github.com/gorilla/mux/mux.go:280:		BuildVarsFunc		0.0%
github.com/gorilla/mux/mux.go:287:		Walk			100.0%
github.com/gorilla/mux/mux.go:300:		walk			95.0%
github.com/gorilla/mux/mux.go:352:		Vars			66.7%
github.com/gorilla/mux/mux.go:364:		CurrentRoute		0.0%
github.com/gorilla/mux/mux.go:371:		setVars			100.0%
github.com/gorilla/mux/mux.go:375:		setCurrentRoute		100.0%
github.com/gorilla/mux/mux.go:385:		getPath			80.0%
github.com/gorilla/mux/mux.go:407:		cleanPath		75.0%
github.com/gorilla/mux/mux.go:425:		uniqueVars		100.0%
github.com/gorilla/mux/mux.go:438:		checkPairs		75.0%
github.com/gorilla/mux/mux.go:449:		mapFromPairsToString	85.7%
github.com/gorilla/mux/mux.go:463:		mapFromPairsToRegex	80.0%
github.com/gorilla/mux/mux.go:480:		matchInArray		100.0%
github.com/gorilla/mux/mux.go:490:		matchMapWithString	100.0%
github.com/gorilla/mux/mux.go:518:		matchMapWithRegex	85.7%
github.com/gorilla/mux/regexp.go:27:		newRouteRegexp		92.7%
github.com/gorilla/mux/regexp.go:151:		Match			100.0%
github.com/gorilla/mux/regexp.go:167:		url			66.7%
github.com/gorilla/mux/regexp.go:195:		getURLQuery		85.7%
github.com/gorilla/mux/regexp.go:208:		matchQueryString	100.0%
github.com/gorilla/mux/regexp.go:214:		braceIndices		84.6%
github.com/gorilla/mux/regexp.go:238:		varGroupName		100.0%
github.com/gorilla/mux/regexp.go:254:		setMatch		100.0%
github.com/gorilla/mux/regexp.go:299:		getHost			100.0%
github.com/gorilla/mux/regexp.go:312:		extractVars		100.0%
github.com/gorilla/mux/route.go:44:		SkipClean		0.0%
github.com/gorilla/mux/route.go:49:		Match			92.9%
github.com/gorilla/mux/route.go:81:		GetError		100.0%
github.com/gorilla/mux/route.go:86:		BuildOnly		0.0%
github.com/gorilla/mux/route.go:94:		Handler			100.0%
github.com/gorilla/mux/route.go:102:		HandlerFunc		100.0%
github.com/gorilla/mux/route.go:107:		GetHandler		0.0%
github.com/gorilla/mux/route.go:115:		Name			83.3%
github.com/gorilla/mux/route.go:128:		GetName			100.0%
github.com/gorilla/mux/route.go:142:		addMatcher		100.0%
github.com/gorilla/mux/route.go:150:		addRegexpMatcher	81.5%
github.com/gorilla/mux/route.go:200:		Match			100.0%
github.com/gorilla/mux/route.go:213:		Headers			80.0%
github.com/gorilla/mux/route.go:225:		Match			100.0%
github.com/gorilla/mux/route.go:238:		HeadersRegexp		80.0%
github.com/gorilla/mux/route.go:266:		Host			100.0%
github.com/gorilla/mux/route.go:277:		Match			100.0%
github.com/gorilla/mux/route.go:282:		MatcherFunc		100.0%
github.com/gorilla/mux/route.go:291:		Match			100.0%
github.com/gorilla/mux/route.go:298:		Methods			100.0%
github.com/gorilla/mux/route.go:326:		Path			100.0%
github.com/gorilla/mux/route.go:342:		PathPrefix		100.0%
github.com/gorilla/mux/route.go:366:		Queries			62.5%
github.com/gorilla/mux/route.go:387:		Match			100.0%
github.com/gorilla/mux/route.go:393:		Schemes			100.0%
github.com/gorilla/mux/route.go:408:		BuildVarsFunc		100.0%
github.com/gorilla/mux/route.go:427:		Subrouter		100.0%
github.com/gorilla/mux/route.go:468:		URL			68.8%
github.com/gorilla/mux/route.go:502:		URLHost			63.6%
github.com/gorilla/mux/route.go:526:		URLPath			63.6%
github.com/gorilla/mux/route.go:551:		GetPathTemplate		80.0%
github.com/gorilla/mux/route.go:566:		GetHostTemplate		80.0%
github.com/gorilla/mux/route.go:578:		prepareVars		75.0%
github.com/gorilla/mux/route.go:586:		buildVars		100.0%
github.com/gorilla/mux/route.go:608:		getNamedRoutes		66.7%
github.com/gorilla/mux/route.go:617:		getRegexpGroup		100.0%
total:						(statements)		85.4%
```

列出了每个文件具体的方法的覆盖率

执行 `go tool cover -html=cover.out`(会在tmp中临时生成html文件) or `go tool cover -html=cover.out -o ~/mux_cover.html`(指定生成指定的html文件)

在浏览器打开这个html文件,会有 `not tracked not covered covered` 三个标签选项,以及可以选择具体的文件,查看代码测试的覆盖率和覆盖区域,不同的代码测试覆盖情况是用不同的颜色显示出来(bingo~)

```
https://golang.org/doc/cmd
http://wiki.jikexueyuan.com/project/go-command-tutorial/0.12.html
https://studygolang.com/articles/5844
https://tonybai.com/2014/10/22/golang-testing-techniques/
```

##### 最简单的理解正向代理和反向代理
+ 正向代理是在客户端的，代理客户端向服务端发起请求，正向代理的IP往往和客户端IP在同一个网络 （shadowsock）
+ 反向代理是在服务端的，代理服务端接受客户端的请求，反向代理的IP往往和服务器IP在同一个网络 （Nginx LVS）

##### 缓存更新机制的两种情况思考

查询到元数据，更新完缓存再返回数据 确保数据一致性 但会增加性能消耗

如果先返回数据再更新缓存 增加了分区容忍性和可用性(用户端能够很快得到数据),因为可能在更新缓存过程中失败,导致缓存数据没有更新,并且也没有返回给用户数据,降低了可用性(可用性是针对使用者来说的)

##### 如何理解interface

插线板有很多的插口，有三孔的、两孔的、支持USB的、支持Type-C的。只要拥有满足对应插口类型的插座电源设备，就能够通过这个插线板得到电能而工作。无论是电视机、电脑、电风扇、手机充电插座，USB数据线等等，能够和插线板插孔配对，则能够使用这个插线板，而这些电器的功能可以是不同的（实现的具体方法逻辑是可以是不同的），可能相同的是：都是两孔或三孔等插座类型（方法定义是相同的，包括名字，参数数量和类型，返回值数量和类型）

>Interface is not real, the real is the data and methods that interface declares

##### 多播、广播、单播
+ 多播支持在IPv4中是可选的，在IPv6中是必需的
+ IPv6不支持广播。使用广播的任何IPv4应用程序一旦移植到IPv6就必需改用多播重新编写
+ 广播和多播要求用于UDP或原始IP，它们不能用于TCP
+ TCP使用的是单播
+ 广播的用途之一是在本地子网定位一个服务器主机，前提是已知或认定这个服务器主机位于本地子网，但是不知道它的单播IP地址。这种操作也称为`资源发现`。另一个用途是在有多个客户主机与单个服务器主机通信的局域网环境中尽量减少分组流通。广播的应用例子： ARP（地址协议解析），ARP使用链路层广播而不是IP层广播。 DHCP、NTP、路由守护进程。
多播可以顶替广播的资源发现和减少网络分组流通

##### 接口文档 is first thing
接口文档没有定义好，后续开发真是寸步难行，而且给调试和测试都会带来很多不必要的额外工作，特别是不同组之间的联调

##### gorm 一些方法参数为地址
例子 db.Create(&user) 不能忽略`&`，要不会报错，并且难以排查。 明白什么时候需要`&`很重要

##### TCP频繁的建立连接问题

+ 三次握手建立连接、四次握手断开连接都会对性能有损耗；
+ 断开的连接断开不会立刻释放，会等待2MSL的时间，大概是1分钟；
+ 大量TIME_WAIT会占用内存，一个连接实测是3.155KB
+ 所以合理设置 keepAlive，能够降低短连接的频繁创建

##### 堆和栈 in Golang

+ 申请到栈内存好处：函数返回直接释放，不会引起垃圾回收，对性能没有影响
+ 申请到堆上面的内存会引起GC回收

```go
func F() {
	a := make([]int, 0, 20)
	b := make([]int, 0, 20000)

	l := 20
	c := make([]int, 0, l)
}
```
a会申请到栈上面，而b，由于申请的内存较大，编译器会把这种申请内存较大的变量转移到堆上面。即使是临时变量，申请过大也会在堆上面申请。

而c，对我们而言其含义和a是一致的，但是编译器对于这种不定长度的申请方式，也会在堆上面申请，即使申请的长度很短。

##### delete map in Golang
```go
for k, _ := range m {
	delete(m, k)
}
```

+ map 被清空。执行完之后调用len函数，结果肯定是0；
+ 内存没有释放。清空只是修改了一个标记，底层内存还是被占用了；
+ 循环遍历了len(m)次。上面的代码每一次遍历都会删除一个元素，而遍历的次数并不会因为之前每次删一个元素导致减少

```go
map = nil
```
之后等待GC进行回收。如果用 map 做缓存，而每次更新只是部分更新，更新的 key 如果偏差比较大，有可能会有内存逐渐增长而不释放的问题

##### 修改MySQL 默认、数据库、表、字段字符集

`MySQL版本：5.7.21`

+ SHOW VARIABLES LIKE 'char%'; 查看MySQL编码

+ sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf， lc-messages-dir = /usr/share/mysql 语句后添加 character-set-server=utf8

+ sudo vim /etc/mysql/conf.d/mysql.cnf 添加 default-character-set=utf8

sudo systemctl restart mysql.service 重启MySQL

ps aux | grep msyql => /usr/sbin/mysqld 说明MySQL正常启动，否则很有可能是配置文件有错误

```

修改数据库字符集：

`ALTER DATABASE db_name DEFAULT CHARACTER SET character_name [COLLATE ...];`

把表默认的字符集和所有字符列（CHAR,VARCHAR,TEXT）改为新的字符集：

`ALTER TABLE tbl_name CONVERT TO CHARACTER SET character_name [COLLATE ...]`
如：`ALTER TABLE logtest CONVERT TO CHARACTER SET utf8 COLLATE utf8_general_ci;`

只是修改表的默认字符集：

`ALTER TABLE tbl_name DEFAULT CHARACTER SET character_name [COLLATE...];`
如：`ALTER TABLE logtest DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;`

修改字段的字符集：
`ALTER TABLE tbl_name CHANGE c_name c_name CHARACTER SET character_name [COLLATE ...];`

如：`ALTER TABLE logtest CHANGE title title VARCHAR(100) CHARACTER SET utf8 COLLATE utf8_general_ci;`

查看数据库编码：
`SHOW CREATE DATABASE db_name;`

查看表编码：
`SHOW CREATE TABLE tbl_name;`

查看字段编码：
`SHOW FULL COLUMNS FROM tbl_name;`
```

##### HTTPS加密的内容
除域名,ip,端口外其它都加密

##### 401 vs 403

+ 401 authenticated 已认证，已验证
+ 403 authorized 经授权

简单例子理解：用户的登入，创建连接，应该是一种认证、验证。用户登入后，不同用户有不同的权利访问不同的资源，比如分普通用户、管理员。如果普通用户访问了只有管理员能访问的资源，则返回403告知没有被授权。

401 和登入，连接创建更有关系

403 和具体权利细分，是否有权利访问该资源更有关系

##### systemd 日志管理工具 journalctl
# 查看所有日志（默认情况下 ，只保存本次启动的日志）
$ sudo journalctl

# 查看内核日志（不显示应用日志）
$ sudo journalctl -k

# 查看系统本次启动的日志
$ sudo journalctl -b
$ sudo journalctl -b -0

# 查看上一次启动的日志（需更改设置）
$ sudo journalctl -b -1

# 查看指定时间的日志
$ sudo journalctl --since="2012-10-30 18:17:16"
$ sudo journalctl --since "20 min ago"
$ sudo journalctl --since yesterday
$ sudo journalctl --since "2015-01-10" --until "2015-01-11 03:00"
$ sudo journalctl --since 09:00 --until "1 hour ago"

# 显示尾部的最新10行日志
$ sudo journalctl -n

# 显示尾部指定行数的日志
$ sudo journalctl -n 20

# 实时滚动显示最新日志
$ sudo journalctl -f

# 查看指定服务的日志
$ sudo journalctl /usr/lib/systemd/systemd

`/etc/systemd/journald.conf`

# 查看指定进程的日志
```
$ sudo journalctl _PID=1

# 查看某个路径的脚本的日志
$ sudo journalctl /usr/bin/bash

# 查看指定用户的日志
$ sudo journalctl _UID=33 --since today

# 查看某个 Unit 的日志
$ sudo journalctl -u nginx.service
$ sudo journalctl -u nginx.service --since today

# 实时滚动显示某个 Unit 的最新日志
$ sudo journalctl -u nginx.service -f

# 合并显示多个 Unit 的日志
$ journalctl -u nginx.service -u php-fpm.service --since today

# 查看指定优先级（及其以上级别）的日志，共有8级
# 0: emerg
# 1: alert
# 2: crit
# 3: err
# 4: warning
# 5: notice
# 6: info
# 7: debug
$ sudo journalctl -p err -b

# 日志默认分页输出，--no-pager 改为正常的标准输出
$ sudo journalctl --no-pager

# 以 JSON 格式（单行）输出
$ sudo journalctl -b -u nginx.service -o json

# 以 JSON 格式（多行）输出，可读性更好
$ sudo journalctl -b -u nginx.serviceqq
 -o json-pretty

# 显示日志占据的硬盘空间
$ sudo journalctl --disk-usage

# 指定日志文件占据的最大空间
$ sudo journalctl --vacuum-size=1G

# 指定日志文件保存多久
$ sudo journalctl --vacuum-time=1years
```
