---
layout: post
title: 短网址、发号器 系统构建分析
date:   2018-02-22 15:27:06
categories: Server
image: /assets/images/post.jpg
---

##### 短网址的作用(来源知乎总结)

+ 节省网址长度，便于社交化传播
+ 方便后台跟踪点击量、地域分布等用户统计
+ 域名屏蔽，减少域名的暴露
+ 隐藏真实url地址，审核做付费推广
+ 结合生成二维码

##### 短网址原理
长网址经过某种转换成为短网址，一个长网址可以拥有一个或多个短网址

```
long url => func() => short url
```

访问短网址，短网址经过 url重定向到长网址url。用户在页面上看到的短网址，最后是重定向到了真实的长网址

```
short url => short server => long url => backend server
```

例子： 访问  http://g.cn/R32DNa， 会访问 http://g.cn 地址的后台，后台通过 R32DNa短码，找到对应的长网址url，再重定向访问长地址url

short server 服务提供重定向到 长网址的功能，还可以在short server 服务上进行点击访问统计，用户地域分析等等统计功能。
像Google、新浪等短网址服务提供商大都集成了类似的统计后台功能（有的可能需要付费）

##### 短网址算法实现

查阅网上资料，有两种较好的算法实现。

这里设计实现的短网址长度是6位

1. Hash算法

```
将长网址md5生成32位签名串，分为4段，每段8个字节；
对这四段循环处理，取8个字节，将他看成16进制串与0x3fffffff(30位1)与操作，即超过30位的忽略处理；
这30位分成6段，每5位的数字作为字母表的索引取得特定字符，依次进行获得6位字符串；
总的md5串可以获得4个6位串；取里面的任意一个就可作为这个长url的短url地址；
```

一种Java代码的实现

```java
public   static   string [] ShortUrl( string  url)  
{  
    //可以自定义生成MD5加密字符传前的混合KEY   
    string  key =  "Leejor" ;  
    //要使用生成URL的字符   
    string [] chars =  new   string []{  
        "a" , "b" , "c" , "d" , "e" , "f" , "g" , "h" ,  
        "i" , "j" , "k" , "l" , "m" , "n" , "o" , "p" ,  
        "q" , "r" , "s" , "t" , "u" , "v" , "w" , "x" ,  
        "y" , "z" , "0" , "1" , "2" , "3" , "4" , "5" ,  
        "6" , "7" , "8" , "9" , "A" , "B" , "C" , "D" ,  
        "E" , "F" , "G" , "H" , "I" , "J" , "K" , "L" ,  
        "M" , "N" , "O" , "P" , "Q" , "R" , "S" , "T" ,  
        "U" , "V" , "W" , "X" , "Y" , "Z"   
    };  

    //对传入网址进行MD5加密   
    string  hex = System.Web.Security.FormsAuthentication.HashPasswordForStoringInConfigFile(key + url,  "md5" );  

    string [] resUrl =  new   string [4];  

    for  ( int  i = 0; i < 4; i++)  
    {  
        //把加密字符按照8位一组16进制与0x3FFFFFFF进行位与运算   
        int  hexint = 0x3FFFFFFF & Convert.ToInt32( "0x"  + hex.Substring(i * 8, 8), 16);  
        string  outChars =  string .Empty;  
        for  ( int  j = 0; j < 6; j++)  
        {  
            //把得到的值与0x0000003D进行位与运算，取得字符数组chars索引   
            int  index = 0x0000003D & hexint;  
            //把取得的字符相加   
            outChars += chars[index];  
            //每次循环按位右移5位   
            hexint = hexint >> 5;  
        }  
        //把字符串存入对应索引的输出数组   
        resUrl[i] = outChars;  
    }  
    return  resUrl;  
}  
```

这种算法,虽然会生成4个短码,但是仍然存在重复几率， hash算法有可能会有碰撞发生

2.自增序列进制转换算法

设置全局自增的id， 将这个id转化为62进制数。 为什么是62呢？因为[a - z, A - Z, 0 - 9] 总共 62 个字母和数字。设计短网址长度为6位，总共有62^6 ～= 568亿种组合，基本够用了。

这里要构建一个大的数组，数组的元素就是[a - z, A - Z, 0 - 9]，每个元素对应62进制数的一位。十进制数id转为62进制数后，每一位对应数组中的一个元素，将这些元素连接在一起就得到了短码，不足的位可以用a补全，这个看具体的需求



```go

// https://github.com/pathbox/learning-go/blob/master/src/shorter_url/shorter/shorter.go

package shorter

import "strings"

var ALPHABET = strings.Split("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789", "")

func GetShortUrl(id int64) string {
	indexAry := Encode62(id)
	return GetString62(indexAry)
}

func Encode62(id int64) []int64 {
	indexAry := []int64{}
	base := int64(len(ALPHABET))

	for id > 0 { // i < 0 时,说明已经除尽了,已经到最高位,数字位已经没有了
		remainder := id % base
		indexAry = append(indexAry, remainder)
		id = id / base
	}

	return indexAry
}

//  输出字符串, 长度不一定为6
func GetString62(indexAry []int64) string {
	result := ""

	for _, val := range indexAry {
		result = result + ALPHABET[val]
	}

	return reverseString(result)
}

// 反转字符串
func reverseString(s string) string {
	runes := []rune(s)
	for from, to := 0, len(runes)-1; from < to; from, to = from+1, to-1 {
		runes[from], runes[to] = runes[to], runes[from]
	}
	return string(runes)
}

// # 进制的转换:

// # 1 取余操作%,得到最后一位,
// # 2 之后整除/操作,过滤调上一步已经得到的一位
// # 3 重复 1 直到 结果< 0 说明位数已经操作完了

```

这里的关键就是自增序列id的生成了。一种方案，可以使用MySQL等关系数据库的id，作为自增序列id，让MySQL帮你完成这个事情。自增序列id其实就是一个号，相当于就是一个发号器服务的设计。

设计MySQL表：

short_codes

| 字段  |  含义 |
|:------:|:------:|
|id|	自增序列id|
|url|	长连接|
|code|	短链接码（短码）|
|hit_count| 点击量 |
|type|	系统: "system" 自定义: "custom"|
|created_at|	创建时间 |
|updated_at|	更新时间 |


关于 点击量、访问的 ip 地域、用户使用的设备 等用户信息分析，还需要创建其他表帮助实现。

长网址每生成一个短网址时，就往short_codes表中插入一条记录。当点击短网址时，则使用短码在表中查找到对应的长网址。

这里的对应关系：一个长网址可以对应多个短网址，一个短网址只能对应一个长网址。

可以看出，当访问量大的时候，MySQL这很容易就成为性能的瓶颈。所以，可以使用分表分库，甚至使用多个MySQL数据库，将负载分散。

> 如何保证发号器的大并发高可用
上面设计看起来有一个单点，那就是发号器。如果做成分布式的，那么多节点要保持同步加1，多点同时写入，这个嘛，以CAP理论看，是不可能真正做到的。其实这个问题的解决非常简单，我们可以退一步考虑，我们是否可以实现两个发号器，一个发单号，一个发双号，这样就变单点为多点了？依次类推，我们可以实现1000个逻辑发号器，分别发尾号为0到999的号。每发一个号，每个发号器加1000，而不是加1。这些发号器独立工作，互不干扰即可。而且在实现上，也可以先是逻辑的，真的压力变大了，再拆分成独立的物理机器单元。1000个节点，估计对人类来说应该够用了。如果你真的还想更多，理论上也是可以的

###最佳方案:使用Redis进行发号器的设计

Redis将数据存储在内存，读写性能比MySQL高很多。定义一个全局的自增id key， short_url_id. 在生成一个短码的时候， incr(short_url_id), 利用得到的这个id生成短码。redis的 INCR 是原子性操作，而且性能很好。可以从10000之后开始，1-10000可以自己保留使用。

然后把其他详细信息数据存到MySQL表中，而Redis的作用只是生成自增序列id。对于使用redis集群，可以使用INCRBY num，分别间隔增加。

对Redis可用性的设计，防止Redis挂了，服务不可用。可以将Redis支持持久化，或者在MySQL表中增加一个字段，存储对应的自增id值

短码和长网址映射表的数据存在MySQL表中，但是可以利用Redis做缓存，这样，能够提高查询的性能。



如果生成短链的性能要求比较高，我们可以使用redis cluster集群。然后实现实现1000个(N个)逻辑发号器。分别发尾号为0到999的号。每发一个号，每个发号器加1000(步长)，而不是加1。这些发号器独立工作，互不干扰即可。而且在实现上，也可以先是逻辑的，真的压力变大了，再拆分成独立的物理机器单元。1000个节点，估计对人类来说应该够用了。如果你真的还想更多，理论上也是可以的。

得到号数(自增id值)之后，对号数进行base62编码即可得到一个短连接。

base62编码的个数表：

| 位数 | 个数      | 区间                    |
| ---- | --------- | ----------------------- |
| 1位  | 62        | 0 - 61                  |
| 2位  | 3844      | 62 - 3843               |
| 3位  | 约 23万   | 3844 - 238327           |
| 4位  | 约 1400万 | 238328 - 14776335       |
| 5位  | 约 9.1亿  | 14776336 - 916132831    |
| 6位  | 约 568亿  | 916132832 - 56800235583 |

### 数据表设计

links 表

| 字段       | 含义                            |
| ---------- | ------------------------------- |
| id         | link_id                         |
| url        | 长连接                          |
| keyword    | 短链接码                        |
| type       | 系统: "system" 自定义: "custom" |
| insert_at  | 插入时间                        |
| updated_at | 更新时间                        |



**那么怎么实现自定义短码呐？**

我是这样处理的：

> 数据库增加一个类型 type 字段，用来标记短码是用户自定义生成的，还是系统自动生成的。
> 如果有用户自定义过短码，把它的类型标记自定义。每次根据 id 计算短码的时候，如果发现对应的短码被占用了，就从类型为自定义的记录里选取一条记录，用它的 id 去计算短码。
> 这样既可以区分哪些长连接是用户自己定义还是系统自动生成的，还可以不浪费被自定义短码占用的 id



 可以直接使用redis的持久化方式+消息队列异步持久化到MySQL的方式。

同一个长地址多次转换，出来还是同一个短地址。

我上面其实讲到了，这个方案最简单的是建立一个长对短的hashtable，这样相当于用空间来换空间，同时换取一个设计上的优雅（真正的一对一）。

实际情况是有很多性价比高的打折方案可以用，这个方案设计因人而异了。那我就说一下我的方案吧。

方案是：用key-value存储，保存“最近”生成的长对短的一个对应关系。注意是“最近”，也就是说，我并不保存全量的长对短的关系，而只保存最近的。比如采用一小时过期的机制来实现LRU淘汰。

这样的话，长转短的流程变成这样： 1 在这个“最近”表中查看一下，看长地址有没有对应的短地址 1.1 有就直接返回，并且将这个key-value对的过期时间再延长成一小时 1.2 如果没有，就通过发号器生成一个短地址，并且将这个“最近”表中，过期时间为1小时

所以当一个地址被频繁使用，那么它会一直在这个key-value表中，总能返回当初生成那个短地址，不会出现重复的问题。如果它使用并不频繁，那么长对短的key会过期，LRU机制自动就会淘汰掉它。

 

>短网址的还原跳转用301还是302呢？
301是永久重定向，302是临时重定向。短地址一经生成就不会变化，所以用301是符合http语义的。同时浏览器会对301请求保存一个比较长期的缓存，这样就减轻对服务器的压力；而且301对于网址的SEO有一定的提升。但是很多情况下我们需要对接口点击或者用户行为进行一些业务监控处理的时候，301明显就不合适了（浏览器直接按照缓存数据跳转了）, 所以很多业务场景下还是采用302比较合适



### 后期功能扩展

统计：点击量、访问的 ip 地域、用户使用的设备

管理后台：删除、数据量

##### 关于短码是否有过期

生成的短码是否有过期性，这个需要根据具体的场景设计方案。有过期，需要考虑过期时间，清除数据回收策略。


参考链接：

```
https://www.zhihu.com/question/20790447
https://www.jianshu.com/p/d1cb7a51e7e5
https://segmentfault.com/a/1190000012088345
http://qn-lf.iteye.com/blog/1084516、
https://www.zhihu.com/question/29270034/answer/46446911
https://www.zhihu.com/question/29270034
```