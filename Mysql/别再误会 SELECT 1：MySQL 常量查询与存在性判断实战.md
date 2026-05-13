### 简介

`SELECT 1` 是 MySQL 里很常见的一句 SQL。

它不是查询第一行，也不是查询第一列，更不是某种特殊语法。它的本质很简单：

```text
返回一个固定的常量值 1
```

最基础写法如下：

```sql
SELECT 1;
```

执行结果：

```text
+---+
| 1 |
+---+
| 1 |
+---+
```

这条 SQL 没有 `FROM`，所以不查任何业务表。MySQL 只需要解析 SQL，然后返回一个常量结果。

类似写法还有：

```sql
SELECT 100;
SELECT 'ok';
SELECT NOW();
SELECT 1 + 1;
```

这些都属于“常量查询”或“表达式查询”。

一句话概括：

```text
SELECT 1 适合用来表达“只关心 SQL 能不能执行”或者“只关心数据是否存在”。
```

### SELECT 1 到底查不查表？

分两种情况看。

#### 不带 FROM

```sql
SELECT 1;
```

这种不查表，只返回一行一列。

#### 带 FROM

```sql
SELECT 1 FROM users;
```

这种会查 `users` 表。

区别在于：它不会返回 `users` 表里的字段，而是表里每匹配一行，就返回一个常量 `1`。

假设 `users` 表有 3 行数据，结果类似：

```text
+---+
| 1 |
+---+
| 1 |
| 1 |
| 1 |
+---+
```

所以：

```sql
SELECT 1;
```

和：

```sql
SELECT 1 FROM users;
```

不是一回事。前者不查表，后者查表。

### 准备一组演示数据

后面的例子可以直接用这组表来跑。

```sql
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS users;

CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL UNIQUE,
  status TINYINT NOT NULL DEFAULT 1
);

CREATE TABLE orders (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id INT NOT NULL,
  order_no VARCHAR(50) NOT NULL,
  amount DECIMAL(10, 2) NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME NOT NULL,
  INDEX idx_user_id (user_id),
  INDEX idx_status (status)
);

INSERT INTO users (username, email, status) VALUES
('张三', 'zhangsan@example.com', 1),
('李四', 'lisi@example.com', 1),
('王五', 'wangwu@example.com', 0),
('赵六', 'zhaoliu@example.com', 1);

INSERT INTO orders (user_id, order_no, amount, status, created_at) VALUES
(1, 'A001', 99.00, 'paid', '2026-01-01 10:00:00'),
(1, 'A002', 188.50, 'paid', '2026-01-03 11:00:00'),
(2, 'A003', 35.00, 'cancelled', '2026-01-04 12:00:00'),
(4, 'A004', 260.00, 'paid', '2026-01-05 13:00:00');
```

### 用法一：检测数据库连接是否正常

`SELECT 1` 最常见的场景之一，就是连接检测。

```sql
SELECT 1;
```

能正常返回结果，说明：

* MySQL 服务能连上
* 账号密码没有问题
* 当前连接可以正常执行 SQL

命令行里可以这样写：

```shell
mysql -uroot -p -e "SELECT 1;"
```

返回结果类似：

```text
+---+
| 1 |
+---+
| 1 |
+---+
```

很多连接池、中间件、健康检查脚本都会用它做探活，因为它不依赖任何业务表。业务表被删了、表结构改了、数据为空了，都不影响这条 SQL 执行。

Spring Boot 中常见配置：

```yaml
spring:
  datasource:
    hikari:
      connection-test-query: SELECT 1
```

Docker 健康检查也可以这么写：

```yaml
healthcheck:
  test: ["CMD", "mysql", "-uroot", "-p123456", "-e", "SELECT 1;"]
  interval: 10s
  timeout: 3s
  retries: 3
```

### 用法二：判断表里是否有数据

只想知道表里有没有数据，不需要知道有多少条，也不需要拿字段内容，可以这样写：

```sql
SELECT 1 FROM users LIMIT 1;
```

如果表里有数据，返回：

```text
+---+
| 1 |
+---+
| 1 |
+---+
```

如果表里没有数据，返回空结果。

这类场景不要上来就写：

```sql
SELECT COUNT(*) FROM users;
```

`COUNT(*)` 的语义是“统计数量”，`SELECT 1 ... LIMIT 1` 的语义是“找到一条就够了”。只是判断有没有数据时，后者更贴近需求。

### 用法三：判断某条记录是否存在

例如判断某个邮箱是否已经注册。

```sql
SELECT 1
FROM users
WHERE email = 'lisi@example.com'
LIMIT 1;
```

查到一行，表示存在；查不到，表示不存在。

也可以直接返回 `0` 或 `1`：

```sql
SELECT EXISTS (
  SELECT 1
  FROM users
  WHERE email = 'lisi@example.com'
) AS user_exists;
```

结果：

```text
+-------------+
| user_exists |
+-------------+
|           1 |
+-------------+
```

换一个不存在的邮箱：

```sql
SELECT EXISTS (
  SELECT 1
  FROM users
  WHERE email = 'no@example.com'
) AS user_exists;
```

结果：

```text
+-------------+
| user_exists |
+-------------+
|           0 |
+-------------+
```

`EXISTS` 很适合这种“有没有”的判断。

### 用法四：配合 EXISTS 查询有订单的用户

查询所有下过单的用户：

```sql
SELECT u.id, u.username
FROM users AS u
WHERE EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
);
```

结果：

```text
+----+----------+
| id | username |
+----+----------+
|  1 | 张三     |
|  2 | 李四     |
|  4 | 赵六     |
+----+----------+
```

这里的重点是 `EXISTS`。

`EXISTS` 只关心子查询有没有返回行，不关心子查询返回了什么字段。所以子查询里写：

```sql
SELECT 1
```

是在表达：

```text
只需要一个存在标记，不需要订单表的真实字段。
```

写成下面这样通常也能得到一样的结果：

```sql
SELECT u.id, u.username
FROM users AS u
WHERE EXISTS (
  SELECT *
  FROM orders AS o
  WHERE o.user_id = u.id
);
```

在 `EXISTS` 场景里，MySQL 优化器通常不会真的把 `*` 里的所有列取出来。`SELECT 1` 的优势更多是语义清楚：一眼能看出只是在判断是否存在。

### 用法五：配合 NOT EXISTS 查询没有订单的用户

查询没有下过单的用户：

```sql
SELECT u.id, u.username
FROM users AS u
WHERE NOT EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
);
```

结果：

```text
+----+----------+
| id | username |
+----+----------+
|  3 | 王五     |
+----+----------+
```

这种写法比 `LEFT JOIN ... IS NULL` 更直观，尤其是条件比较复杂时，可读性更好。

### 用法六：判断是否有已支付订单

业务里经常不是判断“有没有订单”，而是判断“有没有符合条件的订单”。

例如查询有已支付订单的用户：

```sql
SELECT u.id, u.username
FROM users AS u
WHERE EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
    AND o.status = 'paid'
);
```

结果：

```text
+----+----------+
| id | username |
+----+----------+
|  1 | 张三     |
|  4 | 赵六     |
+----+----------+
```

李四有订单，但订单状态是 `cancelled`，所以没有出现在结果里。

### SELECT 1 和 COUNT(*) 怎么选？

两者不是谁替代谁，而是用途不同。

#### 只判断有没有

推荐：

```sql
SELECT EXISTS (
  SELECT 1
  FROM orders
  WHERE user_id = 1
) AS has_order;
```

或者：

```sql
SELECT 1
FROM orders
WHERE user_id = 1
LIMIT 1;
```

#### 需要准确数量

使用：

```sql
SELECT COUNT(*) AS order_count
FROM orders
WHERE user_id = 1;
```

简单说：

| 需求 | 推荐写法 |
| --- | --- |
| 判断有没有 | `EXISTS (SELECT 1 ...)` |
| 找到一条就够 | `SELECT 1 ... LIMIT 1` |
| 统计总数 | `COUNT(*)` |
| 获取业务字段 | `SELECT id, username ...` |

`COUNT(*)` 要回答“有多少”，`EXISTS` 要回答“有没有”。问题不一样，SQL 就不一样。

### SELECT 1 FROM 表 和 SELECT 字段 FROM 表 的区别

```sql
SELECT 1 FROM users WHERE status = 1;
```

返回的是常量：

```text
+---+
| 1 |
+---+
| 1 |
| 1 |
| 1 |
+---+
```

下面才是返回业务字段：

```sql
SELECT id, username FROM users WHERE status = 1;
```

结果：

```text
+----+----------+
| id | username |
+----+----------+
|  1 | 张三     |
|  2 | 李四     |
|  4 | 赵六     |
+----+----------+
```

所以 `SELECT 1 FROM users` 不适合用来取用户数据，它只适合做存在性判断、测试查询、生成固定标记值。

### DUAL 是什么？

有时会看到这种写法：

```sql
SELECT 1 FROM DUAL;
```

`DUAL` 是一个虚拟表名。Oracle 里常用，MySQL 也兼容这种写法。

在 MySQL 中，下面两句效果一样：

```sql
SELECT 1;
SELECT 1 FROM DUAL;
```

实际写 MySQL 时，通常直接用 `SELECT 1;` 就够了。

### 用 SELECT 1 生成简单测试数据

`SELECT 1` 还可以配合 `UNION ALL` 临时造几行数据。

```sql
SELECT 1 AS num
UNION ALL
SELECT 2
UNION ALL
SELECT 3;
```

结果：

```text
+-----+
| num |
+-----+
|   1 |
|   2 |
|   3 |
+-----+
```

例如临时生成订单状态字典：

```sql
SELECT 'paid' AS status, '已支付' AS label
UNION ALL
SELECT 'cancelled', '已取消'
UNION ALL
SELECT 'pending', '待支付';
```

这种写法适合临时调试、报表拼接、小范围演示。正式业务里如果状态值经常复用，还是建字典表更合适。

### 一个常见坑：不要把 SELECT 1 塞进 IN 里乱用

下面这条 SQL 看起来像“查询有订单的用户”，但实际是错的：

```sql
SELECT id, username
FROM users
WHERE id IN (
  SELECT 1
  FROM orders
);
```

子查询返回的是一堆常量 `1`：

```text
1
1
1
1
```

所以外层条件会变成：

```sql
WHERE id IN (1)
```

最后只会查出 `id = 1` 的用户。

正确写法应该是返回 `orders.user_id`：

```sql
SELECT id, username
FROM users
WHERE id IN (
  SELECT user_id
  FROM orders
);
```

或者直接用 `EXISTS`：

```sql
SELECT u.id, u.username
FROM users AS u
WHERE EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
);
```

记住一点：

```text
IN 需要的是一组可比较的值，EXISTS 需要的是有没有行。
```

所以 `SELECT 1` 适合放在 `EXISTS` 里，不适合替代 `IN` 子查询里的真实字段。

### MyBatis 中的存在性判断

MyBatis 里可以直接返回 `boolean`：

```xml
<select id="existsByEmail" resultType="boolean">
  SELECT EXISTS (
    SELECT 1
    FROM users
    WHERE email = #{email}
  )
</select>
```

如果数据库驱动或项目习惯不直接映射 `boolean`，也可以返回 `int`：

```xml
<select id="existsByEmail" resultType="int">
  SELECT EXISTS (
    SELECT 1
    FROM users
    WHERE email = #{email}
  ) AS exists_flag
</select>
```

返回 `1` 表示存在，返回 `0` 表示不存在。

### 实战建议

* 检测 MySQL 连接是否可用，用 `SELECT 1;`
* 判断数据是否存在，优先考虑 `EXISTS (SELECT 1 ...)`
* 只需要找一条记录，用 `SELECT 1 ... LIMIT 1`
* 需要统计数量，用 `COUNT(*)`
* 需要业务数据，明确写字段名，不要写 `SELECT 1`
* `EXISTS` 里写 `SELECT 1` 主要是语义清晰，不是神秘性能开关
* `IN` 子查询里不要随手写 `SELECT 1`，除非外层字段就是要和常量 `1` 比较

### 总结

`SELECT 1` 很简单，但容易被误解。

不带 `FROM` 时，它只是返回常量，不查表；带 `FROM` 时，它会按匹配行返回常量，但不返回表字段。

它最适合两个场景：一个是数据库连接检测，一个是存在性判断。尤其是和 `EXISTS` 搭配时，`SELECT 1` 能把 SQL 意图表达得很清楚：不关心数据内容，只关心有没有。

别把它当成“查询第一行”，也别把它当成万能优化技巧。把问题先分清楚：要判断有没有、要统计数量，还是要拿业务字段。问题清楚了，SQL 写法自然就清楚了。
