---
layout: post
title: RVM Gemset Bundler Rbenv Rack Capistrano原理总结
date:   2016-07-18 15:47:06
categories: Go
image: /assets/images/post.jpg
---

### RVM Gemset Bundler Rbenv Rack Capistrano原理总结

最近有个面试机会，面试官问我是否 知道RVM Gemset Bundler Rbenv Capistrano 的原理知不知道。我当时想，原理，难道是
具体代码是怎么实现的吗？平时都在用这些工具，不过具体代码如何实现这个真的没有研究过。我回答了：“不知道”。回来后，便开始查
这些工具的原理是什么，然后，就是有种“原来如此”的感觉。所以，这篇文章就是对这些原理的一个自我总结。并没有从代码层面去分析，而
只是从他们的功能作用进行总结。

### RVM

Rvm是ruby开发环境的管理工具。能够帮助开发者在不同的ruby版本中切换。包括 不同类型 Cruby和Jruby直接的切换。
配合gemset，Rvm能够配置ruby和rails一起的不同版本的开发环境。而且，各个开发环境互相隔离不干扰。这样，就能很方便的为
不同的项目配不同的开发环境。

基本命令
```
rvm install 2.3.0
rvm use 2.3.0
rvm list
rvm use 2.3.0 --default
rvm remove 2.0.0
which ruby  # 顯示目前環境使用的 ruby 程式位置
```

工作原理
```
我现在用的是ruby 2.3.0版本
echo $PATH
/Users/path/.rvm/gems/ruby-2.3.0/bin:/Users/path/.rvm/gems/ruby-2.3.0@global/bin:/Users/path/.rvm/rubies/ruby-2.3.0/
bin:/Users/path/bin:/usr/local/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/Users/path/.rvm/bin
```

```
我切换到ruby 2.2.3之后
echo $PATH
/Users/path/.rvm/gems/ruby-2.2.3/bin:/Users/path/.rvm/gems/ruby-2.2.3@global/bin:/Users/path/.rvm/rubies/ruby-2.2.3/
bin:/Users/path/.rvm/bin:/Users/path/bin:/usr/local/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

$PATH中关于ruby版本的值变化了。RVM 会从环境变量PATH的最前面安排指定版本的相关执行路径。使得在命令列模式下执行ruby指令时，会因PATH 排列的权重，优先取得「.rvm」下的指定版本，而非系统原本的 /usr/bin/ruby。
简单的说，就是rvm在切换ruby版本的时候，会修改环境变量$PATH中关于ruby版本的值，修改为.rvm文件夹下的ruby版本的路径，并且获得这个路径下的ruby版本环境，而不是系统中的ruby版本。(比如老的mac系统一般自带1.9的ruby)

### RVM 与Gem
在 RVM里，不同版本的ruby的gem分别独立。

> 要注意的是我們在RVM安裝的任何Ruby版本都跟系統的Ruby無關，即使版號相同也是不同的Ruby，gem也不會安裝到系統的Ruby裡，所以可以放心的玩。基本上你只要發現你在安裝gem的時候會需要root權限的時候，通常你用的就是系統版本的Ruby了。


### Gemset
gemset是什么？其实和字面意思类似，就是 gem 的集合。

```
rvm gemset list
gemsets for ruby-2.3.0 (found in /Users/path/.rvm/gems/ruby-2.3.0)
=> (default)
   global
```
上面说明我的gemset是用的 default 并且用的global。
```
在我的 /Users/path/.rvm/gems  pwd
drwxr-xr-x   2 path  staff    68B 10 21  2015 cache
drwxr-xr-x  13 path  staff   442B 11 30  2015 ruby-2.2.3
lrwxr-xr-x   1 path  staff    54B 10 21  2015 ruby-2.2.3@global -> /Users/path/.rvm/rubies/ruby-2.2.3/lib/ruby/gems/2.2.0
drwxr-xr-x  12 path  staff   408B  7  5 14:45 ruby-2.3.0
lrwxr-xr-x   1 path  staff    54B  7  5 09:15 ruby-2.3.0@global -> /Users/path/.rvm/rubies/ruby-2.3.0/lib/ruby/gems/2.3.0
```

我进行下面的操作
```
rvm gemset create rails-5.0.0
/Users/path/.rvm/gems 就变成了
drwxr-xr-x   2 path  staff    68B 10 21  2015 cache
drwxr-xr-x  13 path  staff   442B 11 30  2015 ruby-2.2.3
lrwxr-xr-x   1 path  staff    54B 10 21  2015 ruby-2.2.3@global -> /Users/path/.rvm/rubies/ruby-2.2.3/lib/ruby/gems/2.2.0
drwxr-xr-x  12 path  staff   408B  7  5 14:45 ruby-2.3.0
lrwxr-xr-x   1 path  staff    54B  7  5 09:15 ruby-2.3.0@global -> /Users/path/.rvm/rubies/ruby-2.3.0/lib/ruby/gems/2.3.0
drwxr-xr-x   6 path  staff   204B  7 19 17:02 ruby-2.3.0@rails-5.0.0

rvm gemset list
gemsets for ruby-2.3.0 (found in /Users/path/.rvm/gems/ruby-2.3.0)
=> (default)
   global
   rails-5.0.0

这样就多了rails-5.0.0 的环境配置
rvm gemset use rails-5.0.0 就是使用这个gemset配置
```

这个环境就是ruby2.3.0版本，rails5.0.0版本。在/Users/path/.rvm/gems/ruby-2.3.0@rails-5.0.0/ 下有gems文件夹，其中就保存该环境配置下
使用到的gem包。你可以按照这种原理，自己创建多个gemset，也就可以为不同的项目，配置不同的ruby环境和rails环境，使用不同的gem包，项目之间就可以
互不干扰。

如果你要从別的版本的Ruby直接切换到指定的gemset：
```
rvm 2.3.0@rails-5.0.0
```

清空这个gemset
```
rvm gemset empty rails-5.0.0
```



























