---
layout: post
title: awk 强大的文本分析工具
date:   2018-05-29 21:24:06
categories: Tool
image: /assets/images/post.jpg
---


### 命令： `awk [options] 'script code' file`

- 逐行分析数据
- $0 表示当前行，$1 $2 $3... 表示不同的列
- 默认每列的分隔符为空格和制表符， 可以用 `-F` 参数指定分隔符:`awk -F ':' '{print $1}' /etc/passwd`

```
NR 表示文件中的行号，表示当前是第几行
NF 表示文件中的当前行列的个数，类似于 mysql 数据表里面每一条记录有多少个字段
FS 表示 awk 的输入分隔符，默认分隔符为空格和制表符，你可以对其进行自定义设置
OFS 表示 awk 的输出分隔符，默认为空格，你也可以对其进行自定义设置
FILENAME 表示当前文件的文件名称，如果同时处理多个文件，它也表示当前文件名称
```

`awk '{print NR "\t" $0}' access.log`

- 可以同时处理多个文件
- BEGIN关键字：在脚本代码段前面使用 BEGIN 关键字时，它会在开始读取一个文件之前，运行一次 BEGIN 关键字后面的脚本代码段， BEGIN 后面的脚本代码段只会执行一次，执行完之后 awk 程序就会退出
- END关键字：awk 的 END 指令和 BEGIN 恰好相反，在 awk 读取并且处理完文件的所有内容行之后，才会执行 END 后面的脚本代码段

- awk 脚本中可以使用变量，数学运算

### 实战：

### 读取Nginx access.log日志，得到访问最多的前10个请求及访问次数

```sh
awk '{a[$7]++}END{for(i in a){print a[i] " -- " i}}' access.log | sort -rn | head -10
```

```
256154 -- /
272 -- /spa1/sys_intergration/system_is_alive?id=1
44 -- /spa1/bi
15 -- 400
12 -- /backend//field/info?token=tiandixuanhuangyuzhouhonghuang_62
10 -- /im_client/locales/zh-cn.json
8 -- /im_client/cmps/My97DatePicker/WdatePicker.js?v=1524898256223
7 -- /im_client/cmps/lightbox/dist/images/close.png
......
```
首先分析每一行日志格式是怎样的，目的是找到合适的分割符对每行日志进行分割：

```log
100.109.226.8 - - [29/May/2018:10:58:35 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-"
119.57.115.195 - - [29/May/2018:10:58:58 +0800] "GET /entry/info.json?rand=15275627375696501 HTTP/1.1" 200 63 "http://reocar.udeskt1.com/entry/manage/agent-roles" "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
```

如果以`空格`为分割符的话，可以得到第一列$1是访问人的ip地址，第六列$6是method方法，第七列$7是请求的path。所以我们应该统计$7。我对awk的具体脚本代码规则还没有深入研究，只是根据`{a[$7]++}END{for(i in a){print a[i] " -- " i}}`，可以做出一些理解。$7就是指第七列的数据了， `a[$7]++` 表示对第七列的数据进行累加，但不是第一行到最后一行的累加，a[$7]也许不是一个数据，而是更像lua的table。
`{for(i in a){print a[i] " -- " i}}` 就是for循环递归读取table a的值，i是对应的key，不是普通数组的下标，而是具体的$7第七列的值。所以a应该是类似map dict 这样的hash表的存储结构。而对应的a[i]的每个值，就是前面 `a[$7]++`的时候的累加值。

### 读取Nginx access.log日志，得到访问最多的前10个IP地址及访问次数

很简单，只要选择累加$1第一列就可以

```sh
awk '{a[$1]++}END{for(i in a){print a[i] " -- " i}}' access.log | sort -rn | head -10
```

```
2525 -- 119.57.115.195
1245 -- 100.97.62.40
1242 -- 100.97.62.55
1242 -- 100.97.62.4
1236 -- 100.97.126.60
1235 -- 100.97.62.0
1235 -- 100.97.126.29
1235 -- 100.97.126.26
1231 -- 100.97.126.12
1229 -- 100.97.126.9
```

这样可以发现可以的访问IP，进行黑名单操作。

`awk`应该还有很多可玩的操作，还要继续学习和实践。
