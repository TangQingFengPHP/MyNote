### 简介

`generic` 通常翻译成“泛型”。

Go 从 `1.18` 开始支持泛型。

泛型解决的核心问题很直接：

```text
同一套逻辑，适配多种类型，同时保留编译期类型检查。
```

比如求和。

没有泛型时，`int` 要写一份：

```go
func SumInts(nums []int) int {
	var total int

	for _, n := range nums {
		total += n
	}

	return total
}
```

`float64` 又要写一份：

```go
func SumFloat64s(nums []float64) float64 {
	var total float64

	for _, n := range nums {
		total += n
	}

	return total
}
```

逻辑完全一样，只是类型不同。

使用泛型后，可以合并成一个函数：

```go
func Sum[T int | float64](nums []T) T {
	var total T

	for _, n := range nums {
		total += n
	}

	return total
}
```

一句话概括：

```text
泛型让代码把“类型”也当成参数传进去。
```

### 第一个泛型函数

先看一个最简单的泛型函数：

```go
package main

import "fmt"

func Identity[T any](value T) T {
	return value
}

func main() {
	fmt.Println(Identity(100))
	fmt.Println(Identity("hello"))
	fmt.Println(Identity(true))
}
```

输出：

```text
100
hello
true
```

这里的核心是：

```go
func Identity[T any](value T) T
```

拆开看：

| 部分 | 含义 |
| --- | --- |
| `T` | 类型参数名称 |
| `any` | 类型约束，表示任意类型 |
| `value T` | 参数类型是 T |
| 返回值 `T` | 返回值类型也是 T |

`Identity(100)` 调用时，`T` 会被推断成 `int`。

`Identity("hello")` 调用时，`T` 会被推断成 `string`。

### 类型参数是什么

普通函数的参数是“值参数”。

```go
func Add(a int, b int) int {
	return a + b
}
```

`a`、`b` 是值参数。

泛型函数多了一类参数，叫“类型参数”。

```go
func Add[T int | int64](a T, b T) T {
	return a + b
}
```

`T` 是类型参数。

它不是某个具体值，而是某个具体类型的占位符。

调用时：

```go
Add(1, 2)
```

编译器推断出：

```text
T = int
```

调用：

```go
Add(int64(1), int64(2))
```

编译器推断出：

```text
T = int64
```

### any 约束

`any` 是 `interface{}` 的别名。

```go
type any = interface{}
```

它表示任意类型。

```go
package main

import "fmt"

func Print[T any](value T) {
	fmt.Printf("type=%T value=%v\n", value, value)
}

func main() {
	Print(100)
	Print("go")
	Print([]int{1, 2, 3})
}
```

输出：

```text
type=int value=100
type=string value=go
type=[]int value=[1 2 3]
```

但 `any` 也有限制。

它只表示“任意类型”，不代表这个类型支持所有操作。

下面这种代码不能通过编译：

```go
func Add[T any](a T, b T) T {
	return a + b
}
```

原因是：

```text
any 不保证 T 支持 + 运算。
```

如果要使用 `+`，必须给类型参数加更具体的约束。

### 类型约束

泛型里的约束用来限制类型参数能是什么类型。

例如：

```go
func Add[T int | int64 | float64](a T, b T) T {
	return a + b
}
```

这里的：

```go
T int | int64 | float64
```

表示：

```text
T 只能是 int、int64、float64 这几种类型之一。
```

因为这些类型都支持 `+`，所以函数体里可以写：

```go
return a + b
```

完整示例：

```go
package main

import "fmt"

func Add[T int | int64 | float64](a T, b T) T {
	return a + b
}

func main() {
	fmt.Println(Add(1, 2))
	fmt.Println(Add(int64(10), int64(20)))
	fmt.Println(Add(1.5, 2.5))
}
```

输出：

```text
3
30
4
```

### 自定义约束

如果很多函数都要使用同一组数值类型，可以把约束抽出来。

```go
type Number interface {
	int | int64 | float64
}
```

然后复用：

```go
func Add[T Number](a T, b T) T {
	return a + b
}

func Sum[T Number](nums []T) T {
	var total T

	for _, n := range nums {
		total += n
	}

	return total
}
```

完整示例：

```go
package main

import "fmt"

type Number interface {
	int | int64 | float64
}

func Sum[T Number](nums []T) T {
	var total T

	for _, n := range nums {
		total += n
	}

	return total
}

func main() {
	fmt.Println(Sum([]int{1, 2, 3}))
	fmt.Println(Sum([]int64{10, 20, 30}))
	fmt.Println(Sum([]float64{1.1, 2.2, 3.3}))
}
```

输出：

```text
6
60
6.6
```

### ~ 是什么

`~` 用来匹配底层类型。

先看一个自定义类型：

```go
type UserID int64
```

`UserID` 是一个新类型，它的底层类型是 `int64`。

如果约束写成：

```go
type Integer interface {
	int64
}
```

那么 `UserID` 不满足这个约束。

因为 `UserID` 不是 `int64`，只是底层类型是 `int64`。

如果希望支持这种自定义类型，要写：

```go
type Integer interface {
	~int64
}
```

完整示例：

```go
package main

import "fmt"

type UserID int64

type Integer interface {
	~int | ~int64
}

func Add[T Integer](a T, b T) T {
	return a + b
}

func main() {
	var a UserID = 100
	var b UserID = 200

	fmt.Println(Add(a, b))
}
```

输出：

```text
300
```

可以这样记：

```text
int64 只匹配 int64。
~int64 匹配 int64，以及底层类型是 int64 的自定义类型。
```

### comparable 约束

`comparable` 是 Go 内置约束。

它表示类型可以使用 `==` 和 `!=` 比较。

常见可比较类型：

* 数字
* 字符串
* 布尔
* 指针
* channel
* 字段都可比较的结构体

slice、map、func 不能比较。

泛型里经常用 `comparable` 写查找函数：

```go
package main

import "fmt"

func Contains[T comparable](items []T, target T) bool {
	for _, item := range items {
		if item == target {
			return true
		}
	}

	return false
}

func main() {
	fmt.Println(Contains([]int{1, 2, 3}, 2))
	fmt.Println(Contains([]string{"go", "java"}, "php"))
}
```

输出：

```text
true
false
```

`comparable` 也常用于 map key。

```go
func Keys[K comparable, V any](m map[K]V) []K {
	keys := make([]K, 0, len(m))

	for k := range m {
		keys = append(keys, k)
	}

	return keys
}
```

map 的 key 必须可比较，所以这里的 `K` 要写成 `comparable`。

### 类型推断

调用泛型函数时，很多时候不用手动写类型。

```go
fmt.Println(Contains([]int{1, 2, 3}, 2))
```

编译器可以根据参数推断：

```text
T = int
```

也可以显式指定：

```go
fmt.Println(Contains[int]([]int{1, 2, 3}, 2))
```

多数业务代码不需要显式指定。

但如果类型参数只出现在返回值里，编译器通常推断不出来。

```go
func Zero[T any]() T {
	var zero T
	return zero
}
```

调用时需要写类型：

```go
v := Zero[int]()
```

### 泛型切片工具：Map

`Map` 用来把一个切片转换成另一个切片。

例如把 `[]int` 转成 `[]string`。

```go
package main

import "fmt"

func Map[T any, R any](items []T, mapper func(T) R) []R {
	result := make([]R, 0, len(items))

	for _, item := range items {
		result = append(result, mapper(item))
	}

	return result
}

func main() {
	nums := []int{1, 2, 3}

	texts := Map(nums, func(n int) string {
		return fmt.Sprintf("num=%d", n)
	})

	fmt.Println(texts)
}
```

输出：

```text
[num=1 num=2 num=3]
```

这个函数里有两个类型参数：

```go
func Map[T any, R any](items []T, mapper func(T) R) []R
```

含义：

* `T` 是输入元素类型
* `R` 是输出元素类型

### 泛型切片工具：Filter

`Filter` 用来过滤切片。

```go
package main

import "fmt"

func Filter[T any](items []T, predicate func(T) bool) []T {
	result := make([]T, 0, len(items))

	for _, item := range items {
		if predicate(item) {
			result = append(result, item)
		}
	}

	return result
}

func main() {
	nums := []int{1, 2, 3, 4, 5}

	evens := Filter(nums, func(n int) bool {
		return n%2 == 0
	})

	fmt.Println(evens)
}
```

输出：

```text
[2 4]
```

同一个 `Filter` 也能过滤结构体切片：

```go
type User struct {
	ID     int
	Name   string
	Active bool
}

users := []User{
	{ID: 1, Name: "张三", Active: true},
	{ID: 2, Name: "李四", Active: false},
}

activeUsers := Filter(users, func(u User) bool {
	return u.Active
})
```

### 泛型切片工具：Find

`Find` 用来查找第一个满足条件的元素。

找到了返回元素和 `true`。

没找到返回零值和 `false`。

```go
package main

import "fmt"

func Find[T any](items []T, predicate func(T) bool) (T, bool) {
	var zero T

	for _, item := range items {
		if predicate(item) {
			return item, true
		}
	}

	return zero, false
}

type User struct {
	ID   int
	Name string
}

func main() {
	users := []User{
		{ID: 1, Name: "张三"},
		{ID: 2, Name: "李四"},
	}

	user, ok := Find(users, func(u User) bool {
		return u.ID == 2
	})

	fmt.Println(user, ok)
}
```

输出：

```text
{2 李四} true
```

### 泛型切片工具：Reduce

`Reduce` 用来把一组数据归约成一个结果。

例如求总和、拼接字符串、统计数量。

```go
package main

import "fmt"

func Reduce[T any, R any](items []T, initial R, reducer func(R, T) R) R {
	result := initial

	for _, item := range items {
		result = reducer(result, item)
	}

	return result
}

func main() {
	nums := []int{1, 2, 3, 4}

	total := Reduce(nums, 0, func(sum int, n int) int {
		return sum + n
	})

	fmt.Println(total)
}
```

输出：

```text
10
```

### 泛型结构体

结构体也可以带类型参数。

最简单的是 `Box`：

```go
package main

import "fmt"

type Box[T any] struct {
	Value T
}

func (b Box[T]) Get() T {
	return b.Value
}

func (b *Box[T]) Set(value T) {
	b.Value = value
}

func main() {
	intBox := Box[int]{Value: 100}
	fmt.Println(intBox.Get())

	stringBox := Box[string]{Value: "hello"}
	stringBox.Set("go")
	fmt.Println(stringBox.Get())
}
```

输出：

```text
100
go
```

定义泛型类型时：

```go
type Box[T any] struct
```

给泛型类型定义方法时，接收者也要带上类型参数：

```go
func (b Box[T]) Get() T
func (b *Box[T]) Set(value T)
```

### 实战 Demo：泛型 Stack

栈是典型的泛型数据结构。

`int` 栈、`string` 栈、`User` 栈，逻辑都一样。

```go
package main

import "fmt"

type Stack[T any] struct {
	items []T
}

func NewStack[T any]() *Stack[T] {
	return &Stack[T]{
		items: make([]T, 0),
	}
}

func (s *Stack[T]) Push(item T) {
	s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
	if len(s.items) == 0 {
		var zero T
		return zero, false
	}

	lastIndex := len(s.items) - 1
	item := s.items[lastIndex]
	s.items = s.items[:lastIndex]

	return item, true
}

func (s *Stack[T]) Len() int {
	return len(s.items)
}

func main() {
	stack := NewStack[int]()

	stack.Push(10)
	stack.Push(20)

	v, ok := stack.Pop()
	fmt.Println(v, ok)
	fmt.Println(stack.Len())
}
```

输出：

```text
20 true
1
```

### 实战 Demo：泛型 Set

Go 没有内置 Set，可以用 `map[T]struct{}` 封装。

因为 map key 必须可比较，所以 `T` 需要 `comparable` 约束。

```go
package main

import "fmt"

type Set[T comparable] struct {
	data map[T]struct{}
}

func NewSet[T comparable](items ...T) *Set[T] {
	set := &Set[T]{
		data: make(map[T]struct{}, len(items)),
	}

	for _, item := range items {
		set.Add(item)
	}

	return set
}

func (s *Set[T]) Add(item T) {
	s.data[item] = struct{}{}
}

func (s *Set[T]) Remove(item T) {
	delete(s.data, item)
}

func (s *Set[T]) Contains(item T) bool {
	_, ok := s.data[item]
	return ok
}

func (s *Set[T]) Values() []T {
	values := make([]T, 0, len(s.data))

	for item := range s.data {
		values = append(values, item)
	}

	return values
}

func main() {
	ids := NewSet(1, 2, 2, 3)

	fmt.Println(ids.Contains(2))
	fmt.Println(ids.Values())

	names := NewSet("go", "java", "go")
	fmt.Println(names.Contains("php"))
	fmt.Println(names.Values())
}
```

输出类似：

```text
true
[1 2 3]
false
[go java]
```

map 遍历顺序不固定，所以 `Values()` 输出顺序可能变化。

### 实战 Demo：通用 API 响应

Web 项目里经常有统一响应结构。

没有泛型时，`Data` 可能写成 `any`：

```go
type ApiResponse struct {
	Code    int    `json:"code"`
	Message string `json:"message"`
	Data    any    `json:"data"`
}
```

这种写法灵活，但丢失了 `Data` 的具体类型。

使用泛型可以保留类型：

```go
type ApiResponse[T any] struct {
	Code    int    `json:"code"`
	Message string `json:"message"`
	Data    T      `json:"data"`
}
```

完整示例：

```go
package main

import (
	"encoding/json"
	"fmt"
)

type ApiResponse[T any] struct {
	Code    int    `json:"code"`
	Message string `json:"message"`
	Data    T      `json:"data"`
}

type User struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

func Success[T any](data T) ApiResponse[T] {
	return ApiResponse[T]{
		Code:    200,
		Message: "success",
		Data:    data,
	}
}

func main() {
	resp := Success(User{ID: 1, Name: "张三"})

	data, err := json.Marshal(resp)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(data))
}
```

输出：

```text
{"code":200,"message":"success","data":{"id":1,"name":"张三"}}
```

### 实战 Demo：分页结果

分页结果也很适合泛型。

用户分页、商品分页、订单分页，结构基本一样，只是列表元素类型不同。

```go
package main

import (
	"encoding/json"
	"fmt"
)

type PageResult[T any] struct {
	Page     int   `json:"page"`
	PageSize int   `json:"pageSize"`
	Total    int64 `json:"total"`
	Items    []T   `json:"items"`
}

type Product struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Price int    `json:"price"`
}

func NewPageResult[T any](page int, pageSize int, total int64, items []T) PageResult[T] {
	return PageResult[T]{
		Page:     page,
		PageSize: pageSize,
		Total:    total,
		Items:    items,
	}
}

func main() {
	result := NewPageResult(1, 10, 100, []Product{
		{ID: 1, Name: "键盘", Price: 199},
		{ID: 2, Name: "鼠标", Price: 99},
	})

	data, err := json.Marshal(result)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(data))
}
```

输出：

```text
{"page":1,"pageSize":10,"total":100,"items":[{"id":1,"name":"键盘","price":199},{"id":2,"name":"鼠标","price":99}]}
```

### 实战 Demo：通用内存 Repository

泛型也可以用来封装一些通用仓储逻辑。

先定义一个实体约束：

```go
type Entity interface {
	GetID() int64
}
```

然后 Repository 只接收实现了 `GetID` 的类型。

完整示例：

```go
package main

import "fmt"

type Entity interface {
	GetID() int64
}

type Repository[T Entity] struct {
	data map[int64]T
}

func NewRepository[T Entity]() *Repository[T] {
	return &Repository[T]{
		data: make(map[int64]T),
	}
}

func (r *Repository[T]) Save(entity T) {
	r.data[entity.GetID()] = entity
}

func (r *Repository[T]) FindByID(id int64) (T, bool) {
	entity, ok := r.data[id]
	return entity, ok
}

func (r *Repository[T]) List() []T {
	result := make([]T, 0, len(r.data))

	for _, entity := range r.data {
		result = append(result, entity)
	}

	return result
}

type User struct {
	ID   int64
	Name string
}

func (u User) GetID() int64 {
	return u.ID
}

type Product struct {
	ID    int64
	Name  string
	Price int
}

func (p Product) GetID() int64 {
	return p.ID
}

func main() {
	userRepo := NewRepository[User]()
	userRepo.Save(User{ID: 1, Name: "张三"})

	user, ok := userRepo.FindByID(1)
	fmt.Println(user, ok)

	productRepo := NewRepository[Product]()
	productRepo.Save(Product{ID: 100, Name: "键盘", Price: 199})

	product, ok := productRepo.FindByID(100)
	fmt.Println(product, ok)
}
```

输出：

```text
{1 张三} true
{100 键盘 199} true
```

这种写法适合演示通用逻辑。

真实业务里的 Repository 往往会有复杂查询、事务、权限、缓存等差异，不一定都适合硬抽成泛型。

### 泛型接口

接口也可以带类型参数。

例如定义一个通用 Store：

```go
type Store[T any] interface {
	Save(value T) error
	Find() (T, bool)
}
```

完整示例：

```go
package main

import "fmt"

type Store[T any] interface {
	Save(value T) error
	Find() (T, bool)
}

type MemoryStore[T any] struct {
	value T
	ok    bool
}

func (s *MemoryStore[T]) Save(value T) error {
	s.value = value
	s.ok = true
	return nil
}

func (s *MemoryStore[T]) Find() (T, bool) {
	return s.value, s.ok
}

func main() {
	var store Store[string] = &MemoryStore[string]{}

	_ = store.Save("hello")

	value, ok := store.Find()
	fmt.Println(value, ok)
}
```

输出：

```text
hello true
```

### 泛型方法的注意点

Go 支持给泛型类型定义方法。

```go
type Box[T any] struct {
	Value T
}

func (b Box[T]) Get() T {
	return b.Value
}
```

但方法不能单独声明一组新的类型参数。

下面这种写法不能编译：

```go
func (b Box[T]) Map[R any](mapper func(T) R) R {
	return mapper(b.Value)
}
```

这类需求可以改成普通泛型函数：

```go
func MapBox[T any, R any](b Box[T], mapper func(T) R) R {
	return mapper(b.Value)
}
```

### 泛型 vs interface

`interface` 适合表达行为。

```go
type Writer interface {
	Write(p []byte) (int, error)
}
```

重点是：

```text
对象能做什么。
```

泛型适合保留具体类型并复用算法。

```go
func First[T any](items []T) (T, bool) {
	if len(items) == 0 {
		var zero T
		return zero, false
	}

	return items[0], true
}
```

重点是：

```text
同一套代码处理不同类型，同时返回具体类型。
```

对比：

```go
func FirstAny(items []any) (any, bool) {
	if len(items) == 0 {
		return nil, false
	}

	return items[0], true
}
```

`FirstAny` 返回的是 `any`，调用方还要类型断言。

泛型版本返回的是准确类型：

```go
value, ok := First([]int{1, 2, 3})
```

这里的 `value` 是 `int`。

### 标准库里的泛型工具

Go 标准库里已经有一些泛型工具包，例如 `slices`、`maps`。

示例：

```go
package main

import (
	"fmt"
	"slices"
)

func main() {
	nums := []int{3, 1, 2}

	slices.Sort(nums)

	fmt.Println(nums)
	fmt.Println(slices.Contains(nums, 2))
}
```

输出：

```text
[1 2 3]
true
```

这些工具能直接处理不同元素类型的切片。

日常项目里，能使用标准库工具时，没必要重复封装一套。

### 什么时候适合用泛型

适合使用泛型的场景：

* 多种类型共享同一套逻辑
* 返回值需要保留具体类型
* 集合工具函数，例如 `Map`、`Filter`、`Contains`
* 通用数据结构，例如 `Set`、`Stack`、`Queue`
* 通用返回结构，例如 `ApiResponse[T]`、`PageResult[T]`
* 类型安全比 `any` + 类型断言更重要

示例：

```go
func Contains[T comparable](items []T, target T) bool
type Set[T comparable] struct
type PageResult[T any] struct
```

### 什么时候不适合用泛型

不适合泛型化的场景：

* 只有一个类型会使用
* 各类型之间业务差异很大
* 类型约束写得特别复杂
* 泛型让调用方更难读懂
* 为了少写几行代码而牺牲清晰度

比如两个业务流程只是表面类似，但校验、查询、事务、日志完全不同，强行抽成泛型会让代码变绕。

这种情况下，普通函数或接口可能更合适。

### 常见问题

#### any 和泛型是一回事吗

不是。

`any` 是一个约束，也可以作为普通接口类型使用。

泛型是“带类型参数的代码”。

例如：

```go
func Print[T any](value T)
```

这里使用了泛型，`any` 只是 `T` 的约束。

而下面这个函数不是泛型：

```go
func Print(value any)
```

它只是接收一个空接口值。

#### comparable 可以表示所有能排序的类型吗

不能。

`comparable` 只表示可以用 `==`、`!=` 比较。

它不代表可以用 `<`、`>` 排序。

例如布尔值可以比较相等，但不能排序。

如果需要 `<`、`>`，需要使用自定义约束：

```go
type Ordered interface {
	~int | ~int64 | ~float64 | ~string
}
```

#### 泛型类型可以自动推断类型参数吗

泛型函数通常可以通过参数推断。

```go
Contains([]int{1, 2, 3}, 2)
```

泛型类型初始化时通常需要写出类型参数：

```go
stack := Stack[int]{}
```

如果通过构造函数，函数可以帮助推断一部分类型。

```go
set := NewSet(1, 2, 3)
```

这里 `NewSet` 是泛型函数，编译器可以根据传入参数推断 `T`。

#### 泛型能完全替代 interface 吗

不能。

泛型和接口解决的问题不同。

接口更适合描述行为和解耦实现。

泛型更适合复用类型安全的算法和数据结构。

例如 `io.Reader` 这种“能读取”的行为抽象，用接口更自然。

例如 `Set[T]` 这种“相同逻辑适配不同元素类型”的容器，用泛型更自然。

### 总结

`generic` 的核心可以压缩成几句话：

```text
泛型让类型也能成为参数。
类型参数写在 [] 里。
约束限制类型参数能做什么。
any 表示任意类型。
comparable 表示支持 == 和 !=。
~T 表示包含底层类型是 T 的自定义类型。
泛型适合通用算法、通用容器、通用响应结构。
```

日常选择可以按下面这张表判断：

| 场景 | 常见写法 |
| --- | --- |
| 任意类型参数 | `[T any]` |
| 可比较类型 | `[T comparable]` |
| 数值计算 | 自定义 `Number` 约束 |
| 支持自定义底层类型 | `~int`、`~string` |
| 多类型联合 | `int | int64 | float64` |
| 通用切片转换 | `Map[T, R]` |
| 通用切片过滤 | `Filter[T]` |
| 通用集合 | `Set[T comparable]` |
| 通用响应 | `ApiResponse[T]` |
| 通用分页 | `PageResult[T]` |
| 行为抽象 | interface 更合适 |

泛型不是越多越好。它最适合消除真正重复的类型逻辑，并且让返回值保持具体类型。代码只有一个类型会用、业务规则差异很大、约束写得很绕时，直接写普通代码通常更清楚。
