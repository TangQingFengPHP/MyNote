### 简介

`Ninject` 是一个轻量级、易扩展的开源 `.NET` 依赖注入（`DI`）容器，适用于 `.NET Framework、.NET Core、Xamarin` 等多平台。

设计目标：简单直观、可测试、高可扩展性，支持多种绑定策略和拦截器（`AOP`）。

### 核心组件

| 组件          | 作用                                                             |
| ------------- | ---------------------------------------------------------------- |
| `IKernel`     | 容器核心接口，负责所有绑定和解析；通常使用 `StandardKernel` 实例 |
| `Module`      | 将相关绑定逻辑封装，按需加载                                     |
| `Binding`     | 描述接口到实现的映射规则                                         |
| `Scope`       | 绑定的生命周期管理（瞬时、单例、自定义作用域）                   |
| `Interceptor` | 基于 `Castle.DynamicProxy` 实现 AOP 拦截功能                     |

### 基础使用

```csharp
// 1. 创建 Kernel（容器）
IKernel kernel = new StandardKernel();

// 2. 绑定（Binding）：接口 → 实现
kernel.Bind<IUserRepository>().To<UserRepository>();

// 3. 解析
var repo = kernel.Get<IUserRepository>();
```

### 绑定方式（Binding）详解

| 绑定方法                                   | 含义                                             |
| ------------------------------------------ | ------------------------------------------------ |
| `Bind<T>().To<U>()`                        | 接口 `T` → 实现 `U`                              |
| `Bind<T>().ToConstant(instance)`           | 始终返回同一个预创建实例                         |
| `Bind<T>().ToMethod(ctx => …)`             | 利用工厂方法动态构造实例                         |
| `Bind<T>().ToProvider<MyProv>()`           | 自定义 `IProvider<T>` 负责实例化                 |
| `Bind(typeof(IRepo<>)).To(typeof(Repo<>))` | 开放泛型绑定，让所有 `IRepo<T>` 解析为 `Repo<T>` |

**不同生命周期的注册**

```csharp
// 单例模式（全局唯一实例）
kernel.Bind<IDatabase>().To<SqlDatabase>().InSingletonScope();

// 每次请求创建新实例（默认行为）
kernel.Bind<IService>().To<ServiceImplementation>();

// 每个线程唯一实例
kernel.Bind<IRequestHandler>().To<RequestHandler>().InThreadScope();
```

### 生命周期（Scope）管理

| 作用域                | Ninject 表达                 | 说明                                     |
| --------------------- | ---------------------------- | ---------------------------------------- |
| **瞬时**              | `.InTransientScope()` 默认（无额外配置）           | 每次解析都创建新实例                     |
| **单例**              | `.InSingletonScope()`        | 全局唯一一个实例                         |
| **线程作用域**        | `.InThreadScope()`           | 每线程一个实例                           |
| **请求作用域（Web）** | `.InRequestScope()`          | 在 Web 请求生命周期内重用（需 Web 扩展） |
| **自定义作用域**      | `.InScope(ctx => /* key */)` | 按用户定义的 key 或上下文决定复用粒度    |

```csharp
kernel.Bind<ICache>().To<MemoryCache>().InSingletonScope();
kernel.Bind<IUnitOfWork>().To<UnitOfWork>().InRequestScope();
```

### 注入方式

#### 构造函数注入（Constructor Injection）

```csharp
public class UserService : IUserService
{
    private readonly IUserRepository _repo;
    public UserService(IUserRepository repo)
    {
        _repo = repo;
    }
}
```

`Ninject` 默认使用构造函数注入，自动解析所需的所有参数。

#### 属性注入（Property Injection）

```csharp
public class OrderController
{
    [Inject]
    public IOrderService OrderService { get; set; }
}
// 绑定配置需启用属性注入
kernel.Bind<OrderController>().ToSelf().WithPropertyValue(nameof(OrderController.OrderService));
kernel.Bind<OrderController>().ToSelf().WithPropertyInjection();
```

在需要的位置加 `[Inject]` 特性，`Ninject` 会为公有属性赋值。

#### 方法注入（Method Injection）

```csharp
public class ReportGenerator
{
    [Inject]
    public void Initialize(ILogger logger)
    {
        _logger = logger;
    }
}
```

支持对任意公有方法注入，只要打上 `[Inject]`。

### 模块化（Module）

将绑定逻辑组织到继承 `NinjectModule` 的模块类里，便于拆分和复用。

```csharp
public class RepositoryModule : NinjectModule
{
    public override void Load()
    {
        Bind<IUserRepository>().To<UserRepository>().InRequestScope();
        Bind<IOrderRepository>().To<OrderRepository>().InRequestScope();
    }
}

// 加载模块
IKernel kernel = new StandardKernel(new RepositoryModule(), new ServiceModule());
// 获取服务
var userService = kernel.Get<IUserService>();
```

### 拦截器与 AOP

通过 `Ninject.Extensions.Interception` 与 `Castle.DynamicProxy`，可以在方法调用前后执行横切逻辑。

```csharp
// 1. 安装扩展包： Ninject.Extensions.Interception
// 2. 定义拦截器
public class CallLogger : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
        Console.WriteLine($"Calling {invocation.Request.Method.Name}");
        invocation.Proceed();
        Console.WriteLine($"Done {invocation.Request.Method.Name}");
    }
}

// 3. 绑定并启用拦截
kernel.Bind<IService>().To<Service>()
      .Intercept().With<CallLogger>();
```

### 高级特性

#### 条件绑定

根据上下文动态选择实现：

```csharp
// 仅当注入ConsumerA时使用
kernel.Bind<IService>().To<ServiceA>().WhenInjectedInto<ConsumerA>();
kernel.Bind<IService>().To<ServiceB>().When(request => request.Target.Type.IsDefined(typeof(SpecialAttribute)));
```

#### 命名绑定

解决同一接口多实现的歧义：

```csharp
kernel.Bind<IService>().To<ServiceImpl>().Named("Default");
kernel.Bind<IService>().To<BackupService>().Named("Backup");
IService service = kernel.Get<IService>("Backup"); // 按名称解析
```

#### 工厂模式

动态创建实例：

```csharp
public interface IServiceFactory 
{
    IService Create(bool usePremium);
}

public class ServiceFactory : IServiceFactory 
{
    private readonly IKernel _kernel;
    
    public ServiceFactory(IKernel kernel) 
    {
        _kernel = kernel;
    }
    
    public IService Create(bool usePremium) 
    {
        return usePremium 
            ? _kernel.Get<PremiumService>() 
            : _kernel.Get<StandardService>();
    }
}
```

#### 带参数的绑定

```csharp
// 使用常量参数
kernel.Bind<IConfig>()
      .To<AppConfig>()
      .WithConstructorArgument("connectionString", "Server=localhost;Database=AppDb");

// 使用委托提供参数
kernel.Bind<IMailService>()
      .To<MailService>()
      .WithConstructorArgument("smtpServer", ctx => ConfigurationManager.AppSettings["SmtpServer"]);
```

#### 缓存 Kernel 实例

```csharp
// 应用生命周期内保持单例
public static class Container 
{
    public static IKernel Kernel { get; } = new StandardKernel();
}
```

#### 作用域嵌套

```csharp
using (var childScope = kernel.BeginBlock()) 
{
    var service = childScope.Get<IService>(); // 子作用域内实例
} // 自动释放实现了 `IDisposable` 的对象
```

#### 多实现解析

```csharp
Bind<INotifier>().To<EmailNotifier>();
Bind<INotifier>().To<SmsNotifier>();

// 解析所有实现
var notifiers = kernel.GetAll<INotifier>(); // IEnumerable<INotifier>

kernel.Bind<IUserService>().To<UserService1>();
kernel.Bind<IUserService>().To<UserService2>();

public class MultiService
{
    private readonly IEnumerable<IUserService> _services;

    public MultiService(IEnumerable<IUserService> services)
    {
        _services = services;
    }
}
```

### 与 ASP.NET Core / ASP.NET MVC 集成

#### ASP.NET MVC 5

使用 `Ninject.Web.Mvc` 扩展，在 `Global.asax` 中配置：

```csharp
var kernel = new StandardKernel();
kernel.Load(Assembly.GetExecutingAssembly());
DependencyResolver.SetResolver(new NinjectDependencyResolver(kernel));
```

#### ASP.NET Core

```csharp
Host.CreateDefaultBuilder(args)
    .UseNinject((context, services) =>
    {
        services.AddControllers();
        // 在此通过 kernel 进行 Ninject 绑定
    })
    .ConfigureWebHostDefaults(webBuilder => webBuilder.UseStartup<Startup>());
```

```csharp
// Program.cs
using Ninject;
using Ninject.Web.AspNetCore;

var builder = WebApplication.CreateBuilder(args);

// 替换默认服务提供者工厂
builder.Host.UseServiceProviderFactory(new NinjectServiceProviderFactory());

// 配置 Ninject 模块
builder.Host.ConfigureContainer<IKernel>(kernel => {
    kernel.Load<ApplicationModule>();
});

var app = builder.Build();
```

### 调试与问题排查

#### 查看绑定关系

```csharp
foreach (var binding in kernel.GetBindings(typeof(ILogger))) 
{
    Console.WriteLine($"Target: {binding.Target}");
}
```