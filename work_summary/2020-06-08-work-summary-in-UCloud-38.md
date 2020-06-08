---
layout: post
title: 最近工作总结(38)
date:  2020-06-08 11:12:06
categories: Work
image: /assets/images/post.jpg
---

### TiDB binlog 同步到MySQL时，TiDB会把SQL语句进行重新组装，比如UPADATE 拆分成DELETE 然后REPLACE INTO，这样通过binlog恢复或同步数据会有更高的性能吧。但是如果需要binlog原始操作逻辑进行区分的，这种重组方式就不合适了
