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
