---
layout: post
title: 最近工作总结(十四)
date:   2018-04-03 20:00:06
categories: Work
image: /assets/images/post.jpg
---

### go tool

go build

+ -o 指定输出的文件名，可以带上路径，例如 go build -o a/b/c
+ -i 安装相应的包，编译+go install
+ -a 更新全部已经是最新的包的，但是对标准包不适用
+ -n 把需要执行的编译命令打印出来，但是不执行，这样就可以很容易的知道底层是如何运行的
+ -p n 指定可以并行可运行的编译数目，默认是CPU数目
+ -race 开启编译的时候自动检测数据竞争的情况，目前只支持64位的机器
+ -v 打印出来我们正在编译的包名
+ -work 打印出来编译时候的临时文件夹名称，并且如果已经存在的话就不要删除
+ -x 打印出来执行的命令，其实就是和-n的结果类似，只是这个会执行
+ -ccflags 'arg list' 传递参数给5c, 6c, 8c 调用
+ -compiler name 指定相应的编译器，gccgo还是gc
+ -gccgoflags 'arg list' 传递参数给gccgo编译连接调用
+ -gcflags 'arg list' 传递参数给5g, 6g, 8g 调用
+ -installsuffix suffix 为了和默认的安装包区别开来，采用这个前缀来重新安装那些依赖的包，-race的时候默认已经是-installsuffix race,大家可以通过-n命令来验证
+ -ldflags 'flag list' 传递参数给5l, 6l, 8l 调用
+  -tags 'tag list' 设置在编译的时候可以适配的那些tag，详细的tag限制参考里面的 [Build Constraints](https://golang.org/pkg/go/build/)

go vet / go tool vet

```
检查代码的正确性

– 错误的printf格式
– 错误的构建tag
– 在闭包中使用错误的range循环变量
– 无用的赋值操作
– 无法到达的代码
– 错误使用mutex
    等等。

使用方法：
    go vet [package|filename]
```

go test -v -cover

用 `github.com/gorilla/mux` 包做例子

进入到包中,执行 `go test -v -cover`

```
=== RUN   TestNativeContextMiddleware
--- PASS: TestNativeContextMiddleware (0.00s)
=== RUN   TestHost
......
PASS
coverage: 85.4% of statements
ok  	github.com/gorilla/mux	0.020s
```

可以看到测试覆盖率为 85.4%

执行 `go test -coverprofile=cover.out`
```
PASS
coverage: 85.4% of statements
ok  	github.com/gorilla/mux	0.016s
```
会生成 cover.out 文件

执行 `go tool cover -func=cover.out`
```
github.com/gorilla/mux/context_native.go:10:	contextGet		100.0%
github.com/gorilla/mux/context_native.go:14:	contextSet		66.7%
github.com/gorilla/mux/context_native.go:22:	contextClear		100.0%
github.com/gorilla/mux/mux.go:17:		NewRouter		100.0%
github.com/gorilla/mux/mux.go:61:		Match			100.0%
github.com/gorilla/mux/mux.go:80:		ServeHTTP		95.5%
github.com/gorilla/mux/mux.go:118:		Get			100.0%
github.com/gorilla/mux/mux.go:124:		GetRoute		0.0%
github.com/gorilla/mux/mux.go:142:		StrictSlash		100.0%
github.com/gorilla/mux/mux.go:155:		SkipClean		100.0%
github.com/gorilla/mux/mux.go:170:		UseEncodedPath		100.0%
github.com/gorilla/mux/mux.go:180:		getNamedRoutes		80.0%
github.com/gorilla/mux/mux.go:192:		getRegexpGroup		100.0%
github.com/gorilla/mux/mux.go:199:		buildVars		100.0%
github.com/gorilla/mux/mux.go:211:		NewRoute		100.0%
github.com/gorilla/mux/mux.go:219:		Handle			100.0%
github.com/gorilla/mux/mux.go:225:		HandleFunc		100.0%
github.com/gorilla/mux/mux.go:232:		Headers			100.0%
github.com/gorilla/mux/mux.go:238:		Host			0.0%
github.com/gorilla/mux/mux.go:244:		MatcherFunc		0.0%
github.com/gorilla/mux/mux.go:250:		Methods			0.0%
github.com/gorilla/mux/mux.go:256:		Path			100.0%
github.com/gorilla/mux/mux.go:262:		PathPrefix		100.0%
github.com/gorilla/mux/mux.go:268:		Queries			0.0%
github.com/gorilla/mux/mux.go:274:		Schemes			0.0%
github.com/gorilla/mux/mux.go:280:		BuildVarsFunc		0.0%
github.com/gorilla/mux/mux.go:287:		Walk			100.0%
github.com/gorilla/mux/mux.go:300:		walk			95.0%
github.com/gorilla/mux/mux.go:352:		Vars			66.7%
github.com/gorilla/mux/mux.go:364:		CurrentRoute		0.0%
github.com/gorilla/mux/mux.go:371:		setVars			100.0%
github.com/gorilla/mux/mux.go:375:		setCurrentRoute		100.0%
github.com/gorilla/mux/mux.go:385:		getPath			80.0%
github.com/gorilla/mux/mux.go:407:		cleanPath		75.0%
github.com/gorilla/mux/mux.go:425:		uniqueVars		100.0%
github.com/gorilla/mux/mux.go:438:		checkPairs		75.0%
github.com/gorilla/mux/mux.go:449:		mapFromPairsToString	85.7%
github.com/gorilla/mux/mux.go:463:		mapFromPairsToRegex	80.0%
github.com/gorilla/mux/mux.go:480:		matchInArray		100.0%
github.com/gorilla/mux/mux.go:490:		matchMapWithString	100.0%
github.com/gorilla/mux/mux.go:518:		matchMapWithRegex	85.7%
github.com/gorilla/mux/regexp.go:27:		newRouteRegexp		92.7%
github.com/gorilla/mux/regexp.go:151:		Match			100.0%
github.com/gorilla/mux/regexp.go:167:		url			66.7%
github.com/gorilla/mux/regexp.go:195:		getURLQuery		85.7%
github.com/gorilla/mux/regexp.go:208:		matchQueryString	100.0%
github.com/gorilla/mux/regexp.go:214:		braceIndices		84.6%
github.com/gorilla/mux/regexp.go:238:		varGroupName		100.0%
github.com/gorilla/mux/regexp.go:254:		setMatch		100.0%
github.com/gorilla/mux/regexp.go:299:		getHost			100.0%
github.com/gorilla/mux/regexp.go:312:		extractVars		100.0%
github.com/gorilla/mux/route.go:44:		SkipClean		0.0%
github.com/gorilla/mux/route.go:49:		Match			92.9%
github.com/gorilla/mux/route.go:81:		GetError		100.0%
github.com/gorilla/mux/route.go:86:		BuildOnly		0.0%
github.com/gorilla/mux/route.go:94:		Handler			100.0%
github.com/gorilla/mux/route.go:102:		HandlerFunc		100.0%
github.com/gorilla/mux/route.go:107:		GetHandler		0.0%
github.com/gorilla/mux/route.go:115:		Name			83.3%
github.com/gorilla/mux/route.go:128:		GetName			100.0%
github.com/gorilla/mux/route.go:142:		addMatcher		100.0%
github.com/gorilla/mux/route.go:150:		addRegexpMatcher	81.5%
github.com/gorilla/mux/route.go:200:		Match			100.0%
github.com/gorilla/mux/route.go:213:		Headers			80.0%
github.com/gorilla/mux/route.go:225:		Match			100.0%
github.com/gorilla/mux/route.go:238:		HeadersRegexp		80.0%
github.com/gorilla/mux/route.go:266:		Host			100.0%
github.com/gorilla/mux/route.go:277:		Match			100.0%
github.com/gorilla/mux/route.go:282:		MatcherFunc		100.0%
github.com/gorilla/mux/route.go:291:		Match			100.0%
github.com/gorilla/mux/route.go:298:		Methods			100.0%
github.com/gorilla/mux/route.go:326:		Path			100.0%
github.com/gorilla/mux/route.go:342:		PathPrefix		100.0%
github.com/gorilla/mux/route.go:366:		Queries			62.5%
github.com/gorilla/mux/route.go:387:		Match			100.0%
github.com/gorilla/mux/route.go:393:		Schemes			100.0%
github.com/gorilla/mux/route.go:408:		BuildVarsFunc		100.0%
github.com/gorilla/mux/route.go:427:		Subrouter		100.0%
github.com/gorilla/mux/route.go:468:		URL			68.8%
github.com/gorilla/mux/route.go:502:		URLHost			63.6%
github.com/gorilla/mux/route.go:526:		URLPath			63.6%
github.com/gorilla/mux/route.go:551:		GetPathTemplate		80.0%
github.com/gorilla/mux/route.go:566:		GetHostTemplate		80.0%
github.com/gorilla/mux/route.go:578:		prepareVars		75.0%
github.com/gorilla/mux/route.go:586:		buildVars		100.0%
github.com/gorilla/mux/route.go:608:		getNamedRoutes		66.7%
github.com/gorilla/mux/route.go:617:		getRegexpGroup		100.0%
total:						(statements)		85.4%
```

列出了每个文件具体的方法的覆盖率

执行 `go tool cover -html=cover.out`(会在tmp中临时生成html文件) or `go tool cover -html=cover.out -o ~/mux_cover.html`(指定生成指定的html文件)

在浏览器打开这个html文件,会有 `not tracked not covered covered` 三个标签选项,以及可以选择具体的文件,查看代码测试的覆盖率和覆盖区域,不同的代码测试覆盖情况是用不同的颜色显示出来(bingo~)

```
https://golang.org/doc/cmd
http://wiki.jikexueyuan.com/project/go-command-tutorial/0.12.html
https://studygolang.com/articles/5844
https://tonybai.com/2014/10/22/golang-testing-techniques/
```

##### 最简单的理解正向代理和反向代理
+ 正向代理是在客户端的，代理客户端向服务端发起请求，正向代理的IP往往和客户端IP在同一个网络 （shadowsock）
+ 反向代理是在服务端的，代理服务端接受客户端的请求，反向代理的IP往往和服务器IP在同一个网络 （Nginx LVS）
