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
