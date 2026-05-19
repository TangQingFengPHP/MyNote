### 简介

`CASE WHEN` 是 MySQL 里用来做条件判断的表达式。

它的作用很像编程语言里的 `if...else` 或 `switch...case`：满足不同条件时，返回不同结果。

最常见的场景有这些：

* 把状态码转成状态文字
* 按年龄、金额、分数分段
* 按条件统计数量
* 自定义排序规则
* 报表行转列
* 批量更新不同值
* 处理 `NULL`、空值、异常值

一句话概括：

```text
CASE WHEN 负责在 SQL 里写条件分支。
```

例如订单表里存的是状态码：

```text
0 待支付
1 已支付
2 已发货
3 已完成
```

查询时可以直接转成文字：

```sql
SELECT
  order_no,
  status,
  CASE status
    WHEN 0 THEN '待支付'
    WHEN 1 THEN '已支付'
    WHEN 2 THEN '已发货'
    WHEN 3 THEN '已完成'
    ELSE '未知状态'
  END AS status_text
FROM orders;
```

这样查出来的结果更适合展示，也更适合做报表。

### 两种语法

`CASE WHEN` 有两种写法。

### 第一种：简单 CASE

简单 CASE 适合做等值判断，类似 `switch`。

```sql
CASE 表达式
  WHEN 值1 THEN 结果1
  WHEN 值2 THEN 结果2
  ELSE 默认结果
END
```

示例：

```sql
CASE status
  WHEN 0 THEN '待支付'
  WHEN 1 THEN '已支付'
  ELSE '未知状态'
END
```

它的意思是：

```text
拿 status 分别和 0、1 比较，匹配哪个值，就返回哪个结果。
```

### 第二种：搜索 CASE

搜索 CASE 适合写复杂条件，类似 `if...else if...else`。

```sql
CASE
  WHEN 条件1 THEN 结果1
  WHEN 条件2 THEN 结果2
  ELSE 默认结果
END
```

示例：

```sql
CASE
  WHEN score >= 90 THEN '优秀'
  WHEN score >= 80 THEN '良好'
  WHEN score >= 60 THEN '及格'
  ELSE '不及格'
END
```

这种写法可以使用：

* `>`
* `<`
* `>=`
* `<=`
* `BETWEEN`
* `LIKE`
* `IN`
* `IS NULL`
* `AND`
* `OR`

实际项目里，搜索 CASE 用得更多。

### 执行顺序

`CASE WHEN` 会从上往下依次判断。

一旦某个 `WHEN` 条件成立，就返回对应的 `THEN` 结果，后面的 `WHEN` 不再继续判断。

例如：

```sql
CASE
  WHEN score >= 60 THEN '及格'
  WHEN score >= 90 THEN '优秀'
  ELSE '不及格'
END
```

这段写法有问题。

如果 `score = 95`，第一个条件 `score >= 60` 已经成立，结果会直接返回 `及格`，不会再走到 `score >= 90`。

正确写法应该把更精确、更严格的条件放前面：

```sql
CASE
  WHEN score >= 90 THEN '优秀'
  WHEN score >= 60 THEN '及格'
  ELSE '不及格'
END
```

### ELSE 可以省略吗？

`ELSE` 可以省略。

```sql
CASE
  WHEN score >= 60 THEN '及格'
END
```

如果没有任何条件匹配，结果就是 `NULL`。

所以实际开发里，建议写上 `ELSE`，避免结果出现意料之外的 `NULL`。

```sql
CASE
  WHEN score >= 60 THEN '及格'
  ELSE '不及格'
END
```

### 准备一组演示数据

后面的示例直接基于这几张表。

```sql
DROP TABLE IF EXISTS scores;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS users;

CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  gender TINYINT NOT NULL COMMENT '1 男，2 女，0 未知',
  age INT,
  score INT,
  vip TINYINT NOT NULL DEFAULT 0,
  phone VARCHAR(20),
  created_at DATETIME NOT NULL
);

CREATE TABLE orders (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id INT NOT NULL,
  order_no VARCHAR(50) NOT NULL,
  amount DECIMAL(10, 2) NOT NULL,
  status TINYINT NOT NULL COMMENT '0 待支付，1 已支付，2 已发货，3 已完成，9 已取消',
  level_name VARCHAR(20),
  created_at DATETIME NOT NULL,
  INDEX idx_user_id (user_id),
  INDEX idx_status (status),
  INDEX idx_amount (amount)
);

CREATE TABLE scores (
  id INT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  subject_name VARCHAR(20) NOT NULL,
  score INT NOT NULL
);

INSERT INTO users (username, gender, age, score, vip, phone, created_at) VALUES
('张三', 1, 17, 88, 0, NULL, '2026-01-01 10:00:00'),
('李四', 2, 25, 96, 1, '13800000001', '2026-01-02 10:00:00'),
('王五', 1, 36, 59, 0, '13800000002', '2026-01-03 10:00:00'),
('赵六', 0, NULL, 72, 1, NULL, '2026-01-04 10:00:00');

INSERT INTO orders (user_id, order_no, amount, status, created_at) VALUES
(1, 'A001', 99.00, 0, '2026-01-05 11:00:00'),
(2, 'A002', 680.00, 1, '2026-01-06 11:00:00'),
(2, 'A003', 1280.00, 2, '2026-01-07 11:00:00'),
(3, 'A004', 35.00, 9, '2026-01-08 11:00:00'),
(4, 'A005', 2200.00, 3, '2026-01-09 11:00:00');

INSERT INTO scores (username, subject_name, score) VALUES
('张三', '数学', 90),
('张三', '英语', 82),
('李四', '数学', 96),
('李四', '英语', 91),
('王五', '数学', 58),
('王五', '英语', 64);
```

### 示例一：状态码转文字

数据库里经常存数字状态，页面和报表需要展示文字。

```sql
SELECT
  order_no,
  status,
  CASE status
    WHEN 0 THEN '待支付'
    WHEN 1 THEN '已支付'
    WHEN 2 THEN '已发货'
    WHEN 3 THEN '已完成'
    WHEN 9 THEN '已取消'
    ELSE '未知状态'
  END AS status_text
FROM orders;
```

结果类似：

```text
+----------+--------+-------------+
| order_no | status | status_text |
+----------+--------+-------------+
| A001     |      0 | 待支付      |
| A002     |      1 | 已支付      |
| A003     |      2 | 已发货      |
| A004     |      9 | 已取消      |
| A005     |      3 | 已完成      |
+----------+--------+-------------+
```

这类场景适合使用简单 CASE，因为只是等值匹配。

### 示例二：分数等级

按照分数划分等级，适合使用搜索 CASE。

```sql
SELECT
  username,
  score,
  CASE
    WHEN score >= 90 THEN '优秀'
    WHEN score >= 80 THEN '良好'
    WHEN score >= 60 THEN '及格'
    ELSE '不及格'
  END AS score_level
FROM users;
```

结果类似：

```text
+----------+-------+-------------+
| username | score | score_level |
+----------+-------+-------------+
| 张三     |    88 | 良好        |
| 李四     |    96 | 优秀        |
| 王五     |    59 | 不及格      |
| 赵六     |    72 | 及格        |
+----------+-------+-------------+
```

这种写法比在代码里循环判断更省事，尤其适合报表查询。

### 示例三：年龄分段

`CASE WHEN` 很适合做分段统计。

```sql
SELECT
  username,
  age,
  CASE
    WHEN age IS NULL THEN '未知'
    WHEN age < 18 THEN '未成年'
    WHEN age BETWEEN 18 AND 35 THEN '青年'
    WHEN age BETWEEN 36 AND 59 THEN '中年'
    ELSE '老年'
  END AS age_group
FROM users;
```

这里要注意 `NULL`。

判断空值不能写：

```sql
age = NULL
```

应该写：

```sql
age IS NULL
```

### 示例四：条件统计

报表里很常见的需求：一次查出总人数、VIP 人数、成年人数、及格人数。

```sql
SELECT
  COUNT(*) AS total_count,
  SUM(CASE WHEN vip = 1 THEN 1 ELSE 0 END) AS vip_count,
  SUM(CASE WHEN age >= 18 THEN 1 ELSE 0 END) AS adult_count,
  SUM(CASE WHEN score >= 60 THEN 1 ELSE 0 END) AS pass_count
FROM users;
```

结果类似：

```text
+-------------+-----------+-------------+------------+
| total_count | vip_count | adult_count | pass_count |
+-------------+-----------+-------------+------------+
|           4 |         2 |           2 |          3 |
+-------------+-----------+-------------+------------+
```

核心逻辑很简单：

```text
条件满足返回 1，不满足返回 0，最后 SUM 求和。
```

这比写多条 SQL 再在程序里合并结果更清爽。

### 示例五：COUNT 搭配 CASE WHEN

除了 `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`，也能用 `COUNT(CASE WHEN ... THEN 1 END)`。

```sql
SELECT
  COUNT(CASE WHEN vip = 1 THEN 1 END) AS vip_count,
  COUNT(CASE WHEN score >= 90 THEN 1 END) AS excellent_count
FROM users;
```

`COUNT` 不统计 `NULL`。

当条件不满足时，没有写 `ELSE`，结果就是 `NULL`，所以不会被 `COUNT` 算进去。

两种写法都常见：

```sql
SUM(CASE WHEN 条件 THEN 1 ELSE 0 END)
COUNT(CASE WHEN 条件 THEN 1 END)
```

如果需要更直观，推荐第一种 `SUM` 写法。

### 示例六：按分组统计不同状态订单数

统计每个用户的订单总数、已支付订单数、已取消订单数。

```sql
SELECT
  user_id,
  COUNT(*) AS total_order_count,
  SUM(CASE WHEN status IN (1, 2, 3) THEN 1 ELSE 0 END) AS valid_order_count,
  SUM(CASE WHEN status = 9 THEN 1 ELSE 0 END) AS cancel_order_count
FROM orders
GROUP BY user_id;
```

结果类似：

```text
+---------+-------------------+-------------------+--------------------+
| user_id | total_order_count | valid_order_count | cancel_order_count |
+---------+-------------------+-------------------+--------------------+
|       1 |                 1 |                 0 |                  0 |
|       2 |                 2 |                 2 |                  0 |
|       3 |                 1 |                 0 |                  1 |
|       4 |                 1 |                 1 |                  0 |
+---------+-------------------+-------------------+--------------------+
```

这就是典型的条件聚合。

### 示例七：GROUP BY 里使用 CASE WHEN

按订单金额分区间统计订单数量。

```sql
SELECT
  CASE
    WHEN amount < 100 THEN '100以下'
    WHEN amount < 1000 THEN '100到999'
    ELSE '1000及以上'
  END AS amount_range,
  COUNT(*) AS order_count,
  SUM(amount) AS total_amount
FROM orders
GROUP BY amount_range;
```

结果类似：

```text
+--------------+-------------+--------------+
| amount_range | order_count | total_amount |
+--------------+-------------+--------------+
| 100以下      |           2 |       134.00 |
| 100到999     |           1 |       680.00 |
| 1000及以上   |           2 |      3480.00 |
+--------------+-------------+--------------+
```

这类写法很适合做销售额区间、年龄区间、分数区间统计。

### 示例八：HAVING 里过滤条件统计结果

查出有效订单数大于 1 的用户。

```sql
SELECT
  user_id,
  SUM(CASE WHEN status IN (1, 2, 3) THEN 1 ELSE 0 END) AS valid_order_count
FROM orders
GROUP BY user_id
HAVING valid_order_count > 1;
```

`WHERE` 是分组前过滤，`HAVING` 是分组后过滤。

这里的 `valid_order_count` 是聚合结果，所以放在 `HAVING` 里。

### 示例九：ORDER BY 自定义排序

默认排序不一定符合业务要求。比如订单状态希望按下面顺序展示：

```text
待支付 -> 已支付 -> 已发货 -> 已完成 -> 已取消
```

可以这样写：

```sql
SELECT
  order_no,
  amount,
  status,
  CASE status
    WHEN 0 THEN '待支付'
    WHEN 1 THEN '已支付'
    WHEN 2 THEN '已发货'
    WHEN 3 THEN '已完成'
    WHEN 9 THEN '已取消'
    ELSE '未知状态'
  END AS status_text
FROM orders
ORDER BY
  CASE status
    WHEN 0 THEN 1
    WHEN 1 THEN 2
    WHEN 2 THEN 3
    WHEN 3 THEN 4
    WHEN 9 THEN 5
    ELSE 99
  END,
  created_at DESC;
```

`ORDER BY CASE WHEN` 本质上就是给每一行算一个排序权重。

再比如 VIP 用户排前面：

```sql
SELECT
  id,
  username,
  vip,
  created_at
FROM users
ORDER BY
  CASE WHEN vip = 1 THEN 0 ELSE 1 END,
  created_at DESC;
```

### 示例十：行转列

行转列是 `CASE WHEN` 的高频用法。

原始成绩数据如下：

```text
+----------+--------------+-------+
| username | subject_name | score |
+----------+--------------+-------+
| 张三     | 数学         |    90 |
| 张三     | 英语         |    82 |
| 李四     | 数学         |    96 |
| 李四     | 英语         |    91 |
+----------+--------------+-------+
```

目标结果：

```text
+----------+------------+---------------+
| username | math_score | english_score |
+----------+------------+---------------+
| 张三     |         90 |            82 |
| 李四     |         96 |            91 |
+----------+------------+---------------+
```

SQL 如下：

```sql
SELECT
  username,
  MAX(CASE WHEN subject_name = '数学' THEN score END) AS math_score,
  MAX(CASE WHEN subject_name = '英语' THEN score END) AS english_score
FROM scores
GROUP BY username;
```

为什么要加 `MAX`？

因为 `GROUP BY username` 后，每个用户会合成一行。`CASE WHEN` 先把不同科目的分数放到不同列里，不匹配的地方是 `NULL`，再用 `MAX` 把非空值取出来。

也可以用 `SUM`，前提是同一个用户同一科目只有一条记录：

```sql
SELECT
  username,
  SUM(CASE WHEN subject_name = '数学' THEN score ELSE 0 END) AS math_score,
  SUM(CASE WHEN subject_name = '英语' THEN score ELSE 0 END) AS english_score
FROM scores
GROUP BY username;
```

### 示例十一：批量更新不同值

`CASE WHEN` 不只能用在 `SELECT`，也能用在 `UPDATE`。

根据订单金额更新订单等级：

```sql
UPDATE orders
SET level_name =
  CASE
    WHEN amount >= 1000 THEN '大额订单'
    WHEN amount >= 100 THEN '普通订单'
    ELSE '小额订单'
  END;
```

查看更新结果：

```sql
SELECT order_no, amount, level_name
FROM orders
ORDER BY id;
```

结果类似：

```text
+----------+---------+------------+
| order_no | amount  | level_name |
+----------+---------+------------+
| A001     |   99.00 | 小额订单   |
| A002     |  680.00 | 普通订单   |
| A003     | 1280.00 | 大额订单   |
| A004     |   35.00 | 小额订单   |
| A005     | 2200.00 | 大额订单   |
+----------+---------+------------+
```

这种写法适合一次性修数据、初始化字段、批量打标签。

### 示例十二：在 WHERE 中使用 CASE WHEN

`CASE WHEN` 也可以写在 `WHERE` 里，但要克制。

例如参数 `@only_vip` 为 1 时只查 VIP，为 0 时查全部：

```sql
SET @only_vip = 1;

SELECT
  id,
  username,
  vip
FROM users
WHERE
  CASE
    WHEN @only_vip = 1 THEN vip = 1
    ELSE 1 = 1
  END;
```

这种写法能跑，但可读性一般。

更常见的写法是使用普通逻辑条件：

```sql
SET @only_vip = 1;

SELECT
  id,
  username,
  vip
FROM users
WHERE @only_vip = 0 OR vip = 1;
```

`WHERE` 里优先考虑 `AND`、`OR`、括号组合。`CASE WHEN` 更适合放在 `SELECT`、`ORDER BY`、聚合统计、`UPDATE SET` 里。

### CASE WHEN 和 IF 怎么选？

MySQL 还有一个 `IF` 函数：

```sql
IF(条件, 条件成立的值, 条件不成立的值)
```

示例：

```sql
SELECT
  username,
  IF(score >= 60, '及格', '不及格') AS pass_text
FROM users;
```

等价的 `CASE WHEN` 写法：

```sql
SELECT
  username,
  CASE
    WHEN score >= 60 THEN '及格'
    ELSE '不及格'
  END AS pass_text
FROM users;
```

简单二选一，用 `IF` 更短。

多分支、复杂条件、跨数据库兼容性要求更高时，用 `CASE WHEN` 更合适。

### 常见坑一：忘记 END

错误写法：

```sql
SELECT
  CASE
    WHEN score >= 60 THEN '及格'
    ELSE '不及格'
FROM users;
```

`CASE` 必须用 `END` 结束：

```sql
SELECT
  CASE
    WHEN score >= 60 THEN '及格'
    ELSE '不及格'
  END AS pass_text
FROM users;
```

### 常见坑二：条件顺序写错

错误写法：

```sql
CASE
  WHEN amount >= 100 THEN '普通订单'
  WHEN amount >= 1000 THEN '大额订单'
  ELSE '小额订单'
END
```

`amount = 2000` 时，第一条 `amount >= 100` 已经匹配成功，结果会是 `普通订单`。

正确写法：

```sql
CASE
  WHEN amount >= 1000 THEN '大额订单'
  WHEN amount >= 100 THEN '普通订单'
  ELSE '小额订单'
END
```

### 常见坑三：NULL 判断写错

错误写法：

```sql
CASE
  WHEN phone = NULL THEN '未绑定'
  ELSE '已绑定'
END
```

正确写法：

```sql
CASE
  WHEN phone IS NULL THEN '未绑定'
  ELSE '已绑定'
END
```

SQL 里判断 `NULL` 必须使用 `IS NULL` 或 `IS NOT NULL`。

### 常见坑四：返回类型混乱

不推荐：

```sql
CASE
  WHEN status = 1 THEN 1
  ELSE '未知'
END
```

一个分支返回数字，另一个分支返回字符串，容易触发隐式类型转换。

推荐保持结果类型一致：

```sql
CASE
  WHEN status = 1 THEN '正常'
  ELSE '未知'
END
```

或者全部返回数字：

```sql
CASE
  WHEN status = 1 THEN 1
  ELSE 0
END
```

### 常见坑五：对索引字段做函数计算

`CASE WHEN` 本身不是性能问题，真正要注意的是条件写法。

例如：

```sql
CASE
  WHEN YEAR(created_at) = 2026 THEN '今年订单'
  ELSE '其他订单'
END
```

如果这种条件放在 `WHERE` 过滤里，`YEAR(created_at)` 会让字段先做函数计算，索引更难发挥作用。

更推荐写成范围条件：

```sql
CASE
  WHEN created_at >= '2026-01-01'
   AND created_at < '2027-01-01' THEN '今年订单'
  ELSE '其他订单'
END
```

查询过滤也是同理：

```sql
WHERE created_at >= '2026-01-01'
  AND created_at < '2027-01-01'
```

### 实战建议

* 等值转换用简单 CASE，例如状态码转文字
* 范围判断用搜索 CASE，例如年龄段、金额段、分数段
* 条件统计优先使用 `SUM(CASE WHEN 条件 THEN 1 ELSE 0 END)`
* 行转列常用 `MAX(CASE WHEN 条件 THEN 值 END)`
* 自定义排序可以在 `ORDER BY` 里写 `CASE WHEN`
* 批量更新不同值可以在 `UPDATE SET` 里写 `CASE WHEN`
* `ELSE` 尽量补上，避免意外返回 `NULL`
* 多个分支要注意顺序，范围大的条件不要放太前
* 返回结果尽量保持同一类型，减少隐式转换
* `WHERE` 里优先使用普通条件组合，复杂过滤不要硬塞 `CASE WHEN`

### 总结

`CASE WHEN` 的核心作用，就是在 SQL 里做条件判断。

它不是只能用来把数字转文字，还能做分段、统计、排序、行转列、批量更新。尤其是报表查询中，`SUM(CASE WHEN ...)` 和 `MAX(CASE WHEN ...)` 非常常见。

写的时候重点关注三件事：条件顺序、默认值、返回类型。顺序决定命中哪个分支，`ELSE` 决定兜底结果，返回类型决定后续展示和计算是否稳定。把这三点处理好，`CASE WHEN` 基本就不会写乱。
