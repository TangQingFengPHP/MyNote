### 简介

`interface` 是 Go 里非常重要的类型。

它不保存字段，也不写具体逻辑。

它只定义一组方法。

比如：

```go
type Writer interface {
	Write(data []byte) (int, error)
}
```

这段代码表达的是：

```text
只要某个类型有 Write(data []byte) (int, error) 方法，它就可以当成 Writer 使用。
```

一句话概括：

```text
interface 定义的是行为约定，不关心具体是谁来做。
```

结构体更像“数据长什么样”。

接口更像“能做什么事”。

例如：

```text
User struct：描述用户有哪些字段
UserRepository interface：描述用户仓储要提供哪些能力
Payment interface：描述支付方式要提供哪些能力
```

### 第一个 interface 示例

先看一个最小例子。

```go
package main

import "fmt"

type Speaker interface {
	Speak() string
}

type Dog struct {
	Name string
}

func (d Dog) Speak() string {
	return d.Name + "：汪汪"
}

type Cat struct {
	Name string
}

func (c Cat) Speak() string {
	return c.Name + "：喵喵"
}

func Say(s Speaker) {
	fmt.Println(s.Speak())
}

func main() {
	Say(Dog{Name: "旺财"})
	Say(Cat{Name: "小花"})
}
```

输出：

```text
旺财：汪汪
小花：喵喵
```

`Say` 函数只依赖 `Speaker` 接口。

只要传进来的值有 `Speak() string` 方法，就能使用。

### Go 接口是隐式实现

Go 不需要写 `implements`。

只要方法匹配，就自动实现接口。

```go
type Speaker interface {
	Speak() string
}

type Dog struct{}

func (d Dog) Speak() string {
	return "wang"
}
```

`Dog` 没有声明“实现了 Speaker”。

但它确实有 `Speak() string` 方法，所以它就是 `Speaker`。

这种设计的好处是：

```text
实现类型不需要知道接口存在。
接口也不需要提前绑定具体实现。
```

这就是 Go 接口很灵活的地方。

### interface 只关心方法

接口不关心结构体字段。

```go
type User struct {
	ID   int
	Name string
}
```

即使 `User` 有 `ID` 和 `Name` 字段，也不能因为字段相同就实现接口。

接口只看方法。

```go
type Named interface {
	GetName() string
}

func (u User) GetName() string {
	return u.Name
}
```

有了 `GetName() string` 方法，`User` 才能当成 `Named` 使用。

完整示例：

```go
package main

import "fmt"

type Named interface {
	GetName() string
}

type User struct {
	ID   int
	Name string
}

func (u User) GetName() string {
	return u.Name
}

func PrintName(v Named) {
	fmt.Println(v.GetName())
}

func main() {
	PrintName(User{ID: 1, Name: "张三"})
}
```

输出：

```text
张三
```

### 接口变量里装的是什么

接口变量可以理解成两部分：

```text
动态类型
动态值
```

示例：

```go
package main

import "fmt"

type Speaker interface {
	Speak() string
}

type Dog struct {
	Name string
}

func (d Dog) Speak() string {
	return d.Name + "：汪汪"
}

func main() {
	var s Speaker

	fmt.Println(s == nil)

	s = Dog{Name: "旺财"}

	fmt.Println(s == nil)
	fmt.Printf("%T\n", s)
	fmt.Println(s.Speak())
}
```

输出：

```text
true
false
main.Dog
旺财：汪汪
```

刚声明的接口变量是 nil。

赋值后，接口变量里有了动态类型 `Dog` 和动态值 `{Name:"旺财"}`。

### 接口的 nil 陷阱

接口最经典的坑是：

```text
接口变量本身不是 nil，但里面的动态值是 nil。
```

看一个例子：

```go
package main

import "fmt"

type MyError struct {
	Message string
}

func (e *MyError) Error() string {
	if e == nil {
		return "<nil>"
	}
	return e.Message
}

func returnError() error {
	var err *MyError = nil
	return err
}

func main() {
	err := returnError()

	fmt.Println(err == nil)
	fmt.Printf("%T\n", err)
}
```

输出：

```text
false
*main.MyError
```

`returnError` 返回的是 `error` 接口。

虽然 `*MyError` 的值是 nil，但接口里已经有了动态类型 `*MyError`。

所以接口本身不是 nil。

正确写法通常是直接返回 nil：

```go
func returnError(ok bool) error {
	if ok {
		return nil
	}

	return &MyError{Message: "failed"}
}
```

### 空接口 interface{} 和 any

空接口没有任何方法。

```go
interface{}
```

因为没有方法要求，所以所有类型都实现了空接口。

Go 1.18 之后，`any` 是 `interface{}` 的别名。

```go
type any = interface{}
```

所以这两种写法等价：

```go
func Print(v interface{}) {}
func Print(v any) {}
```

示例：

```go
package main

import "fmt"

func PrintAny(v any) {
	fmt.Printf("type=%T value=%v\n", v, v)
}

func main() {
	PrintAny(100)
	PrintAny("hello")
	PrintAny(true)
	PrintAny([]int{1, 2, 3})
}
```

输出：

```text
type=int value=100
type=string value=hello
type=bool value=true
type=[]int value=[1 2 3]
```

空接口很灵活，但也会丢失具体类型信息。

能用明确类型时，不要把所有参数都写成 `any`。

### 类型断言

接口变量里有动态类型。

如果需要把接口值取回具体类型，可以使用类型断言。

```go
value, ok := x.(T)
```

示例：

```go
package main

import "fmt"

func main() {
	var v any = "hello"

	s, ok := v.(string)
	if ok {
		fmt.Println(s)
	}

	n, ok := v.(int)
	if !ok {
		fmt.Println("not int")
		return
	}

	fmt.Println(n)
}
```

输出：

```text
hello
not int
```

不带 `ok` 的写法如果失败，会直接 panic。

```go
var v any = 100
s := v.(string)
fmt.Println(s)
```

运行会报错：

```text
panic: interface conversion: interface {} is int, not string
```

业务代码里更常用安全写法：

```go
s, ok := v.(string)
```

### 类型选择 type switch

如果接口值可能是多种类型，可以用 type switch。

```go
package main

import "fmt"

func Describe(v any) {
	switch value := v.(type) {
	case nil:
		fmt.Println("nil")
	case int:
		fmt.Println("int:", value)
	case string:
		fmt.Println("string:", value)
	case bool:
		fmt.Println("bool:", value)
	case []int:
		fmt.Println("[]int:", value)
	default:
		fmt.Printf("unknown: %T\n", value)
	}
}

func main() {
	Describe(100)
	Describe("go")
	Describe(true)
	Describe([]int{1, 2, 3})
	Describe(3.14)
}
```

输出：

```text
int: 100
string: go
bool: true
[]int: [1 2 3]
unknown: float64
```

type switch 适合处理“入参类型不确定”的场景。

但如果项目里到处都是 type switch，通常说明抽象边界需要重新设计。

### 值接收者和指针接收者对接口的影响

方法接收者会影响接口实现。

先看值接收者：

```go
package main

import "fmt"

type Printer interface {
	Print()
}

type User struct {
	Name string
}

func (u User) Print() {
	fmt.Println(u.Name)
}

func main() {
	var p Printer

	p = User{Name: "张三"}
	p.Print()

	p = &User{Name: "李四"}
	p.Print()
}
```

输出：

```text
张三
李四
```

值接收者方法属于 `User`，也属于 `*User` 的方法集。

所以 `User` 和 `*User` 都能赋值给 `Printer`。

再看指针接收者：

```go
package main

import "fmt"

type Renamer interface {
	Rename(name string)
}

type User struct {
	Name string
}

func (u *User) Rename(name string) {
	u.Name = name
}

func main() {
	var r Renamer

	// r = User{Name: "张三"} // 编译失败
	r = &User{Name: "张三"}

	r.Rename("李四")

	fmt.Println(r)
}
```

输出：

```text
&{李四}
```

指针接收者方法只属于 `*User` 的方法集。

所以只有 `*User` 能赋值给 `Renamer`。

### 接口嵌入和组合

接口可以组合其他接口。

标准库里很常见。

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}

type ReadWriter interface {
	Reader
	Writer
}
```

`ReadWriter` 等价于：

```go
type ReadWriter interface {
	Read(p []byte) (n int, err error)
	Write(p []byte) (n int, err error)
}
```

组合接口的好处是可以把能力拆小，再按需要组合。

### 标准库里的接口：io.Reader

`io.Reader` 是 Go 里最经典的小接口之一。

它只有一个方法：

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

很多类型都实现了 `io.Reader`：

* 文件
* 网络连接
* 字符串 Reader
* bytes Buffer

所以一个函数只要接收 `io.Reader`，就能处理很多数据来源。

示例：

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"strings"
)

func PrintAll(r io.Reader) {
	data, err := io.ReadAll(r)
	if err != nil {
		fmt.Println("read failed:", err)
		return
	}

	fmt.Println(string(data))
}

func main() {
	PrintAll(strings.NewReader("from string"))

	var buf bytes.Buffer
	buf.WriteString("from buffer")
	PrintAll(&buf)
}
```

输出：

```text
from string
from buffer
```

`PrintAll` 不关心数据来自字符串还是内存缓冲。

它只关心一件事：

```text
传进来的对象能不能 Read。
```

### 标准库里的接口：error

`error` 也是接口。

定义大致如下：

```go
type error interface {
	Error() string
}
```

只要某个类型实现了 `Error() string`，就可以当成 `error` 返回。

示例：

```go
package main

import "fmt"

type ValidationError struct {
	Field   string
	Message string
}

func (e ValidationError) Error() string {
	return e.Field + ": " + e.Message
}

func ValidateName(name string) error {
	if name == "" {
		return ValidationError{
			Field:   "name",
			Message: "required",
		}
	}

	return nil
}

func main() {
	err := ValidateName("")
	if err != nil {
		fmt.Println(err)
	}
}
```

输出：

```text
name: required
```

### 编译期检查是否实现接口

项目里常见这种写法：

```go
var _ UserRepository = (*MemoryUserRepository)(nil)
```

它的作用是：

```text
在编译期检查 MemoryUserRepository 是否实现了 UserRepository。
```

示例：

```go
type UserRepository interface {
	FindByID(id int64) (*User, bool)
	Save(user *User)
}

type MemoryUserRepository struct {
	data map[int64]*User
}

var _ UserRepository = (*MemoryUserRepository)(nil)
```

如果 `MemoryUserRepository` 少实现一个方法，代码会编译失败。

这种写法不生成运行时代码，只是让编译器帮忙检查。

### 实战 Demo：支付方式解耦

支付场景很适合讲接口。

业务只关心“能不能支付”，不关心具体是支付宝、微信还是银行卡。

```go
package main

import "fmt"

type Payment interface {
	Pay(amount float64) error
	Name() string
}

type AliPay struct {
	Account string
}

func (a AliPay) Pay(amount float64) error {
	fmt.Printf("支付宝账号 %s 支付 %.2f 元\n", a.Account, amount)
	return nil
}

func (a AliPay) Name() string {
	return "支付宝"
}

type WechatPay struct {
	OpenID string
}

func (w WechatPay) Pay(amount float64) error {
	fmt.Printf("微信 OpenID %s 支付 %.2f 元\n", w.OpenID, amount)
	return nil
}

func (w WechatPay) Name() string {
	return "微信支付"
}

func Checkout(p Payment, amount float64) error {
	fmt.Println("使用", p.Name())
	return p.Pay(amount)
}

func main() {
	_ = Checkout(AliPay{Account: "ali_1001"}, 99.9)
	_ = Checkout(WechatPay{OpenID: "wx_2001"}, 199.5)
}
```

输出：

```text
使用 支付宝
支付宝账号 ali_1001 支付 99.90 元
使用 微信支付
微信 OpenID wx_2001 支付 199.50 元
```

新增一种支付方式时，只需要实现 `Payment` 接口。

`Checkout` 不需要知道新增类型的细节。

### 实战 Demo：通知服务

通知场景也很适合接口。

短信、邮件、站内信都可以抽象成一个 `Notifier`。

```go
package main

import "fmt"

type Notifier interface {
	Send(to string, message string) error
}

type EmailNotifier struct {
	From string
}

func (n EmailNotifier) Send(to string, message string) error {
	fmt.Printf("email from=%s to=%s message=%s\n", n.From, to, message)
	return nil
}

type SMSNotifier struct {
	Sign string
}

func (n SMSNotifier) Send(to string, message string) error {
	fmt.Printf("sms sign=%s to=%s message=%s\n", n.Sign, to, message)
	return nil
}

type RegisterService struct {
	notifier Notifier
}

func NewRegisterService(notifier Notifier) *RegisterService {
	return &RegisterService{
		notifier: notifier,
	}
}

func (s *RegisterService) Register(email string) error {
	return s.notifier.Send(email, "注册成功")
}

func main() {
	emailService := NewRegisterService(EmailNotifier{From: "noreply@example.com"})
	_ = emailService.Register("user@example.com")

	smsService := NewRegisterService(SMSNotifier{Sign: "系统通知"})
	_ = smsService.Register("13800000000")
}
```

输出：

```text
email from=noreply@example.com to=user@example.com message=注册成功
sms sign=系统通知 to=13800000000 message=注册成功
```

`RegisterService` 依赖的是 `Notifier`。

具体发邮件还是发短信，由外部传入。

### 实战 Demo：Repository 分层

接口常用于 Service 和 Repository 解耦。

下面用内存实现模拟数据库。

```go
package main

import (
	"errors"
	"fmt"
)

type User struct {
	ID   int64
	Name string
}

type UserRepository interface {
	FindByID(id int64) (*User, error)
	Save(user *User) error
}

type MemoryUserRepository struct {
	data map[int64]*User
}

var _ UserRepository = (*MemoryUserRepository)(nil)

func NewMemoryUserRepository() *MemoryUserRepository {
	return &MemoryUserRepository{
		data: make(map[int64]*User),
	}
}

func (r *MemoryUserRepository) FindByID(id int64) (*User, error) {
	user, ok := r.data[id]
	if !ok {
		return nil, errors.New("user not found")
	}

	return user, nil
}

func (r *MemoryUserRepository) Save(user *User) error {
	if user == nil {
		return errors.New("user is nil")
	}

	r.data[user.ID] = user
	return nil
}

type UserService struct {
	repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
	return &UserService{repo: repo}
}

func (s *UserService) Rename(id int64, name string) error {
	user, err := s.repo.FindByID(id)
	if err != nil {
		return err
	}

	user.Name = name
	return s.repo.Save(user)
}

func main() {
	repo := NewMemoryUserRepository()
	_ = repo.Save(&User{ID: 1, Name: "张三"})

	service := NewUserService(repo)
	_ = service.Rename(1, "李四")

	user, _ := repo.FindByID(1)
	fmt.Println(user)
}
```

输出：

```text
&{1 李四}
```

这个例子里：

* `UserService` 只依赖 `UserRepository`
* `MemoryUserRepository` 是其中一种实现
* 后面换成 MySQL、Redis、Mock，只要实现同样接口即可

### 实战 Demo：测试替身

接口还有一个非常常见的用途：替换外部依赖。

例如注册后要发通知。

正式环境用真实通知实现。

测试里可以用假的通知实现记录调用情况。

```go
package main

import "fmt"

type Sender interface {
	Send(to string, message string) error
}

type UserRegister struct {
	sender Sender
}

func NewUserRegister(sender Sender) *UserRegister {
	return &UserRegister{sender: sender}
}

func (r *UserRegister) Register(email string) error {
	return r.sender.Send(email, "welcome")
}

type FakeSender struct {
	Messages []string
}

func (s *FakeSender) Send(to string, message string) error {
	s.Messages = append(s.Messages, to+":"+message)
	return nil
}

func main() {
	fake := &FakeSender{}
	register := NewUserRegister(fake)

	_ = register.Register("user@example.com")

	fmt.Println(fake.Messages)
}
```

输出：

```text
[user@example.com:welcome]
```

这里没有真的发邮件。

但可以验证注册逻辑确实调用了发送接口。

### 泛型里的 interface

Go 1.18 之后，接口还可以作为泛型约束。

普通接口约束的是方法。

泛型约束还可以约束类型集合。

示例：

```go
package main

import "fmt"

type Integer interface {
	~int | ~int64 | ~uint
}

func Sum[T Integer](values []T) T {
	var total T

	for _, value := range values {
		total += value
	}

	return total
}

func main() {
	fmt.Println(Sum([]int{1, 2, 3}))
	fmt.Println(Sum([]int64{10, 20, 30}))
}
```

输出：

```text
6
60
```

这里的 `Integer` 不是普通业务接口，而是类型约束。

`~int | ~int64 | ~uint` 表示允许底层类型是这些整数类型的类型参与计算。

普通项目里，接口作为行为抽象更常见。

泛型约束适合写通用算法和工具函数。

### 什么时候适合定义 interface

适合定义接口的场景：

* 需要替换不同实现，例如 MySQL、Redis、内存实现
* 业务只关心行为，不关心具体类型
* 需要在测试里替换外部依赖
* 一个函数希望接收多种实现，例如 `io.Reader`
* 模块之间需要降低耦合
* 需要组合小能力，例如 `ReadWriter`

示例：

```go
type UserRepository interface {
	FindByID(id int64) (*User, error)
	Save(user *User) error
}
```

### 什么时候不适合定义 interface

不适合为了“看起来高级”提前抽象。

如果当前只有一个实现，业务变化也不明显，直接使用具体类型更简单。

不推荐：

```go
type UserServiceInterface interface {
	CreateUser(name string) (*User, error)
}
```

如果只有一个 `UserService`，也没有替换需求，这个接口可能只是增加文件和跳转成本。

Go 项目里常见建议是：

```text
接收接口，返回具体类型。
接口放在使用方更自然。
```

例如：

```go
func NewUserService(repo UserRepository) *UserService {
	return &UserService{repo: repo}
}
```

`NewUserService` 接收接口，因为它只关心仓储能力。

但它返回具体的 `*UserService`，因为调用方通常需要的是这个具体服务对象。

### 小接口更好组合

Go 标准库里很多接口都很小。

例如：

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}

type Closer interface {
	Close() error
}
```

需要组合时再组合：

```go
type ReadWriteCloser interface {
	Reader
	Writer
	Closer
}
```

小接口的好处是：

* 实现容易
* 复用更灵活
* 测试替身更容易写
* 调用方依赖更少

### 常见问题

#### interface 可以有字段吗

不能。

接口只能定义方法，不能定义字段。

```go
type UserLike interface {
	// Name string // 错误
	GetName() string
}
```

如果需要字段，使用 struct。

如果需要行为约定，使用 interface。

#### any 和 interface{} 有区别吗

没有本质区别。

`any` 是 `interface{}` 的别名。

```go
type any = interface{}
```

`any` 更短，也更适合表达“任意类型”。

#### 接口可以比较吗

接口可以和 nil 比较。

```go
var v any
fmt.Println(v == nil)
```

接口之间也可以比较，但有前提：

```text
动态类型必须可比较。
```

如果接口里装的是 slice、map、func，比较时会 panic。

示例：

```go
var a any = []int{1, 2}
var b any = []int{1, 2}

fmt.Println(a == b)
```

运行会 panic：

```text
panic: runtime error: comparing uncomparable type []int
```

#### 接口应该定义在实现方还是使用方

多数情况下，接口定义在使用方更自然。

例如 `UserService` 需要一个仓储能力：

```go
type UserRepository interface {
	FindByID(id int64) (*User, error)
}
```

这个接口可以放在 service 所在包。

具体的 MySQL、Redis、Memory 实现只负责提供方法。

这样接口不会被实现方提前设计得过大。

#### 接口一定能提升代码质量吗

不一定。

接口用对了可以解耦。

接口用早了会增加间接层。

判断标准很简单：

```text
是否真的存在多个实现？
是否真的需要在测试中替换？
调用方是否只需要一小组行为？
```

如果答案都是否定的，具体类型可能更清楚。

### 总结

`interface` 的核心可以压缩成几句话：

```text
interface 定义行为，不定义字段。
Go 接口是隐式实现，有方法就算实现。
接口变量包含动态类型和动态值。
空接口 interface{} 或 any 可以接收任意类型。
类型断言和 type switch 可以取回具体类型。
小接口更容易组合和复用。
```

日常选择可以按下面这张表判断：

| 场景 | 常见写法 |
| --- | --- |
| 定义行为约定 | `type Reader interface { Read(...) }` |
| 多种实现统一调用 | 函数参数接收接口 |
| 依赖注入 | struct 字段保存接口 |
| 测试替身 | fake/mock 实现接口 |
| 任意类型 | `any` |
| 取回具体类型 | `v, ok := x.(T)` |
| 多类型分支 | `switch v := x.(type)` |
| 组合能力 | 接口嵌入 |
| 编译期实现检查 | `var _ Interface = (*Impl)(nil)` |
| 泛型类型约束 | `type Number interface { ~int | ~float64 }` |

`interface` 不是万能盒子，也不是每个结构体都要配一个接口。它最适合表达“这里需要某种能力”。当调用方只关心行为，不关心具体实现时，接口就能让代码变得更灵活、更容易替换和测试。
