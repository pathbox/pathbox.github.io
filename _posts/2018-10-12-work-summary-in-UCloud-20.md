---
layout: post
title: 最近工作总结(二十)
date:   2018-10-12 17:32:06
categories: Work
image: /assets/images/post.jpg
---


### Linux系统下，查询当前目录下大文件夹,大文件数据

Linux系统下，查询当前目录下大文件夹数据，文件夹深度设为了2，当服务器磁盘报警的时候，可以用于查询是哪个文件夹下的数据占用磁盘最多，不是数据库服务器的话，一般都是日志啦

`sudo du -hm --max-depth=2 -h | sort -nr | head -20`

find . -type f -size +100M #查找100M以上的文件
