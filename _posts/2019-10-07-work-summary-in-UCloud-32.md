---
layout: post
title: 最近工作总结(32)
date:  2019-10-07 09:10:06
categories: Work
image: /assets/images/post.jpg
---

### Which concurrent map to use
>That very much depends on your concurrency needs, your hardware, and your requirements, but a few rules of thumb that might or might not help:

>If you have less than 4 cores, a single map with an RWMutex is all you need.
Many cores and goroutines:
Tons of reads, not many writes: sync.Map is appropriate.
Mixed reads and writes, go for a sharded map implementation. At the moment [1] is a veteran library, well tested. [2] is a newcomer but looks very good. This library is inspired on both. It should be a bit more performant than [1], while using standard Go maps and slightly less dependencies than [2]. Also, this library has native support for uint64 and UUID keys if you want that. However due to my lazyness and because I just use this for personal projects, this lib is not so well tested. Caveat Emptor (and PRs are welcome, or file an issue to motivate me into writing those tests!).
Other invariants to keep together with the map semantics: Write your own data structure or pick one implementation you like and fork it, then add those invariants there.
Mixed value types, TTL, queries... You don't want a map, you want either a cache or an in-memory database, depending on your exact requirements. Check for example [3] and [4].
Installation

### Ruby中只有多线程IO操作时能实现并发，在等待IO的时候GIL会释放，这些线程可以穿插执行

### MySQL ERROR 1071 (42000): Specified key was too long; max key length is 767 bytes
MySQL INNODB varchar字段的索引长度默认限制是767bytes，在utf8编码下，每个字符是3字节，所以varchar(255) 255*3=765 <767这样没有超过限制，如果将varchar(255)设为更大，则会导致报错。
utf8mb4编码每个字符占4个字节，767/4=191，所以在utf8mb4编码下varchar(191)这样是合理的。
