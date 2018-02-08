---
layout: post
title: 用更好的Ruby code style 避免类型错误
date:   2017-04-09 12:39:06
categories: Ruby
image: /assets/images/post.jpg
---

我们知道，Ruby是动态语言。
静态类型：编译的时候就知道每一个变量的类型，因为类型错误而不能做的事情是语法错误。

动态类型：编译的时候不知道每一个变量的类型，因为类型错误而不能做的事情是运行时错误。
譬如说你不能对一个数字a写a[10]当数组用。

对于Ruby来说，当程序运行起来后，才有可能发现类型错误。
> NoMethodError: undefined method 'xxx' for nil:NilClass 这是Ruby中最常见发生的Error

我们是否可以在写代码的时候，就选择更好的方式，来最大程度的避免的错误发生。

##### example1

```ruby
name = first + last
```

如果 first 或 last 是 nil，则会报

> NoMethodError: undefined method ''+' for nil:NilClass

```ruby
name = first.to_s + first.to_s

# better

name = "#{first}#{last}"
```

##### example2

```ruby
def name(first, last, params = nil)
  name = "#{first}#{last}-#{params[:nick_name]}"
end
```

当这样使用该方法时，则会报错

```ruby
# wrong
name("Steven", "Curry") # => NoMethodError: undefined method `[]' for nil:NilClass
```
为了避免这种方法，你需要在name方法中加判断存在的逻辑，是否可以有更好的方式呢？
观察可以知道，params变量其实会是一个hash，所以，我们可以这样：

```ruby
def name(first, last, params = {})
  name = "#{first}#{last}-#{params[:nick_name]}"
end
```

```ruby
name("Steven", "Curry") # => "StevenCurry-"
```

同理，当我们知道params是数组类型的时候，应该params = []
用具体的一种类型的空表示，代替nil。可以避免

> => NoMethodError: undefined method 'xxx' for nil:NilClass

##### example3

别忘了try方法
在Rails中，任何对象都可以使用try方法

```ruby
  1.try(:a)
  "name".try(:a)
  String.try(:a)
```
try 方法中的参数是一个公有方法名，如果没有这个公有方法，则会返回nil值。注意，是公有方法。

try source code
```ruby
def try(*a, &b)
  if a.empty? && block_given?
    yield self
  else
    public_send(*a, &b) if respond_to?(a.first)
  end
end
```

可以发现，try方法可以加block，则会将self代入运行block。
或者调用public_send
```ruby
  "first".try do |name|
    "#{name}_last"
  end

  # => "first_last"
```

未完待续
