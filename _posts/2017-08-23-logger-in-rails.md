---
layout: post
title: Rails中的使用logger实用技巧
date:   2017-08-23 15:30:06
categories: Rails
image: /assets/images/post.jpg
---

##### Gem lograge

[lograge](https://github.com/roidrage/lograge) is awesome for doing log.

```ruby

# Gemfile
gem "lograge"
```

```ruby
# config/initializers/lograge.rb
# OR
# config/environments/production.rb

Rails.application.configure do
  ...
# Logger
  config.lograge.enabled = true
  config.lograge.ignore_actions = ['home#welcome']
  config.lograge.custom_options = lambda do |event|
    params = event.payload[:params].reject do |k|
      ['controller', 'action'].include? k
    end
    options = {
      params: params,    # add params to lograge
      time:   event.time # add time to lograge
    }
    options
  end
  ...
end

```

`lograge` 能帮你格式化日志输出，定义日志输出的格式和内容。去掉了view层渲染的日志内容。如果你不需要日志具体打印view层的渲染内容，用`lograge`是极好的，如果你需要，`lograge` 就不适合你。一般应用不关心view层渲染的具体日志内容。

例子:

环境：Rails 5.1.4

不使用 `lograge`

使用 `lograge`

`lograge`还提供了不同的formatters。

```
Lograge::Formatters::Lines.new
Lograge::Formatters::Cee.new
Lograge::Formatters::Graylog2.new
Lograge::Formatters::KeyValue.new  # default lograge format
Lograge::Formatters::Json.new
Lograge::Formatters::Logstash.new
Lograge::Formatters::LTSV.new
Lograge::Formatters::Raw.new       # Returns a ruby hash object
```

默认的日志格式是 key=value的格式：

```ruby
[2017-12-01 11:49:10] [INFO] [] [localhost] [127.0.0.1] [1593754b-bc95-42] method=GET path=/ format=*/* controller=HomeController action=welcome status=200 duration=472.00 view=0.00 params={} time=2017-12-01 11:49:09 +0800

```

比如，配置 Lograge::Formatters::Json.new

```ruby
config.lograge.formatter = Lograge::Formatters::Json.new
```

得到的日志结果是这样的：

```ruby
[2017-12-01 22:43:34] [INFO] [] [localhost] [127.0.0.1] [74cc1bef-2155-42] {"method":"GET","path":"/","format":"*/*","controller":"HomeController","action":"welcome","status":200,"duration":3.09,"view":0.0,"params":{"home":{}},"time":"2017-12-01 22:43:34 +0800"}
```

```ruby
config.lograge.formatter = Lograge::Formatters::Graylog2.new
```

得到的日志结果是这样的：

```ruby
[2017-12-01 22:47:46] [INFO] [] [localhost] [127.0.0.1] [18654bb8-01ec-48] {:_method=>"GET", :_path=>"/", :_format=>"*/*", :_controller=>"HomeController", :_action=>"welcome", :_status=>200, :_duration=>368.34, :_view=>0.0, :_params=>{"home"=>{}}, :_time=>2017-12-01 22:47:46 +0800, :short_message=>"[200] GET / (HomeController#welcome)"}
```

如果想使用

```ruby
config.lograge.formatter = Lograge::Formatters::Logstash.new
```

需要配合安装 `logstash-event`

```ruby
gem "logstash-event"
```

这样很方便的能够得到你想要的日志内容结构，并且被各种日志监控系统兼容。这是`lograge`简单的功能，更多配置查看[文档](https://github.com/roidrage/lograge)

##### 自定义生成log文件

有时候，我们想要将日志存储到一个新的log文件中，和主日志文件区分开来。

定义一个类方法或全局的方法：

```ruby
def create_my_logger(file_name)
  logger = Logger.new("#{Rails.root}/log/#{file_name}")  # 根据file_name，创建一个logger实例。会在Rails.root/log目录下生成file_name文件，用来记录日志
  logger.level = Logger::DEBUG  # 设置日志的level
  logger.formatter = proc do |severity, datetime, progname, message| # 设置日志的formatter
    "[#{datetime.to_s(:db)}] [#{severity}] #{message}\n"
  end

  logger
end
```

在想要使用该日志实例的代码中：

```ruby
log = create_my_logger("third_api_service.log")

... # 某些操作
result = do_third_api_service
if result
  log.info("Third api service success")
else
  log.error("Third api service fail: #{result}")
end
```

个人认为这种方法适用于调用一些关键服务或操作，想要快速了解该服务的正确性，当有错误的时候，可以快速查阅日志，了解服务信息。也便于调试和快速排查问题

还可以给logger实例扩展一些类方法：

```ruby
module MyLogger
  def log_json
    info = {}
    title = []

    yield title, info

    timestamp  = Time.now.strftime('%Y%m%d_%H%M%S_%L')
    title      = [timestamp, title].flatten.compact
    title = title.join('-')

    data = {}
    data[:title] = title
    data.merge! info

    self.debug data.to_json # 打印json日志内容
  end
end

def create_my_logger(file_name)
  logger = Logger.new("#{Rails.root}/log/#{file_name}")  # 根据file_name，创建一个logger实例。会在Rails.root/log目录下生成file_name文件，用来记录日志
  logger.level = Logger::DEBUG  # 设置日志的level
  logger.formatter = proc do |severity, datetime, progname, message| # 设置日志的formatter
    "[#{datetime.to_s(:db)}] [#{severity}] #{message}\n"
  end
  logger.extend MyLogger

  logger
end
```
