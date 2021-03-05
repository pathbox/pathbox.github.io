---
layout: post
title: 如何更可控、稳定地写Rails脚本操作线上数据库的记录
date:   2016-11-07 16:15:06
categories: Rails
image: /assets/images/post.jpg
---

描述：某个功能需要为数据库增加相关数据。大概逻辑是：根据表Company，根据某些条件往Department和DepartmentsUser表中插入数据。
然后将User表中满足筛选条件的status字段进行更改。

结果：需要超过一个小时才将整个脚本跑完，也就是这一小时都在影响线上数据库

反思：可以更可控、稳定地写Ruby脚本操作线上数据库的表吗？

一、思考脚本是否可以在项目升级前运行

  如果脚本可以在项目升级前运行，则可以在一定程度上更早的发现问题，以做出对正式升级时的预防。比如，操作新的表，或新的字段。这种情况是完全可以提早运行脚本的。
  有的脚本是必须要升级之前运行的，由于线上代码逻辑依赖于这个脚本的数据。所以，需要在升级之前跑一下，升级之后再跑一下。这样的脚本需要写成可重复执行。
  但是有的脚本是只能升级后跑的，则千万别在升级前跑

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
  脚本重复运行要注意的是，避免每次的重复运行增加脏数据。只有update操作的脚本可以重复运行，如果是create的操作，可以在操作前先判断一下是否存在，不存在则创建。
  find_or_create_by方法可以使用block方法，更方便的使用。

五、使用find_each 或 find_in_batches

  使用find_each 或 find_in_batches 分批次的进行查询循环操作。注意，find_in_batches得到的是对象数组集合。

六、使用gem 'ruby-progressbar' 愉快的展示执行进度

  在Gemfile中假如 gem 'ruby-progressbar', require: false。 然后在脚本文件中  require 'ruby-progressbar' 进行使用。可以很方便的展示脚本执行的百分比和进度。

七、统计脚本结果信息，对比正确的信息以确定脚本运行无误。

八、踩过的坑。。。。。

##### 在rails c(用的是pry环境)不要跑脚本，因为脚本代码中如果有next方法，会使用到的是pry的next方法而不是ruby的next方法，然后你就掉坑里了这对rails 4.0 的版本有这个问题,　４.2 以上的版本修复了这个问题

##### 代码中不要定义和数据库字段相同名称的方法，如果有重名的方法，请先确认是需要该方法还是需要的是数据库的字段。是数据库的字段请使用read_attributes

##### 让每个脚本只执行一个循环模块，进行一种数据的更改。尽量保持独立。如果有互相依赖的循环逻辑，再写在一个脚本中。

九、对执行脚本后，线上数据的分析和校验。我觉得这也是最难的一步了，这一步根据不同项目而不同。我觉得应该在执行脚本之前，将验证数据的方案也准备好，

而不是脚本执行完就过了，而不知道线上数据到底正确不正确。所以，执行脚本其实应该分两部分。一部分是正常的更改逻辑，另一部分是验证线上数据正确与否的方案。

也许会因此会有成倍的代码量，但是，如果给线上数据带来隐患，给用户造成损失或不便，这才是更大的问题。

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

无论是写rails脚本,rake脚本, 复杂的migration或者其他语言的脚本操作线上的数据库，都应该思考如何对线上的数据库影响最小，对用户的使用影响最小，以及最终的结果是否正确
也许在本地或测试环境你的脚本运行的没有问题，但是，线上的数据数量往往是本地和测试环境的n倍，并且已经有很多隐藏的脏数据，各种奇妙的事情都有可能发生。即使是在半夜运行脚本，
能够预先做更多的准备对应出现的异常情况，也能更快的找到问题的所在。

准备下一次的项目升级



##### 补充

使用rake脚本，将MySQL数据同步到ElasticSearch



```ruby
STDOUT.sync = true

# 该方法实现find_in_batches 功能，但是order by 是以 updated_at 排序，在使用ES增量同步时，提高了效率
  def find_in_batches(relation, order_cloumn = "updated_at", batch_size = 1000)
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

#coding: utf-8
# 第一次的时候，要确保有对应的索引存在
# rake es_sync:sync_data_es from=2016-03-04 to=2016-09-08 table=Ticket company_id=x batch_size=1000
# 支持 并发 rake parallel=1/4 es_sync:sync_data_es form=2016-03-04 table=Ticket
# batch_size=1000 批量取的数量
# 公司参数可传可不传
# table 参数要是对应model的名称 如: Ticket, to 和conpany_id 可以不传
# 对于有自定义字段的同步，可以在最后加 es_sync=true 优化自定义字段的同步速度，暂时对ticket有效
namespace :es_sync do

  desc 'MySQL 数据记录同步到ES'
  task sync_data_es: :environment do
    from       = ENV['from']
    to         = ENV['to']
    model      = ENV['table']
    company_id = ENV['company_id']
    lot_num    = ENV['parallel'].split('/').first if ENV['parallel']
    batch_size = (ENV['batch_size'] || 1000).to_i

    from = Time.parse(from)
    to   = to.present? ? Time.parse(to) : Time.now
    record_total = 0
    i = 0
    s = Time.now

    time = s.strftime("%Y%m%d%H%M%S")
    log_name = "es_sync_error_log_#{time}_#{model}_#{lot_num}.log"
    LOGGER = create_logger(log_name)
    if model.blank?
      LOGGER.record do |title, log|
        title << "ES Sync Mysql"
        log[:info] = "table param is blank"
      end
      return
    end
    LOGGER.record do |title, log|
      title << "ES Sync Mysql"
      log[:info] = "++++++++++ start sync work ++++++++++"
    end

    model = model.camelize.constantize
    model.index_name # 用于验证是否有对应的ES索引，没有会报错而不再继续进行
    if model == WorkLog
      where_sql = model.where(created_at: (from..to))
    else
      where_sql = model.where(updated_at: (from..to))
    end
    if company_id.present?
      where_sql = where_sql.where(company_id: company_id)
    end
    text_field_fields = $ARGV.last == "es_sync=true" ? TextField.get_text_select_fields : nil
    find_in_batches(where_sql, "updated_at", batch_size) do |records|
      body_ary = []
      i += 1
      next unless Paralleler.valid?(i)
      records.each do |record|
        begin
          record.custom_field_es_hash = text_field_fields if text_field_fields.present?
          body_ary << { index: { _id: record.id, data: record.as_indexed_json } }
        rescue => e
          options                           = {'exception'=>{}}
          options['record_id']              = record.id
          options['occur_at']               = Time.current
          options['exception']['message']   = e.try(:message)
          options['exception']['backtrace'] = e.try(:app_backtrace)
          LOGGER.record  do |title, log|
            title << "ES Sync Mysql"
            log[:info] = "++++++++++++error: #{options}"
          end
        end
      end
      next if body_ary.blank?
      begin
        model.__elasticsearch__.client.bulk({
          index: model.index_name,
          type: model.document_type,
          body: body_ary
        })
        record_total += records.size
        last_record = body_ary.last
        last_id     = last_record.try(:[], :index).try(:[], :_id)
        LOGGER.record do |title, log|
          title << "ES Sync Mysql"
          log[:info] = "NO.#{i} batch imported(#{records.size})"
          log[:record_totaltotal] = "-------- #{record_total}"
          log[:last_id] = " last record id: #{last_id}"
        end
      rescue => e
        options  = {'exception'=>{}}
        first_record = body_ary.first
        first_id     = first_record.try(:[], :index).try(:[], :_id)
        options.merge!({first_id: first_id, last_id: last_id})
        options['occur_at']               = Time.current
        options['exception']['message']   = e.try(:message)
        options['exception']['backtrace'] = e.try(:app_backtrace)
        LOGGER.record do |title, log|
          title << "ES Sync Mysql"
          log[:info] = "++++++++++++error: #{options}"
        end
      end
    end
    model.__elasticsearch__.refresh_index!

# +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++sync deleted record start
  if model != WorkLog
    LOGGER.record do |title, log|
       title << "ES Sync Mysql"
       log[:info] = "sync deleted record start"
     end
    where_sql = EsSyncDeletedRecord.where(record_type: model.to_s).where("updated_at >= ?", from)
    if company_id.present?
      where_sql = where_sql.where(company_id: company_id)
    end
    i = 0
    total = 0
    find_in_batches(where_sql, "updated_at", batch_size) do |records|
      i += 1
      next unless Paralleler.valid?(i)
      body = records.map{|record| {delete: {_id: record.record_id}}}
      model.__elasticsearch__.client.bulk({
        index: model.index_name,
        type: model.document_type,
        body: body
      })
      total += records.size
    end
    LOGGER.record do |title, log|
       title << "ES Sync Mysql Delete"
       log[:info] = "-------- deleted record count #{total}"
       log[:count] = "-------- EsSyncDeletedRecord count #{where_sql.count}"
     end
   end
# +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    e = Time.now
    x = e - s
    LOGGER.record do |title, log|
      title << "ES Sync Mysql"
      log[:info] = "Sync work is done~~~~~~"
      log[:second] = "Need seconds: #{x}"
    end
  end

# rake es_sync:check_sync_data_es from=2016-03-04 to=2016-09-08 table=Ticket company_id=x
# 公司参数可传可不传
# table 参数要是对应model的名称 如: Ticket
# 同步数量小于5w的不适合用本脚本，直接使用import force:true 不断同步就可以
# 没有updated_at 字段的表不可以检验
  desc 'MySQL 数据记录同步到ES校验'
  task check_sync_data_es: :environment do
    from       = ENV['from']
    to         = ENV['to']
    model      = ENV['table']
    company_id = ENV['company_id']
    count_from_mysql = ENV['count'] ? ENV['count'].to_i : nil

    from = Time.parse(from)
    to   = to.present? ? Time.parse(to) : Time.now

    if model.blank?
      puts "table param is invalid or blank"
      return
    end
    puts "++++++++++ check sync work ++++++++++"

    model = model.camelize.constantize
    model_mysql = model.where(updated_at: (from..to))
    if company_id
      model_mysql = model_mysql.where(company_id: company_id)
    end

    count_from_mysql = count_from_mysql || model_mysql.count
    count_from_es    = Elasticsearch::Model.client.count(index: model.index_name)["count"]
    index_name = model.index_name
    puts "Count from Mysql #{index_name}: #{count_from_mysql}"
    puts "Count from ES #{index_name}: #{count_from_es}"
    puts "开始进行MySQL-ES #{index_name}数据的随机抽查"

    num = count_from_mysql / 1000
    shuffle_ids = []
    if num > 999
      # 每num条随机抽一个, 抽取1000个
      1000.times.each do |item|
        min = item * num
        max = min + num
        r_num = rand(min..max)
        shuffle_ids << model.where("id >= #{r_num}").take.try(:id)
      end
    else
      shuffle_ids = model_mysql.order("rand()").limit(1000).pluck(:id)
    end
    hash = {}
    result = model.__elasticsearch__.search(
      _source: {include: 'updated_at'},
      query: {filtered: {filter: {terms: {
        _id: shuffle_ids
        }}}},
      size: 1000
      )

    result.results.to_a.each do |item|
      hash[item[:_id].to_s] = Time.zone.parse(item[:_source][:updated_at])
    end
    model_mysql.where(id: shuffle_ids).pluck(:id, :updated_at).each do |item|
      id = item.first.to_s
      if hash[id] != item.last
        puts "++++++++ disaccorded #{model} id: #{id}"
      else
        puts "++++++++ #{model} id: #{id} OK"
      end
    end
    puts "++++++++++ check sync work finish ++++++++++"
  end
end
```

##### 写rails 脚本,　对ES进行mapping 预定义
```ruby
# bundle exec rails runner -e development 2017-05-11-add_columns_to_users_es_mapping.rb

client = Elasticsearch::Model.client
client.indices.put_mapping index: "users", type: "user", body: {
  user: {
    properties: {
      source_type: {type: :string, index: :not_analyzed},
      source_id: {type: :integer}
    }
  }
}
```

​                   
