---
layout: post
title: Rails的底层cache
date:   2016-06-10 16:45:06
categories: rails
image: /assets/images/post.jpg
---



Rails中使用的缓存方式有很多种，推荐的缓存策略是"俄罗斯套娃缓存"。不过使用这种缓存方式只能缓存html的片段，还是会进行sql查询。
一种方式，可以直接使用redis，rails中redis的gem也有不少，而且封装的功能足够使用。可以直接操作redis进行数据缓存。
另一种方式，rails也封装了底层的缓存的API：ActiveSupport::Cache::Store。在config.cache_store可以配置使用什么存储缓存
数据。可以这样 config.cache_store = :redis_store, "redis://localhost:6379"。就使用了redis缓存数据。

##### 实际使用

首先介绍 ActiveSupport::Cache::Store中的ActiveSupport::Cache::MemoryStore。它会将数据存储在内存中(没有存在redis中)
This cache store keeps entries in memory in the same Ruby process. The cache store has a bounded size specified by the :size options to the initializer (default is 32Mb). When the cache exceeds the allotted size, a cleanup will occur and the least recently used entries will be removed.

```ruby
config.cache_store = :memory_store, { size: 64.megabytes }
```

If you're running multiple Ruby on Rails server processes (which is the case if you're using mongrel_cluster or Phusion Passenger), then your Rails server process instances won't be able to share cache data with each other. This cache store is not appropriate for large application deployments,
but can work well for small,low traffic sites with only a couple of server processes or for development and test environments.
它适合小的web application。中大型的application不合适了。

```ruby
cache = ActiveSupport::Cache::MemoryStore.new
cache.read('city')   # => nil
cache.write('city', "Duckburgh")
cache.read('city')   # => "Duckburgh"
cache.read('city') == cache.read(:city)   # => true 可以使用symbol存储key
```

nil values can be cached.

其他操作

```ruby
cache.delete('city') #=> true
cache.read('city') #=> nil
cache.cleanup #=> Cleanup the cache by removing expired entries.
cache.clear #=> Clear the entire cache. Be careful with this method since it could affect other processes if shared cache is being used.
cache.exist?(name,options=nil)
cache.fetch
cache.write('today', 'Monday')
cache.fetch('today')  # => "Monday"
cache.fetch('city')   # => nil
cache.fetch('city') do
  'Duckburgh'
end
cache.fetch('city')   # => "Duckburgh"
```

about expires_in 过期时间
Setting :expires_in will set an expiration time on the cache. All caches support auto-expiring content after a specified number of seconds. This value can be specified as an option to the constructor (in which case all entries will be affected),
or it can be supplied to the fetch or write method to effect just one entry.

```ruby
cache = ActiveSupport::Cache::MemoryStore.new(expires_in: 5.minutes)
cache.write(key, value, expires_in: 1.minute) # Set a lower value for one entry
```

##### 更好的选择 Rails.cache

```ruby
Rails.cache #=> #<ActiveSupport::Cache::RedisStore:0x007ffc2daf1ac8 @data=#<Redis client v3.2.2 for redis://localhost:6379/0>, @options={}>
```
可以看到，Rails.cache 识别了config.cache_store = :redis_store, "redis://localhost:6379"的配置,使用了redis存储缓存数据
Rails.cache 同样哟 read write fetch 等方法操作。非常方便使用的是fetch

##### 使用

```ruby
Rails.cache.write('first_user_cache', User.first, expires_in: 2.minute)
User Load (0.4ms)  SELECT  `users`.* FROM `users`  ORDER BY `users`.`id` ASC LIMIT 1
=> "OK"

Rails.cache.fetch('first_user_cache')
#<User:0x007ffc301a0a20
 id: 1,
 email: "xxxx@xxxx",
 encrypted_password: "$2a$10$GxloDworS4Ud430zyTfG/.xuSWSAeDvPK3O5JUxo2EyOQlHAc1QAO",
 reset_password_token: nil,
 reset_password_sent_at: nil,
 remember_created_at: nil,
 sign_in_count: 8,
 current_sign_in_at: Fri, 30 Oct 2015 10:45:41 CST +08:00,
 last_sign_in_at: Wed, 28 Oct 2015 14:19:03 CST +08:00,
 current_sign_in_ip: "127.0.0.1",
 last_sign_in_ip: "127.0.0.1",
 created_at: Thu, 22 Oct 2015 16:00:28 CST +08:00,
 updated_at: Tue, 14 Jun 2016 17:58:12 CST +08:00>
```

Rails.cache.fetch的方法

```ruby
Rails.cache.fetch('first_user_cache', expires_in: 2.minute){User.first}

Rails.cache.fetch(key) do
  yield
end
```
可以把yield中的最后一句的返回值存储到key的缓存中

当想要Rails.cache.fetch 缓存像 User.all , User.where 这样的ActiveRecord::Relation结果集时，需要加上load。

```ruby
Rails.cache.fetch('all_user_cache', expires_in: 2.minute){User.all.load}(实际中不缓存all数据,很有爆内存的危险)
```

下面一段内容来自 http://xguox.me/rails-cache-fetch-scope.html

ActiveRecord::Relation 究竟是什么鬼?

```
[7] pry(main)> User.where(id: 1)
User Load (6.9ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` = 1
=> [<User:0x007fb96ee98b88 id: 1, email: "test@test.com">]
  看上去返回的结果很像是数组, 实际上, 却不是数组.
  [8] pry(main)> _.class
  => User::ActiveRecord_Relation(Rails 4)
  => ActiveRecord::Relation(Rails 3)
  btw, Rails 3 中执行 all 是返回 Array的, 而 Rails 4 则是 Relation
  ActiveRecord::Relation 只有当真正需要知道并使用到里面所包含的对象时候才会被执行. 比如在 controller 中:
  class PostsController < ApplicationController
    def index
      @posts    = Post.all
      @channels = Channel.all
      @comments = Comment.all
    end
  end
  假设在 index.html.erb 中, 没有任何地方用到 @comments 的话, 其实是不会执行数据库查询去把所有 comments 抓出来的. 而如果先使用到了 @channels, 比如 @channels.first.name 的话, 也是一样, 先执行
  Channel Load (27.8ms)  SELECT `channels`.* FROM `channels`
  再去执行(假设 view 里面有使用到 @posts)
  Post Load (47.2ms) SELECT `posts`.* FROM `posts`
  所以, Rails.cache.fetch('cache_all_posts', expires_in: 2.minute) { Post.all } 缓存的只是 Relation 对象而不是查询结果. 其实分开来, 使用 Rails.cache.read 和 Rails.cache.write 也是这样的.
```

ActiveRecord::Relation只有真正在使用的时候才会返回数组的结果集。要不只是一个对象。所以，缓存ActiveRecord::Relation并没有把实际的结果集缓存下来。这样每次真正使用到ActiveRecord::Relation的时候，都会再进行
sql的查询(缓存的作用没有达到) 加上load就会事先把ActiveRecord::Relation的数组结果集查询出来，然后保存在缓存中，就达到了缓存的作用。或者是Post.all.to_a、Post.all.order('server_index DESC').map(&:attributes)、干脆Post.all.pluck(:id,:title)这样缓存需要的字段值

本文完
















