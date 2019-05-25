---
layout: post
title: 最近工作总结(27)
date:  2019-05-06 17:00:06
categories: Work
image: /assets/images/post.jpg
---

### Golang构造hook函数方式的思路

- 数据结构: 利用map[string]func() 存储func()

- 注册: 初始化或有方法可以将func()注册到map中

- 执行: 在需要hook 函数的地方，从map中取出相应的func() f, 执行 f()


### ASFK-理解cookie和session
- Cookie由浏览器管理
- Cookie是明文的，用于存储非敏感机密数据
- Cookie具有不可跨域名性
- Cookie中使用Unicode字符时需要对Unicode字符(如中文)进行编码，否则会乱码
- Cookie不仅可以使用ASCII字符与Unicode字符，还可以使用二进制数据。例如在Cookie中使用数字证书，提供安全度。使用二进制数据时也需要进行编码。注意：本程序仅用于展示Cookie中可以存储二进制内容，并不实用。由于浏览器每次请求服务器都会携带Cookie，因此Cookie内容不宜过多，否则影响速度。Cookie的内容应该少而精
- Cookie的maxAge决定着Cookie的有效期，单位为秒（Second）。Cookie中通过getMaxAge()方法与setMaxAge(int maxAge)方法来读写maxAge属性
- Cookie并不提供修改、删除操作。如果要修改某个Cookie，只需要新建一个同名的Cookie，添加到response中覆盖原来的Cookie。如果要删除某个Cookie，只需要新建一个同名的Cookie，并将maxAge设置为0，并添加到response中覆盖原来的Cookie。注意是0而不是负数。负数代表其他的意义。
- 正常情况下，同一个一级域名下的两个二级域名如www.helloweenvsfei.com和 images.helloweenvsfei.com也不能交互使用Cookie，因为二者的域名并不严格相同。如果想所有 helloweenvsfei.com名下的二级域名都可以使用该Cookie，需要设置Cookie的domain参数，例如：
```
Cookie cookie = new Cookie("time","20080808"); // 新建Cookie
cookie.setDomain(".helloweenvsfei.com");           // 设置域名
cookie.setPath("/");                              // 设置路径
cookie.setMaxAge(Integer.MAX_VALUE);               // 设置有效期
response.addCookie(cookie);                       // 输出到客户端
```

- Cookie的路径
```
domain属性决定运行访问Cookie的域名，而path属性决定允许访问Cookie的路径（ContextPath）。例如，如果只允许/sessionWeb/下的程序使用Cookie，可以这么写：
Cookie cookie = new Cookie("time","20080808");     // 新建Cookie
cookie.setPath("/session/");                          // 设置路径
response.addCookie(cookie);                           // 输出到客户端
设置为“/”时允许所有路径使用Cookie。path属性需要使用符号“/”结尾。name相同但domain相同的两个Cookie也是两个不同的Cookie。
注意：页面只能获取它属于的Path的Cookie。例如/session/test/a.jsp不能获取到路径为/session/abc/的Cookie。使用时一定要注意。
```
- Cookie的安全属性
```
HTTP协议不仅是无状态的，而且是不安全的。使用HTTP协议的数据不经过任何加密就直接在网络上传播，有被截获的可 能。使用HTTP协议传输很机密的内容是一种隐患。如果不希望Cookie在HTTP等非安全协议中传输，可以设置Cookie的secure属性为 true。浏览器只会在HTTPS和SSL等安全协议中传输此类Cookie。下面的代码设置secure属性为true：

Cookie cookie = new Cookie("time", "20080808"); // 新建Cookie
cookie.setSecure(true);                           // 设置安全属性
response.addCookie(cookie);                        // 输出到客户端
提示：secure属性并不能对Cookie内容加密，因而不能保证绝对的安全性。如果需要高安全性，需要在程序中对Cookie内容加密、解密，以防泄密。

```
- JavaScript操作Cookie

Cookie是保存在浏览器端的，因此浏览器具有操作Cookie的先决条件。浏览器可以使用脚本程序如 JavaScript或者VBScript等操作Cookie。这里以JavaScript为例介绍常用的Cookie操作。例如下面的代码会输出本页面 所有的Cookie。

<script>document.write(document.cookie);</script>

由于JavaScript能够任意地读写Cookie，有些好事者便想使用JavaScript程序去窥探用户在其他网 站的Cookie。不过这是徒劳的，W3C组织早就意识到JavaScript对Cookie的读写所带来的安全隐患并加以防备了，W3C标准的浏览器会 阻止JavaScript读写任何不属于自己网站的Cookie。换句话说，A网站的JavaScript程序读写B网站的Cookie不会有任何结果。

- Session往往以key-value的结构存储

- Session是另一种记录客户状态的机制，不同的是Cookie保存在客户端浏览器中，而Session保存在服务器上。客户端浏览器访问服务器的时候，服务器把客户端信息以某种形式记录在服务器上。这就是Session。客户端浏览器再次访问时只需要从该Session中查找该客户的状态就可以了。

如果说Cookie机制是通过检查客户身上的“通行证”来确定客户身份的话，那么Session机制就是通过检查服务器上的“客户明细表”来确认客户身份。Session相当于程序在服务器上建立的一份客户档案，客户来访的时候只需要查询客户档案表就可以了

Session保存在服务器端。为了获得更高的存取速度，服务器一般把Session放在内存里。每个用户都会有一个独立的Session。如果Session内容过于复杂，当大量客户访问服务器时可能会导致内存溢出。因此，Session里的信息应该尽量精简。

Session生成后，只要用户继续访问，服务器就会更新Session的最后访问时间，并维护该Session。用户每访问服务器一次，无论是否读写Session，服务器都认为该用户的Session“活跃（active）”了一次。

- 用redis存储session。redis非常适合存储session，满足了key-value存储，速度足够快，有过期时间。
- cookie和session的关系:
1. session数据可以存储在cookie中，为了避免安全信息暴露，需要加密存与cookie
2. session存于redis中，但需要一个Session ID(或session key)存于cookie，用于客户端唯一标记信息。请求时，cookie中带有Session ID 服务端能够取到，则可以在redis中根据Session ID 获取到具体的session数据用于逻辑操作。
3. 浏览器将cookie关了，则session功能也会失效，需要存于URL中

### 快速尝试MySQL Full-Text Searching

MySQL 5.6以上版本支持了 Full-Text Searching，5.7.6 之后支持了中日韩文的全文检索

查看本机MySQL版本：
```
mysql --version
mysql  Ver 14.14 Distrib 5.7.19, for osx10.12 (x86_64) using  EditLine wrapper
```

新增测试表:
```
CREATE TABLE `posts` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `title` varchar(255) NOT NULL DEFAULT '',
  `content` text,
  PRIMARY KEY (`id`),
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

给content字段增加全文索引:
```sql
ALTER TABLE `mars`.`posts`
ADD FULLTEXT INDEX `ft_content` (`content` ASC)  WITH PARSER ngram;
```

给(title,content)字段增加联合的全文索引:
```sql
ALTER TABLE `posts`
ADD FULLTEXT INDEX `ft_posts_title_body` (`title`, `content`) WITH PARSER ngram;
```

插入一些测试数据

使用全文索引查询:
```sql
mysql> SELECT title FROM posts where match(content) against('没有' IN NATURAL LANGUAGE MODE);
+------------------------------+
| title                        |
+------------------------------+
| 第一章                       |
| 第二章-如果是个索引          |
+------------------------------+
2 rows in set (0.00 sec)

SELECT * FROM posts where match(title,content) against('第二章' IN NATURAL LANGUAGE MODE);
+----+------------------------------+--------------------------------------------------+
| id | title                        | content                                          |
+----+------------------------------+--------------------------------------------------+
|  2 | 第二章-如果是个索引          | 火星上没有火星人只有泥土和环形山                 |
+----+------------------------------+--------------------------------------------------+
1 row in set (0.01 sec)
```
- WITH PARSER ngram: 表示索引使用的分词器
- IN NATURAL LANGUAGE MODE: 表示匹配时的模式
- 如果使用的是多字段联合的全文索引，只要任一个字段中能匹配成功，则得到该记录

```
● IN BOOLEAN MODE的特色：
          ·不剔除50%以上符合的row。
          ·不自动以相关性反向排序。
          ·可以对没有FULLTEXT index的字段进行搜寻，但会非常慢。
          ·限制最长与最短的字符串。
          ·套用Stopwords。

● 搜索语法规则：
 +   一定要有(不含有该关键词的数据条均被忽略)。
 -   不可以有(排除指定关键词，含有该关键词的均被忽略)。
 >   提高该条匹配数据的权重值。
 <   降低该条匹配数据的权重值。
 ~   将其相关性由正转负，表示拥有该字会降低相关性(但不像 - 将之排除)，只是排在较后面权重值降低。
 *   万用字，不像其他语法放在前面，这个要接在字符串后面。
 " " 用双引号将一段句子包起来表示要完全相符，不可拆字
 1、预设搜寻是不分大小写，若要分大小写，columne 的 character set要从utf8改成utf8_bin。
 2、预设 MATCH...AGAINST 是以相关性排序，由高到低。
 3、MATCH(title, content)里的字段必须和FULLTEXT(title, content)里的字段一模一样。如果只要单查title或content一个字段，那得另外再建一个 FULLTEXT(title) 或 FULLTEXT(content)，也因为如此，MATCH()的字段一定不能跨table，但是另外两种搜寻方式好像可以
```

### flutter in action
https://github.com/flutterchina/flutter-in-action/blob/master/docs/SUMMARY.md

### 在Golang系统中,处理MySQL null字段值, 为所有字段都设置Default是最好的
Golang sql 包提供了比如：

sql.NullString 这样的字段类型
```go
type NullString string {
  String string
  Valid bool // Valid is true if String is not NULL
}
```
但这样，进行json.Marshal的时候，返回的就会是：
```
name: {
  "String": "my name",
  "Valid": true
}
```
这样的结构，实际希望的是 "my name" or ""

也可以直接 使用*string, 会返回 "my name" or null,当是null时，在response之前想要进行一些逻辑操作就会有困难

最好的方式, 将表的字段都设为Not Allow, 设置对应的Default值

### npm替换为淘宝源
```
1.得到原本的镜像地址

npm get registry

> https://registry.npmjs.org/
设成淘宝的

npm config set registry https://registry.npm.taobao.org/
yarn config set registry https://registry.npm.taobao.org/

2.换成原来的
npm config set registry https://registry.npmjs.org/
```

### LEFT JOIN ON 多个条件 用的是AND 而不是WHERE

```
1. SELECT * FROM product LEFT JOIN product_details
         ON (product.id = product_details.id)
         AND product_details.id=2;
2. SELECT * FROM product LEFT JOIN product_details
         ON (product.id = product_details.id)
         WHERE product_details.id=2;
```
1、2两个查询是完全不同的。

1: 得到的结果数量是product的数量。是真正`LEFT JOIN` 的结果
2: 得到的结果数量是满足`product_details.id=2`条件的记录数量, 是WHERE条件过滤后的结果

### B+树优于B树

我明白了，如果是多条的话，B树需要做局部的中序遍历，可能要跨层访问(可能会增加磁盘操作)。而B+树由于所有数据都在叶子结点，不用跨层，同时由于有链表结构，只需要找到首尾，通过链表就能把所有数据取出来了

### 设计模式视频学习思考
- 稳定与不稳定部分的设计
- 外层和内层的设计
- 耦合的设计

### 构建器Builder
- Builder模式主要用于“分步骤构建一个复杂的对象”。在这其中“分步骤”是一个稳定的算法，而复杂的对象的各个部分则经常变化
- 变化点在哪里，封装哪里。Builder模式主要在于应对“复杂对象各个部分”的频繁需求变动。其缺点在于难以应对“分步骤构建算法”的需求变动
- 数据库ORM设计中，就可以分为Builder部分(准备SQL语句)和执行部分(执行SQL语句)

### 双检查锁设计失效
```
if (m_instance==nullptr) {
  Lock lock; // 加锁
  if (m_instance == nullptr) {
    m_instance = new Signleton();
  }
  return m_instance
}
```
由于内存读写reorder会导致双检查锁失效。
正常顺序 分配内存、实现构造器Signleton()、赋值给m_instance
reorder 的顺序 分配内存、赋值给m_instance、实现构造器Signleton(),则造成m_instance!=nullptr而直接返回，但此时的m_instance是不能用的

不同语言的编译器开始有对其做优化，避免reorder

### 私有(安全)下载
- 是否需要权限，比如登入和下载权限校验
- 下载URL是否具有时效性
如果将上述两种条件况结合到一起使用，安全效果是最好的，既校验了用户权限，又能避免下载URL被恶意传播后还能下载

### ASFK 邮箱、手机、银行账户的真实性
反馈对比机制

### Golang 的 struct method 调用者是struct或&struct{}
```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	s1 := S{}
	s2 := &S{}
	s1.Name()
	s1.Age()

	s2.Name()
	s2.Age()

	t1 := reflect.TypeOf(s1)
	fmt.Println("===============t1")
	for i := 0; i < t1.NumMethod(); i++ {
		fmt.Println(t1.Method(i).Name)
	}

	t2 := reflect.TypeOf(s2)
	fmt.Println("===============t2")
	for i := 0; i < t2.NumMethod(); i++ {
		fmt.Println(t2.Method(i).Name)
	}

  t3 := t2.Elem() // It is t3
  fmt.Println("===============t3")
	for i := 0; i < t3.NumMethod(); i++ {
		fmt.Println(t3.Method(i).Name)
	}
}

type S struct {
}

func (s S) Name() {
	fmt.Println("Name")
}

func (s *S) Age() {
	fmt.Println("Age")
}
```

```
Name
Age
Name
Age

===============t1
Name
===============t2
Age
Name
```

### 多数据源存储
- 商品的基本信息：MySQL等关系数据库
- 商品描述、详情、评价信息(多文字类)：MongoDB
- 商品图片：文件系统TFS
- 商品关键字：搜搜引擎，比如：ES
- 商品热点高频信息：Redis
- 商品交易、价格计算、积分累积：外部交易金融系统

统一数据平台服务层:UDSL，对接了多种数据源数据库数据类型，对开发使用者来说不知道有多种不同的数据源数据库，只根据UDSL的API接口来获取查询得到想要的数据
