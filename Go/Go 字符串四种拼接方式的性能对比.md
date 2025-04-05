### 简介

使用完整的基准测试代码文件，可以直接运行来比较四种字符串拼接方法的性能。

* `for` 索引 `+=` 的方式

* `for range +=` 的方式 

* `strings.Join` 的方式

* `strings.Builder` 的方式

### 写一个基准测试文件

`echo_bench_test.go`

```go
package main

import (
	"os"
	"strings"
	"testing"
)

func echoAll1() string {
	var s, sep string
	for i := 0; i < len(os.Args); i++ {
		s += sep + os.Args[i]
		sep = " "
	}
	return s
}

func echoAll2() string {
	s, sep := "", ""
	for _, arg := range os.Args[:] {
		s += sep + arg
		sep = " | "
	}
	return s
}

func echoAll3() string {
	return strings.Join(os.Args[:], " , ")
}

// strings.Builder 是 Go 推荐的高效字符串拼接方式，尤其在循环中拼接时，
// 可以减少内存分配。


func echoAll4() string {
	var builder strings.Builder
	for i, arg := range os.Args[:] {
		if i > 0 {
			builder.WriteString(" <> ")
		}
		builder.WriteString(arg)
	}
	return builder.String()
}


// ===== Benchmark Functions =====

func BenchmarkEchoAll1(b *testing.B) {
	// 模拟更长参数列表，避免误差过大
	originalArgs := os.Args
	os.Args = make([]string, 100)
	for i := range os.Args {
		os.Args[i] = "arg"
	}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_ = echoAll1()
	}
	os.Args = originalArgs // 恢复
}

func BenchmarkEchoAll2(b *testing.B) {
	originalArgs := os.Args
	os.Args = make([]string, 100)
	for i := range os.Args {
		os.Args[i] = "arg"
	}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_ = echoAll2()
	}
	os.Args = originalArgs
}

func BenchmarkEchoAll3(b *testing.B) {
	originalArgs := os.Args
	os.Args = make([]string, 100)
	for i := range os.Args {
		os.Args[i] = "arg"
	}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_ = echoAll3()
	}
	os.Args = originalArgs
}

func BenchmarkEchoAll4(b *testing.B) {
	originalArgs := os.Args
	os.Args = make([]string, 100)
	for i := range os.Args {
		os.Args[i] = "arg"
	}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_ = echoAll4()
	}
	os.Args = originalArgs
}
```

### 运行基准测试

```shell
go test -bench=. -benchmem
```

示例输出结果（不同机器会略有不同）：

```yaml
goos: darwin
goarch: amd64
pkg: example
BenchmarkEchoAll1-8     500000     3500 ns/op     120 B/op     5 allocs/op
BenchmarkEchoAll2-8     700000     2400 ns/op     104 B/op     4 allocs/op
BenchmarkEchoAll3-8    1000000     1600 ns/op      80 B/op     2 allocs/op
BenchmarkEchoAll4-8    2000000      800 ns/op      32 B/op     1 allocs/op

PASS
ok  	example	3.456s
```

每一行含义：

|  字段   |  含义   |
| --- | --- |
|  BenchmarkEchoAll1   |  	测试函数名   |
|  -8   |  使用的 CPU 线程数（8 核）   |
|  500000   |  b.N 的值，代表该函数跑了 50 万次   |
|  3500 ns/op   |  每次调用耗时 3500 纳秒   |
|  120 B/op   |  每次操作分配的字节数（字节越少越好）   |
|  5 allocs/op   |  每次操作的内存分配次数（次数越少越好）   |

`Go` 的基准测试自动决定运行次数（`b.N`），直到结果足够稳定。


|  方法   |  ns/op   |  B/op   |  allocs/op   |  说明   |
| --- | --- | --- | --- | --- |
|  EchoAll1   |  3500 ns   |  120 B   |  5   |  += 每次创建新字符串，开销大   |
|  EchoAll2	   |  2400 ns	   | 104 B    |  4   |  range + +=，仍然多次内存分配   |
|  EchoAll3   |  1600 ns   |  80 B   |  2   |  Join 比较高效   |
|  EchoAll4   |  800 ns   |  32 B   |  1   |  strings.Builder 最优   |