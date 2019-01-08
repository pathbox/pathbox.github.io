---
layout: post
title: 最近工作总结(23)
date:  2019-01-07 10:20:06
categories: Work
image: /assets/images/post.jpg
---

### Hash Func result length and bit size

MD5 hexdecimal(hex) string length is 32, MD5 result data bit size is 128 bits. One hex string char is 4 bits.

MD5 hashes are 128 bits in length and generally represented by 32 hex digits.

As we know SHA-1 hashes are 160 bits in length and generally represented by 40 hex digits.

SHA-2 hashes are 256bit in length and generally represented by 64 hex digits

So SHA-512 can be represented by 128 hex digits

### MySQL 为order by 使用索引

KEY a_b_c (a,b,c)

order by 能使用最左前缀索引:
- ORDER BY a
- ORDER BY a,b
- ORDER BY a,b,c
- ORDER BY a DESC, b DESC, c DESC

如果WHERE使用索引的最左前缀定义为常量，则ORDER BY 能能使用索引:
- WHERE a = const ORDER BY b, c
- WHERE a = const AND b = const ORDER BY c
- WHERE a = const AND b > const ORDER BY b, c

不能使用索引进行排序
- ORDER BY a ASC, b DESC, c DESC 排序不一致
- WHERE g = const ORDER BY b, c 丢失a索引
- WHERE a = const ORDER BY c 四队b索引
- WHERE a = const ORDER BY a,d d不是索引的一部分
- WHERE a in (...) ORDER BY b,c 对于排序来说，多个相等条件也是范围查询
