---
layout: post
title: 理解Goalng线程安全的sync.Map的实现
date: 2018-04-05 09:00:06
categories: Golang
image: /assets/images/post.jpg
---

最近被一位大牛问了是否知道Golang的线程安全的`sync.Map`是如何实现的,之前有使用过`sync.Map`,但没有仔细阅读过源码.于是花了一些时间认真阅读了`sync.Map`源码,并且查阅了一些其他人做的总结资料.

源码中的注释已经比较详尽了,并且网上也有中文版的代码解释,本文只是总结对sync.Map实现的几点思考.

>Golang选择了 CAS 这种 Lock - Free 的方案,实现并发安全的map.

使用`sync/atomic`包对进行读和写操作
还有互斥量锁的方案和分段锁的方案.[在这篇文章](https://www.jianshu.com/p/43e66dab535b)作者对这三种方案进行了对比和benchmark,值得一看.

### 两个map

定义了`read`和`dirty`两个map

```go
type readOnly struct {
	m       map[interface{}]*entry
	amended bool // true if the dirty map contains some key not in m. // key在dirty中,不在read中
}

dirty map[interface{}]*entry

type entry struct {
	// p points to the interface{} value stored for the entry.
	//
	// If p == nil, the entry has been deleted and m.dirty == nil.
	//
	// If p == expunged, the entry has been deleted, m.dirty != nil, and the entry
	// is missing from m.dirty.  // entry不存在 dirty中
	//
	// Otherwise, the entry is valid and recorded in m.read.m[key] and, if m.dirty
	// != nil, in m.dirty[key].
	//
	// An entry can be deleted by atomic replacement with nil: when m.dirty is
	// next created, it will atomically replace nil with expunged and leave
	// m.dirty[key] unset.
	//
	// An entry's associated value can be updated by atomic replacement, provided
	// p != expunged. If p == expunged, an entry's associated value can be updated
	// only after first setting m.dirty[key] = e so that lookups using the dirty
	// map find the entry.
	p unsafe.Pointer // *interface{}
}
```

`entry.p` 实际是一个指针.所以`read`和`dirty`中存的实际是`key=>值的指针`的结构,会存在存了两份值的指针,但不是存了两份值.这样,在用空间换时间的情况下,如果map存的key很多,也不会消耗大量内存,会增加消耗的是指针定义读内存的占用,这比值的占用要小很多.

> + read中的key是readOnly的（key的集合不会变，删除也只是打标记），value的操作全都可以原子完成，所以这个结构不用锁
+ dirty是一个read的拷贝，用锁的操作在这做，如增加元素、删除元素等（dirty上删除是真删除）

`read`并不是只是读操作,也有原子写操作.在读操作的时候,是使用了`atomic.Value的Load()`,没有用锁,达到了在线程安全的情况下读性能的大大提高

```go
func (m *Map) Store(key, value interface{}) {
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok && e.tryStore(&value) { // 如果 key在read.m中，相当于update操作，会直接修改read(cache层),而不会再去dirty进行加锁的赋值操作
		return
	}
	// ...
}
```

而同时也会把dirty中的对应的值给修改了，因为相同key的entry，在read和dirty中存的是entry的指针，两个指针没有变，指针的值改了，所以read和dirty 对应的entry同时改变。
为什么呢？因为当key不在map中，会进行map新建记录操作，相同的key既会在read中新建，也会在dirty中新建，新建的就是相应entry的指针： `key=>*(entry)`。这样，当进行update操作时，就直接修改read，而不需要再加锁操作dirty，性能好多了。

而将`read`中的值同步到`dirty`或对`dirty`进行读或写操作时,使用了互斥量锁`Mutex`.
写操作分为新增和更新.新增的时候,需要把这个值指针存到`dirty`.更新的时候,更新一个没有被标记过expunged的key,直接对read进行`atomic.CompareAndSwapPointer`操作就可以,如果之前expunged过,将key同步到dirty.
而在Store、Delete操作的时候,都要进行`m.read.Load().(readOnly)`操作.根据上文中推荐的文章进行的benchmark的结果,`sync.Map`的写、删除性能不尽如意,这个结果可以预料到.

> + 当dirty不存在的时候，read就是全部map的数据
+ 当dirty存在的时候，dirty才是正确的map数据

### amended
amended 标记read是否创建了dirty副本

### entry.p
`read`和`dirty`中的map存的元素值是`entry`,`entry`的field是`p unsafe.Pointer`,是指向具体存储值的指针,这个具体值是以interface{}值存在的,所以在Load取出值的时候,要自行做`.(type)`的转换.

假设`read`和`dirty`中的map存在`name`这个key,当`read`进行了`atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i))`操作后,改的是p这个指针所指的值的地址,而entry这个指针没有改变,`read`和`dirty`中的map存在`name`指的是同一个entry.所以,`read`中对p的写原子操作,如果`dirty`中有相同的key,也会同样被更改,因为他们的entry是同一个.

### misses

misses 的作用就是，当从read读取值没有读取到，从`dirty`中读取到了，自增加1. 当这种情况达到 `m.misses < len(m.dirty)`的时候，`dirty`的值就代替为`read.m`(read是readOnly),然后misses重置为0，`dirty`置为nil，重新开始计算值.

[yiz96的文章中](https://blog.yiz96.com/golang-sync-map/)：

>可以把read看成是一个cache，当cache miss到一定数量的时候，dirty中的数据会被提升到read中去。但是决定哪些数据应该过去实在太费时了，倒不如时间换空间，read中的数据我在dirty中也存一份，提升的时候直接整个赋值就好了~

`read`相当于cache层，`dirty`是更底层的数据层，当`read`多次没有命中数据时，达到条件，这个cache层命中率太低了，直接将整个`read`用`dirty`替换，然后`dirty`又重新为nil，不需要马上将`read`同步到`dirty`，而是下一次`Store`一个新的key的时候，再触发进行一次`dirty`的初始同步,并且初始同步在`dirty`的一个生存周期内,只会进行一次.

在进行`Store`和`Load`的时候,其实都是先操作`read`,如果`read`中存在并且对应值没有被expunged过,就执行返回了,如果`read`中不存在,或者对应值被expunged,就需要对`dirty`进行操作,将这个key同步到`dirty`中.

综上所述,如果你的map使用 `读远大于写操作`(这里的写是指新增和删除key, 修改key还是用的原子性操作),`sync.Map`会是很好的选择,而读写性能都比较优秀的的map,Golang中是[current-map](https://github.com/orcaman/concurrent-map).下回也好好阅读一下`current-map`的实现代码.


参考连接:
```
https://www.jianshu.com/p/43e66dab535b
https://blog.yiz96.com/golang-sync-map/
https://blog.csdn.net/jiankunking/article/details/78808978
```
