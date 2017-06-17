---
layout: post
title: Callback order in controller of Rails
date:   2017-06-17 15:34:06
categories: Rails
image: /assets/images/post.jpg
---

在Rails controller中可以定义回调方法。回调方法是一个队列，满足先进先出的执行顺序。但我们知道，回调有before_action, after_action,
around_action 这三种简单的类型。并且还有append_xxx_action、 prepend_xxx_action、和skip_xxx_action。
在RubyChina上有一篇精华帖讲了controller callback 源码的内容 [详解 Rails Controller 中的 Callback](https://ruby-china.org/topics/32357)
这里我们研究除了skip_xxx_action的其他类型的执行顺序。

对于回调的执行顺序，个人觉得在一定场景下，是很有意义的。然而，往往我们写Ruby写的爽了，就忽略了。

举个简单的栗子(不一定符合最佳实践):

```ruby
class User < ApplicationController
  before_action :check_name
  before_action :check_require_params

  def create
  end

  private
  def check_name
    user = User.find_by(name: params[:name])
    if user.blank?
      render json: {code: 1404, message: "User is not found"} and return
    end
  end

  def check_require_params
    if params[:name].blank?
      render json: {code: 1500, message: "params name is blank"} and return
    end
  end
end
```

上面的栗子，如果我们传的params[:name] 为nil。根据队列顺序，从上往下执行回调，通过check_name 后返回 "User is not found"。但是，如果我们将顺序更改为：

```ruby
before_action :check_require_params
before_action :check_name
```

我们则可以通过check_require_params 回调检查出问题，而不需要进行一次数据库的查询。哪种顺序更佳，我想你已经知道了。

如果你遇到了在项目代码中controller有一些回调，调整这些回调，找出合适的执行顺序，我觉得是有意义的。

上面的栗子讲解的是相同回调类型下的执行顺序，不同回调类型的执行顺序会是怎样的呢？

```ruby
class User < ApplicationController
  before_action :log_before_action
  around_action :log_around_action
  after_action  :log_after_action

  def index
    return plain: 'Hello World'
  end

  private
  def log_before_action
    puts "="*10 + "before_action"
  end

  def log_around_action
    puts "-"*10 + "before around_action"
    yield
    puts "-"*10 + "after around_action"
  end

  def log_after_action
    puts "+"*10 + "after_action"
  end
end
```

访问`<index action>`
打印结果：
```
==========before_action
----------before around_action
  Rendering text template
  Rendered text template (0.0ms)
++++++++++after_action
----------after around_action
```

显然，before_action 是最先执行的，之后是around_action中的 进入`<yield>`之前的逻辑，然后是after_action，
最后才是around_action中的 执行完`<yield>`后的逻辑。可以看出，这里比较特别的是around_action。

我们调整回调的顺序为：

```ruby
around_action :log_around_action
before_action :log_before_action
after_action  :log_after_action
```

打印结果：
```ruby
----------before around_action
==========before_action
  Rendering text template
  Rendered text template (0.0ms)
++++++++++after_action
----------after around_action
```
最先执行的是 around_action中的 进入`<yield>`之前的逻辑，然后before_action, after_action, 最后是around_action中的 执行完`<yield>`后的逻辑。也就是说，最先触发的回调是around_action,因为这时候， around_action 在before_action的上面了。

另一种顺序：

```ruby
after_action  :log_after_action
around_action :log_around_action
before_action :log_before_action
```

打印结果：
```
----------before around_action
==========before_action
  Rendering text template
  Rendered text template (0.0ms)
----------after around_action
++++++++++after_action
```
最先执行的是 around_action中的 进入`<yield>`之前的逻辑，然后before_action, around_action中的 执行完`<yield>`后的逻辑,最后是after_action。

把after_action 放到最上面，和around_action 执行完`<yield>`后的逻辑相比较反而最后执行了。

简单的总结：
before_action 和 around_action中的 进入`<yield>`之前的逻辑部分是同一类别，并且满足队列的执行顺序。
after_action 和 around_action中的 执行完`<yield>`后的逻辑部分是统一类别，并且满足栈的执行顺序。
