### 简介

Go 代码里最常见的错误处理大概是这样：

```go
result, err := doSomething()
if err != nil {
	return err
}
```

这几行代码不难，真正容易出问题的是后面的选择：

```text
应该新建错误，还是包装原错误？
应该使用 ==，还是 errors.Is？
什么时候需要自定义错误类型？
错误应该在哪一层记录日志？
多个清理操作同时失败，应该返回哪一个错误？
普通错误、panic 和 recover 到底怎么分工？
```

Go 没有把错误处理藏进异常机制，而是把错误当成普通值显式传递。

这套设计看起来重复，却带来一个直接好处：函数签名会明确说明操作可能失败，调用方也能在失败发生的位置决定返回、重试、降级还是终止。

一句话概括：

```text
error 不只是错误文本，它还是可以分类、包装、传递和组合的值。
```

### error 到底是什么

`error` 是 Go 内置的接口类型，定义非常简单：

```go
type error interface {
	Error() string
}
```

任何类型只要实现了 `Error() string`，就实现了 `error` 接口。

```go
package main

import "fmt"

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

func main() {
	err := validateName("")
	fmt.Println(err)
}
```

输出：

```text
字段 name：不能为空
```

`ValidationError` 没有声明实现某个接口。Go 使用隐式接口实现，只要方法集合满足要求，就可以作为 `error` 返回。

### nil 表示操作成功

函数通常把 `error` 放在最后一个返回值：

```go
func divide(a, b int) (int, error) {
	if b == 0 {
		return 0, errors.New("除数不能为 0")
	}
	return a / b, nil
}
```

约定很明确：

* `err == nil`：操作成功，其他返回值可以使用
* `err != nil`：操作失败，先处理错误

完整示例：

```go
package main

import (
	"errors"
	"fmt"
)

func divide(a, b int) (int, error) {
	if b == 0 {
		return 0, errors.New("除数不能为 0")
	}
	return a / b, nil
}

func main() {
	result, err := divide(12, 3)
	if err != nil {
		fmt.Println("计算失败：", err)
		return
	}

	fmt.Println("计算结果：", result)
}
```

输出：

```text
计算结果： 4
```

错误分支尽早返回，可以减少嵌套：

```go
data, err := loadData()
if err != nil {
	return err
}

result, err := parseData(data)
if err != nil {
	return err
}

return saveResult(result)
```

正常流程保持在左侧，错误处理紧跟在可能失败的调用后面，这就是 Go 项目里常见的写法。

### 创建错误的三种常见方式

#### errors.New：固定错误文本

`errors.New` 适合创建不需要动态参数的简单错误：

```go
err := errors.New("用户名不能为空")
```

完整示例：

```go
package main

import (
	"errors"
	"fmt"
)

func checkAge(age int) error {
	if age < 0 {
		return errors.New("年龄不能小于 0")
	}
	return nil
}

func main() {
	if err := checkAge(-1); err != nil {
		fmt.Println(err)
	}
}
```

#### fmt.Errorf：带动态信息

错误信息需要带上文件名、用户 ID 或参数值时，使用 `fmt.Errorf`：

```go
return fmt.Errorf("用户 %d 不存在", userID)
```

这里仅仅是格式化文本，还没有形成错误链。

```go
package main

import "fmt"

func findUser(id int64) error {
	return fmt.Errorf("用户 %d 不存在", id)
}

func main() {
	fmt.Println(findUser(1001))
}
```

输出：

```text
用户 1001 不存在
```

#### 自定义错误：携带结构化信息

如果调用方需要读取错误码、字段名、重试时间等信息，应定义错误类型，而不是解析错误字符串。

```go
type RateLimitError struct {
	RetryAfter time.Duration
}

func (e *RateLimitError) Error() string {
	return fmt.Sprintf("请求过于频繁，%s 后重试", e.RetryAfter)
}
```

错误文本适合阅读，结构化字段适合程序判断。

### 错误文本不是错误身份

下面两个错误的文本相同，但不是同一个错误值：

```go
first := errors.New("not found")
second := errors.New("not found")

fmt.Println(first == second)
```

输出：

```text
false
```

因此，不能在判断时临时创建一个同文本错误：

```go
if errors.Is(err, errors.New("not found")) {
	// 通常匹配不到
}
```

需要稳定识别的错误，应复用同一个变量：

```go
var ErrNotFound = errors.New("not found")
```

这种包级错误值通常称为哨兵错误。

### 哨兵错误：表示稳定的错误类别

哨兵错误适合表达调用方需要识别的固定状态：

```go
var (
	ErrNotFound   = errors.New("not found")
	ErrConflict   = errors.New("conflict")
	ErrPermission = errors.New("permission denied")
)
```

完整示例：

```go
package main

import (
	"errors"
	"fmt"
)

var ErrUserNotFound = errors.New("user not found")

func findUser(id int64) error {
	if id != 1001 {
		return ErrUserNotFound
	}
	return nil
}

func main() {
	err := findUser(2002)
	if errors.Is(err, ErrUserNotFound) {
		fmt.Println("返回 404")
		return
	}

	if err != nil {
		fmt.Println("返回 500")
	}
}
```

导出的哨兵错误会成为包的公开契约。调用方一旦依赖 `ErrUserNotFound`，后续修改时就要继续维护这个语义。

只想返回一段说明，不希望调用方依赖错误类别时，普通错误文本通常已经够用。

### 为什么要包装错误

底层函数返回的错误往往缺少业务上下文。

例如：

```text
file does not exist
```

只看这句话，不知道读取了哪个文件，也不知道发生在哪个业务流程。

可以使用 `%w` 包装原错误：

```go
return fmt.Errorf("读取配置 %q: %w", filename, err)
```

包装后同时保留两部分信息：

```text
外层上下文：读取配置 "app.json"
底层原因：file does not exist
```

完整示例：

```go
package main

import (
	"errors"
	"fmt"
	"os"
)

func readConfig(filename string) ([]byte, error) {
	data, err := os.ReadFile(filename)
	if err != nil {
		return nil, fmt.Errorf("读取配置 %q: %w", filename, err)
	}
	return data, nil
}

func main() {
	_, err := readConfig("missing.json")
	if err == nil {
		return
	}

	fmt.Println(err)
	if errors.Is(err, os.ErrNotExist) {
		fmt.Println("配置文件不存在，加载默认配置")
	}
}
```

输出类似：

```text
读取配置 "missing.json": open missing.json: no such file or directory
配置文件不存在，加载默认配置
```

`%w` 和 `%v` 的差别非常重要：

```go
fmt.Errorf("读取配置失败: %v", err) // 只拼接文本
fmt.Errorf("读取配置失败: %w", err) // 包装错误，保留错误链
```

两者打印出来可能很像，但只有 `%w` 能让 `errors.Is`、`errors.As` 继续识别底层错误。

### 错误链是怎么形成的

使用 `%w` 包装后，外层错误会提供：

```go
Unwrap() error
```

可以把单链错误理解成：

```text
HTTP 层错误
    ↓ Unwrap
Service 层错误
    ↓ Unwrap
Repository 层错误
    ↓ Unwrap
ErrNotFound
```

示例：

```go
package main

import (
	"errors"
	"fmt"
)

var ErrRecordNotFound = errors.New("record not found")

func queryUser(id int64) error {
	return fmt.Errorf("查询 user_id=%d: %w", id, ErrRecordNotFound)
}

func loadProfile(id int64) error {
	if err := queryUser(id); err != nil {
		return fmt.Errorf("加载用户资料: %w", err)
	}
	return nil
}

func main() {
	err := loadProfile(1001)
	fmt.Println(err)
	fmt.Println(errors.Is(err, ErrRecordNotFound))
	fmt.Println(errors.Unwrap(err))
}
```

输出：

```text
加载用户资料: 查询 user_id=1001: record not found
true
查询 user_id=1001: record not found
```

`errors.Unwrap` 只拆一层。业务判断通常不需要手动循环解包，直接使用 `errors.Is` 或 `errors.As` 即可。

### errors.Is：判断错误值和错误类别

直接使用 `==` 只能比较当前错误值：

```go
err == ErrNotFound
```

错误被包装后，外层错误不再等于哨兵错误：

```go
wrapped := fmt.Errorf("查询失败: %w", ErrNotFound)

fmt.Println(wrapped == ErrNotFound)            // false
fmt.Println(errors.Is(wrapped, ErrNotFound))   // true
```

`errors.Is` 会沿错误树向下检查：

* 当前错误是否等于目标错误
* 当前错误是否实现自定义的 `Is(error) bool`
* 当前错误能否通过 `Unwrap` 继续展开

因此，只要错误可能被包装，就优先使用：

```go
if errors.Is(err, ErrNotFound) {
	// 按未找到处理
}
```

不要比较错误文本：

```go
if err.Error() == "record not found" {
	// 文本稍有变化就失效
}
```

错误文本用于展示和日志，`errors.Is` 用于程序分支。

### errors.As：提取错误链中的具体类型

哨兵错误适合表达固定类别，自定义错误类型适合携带额外字段。

```go
package main

import (
	"errors"
	"fmt"
)

type ValidationError struct {
	Field   string
	Value   any
	Message string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("字段 %s 的值 %v 不合法：%s", e.Field, e.Value, e.Message)
}

func validateAge(age int) error {
	if age < 0 || age > 150 {
		return &ValidationError{
			Field:   "age",
			Value:   age,
			Message: "必须在 0 到 150 之间",
		}
	}
	return nil
}

func register(age int) error {
	if err := validateAge(age); err != nil {
		return fmt.Errorf("注册校验失败: %w", err)
	}
	return nil
}

func main() {
	err := register(200)

	var validationErr *ValidationError
	if errors.As(err, &validationErr) {
		fmt.Printf("field=%s, value=%v, message=%s\n",
			validationErr.Field,
			validationErr.Value,
			validationErr.Message,
		)
	}
}
```

输出：

```text
field=age, value=200, message=必须在 0 到 150 之间
```

直接类型断言只检查最外层动态类型：

```go
validationErr, ok := err.(*ValidationError)
```

如果错误已经被 `%w` 包装，通常会断言失败。

`errors.As` 会遍历错误链，更适合提取可能被包装的自定义错误。

目标变量的写法要与错误类型一致：

```go
var target *ValidationError
if errors.As(err, &target) {
	// target 的类型是 *ValidationError
}
```

### 自定义错误同时保留底层原因

自定义错误不仅可以保存业务字段，也可以实现 `Unwrap` 保留底层错误：

```go
package main

import (
	"errors"
	"fmt"
)

var ErrInsufficientBalance = errors.New("insufficient balance")

type PaymentError struct {
	OrderID int64
	Code    string
	Err     error
}

func (e *PaymentError) Error() string {
	return fmt.Sprintf("订单 %d 支付失败，code=%s: %v", e.OrderID, e.Code, e.Err)
}

func (e *PaymentError) Unwrap() error {
	return e.Err
}

func pay(orderID int64) error {
	return &PaymentError{
		OrderID: orderID,
		Code:    "BALANCE_NOT_ENOUGH",
		Err:     ErrInsufficientBalance,
	}
}

func main() {
	err := pay(9001)

	if errors.Is(err, ErrInsufficientBalance) {
		fmt.Println("提示余额不足")
	}

	var paymentErr *PaymentError
	if errors.As(err, &paymentErr) {
		fmt.Printf("order_id=%d, code=%s\n", paymentErr.OrderID, paymentErr.Code)
	}
}
```

输出：

```text
提示余额不足
order_id=9001, code=BALANCE_NOT_ENOUGH
```

这时同一个错误同时支持两种判断：

* `errors.Is` 判断底层错误类别
* `errors.As` 读取外层结构化信息

### errors.Join：合并多个错误

有些操作可能同时产生多个错误。

例如关闭多个资源、批量校验多个字段、并行执行多个任务。只返回最后一个错误，会把前面的错误丢掉。

Go 1.20 增加了 `errors.Join`：

```go
joined := errors.Join(firstErr, secondErr)
```

它有几个特点：

* 忽略传入的 `nil`
* 所有参数都是 `nil` 时返回 `nil`
* 错误文本通常按换行连接
* `errors.Is` 和 `errors.As` 可以遍历每个分支

批量校验示例：

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

var (
	ErrNameRequired     = errors.New("name is required")
	ErrPasswordTooShort = errors.New("password is too short")
	ErrEmailInvalid     = errors.New("email is invalid")
)

type RegisterRequest struct {
	Name     string
	Email    string
	Password string
}

func validate(request RegisterRequest) error {
	var errs []error

	if strings.TrimSpace(request.Name) == "" {
		errs = append(errs, ErrNameRequired)
	}
	if !strings.Contains(request.Email, "@") {
		errs = append(errs, ErrEmailInvalid)
	}
	if len(request.Password) < 8 {
		errs = append(errs, ErrPasswordTooShort)
	}

	return errors.Join(errs...)
}

func main() {
	err := validate(RegisterRequest{
		Name:     "",
		Email:    "invalid-email",
		Password: "123",
	})

	if err == nil {
		fmt.Println("校验通过")
		return
	}

	fmt.Println(err)
	fmt.Println("邮箱错误：", errors.Is(err, ErrEmailInvalid))
	fmt.Println("密码错误：", errors.Is(err, ErrPasswordTooShort))
}
```

输出：

```text
name is required
email is invalid
password is too short
邮箱错误： true
密码错误： true
```

`errors.Join` 形成的是错误树，不再只是单链。

它实现的是：

```go
Unwrap() []error
```

而 `errors.Unwrap` 只处理 `Unwrap() error`，不会返回 `errors.Join` 的子错误。判断合并错误时，应直接使用 `errors.Is` 和 `errors.As`。

### 一个 fmt.Errorf 可以包装多个错误

现代 Go 允许一个 `fmt.Errorf` 格式串包含多个 `%w`：

```go
err := fmt.Errorf("保存失败，写入错误: %w，关闭错误: %w", writeErr, closeErr)
```

这同样会形成多分支错误树，`errors.Is` 和 `errors.As` 可以检查其中任意分支。

如果只是把多个独立错误汇总起来，`errors.Join` 通常更直接。

如果还需要在一条错误文本中说明每个错误对应的操作，多个 `%w` 更有表达力。

### 实战一：读取并解析配置

下面的 demo 串起文件读取、JSON 解析、错误包装和错误分类。

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"os"
)

type Config struct {
	Address string `json:"address"`
	Port    int    `json:"port"`
}

func loadConfig(filename string) (Config, error) {
	data, err := os.ReadFile(filename)
	if err != nil {
		return Config{}, fmt.Errorf("读取配置文件 %q: %w", filename, err)
	}

	var config Config
	if err := json.Unmarshal(data, &config); err != nil {
		return Config{}, fmt.Errorf("解析配置文件 %q: %w", filename, err)
	}

	if config.Address == "" {
		return Config{}, errors.New("配置 address 不能为空")
	}
	if config.Port <= 0 || config.Port > 65535 {
		return Config{}, fmt.Errorf("配置 port 超出范围: %d", config.Port)
	}

	return config, nil
}

func main() {
	config, err := loadConfig("app.json")
	if err != nil {
		switch {
		case errors.Is(err, os.ErrNotExist):
			fmt.Println("配置文件不存在")
		case errors.Is(err, os.ErrPermission):
			fmt.Println("没有读取配置文件的权限")
		default:
			var syntaxErr *json.SyntaxError
			if errors.As(err, &syntaxErr) {
				fmt.Printf("JSON 第 %d 字节附近存在语法错误\n", syntaxErr.Offset)
				return
			}
			fmt.Println("加载配置失败：", err)
		}
		return
	}

	fmt.Printf("配置加载成功：%s:%d\n", config.Address, config.Port)
}
```

这个例子体现了分层处理方式：

```text
底层负责返回具体原因。
中间层使用 %w 增加操作上下文。
边界层使用 Is 或 As 决定最终动作。
```

### 实战二：统一 HTTP 错误响应

业务层不应该到处拼 HTTP JSON，也不应该把数据库错误原文直接返回给客户端。

可以定义应用错误，在 HTTP 边界统一映射：

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"net/http"
	"net/http/httptest"
)

var ErrUserNotFound = errors.New("user not found")

type AppError struct {
	Status  int
	Code    string
	Message string
	Err     error
}

func (e *AppError) Error() string {
	return fmt.Sprintf("%s: %v", e.Code, e.Err)
}

func (e *AppError) Unwrap() error {
	return e.Err
}

func getUser(id string) error {
	if id == "" {
		return &AppError{
			Status:  http.StatusBadRequest,
			Code:    "INVALID_ARGUMENT",
			Message: "id 不能为空",
			Err:     errors.New("empty user id"),
		}
	}

	return fmt.Errorf("查询用户 id=%s: %w", id, ErrUserNotFound)
}

func writeError(w http.ResponseWriter, err error) {
	status := http.StatusInternalServerError
	code := "INTERNAL_ERROR"
	message := "服务暂时不可用"

	var appErr *AppError
	switch {
	case errors.As(err, &appErr):
		status = appErr.Status
		code = appErr.Code
		message = appErr.Message
	case errors.Is(err, ErrUserNotFound):
		status = http.StatusNotFound
		code = "USER_NOT_FOUND"
		message = "用户不存在"
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(map[string]string{
		"code":    code,
		"message": message,
	})
}

func handler(w http.ResponseWriter, r *http.Request) {
	if err := getUser(r.URL.Query().Get("id")); err != nil {
		writeError(w, err)
		return
	}
	w.WriteHeader(http.StatusNoContent)
}

func main() {
	request := httptest.NewRequest(http.MethodGet, "/user?id=1001", nil)
	recorder := httptest.NewRecorder()

	handler(recorder, request)

	fmt.Println("status:", recorder.Code)
	fmt.Print("body: ", recorder.Body.String())
}
```

输出：

```text
status: 404
body: {"code":"USER_NOT_FOUND","message":"用户不存在"}
```

这里把两类信息分开了：

* 内部错误链：用于日志和排查
* 对外错误码与消息：用于稳定 API 契约

数据库地址、SQL、文件路径和调用栈等内部信息不应该直接暴露给客户端。

### 实战三：只重试可恢复错误

重试不能只看“发生了错误”。参数错误、权限错误等永久性错误，重复执行不会变好。

可以定义带重试信息的错误类型：

```go
package main

import (
	"errors"
	"fmt"
	"time"
)

type TemporaryError struct {
	After time.Duration
	Err   error
}

func (e *TemporaryError) Error() string {
	return fmt.Sprintf("临时错误，%s 后可重试: %v", e.After, e.Err)
}

func (e *TemporaryError) Unwrap() error {
	return e.Err
}

func retry(maxAttempts int, operation func() error) error {
	var lastErr error

	for attempt := 1; attempt <= maxAttempts; attempt++ {
		err := operation()
		if err == nil {
			return nil
		}
		lastErr = err

		var temporaryErr *TemporaryError
		if !errors.As(err, &temporaryErr) {
			return err
		}

		fmt.Printf("第 %d 次执行失败：%v\n", attempt, err)
		if attempt < maxAttempts {
			time.Sleep(temporaryErr.After)
		}
	}

	return fmt.Errorf("重试 %d 次后仍然失败: %w", maxAttempts, lastErr)
}

func main() {
	attempts := 0
	err := retry(3, func() error {
		attempts++
		if attempts < 3 {
			return &TemporaryError{
				After: time.Millisecond,
				Err:   errors.New("远端服务超时"),
			}
		}
		return nil
	})

	fmt.Println("最终错误：", err)
	fmt.Println("执行次数：", attempts)
}
```

输出：

```text
第 1 次执行失败：临时错误，1ms 后可重试: 远端服务超时
第 2 次执行失败：临时错误，1ms 后可重试: 远端服务超时
最终错误： <nil>
执行次数： 3
```

真实项目还应考虑指数退避、随机抖动、上下文取消、请求幂等性和最大总耗时。

### 实战四：保留关闭资源时的错误

很多代码会这样写：

```go
defer file.Close()
```

读取文件时通常可以接受，但写文件、刷新缓冲区或提交数据时，关闭阶段也可能失败。完全忽略 `Close` 错误，可能把写入不完整误判成成功。

可以使用命名返回值和 `errors.Join` 合并主流程错误与关闭错误：

```go
package main

import (
	"errors"
	"fmt"
)

type Writer struct {
	writeErr error
	closeErr error
}

func (w *Writer) Write([]byte) error {
	return w.writeErr
}

func (w *Writer) Close() error {
	return w.closeErr
}

func save(writer *Writer, data []byte) (err error) {
	defer func() {
		err = errors.Join(err, writer.Close())
	}()

	if writeErr := writer.Write(data); writeErr != nil {
		return fmt.Errorf("写入数据: %w", writeErr)
	}

	return nil
}

func main() {
	writeErr := errors.New("磁盘空间不足")
	closeErr := errors.New("刷新缓冲区失败")

	err := save(&Writer{writeErr: writeErr, closeErr: closeErr}, []byte("data"))
	fmt.Println(err)
	fmt.Println("包含写入错误：", errors.Is(err, writeErr))
	fmt.Println("包含关闭错误：", errors.Is(err, closeErr))
}
```

输出：

```text
写入数据: 磁盘空间不足
刷新缓冲区失败
包含写入错误： true
包含关闭错误： true
```

是否必须处理 `Close` 错误取决于资源语义。只读文件和内存缓冲区的风险不同，持久化写入、压缩流和网络连接更值得关注关闭阶段的结果。

### typed nil：看起来是 nil，返回后却不为 nil

`error` 是接口，接口值由动态类型和动态值组成。

下面的函数有隐藏问题：

```go
type QueryError struct{}

func (*QueryError) Error() string {
	return "query failed"
}

func bad() error {
	var err *QueryError
	return err
}
```

`err` 指针虽然是 `nil`，返回到 `error` 接口后，接口中仍然保存了动态类型 `*QueryError`，所以接口本身不等于 `nil`。

完整示例：

```go
package main

import "fmt"

type QueryError struct{}

func (*QueryError) Error() string {
	return "query failed"
}

func bad() error {
	var err *QueryError
	return err
}

func good() error {
	return nil
}

func main() {
	fmt.Println("bad() == nil：", bad() == nil)
	fmt.Println("good() == nil：", good() == nil)
}
```

输出：

```text
bad() == nil： false
good() == nil： true
```

没有错误时应直接返回字面量 `nil`，不要把 nil 具体指针装进 `error` 接口。

### 应该包装，还是直接返回

不是每一层都必须包装。

包装适合补充有价值的上下文：

```go
return fmt.Errorf("读取订单 %d: %w", orderID, err)
```

直接返回适合当前函数没有新增信息的情况：

```go
return repository.Save(order)
```

无意义的层层包装会产生噪声：

```text
handler failed: service failed: use case failed: repository failed: query failed
```

更实用的判断标准是：

```text
当前层能否补充操作名、关键标识或业务阶段？
```

能补充有效上下文就包装，不能就直接返回。

还有一个重要边界：包装底层错误等于允许调用方通过 `errors.Is` 或 `errors.As` 观察它。

公开包如果不希望暴露底层实现细节，可以转换成包自己的错误契约，而不是直接 `%w` 暴露数据库驱动错误。

### 错误应该在哪里记录日志

常见问题是每一层都记录一次：

```text
Repository 记录一次
Service 记录一次
Handler 再记录一次
```

同一个故障最终产生三条甚至更多重复日志。

更清楚的分工是：

```text
底层：返回错误，必要时增加上下文。
中间层：分类、转换或继续包装。
系统边界：记录一次完整日志，并决定响应、退出或重试。
```

HTTP 服务通常在中间件或 Handler 边界记录；命令行程序通常在 `main` 附近记录；后台任务通常在任务执行器边界记录。

如果中间层已经真正处理了错误，例如降级成功、忽略了某个可接受错误，记录一条有业务意义的日志也合理。

### 错误文本怎么写

Go 标准库和常见项目通常使用小写开头、不加句号的错误文本：

```go
errors.New("user not found")
fmt.Errorf("read config %q: %w", filename, err)
```

原因是错误经常会被继续包装：

```text
start server: load config: read config "app.json": file does not exist
```

每一层都写完整句子和句号，组合后会显得断裂。

中文项目不受大小写影响，但仍适合保持短语风格，并包含必要上下文：

```go
fmt.Errorf("查询订单 order_id=%d: %w", orderID, err)
```

不要把密码、令牌、完整身份证号等敏感数据写进错误文本，因为错误很可能进入日志和监控系统。

### error 和 panic 的边界

`error` 用于调用方可以预期并处理的失败：

* 参数不合法
* 文件不存在
* 用户不存在
* 余额不足
* 请求超时
* 数据库暂时不可用

`panic` 更适合程序内部不变量被破坏，或者初始化阶段已经无法继续运行：

* 必需的静态模板无法加载
* 程序内部状态违反不变量
* 数组越界和 nil 指针等编程错误

普通业务失败不应该使用 panic：

```go
if balance < amount {
	return ErrInsufficientBalance
}
```

而不是：

```go
if balance < amount {
	panic("余额不足")
}
```

### recover 不是通用错误处理

`recover` 只能在同一个 goroutine 的延迟函数中捕获 panic。

```go
func runSafely(task func()) (err error) {
	defer func() {
		if value := recover(); value != nil {
			err = fmt.Errorf("任务发生 panic: %v", value)
		}
	}()

	task()
	return nil
}
```

它适合放在进程边界或任务边界，例如 HTTP 中间件、消息消费者和任务执行器，防止单个任务的 panic 直接拖垮整个服务。

recover 之后不能假装什么都没发生。通常还需要记录堆栈、终止当前请求或任务，并确保共享状态没有处于半完成状态。

不要用 panic 加 recover 模拟 try-catch。可预期失败继续使用 `error`。

### 常见错误

#### 忽略 error

不推荐：

```go
data, _ := os.ReadFile("config.json")
```

如果确实允许忽略，应写清楚原因，例如尽力而为的清理操作。普通业务流程直接丢弃错误，很容易把根因变成后续的空值、脏数据或 panic。

#### 使用 %v 破坏错误链

```go
return fmt.Errorf("保存用户失败: %v", err)
```

文本还在，但错误身份丢失。

需要保留底层原因时使用：

```go
return fmt.Errorf("保存用户失败: %w", err)
```

#### 使用 == 判断包装错误

```go
if err == ErrNotFound {
	// 包装后匹配不到
}
```

更稳妥的写法：

```go
if errors.Is(err, ErrNotFound) {
	// 可以沿错误链匹配
}
```

#### 比较 err.Error()

```go
if err.Error() == "user not found" {
}
```

错误文本一旦增加 ID、操作名或本地化内容，判断就会失效。稳定分类应使用哨兵错误、错误类型或明确错误码。

#### 返回 nil 具体指针

```go
func load() error {
	var err *LoadError
	return err
}
```

返回后的接口不等于 `nil`。成功分支直接 `return nil`。

#### 到处记录同一个错误

错误每向上传一层就记录一次，会造成重复告警和日志噪声。通常在真正处理错误的边界记录一次即可。

#### 把内部错误直接返回给客户端

下面的做法可能泄露 SQL、路径和内部结构：

```go
http.Error(w, err.Error(), http.StatusInternalServerError)
```

对外返回稳定的错误码和安全消息，内部日志保留完整错误链。

### 工程实践建议

#### 先定义错误契约

包的调用方需要区分哪些失败，应在设计 API 时明确：

```go
var ErrNotFound = errors.New("not found")
```

或者：

```go
type ValidationError struct {
	Field string
}
```

不需要调用方识别的内部细节，不必全部导出。

#### 增加能定位问题的上下文

好的错误信息通常包含：

* 做了什么操作
* 操作对象的非敏感标识
* 底层原因

例如：

```go
return fmt.Errorf("更新订单 order_id=%d 状态为 %s: %w", orderID, status, err)
```

#### 用 Is 分类，用 As 取字段

```go
if errors.Is(err, ErrNotFound) {
	// 判断稳定错误类别
}

var validationErr *ValidationError
if errors.As(err, &validationErr) {
	// 读取字段名等结构化信息
}
```

#### 在边界完成错误翻译

数据库错误不应该一路原样变成 HTTP 响应。

常见转换关系：

| 内部错误 | HTTP 状态 | 对外错误码 |
| --- | --- | --- |
| 参数校验失败 | 400 | `INVALID_ARGUMENT` |
| 资源不存在 | 404 | `NOT_FOUND` |
| 数据冲突 | 409 | `CONFLICT` |
| 权限不足 | 403 | `PERMISSION_DENIED` |
| 未知系统错误 | 500 | `INTERNAL_ERROR` |

同样的业务错误也可以在 gRPC、消息任务或命令行边界翻译成各自的协议结果。

#### 测试错误语义，不要锁死完整文本

脆弱测试：

```go
if err.Error() != "load user: user not found" {
	// 文案调整就失败
}
```

更稳定的测试：

```go
if !errors.Is(err, ErrUserNotFound) {
	// 错误类别不符合预期
}
```

自定义错误可以结合 `errors.As` 检查关键字段。只有错误文本本身就是公开契约时，才需要完整字符串比较。

### 常见问题

#### errors.New 和 fmt.Errorf 怎么选

固定文本使用 `errors.New`：

```go
errors.New("invalid state")
```

需要插入变量使用 `fmt.Errorf`：

```go
fmt.Errorf("invalid state %q", state)
```

需要保留底层错误链时使用 `%w`：

```go
fmt.Errorf("update state: %w", err)
```

#### errors.Is 会比较错误文本吗

不会。

`errors.Is` 主要根据错误值身份、错误链和自定义 `Is` 方法判断。两个文本相同的 `errors.New` 错误通常不会匹配。

#### errors.As 和类型断言有什么区别

类型断言只看当前接口值的动态类型：

```go
target, ok := err.(*ValidationError)
```

`errors.As` 会沿错误树查找：

```go
var target *ValidationError
ok := errors.As(err, &target)
```

错误可能被包装时使用 `errors.As`。

#### errors.Unwrap 能拆开 errors.Join 吗

不能。

`errors.Unwrap` 只调用 `Unwrap() error`，而 `errors.Join` 返回的错误实现 `Unwrap() []error`。合并错误应使用 `errors.Is`、`errors.As`，或者在确实需要遍历时断言 `interface{ Unwrap() []error }`。

#### 错误包装层数越多越好吗

不是。

包装的价值在于补充上下文和保留原因。没有新增信息的包装只会让文本重复。包边界、业务阶段和外部系统调用通常是比较有价值的包装位置。

#### 自定义错误应该使用值还是指针

两种方式都可以，但需要保持一致。

复杂错误通常使用指针接收者：

```go
func (e *ValidationError) Error() string
```

这样只有 `*ValidationError` 实现 `error`，`errors.As` 的目标也应写成：

```go
var target *ValidationError
errors.As(err, &target)
```

不要一部分代码返回值，一部分代码返回指针，否则匹配逻辑容易混乱。

### 总结

Go error 的核心可以压缩成几句话：

```text
error 是只有 Error() string 方法的内置接口。
nil 表示成功，非 nil 表示失败。
errors.New 创建固定错误，fmt.Errorf 创建动态错误。
fmt.Errorf 配合 %w 可以增加上下文并保留错误链。
errors.Is 用于判断错误值和错误类别。
errors.As 用于提取错误链中的具体错误类型。
errors.Join 用于保留多个并列错误。
错误文本用于阅读，错误值、类型和错误码用于程序判断。
可预期失败返回 error，内部不变量破坏才考虑 panic。
日志通常在系统边界记录一次。
```

日常选择可以按下面这张表判断：

| 场景 | 常见写法 |
| --- | --- |
| 固定错误文本 | `errors.New("...")` |
| 动态错误文本 | `fmt.Errorf("id=%d", id)` |
| 包装底层错误 | `fmt.Errorf("操作失败: %w", err)` |
| 判断哨兵错误 | `errors.Is(err, ErrNotFound)` |
| 提取自定义错误 | `errors.As(err, &target)` |
| 合并多个错误 | `errors.Join(errs...)` |
| 普通业务失败 | 返回 `error` |
| 内部不变量被破坏 | 视边界决定是否 `panic` |
| HTTP 或任务边界 | 记录日志并翻译错误 |

`if err != nil` 只是错误处理的入口。真正稳定的错误体系，需要保留原因、补充上下文、提供可判断的错误语义，并在合适的边界完成日志记录和协议转换。
