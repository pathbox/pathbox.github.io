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