---
layout: post
title: 最近工作总结(十八)
date:   2018-08-01 10:31:06
categories: Work
image: /assets/images/post.jpg
---

### 关于文档和配置文件

最近重构一个老项目，被坑了，一些重要信息存在配置文件中，而配置文件要么没有在代码库中，要么缺少了重要信息，以此总结：

- 没有开发文档，没有架构的需求文档
- 配置文件内容陈旧，和生产环境不一致或丢失重要配置内容
- 配置文件建议也放到代码库中，或者写.example文件保存，敏感安全信息直接用xxx表示即可

### JWT是什么

https://jwt.io/

JWT是一个字符串，由三部分组成

```ruby
header = {
  'typ': 'JWT',
  'alg': 'HS256'
}

# 标准中注册的声明 (建议但不强制使用)
# iss: jwt签发者
# sub: jwt所面向的用户
# aud: 接收jwt的一方
# exp: jwt的过期时间，这个过期时间必须要大于签发时间
# nbf: 定义在什么时间之前，该jwt都是不可用的.
# iat: jwt的签发时间
# jti: jwt的唯一身份标识，主要用来作为一次性token,从而回避重放攻击。


# 传你需要的数据
payload = {
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}

sign = HMACSHA256(  # HS256 算法
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  your-256-bit-secret
)

# javascript
var encodedString = base64UrlEncode(header) + '.' + base64UrlEncode(payload);

var signature = HMACSHA256(encodedString, 'secret');
```

JWT_token = base64(header) + "." + base64(payload) + "." + sign

这样就可以将JWT_token值直接放在url或头部中进行传递

- sign 的作用：对接口请求的鉴权，并防止header和payload被篡改
- 不要传敏感信息，JWT对payload是base64编码，是可以反编码的，并没有加密，所以是不安全的
- JWT的使用场景：单点登入，用户认证和授权系统

https://www.jianshu.com/p/576dbf44b2ae

### MySQL导出数据为excel文件

mysql链接信息 数据库 用户名 密码 然后执行查询语句，定向输出

```
mysql database -h127.0.0.1 -P3306 -uroot -p -e "select * from users" > sql_data.xls
```

这样的缺点就是，如果查询语句耗时很长，会导致慢查询或查询阻塞，如果是线上操作，就可能影响到线上的MySQL服务

### HashMap、Map是无序的

HashMap、Map是无序的，当你使用它们发生诡异的事情时，也许就是因为无序的属性，而你的代码逻辑却按照有序的往下走而导致的

### OpenSSL rand for hex string

openssl rand -hex 20

快速创建随机hex字符串哟

### 系统性能优化建议

性能优化离不开的一些套路：异步、去锁、复用、零拷贝、批量等

### Redis 作为消息队列的Broker

Redis作为消息队列框架或多任务异步队列框架的Broker，往往使用的是Redis的 List 数据结构。无论这个消息消息队列框架功能多么复杂强大，目的都是将参数或message序列化后，
在publisher端：

```
"RPUSH" queueName msg
```

在consumer端不断循环执行：

```
"BLPOP" queueName 1 // 1 只是一个超时时间参数，表示会在1s的时间内，阻塞从queueName读取数据，读取到数据则返回，1s内没读取到数据，超时返回
```

### Golang 链式方法调用

```go

type item struct{}

A() *item { return *item}
(s *item) B() *item { return *item}
(s *item) C() *item { return *item}

A().B().C()
```
Golang的链式方法调用，需要前面链接的方法返回相同的struct

### 读写锁(UNIX环境高级编程)

读写锁与互斥量类似，不过读写锁允许更高的并行性。互斥量要么是锁住状态，要么就是不加锁状态，而且一次只有一个线程可以对其加锁。读写锁可以有3种状态：

- 读模式下加锁状态
- 写模式下加锁状态
- 不加锁状态

一次只有一个线程可以占有写模式的读写锁，但是可以多个线程同时占有读模式的读写锁。

当读写锁是写加锁状态时，在这个锁被解锁之前，所有试图对这个锁加锁的线程都会被阻塞（无论是加读锁还是写锁）。当读写锁在读状态时，所有试图以读模式对它进行加锁的线程都可以得到访问权，但是任何希望以写模式对此锁进行加锁的线程都会阻塞，直到所有的线程释放它们的读锁为止。虽然各个操作系统对读写锁的实现各不相同，当读写锁处于读模式锁住的状态，而这时有一个线程试图以写模式获取锁时，读写锁通常会阻塞随后的度模式锁请求。这样可以避免读模式锁长期占用，而等待的写模式锁请求一直得不到满足。

读写锁非常适合于对数据结构读的次数远大于写的情况。当读写锁在写模式下时，它所保护的数据结构就可以安全地被修改，因为一次只有一个线程可以在写模式下拥有这个锁，当读写锁在读模式下时，只要线程先获取了读模式下的读写锁，该锁保护的数据结构就可以被多个获得读模式锁的线程读取

读写锁也叫做共享互斥锁。当读写锁是读模式锁住时，就可以说成是共享模式锁住，当它是写模式锁住时，就可以说是互斥模式锁住的

带超时的读写锁，互斥锁 可以避免死锁情况，避免永久阻塞状态

写锁还是互斥锁，读锁是共享读模式锁。写锁请求优先级大于之后来的读锁。