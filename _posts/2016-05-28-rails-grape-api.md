---
layout: post
title: rails + grape 快速API简单搭建
date:   2016-05-28 11:26:06
categories: rails
image: /assets/images/post.jpg
---



> grape: An opinionated framework for creating REST-like APIs in Ruby. http://www.ruby-grape.org

这是一篇快速用grape挂载在rails中进行API开发的文章,不知不觉也用grape开发接口半年多了.

首先在Gemfile中加入

```
#API
gem 'grape'
gem 'grape-entity'
gem 'grape-swagger'
gem 'grape-swagger-rails'
gem 'grape-swagger-ui'
gem 'rack-contrib', '~> 1.1.0' # for JSONP
gem 'rack-ssl-enforcer'
gem 'kramdown'
```
执行

##### bundle

在 app 目录下新建api文件夹，在下面新建三个文件

app_default.rb


```ruby
module AppDefault
  extend ActiveSupport::Concern
  included do
	format :json
	prefix :api
	default_format :json
	default_error_status 401
  end
end
```

root_app.rb

```ruby
require 'grape'

class RootApp < Grape::API

	include AppDefault
	desc 'API Root'
	mount PostApp
	add_swagger_documentation(

    hide_documentation_path: true,
		format: :json,
		hide_format: true,
		markdown: GrapeSwagger::Markdown::KramdownAdapter.new,
		api_version: '1.0',
		hide_documentation_path: true,

		info: {
			title: 'APP 文档查阅',
			description: "使用基本身份验证"
		}
	)
	route :any, 'path' do
		error!({error: '错误的接口地址',field: 404, with: Entities::Error}, 404)
	end
end
```

post_app.rb

```ruby
class PostApp < Grape::API
  include AppDefault

  version 'v1', using: :path
  namespace :posts, desc: '文章API' do
    #只捕捉本路由中的404
    rescue_from ActiveRecord::RecordNotFound do |e|
      error_response(message: {error: '文章不存在'}, status: 404)
    end

    desc '1000, 获得文章列表展示' do
      failure [[401, '未授权'],[422, '其他错误']]
    end
    params do
      # requires :token, type: String, desc: '用户token'
      requires :per, type: Integer, desc: '文章数量'
    end
    get '/' do
      posts = Post.limit(params[:per])
      present :posts, posts
    end
  end
end
```
在config/routes.r文件挂载上路由

```ruby
 mount GrapeSwaggerRails::Engine => '/docs'
 mount RootApp => '/'
```
根据 app_default.rb 中定义的 prefix :api，和post_app.rb中version 'v1', using: :path。可以知道接口的url为: localhost:3000/api/v1/posts

rails s -b 0.0.0.0
访问localhost:3000/api/v1/posts?per=10 就能得到10条记录并且是以JSON的格式返回。grape会将数据序列化为JSON后返回。
最简单的grape api就搭建好了。

在 config/initializers/ 创建 grape_swagger_rails.rb文件

```ruby
GrapeSwaggerRails.options.url      = "/swagger"
GrapeSwaggerRails.options.app_name = 'TangApp'
GrapeSwaggerRails.options.app_url  = ''

GrapeSwaggerRails.options.api_auth     = 'basic'
GrapeSwaggerRails.options.api_key_name = 'Authorization'
GrapeSwaggerRails.options.api_key_type = 'header'

GrapeSwaggerRails.options.before_filter do |request|
  authenticate_or_request_with_http_basic 'APP API Docs' do |user, password|
    user == 'admin' && password == 'admin'
  end
end
```
配置swagger自动API文档系统。
访问 127.0.0.1:3000/docs 就可显示swagger API文档的系统页面，非常方便API的调试。这个API文档系统生产环境不需要，所以一般在development环境使用。

我们使用 grape-entity
在app/api/下新建文件 entities_app.rb

```ruby

module EntitiesApp
  class BaseEntity < Grape::Entity
	format_with(:null) {|v| (v != false && v.blank?) ? "" : v }
	format_with(:null_int) {|v| v.blank? ? 0 : v}
	format_with(:null_bool) { |v| v.blank? ? false : v }
  format_with(:short_dt) { |v| v.blank? ? "" : v.strftime("%Y-%m-%d %H:%M:%S") }
	format_with(:short_t) { |v| v.blank? ? "" : v.strftime("%Y-%m-%d %H:%M") }
  end

  class BaseError < Grape::Entity
	expose :error, documentation: {required: true, type: 'String', desc: '错误信息'}
  end

  class Error < BaseError
    expose :field, documentation: {required: true, type: "String", desc: "错误字段"}
  end

  class Post < BaseEntity
    expose :id, documentation: {required: true, type: "Integer", desc: "文章ID"}, format_with: :null_int
    expose :title, documentation: {type: "String", desc: "文章title"}, format_with: :null
    expose :body, documentation: {type: "String", desc: "文章内容"}, format_with: :null
  end
end
```
修改 post_app.rb文件

```ruby

present :posts, posts
```
改为

```ruby
present :posts, posts, with: EntitiesApp::Post
```
这样返回的JSON数据的field的内容就会是对应entity中配置的字段。这样就能方便的配置需要返回什么字段，就返回什么字段。grape-entity还有更高级的用法，具体可以在GitHub上查阅资料文档。

这里没有讲解token参数的设计，对于API设计来说，token的产生和验证是非常重要的。

grape挂载在rails中使用的优点： 可以直接使用model和model中的方法，ActiveRecord::Base直接加载了。方便使用ActiveRecord::Base。如果app中有H5页面，就可以写在rails中，使用rails的路由等。

本文完。
