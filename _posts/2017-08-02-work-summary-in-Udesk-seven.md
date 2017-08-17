---
layout: post
title: 最近工作总结(七)
date:   2017-08-02 16:55:06
categories: Work
image: /assets/images/post.jpg
---

##### 使用正则快速获取括号内的字符串
result = file_name.match(/\A.*\((\S+)\).*\Z/)
result1 = file_name.match(/\A.*\[(\S+)\].*\Z/)
 p $1
 p result.to_a

##### 构建了一个sidekiq worker的思考
当你新建了一个sidekiq worker，需要思考这个sidekiq woker 是否在你的使用环境需要真的被新建，
是否有条件满足的时候才需要新建这个worker，比如某个开关，或某个实例对象存在。

当新建出一个sidekiq worker，思考其中的代码逻辑，是否不再满足某种条件的时候，就直接return，而不再让代码继续执行。

思考上面的两点，可以减少worker的新建次数，可以然后worker真正有效的执行，减少不必要的执行次数

#####  使用sendcloud API发送邮件
阿里云机器把25端口封了，导致smtp发送邮件需要使用25端口，而无法发送邮件。sendcloud提供了API的方式调用发送邮件，
使用sendcloud API发送邮件，解决这个问题

##### 何时需要消息队列
做业务解耦/最终一致性/广播/错峰流控等。反之，如果需要强一致性，关注业务逻辑的处理结果，则RPC显得更为合适

##### Kafka为什么那么快
Kafka速度的秘诀在于，它把所有的消息都变成一个的文件。通过mmap(Memory Mapped Files)提高I/O速度，写入数据的时候它是末尾添加(顺序写入)所以速度最优；读取数据的时候配合sendfile直接暴力输出。阿里的RocketMQ也是这种模式，只不过是用Java写的

##### 增加edge_ngram tokenizer, 优化前缀搜索

rails_script1.rb
```ruby
# rails r rails_script1.rb
# 为索引增加一个新的analyzer

new_settings = {
  analysis: {
    analyzer: {
      edge_prefix_split: {
        type: 'custom',
        tokenizer: 'edge_ngram_tokenizer'
      }
    },
    tokenizer: {
      edge_ngram_tokenizer: {
        type: 'edge_ngram',
        min_gram: 1,
        max_gram: 10,
        token_chars: ['letter','digit']
      }
    }
  }
}

client = Elasticsearch::Model.client.indices
timestamp = Time.now
client.close index: 'customers'
client.put_settings index: 'customers', body: new_settings
client.open index: 'customers'
```

rails_script2.rb
```ruby
# rails r rails_script2.rb
# 在原来索引基础上，增加一个field
indexes :nick_name, type: 'multi_field' do
  indexes :prefix, analyer: 'edge_prefix_split'
end

client = Elasticsearch::Model.client
client.indices.put_mapping index: "users", type: "user", body: {
  user: {
    properties:{
      name: {
        type: "string",
        index: "no",
        fields: {
          prefix: {
            type: "string",
            analyzer: "edge_prefix_split"
          }
        }
      }
    }
  }
}
```

如果你没有使用multi_fiel(elasticsearch 5.0以上已经废弃了)，你需要

```
indexes :name do
  indexes :prefix, type: :string, analyzer: 'edge_prefix_split'
end

client = Elasticsearch::Model.client
client.indices.put_mapping index: "users", type: "user", body: {
  user: {
    properties:{
      name:{
        type: :string, analyzer: :edge_prefix_split
      }
    }
  }
}

```
最后，再慢慢的为所有记录执行：  __elasticsearch__.index_document。 将每个记录的nick_name.prefix，使用 analyzer edge_prefix_split进行分词。

在搜索的时候，使用 {prefix: {"name.prefix"=>"#{search_value}"}} 就可以了。

edge_ngram 真的是天生用于前缀搜索的tokenizer

##### 关于服务器TCP最大连接数
ulimit -n 65536 设为65536

服务端的最大TCP连接数没有这个限制

服务器中的客户端TCP连接数有这个限制。比如：nginx的处理连接数，使用数据库的TCP连接数

##### Vistor项目出现端口无法访问的阿里云报错
原因是，服务器TCP最大连接数为65536，并发数高的时候，TCP连接数达到了上限，使得新的TCP请求该端口没有可用的连接数文件符，使得产生了大量的半连接。
而这些请求自然是无法连接成功。
