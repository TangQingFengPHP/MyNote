### 简介

`log4net` 是 `.NET` 平台上非常成熟的日志组件，源自 `Java` 世界的 `log4j`。它功能丰富、性能高、配置灵活，是企业应用中常见的日志框架之一。

**核心特点**

* 支持多种 输出目标（`Appender`）：文件、数据库、控制台、远程服务等

* 支持多种 格式化（`Layout`）

* 支持 按级别（`Level`）记录日志

* 支持 日志分类（`Logger` 分组、命名空间隔离）

* 配置灵活，可通过 `XML` 文件配置，也可通过代码配置

* 支持 异步日志、按文件大小/时间分割

**核心组件**

* `Logger`：日志记录入口，按层次结构组织（如 `Namespace.Class`）

* `Appender`：日志输出目标（文件、数据库、控制台等）

* `Layout`：日志消息格式化方式

* `Filter`：日志过滤规则



### 安装 log4net

```shell
dotnet add package log4net
```

### 基础使用

#### 添加配置（App.config / Web.config / log4net.config）

```xml
<configuration>
  <configSections>
    <section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net"/>
  </configSections>

  <log4net>
    <appender name="RollingFileAppender" type="log4net.Appender.RollingFileAppender">
      <file value="logs/app.log" />
      <appendToFile value="true" />
      <rollingStyle value="Date" />
      <datePattern value="yyyyMMdd'.log'" />
      <staticLogFileName value="false" />
      <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="%date [%thread] %-5level %logger - %message%newline" />
      </layout>
    </appender>

    <root>
      <level value="DEBUG" />
      <appender-ref ref="RollingFileAppender" />
    </root>
  </log4net>
</configuration>
```

#### 在入口类中初始化 log4net

```csharp
[assembly: log4net.Config.XmlConfigurator(Watch = true)]

class Program
{
    private static readonly log4net.ILog log = log4net.LogManager.GetLogger(typeof(Program));

    static void Main(string[] args)
    {
        log.Info("程序启动");
        log.Debug("调试信息");
        log.Warn("警告信息");
        log.Error("错误信息");
        log.Fatal("严重错误");
    }
}
```

#### 基本日志记录

```csharp
log.Debug("调试信息");
log.Info("用户登录: " + username);
log.Warn("磁盘空间不足");
log.Error("数据库连接失败", ex); // 包含异常
log.Fatal("系统崩溃", fatalException);
```

#### 格式化日志

```csharp
// 字符串格式化
log.InfoFormat("用户 {0} 从 {1} 登录", username, ipAddress);

// 延迟计算（避免不必要的字符串拼接）
log.Debug(() => $"复杂计算: {ComputeExpensiveValue()}");
```

#### 上下文信息记录

```csharp
// 设置线程上下文
log4net.ThreadContext.Properties["User"] = currentUser;
log4net.ThreadContext.Properties["RequestId"] = Guid.NewGuid();

// 配置中使用
<layout type="log4net.Layout.PatternLayout">
  <conversionPattern value="%date [%thread] [%property{User}] [%property{RequestId}] %message%newline"/>
</layout>
```

### 常见 Appender 类型

#### RollingFileAppender（常用）

按文件大小或日期分割文件：

```xml
<appender name="RollingFile" type="log4net.Appender.RollingFileAppender">
  <file value="logs/app.log"/>
  <appendToFile value="true"/>
  <rollingStyle value="Composite"/> <!-- 按大小和时间滚动 -->
  <maxSizeRollBackups value="10"/>  <!-- 保留10个备份 -->
  <maximumFileSize value="10MB"/>   <!-- 单个文件最大10MB -->
  <staticLogFileName value="true"/>
  <datePattern value=".yyyyMMdd"/>  <!-- 日期格式后缀 -->
  <layout type="log4net.Layout.PatternLayout">
    <conversionPattern value="%date [%thread] %-5level %logger - %message%newline"/>
  </layout>
</appender>
```

#### ConsoleAppender

输出到控制台：

```xml
<appender name="ConsoleAppender" type="log4net.Appender.ConsoleAppender">
  <layout type="log4net.Layout.PatternLayout">
    <conversionPattern value="%date %-5level %logger - %message%newline" />
  </layout>
</appender>
```

#### AdoNetAppender

将日志写入数据库（如 `SQL Server`）：

```xml
<appender name="AdoNetAppender" type="log4net.Appender.AdoNetAppender">
  <bufferSize value="1" />
  <connectionType value="System.Data.SqlClient.SqlConnection, System.Data" />
  <connectionString value="Server=localhost;Database=Logs;Trusted_Connection=True;" />
  <commandText value="INSERT INTO Log ([Date],[Level],[Logger],[Message]) VALUES (@log_date, @log_level, @logger, @message)" />

  <parameter>
    <parameterName value="@log_date" />
    <dbType value="DateTime" />
    <layout type="log4net.Layout.RawTimeStampLayout" />
  </parameter>
  <parameter>
    <parameterName value="@log_level" />
    <dbType value="String" />
    <size value="50" />
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="%level" />
    </layout>
  </parameter>
  <parameter>
    <parameterName value="@logger" />
    <dbType value="String" />
    <size value="255" />
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="%logger" />
    </layout>
  </parameter>
  <parameter>
    <parameterName value="@message" />
    <dbType value="String" />
    <size value="4000" />
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="%message" />
    </layout>
  </parameter>
</appender>
```

#### 日志过滤 (Filter)

```xml
<appender name="ErrorFileAppender" type="log4net.Appender.FileAppender">
  <file value="errors.log"/>
  <filter type="log4net.Filter.LevelRangeFilter">
    <levelMin value="ERROR"/>
    <levelMax value="FATAL"/>
  </filter>
  <layout>...</layout>
</appender>
```

### 常用日志等级

| Level   | 说明                 |
| ------- | -------------------- |
| `ALL`   | 所有级别（最低）     |
| `DEBUG` | 调试信息             |
| `INFO`  | 普通信息             |
| `WARN`  | 警告                 |
| `ERROR` | 错误                 |
| `FATAL` | 致命错误（崩溃）     |
| `OFF`   | 关闭所有日志（最高） |

可以根据不同 `logger` 设置不同级别：

```xml
<logger name="MyApp.Controllers.HomeController">
  <level value="ERROR"/>
</logger>
```

### 高级特性

#### 自定义 PatternLayout

```csharp
public class UserInfoPatternConverter : PatternLayoutConverter
{
    protected override void Convert(TextWriter writer, LoggingEvent loggingEvent)
    {
        var user = ThreadContext.Properties["User"] as string;
        writer.Write(user ?? "Anonymous");
    }
}

// 注册自定义转换器
<layout type="log4net.Layout.PatternLayout">
  <converter>
    <name value="user"/>
    <type value="YourNamespace.UserInfoPatternConverter"/>
  </converter>
  <conversionPattern value="%date [%user] %message%newline"/>
</layout>
```

#### 日志审计跟踪

```csharp
public class AuditLogger
{
    private static readonly ILog auditLog = LogManager.GetLogger("AuditLogger");
    
    public void LogAction(string action, string target)
    {
        auditLog.InfoFormat("{0} 执行了 {1} 操作，对象: {2}", 
            GetCurrentUser(), action, target);
    }
}

// 配置中单独设置审计日志
<logger name="AuditLogger" additivity="false">
  <level value="INFO"/>
  <appender-ref ref="AuditFileAppender"/>
</logger>
```

#### 动态配置切换

```csharp
// 运行时更改日志级别
var logger = (log4net.Repository.Hierarchy.Logger)LogManager.GetLogger("root").Logger;
logger.Level = Level.Debug;

// 重新加载配置文件
log4net.Config.XmlConfigurator.Configure(new FileInfo("new_config.xml"));
```

#### 配置管理策略

```xml
<!-- 多环境配置示例 -->
<log4net>
  <!-- 开发环境 -->
  <root>
    <level value="DEBUG"/>
    <appender-ref ref="ConsoleAppender"/>
  </root>
  
  <!-- 生产环境（通过条件编译） -->
  <root condition="!DEBUG">
    <level value="INFO"/>
    <appender-ref ref="RollingFileAppender"/>
    <appender-ref ref="EmailAppender"/>
  </root>
</log4net>
```

#### 全局异常处理

```csharp
// ASP.NET MVC
protected void Application_Error()
{
    var exception = Server.GetLastError();
    log.Fatal("未处理的应用程序异常", exception);
}

// .NET Core 中间件
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionHandler = context.Features.Get<IExceptionHandlerFeature>();
        log.Fatal("未处理的请求异常", exceptionHandler.Error);
        await context.Response.WriteAsync("内部错误");
    });
});
```

#### 请求跟踪中间件

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILog _log = LogManager.GetLogger(typeof(RequestLoggingMiddleware));

    public RequestLoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task Invoke(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        await _next(context);
        sw.Stop();

        _log.InfoFormat("{0} {1} 响应 {2} 耗时 {3}ms",
            context.Request.Method,
            context.Request.Path,
            context.Response.StatusCode,
            sw.ElapsedMilliseconds);
    }
}
```

### 性能优化

#### 启用缓冲：减少 I/O 操作频率

```xml
<appender name="BufferedAppender" type="log4net.Appender.BufferingForwardingAppender">
  <bufferSize value="100"/>
  <appender-ref ref="FileAppender"/>
</appender>
```

#### 使用异步记录器（需要 log4net.Ext.Json）

```xml
<appender name="AsyncAppender" type="log4net.Appender.AsyncAppender">
  <appender-ref ref="FileAppender"/>
</appender>
```

#### 避免不必要的日志：

```csharp
// 使用 IsXXXEnabled 检查
if (log.IsDebugEnabled)
{
    log.Debug(ComputeExpensiveDebugInfo());
}
```

### 不同环境中的集成

#### ASP.NET Core 集成

```csharp
// Program.cs
using Microsoft.Extensions.Logging;
using log4net.Extensions.Logging;

var builder = WebApplication.CreateBuilder(args);

// 配置 log4net
builder.Logging.AddLog4Net(new Log4NetProviderOptions {
    Log4NetConfigFile = "log4net.config"
});

// 其他配置...
```

#### 控制台应用集成

```csharp
// 手动加载配置
XmlConfigurator.Configure(new FileInfo("log4net.config"));
```

#### 依赖注入集成

```csharp
// 在 Startup.cs 中注册
services.AddLogging(loggingBuilder => {
    loggingBuilder.AddLog4Net();
});
```