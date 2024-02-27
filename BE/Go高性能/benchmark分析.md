# benchmark 基准测试 | Go 语言高性能编程

## 2.4 提升准确性
```
// fib_test.go
package main

import "testing"

func BenchmarkFib(b *testing.B) {
	time.Sleep(time.Second * 3) // 模拟耗时准备任务
	b.ResetTimer() // 重置定时器,处理耗时长引起的问题
	for n := 0; n < b.N; n++ {
		fib(30) // run fib(30) b.N times
	}
}
```
```
$ go test -bench='Fib$' -benchtime=5s -cpu=2,4 -count=3 -cpuprofile=cpu.pprof .
```
测试以Fib开头的性能测试类，测试5s，3轮,分别用2、4核; -benchtime=30x 即执行30次
生成pprof文件形式

```
BenchmarkFib-8               202           5980669 ns/op
```
测试使用了8核，共执行202次，每次 xx ns
## 2.5 内存分配情况
```
// generate_test.go
package main

import (
	"math/rand"
	"testing"
	"time"
)

func generateWithCap(n int) []int {
	rand.Seed(time.Now().UnixNano())
	nums := make([]int, 0, n)
	for i := 0; i < n; i++ {
		nums = append(nums, rand.Int())
	}
	return nums
}

func generate(n int) []int {
	rand.Seed(time.Now().UnixNano())
	nums := make([]int, 0)
	for i := 0; i < n; i++ {
		nums = append(nums, rand.Int())
	}
	return nums
}

func BenchmarkGenerateWithCap(b *testing.B) {
	for n := 0; n < b.N; n++ {
		generateWithCap(1000000)
	}
}

func BenchmarkGenerate(b *testing.B) {
	for n := 0; n < b.N; n++ {
		generate(1000000)
	}
}
```
```
go test -bench='Generate'  -benchmem .


goos: darwin
goarch: amd64
pkg: example
BenchmarkGenerateWithCap-8  43  24335658 ns/op  8003641 B/op    1 allocs/op
BenchmarkGenerate-8         33  30403687 ns/op  45188395 B/op  40 allocs/op
PASS
ok      example 2.121s
```
- 参数可以度量内存分配的次数，不合理的切片容量会导致内存重新分配。
- 使用切片容量设置可以一次性申请所需的内存，比不设置切片容量少耗时20%。


## 3 benchmark 注意事项

### 3.1 StopTimer & StartTimer
```
// sort_test.go
package main

import (
	"math/rand"
	"testing"
	"time"
)

func generateWithCap(n int) []int {
	rand.Seed(time.Now().UnixNano())
	nums := make([]int, 0, n)
	for i := 0; i < n; i++ {
		nums = append(nums, rand.Int())
	}
	return nums
}

func bubbleSort(nums []int) {
	for i := 0; i < len(nums); i++ {
		for j := 1; j < len(nums)-i; j++ {
			if nums[j] < nums[j-1] {
				nums[j], nums[j-1] = nums[j-1], nums[j]
			}
		}
	}
}

func BenchmarkBubbleSort(b *testing.B) {
	for n := 0; n < b.N; n++ {
		b.StopTimer()
		nums := generateWithCap(10000)
		b.StartTimer()
		bubbleSort(nums)
	}
}
```
- 在每次函数调用前后需要准备工作和清理工作时，可以使用StopTimer暂停计时和StartTimer开始计时。


