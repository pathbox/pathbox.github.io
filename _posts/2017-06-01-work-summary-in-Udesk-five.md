---
layout: post
title: 最近工作总结(五)
date:   2017-06-01 11:08:06
categories: Work
image: /assets/images/post.jpg
---

##### 多个条件进行操作的时候，难免遇到冲突
比如触发器，不同的触发器的触发条件如果有冲突，顺序不同，影响不同。如果A触发器先执行更改记录，会导致B触发器条件不满足而无法执行。如果让B先执行，则两者都可以顺利执行。

##### 为Ubuntu升级JDK　到java 8
到官网下载　JDK-1.8　版本

mkdir -p /usr/lib/jvm　如果目录不存在。

将压缩包解压到　/usr/lib/jvm

sudo ln -s jdk1.8.0_131 java-8

 修改.bashrc　如果用zsh, 则修改.zshrc

```
export JAVA_HOME=/usr/lib/jvm/java-8  
export JRE_HOME=${JAVA_HOME}/jre  
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
export PATH=${JAVA_HOME}/bin:$PATH  
```

source ~/.bashrc  

配置默认JDK版本

```
sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/java-8/bin/java 300  
sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/java-8/bin/javac 300  
sudo update-alternatives --config java

选择 路径 优先级 状态  
------------------------------------------------------------  
0 /usr/lib/jvm/java-6-openjdk/jre/bin/java 1061 自动模式  
1 /usr/lib/jvm/java-6-openjdk/jre/bin/java 1061 手动模式  
2 /usr/lib/jvm/java-8/bin/java 300 手动模式  
要维持当前值 请按回车键，或者键入选择的编号：2  
update-alternatives: 使用 /usr/lib/jvm/java-8/bin/java 来提供 /usr/bin/java (java)，于 手动模式 中。
```

验证：

```
java -version
java version "1.8.0_131"
Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, mixed mode)

```
