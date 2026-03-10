### 简介

在 `C#.NET` 里，很多人第一次接触表达式树，通常是因为 `LINQ`、`Entity Framework`，或者某段代码里突然冒出了这样一行：

```csharp
Expression<Func<User, bool>> predicate = x => x.Age >= 18;
```

表面上看，它和普通 `Lambda` 很像，但本质完全不同。

* `Func<User, bool>` 表示“可执行代码”；
* `Expression<Func<User, bool>>` 表示“可分析、可遍历、可改写的代码结构”。

这就是表达式树（`Expression Tree`）最核心的价值：

> 把代码从“直接执行”变成“可以像数据一样读取和操作”。

它看起来偏底层，但真实项目里非常常见：

* `LINQ to Entities` 把表达式树翻译成 `SQL`；
* 动态查询组件运行时拼装筛选条件；
* `ORM`、映射器、规则引擎用它生成高性能访问器；
* 框架用它替代部分反射，提高性能和类型安全。

如果你只把表达式树理解成“高级语法”，那会很难学；如果把它理解成“运行时可操作的代码 AST”，很多问题就清楚了。

### 表达式树到底是什么？

表达式树本质上是一棵对象树，用来描述一段表达式代码。

比如：

```csharp
x => x + 1
```

在表达式树里，不是一个“直接可执行的方法体”，而是大致会被拆成：

* 一个参数节点：`x`
* 一个常量节点：`1`
* 一个二元运算节点：`x + 1`
* 一个 `Lambda` 节点：`x => x + 1`

也就是说，表达式树描述的是“这段代码长什么样”，而不只是“这段代码怎么算”。

### 先分清：表达式树和委托不是一回事

这是最重要的入门分界线。

```csharp
Func<int, int> func = x => x + 1;
Expression<Func<int, int>> expr = x => x + 1;
```

它们写起来几乎一样，但语义不同：

| 类型 | 本质 | 能做什么 |
| --- | --- | --- |
| `Func<int, int>` | 委托 | 直接执行 |
| `Expression<Func<int, int>>` | 代码结构对象 | 分析、改写、翻译、编译 |

举个最直接的区别：

```csharp
Func<int, int> func = x => x + 1;
Console.WriteLine(func(10)); // 11
```

你只能执行它。

而表达式树可以先看结构：

```csharp
Expression<Func<int, int>> expr = x => x + 1;

Console.WriteLine(expr);              // x => (x + 1)
Console.WriteLine(expr.Body.NodeType); // Add
```

然后也可以再编译执行：

```csharp
var compiled = expr.Compile();
Console.WriteLine(compiled(10)); // 11
```

所以委托和表达式树的关系可以理解为：

* 委托偏“运行”；
* 表达式树偏“描述 + 运行”。

### 为什么表达式树这么重要？

因为只要一段代码能被表达成结构化对象，框架就有机会做很多事：

* 分析它表达了什么；
* 改写其中一部分；
* 翻译成另一种语言；
* 生成更高性能的运行时代码。

`Entity Framework` 就是最经典的例子：

```csharp
db.Users.Where(x => x.Age >= 18)
```

这里的 `x => x.Age >= 18` 如果只是普通委托，那 `EF Core` 根本没法把它翻译成 `SQL`。

正因为它是表达式树，框架才能看懂：

* 参数是谁；
* 访问了哪个属性；
* 运算符是什么；
* 常量值是多少。

然后再生成对应的 `SQL WHERE Age >= 18`。

### 表达式树的核心类型

表达式树位于：

```csharp
using System.Linq.Expressions;
```

最核心的基类是：

```csharp
Expression
```

常见节点类型如下：

| 类型 | 说明 | 示例 |
| --- | --- | --- |
| `LambdaExpression` | Lambda 节点 | `x => x + 1` |
| `ParameterExpression` | 参数节点 | `x` |
| `ConstantExpression` | 常量节点 | `1`、`"abc"` |
| `BinaryExpression` | 二元运算 | `x + 1`、`x > 18` |
| `UnaryExpression` | 一元运算 | `!isDeleted` |
| `MemberExpression` | 成员访问 | `x.Name` |
| `MethodCallExpression` | 方法调用 | `x.Name.Contains("A")` |
| `ConditionalExpression` | 条件表达式 | `condition ? a : b` |
| `NewExpression` | `new` 对象创建 | `new UserDto(...)` |
| `BlockExpression` | 代码块表达式 | 多步组合表达式 |

表达式树不是“一个类”，而是一整套节点类型系统。

### 表达式树从哪里来？

主要有两种来源：

#### 1. 编译器把 Lambda 自动转换成表达式树

这是最常见的来源。

```csharp
Expression<Func<int, bool>> expr = x => x > 10;
```

注意这里的左侧必须是 `Expression<TDelegate>`，编译器才会把右侧 `Lambda` 转成表达式树。

如果左侧是 `Func<int, bool>`，得到的就是普通委托。

#### 2. 运行时手动构建

这在动态查询、框架开发、规则引擎中很常见。

例如手动构建：

```csharp
x => x > 10
```

可以写成：

```csharp
using System.Linq.Expressions;

ParameterExpression parameter = Expression.Parameter(typeof(int), "x");
ConstantExpression constant = Expression.Constant(10);
BinaryExpression body = Expression.GreaterThan(parameter, constant);

Expression<Func<int, bool>> expr =
    Expression.Lambda<Func<int, bool>>(body, parameter);
```

这两种写法本质等价，只是第二种更适合动态生成。

### 看一个最小完整示例

下面这段代码很好地展示了表达式树的几个关键动作：

* 构建
* 查看结构
* 编译
* 执行

```csharp
using System.Linq.Expressions;

ParameterExpression x = Expression.Parameter(typeof(int), "x");
ConstantExpression one = Expression.Constant(1);
BinaryExpression add = Expression.Add(x, one);

Expression<Func<int, int>> expr =
    Expression.Lambda<Func<int, int>>(add, x);

Console.WriteLine(expr);               // x => (x + 1)
Console.WriteLine(expr.Body.NodeType); // Add

Func<int, int> compiled = expr.Compile();
Console.WriteLine(compiled(10));       // 11
```

如果你能把这段代码彻底看懂，表达式树的基础就已经入门了。

### 不是所有 Lambda 都能变成表达式树

这是一个常见误区。

表达式树主要支持“表达式 Lambda”，也就是右侧本身是一个表达式：

```csharp
Expression<Func<int, int>> ok = x => x + 1;
```

但这种语句体 Lambda 就不行：

```csharp
// 这类写法不能直接转换成 Expression<Func<int, int>>
// x =>
// {
//     var y = x + 1;
//     return y;
// }
```

原因很简单：表达式树最初的设计目标就是表达“表达式结构”，而不是完整 `C#` 语法树。

所以它很强，但不是完整的 Roslyn 语法模型。

### 手动构建动态查询，是表达式树最常见的实战场景

假设你要做一个用户筛选接口，前端可能传：

* `Name = "Alice"`
* `MinAge = 18`
* `IsActive = true`

这时候最常见的需求就是在运行时动态拼一个：

```csharp
x => x.Name == "Alice" && x.Age >= 18 && x.IsActive
```

表达式树就是做这件事的标准工具。

先定义实体：

```csharp
public sealed class User
{
    public string Name { get; set; } = string.Empty;
    public int Age { get; set; }
    public bool IsActive { get; set; }
}
```

然后动态拼装：

```csharp
using System.Linq.Expressions;

public static Expression<Func<User, bool>> BuildUserFilter(
    string? name,
    int? minAge,
    bool? isActive)
{
    ParameterExpression parameter = Expression.Parameter(typeof(User), "x");
    Expression body = Expression.Constant(true);

    if (!string.IsNullOrWhiteSpace(name))
    {
        Expression left = Expression.Property(parameter, nameof(User.Name));
        Expression right = Expression.Constant(name);
        Expression equal = Expression.Equal(left, right);
        body = Expression.AndAlso(body, equal);
    }

    if (minAge.HasValue)
    {
        Expression left = Expression.Property(parameter, nameof(User.Age));
        Expression right = Expression.Constant(minAge.Value);
        Expression greaterThanOrEqual = Expression.GreaterThanOrEqual(left, right);
        body = Expression.AndAlso(body, greaterThanOrEqual);
    }

    if (isActive.HasValue)
    {
        Expression left = Expression.Property(parameter, nameof(User.IsActive));
        Expression right = Expression.Constant(isActive.Value);
        Expression equal = Expression.Equal(left, right);
        body = Expression.AndAlso(body, equal);
    }

    return Expression.Lambda<Func<User, bool>>(body, parameter);
}
```

使用时：

```csharp
var filter = BuildUserFilter("Alice", 18, true);

var users = dbContext.Users.Where(filter).ToList();
```

这类模式在后台管理系统、高级搜索、报表筛选里非常常见。

### 为什么动态查询更适合表达式树，而不是委托？

因为下面两者虽然都能“筛选”，但适用场景完全不同。

#### 委托版本

```csharp
Func<User, bool> filter = x => x.Age >= 18;
```

它只能在内存里执行，例如：

```csharp
users.Where(filter)
```

如果 `users` 是数据库查询源，很多提供程序并不能把这个委托翻译成远端查询。

#### 表达式树版本

```csharp
Expression<Func<User, bool>> filter = x => x.Age >= 18;
```

它可以被查询提供程序解析，比如翻译为 `SQL`。

所以一个简单判断规则是：

* 面向 `IEnumerable<T>` 的内存计算，`Func<T, bool>` 很常见；
* 面向 `IQueryable<T>` 的翻译场景，通常要用 `Expression<Func<T, bool>>`。

### 表达式树也能拿来做高性能访问器

反射很灵活，但频繁调用会慢一些。

表达式树常见的另一个用途，就是生成属性读取器、属性设置器、方法调用器。

例如给某个属性生成 getter：

```csharp
using System.Linq.Expressions;

public static Func<T, object?> BuildGetter<T>(string propertyName)
{
    ParameterExpression parameter = Expression.Parameter(typeof(T), "x");
    MemberExpression property = Expression.Property(parameter, propertyName);
    UnaryExpression convert = Expression.Convert(property, typeof(object));

    return Expression.Lambda<Func<T, object?>>(convert, parameter)
        .Compile();
}
```

如果把这段代码翻译回普通 `Lambda`，它本质上是在动态生成：

```csharp
x => (object)x.Name
```

如果 `T` 是 `User`，`propertyName` 是 `nameof(User.Name)`，那逐行来看就是：

#### 第 1 行：定义参数 `x`

```csharp
ParameterExpression parameter = Expression.Parameter(typeof(T), "x");
```

这一行等价于在写：

```csharp
(T x) => ...
```

也就是说，它先创建了一个 `Lambda` 参数节点，名字叫 `x`，类型是 `T`。

#### 第 2 行：访问属性 `x.Name`

```csharp
MemberExpression property = Expression.Property(parameter, propertyName);
```

这一步是在表达：

```csharp
x.Name
```

如果 `propertyName` 传的是 `nameof(User.Name)`，那这行生成的就是“访问参数 `x` 的 `Name` 属性”。

#### 第 3 行：把属性值转换成 `object`

```csharp
UnaryExpression convert = Expression.Convert(property, typeof(object));
```

为什么这里一定要做转换？

因为方法返回的是：

```csharp
Func<T, object?>
```

也就是说最终生成的委托，返回值必须是 `object?`。

但属性本身的真实类型未必是 `object`：

* `Name` 可能是 `string`
* `Age` 可能是 `int`
* `CreatedTime` 可能是 `DateTime`

所以这里统一做一次：

```csharp
(object)x.Name
```

或者：

```csharp
(object)x.Age
```

如果属性是值类型，比如 `int`，这里还会发生一次装箱。

#### 第 4 行：把前面的节点包装成完整 Lambda 并编译

```csharp
return Expression.Lambda<Func<T, object?>>(convert, parameter)
    .Compile();
```

这一步等价于：

```csharp
Func<T, object?> getter = x => (object)x.Name;
```

只不过这里不是手写 `Lambda`，而是把前面手动拼好的表达式树编译成委托。

所以整段代码真正做的事就是：

1. 先拼出 `x`
2. 再拼出 `x.Name`
3. 再拼出 `(object)x.Name`
4. 最后编译成一个真正可执行的 getter

#### 用 `User.Name` 代入后，再看一遍完整语义

```csharp
var getter = BuildGetter<User>(nameof(User.Name));
var value = getter(new User { Name = "Alice" });
Console.WriteLine(value); // Alice
```

你完全可以把它脑补成：

```csharp
Func<User, object?> getter = x => (object)x.Name;
```

这样就容易理解很多了。

#### 为什么它通常比反射更适合高频场景？

反射版本一般是这样：

```csharp
var property = typeof(User).GetProperty(nameof(User.Name));
var value = property!.GetValue(user);
```

这当然很灵活，但如果在高频路径里反复调用，反射链路通常更重。

而表达式树这种方式是：

* 构建一次；
* 编译一次；
* 后面像普通委托一样直接调用很多次。

所以它真正适合的是：

* 属性访问路径固定；
* 会被频繁执行；
* 希望比反射更快，但又保留运行时动态生成能力。

使用：

```csharp
var getter = BuildGetter<User>(nameof(User.Name));
var value = getter(new User { Name = "Alice" });
Console.WriteLine(value); // Alice
```

这类技巧常用于：

* 对象映射；
* 序列化组件；
* 通用仓储；
* 动态排序；
* 框架内部元编程。

### 表达式树怎么遍历和改写？

这时就轮到 `ExpressionVisitor` 出场了。

它是表达式树世界里最常用的访问器基类，适合做：

* 分析节点；
* 替换某些节点；
* 重写表达式结构。

例如，把所有常量 `18` 替换为 `20`：

```csharp
using System.Linq.Expressions;

public sealed class ReplaceConstantVisitor : ExpressionVisitor
{
    protected override Expression VisitConstant(ConstantExpression node)
    {
        if (node.Type == typeof(int) && node.Value is int value && value == 18)
        {
            return Expression.Constant(20);
        }

        return base.VisitConstant(node);
    }
}
```

使用：

```csharp
Expression<Func<User, bool>> expr = x => x.Age >= 18;

var visitor = new ReplaceConstantVisitor();
var newExpr = (Expression<Func<User, bool>>)visitor.Visit(expr)!;

Console.WriteLine(expr);    // x => (x.Age >= 18)
Console.WriteLine(newExpr); // x => (x.Age >= 20)
```

这个能力非常关键，因为很多动态查询库、规则引擎、缓存键生成器，本质上都在做表达式树遍历或改写。

### 组合多个表达式，是表达式树的高频难点

很多项目里会写这种需求：

* 先有一个 `x => x.Age >= 18`
* 再拼一个 `x => x.IsActive`
* 最后得到 `x => x.Age >= 18 && x.IsActive`

看起来简单，但不能直接拿两个 `Body` 拼，因为参数对象必须统一。

一个更稳妥的写法是做参数替换。

```csharp
using System.Linq.Expressions;

public static class PredicateBuilder
{
    public static Expression<Func<T, bool>> And<T>(
        Expression<Func<T, bool>> left,
        Expression<Func<T, bool>> right)
    {
        ParameterExpression parameter = left.Parameters[0];
        var replacer = new ReplaceParameterVisitor(right.Parameters[0], parameter);
        Expression rightBody = replacer.Visit(right.Body)!;

        Expression body = Expression.AndAlso(left.Body, rightBody);
        return Expression.Lambda<Func<T, bool>>(body, parameter);
    }
}

public sealed class ReplaceParameterVisitor : ExpressionVisitor
{
    private readonly ParameterExpression _source;
    private readonly ParameterExpression _target;

    public ReplaceParameterVisitor(ParameterExpression source, ParameterExpression target)
    {
        _source = source;
        _target = target;
    }

    protected override Expression VisitParameter(ParameterExpression node)
    {
        return node == _source ? _target : base.VisitParameter(node);
    }
}
```

使用：

```csharp
Expression<Func<User, bool>> adult = x => x.Age >= 18;
Expression<Func<User, bool>> active = x => x.IsActive;

var combined = PredicateBuilder.And(adult, active);
Console.WriteLine(combined); // x => ((x.Age >= 18) AndAlso x.IsActive)
```

这段代码第一次看时最容易卡住的地方，通常就是这几行：

```csharp
ParameterExpression parameter = left.Parameters[0];
var replacer = new ReplaceParameterVisitor(right.Parameters[0], parameter);
Expression rightBody = replacer.Visit(right.Body)!;
```

它们的作用，其实就是一句话：

> 把 `right` 里的参数，替换成 `left` 使用的那个参数对象。

#### 为什么不能直接把 `left.Body` 和 `right.Body` 拼起来？

先看这两个表达式：

```csharp
Expression<Func<User, bool>> adult = x => x.Age >= 18;
Expression<Func<User, bool>> active = x => x.IsActive;
```

虽然它们都写成了 `x`，但这两个 `x` 在表达式树里并不是同一个对象。

也就是说：

```csharp
adult.Parameters[0] != active.Parameters[0]
```

它们只是名字都叫 `x`，但实际上是两个不同的 `ParameterExpression` 实例。

这一点非常重要。

表达式树认的是“参数对象本身”，不只是参数名。

#### 如果直接拼，会发生什么？

如果你直接写：

```csharp
Expression body = Expression.AndAlso(adult.Body, active.Body);
return Expression.Lambda<Func<User, bool>>(body, adult.Parameters[0]);
```

那么最终 `Lambda` 只绑定了 `adult.Parameters[0]`，但 `active.Body` 里仍然引用着另一个参数对象。

结果就是：

* 最终树里存在一个没有被当前 `Lambda` 绑定的参数；
* 编译或执行时，通常会报“参数未绑定”之类的异常。

所以不能只看“长得像不像”，必须保证它们引用的是同一个参数实例。

#### `replacer.Visit(right.Body)` 到底做了什么？

这一行：

```csharp
Expression rightBody = replacer.Visit(right.Body)!;
```

本质是在遍历 `right.Body` 这棵子树，然后把里面所有：

```csharp
right.Parameters[0]
```

替换成：

```csharp
left.Parameters[0]
```

替换完后，`rightBody` 就不再引用原来的右侧参数，而是改成引用左侧那个统一参数。

于是最后才能安全地拼成：

```csharp
x => x.Age >= 18 && x.IsActive
```

#### 为什么只处理 `right.Body`，`left.Body` 不用处理？

因为这段实现里已经选定：

```csharp
ParameterExpression parameter = left.Parameters[0];
```

也就是说，左侧参数被选为“最终统一参数”。

既然如此：

* `left.Body` 本来就已经绑定到这个参数上；
* 它天然就是正确的，不需要改；
* 只有 `right.Body` 还在用自己的那套参数，所以才要替换。

你也可以反过来写：

* 以 `right.Parameters[0]` 为基准；
* 再去替换 `left.Body`。

原理完全一样，只是这段代码选择了“左边作为标准参数”。

#### 为什么参数名一样还是不行？

因为表达式树不是按字符串比较变量名，而是按节点对象引用来绑定参数。

也就是说下面两者在表达式树里不是一回事：

* 名字都叫 `x`
* 真的是同一个 `ParameterExpression`

这也是表达式树组合时最容易掉坑的地方。

#### 把组合过程翻译成更直白的话

这段代码：

```csharp
var replacer = new ReplaceParameterVisitor(right.Parameters[0], parameter);
Expression rightBody = replacer.Visit(right.Body)!;
Expression body = Expression.AndAlso(left.Body, rightBody);
```

其实就是在做：

1. 先决定最终统一使用左边那个参数；
2. 把右边表达式里的旧参数全部换成左边参数；
3. 再把两个表达式体用 `AndAlso` 拼起来。

所以它不是在“改业务逻辑”，而是在“对齐参数上下文”。

#### 为什么有些文章喜欢用 `Expression.Invoke`，但这里没用？

有些写法会这样组合：

```csharp
var body = Expression.AndAlso(
    Expression.Invoke(left, parameter),
    Expression.Invoke(right, parameter));
```

这种方式在本地执行时通常没问题，但在 `EF Core` 这类查询翻译场景里，经常不如“参数替换后直接拼接”稳定。

所以如果你的目标包括：

* `IQueryable`
* `EF Core`
* 动态条件拼接后还要翻译成 `SQL`

那参数替换通常是更稳妥的方式。

这比直接使用 `Expression.Invoke` 更稳，尤其是在 `EF Core` 这类查询翻译场景里更容易兼容。

### 表达式树和 IQueryable 的关系，必须理解透

很多人学表达式树时最容易卡在这里。

```csharp
IQueryable<User> query = dbContext.Users;
query = query.Where(x => x.Age >= 18);
```

看起来只是写了一个 `Lambda`，但这里之所以能被翻译成数据库查询，是因为 `Queryable.Where` 接收的不是普通委托，而是：

```csharp
Expression<Func<User, bool>>
```

然后 `IQueryable` 背后的 provider 才能读取表达式结构，决定如何翻译。

也就是说：

* `IEnumerable<T>` 更偏本地枚举；
* `IQueryable<T>` 更偏“表达式 + Provider 翻译”。

如果你对这个点不清楚，就很难真正看懂 `LINQ` 提供程序为什么能工作。

### 性能该怎么看？

表达式树不是“无脑高性能”，它的性能要分两段看。

#### 1. 构建和编译阶段

```csharp
var func = expr.Compile();
```

这一步是有成本的。

如果你每次调用都现场构建、现场 `Compile()`，通常不划算。

#### 2. 编译后执行阶段

一旦编译成委托，多次执行通常会很快，往往比反射调用更有优势。

所以经验上：

* 低频场景：直接反射可能更简单；
* 高频场景：表达式树编译后缓存，通常更合适。

例如：

```csharp
private static readonly Func<User, object?> _nameGetter =
    BuildGetter<User>(nameof(User.Name));
```

这种“编译一次，多次复用”的模式，才是表达式树性能优势真正能发挥出来的地方。

### 表达式树的几个典型限制

表达式树很强，但别把它想成“完整版 C# 语法树”。

最值得记住的几个限制：

#### 1. 不是所有 C# 语法都能表达

尤其是复杂语句体、某些语言糖、新语法，不一定都能直接出现在表达式树里。

#### 2. 能构建，不等于能被 Provider 翻译

这是更实际的限制。

例如你手写了一个很复杂的表达式树，`Compile()` 后本地执行可能没问题，但交给 `EF Core` 之后，未必能翻译成 `SQL`。

也就是说要区分两层：

* `Expression` API 能不能构建；
* 目标框架能不能理解并翻译。

#### 3. 闭包会影响表达式结构

例如：

```csharp
int minAge = 18;
Expression<Func<User, bool>> expr = x => x.Age >= minAge;
```

表面上看是常量 `18`，但表达式树里经常会表现为对闭包对象成员的访问，而不是简单 `Constant(18)`。

这也是为什么有些表达式分析代码不能只盯着 `ConstantExpression`。

#### 4. 节点是不可变的

表达式树一旦创建，节点不能原地修改。

所谓“修改表达式”，其实是：

* 遍历原树；
* 创建新节点；
* 组成一棵新树。

这也是 `ExpressionVisitor` 设计成立的原因。

### 几个很有代表性的使用场景

如果你在业务里遇到下面这些问题，表达式树大概率就是正确工具。

#### 1. 动态筛选、动态排序、动态分页条件

后台管理系统和搜索页最常见。

#### 2. ORM / LINQ Provider 查询翻译

这是表达式树最经典的落地场景。

#### 3. 生成高性能 getter/setter

用于替代高频反射。

#### 4. 分析成员路径

例如从：

```csharp
x => x.Name
```

中安全提取 `"Name"`，而不是手写字符串。

#### 5. 规则引擎和 DSL

把运行时规则转换成可执行或可翻译的逻辑树。

### 表达式树和反射、源生成器怎么选？

这几个东西经常会被放在一起比较。

#### 适合表达式树的场景

* 逻辑需要在运行时动态生成；
* 需要可分析、可改写；
* 生成后要高频执行；
* 要和 `IQueryable` / `LINQ Provider` 协作。

#### 适合反射的场景

* 需求简单；
* 调用频率不高；
* 不值得为性能专门建表达式缓存。

#### 适合源生成器的场景

* 逻辑在编译期就能确定；
* 更追求零运行时构建成本；
* 希望把动态问题尽量前移到编译期。

简单说：

* 反射偏简单；
* 表达式树偏运行时动态；
* 源生成器偏编译期生成。

### 一套比较务实的使用建议

如果你准备在项目里真正使用表达式树，下面这些建议很实用：

* 先分清你要的是委托还是表达式树；
* 涉及 `IQueryable` 翻译时，优先使用 `Expression<Func<...>>`；
* 高频执行的表达式，编译后要缓存；
* 做表达式组合时，注意参数统一，不要直接硬拼；
* 做表达式分析时，注意闭包、成员访问和常量节点的差异；
* 不要假设“表达式能构建出来，ORM 就一定能翻译”。

### 总结

表达式树的本质，不是某种“高深语法”，而是一套把代码表达成对象树的运行时模型。

你可以这样理解它：

* `Lambda` 解决“把行为写得更简洁”；
* 委托解决“把行为传来传去”；
* 表达式树解决“把行为本身当数据来读、改、拼、翻译、再执行”。

在现代 `.NET` 项目里，只要你接触这些能力：

* `LINQ Provider`
* 动态查询
* ORM 翻译
* 运行时代码生成
* 反射性能优化

那表达式树几乎都是绕不过去的一项基础能力。
