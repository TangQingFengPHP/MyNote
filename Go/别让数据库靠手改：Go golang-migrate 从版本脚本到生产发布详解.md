### 简介

业务代码有 Git 记录，数据库结构也应该有版本记录。

没有迁移工具时，数据库变更经常变成这样：

```text
本地手动加了一个字段
测试环境忘了建索引
生产环境不知道跑到哪个版本
上线失败后不知道怎么回滚
多人同时改表，脚本顺序互相打架
```

`golang-migrate` 解决的就是这个问题。

它把每次数据库结构变更写成一对文件：

```text
000001_create_users_table.up.sql
000001_create_users_table.down.sql
```

`up.sql` 负责升级，`down.sql` 负责回滚。执行过哪个版本、当前是不是失败状态，都会记录在数据库里的 `schema_migrations` 表中。

一句话概括：

```text
golang-migrate 让数据库结构变更像代码一样可追踪、可执行、可回滚。
```

### golang-migrate 适合什么场景

适合：

```text
Go 项目需要管理数据库表结构
生产环境不想靠手动改表
需要把 DDL 纳入 Git 评审
需要支持回滚或指定版本迁移
需要在 CI/CD、Docker、Kubernetes 里自动执行迁移
GORM 项目想替代生产环境 AutoMigrate
```

不适合：

```text
只有一次性本地 demo
数据库结构完全不会变化
团队已经有成熟 Liquibase、Flyway、Atlas 流程
复杂数据修复更适合单独脚本和人工审核
```

`golang-migrate` 不负责设计表结构，也不负责判断 SQL 是否安全。它负责按版本顺序执行迁移，并记录执行状态。

### 和 GORM AutoMigrate 的区别

GORM `AutoMigrate` 很方便：

```go
db.AutoMigrate(&User{}, &Order{})
```

但它更适合开发阶段快速建表。

生产环境更看重这些东西：

```text
DDL 变更能评审
执行顺序固定
线上跑过什么版本能查
失败后能定位 dirty 状态
回滚脚本提前准备
大表变更能拆步骤
```

对比一下：

| 维度 | golang-migrate | GORM AutoMigrate |
|---|---|---|
| 变更来源 | SQL 迁移文件 | Go 结构体推断 |
| 版本记录 | 有 `schema_migrations` | 无完整迁移版本 |
| 回滚 | 通过 `down.sql` | 不提供回滚 |
| 评审 | SQL 文件进 Git | 结构体变化比较隐蔽 |
| 生产可控性 | 高 | 低 |
| 常见定位 | 生产迁移 | 本地开发、测试原型 |

常见组合是：

```text
golang-migrate 管表结构
GORM 做 CRUD 和查询
```

### 安装 CLI

macOS 可以用 Homebrew：

```bash
brew install golang-migrate
```

Go 安装方式需要按数据库驱动打 tag。

PostgreSQL：

```bash
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

MySQL：

```bash
go install -tags 'mysql' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

SQLite：

```bash
go install -tags 'sqlite3' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

多个数据库一起支持：

```bash
go install -tags 'postgres mysql sqlite3' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

验证：

```bash
migrate -version
```

Docker 方式：

```bash
docker run --rm migrate/migrate -version
```

### 迁移文件命名规则

文件名格式：

```text
{version}_{title}.up.{extension}
{version}_{title}.down.{extension}
```

例子：

```text
000001_create_users_table.up.sql
000001_create_users_table.down.sql
000002_create_orders_table.up.sql
000002_create_orders_table.down.sql
```

含义：

```text
version：版本号，必须能排序
title：描述文字，只是方便阅读
up：正向迁移
down：反向回滚
extension：文件类型，SQL 数据库通常是 sql
```

版本号可以是递增数字，也可以是时间戳。

推荐团队项目使用 `-seq` 生成递增数字：

```bash
migrate create -ext sql -dir db/migrations -seq create_users_table
```

会生成：

```text
db/migrations/000001_create_users_table.up.sql
db/migrations/000001_create_users_table.down.sql
```

时间戳也可以：

```bash
migrate create -ext sql -dir db/migrations create_users_table
```

时间戳能减少多人冲突，但版本号比较长。序列号更清楚，但合并代码时要注意冲突。

### up 和 down 怎么写

`up.sql` 写正向变更：

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

`down.sql` 写反向回滚：

```sql
DROP TABLE IF EXISTS users;
```

新增字段：

```sql
ALTER TABLE users ADD COLUMN status TEXT NOT NULL DEFAULT 'active';
```

回滚字段要看数据库能力。SQLite 很多版本不支持直接 `DROP COLUMN`，可能需要重建表。MySQL、PostgreSQL 通常可以写：

```sql
ALTER TABLE users DROP COLUMN status;
```

迁移脚本不要只想着“怎么上去”，也要提前想“怎么退回来”。

### schema_migrations 表

执行迁移后，数据库里会出现一张版本表，默认名字是：

```text
schema_migrations
```

常见字段含义：

```text
version：当前迁移版本
dirty：是否处于失败状态
```

正常状态可能是：

```text
version = 2
dirty = false
```

迁移执行中途失败，可能变成：

```text
version = 2
dirty = true
```

dirty 状态下，继续 `up` 或 `down` 通常会失败。需要先人工确认数据库真实状态，再用 `force` 修正版本。

### CLI 常用命令

下面用 SQLite 举例，数据库文件是 `app.db`，迁移目录是 `db/migrations`。

应用所有未执行迁移：

```bash
migrate -path db/migrations -database "sqlite3://app.db" up
```

只应用接下来的 1 个版本：

```bash
migrate -path db/migrations -database "sqlite3://app.db" up 1
```

回滚最近 1 个版本：

```bash
migrate -path db/migrations -database "sqlite3://app.db" down 1
```

回滚所有版本：

```bash
migrate -path db/migrations -database "sqlite3://app.db" down -all
```

跳转到指定版本：

```bash
migrate -path db/migrations -database "sqlite3://app.db" goto 2
```

查看当前版本：

```bash
migrate -path db/migrations -database "sqlite3://app.db" version
```

强制设置版本：

```bash
migrate -path db/migrations -database "sqlite3://app.db" force 2
```

删除数据库所有内容：

```bash
migrate -path db/migrations -database "sqlite3://app.db" drop -f
```

`drop -f` 很危险，只适合本地测试库或明确允许清空的环境。

### MySQL 和 PostgreSQL DSN

MySQL：

```bash
migrate \
  -path db/migrations \
  -database "mysql://root:123456@tcp(127.0.0.1:3306)/shop?charset=utf8mb4&parseTime=True&loc=Local" \
  up
```

PostgreSQL：

```bash
migrate \
  -path db/migrations \
  -database "postgres://postgres:123456@127.0.0.1:5432/shop?sslmode=disable" \
  up
```

连接串里如果有特殊字符，比如密码包含 `@`、`/`、`#`，需要 URL Encode。

更稳的写法是把数据库 URL 放进环境变量：

```bash
export DATABASE_URL="postgres://postgres:123456@127.0.0.1:5432/shop?sslmode=disable"
migrate -path db/migrations -database "$DATABASE_URL" up
```

### 实战 Demo：用 SQLite 管理用户和订单表

项目结构：

```text
migrate-demo/
├── db/
│   └── migrations/
│       ├── 000001_create_users_table.up.sql
│       ├── 000001_create_users_table.down.sql
│       ├── 000002_create_orders_table.up.sql
│       └── 000002_create_orders_table.down.sql
├── go.mod
└── main.go
```

初始化：

```bash
mkdir migrate-demo
cd migrate-demo
go mod init migrate-demo
go get github.com/golang-migrate/migrate/v4
go get github.com/golang-migrate/migrate/v4/database/sqlite3
go get github.com/golang-migrate/migrate/v4/source/file
go get github.com/mattn/go-sqlite3
```

创建迁移目录：

```bash
mkdir -p db/migrations
```

生成迁移文件：

```bash
migrate create -ext sql -dir db/migrations -seq create_users_table
migrate create -ext sql -dir db/migrations -seq create_orders_table
```

### 000001_create_users_table.up.sql

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

### 000001_create_users_table.down.sql

```sql
DROP TABLE IF EXISTS users;
```

### 000002_create_orders_table.up.sql

```sql
CREATE TABLE orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    order_no TEXT NOT NULL UNIQUE,
    amount_cent INTEGER NOT NULL,
    status TEXT NOT NULL DEFAULT 'created',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
```

### 000002_create_orders_table.down.sql

```sql
DROP TABLE IF EXISTS orders;
```

执行 CLI 迁移：

```bash
migrate -path db/migrations -database "sqlite3://app.db" up
```

查看版本：

```bash
migrate -path db/migrations -database "sqlite3://app.db" version
```

回滚一步：

```bash
migrate -path db/migrations -database "sqlite3://app.db" down 1
```

再升级：

```bash
migrate -path db/migrations -database "sqlite3://app.db" up
```

### Go 代码内执行迁移

有些项目会在服务启动时自动执行迁移。

完整 `main.go`：

```go
package main

import (
	"database/sql"
	"errors"
	"fmt"
	"log"
	"os"

	"github.com/golang-migrate/migrate/v4"
	"github.com/golang-migrate/migrate/v4/database/sqlite3"
	_ "github.com/golang-migrate/migrate/v4/source/file"
	_ "github.com/mattn/go-sqlite3"
)

func main() {
	dbPath := "app.db"
	if len(os.Args) > 1 {
		dbPath = os.Args[1]
	}

	db, err := sql.Open("sqlite3", dbPath)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	m, err := newMigrator(db, "file://db/migrations")
	if err != nil {
		log.Fatal(err)
	}
	defer m.Close()

	if err := applyMigrations(m); err != nil {
		log.Fatal(err)
	}

	if err := seedAndQuery(db); err != nil {
		log.Fatal(err)
	}
}

func newMigrator(db *sql.DB, sourceURL string) (*migrate.Migrate, error) {
	driver, err := sqlite3.WithInstance(db, &sqlite3.Config{})
	if err != nil {
		return nil, err
	}

	m, err := migrate.NewWithDatabaseInstance(sourceURL, "sqlite3", driver)
	if err != nil {
		return nil, err
	}
	return m, nil
}

func applyMigrations(m *migrate.Migrate) error {
	if err := m.Up(); err != nil {
		if errors.Is(err, migrate.ErrNoChange) {
			fmt.Println("迁移状态：没有新版本")
			return nil
		}
		return fmt.Errorf("执行迁移失败：%w", err)
	}

	version, dirty, err := m.Version()
	if err != nil {
		return err
	}
	fmt.Printf("迁移状态：version=%d dirty=%v\n", version, dirty)
	return nil
}

func seedAndQuery(db *sql.DB) error {
	_, err := db.Exec(`
		INSERT INTO users(email, name)
		VALUES(?, ?)
		ON CONFLICT(email) DO UPDATE SET name = excluded.name
	`, "zhangsan@example.com", "张三")
	if err != nil {
		return err
	}

	_, err = db.Exec(`
		INSERT INTO orders(user_id, order_no, amount_cent, status)
		SELECT id, ?, ?, ?
		FROM users
		WHERE email = ?
		ON CONFLICT(order_no) DO NOTHING
	`, "NO1001", 29900, "created", "zhangsan@example.com")
	if err != nil {
		return err
	}

	rows, err := db.Query(`
		SELECT u.email, u.name, o.order_no, o.amount_cent
		FROM users u
		JOIN orders o ON o.user_id = u.id
		ORDER BY o.id
	`)
	if err != nil {
		return err
	}
	defer rows.Close()

	for rows.Next() {
		var email string
		var name string
		var orderNo string
		var amountCent int64
		if err := rows.Scan(&email, &name, &orderNo, &amountCent); err != nil {
			return err
		}
		fmt.Printf("%s %s %s %.2f\n", email, name, orderNo, float64(amountCent)/100)
	}
	return rows.Err()
}
```

运行：

```bash
go run main.go
```

输出类似：

```text
迁移状态：version=2 dirty=false
zhangsan@example.com 张三 NO1001 299.00
```

再次运行：

```bash
go run main.go
```

输出类似：

```text
迁移状态：没有新版本
zhangsan@example.com 张三 NO1001 299.00
```

这里复用了已有的 `*sql.DB`：

```go
driver, err := sqlite3.WithInstance(db, &sqlite3.Config{})
```

再交给 migrate：

```go
migrate.NewWithDatabaseInstance(sourceURL, "sqlite3", driver)
```

这样不会因为迁移逻辑额外创建一套数据库连接。

注意 `m.Close()` 的时机。迁移器持有数据库驱动，过早关闭可能连带关闭底层连接。迁移后还要继续使用同一个 `*sql.DB` 时，把 `m.Close()` 放到业务使用结束之后。

### ErrNoChange 不是错误

`m.Up()` 没有新脚本可执行时，会返回：

```go
migrate.ErrNoChange
```

这通常是正常状态。

常见处理：

```go
if err := m.Up(); err != nil {
	if errors.Is(err, migrate.ErrNoChange) {
		return nil
	}
	return err
}
```

不要把 `ErrNoChange` 当成启动失败。

### Dirty 状态怎么处理

迁移执行到一半失败时，数据库会进入 dirty 状态。

常见报错类似：

```text
Dirty database version 2. Fix and force version.
```

处理步骤：

```text
查看失败的 migration 文件
确认数据库是否已经部分执行
手动修复数据库到某个确定状态
使用 force 把版本号调整到真实状态
重新执行 migrate up
```

比如版本 2 的脚本执行失败，确认数据库实际上还停在版本 1，可以执行：

```bash
migrate -path db/migrations -database "sqlite3://app.db" force 1
migrate -path db/migrations -database "sqlite3://app.db" up
```

如果确认版本 2 的变更已经完整完成，只是迁移工具状态没写干净，可以执行：

```bash
migrate -path db/migrations -database "sqlite3://app.db" force 2
migrate -path db/migrations -database "sqlite3://app.db" up
```

`force` 只改版本记录，不会自动修复表结构。执行前必须确认数据库真实状态。

### Go 代码识别 Dirty

`ErrDirty` 是一个具体错误类型，不是固定哨兵值。

可以这样判断：

```go
if err := m.Up(); err != nil {
	var dirty migrate.ErrDirty
	if errors.As(err, &dirty) {
		return fmt.Errorf("数据库迁移处于 dirty 状态，version=%d", dirty.Version)
	}
	if errors.Is(err, migrate.ErrNoChange) {
		return nil
	}
	return err
}
```

dirty 状态不要在程序里自动 `Force`。更稳的方式是停止启动，交给发布流程或运维脚本处理。

### Steps、Goto、Force 对应的 Go API

CLI 命令和 Go API 基本能对应起来：

```text
up             -> m.Up()
up 2           -> m.Steps(2)
down 1         -> m.Steps(-1)
goto 3         -> m.Migrate(3)
force 2        -> m.Force(2)
version        -> m.Version()
```

示例：

```go
if err := m.Steps(-1); err != nil {
	return err
}
```

自动回滚在生产服务启动阶段很少使用。更常见的是发布失败后由流水线或人工确认后执行。

### embed.FS：把迁移文件打进二进制

文件源 `file://db/migrations` 依赖运行目录存在迁移文件。

如果希望迁移文件跟着二进制一起发布，可以使用 `embed.FS` 和 `source/iofs`。

```go
package main

import (
	"database/sql"
	"embed"
	"errors"
	"fmt"
	"log"

	"github.com/golang-migrate/migrate/v4"
	"github.com/golang-migrate/migrate/v4/database/sqlite3"
	"github.com/golang-migrate/migrate/v4/source/iofs"
	_ "github.com/mattn/go-sqlite3"
)

//go:embed db/migrations/*.sql
var migrationFiles embed.FS

func main() {
	db, err := sql.Open("sqlite3", "app.db")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	sourceDriver, err := iofs.New(migrationFiles, "db/migrations")
	if err != nil {
		log.Fatal(err)
	}

	dbDriver, err := sqlite3.WithInstance(db, &sqlite3.Config{})
	if err != nil {
		log.Fatal(err)
	}

	m, err := migrate.NewWithInstance("iofs", sourceDriver, "sqlite3", dbDriver)
	if err != nil {
		log.Fatal(err)
	}
	defer m.Close()

	if err := m.Up(); err != nil && !errors.Is(err, migrate.ErrNoChange) {
		log.Fatal(err)
	}

	version, dirty, err := m.Version()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("version=%d dirty=%v\n", version, dirty)
}
```

适合：

```text
单二进制部署
容器镜像内不想单独复制 migrations 目录
命令行工具自带初始化数据库能力
```

### Docker Compose 里执行 migrate

迁移可以作为独立容器执行。

示例：

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: 123456
      POSTGRES_DB: shop
    ports:
      - "5432:5432"

  migrate:
    image: migrate/migrate
    volumes:
      - ./db/migrations:/migrations
    command:
      [
        "-path",
        "/migrations",
        "-database",
        "postgres://postgres:123456@postgres:5432/shop?sslmode=disable",
        "up"
      ]
    depends_on:
      - postgres
```

实际生产环境还要处理数据库就绪检查。`depends_on` 只保证容器启动顺序，不保证数据库已经能连接。

### CI/CD 中怎么放

常见发布流程：

```text
构建应用镜像
启动数据库或连接目标数据库
执行 migrate up
迁移成功后发布新应用
迁移失败则停止发布
```

GitHub Actions 或其他流水线里，迁移命令通常类似：

```bash
migrate \
  -path db/migrations \
  -database "$DATABASE_URL" \
  -verbose \
  up
```

Kubernetes 里常见两种做法：

```text
Job：发布前跑一次迁移
Init Container：主容器启动前先跑迁移
```

更推荐 Job 或发布流水线控制迁移。应用多副本同时启动时，如果每个副本都尝试迁移，虽然工具有锁机制，但发布流程会变得更难观察。

### 大表变更怎么拆

小表直接 `ALTER TABLE` 问题不大。大表要谨慎。

比如给大表新增一个非空字段，不建议一步到位：

```sql
ALTER TABLE users ADD COLUMN status TEXT NOT NULL DEFAULT 'active';
```

更稳的拆法：

```text
第 1 个版本：新增可空字段
第 2 个版本：后台分批回填历史数据
第 3 个版本：应用代码开始写新字段
第 4 个版本：加 NOT NULL 约束
```

索引也一样。PostgreSQL 大表索引可以考虑 `CREATE INDEX CONCURRENTLY`，但这类语句不能放在普通事务里。不同数据库限制不同，迁移文件要按数据库特性写。

### down 文件怎么写才靠谱

好的 `down.sql` 不是随便写个 `DROP TABLE`。

新增表：

```sql
DROP TABLE IF EXISTS orders;
```

新增索引：

```sql
DROP INDEX IF EXISTS idx_orders_status;
```

新增字段：

```sql
ALTER TABLE orders DROP COLUMN status;
```

新增枚举值、数据修复、字段拆分这类变更，回滚可能很难。确实无法回滚时，也要保留 `down.sql`，里面写清楚原因：

```sql
-- irreversible migration: historical data cannot be reconstructed
```

不要创建 0 字节空文件。某些数据库会把空查询当成错误。

### 迁移文件提交后不要修改

已经合并并执行过的迁移文件，不要再改。

错误做法：

```text
修改 000003_add_status.up.sql
重新提交
```

问题在于：

```text
已经执行过版本 3 的环境不会重新执行
没执行过版本 3 的环境会执行新内容
不同环境数据库结构开始分叉
```

正确做法：

```text
新增 000004_fix_status_default.up.sql
新增 000004_fix_status_default.down.sql
```

迁移文件一旦进入主分支，就应该像发布出去的接口一样对待。

### 多人协作的版本冲突

两个分支都创建了：

```text
000006_add_user_status.up.sql
000006_create_payment_table.up.sql
```

合并时就会冲突。

处理方式：

```text
保留先合并的版本号
后合并的分支重新生成新版本号
检查 up/down 是否还能按顺序执行
重新跑 up、down、up
```

提交迁移前建议跑：

```bash
migrate -path db/migrations -database "$DATABASE_URL" up
migrate -path db/migrations -database "$DATABASE_URL" down 1
migrate -path db/migrations -database "$DATABASE_URL" up
```

这样能提前发现 `down.sql` 写错的问题。

### 常见错误一：只写 up，不认真写 down

只写升级，不写回滚，发布失败时就会很被动。

至少要做到：

```text
能回滚表、字段、索引
不能回滚的数据变更写明原因
上线前测试 down 再 up
```

### 常见错误二：dirty 后直接 force

`force` 不是修复数据库结构。

错误流程：

```bash
migrate force 2
migrate up
```

缺少了最重要的一步：检查数据库真实状态。

正确流程：

```text
查看失败 SQL
查看已执行到哪一步
手动补齐或撤销半成品
确认真实版本
force 到真实版本
继续迁移
```

### 常见错误三：在应用多副本里同时自动迁移

应用启动时自动 `m.Up()` 很方便，但多副本部署要小心。

更推荐：

```text
单独迁移 Job
发布流水线 migration step
应用启动前明确完成迁移
```

如果必须应用启动自动迁移，要确认目标数据库驱动支持锁，并且启动失败能被监控发现。

### 常见错误四：把大量数据修复塞进 DDL 迁移

迁移文件可以写 DML，但不要把几百万行回填直接塞进普通迁移里。

更稳的做法：

```text
DDL 迁移先加字段或表
后台任务分批回填数据
验证完成后再加约束或切换代码
```

这样失败范围更小，也更容易重试。

### 常见错误五：把 drop -f 放进脚本

`drop -f` 会删除数据库内容。

它适合：

```text
本地重置
临时测试库
CI 中一次性数据库
```

不适合：

```text
共享开发库
测试公共库
生产环境
```

脚本里出现 `drop -f`，要特别警惕。

### 工程实践建议

推荐目录：

```text
db/
  migrations/
    000001_create_users_table.up.sql
    000001_create_users_table.down.sql
internal/
  database/
    migrate.go
  model/
  repository/
```

常见规则：

* 每次结构变更新增一对 `up/down`
* 已经提交并执行过的迁移文件不要修改
* `down.sql` 必须跟着 `up.sql` 一起评审
* 发布前至少在临时库跑 `up`
* 重要发布前测试 `down` 再 `up`
* dirty 状态必须人工确认后再 `force`
* 大表变更拆成多个小版本
* 应用代码和数据库迁移要规划先后顺序
* 生产迁移优先放到发布流水线或独立 Job
* GORM `AutoMigrate` 不负责生产结构变更

### 常用命令速查

```bash
# 创建迁移文件
migrate create -ext sql -dir db/migrations -seq create_users_table

# 升级到最新
migrate -path db/migrations -database "$DATABASE_URL" up

# 升级 1 个版本
migrate -path db/migrations -database "$DATABASE_URL" up 1

# 回滚 1 个版本
migrate -path db/migrations -database "$DATABASE_URL" down 1

# 回滚全部
migrate -path db/migrations -database "$DATABASE_URL" down -all

# 查看版本
migrate -path db/migrations -database "$DATABASE_URL" version

# 跳转到版本 3
migrate -path db/migrations -database "$DATABASE_URL" goto 3

# 修正版本
migrate -path db/migrations -database "$DATABASE_URL" force 3
```

### 总结

`golang-migrate` 的核心不复杂：

```text
一对 up/down 文件
一个递增版本号
一张 schema_migrations 表
一组 up、down、goto、force 命令
```

真正难的不是工具命令，而是迁移纪律：

```text
变更进 Git
上线前验证
失败能定位
回滚有脚本
大表分阶段
dirty 不乱 force
```

GORM 能让 CRUD 更顺手，`golang-migrate` 能让数据库结构变更更可控。两者分工清楚后，项目里的表结构就不会再靠口头同步和手动执行 SQL 维持。
