---
layout: post
title: Processing large CSV files with Ruby(翻译)
date:   2016-06-13 15:24:06
categories: Rails
image: /assets/images/post.jpg
---



原文链接： [Processing large CSV files with Ruby](http://dalibornasevic.com/posts/68-processing-large-csv-files-with-ruby)

处理大文件内存密集型操作,可能导致服务器的内存和交换到磁盘。让我们来看看一些方法来处理CSV文件使用Ruby和测量内存消耗和速度性能

#### 准备 CSV数据文件
在我们开始测试前，我们准备一个data.csv测试文件。它存储一百万行记录(75MB)

```ruby
require 'csv'
require_relative './helpers'

headers = ['id', 'name', 'email', 'city', 'street', 'country']

name    = "Pink Panther"
email   = "pink.panther@example.com"
city    = "Pink City"
street  = "Pink Road"
country = "Pink Country"

print_memory_usage do
  print_time_spent do
    CSV.open('data.csv', 'w', write_headers: true, headers: headers) do |csv|
      1_000_000.times do |i|
        csv << [i, name, email, city, street, country]
      end
    end
  end
end
```

#### 内存的使用和时间消耗
上面这个脚本需要helpers.rb脚本文件,定义了两个辅助的方法测量和打印所使用的内存和时间

```ruby
require 'benchmark'

def print_memory_usage
  memory_before = `ps -o rss= -p #{Process.pid}`.to_i
  yield
  memory_after = `ps -o rss= -p #{Process.pid}`.to_i

  puts "Memory: #{((memory_after - memory_before) / 1024.0).round(2)} MB"
end

def print_time_spent
  time = Benchmark.realtime do
    yield
  end

  puts "Time: #{time.round(2)}"
end
```

构造测试CSV文件的打印结果是:

```
$ ruby generate_csv.rb
Time: 5.17
Memory: 1.08 MB
```

不同机器之间的输出会发生变化,但是关键是构建CSV文件时,Ruby进程没有内存使用量激增,因为垃圾收集器(GC)是使用内存回收。
进程的内存增加大约1 MB,它创建了一个CSV文件大小为75 MB

#### 从文件中一次读取CSV文件
让我们构建一个CSV对象从一个文件中(data.csv) 然后进行迭代根据下面的脚本:

```ruby

require_relative './helpers'
require 'csv'

print_memory_usage do
  print_time_spent do
    csv = CSV.read('data.csv', headers: true)
    sum = 0

    csv.each do |row|
      sum += row['id'].to_i
    end

    puts "Sum: #{sum}"
  end
end

```
结果是：

```
$ ruby parse1.rb
Sum: 499999500000
Time: 19.84
Memory: 920.14 MB
```

关注这里消耗的内存达到了 920MB。那是因为我们在内存中构建了整个CSV对象。这导致了大量的String对象被CSV库创建并使用了超过实际CSV文件大小的内存(恐怖的数量)

#### 解析CSV从内存中的String(CSV.parse)

```ruby

require_relative './helpers'
require 'csv'

print_memory_usage do
  print_time_spent do
    content = File.read('data.csv')
    csv = CSV.parse(content, headers: true)
    sum = 0

    csv.each do |row|
      sum += row['id'].to_i
    end

    puts "Sum: #{sum}"
  end
end

```
结果是：

```
$ ruby parse2.rb
Sum: 499999500000
Time: 21.71
Memory: 1003.69 MB
```

从结果我们可以看到,内存的增加,大概是CSV文件大小。(75 mb)

#### 从字符串在内存中逐行解析CSV(CSV.new)

```ruby
require_relative './helpers'
require 'csv'

print_memory_usage do
  print_time_spent do
    content = File.read('data.csv')
    csv = CSV.new(content, headers: true)
    sum = 0

    while row = csv.shift
      sum += row['id'].to_i
    end

    puts "Sum: #{sum}"
  end
end

```
结果是：

```
$ ruby parse3.rb
Sum: 499999500000
Time: 9.73
Memory: 74.64 MB
```

从结果我们可以看到,内存使用的文件大小(75 MB),因为文件内容加载到内存中,处理时间是快两倍。
这种方法是有用的,当我们的内容,我们不需要从文件读它,我们只是想遍历它逐行。

#### 逐行解析CSV文件IO对象
我们能比以前更好的脚本吗?是的,如果我们有CSV文件中的内容。让我们用一个IO直接文件对象

```ruby
require_relative './helpers'
require 'csv'

print_memory_usage do
  print_time_spent do
    File.open('data.csv', 'r') do |file|
      csv = CSV.new(file, headers: true)
      sum = 0

      while row = csv.shift
        sum += row['id'].to_i
      end

      puts "Sum: #{sum}"
    end
  end
end

```
结果是：

```
$ ruby parse4.rb
Sum: 499999500000
Time: 9.88
Memory: 0.58 MB
```

在上一个的脚本,我们看到小于1 MB内存增加。时间似乎是慢一些相比之前的脚本,
因为有更多的IO操作。CSV库已经建成的机制,CSV.foreach:

```ruby
require_relative './helpers'
require 'csv'

print_memory_usage do
  print_time_spent do
    sum = 0

    CSV.foreach('data.csv', headers: true) do |row|
      sum += row['id'].to_i
    end

    puts "Sum: #{sum}"
  end
end
```
结果是：

```
$ ruby parse5.rb
Sum: 499999500000
Time: 9.84
Memory: 0.53 MB
```

总结：从上面的例子，最佳的处理方式是最后两个方案。如果需要速度快，可以使用CSV.new的方案。但是会消耗文件大小的内存(如果你内存丰富的话可以考虑)
