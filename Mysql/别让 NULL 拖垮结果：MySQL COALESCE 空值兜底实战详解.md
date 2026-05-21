# 别让 NULL 拖垮结果：MySQL COALESCE 空值兜底实战详解

### 简介

`COALESCE` 是 MySQL 里非常常用的空值处理函数。

它的作用很简单：

```text
从左到右返回第一个不是 NULL 的值。
```

最基础示例：

```sql
SELECT COALESCE(NULL, NULL, 'hello', 'world');
```

结果：

```text
+-----------------------------------------+
| COALESCE(NULL, NULL, 'hello', 'world') |
+-----------------------------------------+
| hello                                   |
+-----------------------------------------+
```

因为 `'hello'` 是第一个非 `NULL` 值，后面的 `'world'` 不会作为结果返回。

如果所有参数都是 `NULL`：

```sql
SELECT COALESCE(NULL, NULL, NULL);
```

结果还是：

```text
NULL
```

一句话概括：

```text
COALESCE 负责给 NULL 找一个兜底值。
```

它常用于这些场景：

* 字段为空时显示默认文字
* 多个联系方式按优先级取一个
* 聚合统计没有结果时返回 0
* `LEFT JOIN` 没匹配到数据时补默认值
* 拼接字符串时避免结果变成 `NULL`
* 排序时给空值一个排序规则
* 批量更新时保留原值或设置默认值

### 基本语法

```sql
COALESCE(value1, value2, value3, ...)
```

执行规则：

```text
从左到右依次判断，返回第一个非 NULL 的值。
```

示例：

```sql
SELECT COALESCE(NULL, 100, 200);
```

结果：

```text
100
```

再看几个例子：

```sql
SELECT COALESCE('张三', '匿名用户');
```

结果：

```text
张三
```

```sql
SELECT COALESCE(NULL, '匿名用户');
```

结果：

```text
匿名用户
```

```sql
SELECT COALESCE(NULL, NULL);
```

结果：

```text
NULL
```

### 为什么需要 COALESCE？

SQL 里的 `NULL` 不是空字符串，也不是 0，而是“不知道、没有值”。

很多运算碰到 `NULL` 后，结果也会变成 `NULL`。

例如：

```sql
SELECT 100 + NULL;
```

结果：

```text
NULL
```

字符串拼接也一样：

```sql
SELECT CONCAT('手机号：', NULL);
```

结果：

```text
NULL
```

这就是 `NULL` 最容易出问题的地方：只要参与计算或拼接，整个结果可能都没了。

用 `COALESCE` 兜底后：

```sql
SELECT 100 + COALESCE(NULL, 0);
```

结果：

```text
100
```

```sql
SELECT CONCAT('手机号：', COALESCE(NULL, '暂无'));
```

结果：

```text
手机号：暂无
```

### 准备一组演示数据

后面的例子直接使用这几张表。

```sql
DROP TABLE IF EXISTS payments;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS users;
DROP TABLE IF EXISTS departments;

CREATE TABLE departments (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL
);

CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50),
  nickname VARCHAR(50),
  real_name VARCHAR(50),
  mobile VARCHAR(20),
  backup_mobile VARCHAR(20),
  email VARCHAR(100),
  avatar VARCHAR(255),
  department_id INT,
  last_login_at DATETIME,
  created_at DATETIME NOT NULL,
  INDEX idx_mobile (mobile),
  INDEX idx_department_id (department_id)
);

CREATE TABLE orders (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id INT NOT NULL,
  order_no VARCHAR(50) NOT NULL,
  amount DECIMAL(10, 2),
  discount DECIMAL(10, 2),
  sort_no INT,
  updated_at DATETIME,
  created_at DATETIME NOT NULL,
  INDEX idx_user_id (user_id),
  INDEX idx_sort_no (sort_no)
);

CREATE TABLE payments (
  id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT NOT NULL,
  pay_amount DECIMAL(10, 2),
  refund_amount DECIMAL(10, 2),
  INDEX idx_order_id (order_id)
);

INSERT INTO departments (name) VALUES
('研发部'),
('销售部');

INSERT INTO users
(username, nickname, real_name, mobile, backup_mobile, email, avatar, department_id, last_login_at, created_at)
VALUES
('zhangsan', '老张', '张三', '13800000001', NULL, 'zhangsan@example.com', NULL, 1, '2026-01-05 10:00:00', '2026-01-01 10:00:00'),
('lisi', NULL, '李四', NULL, '13900000002', 'lisi@example.com', '/avatar/lisi.png', 2, NULL, '2026-01-02 10:00:00'),
('wangwu', '', '王五', NULL, NULL, 'wangwu@example.com', NULL, NULL, NULL, '2026-01-03 10:00:00'),
(NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, '2026-01-04 10:00:00');

INSERT INTO orders
(user_id, order_no, amount, discount, sort_no, updated_at, created_at)
VALUES
(1, 'A001', 100.00, 10.00, 2, '2026-01-06 10:00:00', '2026-01-05 10:00:00'),
(1, 'A002', 200.00, NULL, NULL, NULL, '2026-01-06 11:00:00'),
(2, 'A003', NULL, NULL, 1, '2026-01-07 10:00:00', '2026-01-07 09:00:00'),
(3, 'A004', 80.00, 5.00, NULL, NULL, '2026-01-08 10:00:00');

INSERT INTO payments (order_id, pay_amount, refund_amount) VALUES
(1, 90.00, NULL),
(2, 200.00, 20.00),
(3, NULL, NULL);
```

### 示例一：字段为空时显示默认值

用户头像为空时，显示默认头像。

```sql
SELECT
  id,
  username,
  COALESCE(avatar, '/avatar/default.png') AS show_avatar
FROM users;
```

结果类似：

```text
+----+----------+---------------------+
| id | username | show_avatar         |
+----+----------+---------------------+
|  1 | zhangsan | /avatar/default.png |
|  2 | lisi     | /avatar/lisi.png    |
|  3 | wangwu   | /avatar/default.png |
|  4 | NULL     | /avatar/default.png |
+----+----------+---------------------+
```

这类场景非常适合 `COALESCE(字段, 默认值)`。

### 示例二：多个字段按优先级取值

用户展示名按这个顺序取：

```text
昵称 -> 真实姓名 -> 用户名 -> 匿名用户
```

SQL：

```sql
SELECT
  id,
  COALESCE(nickname, real_name, username, '匿名用户') AS display_name
FROM users;
```

结果类似：

```text
+----+--------------+
| id | display_name |
+----+--------------+
|  1 | 老张         |
|  2 | 李四         |
|  3 |              |
|  4 | 匿名用户     |
+----+--------------+
```

第三行结果是空字符串，不是 `匿名用户`。

原因是：

```text
COALESCE 只判断 NULL，不会把空字符串 '' 当成 NULL。
```

如果空字符串也要当成空值处理，需要配合 `NULLIF`。

```sql
SELECT
  id,
  COALESCE(
    NULLIF(nickname, ''),
    NULLIF(real_name, ''),
    NULLIF(username, ''),
    '匿名用户'
  ) AS display_name
FROM users;
```

这时第三行会显示 `王五`。

### 示例三：联系方式兜底

联系方式经常不止一个字段，比如手机号、备用手机号、邮箱。

```sql
SELECT
  id,
  COALESCE(mobile, backup_mobile, email, '无联系方式') AS contact
FROM users;
```

结果类似：

```text
+----+---------------------+
| id | contact             |
+----+---------------------+
|  1 | 13800000001         |
|  2 | 13900000002         |
|  3 | wangwu@example.com  |
|  4 | 无联系方式          |
+----+---------------------+
```

这种写法比多层 `CASE WHEN` 更短，也更容易看出优先级。

### 示例四：金额计算时把 NULL 当成 0

订单实付金额 = 订单金额 - 优惠金额。

直接写：

```sql
SELECT
  order_no,
  amount - discount AS actual_amount
FROM orders;
```

如果 `discount` 是 `NULL`，结果也会是 `NULL`。

推荐写法：

```sql
SELECT
  order_no,
  COALESCE(amount, 0) AS amount,
  COALESCE(discount, 0) AS discount,
  COALESCE(amount, 0) - COALESCE(discount, 0) AS actual_amount
FROM orders;
```

结果类似：

```text
+----------+--------+----------+---------------+
| order_no | amount | discount | actual_amount |
+----------+--------+----------+---------------+
| A001     | 100.00 |    10.00 |         90.00 |
| A002     | 200.00 |     0.00 |        200.00 |
| A003     |   0.00 |     0.00 |          0.00 |
| A004     |  80.00 |     5.00 |         75.00 |
+----------+--------+----------+---------------+
```

金额字段参与计算时，`COALESCE(金额字段, 0)` 很常见。

### 示例五：聚合函数没有结果时返回 0

`SUM`、`AVG` 这类聚合函数，在没有匹配数据时可能返回 `NULL`。

例如查询一个不存在用户的订单总额：

```sql
SELECT SUM(amount) AS total_amount
FROM orders
WHERE user_id = 999;
```

结果：

```text
NULL
```

使用 `COALESCE` 兜底：

```sql
SELECT COALESCE(SUM(amount), 0) AS total_amount
FROM orders
WHERE user_id = 999;
```

结果：

```text
0.00
```

统计支付金额、退款金额也很常见：

```sql
SELECT
  COALESCE(SUM(pay_amount), 0) AS total_pay_amount,
  COALESCE(SUM(refund_amount), 0) AS total_refund_amount
FROM payments;
```

注意：`COUNT(*)` 本身没有匹配行时会返回 `0`，通常不需要写成 `COALESCE(COUNT(*), 0)`。

```sql
SELECT COUNT(*) AS order_count
FROM orders
WHERE user_id = 999;
```

结果就是：

```text
0
```

### 示例六：LEFT JOIN 后补默认值

`LEFT JOIN` 没匹配到右表数据时，右表字段会是 `NULL`。

查询用户和部门：

```sql
SELECT
  u.id,
  COALESCE(u.username, '未命名用户') AS username,
  COALESCE(d.name, '未分配部门') AS department_name
FROM users AS u
LEFT JOIN departments AS d ON u.department_id = d.id;
```

结果类似：

```text
+----+------------+-----------------+
| id | username   | department_name |
+----+------------+-----------------+
|  1 | zhangsan   | 研发部          |
|  2 | lisi       | 销售部          |
|  3 | wangwu     | 未分配部门      |
|  4 | 未命名用户 | 未分配部门      |
+----+------------+-----------------+
```

这类写法适合列表页、导出报表、统计看板。

### 示例七：拼接字符串时避免结果变 NULL

MySQL 的 `CONCAT` 只要有一个参数是 `NULL`，结果就会变成 `NULL`。

```sql
SELECT CONCAT(username, ' / ', mobile) AS user_contact
FROM users;
```

如果 `mobile` 是 `NULL`，整段拼接结果也会是 `NULL`。

使用 `COALESCE`：

```sql
SELECT
  CONCAT(
    COALESCE(username, '匿名用户'),
    ' / ',
    COALESCE(mobile, backup_mobile, email, '无联系方式')
  ) AS user_contact
FROM users;
```

结果类似：

```text
+-------------------------------+
| user_contact                  |
+-------------------------------+
| zhangsan / 13800000001        |
| lisi / 13900000002            |
| wangwu / wangwu@example.com   |
| 匿名用户 / 无联系方式         |
+-------------------------------+
```

### 示例八：按更新时间或创建时间排序

很多表都有 `updated_at` 和 `created_at`。

列表排序时，希望优先按更新时间排序；没有更新时间时，按创建时间排序。

```sql
SELECT
  order_no,
  updated_at,
  created_at
FROM orders
ORDER BY COALESCE(updated_at, created_at) DESC;
```

`COALESCE(updated_at, created_at)` 表示：

```text
有更新时间就用更新时间，没有更新时间就用创建时间。
```

### 示例九：没有排序号的排最后

`sort_no` 有值的按 `sort_no` 排，没有值的排最后。

```sql
SELECT
  order_no,
  sort_no
FROM orders
ORDER BY COALESCE(sort_no, 999999) ASC;
```

结果类似：

```text
+----------+---------+
| order_no | sort_no |
+----------+---------+
| A003     |       1 |
| A001     |       2 |
| A002     |    NULL |
| A004     |    NULL |
+----------+---------+
```

这里的 `999999` 是一个足够大的兜底值，让 `NULL` 排到后面。

### 示例十：UPDATE 时保留旧值

接口更新资料时，有些字段可能没传。没传的字段通常不希望被更新成 `NULL`。

假设传入的新手机号为 `@new_mobile`，新头像为 `@new_avatar`：

```sql
SET @new_mobile = NULL;
SET @new_avatar = '/avatar/new.png';

UPDATE users
SET
  mobile = COALESCE(@new_mobile, mobile),
  avatar = COALESCE(@new_avatar, avatar)
WHERE id = 1;
```

含义：

```text
新值不是 NULL 就用新值，新值是 NULL 就保留原字段值。
```

这种写法适合“部分字段更新”。不过如果业务允许把字段主动清空为 `NULL`，就不能单纯依赖这种写法，需要区分“没传字段”和“传了 NULL”。

### COALESCE 和 IFNULL 的区别

`IFNULL` 是 MySQL 常用函数：

```sql
IFNULL(expr1, expr2)
```

含义：

```text
expr1 不是 NULL，返回 expr1；否则返回 expr2。
```

它只能接收两个参数。

```sql
SELECT IFNULL(NULL, '默认值');
```

等价于：

```sql
SELECT COALESCE(NULL, '默认值');
```

`COALESCE` 可以接收多个参数：

```sql
SELECT COALESCE(nickname, real_name, username, '匿名用户');
```

简单二选一时，`IFNULL` 和 `COALESCE` 都可以。多级兜底时，`COALESCE` 更合适。

另外，`COALESCE` 是标准 SQL，跨数据库兼容性更好。

### COALESCE 和 NULLIF 经常一起用

`NULLIF(a, b)` 的意思是：

```text
如果 a = b，返回 NULL；否则返回 a。
```

例如：

```sql
SELECT NULLIF('', '');
```

结果：

```text
NULL
```

所以它经常和 `COALESCE` 组合，用来把空字符串、特殊值先转成 `NULL`，再做兜底。

把空字符串当成空值：

```sql
SELECT COALESCE(NULLIF(nickname, ''), '匿名用户') AS display_name
FROM users;
```

把 `0` 当成无效金额：

```sql
SELECT COALESCE(NULLIF(amount, 0), 1) AS safe_amount
FROM orders;
```

上面表示：如果 `amount` 是 `0`，先转成 `NULL`，再兜底为 `1`。

### COALESCE 和 CASE WHEN 的关系

下面两段 SQL 逻辑接近。

```sql
SELECT COALESCE(nickname, real_name, username, '匿名用户') AS display_name
FROM users;
```

用 `CASE WHEN` 写：

```sql
SELECT
  CASE
    WHEN nickname IS NOT NULL THEN nickname
    WHEN real_name IS NOT NULL THEN real_name
    WHEN username IS NOT NULL THEN username
    ELSE '匿名用户'
  END AS display_name
FROM users;
```

只是判断 `NULL` 并按顺序兜底时，`COALESCE` 更短。

如果要写复杂条件，比如状态、金额区间、分数等级，`CASE WHEN` 更适合。

### 常见坑一：空字符串不是 NULL

```sql
SELECT COALESCE('', '默认值') AS result;
```

结果是空字符串，不是 `默认值`。

因为：

```text
'' 是一个长度为 0 的字符串，但它不是 NULL。
```

如果空字符串也要兜底：

```sql
SELECT COALESCE(NULLIF('', ''), '默认值') AS result;
```

结果：

```text
默认值
```

### 常见坑二：参数类型尽量一致

不推荐：

```sql
SELECT COALESCE(NULL, 100, '未知');
```

参数里既有数字，又有字符串，后续参与计算或排序时容易引发隐式类型转换。

推荐保持同一类结果：

```sql
SELECT COALESCE(CAST(score AS CHAR), '未知') AS score_text
FROM users;
```

或者：

```sql
SELECT COALESCE(score, 0) AS score
FROM users;
```

### 常见坑三：WHERE 里包字段可能影响索引

不推荐：

```sql
SELECT *
FROM users
WHERE COALESCE(mobile, backup_mobile) = '13800000001';
```

这种写法把字段包进函数表达式里，优化器更难直接使用普通索引。

可以改成更明确的条件：

```sql
SELECT *
FROM users
WHERE mobile = '13800000001'
   OR (mobile IS NULL AND backup_mobile = '13800000001');
```

如果查询特别高频，可以考虑补充合适索引、生成列或在数据写入时提前维护一个统一联系方式字段。

### 常见坑四：不要把缺失数据直接伪装成真实数据

`COALESCE` 很适合展示兜底，但不一定适合所有统计场景。

例如平均订单金额：

```sql
SELECT AVG(COALESCE(amount, 0)) AS avg_amount
FROM orders;
```

这表示：`amount` 为 `NULL` 的订单也按 0 参与平均值计算。

如果 `NULL` 的真实含义是“未知金额”，这样会拉低平均值。

另一种写法：

```sql
SELECT AVG(amount) AS avg_amount
FROM orders;
```

`AVG` 会忽略 `NULL`。

所以统计时要先确认 `NULL` 的业务含义：

```text
NULL 是应该当作 0，还是应该当作未知并排除？
```

这个区别会直接影响报表结果。

### 实战建议

* 展示字段缺省值，用 `COALESCE(field, '默认值')`
* 多字段优先级取值，用 `COALESCE(a, b, c, '兜底值')`
* 金额计算前，把可为空金额转成 0，例如 `COALESCE(discount, 0)`
* `SUM` 没有结果时返回 0，用 `COALESCE(SUM(amount), 0)`
* `COUNT(*)` 本身会返回 0，通常不需要再包 `COALESCE`
* 空字符串要当成空值时，配合 `NULLIF(field, '')`
* 只是简单二选一，`IFNULL` 也能用；多级兜底优先用 `COALESCE`
* `WHERE` 中不要随手把索引字段包进 `COALESCE`
* 参数返回类型尽量一致，减少隐式转换
* 统计场景里先判断 `NULL` 的业务含义，再决定是否兜底为 0

### 总结

`COALESCE` 的核心就是“按顺序找第一个非 `NULL` 的值”。

它看起来只是一个小函数，但在列表展示、报表统计、字符串拼接、多字段兼容、`LEFT JOIN` 补默认值中都非常实用。

写 SQL 时，只要遇到“字段可能为空，但结果不能空”的场景，就可以考虑 `COALESCE`。同时要记住两个边界：空字符串不是 `NULL`，统计里的 `NULL` 不一定等于 0。处理好这两点，空值兜底就不容易出错。
