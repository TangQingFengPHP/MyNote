### 简介

`PHP` 从 `8.1` 开始原生支持枚举（`enum`），这是 `PHP` 向类型安全和现代语言特性迈进的重要一步。枚举可以定义一组有穷的、不可变的常量集合，常用于表示状态值、选项类型等。

### 基础语法

`PHP` 支持两种类型的枚举：

#### 纯枚举（Pure Enum）

纯枚举没有绑定值，仅代表自身：

```php
enum Status {
    case Draft;
    case Published;
    case Archived;
}
```

使用：

```php
$status = Status::Published;

if ($status === Status::Draft) {
    echo '草稿';
}
```  

#### 具名枚举 / 带值枚举（Backed Enum）

带有标量值（`string` 或 `int`），可用于数据库或 `API` 等场景。

```php
enum Role: string {
    case Admin = 'admin';
    case User = 'user';
    case Guest = 'guest';
}
```

使用：

```php
$role = Role::Admin;

echo $role->value; // 输出 'admin'
```

反查：

```php
$role = Role::from('user');  // Role::User
$role = Role::tryFrom('unknown'); // null（安全版）
```

### 枚举常用方法和特性

* `->name`：获取枚举名（如 `'Admin'`）

* `>value`：获取值（仅限 `Backed Enum`）

* `::cases()`：返回所有枚举项数组

* `::from($value)`：根据值获取枚举，找不到报错

* `::tryFrom($value)`：根据值获取枚举，找不到返回 `null`

示例：

```php
foreach (Role::cases() as $role) {
    echo $role->name . ' => ' . $role->value . PHP_EOL;
}
```

### 枚举 vs 常量类

|  比较点   |  常量类   |  枚举   |
| --- | --- | --- |
|  类型安全   | ❌ 无类型检查    |  ✅ 类型安全   |
|  IDE 智能提示   |  一般   |  更好   |
|  可迭代性   |  ❌   |  ✅ `::cases()`   |
|  反向查找   |  ❌ 手动维护   |  ✅ `from()、tryFrom()`   |
|  可扩展性   |  低   |  ✅ 支持方法、trait、接口等   |

### 枚举中添加方法

```php
enum Status: string {
    case Draft = 'draft';
    case Published = 'published';
    case Archived = 'archived';

    public function label(): string {
        return match($this) {
            self::Draft => '草稿',
            self::Published => '已发布',
            self::Archived => '已归档',
        };
    }
}
```

### 与属性注解结合

结合 `PHP 8` 的属性，可以为枚举项注解：

```php
use Attribute;

#[Attribute]
class Label {
    public function __construct(public string $text) {}
}

enum Status {
    #[Label('草稿')]
    case Draft;

    #[Label('发布')]
    case Published;

    #[Label('归档')]
    case Archived;
}
```

读取方式：

```php
$ref = new ReflectionEnumUnitCase(Status::class, 'Draft');
$attrs = $ref->getAttributes(Label::class);
$label = $attrs[0]->newInstance()->text;

echo $label; // 草稿
```

### 实战用途

* 用户角色：`enum Role: string { case Admin = 'admin'; ... }`

* 状态管理：`enum OrderStatus { case New; case Shipped; }`

* 数据库映射：`存储 enum->value，加载 Role::from($value)`

* 表单选项：枚举生成所有选项下拉框

### 注意事项

* 枚举不能扩展（`final`），但可以实现接口、使用 `trait`；

* 不支持动态添加项；

* 只能用 `string` 或 `int` 作为 `BackedEnum` 的类型；

* 若用于持久化（如数据库），建议使用 `BackedEnum`。