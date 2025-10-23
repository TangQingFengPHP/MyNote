### 简介

在使用 `SqlKata` 构建 `SQL` 时，虽然其链式 `API` 强大灵活，但仍需通过字符串或匿名字段进行表与列的映射，缺乏对实体类型和字段的静态检查。`FluentSqlKata` 基于 `SqlKata`，提供了一套基于表达式的强类型查询构建能力，能够：

* 通过 `Lambda` 表达式指定实体与列，更安全、可重构

* 保留 `SqlKata` 的所有特性与多数据库编译器支持

* 在运行时动态构造 `ORDER BY` 时，支持别名排序等高级功能

### 支持环境与安装

* 目标框架：`.NET Standard 2.0`，兼容 `.NET Framework 4.6.1+`、`.NET Core 2.0+`、`.NET 5/6/7/8/9/10` 等

* 安装命令：

```shell
dotnet add package FluentSqlKata --version 1.1.7
```

### 核心功能

#### 强类型查询

使用 `FluentQuery.Query()` 开始，`From(() => alias)` 指定实体别名，后续所有列、条件、排序均通过表达式指定：

```csharp
var query = FluentQuery.Query()
    .From(() => myCust)
    .Select(() => result.CustomerId, () => myCust.Id)
    .Where(q => q.Where(() => myCust.Name, "John"))
    .OrderByColumn(() => myCust.Name);
```

#### 动态别名排序

对于 `SelectRaw` 或计算列，支持 `OrderByAlias(() => aliasProp)`，自动以别名生成 `ORDER BY`：

```csharp
.SelectRaw(() => model.Name, "ISNULL({0}, 'Unknown')", () => myCust.Name)
.OrderByAlias(() => model.Name);
```

#### 联表与聚合

支持多种 `Join` 重载，包括表达式构造的 `ON` 条件；`GroupBy`、`SelectCount` 等聚合 `API` 与 `SqlKata` 保持一致：

```csharp
.Join(() => myCont, () => myCont.CustomerId, () => myCust.Id)
.SelectCount(() => result.Count, () => myCont.Id)
.GroupBy(() => myCust.Name);
```

### API 详解

| 方法                             | 说明                                     |
| -------------------------------- | ---------------------------------------- |
| `FluentQuery.Query()`            | 创建一个新的强类型查询构建器             |
| `.From(() => alias)`             | 指定根表实体及其别名                     |
| `.Select(exprAlias, exprColumn)` | 添加列映射，并指定结果字段               |
| `.Where(Func<Query, Query>)`     | 通过内部 SqlKata API 构造过滤条件        |
| `.Join(...)`                     | 多种重载，可按表达式或列映射方式构造联表 |
| `.SelectRaw(alias, fmt, expr…)`  | 原生 SQL 片段映射，同时支持别名排序      |
| `.SelectCount(alias, expr)`      | 生成 `COUNT(...) AS alias` 聚合列        |
| `.GroupBy(expr…)`                | 指定分组字段                             |
| `.OrderByColumn(expr)`           | 按指定列排序                             |
| `.OrderByAlias(expr)`            | 按先前定义的列别名排序                   |
| `.Compile(compiler)`             | 使用指定编译器生成最终 SQL 与参数        |

### 用法示例

```csharp
public class Customer
{
    public string Id { get; set; }
    public string Name { get; set; }
    public DateTime LastUpdated { get; set; }
}
```

#### 基本查询

使用 `FluentQuery` 构建类型安全的 `SELECT` 查询：

```csharp
using FluentSqlKata;
using SqlKata;
using SqlKata.Execution;
using System.Data.SqlClient;

public async Task Main()
{
    using var connection = new SqlConnection("Server=localhost;Database=testdb;Trusted_Connection=True;");
    var compiler = new SqlServerCompiler();
    var db = new QueryFactory(connection, compiler);

    Customer myCust = null;
    (string CustomerId, string CustomerName) result = default;

    var query = FluentQuery.Query()
        .From(() => myCust)
        .Select(() => result.CustomerId, () => myCust.Id)
        .Select(() => result.CustomerName, () => myCust.Name);

    var customers = await db.FromQuery(query).GetAsync<(string, string)>();

    foreach (var customer in customers)
    {
        Console.WriteLine($"ID: {customer.Item1}, Name: {customer.Item2}");
    }
}
```

#### 条件查询

使用类型安全的 `Where` 方法：

```csharp
Customer myCust = null;
(string CustomerId, string CustomerName) result = default;

var query = FluentQuery.Query()
    .From(() => myCust)
    .Select(() => result.CustomerId, () => myCust.Id)
    .Select(() => result.CustomerName, () => myCust.Name)
    .Where(q => q.Where(() => myCust.Name, "John")
                .OrWhereContains(() => myCust.Name, "oh"));

var query_str = new SqlServerCompiler().Compile(query).ToString();
Console.WriteLine(query_str);
```

#### 连接（JOIN）

类型安全的 `JOIN` 查询：

```csharp
Contact myCont = null;
Customer myCust = null;
(string FirstName, string LastName, string CustomerId, string CustomerName) result = default;

var query = FluentQuery.Query()
    .From(() => myCont)
    .Join(() => myCust, () => myCust.Id, () => myCont.CustomerId)
    .Select(() => result.FirstName, () => myCont.FirstName)
    .Select(() => result.LastName, () => myCont.LastName)
    .Select(() => result.CustomerId, () => myCont.CustomerId)
    .Select(() => result.CustomerName, () => myCust.Name);

var query_str = new SqlServerCompiler().Compile(query).ToString();
Console.WriteLine(query_str);
```

### 优缺点

#### 优点

* 类型安全：通过表达式引用表和列，编译器捕获拼写错误。

* 灵活性：继承 `SqlKata` 的复杂查询支持（子查询、`JOIN`、`CTE`）。

* 高性能：结合 `Dapper`，性能接近原生 `ADO.NET`。

* 跨数据库支持：通过 `SqlKata` 的编译器适配多种数据库。

#### 缺点

* 学习曲线：表达式语法（如 `() => myCust.Name`）比 `SqlKata` 的字符串语法复杂。

* 部分功能依赖字符串：插入、更新和删除仍使用 `SqlKata` 的字符串-`based API`。

* 文档有限：`FluentSqlKata` 的文档较少，需参考 `SqlKata` 文档和 `GitHub` 示例。

* 依赖 `SqlKata`：增加了依赖项，需熟悉 `SqlKata` 的核心概念。

### 示例项目

```csharp
using Microsoft.AspNetCore.Mvc;
using FluentSqlKata;
using SqlKata;
using SqlKata.Execution;
using System.Threading.Tasks;

public class Customer
{
    public string Id { get; set; }
    public string Name { get; set; }
    public DateTime LastUpdated { get; set; }
}

[ApiController]
[Route("api/customers")]
public class CustomersController : ControllerBase
{
    private readonly QueryFactory _db;

    public CustomersController(QueryFactory db)
    {
        _db = db;
    }

    [HttpGet]
    public async Task<IActionResult> Search([FromQuery] string searchText)
    {
        Customer myCust = null;
        (string CustomerId, string CustomerName) result = default;

        var query = FluentQuery.Query()
            .From(() => myCust)
            .Select(() => result.CustomerId, () => myCust.Id)
            .Select(() => result.CustomerName, () => myCust.Name);

        if (!string.IsNullOrEmpty(searchText))
        {
            query.Where(q => q.WhereContains(() => myCust.Name, searchText));
        }

        var customers = await _db.FromQuery(query).GetAsync<(string, string)>();
        return Ok(customers);
    }
}
```

启动配置：

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        services.AddTransient<QueryFactory>(sp =>
        {
            var connection = new SqlConnection("Server=localhost;Database=testdb;Trusted_Connection=True;");
            var compiler = new SqlServerCompiler();
            return new QueryFactory(connection, compiler);
        });
    }
}
```