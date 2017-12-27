---
layout: post
title: 最近工作总结(十一)
date:   2017-12-06 16:55:06
categories: Work
image: /assets/images/post.jpg
---

##### Go chan 类型需要初始化操作

如果你使用Go chan 类型,需要进行初始化操作.
例子:

```go
type Socket struct {
  message chan []byte
  close chan struct{}
  conn *websocket.Conn
}

socket := &Scoket{}

for {
  select {
  case msg := <- socket.message:
  case <-socket.close:
  default:
  }
}

这个`select`不会起任何作用.因为 socket没有为chan初始化值,内存里是没有这个指针值的

正确的姿势:

socket := &Scoket{
  message: make(chan []byte),
  close: make(chan struct{}),
}

for {
  select {
  case msg := <- socket.message:
  case <-socket.close:
  default:
  }
}

这样就能正确使用了

```

##### Rails 5 belongs_to foreign_id must be not blank

Rails 5 中

```ruby
class User < ApplicationRecord
  belongs_to :school
  belongs_to :city
end

user = User.new
user.save!

# ActiveRecord::RecordInvalid: 验证失败: School必须存在, City必须存在
# 外键id必须要有值
```

new_framework_defaults.rb

```ruby
Rails.application.config.active_record.belongs_to_required_by_default = false
```
这样就不会进行sql查询,不会强制存在了

或者

```ruby
class User < ApplicationRecord
  belongs_to :school, optional: true  # required: true
  belongs_to :city, optional: true
end
```

##### 两个项目的Sidekiq之间的通信技巧

Sidekiq： 项目A

Sidekiq： 项目B

设计A作为生成者，B作为消费者。由于B中有大量业务逻辑，不想在A中又实现一遍，而是通过Sidekiq，在A中将参数存到B的redis中，B再从中取出进行“消费”

```ruby
B_SIDEKIQ_REDIS = Redis::Namespace.new(B_SIDEKIQ_NAMESPACE, redis: B_REDIS)
B_SIDEKIQ_REDIS_POOL = ConnectionPool.new(timeout: 1) { B_SIDEKIQ_REDIS }
```

Sidekiq 源码中使用到的池是`ConnectionPool.new`, 所以这里实例化了一个全局的池`B_SIDEKIQ_REDIS_POOL`

```ruby
# A项目中
# update_user_worker.rb

class UpdateUserWorker
  include Sidekiq::Worker
  sidekiq_options queue: :default, retry: false, pool: B_SIDEKIQ_REDIS_POOL
end

#代码某处
UpdateUserWorker.perform_async(1, 'Jerry')


# B 项目中
# update_user_worker.rb
class UpdateUserWorker
  include Sidekiq::Worker
  sidekiq_options queue: :default, retry: false

  def perform(user_id, name)
    user = User.find user_id
    user.update!(name: name)
  end
end
```

在两个项目中创建相同的worker: `update_user_worker.rb`

A项目中，`sidekiq_options`选项进行`pool`的配置，替换原来默认的pool，而使用B项目的redis

B项目中，具体实现 `perform`方法逻辑

原理简释: 在A中通过Sidekiq将参数存到B的redis中，B的Sidekiq从redis中取出参数，进行`消费`操作

这样看，redis其实也可以使用相同的redis。只要生产者Sidekiq存的值使用的redis，消费者能够从中取出值就可以了，很简单，但在Rails项目中很实用的一个技巧

##### 图数据库简记

适合 社交网络图谱 企业关系图谱

更大深度的关联关系, 关系数据库当关联深度超过2时会有现严重的性能问题(join超过3个表性能急剧下降)

缺点:

+ 记录大量基于事件的数据（例如日志条目或传感器数据）

+ 对大规模分布式数据进行处理，类似于Hadoop

+ 二进制数据存储

+ 适合于保存在关系型数据库中的结构化数据

##### systemd 对每个服务的文件描述符限制
Ubuntu 16.04之后使用了`systemd`。 `systemd`对监听的服务有文件描述符的限制，默认限制是1024。所以，如果要使用`systemd`，服务需要更多的文件描述符，达到高性能，高并发。需要修改这个配置

```
[Service]
LimitCORE=infinity
LimitNOFILE=100000
LimitNPROC=100000
```

##### nginx-wss 配置例子

```conf
upstream my_app {
  ip_hash;
  server server_ip_a:7001;
  server server_ip_b:7002;
  server server_ip_c:7003;
  keepalive 512;
}

server {
  listen 7001;
  listen 7004 ssl; # 使用https/wss的端口

  ssl_certificate /srv/nginx-wss/conf/app.server.crt;
  ssl_certificate_key /srv/nginx-wss/conf/private.pem;

  proxy_read_time 120;

  location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-NginX-Proxy true;

    add_header Access-Control-Allow-Methods "POST, GET, OPTIONS";
    add_header Access-Control-Allow-Headers "x-requested-with,content-type";

    proxy_pass http://my_app/;
    proxy_redirect off;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

##### Rails 项目中集中管理使用多个MySQL Database

有时候，一个Rails Application需要访问不同的MySQL Database，并且不同的Model，访问不同的MySQL Database，ActiveRecord对此是支持的。

关键的方法： `establish_connection(config = nil)`

+ 临时MySQL连接

```ruby
ActiveRecord::Base.establish_connection(
  adapter:  "mysql2",
  host:     "localhost",
  username: "myuser",
  password: "mypass",
  database: "somedatabase"
)
```

+ 集中管理方式

monkey patch ActiveRecord::Base

```ruby
class ActiveRecord::Basae
  self.develompent
    self.establish_connection :development
  end

  self.mysql_default
    self.establish_connection :default
  end

  self.mysql_database_a
    self.establish_connection :mysql_a
  end

  self.mysql_database_b
    self.establish_connection :mysql_b
  end
end
```

这里 establish_connection 后面加的参数是 symbol，在你的databases.yml 文件中，要配置对应的symbol名称的MySQL配置。这样就会自动读取对应的symbol为key 的MySQL配置

然后， products 表来自 mysql_a, colors 表来自 mysql_b

```ruby
# product.rb
class ApplicationRecord < ActiveRecord::Base
end

class Product < ApplicationRecord
  self.mysql_database_a
end

class Cplor < ApplicationRecord
  self.mysql_database_b
end
```

相当于手动重新进行了MySQL实例的连接操作。如果没有手动，ActiveRecord::Base 默认是根据不同的环境 development，production，staging，test，在databases.yml文件中，读取相应的MySQL实例配置。

##### go tips

select语句，把CSP模式用到了极致

```go
select {
case msg = <-c.memoryMsgChan: // 尝试从内存中读取
case buf = <-c.backend.ReadChan: // 尝试从磁盘读取
    msg, err = decodeMessage(buf)
    if err != nil {
      c.ctx.nsqd.logf("ERROR: failed to decode message - %s", err)
      continue
    }
}
```

ReadFrom's sendFile

```go

// http模块对于文件只是简单地直接打开，获取文件描述符（file descriptor)
// http模块调用io.Copy函数，io.Copy函数开始检查Reader Writer是否特殊的ReadFrom，WriteTo接口

func (c *TCPConn) ReadFrom(r io.Reader) (int64, error) {
  if n, err, handled := sendFile(c.fd, r); handled {
    if err != nil && err != io.EOF{
      err = &OpError{Op: "read", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return n, err
  }
}
```

fasthttp对于header的处理

```go
//fasthttp为什么会比net.http快，其中一个原因就是fasthttp并没有完全地解析所有的http请求header数据。这种做法也称作lazy loading

type RequestHeader struct {
        //...
    contentLength      int
    contentLengthBytes []byte

    method      []byte
    requestURI  []byte
    host        []byte
    contentType []byte
    userAgent   []byte

    h     []argsKV
    bufKV argsKV

    cookies []argsKV

    rawHeaders []byte
}

//从[]byte转化为string时是有copy和alloc行为的。虽然数据量小时，没有什么影响，但是构建高并发系统时，这些小的数据就会变大变多，让gc不堪重负

type argsKV struct {
    key   []byte
    value []byte
}

// net.http中使用了map[string]string来存储各种其他参数，这就需要alloc，为了达成zero alloc，fasthttp采用了遍历所有header参数来返回

```

##### websocket 对网络稳定性的依赖
websocket 对网络稳定性的依赖比想象中的要大，所以，设计合理的重连机制变得如此重要.
然而,设计重连机制,就要对新旧连接直接的状态和上下文进行合理的处理.是进行无缝连接,还是作为新的连接状态

##### Redis Pub/Sub 简单分析

+ 消息发布者（Pub）

即publish客户端，无需独占connection，可以在publish消息的同时，使用同一个redis-client connection进行其他操作，例如：INCR等

+ 消息订阅者（Sub）

即subscribe客户端，需要独占connection，即进行subscribe期间，热第三client无法穿插其他操作，此时client以阻塞的方式等待`publish端`的消息。因此subscribe端需要使用单独的connection，甚至需要在额外的线程中使用。一旦subscribe端断开connection，将失去connection失效期的所有消息。

流程：

1. subscribe端首先向一个Set集合中增加“订阅者ID”，此Set集合保存了“活跃订阅者”，订阅者ID标记每一个唯一的订阅者，例如：sub：email， sub：web。此Set称为“活跃订阅者集合”

2. subscribe端开启订阅操作，并基于Redis创建一个以“订阅者ID”为Key的List数据结构，此List中存储了所有尚未消费的信息。此List称为“订阅者消息队列”

3. publish端 每发布一条消息后，publish端都需要遍历“活跃订阅者集合”，并依次向每个“订阅者消息队列”尾部追加此次发布的消息。

4. 到此位置，我们基本保证发布的每一条消息，都会持久保存在每个“订阅者消息队列”中

5. subscribe端，每收到一个订阅消息，在消费之后，必须删除自己的“订阅者消息队列”头部的一条消息记录

6. subscribe端启动时，如果发现自己的“订阅者消息队列”有残存的记录，那么将会首先消费这些记录，然后再去订阅
