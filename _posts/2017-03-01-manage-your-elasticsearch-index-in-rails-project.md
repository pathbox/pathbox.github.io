---
layout: post
title: Manage and custom your elasticsearch index in Rails project
date: 2017-03-01 17:23:06
categories: Elasticsearch
image: /assets/images/elasticsearch.png
---

这周对Rails项目中的ElasticSearch进行了总结并写成了文档，觉得有一些内容值得记录和分享。
这篇文章主要是在Rails项目中对索引，mapping的设置、管理操作的总结，不涉及搜索方面的内容。
源数据是存在MySQL中，ES的数据是MySQL写操作的时候进行了回调同步到ES中，这个应该和很多人的使用同步策略一样。

#### 1.Model#mapping

在model文件中，使用mapping的声明方式。

```ruby
class Article < ActiveRecord::Base
  include Elasticsearch::Model
  include Elasticsearch::Model::Callbacks
  index_name self.table_name  # 这里可以自定义Article的ES索引名称

  mapping do
    indexes :subject, type: 'multi_field' do
      indexes :raw, index: :not_analyzed
      indexes :tokenized, analyzer: 'ik'
    end
    indexes :content, type: :string, analyzer: 'ik'
  end
end
```

```ruby
indexes :subject, type: 'multi_field' do
  indexes :original, index: :not_analyzed
  indexes :tokenized, analyzer: 'ik'
end
```

使用了 *multi_field* 定义了两个indexes， 这样在ES中会产生下面的mapping结构

```
"subject": {
   "type": "string",
   "index": "no",
   "fields": {
      "original": {
         "type": "string",
         "index": "not_analyzed"
      },
      "tokenized": {
         "type": "string",
         "analyzer": "ik"
      }
   }
}
```

对于MySQL articles表的subject数据，在ES的articles索引中用了两个field来存储 subject.original 和 subject.tokenized 。他们存储的数据是一样的，不同的是subject.original 没有被分词，subject.tokenized 使用了*ik*进行了分词。这样做的效果是当想要对subject进行全文搜索时，就可以使用subject.tokenized， 想要对subject进行条件过滤的时候，就可以使用subject.original了。

```
Tip: ik 是一个优秀的中文分析器 [github地址](https://github.com/medcl/elasticsearch-analysis-ik)
```

我们知道  include Elasticsearch::Model::Callbacks 这个模块帮我们做了：

```ruby
def self.included(base)
  base.class_eval do
    after_commit lambda { __elasticsearch__.index_document  },  on: :create
    after_commit lambda { __elasticsearch__.update_document },  on: :update
    after_commit lambda { __elasticsearch__.delete_document },  on: :destroy
  end
end
```
就是写操作的模块。

我们也可以这样做：

```ruby
after_commit :create_es_index, on: :create

def create_es_index
  begin
    __elasticsearch__.index_document
  rescue => e
    hash = {}
    hash['article_id'] = self.id
    hash['time'] = Time.now
    hash['error'] = {}
    hash['error']['message']   = e.message
    hash['error']['backtrace'] = e.app_backtrace
    ErrorESMailer.send_error_email(hash).deliver  # send a error email
  end
end
```

这样，如果有回调的同步ES的操作有异常，则会捕获这个异常并且发送error的相关信息邮件给开发人员，开发人员可以及时的处理异常。同理 update 和 destroy操作。



#### 2. Model.import  force:true

```ruby
Article.import force:true
```

ES 会根据Article 中setting 和mapping的配置，在ES中 构建articles的索引(清空新建索引+导入数据)，对应的type为article。

下面的三个操作，同样可以创建索引并导入数据

```ruby
1. Article.__elasticsearch__.create_index! force: true  # 根据mapping和setting 创建articles索引，该索引没有任何数据
2. Article.__elasticsearch__.refresh_index!  # refresh 操作
3. Article.find_in_batches do |articles|   # 批量同步导入MySQL articles的数据
    Article.__elasticsearch__.client.bulk({
      index: 'articles',
      type: 'article',
      body: articles.map do |article|
        {index: {_id: article.id, data: article.as_indexed_json }}
      end
    })
   end
```

假设，你需要往ES articles索引中同步一批数据，这批数据已经被导入到了MySQL中，如果数据量不大，使用import: true的方法快速导入完。如果数据量比较大，导入耗时很多。还使用import的方式导入的话，会导致在ES articles被清空后，如果线上有用户在对articles的数据进行ES的搜索，很有可能会导致没有搜索结果。这时候，你可以使用上面的第三个操作，不需要重建整个articles索引。而是往articles索引中添加索引数据。

#### 3. as_indexed_json

*as_indexed_json* 是一个很重要的方法。如果你明白了它，你就能知道ES索引中存的documents数据是怎样的。我们知道ES不是关系型数据库，ES中的documents数据和MongoDB的documents类似，每个document是一个json。所以，知道索引中documents数据存了哪些fields，你才能更好地结合灵活多变的搜索条件和方式，构造多种多样的搜索情况。

下面我们看*as_indexed_json* 的源码:

```ruby
# File 'lib/elasticsearch/model/serializing.rb', line 26
def as_indexed_json(options={})
  # TODO: Play with the `MyModel.indexes` method -- reject non-mapped attributes, `:as` options, etc
  self.as_json(options.merge root: false)
end
```

这里的self 其实是model 的一个instance。我们可以在model中monkey patch这个方法。

```ruby
# article.rb
class Article < ActiveRecord::Base
  include Elasticsearch::Model
  include Elasticsearch::Model::Callbacks
  index_name self.table_name  # 这里可以自定义Article的ES索引名称

  has_many :comments
  has_many :followers

  mapping do
    indexes :subject, type: 'multi_field' do
      indexes :raw, index: :not_analyzed
      indexes :tokenized, analyzer: 'ik'
    end
    indexes :content, type: :string, analyzer: 'ik'
    indexes :created_at, type: :date
  end

  def as_indexed_json(options={})
    hash = as_json(
      except: [:update_at],
      methods: [:parse_content],
      include: {
        comments: {only: [:id, :content]},
        followers: {only: [:id]}
      }
    )
    hash.merge!(other_hash)
    hash
  end

  def other_hash
  	{title: "My title", owner: "My owner"}
  end

  def parse_content
  	"Article: "+ self.content
  end
end
```

我们重写的*as_indexed_json* 方法构造了一个hash然后返回。except 表示排除某个字段，这个字段不会出现在hash的key中，methods 表示将某个方法作为hash的key，对应的value就是该方法的返回值。include 表示会找相关联的表，并取定义的字段作为hash的key，value则为该字段的值。

这样，最后得到的hash 大概是这样的, 比如 Article.first.as_indexed_json:

```ruby
{
  'id'=> 1,
  'subject' => '这是第一篇文章的主题',
  'content' => '这是第一篇文章的内容',
  'title' => 'My title',
  'owner' => 'My owner',
  'created_at' => 'Tue, 02 Aug 2016 20:40:08 CST +08:00'
  'parse_content' => 'Article: 这是第一篇文章的内容',
  'comments' => [{'id'=> 1, 'content'=> '点赞'},{'id'=> 2, 'content'=> 'good'}],
  'followers' => [{'id'=> 1},{'id'=> 2}]
}
```

通过这个事例，你应该能明白*as_indexed_json* 中可以如何构造想要的hash数据了。

在什么时候用到 *as_indexed_json*方法呢？

在index_document 方法的源码中:

```ruby
# File 'lib/elasticsearch/model/indexing.rb', line 333
def index_document(options={})
  document = self.as_indexed_json # Hi! I'm here!

  client.index(
    { index: index_name,
      type:  document_type,
      id:    self.id,
      body:  document }.merge(options)
  )
end
```

这样就明白了，当一个model instance(一条记录) 从MySQL同步到ES的时候, 将这条记录转为hash结构。其实，*as_indexed_json* 返回的这个hash就是 ES 对应的一条 document。这个过程就是：

```ruby
MySQL one record => as_indexed_json => ES one document
```

现在我们明白了*as_indexed_json*，就可以根据自己的需要，往ES索引中存储documents数据了。并不是ES索引的mapping field 只能和数据库对应表的字段一样，如果你什么都不做，ES会用默认的配置帮你做这些事情。如果你想要存储更多的数据字段到ES的document中进行索引分词，你就需要自己动手做更多的自定义的配置。合理的documents数据是ES进行高效搜索功能的保证。

#### 4. ES的mapping只能增加

ES的mapping一旦创建，只能增加字段，而不能修改已经mapping的字段。

```ruby
client = Elasticsearch::Model.client
client.indices.put_mapping index: "articles", type: "article", body: {
  article: {
    properties: {
      organization: {
        properties: {
          id:   { type: :integer },
          name: { type: :string, index: :not_analyzed }
        }
      }
    }
  }
}
```

得到的mapping 是：

```ruby
"organization": {
  "properties": {
    "id": {
      "type": "integer"
      },
    "name": {
      "type": "string",
      "analyzer": "not_analyzed"
      }
    }
  }
```

上面的方法代码可以写成rails脚本，使用rails runner 执行。也可以写成rake命令。然后，建议把新增的mapping field 写在Article 的mapping定义中。让mapping定义在代码层面保持显示最完整的定义结构。

#### 5. 设置自定义的analysis

什么是 analysis、analyzer、tokenizer和 filter，如果不知道或者有遗忘了，请看[ElasticSearch的官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/analysis.html) 。官方文档描述的简单清晰，例子也易懂。

在上面的Article例子中，我们没有对analysis进行设置，这样，ES会使用默认的analysis设置。但是，也许默认的设置并不是我们真正想要的analysis。我们现在开始自定义analysis。

```ruby
# article.rb
class Article < ActiveRecord::Base
  include Elasticsearch::Model
  include Elasticsearch::Model::Callbacks
  index_name self.table_name  # 这里可以自定义Article的ES索引名称

  has_many :comments
  has_many :followers

  settings analysis:{
    analyzer: {
      my_custom_analyzer:{ type: 'custom', tokenizer: 'ngram_tokenizer'}
    },
    tokenizer: {
      ngram_tokenizer: { type: 'nGram', min_gram: 2, max_gram: 3, token_chars: ['lettler, 'digit', 'punctuation']}
    }
} do
    mapping do
      indexes :subject, type: 'multi_field' do
        indexes :raw, index: :not_analyzed
        indexes :tokenized, analyzer: 'ik'
      end
      indexes :content, type: :string, analyzer: 'ik'
      indexes :created_at, type: :date
    end
  end

  def as_indexed_json(options={})
    hash = as_json(
      except: [:update_at],
      methods: [:parse_content],
      include: {
        comments: {only: [:id, :content]},
        followers: {only: [:id]}
      }
    )
    hash.merge!(other_hash)
    hash
  end

  def other_hash
  	{title: "My title", owner: "My owner"}
  end

  def parse_content
  	"Article: "+ self.content
  end
end
```

自定义的主要代码是这部分:

```ruby
settings analysis:{
    analyzer: {
      my_custom_analyzer:{ type: 'custom', tokenizer: 'ngram_tokenizer'}
    },
    tokenizer: {
      ngram_tokenizer: { type: 'nGram', min_gram: 2, max_gram: 3, token_chars: ['lettler, 'digit', 'punctuation']}
    }
}
```

这里你可能需要倒回来看，我们自定义定义了一个 *tokenizer*， 取名为 *ngram_tokenizer* ，type 表示 使用的tokenizer。我们使用的是ES built-in 的 [nGram tokenizer](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/analysis-edgengram-tokenizer.html) ，具体配置参数请看它的文档。ES built-in 了不同的tokenizer，开发人员可以自由选择使用。

我们还定义了一个analyzer，取名为 *my_custom_analyzer*。 type custom 表示这个analyzer是自定义的。使用了 *ngram_tokenizer* 这个tokenizer。 这时你应该发现了，我们使用的就是这里我们自定义的 *ngram_tokenizer* 。这里我们没有对filter进行自定义，看过ES文档的朋友，应该知道filter也是可以自定义的。

官方的示例：

```ruby
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": { ... custom character filters ... },
            "tokenizer":   { ...    custom tokenizers     ... },
            "filter":      { ...   custom token filters   ... },
            "analyzer":    { ...    custom analyzers      ... }
        }
    }
}
```



ElasticSearch 确实是一个优秀的全文搜索引擎。了解和实践更多的ElasticSearch的设置和搜索，能够体会到ElasticSearch更多的功能。即使看过ElasticSearch入门教程的朋友，我觉得ElasticSearch的官方文档也是非常值得阅读的。
