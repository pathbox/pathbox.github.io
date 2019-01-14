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
