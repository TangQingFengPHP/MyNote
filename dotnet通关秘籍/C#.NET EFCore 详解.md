### 简介

`Entity Framework Core` 是 `.NET` 平台下的开源对象关系映射 (`ORM`) 框架，由 `Microsoft` 开发和维护。它允许开发者通过操作 `.NET` 对象来与数据库交互，而无需编写大量 `SQL` 代码，支持多种数据库（`SQL Server、MySQL、PostgreSQL、SQLite` 等）。

核心优势：

* 提高开发效率：减少样板代码，专注业务逻辑

* 支持 `LINQ` 查询：使用强类型查询代替字符串拼接

* 数据库迁移：自动管理数据库结构变更

* 跨平台支持：可在 `Windows、Linux、macOS` 上运行

* 与 `.NET` 生态深度集成：依赖注入、`ASP.NET Core` 等

### 基础概念与术语

#### DbContext：

* `EF Core` 的核心类，负责与数据库的交互

* 包含 `DbSet` 属性，表示数据库中的表

* 管理实体的生命周期和状态

#### 实体 (Entity)：

* 映射到数据库表的 `.NET` 类

* 通常是 `POCO (Plain Old CLR Object)`

#### DbSet<T>：

* 表示数据库中的表，类似于集合

* 可通过 `LINQ` 查询数据

#### 迁移 (Migrations)：

* 记录数据库结构变更的代码文件

* 用于更新数据库架构

### EF Core核心概念

#### ORM框架核心功能

* 对象-关系映射：将数据库表映射为 `C#` 实体类

* 变更跟踪：自动跟踪实体状态变化

* `LINQ` 支持：使用 `LINQ` 语法生成 `SQL` 查询

* 数据库迁移：代码优先方式管理数据库架构

* 事务管理：自动或手动事务控制

#### EF Core vs EF6

|  特性   | EF Core    |  EF6   |
| --- | --- | --- |
|  跨平台   |   ✅  |  ❌   |
|  轻量级   |  ✅   |  ❌  |
|  LINQ翻译   |  更强大   |  较弱   |
|  性能   |  更优   |  较差   |
|  数据库支持   |  更广泛   |  有限   |
|  开发模式   |  代码优先/数据库优先   |  相同   |

### 安装与配置

`NuGet` 包

* 核心包：`Microsoft.EntityFrameworkCore`。

* 数据库提供者：

    * SQL Server：`Microsoft.EntityFrameworkCore.SqlServer`

    * SQLite：`Microsoft.EntityFrameworkCore.Sqlite`

    * MySQL：`Pomelo.EntityFrameworkCore.MySql`

    * PostgreSQL：`Npgsql.EntityFrameworkCore.PostgreSQL`

    * 工具包：`Microsoft.EntityFrameworkCore.Design`（用于迁移）。

#### 基础安装

```shell
# 基本包
dotnet add package Microsoft.EntityFrameworkCore

# SQL Server 支持
dotnet add package Microsoft.EntityFrameworkCore.SqlServer

# 设计时工具（迁移等）
dotnet add package Microsoft.EntityFrameworkCore.Design

# 命令行工具
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

#### 创建DbContext

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    public DbSet<Post> Posts { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlServer("Server=.;Database=BlogDb;Trusted_Connection=True;");
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 配置模型关系
        modelBuilder.Entity<Blog>()
            .HasMany(b => b.Posts)
            .WithOne(p => p.Blog)
            .HasForeignKey(p => p.BlogId);
    }
}
```

#### 实体类定义

```csharp
public class Blog
{
    public int BlogId { get; set; }
    public string Url { get; set; }
    
    // 导航属性
    public List<Post> Posts { get; set; } = new List<Post>();
}

public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    
    // 外键
    public int BlogId { get; set; }
    public Blog Blog { get; set; }
}
```

### 数据库迁移（Code First）

#### 创建迁移

```shell
dotnet ef migrations add InitialCreate
```

生成文件：

* `Migrations/XXXXXXXXXXXX_InitialCreate.cs` - 迁移操作

* `AppDbContextModelSnapshot.cs` - 当前模型快照

#### 应用迁移到数据库

```shell
dotnet ef database update
```

#### 迁移高级操作

```shell
# 回滚到特定迁移
dotnet ef database update PreviousMigrationName

# 生成SQL脚本（不执行）
dotnet ef migrations script -o migration.sql

# 删除最后一次迁移
dotnet ef migrations remove
```

#### 配置迁移历史表

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasDefaultSchema("blogging");
    
    // 配置迁移历史表
    modelBuilder.HasHistoryTable("__EFMigrationsHistory", "blogging");
}
```

#### 迁移文件结构

```csharp
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Users",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(maxLength: 50, nullable: false),
                CreatedAt = table.Column<DateTime>(nullable: false, defaultValueSql: "GETDATE()"),
                Balance = table.Column<decimal>(type: "decimal(18,2)", nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Users", x => x.Id);
            });
        
        // ... 其他表创建 ...
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Users");
    }
}
```

### CRUD操作

#### 创建（Create）

```csharp
using var db = new AppDbContext();

// 添加单个实体
var blog = new Blog { Url = "https://example.com" };
db.Blogs.Add(blog);

// 添加关联实体
blog.Posts.Add(new Post { Title = "Hello EF Core", Content = "Getting started" });

await db.SaveChangesAsync();
```

#### 查询（Read）

```csharp
// 基本查询
var blogs = await db.Blogs
    .Where(b => b.Url.Contains("example"))
    .ToListAsync();

// 包含关联数据（Eager Loading）
var blogWithPosts = await db.Blogs
    .Include(b => b.Posts)
    .FirstOrDefaultAsync(b => b.BlogId == 1);

// 投影查询
var blogTitles = await db.Blogs
    .Select(b => new { b.BlogId, b.Url })
    .ToListAsync();

// 原生SQL查询
var blogs = await db.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Url LIKE '%example%'")
    .ToListAsync();
```

#### 更新（Update）

```csharp
var blog = await db.Blogs.FindAsync(1);
blog.Url = "https://updated-url.com";

// 直接更新（无需先查询）
db.Blogs.Update(new Blog { BlogId = 2, Url = "https://another.com" });

await db.SaveChangesAsync();

await context.Users
    .Where(u => u.Id == 1)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(u => u.Name, "John Updated")
        .SetProperty(u => u.Balance, u => u.Balance + 50));

// 原生SQL更新
context.Database.ExecuteSqlRaw("UPDATE Blogs SET Rating = 5 WHERE Id = 1");
```

#### 删除（Delete）

```csharp
var blog = await db.Blogs.FindAsync(1);
db.Blogs.Remove(blog);

// 批量删除（EF Core 7+）
await db.Blogs
    .Where(b => b.BlogId < 10)
    .ExecuteDeleteAsync();

await db.SaveChangesAsync();
```

### 高级功能

#### 数据注解

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

public class Blog
{
    [Key]  // 主键
    public int BlogId { get; set; }
    
    [Required]  // 非空
    [MaxLength(200)]  // 最大长度
    public string Url { get; set; }
    
    [NotMapped]  // 不映射到数据库
    public string DisplayUrl { get; set; }
    
    [Column(TypeName = "decimal(5, 2)")]  // 指定列类型
    public decimal Rating { get; set; }
}
```

#### 复杂模型配置（Fluent API）

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // 配置实体
    modelBuilder.Entity<Blog>(b =>
    {
        // 表名和架构
        b.ToTable("Blogs", "blogging");
        
        // 主键
        b.HasKey(e => e.BlogId);
        
        // 属性配置
        b.Property(e => e.Url)
            .IsRequired()
            .HasMaxLength(200);
            
        // 索引
        b.HasIndex(e => e.Url)
            .IsUnique();
            
        // 关系配置
        b.HasMany(e => e.Posts)
            .WithOne(e => e.Blog)
            .HasForeignKey(e => e.BlogId)
            .OnDelete(DeleteBehavior.Cascade);
    });
    
    // 数据种子
    modelBuilder.Entity<Blog>().HasData(
        new Blog { BlogId = 1, Url = "https://seededblog.com" }
    );
}
```

#### 事务管理

```csharp
using var transaction = await db.Database.BeginTransactionAsync();

try
{
    // 多个操作
    db.Blogs.Add(new Blog { /* ... */ });
    await db.SaveChangesAsync();
    
    db.Posts.Add(new Post { /* ... */ });
    await db.SaveChangesAsync();
    
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

#### 性能优化

```csharp
// 禁用变更跟踪（只读场景）
var blogs = await db.Blogs
    .AsNoTracking()
    .ToListAsync();

// 批量操作（EF Core 7+）
await db.Blogs
    .Where(b => b.Rating < 3)
    .ExecuteUpdateAsync(b => b.SetProperty(p => p.Rating, p => p.Rating + 1));

// 拆分查询（避免笛卡尔爆炸）
var blogs = await db.Blogs
    .Include(b => b.Posts)
    .AsSplitQuery()
    .ToListAsync();

// 连接池配置
// 在 OnConfiguring 中配置连接池
optionsBuilder.UseSqlServer(connectionString, 
    options => options.EnableRetryOnFailure());

// 批量操作
db.BulkInsert(blogs) // （需 EFCore.BulkExtensions 包）

// 预编译查询
var query = EF.CompileQuery(...)
```

#### 全局查询过滤器

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasQueryFilter(b => !b.IsDeleted);
}

// 临时忽略过滤器
var blogs = await db.Blogs
    .IgnoreQueryFilters()
    .ToListAsync();
```

#### 值转换器

```csharp
// 将枚举转换为字符串
modelBuilder.Entity<Post>()
    .Property(p => p.Status)
    .HasConversion(
        v => v.ToString(),
        v => (PostStatus)Enum.Parse(typeof(PostStatus), v));

// 加密敏感数据
modelBuilder.Entity<User>()
    .Property(u => u.ApiKey)
    .HasConversion(
        key => Encrypt(key),
        dbValue => Decrypt(dbValue));
```

#### 并发控制

```csharp
// 实体类添加并发标记
public class User
{
    [Timestamp]
    public byte[] RowVersion { get; set; }
}

// 处理并发冲突
try
{
    // 更新操作...
    await context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var databaseValues = await entry.GetDatabaseValuesAsync();
    
    // 解决冲突策略（如：客户端优先）
    entry.OriginalValues.SetValues(databaseValues);
    await context.SaveChangesAsync();
}
```

#### 性能分析工具

```csharp
// 1. 记录查询日志
optionsBuilder.LogTo(Console.WriteLine);

// 2. 使用 MiniProfiler
services.AddDbContext<AppDbContext>(options => 
    options.UseSqlServer(connection)
           .AddMiniProfiler());

// 3. 分析查询计划
var query = context.Users.Where(u => u.Balance > 100);
var sql = query.ToQueryString();
```

#### 多数据库支持配置

```csharp
// 根据环境切换数据库
if (Environment.IsDevelopment())
    options.UseSqlite("Data Source=dev.db");
else
    options.UseSqlServer(Configuration.GetConnectionString("ProductionDB"));
```

#### JSON 列支持

```csharp
modelBuilder.Entity<Author>().OwnsOne(a => a.Contact, c => 
{
    c.ToJson();  // 将 Contact 对象序列化为 JSON 列
});
```

#### 高性能子查询优化

```csharp
var blogs = db.Blogs
    .Select(b => new 
    {
        b.Url,
        PostCount = b.Posts.Count(p => p.IsPublished)
    })
    .ToList();  // 生成单次 JOIN 查询
```

#### 使用 `ValueTask<T>` 优化高频查询

```csharp
public async ValueTask<User?> GetUserAsync(int id)
{
    return await _context.Users.FindAsync(id);
}
```

### 实际应用场景

#### 分页查询

```csharp
public async Task<PagedResult<User>> GetUsers(int page = 1, int pageSize = 10)
{
    var query = context.Users.AsQueryable();
    
    var totalCount = await query.CountAsync();
    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();
        
    return new PagedResult<User>(items, totalCount, page, pageSize);
}

public record PagedResult<T>(IEnumerable<T> Items, int TotalCount, int Page, int PageSize);
```

#### 软删除实现

```csharp
public interface ISoftDelete
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
}

public class User : ISoftDelete
{
    // ... 其他属性 ...
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
}

// 全局查询过滤器
modelBuilder.Entity<User>().HasQueryFilter(u => !u.IsDeleted);

// 软删除方法
public async Task SoftDeleteAsync<T>(int id) where T : class, ISoftDelete
{
    await context.Set<T>()
        .Where(e => e.Id == id)
        .ExecuteUpdateAsync(setters => setters
            .SetProperty(e => e.IsDeleted, true)
            .SetProperty(e => e.DeletedAt, DateTime.UtcNow));
}
```

#### 多租户架构

```csharp
public class TenantEntity
{
    public int TenantId { get; set; }
}

// 全局过滤器
modelBuilder.Entity<User>().HasQueryFilter(u => u.TenantId == _tenantProvider.TenantId);

// 自动设置租户ID
public override int SaveChanges()
{
    foreach (var entry in ChangeTracker.Entries<TenantEntity>())
    {
        if (entry.State == EntityState.Added)
        {
            entry.Entity.TenantId = _tenantProvider.TenantId;
        }
    }
    return base.SaveChanges();
}
```

### 架构设计模式

#### 仓储模式实现

```csharp
public interface IRepository<T> where T : class
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Update(T entity);
    void Delete(T entity);
}

public class Repository<T> : IRepository<T> where T : class
{
    private readonly AppDbContext _context;
    private readonly DbSet<T> _dbSet;

    public Repository(AppDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task AddAsync(T entity) => await _dbSet.AddAsync(entity);
    public void Update(T entity) => _dbSet.Update(entity);
    public void Delete(T entity) => _dbSet.Remove(entity);
    
    public async Task<T> GetByIdAsync(int id) => 
        await _dbSet.FindAsync(id);
    
    public async Task<IEnumerable<T>> GetAllAsync() => 
        await _dbSet.ToListAsync();
}
```

#### 工作单元模式

```csharp
public interface IUnitOfWork : IDisposable
{
    IRepository<Blog> Blogs { get; }
    IRepository<Post> Posts { get; }
    Task<int> CommitAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    
    public UnitOfWork(AppDbContext context)
    {
        _context = context;
        Blogs = new Repository<Blog>(context);
        Posts = new Repository<Post>(context);
    }
    
    public IRepository<Blog> Blogs { get; }
    public IRepository<Post> Posts { get; }
    
    public async Task<int> CommitAsync() => 
        await _context.SaveChangesAsync();
    
    public void Dispose() => _context.Dispose();
}
```

### 与 ASP.NET Core 集成

#### 注册 DbContext

```csharp
// Startup.cs (ASP.NET Core 3.1 及以下)
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<BlogContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));
        
    services.AddControllers();
}

// Program.cs (ASP.NET Core 6.0+)
builder.Services.AddDbContext<BlogContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

#### 在控制器中使用

```csharp
public class BlogsController : ControllerBase
{
    private readonly BlogContext _context;
    
    public BlogsController(BlogContext context)
    {
        _context = context;
    }
    
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Blog>>> GetBlogs()
    {
        return await _context.Blogs.ToListAsync();
    }
    
    [HttpPost]
    public async Task<ActionResult<Blog>> PostBlog(Blog blog)
    {
        _context.Blogs.Add(blog);
        await _context.SaveChangesAsync();
        
        return CreatedAtAction(nameof(GetBlog), new { id = blog.BlogId }, blog);
    }
}
```

### 最佳实践

#### 生命周期管理

```csharp
// ASP.NET Core 中注册为 Scoped
services.AddDbContext<AppDbContext>(options => 
    options.UseSqlServer(Configuration.GetConnectionString("Default")));
```

#### 连接字符串管理

```json
// appsettings.json
{
  "ConnectionStrings": {
    "Default": "Server=.;Database=MyAppDb;User=sa;Password=P@ssw0rd;"
  }
}
```

#### 常见错误处理

```csharp
// 1. 处理并发冲突
catch (DbUpdateConcurrencyException) { /* ... */ }

// 2. 处理唯一约束冲突
catch (DbUpdateException ex) when (ex.InnerException is SqlException sqlEx && sqlEx.Number == 2601) { }

// 3. 处理死锁
catch (SqlException ex) when (ex.Number == 1205) { // 重试逻辑 }
```

#### 性能优化注意事项

* 避免 `N+1` 查询（使用 `Include/ThenInclude`）

* 只查询需要的数据（`Select` 投影）

* 只读场景使用 `AsNoTracking`

* 批量操作使用 `ExecuteUpdate/ExecuteDelete`

* 合理使用索引

* 定期分析慢查询