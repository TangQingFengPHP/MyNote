### 简介

`Autofac` 是一个成熟的、功能丰富的 `.NET` 依赖注入（`DI`）容器。

相比于内置容器，它额外提供：模块化注册、装饰器（`Decorator`）、拦截器（`Interceptor`）、强o的属性/方法注入支持、基于约定的程序集扫描等特性。

**核心组件**

* `ContainerBuilder`：用于注册服务的构建器

* `IContainer`：服务容器，负责解析依赖

* `ILifetimeScope`：生命周期作用域，管理对象生命周期

* `Module`：模块化注册的基类

### 核心特性

#### 灵活的服务注册

* 支持按类型、实例、工厂、委托等多种方式注册；

* 支持开放泛型（`IRepository<T> → Repository<T>`）、条件注册。

#### 生命周期管理

* 同样有 `Singleton、InstancePerDependency`（每次）、`InstancePerLifetimeScope`（等同于 `Scoped`）、`InstancePerMatchingLifetimeScope`（自定义标签作用域）等多种模式。

#### 模块化（Module）

* 通过继承 `Module` 将相关注册逻辑封装为可复用、可拆分的单元；

* 支持自动加载整个程序集中的 `Module`。

#### 装饰器（Decorator）

* 可以很方便地为某个服务添加一层或多层“外衣”，例如日志、缓存等横切逻辑。

#### 拦截器（Interceptor）

* 基于 `Castle DynamicProxy`，自动在方法调用前后执行 `AOP` 逻辑，如性能监控、事务管理等。

#### 程序集扫描

* 支持按命名空间、接口、特性等约定，将多个实现一行注册。

### 基本使用

#### 引入 NuGet

```shell
Install-Package Autofac
Install-Package Autofac.Extensions.DependencyInjection   # ASP.NET Core 集成
```

#### 最简示例

```csharp
using Autofac;

var builder = new ContainerBuilder();

// 1. 注册具体类型（默认使用 Transient 生命周期）
builder.RegisterType<OrderService>();

// 2. 注册接口与实现
builder.RegisterType<DbOrderRepository>().As<IOrderRepository>();

// 3. 注册单例
builder.RegisterType<AppConfig>().SingleInstance();

// 4. 注册工厂方法
builder.Register(c => new HttpClient()).InstancePerLifetimeScope();

builder.Register(c => {
    var config = c.Resolve<IConfiguration>();
    return new MyService(config.GetConnectionString("Default"));
}).As<IMyService>();

// 5. 注册已存在的实例
var logger = new NLogLogger();
builder.RegisterInstance(logger).As<ILogger>();

// 6. 注册开放泛型类型
builder.RegisterGeneric(typeof(MyRepository<>)).As(typeof(IRepository<>));

// 构建容器
var container = builder.Build();

// 从容器中解析服务
using (var scope = container.BeginLifetimeScope()) {
    var service = scope.Resolve<OrderService>();
}
```

### 服务暴露 (Exposing Services)

#### `As<T>()`: 将组件暴露为特定服务类型 T (通常是接口)

```csharp
builder.RegisterType<EmailSender>().As<IEmailSender>();
```

#### `As<TService1, TService2>()`: 将组件暴露为多个服务类型

```csharp
builder.RegisterType<FileLogger>().As<ILogger, IFileLogger>();
```

#### `AsImplementedInterfaces()`: 将组件暴露为其实现的所有公共接口

```csharp
builder.RegisterType<CustomerRepository>().AsImplementedInterfaces();
// 等同于 .As<ICustomerRepository, IRepository<Customer>, ...>()
```

#### `AsSelf()`: 将组件暴露为其自身类型。通常与其他 `.As()` 一起使用

```csharp
builder.RegisterType<MySpecialService>().AsSelf().As<ISpecialService>();
```


### 生命周期（Scope）管理

| 生命周期               | 对应 Autofac 方法                          | 场景                             |
| ---------------------- | ------------------------------------------ | -------------------------------- |
| **瞬时 / 每次依赖**    | `.InstancePerDependency()` (默认)          | 无状态、轻量对象                 |
| **单例 / 容器范围**    | `.SingleInstance()`                        | 线程安全、全局共享               |
| **每个生命周期作用域** | `.InstancePerLifetimeScope()`              | 类似 ASP.NET Core 的 Scoped 模式 |
| **每个 HTTP 请求内单例** | `.InstancePerRequest()`              | 仅 ASP.NET Core 集成支持 |
| **自定义标记作用域**   | `.InstancePerMatchingLifetimeScope("tag")` | 某些特定操作需独立作用域         |

```csharp
builder.RegisterType<CacheService>()
       .As<ICacheService>()
       .SingleInstance();

builder.RegisterType<DbContext>()
       .AsSelf()
       .InstancePerLifetimeScope();

var tagScope = container.BeginLifetimeScope("unitOfWork");
var ctx = tagScope.Resolve<DbContext>();  // 属于 "unitOfWork" 作用域
```

### 高级特性

#### 属性注入

```csharp
public class MyService {
    [Inject] // 注入属性
    public ILogger Logger { get; set; }  // 自动注入的属性
}

// 注册时启用属性注入
builder.RegisterType<MyService>()
       .ProjertiesAutowired();
```

#### 手动解析

```csharp
public class MyService
{
    private readonly IComponentContext _context;

    public MyService(IComponentContext context)
    {
        _context = context;
    }

    public void DoWork()
    {
        var userService = _context.Resolve<IUserService>();
        // 使用 userService
    }
}
```

#### 命名与键控服务

```csharp
// 注册多个实现，使用命名区分
builder.RegisterType<SqlDbContext>().Named<DbContext>("sql");
builder.RegisterType<MongoDbContext>().Named<DbContext>("mongo");

// 解析时指定名称
var sqlContext = scope.ResolveNamed<DbContext>("sql");
```

#### 集合解析

自动解析所有实现某一接口的服务

```csharp
container.RegisterType<UserService1>().As<IUserService>();
container.RegisterType<UserService2>().As<IUserService>();

// 解析所有实现
public class MultiService
{
    private readonly IEnumerable<IUserService> _services;

    public MultiService(IEnumerable<IUserService> services)
    {
        _services = services;
    } 
}
```

#### 作用域管理

```csharp
using (var scope = container.BeginLifetimeScope())
{
    var service = scope.Resolve<IUserService>();
    // 使用 service
}
```

#### 高级作用域控制

* 嵌套作用域

```csharp
  using (var parentScope = container.BeginLifetimeScope())
  {
      var serviceA = parentScope.Resolve<ServiceA>();
      using (var childScope = parentScope.BeginLifetimeScope())
      {
          var serviceB = childScope.Resolve<ServiceB>(); // 可访问父作用域实例
      }
  }
```

* 带标签作用域 (Tagged Scopes)

```csharp
  // 注册时标记作用域
  builder.RegisterType<EmailSender>()
         .As<IEmailSender>()
         .InstancePerMatchingLifetimeScope("transaction");

  // 使用标签创建作用域
  using (var transactionScope = container.BeginLifetimeScope("transaction"))
  {
      var emailSender = transactionScope.Resolve<IEmailSender>(); // 事务内共享实例
  }
```

#### 模块化注册

当项目庞大或有多个子系统时，将注册逻辑封装在 `Module` 中，便于维护与复用。

```csharp
// 定义模块
public class DataModule : Module {
    protected override void Load(ContainerBuilder builder) {
        builder.RegisterType<DbConnection>().As<IDbConnection>();
        builder.RegisterType<OrderRepository>().As<IOrderRepository>();
    }
}

// 注册模块
builder.RegisterModule<DataModule>();

public class RepositoryModule : Module
{
    protected override void Load(ContainerBuilder builder)
    {
        // 自动扫描程序集内的 IRepository<> 实现
        builder.RegisterAssemblyTypes(typeof(RepositoryModule).Assembly)
               .AsClosedTypesOf(typeof(IRepository<>))
               .InstancePerLifetimeScope();
    }
}

// 在主程序里加载
var builder = new ContainerBuilder();
builder.RegisterModule<RepositoryModule>();
builder.RegisterModule(new SettingsModule(settings));
```

#### 装饰器（Decorator）

给已有服务动态“套”上日志、缓存等功能，不需改动原实现。

```csharp
// 原始服务
builder.RegisterType<OrderService>()
       .As<IOrderService>();

// 装饰器：构造参数 inner 实例即被装饰的服务
builder.RegisterDecorator<LoggingOrderService, IOrderService>();
```

* 等同于：当解析 `IOrderService` 时，先创建 `OrderService`，再将其传给 `LoggingOrderService` 构造器。

#### AOP 拦截与动态代理

基于 `Castle DynamicProxy`，为接口或虚方法生成代理，在调用前后执行拦截逻辑。

**引入包**

```shell
Install-Package Autofac.Extras.DynamicProxy
Install-Package Castle.Core
```

```csharp
// 定义拦截器
public class LoggingInterceptor : IInterceptor {
    public void Intercept(IInvocation invocation) {
        Console.WriteLine($"调用方法: {invocation.Method.Name}");
        invocation.Proceed();  // 执行原始方法
        Console.WriteLine($"方法返回: {invocation.ReturnValue}");
    }
}

// 注册带拦截器的服务
builder.RegisterType<OrderService>()
       .As<IOrderService>()
       .EnableInterfaceInterceptors()
       .InterceptedBy(typeof(LoggingInterceptor));

builder.RegisterType<LoggingInterceptor>();
```

#### 程序集扫描与约定注册

根据命名、接口或特性，批量注册类型：

```csharp
// 按接口后缀约定：所有以 "Service" 结尾的类，注册为它们所实现的接口
builder.RegisterAssemblyTypes(AppDomain.CurrentDomain.GetAssemblies())
       .Where(t => t.Name.EndsWith("Service"))
       .AsImplementedInterfaces()
       .InstancePerLifetimeScope();
```

#### 条件注册

```csharp
builder.RegisterType<SqlRepository>()
       .As<IRepository>()
       .When(c => c.Parameters.Any(p => p.Name == "useSql" && (bool)p.Value));
```

#### 装配时验证

```csharp
// 验证所有注册的服务可被解析
container.AssertConfigurationIsValid();
```

#### 循环依赖解决

延迟注入

```csharp
  builder.Register(c => new ServiceA(c.Resolve<Lazy<IServiceB>>()));
```

```csharp
builder.Register(c => new ServiceA(c.Resolve<Func<IServiceB>>()))
       .As<IServiceA>();
builder.Register(c => new ServiceB(c.Resolve<IServiceA>()))
       .As<IServiceB>();
```

```csharp
builder.RegisterType<ServiceA>().As<IService>()
       .OnlyIf(reg => Environment.IsDevelopment);
```

#### 混合使用内置 DI 和 Autofac

```csharp
builder.Services.AddScoped<IOtherService, OtherService>(); // 内置 DI
builder.Host.ConfigureContainer<ContainerBuilder>(container =>
{
    container.RegisterType<UserService>().As<IUserService>();
});
```

### 与 ASP.NET Core 集成

在 `Program.cs` 中替换容器

```csharp
// Program.cs
using Autofac.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// 使用 Autofac 替换默认 DI 容器
builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());

// 配置 Autofac 服务
builder.Host.ConfigureContainer<ContainerBuilder>(containerBuilder => {
    containerBuilder.RegisterType<CustomService>().As<IService>();
    containerBuilder.RegisterModule<MyModule>();
});

var app = builder.Build();
```

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .UseServiceProviderFactory(new AutofacServiceProviderFactory())
        .ConfigureContainer<ContainerBuilder>(builder =>
        {
            builder.RegisterModule<RepositoryModule>();
            //… 其他注册
        })
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```

### 在控制台应用中使用

```csharp
var builder = new ContainerBuilder();
builder.RegisterModule<ApplicationModule>();
var container = builder.Build();

using (var scope = container.BeginLifetimeScope()) {
    var service = scope.Resolve<IMyService>();
    service.Run();
}
```

### 最佳实践与常见问题

#### 避免服务定位器反模式

```csharp
// 反模式：直接从容器解析服务
var service = container.Resolve<IMyService>();

// 正确做法：通过构造函数注入
public class MyConsumer {
    private readonly IMyService _service;
    public MyConsumer(IMyService service) { _service = service; }
}
```