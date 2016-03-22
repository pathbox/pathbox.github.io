---
layout: post
title: autoload_paths vs eager_load_paths
date: 2016-03-22 10:00:00
categories: rails
tag: rails
image: /assets/images/post.jpg
---

### autoload_paths vs eager_load_paths

<br>

In Ruby you have to require every .rb file in order to have its code run. However, notice how in Rails you never specifically require any of your models, controllers, or other files in the app/ dir. Why is that? That's because in Rails app/* is in autoload_paths. This means that when you run your rails app in development (for example via rails console) — none of the models and controllers are actually required by ruby yet. Rails uses special magical feature of ruby to actually wait until the code mentions a constant, say Book, and only then it would run require 'book'which it finds in one of the autoload_paths. This gives you faster console and server startup in development, because nothing gets required when you start it, only when code actually needs it.

Now, this behavior is good for local development, but what about production? Imagine that in production your server does the same type of magical constant loading (autoloading). It's not the end of the world really, you start your server in production, and people start browsing your pages slightly slower, because some of the files will need to be autoloaded. Yes, it's slower for those few initial requests, while the server "warms up", but it's not that bad. Except, that's not the end of the story.

If you are running on ruby 1.9.x (if I recall correctly), then auto-requiring files like that is not thread safe. So if you are using a server like puma, you will run into problems. Even if you aren't using a multi-threaded server, you are still probably better off having your whole application get required "proactively", on startup. This means that in production, you want every model, every controller, etc all fully required as you start your app, and you don't mind the longer startup time. This is called eager loading. All ruby files get eagerly loaded, get it? But how can you do that, if your rails app doesn't have a single require statement? That's where eager_load_paths come in. Whatever you put in them, all the files in all the directories underneath those paths will be required at startup in production. Hope this clears it up.

It's important to note that eager_load_paths are not active in development environment, so whatever you put in them will not be eagerly required immediately in development, only in production.

It's also important to note that just putting something into autoload_paths will not make it eager-loaded in production. Unfortunately. You have to explicitly put it into eager_load_paths as well.

Another interesting quirk is that in every rails app, all directories under app/ are automatically in both autoload_paths and eager_load_paths, meaning that adding a directory there requires no further actions.



#### If you add a dir directly under app/

Do nothing. All files in this dir are eager loaded in production and lazy loaded in development by default.

#### If you add a dir under app/something/

(e.g. app/models/concerns/, app/models/products/)

Ask: do I want to namespace modules and classes inside my new dir? For example in app/models/products/ you would need to wrap your class in module Products.

#### If the answer is yes, do nothing. It will just work.

If the answer is no, add config.autoload_paths += %W( #{config.root}/app/models/products ) to your application.rb.

In either case, everything will be eager loaded in production.

#### If you add code in your lib/ directory

##### Option 1

If you put something in the lib/ dir, what you are saying is: "I wrote this library, and I want to depend on it where I decide." This means that if you use your library in a rake task, but not in a rails app, you just require it in your rake task. If you need this library to always be loaded for your rails app, you require it in an initializer. If you need this library for some of your models or controllers, you require_dependency (see below why) it in those files, and since everything under your app/ dir is already auto- and eager- loaded as needed, your library will only be "pulled-in" if something that requires it from app/ or rake, or your custom script, actually gets loaded.

##### Option 2 (bad)

Another option is to add your whole lib dir into autoload_paths: config.autoload_paths += %W( #{config.root}/lib ). This means you shouldn't explicitly require your lib anywhere. As soon as you hit the namespace of your dir in other classes, rails will require it. The problem with this is that in Rails 3 if you just add something to your autoload paths it won't get eager loaded in production. You would need to add it to eager_load_paths instead, which causes a different problem (see below). And in ruby 1.9 autoload is not threadsafe. You probably want eager loading in production. Requiring your lib explicitly, like in option 1, is akin to eager loading it, which is threadsafe.

##### Option 3 (meh)

All the different things under your lib dir should be placed into their own directories, and those directories should be individually added to eager_load_paths.

```

config.eager_load_paths += %W(
  #{config.root}/lib/my_lib1
  #{config.root}/lib/my_lib2
)

```

This means that you can't just throw files into your lib dir. If you have my_lib1.rb, you must put it undermy_lib1/my_lib1.rb and my_lib1 should be added to eager load paths. This means that if you have more files inmy_lib1, you should create a dir my_lib1/my_lib1/extra.rb. This is a bit annoying.

So why not just add lib/ into eager_load_paths?

If you add lib/ into eager_load_paths, everything will work great. Your files will be autoloaded in development, and eager-loaded in production. Except the problem is that eager_load_paths use globbing like lib/**/*.rb, meaning that everything in your lib dir will try to get loaded. Your tasks, your generators, everything. This is not what you want.

Organizing lib

Regardless of which option you pick (option 1, hint hint), in your lib/ dir you should structure your code as if you structure a gem. If you need more than 1 file, you could for example add a same-named directory where everything is properly namespaced, and let your 1 file relatively require files in that directory.

Why use

```
require_dependency (auto-reloading)
```

If you use require_dependency, you are enabling auto-reload of your files in development across requests. require alone won't do it. I suggested to use it in your rails app, but not in initializers or rake tasks because rake tasks only run once, and changing initializers always requires restart.

However, it won't work without one additional piece of configuration. In application.rb you should add this:

```
config.watchable_dirs['lib'] = [:rb]
```

总结一下:
autoload_paths 就是当使用某个常量(比如module class 时),rails会调用const_missing,然后在autoload_paths 的数组目录中查找,找到该常量后加载使用。简单认为,在使用时,才加载相关代码。
优点: 能快速启动rails,在启动rails时减少加载代码,在使用时临时加载进来。
eager_load_paths 在rails启动时,就把eager_load_paths中的目录代码加载进来。production环境时建议使用eager_load_paths加载。这样虽然启动的时候速度会变慢,因为要加载更多代码,
但是,这样加载预热后,在用户使用程序时,就能很快的调用到相关代码,因为已经在启动的时候加载好了.
关于 lib目录中的代码文件根据实际情况需要,选择加载方式。上面也给出了三种加载的方式。


some links:
http://hakunin.com/rails3-load-paths
http://stackoverflow.com/questions/19773266/confusing-about-autoload-paths-vs-eager-load-paths-in-rails-4