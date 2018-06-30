---
layout: post
title: 最近工作总结(十七)
date:   2018-06-26 20:45:06
categories: Work
image: /assets/images/post.jpg
---

### 初识数据库的WAL

>预写式日志（Write-ahead logging，缩写 WAL）是关系数据库系统中用于提供原子性和持久性（ACID属性中的两个）的一系列技术。在使用WAL的系统中，所有的修改在提交之前都要先写入log文件中

>log文件中通常包括redo和undo信息。假设一个程序在执行某些操作的过程中机器掉电了。在重新启动时，程序可能需要知道当时执行的操作是成功了还是部分成功或者是失败了。如果使用了WAL，程序就可以检查log文件，并对突然掉电时计划执行的操作内容跟实际上执行的操作内容进行比较。在这个比较的基础上，程序就可以决定是撤销已做的操作还是继续完成已做的操作，或者是保持原样。

1、修改记录前，一定要先写日志；

2、事务提交过程中，一定要保证日志先落盘，才能算事务提交完成。

MySQL中，存储引擎实现事务的通用方式是基于 redo log 和 undo log。

简单来说，redo log 记录事务修改后的数据, undo log 记录事务前的原始数据

开启了 binlog 的事务执行：
- 先记录 undo/redo log， 确保日志刷到磁盘上持久存储
- 更新数据记录，缓存操作并异步刷盘
- 将事务日志持久化到binlog
- 提交事务，在redo log中写入commit记录
这样，只要binlog没有写成功，整个事务是需要回滚，而binlog写成功后及时MySQL Crash了，都可以恢复事务并完成提交

关于WAL性能优化问题：

WAL机制一方面是为了确保数据即使写入缓存丢失也可以恢复，另一方面是为了集群之间异步复制。默认WAL机制开启且使用同步机制写入WAL。首先考虑业务是否需要写WAL，通常情况下大多数业务都会开启WAL机制（默认），但是对于部分业务可能并不特别关心异常情况下部分数据的丢失，而更关心数据写入吞吐量，比如某些推荐业务，这类业务即使丢失一部分用户行为数据可能对推荐结果并不构成很大影响，但是对于写入吞吐量要求很高，不能造成数据队列阻塞。这种场景下可以考虑关闭WAL写入，写入吞吐量可以提升2x~3x。退而求其次，有些业务不能接受不写WAL，但可以接受WAL异步写入，也是可以考虑优化的，通常也会带来1x～2x的性能提升。

优化推荐：根据业务关注点在WAL机制与写入吞吐量之间做出选择

1. 同步WAL，保证数据安全和数据一致性，吞吐量最差
2. 异步WAL，降低了数据安全和数据一致性，吞吐量提升
3. 关闭WAL，数据安全和数据一致性没有保证，吞吐量最大

参考链接：

http://m.blog.itpub.net/15498/viewspace-2134411/

http://hbasefly.com/2016/12/10/hbase-parctice-write/

### gorilla/websocket 的Write要加锁,解决并发写问题

gorilla/websocket 的write操作在高并发时会有报错导致write操作失败，解决方式是加锁。例子：

```go
func (socket *TSocket) WriteMessage(message []byte) error {
	socket.Lock()
	defer socket.Unlock()
	err := socket.Conn.WriteMessage(TextMsg, message)
	if err != nil {
		socket.Logger.Error("TSocket WriteMessage Error", err)
	}
	return err
}

func (s *TSocket) Write(b []byte) (n int, err error) {
  s.Lock()
  defer s.Unlock()

  var w io.WriteCloser
  if w, err = s.Conn.NextWriter(websocket.BinaryMessage); err == nil {
    if n, err = w.Write(b); err == nil {
      err = w.Close()
    }
  }
  return
}
```

### git push --force-with-lease

不要用 `git push --force`，而要用 `git push --force-with-lease`代替。在你上次提交之后，只要其他人往该分支提交代码，`git push --force-with-lease` 会拒绝覆盖