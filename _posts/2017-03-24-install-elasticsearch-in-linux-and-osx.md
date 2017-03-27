---
layout: post
title: Install Elasticsearch in Ubuntu and macOS
date:   2017-03-24 21:16
categories: Elasticsearch
image: /assets/images/post.jpg
---

##### Ubuntu 14.04环境

```ruby
版本：ElasticSearch 1.4.2

wget https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-1.4.2.deb
sudo dpkg -i elasticsearch-1.4.2.deb

# ➜ sudo dpkg -i elasticsearch-1.4.2.deb
# Selecting previously unselected package elasticsearch.
# (Reading database ... 44363 files and directories currently installed.)
# Unpacking elasticsearch (from elasticsearch-1.4.2.deb) ...
# Setting up elasticsearch (1.4.2) ...
# Adding system user `elasticsearch' (UID 109) ...
# Adding new user `elasticsearch' (UID 109) with group `elasticsearch' ...
# Not creating home directory `/usr/share/elasticsearch'.
# ### NOT starting elasticsearch by default on bootup, please execute
#  sudo update-rc.d elasticsearch defaults 95 10
# ### In order to start elasticsearch, execute
#  sudo /etc/init.d/elasticsearch start
# Processing triggers for ureadahead ...
# ureadahead will be reprofiled on next reboot

## 设置开机启动 elasticsearch
  sudo update-rc.d elasticsearch defaults

## 获取中文分词插件库(会很慢很慢)
git clone git@github.com:medcl/elasticsearch-rtf.git && cd elasticsearch-rtf
git checkout -b 1.4.0 origin/1.4.0

## 拷贝IK的配置
sudo cp -r config/ik /etc/elasticsearch/
sudo chmod -R a+X /etc/elasticsearch/ik
sudo chmod -R a+r /etc/elasticsearch/ik

## 拷贝IK的插件
sudo mkdir /usr/share/elasticsearch/plugins/ -p
sudo cp -r plugins/analysis-ik /usr/share/elasticsearch/plugins/ik
sudo chmod a+x /usr/share/elasticsearch/plugins/ik
sudo chmod a+r /usr/share/elasticsearch/plugins/ik

## 修改配置文件
#############################
sudo vim /etc/elasticsearch/elasticsearch.yml
## 编辑 /etc/elasticsearch/elasticsearch.yml 文件
##   1. 在文件末尾添加
##         index.analysis.analyzer.ik.type : "ik"
##
##   2. 测试环境下，修改 cluster.name 为局域网内唯一的名字。否则局域网内相同名字的 ES 会自动组成集群

## 测试
#############################
curl -XGET "localhost:9200"
curl -XPUT "localhost:9200/test1"
curl -XPOST "localhost:9200/test1/_analyze?analyzer=ik&pretty=true" -d "中华人民共和国"
### 应该能看到分词后的效果

## 安装完成后的目录结构
#############################
# ➜ tree /etc/elasticsearch
# /etc/elasticsearch
# ├── elasticsearch.yml
# ├── ik
# │   ├── custom
# │   │   ├── ext_stopword.dic
# │   │   ├── mydict.dic
# │   │   ├── single_word_full.dic
# │   │   ├── single_word_low_freq.dic
# │   │   └── sougou.dic
# │   ├── IKAnalyzer.cfg.xml
# │   ├── main.dic
# │   ├── preposition.dic
# │   ├── quantifier.dic
# │   ├── stopword.dic
# │   ├── suffix.dic
# │   └── surname.dic
# └── logging.yml
#
# 2 directories, 14 files

# ➜ tree /usr/share/elasticsearch/plugins
# /usr/share/elasticsearch/plugins
# └── ik
#     ├── commons-codec-1.6.jar
#     ├── commons-logging-1.1.3.jar
#     ├── elasticsearch-analysis-ik-1.2.9.jar
#     ├── httpclient-4.3.5.jar
#     └── httpcore-4.3.2.jar
#
# 1 directory, 5 files
```

##### macOS 环境
可以使用
```
brew install elasticsearch

brew info elasticsearch

brew info elasticsearch
elasticsearch: stable 5.2.2, HEAD
Distributed search & analytics engine
https://www.elastic.co/products/elasticsearch
Not installed
From: https://github.com/Homebrew/homebrew-core/blob/master/Formula/elasticsearch.rb
==> Requirements
Required: java >= 1.8 ✘
==> Caveats
Data:    /usr/local/var/elasticsearch/elasticsearch_pathbox/
Logs:    /usr/local/var/log/elasticsearch/elasticsearch_pathbox.log
Plugins: /usr/local/Cellar/elasticsearch/5.2.2/libexec/plugins/
Config:  /usr/local/etc/elasticsearch/
plugin script: /usr/local/Cellar/elasticsearch/5.2.2/libexec/bin/elasticsearch-plugin
```

根据info的内容，你可以可看见一些版本信息和文件路径信息

也可以直接在官网下载压缩包，放到目录解压即可。不过这样需要根据elasticsearch的 config文件内容，
相应的配置一些文件，比如日志文件，data数据存储的目录，和node节点的命名

下面是我的elasticsearch的文件目录内容

```
/usr/local/Cellar/elasticsearch-1.4.2

drwxr-xr-x 14 pathbox admin  476  3  7 14:49 ./
drwxrwxr-x 67 pathbox admin 2.3K  3  6 22:08 ../
-rw-r--r--  1 pathbox admin 6.1K  3  6 22:08 .DS_Store
-rw-r--r--  1 pathbox admin  12K  3  6 22:08 LICENSE.txt
-rw-r--r--  1 pathbox admin  150  3  6 22:08 NOTICE.txt
-rw-r--r--  1 pathbox admin 8.3K  3  6 22:08 README.textile
drwxr-xr-x  9 pathbox admin  306  3  7 11:53 bin/
drwxr-xr-x  5 pathbox admin  170  3  7 11:44 config/
drwxr-xr-x  3 pathbox admin  102  3  7 11:41 data/
-rw-r--r--  1 pathbox admin  929  3  7 14:52 homebrew.mxcl.elasticsearch.plist
drwxr-xr-x 10 pathbox admin  340  3  7 11:32 ik/
drwxr-xr-x 25 pathbox admin  850  3  6 22:08 lib/
drwxr-xr-x 22 pathbox admin  748  3 25 00:03 logs/
drwxr-xr-x  3 pathbox admin  102  3  7 11:44 plugins/
```

注意ik分词包的dic文件包放在 /usr/local/Cellar/elasticsearch-1.4.2/ik中

jar文件包放在 /usr/local/Cellar/elasticsearch-1.4.2/plugins/ik

这和Ubuntu的目录不一样，否则会导致加载不到ik分词包而报错(这折腾了很久)

##### 安装head插件

```
进入到elasticsearch/bin路径

路径下有plugin 命令文件
sudo ./plugin -install mobz/elasticsearch-head

安装完插件之后会在es节点bin路径同级创建一个plugins目录,存放安装的插件

重启elasticsearch
sudo service elasticsearch restart

访问 http://localhost:9200/_plugin/head/

简单的使用教程： http://www.sojson.com/blog/85.html
```
