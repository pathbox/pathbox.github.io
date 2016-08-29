---
layout: post
title: Ruby Concurrency and Parallelism A Practical Tutorial(翻译)
date:   2016-08-02 21:30:06
categories: Ruby
image: /assets/images/top/20150404.jpg
---

特别的，Ruby并发指的是：两个任务可以在重叠的时间段启动、运行和完成。
不过,这并不意味着他们都是运行在同一瞬间(比如：多个线程在单核机上运行)
与此形成鲜明对比的是，并行是指：两个任务运行在相同的时间。(比如：多线程运行在多核心的处理器上)

![1]( /assets/images/20160802/1.png "Optional title")

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

![2]( /assets/images/20160802/2.png "Optional title")

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

![3]( /assets/images/20160802/3.jpg "Optional title")

### 线程不是免费的

提高性能的多个线程可能会让人相信我们可以继续添加更多的线程-基本上无限继续使我们的代码运行得越来越快。
这确实会好如果它是真的,但现实是,线程不是免费的,迟早,你会耗尽资源。

例如,我们想要运行示例Mailer不是100次,10000次。让我们看看会发生什么:

```ruby
threads = []

puts Benchmark.measure{
  10_000.times do |i|
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

Boom!我在OS X 10.8系统下有一个错误当产生了2000个线程:

```
can't create Thread: Resource temporarily unavailable (ThreadError)
```

正如预期的那样,迟早我们开始抖动或完全耗尽资源。所以这种方法的可伸缩性显然是有限的。

### 线程池
幸运的是,有一个更好的办法,即线程池

线程池是一组预先实例化可重用线程可用来执行工作。线程池尤其有用,当有大量的短期任务要执行,而不是一个小数量的时间任务。
这可以防止不得不承担创建一个线程的多次创建的开销。

池递给一个任务执行时,它分配任务当前的空闲线程。如果没有空闲线程(并且已经创建的线程的最大数量)它等待一个线程完成工作并成为空闲线程,
然后分配任务。

![4]( /assets/images/20160802/4.jpg "Optional title")

回到我们的例子中,我们将首先使用队列(因为它是线程安全的数据类型)和使用一个简单的线程池的实现:

```ruby
POOL_SIZE = 10

jobs = Queue.new

10_0000.times{ |i| jobs.push i}

workers = (POOL_SIZE).times.map do
	Thread.new do
		begin
			while x = jobs.pop(true)
				Mailer.deliver do
					from    "eki_#{x}@eqbalq.com"
          to      "jill_#{x}@example.com"
          subject "Threading and Forking (#{x})"
          body    "Some content"
        end
      end
    rescue ThreadError
    end
	end
end

workers.map(&:join)
```

在上面的代码中,我们开始通过创建一个工作队列需要执行的工作。
为此我们使用队列因为它是线程安全的(如果多个线程同时访问它,它将保持一致性),
避免了需要一个更复杂的实现需要使用互斥锁。

之后我们把邮件的IDs 放入jobs 队列，和创建了10个worker线程。

在每个worker线程中，我们从jobs队列中pop取出每个item

因此,一个工作线程的生命周期不断等待任务放入作业队列并执行。

好消息是，这个工作和扩展没有任何问题。不幸的是，虽然这是相当复杂的，即使对于我这个简单的教程。

### Celluloid

多亏了Ruby Gem 的生态系统，很多复杂的多线程被封装在了一个易于使用的Ruby Gem中，而且开箱即用。

一个很好的例子是Celluloid,我最喜欢的一个ruby gems。赛璐珞框架是一个基于actor，在Ruby中简单干净地方法来实现并发系统。
Celluloid使人们构建并发程序的并发对象一样容易他们建造顺序程序的顺序对象

在我们的讨论的背景下在这篇文章中,我特别关注池功能,但帮自己一个忙,更详细地检查一下。
使用赛璐珞你将能够构建多线程Ruby程序而不用担心讨厌的死锁问题,你会发现它容易使用其他更复杂的功能,如期货和承诺。

这里就是简单的多线程版本的Mailer使用赛璐珞程序:

```ruby
require "./lib/mailer"
require "benchmark"
require "celluloid"

class MailWorker
  include Celluloid

  def send_email(id)
    Mailer.deliver do
      from    "eki_#{id}@eqbalq.com"
      to      "jill_#{id}@example.com"
      subject "Threading and Forking (#{id})"
      body    "Some content"
    end       
  end
end

mailer_pool = MailWorker.pool(size: 10)
10_0000.times do |i|
	mailer_pool.async.send_email(i)
end
```

干净，简单，可扩展和健壮的。你还想要什么？

### Background Jobs

当然,另一个潜在可行的选择,这取决于您的业务需求和约束将雇佣背景的工作。许多Ruby Gems支持后台处理(即存在。在一个队列,节省工作和处理以后没有阻塞当前线程)。
著名的例子包括Sidekiq,Resque,Delayed Job,Beanstalkd。

在这篇文章中,我将使用Sidekiq和redis(一个开源的键值缓存和存储)。

```ruby
	require_relative "../lib/mailer"
  require "sidekiq"

  class MailWorker
  	include Sidekiq::Worker

  	def perform(id)
	    Mailer.deliver do
	      from    "eki_#{id}@eqbalq.com"
	      to      "jill_#{id}@example.com"
	      subject "Threading and Forking (#{id})"
	      body    "Some content"
	    end  
	  end
	end
```

我们可以触发Sidekiq mail_worker.rb文件:

```
sidekiq  -r ./mail_worker.rb
```

And then from IRB:

```
⇒  irb
>> require_relative "mail_worker"
=> true
>> 100.times{|i| MailWorker.perform_async(i)}
2014-12-20T02:42:30Z 46549 TID-ouh10w8gw INFO: Sidekiq client with redis options {}
=> 100
```

这可不简单。它可以很容易通过改变工人的数量规模。

另一个选择是使用SuckerPunch,我最喜欢的一个异步RoR处理库。实现使用出其不意将是非常相似的。
我们只需要包括SuckerPunch::工作而不是Sidekiq::Worker,MailWorker.new.async.perform(),
而MailWorker.perform_async()。

### 结论

在Ruby高并发性不仅是可行的,而且比你想象的简单。

一个可行的方法是简单地叉一个运行的进程将其处理能力。另一种方法是利用多线程。虽然线程更轻比进程,需要较少的开销,你仍然可以耗尽的资源如果你开始同时太多的线程。
在某种程度上,你可能会发现有必要使用一个线程池。
幸运的是,许多复杂的多线程更容易利用任何可用的Ruby,如Celluloid和Actor模型。

处理耗时的过程的另一种方法是通过使用后台处理。
有很多库和服务,允许您在应用程序中实现后台作业。一些流行的工具包括数据库支持的工作框架和消息队列。

分叉、线程和后台处理都是可行的选择。
决定使用哪一个取决于您的应用程序的本质,你的操作环境,和要求。
希望本教程提供了一个有用的介绍可用的选项。
