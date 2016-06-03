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
4. 这创造了很多缓存垃圾。