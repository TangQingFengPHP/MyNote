### 简介

`Go` 中的 切片（`slice`） 是 `Go` 最强大、最常用的数据结构之一。它是对数组的轻量封装，比数组更灵活，几乎所有的集合处理都用切片来完成。

### 什么是切片（slice）

切片是一个拥有 长度（`len`）和容量（`cap`） 的 动态数组视图。底层是一个数组，但可以动态扩容、共享数组。

```go
var s []int // nil slice，len=0，cap=0
fmt.Println(s, len(s), cap(s)) // [] 0 0

s := []int{} // 空切片，已初始化但无元素
```

### 切片的创建

#### 使用字面量

```go
s := []int{1, 2, 3}
```

#### 从数组或切片切割而来

```go
arr := [5]int{0, 1, 2, 3, 4}
s := arr[1:4] // 包含索引1~3：1, 2, 3
```

#### 使用 make 创建

```go
s := make([]int, 3)           // len=3, cap=3，默认值0
s := make([]int, 3, 5)        // len=3, cap=5
```

#### 切片表达式

* 简单表达式：`s[low:high]`，左闭右开区间。

* 完整表达式：`s[low:high:max]`，指定容量为 `max-low`，用于限制后续操作的容量。

### 切片的底层结构

```go
type slice struct {
    ptr *T   // 底层数组指针
    len int  // 当前长度
    cap int  // 容量（底层数组的最大长度）
}
```

> 切片只是一个“视图窗口”，多个切片可能共享同一数组。

### 切片的切割和操作

```go
s := []int{10, 20, 30, 40, 50}
s1 := s[1:4]       // [20 30 40]
s2 := s[:3]        // [10 20 30]
s3 := s[2:]        // [30 40 50]
s4 := s[:]         // 全部复制
```

### 常见操作

#### 删除元素

```go
s := []int{1, 2, 3, 4}
s = append(s[:2], s[3:]...) // 删除索引2 → [1, 2, 4]
```

#### 清空切片

```go
s = s[:0]   // 长度置0，保留容量
s = nil     // 释放底层数组
```

#### 切片的遍历

可以使用 `for` 循环或者 `for...range` 循环来遍历切片。

```go
package main

import "fmt"

func main() {
    slice := []int{1, 2, 3, 4, 5}

    // 使用 for 循环遍历
    fmt.Println("Using for loop:")
    for i := 0; i < len(slice); i++ {
        fmt.Println(slice[i])
    }

    // 使用 for...range 循环遍历
    fmt.Println("Using for...range loop:")
    for index, value := range slice {
        fmt.Printf("Index: %d, Value: %d\n", index, value)
    }
}
```

### 高级技巧

#### 预分配容量：减少扩容次数，提升性能。

```go
s := make([]int, 0, 1000) // 预分配容量
```

#### 避免内存泄漏：截取大切片中的小部分后，复制到新切片以释放原数组。

```go
bigSlice := make([]int, 1000000)
smallPart := make([]int, 10)
copy(smallPart, bigSlice[:10])
```

#### 字符串处理

字符串可视为 `[]` byte切片，但需注意中文字符需转为 `[]rune`

```go
str := "你好"
runes := []rune(str)    // 正确处理中文字符
```

### 切片的容量扩展与底层数组共享

```go
s := []int{1, 2, 3}
s1 := s[:2]   // [1 2]
s2 := append(s1, 99) // 可能修改 s 底层内容

fmt.Println(s)
```

> 如果 `append` 后容量没超出原数组，`s1` 与 `s` 仍然共享底层数组。

### append、copy 的用法

#### append 用于追加元素

```go
s := []int{1, 2}
s = append(s, 3, 4)     // [1 2 3 4]
slice = append(slice, anotherSlice...) // 追加另一个切片（使用...展开）
```

#### copy 用于切片复制

```go
a := []int{1, 2, 3}
b := make([]int, 2)
copy(b, a)   // 只复制前2个元素
```

### 函数传参时的切片特性

切片本质是引用类型，传参是复制切片结构体值，但底层数组是共享的。

```go
func modify(s []int) {
	s[0] = 999
}
s := []int{1, 2, 3}
modify(s)
fmt.Println(s) // [999 2 3]
```

### 多维切片（二维数组）

```go
matrix := [][]int{
	{1, 2, 3},
	{4, 5, 6},
}
matrix[1][2] = 9
```

> 多维切片是切片的切片，并不是严格的二维数组结构。

### 切片常见问题与陷阱

#### 共享底层数组导致修改影响原切片

```go
s := []int{1, 2, 3, 4}
s1 := s[:2]
s1[0] = 999
fmt.Println(s) // [999 2 3 4]
```

#### 超过容量自动分配新数组

```go
s := make([]int, 2, 3)
s = append(s, 4) // 使用原数组
s = append(s, 5) // 超过cap，创建新数组
```

#### 判断空切片

使用 `len(s) == 0` 而非 `s == nil`，因空切片可能非 `nil`

### 推荐使用方式总结

|  场景   |  推荐写法   |
| --- | --- |
|  初始化切片   |  `make([]T, len, cap) 或 []T{...}`   |
|  安全扩容   |  `s = append(s, x...)`   |
|  不修改原切片   |  `new := append([]T(nil), old...)`   |
|  复制切片   |  `copy(dst, src)`   |
|  清空切片   |  `s = s[:0] 或 var s []T`   |

### growslice 实现解析（简化版）

```go
func growslice(et *_type, old slice, cap int) slice {
    // 参数解释：
    // et  : 元素类型指针
    // old : 原 slice 数据结构（包括 len, cap, ptr）
    // cap : 需要的新容量

    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        if old.len < 1024 {
            newcap = doublecap
        } else {
            // 对于大 slice，增幅是 1.25 倍
            for newcap < cap {
                newcap += newcap / 4
            }
        }
    }

    // 分配新数组（unsafe.Pointer）
    newptr := mallocgc(et.size * uintptr(newcap), et, true)

    // 拷贝原数据到新内存
    memmove(newptr, old.array, et.size * uintptr(old.len))

    // 构造新的 slice 对象
    return slice{newptr, old.len, newcap}
}
```

#### slice 结构体

```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

#### 分配和拷贝函数详解

* `mallocgc`：分配一块 heap 内存（等价于 `make([]T, cap)`）。

* `memmove`：底层是汇编，类似 `memcpy`。

* `et.size`：每个元素的大小。

#### append 背后的操作

* `cap` 足够：原地追加，新 `slice` 与旧共享数组

* `cap` 不够：分配新数组、复制原数据、添加新元素

* 多次 `append`：容量以倍数或 1.25 倍增长，最多增长到 2ⁿ 或更多

* 返回值：一定是新 `slice`，即使底层没变

* 类型安全：编译器会根据 `T` 类型生成对 `growslice` 的调用

#### growslice 的效率与陷阱

* 复制开销：每次扩容都需要拷贝旧数据（`O(n)` 复杂度）；

* 内存浪费：新数组会有空余空间（以倍数扩容）；

* 共享底层数组：没扩容时，多个 `slice` 会共享底层数组，易出错。

### 数组 vs 切片 GC 行为对比

|  特性   |  数组（[N]T）   |  切片（[]T）   |
| --- | --- | --- |
|  是否固定大小   |  是否固定大小   |  否   |
|  存储位置   | 值类型，复制时复制整个内容   | 引用类型，复制的是头结构    |
|  GC 行为   |  如果被释放，整体数组会被 GC   |  如果原数组被引用，即使只用一部分，整个数组仍保留   |
|  是否共享底层内存   |  不会共享   |  可能多个切片共享底层数组   |
|  GC 是否会收回未使用部分   |  若无引用，整体释放   |  若有切片引用，整个底层数组不会释放   |

#### 内存泄露风险

```go
func getSlice() []byte {
    large := make([]byte, 1<<20) // 1MB
    return large[:100]           // 返回一个小切片，但引用了整个大数组
}
```

上面这段代码虽然只返回了 100 个字节的切片，但实际上因为这个切片仍然引用了整个 1MB 的底层数组，GC 不会释放整个数组，直到这个小切片本身不再被引用。

**解决方法**

```go
func getCopiedSlice() []byte {
    large := make([]byte, 1<<20)
    result := make([]byte, 100)
    copy(result, large[:100])
    return result // 这样只引用小数据
}
```

#### 如何避免 GC 泄漏

|  场景   | 建议做法    |
| --- | --- |
|  截取切片时只需部分数据   |  使用 `copy()` 拷贝到新切片   |
|  数据生命周期短   |  注意避免长生命周期切片引用大量数据   |
|  要释放大数组   |  确保没有切片引用底层数组   |
|  想清空切片   |  使用 `s = nil` 而不是 `s = s[:0]`（后者不会释放内存）   |

### 使用切片来模拟分页

核心功能：

* 模拟有很多行的表格数据

* 支持分页查看（指定页码、每页条数）

* 支持翻页（上一页、下一页）

* 支持跳转页数

#### 示例效果（CLI）

```shell
$ go run main.go
[Page 1/10] Showing 10 of 100 rows:
1. Name: Alice       Age: 25
2. Name: Bob         Age: 30
...

[n] Next Page | [p] Previous Page | [q] Quit | [g 5] Go to page 5
> 
```

#### 项目结构

```go
table-paginator/
├── main.go            // 启动入口
├── paginator/
│   └── paginator.go   // 分页逻辑
├── data/
│   └── mock.go        // 模拟表格数据
```

#### 核心代码框架预览

* data/mock.go

```go
package data

type Person struct {
    Name string
    Age  int
}

func GenerateMockData(total int) []Person {
    people := make([]Person, total)
    for i := 0; i < total; i++ {
        people[i] = Person{
            Name: fmt.Sprintf("User%03d", i+1),
            Age:  20 + (i % 30),
        }
    }
    return people
}
```

* paginator/paginator.go

```go
package paginator

import "fmt"

type Page[T any] struct {
    Items      []T
    PageNo     int
    PageSize   int
    TotalItems int
}

func Paginate[T any](data []T, pageNo, pageSize int) Page[T] {
    total := len(data)
    start := (pageNo - 1) * pageSize
    end := start + pageSize
    if start > total {
        start = total
    }
    if end > total {
        end = total
    }
    return Page[T]{
        Items:      data[start:end],
        PageNo:     pageNo,
        PageSize:   pageSize,
        TotalItems: total,
    }
}
```

* main.go

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
    "table-paginator/data"
    "table-paginator/paginator"
)

func main() {
    people := data.GenerateMockData(100)
    pageSize := 10
    pageNo := 1
    reader := bufio.NewReader(os.Stdin)

    for {
        page := paginator.Paginate(people, pageNo, pageSize)
        fmt.Printf("\n[Page %d/%d] Showing %d of %d rows:\n",
            page.PageNo,
            (page.TotalItems+pageSize-1)/pageSize,
            len(page.Items),
            page.TotalItems)

        for i, p := range page.Items {
            fmt.Printf("%2d. Name: %-10s Age: %d\n", (page.PageNo-1)*pageSize+i+1, p.Name, p.Age)
        }

        fmt.Print("\n[n] Next Page | [p] Previous Page | [g 5] Go to page 5 | [q] Quit\n> ")
        input, _ := reader.ReadString('\n')
        input = strings.TrimSpace(input)

        switch {
        case input == "n":
            pageNo++
        case input == "p":
            if pageNo > 1 {
                pageNo--
            }
        case strings.HasPrefix(input, "g "):
            var newPage int
            fmt.Sscanf(input, "g %d", &newPage)
            if newPage >= 1 {
                pageNo = newPage
            }
        case input == "q":
            return
        default:
            fmt.Println("Invalid command")
        }
    }
}
```

### Go slice切片跟Java的List和C#.NET的List异同

#### 相同点（共性）

|  特性   |  描述   |
| --- | --- |
| 动态增长    |  都可以在运行时动态扩容，无需指定固定长度   |
|  支持下标访问   | 可以通过 [] 操作符访问和修改元素    |
|  有长度和容量（或大小）   |  有长度和容量（或大小）   |
|  内部基于数组实现   |  内部基于数组实现   |
|  支持切片/子列表（部分）   |  Go、Java（`subList()`）、C#（`GetRange()`）都可以实现   |

#### 不同点（详细对比）

|  特性   |  Go slice   |  Java List   |  `C# List<T>`   |
| --- | --- | --- | --- |
|  所在语言   |  Go   |  Java   |  C# (.NET)   |
|  是否原生类型   |  ✅ 是语言内建类型   |  ❌ 是接口（通常用 `ArrayList` 等）   |  ❌ 是类   |
|  是否线程安全   |  ❌ 否   |  ❌ 否（`Collections.synchronizedList` 可封装）   |  ❌ 否（要手动加锁）   |
|  内部扩容机制   |  默认 翻倍/1.25倍 扩容   |  增长约 50%（ArrayList）   |  每次翻倍扩容（默认实现）   |
|  内存管理   |  自动垃圾回收   |  自动垃圾回收   |  自动垃圾回收   |
|  切片是否共享底层数组   |  ✅ 会共享   |  ❌ `subList()` 是新对象引用同数组   |  ✅ `GetRange()` 拷贝数据   |
|  底层可见性   |  有 `len` 和 `cap` 可查看   | 只有 `size()`    |  有 `Count` 和 `Capacity`   |
| 可否使用 `append()`    |  ✅ 内置 `append()`   |  ❌ 使用 `add()` 或 `addAll()`   |  ❌ 使用 `Add()` 或 `AddRange()`   |

#### 动态扩容机制

**Go Slice**

使用 `append` 追加元素时，若容量不足，按以下规则扩容：

* 容量 < 1024：双倍扩容
* 容量 ≥ 1024：按 1.25 倍扩容，扩容后生成新底层数组，原数组可能被垃圾回收。

**Java ArrayList**

默认扩容为当前容量的 1.5 倍（如初始容量 10 → 15 → 22 → ...） 

**C# List`<T>`**

扩容时容量翻倍（如初始容量 4 → 8 → 16 → ...）

#### 常用操作对比

|  操作   |  `Go Slice`  |  `Java List`   |  `C# List<T>`   |
| --- | --- | --- | --- |
|  添加元素   |  `append(slice, element)`   | `add(element)`    |  `Add(element)`   |
|  删除元素   |  通过 `append` 拼接前后片段   |  `remove(index)` 或 `remove(object)`   |  `RemoveAt(index)` 或 `Remove(obj)`   |
|  截取子集   |  `s[start:end]`（共享底层数组）   | `subList(from, to)`    |  `GetRange(from, count)`   |
|  容量预分配   |  `make([]T, len, cap)`   | `new ArrayList<>(initialCapacity)`    |  `new List<T>(capacity)`   |
|  拷贝   |  `copy(dst, src)`（浅拷贝）   | `new ArrayList<>(srcList)`    |  `new List<T>(srcList)`
   |

### 切片典型的代码示例

#### 切片配合结构体（最常见）

* 示例一

```go
package main

import "fmt"

type User struct {
	ID   int
	Name string
}

func main() {
	users := []User{
		{ID: 1, Name: "Alice"},
		{ID: 2, Name: "Bob"},
	}

	for _, user := range users {
		fmt.Printf("ID: %d, Name: %s\n", user.ID, user.Name)
	}
}
```

* 示例二

```go
package main

import (
	"fmt"
	"sort"
)

// 定义结构体
type Person struct {
	Name string
	Age  int
}

// 结构体方法（值接收者）
func (p Person) String() string {
	return fmt.Sprintf("%s (%d)", p.Name, p.Age)
}

// 结构体方法（指针接收者）
func (p *Person) Grow() {
	p.Age++
}

func main() {
	// 创建结构体切片
	people := []Person{
		{"Alice", 25},
		{"Bob", 30},
		{"Charlie", 20},
	}

	// 遍历切片（值传递）
	for _, p := range people {
		fmt.Println(p)
	}

	// 修改结构体（需使用指针切片）
	peoplePtr := []*Person{
		{"Alice", 25},
		{"Bob", 30},
		{"Charlie", 20},
	}
	for _, p := range peoplePtr {
		p.Grow() // 调用指针接收者方法
	}

	// 按年龄排序
	sort.Slice(people, func(i, j int) bool {
		return people[i].Age < people[j].Age
	})
	fmt.Println("Sorted:", people)

	// 动态添加元素
	people = append(people, Person{"David", 40})
}
```

关键点：

* 使用指针切片（`[]*Person`）避免结构体复制开销

* 结合 `sort.Slice` 实现自定义排序

* 通过 `append` 动态添加元素

* 示例三：模拟 `List` 的 `Add` 方法

```go
// 定义泛型结构体，包含切片字段
type Container[T any] struct {
    Elements []T
}

// 添加元素方法（使用泛型接收器）
func (c *Container[T]) Add(element T) {
    c.Elements = append(c.Elements, element)
}

// 使用示例
intContainer := Container[int]{Elements: []int{1, 2}}
intContainer.Add(3)  // Elements: [1,2,3]

strContainer := Container[string]{Elements: []string{"a", "b"}}
strContainer.Add("c")  // Elements: [a,b,c]
```

#### 切片配合接口（处理多种类型）

* 示例一

```go
package main

import "fmt"

type Shape interface {
	Area() float64
}

type Circle struct{ Radius float64 }
type Rectangle struct{ Width, Height float64 }

func (c Circle) Area() float64    { return 3.14 * c.Radius * c.Radius }
func (r Rectangle) Area() float64 { return r.Width * r.Height }

func printAreas(shapes []Shape) {
	for _, s := range shapes {
		fmt.Printf("Area: %.2f\n", s.Area())
	}
}

func main() {
	shapes := []Shape{
		Circle{Radius: 3},
		Rectangle{Width: 4, Height: 5},
	}
	printAreas(shapes)
}
```

* 示例二

```go
// 定义数值类型约束接口
type Numeric interface {
    int | float64
}

// 泛型函数：计算切片元素和
func Sum[T Numeric](s []T) T {
    var total T
    for _, v := range s {
        total += v
    }
    return total
}

// 使用示例
fmt.Println(Sum([]int{1, 2, 3}))       // 6
fmt.Println(Sum([]float64{1.1, 2.2}))  // 3.3
```

关键点：`Numeric` 接口通过类型集合约束切片元素类型，确保操作合法性

* 示例三

```go
// 定义数据访问接口
type DataStore[T any] interface {
    GetAll() []T
    Add(item T)
}

// 实现接口的泛型结构体
type GenericSliceStore[T any] struct {
    data []T
}

func (g *GenericSliceStore[T]) GetAll() []T {
    return g.data
}

func (g *GenericSliceStore[T]) Add(item T) {
    g.data = append(g.data, item)
}

// 使用示例
store := &GenericSliceStore[int]{data: []int{10}}
store.Add(20)
fmt.Println(store.GetAll())  // [10,20]
```

关键点：接口通过泛型类型参数定义方法，结构体实现接口时绑定具体类型

#### 切片配合泛型（Go 1.18+）

* 示例一

```go
package main

import "fmt"

// 定义泛型函数，打印任何类型切片
func PrintSlice[T any](items []T) {
	for _, item := range items {
		fmt.Println(item)
	}
}

func main() {
	ints := []int{1, 2, 3}
	strs := []string{"Go", "Rust", "Python"}

	PrintSlice[int](ints)
	PrintSlice[string](strs)
}
```

* 示例二

```go
package main

import "fmt"

// 泛型过滤函数（过滤满足条件的元素）
func Filter[T any](slice []T, test func(T) bool) []T {
	result := make([]T, 0)
	for _, v := range slice {
		if test(v) {
			result = append(result, v)
		}
	}
	return result
}

// 泛型映射函数（转换元素类型）
func Map[T any, U any](slice []T, mapper func(T) U) []U {
	result := make([]U, len(slice))
	for i, v := range slice {
		result[i] = mapper(v)
	}
	return result
}

// 泛型结构体（存储切片）
type Repository[T any] struct {
	Data []T
}

func (r *Repository[T]) Add(item T) {
	r.Data = append(r.Data, item)
}

func main() {
	// 过滤整数切片
	numbers := []int{1, 2, 3, 4, 5}
	evenNumbers := Filter(numbers, func(n int) bool {
		return n%2 == 0
	})
	fmt.Println("Even numbers:", evenNumbers) // [2 4]

	// 转换字符串切片为长度切片
	words := []string{"apple", "banana", "cherry"}
	lengths := Map(words, func(s string) int {
		return len(s)
	})
	fmt.Println("Word lengths:", lengths) // [5 6 6]

	// 使用泛型结构体
	repo := Repository[string]{Data: []string{"Go", "Rust"}}
	repo.Add("C++")
	fmt.Println("Repository:", repo.Data) // [Go Rust C++]
}
```

关键点：

* 使用 `T any` 定义泛型类型参数

* 泛型函数支持类型安全操作（`Filter、Map`）

* 泛型结构体（`Repository[T]`）封装切片操作

* 示例三：模拟队列

利用切片实现泛型队列，支持动态类型存储：

```go
type Queue[T any] []T

// 入队方法
func (q *Queue[T]) Enqueue(item T) {
    *q = append(*q, item)
}

// 出队方法（返回泛型零值）
func (q *Queue[T]) Dequeue() T {
    if len(*q) == 0 {
        return *new(T) // 返回类型T的零值
    }
    item := (*q)[0]
    *q = (*q)[1:]
    return item
}

// 使用示例
var q Queue[string]
q.Enqueue("first")
q.Enqueue("second")
fmt.Println(q.Dequeue())  // "first"
```

关键点：通过 `any` 类型约束实现多类型队列，利用切片特性动态调整

* 示例四

```go
// 用户订单结构体
type UserOrder[T comparable] struct {
    UserID  T
    Items   []string  // 字符串切片字段
}

// 泛型订单管理器
type OrderManager[K comparable, V any] struct {
    Orders map[K][]V  // 键值对中值类型为切片
}

// 添加订单方法
func (m *OrderManager[K, V]) Add(key K, value V) {
    m.Orders[key] = append(m.Orders[key], value)
}

// 使用示例
manager := OrderManager[int, string]{Orders: make(map[int][]string)}
manager.Add(1001, "itemA")
manager.Add(1001, "itemB")  // Orders[1001]: [itemA, itemB]
```

关键点：结合 `map` 和切片实现多层级泛型数据存储，支持复杂业务场景

* 示例五

```go
// 切片去重（需comparable约束）
func Deduplicate[T comparable](s []T) []T {
    seen := make(map[T]bool)
    result := []T{}
    for _, v := range s {
        if !seen[v] {
            seen[v] = true
            result = append(result, v)
        }
    }
    return result
}

// 使用示例
nums := []int{1, 2, 2, 3}
strs := []string{"a", "a", "b"}
fmt.Println(Deduplicate(nums)) // [1 2 3]
fmt.Println(Deduplicate(strs)) // [a b]
```

* 示例六

```go
func SliceToMap[T any, K comparable](s []T, keyFunc func(T) K) map[K]T {
    m := make(map[K]T)
    for _, item := range s {
        m[keyFunc(item)] = item
    }
    return m
}

// 使用示例
userMap := SliceToMap(users, func(u User) int { return u.ID })
// 输出map[1:{1 Alice} 2:{2 Bob} 3:{3 Charlie}]
```

#### 切片 + 泛型 + 结构体（组合使用）

* 示例一

```go
package main

import "fmt"

type User struct {
	ID   int
	Name string
}

func FilterSlice[T any](items []T, filter func(T) bool) []T {
	var result []T
	for _, item := range items {
		if filter(item) {
			result = append(result, item)
		}
	}
	return result
}

func main() {
	users := []User{
		{ID: 1, Name: "Alice"},
		{ID: 2, Name: "Bob"},
		{ID: 3, Name: "Eve"},
	}

	filtered := FilterSlice(users, func(u User) bool {
		return u.ID%2 == 1 // 只保留奇数 ID
	})

	for _, u := range filtered {
		fmt.Println(u)
	}
}
```

* 示例二

```go
package main

import "fmt"

// 定义接口
type IDisplay interface {
	Display() string
}

// 实现接口的结构体
type Product struct {
	Name  string
	Price float64
}

func (p Product) Display() string {
	return fmt.Sprintf("%s ($%.2f)", p.Name, p.Price)
}

type User struct {
	Username string
	Email    string
}

func (u User) Display() string {
	return fmt.Sprintf("%s <%s>", u.Username, u.Email)
}

// 泛型容器（要求类型实现 IDisplay 接口）
type DisplayBox[T IDisplay] struct {
	Items []T
}

func (b *DisplayBox[T]) Add(item T) {
	b.Items = append(b.Items, item)
}

func (b *DisplayBox[T]) ShowAll() {
	for _, item := range b.Items {
		fmt.Println(item.Display())
	}
}

func main() {
	// 创建容器并添加不同类型数据
	box := DisplayBox[IDisplay]{}
	box.Add(Product{"Laptop", 999.99})
	box.Add(User{"Alice", "alice@example.com"})

	box.ShowAll()
	// 输出：
	// Laptop ($999.99)
	// Alice <alice@example.com>
}
```

关键点：

* 结合接口约束（`T IDisplay`）实现类型安全

* 泛型容器存储接口类型切片

* 统一调用接口方法（`item.Display()`）

---

* 示例三

```go
type Identifiable interface {
    GetID() int
}

type Product struct {
    ID    int
    Name  string
    Price float64
}
func (p Product) GetID() int { return p.ID }

// 通用查询函数
func FindByID[T Identifiable](items []T, id int) (T, bool) {
    for _, item := range items {
        if item.GetID() == id {
            return item, true
        }
    }
    var zero T
    return zero, false
}

// 使用示例
products := []Product{{1, "Laptop", 999.9}}
result, found := FindByID(products, 1)
```

* 多级切片处理

```go
type Matrix[T any] [][]T

func (m Matrix[T]) Flatten() []T {
    var result []T
    for _, row := range m {
        result = append(result, row...)
    }
    return result
}

// 使用示例
intMatrix := Matrix[int]{{1,2}, {3,4}}
strMatrix := Matrix[string]{{"a","b"}, {"c"}}
fmt.Println(intMatrix.Flatten()) // [1 2 3 4]
fmt.Println(strMatrix.Flatten()) // [a b c]
```

#### 切片 + 泛型接口 + 排序（高级组合）

```go
package main

import (
	"fmt"
	"slices" // Go 1.21+ 官方 slices 工具包
)

type Person struct {
	Name string
	Age  int
}

func main() {
	people := []Person{
		{"Alice", 25},
		{"Bob", 19},
		{"Eve", 31},
	}

	// 按 Age 排序
	slices.SortFunc(people, func(a, b Person) int {
		return a.Age - b.Age
	})

	fmt.Println(people)
}
```

#### 总结

* 切片 + 结构体：最常见的组合，适合表示表格、列表等数据结构

* 切片 + 接口：支持多态，适合处理多种类型（如图形、设备等）

* 切片 + 泛型：类型安全、复用性强，适合通用算法/工具方法

* 切片 + 泛型 + 结构体：结构化数据处理 + 高性能泛型，写库非常合适

### 通用分页 + 泛型过滤 + 可视化表格

#### 功能

* 任意结构体类型的数据切片

* 泛型过滤（支持传入条件函数）

* 分页（页码 + 每页大小）

* Web UI 表格展示（用 Go Serve HTML）

#### 目录结构

```go
slice-table-demo/
├── main.go
├── data.go       // 模拟数据和数据结构
├── pagination.go // 泛型分页 + 过滤
├── templates/
│   └── index.html
```

#### 代码示例

* data.go — 模拟数据定义

```go
package main

type User struct {
	ID    int
	Name  string
	Email string
	Age   int
}

func GetMockUsers() []User {
	users := make([]User, 100)
	for i := range users {
		users[i] = User{
			ID:    i + 1,
			Name:  "User_" + string('A'+(i%26)),
			Email: fmt.Sprintf("user%d@example.com", i+1),
			Age:   18 + (i % 30),
		}
	}
	return users
}
```

* pagination.go — 泛型分页与过滤

```go
package main

func FilterSlice[T any](items []T, filter func(T) bool) []T {
	var result []T
	for _, item := range items {
		if filter(item) {
			result = append(result, item)
		}
	}
	return result
}

func PaginateSlice[T any](items []T, page, pageSize int) []T {
	start := (page - 1) * pageSize
	end := start + pageSize
	if start >= len(items) {
		return []T{}
	}
	if end > len(items) {
		end = len(items)
	}
	return items[start:end]
}
```

* templates/index.html — Web 表格模板（带分页）

```go
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>User Table</title>
  <style>
    table { border-collapse: collapse; width: 100%; }
    th, td { padding: 8px; border: 1px solid #ccc; text-align: left; }
  </style>
</head>
<body>
  <h1>User List (Page {{.Page}})</h1>
  <table>
    <thead>
      <tr>
        <th>ID</th><th>Name</th><th>Email</th><th>Age</th>
      </tr>
    </thead>
    <tbody>
    {{range .Users}}
      <tr>
        <td>{{.ID}}</td><td>{{.Name}}</td><td>{{.Email}}</td><td>{{.Age}}</td>
      </tr>
    {{end}}
    </tbody>
  </table>
  <p>
    <a href="/?page={{.PrevPage}}">Prev</a> |
    <a href="/?page={{.NextPage}}">Next</a>
  </p>
</body>
</html>
```

* main.go — 启动 HTTP Server

```go
package main

import (
	"fmt"
	"html/template"
	"net/http"
	"strconv"
)

type PageData struct {
	Users    []User
	Page     int
	PrevPage int
	NextPage int
}

func main() {
	tmpl := template.Must(template.ParseFiles("templates/index.html"))

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		pageSize := 10
		pageStr := r.URL.Query().Get("page")
		page, _ := strconv.Atoi(pageStr)
		if page <= 0 {
			page = 1
		}

		users := GetMockUsers()

		// 过滤条件：例如只显示年龄 > 25 的用户
		filtered := FilterSlice(users, func(u User) bool {
			return u.Age > 25
		})

		pagedUsers := PaginateSlice(filtered, page, pageSize)

		data := PageData{
			Users:    pagedUsers,
			Page:     page,
			PrevPage: max(1, page-1),
			NextPage: page + 1,
		}

		tmpl.Execute(w, data)
	})

	fmt.Println("Listening on http://localhost:8080")
	http.ListenAndServe(":8080", nil)
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}
```

#### 运行

```shell
go run main.go

# 浏览器访问：http://localhost:8080
```

#### 总结与选型建议

|  场景   |  推荐方案   |  优势   |
| --- | --- | --- |
|  固定结构集合   |  结构体切片   |  直观、类型安全   |
|  跨类型数据操作   |  泛型切片+接口约束   |  代码复用率高   |
|  高性能批处理   | 预分配切片+数组重用    |  减少GC压力   |
|  复杂数据结构   | 嵌套泛型结构体    |  灵活扩展多维数据   |
|  类型转换优化   | 泛型转换工具函数    |  避免冗余代码   |

### slices 包处理切片（slice）的常见操作

#### 常用函数

* `slices.Sort`：对切片排序（需要元素可比较）

* `slices.BinarySearch`：二分查找（已排序切片）

* `slices.Index`：查找第一个等于给定值的索引

* `slices.Contains`：判断切片中是否包含某值

* `slices.Equal`：判断两个切片是否相等（顺序和值都一样）

* `slices.Clone`：拷贝一个切片

* `slices.Compact`：移除相邻重复值（适合排好序的切片）

* `slices.Delete`：删除指定索引范围的元素

* `slices.Insert`：插入元素到切片中

* `slices.Reverse`：反转切片

#### 使用 slices 包示例

```go
package main

import (
	"fmt"
	"slices"
)

func main() {
	ages := []int{32, 18, 45, 21, 18}

	// 排序
	slices.Sort(ages)
	fmt.Println("Sorted:", ages)

	// 查找
	idx := slices.Index(ages, 45)
	fmt.Println("Index of 45:", idx)

	// 是否包含
	fmt.Println("Contains 21:", slices.Contains(ages, 21))

	// 去重（只去相邻重复）
	dedup := slices.Clone([]int{1, 1, 2, 2, 2, 3})
	slices.Compact(dedup)
	fmt.Println("Compact:", dedup)

	// 插入
	inserted := slices.Insert([]int{1, 2, 4}, 2, 3)
	fmt.Println("After insert:", inserted)

	// 删除
	deleted := slices.Delete(inserted, 1, 3) // 删除第1到3个元素
	fmt.Println("After delete:", deleted)
}
```

### 多个 goroutine 并发修改切片

#### 使用 sync.Mutex 加锁 来保证并发安全

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var (
		slice []int
		mu    sync.Mutex
		wg    sync.WaitGroup
	)

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(val int) {
			defer wg.Done()

			mu.Lock()
			slice = append(slice, val)
			mu.Unlock()
		}(i)
	}

	wg.Wait()
	fmt.Println("Final slice:", slice)
}
```

* 每个 `goroutine` 在 `append` 前先加锁，操作完成后释放锁。

* `sync.Mutex` 保证了 `append` 的原子性，避免了竞态。

#### 使用 channel 做串行通信，避免共享内存

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	slice := []int{}
	ch := make(chan int)
	var wg sync.WaitGroup

	// 启动一个 goroutine 专门负责写入切片
	go func() {
		for val := range ch {
			slice = append(slice, val)
		}
	}()

	// 启动多个生产者 goroutine
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(val int) {
			defer wg.Done()
			ch <- val
		}(i)
	}

	wg.Wait()
	close(ch) // 所有写入完成后关闭通道

	fmt.Println("Final slice:", slice)
}
```

* 所有写入通过 `channel` 串行进行，避免了加锁。

* 这是 Go 推荐的方式：不要通过共享内存来通信，而应该通过通信来共享内存。

#### 对比总结

|  方法   |  优点   |  缺点   |
| --- | --- | --- |
|  `sync.Mutex`   |  简单高效，适合低冲突场景   |  容易死锁，复杂操作难管理   |
|  `channel`   |  更符合 Go 哲学，逻辑清晰   | 需要一个写入协程，性能略低    |
