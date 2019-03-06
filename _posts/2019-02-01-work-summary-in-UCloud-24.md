---
layout: post
title: 最近工作总结(24)
date:  2019-02-01 16:44:06
categories: Work
image: /assets/images/post.jpg
---

### TiDB 通过索引查询和直接扫描 Table 的区别

TiDB实现了全局索引，所以"索引和Table中的数据并不一定在一个数据分片上"，通过索引查询的时候，需要先扫描索引，得到对应的行ID，然后通过行ID去取数据，索引可能会涉及到两次网络请求，会有一定的行难呢过开销。

如果查询涉及到大量的行，那么扫描索引是并发镜像，只要第一批结果已经返回，就可以去取Table的数据，所以这里是一个并行+Pipeline的模式，虽然有两次访问的开销，但是延迟并不会很大。

### 文本协议和二进制协议

文本协议对人友好，二进制协议对机器友好

### 理解覆盖索引

>索引中的列已经满足了查询需求。比如 Table t 上面的列 c 有索引，查询是 select c from t where c > 10;，这个时候，只需要访问索引，就可以拿到所需要的全部数据。这种情况我们称之为覆盖索引(Covering Index)。所以如果很关注查询性能，可以将部分不需要过滤但是需要在查询结果中返回的列放入索引中，构造成组合索引，比如这个例子： select c1, c2 from t where c1 > 10;，要优化这个查询可以创建组合索引 Index c12 (c1, c2)。

### Google是一家大数据公司

数据的收集来源于:

- 从搜索引擎
- 智能家居
- 手机系统
- 无人驾驶汽车
- YouTube
- 等等

### 看完智能时代的总结

现有产业+新技术(大数据和机械智能)=新产业

在未来会有越来越多的工作被机器人取代，包括低级的IT技术工作，这是历史趋势和进步潮流，更是资本主义市场经济的选择，没有人能幸免。所以，现在开始要做好不被机器人取代的准备，等到那时刻真正来临时在做改变，要做到不被淘汰，就太晚了。

### 最大熵原理

这个原理的大意是说，当我们要对未知的事件寻找一个概率模型时，这个模型应当满足我们所有已经看到的数据，但是对未知的情况不要做任何主观假设。在很多领域，尤其是金融领域，采用最大熵原理要比任何人为假定的理论更有效，因此它被广泛地用于机器学习

### 社会的公平

>社会公平只能反映在机会平等上，而不是结果的公平。实际上，只要个人的智力有差异, 努力的程度不同，以及每个人的运气不 同，即使劫富济贫，也无法保证社会公平

### 争当2%的人

>“虽然我们不知道如何在短期内创造 出能消化几十亿劳动力的产业，但是我 们很清楚如何让自己在智能革命中受 益，而不是被抛弃。这个答案很简单，就 是争当2%的人”

>“在历次技术革命中，一个人、一家企 业，甚至一个国家,可以选择的道路只有 两条：要么进入前2%的行列，要么被淘 汰。抱怨是没有用的。至于当下怎么才能 成为这2%，其实很简单，就是踏上智能 革命的浪潮。”

而是希望大家接受 一个新的思维方式，利用好大数据和机 器智能

### 并行计算魔力有限
并行计算能力并不是随机器数成倍增加

“如果在一个任务中能 够并行处理的比例越高，实际的加速越多，但是即便只有5%的计算不能并行, 那么无论使用多少台服务器，实际的加速也不会超过20倍。”
并行计算的时间是远远做不到和服务器数量成反比，事实上，使用的处理器越多，并行计算的效率越低

大数据的另一个挑战是对实时性的要求

### 链表、树数据结构比数组更耗内存

### 理解工作的本质
我们需要理解工作的本质 —— 工作的本质是价值的创造与交换过程，我们在工作中获得更高收入的最好途径就是找到一个快速发发展的平台，100%发挥出自己的潜能。这是男人能实现自我价值，获得自由的最好方式。千万不要把工作理解成一份工资，如果这样，你在贱卖自己最宝贵的不可再生资源：时间。从一开始，就要选择成为合伙人，这首先是一个理念，其次才能成为现实。告诉自己，每一分钟都是为了自己，不是为别人。这样你才会珍惜自己在工作中的每一天，也才不会只被金钱所驱动。

### 五级工程师概念

### 拜占庭问题和共识问题区别
- 拜占庭问题：有一个独立的进程提供一个值，其他的进程来决定是否采取这个值
- 共识问题：每个进程都提议一个值

### TCP滑动窗口原理

- 动画链接：http://www.exa.unicen.edu.ar/catedras/comdat1/material/Filminas3_Practico3.swf
- https://blog.csdn.net/yao5hed/article/details/81046945
- https://www.cnblogs.com/diegodu/p/4538897.html

窗口的概念

发送方的发送缓存内的数据都可以被分为4类:
1. 已发送，已收到ACK
2. 已发送，未收到ACK
3. 未发送，但允许发送
4. 未发送，但不允许发送

其中类型2和3都属于发送窗口。

接收方的缓存数据分为3类：
1. 已接收
2. 未接收但准备接收
3. 未接收而且不准备接收

其中类型2属于接收窗口。

窗口大小代表了设备一次能从对端处理多少数据，之后再传给应用层。缓存传给应用层的数据不能是乱序的，窗口机制保证了这一点。现实中，应用层可能无法立刻从缓存中读取数据。

滑动机制

发送窗口只有收到发送窗口内字节的ACK确认，才会移动发送窗口的左边界。

接收窗口只有在前面所有的段都确认的情况下才会移动左边界。当在前面还有字节未接收但收到后面字节的情况下，窗口不会移动，并不对后续字节确认。以此确保对端会对这些数据重传。

遵循快速重传、累计确认、选择确认等规则。

发送方发的window size = 8192;就是接收端最多发送8192字节，这个8192一般就是发送方接收缓存的大小。

TCP滑动窗口为了解决:`可靠传输以及包乱序（reordering）的问题`

>TCP头里有一个字段叫Window，又叫Advertised-Window，这个字段是接收端告诉发送端自己还有多少缓冲区可以接收数据。于是发送端就可以根据这个接收端的处理能力来发送数据，而不会导致接收端处理不过来

接收端在给发送端回ACK中会汇报自己的AdvertisedWindow = MaxRcvBuffer – LastByteRcvd – 1

>如果网络上的延时突然增加，那么，TCP对这个事做出的应对只有重传数据，但是，重传会导致网络的负担更重，于是会导致更大的延迟以及更多的丢包，于是，这个情况就会进入恶性循环被不断地放大。试想一下，如果一个网络内有成千上万的TCP连接都这么行事，那么马上就会形成“网络风暴”，TCP这个协议就会拖垮整个网络

拥塞控制主要是四个算法：

1.慢启动
2.拥塞避免
3.拥塞发生
4.快速恢复

### 连接的五元组
同一个连接需要五元组来确定（src_ip, src_port, dst_ip, dst_port）和协议(TCP、UDP)


### TCP TIME_WAIT
>关于TIME_WAIT数量太多。从上面的描述我们可以知道，TIME_WAIT是个很重要的状态，但是如果在大并发的短链接下，TIME_WAIT 就会太多，这也会消耗很多系统资源。只要搜一下，你就会发现，十有八九的处理方式都是教你设置两个参数，一个叫tcp_tw_reuse，另一个叫tcp_tw_recycle的参数，这两个参数默认值都是被关闭的，后者recyle比前者resue更为激进，resue要温柔一些。另外，如果使用tcp_tw_reuse，必需设置tcp_timestamps=1，否则无效。这里，你一定要注意，打开这两个参数会有比较大的坑——可能会让TCP连接出一些诡异的问题（因为如上述一样，如果不等待超时重用连接的话，新的连接可能会建不上。正如官方文档上说的一样“It should not be changed without advice/request of technical experts”）。
关于tcp_tw_reuse。官方文档上说tcp_tw_reuse 加上tcp_timestamps（又叫PAWS, for Protection Against Wrapped Sequence Numbers）可以保证协议的角度上的安全，但是你需要tcp_timestamps在两边都被打开（你可以读一下tcp_twsk_unique的源码 ）。我个人估计还是有一些场景会有问题。
关于tcp_tw_recycle。如果是tcp_tw_recycle被打开了话，会假设对端开启了tcp_timestamps，然后会去比较时间戳，如果时间戳变大了，就可以重用。但是，如果对端是一个NAT网络的话（如：一个公司只用一个IP出公网）或是对端的IP被另一台重用了，这个事就复杂了。建链接的SYN可能就被直接丢掉了（你可能会看到connection time out的错误）（如果你想观摩一下Linux的内核代码，请参看源码 tcp_timewait_state_process）。
关于tcp_max_tw_buckets。这个是控制并发的TIME_WAIT的数量，默认值是180000，如果超限，那么，系统会把多的给destory掉，然后在日志里打一个警告（如：time wait bucket table overflow），官网文档说这个参数是用来对抗DDoS攻击的。也说的默认值180000并不小。这个还是需要根据实际情况考虑。
Again，使用tcp_tw_reuse和tcp_tw_recycle来解决TIME_WAIT的问题是非常非常危险的，因为这两个参数违反了TCP协议（RFC 1122）

其实，TIME_WAIT表示的是你主动断连接，所以，这就是所谓的“不作死不会死”。试想，如果让对端断连接，那么这个破问题就是对方的了，呵呵。另外，如果你的服务器是于HTTP服务器，那么设置一个HTTP的KeepAlive有多重要（浏览器会重用一个TCP连接来处理多个HTTP请求），然后让客户端去断链接（你要小心，浏览器可能会非常贪婪，他们不到万不得已不会主动断连接）

也可以增加机器，增加资源进行解决

### 关于排序的几点

- 原因很简单，它还是强调少做事情
- 在一般情况下，快速排序是世界上最快的排序算法
- Nlog(N)是排序算法的极限、边界，没有排序算法的时间复杂度比这个更快
- 在工程上，快速排序算法一般情况下比归并排序快两三倍，因此在工程上还是有意义的
- 假如有一个学区，里面有20000名高中学生，如果让大家到一个超级大的学校上大课，再从中挑出学生中的尖子，效率一定高不了。这就相当于是昨天一开始讲的冒泡排序，每一个人都要和所有人去比。如果我们把2万人放到10所学校中，每所学校只有两千人，从各个学校先各自挑出学习尖子，再彼此进行比较，这就有效得多了。这就是昨天说的归并排序原理。
如果我们先划出几个分数线，根据个人成绩的高低把20000个学生分到十所学校去，第一所学校里的学生成绩最好，第十所最差，再找出学习尖子，那就容易了，工作量最小，这就是快速排序的原理，也是快速排序比归并排序快的原因

### 实际开发中使用AES加密需要注意的地方

服务端和我们客户端必须使用一样的密钥和初始向量IV。
服务端和我们客户端必须使用一样的加密模式。
服务端和我们客户端必须使用一样的Padding模式。

以上三条有一个不满足，双方就无法完成互相加解密。
同时针对对称加密密钥传输问题这个不足：我们一般采用RSA+AES加密相结合的方式，用AES加密数据，而用RSA加密AES的密钥。同时密钥和IV可以随机生成，这要是128位16个字节就行，但是必须由服务端来生成，因为如果由我们客户端生成的话，就好比我们客户端存放了非对称加密的私钥一样，这样容易被反编译，不安全，一定要从服务端请求密钥和初始向量IV。

https://blog.csdn.net/lrwwll/article/details/78069013

### go module 快速使用

1. 首先将你的版本更新到最新的Go版本1.11，如何更新版本可以自行百度。
2. 通过go命令行，进入到你当前的工程目录下，在命令行设置临时环境变量export GO111MODULE=on；
3. 执行命令`go mod init`在当前目录下生成一个`go.mod`文件，执行这条命令时，当前目录不能存在`go.mod`文件。如果 `go mod init`报错，则执行`go mod init app_name`, 如果之前生成过，要先删除；如果你工程中存在一些不能确定版本的包，那么生成的go.mod文件可能就不完整，因此继续执行下面的命令；
4. 执行`go mod tidy`命令，它会添加缺失的模块以及移除不需要的模块。执行后会生成go.sum文件(模块下载条目)。添加参数-v，例如`go mod tidy -v`可以将执行的信息，即删除和添加的包打印到命令行；
5. 执行命令`go mod verify`来检查当前模块的依赖是否全部下载下来，是否下载下来被修改过。如果所有的模块都没有被修改过，那么执行这条命令之后，会打印all modules verified。
6. 执行命令`go mod vendor`生成vendor文件夹，该文件夹下将会放置你go.mod文件描述的依赖包，文件夹下同时还有一个文件modules.txt，它是你整个工程的所有模块。在执行这条命令之前，如果你工程之前有vendor目录，应该先进行删除。同理go mod vendor -v会将添加到

```
go mod init
       tidy
       verify
       vendor
```

### 别太在意感受，在意工作表现

### OKR - 具有挑战的目标，基于 “蹦一下才能碰得到” 的原则

### 学习需要系统化
学习需要系统化，做学问不能全靠查资料和主观的演绎推理，那只是对已有知识的诠释，而只有实验科学才能获得新知识

系统的学习有助于在学习者的心中搭建一整套框架，由点到线、由线到面，再由面到体，形成网状的、三维的看待问题的视角。学习系统化了，我们发现问题、解决问题的思路就容易活络起来。

### 物联网设备的安全问题

越来越多的家用电器，可以连接互联网，但是它们的网络安全却没有得到足够重视。其中一个问题是，当这些家用电器丢弃的时候，别人可以把这些东西捡回去，轻易读取里面包含的信息，其中最常见的就是 WiFi 网络的密钥。

所以，比较安全的做法是，物联网设备只连接专门的子网，或者定期更换密码

### MD5已经完全不再安全
一个小于8位的MD5长度密码已经能被破解软件在十几秒内破解出来

### 对资源系统的理解
资源系统是一个抽象层，统一管理各种资源

各种资源可以各不相同，但有唯一对应的资源id和资源系统关联

资源系统统一调度操作资源，在资源系统中有记录，才展示给用户，即使在这个资源具体表中有记录，也以资源系统为准

资源系统的统一管理，避免了资源种类、产品种类变多时，没有管理和各自分散显示对代码和架构上的复杂度急剧增加

但资源系统时一个独立的系统，和具体产品的系统之间通过http接口沟通，资源系统使用独立的数据库，这样，避免不了的会出现资源系统和产品系统之间数据不一致的情况

要努力使用各种方法，降低数据不一致的情况

资源系统的查询要尽可能的快

### 掌握更多的工具，弥补智力的不足

### redis 一个key或value大小最大是512M
- The maximum allowed key size is 512MB
- A value can't be bigger than 512MB

### 什么场景下用到runtime.LockOSThrea
```
golang的scheduler可以理解为公平协作调度和抢占的综合体，他不支持优先级调度。当你开了几十万个goroutine，并且大多数协程已经在runq等待调度了, 那么如果你有一个重要的周期性的协程需要优先执行该怎么办？
可以借助runtime.LockOSThread()方法来绑定线程，绑定线程M后的好处在于，他可以由system kernel内核来调度，因为他本质是线程了。  
先前我们有在定时器场景中使用runtime.LockOSThread，达到少许的优先级效果。效果不明显的原因是，我们自定义的定时器需要time.sleep来解决cpu忙讯轮，但time.sleep又依赖于go自身的heap定时器…. 解决方法是，独立一个M线程后，使用syscall来实现时间等待. 

总结:
runtime.LockOSThread会锁定当前协程只跑在一个系统线程上，这个线程里也只跑该协程。他们是相互牵制的 
```

### 提高效率的方法：少做"事"

无论是工作还是设计程序、设计算法，提高效率的方法都是：少做无用功，那些可以不用做的事情就不做

比如排序，很多问题可以通过排序解决，但是效率太低，因为所有数据都被排序了。也许，你需要的只是比较，而不是真的排序