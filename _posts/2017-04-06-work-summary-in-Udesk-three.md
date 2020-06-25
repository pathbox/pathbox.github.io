---
layout: post
title: 最近工作总结(三)
date:   2017-04-06 17:32:06
categories: Work
image: /assets/images/post.jpg
---

##### !!符号
!! 符号可以将nil转为true之后，再转为false。这样可以将false或nil都以false结果进行判断

```ruby

!!(Integer(id) rescue nil)
```
##### 调用者可信与传送数据可信
调用者可信 只需要双方定义一个密文，比如token。A方构造token给B方，
B方根据相同的算法构造token和传过来的token比较是否相同即可。

传送数据可信 最好的例子就是https了。用非对称加密的方式，加密传递的数据。接到数据后再解密。
保证了在传输过程中都不会泄露数据

##### order by 利用索引优化，你可以看
http://stackoverflow.com/questions/12148943/mysql-performance-slow-using-filesort

##### some ES
ES支持给索引预定义property和mapping
ElasticSearch refresh操作只是写到文件缓存系统
当 segment 刷到磁盘，translog 才进行清空

##### 邮件问题
密送的邮件收信人，Postfix没能收到密送的收信人地址

> Postfix -> Ruby script -> (post) parse build eml file -> post proj action parse build mail(database operation)

在通过subject取id信息的时候，subject编码分成了两行，由于id是在后面，也就是在第二行编码中
创建一个新的eml文件的时候，丢失了id内容，传到下一个action时候，解析拆分的方法没把id解析到，导致了异常
原因是 gem mail 2.5.4的一个bug，2.6.4之后修复了这个bug，所以需要升级使用的mail gem

##### render is not return
在controller中，render is not return

```ruby
def create
  user = User.Create(params)
  code = user ? 200 : 500
  render :json => {code: code}
  UserExampleWorker.perform_async(params)  # 上面及时render了，这一句也会继续执行下去。redirect_to 和这里类似
end

```

##### 序列化和反序列化

主要用于存储对象状态为另一种通用格式，比如存储为二进制、xml、json等等，把对象转换成这种格式就叫序列化，而反序列化通常是从这种格式转换回来。
使用序列化主要是因为跨平台和对象存储的需求，因为网络上只允许字符串或者二进制格式，而文件需要使用二进制流格式，如果想把一个内存中的对象存储下来就必须使用序列化转换为xml（字符串）、json（字符串）或二进制（流）

example
```
内存对象 => 序列化 => JSON
JSON => 反序列化 => 内存对象
```

##### 为空
cellphone = "" 或者 cellphone 不传 是可以判断电话号码为空
而 cellphone = "null" 这个不是为空。。。。。。
不同语言交互的接口中，建议为空别传nil，不同语言中的nil表示的可能会不一样，从而产生了上面的问题
所以，用""，空字符串或者就不传这个参数表示为空是合理的

##### Maybe it is not good way to use ES
对于超过1M，大于字节长度为100w的字符串的字段存到ES并且进行分词时，CPU使用率一下飙到了100%
非常容易造成ES的其他进程阻塞或ES直接负载过高而崩溃。所以，对于长度超过10w的字符串，建议不再存储到ES中，
对于有可能长度超过10w的字段，直接过滤，在ES中存空字符串。ES很强大，但是不能滥用。否则，ES系统并不稳健。

##### 图片存到了字段中
图片以二进制字符串的形式存储到了字段中，这是不好的选择。应该让图片，以文件形式存到服务器，字段中存储url。
以二进制字符串的形式存储到字段中的意义不大，除非就是需要这样做。有的前端插件，再文本输入框中，拖拽图片进去然后提交，
会导致将图片以二进制字符串的形式存储到content字段中。

##### MySQL is too busy
MySQL IO 几乎快满了，会影响到回调的时候 同步ES的操作，导致数据没有同步ES成功
当ES磁盘快满的时候，也会导致无法再往ES中写入数据

##### custom field
在新增自定义字段的时候，要给使用了自定义字段的表的ES，比如(Ticket) 使用回调方法，为对应的ES新增这个新增的自定义字段属性
You know this is called predefine mapping

##### Outlook email
Why Outlook email and sina email is different

##### websocket-bench with http
当用websocket-bench 模拟websocket做压力测试的时候，别忘了url要以http:// 开头。。。

##### go-socket.io
今天做测试，用websocket-bench 压测。 项目用的是go-socket.io 底层其实用的是 gorilla/websocket
建立一万个空连接，消耗了1g的内存

##### Why we use verification code ？
可以用来防止简单的爬虫和无意义的请求，提高网站的安全性

##### Callback is not agent in Rails
我们经常在Rails项目中使用各种回调。比如在model层使用after or before 回调，这方便且友好的帮我们处理了各种逻辑。当一个系统变得越来越庞大，越来越多的逻辑被加入到回调中，这时候，回调中的逻辑变得耦合或者是互相影响。Now, Callback become devil.
总结几点使用回调的注意点：
１　回调的逻辑尽量单一，尽量一个回调只做一件事
２　回调中尽量不要混合多种逻辑，将不同的逻辑分到不同的回调中
３　使用异步队列方式代替回调
４　查看回调是否能容忍报异常中断逻辑，如果不能，请捕捉异常
５　回调中如果有不同的写操作，考虑这些不同的写操作之间的耦合性
现在系统中，有的回调逻辑很复杂，使得排错处理难度很大。
在处理工单回调比如触发器等，其中又有ＥＳ同步的回调，使得当触发器的回调中报异常中止的话，会导致ＥＳ同步没有执行，使得使用回调机制同步ＥＳ很脆落

```ruby
after_save :put_test
after_save :raise_test

def raise_test
  raise "now it is raise"
end

def put_test
  puts "===================="
  self.name = 'Name'
end
#
# (0.2ms)  BEGIN
#   SQL (0.4ms)  INSERT INTO `cities` (`code`, `name`) VALUES (1000, 'beijing')
# ====================
#    (2.3ms)  ROLLBACK
# RuntimeError: now it is raise
# from /home/user/Documents/udesk_proj/app/models/city.rb:10:in `raise_test'
# [5] pry(main)> c
# => #<City id: 3, code: 1000, name: "nono", province_id: nil>

# 上面代码等价于

after_save :put_and_raise

def put_and_raise
  puts "===================="
  self.name = 'Name'
  raise "now it is raise"
end
```

即使分开写了两个after_save方法，但是执行顺序是队列，从上往下执行。如果其中有错误中断了，则不会继续执行下去。
总结就是，相同的回调操作是按照队列顺序进行，并且先执行的逻辑会影响后执行的逻辑方法
