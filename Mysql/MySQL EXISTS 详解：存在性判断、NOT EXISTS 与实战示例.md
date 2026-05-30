### 简介

`EXISTS` 是 MySQL 里用来做存在性判断的语法。

它关心的问题很直接：

```text
子查询里有没有返回数据。
```

有数据，`EXISTS` 的结果为真；没有数据，`EXISTS` 的结果为假。

对应地，`NOT EXISTS` 判断的是：

```text
子查询里没有返回数据。
```

最常见的业务场景有这些：

* 查询有订单的用户
* 查询没有订单的用户
* 判断某条记录是否存在
* 查询满足关联条件的数据
* 查询缺少关联数据的数据
* 防重复插入
* 按存在性条件更新或删除数据
* 替代部分 `COUNT(*) > 0` 判断

一句话概括：

```text
EXISTS 用来判断“存在”，NOT EXISTS 用来判断“不存在”。
```

### 基本语法

`EXISTS` 的语法：

```sql
SELECT 字段
FROM 表A
WHERE EXISTS (
  SELECT 1
  FROM 表B
  WHERE 表B.关联字段 = 表A.关联字段
);
```

`NOT EXISTS` 的语法：

```sql
SELECT 字段
FROM 表A
WHERE NOT EXISTS (
  SELECT 1
  FROM 表B
  WHERE 表B.关联字段 = 表A.关联字段
);
```

常见写法里，子查询会写 `SELECT 1`。

原因是 `EXISTS` 不关心子查询返回什么字段，只关心有没有行。

下面几种写法在存在性判断上表达的结果一致：

```sql
EXISTS (SELECT 1 FROM orders WHERE orders.user_id = users.id)
EXISTS (SELECT * FROM orders WHERE orders.user_id = users.id)
EXISTS (SELECT order_no FROM orders WHERE orders.user_id = users.id)
```

实际项目里常写 `SELECT 1`，语义更清楚：

```text
只需要判断是否存在，不需要取业务字段。
```

### 准备一组演示数据

后面的示例可以直接基于这几张表运行。

```sql
DROP TABLE IF EXISTS refunds;
DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS user_roles;
DROP TABLE IF EXISTS users;

CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL,
  status TINYINT NOT NULL DEFAULT 1,
  vip TINYINT NOT NULL DEFAULT 0,
  created_at DATETIME NOT NULL,
  UNIQUE KEY uk_email (email)
);

CREATE TABLE orders (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id INT NULL,
  order_no VARCHAR(50) NOT NULL,
  amount DECIMAL(10, 2) NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME NOT NULL,
  INDEX idx_user_id (user_id),
  INDEX idx_status (status),
  INDEX idx_user_status (user_id, status)
);

CREATE TABLE order_items (
  id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT NOT NULL,
  product_name VARCHAR(100) NOT NULL,
  quantity INT NOT NULL,
  price DECIMAL(10, 2) NOT NULL,
  INDEX idx_order_id (order_id),
  INDEX idx_product_name (product_name)
);

CREATE TABLE refunds (
  id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT NOT NULL,
  reason VARCHAR(100),
  created_at DATETIME NOT NULL,
  INDEX idx_order_id (order_id)
);

CREATE TABLE user_roles (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id INT NOT NULL,
  role_code VARCHAR(50) NOT NULL,
  INDEX idx_user_role (user_id, role_code)
);

INSERT INTO users (username, email, status, vip, created_at) VALUES
('张三', 'zhangsan@example.com', 1, 0, '2026-01-01 10:00:00'),
('李四', 'lisi@example.com', 1, 1, '2026-01-02 10:00:00'),
('王五', 'wangwu@example.com', 1, 0, '2026-01-03 10:00:00'),
('赵六', 'zhaoliu@example.com', 0, 0, '2026-01-04 10:00:00'),
('钱七', 'qianqi@example.com', 1, 0, '2026-01-05 10:00:00');

INSERT INTO orders (user_id, order_no, amount, status, created_at) VALUES
(1, 'A001', 99.00, 'paid', '2026-02-01 10:00:00'),
(1, 'A002', 260.00, 'paid', '2026-02-02 10:00:00'),
(2, 'A003', 35.00, 'cancelled', '2026-02-03 10:00:00'),
(3, 'A004', 580.00, 'paid', '2026-02-04 10:00:00'),
(NULL, 'A005', 100.00, 'paid', '2026-02-05 10:00:00');

INSERT INTO order_items (order_id, product_name, quantity, price) VALUES
(1, '键盘', 1, 99.00),
(2, '显示器', 1, 260.00),
(3, '鼠标垫', 1, 35.00),
(4, 'iPhone', 1, 580.00);

INSERT INTO refunds (order_id, reason, created_at) VALUES
(3, '用户取消', '2026-02-04 09:00:00');

INSERT INTO user_roles (user_id, role_code) VALUES
(2, 'VIP'),
(3, 'VIP'),
(4, 'ADMIN');
```

### 示例一：判断某条记录是否存在

`EXISTS` 可以直接返回 `1` 或 `0`。

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

查询一个不存在的邮箱：

```sql
SELECT EXISTS (
  SELECT 1
  FROM users
  WHERE email = 'none@example.com'
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

这种写法适合做“邮箱是否已注册”“用户名是否存在”“订单号是否存在”这类判断。

### 示例二：查询有订单的用户

查询至少有一笔订单的用户：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
);
```

结果类似：

```text
+----+----------+
| id | username |
+----+----------+
|  1 | 张三     |
|  2 | 李四     |
|  3 | 王五     |
+----+----------+
```

执行含义：

```text
对 users 中的每一行，到 orders 中查是否有对应订单。
只要找到一条匹配订单，这个用户就满足条件。
```

`EXISTS` 不会把订单字段返回出来。它只负责判断是否存在关联数据。

### 示例三：查询没有订单的用户

查询没有订单的用户：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE NOT EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
);
```

结果类似：

```text
+----+----------+
| id | username |
+----+----------+
|  4 | 赵六     |
|  5 | 钱七     |
+----+----------+
```

`NOT EXISTS` 很适合表达“没有关联记录”的需求，比如：

* 没有订单的用户
* 没有分配角色的账号
* 没有明细的主表记录
* 没有退款记录的订单
* 没有登录记录的用户

### 示例四：带条件的 EXISTS

查询有已支付订单的用户：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
    AND o.status = 'paid'
);
```

结果类似：

```text
+----+----------+
| id | username |
+----+----------+
|  1 | 张三     |
|  3 | 王五     |
+----+----------+
```

李四有订单，但订单状态是 `cancelled`，所以不在结果中。

再加金额条件：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
    AND o.status = 'paid'
    AND o.amount >= 200
);
```

这种写法适合“只要存在一条满足条件的关联记录即可”的场景。

### 示例五：多表条件存在性判断

查询买过 `iPhone` 的用户。

订单和商品明细是两张表，需要在子查询里继续关联：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE EXISTS (
  SELECT 1
  FROM orders AS o
  JOIN order_items AS oi ON oi.order_id = o.id
  WHERE o.user_id = u.id
    AND oi.product_name = 'iPhone'
);
```

结果类似：

```text
+----+----------+
| id | username |
+----+----------+
|  3 | 王五     |
+----+----------+
```

这里仍然不需要返回订单字段或商品字段，只需要判断是否存在符合条件的明细。

### 示例六：组合 EXISTS 和 NOT EXISTS

查询满足这些条件的用户：

* 有已支付订单
* 没有退款订单
* 拥有 `VIP` 角色

SQL：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
    AND o.status = 'paid'
)
AND NOT EXISTS (
  SELECT 1
  FROM orders AS o
  JOIN refunds AS r ON r.order_id = o.id
  WHERE o.user_id = u.id
)
AND EXISTS (
  SELECT 1
  FROM user_roles AS ur
  WHERE ur.user_id = u.id
    AND ur.role_code = 'VIP'
);
```

这种写法适合复杂业务条件过滤。每个 `EXISTS` 都表达一个独立条件，读起来比较清楚。

### 示例七：EXISTS 和 COUNT 的取舍

只判断“有没有”时，可以直接使用 `EXISTS`。

```sql
SELECT EXISTS (
  SELECT 1
  FROM orders
  WHERE user_id = 1
) AS has_order;
```

如果写成 `COUNT(*)`：

```sql
SELECT COUNT(*) AS order_count
FROM orders
WHERE user_id = 1;
```

它回答的是“有多少条”。

两者语义不同：

| 需求 | 写法 |
| --- | --- |
| 判断有没有 | `EXISTS (SELECT 1 ...)` |
| 统计有多少 | `COUNT(*)` |

存在性判断里，`EXISTS` 语义更贴合。需要具体数量时，`COUNT(*)` 更合适。

### 示例八：EXISTS 和 IN 的区别

查询有订单的用户，也可以写成 `IN`：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE u.id IN (
  SELECT o.user_id
  FROM orders AS o
);
```

`EXISTS` 写法：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
);
```

两者都能表达“有订单的用户”，但关注点不同：

| 写法 | 关注点 |
| --- | --- |
| `IN` | 外层值是否在子查询返回的值列表中 |
| `EXISTS` | 子查询是否能找到匹配行 |

在现代 MySQL 中，优化器可能把二者转换成相近的执行计划。实际性能要结合数据量、索引、执行计划判断。

一般理解：

* 子查询结果集较小，`IN` 写法也很自然
* 只表达存在性，`EXISTS` 语义更直接
* 子查询关联字段有索引时，`EXISTS` 通常表现稳定

### 示例九：NOT EXISTS 和 NOT IN 的 NULL 差异

`NOT EXISTS` 和 `NOT IN` 都可以表达“不存在”，但 `NULL` 会影响 `NOT IN` 的结果。

当前演示数据里，`orders.user_id` 有一条 `NULL`。

看这条 SQL：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE u.id NOT IN (
  SELECT o.user_id
  FROM orders AS o
);
```

由于子查询结果里包含 `NULL`，`NOT IN` 的判断会变成不确定，可能查不出预期结果。

使用 `NOT EXISTS`：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE NOT EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
);
```

结果仍然是没有订单的用户：

```text
+----+----------+
| id | username |
+----+----------+
|  4 | 赵六     |
|  5 | 钱七     |
+----+----------+
```

如果使用 `NOT IN`，可以在子查询里过滤掉 `NULL`：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE u.id NOT IN (
  SELECT o.user_id
  FROM orders AS o
  WHERE o.user_id IS NOT NULL
);
```

涉及反向存在性判断时，`NOT EXISTS` 通常更容易表达清楚，也能自然避开 `NULL` 对 `NOT IN` 的影响。

### 示例十：EXISTS 和 JOIN 的区别

`JOIN` 适合获取关联表字段。

例如查询用户和订单：

```sql
SELECT
  u.id,
  u.username,
  o.order_no,
  o.amount
FROM users AS u
JOIN orders AS o ON o.user_id = u.id;
```

如果一个用户有多笔订单，结果里会出现多行用户数据。

`EXISTS` 适合判断关联数据是否存在：

```sql
SELECT
  u.id,
  u.username
FROM users AS u
WHERE EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
);
```

这里每个用户最多返回一行，不会因为多笔订单而重复。

如果使用 `JOIN` 只为了判断存在性，通常还要加 `DISTINCT`：

```sql
SELECT DISTINCT
  u.id,
  u.username
FROM users AS u
JOIN orders AS o ON o.user_id = u.id;
```

两种写法各有用途：

| 需求 | 更贴近的写法 |
| --- | --- |
| 需要返回订单字段 | `JOIN` |
| 只判断是否有订单 | `EXISTS` |
| 查没有订单的用户 | `NOT EXISTS` 或 `LEFT JOIN ... IS NULL` |

### 示例十一：防重复插入

插入角色前，先判断是否已经存在。

```sql
INSERT INTO user_roles (user_id, role_code)
SELECT 1, 'VIP'
WHERE NOT EXISTS (
  SELECT 1
  FROM user_roles AS ur
  WHERE ur.user_id = 1
    AND ur.role_code = 'VIP'
);
```

含义：

```text
只有 user_id = 1 且 role_code = 'VIP' 的记录不存在时，才插入新记录。
```

如果这是强一致的唯一约束场景，还应在表上建立唯一索引：

```sql
ALTER TABLE user_roles
ADD UNIQUE KEY uk_user_role (user_id, role_code);
```

`NOT EXISTS` 可以表达插入条件，唯一索引用来保证并发情况下的数据一致性。

### 示例十二：UPDATE 中使用 EXISTS

把有已支付订单的用户标记为 VIP：

```sql
UPDATE users AS u
SET u.vip = 1
WHERE EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.user_id = u.id
    AND o.status = 'paid'
);
```

更新后可以查询：

```sql
SELECT id, username, vip
FROM users
ORDER BY id;
```

`EXISTS` 在这里表示“满足关联条件的用户才更新”。

### 示例十三：DELETE 中使用 NOT EXISTS

删除没有对应订单的退款记录。

```sql
DELETE r
FROM refunds AS r
WHERE NOT EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.id = r.order_id
);
```

这个例子表达的是：如果退款记录找不到对应订单，就删除退款记录。

实际执行删除前，可以先用 `SELECT` 查看影响范围：

```sql
SELECT r.*
FROM refunds AS r
WHERE NOT EXISTS (
  SELECT 1
  FROM orders AS o
  WHERE o.id = r.order_id
);
```

### 索引建议

`EXISTS` 常见写法里，子查询会通过关联字段查数据：

```sql
WHERE o.user_id = u.id
```

这里的 `orders.user_id` 很适合建索引：

```sql
CREATE INDEX idx_user_id ON orders(user_id);
```

如果还经常按状态过滤：

```sql
WHERE o.user_id = u.id
  AND o.status = 'paid'
```

可以考虑联合索引：

```sql
CREATE INDEX idx_user_status ON orders(user_id, status);
```

索引设计要结合真实查询条件。`EXISTS` 本身只是表达存在性，性能关键通常在关联条件和过滤条件能否高效定位数据。

### 使用注意

* 子查询里通常写 `SELECT 1`，表示只关心是否存在
* 子查询需要写清楚和外层表的关联条件
* 只判断有没有时，`EXISTS` 比 `COUNT(*)` 更贴近语义
* 需要获取关联表字段时，`JOIN` 更合适
* 涉及“不存在”判断时，`NOT EXISTS` 对 `NULL` 的表现更直观
* `IN`、`EXISTS`、`JOIN` 没有绝对固定的性能顺序，应结合索引和执行计划判断
* 高频查询要关注子查询关联字段的索引
* 防重复插入场景里，`NOT EXISTS` 可以表达条件，唯一索引负责最终约束

### 总结

`EXISTS` 的核心是判断子查询是否有结果，`NOT EXISTS` 的核心是判断子查询是否没有结果。

它们适合表达存在性关系：有订单、无订单、有角色、无退款、有明细、无关联记录。只要业务问题是“有没有”，`EXISTS` 往往能把 SQL 意图写得很清楚。

和 `IN`、`JOIN` 相比，`EXISTS` 的重点不是返回值列表，也不是取关联表字段，而是判断关联条件下是否能找到行。写这类 SQL 时，重点关注两件事：关联条件是否准确，关联字段是否有合适索引。
