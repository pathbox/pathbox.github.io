---
layout: post
title: include extend prepend ActiveSupport::Concern(Mixin)
date:   2016-07-25 12:13:06
categories: Ruby
image: /assets/images/post.jpg
---



### 该总结的还是要总结的

网上关于include extend prepend ActiveSupport::Concern文章不少，而且质量也很高。
但是，也许不是自己亲自再总结整理，就感觉不是自己的"东西"。(现实是：看了别人的文章n遍，项目中可以正常使用，但是要回想或深究原理时，就卡壳了)所以，还是给大脑来次激烈的碰撞。

### include and extend

```ruby
  module A
    def my_method
      p "here is A"
    end
  end

  class A_include
    include A
  end

  class A_extend
    extend A
  end

  A_include.new.my_method
  #=>  "here is A"

  A_extend.my_method
  #=>  "here is A"

```

A_include 增加了一个“实例方法” my_method，A_extend 增加了一个“类方法” my_method。同时，A_include的祖先链(ancestors)
中将增加A module，而A_extend的祖先链中没有A module。

ruby方法调用中的lookup:

1.实例方法，首先在实例中找，然后在实例对象的单件类中找，之后沿祖先链依次向上找

2.类方法，首先在类的单件类中找，然后沿着类的单件类的祖先链依次向上找（类的单件类的祖先链 = 类的祖先链的各个节点的单件类组成的链）

使用用include扩展类方法：(实际项目中不这么做，因为，完全没有必要)

```ruby
  module A
    def self.included(base)
      base.extend(ClassMethods)
    end

    module ClassMethods
      def my_method
        p "here is A module"
      end
    end
  end

  class A_include_class_method
    include A
  end

  puts A_include_class_method.my_method  #=>  "here is A module"

```



### prepend and include

prepend 与 include 类似，首先都是添加实例方法的，不同的是扩展module在祖先链上的放置位置不同。

```ruby
  module A
    def my_method
      p "here is A module"
    end
  end

  class B_include
    include A
    def my_method
      p "here is B_include"
    end
  end

  class B_prepend
    prepend A
    def my_method
      p "here is B_prepend"
    end
  end

  B_include.new.my_method  #=>  "here is B_include"
  B_prepend.new.my_method  #=>  "here is A module"

  puts B_include.ancestors
  #B_include
  #A
  #Object
  #Kernel
  #BasicObject

  puts B_prepend.ancestors
  #A
  #B_include
  #Object
  #Kernel
  #BasicObject

```

### 多个include加载

```ruby
  module One
    def my_method
      "Here is One"
    end
  end
  module Two
    def my_method
      "Here is Two"
    end
  end
  module Three
    def my_method
      "Here is Three"
    end
  end
  module Four
    def my_method
      "Here is Four"
    end
  end

  class Num
    include One
    include Two
    include Three
  end
  Num.new.my_method
  # "Here is Three"
  Num.ancestors
  # Num
  # Three
  # Two
  # One
  # Object
  # Kernel
  # BasicObject

  # include 是栈结构的加载，”后进先出”。所以，最后被include的 Three会先被查找。想起了before_action
  # 是 队列 结构的加载，最先的 before_action 条件会先被执行

  class Num
    prepend Four
    include One
    include Two
    include Three
  end

  Num.new.my_method
  # "Here is Three"
  Num.ancestors
  # Four
  # Num
  # Three
  # Two
  # One
  # Object
  # Kernel
  # BasicObject

  # 由于 Four 是 prepend的加载，所以，无论prepend Four的顺序如何，总是第一个被查找

```


### Ruby没有interface，but 我们有 Mixin

```ruby

   module ProductInfo
     def self.included(base)
       base.extend ClassMethods
       base.class_eval do
         scope :disabled, -> { where(disabled: true) }
       end

      #  include InstanceMethods
   end

     module ClassMethods
       include ProductInfo::InstanceMethods
       def provider
         "I am a provider"
       end
     end

     module InstanceMethods
       def price
         "100.00"
       end
     end

     def desc
       "Here are desc for product"
     end
   end

   class Shop
     include ProductInfo
   end

   shop = Shop.new

   puts Shop.provider  #=>  "I am a provider"
   puts shop.price     #=>  "100.00"
   puts shop.desc      #=>  "Here are desc for product"

 ```

 这里有两种方法定义实例方法。一种直接定义实例方法，入方法 desc。 另一种，在module InstanceMethods
 中定义实例方法，然后在def self.included(base) 中加载 “base.include InstanceMethods”,或者是
 在ClassMethods中 include ProductInfo::InstanceMethods 加载。这样 price方法就会被Shop加载为实例方法。

include有一个叫included的钩子，正是通过这个钩子，我们可以用include实现添加类方法和实例方法。我们来看看included这个钩子到底做了什么：

```ruby
  module A
    def self.included(mod)
      puts "#{self} included in #{mod}"
    end
  end

  class A_include
    include A
  end
  # A included in A_include
  # => A_include

```

included类方法作用域self为Module A，并传入include A的receiver Class A_include。

### Mixin的尖兵利器 ActiveSupport::Concern

改造上面的例子

```ruby
  require 'active_support/concern'
  module ProductInfo
    extend ActiveSupport::Concern

    included do
      scope :disabled, -> { where(disabled: true) }
      has_many :categories

      include InstanceMethods
    end

    module ClassMethods
      # include ProductInfo::InstanceMethods
      def provider
        "I am a provider"
      end
    end

    module InstanceMethods
      def price
        "100.00"
      end
    end

    def desc
      "Here are desc for product"
    end
  end

  class Shop
    include ProductInfo
  end

```

ActiveSupport::Concern 代码解析。

```ruby
module ActiveSupport
  module Concern
    #自定义了一种错误类型，在一个extend ActiveSupport::Concern的类中如果多次使用included方法，会报这个异常
    class MultipleIncludedBlocks < StandardError #:nodoc:
      def initialize
        super "Cannot define multiple 'included' blocks for a Concern"
      end
    end
    # 扩展这个module的时候注意使用extend，而不是include
    # 如果当前类extend了ActiveSupport::Concern,则@_dependencies会被定义。
    def self.extended(base) #:nodoc:  当发现 extend ActiveSupport::Concern
      base.instance_variable_set(:@_dependencies, [])  #定义了@_dependencies 数组
    end

    #include一个module的时候，会先调用append_features，在调用included，base为外层发起调用的module或class
    def append_features(base)
      if base.instance_variable_defined?(:@_dependencies)
        # 这里判断当前类(base)是否定义了@_dependencies，如果被定义，则把当前module加入@_dependencies。怎么说呢？
        base.instance_variable_get(:@_dependencies) << self
        return false
      else
        return false if base < self
        # 模块通过依赖加载
        @_dependencies.each { |dep| base.send(:include, dep) }
        #base 就是include ActiveSupport::Concern module的class或module。这里相当于 base 从@_dependencies数组中，逐个include 对象。比如: include 其中定义的各种module
        super
        #include module时，为base同时扩展类方法。找到 ClassMethods， 然后 进行extend，就扩展了其中的类方法
        base.extend const_get(:ClassMethods) if const_defined?(:ClassMethods)
        #include module时，可以执行included中的方法体。 这里就是included 的钩子了，看出是 base.class_eval。将@_included_block中的内容都以类对象的方式执行。@_included_block就是included block
        base.class_eval(&@_included_block) if instance_variable_defined?(:@_included_block)
      end
    end
    #被引用时执行的方法，这里的参数设计神了...
    def included(base = nil, &block)
      if base.nil?
        raise MultipleIncludedBlocks if instance_variable_defined?(:@_included_block)

        @_included_block = block
      else
        super
      end
    end

    def class_methods(&class_methods_module_definition)
      mod = const_defined?(:ClassMethods, false) ?
        const_get(:ClassMethods) :
        const_set(:ClassMethods, Module.new)

      mod.module_eval(&class_methods_module_definition)
    end
  end
end

```

append_features也是module的一个callback，会在include之后，为当前class添加module的变量，常量，方法等。append_features 会先与included 被调用，详见：[append_features](http://www.thecodingforums.com/threads/append_features-vs-include.834205/)

Reference: https://ruby-china.org/topics/21501
           https://ruby-china.org/topics/26208
           https://github.com/rails/rails/blob/master/activesupport/lib/active_support/concern.rb
