### 简介

MySQL 从 `5.7.8` 开始支持原生 `JSON` 数据类型。

它适合存储结构不完全固定、但仍然需要按路径查询和修改的数据，例如：

* 商品扩展属性
* 用户个性化配置
* 动态表单内容
* 第三方接口原始响应
* 审批流和规则配置
* 事件附加信息

例如商品的颜色、材质、尺寸可能各不相同，全部拆成普通字段会产生大量可空列：

```json
{
  "brand": "Keychron",
  "color": "black",
  "connection": ["Bluetooth", "2.4G", "USB-C"],
  "specs": {
    "layout": "75%",
    "weight": 0.82
  }
}
```

这类扩展数据可以放进一个 `JSON` 字段，同时保留 MySQL 的事务、关联查询和普通索引能力。

一句话概括：

```text
固定且重要的数据放普通列，结构灵活的扩展数据放 JSON。
```

### JSON 类型和 TEXT 有什么区别？

使用 `TEXT` 也能保存 JSON 字符串，但数据库只会把它当作普通文本。

原生 `JSON` 类型主要有这些特点：

* 插入时自动校验 JSON 格式
* 使用二进制格式存储，读取对象成员和数组元素时不必每次重新解析完整文本
* 支持路径提取、搜索、修改、合并和数组操作
* 支持通过生成列为常用路径建立索引
* 支持 `JSON_TABLE` 把数组或对象展开成关系表

创建 JSON 字段：

```sql
CREATE TABLE demo (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  document JSON
);
```

插入合法 JSON：

```sql
INSERT INTO demo (document)
VALUES ('{"name":"张三","age":28}');
```

插入格式不完整的 JSON：

```sql
INSERT INTO demo (document)
VALUES ('{"name":"张三"');
```

MySQL 会拒绝第二条数据，因为缺少右大括号。

### JSON 适合放什么数据？

比较合适的内容：

```text
商品扩展属性、用户偏好、动态配置、第三方原始数据、变化频率较低的附加信息。
```

更适合普通列的内容：

```text
主键、用户名、订单号、金额、状态、时间、外键以及经常筛选、排序、关联的字段。
```

例如商品价格经常参与排序、统计和范围查询，放在 `DECIMAL` 列里更合适；不同品类才有的颜色、材质、接口类型，可以放进 JSON。

### 准备一组演示数据

下面建立商品表。核心字段使用普通列，灵活属性使用 JSON。

```sql
DROP TABLE IF EXISTS products;

CREATE TABLE products (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  product_name VARCHAR(100) NOT NULL,
  category VARCHAR(50) NOT NULL,
  price DECIMAL(10, 2) NOT NULL,
  attributes JSON,
  tags JSON,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_category (category),
  INDEX idx_price (price)
);
```

插入测试数据：

```sql
INSERT INTO products
(product_name, category, price, attributes, tags, created_at)
VALUES
(
  'K2 机械键盘',
  'keyboard',
  699.00,
  '{
    "brand": "Keychron",
    "color": "black",
    "wireless": true,
    "specs": {
      "layout": "75%",
      "weight": 0.82
    },
    "connections": ["Bluetooth", "USB-C"],
    "variants": [
      {"sku": "K2-BLACK", "color": "black", "stock": 12},
      {"sku": "K2-GRAY", "color": "gray", "stock": 5}
    ]
  }',
  '["办公", "机械键盘", "无线"]',
  '2026-01-01 10:00:00'
),
(
  'MX Master 鼠标',
  'mouse',
  599.00,
  '{
    "brand": "Logitech",
    "color": "gray",
    "wireless": true,
    "specs": {
      "dpi": 8000,
      "weight": 0.141
    },
    "connections": ["Bluetooth", "2.4G"],
    "variants": [
      {"sku": "MX-GRAY", "color": "gray", "stock": 20}
    ]
  }',
  '["办公", "鼠标", "无线"]',
  '2026-01-02 10:00:00'
),
(
  '27 英寸显示器',
  'monitor',
  1899.00,
  JSON_OBJECT(
    'brand', 'Dell',
    'color', 'black',
    'wireless', FALSE,
    'warranty_years', 3,
    'specs', JSON_OBJECT(
      'resolution', '4K',
      'refresh_rate', 60,
      'weight', 5.4
    ),
    'connections', JSON_ARRAY('HDMI', 'DisplayPort', 'USB-C'),
    'variants', JSON_ARRAY(
      JSON_OBJECT('sku', 'U2724Q', 'color', 'black', 'stock', 8)
    )
  ),
  JSON_ARRAY('办公', '显示器', '4K'),
  '2026-01-03 10:00:00'
),
(
  '桌面收纳架',
  'accessory',
  129.00,
  '{
    "brand": null,
    "color": "white",
    "specs": {
      "material": "wood"
    }
  }',
  '["收纳", "桌面"]',
  '2026-01-04 10:00:00'
);
```

这里展示了两种构造 JSON 的方式：

* 直接写合法 JSON 字符串
* 使用 `JSON_OBJECT()` 和 `JSON_ARRAY()` 构造

业务代码中使用构造函数或参数化写入，可以减少引号和转义问题。

### 使用 JSON_OBJECT 创建对象

`JSON_OBJECT` 接收一组键和值：

```sql
SELECT JSON_OBJECT(
  'id', 1,
  'name', '张三',
  'active', TRUE
) AS user_json;
```

结果：

```json
{"id": 1, "name": "张三", "active": true}
```

键和值需要成对出现。

### 使用 JSON_ARRAY 创建数组

```sql
SELECT JSON_ARRAY('Java', 'MySQL', '.NET') AS skills;
```

结果：

```json
["Java", "MySQL", ".NET"]
```

对象和数组可以继续嵌套：

```sql
SELECT JSON_OBJECT(
  'name', '张三',
  'skills', JSON_ARRAY('Java', 'MySQL'),
  'address', JSON_OBJECT('city', '上海', 'district', '浦东')
) AS profile;
```

### JSON 路径语法

MySQL 通过 JSON Path 定位文档里的值。

常用写法：

| 路径 | 含义 |
| --- | --- |
| `$` | 整个 JSON 文档的根 |
| `$.brand` | 根对象里的 `brand` |
| `$.specs.weight` | 嵌套对象里的 `weight` |
| `$.connections[0]` | 数组第一个元素 |
| `$.connections[*]` | 数组所有元素 |
| `$.*` | 根对象的所有成员 |

数组下标从 `0` 开始。

如果键名里包含空格或特殊字符，可以使用双引号：

```sql
SELECT JSON_EXTRACT('{"product name":"键盘"}', '$."product name"');
```

### 使用 JSON_EXTRACT 读取值

读取商品品牌：

```sql
SELECT
  product_name,
  JSON_EXTRACT(attributes, '$.brand') AS brand
FROM products;
```

JSON 字符串值会保留双引号：

```text
"Keychron"
```

`->` 是 `JSON_EXTRACT` 的简写：

```sql
SELECT
  product_name,
  attributes -> '$.brand' AS brand
FROM products;
```

两种写法效果相同。

### 使用 ->> 获取普通文本

页面展示或字符串比较时，通常需要去掉 JSON 字符串外层的双引号。

```sql
SELECT
  product_name,
  attributes ->> '$.brand' AS brand
FROM products;
```

结果类似：

```text
+------------------+-----------+
| product_name     | brand     |
+------------------+-----------+
| K2 机械键盘      | Keychron  |
| MX Master 鼠标   | Logitech  |
| 27 英寸显示器    | Dell      |
| 桌面收纳架       | null      |
+------------------+-----------+
```

三种写法的关系：

```sql
attributes -> '$.brand'

JSON_EXTRACT(attributes, '$.brand')
```

上面两种返回 JSON 值。

```sql
attributes ->> '$.brand'

JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.brand'))
```

上面两种返回去掉 JSON 引号后的值。

### 读取嵌套对象和数组

读取键盘布局和重量：

```sql
SELECT
  product_name,
  attributes ->> '$.specs.layout' AS keyboard_layout,
  attributes ->> '$.specs.weight' AS weight
FROM products
WHERE category = 'keyboard';
```

读取连接方式数组的第一个元素：

```sql
SELECT
  product_name,
  attributes ->> '$.connections[0]' AS first_connection
FROM products;
```

读取整个数组：

```sql
SELECT
  product_name,
  attributes -> '$.connections' AS connections
FROM products;
```

### 使用 JSON_VALUE 提取指定类型

MySQL `8.0.21` 及以上版本可以使用 `JSON_VALUE` 提取标量值，并指定返回类型。

```sql
SELECT
  product_name,
  JSON_VALUE(
    attributes,
    '$.specs.weight' RETURNING DECIMAL(10, 3)
  ) AS weight
FROM products;
```

这种写法适合数字比较、排序和类型要求明确的查询。

### 在 WHERE 中按 JSON 值筛选

查询品牌为 `Logitech` 的商品：

```sql
SELECT
  id,
  product_name
FROM products
WHERE attributes ->> '$.brand' = 'Logitech';
```

查询支持无线连接的商品：

```sql
SELECT
  id,
  product_name
FROM products
WHERE attributes -> '$.wireless' = CAST('true' AS JSON);
```

也可以把值提取为文本：

```sql
SELECT
  id,
  product_name
FROM products
WHERE attributes ->> '$.wireless' = 'true';
```

### JSON 数字比较需要明确类型

`->>` 提取出的内容适合展示，但数字范围查询最好显式转换类型。

查询重量大于 `1` 千克的商品：

```sql
SELECT
  product_name,
  attributes ->> '$.specs.weight' AS weight
FROM products
WHERE CAST(
        attributes ->> '$.specs.weight'
      AS DECIMAL(10, 3)) > 1;
```

MySQL `8.0.21` 及以上版本也可以使用 `JSON_VALUE`：

```sql
SELECT
  product_name,
  JSON_VALUE(
    attributes,
    '$.specs.weight' RETURNING DECIMAL(10, 3)
  ) AS weight
FROM products
WHERE JSON_VALUE(
        attributes,
        '$.specs.weight' RETURNING DECIMAL(10, 3)
      ) > 1;
```

显式类型可以避免字符串比较带来的结果偏差。例如字符串 `'10'` 和 `'2'` 按字符顺序比较时，与数字比较结果不同。

### 使用 JSON_CONTAINS 判断是否包含值

查询标签中包含“办公”的商品：

```sql
SELECT
  product_name,
  tags
FROM products
WHERE JSON_CONTAINS(tags, JSON_QUOTE('办公'));
```

也可以指定路径：

```sql
SELECT
  product_name
FROM products
WHERE JSON_CONTAINS(
  attributes,
  JSON_QUOTE('Bluetooth'),
  '$.connections'
);
```

`JSON_CONTAINS` 的候选值必须是合法 JSON。字符串需要带 JSON 双引号，所以这里使用 `JSON_QUOTE` 构造。

### 使用 MEMBER OF 判断数组成员

MySQL `8.0.17` 及以上版本可以使用 `MEMBER OF` 判断某个值是否属于 JSON 数组：

```sql
SELECT
  product_name
FROM products
WHERE '无线' MEMBER OF(tags);
```

这种写法很适合单个数组成员判断。

### 使用 JSON_OVERLAPS 判断数组是否有交集

MySQL `8.0.17` 及以上版本可以使用 `JSON_OVERLAPS` 判断两个 JSON 文档是否有共同元素。

查询标签中包含“无线”或“4K”的商品：

```sql
SELECT
  product_name,
  tags
FROM products
WHERE JSON_OVERLAPS(
  tags,
  JSON_ARRAY('无线', '4K')
);
```

`JSON_OVERLAPS` 只要找到一个共同元素就返回 `1`。

### 判断路径是否存在

`JSON_CONTAINS_PATH` 可以检查一个或多个路径。

查询包含质保年限字段的商品：

```sql
SELECT
  product_name
FROM products
WHERE JSON_CONTAINS_PATH(
  attributes,
  'one',
  '$.warranty_years'
);
```

第二个参数：

* `'one'`：给出的路径中至少存在一个
* `'all'`：给出的路径必须全部存在

同时检查品牌和颜色：

```sql
SELECT
  product_name
FROM products
WHERE JSON_CONTAINS_PATH(
  attributes,
  'all',
  '$.brand',
  '$.color'
);
```

### 使用 JSON_SEARCH 查找值的位置

`JSON_SEARCH` 返回匹配值所在的路径。

```sql
SELECT
  product_name,
  JSON_SEARCH(attributes, 'one', 'USB-C') AS matched_path
FROM products;
```

可能返回：

```text
"$.connections[1]"
```

只需要判断数组是否包含某个确定值时，`JSON_CONTAINS` 或 `MEMBER OF` 通常更直接；需要知道值出现在哪条路径时，可以使用 `JSON_SEARCH`。

### JSON_SET：存在则更新，不存在则新增

为键盘增加质保年限，同时修改颜色：

```sql
UPDATE products
SET attributes = JSON_SET(
  attributes,
  '$.warranty_years', 2,
  '$.color', 'navy'
)
WHERE id = 1;
```

`JSON_SET` 一次可以处理多组路径和值。

### JSON_INSERT：只在路径不存在时新增

```sql
UPDATE products
SET attributes = JSON_INSERT(
  attributes,
  '$.warranty_years', 1
)
WHERE id = 1;
```

如果 `warranty_years` 已经存在，原值保持不变。

### JSON_REPLACE：只替换已有路径

```sql
UPDATE products
SET attributes = JSON_REPLACE(
  attributes,
  '$.color', 'white',
  '$.unknown_key', 'test'
)
WHERE id = 1;
```

`color` 已存在，会被修改；`unknown_key` 不存在，不会新增。

三者可以这样区分：

| 函数 | 路径已存在 | 路径不存在 |
| --- | --- | --- |
| `JSON_SET` | 更新 | 新增 |
| `JSON_INSERT` | 保留原值 | 新增 |
| `JSON_REPLACE` | 更新 | 保持不变 |

### JSON_REMOVE：删除对象成员或数组元素

删除质保年限：

```sql
UPDATE products
SET attributes = JSON_REMOVE(
  attributes,
  '$.warranty_years'
)
WHERE id = 1;
```

删除连接方式数组里的第一个元素：

```sql
UPDATE products
SET attributes = JSON_REMOVE(
  attributes,
  '$.connections[0]'
)
WHERE id = 1;
```

删除数组元素后，后面的元素会向前移动，下标会重新排列。

### 修改 JSON 数组

在数组末尾追加元素：

```sql
UPDATE products
SET tags = JSON_ARRAY_APPEND(tags, '$', '新品')
WHERE id = 1;
```

在指定位置插入元素：

```sql
UPDATE products
SET tags = JSON_ARRAY_INSERT(tags, '$[1]', '热销')
WHERE id = 1;
```

设置指定下标：

```sql
UPDATE products
SET tags = JSON_SET(tags, '$[0]', '键盘')
WHERE id = 1;
```

### 合并 JSON 文档

MySQL 提供两种常见合并方式。

#### JSON_MERGE_PATCH

后一个对象中的同名键覆盖前一个对象：

```sql
SELECT JSON_MERGE_PATCH(
  '{"color":"black","stock":10}',
  '{"color":"white","warranty":2}'
) AS merged;
```

结果：

```json
{"color": "white", "stock": 10, "warranty": 2}
```

补丁对象里的值为 JSON `null` 时，对应键会被删除：

```sql
SELECT JSON_MERGE_PATCH(
  '{"color":"black","stock":10}',
  '{"stock":null}'
);
```

结果：

```json
{"color": "black"}
```

#### JSON_MERGE_PRESERVE

同名键的值会保留下来，并组合成数组：

```sql
SELECT JSON_MERGE_PRESERVE(
  '{"color":"black"}',
  '{"color":"white"}'
) AS merged;
```

结果：

```json
{"color": ["black", "white"]}
```

配置覆盖通常更接近 `JSON_MERGE_PATCH`；需要保留重复值时可以使用 `JSON_MERGE_PRESERVE`。

### 常用辅助函数

#### JSON_VALID

判断字符串是否为合法 JSON：

```sql
SELECT
  JSON_VALID('{"name":"张三"}') AS valid_json,
  JSON_VALID('{name:"张三"}') AS invalid_json;
```

结果：

```text
+------------+--------------+
| valid_json | invalid_json |
+------------+--------------+
|          1 |            0 |
+------------+--------------+
```

JSON 列在写入时已经自动校验。`JSON_VALID` 更适合校验普通字符串或导入前的数据。

#### JSON_TYPE

查看 JSON 值的类型：

```sql
SELECT
  JSON_TYPE('{"a":1}') AS object_type,
  JSON_TYPE('[1,2,3]') AS array_type,
  JSON_TYPE('123') AS number_type,
  JSON_TYPE('null') AS null_type;
```

常见结果包括 `OBJECT`、`ARRAY`、`STRING`、`INTEGER`、`DOUBLE`、`BOOLEAN` 和 `NULL`。

#### JSON_LENGTH

对象返回成员数量，数组返回元素数量：

```sql
SELECT
  product_name,
  JSON_LENGTH(attributes) AS attribute_count,
  JSON_LENGTH(tags) AS tag_count
FROM products;
```

#### JSON_KEYS

返回对象的键名数组：

```sql
SELECT
  product_name,
  JSON_KEYS(attributes) AS attribute_keys
FROM products;
```

#### JSON_PRETTY

格式化 JSON，适合调试和查看：

```sql
SELECT JSON_PRETTY(attributes)
FROM products
WHERE id = 1;
```

#### JSON_STORAGE_SIZE

查看 JSON 文档二进制存储占用的字节数：

```sql
SELECT
  product_name,
  JSON_STORAGE_SIZE(attributes) AS storage_bytes
FROM products;
```

### SQL NULL、JSON null 和路径缺失

这三种情况含义不同：

```text
SQL NULL：整个数据库字段没有值。
JSON null：JSON 文档里明确存了 null。
路径缺失：JSON 文档中没有这个键。
```

演示：

```sql
SELECT
  JSON_TYPE(
    JSON_EXTRACT('{"brand":null}', '$.brand')
  ) AS json_null_type,
  JSON_EXTRACT('{}', '$.brand') IS NULL AS path_missing;
```

结果含义：

* `json_null_type` 为字符串 `NULL`，表示 JSON 类型是 null
* `path_missing` 为 `1`，表示路径不存在时提取结果是 SQL `NULL`

在表中完整区分三种状态：

```sql
SELECT
  product_name,
  CASE
    WHEN attributes IS NULL THEN 'SQL NULL'
    WHEN JSON_CONTAINS_PATH(attributes, 'one', '$.brand') = 0
      THEN '路径不存在'
    WHEN JSON_TYPE(JSON_EXTRACT(attributes, '$.brand')) = 'NULL'
      THEN 'JSON null'
    ELSE '有具体值'
  END AS brand_state
FROM products;
```

### 使用 JSON_ARRAYAGG 聚合成数组

把同一分类下的商品名聚合成 JSON 数组：

```sql
SELECT
  category,
  JSON_ARRAYAGG(product_name) AS product_names
FROM products
GROUP BY category;
```

结果类似：

```json
["K2 机械键盘", "另一款键盘"]
```

### 使用 JSON_OBJECTAGG 聚合成对象

以商品 ID 为键、商品名为值，聚合成 JSON 对象：

```sql
SELECT JSON_OBJECTAGG(id, product_name) AS product_map
FROM products;
```

结果类似：

```json
{
  "1": "K2 机械键盘",
  "2": "MX Master 鼠标",
  "3": "27 英寸显示器",
  "4": "桌面收纳架"
}
```

### 使用 JSON_TABLE 把数组展开成行

`JSON_TABLE` 可以把 JSON 数组转换成关系表，适用于查询、关联和统计。该功能在 MySQL 8.0 中可用。

把每个商品的连接方式展开：

```sql
SELECT
  p.id,
  p.product_name,
  jt.connection_name
FROM products AS p
JOIN JSON_TABLE(
  p.attributes,
  '$.connections[*]'
  COLUMNS (
    connection_name VARCHAR(50) PATH '$'
  )
) AS jt ON TRUE;
```

结果类似：

```text
+----+------------------+-----------------+
| id | product_name     | connection_name |
+----+------------------+-----------------+
|  1 | K2 机械键盘      | Bluetooth       |
|  1 | K2 机械键盘      | USB-C           |
|  2 | MX Master 鼠标   | Bluetooth       |
|  2 | MX Master 鼠标   | 2.4G            |
|  3 | 27 英寸显示器    | HDMI            |
|  3 | 27 英寸显示器    | DisplayPort     |
|  3 | 27 英寸显示器    | USB-C           |
+----+------------------+-----------------+
```

展开对象数组：

```sql
SELECT
  p.product_name,
  v.sku,
  v.color,
  v.stock
FROM products AS p
JOIN JSON_TABLE(
  p.attributes,
  '$.variants[*]'
  COLUMNS (
    sku VARCHAR(50) PATH '$.sku',
    color VARCHAR(30) PATH '$.color',
    stock INT PATH '$.stock'
  )
) AS v ON TRUE;
```

统计各 SKU 库存：

```sql
SELECT
  p.product_name,
  SUM(v.stock) AS total_stock
FROM products AS p
JOIN JSON_TABLE(
  p.attributes,
  '$.variants[*]'
  COLUMNS (
    stock INT PATH '$.stock'
  )
) AS v ON TRUE
GROUP BY p.id, p.product_name;
```

### 为 JSON 内部属性建立索引

JSON 列不能直接使用普通索引覆盖内部所有路径。常见方案是提取固定路径到生成列，再为生成列建立索引。

为品牌和重量创建生成列：

```sql
ALTER TABLE products
ADD COLUMN attr_brand VARCHAR(50)
  GENERATED ALWAYS AS (
    attributes ->> '$.brand'
  ) STORED,
ADD COLUMN attr_weight DECIMAL(10, 3)
  GENERATED ALWAYS AS (
    attributes ->> '$.specs.weight'
  ) STORED;
```

建立索引：

```sql
CREATE INDEX idx_attr_brand ON products(attr_brand);
CREATE INDEX idx_attr_weight ON products(attr_weight);
```

查询时直接使用生成列：

```sql
SELECT
  id,
  product_name,
  attr_brand
FROM products
WHERE attr_brand = 'Logitech';
```

范围查询：

```sql
SELECT
  product_name,
  attr_weight
FROM products
WHERE attr_weight > 1;
```

检查执行计划：

```sql
EXPLAIN
SELECT
  id,
  product_name
FROM products
WHERE attr_brand = 'Logitech';
```

高频筛选、排序、分组的 JSON 属性适合提取成生成列。偶尔展示的扩展属性可以继续直接通过 JSON 路径读取。

### JSON 默认值的版本差异

为兼容不同 MySQL 版本，JSON 列常写成可空列或由写入语句显式提供值：

```sql
attributes JSON NULL
```

MySQL `8.0.13` 及以上版本支持表达式默认值，JSON 默认值需要写成表达式：

```sql
CREATE TABLE user_settings (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  settings JSON NOT NULL DEFAULT (JSON_OBJECT())
);
```

直接把 JSON 文本写成普通字面量默认值，在部分版本中不被接受：

```sql
settings JSON DEFAULT '{}'
```

需要兼容 MySQL 5.7 或较早 MySQL 8.0 时，可以在插入数据时显式写入 `JSON_OBJECT()`。

### JSON 更新和存储开销

`JSON_SET`、`JSON_REPLACE`、`JSON_REMOVE` 都会返回新的 JSON 值，因此更新语句需要把结果重新赋给原字段：

```sql
UPDATE products
SET attributes = JSON_SET(attributes, '$.color', 'blue')
WHERE id = 1;
```

MySQL 8.0 的 InnoDB 在满足条件时可以对部分 JSON 更新进行原地优化，但这不是所有修改都能触发。文档很大、更新频繁时，仍然需要关注存储、日志量和更新成本。

可以查看存储大小和部分更新留下的可用空间：

```sql
SELECT
  JSON_STORAGE_SIZE(attributes) AS storage_size,
  JSON_STORAGE_FREE(attributes) AS storage_free
FROM products
WHERE id = 1;
```

### 常见问题

#### 使用 LIKE 搜索 JSON 数组

下面的写法把 JSON 当普通字符串处理：

```sql
WHERE tags LIKE '%办公%'
```

它可能匹配到字符串片段，也无法准确表达数组成员关系。

数组成员查询可以使用：

```sql
WHERE JSON_CONTAINS(tags, JSON_QUOTE('办公'))
```

#### 数字提取后直接按字符串比较

```sql
WHERE attributes ->> '$.specs.weight' > '2'
```

需要数字语义时，可以使用 `CAST` 或 `JSON_VALUE ... RETURNING` 明确类型。

#### 把所有业务字段放进 JSON

如果订单金额、用户 ID、状态、创建时间都放进 JSON，关联、约束、索引、排序和统计会变得更复杂。

更稳妥的结构是：

```text
核心字段使用普通列，扩展属性使用 JSON。
```

#### 在 JSON 里保存超大数组

数组持续增长时，每次读取和修改的成本也会增长。如果数组元素需要独立分页、关联、更新或设置唯一约束，拆成明细表通常更合适。

#### 依赖 JSON 对象键的显示顺序

JSON 对象用于按键取值，不适合依赖键的展示顺序。MySQL 存储时会规范化 JSON 文档，键的显示顺序不应作为业务规则。

#### 重复键名

同一个对象出现重复键时，MySQL 规范化后通常保留后面的值：

```sql
SELECT CAST('{"color":"black","color":"white"}' AS JSON);
```

结果只保留一个 `color`。业务数据应保持键名唯一。

### 实战建议

* 核心字段、关联字段和高频查询字段使用普通列
* 扩展属性、动态配置和弱结构数据使用 JSON
* 展示字符串常用 `->>`，保留 JSON 类型时使用 `->`
* 数字筛选使用 `CAST` 或 `JSON_VALUE` 指定类型
* 数组包含判断使用 `JSON_CONTAINS`、`MEMBER OF` 或 `JSON_OVERLAPS`
* 新增或修改路径使用 `JSON_SET`
* 仅新增使用 `JSON_INSERT`，仅替换使用 `JSON_REPLACE`
* 删除路径使用 `JSON_REMOVE`
* JSON 数组需要按行处理时使用 `JSON_TABLE`
* 高频查询路径使用生成列加索引
* 区分 SQL `NULL`、JSON `null` 和路径不存在
* 大型、频繁更新、需要关联的数组考虑拆成明细表

### 常用语法速查

| 需求 | 写法 |
| --- | --- |
| 创建对象 | `JSON_OBJECT('key', value)` |
| 创建数组 | `JSON_ARRAY(value1, value2)` |
| 提取 JSON 值 | `column -> '$.key'` |
| 提取普通文本 | `column ->> '$.key'` |
| 指定类型提取 | `JSON_VALUE(column, '$.key' RETURNING type)` |
| 判断数组包含 | `JSON_CONTAINS(array, candidate)` |
| 判断数组成员 | `value MEMBER OF(array)` |
| 判断数组有交集 | `JSON_OVERLAPS(array1, array2)` |
| 判断路径存在 | `JSON_CONTAINS_PATH(document, 'one', path)` |
| 搜索值所在路径 | `JSON_SEARCH(document, 'one', value)` |
| 新增或修改 | `JSON_SET(document, path, value)` |
| 仅新增 | `JSON_INSERT(document, path, value)` |
| 仅替换 | `JSON_REPLACE(document, path, value)` |
| 删除路径 | `JSON_REMOVE(document, path)` |
| 数组尾部追加 | `JSON_ARRAY_APPEND(document, path, value)` |
| 合并并覆盖同名键 | `JSON_MERGE_PATCH(document1, document2)` |
| 合并并保留重复值 | `JSON_MERGE_PRESERVE(document1, document2)` |
| 获取长度 | `JSON_LENGTH(document)` |
| 获取对象键名 | `JSON_KEYS(document)` |
| 校验 JSON | `JSON_VALID(text)` |
| 查看 JSON 类型 | `JSON_TYPE(document)` |
| 展开成关系表 | `JSON_TABLE(document, path COLUMNS (...))` |
| 聚合成数组 | `JSON_ARRAYAGG(value)` |
| 聚合成对象 | `JSON_OBJECTAGG(key, value)` |

### 总结

MySQL JSON 的价值不只是把一段 JSON 字符串存进数据库，而是让半结构化数据可以被校验、按路径读取、精确搜索、局部修改、展开和索引。

日常使用最常见的几组能力是：

```text
-> 和 ->> 负责读取；
JSON_CONTAINS 负责数组包含判断；
JSON_SET、JSON_REMOVE 负责修改；
JSON_TABLE 负责把数组展开成行；
生成列加索引负责高频查询优化。
```

表结构设计仍然是关键。普通列负责稳定的关系模型，JSON 负责灵活的扩展部分，两者配合使用，才能同时保留结构化查询能力和字段扩展空间。

### 参考资料

* [MySQL 8.4 Reference Manual：The JSON Data Type](https://dev.mysql.com/doc/refman/8.4/en/json.html)
* [MySQL 8.4 Reference Manual：JSON Function Reference](https://dev.mysql.com/doc/refman/8.4/en/json-function-reference.html)
* [MySQL 8.4 Reference Manual：JSON 生成列索引](https://dev.mysql.com/doc/refman/8.4/en/create-table-secondary-indexes.html)
* [MySQL 8.4 Reference Manual：Data Type Default Values](https://dev.mysql.com/doc/refman/8.4/en/data-type-defaults.html)
