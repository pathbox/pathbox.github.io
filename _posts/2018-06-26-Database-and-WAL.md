---
layout: post
title: 初识数据库的WAL
date:   2018-06-02 20:45:06
categories: Work
image: /assets/images/post.jpg
---


>预写式日志（Write-ahead logging，缩写 WAL）是关系数据库系统中用于提供原子性和持久性（ACID属性中的两个）的一系列技术。在使用WAL的系统中，所有的修改在提交之前都要先写入log文件中

log文件中通常包括redo和undo信息。假设一个程序在执行某些操作的过程中机器掉电了。在重新启动时，程序可能需要知道当时执行的操作是成功了还是部分成功或者是失败了。如果使用了WAL，程序就可以检查log文件，并对突然掉电时计划执行的操作内容跟实际上执行的操作内容进行比较。在这个比较的基础上，程序就可以决定是撤销已做的操作还是继续完成已做的操作，或者是保持原样。


参考链接：

http://m.blog.itpub.net/15498/viewspace-2134411/

http://hbasefly.com/2016/12/10/hbase-parctice-write/