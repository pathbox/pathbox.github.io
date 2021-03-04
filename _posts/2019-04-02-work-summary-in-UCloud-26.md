---
ayout: post
title: 最近工作总结(26)
date:  2019-04-02 17:00:06
categories: Work
image: /assets/images/post.jpg
---

### Microservices Concerns

- Config Management                  配置中心
- Auto Scaling & Self Healing        自动扩展和自我修复
- Scheduling & Deployment            调度和部署
- Distributed Tracing                分布式追踪
- Centralized Metrics                集中式度量中心
- Centralized Logging                集中式日志中心
- Service Security                   服务安全
- API Management                     API管理
- Resilience & Fault Tolerance       服务弹力性和失败容忍，节流
- Service Discovery & LB             服务发现与负载均衡

### slice的扩容规则
若 Slice cap 大于 doublecap且大于1024，则扩容后容量大小为 新 Slice 的容量（超了基准值，我就只给你需要的容量大小）
若 Slice len 小于 1024 个，在扩容时，增长因子为 1（也就是 3 个变 6 个）
若 Slice len 大于 1024 个，在扩容时，增长因子为 0.25（原本容量的四分之一）
注：也就是小于 1024 个时，增长 2 倍。大于 1024 个时，增长 1.25 倍 因为进行了内存对齐操作

nums: [1 0 0] , len: 3, cap: 3
nums: [1 0 0] ,len: 3, cap: 3
dnums: [1 1 2 3], len: 4, cap: 6
往 Slice append 元素时，若满足扩容策略，也就是假设插入后，原本数组的容量就超过最大值了

这时候内部就会重新申请一块内存空间，将原本的元素拷贝一份到新的内存空间上。此时其与原本的数组就没有任何关联关系了，再进行修改值也不会变动到原始数组

翻新扩展：当前元素为 kindNoPointers，将在老 Slice cap 的地址后继续申请空间用于扩容
举家搬迁：重新申请一块内存地址，整体迁移并扩容

### 上游超时必须始终大于下游总超时

(前端)上游超时必须始终大于(后端)下游总超时（包括重试次数

上游超时应设置在“边缘”服务器上并始终级联

例子：两个OpenResty， A，B，A在B的前面。A的请求连接超时 < B的请求连接超时。 一种情况：当A因超时和B之间的请求连接断开了，B的返回就无法到达A了，B实际上还没超时，可能正常返回，这样就出现了B时间正常返回了，然而A返回给前端的是超时或报错，从而出现诡异的现象

所以，应该注意要：`A的请求连接超时 > B的请求连接超时`

### Golang的goroutinue调度器
Golang老的goroutinue调度器模型是：`M → G` 模型

缺点

1. 只有一个G全局队列，取g时需要加锁，导致锁竞争激烈

2. M转移G会造成延迟和额外的系统负载。比如当G中包含创建新协程的时候，M创建了G’，为了继续执行G，需要把G’交给M’执行，也造成了很差的局部性，因为G’和G是相关的，最好放在M上执行，而不是其他M'。

2. M中的mcache是用来存放小对象的，mcache和栈都和M关联造成了大量的内存开销和差的局部性。

3. 系统调用导致频繁的线程阻塞和取消阻塞操作增加了系统开销

Golang新的调度器模型: `M → P → G`
在用户态引入了`P`

两层调度关系

优点

1. 有全局的G队列，P也有本地G队列，这样将G分到多个G队列中，极大的减小了锁的竞争
2. work stealing，当本线程无可运行的G时，尝试从其他线程绑定的P偷取G，而不是销毁线程
3. hand off，当本线程因为G进行系统调用阻塞时，线程释放绑定的P，把P转移给其他空闲的线程执行
4. 抢占式调度: 在coroutine中要等待一个协程主动让出CPU才执行下一个协程，在Go中，一个goroutine最多占用CPU 10ms，防止其他goroutine被饿死，这就是goroutine不同于coroutine的一个地方


### HTTP(S) Benchmark Tools

HTTP(S) benchmark tools, testing/debugging, & restAPI (RESTful)
---------------------------------------------------------------
*Located in alphabetical order (not prefer)*

HTTP(S) Benchmark Tools
=======================
* [__ab__](http://en.wikipedia.org/wiki/ApacheBench) – slow and single threaded, written in `C`
* [__apib__](https://github.com/apigee/apib) – most of the features of ApacheBench (`ab`), also designed as a [more modern replacement](https://github.com/apigee/apib#design), written in `C`
* [__baloo__](https://github.com/h2non/baloo) – Expressive end-to-end HTTP API testing made easy, written in Go (`golang`)
* [__baton__](https://github.com/americanexpress/baton) – HTTP load testing, written in Go (`golang`)
* [__bombardier__](https://github.com/codesenberg/bombardier) – Fast crossplatform HTTP benchmarking tool, written in Go (`golang`)
* [__curl-loader__](http://curl-loader.sourceforge.net/) – performance loading of various application services and traffic generation, written in `C`
* [__drill__](https://github.com/fcsonline/drill) – Drill is a HTTP load testing application inspired by Ansible syntax, written in `Rust`
* [__fasthttploader__](https://github.com/hagen1778/fasthttploader) – benchmark (kinda ab) with autoadjustment and charts based on fasthttp library, written in Go (`golang`)
* [__fbender__](https://github.com/facebookincubator/fbender) – A load-testing command line tool for generic network protocols (`HTTP`, `DNS`, `DHCP`, …), written in Go (`golang`)
* [__fortio__](https://github.com/istio/fortio) – load testing library and command line tool and web UI. Allows to specify a set query-per-second load and record latency histograms and other useful stats, written in Go (`golang`)
* [__gatling__](http://gatling.io) – High performance load testing framework based on Scala, Akka and Netty, written in `Scala`
* [__go-wrk__](https://github.com/tsliwowicz/go-wrk) – a HTTP benchmarking tool based in spirit on the excellent wrk tool ([`wg/wrk`](https://github.com/wg/wrk)), written in Go (`golang`)
* [__goad__](https://github.com/gophergala2016/goad) – Goad is an AWS Lambda powered, highly distributed, load testing tool, written in Go (`golang`)
* [__gobench__](https://github.com/cmpxchg16/gobench) – HTTP/HTTPS load testing and benchmarking tool, written in Go (`golang`)
* [__gohttpbench__](https://github.com/parkghost/gohttpbench) – `ab`-like benchmark tool run on multi-core cpu, written in Go (`golang`)
* [__goloris__](https://github.com/valyala/goloris) – Slowloris for NGINX DoS attack, written in Go (`golang`)
* [__hey__](https://github.com/rakyll/hey) – HTTP(S) load generator, ApacheBench (`ab`) replacement, formerly known as [__rakyll/boom__](https://github.com/rakyll/boom), written in Go (`golang`)
* [__htstress__](https://github.com/arut/htstress) – multithreading high-load bechmarking services (>5K rps), written in `C`/`Linux`
* [__httperf__](https://github.com/httperf/httperf) – difficult configuration, slow and single threaded, written in `C`
* [__inundator__](https://github.com/opsengine/inundator) – A simple and high-throughput HTTP flood program, written in `C`/`Linux`
* [__jmeter__](http://jmeter.apache.org/) – Apache  JMeter™, pure application designed to load test performance both on static and dynamic resources, written in `Java`
* [__k6__](https://github.com/loadimpact/k6) - A modern load testing tool scriptable in ES6 JS with support for HTTP/1.1, HTTP/2.0 and WebSocket, written in Go (`golang`)
* [__locust__](https://locust.io/) – easy-to-use, distributed load testing tool with real-time web UI. Simulates a swarm of concurrent users, the behavior of each of them is defined by your python code. Written in `Python`
* [__mgun__](https://github.com/byorty/mgun) – A modern tool for load testing HTTP servers, written in Go (`golang`)
* [__pounce__](https://github.com/X4/pounce) – evented, but results fluctuate, it's sometimes faster than `htstress`, written in `C`
* [__siege__](http://www.joedog.org/siege-home/) – slow and single threaded, written in `C`
* [__slapper__](https://github.com/ikruglov/slapper) – Simple load testing tool with real-time updated histogram of request timings, written in Go (`golang`)
* [__slow_cooker__](https://github.com/BuoyantIO/slow_cooker) – A load tester focused on lifecycle issues and long-running tests, service with a predictable load and concurrency level for a long period of time, written in Go (`golang`)
* [__sniper__](https://github.com/lubia/sniper) – powerful & high-performance http load tester, written in Go (`golang`)
* [__thrash__](https://github.com/TylerBrock/thrash) – HTTP Micro Benchmarker, written in Go (`golang`)
* [__tsung__](http://tsung.erlang-projects.org/) – Simulate stress users in order to test the scalability and performance of IP based client/server applications `HTTP`, `WebDAV`, `SOAP`, `PostgreSQL`, `MySQL`, `LDAP` and `Jabber`/`XMPP` servers, written in `Erlang`
* [__vegeta__](https://github.com/tsenart/vegeta) – HTTP load testing tool and library, written in Go (`golang`)
* [__weighttp__](https://github.com/lighttpd/weighttp) – multithreaded, but slower than htstress without keepalive, written in `C`
* [__welle__](https://github.com/rylev/welle) – ab (Apache Benchmark) like tool, written in `Rust`
* [__wrk__](https://github.com/wg/wrk) – multithreaded, ~~but doesn't offer concurrent connections and a keepalive switch~~, written in `C`/`Lua`
* [__wrk2__](https://github.com/giltene/wrk2) – constant throughput, correct latency recording variant of wrk, written in `C`/`Lua`

        Concurrent connections are enabled with:
          -c, --connections <N>  Connections to keep open
        And keepalive (which is default) can be disabled using:
          -H "Connection: close"
* [__yandex-tank__](https://github.com/yandex/yandex-tank) – Load and performance benchmark tool, written in `Python`/`C|C++|Asm` ([phantom](https://github.com/yandex-load/phantom))

Toolkit for testing/debugging HTTP(S) and restAPI (RESTful)
===========================================================
* [__bat__](https://github.com/astaxie/bat) – Go implement CLI, cURL-like tool for humans, written in Go (`golang`)
* [__curl__](https://github.com/curl/curl) – Powerful features command-line tool for transferring data specified with URL syntax, written in `C`
  - [Online curl command line builde](https://curlbuilder.com/)
* [__curlconverter__](https://github.com/NickCarneiro/curlconverter) – convert curl commands to python, javascript, php
* [__httpie__](https://github.com/jkbrzt/httpie) – client, user-friendly curl replacement with intuitive UI, JSON support, syntax highlighting, wget-like downloads, extensions, written in `Python`
  * [__curlie__](https://curlie.io) – If you like the interface of [HTTPie](https://httpie.org) but miss the features of [curl](https://curl.haxx.se), curlie is what you are searching for. Curlie is a drop-in replacement for `httpie` that use `curl` to perform operations, written in Go (`golang`)
* [__jaggr__](https://github.com/rs/jaggr) – JSON Aggregation CLI, Jaggr can be used to integrate [vegeta](https://github.com/tsenart/vegeta) with [jplot](https://github.com/rs/jplot), written in Go (`golang`)
* [__jq__](https://github.com/stedolan/jq) – is a lightweight and flexible command-line JSON processor, written in `C`
* [__httpstat__](https://github.com/davecheney/httpstat) - It's like curl -v, with colours

SaaS/PaaS
=========
* [__BlazeMeter__](https://blazemeter.com/) – offers a cross-enterprise test automation framework for the entire technical team (developers, devops, ops and QA) throughout the product development lifecycle. Run continuous or ‘on demand’ testing for APIs, mobile apps and websites. Run from the cloud, on-premise or as a hybrid solution. Use with JMeter &amp; Selenium WebDriver &amp; integrate with your existing CI, CD &amp; APM tools.
* [__NewRelic__](http://newrelic.com/) – software analytics tool suite used by developers, ops, and software companies to understand how your applications are performing in development and production
* [__NGINX Amplify__](https://amplify.nginx.com/) – Visually identify performance bottlenecks, overloaded servers, or potential DDoS attacks. Improve and optimize NGINX performance with intelligent advice and recommendations. Get alerts when something is wrong with the delivery of your application. Plan capacity and performance for web applications. Keep track of systems running NGINX [<sup>1</sup>](https://www.nginx.com/blog/announcing-amplify/)


Links
=====
- [Multi-Mechanize | Performance Test Framework](http://testutils.org/multi-mechanize/)
- [Performance test tools list](http://www.opensourcetesting.org/performance.php)
- [REST API client](https://insomnia.rest/) - simple, beautiful, and free REST API client (`Mac`, `Windows`, and `Linux`)

### The Four Pillars of Observability
分布式系统，微服务 四个可观察性的关键支柱
- Logging: Recording of discrete events
- Metrics: Aggregation of similar events to gain a higher level of insight
- Tracing: Recording, oedering and binging of data from connected events to provide context.
- Alerting: Notification when event behavior fallsoutside of acceptable threshold and could potentially become problematic

### Big Endian and Little Endian

big endian是指低地址存放最高有效字节（MSB），而little endian则是低地址存放最低有效字节（LSB）

比如数字0x12345678在两种不同字节序CPU中的存储顺序如下所示：

Big Endian

低地址                                            高地址
----------------------------------------->
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     12     |      34    |     56      |     78    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Little Endian

低地址                                            高地址
----------------------------------------->
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     78     |      56    |     34      |     12    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

各自优点:
Big Endian: 符号位的判定固定为第一个字节，容易判断正负。
Little Endian: 长度为1，2，4字节的数，排列方式都是一样的，数据类型转换非常方便。

### 一个多选其一的参数搜索问题方式的思考
可以传多个参数，但只会选择其中一个进行过滤搜索，你会选择强制要求只传一个，还是可以传多个，但会按照参数一定的优先级来判断进行搜索呢。我觉得从接口友好度的情况出发，我会选择`按照参数一定的优先级`来判断进行搜索，这个优先级可以根据业务需求定义，可以根据性能角度定义

### 当并发超过连接池连接时，查询会完全僵死，而之前使用的连接又不得close归还到连接池中

解决方法: 增大连接池大小

```go
package main

import (
	"database/sql"
	"fmt"
	"time"

	_ "github.com/go-sql-driver/mysql"
)

// 当并发超过连接池连接时，查询会完全僵死，而之前使用的连接又不得close归还到连接池中  

func main() {
	url := fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=utf8&timeout=3s&readTimeout=30s&writeTimeout=60s",
		"xxx", "xxx.cn", "xxx", 3306, "xxx")

	db, err := sql.Open("mysql", url)
	if err != nil {
		panic(err)
	}

	// db setting
	db.SetConnMaxLifetime(time.Minute * 3)
	db.SetMaxIdleConns(10)
	db.SetMaxOpenConns(20)

	for i := 0; i < 100; i++ {
		fmt.Printf("Num:%d\n", i)
		go query(db, i)
	}

	c := make(chan int)
	<-c
}

func query(db *sql.DB, i int) {

	var companyID int

	sqlS := `
		SELECT company_id
		FROM t_company
		WHERE company_id= 100 limit 1
		`
	rows, err := db.Query(sqlS)
	if err != nil {
		panic(err)
	}
	defer rows.Close()

	for rows.Next() {
		rows.Scan(&companyID)
		sql2 := `
		SELECT company_id
		FROM t_member
		WHERE company_id= 100
		`
		fmt.Println("Start...")
		rows, err = db.Query(sql2)
		if err != nil {
			panic(err)
		}
		defer rows.Close()

		for rows.Next() {
			rows.Scan(&companyID)
		}
		fmt.Printf("Company ID: %d\n", companyID)
	}
	fmt.Printf("Number: %d\n", i)
}
```

### macOS下解决Excel中身份证号被科学计数问题
- 数据库导出csv文件
- 使用Numbers打开csv文件,Numbers打开的csv文件，身份证号码保持不变
- 另存为excel文件

### 消息队列中的单播和多播

- 单播: 一个消息只能被一个消费者消费
- 多播: 一个消息可以被多个满足条件的消费者消费
他们考虑的场景是不同的
