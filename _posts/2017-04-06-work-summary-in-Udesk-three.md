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
