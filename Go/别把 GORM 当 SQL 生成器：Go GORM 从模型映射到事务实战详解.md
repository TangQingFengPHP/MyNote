### 简介

Go 访问数据库有两条常见路线。

一条是标准库 `database/sql`：

```go
rows, err := db.QueryContext(ctx, "select id, name from users where status = ?", "active")
if err != nil {
	return err
}
defer rows.Close()

for rows.Next() {
	var id int64
	var name string
	if err := rows.Scan(&id, &name); err != nil {
		return err
	}
	fmt.Println(id, name)
}
```

这种方式最直接，SQL 完全可控，但样板代码很多。

另一条是 ORM，比如 GORM：

```go
var users []User
err := db.WithContext(ctx).
	Where("status = ?", "active").
	Find(&users).Error
```

GORM 把结构体、表、字段、关联、事务、钩子、软删除等能力放到一套 API 里。它不是为了让 SQL 消失，而是让常见数据访问代码更少、更统一。

一句话概括：

```text
GORM 不是魔法，也不是 SQL 替代品。它是把 Go 结构体和数据库表连接起来的一层工程化封装。
```

### GORM 适合解决什么问题

GORM 适合这些场景：

```text
CRUD 很多，SQL 形态比较稳定
需要模型和表字段自动映射
需要统一处理创建时间、更新时间、软删除
需要事务、关联查询、预加载
需要在业务代码里少写重复 Scan 和拼接逻辑
```

不太适合这些场景：

```text
复杂报表 SQL 很多
强依赖数据库特有语法
需要极致控制执行计划
团队更习惯手写 SQL
```

实际项目里经常混用：

```text
普通 CRUD：GORM
复杂统计报表：原生 SQL
批量导入导出：按性能要求选择 GORM 或 database/sql
```

这样更现实。

### 安装 GORM

GORM 核心包：

```bash
go get gorm.io/gorm
```

数据库驱动按实际数据库选择。

SQLite：

```bash
go get gorm.io/driver/sqlite
```

MySQL：

```bash
go get gorm.io/driver/mysql
```

PostgreSQL：

```bash
go get gorm.io/driver/postgres
```

SQL Server：

```bash
go get gorm.io/driver/sqlserver
```

下面的 demo 使用 SQLite。原因很简单：不需要额外安装数据库服务，复制代码就能跑。

### 第一个最小示例

```go
package main

import (
	"fmt"
	"log"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

type Product struct {
	ID    uint
	Code  string
	Price uint
}

func main() {
	db, err := gorm.Open(sqlite.Open("demo.db"), &gorm.Config{})
	if err != nil {
		log.Fatal(err)
	}

	if err := db.AutoMigrate(&Product{}); err != nil {
		log.Fatal(err)
	}

	db.Create(&Product{Code: "P1001", Price: 199})

	var product Product
	if err := db.First(&product, "code = ?", "P1001").Error; err != nil {
		log.Fatal(err)
	}

	fmt.Printf("%d %s %d\n", product.ID, product.Code, product.Price)
}
```

运行：

```bash
go run main.go
```

输出类似：

```text
1 P1001 199
```

这段代码做了几件事：

```text
gorm.Open：打开数据库连接
AutoMigrate：根据结构体创建或更新表结构
Create：插入记录
First：查询第一条记录
Error：拿到执行错误
```

### gorm.DB 到底是什么

`*gorm.DB` 不是单纯的一条数据库连接。它更像 GORM 的操作入口，里面包含：

```text
底层连接池
当前 SQL 构建状态
配置项
错误信息
RowsAffected
回调链
上下文
```

常见代码：

```go
result := db.Where("status = ?", "active").Find(&users)
```

`result` 仍然是 `*gorm.DB`，里面可以取：

```go
result.Error
result.RowsAffected
```

所以 GORM 的错误处理通常长这样：

```go
if err := db.Create(&user).Error; err != nil {
	return err
}
```

或者：

```go
result := db.Where("age > ?", 18).Find(&users)
if result.Error != nil {
	return result.Error
}
fmt.Println(result.RowsAffected)
```

### 连接数据库

SQLite 连接：

```go
db, err := gorm.Open(sqlite.Open("app.db"), &gorm.Config{})
```

MySQL 连接：

```go
dsn := "root:123456@tcp(127.0.0.1:3306)/shop?charset=utf8mb4&parseTime=True&loc=Local"
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
```

PostgreSQL 连接：

```go
dsn := "host=127.0.0.1 user=postgres password=123456 dbname=shop port=5432 sslmode=disable TimeZone=Asia/Shanghai"
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
```

拿到底层 `*sql.DB` 后，可以配置连接池：

```go
sqlDB, err := db.DB()
if err != nil {
	return err
}

sqlDB.SetMaxOpenConns(50)
sqlDB.SetMaxIdleConns(10)
sqlDB.SetConnMaxLifetime(time.Hour)
```

GORM 负责 ORM，连接池仍然是标准库 `database/sql` 负责。

### 初始化配置怎么取舍

`gorm.Config` 里有几个常见配置项：

```go
db, err := gorm.Open(sqlite.Open("app.db"), &gorm.Config{
	Logger:                 logger.Default.LogMode(logger.Info),
	SkipDefaultTransaction: true,
	PrepareStmt:            true,
})
```

含义：

```text
Logger：控制 SQL 日志
SkipDefaultTransaction：跳过默认写事务
PrepareStmt：缓存预编译语句
```

GORM 默认会把写操作包在事务里，安全性更稳，但有额外开销。

如果业务已经在 service 层显式管理事务，或者某些写入对默认事务要求不高，可以考虑：

```go
SkipDefaultTransaction: true
```

`PrepareStmt` 会缓存预编译 SQL，适合重复执行相同 SQL 形态的场景：

```go
PrepareStmt: true
```

这两个配置都不要无脑开。更稳的做法是：

```text
先用默认配置保证正确性
压测发现瓶颈后再打开性能配置
核心写流程仍然用显式事务包住
```

### 模型和表怎么映射

GORM 默认按约定映射模型。

```go
type User struct {
	ID        uint
	Name      string
	Email     string
	CreatedAt time.Time
	UpdatedAt time.Time
}
```

默认规则：

```text
结构体 User -> 表 users
字段 Name -> 列 name
字段 CreatedAt -> 自动维护创建时间
字段 UpdatedAt -> 自动维护更新时间
字段 ID -> 默认主键
```

如果不想使用默认表名，可以实现 `TableName`：

```go
func (User) TableName() string {
	return "app_users"
}
```

也可以用 tag 控制字段：

```go
type User struct {
	ID    uint   `gorm:"primaryKey"`
	Name  string `gorm:"size:64;not null"`
	Email string `gorm:"size:128;uniqueIndex"`
	Age   int    `gorm:"default:18"`
}
```

常见 tag：

```text
primaryKey：主键
column：指定列名
type：指定数据库类型
size：字段长度
not null：非空
default：默认值
uniqueIndex：唯一索引
index：普通索引
autoCreateTime：自动创建时间
autoUpdateTime：自动更新时间
```

### gorm.Model 是什么

GORM 内置了一个基础模型：

```go
type Model struct {
	ID        uint `gorm:"primarykey"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

嵌入后，模型自动拥有这几个字段：

```go
type User struct {
	gorm.Model
	Name  string
	Email string
}
```

适合快速开发。

如果项目里主键类型、时间字段、删除字段有统一规范，更推荐自己定义基础字段：

```go
type BaseModel struct {
	ID        uint           `gorm:"primaryKey"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt gorm.DeletedAt `gorm:"index"`
}

type User struct {
	BaseModel
	Name string
}
```

### AutoMigrate 能做什么

`AutoMigrate` 会根据模型创建表、补充字段、补充索引：

```go
err := db.AutoMigrate(&User{}, &Order{})
```

它适合：

```text
本地 demo
测试环境
早期原型
小型内部工具
```

生产环境要谨慎。

原因是数据库变更需要可审计、可回滚、可灰度。生产环境更常见的做法是使用迁移工具管理 DDL，比如 Goose、Migrate、Atlas、Liquibase 等。

`AutoMigrate` 可以减少启动成本，但不要把它当成完整的数据库变更系统。

### Create：新增数据

新增单条记录：

```go
user := User{Name: "张三", Email: "zhangsan@example.com"}
if err := db.Create(&user).Error; err != nil {
	return err
}
fmt.Println(user.ID)
```

插入成功后，主键会回填到结构体里。

批量新增：

```go
users := []User{
	{Name: "张三", Email: "zhangsan@example.com"},
	{Name: "李四", Email: "lisi@example.com"},
}

if err := db.Create(&users).Error; err != nil {
	return err
}
```

指定批量大小：

```go
if err := db.CreateInBatches(users, 100).Error; err != nil {
	return err
}
```

只插入部分字段：

```go
db.Select("Name", "Email").Create(&user)
```

忽略部分字段：

```go
db.Omit("Password").Create(&user)
```

### First、Take、Last、Find 的区别

查询单条：

```go
var user User
err := db.First(&user, 1).Error
```

含义：

```text
First：按主键升序取第一条
Last：按主键降序取第一条
Take：不指定排序，取一条
Find：查询多条，也可以查单条，但查不到时不返回 ErrRecordNotFound
```

按条件查询：

```go
var user User
err := db.Where("email = ?", "zhangsan@example.com").First(&user).Error
```

查询列表：

```go
var users []User
err := db.Where("status = ?", "active").
	Order("id desc").
	Limit(20).
	Find(&users).Error
```

判断查不到：

```go
err := db.Where("email = ?", email).First(&user).Error
if errors.Is(err, gorm.ErrRecordNotFound) {
	return nil
}
if err != nil {
	return err
}
```

`ErrRecordNotFound` 只表示“没有记录”，不等于数据库异常。

### Where 条件写法

字符串条件：

```go
db.Where("age >= ? and status = ?", 18, "active").Find(&users)
```

结构体条件：

```go
db.Where(&User{Name: "张三"}).Find(&users)
```

map 条件：

```go
db.Where(map[string]any{
	"name":   "张三",
	"status": "active",
}).Find(&users)
```

IN 查询：

```go
db.Where("id in ?", []uint{1, 2, 3}).Find(&users)
```

LIKE 查询：

```go
db.Where("name like ?", "%张%").Find(&users)
```

范围查询：

```go
db.Where("created_at between ? and ?", start, end).Find(&users)
```

原生条件字符串不要拼接用户输入：

```go
db.Where("email = ?", email).First(&user)
```

不要这样写：

```go
db.Where("email = '" + email + "'").First(&user)
```

占位符能避免 SQL 注入，也能让 SQL 更稳定。

### Updates 的零值陷阱

更新单个字段：

```go
db.Model(&user).Update("name", "新名字")
```

用结构体更新多个字段：

```go
db.Model(&user).Updates(User{
	Name: "新名字",
	Age:  0,
})
```

这里有一个高频坑：结构体更新默认只更新非零值。

上面的 `Age: 0` 不会更新。

如果确实要把年龄更新成 `0`，可以用 map：

```go
db.Model(&user).Updates(map[string]any{
	"name": "新名字",
	"age":  0,
})
```

也可以用 `Select` 明确字段：

```go
db.Model(&user).
	Select("name", "age").
	Updates(User{Name: "新名字", Age: 0})
```

这一点很重要。很多“更新没生效”的问题，都出在零值上。

### Delete 和软删除

普通删除：

```go
db.Delete(&user)
```

如果模型里有 `gorm.DeletedAt` 字段，GORM 默认执行软删除：

```go
type User struct {
	ID        uint
	Name      string
	DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

软删除不是物理删除，而是更新 `deleted_at`。

查询时默认排除已软删记录：

```go
db.Find(&users)
```

查询包含软删记录：

```go
db.Unscoped().Find(&users)
```

物理删除：

```go
db.Unscoped().Delete(&user)
```

软删除适合业务数据留痕。日志表、临时表、导入中间表不一定需要软删除。

### Select、Omit 控制字段

只查部分字段：

```go
var users []User
db.Select("id", "name").Find(&users)
```

更新时只更新指定字段：

```go
db.Model(&user).Select("name", "age").Updates(input)
```

新增时忽略字段：

```go
db.Omit("password").Create(&user)
```

字段越多，越应该有意识地控制读写范围。

尤其是密码、密钥、大文本、JSON 字段，不要随手 `Find` 出全字段。

### Order、Limit、Offset 分页

基础分页：

```go
page := 2
size := 20
offset := (page - 1) * size

db.Order("id desc").
	Limit(size).
	Offset(offset).
	Find(&users)
```

查询总数：

```go
var total int64
db.Model(&User{}).Where("status = ?", "active").Count(&total)
```

分页列表一般分两步：

```go
query := db.Model(&User{}).Where("status = ?", "active")

var total int64
if err := query.Count(&total).Error; err != nil {
	return err
}

var users []User
if err := query.Order("id desc").Limit(size).Offset(offset).Find(&users).Error; err != nil {
	return err
}
```

复杂场景下要注意：复用 `query` 时，不要让上一段链式状态污染下一段。可以用 `Session(&gorm.Session{})` 创建新的会话。

### Scopes：复用查询条件

常见过滤条件可以封装成 Scope。

```go
func ActiveOnly(db *gorm.DB) *gorm.DB {
	return db.Where("status = ?", "active")
}

func ByKeyword(keyword string) func(*gorm.DB) *gorm.DB {
	return func(db *gorm.DB) *gorm.DB {
		if keyword == "" {
			return db
		}
		return db.Where("name like ?", "%"+keyword+"%")
	}
}
```

使用：

```go
db.Scopes(ActiveOnly, ByKeyword("张")).Find(&users)
```

分页也可以封装：

```go
func Paginate(page, size int) func(*gorm.DB) *gorm.DB {
	return func(db *gorm.DB) *gorm.DB {
		if page < 1 {
			page = 1
		}
		if size <= 0 || size > 100 {
			size = 20
		}
		return db.Offset((page - 1) * size).Limit(size)
	}
}
```

Scope 的好处是条件组合清晰，不用到处复制 `Where`。

### FindInBatches：大批量遍历

数据量很大时，不适合一次性 `Find` 到内存里。

比如给所有活跃用户刷新统计字段，可以分批处理：

```go
var users []User

err := db.Where("status = ?", "active").
	FindInBatches(&users, 500, func(tx *gorm.DB, batch int) error {
		for _, user := range users {
			fmt.Println("处理用户：", user.ID)
		}
		return nil
	}).Error
```

`FindInBatches` 每次查一批，回调处理完再查下一批。

适合：

```text
批量补数据
批量发通知
批量重建索引
导出大数据
```

批处理里仍然要注意单批大小、事务范围、失败重试和幂等。

### 关联关系：一对多

订单属于用户，用户有多个订单：

```go
type User struct {
	ID     uint
	Name   string
	Orders []Order
}

type Order struct {
	ID     uint
	UserID uint
	No     string
	Amount int64
}
```

创建用户和订单：

```go
user := User{
	Name: "张三",
	Orders: []Order{
		{No: "O1001", Amount: 19900},
		{No: "O1002", Amount: 29900},
	},
}

db.Create(&user)
```

预加载订单：

```go
var users []User
db.Preload("Orders").Find(&users)
```

`Preload` 通常会发两条 SQL：

```text
select * from users;
select * from orders where user_id in (...);
```

它能避免典型的 N + 1 查询。

### Association：操作关联数据

除了 `Preload` 查询关联，GORM 还提供了 Association Mode，用来追加、替换、删除关联关系。

继续使用用户和订单模型：

```go
var user User
db.First(&user, 1)

order := Order{No: "O1003", Amount: 39900}
db.Model(&user).Association("Orders").Append(&order)
```

替换关联：

```go
db.Model(&user).Association("Orders").Replace([]Order{
	{No: "O1004", Amount: 9900},
	{No: "O1005", Amount: 19900},
})
```

删除某个关联：

```go
db.Model(&user).Association("Orders").Delete(&order)
```

清空关联：

```go
db.Model(&user).Association("Orders").Clear()
```

注意一点：关联操作通常处理的是“关系”，不一定等于物理删除关联表里的数据。

比如一对多里删除关联，常见效果是把外键置空；多对多里删除关联，常见效果是删除中间表记录。具体行为要结合关系类型、外键约束和数据库表结构判断。

### Preload 和 Joins 怎么选

`Preload` 适合加载关联对象：

```go
db.Preload("Orders").Find(&users)
```

带条件的预加载：

```go
db.Preload("Orders", "status = ?", "paid").Find(&users)
```

也可以在预加载里排序：

```go
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
	return db.Order("created_at desc")
}).Find(&users)
```

嵌套预加载：

```go
db.Preload("Orders.OrderItems.Product").Find(&users)
```

`Joins` 适合按关联表过滤或排序：

```go
db.Joins("join orders on orders.user_id = users.id").
	Where("orders.amount > ?", 10000).
	Find(&users)
```

简单判断：

```text
只是把关联数据带出来：Preload
需要用关联表字段过滤、排序、聚合：Joins
复杂报表：原生 SQL
```

### 多对多关系

文章和标签是常见多对多：

```go
type Article struct {
	ID    uint
	Title string
	Tags  []Tag `gorm:"many2many:article_tags;"`
}

type Tag struct {
	ID   uint
	Name string
}
```

GORM 会使用中间表 `article_tags`。

创建：

```go
article := Article{
	Title: "GORM 入门",
	Tags: []Tag{
		{Name: "Go"},
		{Name: "ORM"},
	},
}
db.Create(&article)
```

查询并预加载：

```go
var articles []Article
db.Preload("Tags").Find(&articles)
```

多对多看起来方便，但生产项目里经常需要在中间表加字段，比如创建时间、排序、来源。这时可以显式定义中间表模型，而不是只靠默认 many2many。

### 事务

事务适合多个写操作必须同时成功或同时失败的场景。

```go
err := db.Transaction(func(tx *gorm.DB) error {
	if err := tx.Create(&order).Error; err != nil {
		return err
	}
	if err := tx.Model(&Account{}).
		Where("id = ? and balance >= ?", accountID, amount).
		Update("balance", gorm.Expr("balance - ?", amount)).Error; err != nil {
		return err
	}
	return nil
})
```

返回 `nil`，事务提交。

返回 `error`，事务回滚。

不要在事务回调里混用外层 `db`：

```go
db.Transaction(func(tx *gorm.DB) error {
	tx.Create(&order)
	db.Create(&log) // 这条不在事务里
	return nil
})
```

事务里统一使用 `tx`。

### 手动事务和保存点

大多数场景用 `db.Transaction` 就够了。

需要更细控制时，可以手动开启事务：

```go
tx := db.Begin()
if tx.Error != nil {
	return tx.Error
}

if err := tx.Create(&order).Error; err != nil {
	tx.Rollback()
	return err
}

if err := tx.Create(&orderLog).Error; err != nil {
	tx.Rollback()
	return err
}

if err := tx.Commit().Error; err != nil {
	return err
}
```

手动事务必须记住两件事：

```text
任何失败都要 Rollback
Commit 本身也要检查 Error
```

保存点适合“局部失败可以回退，外层事务继续”的场景：

```go
err := db.Transaction(func(tx *gorm.DB) error {
	if err := tx.Create(&order).Error; err != nil {
		return err
	}

	tx.SavePoint("before_log")
	if err := tx.Create(&orderLog).Error; err != nil {
		tx.RollbackTo("before_log")
	}

	return nil
})
```

这里订单创建成功后，即使日志写入失败，也可以回滚到保存点，再继续提交订单事务。

这种写法要谨慎使用。日志这类非核心数据，很多时候更适合异步写入，而不是让主事务承担额外复杂度。

### Context

Web 服务里，数据库操作应该带上请求上下文：

```go
func (r UserRepo) FindByID(ctx context.Context, id uint) (User, error) {
	var user User
	err := r.db.WithContext(ctx).First(&user, id).Error
	return user, err
}
```

这样请求取消、超时、链路追踪等信息可以传到数据库层。

常见写法：

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

err := db.WithContext(ctx).Find(&users).Error
```

### Logger 和 Debug

开发时可以打开详细 SQL 日志：

```go
db = db.Debug()
```

也可以在初始化时配置 logger：

```go
db, err := gorm.Open(sqlite.Open("app.db"), &gorm.Config{
	Logger: logger.Default.LogMode(logger.Info),
})
```

生产环境不建议无脑打印所有 SQL。更常见的是设置慢 SQL 阈值、错误级别日志、脱敏输出。

### DryRun：只生成 SQL 不执行

想查看 GORM 会生成什么 SQL，可以用 DryRun：

```go
stmt := db.Session(&gorm.Session{DryRun: true}).
	Where("email = ?", "zhangsan@example.com").
	First(&User{}).Statement

fmt.Println(stmt.SQL.String())
fmt.Println(stmt.Vars)
```

DryRun 适合调试复杂链式查询。

### Hook：模型钩子

GORM 支持在创建、更新、删除、查询前后执行钩子。

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
	if u.Email == "" {
		return fmt.Errorf("邮箱不能为空")
	}
	return nil
}
```

创建用户时会自动执行：

```go
db.Create(&user)
```

Hook 适合做和模型强相关的事情：

```text
默认值补充
字段格式化
基础校验
审计字段
```

不适合塞复杂业务流程。复杂业务放到 service 层更清楚。

### 实战 Demo：订单系统 Repository

下面写一个完整可运行示例：用户下单、扣减库存、查询订单列表。

项目结构：

```text
gorm-shop/
├── go.mod
└── main.go
```

初始化：

```bash
mkdir gorm-shop
cd gorm-shop
go mod init gorm-shop
go get gorm.io/gorm
go get gorm.io/driver/sqlite
```

完整代码：

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"time"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

type User struct {
	ID        uint           `gorm:"primaryKey"`
	Name      string         `gorm:"size:64;not null"`
	Email     string         `gorm:"size:128;uniqueIndex;not null"`
	Orders    []Order
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt gorm.DeletedAt `gorm:"index"`
}

type Product struct {
	ID        uint   `gorm:"primaryKey"`
	Name      string `gorm:"size:128;not null"`
	Stock     int    `gorm:"not null"`
	PriceCent int64  `gorm:"not null"`
	CreatedAt time.Time
	UpdatedAt time.Time
}

type Order struct {
	ID         uint   `gorm:"primaryKey"`
	OrderNo    string `gorm:"size:32;uniqueIndex;not null"`
	UserID     uint   `gorm:"index;not null"`
	User       User
	TotalCent  int64  `gorm:"not null"`
	Status     string `gorm:"size:24;not null"`
	OrderItems []OrderItem
	CreatedAt  time.Time
	UpdatedAt  time.Time
}

type OrderItem struct {
	ID        uint `gorm:"primaryKey"`
	OrderID   uint `gorm:"index;not null"`
	ProductID uint `gorm:"index;not null"`
	Product   Product
	Quantity  int   `gorm:"not null"`
	PriceCent int64 `gorm:"not null"`
}

type CreateOrderInput struct {
	UserID    uint
	ProductID uint
	Quantity  int
}

type OrderRepo struct {
	db *gorm.DB
}

func NewOrderRepo(db *gorm.DB) OrderRepo {
	return OrderRepo{db: db}
}

func (r OrderRepo) CreateOrder(ctx context.Context, input CreateOrderInput) (Order, error) {
	if input.Quantity <= 0 {
		return Order{}, fmt.Errorf("购买数量必须大于 0")
	}

	var created Order

	err := r.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
		var user User
		if err := tx.First(&user, input.UserID).Error; err != nil {
			if errors.Is(err, gorm.ErrRecordNotFound) {
				return fmt.Errorf("用户不存在：%d", input.UserID)
			}
			return err
		}

		var product Product
		if err := tx.First(&product, input.ProductID).Error; err != nil {
			if errors.Is(err, gorm.ErrRecordNotFound) {
				return fmt.Errorf("商品不存在：%d", input.ProductID)
			}
			return err
		}

		if product.Stock < input.Quantity {
			return fmt.Errorf("库存不足：剩余 %d，需要 %d", product.Stock, input.Quantity)
		}

		result := tx.Model(&Product{}).
			Where("id = ? and stock >= ?", product.ID, input.Quantity).
			Update("stock", gorm.Expr("stock - ?", input.Quantity))
		if result.Error != nil {
			return result.Error
		}
		if result.RowsAffected == 0 {
			return fmt.Errorf("库存不足或商品已变化")
		}

		order := Order{
			OrderNo:   fmt.Sprintf("NO%d", time.Now().UnixNano()),
			UserID:    user.ID,
			TotalCent: product.PriceCent * int64(input.Quantity),
			Status:    "created",
			OrderItems: []OrderItem{
				{
					ProductID: product.ID,
					Quantity:  input.Quantity,
					PriceCent: product.PriceCent,
				},
			},
		}

		if err := tx.Create(&order).Error; err != nil {
			return err
		}

		created = order
		return nil
	})

	return created, err
}

func (r OrderRepo) ListOrders(ctx context.Context, userID uint, page, size int) ([]Order, int64, error) {
	if page < 1 {
		page = 1
	}
	if size <= 0 || size > 100 {
		size = 10
	}

	query := r.db.WithContext(ctx).Model(&Order{}).Where("user_id = ?", userID)

	var total int64
	if err := query.Count(&total).Error; err != nil {
		return nil, 0, err
	}

	var orders []Order
	err := query.
		Preload("OrderItems.Product").
		Order("id desc").
		Limit(size).
		Offset((page - 1) * size).
		Find(&orders).Error
	if err != nil {
		return nil, 0, err
	}

	return orders, total, nil
}

func openDB() (*gorm.DB, error) {
	return gorm.Open(sqlite.Open("file::memory:?cache=shared"), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Silent),
	})
}

func migrate(db *gorm.DB) error {
	return db.AutoMigrate(&User{}, &Product{}, &Order{}, &OrderItem{})
}

func seed(db *gorm.DB) error {
	user := User{Name: "张三", Email: "zhangsan@example.com"}
	if err := db.Create(&user).Error; err != nil {
		return err
	}

	products := []Product{
		{Name: "机械键盘", Stock: 10, PriceCent: 29900},
		{Name: "无线鼠标", Stock: 20, PriceCent: 9900},
	}
	return db.Create(&products).Error
}

func main() {
	db, err := openDB()
	if err != nil {
		log.Fatal(err)
	}
	if err := migrate(db); err != nil {
		log.Fatal(err)
	}
	if err := seed(db); err != nil {
		log.Fatal(err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	repo := NewOrderRepo(db)

	order, err := repo.CreateOrder(ctx, CreateOrderInput{
		UserID:    1,
		ProductID: 1,
		Quantity:  2,
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("创建订单成功：%s，金额：%.2f\n", order.OrderNo, float64(order.TotalCent)/100)

	orders, total, err := repo.ListOrders(ctx, 1, 1, 10)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("订单总数：%d\n", total)
	for _, item := range orders {
		fmt.Printf("订单：%s，状态：%s，明细数：%d\n", item.OrderNo, item.Status, len(item.OrderItems))
		for _, detail := range item.OrderItems {
			fmt.Printf("- 商品：%s，数量：%d，单价：%.2f\n",
				detail.Product.Name,
				detail.Quantity,
				float64(detail.PriceCent)/100,
			)
		}
	}

	var product Product
	if err := db.First(&product, 1).Error; err != nil {
		log.Fatal(err)
	}
	fmt.Printf("剩余库存：%d\n", product.Stock)
}
```

运行：

```bash
go run main.go
```

输出类似：

```text
创建订单成功：NO1720000000000000000，金额：598.00
订单总数：1
订单：NO1720000000000000000，状态：created，明细数：1
- 商品：机械键盘，数量：2，单价：299.00
剩余库存：8
```

这个 demo 涵盖了几个真实项目里常见的点：

```text
模型映射
AutoMigrate
Create
First
ErrRecordNotFound
事务
库存扣减
RowsAffected 判断
Preload 多级预加载
分页查询
Repository 分层
Context
```

### 实战拆解：为什么扣库存要看 RowsAffected

扣库存代码：

```go
result := tx.Model(&Product{}).
	Where("id = ? and stock >= ?", product.ID, input.Quantity).
	Update("stock", gorm.Expr("stock - ?", input.Quantity))
if result.Error != nil {
	return result.Error
}
if result.RowsAffected == 0 {
	return fmt.Errorf("库存不足或商品已变化")
}
```

关键在这个条件：

```sql
where id = ? and stock >= ?
```

库存足够时才更新。

如果并发下库存被其他请求扣掉，`RowsAffected` 会变成 0，事务直接返回错误。

这比先查库存、再无条件更新更稳。

### 实战拆解：Preload("OrderItems.Product")

查询订单时：

```go
Preload("OrderItems.Product")
```

意思是：

```text
加载订单
加载订单明细
再加载订单明细里的商品
```

这样最终打印商品名时，不需要手动再查一次商品表。

如果没有预加载，循环订单明细时再查商品，很容易变成 N + 1 查询。

### 原生 SQL

GORM 不排斥原生 SQL。

查询：

```go
type OrderSummary struct {
	UserID uint
	Total  int64
}

var rows []OrderSummary
err := db.Raw(`
	select user_id, sum(total_cent) as total
	from orders
	where status = ?
	group by user_id
`, "created").Scan(&rows).Error
```

执行：

```go
err := db.Exec("update products set stock = stock + ? where id = ?", 10, 1).Error
```

复杂统计、数据库专有函数、大批量修复脚本，用原生 SQL 很正常。

### Generics API 简单了解

较新的 GORM 文档里提供了 Generics API，用法更偏类型安全。

示例：

```go
ctx := context.Background()

user, err := gorm.G[User](db).
	Where("email = ?", "zhangsan@example.com").
	First(ctx)
```

创建：

```go
user := User{Name: "张三", Email: "zhangsan@example.com"}
err := gorm.G[User](db).Create(ctx, &user)
```

传统 API 仍然大量存在：

```go
var user User
err := db.Where("email = ?", "zhangsan@example.com").First(&user).Error
```

新项目可以关注 Generics API，存量项目使用传统 API 也完全正常。核心思想没变：模型、条件、错误、事务仍然是那几块。

### 常见错误一：忽略 Error

错误写法：

```go
db.Create(&user)
```

正确写法：

```go
if err := db.Create(&user).Error; err != nil {
	return err
}
```

GORM 的链式调用看起来很顺，但数据库操作随时可能失败。唯一约束、连接超时、字段类型错误、事务回滚都需要处理。

### 常见错误二：Find 查不到不报错

```go
var user User
err := db.Where("email = ?", email).Find(&user).Error
```

这里查不到时，`err` 可能是 `nil`。

需要“查不到就是错误”时，用 `First`、`Take`、`Last`：

```go
err := db.Where("email = ?", email).First(&user).Error
if errors.Is(err, gorm.ErrRecordNotFound) {
	return fmt.Errorf("用户不存在")
}
```

需要查询列表时，用 `Find`：

```go
var users []User
err := db.Where("status = ?", "active").Find(&users).Error
```

列表为空不是错误。

### 常见错误三：Save 乱用

`Save` 会保存所有字段，容易把零值也写入数据库。

```go
db.Save(&user)
```

更推荐明确表达意图：

```go
db.Model(&user).Update("name", "新名字")
```

或者：

```go
db.Model(&user).Updates(map[string]any{
	"name": "新名字",
	"age":  0,
})
```

更新接口尤其要小心：输入结构里没传的字段，不应该被零值覆盖。

### 常见错误四：把 DTO 当模型直接更新

请求体：

```go
type UpdateUserRequest struct {
	Name string `json:"name"`
	Age  int    `json:"age"`
}
```

直接这样更新有风险：

```go
db.Model(&user).Updates(req)
```

如果 `Age` 没传，默认是 `0`，业务含义可能完全不同。

更稳的方式是用指针表达“传没传”：

```go
type UpdateUserRequest struct {
	Name *string `json:"name"`
	Age  *int    `json:"age"`
}
```

再按字段组装 map：

```go
updates := map[string]any{}
if req.Name != nil {
	updates["name"] = *req.Name
}
if req.Age != nil {
	updates["age"] = *req.Age
}

db.Model(&user).Updates(updates)
```

### 常见错误五：关联查询造成 N + 1

错误写法：

```go
var users []User
db.Find(&users)

for _, user := range users {
	var orders []Order
	db.Where("user_id = ?", user.ID).Find(&orders)
}
```

用户 100 个，就可能查 101 次。

改成：

```go
var users []User
db.Preload("Orders").Find(&users)
```

关联数据较大时，不要盲目预加载全部字段。可以拆成专门查询，或者用 `Select` 控制字段。

### 常见错误六：事务里误用 db

错误写法：

```go
db.Transaction(func(tx *gorm.DB) error {
	if err := tx.Create(&order).Error; err != nil {
		return err
	}
	return db.Create(&orderLog).Error
})
```

`orderLog` 使用的是外层 `db`，不在当前事务里。

正确写法：

```go
db.Transaction(func(tx *gorm.DB) error {
	if err := tx.Create(&order).Error; err != nil {
		return err
	}
	return tx.Create(&orderLog).Error
})
```

事务内始终使用 `tx`。

### 常见错误七：把 AutoMigrate 当生产迁移

`AutoMigrate` 很方便：

```go
db.AutoMigrate(&User{}, &Order{})
```

但生产数据库变更需要更多控制：

```text
变更脚本审查
发布时间窗口
回滚方案
字段回填策略
索引创建方式
数据量评估
```

`AutoMigrate` 适合 demo、测试和早期原型。生产环境更适合迁移脚本。

### 工程实践建议

GORM 项目可以按这个结构拆：

```text
internal/
  model/          GORM 模型
  repository/     数据访问
  service/        业务流程和事务边界
  database/       数据库初始化、连接池、日志配置
```

常见规则：

* 模型只描述表结构和简单钩子
* Repository 负责查询和持久化
* Service 负责业务流程和事务
* Handler 不直接拼 GORM 查询
* 所有数据库操作都检查 `Error`
* Web 请求路径统一使用 `WithContext`
* 更新接口谨慎处理零值
* 列表查询必须限制 `Limit`
* 大批量遍历优先考虑 `FindInBatches`
* 显式事务里始终使用 `tx`
* 复杂 SQL 可以用 `Raw`
* 生产环境不要依赖 `AutoMigrate` 管理全部变更
* 性能配置先压测再打开，不要一开始就堆满开关

### GORM 和 database/sql 怎么选

可以按这个思路判断：

```text
简单 CRUD 多：GORM
模型关联多：GORM
需要快速开发后台：GORM
复杂查询多：database/sql 或 sqlc
性能压测要求极高：手写 SQL 更可控
团队 SQL 能力强且规范成熟：手写 SQL 也很好
```

GORM 的价值是减少常规数据访问代码，不是完全替代 SQL。

### 总结

GORM 最重要的不是 API 背诵，而是边界感。

它擅长：

```text
模型映射
常规 CRUD
事务封装
关联预加载
软删除
钩子
日志和调试
```

它不擅长：

```text
替代所有 SQL
自动设计表结构
解决所有性能问题
无脑处理复杂报表
代替数据库迁移流程
```

真正稳定的写法通常是：

```text
GORM 做常规读写
Repository 收口查询
Service 管事务
复杂 SQL 直接 Raw
生产 DDL 走迁移工具
```

这样用，GORM 才是提高效率的工具，而不是隐藏问题的黑盒。
