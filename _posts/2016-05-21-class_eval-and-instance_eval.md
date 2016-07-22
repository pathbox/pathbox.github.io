---
layout: post
title: 理解class_eval and instance_eval
date:   2016-05-21 22:21:06
categories: rails
image: /assets/images/post.jpg
---



##### 例子1.

```ruby
class A
end

a = A.new
a.instance_eval do
  puts self  # => a
  #current class => a's siingleton class
  
  def method1
    puts "this is method1"
  end
end

a.method1
#=> this is method1

b = A.new
b.method1
#=> NoMethodError: undefined method `method1' for #<A:0x007fbc2ced9550>
  
```

##### 例子2.

```ruby
class A
end

A.instance_eval do
  puts self #=> A
  # current class => A's singleton class
  def method1
    puts "this is method1"
  end
end

A.method1
#=> this is method1

A.new.method1
#=> undefined method `method1' for #<A:0x0000000710f728>

```

##### 例子3.

```ruby
class A
end

a = A.new
a.method1
#=> NoMethodError: undefined method `method1' for <A:0x007fbc29a826f8> from (pry):21:in `<main>'

A.class_eval do
  # method1 is the instance method of A
  def method1
    puts "this is method1"
  end
end

a.method1
#=> this is a method1

```

##### 例子4.

```ruby
class A
end

A.class_eval do
  # method1 is the class method of A
  def self.method1
    puts "this is method1"
  end
end

a = A.new
a.method1 
#=> NoMethodError: undefined method `method1' for <A:0x002fbc69a446e8>

A.method1
#=> this is method1

```

##### 例子5.

```ruby
class A
end

a = A.new
a.class_eval do
  puts self
  # current class => a's singleton class
  def method1
    puts "this is method1"
  end
end

a.method1
#=> this is method1

A.method1
#=> undefined method `method1' for A:Class

b = A.new
b.method1
#=> undefined method `method1' for #<A:0x000000063eb678>
```

##### 例子6.

```ruby
class A
end

a = A.new
a.class_eval do
  puts self
  # current class => a's singleton class
  def self.method1
    puts "this is method1"
  end
  method1
end

#=> <Class:#<A:0x00000005597e78>>
#=> this is method1

a
#=>　#<A:0x00000005597e78> 
a.mthod1
#=>undefined method `method1' for #<A:0x00000005597e78>
#=>此时method1为 <Class:#<A:0x00000005597e78>>的方法而不是a<A:0x00000005597e78>的方法所以不能在外部被a调用
```


别人的总结: 
1. instance_eval必须由instance来调用，可以用来定义单例函数（singleton_methods)
2. class_eval必须是由class来调用，可以用来定义类的实例方法(instance_methods)

我的总结：
instance_eval 定义的method只能由instance_eval的定义者self来调用

class_eval 如果是类为定义者,方法 是def method1，则定义了实例方法。方法是 def self.method1 则定义了类方法。
如果实例对象为定义者,方法是def method 则定义的方法都是该实例对象的单例方法。 只能由该实例对象（例子中的a）调用。如果方法是 def self.method1 则method1不能在外部调用, 只能在内部调用。它是a单例类的方法，而不是a的单例方法（不知道是否可以这样理解。因为其中的 self对象是<Class:#<A:0x00000005597e78>>）

参考文章：https://ruby-china.org/topics/26208




