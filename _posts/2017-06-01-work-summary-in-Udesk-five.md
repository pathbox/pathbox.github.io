---
layout: post
title: 最近工作总结(五)
date:   2017-06-01 11:08:06
categories: Work
image: /assets/images/post.jpg
---

##### 多个条件进行操作的时候，难免遇到冲突
比如触发器，不同的触发器的触发条件如果有冲突，顺序不同，影响不同。如果A触发器先执行更改记录，会导致B触发器条件不满足而无法执行。如果让B先执行，则两者都可以顺利执行。

##### 为Ubuntu升级JDK　到java 8
到官网下载　JDK-1.8　版本

mkdir -p /usr/lib/jvm　如果目录不存在。

将压缩包解压到　/usr/lib/jvm

sudo ln -s jdk1.8.0_131 java-8

 修改.bashrc　如果用zsh, 则修改.zshrc

```
export JAVA_HOME=/usr/lib/jvm/java-8  
export JRE_HOME=${JAVA_HOME}/jre  
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
export PATH=${JAVA_HOME}/bin:$PATH  
```

source ~/.bashrc  

配置默认JDK版本

```
sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/java-8/bin/java 300  
sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/java-8/bin/javac 300  
sudo update-alternatives --config java

选择 路径 优先级 状态  
------------------------------------------------------------  
0 /usr/lib/jvm/java-6-openjdk/jre/bin/java 1061 自动模式  
1 /usr/lib/jvm/java-6-openjdk/jre/bin/java 1061 手动模式  
2 /usr/lib/jvm/java-8/bin/java 300 手动模式  
要维持当前值 请按回车键，或者键入选择的编号：2  
update-alternatives: 使用 /usr/lib/jvm/java-8/bin/java 来提供 /usr/bin/java (java)，于 手动模式 中。
```

验证：

```
java -version
java version "1.8.0_131"
Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, mixed mode)

```

#####　关于全文搜索的优化和观点
如果客户能输入一个更有意义，更准确的词去进行全文搜索，这样才是最好的搜索优化。
因为，一个更合适的词，能够更好的从大数据中进行匹配筛选，得到的数据也是更接近于想要搜索得到的。
举个例子：想要得到天安门的地址。　如果你输入"的地"和输入"天安门"，显然"天安门"能够更准确和快速的搜索得到
想要的结果。

在项目中，使用了nGram这样的分词。这个分词的作用是: abc => [a,ab,abc, ac, b, bc,c]这样的分词。如果用这样的分词，
是能够通过"的地"把结构搜索出来。但是，这样的分词有性能问题，产生了大量实际中使用不到的token。而且，在产生大量分词的时候，
对CPU的使用率是非常消耗的。并且在搜索长的字符串的时候，也会产生性能问题，或者慢日志。虽然，这种分词能够涵盖的搜索情况最大，因为分的
token最多。但是，随着数据量的增加，如果到一定的数据量，ES会很快就到性能瓶颈。即使增加集群，也会随着数据量增加而很快到达瓶颈。而进入
性能黑洞。所以，使用IK或英文的分词，将有意义的词分成token，产生倒排索引。当用户能够输入有意义的词语进行搜索的时候，就能很好的搜索到结果。如果输入的是没有什么意义的词，就直接不匹配到结果，这是合理的情况。而兼顾没有意义的分词的情况，这不叫智能。这也失去了真正全文搜索的真实意义。我觉得全文搜索的真实意义是：得到更加精确的匹配结果，而不是得到更多匹配的结果。

##### ES同样有分页性能瓶颈
今天遇到一个ES的问题。ES在一段时间突然负载变大，涨了5倍。查找原因，分析日志后，结果是一个客户写了爬虫疯狂的爬我们的列表页面(他们把cookie的取出来用在爬虫中)，而这个列表页面用的是ES。然后，爬虫开始分页，从第一页爬到5w多页。系统中用的是普通的ES分页方法。from 数量，size 数量。这种分页方案在数据量大且翻页到一定大的页数时，会产生严重的性能问题。而且，对方不是爬一次就结束了，而是一分钟疯狂的爬。所以，导致了ES的性能问题。

##### rails controller callback 顺序
首先，controller 中的 callback是一个队列，当有同名的callback的时候，调用顺序是按照先进先出，也就是从上到下的顺序。

把before_action 和 after_action 看成等价的。所以，他们是按照队列顺序调用。

after_action 无论是否放在上面，都是最后执行的。

如果有prepend_xxx 这是最先执行的

prepend_before_action  prepend_after_action, prepend_around_action

```ruby
after_action :log_a
before_action :log_b

def index
  puts "run"
end

def log_a
  puts "===around action 1"
  yield
  puts "===around action 2"
end

def log_b
  puts "===before action"
end

# 这样的结果是

===around action 1
===before action
run
===around action 2
```

##### 关于tags标签表的设计
有时候系统中需要使用标签功能， 并且不是一个地方或一个表需要标签。

新建一个tags表，tags的 name 名称就是唯一标识的。

新建一个中间表， taggings表。 表中有 tag_id 和 tagging_id tagging_type

tags 和 对应的某个表就是 多对多的关系

例如： user、product、post等地方需要tag，就可以共用tags一张表，对taggings来说他们都是多态

##### ES mapping field customer.level  和level 的搜索情况

索引 tickets 其中有这样的mapping

```
"customer": {
              "properties": {
                 "level": {
                    "type": "string",
                    "index": "not_analyzed"
                 },
                 "nick_name": {
                    "type": "string",
                    "analyzer": "char_split"
                 }
              }
           }
```

进行下面搜索时
```
1.

GET tickets/ticket/_search
{
    "query":{
    "filtered":{
        "filter":{
            "bool":{
                "must": [
                   {"term":{
                   "company_id": 1
                   }},
                   {"term":{
                   "customer.level":"vip"
                   }}
                ]
            }
        }
    }
  }
}

2.
GET tickets/ticket/_search
{
    "query":{
    "filtered":{
        "filter":{
            "bool":{
                "must": [
                   {"term":{
                   "company_id": 1
                   }},
                   {"term":{
                   "level":"vip"
                   }}
                ]
            }
        }
    }
  }
}
```

这两种搜索都能得到结果。但个人会选择第一种方式，避免以后有可能造成的冲突

##### Rails send_data and send_file

```ruby
send_data(data, options = {})
send_file(path, options = {})

Main difference here is that you pass DATA (binary code or whatever) with send_data or file PATH with send_file
So you can generate some data and send it as an inline text or as an attachment without generating file on your server via send_data. Or you can send ready file with send_file

```

```ruby
data = "Hello World!"
send_data( data, :filename => "my_file.txt" )
```

OR

```ruby
data = "Hello World!"
file = "my_file.txt"
File.open(file, "w"){ |f| f << data }
send_file( file )
```

##### 使用oink分析内存问题
最近公司的服务器内存涨了4G，是在上回发版后开始上升。在访问请求并没有上涨的情况下，增加了这么多内存。公司原来就有装oink这个公司。

登入一台服务器，然后 输入  bundle exec oink --threshold=10 log/oink.log
--threshold=10 表示10M内存

可以得到以下的信息(这是在测试服务器上测试的)：
```
---- MEMORY THRESHOLD ----
THRESHOLD: 10 MB

-- SUMMARY --
Worst Requests:
1. Jun 16 15:03:08, 87944 KB, home#index
2.
3.
......
10.

Worst Actions:
23, home#index

Aggregated Totals:
Action                         	Max	Mean	Min	Total	Number of requests
home#index                     	53260	17994	10964	197944	11
```

然后可以查看哪些action和请求url是消耗内存较多的，查看这些action的相关代码。查出一个昨天刚离职的同事之前写了一个优化count语句的代码修改，他把所有相关id找出来，然后存在数组中在size来优化原来的count。。。这样这个数据可以有几十万个元素，内存自然暴涨了

oink果然是个神器

##### rails 上传文件file 的 read 和 rewind
从params[:file] 上传得到一个文件流对象。rails会帮你封装好这个对象并且提供许多方法。

file = params[:file]

当file.read , 再执行 file.read的时候， 是读取到""。 因为file已经被读过了，对应的文件指针应该是到了最尾部。此时的file的流内容是空的。如果，需要重新读这个file的内容，需要 file.rewind。 这样，之后再执行 file.read 就又可以读取到内容。

file 还有一个重要的属性，就是tempfile。file.tempfile.path，可以得到临时文件的位置。实际上，上传文件操作，服务器先生成了一个临时文件，在对这个临时文件进行读取或者是另存为一个新的文件，最后这个临时文件就没有用了，可以被删除。
