### 简介

* `LinqToDB` 是一个轻量、高性能的 `ORM`/微型 `DAL`，核心特点是对 `SQL` 的最小封装，保留 `LINQ` 的表达式能力，同时最大化控制底层 `SQL`，减少运行时开销。

* 与 `Dapper、FreeSql` 等对比：

    * `Dapper`：低级别映射，`SQL` 全手写；

    * `FreeSql`：功能丰富，自动结构同步；

    * `LinqToDB`：`LINQ→SQL` 转换最优、支持静态编译查询、拥有强大的 `BulkCopy`、跨数据库迁移脚本等工具。

**LinqToDB 的设计理念**

* 类型安全：使用 `LINQ` 表达式，避免字符串拼接的 `SQL` 查询，编译器可检查查询语法。

* 高性能：接近 `Dapper` 的性能，优于 `Entity Framework Core`，减少不必要的开销。

* 轻量级：无复杂的功能（如变更跟踪），提供薄的抽象层，开发者可直接控制 `SQL`。

* 灵活性：支持 `LINQ`、原始 `SQL`、批量操作、事务和数据库特定功能。

### LinqToDB 的核心特性

#### 类型安全的 LINQ 查询：

* 使用 `LINQ` 表达式查询数据库，编译器可验证查询语法，减少运行时错误。

* 支持复杂的查询（如联接、分组、子查询）。

#### 多数据库支持：

* 支持主流数据库：SQL Server、MySQL、PostgreSQL、SQLite、Oracle、Access、Firebird、Informix、DB2 等。

* 提供数据库特定的扩展（如 SQL Server 的 MERGE、PostgreSQL 的 FOR UPDATE）。

#### 高性能：

* 接近 `ADO.NET` 和 `Dapper` 的性能，优于 `Entity Framework Core`。

* 支持批量插入（`Bulk Copy`）、异步查询和高效的对象映射。

#### 灵活的映射：

* 支持特性映射（如 `[Table]、[Column]`）和 `Fluent` 映射。

* 支持动态表名、跨数据库查询和自定义映射。

#### 无变更跟踪：

* 不像 `Entity Framework Core，LinqToDB` 不跟踪对象状态，开发者需手动管理更新。

* 优点是更低的开销和更高的控制权。

#### 扩展性：

* 支持自定义 `SQL` 提示（`Hints`）、拦截器和数据库特定功能。

* 集成 `ASP.NET Core` 依赖注入和 `Entity Framework Core`。

#### 其他功能：

* 支持事务、窗口函数（`Window Functions`）、`MERGE` 语句。

* 提供 `CLI` 工具和 `T4` 模板用于数据库模型生成。

### 安装与基础配置

#### 安装 NuGet 包

```shell
dotnet add package linq2db
dotnet add package linq2db.SqlServer    # 或 MySql/PostgreSQL/Oracle 等具体 Provider
```

#### 连接字符串与上下文

```csharp
// 继承 DataConnection，或直接使用 DataConnection
public class AppDb : LinqToDB.Data.DataConnection
{
    public AppDb() : base("MyConnectionStringName") { }

    public ITable<User> Users => GetTable<User>();
    public ITable<Order> Orders => GetTable<Order>();
}
```

`"MyConnectionStringName"` 对应 `appsettings.json` 或 `ConfigurationManager.ConnectionStrings` 中的连接串。

#### 继承 DataContext 类封装连接

```csharp
public class AppDbContext : DataContext
{
    public AppDbContext() : base(ProviderName.SQLite, "Data Source=app.db") { }

    public ITable<Product> Products => GetTable<Product>();
}
```

* `DataConnection`：保持连接直到 `Dispose`，适合需要长时间保持连接的场景。

* `DataContext`：每次查询打开和关闭连接，类似 `Entity Framework`。

#### 注册到 DI（.NET Core/.NET 6+）

```csharp
builder.Services
  .AddLinqToDBContext<AppDb>((provider, options) =>
      options
        .UseConnectionString(ProviderName.SqlServer, configuration.GetConnectionString("Default"))
        .UseDefaultLogging(provider.GetService<ILoggerFactory>())
  );
```

### 核心 CRUD 与 LINQ 查询

#### 查询

```csharp
using var db = new AppDb();
// 简单查询
var adults = db.Users
    .Where(u => u.Age >= 18)
    .OrderBy(u => u.Name)
    .ToList();

// LINQ 查询
var query = from p in db.Products
    where p.Price > 15.00m
    orderby p.Name descending
    select p;
var result = query.ToList();

// 关联查询
var orders = db.Employees
    .Where(e => e.EmployeeID == 1)
    .SelectMany(e => e.Orders)
    .Where(o => o.OrderDate > DateTime.Today.AddYears(-1))
    .ToList();


// 投影查询
var names = db.Users
    .Where(u => u.IsActive)
    .Select(u => new { u.Id, u.Name })
    .ToList();
```

#### 插入/更新/删除

```csharp
// 插入单条并返回自增 Id
var newUser = new User { Name = "Alice", Age = 28 };
int id = db.InsertWithInt32Identity(newUser);

// 插入
db.Insert(new Product { Name = "Laptop", Price = 1200 });

// 批量插入
db.BulkCopy(new BulkCopyOptions(), listOfUsers);

// 更新
int upd = db.Users
    .Where(u => u.Id == id)
    .Set(u => u.Age, 30)
    .Update();

// 删除
int del = db.Users
    .Where(u => u.Age < 18)
    .Delete();
```

### 实体映射与模型配置

#### 属性映射

```csharp
// 方式1：使用特性
[Table(Name = "Users")]
public class User
{
    [PrimaryKey, Identity]
    public int Id { get; set; }
    
    [Column(Name = "UserName"), NotNull]
    public string Name { get; set; }
    
    [Column(IsColumn = false)] // 不映射到数据库
    public string FullName => $"{LastName}, {FirstName}";
}
```

#### Fluent API 配置

```csharp
// 方式2：使用 Fluent API
public class MyMappingSchema : MappingSchema
{
    public MyMappingSchema() : base("MySchema")
    {
        var builder = GetFluentMappingBuilder();
        
        builder.Entity<User>()
            .HasTableName("Users")
            .HasPrimaryKey(u => u.Id)
            .Property(u => u.Id).IsIdentity()
            .Property(u => u.Name).HasColumnName("UserName");
    }
}
```

#### 多库与多模式支持

* 在 `UseConnectionString` 中指定不同的 `Provider`：

```csharp
options.UseConnectionString(ProviderName.PostgreSQL, pgConn)
       .UseConnectionString(ProviderName.MySql, myConn);
```

* 支持在同一上下文中查询不同数据库，只要映射定义一致。

### 进阶特性

#### 多数据库连接

通过不同连接字符串名区分多个数据库

```csharp
using (var db1 = new DataContext("DB1_Config"))
using (var db2 = new DataContext("DB2_Config")) 
{ /* 操作多个库 */ }
```

#### 编译查询（Compiled Queries）

对性能要求极高场景，可预编译 `LINQ→SQL` 转换：

```csharp
static readonly Func<AppDb, int, List<User>> GetByMinAge =
    CompiledQuery.Compile((AppDb db, int minAge) =>
        db.Users.Where(u => u.Age >= minAge).ToList());

var adults = GetByMinAge(db, 18);
```

* 减少表达式树解析开销，适合高并发和热路径。

#### BulkCopy 与批量操作

* `BulkCopy`：对比 `InsertWithIdentity`，批量插入数万条性能提升数十倍。

* `Merge`：基于 `SQL Server` 的 `MERGE` 语法，实现 `UPSERT`：

```csharp
db.BulkCopy(new BulkCopyOptions { BulkCopyType = BulkCopyType.Merge }, list);
```

#### 事务处理

```csharp
using (var db = new MyDataContext())
using (var transaction = db.BeginTransaction())
{
    try
    {
        // 操作1
        db.Insert(new User { Name = "Transaction User", Age = 30 });
        
        // 操作2
        db.Users
            .Where(u => u.Name == "Old User")
            .Set(u => u.Age, u => u.Age + 1)
            .Update();
        
        // 提交事务
        transaction.Commit();
    }
    catch (Exception)
    {
        // 回滚事务
        transaction.Rollback();
        throw;
    }
}
```

#### 批量操作

```csharp
using (var db = new MyDataContext())
{
    // 批量插入
    var users = new List<User>
    {
        new User { Name = "User1", Age = 20 },
        new User { Name = "User2", Age = 21 },
        new User { Name = "User3", Age = 22 }
    };
    
    db.BulkCopy(users);
    
    // 批量更新
    var usersToUpdate = db.Users
        .Where(u => u.Age < 25)
        .ToList();
    
    foreach (var user in usersToUpdate)
    {
        user.Age += 1;
    }
    
    db.BulkUpdate(usersToUpdate);
}
```

#### 异步操作

```csharp
using (var db = new MyDataContext())
{
    // 异步查询
    var users = await db.Users
        .Where(u => u.Age > 18)
        .ToListAsync();
    
    // 异步插入
    var newUser = new User { Name = "Async User", Age = 30 };
    await db.InsertAsync(newUser);
}
```

#### 复杂查询支持

* `join` 查询

```csharp
using (var db = new DbNorthwind())
{
    var query = from p in db.Products
                join c in db.Categories on p.CategoryID equals c.CategoryID
                select new { ProductName = p.Name, CategoryName = c.Name };
    var result = query.ToList();
}
```

* 子查询：

```csharp
using (var db = new DbNorthwind())
{
    var subQuery = db.Products.Where(p => p.Price > 20.00m).Select(p => p.CategoryID);
    var query = db.Categories.Where(c => subQuery.Contains(c.CategoryID));
    var result = query.ToList();
}
```

* 分页：

```csharp
using (var db = new DbNorthwind())
{
    var query = db.Products.OrderBy(p => p.Name).Skip(10).Take(5);
    var result = query.ToList();
}
```

* 联合查询（`Union`）

```csharp
var query1 = db.Products.Where(p => p.Price < 10);
var query2 = db.Products.Where(p => p.Price > 100);
var unionResults = query1.Union(query2).ToList();
```

* 关联查询：通过 `Association` 属性定义导航关系

```csharp
[Association(ThisKey = "DepartmentID", OtherKey = "DepartmentID")]
public Department Department { get; set; }
```

#### 存储过程调用

```csharp
[StoredProcedure("CustOrderHist")]
public class CustOrderHistResult
{
    [Column("ProductName")]
    public string ProductName { get; set; }
    
    [Column("Total")]
    public int Total { get; set; }
}

using (var db = new NorthwindDataConnection())
{
    var parameters = new DataParameter[]
    {
        new DataParameter("@CustomerID", "ALFKI")
    };
    
    var results = db.QueryProc<CustOrderHistResult>("CustOrderHist", parameters);
}
```

#### 窗口函数支持

```csharp
var rankedEmployees = db.Employees
    .Select(e => new
    {
        e.LastName,
        e.FirstName,
        RowNumber = Sql.Ext.RowNumber()
            .Over()
            .PartitionBy(e.Title)
            .OrderBy(e.HireDate)
            .ToValue()
    })
    .ToList();
```

#### SQL 拦截与日志

```csharp
db.OnTrace = new TraceInfo
{
    BeforeExecute = info => Console.WriteLine(info.SqlText),
    AfterExecute  = info => Console.WriteLine($"Count:{info.RecordCount}")
};
```

* 可在拦截器中修改 `SQL`、参数，或将日志写入自定义管道。

### 性能优化与最佳实践

#### 重用 DataConnection

* 在同一请求/作用域内复用，不要频繁 `new`；

* 配合 `DI` 容器管理生命周期。

#### 使用编译查询

* 热点路径、复杂查询强烈推荐，能显著降低 `CPU` 开销。

#### Bulk 操作

* 对大批量数据，务必使用 `BulkCopy` 或 `Merge`，避免逐条插入。

#### 显式投影

* 不要在 `LINQ` 中拉取整个实体，尽量 `Select` 出需要字段，减少网络与内存压力。

#### 控制查询深度

* 避免不必要的 `Join`；对大对象图，分步查询或分割 `DTO`。

#### 日志与监控

* 在开发/测试环境打开 `OnTrace`，生产环境可降级为 `WARN` 级别或写文件。

#### 事务粒度

* 控制在最小范围内使用事务，长事务会锁表、阻塞并发。

#### 避免 N+1 查询：使用 LoadWith 预加载关联数据

```csharp
var employees = db.Employees
    .LoadWith(e => e.Orders)
    .ThenLoad(o => o.OrderDetails)
    .ToList();
```

### 资源与扩展

* 官方文档：https://linq2db.github.io/

* `GitHub` 仓库：https://github.com/linq2db/linq2db

* 示例项目：https://github.com/linq2db/examples

* 扩展包

    * `linq2db.EntityFrameworkCore`：在 `EF Core` 项目中使用 `LinqToDB` 功能。

    * `LinqToDB.Identity`：`ASP.NET Core Identity` 提供程序

* `CLI` 工具：支持数据库模型生成和 `T4` 模板定制：https://linq2db.github.io/articles/CLI.html

### 总结

`LinqToDB` 是 `.NET` 生态中一个高性能、轻量级的 `ORM` 解决方案，特别适合：

* 需要接近原生数据库性能的应用

* 处理复杂查询和大数据量的场景

* 对启动时间和内存占用敏感的环境

* 需要与多种数据库兼容的项目

**核心优势：**

* 极佳的性能表现

* 简洁轻量的设计

* 强大的 `SQL` 生成能力

* 全面的数据库支持

**推荐使用场景：**

* 报表生成系统

* 数据分析和 `ETL` 处理

* 高性能 `API` 后端

* 数据库迁移工具