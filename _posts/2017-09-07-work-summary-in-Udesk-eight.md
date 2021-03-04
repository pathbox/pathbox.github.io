---
layout: post
title: 最近工作总结(八)
date:   2017-09-07 10:25:06
categories: Work
image: /assets/images/post.jpg
---

##### ActiveRecord reload
reload方法：数据库更新不可能反馈到变更前创建的对象上。通过reload方法让对象重新加载数据库最新的变更。

##### SaaS PaaS IaaS 开设一家披萨店
阮一峰关于 SaaS PaaS IaaS 的解释文章
http://www.ruanyifeng.com/blog/2017/07/iaas-paas-saas.html

##### 传current_user 还是传current_user_id

一个方法的参数，是选择 传current_user 还是 current_user_id

如果选择传 current_user， 在方法使用中往往还需要考虑，current_user 会不会为nil。为nil的时候，代码逻辑会不会报错，需要做什么处理。
而当传current_user_id的时候，如果方法中只是需要current_user.id，并不需要更多的current_user的属性，我会选择传current_user_id
因为我发现，传current_user会产生更复杂的情况

##### Many RUN is a shit in Dockerfile

`shit`
用了N层，有几个RUN就有几层

```
FROM	debian:jessie
RUN	apt-get	update
RUN	apt-get	install	-y	gcc	libc6-dev	make
RUN	wget	-O	redis.tar.gz	"http://download.redis.io/releases/redi
s-3.2.5.tar.gz"
RUN	mkdir	-p	/usr/src/redis
RUN	tar	-xzf	redis.tar.gz	-C	/usr/src/redis	--strip-components=1
RUN	make	-C	/usr/src/redis
RUN	make	-C	/usr/src/redis	install
```

`nice`

这样才用了一层
```
FROM	debian:jessie
RUN	buildDeps='gcc	libc6-dev	make'	\
				&&	apt-get	update	\
				&&	apt-get	install	-y	$buildDeps	\
				&&	wget	-O	redis.tar.gz	"http://download.redis.io/releases/r
edis-3.2.5.tar.gz"	\
				&&	mkdir	-p	/usr/src/redis	\
				&&	tar	-xzf	redis.tar.gz	-C	/usr/src/redis	--strip-component
s=1	\
				&&	make	-C	/usr/src/redis	\
				&&	make	-C	/usr/src/redis	install	\
				&&	rm	-rf	/var/lib/apt/lists/*	\
				&&	rm	redis.tar.gz	\
				&&	rm	-r	/usr/src/redis	\
				&&	apt-get	purge	-y	--auto-remove	$buildDeps
```

##### 变量取名思考

根据变量类型取名

根据变量功能取名

##### 在rails项目中同时使用两个ES集群

/initializers/elasticsearch.rb

```ruby
# 默认的elasticsearch client
Elasticsearch::Model.client = Elasticsearch::Client.new(
  host: "127.0.0.1:9200",
  randomize_hosts: true,
  retry_on_failure: 0,
  log: true,
  transport_options: { request: { timeout: 10 } }
)

# 自定义的第二个elasticsearch client
MySecondClient = Elasticsearch::Client.new(
  host: "127.0.0.1:9500",
  randomize_hosts: true,
  retry_on_failure: 0,
  log: true,
  transport_options: { request: { timeout: 10 } }
)
```

app/model/post.rb

```ruby
#当直接使用：
Post.__elasticsearch__.search(query: query)

# 上面ES查询语句访问的是默认的elasticsearch
```

使用自定义的elasticsearch

```ruby
class Post < ActiveRecord::Base
  self.__elasticsearch__.client = MySecondClient # 指定使用第二个elasticsearch

  Post.__elasticsearch__.search(query: query)

  # 这时候，访问的是第二个elasticsearch
end
```

##### 把二进制命令放入 /usr/bin目录中，后会发生神奇的事
把二进制命令放入 /usr/bin目录中，你就可以在终端使用这个命令啦～

##### includes + map tip

```ruby
current_user.user_groups.includes(:users).map(&:users).flatten
```

```sql
UserGroup Load (0.6ms)  SELECT `user_groups`.* FROM `user_groups` INNER JOIN `users_user_groups` ON `user_groups`.`id` = `users_user_groups`.`user_group_id` WHERE `users_user_groups`.`user_id` = 2
  UsersUserGroup Load (0.8ms)  SELECT `users_user_groups`.* FROM `users_user_groups` WHERE `users_user_groups`.`user_group_id` IN (1, 2)
  User Load (3.2ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` IN (2, 3, 4, 5, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 36, 37, 38, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100, 101, 102, 103, 104, 105, 106, 107, 108, 109, 110, 122, 13, 100146, 100149, 100150, 100154, 100155, 100156, 10, 39, 64, 100157)

```
得到结果是一个user对象数组

如果只是想要得到user_id

其实只要执行两句sql查询
```
user_group_ids = current_user.user_groups.pluck(:id)
user_ids = UsersUserGroup.where(user_group_id: user_group_ids).pluck(:user_id).uniq
```

而且用到了pluck，不会产生大量的user对象数组

但是如果你是需要使用到user对象的其他字段的时候，就不能这样做了

##### 和`elasticsearch` 有关的 `gem` `document`
http://www.rubydoc.info/gems/elasticsearch-watcher

http://www.rubydoc.info/gems/elasticsearch-api/Elasticsearch/API

http://www.rubydoc.info/gems/elasticsearch-model/Elasticsearch

http://www.rubydoc.info/gems/elasticsearch-transport


##### ActiveRecord merge

```ruby
User.where(company_id: 1).joins(:tickets).merge(Ticket.where(company_id: 1))

User Load (1236.6ms)  SELECT `users`.* FROM `users` INNER JOIN `tickets` ON `tickets`.`user_id` = `users`.`id` WHERE `users`.`company_id` = 1 AND `tickets`.`company_id` = 1

User.where(company_id: 1).joins(:tickets).merge(->{joins(:tickets)})
User Load (1205.6ms)  SELECT `users`.* FROM `users` INNER JOIN `tickets` ON `tickets`.`user_id` = `users`.`id` WHERE `users`.`company_id` = 1
```

`ActiveRecord merge` 帮助你在joins连接表操作的时候，可以优雅的增加`where` 条件操作，就不用自己手写SQL了

##### create_index! in elasticsearch

```ruby
def create_index!(options={})
  options = options.clone

  target_index = options.delete(:index) || self.index_name
  settings =  options.delete(:settings) || self.settings.to_hash
  mappings = options.delete(:mappings) || self.mappings.to_hash

  delete_index!(options.merge index: target_index) if options[:force]

  unless index_exists?(index: target_index)
    self.client.indices.create index: target_index,
      body: {
        settings: settings,
        mappings: mappings
      }
  end
end

```
