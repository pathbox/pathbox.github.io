---
layout: post
title: CSV operation in Go
date:   2017-01-20 14:41:06
categories: Golang
image: /assets/images/post.jpg
---

##### 简单的读取

```go
package main

import (
	"encoding/csv"
	"fmt"
	"io"
	"os"
)

func main() {
	file, err := os.Open("csv_file.csv") // 打开想要读取的文件
	if err != nil {
		return
	}
	defer file.Close()
	reader := csv.NewReader(file)  // 创建reader 读取器
	for {                          // 循环读取，直到结尾
		record, err := reader.Read()
		if err == io.EOF {
			break
		} else if err != nil {
			fmt.Println("Error:", err)
			return
		}
		fmt.Println(record)
	}
}
```

##### 使用iconv-go解析编码

由于Go只支持UTF-8编码，所以如果有其他编码的数据会产生乱码的情况
iconv-go可以帮我们解决这个问题

```go

package main

import (
	"encoding/csv"
	"fmt"
	iconv "github.com/djimenez/iconv-go"
	"io"
	"os"
	"strings"
)

func main() {
	file, err := os.Open("csv_file.csv") // 打开想要读取的文件
	if err != nil {
		return
	}
	defer file.Close()
	reader := csv.NewReader(file)                      // 创建reader 读取器
	converter, _ := iconv.NewConverter("gbk", "utf-8") // 将读取的数据由gbk编码转为UTF-8编码
	for {                                              // 循环读取，直到结尾
		record, err := reader.Read()
		if err == io.EOF {
			break
		} else if err != nil {
			fmt.Println("Error:", err)
			return
		}
		joinString := strings.Join(record, " ")
		output, _ := converter.ConvertString(joinString)
		fmt.Println(output)
	}
}

```

还可以这样一句代码

```go
output, _ := iconv.ConvertString("Hello World!", "utf-8", "windows-1252")
```

##### 导出生成CSV文件

```go
package main

import (
	"encoding/csv"
	"os"
)

func main() {
	file, err := os.Create("export_csv.csv")
	if err != nil {
		panic(err)
	}
	defer file.Close()
	file.WriteString("\xEF\xBB\xBF") // 写入UTF-8 BOM
	writer := csv.NewWriter(file)
	writer.Write([]string{"id", "姓名", "分数"})
	writer.Write([]string{"1", "张三", "23"})
	writer.Write([]string{"2", "李四", "24"})
	writer.Write([]string{"3", "王五", "25"})
	writer.Write([]string{"4", "赵六", "26"})

	for i := 0; i < 100000; i++ {
		writer.Write([]string{"Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World,Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World"})
	}
	writer.Flush()
}

```

同样的，导出的数据也可以使用iconv-go 进行编码的转换

##### 和Ruby 导出CSV进行一个性能比较
用Linux自带的time命令计算耗时，导出100w相同的数据

Go
```go
package main
import (
	"encoding/csv"
	"os"
)
func main() {
	file, err := os.Create("export_go.csv")
	if err != nil {
		panic(err)
	}
	defer file.Close()
	file.WriteString("\xEF\xBB\xBF") // 写入UTF-8 BOM
	writer := csv.NewWriter(file)
	for i := 0; i < 1000000; i++ {
		writer.Write([]string{"Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World,Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World"})
	}
	writer.Flush()
}

```

Ruby
```ruby
# Ruby 2.3.3
require 'csv'

CSV.open("export_ruby.csv", "wb") do |line|
  1000000.times.each do |i|
    line << ["Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World,Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World", "Hello World"]
  end
end
```

结果:
```
Go: go run export.go  6.71s user 0.42s system 99% cpu 7.183 total

Ruby: ruby export.rb  24.36s user 0.39s system 99% cpu 27.001 total

```

并且观察htop，发现Go所需要的内存也比Ruby的少，当然Go的代码量多了很多行

It make me think the word "trade-off"
