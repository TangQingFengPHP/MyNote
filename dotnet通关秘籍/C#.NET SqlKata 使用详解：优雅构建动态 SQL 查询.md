### 简介

* 在复杂项目中，为了保持 `SQL` 灵活性与可读性，开发者往往需要手写大量拼接字符串或使用 `ORMs` 附带的 `LINQ`，但两者各有局限：手写拼接易出错、难以维护；`LINQ` 在某些场景下生成的 `SQL` 不够直观或性能不佳。

* `SqlKata` 是一款轻量级、数据库无关的查询构建器（`Query Builder`），提供——

    * 流式 `API`，链式调用拼装 `SQL`

    * 可切换编译器，支持多种数据库方言（`SQL Server、PostgreSQL、MySQL、SQLite、Oracle` 等）

    * 语法可读，生成的 `SQL` 与手写风格接近，便于调试和维护

### 支持环境与安装

* 目标框架：`.NET Standard 2.0+`，兼容 `.NET Framework 4.6.1` 及更高、`.NET Core 2.1+、.NET 5/6/7/8+`。

* 安装 `NuGet` 包：

```shell
Install-Package SqlKata
```

* 若需内置执行支持（与 `Dapper` 集成），可安装：

```shell
Install-Package SqlKata.Execution
```

* 在代码中引入命名空间：

```csharp
using SqlKata;
using SqlKata.Compilers;
using SqlKata.Execution;
```

#### 数据库驱动

根据目标数据库，安装对应的 `ADO.NET` 提供程序：

* `SQL Server：System.Data.SqlClient`

* `MySQL：MySql.Data` 或  `MysqlConnector`

* `PostgreSQL：Npgsql`

* `SQLite：System.Data.SQLite`

* `Oracle：Oracle.ManagedDataAccess`

* `Firebird：FirebirdSql.Data.FirebirdClient`

#### 项目配置

在 `ASP.NET Core` 项目中，可以通过依赖注入（`DI`）配置 `QueryFactory`：

```csharp
using Microsoft.Extensions.DependencyInjection;
using SqlKata;
using SqlKata.Execution;
using System.Data.SqlClient;

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddTransient<QueryFactory>(sp =>
        {
            var connection = new SqlConnection("你的连接字符串");
            var compiler = new SqlServerCompiler();
            return new QueryFactory(connection, compiler);
        });
    }
}
```

### 核心功能

#### Query Builder

* 基本查询：

```csharp
var query = new Query("Users")
    .Select("Id", "Name", "Email")
    .Where("IsActive", true)
    .OrderByDesc("CreatedAt")
    .Limit(10, 20);  // OFFSET 10 ROWS FETCH NEXT 20 ROWS
```

* 链式拼装：支持 `.Where(), .OrWhere(), .WhereIn(), .WhereBetween(), .Join(), .GroupBy(), .Having()` 等常用子句。

* 条件查询

使用 `Where` 方法添加条件：

```csharp
var cars = await db.Query("cars")
    .Where("price", ">", 20000)
    .Where("name", "like", "%Audi%")
    .GetAsync<Car>();
```

* 动态查询

根据条件动态构建查询：

```csharp
public async Task<IEnumerable<Car>> SearchCars(string searchText, int? maxPrice)
{
    var query = db.Query("cars");

    if (!string.IsNullOrEmpty(searchText))
    {
        query.WhereLike("name", $"%{searchText}%");
    }

    if (maxPrice.HasValue)
    {
        query.Where("price", "<=", maxPrice.Value);
    }

    return await query.GetAsync<Car>();
}
```

* 连接（`JOIN`）

支持多种连接类型（如内连接、左连接）：

```csharp
var query = db.Query("Course")
    .Join("Department", "Department.ID", "Course.DepartmentID")
    .LeftJoin("Instructor", "Instructor.ID", "Course.InstructorID")
    .Select("Course.Title", "Department.Name as DepartmentName")
    .Where("Department.ID", 1);

var results = await query.GetAsync();
```

* 子查询和复杂查询

支持子查询和嵌套条件：

```csharp
var lastPurchaseQuery = db.Query("Transactions")
    .Where("Type", "Purchase")
    .GroupBy("UserId")
    .SelectRaw("MAX([Date]) as LastPurchaseDate");

var users = await db.Query("Users")
    .Include("LastPurchase", lastPurchaseQuery)
    .ForPage(1, 10)
    .GetAsync();
```

* 插入、更新和删除

`SqlKata` 也可以执行插入、更新和删除操作：

```csharp
// 插入
var insertQuery = db.Query("cars").AsInsert(new { name = "BMW", price = 30000 });
await insertQuery.ExecuteAsync();

// 更新
var updateQuery = db.Query("cars")
    .Where("id", 1)
    .AsUpdate(new { price = 35000 });
await updateQuery.ExecuteAsync();

// 删除
var deleteQuery = db.Query("cars")
    .Where("id", 1)
    .AsDelete();
await deleteQuery.ExecuteAsync();
```

#### 编译器（Compiler）

* 多数据库方言：根据目标数据库，选择对应 `Compiler`，生成符合方言的 `SQL` 与参数。

```csharp
var compiler = new SqlServerCompiler();
var sqlResult = compiler.Compile(query);

// sqlResult.Sql => "SELECT [Id], [Name], [Email] FROM [Users] WHERE [IsActive] = @p0 ORDER BY [CreatedAt] DESC OFFSET @p1 ROWS FETCH NEXT @p2 ROWS ONLY"
// sqlResult.NamedBindings => { p0 = true, p1 = 10, p2 = 20 }
```

* 参数化安全：所有输入自动转为参数，防止 `SQL` 注入。

#### 扩展执行层（Execution）

* 与 `Dapper` 整合：通过 `QueryFactory`，可直接执行并映射结果。

```csharp
// 创建连接与工厂
using var connection = new SqlConnection(connectionString);
var compiler = new SqlServerCompiler();
var db = new QueryFactory(connection, compiler);

// 查询单条
var user = await db.Query("Users")
                   .Where("Id", 123)
                   .FirstOrDefaultAsync<User>();

// 查询列表
var activeUsers = await db.Query("Users")
                          .Where("IsActive", true)
                          .GetAsync<User>();

// 插入并返回自增 ID
var newId = await db.Query("Users")
                    .InsertGetIdAsync<int>(new {
                        Name = "Bob",
                        Email = "bob@example.com",
                        CreatedAt = DateTime.UtcNow
                    });
```

* 事务支持：在 `QueryFactory` 上使用 `db.Transaction(...)` 或手动传入 `IDbTransaction`。

#### 原生 SQL 混用

* `Raw SQL`：可在查询中插入原生片段，或完全执行自定义语句。

```csharp
var query = new Query()
    .FromRaw("Users u INNER JOIN Orders o ON u.Id = o.UserId")
    .SelectRaw("u.Id, u.Name, COUNT(o.Id) AS OrderCount")
    .GroupByRaw("u.Id, u.Name");
```

* 自定义函数：原样插入函数调用，例如 `WhereRaw("DATEDIFF(day, CreatedAt, GETDATE()) < 30")`。

### 常用 API 详解

| API                                        | 说明                                                      |
| ------------------------------------------ | --------------------------------------------------------- |
| `new Query(table)`                         | 创建针对指定表的查询对象                                  |
| `.Select(cols…)`                           | 指定要查询的列                                            |
| `.Where(col, op, val)`                     | 添加 `WHERE` 条件（支持省略 `op`，默认为 `=`）            |
| `.OrWhere(...)`                            | 添加 `OR` 条件                                            |
| `.WhereIn(col, array)`                     | `WHERE col IN (…)`                                        |
| `.WhereBetween(col, lo, hi)`               | `WHERE col BETWEEN lo AND hi`                             |
| `.Join(table, c1, op, c2)`                 | 内连接                                                    |
| `.LeftJoin(…)`/.RightJoin                  | 左/右连接                                                 |
| `.GroupBy(cols…)`                          | 分组                                                      |
| `.Having(...)`                             | `HAVING` 子句                                             |
| `.OrderBy(col)`                            | 升序排序                                                  |
| `.OrderByDesc(col)`                        | 降序排序                                                  |
| `.Limit(offset, count)`                    | 分页（SQL Server 使用 OFFSET…FETCH，MySQL/PG 使用 LIMIT） |
| `.Compile(compiler)`                       | 生成 SQL 文本与参数                                       |
| `QueryFactory.GetAsync<T>()`               | 执行查询并映射为实体列表                                  |
| `InsertAsync(object)`                      | 插入新记录                                                |
| `InsertGetIdAsync<T>(object)`              | 插入并返回自增主键                                        |
| `UpdateAsync(object)`                      | 根据实体的主键更新记录                                    |
| `DeleteAsync(object)`                      | 根据实体的主键删除记录                                    |
| `QueryFactory.StatementAsync(sql, params)` | 执行任意 SQL 语句                                         |

### 实现原理

* 查询构建：`SqlKata` 使用 `Query` 类表示 `SQL` 查询，通过链式方法构造查询树（`Query Tree`）。

* 编译器：`Compiler`（如 `SqlServerCompiler、PostgresCompiler`）将查询树转换为特定数据库的 `SQL` 语句，并生成参数化查询。

* 执行：`SqlKata.Execution` 包通过 `XQuery` 类和 `Dapper` 执行编译后的 `SQL`，映射结果到 `C#` 对象。

* 参数化：`SqlKata` 默认使用参数化查询，防止 `SQL` 注入。

### 性能与对比

* 与手写 `SQL`：

    * `SqlKata` 在拼装 `SQL` 和生成参数时有非常轻量的开销，实际运行时与 `Dapper` + 手写 `SQL` 相差极小。

    * 优势在于可读性与可维护性，更少的拼接错误和参数绑定麻烦。

* 与 `LINQ to SQL` / `EF Core`：

    * `LINQ` 在复杂联表、子查询场景下生成的 `SQL` 往往较冗长，且性能优化受限。

    * `SqlKata` 生成的 `SQL` 与手写几乎一致，开发者可更精细地控制索引使用和执行计划。

* 跨数据库兼容性：同一套查询构建逻辑，通过不同 `Compiler` 可无缝切换数据库，减少重复代码和维护成本。

### 实战项目

```csharp
using Microsoft.AspNetCore.Mvc;
using SqlKata;
using SqlKata.Execution;
using System.Data.SqlClient;
using System.Threading.Tasks;

public class Car
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Price { get; set; }
}

[ApiController]
[Route("api/cars")]
public class CarsController : ControllerBase
{
    private readonly QueryFactory _db;

    public CarsController(QueryFactory db)
    {
        _db = db;
    }

    [HttpGet]
    public async Task<IActionResult> Search([FromQuery] string searchText, [FromQuery] int? maxPrice)
    {
        var query = _db.Query("cars");

        if (!string.IsNullOrEmpty(searchText))
        {
            query.WhereLike("name", $"%{searchText}%");
        }

        if (maxPrice.HasValue)
        {
            query.Where("price", "<=", maxPrice.Value);
        }

        var cars = await query.GetAsync<Car>();
        return Ok(cars);
    }
}
```

启动配置：

```csharp
builder.Services.AddTransient<QueryFactory>(sp =>
{
    var connection = new MySqlConnection("Server=localhost;Database=master;User Id=root;Password=");
    var compiler = new MySqlCompiler();
    return new QueryFactory(connection, compiler);
});
```

### 资源和文档

* 官方文档：`https://sqlkata.com`

* `GitHub` 仓库：`https://github.com/sqlkata/querybuilder`

* `NuGet` 包：

    * `SqlKata`：`https://www.nuget.org/packages/SqlKata`

    * `SqlKata.Execution`：`https://www.nuget.org/packages/SqlKata.Execution`