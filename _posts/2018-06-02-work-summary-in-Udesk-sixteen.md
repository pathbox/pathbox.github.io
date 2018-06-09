---
layout: post
title: 最近工作总结(十六)
date:   2018-06-02 20:45:06
categories: Work
image: /assets/images/post.jpg
---

##### Tow point about Heap struct

 - Heap property (比如: 父节点总比子节点小)
 - Shape property (树形)

 http://www.icourse163.org/learn/NTHU-451013?tid=522006#/learn/content?type=detail&id=947058&sm=1

Heap 的`insert`，先插入一个新的节点到最后的子节点的位置，这样保证了 Shape property的合理性，然后再逐一和父节点比较，进行交换，直到不满足交换条件的位置停止，这时满足 Heap property。

Heap (小顶堆)的找自小的数，并取出。O(log(n))

##### Heap、Stack内存模型

- Heap: 程序运行的时候，操作系统会给它分配一段内存，用来储存程序和运行产生的数据.程序运行过程中，对于动态的内存占用请求（比如新建对象，或者使用malloc命令），系统就会从预先分配好的那段内存之中，划出一部分给用户，具体规则是从起始地址开始划分（实际上，起始地址会有一段静态数据，这里忽略）。举例来说，用户要求得到10个字节内存，那么从起始地址0x1000开始给他分配，一直分配到地址0x100A，如果再要求得到22个字节，那么就分配到0x1020。这种因为用户主动请求而划分出来的内存区域，叫做 Heap（堆）。它由起始地址开始，从低位（地址）向高位（地址）增长。Heap 的一个重要特点就是不会自动消失，必须手动释放，或者由垃圾回收机制来回收。

- Stack: Stack 是由于函数运行而临时占用的内存区域。系统开始执行main函数时，会为它在内存里面建立一个帧（frame），所有main的内部变量（比如a和b）都保存在这个帧里面。main函数执行结束后，该帧就会被回收，释放所有的内部变量，不再占用空间.

```c
int main() {
   int a = 2;
   int b = 3;
   return add_a_and_b(a, b);
}
```

```
上面代码中，main函数内部调用了add_a_and_b函数。执行到这一行的时候，系统也会为add_a_and_b新建一个帧，用来储存它的内部变量。也就是说，此时同时存在两个帧：main和add_a_and_b。一般来说，调用栈有多少层，就有多少帧.等到add_a_and_b运行结束，它的帧就会被回收，系统会回到函数main刚才中断执行的地方，继续往下执行。通过这种机制，就实现了函数的层层调用，并且每一层都能使用自己的本地变量。

所有的帧都存放在 Stack，由于帧是一层层叠加的，所以 Stack 叫做栈。生成新的帧，叫做"入栈"，英文是 push；栈的回收叫做"出栈"，英文是 pop。Stack 的特点就是，最晚入栈的帧最早出栈（因为最内层的函数调用，最先结束运行），这就叫做"后进先出"的数据结构。每一次函数执行结束，就自动释放一个帧，所有函数执行结束，整个 Stack 就都释放了。
```

##### Algorithms Behind Modern Storage Systems

Different uses for read-optimized B-trees and write-optimized LSM-trees

>https://queue.acm.org/detail.cfm?id=3220266

##### quick find in Linux

- sudo find ~/ -name 完整名称 在home 目录寻找文件
- sudo find / -name 完整名称 在整个根目录寻找文件

##### Golang 交叉编译 在Mac下编译Linux二进制文件
```
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build main.go

#写到Makefile中

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build ${LDFLAGS} -o bin/udesk_ivr udesk/ivr
```

##### 一致性hash算法总结

一致性hash算法主要用在分布式缓存中。普通的hash算法使用方式是：key%N(N为服务器数目), 当服务器数目发送增加或减少时, 分配方式则变为key%(N+1)或key%(N-1).将会有大量的key失效迁移,如果后端key对应的是有状态的存储数据,这种做法将导致服务器间大量的数据迁移,从而照成服务的不稳定

一致性hash算法尽可能的减少了key的失效迁移，只是导致失效的那台节点服务器的key的迁移，这是合理的

一致性hash的核心思想为将key作hash运算, 并按一定规律取整得出0-2^32-1之间的值, 环的大小为2^32，key计算出来的整数值则为key在hash环上的位置，如何将一个key，映射到一个节点， 这里分为两步.
第一步, 将服务的key按该hash算法计算,得到在服务在一致性hash环上的位置.
第二步, 将缓存的key，用同样的方法计算出hash环上的位置，按顺时针方向，找到第一个大于等于该hash环位置的服务key，从而得到该key需要分配的服务器。

虚拟节点提高均衡性

 由于节点只有3个，存在某些节点所在位置周围有大量的hash点从而导致分配到这些节点到key要比其他节点多的多，这样会导致集群中各节点负载不均衡，为解决这个问题，引入虚拟节点， 即一个实节点对应多个虚拟节点。缓存的key作映射时，先找到对应的虚拟节点，再对应到实节点。每个节点虚拟出多个虚拟节点，从而提高均衡性

对于集群中缓存类数据key的节点分配问题，有这几种解决方法，简单的hash取模，槽映射，一致性hash。

hash取模
对于hash取模，均衡性没有什么问题，但是如果集群中新增一个节点时，将会有N／（N+1）的数据实效，当N值越大，失效率越高。这显然是不可接受的。

槽映射
redis采用的就是这种算法, 其思想是将key值做一定运算（如crc16， crc32，hash）， 获得一个整数值，再将该值与固定的槽数取模（slots）， 每个节点处理固定的slots。获取key所在的节点时，先要计算出key与槽的对应关系，再通过槽与节点的对应关系找到节点，这里每次新增节点时，只需要迁移一定槽对应的key即可，而不迁移的槽点key值则不会实效，这种方式将实效率降低到了 N／（N+1）。不过这种方式有个缺点就是所有节点都需要知道槽与节点对应关系，如果client端不保存槽与节点的对应关系的话，它需要实现重定向的逻辑。

一致性hash
一致性hash如上文所言，其新增一个节点的实效率仅为N／（N+1），通过一致性hash最大程度的降低了实效率。同时相比于槽映射的方式，不需要引人槽来做中间对应，最大限度的简化了实现

```
hashRing 用[]uint32表示，可以知道hashRing的最大值就是 2^64-1。理论上2^32-1就够了。构造虚拟节点，并不表示需要构造2^32-1个。虚拟节点的key尽量随机分配在hashRing中，利于平衡性。每个虚拟节点对应一个真实节点，这里用了map结构存虚拟节点和真实节点的对应关系，key就是0-2^32-1的hash数值，value就是真实节点struct。

不可或缺的hash算法：crc32.ChecksumIEEE([]byte(key)) 会得到一个0-2^32-1的hash值。用于产生虚拟节点的key，再把这个key值存到hashRing中，排序。这样，虚拟节点就在hashRing hash环上了

查询key，对其也做hash算法，得到0-2^32-1的hash值。利用二叉查找算法sort.Search，在hashRing中，找到第一个大于等于该hash环位置的虚拟节点的key值，得到这个虚拟节点，从而得到虚拟节点对应的真实节点值。
```
