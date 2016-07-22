---
layout: post
title: RVM Gemset Bundler Rbenv Rack Capistrano原理总结
date:   2016-07-18 15:47:06
categories: Ruby
image: /assets/images/post.jpg
---



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
which ruby  # 显示目前环境使用的ruby位置
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

### Rbenv

你不能同时使用rbenv和RVM；你需要二选一。因此，如果你已经安装了RVM并且想尝试使用rbenv，在此之前，请确保使用你的脚本初始化文件完全卸载RVM以及它的依赖。
rbenv工作原理是拦截Ruby命令使用可执行的垫片（shim）来执行你的PATH。它会决定对指定的应用使用指定的Ruby版本，然后将你的命令传递到指定的Ruby的安装。

垫片（shim）在计算机中是一个小的拦截API调用的类库，改变传递的参数，处理或透明的重定向操作。
rbenv会在你的PATH变量开始出中插入shim的目录：

```
~/.rbenv/shims:/usr/local/bin:/usr/bin:/bin
```

rbenv简单的命令:

```
rbenv    install    2.2.2
rbenv    local    2.2.2

取消设置本地Ruby版本
rbenv    local    --unset
你可以决定是否设置一个默认的全局的Ruby版本
rbenv    global    2.2.2
rbenv    rehash
```

rbenv 创建 垫片 的所有命令 (ruby，irb，rake，gem，等等) 在您已安装的 Ruby 版本。这个过程被称为 重复。每次你安装新版本的 Ruby 或安装一个宝石，提供了一个命令，运行 rbenv rehash，以确保任何新的命令尖刀。这些垫片活在单个目录中 (~/.rbenv/shims，默认情况下)。若要使用 rbenv，你只需要添加到前面的你 PATH垫片目录:

```
export PATH="$HOME/.rbenv/shims:$PATH"
这就是上面的那个路径了
```

然后任何时间你从命令行运行 ruby或运行其一切读取 #!/usr/bin/env ruby脚本，您的操作系统会先找到 ~/.rbenv/shims/ruby并运行它而不是任何其他 ruby可执行文件您可能已经安装。
每个垫片是一个小小的 Bash 脚本，反过来运行 rbenv exec。所以在您的路径中的 rbenv，irb是相当于 rbenv exec irb，和 ruby -e "puts 42"等于 rbenv exec ruby -e "puts 42".
RBENV_VERSION环境变量设置，如果它的值确定的 Ruby 使用版本。

如果当前工作目录有一个 .rbenv-version文件，其内容用来将 RBENV_VERSION环境变量设置。
如果有是没有 .rbenv-version文件在当前目录中的rbenv 搜索每个父目录的 .rbenv-version文件直到它击中你的文件系统的根目录。如果找到一个，它的内容用来设置 RBENV_VERSION环境变量。
如果 RBENV_VERSION仍未设置，rbenv 试图设置它使用 ~/.rbenv/version文件的内容。
如果没有指定版本在任何地方，rbenv 假定您想要使用"系统"如果 rbenv 不在您的路径中，将运行任何版本 Ruby—i.e。
（你可以设置特定于项目的 Ruby 版本中使用 rbenv local命令，在当前目录中创建一个 .rbenv-version文件。同样，rbenv global命令修改 ~/.rbenv/version文件。）
武装部队与 RBENV_VERSION环境变量，rbenv 将 ~/.rbenv/versions/$RBENV_VERSION/bin添加到前面的你的PATH，然后命令和参数传递给rbenv exec的执行。

### Bundler

把Bundler 想象成一位严格的 gem 管理员。也就是说，Bundler 帮你安装你所需要的正确版本的 gems，并强制你的应用只使用你指定的版本。
Bundle 最关键的作用是安装并隔离你的 gems，但却不止于此。先了解一下 Gemfile 中的 gems 的代码是如何被加载到 Rails 应用里面的。

bin/rails文件：

```ruby

#!/usr/bin/env ruby
APP_PATH = File.expand_path('../../config/application',  __FILE__)
require_relative '../config/boot'
require 'rails/commands'
```

能看到它通过 require ../config/boot 来启动 Rails。

config/boot文件：

```ruby

# Set up gems listed in the Gemfile.
ENV['BUNDLE_GEMFILE'] ||= File.expand_path('../../Gemfile', __FILE__)

require 'bundler/setup' if File.exist?(ENV['BUNDLE_GEMFILE'])

```

第二行就是bundler的加载。你可以通过设定环境变量 BUNDLE_GEMFILE 来指定不同的 Gemfile。
bundler/setup 做了几件事：

- 它移除了 $LOAD_PATH 中所有 gems 的路径（相当于将 RubyGems 所做的工作都撤销了）
- 然后，它仅将 Gemfile.lock 中出现的 gems 加到 $LOAD_PATH 中

对，它就是会对Gemfile.lock中的gems进行重新加载到$LOAD_PATH中。使rails环境能使用到最新更新过的gem。
你能 require 的 gems 仅限于 Gemfile 中的那些 gems 了。
来看一下 config/application.rb，这个文件在 Rails 启动后就会被执行：

```ruby
# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)
```

Bundler.require 会加载传递给它的 group 中所有的 gems。（group 是指你在 Gemfile 中指定的 group）
以 development 模式启动 Rails 时，Rails.groups 的值为 [:default, :development]，而以 production 模式启动 Rails 时，Rails.groups 的值为 [:default, :production]，等等。

所以，Bundler 会去 Gemfile 中查找属于指定 group 的 gems，并且对每个找到的 gem 执行 require。如果你写了 nokogiri 这个 gem，它就会替你执行 require "nokogiri"。这就解释了为什么你无需写任何多余的代码，就能使你的 gems 在 Rails 中正常工作。





























