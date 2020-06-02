---
layout: post
title: 最近工作总结(十五)
date:   2018-05-01 11:24:06
categories: Work
image: /assets/images/post.jpg
---


### 理解互斥锁、读写锁

读写锁
- 只要没有线程持有某个给定的读写锁用于写，那么任意数目的线程可以持有该读写锁用于读
- 仅当没有线程持有某个给定的读写锁用于读或写时，才能分配该读写锁用于写。

>只要没有线程在修改某个给定的数据，那么任意数目的线程都可以拥有该数据的读访问权。仅当没有其他线程在读或修改某给给定的数据时，当前线程才可以修改它

规律：临界区没有加写锁，可以被多个线程获取读锁读和一个线程获取写锁写；资源加了读锁，只能被其他线程获取读锁读。

而互斥锁不分读写，保护临界区某个时段只有一个线程对临界区进行读写操作。相比读写锁，损失了读的并发性能。

### Restarting without closing the socket

Graceful Restart in Golang

- Fork a new process which inherits the listening socket
- The child performs initialization and starts accepting connections on the socket
- Immediately after, child sends a signal to the parent causing the parent to stop accepting connections and terminate

- Fork 一个新的进程，这个进程继承正在监听的socket的所有属性信息
- 子进程执行初始化，之后开始接受新的socket连接
- 之后，子进程马上发送signal信号通知父进程停止接受新的连接然后终止

> https://grisha.org/blog/2014/06/03/graceful-restart-in-golang/

### 数据库四种食物隔离级别先行小记

+ 未提交读(Read Uncommitted): 允许脏读，也就是可能读取到其他会话中未提交事物修改的数据
+ 提交读(Read Committted): 只能读取到已提交的数据。Oracle等多数数据库默认是这个级别
+ 可重复读(Repeated Read): 可重复读。在同一个事物内的查询都是事物开始时刻一致的，InnoDB默认级别。该级别消除了不可重复读，但是还存在幻读
+ 串行读(Serializable): 完全串行化的读，每次读都需要获得表级共享锁，读写互相都会阻塞
---
- 脏读：是指当一个事务A正在访问数据，并且对数据进行了修改，而这个修改还没有提交到数据库中，这时，另一个事务也访问了这个数据，访问到的是A修改前的数据，然后使用了这个数据
- 不可重复读：是指在一个事务内，多次相同条件读取同一数据。在这个事务还没有结束时，另一个事务也访问了同一数据。在第一个事务中的两次读取数据之间，由于第二个事务的修改，那么第一个事务两次读到的数据可能是不一样的。这样就发生了在一个事务内两次读到的数据是不一样的，因此称为是不可重复读
- 可重复读：消除了不可重复读的的情况。在同一个事务内，多次相同条件读取同一数据，不受其他事务对这个数据的修改的影响，都是读取到同样的数据，但不能消除幻读的影响
- 幻读：第一个事务对一个表中的数据进行了修改，这种修改涉及到表中的全部数据行。同时，第二个事务也修改这个表中的数据，这种修改是向表中插入一行或多行新数据。那么，以后就会发生操作第一个事务的用户发现表中还有没有修改的数据行（发现这新增的数据记录），就好象发生了幻觉一样


### 数据库三星索引原则

##### 索引的优点

1 索引大大减少了服务器需要扫描的数据量

2 索引可以帮助服务器避免排序和临时表

3 索引可以将随机I/O变成顺序I/O

##### 索引的缺点

1.降低了带索引的数据列插入/修改/删除的速度

2.索引越多，占据的磁盘空间越大

##### 三星索引

索引将相关的记录放到一起则获得一星；如果索引的数据顺序和查找的排列顺序一致则获得二星；如果索引中的列包含了查询中需要的全部列则获得三星。

##### 索引选择性

索引的选择性指不重复的索引值（基数，cardinality）和数据表的记录总数（#T）的比值，范围在1/#T和1之间。索引的选择性越接近1，查询效率就越高。唯一性索引的选择性为1，因此性能也是最好的。

##### 前缀索引

对于很长的字符列，创建索引可能索引变得很大且慢，一种方法就是使用哈希索引，另一种方法就是对列的前缀进行索引，这样可大大节约索引空间，提供索引效率，但会降低索引的选择性。

对于Blob，Text或者很长的varchar类型的列，必须使用前缀索引，因为mysql不允许索引这些列的完整长度。

建立前缀索引的要点：选择足够长的前缀以保证较高的选择性，使得前缀索引选择性尽量接近于索引整个列

### type is func() in Golang

```go
package main

import "fmt"

// 定义一个func类型，定义好参数和返回值，具体这个selfDoHandler的逻辑代码是怎样的，在初始化这个类型具体实例的时候定义
type selfDoHandler func(param string) string

func main() {

	var sdh selfDoHandler //定义一个变量sdh 为 selfDoHandler类型

	sdh = func(p string) string { // 这个func 的参数和返回值和 selfDoHandler 类型一致就可以
		s := "Hello" + p // 具体实现selfDoHandler 方法逻辑
		return s
	}

	fmt.Println(sdh(" World"))
}

/*
可以用在哪？ 定义一个map
map[string]selfDoHandler

根据key，可以设置不同的selfDoHandler，但是他们的参数和返回值都和selfDoHandler一致，只是具体实现的时候的代码逻辑可以不一样

这样不同的key相当于进行了不同的逻辑处理。具体的逻辑实现还可以用定义一个func的方式
*/
```

##### 理解回调函数

- 第一步：先注册函数，可以注册一个或多个函数，但不执行它们
- 第二步：当程序执行到某处时，才调用其中一个或多个函数

##### 关于字符编码

- 一个表示：一个字节byte 是 8位bit。 1byte = 8bit(0000 0000)

- ASCII编码

`ASCII`（American Standard Code for Information Interchange，美国信息互换标准代码）编码方案，用来保存英文字符

现在应该有1-255位

- 用于中文的GB*编码

`GB2312`是对ASCII的扩展，两个字节长的汉字字符和一个字符长的英文字符并存在一套编码方案里。

`是` 十六进制编码(0xCAC7) 二进制编码(11001010 11000111)

```
但中国的汉字实在太多，后来又对GB2312进行了扩展，低字节就没有要求必须大于127了，只要第一个字节大于127就可以固定表示一个汉字，这个编码方案就是GBK，GBK包含了GB2312的所有内容，同时增加了20000个新汉字（包括繁体字）和符号。

后来少数名族也开始使用计算机，我们又对GBK进行了扩充，新的方案叫做GB18030，现在就好了，中国人民的悠久历史可以全部存到计算机中啦。

可是后来问题又来了，全世界很多国家都像中国一样搞了一套自己的编码标准，互相之间一点也不兼容，用国外的软件除非安装一套他们的编码系统，否则就用不了
```

- UNICODE编码

包含地球上所有字符的编码！这就是UNICODE，但是UNICODE并不是一种具体的编码方案，UNICODE只是定义了字符的集合和唯一确定的编号，具体储存为什么样的字节流，取决于字符编码的方案，比如UTF-8和UTF-16，还有上述的GB18030。也就是说，虽然每个字符在UNICODE字符集种能找到唯一的编号，但最终决定字符流的是具体的字符编码。例如，同样的字符"A"，UTF-8得到的字节流是0x41，UTF-16得到得是0x00 0x41

UTF-8是目前使用得一套最广泛的UNICODE编码，它使用1-4个字节来编码字符，单字节和ASCII一样，对于其他字符，使用2-4个字节表示，比如汉字就用了3个字节表示

```
UTF8  十进制  十六进制
'a'    97      0x61
'A'    65      0x41
'0'    48      0x30
 0      0      0x00
```

Golang中的示例：

>Go使用rune来存储字符串，它是int32的别名，因为一个UTF-8字符的长度可能是1，2，3，4个字节，所以如果要统计字符数，就需要计算的是rune的个数而不是字节数了。

```go
str := "你好，世界"
fmt.Println("rune len:", len([]rune(str)))   // 5
fmt.Println("int32 len:", len([]int32(str))) // 5
fmt.Println("Byte len", len(str))            // 15
```

##### 使用Gateway进行灰度发布思考

当请求进入到Gateway，对头部的参数进行分析，或者对Body的参数进行分析(比解析头部慢)。如果某个参数满足一定条件，则转发到灰度发布的服务器上。比如按公司ID、IP地址归属地、手机号归属地、版本号或按照一定规则抽取等。转到灰度发布服务器的请求一般不要太大量。

##### Redis分布式锁原理的深入了解

通过以下命令对资源加锁：

```
SET resource_name my_uuid_value NX PX 3000
```

SET NX 命令只会在key不存在的时候给key赋值，PX命令通知Redis保存这个key 3000ms

通过PX设置超时，当资源被锁定超过这个时间时，锁将自动释放，避免死锁产生，但是获得锁的客户端如果没有在这个时间窗口内完成操作，就可能会有其他客户端获得锁，引起争用问题

```lua
if redis.call("get",KEYS[1]) == my_uuid_value
  return redis.call("delete", KEYS[1])
else
  return 0
end
```

通过上述的方式释放锁，能够避免client释放了其他client申请的锁，client的解锁操作只会解锁自己曾经加锁的资源。

my_uuid_value 的随机唯一性，`redis.call("get",KEYS[1]) == my_uuid_value` 在删除锁之前先做判断，可以防止其他命令操作(非本client解锁操作)，把KEYS[1]这个锁key给删除了, my_uuid_value相当于是身份标记

关于value的生成，官方推荐从 /dev/urandom 中取20个byte做为随机数。更简单的是使用`时间戳+客户端编号`的方式产生随机数。

Redis分布式锁的bug情况：

```
如果 Client A 先取得了锁。

其它 Client（比如说 Client B）在等待 Client A 的工作完成。

这个时候，如果 Client A 被挂在了某些事上，比如一个外部的阻塞调用，或是 CPU 被别的进程吃满，或是不巧碰上了 Full GC，导致 Client A 花了超过平时几倍的时间。

然后，我们的锁服务因为怕死锁，就在一定时间后，把锁给释放掉了。

此时，Client B 获得了锁并更新了资源。

这个时候，Client A 服务缓过来了，然后也去更新了资源。于是乎，把 Client B 的更新给冲掉了。
```

简单说就是 A操作超时，释放了锁，B获得了锁更新了资源，之后A操作超时，但不表示操作终止，当A‘缓过来了’，就进行了更新资源操作，则B更新资源的操作就被A覆盖了

解决方案，增加fence(栅栏)技术，就是`乐观锁机制`。

锁服务需要有一个单调递增的版本号

写数据的时候，也需要带上自己的版本号

数据库服务需要保存数据的版本号，然后对请求做检查

如果使用 ZooKeeper 做锁服务的话，那么可以使用 zxid 或 znode 的版本号来做这个 fence 版本号

>需要分清楚：我是用来做修改某个共享源的，还是用来做不同进程间的同步或是互斥的。如果使用 CAS 这样的方式（无锁方式）来更新数据，那么我们是不需要使用分布式锁服务的，而后者可能是需要的。所以，这是我们在决定使用分布式锁服务前需要考虑的第一个问题——我们是否需要？

##### zgrep 使用
`zgrep`帮你不用解压就能grep `.gz`压缩文件

```sh
zgrep "/api" access_log.gz
zgrep "/api" access_log.gz access_log_1.gz
```

延伸：

```sh
zcat access.tar.gz | grep -a '/api'
zgrep -a "/api" access.tar.gz
zcat  解压文件并将内容输出到标准输出
zcmp  解压文件并且 byte by byte 比较两个文件
zdiff 解压文件并且 line by line 比较两个文件
zgrep 解压文件并且根据正则搜索文件内容
ztest - Tests integrity of compressed files.
zupdate - Recompresses files to lzip format.
```

这些命令支持 bzip2, gzip, lzip and xz 格式

##### interface conversion: interface {} is func() string, not string

```go
interface conversion: interface {} is func() string, not string
```

这个报错，是由于给变量赋的值是接口的方法名，而不是字符串。例子：

```go
func (p Person) GetName() string {...}
p := &Person{}
m := make(map[string]string)
m["name"] = p.GetName // Wrong
// 正确的是
m["name"] = p.GetName()
```

##### Golang func()类型变量之间转换
type JobFunc func()

func (j JobFunc) Run() {
	j()
}

JobFunc 这个 adapter 的设计很有技巧。首先，JobFunc也是一种类型，和struct是同等的，类型就是 func()JobFunc 实现了Run 方法，复合Job接口，这样就实现了Job接口，所以可以赋值给Corn

定义一个 j := func(){...具体代码}，j是一个func类型的变量，要将j转为 JobFunc类型，只要 jNow := JobFunc(j)就OK啦！ 这个和 i := 32; i64 := int64(i) 原理是一样的。 因为j的底层类型是fun()和 JobFunc的底层类型func()是一样的，Golang是允许他们可以互相转换的。方法就是 Type() 这样，非常简便。

反过来说，如果两个变量的底层类型是不能互相转换的，就无法使用上述的方法
