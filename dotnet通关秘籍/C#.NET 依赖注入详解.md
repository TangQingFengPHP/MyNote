### 简介

在 `C#.NET` 中，依赖注入（`Dependency Injection`，简称 `DI`） 是一种设计模式，用于实现控制反转（`Inversion of Control，IoC`），以降低代码耦合、提高可测试性和可维护性。

依赖注入是将一个对象的依赖（即它所需的其他对象或服务）通过外部提供（注入）的方式传递给它，而不是由对象自身创建或查找依赖。

其核心思想是将对象的创建和依赖管理交给容器（IoC 容器），从而解耦代码。`DI` 是现代 `.NET` 开发（尤其是 `ASP.NET Core`）的核心特性之一，广泛应用于企业级应用。

* 传统模式（紧耦合）：

```csharp
public class UserService {
    private readonly DbContext _dbContext = new SqlDbContext(); // 直接创建依赖
    public void SaveUser(User user) { _dbContext.Save(user); }
}
```

* 依赖注入模式（松耦合）：

```csharp
public class UserService {
    private readonly DbContext _dbContext;
    // 依赖通过构造函数注入
    public UserService(DbContext dbContext) { _dbContext = dbContext; }
    public void SaveUser(User user) { _dbContext.Save(user); }
}
```

### NET DI 的核心实现机制

#### 依赖注入（DI）三种形式

* 构造函数注入（`Constructor Injection`）——最常用，依赖通过构造器参数传入。

* 属性注入（`Property Injection`）——容器通过公有可写属性注入依赖。

* 方法注入（`Method Injection`）——依赖作为某个方法的参数，由容器在调用时提供。

```csharp
// 三种注入方式示例
public class SampleService {
    // 构造函数注入（必须依赖）
    private readonly ILogger _logger;
    public SampleService(ILogger logger) { _logger = logger; }
    
    // 属性注入（可选依赖）
    public ICache Cache { get; set; }
    
    // 方法注入（临时依赖）
    public void ProcessData(IDataProcessor processor) {
        processor.Process();
    }
}
```

#### IServiceProvider 与 ServiceCollection

`.NET` 从 `Core 3.0` 开始内置依赖注入框架，核心组件包括：

* `IServiceCollection`：用于注册服务的接口

* `IServiceProvider`：用于解析服务的接口

* `ServiceLifetime`：定义服务的生命周期


#### 服务生命周期（Service Lifetime）

| 生命周期      | 实例作用域                                        | 典型场景                                     |
| ------------- | ------------------------------------------------- | -------------------------------------------- |
| **Transient** | 每次从容器解析（或每次注入）都会创建新实例        | 无状态、轻量、并发安全；如邮件发送、日志记录 |
| **Scoped**    | 在同一个作用域（Scope）内重用，通常对应 HTTP 请求 | Web 请求中同一 DbContext、同一事务           |
| **Singleton** | 容器生命周期内仅创建一次，所有请求共享同一实例    | 配置、缓存、跨请求共享的服务                 |

```csharp
// 注册示例（Startup.cs 或 Program.cs）
services.AddSingleton<AppConfig>();         // 单例
services.AddScoped<DbContext>();            // 作用域
services.AddTransient<IMailService, MailService>(); // 瞬时

// 配置选项模式
builder.Services.Configure<EmailSettings>(builder.Configuration.GetSection("Email"));
```

#### DI 容器工作流程（RRR 原则）

`注册Register->解析Resolve->释放Release`

* 注册：在启动时配置服务与实现的映射（如 `services.AddScoped<ILogger, FileLogger>()`）

* 解析：容器自动构建依赖树，按需创建实例（递归解析构造函数参数）

* 释放：对实现了 `IDisposable` 的服务，容器自动调用 `Dispose()`（作用域结束时释放作用域服务）

### 注入方式

#### 构造函数注入

```csharp
public class UserService : IUserService
{
    private readonly IUserRepository _repo;
    private readonly IEmailSender _email;

    public UserService(IUserRepository repo, IEmailSender email)
    {
        _repo   = repo;
        _email  = email;
    }
    // …
}
```

* 优点：依赖清晰、强制性高，易于单元测试。

* 缺点：参数过多时构造函数臃肿，可考虑组合参数对象或拆分职责。

#### 属性注入

```csharp
public class ReportGenerator
{
    [Inject]  // 或容器约定：仅对 public 可写属性注入
    public ILogger Logger { get; set; }

    public void Generate() => Logger.Log("Report");
}
```

* 使用场景：当依赖为可选，或用在框架层（如 `MVC Controller` 中 `[FromServices]` 注入）。

#### 方法注入

```csharp
public class DataImporter
{
    public void Import([FromServices] IDataReader reader)
    {
        reader.Read();
    }
}
```

* 典型场景：`ASP.NET Core` 中的 `Action` 方法参数注入。

### 高级特性

#### 工厂模式注册

```csharp
services.AddTransient<IConnection>(sp =>
{
    var cfg = sp.GetRequiredService<IConfiguration>();
    return new SqlConnection(cfg.GetConnectionString("Default"));
});

// 注册工厂函数
services.AddTransient<IService>(provider => 
{
    var config = provider.GetRequiredService<IConfiguration>();
    return config["ServiceType"] == "A" 
        ? new ServiceA() 
        : new ServiceB();
});
```

* 在注册时通过 `IServiceProvider` 动态创建实例。

#### 装饰器模式与拦截器一

```csharp
// 注册日志装饰器
services.AddScoped(typeof(ILogger<>), typeof(LoggingDecorator<>));

services.AddScoped<IService, CoreService>();
services.Decorate<IService, LoggingDecorator>();
services.Decorate<IService, CachingDecorator>();
```

#### 装饰器模式与拦截器二

```csharp
public interface ICommandHandler<T>
{
    void Handle(T command);
}

public class LoggingDecorator<T> : ICommandHandler<T>
{
    private readonly ICommandHandler<T> _inner;
    private readonly ILogger _logger;

    public LoggingDecorator(ICommandHandler<T> inner, ILogger<LoggingDecorator<T>> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public void Handle(T command)
    {
        _logger.LogInformation("Handling command");
        _inner.Handle(command);
    }
}

// 注册装饰器
builder.Services.AddScoped(typeof(ICommandHandler<>), typeof(LoggingDecorator<>));
builder.Services.AddScoped<ICommandHandler<MyCommand>, MyCommandHandler>();
```

#### 泛型与开放式泛型注册

```csharp
services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

// 使用
public class ProductService
{
    public ProductService(IRepository<Product> productRepo)
    {
        // ...
    }
}
```

* 支持同时注册一类类型的泛型接口与实现。

#### 条件注册

根据环境或配置动态注册服务

```csharp
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddScoped<IUserService, MockUserService>();
}
else
{
    builder.Services.AddScoped<IUserService, UserService>();
}
```

#### 多实现处理

```csharp
// 注册多个实现
services.AddTransient<INotificationService, EmailNotification>();
services.AddTransient<INotificationService, SmsNotification>();
services.AddTransient<INotificationService, PushNotification>();

// 注入所有实现
public class NotificationManager
{
    private readonly IEnumerable<INotificationService> _services;
    
    public NotificationManager(IEnumerable<INotificationService> services)
    {
        _services = services;
    }
    
    public void SendAll(string message)
    {
        foreach (var service in _services)
        {
            service.Send(message);
        }
    }
}
```

#### 选项模式(Options Pattern)

```csharp
// 配置类
public class EmailSettings
{
    public string Host { get; set; }
    public int Port { get; set; }
}

// 注册配置
services.Configure<EmailSettings>(Configuration.GetSection("Email"));

// 注入使用
public class EmailService
{
    private readonly EmailSettings _settings;
    
    public EmailService(IOptions<EmailSettings> options)
    {
        _settings = options.Value;
    }
}
```

#### 服务覆盖

```csharp
// 默认注册
services.AddScoped<IService, DefaultService>();

// 环境特定覆盖
if (env.IsDevelopment())
{
    services.AddScoped<IService, MockService>();
}
```

#### 生命周期验证

```csharp
// 在开发环境启用验证
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddOptions().ValidateOnStart();
}
```

#### 避免服务定位器反模式

反模式示例（直接通过 `ServiceProvider` 解析服务）

破坏单一职责原则，导致代码难以测试和维护

```csharp
public class BadService {
    private readonly IServiceProvider _provider;
    public BadService(IServiceProvider provider) { _provider = provider; }
    public void DoWork() {
        var logger = _provider.GetService<ILogger>(); // 服务定位器反模式
        logger.Log("Work done");
    }
}
```

#### 依赖注入与异步初始化

当服务需要异步初始化时（如数据库连接），可使用工厂模式

```csharp
public class AsyncService : IAsyncService {
    private readonly HttpClient _httpClient;
    private bool _isInitialized = false;
    
    public AsyncService(HttpClient httpClient) {
        _httpClient = httpClient;
    }
    
    public async Task InitializeAsync() {
        if (!_isInitialized) {
            // 异步初始化逻辑
            await _httpClient.GetStringAsync("https://api.example.com/init");
            _isInitialized = true;
        }
    }
}

// 注册时使用工厂确保异步初始化
services.AddTransient<IAsyncService>(provider => {
    var service = new AsyncService(provider.GetRequiredService<HttpClient>());
    _ = service.InitializeAsync(); // 非阻塞初始化
    return service;
});
```

#### 性能优化

```csharp
// 使用TryAdd防止重复注册
services.TryAddSingleton<ICacheService, MemoryCache>();

// 预编译服务提供者
var serviceProvider = services.BuildServiceProvider(
    new ServiceProviderOptions
    {
        ValidateScopes = true,
        ValidateOnBuild = true
    });
```

#### 生命周期验证工具

```csharp
// dotnet add package Microsoft.Extensions.DependencyInjection.Diagnostic
var analyzer = new DependencyInjectionDiagnosticAnalyzer();
analyzer.AnalyzeServices(builder.Services); // 检测生命周期冲突
```

#### 异常处理

构造函数中检查依赖是否为 `null`

```csharp
public MyService(IUserService userService)
{
    _userService = userService ?? throw new ArgumentNullException(nameof(userService));
}
```

#### 模块化注册

将服务注册逻辑封装到扩展方法

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddMyServices(this IServiceCollection services)
    {
        services.AddScoped<IUserService, UserService>();
        return services;
    }
}

// 使用
builder.Services.AddMyServices();
```

### 依赖注入在 `ASP.NET Core` 中的应用

#### 内置 DI 与中间件集成

```csharp
// Program.cs 配置示例
var builder = WebApplication.CreateBuilder(args);

// 注册服务
builder.Services.AddControllers();
builder.Services.AddDbContext<AppDbContext>(); // 自动使用Scoped生命周期
builder.Services.AddSingleton<IConfiguration>(builder.Configuration);

var app = builder.Build();

// 在中间件中使用依赖注入
app.Use(async (context, next) => {
    var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
    logger.LogInformation("请求开始");
    await next();
    logger.LogInformation("请求结束");
});
```

#### 控制器与依赖注入

`ASP.NET Core` 控制器自动支持构造函数注入

```csharp
public class UserController : ControllerBase {
    private readonly IUserService _userService;
    private readonly ILogger<UserController> _logger;
    
    public UserController(IUserService userService, ILogger<UserController> logger) {
        _userService = userService;
        _logger = logger;
    }
    
    [HttpGet]
    public IActionResult GetAll() {
        var users = _userService.GetAll();
        _logger.LogInformation("获取用户列表");
        return Ok(users);
    }
}
```

#### 典型 Web API 项目 示例

```csharp
// 模型和接口
public record User(int Id, string Name);

public interface IUserRepository
{
    User GetUser(int id);
}

public interface IUserService
{
    string GetUserName(int id);
}

// 实现
public class UserRepository : IUserRepository
{
    public User GetUser(int id) => new User(id, $"User_{id}");
}

public class UserService : IUserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }

    public string GetUserName(int id) => _repository.GetUser(id).Name;
}

// 注册服务
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IUserService, UserService>();

// 控制器
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;

    public UsersController(IUserService userService)
    {
        _userService = userService;
    }

    [HttpGet("{id}")]
    public IActionResult Get(int id)
    {
        return Ok(_userService.GetUserName(id));
    }
}
```

### 依赖注入的测试与调试

#### 单元测试中的依赖注入

使用 `Moq` 等框架模拟依赖

```csharp
[Test]
public void UserService_ShouldSaveUser() {
    // 1. 创建模拟依赖
    var mockRepo = new Mock<IUserRepository>();
    var mockLogger = new Mock<ILogger<UserService>>();
    
    // 2. 注入模拟对象
    var service = new UserService(mockRepo.Object, mockLogger.Object);
    
    // 3. 设置预期行为
    mockRepo.Setup(x => x.Save(It.IsAny<User>())).Returns(true);
    
    // 4. 执行测试
    var result = service.Save(new User { Id = 1, Name = "Test" });
    
    // 5. 验证结果
    Assert.IsTrue(result);
    mockRepo.Verify(x => x.Save(It.IsAny<User>()), Times.Once);
}
```

#### 运行时依赖注入调试

使用 `.NET` 内置的 `IServiceProvider` 诊断工具

```csharp
// 打印所有注册的服务
var serviceDescriptors = builder.Services
    .Where(sd => sd.ImplementationType != null)
    .Select(sd => $"{sd.Lifetime}: {sd.ServiceType.Name} -> {sd.ImplementationType.Name}");
foreach (var descriptor in serviceDescriptors) {
    Console.WriteLine(descriptor);
}
```

### 第三方容器集成

#### Autofac 集成

```csharp
// 安装包: Autofac.Extensions.DependencyInjection
builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());

// 配置模块
builder.Host.ConfigureContainer<ContainerBuilder>(builder => 
{
    builder.RegisterModule<MyApplicationModule>();
});
```

#### DryIoc 集成

```csharp
// 安装包: DryIoc.Microsoft.DependencyInjection
builder.Host.UseServiceProviderFactory(new DryIocServiceProviderFactory());
```

### 常见陷阱及解决方案

|  陷阱   |  现象   |  解决方案   |
| --- | --- | --- |
|  生命周期不匹配   |  作用域服务注入单例服务   |  确保依赖的生命周期 <= 消费者   |
|  捕获依赖   |  单例服务持有瞬时服务引用   |  使用IServiceScopeFactory创建作用域   |
|  循环依赖   |  StackOverflow异常   |  重构设计，引入中介者模式   |
|  过多构造函数参数   |  类职责过重   |  	应用领域驱动设计，拆分聚合   |