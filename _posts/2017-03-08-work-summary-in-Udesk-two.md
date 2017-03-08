---
layout: post
title: 最近工作总结(二)
date:   2017-03-08 17:55:06
categories: Work
image: /assets/images/post.jpg
---

##### serialize Hash
在serialize Hash 有个小坑。 普通的hash和ActionController::Parameters 或 Hashie::Mash 等类型都能存。这就会导致了，如果代码中原本使用symbol方式取值正确的逻辑，当该字段值用hash类型，并且hash对应的key用字符串形式存了，原本原本使用symbol方式取值正确的逻辑取值就变为空了。所以，推荐当取值的时候都用字符串形式。第二个是，如果数据库存了ActionController::Parameters类型的值，在rails 5中取这个值的时候就报错,,直接导致用这个model的时候就报错。因为ActionController::Parameters 在rails 5不再继承于Hash，而进行 serialize hash 的时候就不行了。在rails 4 中把这个数据用普通hash更改一下，在rails 5 项目中就恢复正常

##### self.abstract_class = true

```ruby
class SuperClass < ActiveRecord::Base
  self.abstract_class = true
end
class Child < SuperClass
  self.table_name = 'the_table_i_really_want'
end
# self.abstract_class = true is required to make Child<.find,.create, or any Arel method> use the_table_i_really_want instead of a table called super_classes
```

##### API 鉴权接口时间戳的限制
API 鉴权接口时间戳的限制，一般时间过期时间定义为10分钟，并且可以对于当前时间来说，多10-60s。这样可以避免，不同电脑的时间和服务器可能会有一些误差，所以，定义一个可接受的误差范围，可以避免“迷之鉴权失败”

##### 测试服务器上的elasticsearch bug
某次测试服务器上某个company的A索引无法再新建数据，其他company则正常。重新A.import force:true。 恢复正常

##### destroy_all vs delete_all
destroy / destroy_all: The associated objects are destroyed alongside this object by calling their destroy method
delete / delete_all: All associated objects are destroyed immediately without calling their :destroy method

##### elasticsearch 的mapping
elasticsearch 的mapping 能新增field，不能更改原来的field，更改需要重建索引，即清空索引数据，然后重建索引结构，最后导入数据

##### find_in_batches
不是find_in_batch. 可以这样 where().where().find_in_batches

##### Sidekiq 对参数的要求
The arguments you pass to perform_async must be composed of simple JSON datatypes: string, integer, float, boolean, null, array and hash. The Sidekiq client API uses JSON.dump to send the data to Redis. The Sidekiq server pulls that JSON data from Redis and uses JSON.load to convert the data back into Ruby types to pass to your perform method. Don't pass symbols, named parameters or complex Ruby objects (like Date or Time!) as those will not survive the dump/load round trip correctly.
Passing a Ruby hash is valid as it is able to serialize into valid JSON. So to answer your question, it's safe practice.

Sidekiq对hash对象的序列号，是使用了JSON进行序列化，所以存在redis中的还是json。取出的时候，又将json序列化回hash对象。
这样，就将symbol key 转为了 字符串key

```ruby
serialized_args = JSON.dump({email_address: "someemailaddress@gmail.com", something_else: "thing"})
=> "{\"email_address\":\"someemailaddress@gmail.com\",\"something_else\":\"thing\"}"

JSON.load(serialized_args)
=> {"email_address"=>"someemailaddress@gmail.com", "something_else"=>"thing"}
```

所以，如果你传了 hash = {name: "My name", age: 20} 参数， 在 worker 代码里， 得到的其实是 {"name": "My name", "age": 20}

No symbol key again
