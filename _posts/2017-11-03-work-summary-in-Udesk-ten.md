---
layout: post
title: 最近工作总结(十)
date:   2017-11-03 16:28:06
categories: Work
image: /assets/images/post.jpg
---

##### rails find_by 别忘了查询条件
线上出现了一个诡异的bug。查了很久终于发现是find_by查询没有加条件。举个例子：

```ruby
user = User.find_by(user_id)
```

结果导致user获取错误。(我的锅)

查了rails源码：

find_by源码

```ruby
def find_by(arg, *args)
  where(arg, *args).take
rescue ::RangeError
  nil
end
```

find_by就是where查询，加上take

在rails console中执行：

```ruby
User.find_by(1)
# => User Load (3.4ms)  SELECT `users`.* FROM `users` WHERE (1) LIMIT 1
```

得到的是表中的第一个user记录，所以导致了传的｀user_id｀　没有起到查询的作用，SQL语句默认是true而查询了进行了全表查询，之后再LIMIT　1　得到第一条记录

正确的代码是

```ruby
user = User.find_by(id: user_id)
```

Don't do it again

##### redis服务挂了
周末redis服务器挂了，持续了20分钟。影响到了登入后的初始界面，使得初始界面无法访问。初始界面会调的接口，应该没有redis请求的相关逻辑，避免客户连登入都登入不了。
而是哪个模块需要redis，在那个模块的接口再请求redis。如果不可避免需要请求redis，也许要思考redis集群方案，或redis挂了后及时的报警机制，以及时定位问题
单点故障就是像地雷一样

##### 从conn 读取数据时，慎用bufio

good read

```go
func read(conn net.Conn) (string, error) {
  readBytes := make([]byte, 1)
  var buffer bytes.Buffer
  for {
    _, err := conn.Read(readBytes) // conn => readBytes
    if err != nil {
      return "", err
    }
    readByte := readBytes[0]
    if readByte == DELIMITER{
      break
    }
    buffer.WriteByte(readByte) // readByte => buffer
  }
  return buffer.String(), nil
}
```

bad read

```go
func read(conn net.Conn) (string, error) {
  reader := bufio.NewReader(conn)
  readBytes, err := bufio.NewReader(conn)
  if err != nil {
    return "", err
  }
  return string(readBytes[:len(readBytes) - 1]), nil
}
```

在这里，我们队read函数的每一次调用都会导致一个新的针对当前连接的缓冲读取器被创建，我们实际上是在使用不同的缓冲读取器试图从同一个连接上读取数据。这显然会造成一些问题，因为没有任何机制来协调它们读取操作，本应该留给后面的缓冲读取器读取的数据却被提前读取到了前面的缓冲读取器的缓冲区。这不但会导致一些数据块的不完整，甚至还可能会使一些数据块被漏掉。
解决方法也很简单，直接在for代码块之前初始化一个缓冲读取器，并且保证在for循环中总是使用同一个缓冲读取器来读取护具。
