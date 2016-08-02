---
layout: post
title: Ruby Concurrency and Parallelism A Practical Tutorial(翻译)
date:   2016-08-02 21:30:06
categories: Ruby
image: /assets/images/post.jpg
---

特别的，Ruby并发指的是：两个任务可以在重叠的时间段启动、运行和完成。
不过,这并不意味着他们都是运行在同一瞬间(比如：多个线程在单核机上运行)
与此形成鲜明对比的是，并行是指：两个任务运行在相同的时间。(比如：多线程运行在多核心的处理器上)

图

这个关键点是并发的线程或进程不是必须的要并行的运行。
本教程提供了一个实用的(而不是理论)在Ruby中适用的各种技术和方法可用于并发和并行性。

### Our Test Case

一个简单的测试例子。我将创建一个Mailer 类和增加Fibonacci 函数构造CPU密集型的请求操作。

```ruby
	class Mailer

  def self.deliver(&block)
    mail = MailBuilder.new(&block).mail
    mail.send_mail
  end

  Mail = Struct.new(:from, :to, :subject, :body) do 
    def send_mail
      fib(30)
      puts "Email from: #{from}"
      puts "Email to  : #{to}"
      puts "Subject   : #{subject}"
      puts "Body      : #{body}"
    end

    def fib(n)
      n < 2 ? n : fib(n-1) + fib(n-2)
    end  
  end

  class MailBuilder
    def initialize(&block)
      @mail = Mail.new
      instance_eval(&block)
    end
    
    attr_reader :mail

    %w(from to subject body).each do |m|
      define_method(m) do |val|
        @mail.send("#{m}=", val)
      end
    end
  end
end
```

我们可以调用这个 Mailer类发送邮件如下:

```ruby
Mailer.deliver do 
  from    "eki@eqbalq.com"
  to      "jill@example.com"
  subject "Threading and Forking"
  body    "Some content"
end
```
建立一个进行比较目的,我们先做一个简单的基准,调用Mailer100次。

```ruby
puts Benchmark.measure{
  100.times do |i|
    Mailer.deliver do 
      from    "eki_#{i}@eqbalq.com"
      to      "jill_#{i}@example.com"
      subject "Threading and Forking (#{i})"
      body    "Some content"
    end
  end
}
```
在四核CPU和 MRI Ruby 2.0.0版本中，运行结果：

```
15.250000   0.020000  15.270000 ( 15.304447)
```

### Multiple Processes vs. Multithreading
没有“一刀切”答案的时候决定是否使用多个进程或者Ruby应用程序多流。下面的表总结了需要考虑的一些关键因素

图

