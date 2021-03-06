---
layout: post
title: 最近工作总结(23)
date:  2019-01-07 10:20:06
categories: Work
image: /assets/images/post.jpg
---

### Hash Func result length and bit size

MD5 hexdecimal(hex) string length is 32, MD5 result data bit size is 128 bits. One hex string char is 4 bits.

MD5 hashes are 128 bits in length and generally represented by 32 hex digits.

As we know SHA-1 hashes are 160 bits in length and generally represented by 40 hex digits.

SHA-2 hashes are 256bit in length and generally represented by 64 hex digits

So SHA-512 can be represented by 128 hex digits

### MySQL 为order by 使用索引

KEY a_b_c (a,b,c)

order by 能使用最左前缀索引:
- ORDER BY a
- ORDER BY a,b
- ORDER BY a,b,c
- ORDER BY a DESC, b DESC, c DESC

如果WHERE使用索引的最左前缀定义为常量，则ORDER BY 能能使用索引:
- WHERE a = const ORDER BY b, c
- WHERE a = const AND b = const ORDER BY c
- WHERE a = const AND b > const ORDER BY b, c

不能使用索引进行排序
- ORDER BY a ASC, b DESC, c DESC 排序不一致
- WHERE g = const ORDER BY b, c 丢失a索引
- WHERE a = const ORDER BY c 四队b索引
- WHERE a = const ORDER BY a,d d不是索引的一部分
- WHERE a in (...) ORDER BY b,c 对于排序来说，多个相等条件也是范围查询

### Docker的RUN、CMD

RUN 有两种格式：
1. Shell 格式：RUN 
2. Exec 格式：RUN ["executable", "param1", "param2"] 

RUN会在镜像上构造新的一层

```
RUN apt-get update && apt-get install -y \  
 bzr \
 cvs \
 git \
 mercurial \
 subversion
```

此命令会在容器启动且 docker run 没有指定其他命令时运行。
1. 如果 docker run 指定了其他命令，CMD 指定的默认命令将被忽略。 
2. 如果 Dockerfile 中有多个 CMD 指令，只有最后一个 CMD 有效。 

CMD 有三种格式：
1. Exec 格式：CMD ["executable","param1","param2"] 这是 CMD 的推荐格式。 
2. CMD ["param1","param2"] 为 ENTRYPOINT 提供额外的参数，此时 ENTRYPOINT 必须使用 Exec 格式。 
3. Shell 格式：CMD command param1 param2  

### Be a coach, not a leader

不要想去成为团队的Leader，而是去成为团队的Coach

教练领导者：是镜子，是向导

有效对话：层层深入的提问 不断地问问题 直到员工把最深层次的问题告诉你

### pm2 简单命令教程

```
pm2 start my_server(go编译后的二进制文件) --name=my_server -i 0

--name=my_server 命名为my_server
-i 0  根据CPU核数启动进程个数

最简单的启动就是： pm2 start my_server

pm2 start app.js --watch      # 当文件变化时自动重启应用，相当于热reload功能
pm2 start script.sh           # 启动bash脚本
pm2 list/l                    # 列表 PM2 启动的所有的应用程序
pm2 monit(monit id/name)      # 显示每个/某个应用程序的CPU和内存占用情况
pm2 show(id/name)             # 显示应用程序的所有信息
pm2 logs(id/name)             # 显示所有应用程序的日志
pm2 stop all                  # 停止所有的应用程序
pm2 stop(id/name)             # 停止 id为 0的指定应用程序
pm2 restart all               # 重启所有应用
pm2 reload all                # 重启 cluster mode下的所有应用
pm2 delete all                # 关闭并删除所有应用
pm2 delete 0                  # 删除指定应用 id 0
pm2 scale api 10              # 把名字叫api的应用扩展到10个实例pm2 reset [app-name]          # 重置重启数量
pm2 startup                   # 创建开机自启动命令
pm2 save                      # 保存当前应用列表
pm2 dump                      # 保存当前应用列表
pm2 resurrect                 # 重新加载保存的应用列表
pm2 start app.js --max-memory-restart 20M #内存使用超过上限自动重启 可以加上--max-memory-restart参数

pm2 startup
pm2 save    来开机启动监控的程序
```

### 伪工作者的特征

- 那些不能给公司带来较大收益，又不能给用户带来价值的改进和“升级”，很多是伪工作。比如，在互联网行业里，如果一个产品中某些功能或者设计上线之后生命周期不到三个月，那么当初很多开发的工作都是伪工作

- 有的人明明能够通过学习一种新技能更有效地工作，却偏偏要守着过去的旧工作，甚至手工操作，这种人事典型的伪工作者

- 在做事情前不认真思考，做事时通过简单的试错方法(trial and error)盲目寻找答案

- 做产品不讲究质量、不认真测试、上线后不停修补，总是花费很多的时间和精力找漏洞修bug

- 不注重用有限的资源解决95%的问题，而是把大部分时间和精力用于纠结不重要的5%的问题

- 每次开会找来大量不必要的人员旁听，或者总去参加那些不必要参加的会议

不想要成为伪工作者，逆向思维，不要做上述的几个事情

### 突破职业天花板之说

>我觉得解决办法就是自我的通识教育。我们常常把那些能够在职场上不断提升的人称为“有后劲儿”。那么有后劲儿的人和那些很快在职场上遇到天花板的人相比有什么不同呢？一个非常重要的差别是，有后劲儿的人有着更广的视野，而这种视野常常来自良好的博雅教育。在美国，一工作就收入比较高的是那些功课学院的毕业生，而像哈佛大学或者普林斯顿大学这样顶级名校的毕业生一开始工作的时候，收入相对要少很多。因为他们所接受的那些人文教育、博雅教育不是直接的工作技能。但是，如果再看一下10年后大家的收入就会发现，名牌大学出来的那些有着良好人文教育背景的人，后来者居上，而且社会地位提高更快，也就是说他们更容易突破天花板。其实，这些人在大学里追求的是类似帝道和王到

-- 吴军《见识》


### 微服务结点之间的三种通信方式
- http
- RPC  
- MQ消息队列

### Man-in-the-middle_attack

理解中间人攻击: 中间人替换了Alice的Bob公钥为自己的公钥,之后就能假扮为Bob给Alice发送数据，而Alice以为是Bob发的

```
1.Alice sends a message to Bob, which is intercepted by Mallory:
Alice "Hi Bob, it's Alice. Give me your key." →     Mallory     Bob

2.Mallory relays this message to Bob; Bob cannot tell it is not really from Alice:
Alice     Mallory "Hi Bob, it's Alice. Give me your key." →     Bob

3.Bob responds with his encryption key:
Alice     Mallory     ← [Bob's key] Bob

4.Mallory replaces Bob's key with her own, and relays this to Alice, claiming that it is Bob's key:
Alice     ← [Mallory's key] Mallory     Bob

5.Alice encrypts a message with what she believes to be Bob's key, thinking that only Bob can read it:
Alice "Meet me at the bus stop!" [encrypted with Mallory's key] →     Mallory     Bob

6.However, because it was actually encrypted with Mallory's key, Mallory can decrypt it, read it, modify it (if desired), re-encrypt with Bob's key, and forward it to Bob:
Alice     Mallory "Meet me at the van down by the river!" [encrypted with Bob's key] →     Bob

7.Bob thinks that this message is a secure communication from Alice.
8.Bob goes to the van down by the river and gets robbed by Mallory.
9.Alice does not know that Bob was robbed by Mallory thinking Bob is late.
10.Not seeing Bob for a while, she determines something happened to Bob.
```
>https://en.wikipedia.org/wiki/Man-in-the-middle_attack

### 不学的还是不学，想去上课的还是会去上课

### 开发接口文档，越简单越好，越无脑越好

### 上帝喜欢笨人

原因很简单，上帝不喜欢比自己聪明的人。这其实反映了一个人是否对自己的能力和本领有正确的认识。

### 消息队列在日志收集服务中

在很多日志收集服务中会使用到消息队列，日志消息先到消息队列，消息队列在传给服务写库。
当你要修改或更换日志收集服务数据库时，可以将写库的日志收集服务停服，日志消息不会丢失，消息队列会缓存停服这段时间的日志消息，待处理完，重新启动服务后，消息队列会将日志消息再发送给写库服务，你可以安心的停服处理完事务，bingo!

### 本地环境测试和服务器环境测试永远不同
本地环境测试和服务器环境测试永远不同，常常在本地测试是正常的，布到服务器后就发生了奇妙的事情，往往不是代码逻辑的问题，可以从配置文件，环境变量，文件目录权限等方面排查

### OKR息息相关的五个问题

1. 结构和清晰度：我们团队的目标、角色和执行计划是清晰而明确的吗？
2. 心理安全：我们能够感到安全而且从容地在这个团队中程度风险吗？
3. 工作的意义：我们是否在做一些对我们每个人都很重要的事情？
4. 可靠性：我们能彼此信赖并按时完成高质量的工作吗？
5. 工作的影响：我们是否发自内心地认为我们做的工作是真正有意义的？

### 这才是敏捷开发(Agile development)

敏捷开发是一种以人为核心、迭代、循序渐进的开发方法。在敏捷开发中，软件项目的构建被切分成多个子项目，各个子项目的成果都经过测试，具备集成和可运行的特征。换言之，就是把一个大项目分为多个相互联系，但也可独立运行的小项目，并分别完成，在此过程中软件一直处于可使用状态。

然而，国内很多人误解了敏捷开发，觉得敏捷开发就是开发速度快，其实这不是敏捷开发的本质。敏捷开发，是一个人或几个人将开发、测试、部署、甚至前后端都在一个开发流程中完成，而不依赖其他部门，极大减少了部门之间无意义的沟通延迟时间，有的牛人是一个人将所有都完成了。但是，对软件或代码的质量，测试的质量标准并没有下降，而不是为了速度而放弃系统的质量.
