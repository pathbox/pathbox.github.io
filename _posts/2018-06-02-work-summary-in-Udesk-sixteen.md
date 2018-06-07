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
