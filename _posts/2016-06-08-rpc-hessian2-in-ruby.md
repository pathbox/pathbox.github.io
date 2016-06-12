---
layout: post
title: Hessian in ruby
date:   2016-06-08 16:53:06
categories: rails
image: /assets/images/post.jpg
---

### Hessian in ruby

简单介绍:
Hessian is a binary web service protocol  that makes web services usable without requiring a large framework, and without learning a new set of protocols.
Because it is a binary protocol, it is well-suited to sending binary data without any need to extend the protocol with attachments.
Hessian是由caucho提供的一个基于binary-RPC实现的远程通讯library。
网上传的一个性能效率对比结果
测试结果显示，几种协议的通讯效率依次为：
RMI > Httpinvoker >= Hessian >> Burlap >> web service

这篇文章主要讲在ruby中如何使用Hessian。这里用到的是rails的项目

现在Gemfile中加入

```ruby

gem 'hessian2'

```

然后在 lib目录下新建一个文件 hessian_client.rb

```ruby

class HessianClient

  def add(a,b)
    url = "hessian_remote_url"  # => 127.0.0.0.1:9090/xxx/remote/
    client = Hessian2::Client.new(url)
    sum = client.add(a, b)
    puts sum
  end
end

```

如果远端已经部署好Hessian协议的服务了,得到url,创建一个Hessian2::Client实例,然后按照远程Hessian服务中定义的方法传入该方法有效的参数,就可以
调用该方法而得到计算结果。
注意问题: 方法要是服务端所定义的方法,参数也需要按照服务端要求。(就像本地调用方法类似)

