### 简介

`NLog` 是 `.NET` 平台上最流行的开源日志框架之一，特色是 灵活的配置、丰富的输出目标（`Target`），以及 高性能 的异步写入能力。

适用场景：从控制台、文件、数据库、网络 到 `ElasticSearch、Seq、Azure Table Storage` 等各种日志收集后端。

支持文件、数据库（`SQL/NoSQL`）、控制台、邮件、`Elasticsearch` 等 50+ 内置目标，并可通过插件扩展

原生兼容 `JSON` 格式，可输出带上下文信息的结构化日志，便于 `ELK` 等系统分析

**性能优势**

* 异步日志：默认异步写入，不影响主线程

* 缓冲机制：批量写入减少 I/O 操作

* 低开销：轻量级设计，最小化性能影响

### 安装

#### 安装核心包

```shell
dotnet add package NLog
```

#### 安装扩展包以集成 Microsoft.Extensions.Logging

```shell
dotnet add package NLog.Extensions.Logging
dotnet add package NLog.Web.AspNetCore
```

### NLog.config 文件

在项目根目录添加一个 `NLog.config`（或 `nlog.config`）`XML` 文件，IDE 会自动识别并在运行时加载

```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <!-- 全局设置 -->
  <extensions>
    <!-- 扩展目标或布局，如：Seq、GELF 等 -->
  </extensions>

  <!-- 输出目标（Target） -->
  <targets>
    <!-- 控制台 -->
    <target xsi:type="Console" name="console" layout="${longdate}|${level:uppercase=true}|${logger}|${message}${onexception:inner=${newline}${exception:format=toString}}" />

    <!-- 文件 -->
    <target xsi:type="File"    name="file"
            fileName="logs/app.${shortdate}.log"
            layout="${longdate}|${level}|${logger}|${message}${onexception:${newline}${exception:format=toString}}" 
            archiveEvery="Day"
            maxArchiveFiles="7"
            concurrentWrites="true"
            keepFileOpen="false" />
  </targets>

  <!-- 日志规则（Rule）：决定哪些 Logger 名称、哪些级别，写入哪些目标 -->
  <rules>
    <!-- 所有来自 “*” 的日志 ≥ Info  写到控制台 -->
    <logger name="*" minlevel="Info" writeTo="console" />
    <!-- 所有日志 ≥ Debug 写到文件 -->
    <logger name="*" minlevel="Debug" writeTo="file" />
  </rules>
</nlog>
```

### appsettings.json 配置方案

```json
{
  "NLog": {
    "targets": {
      "logfile": {
        "type": "File",
        "fileName": "logs/${shortdate}.log"
      }
    },
    "rules": [
      {
        "logger": "*",
        "minLevel": "Debug",
        "writeTo": "logfile"
      }
    ]
  }
}
```

### C# 代码配置

```csharp
var config = new LoggingConfiguration();

// 文件目标（按类名分目录）
var fileTarget = new FileTarget("timeLevelClassFile") {
    FileName = "NLog/${shortdate}/${logger}.log",
    Layout = "${longdate} ${level:uppercase=true} ${logger} ${message}",
    ArchiveEvery = FileArchivePeriod.Day,
    MaxArchiveDays = 30
};

// 异步包装提升性能
var asyncFileTarget = new AsyncTargetWrapper(fileTarget);

config.AddTarget(asyncFileTarget);
config.AddRule(LogLevel.Info, LogLevel.Fatal, asyncFileTarget);

// 应用配置
LogManager.Configuration = config;
```

### 核心组件

| 组件       | 作用                                                                                     |
| ---------- | ---------------------------------------------------------------------------------------- |
| **Logger** | 记录日志的入口，通常按类或命名空间获取：`var log = LogManager.GetCurrentClassLogger();`  |
| **Target** | 日志输出目标，如 `Console`, `File`, `Database`, `Mail`, `Network`, `Custom` 等           |
| **Layout** | 输出格式模板，可包含 `${longdate}`,`${level}`, `${message}`, `${exception}` 等内置渲染器 |
| **Rule**   | 日志规则，匹配 Logger 名称与级别后，将日志路由到一个或多个 Target                        |
| **Filter** | 规则的高级过滤器，精细控制哪些消息被忽略或处理                                           |

#### 在代码中使用

```csharp
using NLog;

public class MyService
{
    private static readonly ILogger Logger = LogManager.GetCurrentClassLogger();

    public void DoWork()
    {
        Logger.Trace("Trace message");
        Logger.Debug("Debug message");
        Logger.Info("Application started");
        
        try
        {
            // ...
            throw new InvalidOperationException("Oops");
        }
        catch (Exception ex)
        {
            Logger.Error(ex, "Unhandled exception in DoWork");
        }
    }
}
```

* Log Levels

`Trace < Debug < Info < Warn < Error < Fatal`

* 参数化日志

```csharp
Logger.Info("Processing {OrderId} for {Customer}", order.Id, customer.Name);
```

#### 常用布局渲染器

* `${longdate}`：完整日期时间

* `${level}`：日志级别

* `${logger}`：记录器名称

* `${message}`：日志消息

* `${exception}`：异常信息

* `${callsite}`：调用位置（类名+方法名）

* `${stacktrace}`：堆栈跟踪

* `${machinename}`：机器名

* `${processid}`：进程 ID

* `${threadid}`：线程 ID

* `${aspnet-request-url}`：ASP.NET 请求 URL

### 高级用法

#### 结构化日志

```csharp
Logger.Info("订单 {OrderId} 创建，用户 {UserId}", orderId, userId);
```

#### Mapped Diagnostics Context (MDC) / Mapped Diagnostics Logical Context (MDLC)

用于给日志记录中添加每个执行上下文的额外属性，比如请求 ID、用户 ID 等。

```csharp
using NLog.MappedDiagnosticsLogicalContext;

public void HandleRequest(HttpContext ctx)
{
    MDLC.Set("RequestId", ctx.TraceIdentifier);
    MDLC.Set("User", ctx.User?.Identity?.Name ?? "anonymous");

    Logger.Info("Handling request");
    // …
    MDLC.Remove("User");
}
```

在 Layout 中引用：

```xml
layout="${longdate}|${mdlc:item=RequestId}|${mdlc:item=User}|${message}"
```

#### 异步日志记录

```xml
<targets async="true">
  <target name="file" xsi:type="File" fileName="logs/${shortdate}.log" />
</targets>
```

#### 使用缓存

```xml
<target name="file" xsi:type="File">
  <layout cache="${longdate} ${message}" />
</target>
```

#### 异步与缓存

* `AsyncWrapper`：将任何目标包裹为异步写入，避免日志写入阻塞业务线程

```csharp
<target xsi:type="AsyncWrapper" name="asyncFile" overflowAction="Discard"
        timeToSleepBetweenBatches="50" batchSize="100">
  <target-wrapper>
    <target xsi:type="File" fileName="logs/async.log" layout="…"/>
  </target-wrapper>
</target>
```

* `BufferingWrapper`：按条数或时间批量写入，适合数据库、网络等慢目标。

```csharp
<target name="buffered" xsi:type="BufferingWrapper" bufferSize="100">
  <target xsi:type="File" fileName="buffered.log" />
</target>
```

#### 条件过滤

```xml
<rules>
  <!-- 仅记录包含特定关键字的日志 -->
  <logger name="*" writeTo="file">
    <filters>
      <when condition="contains(message, 'payment')" action="Log" />
    </filters>
  </logger>
</rules>
```

#### 文件归档策略

```xml
<target name="rollingFile" xsi:type="File"
        fileName="${basedir}/logs/app.log"
        archiveFileName="${basedir}/logs/archive/app.{#}.log"
        archiveEvery="Day"
        archiveNumbering="Rolling"
        maxArchiveFiles="30"
        layout="${longdate} ${message}" />
```

#### 自定义扩展

* 创建自定义 `Target`

```csharp
[Target("CustomTarget")]
public sealed class CustomTarget : TargetWithLayout
{
    protected override void Write(LogEventInfo logEvent)
    {
        string logMessage = RenderLogEvent(Layout, logEvent);
        // 自定义输出逻辑（如发送到第三方服务）
        SendToCustomService(logMessage);
    }
}
```

* 配置自定义 `Target`

```csharp
<targets>
  <target name="custom" xsi:type="CustomTarget" 
          layout="${message}" />
</targets>
```

#### 多环境配置

```xml
<!-- 开发环境 -->
<logger name="*" minlevel="Debug" writeTo="console,debug-file" />

<!-- 生产环境 -->
<logger name="*" minlevel="Info" writeTo="file" />
<logger name="Microsoft.*" minlevel="Warning" final="true" />
```

#### 配置变量

```xml
<variable name="logDirectory" value="${basedir}/logs" />
<target name="file" xsi:type="File" fileName="${logDirectory}/${shortdate}.log" />
```

#### 避免日志记录开销

```csharp
// 避免不必要的字符串拼接
if (Logger.IsDebugEnabled) {
    Logger.Debug("复杂计算结果: {0}", ExpensiveCalculation());
}

// 或使用结构化日志
Logger.Debug("用户 {UserName} 登录", userName);
```

#### 启用内部日志调试

```xml
<nlog internalLogFile="nlog-internal.log" internalLogLevel="Debug">
```

#### 按级别/命名空间过滤

```xml
<rules>
  <!-- Controllers 命名空间下只记录 Error+ -->
  <logger name="MyApp.Controllers.*" minlevel="Error" writeTo="file" />
  
  <!-- 全局记录 Info+ -->
  <logger name="*" minlevel="Info" writeTo="file" />
</rules>
```

#### 数据库日志写入

```xml
<target name="database" xsi:type="Database"
        connectionString="Server=.;Database=Logs;Trusted_Connection=True;"
        commandText="INSERT INTO Logs (Time, Level, Logger, Message) VALUES (@time, @level, @logger, @message)">
  <parameter name="@time" layout="${longdate}" />
  <parameter name="@level" layout="${level}" />
  <parameter name="@logger" layout="${logger}" />
  <parameter name="@message" layout="${message}" />
</target>
```

#### 结构化日志（Elasticsearch 集成）

```xml
<target name="elastic" xsi:type="ElasticSearch"
        uri="http://localhost:9200"
        index="applogs-${date:format=yyyy.MM}">
  <field name="message" layout="${message}" />
  <field name="level" layout="${level}" />
  <field name="exception" layout="${exception:format=toString}" />
</target>
```

#### 故障转移：FallbackGroup 目标（主目标失败时切备用）

```xml
<target name="failover" xsi:type="FallbackGroup">
  <target xsi:type="Database" connectionString="..." />
  <target xsi:type="File" fileName="fallback.log" /> <!--  -->
</target>
```

### 不同环境下集成

#### 配置 Program.cs

```csharp
// Program.cs
using NLog.Web;

var builder = WebApplication.CreateBuilder(args);

// 配置 NLog
builder.Logging.ClearProviders();
builder.Host.UseNLog();

// 其他配置...
```  

#### 配置 Startup.cs

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddLogging(loggingBuilder => {
        loggingBuilder.ClearProviders();
        loggingBuilder.AddNLog();
    });
}
```

#### 控制台应用集成

```csharp
// 手动加载配置
LogManager.LoadConfiguration("NLog.config");
```

#### 依赖注入集成

```csharp
// 在 Startup.cs 中注册
services.AddSingleton<ILogger>(LogManager.GetCurrentClassLogger());
```

#### 注入 ILogger

```csharp
public class HomeController : Controller
{
    private readonly ILogger<HomeController> _logger;

    public HomeController(ILogger<HomeController> logger)
    {
        _logger = logger;
    }

    public IActionResult Index()
    {
        _logger.LogInformation("访问首页");
        return View();
    }
}
```

#### 中间件记录请求

```csharp
app.Use(async (context, next) =>
{
    var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
    
    var sw = Stopwatch.StartNew();
    await next();
    sw.Stop();
    
    logger.LogInformation("请求 {Method} {Path} 耗时 {Elapsed}ms", 
        context.Request.Method, 
        context.Request.Path, 
        sw.ElapsedMilliseconds);
});
```

### AsyncWrapper VS async="true"

| 特性         | `AsyncWrapper`                                            | `async="true"`（简化配置）          |
| ------------ | --------------------------------------------------------- | ----------------------------------- |
| **写法**     | 显式使用 `<target xsi:type="AsyncWrapper">` 包裹          | 在普通 target 上设置 `async="true"` |
| **能力**     | 更强大、更灵活（支持缓冲大小、丢弃策略、包装多个 target） | 语法更简洁，适合简单异步场景        |
| **推荐使用** | ✅ 更推荐生产使用                                          | ✅ 快速开发或小型项目时可用          |


#### 背后机制

**`AsyncWrapper`**

`NLog` 提供的一个强大的异步包装器，适用于任何 `target`。它使用独立的后台线程池写日志，并允许配置行为细节

```xml
<target xsi:type="AsyncWrapper"
        name="asyncFile"
        overflowAction="Discard"
        batchSize="100"
        timeToSleepBetweenBatches="50">
  <target xsi:type="File"
          fileName="logs/app.log"
          layout="${longdate}|${level}|${message}" />
</target>
```

* `overflowAction`：当队列满时的行为（默认 `Block`, 可设为 `Discard` 或 `Grow`）

* `batchSize`：每次最多写多少条

* `timeToSleepBetweenBatches`：线程空闲等待时间

* 支持多个目标包裹（多个 `<target>`）

**async="true"**

是 `NLog` 针对简单异步写日志的简化语法，本质是自动为该 `target` 添加一个 `AsyncWrapper` 包裹层

```xml
<target xsi:type="File" name="file" async="true"
        fileName="logs/app.log"
        layout="${longdate}|${level}|${message}" />
```

相当于 `NLog` 内部自动做了：

```xml
<target xsi:type="AsyncWrapper" name="file">
  <target xsi:type="File" ... />
</target>
```

但它不支持配置 `batchSize、overflowAction` 等高级参数。

使用场景建议：

| 场景                                        | 推荐方式       | 原因说明                                                            |
| ------------------------------------------- | -------------- | ------------------------------------------------------------------- |
| ✅ **中小型项目，日志量不大**                | `async="true"` | 简洁、易用，性能也足够                                              |
| ✅ **生产环境、日志量大（高并发写日志）**    | `AsyncWrapper` | 提高性能、可配置批处理大小、溢出策略等，适合高负载                  |
| ❌ 需要**多个目标异步写入**（如控制台+文件） | `AsyncWrapper` | 一个 `AsyncWrapper` 可包裹多个目标，`async="true"` 无法组合多个目标 |
| ❌ 需要控制日志队列行为                      | `AsyncWrapper` | 可配置缓冲区行为：Discard / Block / Grow 等                         |
| ✅ 快速调试开发                              | `async="true"` | 配置文件更简洁、清晰                                                |

### 终极文件目标配置

```xml
<?xml version="1.0" encoding="utf-8"?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <!-- 全局设置 -->
  <targets>

    <!-- 异步包装器：所有 File 写入都经过异步队列 -->
    <target xsi:type="AsyncWrapper" name="asyncFile"
            overflowAction="Discard"
            batchSize="200"
            timeToSleepBetweenBatches="50">

      <!-- 真正的文件目标 -->
      <target xsi:type="File"
              name="dailyRollingFile"
              fileName="logs/app.${shortdate}.log"
              layout="${longdate}|${level:uppercase=true}|${logger}|${message}${onexception:${newline}${exception:format=toString}}"

              <!-- 每天归档 -->
              archiveEvery="Day"
              
              <!-- 大小到 10MB 时滚动归档 -->
              archiveAboveSize="10485760"            

              <!-- 归档文件命名：app.2025-06-24.0.log, app.2025-06-24.1.log, ... -->
              archiveNumbering="Rolling"

              <!-- 归档文件名称模板（可选，默认已自动添加编号） -->
              archiveFileName="logs/archives/app.${shortdate}.{#}.log"

              <!-- 保留最近 7 个归档文件 -->
              maxArchiveFiles="7"

              <!-- 并发写入设置 -->
              concurrentWrites="true"
              keepFileOpen="false"
              encoding="utf-8"
              />

    </target>

    <!-- 也可以单独给控制台输出异步 -->
    <target xsi:type="AsyncWrapper" name="asyncConsole">
      <target xsi:type="Console" 
              layout="${longdate}|${level}|${message}" />
    </target>
    
  </targets>

  <rules>
    <!-- 所有日志 >= Info 写到控制台 -->
    <logger name="*" minlevel="Info" writeTo="asyncConsole" />

    <!-- 所有日志 >= Debug 写到文件 -->
    <logger name="*" minlevel="Debug" writeTo="asyncFile" />
  </rules>
</nlog>
```

配置说明：

* `fileName="logs/app.${shortdate}.log"`
每天都会以 app.YYYY-MM-DD.log 的文件名写入新日志。

* `archiveEvery="Day"`
在每个日历天的开头，都会对前一天的文件进行归档（配合 archiveNumbering 控制编号方式）。

* `archiveAboveSize="10485760"`
当当天的当前日志文件大小超过 10 MB（10×1024×1024 bytes）时，会自动把它归档并新建一个同名空文件继续写日志。

* `archiveNumbering="Rolling"`
使用滚动编号方式：第一个归档为 .0.log，第二个为 .1.log，以此类推。

* `archiveFileName="logs/archives/app.${shortdate}.{#}.log"`
将归档文件集中到 logs/archives/ 目录下，便于管理。{#} 会被替换成自增的归档号。

* `maxArchiveFiles="7"`
最多保留最近 7 个归档文件，超过则自动删除最旧的。

* `AsyncWrapper`

    * `overflowAction="Discard"`：队列满时丢弃最旧的日志，避免阻塞；

    * `batchSize="200"`：每批写入 200 条；

    * `timeToSleepBetweenBatches="50"`：批量写入后等待 50 ms。

* `concurrentWrites="true" & keepFileOpen="false"`
在并发写入和文件锁管理上更稳健，适合多线程/多进程场景。


