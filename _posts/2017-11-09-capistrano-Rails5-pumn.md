---
layout: post
title: Capistrano 部署Rails5 puma 项目小结
date:   2017-11-09 15:44:06
categories: Work
image: /assets/images/rails.jpg
---

项目目录名为: cap_puma

本文不讨论Capistrano原理机制，只是部署过程记录

Rails version 5.1.4

##### 在Gemfile加入相关gem

```ruby

group :development do
  gem 'capistrano',            '3.7'
  gem 'capistrano-rails',      '1.3'
  gem 'capistrano-bundler',    '1.2'
  gem 'capistrano-rbenv',      '2.1'
  gem 'capistrano3-puma',      '1.2.1'
  gem 'capistrano-sidekiq',    '0.10.0'
end
```

bundler

##### cap install 命令
执行cap install

然后会生成以下文件


```ruby
cap_puma/Capfile
cap_puma/config/deploy/production.rb
cap_puma/config/deploy/staging.rb
cap_puma/config/deploy.rb
```

##### Capfile

`cap_puma/Capfile`

```ruby
# Load DSL and set up stages
require "capistrano/setup"
require "capistrano/deploy"
require 'capistrano/console'
require "capistrano/rbenv"
require 'capistrano/bundler'
require 'capistrano/puma'
require 'capistrano/puma/nginx'
require 'capistrano/sidekiq'
require "capistrano/scm/git"
require 'capistrano/rails/migrations'

install_plugin Capistrano::SCM::Git

# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }

```

##### deploy.rb

`cap_puma/config/deploy.rb`

```ruby
# config valid only for current version of Capistrano
lock "3.7.0"

set :application, "cap_puma"
set :repo_url, "git@xxx/cap_puma.git"

set :rbenv_ruby, '2.3.3'
set :user, 'webuser'

set :ssh_options, {
  user: fetch(:user),
  forward_agent: true
}

# 服务器 Puma的自定义配置 参考这个文档
# https://github.com/seuros/capistrano-puma\
set :deploy_to, "/web/www/#{fetch :application}"

ask :branch, `git rev-parse --abbrev-ref HEAD`.chomp
append :linked_files, "config/database.yml", "config/property.yml", "config/secrets.yml"

append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system"

def production_warning
  return if ARGV == ["-T"]

  puts "注意!!! 你正在尝试操作'正式站的服务器'"
  print "该操作非常的危险! 如果继续请输入yes: "
  value = $stdin.gets.chomp
  unless value == 'yes'
    abort "你输入的不正确, 该操作已被终止"
  end
end

```

##### production.rb

`cap_puma/config/deploy.rb`

```rubyproduction_warning
set :rails_env, 'production'
set :nginx_server_name, 'www.cappuma.cn'
set :puma_workers, 2

list = []

role :app, list
role :web, list
role :db, %w{127.1.1.1} # IP

```

##### staging.rb

`cap_puma/config/deploy/staging.rb`

```ruby

role :app, %w{127.1.1.1}
role :web, %w{127.1.1.1}
role :db,  %w{127.1.1.1}

set :rails_env, 'staging' # environment

set :sidekiq_config, "#{current_path}/config/sidekiq.yml" # chose sidekiq.yml
set :nginx_server_name, 'www.cappuma.com'
set :puma_workers, 2

```

you can make a new /deploy/file.rb

##### test_puma

`cap_puma/config/deploy/test_puma.rb`

```ruby
role :app, %w{127.1.1.1}
role :web, %w{127.1.1.1}
role :db,  %w{127.1.1.1}

set :rails_env, 'staging' # environment

set :sidekiq_config, "#{current_path}/config/sidekiq.yml" # chose sidekiq.yml
set :nginx_server_name, 'www.cappuma.com'
set :puma_workers, 2
```

##### Go to the Server and config your nginx

`/etc/nginx/sites-available/cap_puma`

```

upstream cap_puma_puma_server {
  server unix:/web/www/cap_puma/shared/tmp/sockets/puma.sock fail_timeout=0;
}

server {

  listen 80;

  client_max_body_size 4G;
  keepalive_timeout 10;

  error_page 500 502 504 /500.html;
  error_page 503 @503;

  server_name vcall.udeskcat.com;
  root /web/www/cap_puma/current/public;
  try_files $uri/index.html $uri @cap_puma_puma_server;

  location @cap_puma_puma_server {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;

    proxy_pass http://cap_puma_puma_server;
    # limit_req zone=one;
    access_log /web/www/cap_puma/shared/log/nginx.access.log;
    error_log /web/www/cap_puma/shared/log/nginx.error.log;
  }

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  location = /50x.html {
    root html;
  }

  location = /404.html {
    root html;
  }

  location @503 {                                                                                          
    error_page 405 = /system/maintenance.html;
    if (-f $document_root/system/maintenance.html) {
      rewrite ^(.*)$ /system/maintenance.html break;
    }
    rewrite ^(.*)$ /503.html break;
  }

  if ($request_method !~ ^(GET|HEAD|PUT|PATCH|POST|DELETE|OPTIONS)$ ){
    return 405;
  }

  if (-f $document_root/system/maintenance.html) {
    return 503;
  }

  location ~ \.(php|html)$ {
    return 405;
  }
}

```

cd /etc/nginx/sites-enabled

ln -s /etc/nginx/sites-available/cap_puma cap_puma.conf 

##### puma.rb

`/web/www/udesk_vcall/shared/puma.rb`

config the puma.rb for environment  

```ruby
#!/usr/bin/env puma

directory '/web/www/cap_puma/current'
rackup "/web/www/cap_puma/current/config.ru"
environment 'staging'  # environment 'production'

pidfile "/web/www/cap_puma/shared/tmp/pids/puma.pid"
state_path "/web/www/cap_puma/shared/tmp/pids/puma.state"
stdout_redirect '/web/www/cap_puma/shared/log/puma_access.log', '/web/www/cap_puma/shared/log/puma_error.log', true

threads 0,16

bind 'unix:///web/www/cap_puma/shared/tmp/sockets/puma.sock'

workers 2

prune_bundler

on_restart do
  puts 'Refreshing Gemfile'
  ENV["BUNDLE_GEMFILE"] = "/web/www/cap_puma/current/Gemfile"
end

```

##### cap test_puma deploy
Then error happen.

create `database.yml` `property.yml` `secrets.yml` in `/web/www/udesk_vcall/shared/config`
to fix the error


cap test_puma deploy

It works~
