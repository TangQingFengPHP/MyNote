### 简介

* `Mapster` 是一个轻量、高性能的对象映射库，支持静态 & 动态映射、编译时代码生成、`LINQ` 投影等多种模式。

* 与 `AutoMapper`、`ExpressMapper` 等对比：

    * 性能更优：`Mapster` 在运行时基于表达式树生成委托，亦可使用 `Source Generator` 生成静态代码，启动一次性解析映射，后续调用近乎原生赋值速度；

    * 零侵入 & 零配置：默认按名称映射，亦支持通过 `TypeAdapterConfig` 全局或局部配置，无需继承或打标；

    * 灵活多样：支持深度映射、扁平化、条件映射、`After/Before` 钩子、接口映射、枚举映射、集合映射、`LINQ` 投影等。

* 核心特性亮点

    * 零配置映射：支持自动映射同名属性

    * 编译时映射：生成高效 `IL` 代码

    * 丰富扩展：内置集合映射、深度克隆等

    * 轻量无依赖：仅 200KB 的轻量级库

    * 高度可扩展：支持自定义转换规则

### 安装与配置

#### 安装 NuGet 包

```csharp
Install-Package Mapster
Install-Package Mapster.DependencyInjection   # 若使用 DI 集成
Install-Package Mapster.Tool                 # 若需要源代码生成
```

#### 在 ASP.NET Core 中注册

```csharp
// Program.cs
builder.Services.AddMapster(config =>
{
    // 注册全局配置
    config.ForType<User, UserDto>()
          .Map(dest => dest.FullName, src => $"{src.FirstName} {src.LastName}");
});
builder.Services.AddScoped<IMapper, ServiceMapper>();
```

```csharp
// 1. 创建映射配置类
public class MappingConfig : IRegister
{
    public void Register(TypeAdapterConfig config)
    {
        config.NewConfig<UserDto, UserViewModel>()
            .Map(dest => dest.Name, src => src.FullName);
            
        // 其他映射配置
    }
}

// 2. 在 Startup.cs 中注册
public void ConfigureServices(IServiceCollection services)
{
    // 注册 Mapster
    services.AddMapster(config =>
    {
        // 扫描程序集中的所有 IRegister 实现
        config.Scan(typeof(Startup).Assembly);
    });
    
    services.AddControllers();
}

// 3. 在控制器中使用
public class UserController : ControllerBase
{
    private readonly IMapper _mapper;
    
    public UserController(IMapper mapper)
    {
        _mapper = mapper;
    }
    
    [HttpGet]
    public IActionResult GetUser(int id)
    {
        var userDto = _userService.GetUser(id);  // 获取 DTO
        var viewModel = _mapper.Map<UserViewModel>(userDto);  // 映射到 ViewModel
        return Ok(viewModel);
    }
}
```

#### 源生成器配置

```csharp
[Mapper]
public interface IMapper
{
    UserDto Map(User user);
}

[AdaptTo("[name]Dto"), GenerateMapper]
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
}

public class UserDto
{
    public int Id { get; set; }
    public string FullName { get; set; }
    public int AgeInYears { get; set; }
}
```

* `[AdaptTo]` 和 `[GenerateMapper]`：源生成器特性，生成静态映射代码。

### 核心用法

#### 基本映射

```csharp
// 定义实体和DTO
public class User 
{
    public int Id { get; set; }
    public string Name { get; set; }
}

public class UserDto
{
    public int Id { get; set; }
    public string FullName { get; set; }
}

// 简单映射（无需配置）
var user = new User { Id = 1, Name = "Alice" };
var dto = user.Adapt<UserDto>(); // 自动映射同名属性

Console.WriteLine(dto.FullName); // 输出: Alice
```

#### 配置映射规则

```csharp
// 全局配置（通常在应用启动时设置）
TypeAdapterConfig.GlobalSettings.Default
    .NameMatchingStrategy(NameMatchingStrategy.Flexible)  // 灵活匹配策略
    .PreserveReference(true);  // 保留对象引用

// 特定类型映射配置
TypeAdapterConfig<UserDto, UserViewModel>.NewConfig()
    .Map(dest => dest.Name, src => src.FullName)  // 属性重命名映射
    .Map(dest => dest.YearsOld, src => src.Age)    // 简单映射
    .Map(dest => dest.DateOfBirth, src => src.BirthDate);  // 日期映射

// 计算属性映射
TypeAdapterConfig<OrderDto, OrderViewModel>.NewConfig()
    .Map(dest => dest.DiscountPrice, src => src.IsOnSale ? src.Price * 0.9m : src.Price);

// 忽略属性
TypeAdapterConfig<UserDto, UserViewModel>.NewConfig()
    .Ignore(dest => dest.SomeProperty);
```

#### 集合映射

```csharp
var userDtos = new List<UserDto>
{
    new UserDto { FullName = "Alice", Age = 25 },
    new UserDto { FullName = "Bob", Age = 30 }
};

// 映射集合
var viewModels = userDtos.Adapt<List<UserViewModel>>();

// 或者使用静态 API
var viewModels2 = Mapster.TypeAdapter.Adapt<List<UserViewModel>>(userDtos);
```

#### 继承与多态映射

```csharp
// 基类
public class AnimalDto
{
    public string Name { get; set; }
}

// 派生类
public class DogDto : AnimalDto
{
    public string Breed { get; set; }
}

// 目标基类
public class AnimalViewModel
{
    public string AnimalName { get; set; }
    public string Type { get; set; }
}

// 目标派生类
public class DogViewModel : AnimalViewModel
{
    public string DogBreed { get; set; }
}

// 配置映射
TypeAdapterConfig<AnimalDto, AnimalViewModel>.NewConfig()
    .Map(dest => dest.AnimalName, src => src.Name);

TypeAdapterConfig<DogDto, DogViewModel>.NewConfig()
    .IncludeBase<AnimalDto, AnimalViewModel>()  // 继承基类映射
    .Map(dest => dest.DogBreed, src => src.Breed)
    .Map(dest => dest.Type, src => "Dog");

// 映射
var dogDto = new DogDto { Name = "Buddy", Breed = "Golden Retriever" };
var dogViewModel = dogDto.Adapt<DogViewModel>();
```

#### 条件映射与忽略

```csharp
cfg.ForType<Product, ProductDto>()
   .MapIf(dest => dest.Description, src => src.Desc, src => !string.IsNullOrEmpty(src.Desc))
   .IgnoreNullValues(true);
```

#### 嵌套 & 扁平化

```csharp
public class User { public Address Addr { get; set; } }
public class AddressDto { public string City { get; set; } }
public class UserDto {
    public string Addr_City { get; set; }
}
// 扁平化
cfg.FlatteningEnabled = true;
var dto = user.Adapt<UserDto>();
// Addr.City → Addr_City
```

#### 双向映射 & 反向映射

```csharp
// 配置双向映射
TypeAdapterConfig<User, UserDto>.NewConfig()
    .TwoWays(); // 启用反向映射

// 反向转换
User user = userDto.Adapt<User>();
```

### 进阶特性

#### 表达式树生成

```csharp
// 生成表达式树
Expression<Func<UserDto, UserViewModel>> expression = 
    TypeAdapterConfig<UserDto, UserViewModel>.GetProjectionExpression();

// 可以用于 Entity Framework 查询
var users = dbContext.Users
    .Select(expression)
    .ToList();
```

#### 全局 & 局部配置

```csharp
// 全局
TypeAdapterConfig.GlobalSettings.Default
    .NameMatchingStrategy(NameMatchingStrategy.Flexible);

// 局部
var cfg = new TypeAdapterConfig();
cfg.ForType<Order, OrderDto>()
   .Map(dest => dest.Total, src => src.Quantity * src.UnitPrice);
```

#### After/Before 处理

```csharp
TypeAdapterConfig<User, UserDto>.NewConfig()
    .BeforeMapping((src, dest) => Console.WriteLine("映射开始"))
    .AfterMapping((src, dest) => dest.MappedAt = DateTime.Now);
```

#### LINQ 投影（ProjectToType）

将映射规则转换为表达式，用于 `EF Core IQueryable` 投影，避免加载完整实体：

```csharp
var dtos = dbContext.Users
    .ProjectToType<UserDto>(TypeAdapterConfig.GlobalSettings)
    .ToList();
```

#### 自定义类型转换器

```csharp
public class DateTimeConverter : IConverter<DateTime, string>
{
    public string Convert(DateTime source)
        => source.ToString("yyyy-MM-dd HH:mm");
}

// 注册转换器
TypeAdapterConfig<DateTime, string>.NewConfig()
    .MapWith(src => new DateTimeConverter());
```

#### 深度克隆

```csharp
var original = new User { Name = "Bob" };
var clone = original.Adapt<User>(); // 深度克隆
```

#### 基于特性的映射

```csharp
public class UserDto
{
    [AdaptMember("Name")]
    public string FullName { get; set; }
    
    [Ignore]
    public string Password { get; set; }
}

// 自动应用特性配置
var dto = user.Adapt<UserDto>();
```

### 性能优化技巧

#### 预编译映射

```csharp
// 在应用启动时进行预热
TypeAdapterConfig.GlobalSettings.Compile();

// 预编译映射委托（提升30%性能）
var mapper = TypeAdapterConfig<User, UserDto>.NewConfig()
    .Compile() // 显式编译
    .CreateMapExpression<User, UserDto>()
    .Compile();

UserDto dto = mapper(user);
```

#### 缓存配置

```csharp
// 全局单例配置
public static class MapperConfig
{
    public static void Register()
    {
        TypeAdapterConfig<User, UserDto>.NewConfig()
            .Map(dest => dest.FullName, src => src.Name);
            
        TypeAdapterConfig.GlobalSettings.Compile(); // 预编译所有配置
    }
}

// 启动时注册
MapperConfig.Register();
```

### 性能对比

| 特性          | Mapster（运行时）  | Mapster（源生成） | AutoMapper        |
| ------------- | ------------------ | ----------------- | ----------------- |
| 首次调用开销  | 中等（表达式编译） | 启动时一次性      | 高（反射+表达式） |
| 后续调用速度  | 快                 | 极快（直调用）    | 较快              |
| 内存占用      | 低                 | 中                | 中高              |
| LINQ 投影支持 | ✔                  | ✔                 | ✔                 |
| 配置复杂度    | 低                 | 低                | 中                |

### 最佳实践

* 复用配置：将 `TypeAdapterConfig` 作为单例注册，避免重复注册开销。

* 按模块拆分 `Profile`：类似 `AutoMapper`，将映射配置按业务模块分文件管理。

* 优先使用 `ProjectToType`：在数据库查询中直接投影到 `DTO`，减少内存压力。

* 启用源生成：生产环境建议通过 `Mapster.Tool` 生成静态 `Mapper`，确保映射零反射。

* 避免深度递归：对超大对象图，考虑只映射需要字段，或分批映射。

* 测试映射完整性：在单元测试中调用 `TypeAdapterConfig.GlobalSettings.AssertConfigurationIsValid()`，及时发现遗漏。

### 常见问题解决方案

#### 循环引用处理

```csharp
TypeAdapterConfig.GlobalSettings.Default
    .PreserveReference(true); // 启用引用保留

// 或忽略引用属性
config.NewConfig<Parent, ParentDto>()
    .Ignore(dest => dest.Children);
```

#### 映射配置冲突

```csharp
// 清除特定配置
TypeAdapterConfig<User, UserDto>.Clear();

// 重新配置
TypeAdapterConfig<User, UserDto>.NewConfig()
    .Map(/* 新规则 */);
```

#### 空值处理策略

```csharp
// 全局配置
TypeAdapterConfig.GlobalSettings.Default
    .IgnoreNullValues(true)
    .IgnoreIf((src, dest, member) => member == null);
```

#### 枚举映射

```csharp
public enum UserStatus { Active, Inactive }

public enum UserStatusDto { Active, Disabled }

TypeAdapterConfig<UserStatus, UserStatusDto>.NewConfig()
    .MapWith(status => status == UserStatus.Inactive 
        ? UserStatusDto.Disabled 
        : UserStatusDto.Active);
```

### 资源与扩展

* `GitHub`：https://github.com/MapsterMapper/Mapster

* 文档：https://mapstermapper.github.io