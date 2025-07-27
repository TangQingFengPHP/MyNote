### 简介

`Dapper` 是由 `Stack Overflow` 团队开发的一个简单、高性能的微型 `ORM（Object‑Relational Mapper）`，仅几千行代码，依赖于 `ADO.NET` 的 `IDbConnection`，通过动态生成 `IL` 来映射结果到实体对象。

与 `EF、NHibernate` 这类全功能 `ORM` 相比，`Dapper` 提供了更直接、更接近 `SQL` 的操作方式，性能非常接近手写 `ADO.NET`。

### 基本用法

#### 安装与配置

```shell
dotnet add package Dapper
dotnet add package Dapper.Contrib
dotnet add package System.Data.SqlClient # SQL Server
```

#### 打开连接

```csharp
using (var conn = new SqlConnection(connectionString))
{
    conn.Open();
    // … 执行 Dapper 操作
}
```

#### 查询单个实体

```csharp
string sql = "SELECT Id, Name, Age FROM Users WHERE Id = @Id";
var user = conn.QueryFirstOrDefault<User>(sql, new { Id = 123 });
```

* `QueryFirstOrDefault<T>`：返回第一行并映射为 `T`，无数据时返回 `default(T)`。

* 还有 `QuerySingle<T>、Query<T>`（返回 `IEnumerable<T>`）等方法。

#### 查询多个对象

```csharp
using (var connection = new SqlConnection(connectionString))
{
    connection.Open();
    var users = connection.Query<User>(
        "SELECT * FROM Users WHERE Age > @Age", 
        new { Age = 18 });
    return users.ToList();
}
```

#### 执行增删改

```csharp
string insert = "INSERT INTO Users (Name, Age) VALUES (@Name, @Age)";
int rows = conn.Execute(insert, new { Name = "Alice", Age = 28 });
```

* `Execute`：用于执行不返回结果集的 `SQL`，返回受影响行数。

#### 基础 CRUD 操作

```csharp
using var connection = new SqlConnection(connectionString);

// 查询
var product = connection.QueryFirstOrDefault<Product>(
    "SELECT * FROM Products WHERE Id = @Id", 
    new { Id = 1 });

// 插入
var newProduct = new Product { Name = "Laptop", Price = 999.99m };
var insertedId = connection.ExecuteScalar<int>(
    "INSERT INTO Products (Name, Price) OUTPUT INSERTED.Id VALUES (@Name, @Price)",
    newProduct);

// 更新
var rowsAffected = connection.Execute(
    "UPDATE Products SET Price = @Price WHERE Id = @Id",
    new { Id = 1, Price = 899.99m });

// 删除
connection.Execute(
    "DELETE FROM Products WHERE Id = @Id", 
    new { Id = 10 });
```

**关键方法详解**

|  方法   |  描述   |  示例   |
| --- | --- | --- |
|  `Query<T>`   |  返回实体列表   |  `connection.Query<Product>("SELECT * FROM Products")`   |
|  `QueryFirst<T>`   |  返回第一条结果（无结果抛异常）   |  `connection.QueryFirst<Product>("SELECT ... WHERE Id=@id", new {id=1})`   |
|  `QueryFirstOrDefault<T>`   | 返回第一条或默认值（无结果返回 null）    |  `connection.QueryFirstOrDefault<Product>(...)`   |
|  `QuerySingle<T>`   |  返回单条结果（结果不唯一抛异常）   |  适用于主键查询   |
|  `Execute`   |  执行非查询操作（增删改），返回受影响行数   |  `connection.Execute("DELETE FROM Products WHERE Id=@id", new {id=10})`   |
|  `ExecuteScalar<T>`   |  返回单个值（如 COUNT、SUM）   |  `int count = connection.ExecuteScalar<int>("SELECT COUNT(*) FROM Products")`   |

### 高级查询技术

#### 多结果集处理

```csharp
using var multi = connection.QueryMultiple(
    "SELECT * FROM Products; SELECT * FROM Categories");

var products = multi.Read<Product>();
var categories = multi.Read<Category>();
```

#### 多表关联映射（Multi‑Mapping）

当一个查询返回多张表的数据，需要映射到不同实体并组合起来时：

```csharp
string sql = @"
SELECT o.Id, o.OrderDate,
       c.Id, c.Name 
FROM Orders o
JOIN Customers c ON o.CustomerId = c.Id";

var list = conn.Query<Order, Customer, Order>(
    sql,
    (order, cust) => { order.Customer = cust; return order; },
    splitOn: "Id");
```

* `splitOn` 指定从哪个列开始分割下一张表的映射。

#### 一对多关系映射

```csharp
var sql = @"
    SELECT o.*, i.* 
    FROM Orders o
    INNER JOIN OrderItems i ON o.Id = i.OrderId
    WHERE o.CustomerId = @CustomerId";

var orderDictionary = new Dictionary<int, Order>();

var orders = connection.Query<Order, OrderItem, Order>(
    sql,
    (order, item) =>
    {
        if (!orderDictionary.TryGetValue(order.Id, out var existingOrder))
        {
            existingOrder = order;
            existingOrder.Items = new List<OrderItem>();
            orderDictionary.Add(existingOrder.Id, existingOrder);
        }
        existingOrder.Items.Add(item);
        return existingOrder;
    },
    new { CustomerId = 123 },
    splitOn: "Id");
```

### Dapper.Contrib 扩展

#### 自动 CRUD 操作

```csharp
// 实体类标记
[Table("Products")]
public class Product
{
    [Key]
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// 自动操作
var product = connection.Get<Product>(1);          // 查询
connection.Insert(new Product { ... });           // 插入
connection.Update(product);                       // 更新
connection.Delete(product);                       // 删除

// 批量操作
connection.InsertAll(products);                   // 批量插入
```

#### 自定义映射规则

```csharp
SqlMapper.SetTypeMap(
    typeof(Product),
    new CustomPropertyTypeMap(
        typeof(Product),
        (type, columnName) => 
            type.GetProperties().FirstOrDefault(prop => 
                prop.GetCustomAttributes(false)
                    .OfType<ColumnAttribute>()
                    .Any(attr => attr.Name == columnName))));
```

### 性能优化技巧

#### 参数化查询最佳实践

```csharp
// 正确：参数化防止SQL注入
connection.Query("SELECT * FROM Users WHERE Name = @Name", new { Name = "Alice" });

// 错误：拼接SQL风险
connection.Query($"SELECT * FROM Users WHERE Name = '{"Alice"}'");
```

#### 批处理与事务

```csharp
using var transaction = connection.BeginTransaction();

try
{
    // 批量插入
    connection.Execute(
        "INSERT INTO Logs (Message) VALUES (@Message)",
        logs.Select(l => new { l.Message }),
        transaction);
    
    // 批量更新
    connection.Execute(
        "UPDATE Products SET Stock = Stock - @Quantity WHERE Id = @Id",
        orderItems.Select(i => new { i.Quantity, i.ProductId }),
        transaction);
    
    transaction.Commit();
}
catch
{
    transaction.Rollback();
    throw;
}
```

#### 异步操作支持

```csharp
public async Task<Product> GetProductAsync(int id)
{
    using var conn = new SqlConnection(connectionString);
    return await conn.QueryFirstOrDefaultAsync<Product>(
        "SELECT * FROM Products WHERE Id = @Id", 
        new { Id = id });
}
```

### 高级应用场景

#### 动态查询构建

```csharp
public IEnumerable<Product> SearchProducts(ProductSearchCriteria criteria)
{
    var sql = new StringBuilder("SELECT * FROM Products WHERE 1=1");
    var parameters = new DynamicParameters();
    
    if (!string.IsNullOrEmpty(criteria.Name))
    {
        sql.Append(" AND Name LIKE @Name");
        parameters.Add("Name", $"%{criteria.Name}%");
    }
    
    if (criteria.MinPrice.HasValue)
    {
        sql.Append(" AND Price >= @MinPrice");
        parameters.Add("MinPrice", criteria.MinPrice);
    }
    
    return connection.Query<Product>(sql.ToString(), parameters);
}
```

#### JSON 数据处理（SQL Server 2016+）

```csharp
// 查询JSON列
var orders = connection.Query<Order>(@"
    SELECT 
        Id,
        JSON_VALUE(Details, '$.Customer.Name') AS CustomerName,
        JSON_VALUE(Details, '$.TotalAmount') AS TotalAmount
    FROM Orders");

// 插入JSON数据
connection.Execute(
    "INSERT INTO Orders (Details) VALUES (@Details)",
    new { Details = JsonConvert.SerializeObject(orderDetails) });
```

### 参数化查询

`Dapper` 默认将匿名对象的属性映射到 `SQL` 中的参数，杜绝 `SQL` 注入：

```csharp
var parameters = new DynamicParameters();
parameters.Add("MinPrice", 100);
parameters.Add("MaxPrice", 500);
var products = conn.Query<Product>(
    "SELECT * FROM Products WHERE Price BETWEEN @MinPrice AND @MaxPrice",
    parameters);
```

* `DynamicParameters` 支持输出参数、存储过程参数等高级场景。

### 最佳实践指南

#### 与 EF Core 混合使用

```csharp
// 复杂查询用 Dapper
public List<ReportItem> GetSalesReport(DateTime start, DateTime end)
{
    return _dapperConnection.Query<ReportItem>(...);
}

// 事务性操作用 EF Core
public void PlaceOrder(Order order)
{
    using var transaction = _dbContext.Database.BeginTransaction();
    try
    {
        _dbContext.Orders.Add(order);
        _dbContext.SaveChanges();
        
        // 调用库存服务（Dapper）
        _dapperConnection.Execute(...);
        
        transaction.Commit();
    }
    catch
    {
        transaction.Rollback();
        throw;
    }
}
```

#### 性能关键路径优化

```csharp
// 缓存查询结果（高频读取）
private static readonly string ProductByIdSql = 
    "SELECT * FROM Products WHERE Id = @Id";

public Product GetProduct(int id)
{
    return _connection.QueryFirstOrDefault<Product>(
        ProductByIdSql, new { Id = id });
}

// 使用存储过程
var products = _connection.Query<Product>(
    "usp_GetProductsByCategory", 
    new { CategoryId = 5 }, 
    commandType: CommandType.StoredProcedure);
```

### Dapper 扩展生态

#### 常用扩展库

|  库名   |  功能   |  安装命令   |
| --- | --- | --- |
|  Dapper.Contrib   |  基础CRUD扩展   |  `Install-Package Dapper.Contrib`   |
|  Dapper.FluentMap   |  高级映射配置   |  `Install-Package Dapper.FluentMap`   |
|  Dapper.SimpleCRUD   |  自动化CRUD操作   |  `Install-Package Dapper.SimpleCRUD`   |
|  Dapper.SqlBuilder   |  动态SQL构建   |  `Install-Package Dapper.SqlBuilder`   |

#### 数据库提供程序支持

```csharp
// PostgreSQL
using var conn = new NpgsqlConnection(pgConnectionString);

// MySQL
using var conn = new MySqlConnection(mysqlConnectionString);

// SQLite
using var conn = new SQLiteConnection(sqlliteConnectionString);
```

### Dapper 适用场景分析

**推荐使用场景**

* 高性能报表查询：大数据量复杂查询

* 批量数据处理：`ETL` 管道操作

* 微服务架构：轻量级数据访问层

* 遗留系统集成：替代原生 `ADO.NET`

* 读写分离架构：读操作专用

**不推荐场景**

* 复杂领域模型：需要变更跟踪

* 数据库迁移需求：缺乏迁移工具

* 自动 `SQL` 生成：需手写 `SQL`

* 简单 `CRUD` 应用：`EF Core` 更高效

**总结：Dapper 核心价值**

* 极致性能：接近原生 `ADO.NET` 的执行效率

* 精细控制：完全掌控 `SQL` 语句

* 轻量灵活：最小化 `ORM` 开销

* 易于集成：与现有项目无缝结合

* 丰富扩展：强大的社区生态支持

### 存储过程调用

```csharp
var users = conn.Query<User>(
    "sp_GetActiveUsers",
    new { IsActive = true },
    commandType: CommandType.StoredProcedure);
```

* 也可用 `Execute、QueryMultiple`（一次取多组结果）等。

### 性能优势

* 由于 `Dapper` 直接在运行时生成 `IL` 来做对象映射，并且复用 `ADO.NET` 连接与命令，对大量并发、批量查询场景性能非常优秀。

* 对比 `EF Core`，`Dapper` 的查询速度常常有 2–10 倍优势，尤其是返回扁平对象时。

### 与 EF 的对比

|          | Dapper                     | EF Core                                |
| -------- | -------------------------- | -------------------------------------- |
| 编写 SQL | 手写、完全自定义           | LINQ 语句                              |
| 配置映射 | 自动映射或手动配置（少量） | 属性与 Fluent API                      |
| 性能     | 极高，接近原生 ADO.NET     | 较高，但 LINQ 翻译与变更追踪有额外开销 |
| 灵活度   | 完全掌控 SQL               | 高，受限于 LINQ 表达式能力             |
| 学习成本 | 低                         | 相对高（迁移、关系、导航属性等）       |

### 常见问题与最佳实践

* 连接管理：建议每次操作单独 `using` 打开连接，避免长连接导致资源占用。

* 参数重用：对于频繁相同参数的查询，可重用 `DynamicParameters` 对象，提升性能。

* 大型结果集：如果结果集很大，使用 `Query<T>` 时可结合 `AsList()` 分批处理，或使用 `IDataReader` 手动读取。

* 分页：可在 `SQL` 中直接使用 `OFFSET…FETCH`，或结合动态参数快速实现。

* 缓存：`Dapper` 自带简单的 `SQL` 缓存（`IL` 生成后会缓存映射），但不缓存查询结果；如需缓存结果，可加分布式/内存缓存。