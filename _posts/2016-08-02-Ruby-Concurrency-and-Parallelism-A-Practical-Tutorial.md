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

用多进程的Ruby解决方案实例：

Resque：一个基于redis存储的Ruby库，用于创建后台job任务，把jobs任务放入多个队列，然后执行。

Unicorn： 用于服务Rack 应用的HTTP服务器。能服务快速的客户端，特点是低延迟，高带宽的连接，在Unix-like核心服务器上有不错的性能

用多线程的Ruby解决方案实例：

Sidekiq：一个全功能的后台处理Ruby框架。它的目标是简单的与任何现代Rails应用程序集成和更高的性能比其他现有的解决方案

Puma：一个支持并发的Ruby web服务器。

Thin：一个快速简单的Ruby web服务器。

### 多进程

在我们看着Ruby多线程选择之前,让我们探索简单生成多进程的方法。

在Ruby中，fork() 系统调用是用于创建一个当前进程的“复制”。这个新的进程将在操作系统级别，所以它可以并发运行与
原来的程序，就像任何其他的独立过程。(注意： fork()是一个POSIX系统调用，因此在windows平台是不可用的)

OK，让我们运行测试例子，但是 这次用fork()去启动多进程：

```ruby
puts Benchmark.measure{
  100.times do |i|
    fork do     
      Mailer.deliver do 
        from    "eki_#{i}@eqbalq.com"
        to      "jill_#{i}@example.com"
        subject "Threading and Forking (#{i})"
        body    "Some content"
      end
    end
  end
  Process.waitall
}
```
Process.waitall 等待所有的子进程退出和返回进程状态的数组。

代码执行结果：

```ruby
	0.000000   0.030000  27.000000 (  3.788106)
```
不是太寒酸!我们使Mailer快了5倍,只需要修改几行代码(即：使用fork())。

不要过于兴奋。虽然尝试使用forking 是一个简单的Ruby并发解决方式，但是它有意个主要的缺陷是：
这样消耗的内存数量。使用fork有点昂贵，特别是一个一个 Copy-on-Write 不是利用你的Ruby解释器。
如果你的app应用使用20mb的内存，fork 100次，就需要消耗2GB的内存！

### Ruby的多线程

OK，现在让我们尝试用Ruby的多线程技术来处理这个程序。

多个线程在一个过程开销大大低于相应数量的进程因为他们共享地址空间和内存

考虑到这一点,让我们重新审视我们的测试用例,但这一次使用Ruby的线程类:

```ruby
threads = []

puts Benchmark.measure{
  100.times do |i|
    threads << Thread.new do     
      Mailer.deliver do 
        from    "eki_#{i}@eqbalq.com"
        to      "jill_#{i}@example.com"
        subject "Threading and Forking (#{i})"
        body    "Some content"
      end
    end
  end
  threads.map(&:join)
}
```

代码执行结果：

```
13.710000   0.040000  13.750000 ( 13.740204)
```

这肯定不是令人印象深刻!那么发生了什么?为什么这个生产几乎相同的结果正如我们当我们跑了代码同步?

答案,存在许多Ruby程序员的克星，全局解释器锁(GIL)。由于GIL，CRuby(MRI实现)并不支持并行线程。

全局解释器锁是一种机制中使用计算机语言翻译同步线程的执行这一次只有一个线程可以执行。解释器使用GIL将
允许在某个时刻只有一个线程在执行。即使是在多核处理器上运行。Ruby MRI和CPython的两个最常见的例子是流行的解释器,，拥有GIL。
回到我们的问题,我们如何利用多线程在Ruby中提高性能的GIL?

在MRI(CRuby)，不幸的答案是,基本上你只能有很少的,多线程可以为你做的。

Ruby并发没有并行性仍然可以是非常有用的，虽然，对于IO密集型任务。所以，线程任然是有用的在MRI中，用于IO-heavy任务。
显存是有原因的，毕竟，发明和使用多核服务器之前就很常见。

但说,如果你选择使用成为CRuby以外的一个版本,您可以使用另一个Ruby实现,例如JRuby或Rubinius,因为他们没有一个GIL和真正支持Ruby线程并行。

图

### 线程不是安全的

