---
layout: post
title: 如何更可控、稳定地写Ruby脚本操作线上数据库的表
date:   2016-11-07 16:15:06
categories: rails
image: /assets/images/post.jpg
---

描述：某个功能需要为数据库增加相关数据。大概逻辑是：根据表Company，根据某些条件往Department和DepartmentsUser表中插入数据。
然后将User表中满足筛选条件的status字段进行更改。

结果：需要超过一个小时才将整个脚本跑完，也就是这一小时都在影响线上数据库

反思：可以更更可控、稳定地写Ruby脚本操作线上数据库的表吗？

一、思考脚本是否可以在项目升级前运行

  如果脚本可以在项目升级前运行，则可以在一定程度上更早的发现问题，以做出对正式升级时的预防。比如，操作新的表，或新的字段。这种情况是完全可以提早运行脚本的。

二、是否有长时间commit操作

  警惕长时间commit的操作。举个例子：比如用update_all或批量插入在一定程度上可以更快的运行sql，但是，这些操作的commit的时间可能会很长(如果你用的是MySQL)。这时候用户可以进行select 或 insert操作，但是对update操作就会阻塞了。
  如果时间过长则很有可能超时。这样会让用户感知到系统的延迟或崩溃。所以，不使用这种批量操作，排除commit长时间的操作，使用粒度小的create，update，destroy操作或者每次小批量的commit操作。如果一个commit很大，中间异常断了，则前面跑的
  sql语句就废弃了。而粒度小的操作则可以避免这样。虽然，总的执行时间会更长，但是，这样就不会长时间占用数据库的某个表而影响用户的一些使用。
  如果真的无法避免长时间的操作，就需要在后半夜大家睡觉的时候跑脚本或者进行数据库操作了(比如新增字段，新增索引等)。

三、捕获异常错误

  使用异常，捕获异常错误，将异常打印出来。或者可以在全局新建一个数组，然后将某次异常的操作中相关记录的id存入数组中，最后打印出来，就可以知道在执行过程中哪个记录是有问题的，可以很方便的追踪错误，后续就可以
  很方便的进行验证和纠错了。

四、让脚本可以重复运行

  尽量让脚本可以重复运行。即使加入额外的find或者find_or_create_by查询。或者捕获异常，而不是让异常中断操作。不要使用清空数据库的方式，因为这样需要从0条记录开始跑，所需要的时间更长。
  脚本重复运行要注意的是，避免每次的重复运行增加脏数据

五、使用find_each 或 find_in_batches

  使用find_each 或 find_in_batches 分批次的进行查询循环操作。注意，find_in_batches得到的是对象数组集合。

六、使用gem 'ruby-progressbar' 愉快的展示执行进度

  在Gemfile中假如 gem 'ruby-progressbar', require: false。 然后在脚本文件中  require 'ruby-progressbar' 进行使用。可以很方便的展示脚本执行的百分比和进度。

七、统计脚本结果信息，对比正确的信息以确定脚本运行无误。

代码示例分析：

```ruby
require 'ruby-progressbar'
progressbar = ProgressBar.create :total => Company.count, format: "%p%% [%B] %c/%C %E"
progressbar.log "= 开始初始化 ="
Company.find_each do |company|
  begin
    department = Department.create!(company_id: company.id, name: company.subdomain, parent_id: 0, is_default: true)
    company.agents.each do |agent|
      begin
        DepartmentsUser.create!(user_id: agent.id, department_id: department.id)
      rescue => e
        puts e
        next
      end
    end
  rescue => e
    puts e
    next
  end
  progressbar.increment
end

progressbar.log "=== 初始化完毕 ==="
User.where(p: ['A','UG']).update_all(status: true)
```

代码示例小结:

1.脚本是为新的数据库补充对应数据，所以可以在项目升级前运行

2.脚本使用了find_each

3.脚本进行了异常捕获

4.脚本使用了ruby-progressbar

可以改进的地方:

1.脚本不能重复跑，如果重复跑的话，对于create操作会产生脏数据；将create!改为find_or_create_by!

2.没有收集异常错误的记录

3.没有统计脚本结果信息，对比正确的信息

4.建议将progressbar.increment 放到循环的最开始

5.使用了update_all 这种可能长时间锁表的操作

代码示例改进:

```ruby
require 'ruby-progressbar'
progressbar = ProgressBar.create :total => Company.count, format: "%p%% [%B] %c/%C %E"
progressbar.log "= 开始初始化 ="
error_user = []
error_company = []
update_user_error = []
Company.find_each do |company|
  progressbar.increment
  begin
    department = Department.find_or_create_by!(company_id: company.id, name: company.subdomain, parent_id: 0, is_default: true)
    company.users.each do |user|
      begin
        DepartmentsUser.find_or_create_by!(user_id: user.id, department_id: department.id)
      rescue => e
        puts e
        error_user << user.id
        next
      end
    end
  rescue => e
    puts e
    error_company << company.id
    next
  end
end

User.where(p: ['A','UG']).find_each do |user|
  begin
    user.update!(status: true)
  rescue => e
    puts e
    update_user_error << user.id
  end
end

progressbar.log "Company count: #{Company.count} -- Department count: #{Department.count} -- DepartmentsUser count: #{DepartmentsUser.count} User count: #{User.count}"
progressbar.log "error: error_user: #{error_user}; error_company: #{error_company}"
progressbar.log "=== 初始化完毕 ==="
```

```
执行： bundle exec rails runner -e development init_department.rb

= 开始初始化 =                                                                                                                                                                        
100% [======================================================================================================================================================================] xxx/xxx Time: 00:00:00
Company count: xxx -- Department count: xxx -- DepartmentsUser count: xxx User count: xxx                                                                                                          
error: error_user: []; error_company: []                                                                                                                                                         
=== 初始化完毕 ===

```

本文提供的几个思路其实都不难，只是往往我们都忽略了。

无论是写ruby脚本,rake脚本, 复杂的migration或者其他语言的脚本操作线上的数据库，都应该思考如何对线上的数据库影响最小，对用户的使用影响最小，以及最终的结果是否正确
也许在本地或测试环境你的脚本运行的没有问题，但是，线上的数据数量往往是本地和测试环境的n倍，并且已经有很多隐藏的脏数据，各种奇妙的事情都有可能发生。即使是在半夜运行脚本，
能够预先做更多的准备对应出现的异常情况，也能更快的找到问题的所在。

准备下一次的项目升级                             
