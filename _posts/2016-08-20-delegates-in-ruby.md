---
layout: post
title: Delegation in Ruby and Rails
date:   2016-08-20 10:35:06
categories: Rails
image: /assets/images/post.jpg
---

这是一篇在Ruby和Rails中使用代理模式各种方法的总结文章。

##### delegate in Ruby

```ruby
require 'delegate'
class Boss
  def initialize(name)
    @name = name
  end

  def take_ticket
    p "(#{@name}) Help me take the ticket."
  end

  def plan
    p "(#{@name}) Help me make a plan."
  end
end

class Assistant < DelegateClass(Boss)
  def initialize(boss)
    super(boss)
  end

  def prepare_meeting
    p "I am going to prepare the meeting."
  end
end

boss = Boss.new("Anna")
anne = Assistant.new(boss)
anne.prepare_meeting
anne.take_ticket
anne.plan
```
anne 是 Assistant的实例, 是没有定义 take_ticket 和 plan 实例方法的。由于 Assistant 继承于
DelegateClass(Boss). 所以 anne = Assistant.new(boss) 这一步，让anne 能够调用 boos的实例
方法。这样anne 就能帮 boss 订票和制定计划了。


##### delegate in Rails

```ruby
class Blog < ActiveRecord::Base
  belongs_to :user
  delegate :name, :address, :age, to: :user, prefix: true, allow_nil: true
end

@blog.user.name <=> @blog.user_name
@blog.user.address <=> @blog.user_address
@blog.user.age <=> @blog.user_age
```

@blog = Blog.new
@blog 没有name实例方法，user 有name实例方法。如果 @blog想要调用user的name实例方法，一般情况是这样: @blog.user.name
现在，使用了: delegate :name, to: :user, prefix: true, allow_nil: true
@blog可以直接调用: @blog.user_name 就表示@blog.user.name 。这就是delegate 的作用。

再看下面的例子:
```ruby
class Greeter < ActiveRecord::Base
  def hello
    'hello'
  end

  def goodbye
    'goodbye'
  end
end

class Foo < ActiveRecord::Base
  belongs_to :greeter
  delegate :hello, to: :greeter
end

Foo.new.hello   # => "hello"
Foo.new.goodbye # => NoMethodError: undefined method `goodbye' for #<Foo:0x1af30c>

```
按照道理来说， Foo 没有定义hello 实例方法， 而Foo.new.hello 确没有报错。这就是delegate的作用了。

其实 Foo.new.hello 等价于 Foo.new.greeter.hello。 调用的正式Greeter的 hello 实例方法。
用了代理，省去了 greeter的写法。 和上面例子的区别是 prefix的配置，没有配置前缀。但是，一定配置了前缀，
就需要带上前缀。当然，前缀还可以不写true，写true rails会帮你将前缀定义为 to 的对象。可以自己定义前缀的名称，

```ruby
delegate :name, :address, to: :client, prefix: :customer
```

就变为 @blog.customer_name、@blog.customer_address

OK，我们继续看例子：
```ruby
class Foo
  CONSTANT_ARRAY = [0,1,2,3]
  @@class_array  = [4,5,6,7]

  def initialize
    @instance_array = [8,9,10,11]
  end
  delegate :sum, to: :CONSTANT_ARRAY
  delegate :min, to: :@@class_array
  delegate :max, to: :@instance_array
end

Foo.new.sum # => 6
Foo.new.min # => 4
Foo.new.max # => 11
```

这里Foo 配置了代理 sum，min，max 方法。并且 配置了to的对象。 我们知道，Foo并没有定义sum，min，max
这三个实例方法。 这三个实例方法其实就是Ruby数组库里面的方法，Ruby的数组对象，有sum，min，max三个实力方法。
所以，Foo代理就可以调用成功了。实际上，上面的代理操作等价于下面：

```ruby
Foo.new::CONSTANT_ARRAY.sum # => 6
Foo.new.class_array.min # => 4
Foo.new.instance_array.max # => 11
```

你可以简单的认为 Foo.new 代理了 CONSTANT_ARRAY 、@@class_array 、@instance_array 三个数组

```ruby
class Foo
  def self.hello
    "world"
  end

  delegate :hello, to: :class
end

Foo.new.hello # => "world"

```
还能直接代理类方法。
