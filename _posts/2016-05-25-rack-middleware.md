---
layout: post
title: 理解rack中间件(转)
date:   2016-05-25 13:55:06
categories: rails
image: /assets/images/post.jpg
---

### 理解rack中间件
找到一篇讲解rack中间件的好文章，讲解的通俗易懂。于是便转载过来了。

#### 1. 介绍与原理

[rack与中间件]这篇文章有提到rack的介绍和原理，讲到webrick这种handler对请求的处理，然而本篇文章是要讲到中间件的部分。

中间件就是对请求的参数进行进一步的处理。

默认情况下，rack是会自动加载一些中间件的。

```ruby

# https://github.com/rack/rack/blob/028438ffffd95ce1f6197d38c04fa5ea6a034a85/lib/rack/server.rb#L228
def default_middleware_by_environment
  m = Hash.new {|h,k| h[k] = []}
  m["deployment"] = [
    [Rack::ContentLength],
    [Rack::Chunked],
    logging_middleware,
    [Rack::TempfileReaper]
  ]
  m["development"] = [
    [Rack::ContentLength],
    [Rack::Chunked],
    logging_middleware,
    [Rack::ShowExceptions],
    [Rack::Lint],
    [Rack::TempfileReaper]
  ]

  m
end
```

一般来说，线上的环境是deployment，开发环境是development。

所以的中间件的源码都存在于
> https://github.com/rack/rack/tree/028438ffffd95ce1f6197d38c04fa5ea6a034a85/lib/rack目录中。

里面定义的中间件很多，但是默认情况下rack加载的只有几个，就是上面所列出的。

我们找Rack::ContentLength这个中间件来研究一下：

```ruby

# https://github.com/rack/rack/blob/028438ffffd95ce1f6197d38c04fa5ea6a034a85/lib/rack/content_length.rb
require 'rack/utils'
require 'rack/body_proxy'

module Rack

  # Sets the Content-Length header on responses with fixed-length bodies.
  class ContentLength
    include Rack::Utils

    def initialize(app)
      @app = app
    end

    def call(env)
      status, headers, body = @app.call(env)
      headers = HeaderHash.new(headers)

      if !STATUS_WITH_NO_ENTITY_BODY.include?(status.to_i) &&
         !headers[CONTENT_LENGTH] &&
         !headers[TRANSFER_ENCODING] &&
         body.respond_to?(:to_ary)

        obody = body
        body, length = [], 0
        obody.each { |part| body << part; length += part.bytesize }

        body = BodyProxy.new(body) do
          obody.close if obody.respond_to?(:close)
        end

        headers[CONTENT_LENGTH] = length.to_s
      end

      [status, headers, body]
    end
  end
end
```

很简单，只定义了两个方法，一个是initialize，另一个是call方法。

众所周知的是，rails中默认就有很多中间件。

```ruby

$ rake middleware
use Rack::MiniProfiler
use Rack::Sendfile
use ActionDispatch::Static
use Rack::Lock
use Rack::Runtime
use Rack::MethodOverride
use ActionDispatch::RequestId
use Rails::Rack::Logger
use ActionDispatch::ShowExceptions
use WebConsole::Middleware
use ActionDispatch::DebugExceptions
use ActionDispatch::RemoteIp
use ActionDispatch::Reloader
use ActionDispatch::Callbacks
use ActiveRecord::Migration::CheckPending
use ActiveRecord::ConnectionAdapters::ConnectionManagement
use ActiveRecord::QueryCache
use ActionDispatch::Cookies
use ActionDispatch::Session::CookieStore
use ActionDispatch::Flash
use ActionDispatch::ParamsParser
use Rack::Head
use Rack::ConditionalGet
use Rack::ETag
use HttpAcceptLanguage::Middleware
use ExceptionNotification::Rack
run Rails365::Application.routes
那在rails源码中是如何添加这些中间件的呢？

# https://github.com/rails/rails/blob/1f85e1c9f34c7b0bdc1bddad5f914d61cb2a5435/railties/lib/rails/application/default_middleware_stack.rb#L54
...
middleware.use ::Rack::Runtime
middleware.use ::Rack::MethodOverride unless config.api_only
middleware.use ::ActionDispatch::RequestId

# Must come after Rack::MethodOverride to properly log overridden methods
middleware.use ::Rails::Rack::Logger, config.log_tags
middleware.use ::ActionDispatch::ShowExceptions, show_exceptions_app
middleware.use ::ActionDispatch::DebugExceptions, app, config.debug_exception_response_format
middleware.use ::ActionDispatch::RemoteIp, config.action_dispatch.ip_spoofing_check, config.action_dispatch.trusted_proxies
...

```

部分以ActionDispatch开头的middleware的源码存在于：
> https://github.com/rails/rails/tree/master/actionpack/lib/action_dispatch/middleware。

有这么多的中间件，但是有些用不到，或不想用，想要删除，或者说，要添加新的中间件?

可以有两种方法来做到。

第一种是修改config.rb文件，比如：

```ruby
# This file is used by Rack-based servers to start the application.

require ::File.expand_path('../config/environment', __FILE__)
use Rack::Static, :urls => ['/carrierwave'], :root => 'tmp' # adding this line
run Rails.application

```

第二种是修改config/appliction.rb，比如：

```ruby
# config/application.rb
config.middleware.delete "Rack::Lock"
config.middleware.insert_after "Rails::Rack::Logger", "MyMiddleware"
config.middleware.swap "ActionDispatch::ParamsParser", "MyParamsParser"

```
至于更详细的内容可以查看rails guides。

#### 2. 写自己的middleware

```ruby
创建app/middleware/delta_logger.rb文件，内容如下：

class DeltaLogger
  def initialize app, formatting_char = '='
    @app = app
    @formatting_char = formatting_char
  end

  def call env
    request_started_on = Time.now
    @status, @headers, @response = @app.call(env)
    request_ended_on = Time.now

    Rails.logger.debug @formatting_char * 50
    Rails.logger.debug "Request delta time: #{request_ended_on - request_started_on} seconds."
    Rails.logger.debug @formatting_char * 50

    [@status, @headers, @response]
  end
end
```

然后在config/application.rb中添加下面一行：

```ruby
config.middleware.use "DeltaLogger", "*"

```
重启rails应用，可以在日志中类似下面的输出：

**************************************************
Request delta time: 1.506944 seconds.
**************************************************

原文地址: http://www.rails365.net/articles/rack-yu-zhong-jian-jian
