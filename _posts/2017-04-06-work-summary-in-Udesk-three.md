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
