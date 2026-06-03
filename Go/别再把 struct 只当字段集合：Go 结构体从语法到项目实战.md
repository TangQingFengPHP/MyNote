### 简介

`struct` 是 Go 里最重要的复合类型之一。

它用来把多个字段组合成一个整体。

比如一个用户有 ID、名称、邮箱、年龄：

```go
type User struct {
	ID    int
	Name  string
	Email string
	Age   int
}
```

这段代码表达的是：

```text
User 是一个结构体类型，里面有 ID、Name、Email、Age 四个字段。
```

Go 没有传统意义上的 `class`，但很多业务模型都会用 `struct` 表达：

* 用户
* 商品
* 订单
* 配置
* 请求参数
* 响应结果
* 数据库实体

一句话概括：

```text
struct 用来组织一组相关字段，是 Go 项目里描述业务数据的核心工具。
```

### struct 的基本定义

结构体定义语法如下：

```go
type StructName struct {
	FieldName FieldType
}
```

示例：

```go
type Product struct {
	ID    int
	Name  string
	Price int
}
```

字段可以是任意类型：

```go
type Order struct {
	ID        int
	No        string
	Paid      bool
	Amount    float64
	Items     []OrderItem
	Extra     map[string]string
	CreatedAt int64
}

type OrderItem struct {
	ProductID int
	Quantity  int
}
```

这里的 `Order` 里既有基础类型，也有切片和 map，还包含另一个结构体 `OrderItem`。

### 最小可运行示例

```go
package main

import "fmt"

type User struct {
	ID   int
	Name string
	Age  int
}

func main() {
	u := User{
		ID:   1,
		Name: "张三",
		Age:  20,
	}

	fmt.Println(u.ID)
	fmt.Println(u.Name)
	fmt.Println(u.Age)
	fmt.Printf("%+v\n", u)
}
```

输出：

```text
1
张三
20
{ID:1 Name:张三 Age:20}
```

`fmt.Printf("%+v\n", u)` 会把字段名也打印出来，调试结构体时很常用。

### 结构体的零值

只声明结构体，不手动赋值时，每个字段都会是对应类型的零值。

```go
package main

import "fmt"

type User struct {
	ID     int
	Name   string
	Active bool
	Tags   []string
	Meta   map[string]string
}

func main() {
	var u User

	fmt.Printf("%+v\n", u)
	fmt.Println(u.Tags == nil)
	fmt.Println(u.Meta == nil)
}
```

输出：

```text
{ID:0 Name: Active:false Tags:[] Meta:map[]}
true
true
```

注意，`fmt` 打印 nil slice 和 nil map 时，看起来像 `[]`、`map[]`，但它们本身仍然是 `nil`。

结构体零值有一个好处：

```text
很多结构体不需要构造函数，声明出来就处于一个合法状态。
```

例如计数器：

```go
type Counter struct {
	value int
}

func (c *Counter) Inc() {
	c.value++
}
```

`Counter` 的零值就可以直接用。

### 创建结构体的几种方式

#### 先声明再赋值

```go
var u User
u.ID = 1
u.Name = "张三"
u.Age = 20
```

这种写法适合字段需要一步步计算出来的情况。

#### 按字段名初始化

```go
u := User{
	ID:   1,
	Name: "张三",
	Age:  20,
}
```

这是项目里最常见的写法。

好处是字段含义清楚，字段顺序调整后也不容易出错。

#### 只初始化部分字段

```go
u := User{
	Name: "张三",
}
```

未指定的字段会使用零值。

等价于：

```text
ID = 0
Name = "张三"
Age = 0
```

#### 按顺序初始化

```go
u := User{1, "张三", 20}
```

这种写法要求值的顺序和字段定义顺序完全一致。

字段少时能看懂，字段多了就很容易混乱。

不推荐在业务代码里大量使用。

#### 创建结构体指针

```go
u := &User{
	ID:   1,
	Name: "张三",
	Age:  20,
}
```

`u` 的类型是 `*User`。

完整示例：

```go
package main

import "fmt"

type User struct {
	ID   int
	Name string
}

func main() {
	u := &User{
		ID:   1,
		Name: "张三",
	}

	fmt.Printf("%T\n", u)
	fmt.Printf("%+v\n", u)
}
```

输出：

```text
*main.User
&{ID:1 Name:张三}
```

### new 创建结构体

结构体也可以用 `new` 创建。

```go
u := new(User)
```

它等价于创建一个零值 `User`，然后返回指针。

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

如果需要顺手设置字段，`&User{...}` 更直接：

```go
u := &User{
	ID:   1,
	Name: "张三",
}
```

如果只需要一个零值指针，`new(User)` 也可以。

### 访问和修改字段

结构体字段用点号访问。

```go
u := User{Name: "张三", Age: 20}

fmt.Println(u.Name)
u.Age = 21
```

结构体指针也用点号访问，Go 会自动解引用。

```go
u := &User{Name: "张三", Age: 20}

fmt.Println(u.Name)
u.Age = 21
```

不用写成：

```go
(*u).Age = 21
```

Go 会自动处理这层语法。

### 字段命名和可见性

Go 通过首字母大小写控制包外可见性。

```go
type User struct {
	ID       int
	Name     string
	password string
}
```

含义：

* `ID` 和 `Name` 首字母大写，包外可以访问
* `password` 首字母小写，只能在当前包内访问

这个规则不只适用于字段，也适用于类型、函数、方法、变量。

示例：

```go
type User struct {
	ID       int
	Name     string
	password string
}

func NewUser(id int, name string, password string) User {
	return User{
		ID:       id,
		Name:     name,
		password: password,
	}
}
```

包外代码可以拿到 `ID` 和 `Name`，但不能直接读取 `password`。

### struct 是值类型

结构体默认按值传递。

把结构体传给函数时，会复制一份。

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func rename(u User) {
	u.Name = "李四"
}

func main() {
	u := User{Name: "张三", Age: 20}

	rename(u)

	fmt.Println(u.Name)
}
```

输出：

```text
张三
```

`rename` 里修改的是副本，不会影响原来的 `u`。

如果需要修改原结构体，要传指针。

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func rename(u *User) {
	u.Name = "李四"
}

func main() {
	u := User{Name: "张三", Age: 20}

	rename(&u)

	fmt.Println(u.Name)
}
```

输出：

```text
李四
```

### 给 struct 定义方法

Go 的方法是带接收者的函数。

```go
func (receiver Type) MethodName() {
}
```

示例：

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func (u User) Info() string {
	return fmt.Sprintf("%s/%d", u.Name, u.Age)
}

func main() {
	u := User{Name: "张三", Age: 20}

	fmt.Println(u.Info())
}
```

输出：

```text
张三/20
```

这里的 `(u User)` 就是接收者。

可以理解成这个方法属于 `User` 类型。

### 值接收者和指针接收者

方法接收者有两种：

```go
func (u User) Method() {}
func (u *User) Method() {}
```

#### 值接收者

值接收者拿到的是一份拷贝。

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

`Inc` 改的是副本，原来的 `c` 没变。

#### 指针接收者

指针接收者可以修改原值。

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

### 接收者怎么选

可以按下面几个原则判断：

| 场景 | 推荐 |
| --- | --- |
| 方法需要修改结构体 | 指针接收者 |
| 结构体比较大，复制成本高 | 指针接收者 |
| 结构体里有 mutex 等不能复制的字段 | 指针接收者 |
| 方法只是读取小结构体 | 值接收者或指针接收者都可以 |
| 同一个类型已经有指针接收者方法 | 通常保持一致，继续用指针接收者 |

示例：

```go
type Point struct {
	X int
	Y int
}

func (p Point) String() string {
	return fmt.Sprintf("(%d,%d)", p.X, p.Y)
}
```

`Point` 很小，`String` 也不修改原值，用值接收者没问题。

再看一个账户示例：

```go
type Account struct {
	Balance int
}

func (a *Account) Deposit(amount int) {
	a.Balance += amount
}
```

`Deposit` 要修改余额，所以使用指针接收者。

### 实战 Demo：账户充值和扣款

```go
package main

import (
	"errors"
	"fmt"
)

type Account struct {
	ID      int
	Owner   string
	Balance int
}

func (a *Account) Deposit(amount int) error {
	if amount <= 0 {
		return errors.New("amount must be positive")
	}

	a.Balance += amount
	return nil
}

func (a *Account) Withdraw(amount int) error {
	if amount <= 0 {
		return errors.New("amount must be positive")
	}

	if amount > a.Balance {
		return errors.New("insufficient balance")
	}

	a.Balance -= amount
	return nil
}

func (a Account) String() string {
	return fmt.Sprintf("Account{ID:%d Owner:%s Balance:%d}", a.ID, a.Owner, a.Balance)
}

func main() {
	account := &Account{
		ID:      1,
		Owner:   "张三",
		Balance: 100,
	}

	_ = account.Deposit(50)
	_ = account.Withdraw(30)

	fmt.Println(account)
}
```

输出：

```text
Account{ID:1 Owner:张三 Balance:120}
```

这个例子里：

* `Deposit` 和 `Withdraw` 会修改余额，所以用指针接收者
* `String` 只是读取字段，所以用值接收者也可以

### 结构体嵌套

结构体字段可以是另一个结构体。

```go
type Address struct {
	Province string
	City     string
}

type User struct {
	ID      int
	Name    string
	Address Address
}
```

初始化：

```go
u := User{
	ID:   1,
	Name: "张三",
	Address: Address{
		Province: "广东",
		City:     "深圳",
	},
}
```

访问：

```go
fmt.Println(u.Address.City)
```

这种写法是命名字段嵌套。

字段名是 `Address`，访问路径也很清楚。

### 匿名字段和组合

结构体也可以嵌入另一个类型，不写字段名。

```go
type User struct {
	ID   int
	Name string
}

type Admin struct {
	User
	Level int
}
```

`Admin` 里嵌入了 `User`。

完整示例：

```go
package main

import "fmt"

type User struct {
	ID   int
	Name string
}

func (u User) DisplayName() string {
	return fmt.Sprintf("%d-%s", u.ID, u.Name)
}

type Admin struct {
	User
	Level int
}

func main() {
	admin := Admin{
		User: User{
			ID:   1,
			Name: "张三",
		},
		Level: 10,
	}

	fmt.Println(admin.Name)
	fmt.Println(admin.User.Name)
	fmt.Println(admin.DisplayName())
}
```

输出：

```text
张三
张三
1-张三
```

`admin.Name` 是字段提升的效果，本质上还是 `admin.User.Name`。

`admin.DisplayName()` 也是方法提升。

### 嵌入不是继承

Go 里的嵌入经常被拿来类比继承，但它不是传统继承。

更准确的说法是：

```text
嵌入是一种组合，Go 只是把嵌入字段的方法和字段提升到了外层。
```

示例：

```go
type Animal struct {
	Name string
}

func (a Animal) Speak() string {
	return "..."
}

type Dog struct {
	Animal
}

func (d Dog) Speak() string {
	return "wang"
}
```

`Dog` 有自己的 `Speak`，会优先调用 `Dog.Speak`。

如果要调用嵌入类型的方法，需要显式写：

```go
dog.Animal.Speak()
```

### 字段冲突

多个嵌入字段里有同名字段时，直接访问会产生歧义。

```go
type A struct {
	Name string
}

type B struct {
	Name string
}

type C struct {
	A
	B
}
```

下面这种写法会编译失败：

```go
c := C{}
fmt.Println(c.Name)
```

因为 `A` 和 `B` 都有 `Name`。

必须显式指定：

```go
fmt.Println(c.A.Name)
fmt.Println(c.B.Name)
```

### 结构体标签

结构体字段后面可以加标签。

标签本质上是一段字符串元数据。

最常见的是 JSON 标签：

```go
type User struct {
	ID       int    `json:"id"`
	Name     string `json:"name"`
	Email    string `json:"email,omitempty"`
	Password string `json:"-"`
}
```

含义：

| 标签 | 说明 |
| --- | --- |
| `json:"id"` | JSON 字段名叫 `id` |
| `json:"email,omitempty"` | 空值时省略 |
| `json:"-"` | 序列化时忽略 |

### 实战 Demo：JSON 序列化

```go
package main

import (
	"encoding/json"
	"fmt"
)

type User struct {
	ID       int    `json:"id"`
	Name     string `json:"name"`
	Email    string `json:"email,omitempty"`
	Password string `json:"-"`
}

func main() {
	u := User{
		ID:       1,
		Name:     "张三",
		Email:    "",
		Password: "123456",
	}

	data, err := json.Marshal(u)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(data))
}
```

输出：

```text
{"id":1,"name":"张三"}
```

`Email` 是空字符串，并且配置了 `omitempty`，所以被省略。

`Password` 配置了 `json:"-"`，所以不会输出。

### 实战 Demo：JSON 解析请求参数

Web 开发里，请求参数通常会映射到结构体。

```go
package main

import (
	"encoding/json"
	"fmt"
)

type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email"`
	Age   int    `json:"age"`
}

func main() {
	body := []byte(`{"name":"张三","email":"zhangsan@example.com","age":20}`)

	var req CreateUserRequest
	if err := json.Unmarshal(body, &req); err != nil {
		panic(err)
	}

	fmt.Printf("%+v\n", req)
}
```

输出：

```text
{Name:张三 Email:zhangsan@example.com Age:20}
```

注意这里传给 `json.Unmarshal` 的是 `&req`。

原因是解析 JSON 时需要修改 `req`，必须传指针。

### Entity、DTO、VO 的常见拆法

Go 项目里，经常会把不同层的数据结构拆开。

例如用户表实体：

```go
type UserEntity struct {
	ID        int64
	Username  string
	Password  string
	Email     string
	CreatedAt int64
}
```

接口响应结构：

```go
type UserVO struct {
	ID       int64  `json:"id"`
	Username string `json:"username"`
	Email    string `json:"email"`
}
```

创建用户请求：

```go
type CreateUserDTO struct {
	Username string `json:"username"`
	Password string `json:"password"`
	Email    string `json:"email"`
}
```

为什么不直接把 `UserEntity` 返回给前端？

原因很简单：

```text
数据库结构、请求结构、响应结构经常不是一回事。
```

例如 `Password` 存在数据库里，但不应该出现在响应里。

转换函数可以这样写：

```go
func ToUserVO(entity UserEntity) UserVO {
	return UserVO{
		ID:       entity.ID,
		Username: entity.Username,
		Email:    entity.Email,
	}
}
```

### 实战 Demo：简单用户服务

下面这个例子把结构体、方法、切片、map、JSON 标签放到一起。

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
)

type UserEntity struct {
	ID       int64
	Username string
	Password string
	Email    string
}

type CreateUserDTO struct {
	Username string `json:"username"`
	Password string `json:"password"`
	Email    string `json:"email"`
}

type UserVO struct {
	ID       int64  `json:"id"`
	Username string `json:"username"`
	Email    string `json:"email"`
}

type UserService struct {
	nextID int64
	users  map[int64]UserEntity
}

func NewUserService() *UserService {
	return &UserService{
		nextID: 1,
		users:  make(map[int64]UserEntity),
	}
}

func (s *UserService) Create(dto CreateUserDTO) (UserVO, error) {
	if dto.Username == "" {
		return UserVO{}, errors.New("username is required")
	}

	user := UserEntity{
		ID:       s.nextID,
		Username: dto.Username,
		Password: dto.Password,
		Email:    dto.Email,
	}

	s.users[user.ID] = user
	s.nextID++

	return ToUserVO(user), nil
}

func (s *UserService) FindByID(id int64) (UserVO, bool) {
	user, ok := s.users[id]
	if !ok {
		return UserVO{}, false
	}

	return ToUserVO(user), true
}

func ToUserVO(entity UserEntity) UserVO {
	return UserVO{
		ID:       entity.ID,
		Username: entity.Username,
		Email:    entity.Email,
	}
}

func main() {
	service := NewUserService()

	dto := CreateUserDTO{
		Username: "zhangsan",
		Password: "123456",
		Email:    "zhangsan@example.com",
	}

	created, err := service.Create(dto)
	if err != nil {
		panic(err)
	}

	found, ok := service.FindByID(created.ID)
	if !ok {
		panic("user not found")
	}

	data, err := json.Marshal(found)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(data))
}
```

输出：

```text
{"id":1,"username":"zhangsan","email":"zhangsan@example.com"}
```

这个 demo 里：

* `UserEntity` 模拟数据库实体
* `CreateUserDTO` 模拟创建请求
* `UserVO` 模拟接口响应
* `UserService` 用结构体保存服务状态
* `NewUserService` 负责初始化内部 map
* `ToUserVO` 负责隐藏密码字段

### 结构体比较

结构体能不能用 `==` 比较，取决于字段。

如果所有字段都可以比较，结构体就可以比较。

```go
package main

import "fmt"

type Point struct {
	X int
	Y int
}

func main() {
	p1 := Point{X: 1, Y: 2}
	p2 := Point{X: 1, Y: 2}

	fmt.Println(p1 == p2)
}
```

输出：

```text
true
```

如果结构体包含 slice、map、func，就不能直接比较。

```go
type User struct {
	Name string
	Tags []string
}
```

下面这行会编译失败：

```go
fmt.Println(u1 == u2)
```

可以手动比较关键字段，或者使用 `reflect.DeepEqual`。

```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Name string
	Tags []string
}

func main() {
	u1 := User{Name: "张三", Tags: []string{"go", "java"}}
	u2 := User{Name: "张三", Tags: []string{"go", "java"}}

	fmt.Println(reflect.DeepEqual(u1, u2))
}
```

输出：

```text
true
```

`reflect.DeepEqual` 很方便，但业务代码里通常更推荐明确比较真正关心的字段。

### 空结构体 struct{}

空结构体没有字段。

```go
type Empty struct{}
```

它最常见的特点是几乎不占用空间。

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	fmt.Println(unsafe.Sizeof(struct{}{}))
}
```

输出：

```text
0
```

空结构体常用于两个场景：

* map 实现 set
* channel 只传信号，不传数据

### 实战 Demo：map + struct{} 实现 Set

Go 没有内置 set，可以用 map 模拟。

```go
package main

import "fmt"

type StringSet map[string]struct{}

func NewStringSet(items ...string) StringSet {
	set := make(StringSet, len(items))

	for _, item := range items {
		set[item] = struct{}{}
	}

	return set
}

func (s StringSet) Add(item string) {
	s[item] = struct{}{}
}

func (s StringSet) Contains(item string) bool {
	_, ok := s[item]
	return ok
}

func main() {
	set := NewStringSet("go", "java")

	set.Add("mysql")

	fmt.Println(set.Contains("go"))
	fmt.Println(set.Contains("redis"))
}
```

输出：

```text
true
false
```

`map[string]struct{}` 里的 value 不关心内容，只关心 key 是否存在。

### 实战 Demo：chan struct{} 做完成信号

如果 channel 只需要表达“完成了”，不用传具体数据，可以用 `struct{}`。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	done := make(chan struct{})

	go func() {
		time.Sleep(500 * time.Millisecond)
		fmt.Println("job done")
		close(done)
	}()

	<-done
	fmt.Println("main exit")
}
```

输出：

```text
job done
main exit
```

这里关心的是信号，不关心数据内容，所以 `chan struct{}` 很合适。

### 匿名结构体

有时只需要临时组织一组字段，不想单独定义类型，可以使用匿名结构体。

```go
package main

import "fmt"

func main() {
	config := struct {
		Host string
		Port int
	}{
		Host: "127.0.0.1",
		Port: 8080,
	}

	fmt.Printf("%+v\n", config)
}
```

输出：

```text
{Host:127.0.0.1 Port:8080}
```

匿名结构体适合很小的临时数据。

如果多个地方都要用，应该定义成具名结构体。

### 结构体和接口

结构体可以通过方法实现接口。

Go 不需要显式写 `implements`。

只要方法集合匹配，就算实现了接口。

```go
package main

import "fmt"

type Notifier interface {
	Notify(message string) error
}

type EmailNotifier struct {
	Address string
}

func (e EmailNotifier) Notify(message string) error {
	fmt.Printf("send email to %s: %s\n", e.Address, message)
	return nil
}

func Send(n Notifier, message string) error {
	return n.Notify(message)
}

func main() {
	email := EmailNotifier{Address: "admin@example.com"}

	_ = Send(email, "server started")
}
```

输出：

```text
send email to admin@example.com: server started
```

这就是 Go 里常见的设计方式：

```text
struct 保存数据和实现方法，interface 描述需要的行为。
```

### 常见问题

#### 结构体字段一定要大写吗

不一定。

如果字段需要被其他包访问，首字母要大写。

如果字段只在当前包内部使用，首字母小写更合适。

JSON 序列化时，如果字段是小写，即使写了 tag，标准库也不会导出它。

```go
type User struct {
	name string `json:"name"`
}
```

`name` 是未导出字段，`encoding/json` 不能正常序列化它。

#### 方法接收者一定要用指针吗

不一定。

需要修改原对象、大结构体避免拷贝、包含锁等不能复制的字段时，用指针接收者。

小结构体只读方法，用值接收者也很正常。

#### 结构体可以包含自己吗

不能直接包含自己。

```go
type Node struct {
	Next Node
}
```

这种写法会导致无限大小，编译失败。

可以包含自己的指针：

```go
type Node struct {
	Value int
	Next  *Node
}
```

链表、树结构都会用这种写法。

#### 结构体适合到处传值吗

小结构体传值问题不大。

大结构体、需要修改状态、包含锁或大量字段时，更适合传指针。

### 总结

`struct` 的核心可以压缩成几句话：

```text
struct 用来把一组字段组织成一个业务对象。
struct 是值类型，传参默认会复制。
struct 可以定义方法，方法通过接收者绑定到类型上。
struct 可以通过嵌入实现组合，但嵌入不是传统继承。
struct tag 常用于 JSON、ORM、参数校验等场景。
```

日常使用可以按下面这张表判断：

| 场景 | 常见写法 |
| --- | --- |
| 定义业务模型 | `type User struct { ... }` |
| 初始化结构体 | `User{Field: value}` |
| 初始化结构体指针 | `&User{Field: value}` |
| 零值指针 | `new(User)` |
| 修改结构体状态 | 指针接收者 |
| 只读小结构体方法 | 值接收者也可以 |
| 复用字段和方法 | 匿名嵌入 |
| JSON 字段映射 | struct tag |
| set 占位值 | `struct{}` |
| 完成信号 | `chan struct{}` |

写 Go 项目时，`struct` 不是单纯的字段集合。它会出现在实体、配置、请求响应、服务对象、数据结构、接口实现里。理解值语义、指针接收者、组合和标签，结构体相关代码就会清楚很多。
