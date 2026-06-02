### 简介

`make` 是 Go 里的内置函数，专门用来创建并初始化三种类型：

* slice
* map
* channel

语法大概长这样：

```go
make(Type, args...)
```

常见写法：

```go
s := make([]int, 0, 10)
m := make(map[string]int)
ch := make(chan int, 3)
```

一句话概括：

```text
make 用来把 slice、map、channel 初始化成可用状态。
```

这点和 `new` 不一样。

`new` 返回指针，得到的是某个类型的零值指针。

`make` 返回类型本身，得到的是已经初始化好的 slice、map、channel。

### 为什么需要 make

先看三个变量声明：

```go
var s []int
var m map[string]int
var ch chan int
```

它们都是零值状态：

```go
package main

import "fmt"

func main() {
	var s []int
	var m map[string]int
	var ch chan int

	fmt.Println(s == nil)
	fmt.Println(m == nil)
	fmt.Println(ch == nil)
}
```

输出：

```text
true
true
true
```

nil slice、nil map、nil channel 都不是完全不能用，但限制很多：

* nil slice 可以 `append`，但不能按下标赋值
* nil map 可以读取，但不能写入
* nil channel 发送和接收都会一直阻塞

`make` 的价值就在这里：

```text
创建底层数据结构，让 slice、map、channel 进入可正常使用的状态。
```

### make 的返回值不是指针

`make` 返回的是类型本身，不是指针。

```go
package main

import "fmt"

func main() {
	s := make([]int, 3)
	m := make(map[string]int)
	ch := make(chan int)

	fmt.Printf("s type: %T\n", s)
	fmt.Printf("m type: %T\n", m)
	fmt.Printf("ch type: %T\n", ch)
}
```

输出：

```text
s type: []int
m type: map[string]int
ch type: chan int
```

没有 `*[]int`、`*map[string]int`、`*chan int`。

这也是 `make` 和 `new` 最直观的区别。

### make 和 new 的区别

| 对比项 | `make` | `new` |
| --- | --- | --- |
| 适用类型 | 只能用于 slice、map、channel | 任意类型 |
| 返回值 | 类型本身 | 指针 |
| 返回类型 | `T` | `*T` |
| 初始化效果 | 初始化底层结构，可直接使用 | 分配零值内存 |
| 常见用途 | 创建集合和通道 | 创建零值指针 |

示例：

```go
package main

import "fmt"

func main() {
	s1 := make([]int, 0)
	s2 := new([]int)

	fmt.Printf("s1 type: %T, s1 == nil: %v\n", s1, s1 == nil)
	fmt.Printf("s2 type: %T, *s2 == nil: %v\n", s2, *s2 == nil)
}
```

输出：

```text
s1 type: []int, s1 == nil: false
s2 type: *[]int, *s2 == nil: true
```

`make([]int, 0)` 得到的是一个已经初始化的空切片。

`new([]int)` 得到的是一个指向 nil 切片的指针。

普通业务代码里，创建 slice、map、channel 时优先考虑 `make`。

### make 创建切片

切片的 `make` 有两种常见写法：

```go
make([]T, len)
make([]T, len, cap)
```

`len` 表示当前长度，`cap` 表示容量。

如果不写 `cap`，容量默认等于长度。

```go
package main

import "fmt"

func main() {
	a := make([]int, 3)
	b := make([]int, 3, 5)

	fmt.Println(a, len(a), cap(a))
	fmt.Println(b, len(b), cap(b))
}
```

输出：

```text
[0 0 0] 3 3
[0 0 0] 3 5
```

`make([]int, 3)` 的意思是：

```text
创建一个长度为 3 的 int 切片，元素都是 int 零值 0。
```

`make([]int, 3, 5)` 的意思是：

```text
创建一个长度为 3、容量为 5 的 int 切片。
```

### 切片长度和容量怎么选

切片最容易写错的地方，就是混淆长度和容量。

#### 需要直接按下标赋值

如果要直接写 `s[i] = value`，长度必须够。

```go
package main

import "fmt"

func main() {
	s := make([]int, 3)

	s[0] = 10
	s[1] = 20
	s[2] = 30

	fmt.Println(s)
}
```

输出：

```text
[10 20 30]
```

#### 需要不断 append

如果主要靠 `append` 追加，通常把长度设为 `0`，容量设为预计数量。

```go
package main

import "fmt"

func main() {
	s := make([]int, 0, 3)

	s = append(s, 10)
	s = append(s, 20)
	s = append(s, 30)

	fmt.Println(s)
	fmt.Println("len:", len(s), "cap:", cap(s))
}
```

输出：

```text
[10 20 30]
len: 3 cap: 3
```

这种写法的好处是：先预留空间，再按实际元素数量追加。

### 切片常见坑：make([]int, 3) 后又 append

看一段常见错误：

```go
package main

import "fmt"

func main() {
	s := make([]int, 3)

	s = append(s, 10)
	s = append(s, 20)

	fmt.Println(s)
}
```

输出：

```text
[0 0 0 10 20]
```

这不是 Go 出错，而是写法含义不对。

`make([]int, 3)` 已经创建了 3 个元素：

```text
[0 0 0]
```

后面的 `append` 是在这 3 个元素后面继续追加。

如果想得到 `[10 20]`，应该写：

```go
s := make([]int, 0, 3)
s = append(s, 10)
s = append(s, 20)
```

### 切片扩容演示

容量不够时，`append` 会触发扩容。

```go
package main

import "fmt"

func main() {
	s := make([]int, 0, 2)

	for i := 1; i <= 5; i++ {
		s = append(s, i)
		fmt.Printf("append %d -> len=%d cap=%d value=%v\n", i, len(s), cap(s), s)
	}
}
```

输出类似：

```text
append 1 -> len=1 cap=2 value=[1]
append 2 -> len=2 cap=2 value=[1 2]
append 3 -> len=3 cap=4 value=[1 2 3]
append 4 -> len=4 cap=4 value=[1 2 3 4]
append 5 -> len=5 cap=8 value=[1 2 3 4 5]
```

扩容规则会受 Go 版本和运行时策略影响，不适合死记具体倍数。

需要记住的是：

```text
容量不够时，append 会申请更大的底层数组，并把旧数据搬过去。
```

所以已知大概数量时，提前设置容量可以减少扩容和数据拷贝。

### 实战 Demo：预分配切片收集 ID

假设有一批用户数据，需要提取所有有效用户 ID。

```go
package main

import "fmt"

type User struct {
	ID     int
	Name   string
	Active bool
}

func collectActiveIDs(users []User) []int {
	ids := make([]int, 0, len(users))

	for _, user := range users {
		if user.Active {
			ids = append(ids, user.ID)
		}
	}

	return ids
}

func main() {
	users := []User{
		{ID: 1, Name: "张三", Active: true},
		{ID: 2, Name: "李四", Active: false},
		{ID: 3, Name: "王五", Active: true},
	}

	ids := collectActiveIDs(users)

	fmt.Println(ids)
}
```

输出：

```text
[1 3]
```

这里使用：

```go
ids := make([]int, 0, len(users))
```

原因是结果最多不会超过 `len(users)`，提前预留容量比较合适。

### make 创建 map

map 的 `make` 常见写法：

```go
make(map[K]V)
make(map[K]V, hint)
```

`hint` 表示预估元素数量。

示例：

```go
package main

import "fmt"

func main() {
	scores := make(map[string]int)

	scores["Go"] = 100
	scores["Java"] = 90
	scores["Python"] = 95

	fmt.Println(scores)
	fmt.Println(scores["Go"])
}
```

输出：

```text
map[Go:100 Java:90 Python:95]
100
```

map 必须初始化后才能写入。

### nil map 的坑

只声明 map，不会自动创建底层哈希表。

```go
package main

func main() {
	var scores map[string]int

	scores["Go"] = 100
}
```

运行会报错：

```text
panic: assignment to entry in nil map
```

正确写法：

```go
scores := make(map[string]int)
scores["Go"] = 100
```

nil map 可以读取：

```go
package main

import "fmt"

func main() {
	var scores map[string]int

	fmt.Println(scores["Go"])
	fmt.Println(scores == nil)
}
```

输出：

```text
0
true
```

读取不存在的 key 会返回 value 类型的零值。

但写入 nil map 会 panic。

### map 容量 hint 是什么

`make(map[K]V, hint)` 里的 `hint` 不是固定容量，也不是最大容量。

它只是告诉运行时：

```text
大概要放这么多元素，可以提前准备一下。
```

map 后面仍然可以继续增加元素。

```go
package main

import "fmt"

func main() {
	m := make(map[int]string, 2)

	m[1] = "one"
	m[2] = "two"
	m[3] = "three"
	m[4] = "four"

	fmt.Println(m)
}
```

输出：

```text
map[1:one 2:two 3:three 4:four]
```

`hint` 不是限制，只是优化提示。

### 实战 Demo：用 map 做单词计数

map 很适合做计数器。

```go
package main

import (
	"fmt"
	"strings"
)

func countWords(text string) map[string]int {
	words := strings.Fields(text)
	counts := make(map[string]int, len(words))

	for _, word := range words {
		counts[word]++
	}

	return counts
}

func main() {
	text := "go make go map make slice go"
	counts := countWords(text)

	fmt.Println(counts)
}
```

输出：

```text
map[go:3 make:2 map:1 slice:1]
```

这里使用：

```go
counts := make(map[string]int, len(words))
```

因为单词种类最多不会超过单词总数。

### 实战 Demo：用 map 做索引

项目里经常需要把列表转成按 ID 查询的索引。

```go
package main

import "fmt"

type Product struct {
	ID    int
	Name  string
	Price int
}

func buildProductIndex(products []Product) map[int]Product {
	index := make(map[int]Product, len(products))

	for _, product := range products {
		index[product.ID] = product
	}

	return index
}

func main() {
	products := []Product{
		{ID: 101, Name: "键盘", Price: 199},
		{ID: 102, Name: "鼠标", Price: 99},
		{ID: 103, Name: "显示器", Price: 899},
	}

	index := buildProductIndex(products)

	fmt.Println(index[102])
}
```

输出：

```text
{102 鼠标 99}
```

这种写法比每次遍历列表查找更直接。

### make 创建 channel

channel 用于 goroutine 之间通信，也需要用 `make` 初始化。

常见写法：

```go
make(chan T)
make(chan T, buffer)
```

`make(chan T)` 创建无缓冲 channel。

`make(chan T, buffer)` 创建有缓冲 channel。

### 无缓冲 channel

无缓冲 channel 的特点是：

```text
发送和接收必须同时准备好。
```

示例：

```go
package main

import "fmt"

func main() {
	ch := make(chan string)

	go func() {
		ch <- "hello"
	}()

	msg := <-ch
	fmt.Println(msg)
}
```

输出：

```text
hello
```

如果没有 goroutine 接收，直接发送会阻塞。

```go
ch := make(chan string)
ch <- "hello"
```

这段代码会一直卡住。

### 有缓冲 channel

有缓冲 channel 可以先放入一定数量的数据。

```go
package main

import "fmt"

func main() {
	ch := make(chan string, 2)

	ch <- "one"
	ch <- "two"

	fmt.Println(<-ch)
	fmt.Println(<-ch)
}
```

输出：

```text
one
two
```

缓冲区没满时，发送不会阻塞。

缓冲区空时，接收会阻塞。

缓冲区满时，继续发送会阻塞。

### nil channel 的坑

只声明 channel，不会创建通信通道。

```go
var ch chan int
```

此时 `ch == nil`。

nil channel 发送和接收都会一直阻塞：

```go
package main

func main() {
	var ch chan int

	ch <- 1
}
```

运行后会卡住，最后可能出现：

```text
fatal error: all goroutines are asleep - deadlock!
```

正确写法：

```go
ch := make(chan int)
```

### 实战 Demo：用 channel 做任务队列

下面是一个简单的 worker demo。

```go
package main

import (
	"fmt"
	"sync"
)

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
	defer wg.Done()

	for job := range jobs {
		fmt.Printf("worker %d handle job %d\n", id, job)
		results <- job * 2
	}
}

func main() {
	const jobCount = 6
	const workerCount = 2

	jobs := make(chan int, jobCount)
	results := make(chan int, jobCount)

	var wg sync.WaitGroup

	for i := 1; i <= workerCount; i++ {
		wg.Add(1)
		go worker(i, jobs, results, &wg)
	}

	for i := 1; i <= jobCount; i++ {
		jobs <- i
	}
	close(jobs)

	wg.Wait()
	close(results)

	for result := range results {
		fmt.Println("result:", result)
	}
}
```

输出类似：

```text
worker 1 handle job 1
worker 2 handle job 2
worker 1 handle job 3
worker 2 handle job 4
worker 1 handle job 5
worker 2 handle job 6
result: 2
result: 4
result: 6
result: 8
result: 10
result: 12
```

实际输出顺序可能变化，因为 goroutine 调度顺序不固定。

这里的两个 channel 都用了缓冲：

```go
jobs := make(chan int, jobCount)
results := make(chan int, jobCount)
```

这样可以先把任务放进队列，也能避免结果写入时因为没人立刻接收而卡住。

### 实战 Demo：二维切片

二维切片经常用于表格、棋盘、矩阵。

```go
package main

import "fmt"

func newMatrix(rows, cols int) [][]int {
	matrix := make([][]int, rows)

	for i := range matrix {
		matrix[i] = make([]int, cols)
	}

	return matrix
}

func main() {
	matrix := newMatrix(3, 4)

	matrix[1][2] = 99

	for _, row := range matrix {
		fmt.Println(row)
	}
}
```

输出：

```text
[0 0 0 0]
[0 0 99 0]
[0 0 0 0]
```

注意这里需要两层 `make`：

```go
matrix := make([][]int, rows)
matrix[i] = make([]int, cols)
```

第一层创建“行切片”，第二层给每一行创建真正的列数据。

如果只写第一层：

```go
matrix := make([][]int, rows)
matrix[0][0] = 1
```

会 panic，因为 `matrix[0]` 还是 nil slice。

### 实战 Demo：map 里放切片

有时需要按分类收集数据，例如按城市分组用户名。

```go
package main

import "fmt"

type User struct {
	Name string
	City string
}

func groupByCity(users []User) map[string][]string {
	groups := make(map[string][]string)

	for _, user := range users {
		groups[user.City] = append(groups[user.City], user.Name)
	}

	return groups
}

func main() {
	users := []User{
		{Name: "张三", City: "上海"},
		{Name: "李四", City: "北京"},
		{Name: "王五", City: "上海"},
	}

	groups := groupByCity(users)

	fmt.Println(groups)
}
```

输出：

```text
map[上海:[张三 王五] 北京:[李四]]
```

这里有一个细节：

```go
groups[user.City] = append(groups[user.City], user.Name)
```

如果某个城市还不存在，`groups[user.City]` 返回的是 nil slice。nil slice 可以 `append`，所以这段代码可以正常工作。

这也是 Go 里很常见的分组写法。

### make 的限制

`make` 只能用于 slice、map、channel。

下面这些写法都会编译失败：

```go
make(int)
make(string)
make(User)
make([3]int)
```

基本类型、数组、结构体不使用 `make`。

常见写法如下：

```go
i := 0
name := ""
arr := [3]int{}
u := &User{}
```

### make 参数错误

切片的长度不能大于容量。

```go
s := make([]int, 5, 3)
```

这行代码会编译失败：

```text
len larger than cap in make([]int)
```

长度和容量也不能是负数。

```go
n := -1
s := make([]int, n)
```

运行时会 panic：

```text
panic: runtime error: makeslice: len out of range
```

实际项目里，如果长度来自外部输入，需要先校验。

```go
func createBuffer(size int) []byte {
	if size < 0 {
		size = 0
	}

	return make([]byte, size)
}
```

### 什么时候适合用 make

`make` 适合这些场景：

* 创建可写入的 map
* 创建用于收发数据的 channel
* 创建指定长度的 slice
* 创建需要频繁 append 的 slice，并预分配容量
* 已知大概元素数量，想减少扩容
* 创建二维切片、分组 map、任务队列等复合结构

示例：

```go
ids := make([]int, 0, len(users))
index := make(map[int]User, len(users))
jobs := make(chan Job, 100)
```

### 什么时候不用 make

#### 切片字面量更直观

如果元素已经确定，直接用字面量：

```go
names := []string{"张三", "李四", "王五"}
```

没必要写：

```go
names := make([]string, 0, 3)
names = append(names, "张三", "李四", "王五")
```

#### nil slice 本身就够用

有些函数只需要返回一个可 append 的切片，nil slice 也可以。

```go
func filterPositive(nums []int) []int {
	var result []int

	for _, n := range nums {
		if n > 0 {
			result = append(result, n)
		}
	}

	return result
}
```

如果很在意返回空结果时是 `[]` 还是 `null`，例如 JSON 输出，那就需要根据场景选择。

#### 结构体用字面量或构造函数

结构体不使用 `make`。

```go
u := &User{ID: 1, Name: "张三"}
```

如果有业务默认值，使用工厂函数：

```go
func NewUser(name string) *User {
	return &User{
		Name: name,
	}
}
```

### 常见问题

#### make([]int, 0) 和 []int{} 有什么区别

两者都是非 nil 空切片。

```go
s1 := make([]int, 0)
s2 := []int{}
```

一般都可以用。

如果需要指定容量，用 `make` 更合适：

```go
s := make([]int, 0, 100)
```

#### make([]int, 3) 和 make([]int, 0, 3) 怎么选

需要按下标赋值，选：

```go
s := make([]int, 3)
s[0] = 10
```

需要 `append` 追加，选：

```go
s := make([]int, 0, 3)
s = append(s, 10)
```

#### map 的 hint 会限制元素数量吗

不会。

```go
m := make(map[string]int, 1)
m["a"] = 1
m["b"] = 2
m["c"] = 3
```

可以继续写入。

`hint` 只是预估容量，不是上限。

#### channel 缓冲越大越好吗

不是。

缓冲过小，生产者和消费者容易互相等待。

缓冲过大，可能积压太多任务，内存占用变高，还会掩盖消费速度太慢的问题。

缓冲大小应该根据业务吞吐、处理速度、峰值流量来定。

### 总结

`make` 的核心可以压缩成三句话：

```text
make 只能用于 slice、map、channel。
make 返回类型本身，不返回指针。
make 会初始化底层结构，让对象进入可用状态。
```

日常选择可以按下面这张表判断：

| 场景 | 推荐写法 |
| --- | --- |
| 创建可 append 的切片并预分配容量 | `make([]T, 0, cap)` |
| 创建固定长度切片并按下标赋值 | `make([]T, len)` |
| 创建可写入 map | `make(map[K]V)` |
| 已知 map 大概数量 | `make(map[K]V, hint)` |
| 创建无缓冲 channel | `make(chan T)` |
| 创建有缓冲 channel | `make(chan T, size)` |
| 创建结构体 | `&T{}` 或 `NewT()` |
| 创建零值指针 | `new(T)` |

`make` 本身不难，关键是记住它只服务三种类型：切片、映射、通道。切片看清 `len` 和 `cap`，map 初始化后再写入，channel 初始化后再收发，大多数坑就能避开。
