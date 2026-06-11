### 简介

`Emit` 通常指的是：

```csharp
System.Reflection.Emit
```

它是 `.NET` 里一套非常底层的运行时代码生成 API。

一句话概括：

```text
Emit 可以在程序运行时动态生成 IL、方法、类型、属性、字段和程序集。
```

普通 C# 代码是先写源码，再编译成程序集，然后运行。

Emit 的思路不一样：

```text
运行时
  |
  v
动态拼 IL 指令
  |
  v
生成方法或类型
  |
  v
直接调用
```

这听起来有点像“运行时手写 IL”。

实际也差不多。

### Emit 能解决什么问题？

Emit 不是日常业务代码的首选工具。

它更常见于框架底层和性能敏感场景。

典型场景有这些：

* ORM 把数据库行快速映射成对象
* JSON 序列化器生成属性访问代码
* Mapper 框架生成对象复制代码
* AOP / 动态代理框架生成代理类型
* Mock 框架运行时生成接口实现
* 表达式引擎把公式编译成方法
* 替代高频反射调用，减少性能损耗

比如一个 ORM 要把查询结果填充到对象：

```csharp
user.Id = reader.GetInt32(0);
user.Name = reader.GetString(1);
```

如果每次都用反射：

```csharp
property.SetValue(user, value);
```

灵活是灵活，但频繁调用时开销明显。

Emit 可以在运行时生成类似手写赋值的代码，然后缓存成委托反复调用。

这样第一次生成稍微复杂一点，后面调用就很快。

### Emit 和反射的区别

反射偏“运行时读取和调用已有结构”。

Emit 偏“运行时生成新的结构或新的执行逻辑”。

| 对比项 | 反射 | Emit |
| --- | --- | --- |
| 主要作用 | 读取类型信息、调用已有成员 | 动态生成 IL、方法、类型 |
| 使用难度 | 较低 | 较高 |
| 性能 | 直接调用较慢 | 生成委托后通常更快 |
| 可读性 | 较好 | 较差 |
| 适合场景 | 配置、扫描、插件、低频调用 | 框架底层、高频调用、动态代理 |

简单理解：

```text
反射是查字典和照着调用。
Emit 是现场生成一段更接近手写代码的执行逻辑。
```

### Emit 和表达式树、源生成器的区别

这几个东西都和“生成代码”有关，但时间点和使用方式不同。

| 技术 | 生成时间 | 生成内容 | 常见用途 |
| --- | --- | --- | --- |
| Emit | 运行时 | IL / 动态类型 / 动态方法 | 动态代理、运行时优化 |
| Expression Tree | 运行时 | 表达式树，可编译成委托 | 动态查询、动态委托 |
| Source Generator | 编译期 | C# 源码 | 静态代码生成、AOT 友好 |

如果需求在编译期就能确定，源生成器通常更稳。

如果需求必须运行时才知道，比如运行时拿到类型、字段、接口，再临时生成方法或代理，Emit 仍然有价值。

### Emit 的核心类

常用类型如下：

| 类型 | 作用 |
| --- | --- |
| `DynamicMethod` | 在内存中创建一个轻量级动态方法 |
| `ILGenerator` | 发出 IL 指令 |
| `AssemblyBuilder` | 创建动态程序集 |
| `ModuleBuilder` | 创建动态模块 |
| `TypeBuilder` | 创建动态类型 |
| `FieldBuilder` | 创建字段 |
| `PropertyBuilder` | 创建属性 |
| `MethodBuilder` | 创建方法 |
| `ConstructorBuilder` | 创建构造函数 |

如果只是生成一个临时方法，优先看 `DynamicMethod`。

如果要生成一个完整类、属性、接口实现，就要用：

```text
AssemblyBuilder
ModuleBuilder
TypeBuilder
MethodBuilder
ILGenerator
```

### IL 是栈式执行模型

Emit 最难的地方不是 API 多，而是要理解 IL 的栈模型。

例如要生成这个方法：

```csharp
int Add(int a, int b)
{
    return a + b;
}
```

对应 IL 思路是：

```text
把 a 压栈
把 b 压栈
弹出两个值相加
把结果压栈
返回栈顶结果
```

对应指令：

```csharp
il.Emit(OpCodes.Ldarg_0);
il.Emit(OpCodes.Ldarg_1);
il.Emit(OpCodes.Add);
il.Emit(OpCodes.Ret);
```

常见指令先记这些就够用：

| 指令 | 作用 |
| --- | --- |
| `Ldarg_0` | 加载第 0 个参数，实例方法里通常是 `this` |
| `Ldarg_1` | 加载第 1 个参数 |
| `Ldstr` | 加载字符串常量 |
| `Ldfld` | 读取实例字段 |
| `Stfld` | 写入实例字段 |
| `Call` | 调用静态方法或非虚方法 |
| `Callvirt` | 调用虚方法或实例方法 |
| `Castclass` | 引用类型转换 |
| `Box` | 值类型装箱 |
| `Unbox_Any` | 拆箱成值类型 |
| `Newobj` | 调用构造函数创建对象 |
| `Ret` | 返回 |

### Demo 目标

下面写一个控制台 Demo，覆盖几个真实场景：

```text
动态生成加法方法
动态生成属性 getter
动态生成属性 setter
动态生成对象映射器
动态生成一个类
动态生成一个接口实现
```

创建项目：

```bash
mkdir EmitDemo
cd EmitDemo

dotnet new console
```

修改 `Program.cs`。

### 准备实体类

先准备两个普通类型：

```csharp
using System;
using System.Collections.Generic;
using System.Reflection;
using System.Reflection.Emit;

Console.WriteLine("Emit Demo");

public class User
{
    public int Id { get; set; }

    public string Name { get; set; } = "";

    public int Age { get; set; }
}

public class UserDto
{
    public int Id { get; set; }

    public string Name { get; set; } = "";
}
```

后面的示例都基于这两个类。

### DynamicMethod：动态生成加法方法

先从最小例子开始。

目标是运行时生成一个方法：

```csharp
int Add(int a, int b)
{
    return a + b;
}
```

Emit 代码：

```csharp
static Func<int, int, int> CreateAdd()
{
    DynamicMethod method = new(
        name: "Add",
        returnType: typeof(int),
        parameterTypes: new[] { typeof(int), typeof(int) },
        m: typeof(User).Module);

    ILGenerator il = method.GetILGenerator();

    il.Emit(OpCodes.Ldarg_0);
    il.Emit(OpCodes.Ldarg_1);
    il.Emit(OpCodes.Add);
    il.Emit(OpCodes.Ret);

    return method.CreateDelegate<Func<int, int, int>>();
}
```

调用：

```csharp
Func<int, int, int> add = CreateAdd();

Console.WriteLine(add(10, 20));
```

输出：

```text
30
```

这段代码的关键不是加法，而是理解 Emit 的基本流程：

```text
创建 DynamicMethod
获取 ILGenerator
逐条发出 IL 指令
CreateDelegate 转成可调用委托
```

### 动态生成属性 Getter

接下来做一个更贴近实际项目的例子。

目标是根据类型和属性名，动态生成类似下面的方法：

```csharp
object? GetValue(object instance)
{
    return ((User)instance).Name;
}
```

Emit 实现：

```csharp
static Func<object, object?> CreateGetter(Type type, string propertyName)
{
    PropertyInfo property = type.GetProperty(propertyName)
        ?? throw new InvalidOperationException($"属性不存在：{propertyName}");

    MethodInfo getMethod = property.GetGetMethod()
        ?? throw new InvalidOperationException($"属性没有 getter：{propertyName}");

    DynamicMethod method = new(
        name: "Get_" + type.Name + "_" + propertyName,
        returnType: typeof(object),
        parameterTypes: new[] { typeof(object) },
        m: type.Module,
        skipVisibility: true);

    ILGenerator il = method.GetILGenerator();

    il.Emit(OpCodes.Ldarg_0);
    il.Emit(OpCodes.Castclass, type);
    il.Emit(OpCodes.Callvirt, getMethod);

    if (property.PropertyType.IsValueType)
    {
        il.Emit(OpCodes.Box, property.PropertyType);
    }

    il.Emit(OpCodes.Ret);

    return method.CreateDelegate<Func<object, object?>>();
}
```

调用：

```csharp
var user = new User
{
    Id = 1,
    Name = "张三",
    Age = 20
};

Func<object, object?> getName = CreateGetter(typeof(User), "Name");
Func<object, object?> getAge = CreateGetter(typeof(User), "Age");

Console.WriteLine(getName(user));
Console.WriteLine(getAge(user));
```

输出：

```text
张三
20
```

这里有一个关键点：

```csharp
if (property.PropertyType.IsValueType)
{
    il.Emit(OpCodes.Box, property.PropertyType);
}
```

因为委托返回的是 `object?`。

如果属性是 `int`、`DateTime` 这种值类型，需要装箱成 `object`。

### 动态生成属性 Setter

Getter 能读，Setter 负责写。

目标是动态生成类似下面的方法：

```csharp
void SetValue(object instance, object? value)
{
    ((User)instance).Age = (int)value!;
}
```

Emit 实现：

```csharp
static Action<object, object?> CreateSetter(Type type, string propertyName)
{
    PropertyInfo property = type.GetProperty(propertyName)
        ?? throw new InvalidOperationException($"属性不存在：{propertyName}");

    MethodInfo setMethod = property.GetSetMethod()
        ?? throw new InvalidOperationException($"属性没有 setter：{propertyName}");

    DynamicMethod method = new(
        name: "Set_" + type.Name + "_" + propertyName,
        returnType: typeof(void),
        parameterTypes: new[] { typeof(object), typeof(object) },
        m: type.Module,
        skipVisibility: true);

    ILGenerator il = method.GetILGenerator();

    il.Emit(OpCodes.Ldarg_0);
    il.Emit(OpCodes.Castclass, type);

    il.Emit(OpCodes.Ldarg_1);

    if (property.PropertyType.IsValueType)
    {
        il.Emit(OpCodes.Unbox_Any, property.PropertyType);
    }
    else
    {
        il.Emit(OpCodes.Castclass, property.PropertyType);
    }

    il.Emit(OpCodes.Callvirt, setMethod);
    il.Emit(OpCodes.Ret);

    return method.CreateDelegate<Action<object, object?>>();
}
```

调用：

```csharp
Action<object, object?> setName = CreateSetter(typeof(User), "Name");
Action<object, object?> setAge = CreateSetter(typeof(User), "Age");

setName(user, "李四");
setAge(user, 28);

Console.WriteLine(user.Name);
Console.WriteLine(user.Age);
```

输出：

```text
李四
28
```

Setter 里要注意拆箱：

```csharp
il.Emit(OpCodes.Unbox_Any, property.PropertyType);
```

因为传进来的 `value` 是 `object`，但真正写入 `Age` 时需要 `int`。

### 实战：动态生成对象映射器

很多 Mapper 框架都会做类似的事情：

```csharp
User -> UserDto
```

手写代码是这样：

```csharp
var dto = new UserDto
{
    Id = user.Id,
    Name = user.Name
};
```

下面用 Emit 生成一个映射委托：

```csharp
Func<User, UserDto> mapper = CreateMapper<User, UserDto>();

UserDto dto = mapper(new User
{
    Id = 100,
    Name = "王五",
    Age = 30
});

Console.WriteLine($"{dto.Id} - {dto.Name}");
```

输出：

```text
100 - 王五
```

实现代码：

```csharp
static Func<TSource, TTarget> CreateMapper<TSource, TTarget>()
    where TTarget : new()
{
    Type sourceType = typeof(TSource);
    Type targetType = typeof(TTarget);

    DynamicMethod method = new(
        name: "Map_" + sourceType.Name + "_To_" + targetType.Name,
        returnType: targetType,
        parameterTypes: new[] { sourceType },
        m: targetType.Module,
        skipVisibility: true);

    ILGenerator il = method.GetILGenerator();

    LocalBuilder targetLocal = il.DeclareLocal(targetType);

    ConstructorInfo ctor = targetType.GetConstructor(Type.EmptyTypes)
        ?? throw new InvalidOperationException($"{targetType.Name} 缺少无参构造函数");

    il.Emit(OpCodes.Newobj, ctor);
    il.Emit(OpCodes.Stloc, targetLocal);

    foreach (PropertyInfo targetProperty in targetType.GetProperties())
    {
        if (!targetProperty.CanWrite)
        {
            continue;
        }

        PropertyInfo? sourceProperty = sourceType.GetProperty(targetProperty.Name);
        if (sourceProperty is null || !sourceProperty.CanRead)
        {
            continue;
        }

        if (sourceProperty.PropertyType != targetProperty.PropertyType)
        {
            continue;
        }

        MethodInfo getMethod = sourceProperty.GetGetMethod()
            ?? throw new InvalidOperationException($"源属性没有 getter：{sourceProperty.Name}");

        MethodInfo setMethod = targetProperty.GetSetMethod()
            ?? throw new InvalidOperationException($"目标属性没有 setter：{targetProperty.Name}");

        il.Emit(OpCodes.Ldloc, targetLocal);
        il.Emit(OpCodes.Ldarg_0);
        il.Emit(OpCodes.Callvirt, getMethod);
        il.Emit(OpCodes.Callvirt, setMethod);
    }

    il.Emit(OpCodes.Ldloc, targetLocal);
    il.Emit(OpCodes.Ret);

    return method.CreateDelegate<Func<TSource, TTarget>>();
}
```

这段 IL 的核心逻辑是：

```text
new UserDto()
保存到局部变量
读取 source.Id
写入 target.Id
读取 source.Name
写入 target.Name
返回 target
```

这种映射器通常会缓存起来：

```csharp
private static readonly Func<User, UserDto> CachedMapper = CreateMapper<User, UserDto>();
```

不要每次映射都重新 Emit。

正确方式是：

```text
第一次生成委托
缓存委托
后续重复调用委托
```

### 生成完整动态类型

`DynamicMethod` 适合生成单个方法。

如果要生成一个类，就要用 `AssemblyBuilder`、`ModuleBuilder`、`TypeBuilder`。

目标是运行时生成这个类：

```csharp
public class RuntimePerson
{
    private string _name;

    public RuntimePerson(string name)
    {
        _name = name;
    }

    public string Name
    {
        get => _name;
        set => _name = value;
    }

    public string SayHello()
    {
        return "Hello " + _name;
    }
}
```

Emit 代码：

```csharp
static Type CreatePersonType()
{
    AssemblyName assemblyName = new("RuntimePersonAssembly");

    AssemblyBuilder assemblyBuilder = AssemblyBuilder.DefineDynamicAssembly(
        assemblyName,
        AssemblyBuilderAccess.RunAndCollect);

    ModuleBuilder moduleBuilder = assemblyBuilder.DefineDynamicModule("MainModule");

    TypeBuilder typeBuilder = moduleBuilder.DefineType(
        "RuntimePerson",
        TypeAttributes.Public | TypeAttributes.Class);

    FieldBuilder nameField = typeBuilder.DefineField(
        "_name",
        typeof(string),
        FieldAttributes.Private);

    ConstructorBuilder constructor = typeBuilder.DefineConstructor(
        MethodAttributes.Public,
        CallingConventions.Standard,
        new[] { typeof(string) });

    ILGenerator ctorIl = constructor.GetILGenerator();
    ctorIl.Emit(OpCodes.Ldarg_0);
    ctorIl.Emit(OpCodes.Call, typeof(object).GetConstructor(Type.EmptyTypes)!);
    ctorIl.Emit(OpCodes.Ldarg_0);
    ctorIl.Emit(OpCodes.Ldarg_1);
    ctorIl.Emit(OpCodes.Stfld, nameField);
    ctorIl.Emit(OpCodes.Ret);

    PropertyBuilder nameProperty = typeBuilder.DefineProperty(
        "Name",
        PropertyAttributes.None,
        typeof(string),
        Type.EmptyTypes);

    MethodBuilder getName = typeBuilder.DefineMethod(
        "get_Name",
        MethodAttributes.Public | MethodAttributes.SpecialName | MethodAttributes.HideBySig,
        typeof(string),
        Type.EmptyTypes);

    ILGenerator getIl = getName.GetILGenerator();
    getIl.Emit(OpCodes.Ldarg_0);
    getIl.Emit(OpCodes.Ldfld, nameField);
    getIl.Emit(OpCodes.Ret);

    MethodBuilder setName = typeBuilder.DefineMethod(
        "set_Name",
        MethodAttributes.Public | MethodAttributes.SpecialName | MethodAttributes.HideBySig,
        typeof(void),
        new[] { typeof(string) });

    ILGenerator setIl = setName.GetILGenerator();
    setIl.Emit(OpCodes.Ldarg_0);
    setIl.Emit(OpCodes.Ldarg_1);
    setIl.Emit(OpCodes.Stfld, nameField);
    setIl.Emit(OpCodes.Ret);

    nameProperty.SetGetMethod(getName);
    nameProperty.SetSetMethod(setName);

    MethodBuilder sayHello = typeBuilder.DefineMethod(
        "SayHello",
        MethodAttributes.Public,
        typeof(string),
        Type.EmptyTypes);

    ILGenerator sayIl = sayHello.GetILGenerator();
    sayIl.Emit(OpCodes.Ldstr, "Hello ");
    sayIl.Emit(OpCodes.Ldarg_0);
    sayIl.Emit(OpCodes.Ldfld, nameField);
    sayIl.Emit(OpCodes.Call, typeof(string).GetMethod(
        nameof(string.Concat),
        new[] { typeof(string), typeof(string) })!);
    sayIl.Emit(OpCodes.Ret);

    return typeBuilder.CreateType()
        ?? throw new InvalidOperationException("动态类型创建失败");
}
```

调用：

```csharp
Type personType = CreatePersonType();

object person = Activator.CreateInstance(personType, "赵六")
    ?? throw new InvalidOperationException("实例创建失败");

Console.WriteLine(personType.GetProperty("Name")!.GetValue(person));

personType.GetProperty("Name")!.SetValue(person, "钱七");

Console.WriteLine(personType.GetMethod("SayHello")!.Invoke(person, null));
```

输出：

```text
赵六
Hello 钱七
```

### 动态实现接口

Emit 也可以运行时生成接口实现。

先准备接口：

```csharp
public interface ICalculator
{
    int Add(int a, int b);
}
```

目标是动态生成一个类：

```csharp
public class RuntimeCalculator : ICalculator
{
    public int Add(int a, int b)
    {
        Console.WriteLine("Add invoked");
        return a + b;
    }
}
```

Emit 实现：

```csharp
static Type CreateCalculatorType()
{
    AssemblyBuilder assemblyBuilder = AssemblyBuilder.DefineDynamicAssembly(
        new AssemblyName("RuntimeCalculatorAssembly"),
        AssemblyBuilderAccess.RunAndCollect);

    ModuleBuilder moduleBuilder = assemblyBuilder.DefineDynamicModule("MainModule");

    TypeBuilder typeBuilder = moduleBuilder.DefineType(
        "RuntimeCalculator",
        TypeAttributes.Public | TypeAttributes.Class,
        typeof(object),
        new[] { typeof(ICalculator) });

    ConstructorBuilder constructor = typeBuilder.DefineDefaultConstructor(MethodAttributes.Public);

    MethodBuilder addMethod = typeBuilder.DefineMethod(
        "Add",
        MethodAttributes.Public | MethodAttributes.Virtual | MethodAttributes.HideBySig,
        typeof(int),
        new[] { typeof(int), typeof(int) });

    ILGenerator il = addMethod.GetILGenerator();

    il.Emit(OpCodes.Ldstr, "Add invoked");
    il.Emit(OpCodes.Call, typeof(Console).GetMethod(
        nameof(Console.WriteLine),
        new[] { typeof(string) })!);

    il.Emit(OpCodes.Ldarg_1);
    il.Emit(OpCodes.Ldarg_2);
    il.Emit(OpCodes.Add);
    il.Emit(OpCodes.Ret);

    MethodInfo interfaceMethod = typeof(ICalculator).GetMethod(nameof(ICalculator.Add))
        ?? throw new InvalidOperationException("接口方法不存在");

    typeBuilder.DefineMethodOverride(addMethod, interfaceMethod);

    return typeBuilder.CreateType()
        ?? throw new InvalidOperationException("动态类型创建失败");
}
```

调用：

```csharp
Type calculatorType = CreateCalculatorType();

ICalculator calculator = (ICalculator)(Activator.CreateInstance(calculatorType)
    ?? throw new InvalidOperationException("实例创建失败"));

Console.WriteLine(calculator.Add(3, 5));
```

输出：

```text
Add invoked
8
```

这类能力就是很多动态代理、Mock、AOP 框架的基础。

真实框架会在方法前后织入更多逻辑：

```text
记录日志
权限检查
事务开启
调用真实对象
事务提交
异常处理
```

### Emit 生成的代码要缓存

Emit 的生成成本不低。

如果每次调用都重新生成动态方法，性能会很差。

错误思路：

```text
每次读取属性
  |
  v
重新创建 DynamicMethod
  |
  v
CreateDelegate
  |
  v
调用一次
```

正确思路：

```text
第一次遇到类型和属性
  |
  v
生成 Getter / Setter 委托
  |
  v
放入缓存
  |
  v
后续直接调用委托
```

简单缓存示例：

```csharp
static readonly Dictionary<string, Func<object, object?>> GetterCache = new();

static Func<object, object?> GetOrCreateGetter(Type type, string propertyName)
{
    string key = type.FullName + "." + propertyName;

    if (GetterCache.TryGetValue(key, out Func<object, object?>? getter))
    {
        return getter;
    }

    getter = CreateGetter(type, propertyName);
    GetterCache[key] = getter;

    return getter;
}
```

多线程环境里可以换成：

```csharp
ConcurrentDictionary<string, Func<object, object?>>
```

### 常见错误

### 1. 栈不平衡

IL 是栈模型。

方法返回前，栈上的数据必须和返回值匹配。

例如返回 `int` 的方法，`ret` 前栈顶必须有一个 `int`。

错误示例：

```csharp
il.Emit(OpCodes.Ldarg_0);
il.Emit(OpCodes.Ldarg_1);
il.Emit(OpCodes.Ret);
```

这里压了两个参数，但没有做运算，返回时栈不符合预期。

轻则运行时报错，重则排查很痛苦。

### 2. 实例方法参数编号搞错

实例方法里：

```text
Ldarg_0 = this
Ldarg_1 = 第一个业务参数
Ldarg_2 = 第二个业务参数
```

静态方法里：

```text
Ldarg_0 = 第一个业务参数
Ldarg_1 = 第二个业务参数
```

动态实现接口时，`Add(int a, int b)` 是实例方法，所以加法要用：

```csharp
il.Emit(OpCodes.Ldarg_1);
il.Emit(OpCodes.Ldarg_2);
il.Emit(OpCodes.Add);
```

### 3. 值类型忘记 Box / Unbox

Getter 返回 `object` 时，值类型要装箱：

```csharp
il.Emit(OpCodes.Box, property.PropertyType);
```

Setter 接收 `object` 时，写入值类型属性前要拆箱：

```csharp
il.Emit(OpCodes.Unbox_Any, property.PropertyType);
```

### 4. 把 Emit 当业务开发工具

Emit 很强，但不适合普通业务代码。

这些场景通常没必要用 Emit：

* 普通增删改查
* 简单对象映射
* 一次性反射调用
* 配置读取
* 普通条件分支

能用普通 C# 写清楚，就不要上 Emit。

### 5. 忽略 AOT 和裁剪限制

Emit 属于运行时代码生成。

在 Native AOT、受限运行环境、强裁剪发布场景里，Emit 可能不可用或不适合。

如果目标是 AOT 友好，通常更应该考虑：

* 源生成器
* 手写静态代码
* 表达式树解释执行
* 框架提供的 AOT 模式

### 现代 .NET 里能保存动态程序集吗？

.NET Framework 时代可以用 `RunAndSave` 把动态程序集保存成文件。

现代 .NET 更常见的是内存运行：

```csharp
AssemblyBuilderAccess.Run
AssemblyBuilderAccess.RunAndCollect
```

如果只是为了调试 Emit，可以考虑这些方式：

* 先用普通 C# 写等价代码，再看 IL
* 用 SharpLab 查看 IL
* 把 Emit 逻辑拆成小方法逐段验证
* 给动态方法生成清晰的名称
* 尽量用单元测试覆盖动态生成逻辑

### Emit、Expression Tree、Source Generator 怎么选？

| 场景 | 更合适的选择 |
| --- | --- |
| 编译期就能知道要生成什么 | Source Generator |
| 运行时拼简单访问器或动态条件 | Expression Tree |
| 运行时生成高性能方法或类型 | Emit |
| AOT / 裁剪友好 | Source Generator |
| 临时读取元数据 | 反射 |
| 框架底层动态代理 | Emit 或 Castle DynamicProxy |

简单判断：

```text
普通业务代码：手写
低频动态调用：反射
中等复杂动态委托：表达式树
编译期生成代码：源生成器
运行时生成类型或极致控制 IL：Emit
```

### 总结

Emit 是 `.NET` 里最底层、最灵活的动态代码生成能力之一。

它可以：

* 生成动态方法
* 生成属性访问器
* 生成对象映射器
* 生成完整类型
* 生成接口实现
* 直接发出 IL 指令

但它也有明显代价：

* 写法复杂
* 调试困难
* 容易写错 IL
* 对 AOT 不友好
* 维护成本高

实战里的合理姿势是：

```text
把 Emit 放在框架底层和性能热点里。
生成一次，缓存起来，反复调用。
普通业务逻辑不要为了炫技使用 Emit。
```

理解 Emit 后，再看 ORM、Mapper、动态代理、Mock 框架的底层实现，会更容易明白它们为什么能在运行时“变出”方法和类型。
