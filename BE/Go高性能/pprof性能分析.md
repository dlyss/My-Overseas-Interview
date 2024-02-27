# 生成pprof
使用一个易用性更强的库 pkg/profile 来采集性能数据，pkg/profile 封装了 runtime/pprof 的接口，使用起来更简单。
以下是内存
```
package main

import (
	"github.com/pkg/profile"
	"math/rand"
)

const letterBytes = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

func randomString(n int) string {
	b := make([]byte, n)
	for i := range b {
		b[i] = letterBytes[rand.Intn(len(letterBytes))]
	}
	return string(b)
}

func concat(n int) string {
	s := ""
	for i := 0; i < n; i++ {
		s += randomString(n)
	}
	return s
}

func main() {
	defer profile.Start(profile.MemProfile, profile.MemProfileRate(1)).Stop()
	concat(100)
}

```
比如我们想度量 concat() 的 CPU 性能数据，只需要一行代码即可生成 profile 文件。
```
import (
	"github.com/pkg/profile"
)

func main() {
	defer profile.Start().Stop()
	concat(100)
}
```
```
$ go run main.go
2020/11/22 18:38:29 profile: cpu profiling enabled, /tmp/profile068616584/cpu.pprof
2020/11/22 18:39:12 profile: cpu profiling disabled, /tmp/profile068616584/cpu.pprof

```
# 分析数据
```
go tool pprof -http=:9999 /tmp/profile215959616/mem.pprof
```
分析结论：concat 函数仅仅是将 randomString 生成的字符串拼接起来，消耗的内存应该和 randomString 一致，但怎么会产生 20 倍的差异呢？
这和 Go 语言字符串内存分配的方式有关系。字符串是不可变的，因为将两个字符串拼接时，相当于是产生新的字符串，如果当前的空间不足以容纳新的字符串，
则会申请更大的空间，将新字符串完全拷贝过去，这消耗了 2 倍的内存空间。
使用 strings.Builder 替换 + 进行字符串拼接，将有效地降低内存消耗
```
func concat(n int) string {
	var s strings.Builder
	for i := 0; i < n; i++ {
		s.WriteString(randomString(n))
	}
	return s.String()
}
```
