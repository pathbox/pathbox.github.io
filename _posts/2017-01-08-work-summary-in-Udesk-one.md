---
layout: post
title: 最近工作总结(一)
date:   2017-01-09 18:24:06
categories: Work
image: /assets/images/post.jpg
---

##### ES 慢查询优化
对搜索客户手机号码的ES慢查询进行优化。由于表的记录已经达到一定数量级，对手机号码进行char_split 的分词方法，
会产生性能问题，因为该分词方法会把手机号码进行拆分组合，这里很耗时。
优化方法：用正则表达式检验出传的参数是手机号码，不使用char_split分词查询，而是使用完全匹配的方式进行过滤

##### gorm 的连接池方案
Go项目中使用了gorm，打算使用连接池。定义了一个全局变量：

```go
var MyDB *gorm.DB

func init() {
    var err error
    dsn := fmt.Sprintf(
      "%s:%s@tcp(%s:%d)/%s?charset=utf8&parseTime=True&loc=Local",
      config.DBUser(),
      config.DBPassword(),
      config.DBHost(),
      config.DBPort(),
      config.DBName(),
    )

    MyDB, err = gorm.Open("mysql", dsn)
    // 连接池
     if err == nil {
       MyDB.DB().SetMaxIdleConns(20)
       MyDB.DB().SetMaxOpenConns(50)
       MyDB.DB().Ping()
       MyDB.LogMode(true)
     } else {
       log.Panic("Gorm Open Error")
     }

```

需要使用到数据库的地方，则导入该包，然后直接使用这个全局变量就可以了。这样就只初始化了一个数据库实例，可以给需要用的地方使用。
不用每次都新建数据库实例的连接
使用的地方，不要使用Close()。
因为这样会把这个唯一的数据库实例给关了。
关于数据库连接池，配置好后，sql查询会自动的使用这个连接池。对产生rows的sql查询需要手动rows.Close()，其他查询不需要。
使用show processlist 命令观察，明白了Go MySQL包连接池的原理。
首先，会创建一个连接池，并且其中创建一个连接。如果这个连接足够应对服务器，则不会有新的连接被创建，当服务器负载或并发变大，
连接池会创建新的连接，一直可以创建到最大连接数量，这里是500.当这一轮访问过去了，服务器负载变小了。数据库的连接会被关闭，
但不是全部关闭，而是保留连接池配置的连接数量，这里是200.也就是连接池中有200个连接提供服务。要注意的是，gorm最大连接数配置不能超过
MySQL中设置的最大连接数，当超过时，会以MySQL中的设置为准。
也就是 实际的最大连接数 <= gorm配置的最大连接数 <= MySQL配置的最大连接数

##### 使用Go context

```go
 body, err := ioutil.ReadAll(r.Body)
 CheckErr(err, w)
 params := string(body)
 url := r.URL.String()
 log.Println("URL Info: ", r.Method, r.URL.Scheme, url, " from: ", r.RemoteAddr)
 if params != "" {
    ctx := context.WithValue(r.Context(), "params", body) // Use context to pass params to action
   log.Println("Request Params: ", params)
   next.ServeHTTP(w, r.WithContext(ctx))
 } else {
   next.ServeHTTP(w, r)
 }

 body := req.Context().Value("params").([]byte)

```

ioutil.ReadAll(r.Body) 只能读取一次，这一次读完了，相同的请求再读就是空的了(比如在下一个middleware中读取时)

所以，用context的方法，将这个值进行临时保存和传递

##### Magic Number

避免代码中出现Magic Number。定义合理的常量来代替Magic Number 的出现

##### Don't use DateTime.parse

```ruby
T = "2017-01-20 12:00:00"
DateTime.parse(T)  #=> Fri, 20 Jan 2017 12:00:00 +0000
Time.parse(T)      #=> 2017-01-20 12:00:00 +0800
```

DateTime.parse 解析的结果没有时区，存到数据库的话，会多加8个小时(在中国)，这样在数据库的值就变为了“2017-01-20 20:00:00”
并不是我们想要的结果，如果用Time.parse 的话，就会帮你自动转时区了
