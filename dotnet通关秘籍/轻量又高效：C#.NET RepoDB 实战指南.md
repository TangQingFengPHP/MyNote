### 简介

* `RepoDB` 是一个“混合” `ORM（Hybrid ORM）`，旨在弥合微型 `ORM`（如 `Dapper`）和全功能 `ORM`（如 `EF Core`）之间的鸿沟，既提供对 `SQL` 的直接控制，又封装了常用的高级操作

#### 核心特性

* 混合 `ORM` 功能

    * 支持微型 `ORM` 的原始 `SQL` 查询（`ExecuteQuery<T>`）和 `Fluent API（QueryAll<T>）`。

    * 提供完整 `ORM` 的高级功能，如批量操作（`BulkInsert`）、缓存和属性处理器。

* 高性能

    * 使用编译表达式缓存（`Compiled Expression Caching`）减少运行时开销。

    * 批量和批量插入（`BulkInsert`）利用 `ADO.NET` 的 `SqlBulkCopy` 等技术，性能优异。

    * 官方基准测试表明，`RepoDB` 在单记录和批量操作中性能优于 `Dapper` 和 `Entity Framework Core`。

* 内置仓储支持

    * 提供 `BaseRepository<T, TDbConnection>` 基类，简化仓储模式实现。

    * 支持自定义仓储，灵活性高。

* 第二层缓存（`2nd-Layer Cache`）

    * 支持内存缓存（默认 180 分钟），可自定义缓存实现。*

    * 显著减少数据库查询，提升性能（可达 90% 以上）。

* 数据库支持

    * 支持 `SQL Server、MySQL、PostgreSQL、SQLite` 等主流数据库。

    * 通过扩展包（如 `RepoDb.SqlServer`）支持特定数据库功能。

* `Fluent API` 和原始 `SQL`

    * `Fluent API` 提供强类型操作（如 `Insert<T>、Query<T>`）。

    * 原始 `SQL` 支持复杂查询，结合参数化查询防止 `SQL` 注入。

* 批量和批量操作

    * 支持 `InsertAll、UpdateAll、DeleteAll`（批量操作）和 `BulkInsert、BulkMerge`（高效批量操作）。

    * 自动处理自增列值回写。

* 扩展性

    * 支持属性处理器（`Property Handlers`）自定义映射逻辑。

    * 提供跟踪（`Tracing`）和查询提示（`Hints`）优化数据库操作。

* 开源和社区支持

    * `Apache-2.0` 许可证，免费开源。

    * `GitHub` 仓库（https://github.com/mikependon/RepoDB）提供文档和支持。

### 安装与配置

#### 安装 NuGet 包

```shell
# 核心
Install-Package RepoDb
# SQL Server 提供程序
Install-Package RepoDb.SqlServer   
# MySQL 提供程序
Install-Package RepoDb.MySql  
# MySqlConnector 
Install-Package RepoDb.MySqlConnector
```

#### 初始化依赖

**SqlServer**

* 最新版本

```csharp
GlobalConfiguration
	.Setup()
	.UseSqlServer();
```

* 1.13.0 版本之前

```csharp
RepoDb.SqlServerBootstrap.Initialize();
```

**MySql**

* 最新版本

```csharp
GlobalConfiguration
	.Setup()
	.UseMySql();

GlobalConfiguration
	.Setup()
	.UseMySqlConnector();
```

* 1.13.0 版本之前

```csharp
RepoDb.MySqlBootstrap.Initialize();

RepoDb.MySqlConnectorBootstrap.Initialize();
```

#### 标准用法

```csharp
using (var connection = new SqlConnection(connectionString))
{
    connection.Open();
    // 之后可直接调用 RepoDB 的扩展方法
}
```

#### 仓储模式

```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
}

public class UserDto
{
    public int Id { get; set; }
    public string FullName { get; set; }
}

public class UserRepository : BaseRepository<User, SqlConnection>
{
    public UserRepository(string connectionString)
        : base(connectionString)
    {
    }
}

var builder = WebApplication.CreateBuilder(args);

// 初始化 RepoDB
SqlServerBootstrap.Initialize();

// 配置 Mapster
TypeAdapterConfig<User, UserDto>
    .NewConfig()
    .Map(dest => dest.FullName, src => src.Name);
builder.Services.AddMapster();

// 配置仓储
builder.Services.AddScoped(_ => new UserRepository("Server=(local);Database=TestDb;Integrated Security=True;"));

var app = builder.Build();

app.MapGet("/users", async (UserRepository repo) =>
{
    var users = await repo.QueryAllAsync();
    var userDtos = users.Adapt<List<UserDto>>();
    return Results.Ok(userDtos);
});

app.Run();
```

#### 实体映射

```csharp
// 自动映射（表名 = 类名复数，列名 = 属性名）
[Map("Products")]
public class Product
{
    public int Id { get; set; }
    
    [Map("ProductName")]
    public string Name { get; set; }
    
    public decimal Price { get; set; }
    
    [Map(ignore: true)] // 忽略映射
    public string TempData { get; set; }
}
```

### 核心 CRUD 操作

所有操作均基于对 `IDbConnection` 的扩展方法。

#### 插入

```csharp
// 单条插入，返回自增 Id
var person = new Person { Name = "John", Age = 54, CreatedDateUtc = DateTime.UtcNow };
var id = connection.Insert(person);

// 多条插入
var people = GetPeople(100);
var rows = connection.InsertAll(people);
```

#### 查询

```csharp
public static void Main()
{
    SqlServerBootstrap.Initialize();
    var connectionString = "Server=(local);Database=TestDb;Integrated Security=True;";

    using var connection = new SqlConnection(connectionString);

    // Fluent API 查询
    var users = connection.QueryAll<User>();
    foreach (var user in users)
    {
        Console.WriteLine($"Id: {user.Id}, Name: {user.Name}");
    }

    // 原始 SQL 查询
    var sqlUsers = connection.ExecuteQuery<User>("SELECT * FROM Users WHERE Age > @Age", new { Age = 18 });
    foreach (var user in sqlUsers)
    {
        Console.WriteLine($"Id: {user.Id}, Name: {user.Name}");
    }
}
```

```csharp
// 按主键查询单条
var person = connection.Query<Person>(id).FirstOrDefault();

// 带表达式过滤
var adults = connection.Query<Person>(e => e.Age >= 18);

// 查询所有
var all = connection.QueryAll<Person>();
```

#### 更新

```csharp
// 根据实体主键更新所有可写属性
person.Name = "James";
var updated = connection.Update(person);

// 批量更新
people.ForEach(p => p.Age += 1);
var updRows = connection.UpdateAll(people);
```

#### 删除

```csharp
// 按主键删除
var del = connection.Delete<Person>(10045);

// 删除所有
var delAll = connection.DeleteAll<Person>();
```

### 高级查询

#### 多表查询（JOIN）

```csharp
using (var connection = new SqlConnection(connectionString))
{
    var result = connection.QueryMultiple<Product, Category>(
        p => p.CategoryId,          // 产品外键
        c => c.Id,                   // 类别主键
        (product, category) =>       // 结果映射
        {
            product.Category = category;
            return product;
        },
        where: p => p.Price > 500    // 筛选条件
    );
}
```

#### 分页查询

```csharp
using (var connection = new SqlConnection(connectionString))
{
    // 排序 + 分页
    var orderBy = OrderField.Parse(new { Price = Order.Descending });
    var result = connection.Query<Product>(
        where: p => p.Price > 100,
        orderBy: orderBy,
        page: 2,        // 第2页
        rowsPerBatch: 10 // 每页10条
    );
    
    // 分页对象
    var page = new Page(2, 10); // 第2页，每页10条
    var pagedResult = connection.Query<Product>(page: page);
}
```

#### 存储过程调用

```csharp
using (var connection = new SqlConnection(connectionString))
{
    var parameters = new { CategoryId = 5, MinPrice = 100.00m };
    var result = connection.ExecuteQuery<Product>(
        "usp_GetProductsByCategory", 
        parameters, 
        commandType: CommandType.StoredProcedure);
}
```

#### 动态查询

```csharp
using (var connection = new SqlConnection(connectionString))
{
    // 动态条件
    var where = new { CategoryId = 3, Price = (decimal?)100.00m };
    var products = connection.Query<Product>(where: where);
    
    // 动态返回类型
    var result = connection.ExecuteQuery("SELECT * FROM Products WHERE CategoryId = @CategoryId", 
        new { CategoryId = 3 });
}
```

### 批量操作与性能优化

#### 批量插入（BulkInsert）

```csharp
using (var connection = new SqlConnection(connectionString))
{
    var products = GenerateProducts(10000); // 生成10000个产品
    var insertedCount = connection.BulkInsert(products);
}
```

#### 批量合并（BulkMerge）

```csharp
using (var connection = new SqlConnection(connectionString))
{
    var products = GetChangedProducts(); // 获取变更的产品（新增+更新）
    var mergedCount = connection.BulkMerge(products);
}
```

#### 二级缓存

```csharp
using (var connection = new SqlConnection(connectionString))
{
    // 查询结果缓存30分钟
    var products = connection.Query<Product>(
        cacheKey: "expensive_products", 
        cacheItemExpiration: TimeSpan.FromMinutes(30),
        where: p => p.Price > 1000);
    
    // 清除缓存
    CacheFactory.Flush();
}
```

### 事务管理

#### 基本事务


```csharp
using (var connection = new SqlConnection(connectionString))
{
    using (var transaction = connection.EnsureOpen().BeginTransaction())
    {
        try
        {
            // 减少库存
            connection.DecreaseStock(productId, quantity, transaction: transaction);
            
            // 创建订单
            connection.Insert(new Order { /* ... */ }, transaction: transaction);
            
            transaction.Commit();
        }
        catch
        {
            transaction.Rollback();
            throw;
        }
    }
}
```

#### 分布式事务

```csharp
using (var scope = new TransactionScope())
{
    using (var connection1 = new SqlConnection(connString1))
    using (var connection2 = new SqlConnection(connString2))
    {
        // 数据库1操作
        connection1.Insert(entity1);
        
        // 数据库2操作
        connection2.Insert(entity2);
    }
    
    scope.Complete();
}
```

### 扩展功能

#### 数据跟踪（变更追踪）

```csharp
// 启用变更追踪
var product = connection.Query<Product>(1).First();
product.Price = 899.99m;

// 获取变更
var changes = PropertyCache.Get<Product>().GetChanges(product);
// changes = { { "Price", (原值, 899.99m) } }

// 仅更新变更字段
connection.Update<Product>(product, changes);
```

#### 字段映射扩展

```csharp
// 自定义类型转换
TypeMapper.Map(typeof(ProductStatus), DbType.Int32, true);

// 查询时转换
var products = connection.Query<Product>(p => p.Status == ProductStatus.Active);

// 自定义值转换器
public class ProductStatusConverter : IPropertyHandler<string, ProductStatus>
{
    public ProductStatus Get(string input, PropertyHandlerGetOptions options) => 
        Enum.Parse<ProductStatus>(input);
    
    public string Set(ProductStatus input, PropertyHandlerSetOptions options) => 
        input.ToString();
}
```

#### 事件钩子

```csharp
// 全局事件
GlobalConfiguration
    .BeforeInsert += (sender, args) => 
        args.Entity.CreatedDate = DateTime.UtcNow;
    
    .AfterQuery += (sender, args) => 
        Logger.LogDebug($"Query executed: {args.Statement}");
```

#### 动态类型与原生表名操作

除了强类型实体，`RepoDB` 还支持 `Dynamics`，可对任意表执行 `CRUD`：

```csharp
// 动态查询
var dyn = connection.Query(
    tableName: "[dbo].[Customer]",
    param: new { FirstName = "John", LastName = "Doe" }
).FirstOrDefault();

// 指定列
var cust = connection.Query(
    "[dbo].[Customer]",
    10045,
    Field.From("Id","FirstName","LastName")
).FirstOrDefault();
```

#### 映射配置与属性处理器

* 默认按 属性名 ↔ 列名 进行映射，支持大小写与下划线风格自动识别。

* 可通过 `FluentMapper` 灵活定义表名、列名、主键、类型映射及自定义属性处理器

```csharp
FluentMapper
  .Entity<Customer>()
  .Table("[sales].[Customer]")
  .Primary(e => e.Id)
  .Identity(e => e.Id)
  .Column(e => e.FirstName, "[FName]")
  .PropertyHandler<AddressPropertyHandler>(e => e.Address);
```

#### 异步操作

```csharp
await connection.InsertAsync(person);
var list = await connection.QueryAsync<Person>(e => e.Age > 18);
```

### 跟踪（Tracing）与日志

* 通过全局或实例级别的事件，捕获并记录 `SQL` 语句与参数：

```csharp
connection.OnTraceExecuting = (command, trace) =>
{
    Console.WriteLine($"SQL: {trace.CommandText}");
    Console.WriteLine($"Param: {string.Join(", ", trace.ParameterValues)}");
};
```

### 最佳实践

#### 初始化 RepoDB

* 在应用程序启动时调用 `SqlServerBootstrap.Initialize()` 或其他数据库初始化方法

```csharp
SqlServerBootstrap.Initialize();
```

#### 使用仓储模式

* 继承 `BaseRepository` 实现标准化的数据访问层

```csharp
public class ProductRepository : BaseRepository<Product, SqlConnection>
{
    public ProductRepository(string connectionString) 
        : base(connectionString) { }
    
    public IEnumerable<Product> GetExpensiveProducts(decimal minPrice)
    {
        return Query(p => p.Price >= minPrice);
    }
    
    public int BulkUpdatePrices(decimal multiplier)
    {
        var products = QueryAll();
        products.ForEach(p => p.Price *= multiplier);
        return UpdateAll(products);
    }
}
```

#### 参数化查询

* 使用匿名对象或字典防止 `SQL` 注入

```csharp
connection.ExecuteQuery<User>("SELECT * FROM Users WHERE Age > @Age", new { Age = 18 });
```

#### 工作单元模式

```csharp
public class UnitOfWork : IDisposable
{
    private readonly SqlConnection _connection;
    private SqlTransaction _transaction;
    
    public UnitOfWork(string connectionString)
    {
        _connection = new SqlConnection(connectionString);
        _connection.Open();
        _transaction = _connection.BeginTransaction();
    }
    
    public void Commit()
    {
        _transaction.Commit();
        Dispose();
    }
    
    public void Rollback()
    {
        _transaction.Rollback();
        Dispose();
    }
    
    public void Dispose()
    {
        _transaction?.Dispose();
        _connection?.Dispose();
    }
    
    // 仓储访问器
    public ProductRepository Products => 
        new ProductRepository(_connection, _transaction);
}
```

#### 性能关键路径优化

```csharp
// 1. 使用字段选择器（避免 SELECT *）
var products = connection.Query<Product>(
    select: e => new { e.Id, e.Name }); // 只查询需要的字段

// 2. 启用批处理
GlobalConfiguration.Setup().UseSqlServer().SetBatchSize(1000);

// 3. 异步操作
public async Task<IEnumerable<Product>> GetProductsAsync()
{
    using (var connection = new SqlConnection(connectionString))
    {
        return await connection.QueryAllAsync<Product>();
    }
}

// 4. 缓存常用查询
public class ProductCache
{
    private static IEnumerable<Product> _products;
    
    public static IEnumerable<Product> GetAll()
    {
        return _products ??= 
            new SqlConnection(connectionString)
                .QueryAll<Product>(cacheKey: "all_products", 
                    cacheItemExpiration: TimeSpan.FromHours(1));
    }
}
```

* 短生命的 `IDbConnection` 实例：每次操作打开/关闭连接或结合 `DI` 容器按作用域管理。

* 参数化方式：始终使用 `RepoDB` 的扩展方法代替手写拼接，防止 `SQL` 注入并优化执行计划。

* 分级缓存：对热点查询开启二级缓存，写操作后记得手动清除相关缓存。

* 合理分批：海量数据时优先使用 `BulkInsert/BulkMerge`，避免一次性操作过大导致 `OOM`。

* 映射校验：在单元测试中，使用 `FluentMapper.Validate()` 确保映射配置完整且无冲突。

* 监控与告警：结合 `Trace` 事件和 `APM` 工具，监控慢 `SQL` 与批量操作时长。

### 常见问题解决方案

#### 复杂类型映射

```csharp
public class Product
{
    // JSON列映射
    [TypeMap(typeof(JsonbTypeMapper))]
    public Dictionary<string, string> Specifications { get; set; }
}

public class JsonbTypeMapper : ITypeMapper
{
    public object Map(object value)
    {
        return JsonConvert.DeserializeObject<Dictionary<string, string>>(value?.ToString());
    }
}
```

#### 多数据库支持

```csharp
// 应用启动时
if (DatabaseType == DatabaseType.SqlServer)
{
    RepoDb.SqlServerBootstrap.Initialize();
}
else if (DatabaseType == DatabaseType.PostgreSql)
{
    RepoDb.PostgreSqlBootstrap.Initialize();
}

// 通用仓储
public class GenericRepository<T> : BaseRepository<T, IDbConnection>
{
    public GenericRepository(IDbConnection connection) : base(connection) { }
}
```

#### 分库分表查询

```csharp
// 分表策略
public class OrderShardingStrategy : IShardingStrategy
{
    public string GetTableName(object entity)
    {
        var order = entity as Order;
        return $"Orders_{order.CreatedDate:yyyyMM}";
    }
}

// 分表查询
var orders = connection.Query<Order>(
    tableName: ShardingStrategy.ResolveTableName<Order>(),
    where: o => o.CustomerId == customerId);
```

### 资源与扩展：

* `GitHub` 仓库：https://github.com/mikependon/RepoDB

* 官方文档：https://repodb.net/

* 扩展包：

    * `RepoDb.SqlServer`：SQL Server 支持。

    * `RepoDb.MySql`：MySQL 支持。

    * `RepoDb.SQLite`：SQLite 支持。

