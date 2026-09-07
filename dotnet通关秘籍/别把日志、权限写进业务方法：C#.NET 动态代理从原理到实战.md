### 简介

业务方法里经常混着这些代码：记录日志、检查权限、统计耗时、开启事务、读写缓存、捕获异常。

```csharp
public Order CreateOrder(CreateOrderRequest request)
{
    logger.LogInformation("开始创建订单");
    CheckPermission();
    var stopwatch = Stopwatch.StartNew();

    // 真正的业务代码
    var order = orderRepository.Create(request);

    logger.LogInformation("创建订单完成，耗时 {Elapsed}ms", stopwatch.ElapsedMilliseconds);
    return order;
}
```

方法少时还能接受，方法一多，横切逻辑就会到处复制。修改日志格式要改很多处，漏掉一次权限检查也很危险。

动态代理提供了一层运行时生成的“外壳”：调用方拿到的仍然是原来的接口或类型，方法调用却会先经过代理和拦截器，再进入真实对象。

```text
调用方
  ↓
代理对象
  ↓
日志 → 权限 → 计时
  ↓
真实对象
```

动态代理是 AOP（面向切面编程）常见的实现方式，但它不是万能的装饰器。代理能拦截什么，取决于所使用的实现方案以及目标方法的可重写性。

本文使用 .NET 自带的 `DispatchProxy` 和常见的 `Castle.DynamicProxy`，通过完整代码说明动态代理到底做了什么、如何选型，以及哪些代码不会被拦截。

### 动态代理到底解决了什么问题

先看手写的静态代理：

```csharp
public interface IUserService
{
    string GetName(int id);
}

public sealed class UserServiceProxy : IUserService
{
    private readonly IUserService target;

    public UserServiceProxy(IUserService target)
    {
        this.target = target;
    }

    public string GetName(int id)
    {
        Console.WriteLine("调用开始");
        var result = target.GetName(id);
        Console.WriteLine("调用结束");
        return result;
    }
}
```

这个代理完全没有问题，只是每个接口、每个方法都要手写一遍。动态代理把“生成代理类型”和“拦截调用”的工作交给运行时或库完成：

```text
静态代理：开发阶段写好代理类，编译成程序集
动态代理：运行时生成代理类型，把调用转发到拦截器
```

拦截器一般负责四类动作：

* 调用前做检查，例如权限、参数校验；
* 调用真实方法，例如 `Proceed()`；
* 调用成功后处理结果，例如记录耗时、写缓存；
* 调用失败时记录异常、重试或转换异常。

### .NET 动态代理方案怎么选

| 方案 | 特点 | 适合场景 |
| --- | --- | --- |
| `DispatchProxy` | .NET 自带，主要用于接口代理 | 轻量日志、客户端封装、简单 AOP |
| Castle DynamicProxy | 支持接口代理和类代理，拦截器链成熟 | 业务项目、框架集成、复杂 AOP |
| `Reflection.Emit` | 直接生成动态程序集和 IL | 框架开发、特殊性能或运行时生成需求 |
| 源生成器 / IL Weaving | 编译期生成或改写代码 | 追求可分析性、减少运行时代理开销 |

大多数业务代码只需要掌握前两种：接口设计比较规整时使用 `DispatchProxy`；需要代理具体类、多个拦截器或接入现成 IoC 方案时使用 Castle DynamicProxy。

### 方案一：使用 DispatchProxy 实现接口代理

`DispatchProxy` 位于 `System.Reflection` 命名空间，不需要安装第三方 NuGet 包。它的思路很直接：运行时生成一个实现接口的代理对象，每次接口方法调用都会回调 `Invoke`。

#### 创建一个可复用的日志代理

下面的 Demo 可以直接放进 .NET 8 或更高版本的控制台项目中运行：

```csharp
using System.Reflection;
using System.Runtime.ExceptionServices;

public interface IUserService
{
    string GetName(int id);
    void Rename(int id, string name);
}

public sealed class UserService : IUserService
{
    public string GetName(int id)
    {
        Console.WriteLine($"查询用户：{id}");
        return $"User-{id}";
    }

    public void Rename(int id, string name)
    {
        Console.WriteLine($"修改用户 {id} 的名称为：{name}");
    }
}

public sealed class LoggingDispatchProxy<T> : DispatchProxy where T : class
{
    private T? target;

    public static T Create(T target)
    {
        ArgumentNullException.ThrowIfNull(target);

        var proxy = Create<T, LoggingDispatchProxy<T>>();
        ((LoggingDispatchProxy<T>)(object)proxy).target = target;
        return proxy;
    }

    protected override object? Invoke(MethodInfo? targetMethod, object?[]? args)
    {
        if (targetMethod is null || target is null)
        {
            throw new InvalidOperationException("代理尚未初始化");
        }

        var arguments = args ?? Array.Empty<object?>();
        Console.WriteLine($"[开始] {targetMethod.Name}({string.Join(", ", arguments)})");

        try
        {
            var result = targetMethod.Invoke(target, args);
            Console.WriteLine($"[完成] {targetMethod.Name}，返回值：{result ?? "null"}");
            return result;
        }
        catch (TargetInvocationException ex) when (ex.InnerException is not null)
        {
            // MethodInfo.Invoke 会把真实异常包在 TargetInvocationException 中。
            Console.WriteLine($"[异常] {targetMethod.Name}：{ex.InnerException.Message}");
            ExceptionDispatchInfo.Capture(ex.InnerException).Throw();
            throw; // 仅用于满足编译器的返回路径分析
        }
    }
}

var service = LoggingDispatchProxy<IUserService>.Create(new UserService());

Console.WriteLine(service.GetName(100));
service.Rename(100, "张三");
```

输出类似下面这样：

```text
[开始] GetName(100)
查询用户：100
[完成] GetName，返回值：User-100
User-100
[开始] Rename(100, 张三)
修改用户 100 的名称为：张三
[完成] Rename，返回值：null
```

这段代码里有三个关键点：

* `Create<T, TProxy>()` 负责生成实现 `T` 接口的代理对象；
* `Invoke` 是统一入口，方法名、参数和返回值都可以在这里处理；
* `targetMethod.Invoke(target, args)` 才是对真实对象的调用。

#### DispatchProxy 的边界

`DispatchProxy` 适合代理接口，不能把一个普通类直接变成代理类：

```csharp
public class ReportService
{
    public void Export() { }
}

// 不能写成 DispatchProxy<ReportService>，ReportService 不是接口
```

接口代理还有几个容易忽略的点：

* `Invoke` 是同步回调，返回 `Task` 的方法需要把真实 `Task` 原样返回；此时“完成日志”只代表拿到了 Task，不代表异步操作已经结束；
* `MethodInfo.Invoke` 会产生反射调用开销，不适合把极高频、极短的方法全部包在这里；
* 代理通常只在接口引用上生效，代码绕过接口直接调用真实对象时不会经过代理。

### 方案二：Castle DynamicProxy 实战

安装 `Castle.Core`：

```shell
dotnet add package Castle.Core
```

Castle DynamicProxy 使用 `ProxyGenerator` 生成代理，使用 `IInterceptor` 编写拦截逻辑，使用 `IInvocation` 读取当前调用的上下文。

#### 接口代理：日志、耗时和异常一次完成

```csharp
using Castle.DynamicProxy;
using System.Diagnostics;

public interface IOrderService
{
    void Place(string product, int quantity);
    decimal GetTotal(decimal price, int quantity);
}

public sealed class OrderService : IOrderService
{
    public void Place(string product, int quantity)
    {
        Console.WriteLine($"创建订单：{product} × {quantity}");
    }

    public decimal GetTotal(decimal price, int quantity)
    {
        return price * quantity;
    }
}

public sealed class LogAndTimingInterceptor : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
        var stopwatch = Stopwatch.StartNew();
        Console.WriteLine($"[开始] {invocation.Method.Name}");

        try
        {
            invocation.Proceed();
            Console.WriteLine($"[完成] {invocation.Method.Name}，耗时 {stopwatch.ElapsedMilliseconds}ms");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[失败] {invocation.Method.Name}：{ex.Message}");
            throw;
        }
    }
}

var generator = new ProxyGenerator();
var target = new OrderService();
var proxy = generator.CreateInterfaceProxyWithTarget<IOrderService>(
    target,
    new LogAndTimingInterceptor());

proxy.Place("机械键盘", 2);
Console.WriteLine($"订单金额：{proxy.GetTotal(399m, 2)}");
```

`IInvocation` 中常用的成员如下：

| 成员 | 作用 |
| --- | --- |
| `Method` | 当前被调用的方法 |
| `Arguments` | 方法参数，可读取，也可通过 `SetArgumentValue` 修改 |
| `ReturnValue` | 真实方法返回值，可读取或替换 |
| `Proceed()` | 继续执行下一个拦截器，最后到真实方法 |
| `Proxy` | 当前代理对象 |
| `InvocationTarget` | 当前真实目标对象 |

`Proceed()` 不调用，调用链就会在当前拦截器停止。这个特性可用来实现缓存命中、权限拒绝或短路返回。

#### 类代理：为什么方法必须是 virtual

类代理本质上是运行时生成一个继承目标类的子类，然后重写可重写的方法。因此目标类的方法必须是 `virtual`、`abstract` 或接口实现中可被代理的成员：

```csharp
public class PaymentService
{
    public virtual bool Pay(decimal amount)
    {
        Console.WriteLine($"实际支付：{amount}");
        return amount > 0;
    }

    public bool QueryBalance()
    {
        Console.WriteLine("查询余额");
        return true;
    }
}

var paymentProxy = generator.CreateClassProxy<PaymentService>(
    new LogAndTimingInterceptor());

paymentProxy.Pay(99m);       // 会经过拦截器
paymentProxy.QueryBalance(); // 不会经过拦截器，方法不可重写
```

类代理还要求目标类型能够被继承和实例化，所以 `sealed` 类、`static` 类、非 `virtual` 方法都不能按这种方式拦截。需要代理已有实例时，可以使用：

```csharp
var targetPayment = new PaymentService();
var proxyWithTarget = generator.CreateClassProxyWithTarget(
    targetPayment,
    new LogAndTimingInterceptor());
```

#### 多个拦截器和执行顺序

一个代理可以接收多个拦截器：

```csharp
public sealed class PermissionInterceptor : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
        if (invocation.Method.Name == nameof(IOrderService.Place))
        {
            // 实际项目中从当前请求上下文、Token 或权限服务读取
            var allowed = true;
            if (!allowed)
            {
                throw new UnauthorizedAccessException("没有下单权限");
            }
        }

        invocation.Proceed();
    }
}

var securedProxy = generator.CreateInterfaceProxyWithTarget<IOrderService>(
    new OrderService(),
    new PermissionInterceptor(),
    new LogAndTimingInterceptor());
```

执行顺序可以理解为一层层套娃：

```text
Permission.Before
  └─ Logging.Before
       └─ OrderService.Place
  └─ Logging.After
Permission.After
```

排列拦截器时，通常把权限、幂等、事务放在外层，把日志、耗时放在内层。这样被权限拒绝的请求也能记录下来，同时不会开启不必要的事务。

#### 修改参数和返回值

拦截器不仅能观察调用，还能在合适的时机改写参数或返回值：

```csharp
public sealed class PriceInterceptor : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
        if (invocation.Method.Name == nameof(IOrderService.GetTotal))
        {
            var price = (decimal)invocation.Arguments[0];
            invocation.SetArgumentValue(0, price * 0.9m);
        }

        invocation.Proceed();

        if (invocation.Method.Name == nameof(IOrderService.GetTotal))
        {
            Console.WriteLine($"折后金额：{invocation.ReturnValue}");
        }
    }
}
```

修改参数必须保证类型兼容；修改返回值也必须符合原方法声明的返回类型，否则会在运行时抛出异常。缓存拦截器则可以在命中时直接设置 `ReturnValue`，不调用 `Proceed()`。

### 异步方法拦截不能只套一层 try/finally

下面这种写法只能统计“创建 Task 的时间”，不能统计真正的异步执行时间：

```csharp
public void Intercept(IInvocation invocation)
{
    var stopwatch = Stopwatch.StartNew();
    invocation.Proceed();
    Console.WriteLine($"耗时：{stopwatch.ElapsedMilliseconds}ms");
}
```

对于 `Task` 或 `Task<T>`，`Proceed()` 通常很快就返回了，真正的工作在后面才完成。生产项目可使用专门的异步拦截器库；手写时至少要在返回的 Task 完成后再记录日志。一个只处理 `Task` 的简单示例：

```csharp
public sealed class AsyncTimingInterceptor : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
        invocation.Proceed();

        if (invocation.ReturnValue is Task task)
        {
            invocation.ReturnValue = MeasureAsync(task, invocation.Method.Name);
            return;
        }

        Console.WriteLine($"同步方法完成：{invocation.Method.Name}");
    }

    private static async Task MeasureAsync(Task task, string methodName)
    {
        var stopwatch = Stopwatch.StartNew();
        try
        {
            await task.ConfigureAwait(false);
        }
        finally
        {
            Console.WriteLine($"异步方法完成：{methodName}，耗时 {stopwatch.ElapsedMilliseconds}ms");
        }
    }
}
```

`Task<T>` 需要保留泛型结果，不能直接把 `Task<T>` 替换成 `Task`。通用实现要通过反射创建对应的 `Task<T>` 包装，或者使用 Castle 社区提供的异步拦截器扩展。异步拦截的重点是：`ReturnValue` 必须在代理方法返回之前设置为正确类型的 Task。

### 接入依赖注入：代理应该在哪里创建

手动 `new ProxyGenerator()` 适合 Demo。ASP.NET Core 项目通常在组合根创建一次代理生成器，再把代理注册到容器中：

```csharp
using Castle.DynamicProxy;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<ProxyGenerator>();
builder.Services.AddSingleton<LogAndTimingInterceptor>();
builder.Services.AddScoped<OrderService>();
builder.Services.AddScoped<IOrderService>(sp =>
{
    var generator = sp.GetRequiredService<ProxyGenerator>();
    var target = sp.GetRequiredService<OrderService>();
    var interceptor = sp.GetRequiredService<LogAndTimingInterceptor>();

    return generator.CreateInterfaceProxyWithTarget<IOrderService>(target, interceptor);
});
```

Castle 官方文档特别建议长生命周期进程复用同一个 `ProxyGenerator`，因为它会缓存生成的代理类型。每次解析服务都重新创建 `ProxyGenerator`，会带来额外 CPU 和内存开销。

### 动态代理和装饰器有什么区别

两者都能在调用前后加逻辑，但关注点不同：

```text
装饰器：手写一个明确的包装类，类型安全、依赖关系清楚
动态代理：运行时批量生成包装逻辑，适合通用横切关注点
```

业务规则、计费策略、缓存键规则等需要清晰表达的逻辑，适合装饰器或普通业务类。日志、耗时、权限、事务这类跨很多服务且结构相似的逻辑，适合拦截器。动态代理用得过多，会让调用链隐藏在容器配置里，排查问题时反而更困难。

### 常见错误

#### 只在目标类上加 virtual，却通过真实对象调用

```csharp
var real = new PaymentService();
real.Pay(10); // 没有代理，当然不会进入拦截器
```

必须把代理对象注入并传递给调用方，通常让调用方只依赖接口。

#### 忘记调用 Proceed

不调用 `Proceed()` 就不会进入后续拦截器和真实方法。只有缓存命中、权限拒绝、Mock 等明确需要短路时，才省略它。

#### 把异常吞掉

```csharp
catch (Exception ex)
{
    logger.LogError(ex, "调用失败");
    throw; // 保留原始堆栈
}
```

日志拦截器通常记录后继续抛出。写成 `throw ex` 会重置堆栈起点，排查问题更困难。

#### 忽略异步返回值

`Proceed()` 返回成功，只说明方法返回了一个 Task。异步操作是否成功，需要等待 Task 完成后才能判断。

#### 代理所有方法

代理会增加调用层级和一定运行时开销。优先代理服务边界，把数据库实体、简单 DTO、极高频小方法排除在外。

### 什么时候不该使用动态代理

以下场景优先考虑普通代码或装饰器：

* 只有一个类、一个方法需要加一段逻辑；
* 逻辑本身是业务规则，不能隐藏在通用拦截器里；
* 需要调试器中清晰展示完整调用关系；
* 方法极高频，对额外调用和反射开销非常敏感；
* 目标类型是 `sealed`，方法也不是 `virtual`，没有可拦截的入口。

### 总结

动态代理的核心不是“把类变魔法”，而是在调用方和真实对象之间插入一条可配置的拦截链。

```text
接口代理 → DispatchProxy 或 Castle DynamicProxy
类代理   → Castle DynamicProxy，目标方法通常必须 virtual
横切逻辑 → IInterceptor / Invoke
继续调用 → Proceed()
短路返回 → 设置 ReturnValue，不调用 Proceed()
```

轻量项目可以从 `DispatchProxy` 开始；需要类代理、多个拦截器和 IoC 集成时，选择 Castle DynamicProxy 更合适。无论采用哪种方案，都应明确代理边界、异步处理方式和拦截器顺序，避免把重要业务流程藏得过深。

参考资料：

* [Microsoft Learn：DispatchProxy](https://learn.microsoft.com/dotnet/api/system.reflection.dispatchproxy)
* [Castle DynamicProxy 官方简介](https://github.com/castleproject/Core/blob/master/docs/dynamicproxy-introduction.md)
* [Castle DynamicProxy 异步拦截说明](https://github.com/castleproject/Core/blob/master/docs/dynamicproxy-async-interception.md)
