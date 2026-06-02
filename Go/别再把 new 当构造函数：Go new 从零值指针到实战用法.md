### 简介

`new` 是 Go 里的内置函数，作用很简单：

```text
给某个类型分配一块零值内存，然后返回这块内存的指针。
```

语法如下：

```go
p := new(Type)
```

比如：

```go
p := new(int)
```

这行代码会得到一个 `*int` 类型的指针，指针指向的值是 `int` 的零值，也就是 `0`。

一句话概括：

```text
new(T) 返回的是 *T，里面放的是 T 类型的零值。
```

注意，`new` 不是很多语言里的“构造函数”。它不会调用构造方法，也不会帮结构体字段填业务默认值，只做两件事：

* 分配内存
* 清零并返回指针

### new 的基本用法

先看一个最小示例：

```go
package main

import "fmt"

func main() {
	p := new(int)

	fmt.Printf("type: %T\n", p)
	fmt.Printf("value: %d\n", *p)

	*p = 100
	fmt.Printf("new value: %d\n", *p)
}
```

输出：

```text
type: *int
value: 0
new value: 100
```

这段代码里：

```go
p := new(int)
```

可以理解成：

```go
var v int
p := &v
```

所以 `p` 不是 `nil`，它是一个合法指针，只是指向的值刚开始是零值。

### 什么是零值

Go 里每种类型都有零值。`new` 分配出来的内容，一定是该类型的零值。

常见零值如下：

| 类型 | 零值 |
| --- | --- |
| `int` | `0` |
| `float64` | `0` |
| `bool` | `false` |
| `string` | `""` |
| 指针 | `nil` |
| slice | `nil` |
| map | `nil` |
| channel | `nil` |
| struct | 每个字段都是各自类型的零值 |
| array | 每个元素都是各自类型的零值 |

示例：

```go
package main

import "fmt"

type User struct {
	ID     int
	Name   string
	Active bool
}

func main() {
	i := new(int)
	s := new(string)
	b := new(bool)
	u := new(User)

	fmt.Printf("int: %v\n", *i)
	fmt.Printf("string: %q\n", *s)
	fmt.Printf("bool: %v\n", *b)
	fmt.Printf("user: %+v\n", *u)
}
```

输出：

```text
int: 0
string: ""
bool: false
user: {ID:0 Name: Active:false}
```

### 结构体使用 new

结构体是 `new` 最常见的使用对象之一。

```go
package main

import "fmt"

type User struct {
	ID   int
	Name string
	Age  int
}

func main() {
	u := new(User)

	fmt.Printf("type: %T\n", u)
	fmt.Printf("value: %+v\n", u)

	u.ID = 1
	u.Name = "张三"
	u.Age = 20

	fmt.Printf("after set: %+v\n", u)
}
```

输出：

```text
type: *main.User
value: &{ID:0 Name: Age:0}
after set: &{ID:1 Name:张三 Age:20}
```

这里有一个 Go 语法糖：

```go
u.Name = "张三"
```

虽然 `u` 是 `*User`，但访问字段时不用写成：

```go
(*u).Name = "张三"
```

Go 会自动处理结构体指针的字段访问。

### new 和 &T{} 的区别

在 Go 项目里，更常见的写法其实是：

```go
u := &User{}
```

它和下面这句效果基本一样：

```go
u := new(User)
```

完整示例：

```go
package main

import "fmt"

type User struct {
	ID   int
	Name string
}

func main() {
	u1 := new(User)
	u2 := &User{}

	fmt.Printf("u1: %T %+v\n", u1, u1)
	fmt.Printf("u2: %T %+v\n", u2, u2)
}
```

输出：

```text
u1: *main.User &{ID:0 Name:}
u2: *main.User &{ID:0 Name:}
```

两者都是创建一个 `User` 的零值，并返回 `*User`。

区别在于，`&T{}` 可以顺手初始化字段：

```go
u := &User{
	ID:   1,
	Name: "张三",
}
```

如果用 `new`，通常要后面再赋值：

```go
u := new(User)
u.ID = 1
u.Name = "张三"
```

所以日常写业务结构体时，常见选择是：

```go
u := &User{ID: 1, Name: "张三"}
```

当只需要一个零值指针时，`new(User)` 也很清楚。

### new 不是构造函数

很多语言里的 `new User()` 会执行构造逻辑。

Go 里的 `new(User)` 不会。

看下面这个配置示例：

```go
package main

import (
	"fmt"
	"time"
)

type Config struct {
	Host    string
	Port    int
	Timeout time.Duration
}

func main() {
	c := new(Config)

	fmt.Printf("%+v\n", c)
}
```

输出：

```text
&{Host: Port:0 Timeout:0s}
```

`Host` 不会自动变成 `localhost`，`Port` 不会自动变成 `8080`，`Timeout` 也不会自动变成 `30s`。

如果需要业务默认值，通常写一个工厂函数：

```go
package main

import (
	"fmt"
	"time"
)

type Config struct {
	Host    string
	Port    int
	Timeout time.Duration
}

func NewConfig() *Config {
	return &Config{
		Host:    "127.0.0.1",
		Port:    8080,
		Timeout: 30 * time.Second,
	}
}

func main() {
	c := NewConfig()
	fmt.Printf("%+v\n", c)
}
```

输出：

```text
&{Host:127.0.0.1 Port:8080 Timeout:30s}
```

这种 `NewConfig` 才更接近业务上的“构造函数”。

### new 和 make 的区别

`new` 和 `make` 是 Go 里最容易混淆的一组。

| 对比项 | `new` | `make` |
| --- | --- | --- |
| 作用 | 分配零值内存 | 初始化 slice、map、channel |
| 返回值 | 指针，`*T` | 类型本身，`T` |
| 可用类型 | 任意类型 | 只能用于 slice、map、channel |
| 常见结果 | `*User`、`*int`、`*[3]int` | `[]int`、`map[string]int`、`chan int` |

最关键的一点：

```text
new 返回指针，make 返回可直接使用的 slice、map、channel。
```

示例：

```go
package main

import "fmt"

func main() {
	p := new([]int)
	s := make([]int, 0, 3)

	fmt.Printf("p type: %T, p == nil: %v, *p == nil: %v\n", p, p == nil, *p == nil)
	fmt.Printf("s type: %T, s == nil: %v, len: %d, cap: %d\n", s, s == nil, len(s), cap(s))
}
```

输出：

```text
p type: *[]int, p == nil: false, *p == nil: true
s type: []int, s == nil: false, len: 0, cap: 3
```

`new([]int)` 得到的是一个指向 nil slice 的指针。

`make([]int, 0, 3)` 得到的是一个已经初始化好的 slice。

### new 创建 slice、map、channel 的坑

`new` 可以用于 slice、map、channel，因为它可以用于任意类型。

但这不代表适合这么用。

#### new slice

```go
package main

import "fmt"

func main() {
	p := new([]int)

	fmt.Println(*p == nil)

	*p = append(*p, 1)
	*p = append(*p, 2)

	fmt.Println(*p)
}
```

输出：

```text
true
[1 2]
```

nil slice 可以 `append`，所以这段代码能运行。

但写成这样更自然：

```go
s := make([]int, 0)
s = append(s, 1, 2)
```

#### new map

map 就不一样了。

```go
package main

func main() {
	p := new(map[string]int)

	(*p)["go"] = 100
}
```

运行会报错：

```text
panic: assignment to entry in nil map
```

原因是 `*p` 是 nil map。nil map 可以读取，但不能写入。

正确写法：

```go
package main

import "fmt"

func main() {
	m := make(map[string]int)
	m["go"] = 100

	fmt.Println(m)
}
```

如果已经用了 `new(map[string]int)`，必须再初始化一次：

```go
p := new(map[string]int)
*p = make(map[string]int)
(*p)["go"] = 100
```

这种写法绕了一圈，普通业务代码里没有必要。

#### new channel

```go
p := new(chan int)
```

此时 `*p` 是 nil channel。

nil channel 发送和接收都会一直阻塞：

```go
package main

func main() {
	p := new(chan int)

	*p <- 1
}
```

这段程序会卡住，最后可能出现：

```text
fatal error: all goroutines are asleep - deadlock!
```

channel 应该用 `make`：

```go
ch := make(chan int)
```

### nil 指针和 new 指针的区别

只声明一个指针变量时，它的值是 `nil`：

```go
var p *int
```

这个指针没有指向任何内存，直接解引用会崩溃：

```go
package main

func main() {
	var p *int
	*p = 10
}
```

运行结果：

```text
panic: runtime error: invalid memory address or nil pointer dereference
```

使用 `new` 后，指针已经指向一块合法内存：

```go
package main

import "fmt"

func main() {
	p := new(int)
	*p = 10

	fmt.Println(*p)
}
```

输出：

```text
10
```

这也是 `new` 的一个实际价值：

```text
需要一个合法指针，但当前只需要零值。
```

### 实战 Demo：用 new 创建链表节点

链表节点通常需要指针连接下一个节点，`new` 可以直接创建零值节点。

```go
package main

import "fmt"

type ListNode struct {
	Value int
	Next  *ListNode
}

func appendNode(tail *ListNode, value int) *ListNode {
	node := new(ListNode)
	node.Value = value
	tail.Next = node
	return node
}

func main() {
	head := new(ListNode)
	head.Value = 1

	tail := head
	tail = appendNode(tail, 2)
	tail = appendNode(tail, 3)
	tail = appendNode(tail, 4)

	for cur := head; cur != nil; cur = cur.Next {
		fmt.Print(cur.Value)
		if cur.Next != nil {
			fmt.Print(" -> ")
		}
	}
}
```

输出：

```text
1 -> 2 -> 3 -> 4
```

这个例子里，`new(ListNode)` 先创建一个空节点，再填充 `Value`。

实际项目里，也可以直接写成：

```go
node := &ListNode{Value: value}
```

字段较少时，`&ListNode{Value: value}` 可读性通常更好。

### 实战 Demo：指针接收者方法

当方法需要修改结构体内部状态时，通常会使用指针接收者。

```go
package main

import "fmt"

type Counter struct {
	value int
}

func (c *Counter) Inc() {
	c.value++
}

func (c *Counter) Value() int {
	return c.value
}

func main() {
	c := new(Counter)

	c.Inc()
	c.Inc()
	c.Inc()

	fmt.Println(c.Value())
}
```

输出：

```text
3
```

`new(Counter)` 得到 `*Counter`，刚好可以直接调用指针接收者方法。

不过下面这种写法也完全可以：

```go
c := &Counter{}
```

### 实战 Demo：泛型里创建零值指针

泛型场景中，`new(T)` 有时很顺手。

比如写一个通用的 `Ptr` 函数，把普通值转成指针：

```go
package main

import "fmt"

func Ptr[T any](v T) *T {
	p := new(T)
	*p = v
	return p
}

func main() {
	name := Ptr("Go")
	age := Ptr(18)
	active := Ptr(true)

	fmt.Printf("%T %v\n", name, *name)
	fmt.Printf("%T %v\n", age, *age)
	fmt.Printf("%T %v\n", active, *active)
}
```

输出：

```text
*string Go
*int 18
*bool true
```

这个例子里，`T` 是一个类型参数。`new(T)` 可以为这个类型参数创建一个零值指针。

当然，这个函数也可以写成：

```go
func Ptr[T any](v T) *T {
	return &v
}
```

两种写法都常见。

### 实战 Demo：内嵌指针字段的坑

`new` 只会初始化当前结构体本身，不会递归初始化结构体里的指针字段。

```go
package main

import "fmt"

type Logger struct {
	Level string
}

type Service struct {
	Name   string
	Logger *Logger
}

func main() {
	s := new(Service)

	fmt.Printf("service: %+v\n", s)
	fmt.Println(s.Logger == nil)
}
```

输出：

```text
service: &{Name: Logger:<nil>}
true
```

如果直接使用 `s.Logger.Level`，会出现 nil 指针问题：

```go
s.Logger.Level = "debug"
```

正确做法是先初始化内部指针字段：

```go
package main

import "fmt"

type Logger struct {
	Level string
}

type Service struct {
	Name   string
	Logger *Logger
}

func main() {
	s := new(Service)
	s.Name = "order-service"
	s.Logger = &Logger{Level: "debug"}

	fmt.Printf("%+v\n", s)
}
```

输出：

```text
&{Name:order-service Logger:0x14000010230}
```

如果希望打印指针字段里的内容，可以使用：

```go
fmt.Printf("%+v\n", *s.Logger)
```

输出：

```text
{Level:debug}
```

### new 一定分配在堆上吗

不一定。

很多资料会说 `new` 是“在堆上分配内存”。这个说法不严谨。

Go 编译器会做逃逸分析。变量到底放在栈上还是堆上，主要看它有没有逃逸出当前函数，而不是看用了 `new` 还是 `&T{}`。

示例：

```go
func local() int {
	p := new(int)
	*p = 100
	return *p
}

func escape() *int {
	p := new(int)
	*p = 100
	return p
}
```

`local` 里只返回值，指针没有逃逸出函数，编译器有机会把它放在栈上，甚至直接优化掉。

`escape` 返回了指针，这块内存函数返回后还要继续有效，通常会逃逸到堆上。

所以更准确的说法是：

```text
new 表达的是“创建一个零值指针”，至于内存放栈还是堆，由编译器决定。
```

### 什么时候适合用 new

`new` 适合这些场景：

* 需要一个合法指针，并且零值就够用
* 结构体字段很多，但当前不需要给字段设置初始值
* 数据结构节点创建，例如链表、树节点
* 泛型代码里需要创建 `*T`
* 需要强调“这里就是零值指针”

示例：

```go
counter := new(Counter)
node := new(ListNode)
value := new(int)
```

### 什么时候不推荐用 new

#### 结构体需要初始化字段

不推荐：

```go
u := new(User)
u.ID = 1
u.Name = "张三"
u.Age = 20
```

更清楚：

```go
u := &User{
	ID:   1,
	Name: "张三",
	Age:  20,
}
```

#### 创建 slice、map、channel

不推荐：

```go
s := new([]int)
m := new(map[string]int)
ch := new(chan int)
```

更常见：

```go
s := make([]int, 0)
m := make(map[string]int)
ch := make(chan int)
```

#### 想表达业务默认值

不推荐把业务默认值散落在外面：

```go
c := new(Config)
c.Host = "127.0.0.1"
c.Port = 8080
c.Timeout = 30 * time.Second
```

更适合封装成构造函数：

```go
func NewConfig() *Config {
	return &Config{
		Host:    "127.0.0.1",
		Port:    8080,
		Timeout: 30 * time.Second,
	}
}
```

### 常见问题

#### new(Type) 和 var v Type; &v 一样吗

效果基本一样。

```go
p1 := new(int)

var v int
p2 := &v
```

`p1` 和 `p2` 都是 `*int`，都指向一个值为 `0` 的 int。

#### new(User) 和 &User{} 一样吗

在创建零值结构体指针时，效果一样。

```go
u1 := new(User)
u2 := &User{}
```

两者都是 `*User`。

如果需要设置字段，`&User{...}` 更方便。

#### new 可以创建接口吗

可以，但通常没有意义。

```go
var p = new(any)
fmt.Printf("%T %v\n", p, *p)
```

`p` 的类型是 `*interface{}`，也就是指向接口值的指针。`*p` 是 nil 接口。

日常代码里很少需要 `*interface{}`，多数情况下直接使用接口值即可。

#### new 创建出来的指针需要手动释放吗

不需要。

Go 有垃圾回收，已经没有引用的对象会由 GC 回收。

### 总结

`new` 的核心可以压缩成三句话：

```text
new(T) 分配 T 类型的零值内存。
new(T) 返回 *T。
new(T) 不是业务构造函数。
```

日常选择可以按下面这张表判断：

| 场景 | 推荐写法 |
| --- | --- |
| 需要零值指针 | `new(T)` 或 `&T{}` |
| 结构体需要初始化字段 | `&T{Field: value}` |
| 创建 slice | `make([]T, len, cap)` |
| 创建 map | `make(map[K]V)` |
| 创建 channel | `make(chan T)` |
| 需要业务默认值 | `NewXxx()` 工厂函数 |
| 泛型里创建类型参数指针 | `new(T)` |

`new` 本身不复杂，真正容易出问题的是把它当成构造函数，或者拿它去代替 `make`。记住“零值指针”这四个字，大多数场景就能判断清楚。
