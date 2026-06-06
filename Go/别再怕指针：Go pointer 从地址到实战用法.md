### 简介

`pointer` 是 Go 里绕不开的基础概念。

中文通常叫“指针”。

它的作用很简单：

```text
保存一个变量的内存地址，然后通过这个地址访问或修改变量。
```

先看一个最小例子：

```go
a := 10
p := &a
```

这里：

* `a` 是普通变量，值是 `10`
* `&a` 是 `a` 的地址
* `p` 是指针变量，保存了 `a` 的地址
* `*p` 可以拿到 `p` 指向的值

一句话概括：

```text
指针保存地址，解引用指针可以操作地址背后的值。
```

Go 的指针比 C/C++ 更克制。

Go 有指针，但没有指针运算。

也就是说，Go 里可以拿地址、传地址、通过地址改值，但不能写 `p++` 去移动地址。

这让 Go 保留了指针的实用性，同时减少了很多内存越界问题。

### & 和 * 是什么

指针最核心的两个符号是：

| 符号 | 作用 |
| --- | --- |
| `&` | 取地址 |
| `*` | 声明指针类型，或者解引用指针 |

示例：

```go
package main

import "fmt"

func main() {
	a := 10
	p := &a

	fmt.Println(a)
	fmt.Println(p)
	fmt.Println(*p)
}
```

输出类似：

```text
10
0x1400000e0a8
10
```

地址每次运行可能不同，具体数值不用记。

重点是：

```text
p 里放的是 a 的地址。
*p 表示通过这个地址拿到 a 的值。
```

### 通过指针修改原变量

指针不只是能读取，还能修改。

```go
package main

import "fmt"

func main() {
	a := 10
	p := &a

	*p = 99

	fmt.Println(a)
}
```

输出：

```text
99
```

`*p = 99` 的意思是：

```text
把 p 指向的那个变量改成 99。
```

因为 `p` 指向的是 `a`，所以 `a` 被改成了 `99`。

### 指针类型

`*int` 表示“指向 int 的指针”。

```go
var p *int
```

`*string` 表示“指向 string 的指针”。

```go
var p *string
```

完整示例：

```go
package main

import "fmt"

func main() {
	age := 18
	name := "张三"

	var agePtr *int = &age
	var namePtr *string = &name

	fmt.Println(*agePtr)
	fmt.Println(*namePtr)
}
```

输出：

```text
18
张三
```

指针类型要匹配。

`*int` 只能保存 `int` 变量的地址。

`*string` 只能保存 `string` 变量的地址。

### 指针的零值是 nil

只声明指针，不赋值时，它的零值是 `nil`。

```go
package main

import "fmt"

func main() {
	var p *int

	fmt.Println(p == nil)
}
```

输出：

```text
true
```

nil 指针没有指向任何变量。

不能直接解引用 nil 指针。

```go
package main

func main() {
	var p *int

	*p = 10
}
```

运行会 panic：

```text
panic: runtime error: invalid memory address or nil pointer dereference
```

安全写法：

```go
if p != nil {
	fmt.Println(*p)
}
```

### new 创建非 nil 指针

`new(T)` 会创建一个 `T` 类型的零值，并返回它的指针。

```go
package main

import "fmt"

func main() {
	p := new(int)

	fmt.Println(p == nil)
	fmt.Println(*p)

	*p = 100
	fmt.Println(*p)
}
```

输出：

```text
false
0
100
```

`new(int)` 返回的是 `*int`，指向一个值为 `0` 的 int。

结构体也可以用 `new`：

```go
type User struct {
	ID   int
	Name string
}

u := new(User)
u.Name = "张三"
```

不过结构体更常见的写法是：

```go
u := &User{
	ID:   1,
	Name: "张三",
}
```

### Go 函数参数都是值传递

Go 的函数参数都是值传递。

传普通变量时，函数里拿到的是一份拷贝。

```go
package main

import "fmt"

func change(x int) {
	x = 100
}

func main() {
	a := 10

	change(a)

	fmt.Println(a)
}
```

输出：

```text
10
```

`change` 修改的是参数副本，不会影响外面的 `a`。

### 指针传参可以修改原值

如果函数需要修改外部变量，可以传指针。

```go
package main

import "fmt"

func change(x *int) {
	*x = 100
}

func main() {
	a := 10

	change(&a)

	fmt.Println(a)
}
```

输出：

```text
100
```

这里传进去的依然是值。

只不过这个值是地址的副本。

地址副本和原地址指向同一块数据，所以函数里可以改到原变量。

### 实战 Demo：交换两个数

交换两个变量，必须修改外部变量，所以使用指针。

```go
package main

import "fmt"

func swap(a, b *int) {
	*a, *b = *b, *a
}

func main() {
	x := 10
	y := 20

	swap(&x, &y)

	fmt.Println(x, y)
}
```

输出：

```text
20 10
```

如果不传指针，只能交换函数内部的副本，外面的 `x`、`y` 不会变。

### 结构体指针

结构体指针在 Go 项目里非常常见。

```go
package main

import "fmt"

type User struct {
	ID    int
	Name  string
	Email string
}

func main() {
	u := &User{
		ID:   1,
		Name: "张三",
	}

	u.Email = "zhangsan@example.com"

	fmt.Printf("%+v\n", u)
}
```

输出：

```text
&{ID:1 Name:张三 Email:zhangsan@example.com}
```

`u` 是 `*User`。

访问字段时可以直接写：

```go
u.Name
```

不用写成：

```go
(*u).Name
```

Go 会自动处理结构体指针的字段访问。

### 实战 Demo：修改用户邮箱

```go
package main

import "fmt"

type User struct {
	ID    int
	Name  string
	Email string
}

func UpdateEmail(user *User, email string) {
	if user == nil {
		return
	}

	user.Email = email
}

func main() {
	user := &User{
		ID:   1,
		Name: "张三",
	}

	UpdateEmail(user, "zhangsan@example.com")

	fmt.Printf("%+v\n", user)
}
```

输出：

```text
&{ID:1 Name:张三 Email:zhangsan@example.com}
```

这里的 `UpdateEmail` 接收 `*User`，因为函数需要修改原用户对象。

函数开头判断 `user == nil`，可以避免空指针 panic。

### 方法里的指针接收者

结构体方法也经常用指针。

先看值接收者：

```go
package main

import "fmt"

type Counter struct {
	value int
}

func (c Counter) Inc() {
	c.value++
}

func main() {
	c := Counter{}

	c.Inc()
	c.Inc()

	fmt.Println(c.value)
}
```

输出：

```text
0
```

原因是 `Inc` 拿到的是 `Counter` 的副本。

再看指针接收者：

```go
package main

import "fmt"

type Counter struct {
	value int
}

func (c *Counter) Inc() {
	c.value++
}

func main() {
	c := Counter{}

	c.Inc()
	c.Inc()

	fmt.Println(c.value)
}
```

输出：

```text
2
```

`func (c *Counter) Inc()` 里的 `c` 是指针，所以能修改原结构体。

### 接收者自动取地址

Go 在方法调用时会做一些自动转换。

```go
type User struct {
	Name string
}

func (u *User) Rename(name string) {
	u.Name = name
}
```

下面这样可以调用：

```go
u := User{Name: "张三"}
u.Rename("李四")
```

虽然 `Rename` 需要 `*User`，但 `u` 是可取地址的变量，Go 会自动变成：

```go
(&u).Rename("李四")
```

如果值不可取地址，就不能自动取地址。

例如：

```go
User{Name: "张三"}.Rename("李四")
```

这种写法会编译失败，因为临时字面量不能这样隐式取地址调用指针接收者方法。

可以改成：

```go
(&User{Name: "张三"}).Rename("李四")
```

### 接收者怎么选

可以按下面几条判断：

| 场景 | 推荐 |
| --- | --- |
| 方法需要修改结构体 | 指针接收者 |
| 结构体比较大，复制成本高 | 指针接收者 |
| 结构体里包含 `sync.Mutex` 等不能复制的字段 | 指针接收者 |
| 方法只读，并且结构体很小 | 值接收者也可以 |
| 同一个类型已经有指针接收者方法 | 通常保持一致，继续用指针接收者 |

不要为了“看起来高性能”到处传指针。

指针会带来共享可变状态，代码也更容易被外部修改。

小结构体只读场景，值传递反而更清楚。

### 指针和数组

数组是值类型。

传数组给函数时，会复制整个数组。

```go
package main

import "fmt"

func change(nums [3]int) {
	nums[0] = 100
}

func main() {
	nums := [3]int{1, 2, 3}

	change(nums)

	fmt.Println(nums)
}
```

输出：

```text
[1 2 3]
```

使用数组指针可以修改原数组。

```go
package main

import "fmt"

func change(nums *[3]int) {
	nums[0] = 100
}

func main() {
	nums := [3]int{1, 2, 3}

	change(&nums)

	fmt.Println(nums)
}
```

输出：

```text
[100 2 3]
```

实际项目里，数组指针不算常见。

更多时候会用 slice。

### 数组指针和指针数组

这两个名字很像，但完全不是一回事。

```go
var a *[3]int
var b [3]*int
```

含义：

| 写法 | 含义 |
| --- | --- |
| `*[3]int` | 指向数组的指针 |
| `[3]*int` | 元素是指针的数组 |

示例：

```go
package main

import "fmt"

func main() {
	nums := [3]int{10, 20, 30}

	var arrayPtr *[3]int = &nums
	fmt.Println(arrayPtr[0])

	var ptrArray [3]*int
	for i := range nums {
		ptrArray[i] = &nums[i]
	}

	*ptrArray[0] = 99

	fmt.Println(nums)
}
```

输出：

```text
10
[99 20 30]
```

### 指针和 slice

slice 本身是一个小结构，大致包含：

```text
底层数组指针
长度
容量
```

把 slice 传给函数时，会复制这个小结构。

但它们指向同一个底层数组。

所以修改元素会影响原 slice。

```go
package main

import "fmt"

func change(nums []int) {
	nums[0] = 100
}

func main() {
	nums := []int{1, 2, 3}

	change(nums)

	fmt.Println(nums)
}
```

输出：

```text
[100 2 3]
```

这种场景不需要传 `*[]int`。

### append 后想影响外部 slice 怎么办

如果函数里 `append`，并且希望外部 slice 也拿到新长度，通常返回新 slice。

```go
package main

import "fmt"

func add(nums []int, value int) []int {
	nums = append(nums, value)
	return nums
}

func main() {
	nums := []int{1, 2, 3}

	nums = add(nums, 4)

	fmt.Println(nums)
}
```

输出：

```text
[1 2 3 4]
```

也可以传 `*[]int`：

```go
func add(nums *[]int, value int) {
	*nums = append(*nums, value)
}
```

但这种写法通常不如返回新 slice 清楚。

### 指针和 map

map 传给函数时，也能修改底层数据。

```go
package main

import "fmt"

func addScore(scores map[string]int) {
	scores["go"] = 100
}

func main() {
	scores := make(map[string]int)

	addScore(scores)

	fmt.Println(scores)
}
```

输出：

```text
map[go:100]
```

普通增删改不需要传 `*map[K]V`。

只有需要把整个 map 变量替换成另一个 map 时，才可能考虑传 map 指针。

多数情况下，直接返回新 map 更清楚。

### 指针和 channel

channel 也不需要经常传指针。

```go
package main

import "fmt"

func send(ch chan int) {
	ch <- 100
}

func main() {
	ch := make(chan int)

	go send(ch)

	fmt.Println(<-ch)
}
```

输出：

```text
100
```

channel 本身就能在函数间传递通信能力。

一般不用写 `*chan int`。

### 不推荐 *interface{}

`interface{}` 或 `any` 本身已经可以保存“类型信息 + 值”。

大多数情况下不需要 `*interface{}`。

不推荐：

```go
func handle(v *any) {
}
```

更常见：

```go
func handle(v any) {
}
```

如果需要类型安全，可以用泛型：

```go
func handle[T any](v T) {
}
```

`*interface{}` 容易让代码变复杂，除非确实需要修改接口变量本身。

### 返回局部变量指针安全吗

Go 里可以返回局部变量的指针。

```go
package main

import "fmt"

func NewInt(value int) *int {
	x := value
	return &x
}

func main() {
	p := NewInt(100)
	fmt.Println(*p)
}
```

输出：

```text
100
```

这在 Go 里是安全的。

编译器会做逃逸分析。

如果局部变量的地址需要在函数返回后继续使用，编译器会把它放到合适的位置。

所以不需要套用 C/C++ 里“不能返回局部变量地址”的规则。

真正需要关注的是：

```text
返回指针后，外部是否真的应该共享并修改这份数据。
```

### 指针的指针

指针也可以有自己的地址。

所以会出现指针的指针：

```go
package main

import "fmt"

func main() {
	a := 10
	p := &a
	pp := &p

	fmt.Println(a)
	fmt.Println(*p)
	fmt.Println(**pp)
}
```

输出：

```text
10
10
10
```

`pp` 的类型是 `**int`。

日常业务代码里，多级指针很少见。

出现 `**T` 时，通常说明函数需要修改一个指针变量本身。

多数场景可以通过返回值让代码更清楚。

### 实战 Demo：链表

链表节点天生适合用指针。

每个节点都通过指针指向下一个节点。

```go
package main

import "fmt"

type Node struct {
	Value int
	Next  *Node
}

func Append(head *Node, value int) *Node {
	node := &Node{Value: value}

	if head == nil {
		return node
	}

	cur := head
	for cur.Next != nil {
		cur = cur.Next
	}

	cur.Next = node
	return head
}

func Print(head *Node) {
	for cur := head; cur != nil; cur = cur.Next {
		fmt.Print(cur.Value)
		if cur.Next != nil {
			fmt.Print(" -> ")
		}
	}
	fmt.Println()
}

func main() {
	var head *Node

	head = Append(head, 1)
	head = Append(head, 2)
	head = Append(head, 3)

	Print(head)
}
```

输出：

```text
1 -> 2 -> 3
```

这里 `var head *Node` 的 nil 值刚好可以表示空链表。

### 实战 Demo：Service 保存状态

服务对象内部有状态时，方法通常使用指针接收者。

```go
package main

import "fmt"

type User struct {
	ID   int
	Name string
}

type UserService struct {
	nextID int
}

func NewUserService() *UserService {
	return &UserService{nextID: 1}
}

func (s *UserService) CreateUser(name string) *User {
	user := &User{
		ID:   s.nextID,
		Name: name,
	}

	s.nextID++

	return user
}

func main() {
	service := NewUserService()

	u1 := service.CreateUser("张三")
	u2 := service.CreateUser("李四")

	fmt.Println(u1)
	fmt.Println(u2)
}
```

输出：

```text
&{1 张三}
&{2 李四}
```

`CreateUser` 会修改 `s.nextID`，所以 `UserService` 方法使用指针接收者。

### 实战 Demo：Repository 保存对象指针

有时 map 里会保存对象指针。

```go
package main

import "fmt"

type User struct {
	ID   int
	Name string
}

type UserRepository struct {
	data map[int]*User
}

func NewUserRepository() *UserRepository {
	return &UserRepository{
		data: make(map[int]*User),
	}
}

func (r *UserRepository) Save(user *User) {
	if user == nil {
		return
	}

	r.data[user.ID] = user
}

func (r *UserRepository) FindByID(id int) (*User, bool) {
	user, ok := r.data[id]
	return user, ok
}

func main() {
	repo := NewUserRepository()

	user := &User{
		ID:   1,
		Name: "张三",
	}

	repo.Save(user)

	found, ok := repo.FindByID(1)
	if !ok {
		panic("user not found")
	}

	found.Name = "李四"

	fmt.Println(repo.data[1])
}
```

输出：

```text
&{1 李四}
```

`repo.data` 保存的是 `*User`。

`found` 和 map 里的值指向同一个用户对象，所以修改 `found.Name` 会影响仓库里的对象。

这种共享修改很方便，但也要小心副作用。

如果不希望外部修改仓库内部数据，可以返回值拷贝。

### 实战 Demo：购物车商品数量

```go
package main

import "fmt"

type Goods struct {
	Name  string
	Price float64
	Count int
}

func AddCount(goods *Goods, count int) {
	if goods == nil || count <= 0 {
		return
	}

	goods.Count += count
}

func Total(goods Goods) float64 {
	return goods.Price * float64(goods.Count)
}

func main() {
	apple := Goods{
		Name:  "苹果",
		Price: 6.5,
		Count: 2,
	}

	AddCount(&apple, 3)

	fmt.Printf("%s 数量=%d 总价=%.2f\n", apple.Name, apple.Count, Total(apple))
}
```

输出：

```text
苹果 数量=5 总价=32.50
```

`AddCount` 要修改商品数量，所以接收 `*Goods`。

`Total` 只读取数据，所以接收 `Goods` 值也可以。

### 什么时候适合用指针

适合使用指针的场景：

* 函数需要修改传入变量
* 方法需要修改结构体字段
* 结构体比较大，复制成本明显
* 结构体里包含 `sync.Mutex` 等不能复制的字段
* 需要表达“可能没有值”，例如 `*User` 为 nil 表示没查到
* 链表、树等数据结构需要节点互相引用
* 工厂函数需要返回对象并让外部继续操作

示例：

```go
func UpdateUser(user *User) {}
func (s *UserService) CreateUser(name string) *User {}
func NewUserRepository() *UserRepository {}
```

### 什么时候不适合用指针

不适合滥用指针的场景：

* 小结构体只读访问
* slice、map、channel 普通传参
* 只是为了“看起来更省内存”
* 不希望函数修改外部数据
* 指针层级已经让代码读起来费劲

示例：

```go
type Point struct {
	X int
	Y int
}

func Distance(a, b Point) int {
	dx := a.X - b.X
	dy := a.Y - b.Y
	return dx*dx + dy*dy
}
```

`Point` 很小，函数只读，传值很清楚。

### 常见问题

#### Go 指针能做加减吗

不能。

下面这种写法不允许：

```go
p++
p = p + 1
```

Go 不支持普通指针算术。

需要做底层内存操作时可以接触 `unsafe.Pointer`，但那已经是更低层的内容，普通业务代码不应该依赖它。

#### nil 指针可以调用方法吗

可以调用，但方法里必须自己处理 nil。

```go
package main

import "fmt"

type Node struct {
	Value int
	Next  *Node
}

func (n *Node) IsEmpty() bool {
	return n == nil
}

func main() {
	var n *Node

	fmt.Println(n.IsEmpty())
}
```

输出：

```text
true
```

如果方法里直接访问字段：

```go
func (n *Node) ValueText() string {
	return fmt.Sprintf("%d", n.Value)
}
```

当 `n == nil` 时就会 panic。

#### 指针一定更快吗

不一定。

指针可以避免大对象拷贝，但也可能让对象逃逸到堆上，增加 GC 压力。

小对象传值通常已经足够清楚，也可能更容易被编译器优化。

性能敏感场景应该看基准测试，而不是只凭“指针更快”判断。

#### 返回结构体还是返回结构体指针

取决于语义。

如果返回的是一个小值对象，不需要共享修改，返回值很自然。

```go
func NewPoint(x, y int) Point {
	return Point{X: x, Y: y}
}
```

如果返回的是服务对象、仓储对象、需要持续修改状态的对象，返回指针更常见。

```go
func NewUserService() *UserService {
	return &UserService{nextID: 1}
}
```

#### 指针和引用是一回事吗

Go 没有 C++ 那种引用语法。

Go 指针是明确的地址值。

slice、map、channel 有引用语义，但它们不是普通指针。

理解成下面这样更准确：

```text
指针：显式保存地址。
slice/map/channel：值里包含指向底层数据结构的引用信息。
```

### 总结

`pointer` 的核心可以压缩成几句话：

```text
& 取地址。
*T 表示 T 类型指针。
*p 表示访问 p 指向的值。
指针零值是 nil，nil 不能随便解引用。
Go 函数参数都是值传递，传指针可以共享并修改同一份数据。
```

日常选择可以按下面这张表判断：

| 场景 | 常见写法 |
| --- | --- |
| 获取变量地址 | `p := &x` |
| 读取指针指向的值 | `value := *p` |
| 修改指针指向的值 | `*p = newValue` |
| 创建零值指针 | `p := new(T)` |
| 创建结构体指针 | `p := &User{...}` |
| 函数修改外部变量 | `func update(x *T)` |
| 方法修改结构体 | `func (x *T) Method()` |
| slice 追加后影响外部 | 返回新 slice |
| map 普通增删改 | 直接传 map |
| channel 收发 | 直接传 channel |

指针不是越多越好。它真正解决的是“共享同一份数据”和“避免不必要的大对象拷贝”。需要修改原值、表达可空对象、构建链表树结构、维护服务状态时，指针非常合适；只读小值、普通 slice/map/channel 传参时，值语义往往更清楚。
