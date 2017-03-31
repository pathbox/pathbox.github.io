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
不是find_in_batch. 可以这样 where().where().find_in_batches, 默认是一次处理1000个记录

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

##### 使用Get 请求进行更新数据操作的话，有安全漏洞。CSRF攻
示例：

　　银行网站A，它以GET请求来完成银行转账的操作，如：http://www.mybank.com/Transfer.php?toBankId=11&money=1000

　　危险网站B，它里面有一段HTML的代码如下：

　　<img src=http://www.mybank.com/Transfer.php?toBankId=11&money=1000>
　　首先，你登录了银行网站A，然后访问危险网站B，噢，这时你会发现你的银行账户少了1000块......

　　为什么会这样呢？原因是银行网站A违反了HTTP规范，使用GET请求更新资源。在访问危险网站B的之前，你已经登录了银行网站A，而B中的<img>以GET的方式请求第三方资源（这里的第三方就是指银行网站了，原本这是一个合法的请求，但这里被不法分子利用了），所以你的浏览器会带上你的银行网站A的Cookie发出Get请求，去获取资源“http://www.mybank.com/Transfer.php?toBankId=11&money=1000”，结果银行网站服务器收到请求后，认为这是一个更新资源操作（转账操作），所以就立刻进行转账操作......

##### Sidekiq queue 的设置
如果在Sidekiq的配置文件中没有配置一个队列A，而在worker文件中指定了队列A，这个worker不会执行。因为，找不到队列A。
所以，需要在配置文件配置对应的队列A才行。如果在worker文件中不写使用哪个队列，则会使用默认的队列default。default
最好也在Sidekiq配置文件中配置好

##### Elasticseatch 查看某个索引document count数量
User.__elasticsearch__.client.count(index:'users')

##### 对于joins表操作时，相同字段的条件需要指定表名
A B 两个表进行joins操作，他们都有created_at 和 flag字段
A.joins(:b).where("table_a.created_at > ?", Time.now) # Right
A.joins(:b).where("created_at > ?", Time.now) # ActiveRecord::StatementInvalid: Mysql2::Error: Column 'created_at' in where clause is ambiguous
A.joins(:b).where(flag: true) # Right， flag 会是 A表的字段
A.joins(:b).where(" a_name = ? ", "Kitty") # Right, 连接表中只有唯一的a_name 字段
A.joins(:b).where(" b_name = ? ", "Kitty") # Right, 连接表中只有唯一的b_name 字段

##### redirect_to
在controller中

```ruby
redirect_to home_path
puts "Hello World"  # 仍然会执行
```

##### build a find_in_batches for myself

```ruby
class FindInBatchesExtend

  # 该方法实现find_in_batches 功能，但是order by 是以 updated_at 排序，在使用ES增量同步时，提高了效率
  def self.find_in_batches(relation, order_cloumn = "updated_at", batch_size = 1000)
    relation = relation.order("#{order_cloumn} ASC").limit(batch_size)
    records = relation.to_a

    while records.any?
      records_size = records.size
      offset_column = records.last.send("#{order_cloumn}")

      yield records

      break if records_size < batch_size
      records = relation.where("#{order_cloumn} >= ?", offset_column).to_a
    end
  end
end
```

##### order by id 不一定是最高效的
```
SELECT COUNT(*) FROM `tickets` WHERE `tickets`.`company_id` = 5899 AND `tickets`.`status_id` = 3 AND (closed_at >= '2017-03-13 00:00:00') ORDER BY `tickets.id` ASC
```

上面已经建立了 company_id status_id closed_at 的联合索引，但是任然很慢。通过explain查看没有使用到联合索引
将代码改成执行下面的sql

```
SELECT COUNT(*) FROM `tickets` WHERE `tickets`.`company_id` = 5899 AND `tickets`.`status_id` = 3 AND (closed_at >= '2017-03-13 00:00:00') ORDER BY `tickets.closed_at` ASC
```

则使用到了联合索引，速度变为10+ms，优化成功
