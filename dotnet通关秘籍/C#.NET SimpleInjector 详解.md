### 简介

`Simple Injector` 是一个专注于高性能、易用性和可测试性的 `.NET` 依赖注入（`DI`）容器，支持 `.NET Framework、.NET Core、Xamarin` 等多平台。

设计原则：简单、快速、可预测。它通过编译时及运行时验证帮助早发现配置错误，并力求将依赖解析的开销降到最低。

核心特性：

* 高性能：使用表达式树和编译器优化，解析速度快，接近手动构造。

* 严格诊断：内置诊断工具，检测配置错误（如 `Captive Dependency`）。

* 简单 `API`：专注于构造函数注入，配置清晰，避免复杂绑定。

* 跨平台支持：支持 `.NET Framework、.NET Core` 和 `.NET 5+`。

* `ASP.NET Core` 集成：提供官方适配器，替换默认 `DI` 容器。

* 扩展性：支持拦截器、装饰器和自定义生命周期。

* 开源免费：`MIT` 许可，活跃社区维护。

优点：

* 极高的解析性能，接近手写代码。

* 强大的诊断功能，捕获配置错误。

* `API` 简洁，专注于构造函数注入，鼓励最佳实践。

* 与 `ASP.NET Core` 集成良好，适合现代 `.NET` 项目。

缺点：

* 不支持属性注入或方法注入，强制构造函数注入（可能限制某些场景）。

* 学习曲线略陡，尤其在诊断和配置方面。

* 功能较 `Autofac` 少，复杂绑定支持有限。

* 配置需手动验证作用域，可能增加开发成本。

### 基础用法

#### 安装与配置

```csharp
dotnet add package SimpleInjector
```

#### 基本服务注册

```csharp
using SimpleInjector;

// 定义服务接口和实现
public interface ILogger {
    void Log(string message);
}

public class ConsoleLogger : ILogger {
    public void Log(string message) {
        Console.WriteLine($"[LOG] {message}");
    }
}

// 创建容器并注册服务
var container = new Container();
container.Register<ILogger, ConsoleLogger>(Lifestyle.Transient);

// 验证配置（可选但推荐）
container.Verify();

// 解析服务
var logger = container.GetInstance<ILogger>();
logger.Log("Hello from SimpleInjector!");
```

#### 注册与解析服务

```csharp
// 基础注册
container.Register<IUserRepository, SqlUserRepository>(Lifestyle.Transient);
container.Register<ILogger, FileLogger>(Lifestyle.Singleton);

// 解析实例
var userService = container.GetInstance<IUserService>();
```

#### 模块化配置（IPackage）

```csharp
// 定义模块
public class DataPackage : IPackage {
    public void RegisterServices(Container container) {
        container.Register<IDbContext, AppDbContext>(Lifestyle.Scoped);
    }
}

// 主入口点加载模块
container.RegisterPackages(AppDomain.CurrentDomain.GetAssemblies());
```

### 核心类型与 API

#### Container

* `Simple Injector` 的核心容器类型，负责注册、验证和解析所有服务。

#### Lifestyle

* 描述实例创建与缓存策略的枚举或类型：`Transient, Singleton, Scoped`（以及 `AsyncScoped`）等。

#### Scope / AsyncScopedLifestyle

* 管理请求作用域（如 `Web` 请求）或异步操作作用域内的实例。

### 注册服务与生命周期

| 生命周期         | 注册方式                                         | 场景                               |
| ---------------- | ------------------------------------------------ | ---------------------------------- |
| **Transient**    | `Register<TService, TImpl>()` (默认)             | 无状态、轻量服务                   |
| **Singleton**    | `Register<TService, TImpl>(Lifestyle.Singleton)` | 全局共享、线程安全                 |
| **Scoped**       | `Register<TService, TImpl>(Lifestyle.Scoped)`    | Web 请求、工作单元（Unit of Work） |
| **Async Scoped** | `Lifestyle.AsyncScoped` + `Register<…>`          | 异步流/后台任务作用域              |


```csharp
var container = new Container();

// 默认即为 Transient
container.Register<IUserRepository, UserRepository>();

// Singleton（容器生命周期内唯一实例）
container.Register<IAppSettings, AppSettings>(Lifestyle.Singleton);

// Scoped（Scoped 需先调用 container.BeginLifetimeScope()）
container.Register<IDbContext, MyDbContext>(Lifestyle.Scoped);
```

### 作用域管理

#### 同步作用域

```csharp
using (AsyncScopedLifestyle.BeginScope(container))
{
    var svc = container.GetInstance<IMyScopedService>();
    // 同一作用域内多次 GetInstance 返回同一实例
}
```

#### 异步作用域

```csharp
await AsyncScopedLifestyle.BeginScopeAsync(container);
// 在异步方法中使用 container.GetInstance<…>()
```

#### ASP.NET Core 集成

```csharp
services.AddSimpleInjector(container, options =>
{
    options.AddAspNetCore()
           .AddControllerActivation();
});
// Startup.Configure:
app.UseSimpleInjector(container);
container.Verify();
```

### 高级用法

#### 泛型和开放泛型

```csharp
// IRepository<T> → Repository<T>
container.Register(typeof(IRepository<>), typeof(Repository<>), Lifestyle.Transient);
```

#### 多实现注册

```csharp
container.Collection.Register<IUserService>(typeof(UserService1), typeof(UserService2));

var services = container.GetAllInstances<IUserService>();
```

#### 条件注册

```csharp
container.RegisterConditional<IUserService, UserService>(
    c => c.Consumer.ImplementationType == typeof(UserController), Lifestyle.Scoped);
```

#### 基于约定的批量注册

```csharp
// 从程序集中批量注册服务
var assembly = typeof(Program).Assembly;
container.Register<IRepository, CustomerRepository>(Lifestyle.Scoped);
container.Register<IValidator, CustomerValidator>(Lifestyle.Scoped);

// 或使用约定批量注册
container.Register(typeof(IRepository<>), assembly);
container.Register(typeof(IValidator<>), assembly);

// 注册同一命名空间下所有实现
var types = container.GetTypesToRegister(
    typeof(IHandler<>), 
    new[] { typeof(OrderHandler).Assembly }
);

container.Collection.Register(typeof(IHandler<>), types);
```

#### 带参数的注册

```csharp
// 使用委托注册
container.Register<IMailService>(() => {
    var config = container.GetInstance<IConfiguration>();
    return new SmtpMailService(config.SmtpServer);
}, Lifestyle.Scoped);

// 注册已存在的实例
var settings = new AppSettings { ApiKey = "12345" };
container.RegisterInstance<IAppSettings>(settings);
```

#### 委托工厂注册

```csharp
container.Register(() =>
{
    var cfg = container.GetInstance<IConfiguration>();
    return new SqlConnection(cfg.GetConnectionString("Default"));
}, Lifestyle.Singleton);

container.Register<IPaymentService>(() => {
    var config = container.GetInstance<IConfiguration>();
    return new PaymentService(config.ApiKey);
}, Lifestyle.Singleton);
```

#### 装饰器

```csharp
// 基础实现
container.Register<IOrderService, OrderService>();

// 注册装饰器（执行顺序由下到上）
container.RegisterDecorator<IOrderService, OrderServiceLogDecorator>();
container.RegisterDecorator<IOrderService, OrderServiceCacheDecorator>();

// 解析时得到：CacheDecorator > LogDecorator > OrderService
```

#### 拦截器与 AOP

```csharp
// 使用委托拦截方法调用
container.Register<IService, ServiceImplementation>();
container.InterceptWith<LoggingInterceptor>(service => service is IService);

public class LoggingInterceptor : IInterceptor {
    private readonly ILogger _logger;

    public LoggingInterceptor(ILogger logger) {
        _logger = logger;
    }

    public object Intercept(Invocation invocation) {
        _logger.Log($"调用方法: {invocation.Method.Name}");
        var result = invocation.Proceed();
        _logger.Log($"方法返回: {result}");
        return result;
    }
}
```

#### 异步支持

使用 `AsyncScopedLifestyle` 支持异步操作：

```csharp
container.Options.DefaultScopedLifestyle = new AsyncScopedLifestyle();
using (AsyncScopedLifestyle.BeginScope(container))
{
    var service = container.GetInstance<IUserService>();
}
```

#### 工厂模式

通过工厂动态创建实例：

```csharp
container.Register<IUserServiceFactory, UserServiceFactory>();
public class UserServiceFactory
{
    private readonly Container _container;

    public UserServiceFactory(Container container)
    {
        _container = container;
    }

    public IUserService Create() => _container.GetInstance<IUserService>();
}
```

#### 预编译注册：提升首次解析性能

```csharp
container.Register<IFastService, FastService>(Lifestyle.Singleton);
container.Compile(); // 预编译注册
```

#### 预编译容器（.NET 8+）

```csharp
// 生成预编译表达式树
var expression = container.GetRegistration(typeof(IService)).BuildExpression();
var factory = Expression.Lambda<Func<IService>>(expression).Compile();

// 高频调用直接使用 factory()
var service = factory();
```

#### 避免反射扫描

```csharp
// 明确指定程序集（加速启动）
container.Collection.Register<IValidator>(
    typeof(UserValidator).Assembly.GetExportedTypes()
    .Where(t => t.Name.EndsWith("Validator")));
```

#### 编译时验证

```csharp
// 启动时检查问题（避免运行时错误）
container.Verify(VerificationOption.VerifyAndDiagnose);

// 输出诊断结果
var diagnostics = container.GetRegistrationDiagnostics();
File.WriteAllText("diag.txt", diagnostics);
```

#### 分析工具

```csharp
// 输出依赖图
var graph = container.GetRegistrationGraph();
File.WriteAllText("graph.dot", graph.ToDot());

// 转化为可视化图表
// dot -Tpng graph.dot -o graph.png
```

### 集成 ASP.NET / ASP.NET Core

#### ASP.NET MVC / Web API

```csharp
var container = new Container();
container.Options.DefaultScopedLifestyle = new WebRequestLifestyle();

container.RegisterMvcControllers(Assembly.GetExecutingAssembly());
container.Verify();

DependencyResolver.SetResolver(new SimpleInjectorDependencyResolver(container));
```

#### ASP.NET Core

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var container = new Container();
container.Options.DefaultScopedLifestyle = new AsyncScopedLifestyle();

builder.Services.AddSimpleInjector(container, options =>
{
    options.AddAspNetCore()
           .AddControllerActivation()
           .AddViewComponentActivation();
});

var app = builder.Build();
app.UseSimpleInjector(container);
container.Verify();
```

### 生命周期管理与验证

#### 生命周期验证

```csharp
// 验证容器配置
try {
    container.Verify();
} catch (ValidationException ex) {
    Console.WriteLine("容器配置错误: " + ex.Message);
}
```

#### 处理作用域

```csharp
// 在控制台应用中手动管理作用域
using (var scope = container.BeginExecutionContextScope()) {
    var service = container.GetInstance<IScopedService>();
    service.Process();
} // 作用域结束时释放资源
```