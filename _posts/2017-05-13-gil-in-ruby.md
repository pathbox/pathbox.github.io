---
layout: post
title: Ruby 的GIL 到底做了什么
date:   2017-05-13 15:44:06
categories: Ruby
image: /assets/images/post.jpg
---

> GIL 是　Global Interpreter Lock 也叫 GVL

[无人知晓的GIL中文版](https://ruby-china.org/topics/28415)　这篇文章对GIL描述的比较清楚了。Ruby 或者说Rails中由于GIL是没有并行操作的，由于GIL的存在，每次操作只能使用一个CPU的核心。

This is bullshit。

看下面的例子, 是在CRuby中:

```ruby
require 'benchmark'

@threads = []
Benchmark.bm(14) do |x|
  x.report('single-thread') do
    8.times do
      tmp_array = []
      10_000_000.times { |n| tmp_array << n }
    end
  end

  x.report('multi-thread') do
    8.times do
      @threads << Thread.new do
        tmp_array = []
        10_000_000.times { |n| tmp_array << n }
      end
    end
    @threads.each(&:join)
  end
end

     user     system      total        real
single-thread
  5.240000   0.580000   5.820000 (  5.880398)
multi-thread
  6.450000   0.710000   7.160000 (  7.234476)

```

multi-thread 比single-thread 还慢。这里是由于GIL的作用，导致了这样的结果。

```ruby
require 'faraday'

@conn = Faraday.new(url: 'https://ruby-china.org')
@threads = []

Benchmark.bm(14) do |x|
  x.report('single-thread') do
    20.times { @conn.get }
  end

  x.report('multi-thread') do
    20.times do
      @threads << Thread.new { @conn.get }
    end
    @threads.each(&:join)
  end
end

#    user     system      total        real
# single-thread
#   0.190000   0.020000   0.210000 (  4.345562)
# multi-thread     0.090000   0.020000   0.110000 (  0.269009)

```
multi-thread 的消耗时间比　single-thread　少快20倍。可以看出，和上一个例子相比，多线程起到了作用。

为什么会这样呢？Ruby的GIL没有起到作用吗？

Ruby GIL doesn't block IO operations. 所以，像http　requests，　文件IO操作，是不会被GIL所限制的。

对于MySQL connection，　使用mysql2　的gem，也解决了GIL限制的问题，Ruby 程序也能并行的向MySQL Server　
发送连接请求，或者并行使用连接池查寻。No GIL。

关于Ruby 的web服务器。　Puma 使用的是多线程的方案。　对于http 请求, Puma　是能真正使用多线程处理的，而不会被GIL所限制，　所以Puma拥有更好的并发处理能力(理论上来说)。　但是,多线程拥有永恒的问题，竞态条件、竞态资源问题。如果，你写了并非是线程安全的代码，也许就会导致奇特的bug出现了。比如：

```ruby
@name ||= name

# source code
if @name == nil
  @name = name
end
name
```
这种先检查－再设置的操作，就有可能让@name 初始化两次。如果你有复杂的代码逻辑，也许就会出现奇怪的问题。

也许GIL限制了Ruby现在的发展，限制了Ruby的并发性能。期待Ruby3，能给Ruby带来新的革命。

 　
