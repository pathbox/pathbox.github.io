---
layout: post
title: 通配符匹配选择glob还是正则
date:  2020-10-29 19:15:06
categories: Nice
image: /assets/images/post.jpg


---



https://tldp.org/LDP/GNU-Linux-Tools-Summary/html/x11655.htm



最近遇到需要实现一个匹配相关的逻辑代码，需要能够实现前缀匹配、中间包含和后缀匹配三种情况。具体例子：

```
*abc/foo 
abc*/foo
abc/foo*
```

*表示任意多个任意字符，也就是可以匹配0个也可以匹配多个，可以在前、中、后三个位置。

快速实现第一版：找到`*`的位置，以此将字符串分割为多个部分，才进行对应值是否相等。这种方式很简单，在业务中可行，但是扩展性不行，在我们的业务中难以封装通用的模块逻辑，于是寻找新的方法。关于匹配问题，自然会想到的是`正则匹配`。使用正则匹配可以满足需要的功能，但是正则匹配的性能是需要考虑的问题。由于该业务会有大量的匹配操作，正则匹配可能导致性能瓶颈。

有没有比正则匹配性能更好的匹配方式呢？有的，就是`globbing patterns`。

`globbing patterns`是一种标准的通配符匹配方式。其实在Linux系统中你会经常遇到，比如删除操作：rm fo*，这样就把当前目录下fo为前缀的所有文件都删除。

`globbing patterns`能够满足需求，那它的性能和正则匹配相比如何呢？

`globbing patterns`在golang中的path/filepath 这个包能实现相应的功能

```go
package main

import (
	"fmt"
	"path/filepath"
)

func main() {
	var b bool
	var err error
	t1 := "xxx:xxx:name"
	p1 := "xxx:xxx:*"

	b, err = filepath.Match(p1, t1)
	fmt.Println(b, err)
}
```

在github上查找到了第三方的包[glob](https://github.com/gobwas/glob)。该包的匹配语法符合  [standard wildcards](http://tldp.org/LDP/GNU-Linux-Tools-Summary/html/x11655.htm)。

将 正则表达式匹配、`path/filepath` 和[glob](https://github.com/gobwas/glob)进行benchmark测试。

```go
package benchmark

import (
	"path/filepath"
	"regexp"
	"testing"

	"github.com/gobwas/glob"
)

const (
	pattern_prefix                 = "abc*"
	regexp_prefix                  = `^abc.*$`
	pattern_suffix                 = "*def"
	regexp_suffix                  = `^.*def$`
	pattern_prefix_suffix          = "ab*ef"
	regexp_prefix_suffix           = `^ab.*ef$`
	fixture_prefix_suffix_match    = "abcdef"
	fixture_prefix_suffix_mismatch = "af"
)



func BenchmarkPrefixRegexpMatch(b *testing.B) {
	m := regexp.MustCompile("^aaa:bbb:.*:cccccc:myphotos/hangzhou/2015/.*$")
	f := []byte("aaa:bbb:b:cccccc:myphotos/hangzhou/2015/aaa")

	for i := 0; i < b.N; i++ {
		_ = m.Match(f)
	}
}

func BenchmarkPrefixFilepathMatch(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_, _ = filepath.Match("aaa:bbb:*:cccccc:myphotos/hangzhou/2015/*", "aaa:bbb:b:cccccc:myphotos/hangzhou/2015/aaa")
	}
}

func BenchmarkPrefixGlobMatch(b *testing.B) {
	var g glob.Glob

	// create simple glob
	g = glob.MustCompile("aaa:bbb:*:cccccc:myphotos/hangzhou/2015/*")

	for i := 0; i < b.N; i++ {
		g.Match("aaa:bbb:b:cccccc:myphotos/hangzhou/2015/aaa") // true
	}
}


func BenchmarkSuffixRegexpMatch(b *testing.B) {
	m := regexp.MustCompile("^.*:aaa:abcabcabc")
	f := []byte("123:aaa:abcabcabc")

	for i := 0; i < b.N; i++ {
		_ = m.Match(f)
	}
}

func BenchmarkSuffixFilepathMatch(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_, _ = filepath.Match("*:aaa:abcabcabc", "123:aaa:abcabcabc")
	}
}

func BenchmarkSuffixGlobMatch(b *testing.B) {
	var g glob.Glob

	// create simple glob
	g = glob.MustCompile("*:aaa:abcabcabc")

	for i := 0; i < b.N; i++ {
		g.Match("123:aaa:abcabcabc") // true
	}
}


func BenchmarkPrefixSuffixRegexpMatch(b *testing.B) {
	m := regexp.MustCompile(regexp_prefix_suffix)
	f := []byte(fixture_prefix_suffix_match)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_ = m.Match(f)
	}
}

func BenchmarkPrefixSuffixFilepathMatch(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_, _ = filepath.Match(pattern_prefix_suffix, fixture_prefix_suffix_match)
	}
}

func BenchmarkPrefixSuffixGlobMatch(b *testing.B) {
	var g glob.Glob

	// create simple glob
	g = glob.MustCompile(pattern_prefix_suffix)

	for i := 0; i < b.N; i++ {
		g.Match(fixture_prefix_suffix_match) // true
	}
}

// go test -bench=. benchmark_test.go

/*
goos: darwin
goarch: amd64
BenchmarkPrefixRegexpMatch-4              695192              2333 ns/op
BenchmarkPrefixFilepathMatch-4           3774104               404 ns/op
BenchmarkPrefixGlobMatch-4              20142326                71.3 ns/op
BenchmarkSuffixRegexpMatch-4             1470373               713 ns/op
BenchmarkSuffixFilepathMatch-4          10244230               103 ns/op
BenchmarkSuffixGlobMatch-4              147599737                7.88 ns/op
BenchmarkPrefixSuffixRegexpMatch-4       4915987               228 ns/op
BenchmarkPrefixSuffixFilepathMatch-4    19263058                61.9 ns/op
BenchmarkPrefixSuffixGlobMatch-4        90101554                13.0 ns/op
PASS
ok      command-line-arguments  13.843s
*/

```

测试结果：

1. [glob](https://github.com/gobwas/glob)

2. path/filepath

3. regexp

可以看到[glob](https://github.com/gobwas/glob) 的性能远好于 `path/filepath` 和正则。所以决定使用[glob](https://github.com/gobwas/glob)。



在匹配功能中，正则匹配是强而全的。如果你的具体业务通配符匹配就能满足，请选择通配符，它具有更好的匹配性能



参考链接:

https://github.com/gobwas/glob

https://tldp.org/LDP/GNU-Linux-Tools-Summary/html/x11655.htm