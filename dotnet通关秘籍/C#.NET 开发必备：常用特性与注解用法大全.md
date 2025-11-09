### 特性基础

#### 什么是特性

特性是附加到代码元素（程序集、类型、成员、参数等）上的元数据。编译后写入 IL，可在运行时通过反射读取或由运行时/框架识别并做相应处理。

#### 定义特性

自定义特性需继承自 `System.Attribute`，并可通过 `AttributeUsage` 限制其作用目标和允许多重使用。

```csharp
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
public class MyCustomAttribute : Attribute
{
    public string Name { get; }
    public MyCustomAttribute(string name) => Name = name;
}
```

### CLR 内置通用特性

| 特性                    | 作用                                               | 示例                                                              |
| ----------------------- | -------------------------------------------------- | ----------------------------------------------------------------- |
| `[Obsolete]`            | 标记已废弃的 API，调用时报编译警告或错误           | `[Obsolete("Use NewMethod instead", true)] public void Old() { }` |
| `[Serializable]`        | 标记可通过二进制/SOAP 序列化                       | `[Serializable] public class Person { ... }`                      |
| `[NonSerialized]`       | 与 `[Serializable]` 配合使用，标记字段不参与序列化 | `[NonSerialized] private int _tempCache;`                         |
| `[DebuggerStepThrough]` | 调试时跳过该方法或类，不单步进入                   | `[DebuggerStepThrough] void Helper() { ... }`                     |
| `[DebuggerDisplay]`     | 自定义调试器中显示的信息                           | `[DebuggerDisplay("{Id} - {Name}")] public class User { ... }`    |
| `[CallerMemberName]`    | 参数装饰，获取调用者的方法名                       | `void Log([CallerMemberName] string caller = null) { ... }`       |
| `[CallerFilePath]`      | 获取调用者源文件路径                               | 同上                                                              |
| `[CallerLineNumber]`    | 获取调用者行号                                     | 同上                                                              |

### 数据绑定与验证（Data Annotations）

位于 `System.ComponentModel.DataAnnotations`，常用于 `ASP.NET MVC / EF Core / Blazor` 等框架：

| 特性                    | 作用                    | 示例                                                            |
| ----------------------- | ----------------------- | --------------------------------------------------------------- |
| `[Required]`            | 属性不能为空            | `[Required] public string Name { get; set; }`                   |
| `[StringLength]`        | 限制字符串最大/最小长度 | `[StringLength(100, MinimumLength = 5)]`                        |
| `[Range]`               | 数值或日期范围验证      | `[Range(1, 100)] public int Age { get; set; }`                  |
| `[RegularExpression]`   | 正则表达式校验          | `[RegularExpression(@"^\d{3}-\d{4}$")] public string Phone;`    |
| `[EmailAddress]`        | 电子邮件格式验证        | `[EmailAddress] public string Email { get; set; }`              |
| `[Key]`                 | 标记实体主键（EF Core） | `[Key] public int Id { get; set; }`                             |
| `[Timestamp]`           | 并发检查（行版本号）    | `[Timestamp] public byte[] RowVersion { get; set; }`            |
| `[Display(Name="...")]` | 指定显示名称            | `[Display(Name="用户名")] public string UserName { get; set; }` |

### 序列化与 Web API

#### JSON.NET / System.Text.Json

| 特性                         | 作用                                   | 示例                                                               |
| ---------------------------- | -------------------------------------- | ------------------------------------------------------------------ |
| `[JsonIgnore]`               | 忽略属性序列化                         | `[JsonIgnore] public string InternalNote { get; set; }`            |
| `[JsonProperty("name")]`     | 指定 JSON 字段名称（Newtonsoft）       | `[JsonProperty("user_name")] public string Name { get; set; }`     |
| `[JsonPropertyName("name")]` | 指定 JSON 字段名称（System.Text.Json） | `[JsonPropertyName("user_name")] public string Name { get; set; }` |

#### XML 序列化

| 特性                   | 作用              |
| ---------------------- | ----------------- |
| `[XmlElement("Name")]` | 指定元素名        |
| `[XmlAttribute]`       | 序列化为 XML 属性 |
| `[XmlIgnore]`          | 忽略字段          |

### 依赖注入与框架集成

#### ASP.NET Core

| 特性                           | 作用                                      |
| ------------------------------ | ----------------------------------------- |
| `[ApiController]`              | 启用自动参数绑定、400 响应等 Web API 特性 |
| `[Route("api/[controller]")]`  | 定义控制器路由                            |
| `[HttpGet]`, `[HttpPost]` 等   | 标记 Action 支持的 HTTP 动词              |
| `[FromServices]`               | 从 DI 容器中解析参数                      |
| `[FromQuery]`, `[FromBody]` 等 | 指定参数绑定来源                          |

### 线程与并发

| 特性                                           | 作用                               |
| ---------------------------------------------- | ---------------------------------- |
| `[MethodImpl(MethodImplOptions.Synchronized)]` | 将方法锁定为单线程访问             |
| `[ThreadStatic]`                               | 标记字段为线程静态，每线程独立实例 |
| `[AsyncStateMachine]`                          | 编译器生成，用于标记 async 方法    |

### 平台兼容与版本

| 特性                      | 作用                               |
| ------------------------- | ---------------------------------- |
| `[SupportedOSPlatform]`   | 指示 API 在指定平台可用（.NET 5+） |
| `[UnsupportedOSPlatform]` | 指示 API 在指定平台不可用          |
| `[ObsoletedOSPlatform]`   | 标记 API 在平台上的过时版本        |

```csharp
[SupportedOSPlatform("windows")]
public void WindowsOnly() { … }
```

### 自定义特性

* 定义：继承 `Attribute`

* 限制作用目标：使用 `[AttributeUsage]`

* 读取：通过反射获取 `MemberInfo.GetCustomAttributes<T>()`

```csharp
[AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
public class AuditAttribute : Attribute
{
    public string Operation { get; }
    public AuditAttribute(string operation) => Operation = operation;
}

// 在方法上使用
[Audit("Create"), Audit("Validate")]
public void CreateUser() { … }
```

### 优缺点

优点

* 声明式编程：简化配置，代码更清晰。

* 框架集成：与 `ASP.NET Core`、`EF Core` 等无缝协作。

* 可扩展性：支持自定义特性，扩展功能。

* 跨场景支持：适用 `Web`、数据库、测试等。

缺点

* 反射性能：自定义特性使用反射可能影响性能。

* 配置复杂：大量特性可能导致代码难以维护。

* 调试难度：特性行为需通过反射或日志调试。

* 依赖框架：部分特性（如 `[Route]`）特定于框架。

### 使用场景

* `Web API`：

    * 使用 `[Route]、[HttpGet]` 等定义 `RESTful` 端点。

    * 示例：用户管理 `API`。

* 数据库映射：

    * 使用 `[Key]、[Required]` 配置 `EF Core` 模型。

    * 示例：批次表头和明细表。

* 模型验证：

    * 使用 `[Required]、[StringLength]` 验证输入。

    * 示例：导入数据验证。

* 安全控制：

    * 使用 `[Authorize]、[ValidateAntiForgeryToken]` 保护端点。

    * 示例：管理员导入接口。

* 日志和监控：

    * 使用自定义特性（如 `[LogExecutionTime]`）记录性能。

    * 示例：监控导入时间。

* 测试：

    * 使用 `[Test]、[Fact]` 编写单元测试。

    * 示例：测试导入逻辑。