## 一、通配符

### 什么是通配符？

通配符用于替换字符串中的一个或多个字符。

通配符通常与LIKE、NOT LIKE操作符一起使用。LIKE操作符在WHERE子句中用于搜索列中的指定模式。

### Mysql 有哪些通配符？

* `%` ：百分号通配符，表示匹配0个或多个字符。

* `_` ：短划线通配符，表示匹配任意的一个字符。

### 通配符用法示例

* 使用 `%` 匹配以指定值开头的

```sql
WHERE name LIKE 'a%'
```

* 使用 `%` 匹配以指定值结尾的

```sql
WHERE name LIKE '%a'
```

* 使用两个 `%` 匹配包含指定值的

```sql
WHERE name LIKE '%a%'
```

* 例如 `a%b` %放在值中间，匹配以 a开头，以b结尾的

```sql
WHERE name LIKE 'a%b'
```

* 例如 `_ondon` 可以匹配 london、aondon等

```sql
WHERE name LIKE '_ondon'
```

* 例如 `londo_` 可以匹配 london、londom等

```sql
WHERE name LIKE 'londo_'
```

* 例如 `h_t` 可以匹配 hot、hat等，中间可以是任意一个字符

```sql
WHERE name LIKE 'h_t'
```

* 例如 `_r%` 可以匹配以某某r开头的值

```sql
WHERE name LIKE '_r%'
```

* 例如 `a_%_%` 可以匹配以a开头后面接两个任意字符的值

```sql
WHERE name LIKE 'a_%_%'
```

* 例如匹配长度为5的任意字符，则使用5个 `_`

```sql
WHERE name LIKE '_____'
```

* `%` 或 `_` 本身作为匹配的字符时，使用 `ESCAPE` 来转义处理

```sql
WHERE name LIKE '67#%%' ESCAPE '#'

此处使用 ESCAPE 语句指定 # 作为转义字符，用来转义第一个%，所以此查询可以查询 67%

既然ESCAPE可以自定义指定转义字符，那么，如下：

WHERE name LIKE '67=%%' ESCAPE '='，与上面是对等的。
```

* 通过 `\` 直接转义，与 `ESCAPE` 效果相同，例如查询包含 `_` 的记录，即可以使用 `ESCAPE` ，也可以使用 `\` 来直接转义，如下：

```sql
WHERE name LIKE '%#_%' ESCAPE '#' 或

WHERE name LIKE '%\_%'
```

## 二、模式匹配

除了使用 `LIKE`、`NOT LIKE`，还可以使用 `REGEXP` 和 `NOT REGEXP` 操作符，同时还有 `RLIKE` 和 `NOT RLIKE` ，此为 `REGEXP` 和 `NOT REGEXP` 的同义词，两者效果相同。

### 模式匹配示例

* 使用 `REGEXP` 匹配指定值开头的，默认不区分大小写

```sql
WHERE name REGEXP '^b'
```

* 使用 `REGEXP` 匹配指定值开头的，且区分大小写

```sql
WHERE name REGEXP BINARY '^b'

使用BINARY关键字使其中一个字符串成为二进制字符串
```

* 使用 `REGEXP` 匹配指定值结尾的

```sql
WHERE name REGEXP 'b$'
```

* 使用 `REGEXP` 匹配包含指定值的

```sql
WHERE name REGEXP 'b'
```

* 使用 `REGEXP` 匹配长度为5的值（方法一）

```sql
WHERE name REGEXP '^.....$'

. 表示匹配任意值
```

* 使用 `REGEXP` 匹配长度为5的值（方法二）

```sql
WHERE name REGEXP '^.{5}$'

{5} 表示重复前面的值5次，此处为.即重复任意值5次
```