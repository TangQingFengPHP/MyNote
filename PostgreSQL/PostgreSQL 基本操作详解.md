# PostgreSQL 基本操作详解（常用命令 + SQL + 实战）

PostgreSQL（简称 PG）是一款功能强大的开源关系型数据库。它和 MySQL 在基本 SQL 操作上非常相似，但在数据库管理、Schema、数据类型、自增主键、权限管理等方面存在一些区别。

下面按照实际开发和 Linux 服务器运维的场景，整理一份 PostgreSQL 常用操作手册。

## 一、连接 PostgreSQL

### 1. 登录数据库

Linux 服务器安装 PostgreSQL 后，通常会创建一个 `postgres` 系统用户和同名数据库角色。

```
# 切换到 postgres 系统用户
sudo -iu postgres

# 进入 PostgreSQL 命令行
psql
```

也可以直接执行：

```
sudo -u postgres psql
```

使用用户名、数据库名连接：

```
psql -U postgres -d postgres
```

连接远程数据库：

```
psql -h 192.168.1.100 -p 5432 -U postgres -d testdb
```

参数说明：

| 参数 | 作用             |
| ---- | ---------------- |
| `-h` | 数据库服务器地址 |
| `-p` | 端口，默认 5432  |
| `-U` | 数据库用户名     |
| `-d` | 数据库名称       |
| `-W` | 强制提示输入密码 |

注意：本地连接可能采用 `peer` 认证，即使密码正确，也可能无法通过 `psql -U postgres` 登录。这时可以使用 `sudo -u postgres psql`，或者指定 `-h 127.0.0.1` 走 TCP 连接。

### 2. psql 常用命令

进入 PostgreSQL 后：

```
-- 查看所有数据库
\l

-- 查看当前数据库
SELECT current_database();

-- 切换数据库
\c testdb

-- 查看当前数据库的所有表
\dt

-- 查看所有 Schema
\dn

-- 查看指定表结构
\d users

-- 查看详细表结构
\d+ users

-- 查看所有数据库角色
\du

-- 查看当前连接信息
\conninfo

-- 查看 SQL 执行时间
\timing on

-- 开启扩展显示（适合宽表）
\x auto

-- 帮助
\?

-- 退出
\q
```

注意： `\l`、`\dt`、`\d` 等是 `psql` 客户端的元命令，不是标准 SQL，不需要添加分号。

## 二、数据库基本操作

### 1. 创建数据库

```
CREATE DATABASE testdb;
```

指定所有者和编码：

```
CREATE DATABASE testdb
    WITH OWNER = postgres
    ENCODING = 'UTF8'
    TEMPLATE = template0;
```

### 2. 查看数据库

```
\l
```

或者：

```
SELECT datname
FROM pg_database
WHERE datistemplate = false;
```

### 3. 切换数据库

```
\c testdb
```

PostgreSQL 与 MySQL 的区别：

```
-- MySQL
USE testdb;

-- PostgreSQL（psql 命令）
\c testdb
```

PostgreSQL 不支持 MySQL 的 `USE database` 语法。通常需要重新建立到目标数据库的连接。

### 4. 删除数据库

```
DROP DATABASE testdb;
```

删除前需要连接到其他数据库，例如 `postgres`，不能在当前连接的数据库中直接删除它。

PostgreSQL 13+ 还支持：

```
DROP DATABASE testdb WITH (FORCE);
```

该命令会尝试终止其他连接，但仍受权限、预备事务等条件限制。

## 三、数据表基本操作

### 1. 创建表

以用户表为例：

```
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    age INT DEFAULT 0,
    salary NUMERIC(12, 2),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

常见数据类型：

| PostgreSQL        | MySQL 对应类型        | 说明               |
| ----------------- | --------------------- | ------------------ |
| `SMALLINT`        | SMALLINT              | 小整数             |
| `INTEGER` / `INT` | INT                   | 整数               |
| `BIGINT`          | BIGINT                | 大整数             |
| `VARCHAR(n)`      | VARCHAR(n)            | 变长字符串         |
| `TEXT`            | TEXT                  | 文本               |
| `BOOLEAN`         | TINYINT(1) 常见用法   | 布尔值             |
| `NUMERIC(12,2)`   | DECIMAL(12,2)         | 精确小数           |
| `TIMESTAMP`       | DATETIME              | 不带时区的日期时间 |
| `TIMESTAMPTZ`     | TIMESTAMP（语义不同） | 带时区语义的时间   |
| `UUID`            | CHAR(36) 等           | UUID               |
| `JSONB`           | JSON                  | 二进制表示的 JSON  |

PostgreSQL 推荐使用标准的 `GENERATED ... AS IDENTITY` 实现自增主键。

也可以使用旧语法：

```
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100)
);
```

### 2. 查看表

```
-- 当前 Schema 搜索路径下的表
\dt

-- 查看所有 Schema 下的表
\dt *.*

-- 查看表结构
\d users
```

使用 SQL 查询：

```
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
  AND table_schema NOT IN ('pg_catalog', 'information_schema');
```

### 3. 修改表结构

```
-- 添加字段
ALTER TABLE users
ADD COLUMN phone VARCHAR(20);

-- 修改字段类型
ALTER TABLE users
ALTER COLUMN username TYPE VARCHAR(100);

-- 添加非空约束
ALTER TABLE users
ALTER COLUMN username SET NOT NULL;

-- 设置默认值
ALTER TABLE users
ALTER COLUMN age SET DEFAULT 18;

-- 重命名字段
ALTER TABLE users
RENAME COLUMN phone TO mobile;

-- 删除字段
ALTER TABLE users
DROP COLUMN mobile;

-- 重命名表
ALTER TABLE users RENAME TO app_users;
```

### 4. 删除表与清空数据

```
-- 删除表
DROP TABLE users;

-- 表存在才删除
DROP TABLE IF EXISTS users;

-- 清空表数据
TRUNCATE TABLE users;

-- 清空数据并重置自增序列
TRUNCATE TABLE users RESTART IDENTITY;
```

如果表存在外键依赖，可以使用 `CASCADE`，但会影响依赖对象或相关表，生产环境应谨慎使用。

## 四、数据增删改查（CRUD）

以下示例假设使用前面创建的 `users` 表。

### 1. INSERT 插入数据

```
INSERT INTO users (username, email, age)
VALUES ('zhangsan', 'zhangsan@example.com', 25);
```

批量插入：

```
INSERT INTO users (username, email, age)
VALUES
('lisi', 'lisi@example.com', 28),
('wangwu', 'wangwu@example.com', 30);
```

PostgreSQL 支持 `RETURNING`，插入后直接返回数据：

```
INSERT INTO users (username, email, age)
VALUES ('zhaoliu', 'zhaoliu@example.com', 26)
RETURNING id, username, created_at;
```

这在 .NET、Java 等后端开发中非常实用，不需要再执行一次查询来获取新生成的 ID。

### 2. SELECT 查询数据

```
-- 查询全部
SELECT * FROM users;

-- 指定字段
SELECT id, username, age FROM users;

-- 条件查询
SELECT * FROM users WHERE age > 25;

-- 多条件
SELECT *
FROM users
WHERE age >= 18 AND is_active = TRUE;

-- 模糊查询（区分大小写）
SELECT * FROM users
WHERE username LIKE '%zhang%';

-- 模糊查询（不区分大小写）
SELECT * FROM users
WHERE username ILIKE '%ZHANG%';

-- 排序
SELECT * FROM users
ORDER BY created_at DESC;

-- 分页
SELECT * FROM users
ORDER BY id
LIMIT 10 OFFSET 20;

-- 聚合
SELECT COUNT(*) FROM users;

SELECT AVG(age) FROM users;

-- 分组
SELECT is_active, COUNT(*)
FROM users
GROUP BY is_active;
```

### 3. UPDATE 修改数据

```
UPDATE users
SET age = 30
WHERE id = 1;
```

修改并返回更新后的数据：

```
UPDATE users
SET age = age + 1,
    salary = 15000
WHERE id = 1
RETURNING *;
```

### 4. DELETE 删除数据

```
DELETE FROM users
WHERE id = 1;
```

删除并返回：

```
DELETE FROM users
WHERE id = 1
RETURNING *;
```

注意：`UPDATE` 和 `DELETE` 如果不写 `WHERE`，会影响整张表。

## 五、PostgreSQL 特有的常用功能

### 1. UPSERT：存在则更新，不存在则插入

MySQL：

```
INSERT INTO users (username, email)
VALUES ('zhangsan', 'zhangsan@example.com')
ON DUPLICATE KEY UPDATE
    email = VALUES(email);
```

PostgreSQL：

```
INSERT INTO users (username, email)
VALUES ('zhangsan', 'zhangsan@example.com')
ON CONFLICT (email)
DO UPDATE SET
    username = EXCLUDED.username;
```

其中 `EXCLUDED` 表示本次尝试插入的新数据。这里要求 `email` 有唯一约束或匹配的唯一索引。

如果冲突时不需要处理：

```
INSERT INTO users (username, email)
VALUES ('zhangsan', 'zhangsan@example.com')
ON CONFLICT (email) DO NOTHING;
```

### 2. JSONB 数据操作

PostgreSQL 的 JSONB 是它比较有特色的能力之一。

创建表：

```
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_no VARCHAR(50),
    metadata JSONB
);
```

插入：

```
INSERT INTO orders (order_no, metadata)
VALUES (
    'ORD001',
    '{"source":"web","customer":{"name":"Tom"},"tags":["vip","new"]}'
);
```

查询 JSON 字段：

```
SELECT
    metadata->>'source' AS source,
    metadata->'customer'->>'name' AS customer_name
FROM orders;
```

按 JSON 内容过滤：

```
SELECT *
FROM orders
WHERE metadata @> '{"source":"web"}'::jsonb;
```

创建 JSONB GIN 索引：

```
CREATE INDEX idx_orders_metadata
ON orders USING GIN (metadata);
```

### 3. Schema 模式

PostgreSQL 的层级关系：

![alt text](/images/pgsql/image.png)

创建 Schema：

```
CREATE SCHEMA sales;
```

在指定 Schema 创建表：

```
CREATE TABLE sales.products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100)
);
```

查询：

```
SELECT * FROM sales.products;
```

查看当前搜索路径：

```
SHOW search_path;
```

设置当前会话的搜索路径：

```
SET search_path TO sales, public;
```

PostgreSQL 的 Schema 不是独立数据库，而是数据库内部的命名空间。

## 六、索引管理

创建普通索引：

```
CREATE INDEX idx_users_username
ON users(username);
```

创建联合索引：

```
CREATE INDEX idx_users_age_created
ON users(age, created_at);
```

创建唯一索引：

```
CREATE UNIQUE INDEX idx_users_email_unique
ON users(email);
```

由于前面的 `email` 已经定义了 `UNIQUE`，实际上不需要重复创建该唯一索引，这里只是演示语法。

查看索引：

```
\di
```

查询指定表的索引：

```
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'users';
```

删除索引：

```
DROP INDEX idx_users_username;
```

生产环境大表创建索引可以考虑：

```
CREATE INDEX CONCURRENTLY idx_users_age
ON users(age);
```

`CONCURRENTLY` 能减少对并发写入的阻塞，但不能放在显式事务块中执行。

## 七、事务操作

```
BEGIN;

UPDATE users
SET salary = salary - 1000
WHERE id = 1;

UPDATE users
SET salary = salary + 1000
WHERE id = 2;

COMMIT;
```

发生异常时：

```
ROLLBACK;
```

使用保存点：

```
BEGIN;

UPDATE users SET age = 20 WHERE id = 1;

SAVEPOINT sp1;

UPDATE users SET age = 30 WHERE id = 2;

ROLLBACK TO SAVEPOINT sp1;

COMMIT;
```

查看事务隔离级别：

```
SHOW transaction_isolation;
```

PostgreSQL 默认隔离级别是 `READ COMMITTED`，并使用 MVCC（多版本并发控制）管理并发访问。

## 八、用户与权限管理

### 1. 创建用户

```
CREATE USER app_user WITH PASSWORD 'ReplaceWithStrongPassword';
```

PostgreSQL 中 `USER` 本质上是默认具有 `LOGIN` 权限的 `ROLE`。

### 2. 创建数据库并指定所有者

```
CREATE DATABASE appdb OWNER app_user;
```

### 3. 授权

```
-- 允许连接数据库
GRANT CONNECT ON DATABASE appdb TO app_user;
```

切换到 `appdb` 后执行：

```
-- 允许使用 public Schema
GRANT USAGE ON SCHEMA public TO app_user;

-- 已存在表的增删改查权限
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA public
TO app_user;

-- 已存在序列的使用权限
GRANT USAGE, SELECT
ON ALL SEQUENCES IN SCHEMA public
TO app_user;
```

如果后续还要自动授权新建的表，可以配置默认权限：

```
ALTER DEFAULT PRIVILEGES FOR ROLE postgres
IN SCHEMA public
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;
```

这里仅影响 `postgres` 角色今后创建的表；不同建表角色需要分别设置。

### 4. 修改密码

```
ALTER USER app_user
WITH PASSWORD 'NewStrongPassword';
```

### 5. 删除用户

```
DROP USER app_user;
```

如果该用户仍拥有数据库对象或权限依赖，需要先处理所有权及权限，才能删除。

## 九、数据库备份与恢复

这一部分是服务器运维中经常需要用到的。

### 1. 备份单个数据库

```
pg_dump -h localhost -U postgres -d testdb \
  -F c -f /backup/testdb.dump
```

`-F c` 表示 PostgreSQL 自定义备份格式，适合配合 `pg_restore` 恢复。

### 2. 恢复数据库

先创建目标数据库：

```
createdb -h localhost -U postgres restoredb
```

恢复：

```
pg_restore -h localhost -U postgres \
  -d restoredb /backup/testdb.dump
```

### 3. 导出 SQL 文件

```
pg_dump -U postgres -d testdb \
  -f /backup/testdb.sql
```

恢复 SQL 文件：

```
psql -U postgres -d restoredb \
  -f /backup/testdb.sql
```

### 4. 备份整个实例中的数据库及全局对象

```
pg_dumpall -U postgres > all_databases.sql
```

恢复：

```
psql -U postgres -d postgres -f all_databases.sql
```

注意：`pg_dump` 和 `pg_dumpall` 是逻辑备份工具，不是完整的物理备份或 PITR（时间点恢复）方案。

## 十、服务器运维常用命令

### 查看 PostgreSQL 服务

```
# systemd 管理的 Linux 系统
systemctl status postgresql

# 启动
sudo systemctl start postgresql

# 停止
sudo systemctl stop postgresql

# 重启
sudo systemctl restart postgresql
```

Rocky Linux、AlmaLinux 或 PGDG 仓库安装的 PostgreSQL，服务名称可能带版本号，例如 `postgresql-16`，需要以实际安装的服务名为准。

### 查看 PostgreSQL 版本

```
SELECT version();
```

或者：

```
psql --version
```

### 查看配置文件路径

```
SHOW config_file;
SHOW hba_file;
SHOW data_directory;
```

### 查看当前连接数

```
SELECT COUNT(*)
FROM pg_stat_activity;
```

查看连接详情：

```
SELECT
    pid,
    usename,
    datname,
    client_addr,
    state,
    query
FROM pg_stat_activity;
```

### 查看最大连接数

```
SHOW max_connections;
```

### 查看数据库大小

```
SELECT
    pg_database.datname,
    pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
WHERE datallowconn;
```

### 查看表大小

```
SELECT
    pg_size_pretty(pg_total_relation_size('users')) AS total_size;
```

### 查看慢 SQL 执行计划

```
EXPLAIN ANALYZE
SELECT *
FROM users
WHERE age > 25;
```

`EXPLAIN ANALYZE` 会真正执行 SQL，因此对于 `UPDATE`、`DELETE` 等语句，需特别谨慎。

## 十一、PostgreSQL 与 MySQL 常用操作对比

| 操作             | MySQL                      | PostgreSQL              |
| ---------------- | -------------------------- | ----------------------- |
| 查看数据库       | `SHOW DATABASES;`          | `\l`                    |
| 切换数据库       | `USE db;`                  | `\c db`                 |
| 查看表           | `SHOW TABLES;`             | `\dt`                   |
| 查看表结构       | `DESC users;`              | `\d users`              |
| 查看建表语句     | `SHOW CREATE TABLE users;` | `pg_dump -s -t users`   |
| 自增字段         | `AUTO_INCREMENT`           | `IDENTITY` / `SERIAL`   |
| 获取插入 ID      | `LAST_INSERT_ID()`         | `RETURNING id`          |
| 不区分大小写匹配 | 依赖排序规则等             | `ILIKE`                 |
| 分页             | `LIMIT 10 OFFSET 20`       | `LIMIT 10 OFFSET 20`    |
| 冲突更新         | `ON DUPLICATE KEY UPDATE`  | `ON CONFLICT DO UPDATE` |
| JSON             | `JSON`                     | `JSON` / `JSONB`        |
| 导出数据库       | `mysqldump`                | `pg_dump`               |
| 导入数据库       | `mysql`                    | `psql` / `pg_restore`   |

## 十二、常用命令速查

PostgreSQL 命令速查表

&#x20;复制全部

连接数据库

`psql -h localhost -U postgres -d testdb`

列出数据库

`\l`

切换数据库

`\c testdb`

查看表

`\dt`

查看表结构

`\d users`

查看角色

`\du`

查看当前连接

`\conninfo`

查看数据库大小

`SELECT pg_size_pretty(pg_database_size(current_database()));`

查看连接数

`SELECT COUNT(*) FROM pg_stat_activity;`

查看版本

`SELECT version();`

退出

`\q`