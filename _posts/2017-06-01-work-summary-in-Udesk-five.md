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

#####　关于全文搜索的优化和观点
如果客户能输入一个更有意义，更准确的词去进行全文搜索，这样才是最好的搜索优化。
因为，一个更合适的词，能够更好的从大数据中进行匹配筛选，得到的数据也是更接近于想要搜索得到的。
举个例子：想要得到天安门的地址。　如果你输入"的地"和输入"天安门"，显然"天安门"能够更准确和快速的搜索得到
想要的结果。

在项目中，使用了nGram这样的分词。这个分词的作用是: abc => [a,ab,abc, ac, b, bc,c]这样的分词。如果用这样的分词，
是能够通过"的地"把结构搜索出来。但是，这样的分词有性能问题，产生了大量实际中使用不到的token。而且，在产生大量分词的时候，
对CPU的使用率是非常消耗的。并且在搜索长的字符串的时候，也会产生性能问题，或者慢日志。虽然，这种分词能够涵盖的搜索情况最大，因为分的
token最多。但是，随着数据量的增加，如果到一定的数据量，ES会很快就到性能瓶颈。即使增加集群，也会随着数据量增加而很快到达瓶颈。而进入
性能黑洞。所以，使用IK或英文的分词，将有意义的词分成token，产生倒排索引。当用户能够输入有意义的词语进行搜索的时候，就能很好的搜索到结果。如果输入的是没有什么意义的词，就直接不匹配到结果，这是合理的情况。而兼顾没有意义的分词的情况，这不叫智能。这也失去了真正全文搜索的真实意义。我觉得全文搜索的真实意义是：得到更加精确的匹配结果，而不是得到更多匹配的结果。
