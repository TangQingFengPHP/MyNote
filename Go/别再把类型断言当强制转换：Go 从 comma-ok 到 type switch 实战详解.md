### 简介

Go 代码里经常能看到这样的写法：

```go
name, ok := value.(string)
```

表面看起来像类型转换，实际含义完全不同。

类型转换是在两个允许转换的类型之间改变值的类型：

```go
n := int64(100)
```

类型断言则是在检查一个接口值：

```text
接口里当前装的值，能不能当成目标类型使用？
```

类型断言不会把 `int` 变成 `string`，也不会把 `User` 自动变成 `*User`。它只负责检查接口值里的动态类型，并在匹配时取出对应的值。

一句话概括：

```text
类型转换负责“转换”，类型断言负责“检查并取出”。
```

### 接口值里到底保存了什么

理解类型断言之前，先看接口变量的组成。

一个接口值可以理解成两部分：

```text
动态类型 + 动态值
```

示例：

```go
package main

import "fmt"

func main() {
	var value any = "Go"

	fmt.Printf("动态类型：%T\n", value)
	fmt.Printf("动态值：%v\n", value)
}
```

输出：

```text
动态类型：string
动态值：Go
```

变量 `value` 在源码中的声明类型是 `any`，但接口内部保存的动态类型是 `string`，动态值是 `"Go"`。

由于编译器只允许按照接口公开的能力使用变量，下面的代码无法通过编译：

```go
var value any = "Go"

// len(value) // 编译错误，value 的静态类型是 any
```

取出字符串后才能调用字符串相关操作：

```go
text := value.(string)
fmt.Println(len(text))
```

类型断言正是连接接口静态类型和内部动态类型的工具。

### 类型断言的两种写法

类型断言的基本语法是：

```go
value := source.(TargetType)
```

其中：

* `source` 必须是接口类型的表达式
* `TargetType` 可以是具体类型，也可以是接口类型
* 目标为具体类型时，具体类型必须实现源接口

实际开发中有两种常见写法。

#### 单返回值写法

```go
text := value.(string)
```

断言成功时，`text` 得到字符串值。

断言失败时，程序直接 panic。

```go
package main

import "fmt"

func main() {
	var value any = 100

	text := value.(string)
	fmt.Println(text)
}
```

运行结果类似：

```text
panic: interface conversion: interface {} is int, not string
```

这种写法只适合类型已经由程序结构严格保证的场景。只要值来自配置、JSON、第三方库、上下文或外部输入，就不该把类型不匹配直接变成进程崩溃。

#### comma-ok 写法

更常见的安全写法是：

```go
text, ok := value.(string)
```

`ok` 表示断言是否成功：

* 匹配成功：返回目标类型的值，`ok` 为 `true`
* 匹配失败：返回目标类型的零值，`ok` 为 `false`

完整示例：

```go
package main

import "fmt"

func main() {
	var value any = 100

	text, ok := value.(string)
	fmt.Printf("text=%q, ok=%t\n", text, ok)
}
```

输出：

```text
text="", ok=false
```

这里有个容易忽略的细节：

```text
不能根据返回值是不是零值判断断言是否成功，必须检查 ok。
```

因为目标值本身完全可能就是零值：

```go
package main

import "fmt"

func main() {
	var value any = ""

	text, ok := value.(string)
	fmt.Printf("text=%q, ok=%t\n", text, ok)
}
```

输出：

```text
text="", ok=true
```

两次断言都返回空字符串，但 `ok` 不同。

### 类型必须精确匹配

目标为具体类型时，类型断言检查的是接口中的实际动态类型，不会顺手执行类型转换。

#### int 不能直接断言成 int64

```go
package main

import "fmt"

func main() {
	var value any = int(10)

	number, ok := value.(int64)
	fmt.Printf("number=%d, ok=%t\n", number, ok)
}
```

输出：

```text
number=0, ok=false
```

正确处理方式是先断言，再转换：

```go
number, ok := value.(int)
if ok {
	result := int64(number)
	fmt.Println(result)
}
```

#### 值类型和指针类型不是同一种类型

`User` 和 `*User` 是两个不同的类型。

```go
package main

import "fmt"

type User struct {
	Name string
}

func main() {
	var value any = &User{Name: "张三"}

	userValue, valueOK := value.(User)
	userPointer, pointerOK := value.(*User)

	fmt.Printf("User:  %+v, ok=%t\n", userValue, valueOK)
	fmt.Printf("*User: %+v, ok=%t\n", userPointer, pointerOK)
}
```

输出：

```text
User:  {Name:}, ok=false
*User: &{Name:张三}, ok=true
```

接口里装的是 `*User`，目标类型也必须写成 `*User`。

#### 定义类型和原类型也要区分

使用 `type` 定义出来的新类型，与底层类型不是同一个类型：

```go
package main

import "fmt"

type UserID int64

func main() {
	var value any = UserID(1001)

	id, idOK := value.(UserID)
	number, numberOK := value.(int64)

	fmt.Printf("UserID: %d, ok=%t\n", id, idOK)
	fmt.Printf("int64: %d, ok=%t\n", number, numberOK)
}
```

输出：

```text
UserID: 1001, ok=true
int64: 0, ok=false
```

如果声明的是类型别名，情况不同：

```go
type ID = int64
```

`ID` 和 `int64` 是同一个类型，不需要额外转换。

### 类型断言只能用于接口值

下面的代码无法编译：

```go
text := "Go"

// value := text.(string)
// invalid operation: text is not an interface
```

因为 `text` 已经是明确的 `string`，没有接口中的动态类型需要检查。

如果目的是在数值类型之间转换，应使用类型转换：

```go
number := 10
result := int64(number)
```

如果目的是检查任意接口值，才使用类型断言：

```go
var value any = 10
number, ok := value.(int)
```

### 类型断言和类型转换的区别

两者都使用圆括号，但解决的问题不同。

| 对比项 | 类型断言 | 类型转换 |
| --- | --- | --- |
| 写法 | `value.(T)` | `T(value)` |
| 数据来源 | 接口值 | 可转换的普通值 |
| 核心作用 | 检查动态类型并取值 | 把值转换成另一种类型 |
| 失败方式 | `ok=false` 或 panic | 通常在编译期报错 |
| 是否产生新表示 | 一般只是取出接口中的值 | 可能改变值的表示 |

示例：

```go
package main

import "fmt"

func main() {
	var source any = int32(65)

	number, ok := source.(int32)
	if !ok {
		fmt.Println("断言失败")
		return
	}

	fmt.Println(number)
	fmt.Println(int64(number))
	fmt.Println(string(number))
}
```

输出：

```text
65
65
A
```

第一步用断言取出 `int32`，后两步才是类型转换。

### 多种类型分支：type switch

接口值可能有多种动态类型时，连续写多组断言会很啰嗦：

```go
if text, ok := value.(string); ok {
	// ...
} else if number, ok := value.(int); ok {
	// ...
} else if enabled, ok := value.(bool); ok {
	// ...
}
```

这类场景更适合 `type switch`：

```go
switch current := value.(type) {
case string:
	// current 是 string
case int:
	// current 是 int
case bool:
	// current 是 bool
default:
	// 其他类型
}
```

`value.(type)` 只能写在 type switch 中，不能单独赋值。

完整示例：

```go
package main

import "fmt"

func describe(value any) {
	switch current := value.(type) {
	case nil:
		fmt.Println("nil")
	case string:
		fmt.Printf("字符串：%q，长度：%d\n", current, len(current))
	case int:
		fmt.Printf("整数：%d，平方：%d\n", current, current*current)
	case bool:
		fmt.Printf("布尔值：%t\n", current)
	case []int:
		fmt.Printf("整数切片：%v，元素数量：%d\n", current, len(current))
	default:
		fmt.Printf("未处理类型：%T，值：%v\n", current, current)
	}
}

func main() {
	describe(nil)
	describe("Go")
	describe(12)
	describe(true)
	describe([]int{1, 2, 3})
	describe(3.14)
}
```

输出：

```text
nil
字符串："Go"，长度：2
整数：12，平方：144
布尔值：true
整数切片：[1 2 3]，元素数量：3
未处理类型：float64，值：3.14
```

#### 一个 case 匹配多个类型

多个类型可以放在同一个 case：

```go
switch current := value.(type) {
case int, int32, int64:
	fmt.Printf("整数类型：%T，值：%v\n", current, current)
}
```

这里要注意：

```text
当一个 case 只写一种类型时，current 在分支内就是该类型。
当一个 case 写多种类型时，current 仍然保持 switch 表达式的接口类型。
```

所以多类型 case 里不能直接执行只属于某个具体类型的运算：

```go
switch current := value.(type) {
case int, int64:
	// fmt.Println(current + 1) // 编译错误，current 仍是接口类型
	fmt.Printf("%T %v\n", current, current)
}
```

需要分别计算时，应拆成不同 case。

### 断言成另一个接口

类型断言的目标不一定是结构体、指针或基础类型，也可以是接口。

这种写法常用于检查一个对象是否额外支持某种能力。

```go
package main

import "fmt"

type Sender interface {
	Send(message string) error
}

type Closer interface {
	Close() error
}

type ConsoleSender struct{}

func (ConsoleSender) Send(message string) error {
	fmt.Println("发送：", message)
	return nil
}

func (ConsoleSender) Close() error {
	fmt.Println("释放发送器")
	return nil
}

func sendMessage(sender Sender, message string) error {
	if err := sender.Send(message); err != nil {
		return err
	}

	if closer, ok := sender.(Closer); ok {
		return closer.Close()
	}

	return nil
}

func main() {
	if err := sendMessage(ConsoleSender{}, "订单创建成功"); err != nil {
		fmt.Println("执行失败：", err)
	}
}
```

输出：

```text
发送： 订单创建成功
释放发送器
```

`sendMessage` 的基本要求只有 `Sender`。关闭能力是可选的，运行时通过断言检查。

标准库中也有很多类似用法，例如检查对象是否实现 `io.Closer`、`fmt.Stringer` 或 `http.Flusher`。

这种模式可以概括为：

```text
先依赖最小接口，再按需探测额外能力。
```

### nil 接口与含 nil 指针的接口

类型断言遇到 `nil` 时，最容易出现两种看起来相似、实际不同的情况。

#### 接口本身是 nil

```go
package main

import "fmt"

func main() {
	var value any

	text, ok := value.(string)
	fmt.Println(value == nil)
	fmt.Printf("text=%q, ok=%t\n", text, ok)
}
```

输出：

```text
true
text="", ok=false
```

此时接口没有动态类型，也没有动态值，断言任何具体类型都会失败。

#### 接口里装着 nil 指针

```go
package main

import "fmt"

type User struct {
	Name string
}

func main() {
	var user *User
	var value any = user

	result, ok := value.(*User)

	fmt.Println(value == nil)
	fmt.Println(ok)
	fmt.Println(result == nil)
}
```

输出：

```text
false
true
true
```

接口中存在动态类型 `*User`，所以接口不等于 `nil`，断言也会成功；只是取出的指针值仍然是 `nil`。

因此，断言指针后还可能需要再检查一次：

```go
user, ok := value.(*User)
if !ok || user == nil {
	return
}

fmt.Println(user.Name)
```

只检查 `ok` 就访问字段，仍然可能触发空指针 panic。

### 实战一：安全读取动态配置

配置中心、插件参数和通用元数据经常使用 `map[string]any`。直接在业务代码里反复断言，容易出现重复逻辑和模糊错误。

可以把断言集中到配置类型中：

```go
package main

import (
	"fmt"
	"time"
)

type Config map[string]any

func (c Config) String(key string) (string, error) {
	value, exists := c[key]
	if !exists {
		return "", fmt.Errorf("配置 %q 不存在", key)
	}

	result, ok := value.(string)
	if !ok {
		return "", fmt.Errorf("配置 %q 需要 string，实际为 %T", key, value)
	}

	return result, nil
}

func (c Config) Duration(key string) (time.Duration, error) {
	text, err := c.String(key)
	if err != nil {
		return 0, err
	}

	duration, err := time.ParseDuration(text)
	if err != nil {
		return 0, fmt.Errorf("解析配置 %q: %w", key, err)
	}

	return duration, nil
}

func main() {
	config := Config{
		"address": "127.0.0.1:8080",
		"timeout": "3s",
		"retries": 3,
	}

	address, err := config.String("address")
	if err != nil {
		fmt.Println(err)
		return
	}

	timeout, err := config.Duration("timeout")
	if err != nil {
		fmt.Println(err)
		return
	}

	_, retriesErr := config.String("retries")

	fmt.Println("监听地址：", address)
	fmt.Println("超时时间：", timeout)
	fmt.Println("读取 retries：", retriesErr)
}
```

输出：

```text
监听地址： 127.0.0.1:8080
超时时间： 3s
读取 retries： 配置 "retries" 需要 string，实际为 int
```

这类封装的价值不只是少写几次断言，还包括：

* 缺少键和类型错误能明确区分
* 错误信息集中统一
* 业务代码不再直接依赖动态数据结构
* 后续切换到强类型配置更容易

不过，字段固定的配置仍然优先使用结构体：

```go
type AppConfig struct {
	Address string
	Timeout time.Duration
	Retries int
}
```

`map[string]any` 更适合结构确实会变化的边界数据，不适合替代所有结构体。

### 实战二：处理 JSON 动态数据

JSON 解码到 `any` 后，常见映射关系如下：

| JSON 类型 | 默认 Go 动态类型 |
| --- | --- |
| object | `map[string]any` |
| array | `[]any` |
| string | `string` |
| number | `float64` |
| boolean | `bool` |
| null | `nil` |

下面是一段完整示例：

```go
package main

import (
	"encoding/json"
	"fmt"
)

func main() {
	raw := []byte(`{
		"name": "键盘",
		"price": 299.5,
		"stock": 20,
		"tags": ["数码", "外设"]
	}`)

	var data any
	if err := json.Unmarshal(raw, &data); err != nil {
		fmt.Println("解析失败：", err)
		return
	}

	object, ok := data.(map[string]any)
	if !ok {
		fmt.Printf("根节点不是对象：%T\n", data)
		return
	}

	name, nameOK := object["name"].(string)
	price, priceOK := object["price"].(float64)
	stock, stockOK := object["stock"].(float64)
	tags, tagsOK := object["tags"].([]any)

	if !nameOK || !priceOK || !stockOK || !tagsOK {
		fmt.Println("字段类型不符合预期")
		return
	}

	fmt.Printf("商品：%s，价格：%.1f，库存：%d\n", name, price, int(stock))
	for index, item := range tags {
		tag, ok := item.(string)
		if !ok {
			fmt.Printf("标签 %d 不是字符串\n", index)
			continue
		}
		fmt.Println("标签：", tag)
	}
}
```

输出：

```text
商品：键盘，价格：299.5，库存：20
标签： 数码
标签： 外设
```

这里最常见的错误是：

```go
stock, ok := object["stock"].(int)
```

使用 `json.Unmarshal` 解码到 `any` 时，JSON 数字默认是 `float64`，即使原文写的是 `20`，断言成 `int` 也会失败。

需要保留数字文本时，可以使用 `json.Decoder.UseNumber`：

```go
package main

import (
	"encoding/json"
	"fmt"
	"strings"
)

func main() {
	decoder := json.NewDecoder(strings.NewReader(`{"order_id": 9007199254740993}`))
	decoder.UseNumber()

	var data map[string]any
	if err := decoder.Decode(&data); err != nil {
		fmt.Println(err)
		return
	}

	number, ok := data["order_id"].(json.Number)
	if !ok {
		fmt.Printf("order_id 类型错误：%T\n", data["order_id"])
		return
	}

	orderID, err := number.Int64()
	if err != nil {
		fmt.Println("order_id 不是 int64：", err)
		return
	}

	fmt.Println(orderID)
}
```

输出：

```text
9007199254740993
```

如果 JSON 结构稳定，直接解码到结构体通常更可靠：

```go
type Product struct {
	Name  string   `json:"name"`
	Price float64  `json:"price"`
	Stock int      `json:"stock"`
	Tags  []string `json:"tags"`
}
```

类型断言适合动态结构，不该成为绕过强类型模型的固定套路。

### 实战三：从 context 中读取业务值

`context.Context` 的 `Value` 方法返回 `any`，读取时通常需要类型断言。

```go
package main

import (
	"context"
	"fmt"
)

type contextKey string

const userKey contextKey = "current-user"

type User struct {
	ID   int64
	Name string
}

func withUser(ctx context.Context, user *User) context.Context {
	return context.WithValue(ctx, userKey, user)
}

func userFromContext(ctx context.Context) (*User, bool) {
	user, ok := ctx.Value(userKey).(*User)
	return user, ok && user != nil
}

func handleRequest(ctx context.Context) error {
	user, ok := userFromContext(ctx)
	if !ok {
		return fmt.Errorf("上下文中缺少当前用户")
	}

	fmt.Printf("当前用户：%d %s\n", user.ID, user.Name)
	return nil
}

func main() {
	ctx := withUser(context.Background(), &User{ID: 1001, Name: "李四"})

	if err := handleRequest(ctx); err != nil {
		fmt.Println(err)
	}
}
```

输出：

```text
当前用户：1001 李四
```

这里没有直接使用字符串作为 key，而是定义了包内专用类型：

```go
type contextKey string
```

这样可以降低不同包使用相同字符串导致键冲突的风险。

同时，断言逻辑被收进 `userFromContext`，业务函数不需要重复处理 `any`。

`context.Value` 适合保存请求范围内的附加信息，例如请求 ID、认证结果和链路信息，不适合用来传递普通函数参数或可选配置。

### 实战四：错误类型判断与 errors.As

自定义错误经常携带状态码、业务码等额外信息。

```go
package main

import (
	"errors"
	"fmt"
)

type ValidationError struct {
	Field   string
	Message string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("字段 %s：%s", e.Field, e.Message)
}

func validateName(name string) error {
	if name == "" {
		return &ValidationError{Field: "name", Message: "不能为空"}
	}
	return nil
}

func createUser(name string) error {
	if err := validateName(name); err != nil {
		return fmt.Errorf("创建用户失败: %w", err)
	}
	return nil
}

func main() {
	err := createUser("")
	if err == nil {
		return
	}

	_, directOK := err.(*ValidationError)
	fmt.Println("直接断言：", directOK)

	var validationErr *ValidationError
	if errors.As(err, &validationErr) {
		fmt.Printf("参数错误：field=%s, message=%s\n", validationErr.Field, validationErr.Message)
		return
	}

	fmt.Println(err)
}
```

输出：

```text
直接断言： false
参数错误：field=name, message=不能为空
```

直接类型断言只检查当前 `error` 接口中装着的动态类型。`createUser` 使用 `%w` 包装错误后，最外层已经不是 `*ValidationError`，所以直接断言失败。

`errors.As` 会沿着错误链查找匹配类型，适合处理可能被包装的错误。

选择规则可以压缩成两句话：

```text
只检查当前接口值：使用类型断言。
检查整条错误链：使用 errors.As。
```

判断某个固定错误值是否出现在错误链中，则使用 `errors.Is`。

### 实战五：用 type switch 处理事件

事件处理器经常需要接收多种事件，每种事件携带不同字段。

下面的 demo 展示一套完整的事件分发逻辑：

```go
package main

import (
	"fmt"
	"time"
)

type UserCreated struct {
	UserID int64
	Name   string
}

type OrderPaid struct {
	OrderID int64
	Amount  int64
}

type StockWarning struct {
	SKU       string
	Remaining int
}

func handleEvent(event any) error {
	switch current := event.(type) {
	case UserCreated:
		fmt.Printf("创建欢迎任务：user_id=%d, name=%s\n", current.UserID, current.Name)
	case *OrderPaid:
		if current == nil {
			return fmt.Errorf("OrderPaid 事件不能为空")
		}
		fmt.Printf("记录支付流水：order_id=%d, amount=%d\n", current.OrderID, current.Amount)
	case StockWarning:
		fmt.Printf("发送库存告警：sku=%s, remaining=%d\n", current.SKU, current.Remaining)
	case time.Time:
		fmt.Println("收到定时信号：", current.Format(time.DateTime))
	case nil:
		return fmt.Errorf("事件不能为空")
	default:
		return fmt.Errorf("不支持的事件类型 %T", current)
	}

	return nil
}

func main() {
	events := []any{
		UserCreated{UserID: 1001, Name: "王五"},
		&OrderPaid{OrderID: 2001, Amount: 19900},
		StockWarning{SKU: "KB-87", Remaining: 3},
		"unknown-event",
	}

	for _, event := range events {
		if err := handleEvent(event); err != nil {
			fmt.Println("处理失败：", err)
		}
	}
}
```

输出：

```text
创建欢迎任务：user_id=1001, name=王五
记录支付流水：order_id=2001, amount=19900
发送库存告警：sku=KB-87, remaining=3
处理失败： 不支持的事件类型 string
```

这种写法适合事件类型数量较少、分支集中、处理逻辑简单的场景。

如果事件持续增加，巨大的 type switch 会变成维护负担。更合适的设计可能是让事件实现统一接口，或者使用注册表把事件类型映射到处理器。

### 单返回值断言什么时候可以用

comma-ok 更稳妥，但单返回值写法并非完全不能使用。

例如程序内部已经通过固定流程保证类型：

```go
type tokenKind int

const stringToken tokenKind = iota

type token struct {
	kind  tokenKind
	value any
}

func readString(t token) string {
	if t.kind != stringToken {
		panic("readString 收到非字符串 token")
	}

	return t.value.(string)
}
```

这里的类型不匹配意味着程序内部约束被破坏，属于代码缺陷。让程序尽早失败，反而有助于暴露错误。

不过，一旦值来自外部输入或跨模块边界，就应返回明确错误，而不是依赖 panic。

### 类型断言、反射和泛型怎么选

这三种工具都会接触“多种类型”，但适用场景不同。

#### 类型断言

适合运行时处理有限、已知的类型集合：

```go
switch value := input.(type) {
case string:
	// ...
case int:
	// ...
}
```

典型场景包括：

* 从 `any` 中取值
* 探测额外接口能力
* 处理少量消息或事件类型
* 读取 `context.Value`

#### 泛型

适合一套算法在编译期支持多种类型：

```go
func First[T any](values []T) (T, bool) {
	if len(values) == 0 {
		var zero T
		return zero, false
	}
	return values[0], true
}
```

调用 `First([]int{1, 2})` 后，返回值直接就是 `int`，不需要类型断言。

如果类型在编译时已经明确，只是想消除重复代码，泛型通常比 `any` 加断言更合适。

#### 反射

适合类型集合无法提前列出，还需要读取字段、标签、方法等结构信息的场景，例如序列化、ORM 和依赖注入框架。

普通业务代码只有几个已知分支时，type switch 通常更直接，也更容易读懂。

| 需求 | 更合适的工具 |
| --- | --- |
| 从接口中取出已知类型 | 类型断言 |
| 检查对象是否有额外能力 | 断言到接口 |
| 同一算法支持多种类型 | 泛型 |
| 遍历未知结构的字段和标签 | 反射 |
| 普通数值类型互转 | 类型转换 |

### 常见错误

#### 忽略 ok

不推荐：

```go
name, _ := value.(string)
```

断言失败和真实空字符串会被混在一起，数据问题被悄悄吞掉。

更清楚的写法：

```go
name, ok := value.(string)
if !ok {
	return fmt.Errorf("name 需要 string，实际为 %T", value)
}
```

#### 为了省事到处使用 any

不推荐：

```go
func CreateUser(data map[string]any) error
```

如果字段结构固定，更适合定义明确模型：

```go
type CreateUserRequest struct {
	Name string
	Age  int
}

func CreateUser(request CreateUserRequest) error
```

强类型结构可以让大量错误提前到编译期暴露，也能让编辑器提供字段补全和重构支持。

#### 把断言当成转换

错误思路：

```go
var value any = int32(10)

// number, ok := value.(int64)
```

正确流程：

```go
number32, ok := value.(int32)
if !ok {
	return
}

number64 := int64(number32)
```

#### 忘记指针和值的差别

接口里装的是 `User`，就断言 `User`。

接口里装的是 `*User`，就断言 `*User`。

这件事还会受到方法接收者影响：

```go
type Saver interface {
	Save() error
}

type Document struct{}

func (*Document) Save() error {
	return nil
}
```

这里只有 `*Document` 实现了 `Saver`，`Document` 没有实现。

#### JSON 数字默认断言成 int

解码到 `any` 后，普通 JSON 数字默认是 `float64`。

处理整数有三种常见方式：

* 结构固定时，直接解码到带 `int` 或 `int64` 字段的结构体
* 动态结构需要精确整数时，使用 `Decoder.UseNumber`
* 确认范围和小数部分后，再把 `float64` 转成整数

不能只靠 `int(number)` 粗暴转换，否则小数和越界问题可能被掩盖。

#### 在错误链上只做直接断言

错误经过 `%w` 包装后，直接断言通常只能看到最外层类型。

```go
var target *ValidationError
if errors.As(err, &target) {
	// 找到了错误链中的 ValidationError
}
```

错误链场景优先考虑 `errors.Is` 和 `errors.As`。

### 工程实践建议

#### 默认使用 comma-ok

外部输入、共享上下文和动态数据的类型都有可能不符合预期，安全断言应作为默认选择：

```go
value, ok := source.(TargetType)
if !ok {
	return fmt.Errorf("类型错误：期望 %T", value)
}
```

实际错误信息通常还应输出源值的类型：

```go
return fmt.Errorf("字段 name 需要 string，实际为 %T", source)
```

#### 多分支使用 type switch

同一个接口值要判断三种以上类型时，type switch 通常比连续的 if-else 更清楚。

每个分支都可以直接得到对应的具体类型，字段和方法调用也更直观。

#### 把动态边界收窄

动态数据最好只停留在系统边界：

```text
JSON / 配置 / 插件 / context
          ↓
    校验和类型断言
          ↓
      强类型业务模型
```

越早把 `any` 转成明确结构，后续业务代码越简单。

#### 错误信息写出实际类型

`%T` 对排查类型断言失败很有帮助：

```go
return fmt.Errorf("期望 []string，实际为 %T", value)
```

只有“类型错误”四个字，很难判断接口中到底装了什么。

#### 不要用 panic 处理正常输入错误

请求字段格式不对、配置缺失、JSON 类型错误都属于可预期问题，应返回错误并交给上层处理。

panic 更适合无法继续运行的内部不变量破坏，不适合普通校验流程。

### 常见问题

#### any 和 interface{} 会影响断言结果吗

不会。

`any` 是 `interface{}` 的别名：

```go
type any = interface{}
```

下面两种声明完全等价：

```go
var first any = "Go"
var second interface{} = "Go"
```

都可以使用：

```go
text, ok := first.(string)
```

#### 断言成功后会复制值吗

断言结果遵循普通赋值规则。

结构体等值类型赋值后得到值副本；指针、slice、map、channel 等值复制的是对应描述值或引用语义载体，不会因为类型断言自动执行深拷贝。

类型断言的职责只是检查动态类型并取得值，不负责克隆对象。

#### type switch 可以写 nil 吗

可以。

```go
switch value := input.(type) {
case nil:
	fmt.Println("接口为 nil")
case string:
	fmt.Println(value)
}
```

`case nil` 匹配的是没有动态类型的 nil 接口。

如果接口里装着 `(*User)(nil)`，它会匹配 `case *User`，不会匹配 `case nil`。

#### type switch 的 case 顺序重要吗

对不同具体类型通常不重要，每个类型最多只能出现在一个 case 中。

但目标为接口时，一个动态值可能同时实现多个接口，type switch 只执行第一个匹配的 case。因此，更具体的能力接口通常放在更前面。

```go
switch value := source.(type) {
case interface {
	Read([]byte) (int, error)
	Close() error
}:
	// 同时支持读取和关闭
case interface {
	Read([]byte) (int, error)
}:
	// 只要求读取
}
```

#### 类型断言能判断底层类型相同的自定义类型吗

不能自动匹配。

```go
type UserID int64

var value any = UserID(1)
_, ok := value.(int64) // false
```

动态类型是 `UserID`，不是 `int64`。先断言为 `UserID`，再显式转换即可。

### 总结

类型断言的核心可以压缩成几句话：

```text
类型断言只作用于接口值。
value.(T) 失败会 panic。
value, ok := source.(T) 失败时返回零值和 false。
具体类型断言要求动态类型精确匹配。
User 和 *User、int 和 int64 都不会自动互转。
多个类型分支适合使用 type switch。
目标类型也可以是接口，用于探测额外能力。
nil 接口与装着 nil 指针的接口不是一回事。
包装错误的类型检查优先使用 errors.As。
```

日常选择可以按下面这张表判断：

| 场景 | 常见写法 |
| --- | --- |
| 安全取出具体类型 | `value, ok := source.(T)` |
| 类型已被内部约束严格保证 | `value := source.(T)` |
| 判断多种动态类型 | `switch value := source.(type)` |
| 检查额外能力 | `closer, ok := source.(io.Closer)` |
| 处理包装错误 | `errors.As(err, &target)` |
| 数值类型互转 | `int64(number)` |
| 编译期复用同一算法 | 泛型 |
| 遍历未知类型结构 | 反射 |

类型断言不是给静态类型系统开后门，而是用来处理接口边界。把断言留在动态数据进入系统的位置，尽早转成明确类型，后面的业务逻辑会更稳定，也更容易维护。
