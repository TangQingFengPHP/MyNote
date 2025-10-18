### 简介

* `FluentValidation` 是一个基于“流式 `API`”（`Fluent API`）的 `.NET` 验证框架，用于在应用层对模型（`DTO、ViewModel、Entity` 等）进行声明式验证。

* 核心优势：

    * 高可读性：通过链式方法配置验证规则，逻辑清晰；

    * 可复用：将验证代码从业务逻辑中分离，易于单元测试；

    * 丰富的内置规则：邮箱、长度、正则、多字段联动、集合验证等；

    * 可扩展：支持自定义验证器、异步验证、跨属性验证。

* 适用场景：

    * `Web API` 模型验证

    * 复杂业务规则验证

    * 需要高度可定制验证逻辑的系统

    * 多语言验证消息需求

    * 需要测试覆盖的验证逻辑

### 安装与基础配置

* `NuGet` 包

```shell
Install-Package FluentValidation
Install-Package FluentValidation.DependencyInjectionExtensions
```

* 引用命名空间

```shell
using FluentValidation;
```

### 核心用法

#### 定义 Model 与 Validator

```csharp
public class UserDto
{
    public string Username { get; set; }
    public string Email    { get; set; }
    public int    Age      { get; set; }
}

public class UserDtoValidator : AbstractValidator<UserDto>
{
    public UserDtoValidator()
    {
        // NotEmpty / NotNull
        RuleFor(x => x.Username)
            .NotEmpty().WithMessage("用户名不能为空")
            .Length(3, 20).WithMessage("用户名长度须在3到20之间");

        // Email 格式
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("邮箱不能为空")
            .EmailAddress().WithMessage("邮箱格式不正确");

        // 数值范围
        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120)
            .WithMessage("年龄须在18到120之间");
    }
}
```

#### 执行验证

```csharp
var user = new UserDto { Username = "", Email = "bad", Age = 10 };
var validator = new UserDtoValidator();
var result = validator.Validate(user);

if (!result.IsValid)
{
    foreach (var failure in result.Errors)
    {
        Console.WriteLine($"{failure.PropertyName}: {failure.ErrorMessage}");
    }
}

// ASP.NET Core 自动验证
[HttpPost]
public IActionResult CreateUser([FromBody] UserDto user)
{
    // 模型绑定后自动验证
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }
    // ...
}
```

### 常用验证规则

| 方法                             | 作用           |
| -------------------------------- | -------------- |
| `NotNull()` / `NotEmpty()`       | 非空或非空串   |
| `Length(min, max)`               | 字符串长度范围 |
| `EmailAddress()`                 | 邮箱格式       |
| `Matches(regex)`                 | 正则匹配       |
| `InclusiveBetween(min, max)`     | 数值范围       |
| `GreaterThan(x)` / `LessThan(x)` | 大小比较       |
| `Must(predicate)`                | 自定义同步条件 |
| `MustAsync(asyncPredicate)`      | 自定义异步条件 |

### 进阶特性

#### 跨属性验证

```csharp
RuleFor(x => x.EndDate)
    .GreaterThan(x => x.StartDate)
    .WithMessage("结束时间必须晚于开始时间");
```

#### 条件验证

```csharp
RuleFor(x => x.Password)
    .NotEmpty().When(x => x.RequirePassword)
    .WithMessage("密码不能为空");
```

#### 集合与嵌套对象

```csharp
public class OrderDto { public List<OrderItemDto> Items { get; set; } }
public class OrderItemDto { public int Quantity { get; set; } }

public class OrderDtoValidator : AbstractValidator<OrderDto>
{
    public OrderDtoValidator()
    {
        RuleForEach(x => x.Items)
            .ChildRules(items =>
            {
                items.RuleFor(i => i.Quantity)
                     .GreaterThan(0).WithMessage("数量须大于0");
            });
    }
}
```

#### 异步验证

```csharp
RuleFor(x => x.Username)
    .MustAsync(async (name, ct) => !await userRepo.ExistsAsync(name))
    .WithMessage("用户名已存在");
```

#### 级联验证

使用 `CascadeMode.Stop` 提高性能

```csharp
RuleFor(x => x.Name).Cascade(CascadeMode.Stop).NotEmpty().MaximumLength(50);
```

### 验证规则组织

#### 规则集(RuleSets)

```csharp
public class UserValidator : AbstractValidator<UserDto>
{
    public UserValidator()
    {
        // 公共规则
        RuleFor(user => user.Name).NotEmpty();
        
        // 创建规则集
        RuleSet("Admin", () => 
        {
            RuleFor(user => user.IsAdmin).Must(b => b == true)
                .WithMessage("管理员用户必须设置管理员标志");
        });

        RuleSet("PaymentInfo", () => {
            RuleFor(c => c.CreditCardNumber).NotEmpty();
            RuleFor(c => c.CreditCardExpiry).NotEmpty();
        });
    }
}

// 使用指定规则集
var result = validator.Validate(user, options => 
{
    options.IncludeRuleSets("Admin", "PaymentInfo");
});
```

#### 继承与组合

```csharp
// 基础验证器
public class PersonValidator : AbstractValidator<PersonDto>
{
    public PersonValidator()
    {
        RuleFor(p => p.Name).NotEmpty();
        RuleFor(p => p.BirthDate).LessThan(DateTime.Now);
    }
}

// 继承扩展
public class EmployeeValidator : PersonValidator
{
    public EmployeeValidator()
    {
        Include(new PersonValidator()); // 包含基础规则
        RuleFor(e => e.EmployeeId).NotEmpty();
        RuleFor(e => e.Department).NotEmpty();
    }
}

// 组合验证
public class AdvancedUserValidator : AbstractValidator<UserDto>
{
    public AdvancedUserValidator()
    {
        Include(new UserValidator());
        RuleFor(u => u.SecurityLevel).InclusiveBetween(1, 5);
    }
}
```

### 自定义扩展

#### 自定义验证器

```csharp
public static class CustomValidators
{
    public static IRuleBuilderOptions<T, string> ValidIdCard<T>(this IRuleBuilder<T, string> ruleBuilder)
    {
        return ruleBuilder.Must(id => Regex.IsMatch(id, @"^[1-9]\d{16}[0-9X]$"))
                          .WithMessage("身份证格式不正确");
    }
}

// 使用
RuleFor(x => x.IdCard).ValidIdCard();
```

#### 自定义属性比较器

```csharp
public class DateRangeValidator : PropertyValidator
{
    public DateRangeValidator() : base("{PropertyName} 时间范围不合法") { }

    protected override bool IsValid(PropertyValidatorContext context)
    {
        var dto = (MyDto)context.InstanceToValidate;
        return dto.End > dto.Start;
    }
}

// 在 Validator 中
RuleFor(x => x.Start).SetValidator(new DateRangeValidator());
```

### ASP.NET Core 集成

#### 注册服务（Program.cs）

```csharp
builder.Services
    .AddControllers()
    .AddFluentValidation(cfg =>
    {
        // 自动注册当前程序集所有继承 AbstractValidator 的类型
        cfg.RegisterValidatorsFromAssemblyContaining<Startup>();
        // 禁用 DataAnnotations 验证（可选）
        cfg.RunDefaultMvcValidationAfterFluentValidationExecutes = false;
    });
```

#### 自动触发

* `ASP.NET Core` 在模型绑定后会自动调用对应 `Validator`，并将错误添加到 `ModelState`。

* 在 `Controller` 中可直接检查 `if (!ModelState.IsValid)` 或依赖` [ApiController]` 的自动返回行为。

#### 自定义错误响应

```csharp
// 配置全局异常处理
services.Configure<ApiBehaviorOptions>(options =>
{
    options.InvalidModelStateResponseFactory = context =>
    {
        var errors = context.ModelState
            .Where(e => e.Value.Errors.Count > 0)
            .ToDictionary(
                kvp => kvp.Key,
                kvp => kvp.Value.Errors.Select(e => e.ErrorMessage).ToArray()
            );
        
        return new BadRequestObjectResult(new
        {
            Code = 400,
            Message = "请求验证失败",
            Errors = errors
        });
    };
});
```

### 最佳实践

* 分层组织规则：

    * 针对同一模型，可拆分多个 `Validator`（或使用 `Include()`），保持单一职责。

* 复用规则集：

    * 对于常见字段（如邮箱、手机号），可定义公共规则并通过扩展方法重用。

* 错误消息国际化：

    * 将消息文本放入资源文件，使用 `.WithMessage(x => Resources.FieldRequired)`。

* 性能考虑：

    * 异步验证会序列化执行，若有多个异步规则，可合并或避免不必要的数据库调用。

* 测试验证器：

    * 为每个 `Validator` 编写单元测试，覆盖正常和边界情况，确保规则生效。

* 日志与监控：

    * 在全局捕获验证失败日志，统计常见错误，优化用户体验。

### 资源与扩展

* GitHub：`https://github.com/FluentValidation/FluentValidation`

* 文档：`https://docs.fluentvalidation.net`

* `NuGet` 包：

    * `FluentValidation`：核心验证库。

    * `FluentValidation.DependencyInjectionExtensions`：`ASP.NET Core` 集成。

* 扩展：

    * 支持 `Blazor、MVC` 和 `Web API`。

    * 提供多语言支持和客户端验证。