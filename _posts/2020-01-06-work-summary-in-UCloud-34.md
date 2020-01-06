---
layout: post
title: 最近工作总结(34)
date:  2020-01-06 11:25:06
categories: Work
image: /assets/images/post.jpg
---

### 用struct{}{}来表示空或无用值能节省空间,struct{}{}的size是0

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	am := make(map[string]bool)
	bm := make(map[string]struct{})
	am["1"] = true
	am["2"] = true
	am["3"] = true
	am["4"] = true
	bm["1"] = struct{}{}
	bm["2"] = struct{}{}
	bm["3"] = struct{}{}

	fmt.Printf("bool size: %d\n", unsafe.Sizeof(true))
	fmt.Printf("struct size: %d\n", unsafe.Sizeof(struct{}{}))
	fmt.Printf("am size: %d\n", unsafe.Sizeof(am))
	fmt.Printf("bm size: %d\n", unsafe.Sizeof(bm))
}

/*
bool size: 1
struct size: 0
am size: 4
bm size: 4
*/
```
