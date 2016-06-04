---
layout: post
title: How key-based cache expiration works(翻译)
date:   2016-06-03 13:40:06
categories: rails
image: /assets/images/post.jpg
---

### How key-based cache expiration works

原文链接： [How key-based cache expiration works](https://signalvnoise.com/posts/3113-how-key-based-cache-expiration-works)

大神DHH的文章。时间有点久，不过写的通俗易懂。并不是详细的写解决方案，而是给予一定的方向思考。翻译是为了能让自己加深印象。(翻译能力有限)

手动的使缓存失效是一件非常令人沮丧和容易出错的处理过程。你很容易忘记这点和让陈旧的数据使用。这足够关闭大多数人的"俄罗斯套娃"缓存结构，就像我们下一步要在Basecamp中使用的。
谢天谢地有一个更好的方法。一个好得多的方法。它就是 密钥缓存过期。 它的工作机制是这样的：

1. 缓存key是流体部分和缓存的内容是固定的部分。一个给定的key总是返回相同的内容，当缓存创建后，你永远不更新内容，你就永远不会使它过期。
2. key是计算同步对象锁代表的内容。这通常通过时间戳的key的部分。比如  [class]/[id]-[timestamp]， 就像是todos/5-20110218104500或projects/15-20110218104500, 当你调用 cache_key 方法时，Rails中的ActivRecord会默认帮你做这些事情。
3. 当key改变了，你可以简单地用新的key写入新的内容。这样，如果你更新了todo，key从todos/5-20110218104500变为todos/5-20110218105545。因此，新的内容的写入时基于更新的对象。
4. 这创造了很多缓存垃圾。一旦我们更新这todo，旧的缓存将永远不会再被读取。系统中美好的地方是，你不用关心它。Memcached 将会自动的清除最旧的keys当Memcached要耗尽内存空间的时候。Memcached能这样处理的原因是：当最后一次读取缓存，它会进行追踪。
5. 在更新的时候，你可以处理依赖的关系结构通过将模型对象绑定在一起。这样如果你改变一个属于project下的todolist的todo，你可以更新updated_at timestamp 在每个关系链部分，这样就自动地更新了它们的缓存keys。在Rails中，你可以这样描述：

```ruby
class Project < ActiveRecord::Base
end

class Todolist < ActiveRecord::Base
  belongs_to :project, touch: true
end

class Todo < ActiveRecord::Base
  belongs_to :todolist, touch: true
end

```

6.关于基于partials的缓存问题。这可以巧妙地嵌套像下面每个调用缓存调用 #cache_key 传入的数组元素。在第一种情况下，缓存key最终被像 v5/projects/5-20110219102600.
关键是更新描述的对象被更新在上述过程中,和适当的缓存总是获取。


```ruby
<%# projects/show.html.erb %>
<% cache ["v5", project] do %>
  <p> All my todo lists:</p>
  <%= render project.todolists %>
<% end %>

<%# todolists/_todolist.html.erb %>
<% cache["v3", todolist] do %>
  <p><%= render todolist.todos %></p>
<% end %>

<%# todos/_todo.html.erb%>
<% cache ["v1", todo] do %>
  <p><%= todo.name %></p>
<% end %>

```

这个过程使它容易实现缓存方案,相信你永远不会使用到旧的缓存数据。没有混乱的清理处理因为你没有义务来追踪每一个点可能会更新一个对象。updated_at字段的所有缓存key自动为您的处理,无论哪里发生了更新操作。


 