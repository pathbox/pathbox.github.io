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

比如，配置 Lograge::Formatters::Json.new

```ruby
config.lograge.formatter = Lograge::Formatters::Json.new
```

得到的日志结果是这样的：


```ruby
config.lograge.formatter = Lograge::Formatters::Logstash.new
```

得到的日志结果是这样的：



很方便的能够得到你想要的日志内容结构。
