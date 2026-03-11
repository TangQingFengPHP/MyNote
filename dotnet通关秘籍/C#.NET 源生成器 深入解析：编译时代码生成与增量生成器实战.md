### 简介

在 `.NET` 项目里，很多“重复但又不能随便写错”的代码，本质上都不值得手写。

例如：

* DTO 映射代码；
* `INotifyPropertyChanged` 模板代码；
* 接口注册代码；
* 序列化上下文；
* 特性驱动的样板方法；
* 各种“按规则扫描代码再生成辅助类”的场景。

过去这类问题通常有 3 种解法：

* 手写，最直接，但重复劳动多；
* 运行时反射，灵活，但有开销；
* T4 / 脚本 / CLI 代码生成，能用，但维护成本高。

`Source Generator` 的出现，本质上就是给这类问题一个更现代的答案：

> 在编译期间分析你的代码，并自动生成新的 `C#` 源文件，让它们和手写代码一起参与编译。

这就是为什么源生成器看起来像“高级编译器功能”，但实际项目里它非常务实。

### Source Generator 到底是什么？

`Source Generator` 是 `Roslyn` 编译器扩展机制的一部分。

它做的事情可以简化成：

```text
你的源码 -> Roslyn 分析语法树和语义信息
         -> Source Generator 读取这些信息
         -> 生成新的 .cs 代码
         -> 编译器把手写代码和生成代码一起编译
```

这意味着几个关键点：

* 它运行在编译期，不是运行期；
* 它不会修改你原来的文件；
* 它只能“新增源代码”，不能直接篡改已有代码；
* 生成出来的代码会参与编译、报错、补全和类型检查。

所以你可以把它理解成：

* 不是“运行时代码注入”；
* 而是“编译时代码补齐”。

### 为什么源生成器值得学？

因为它解决的是一类很典型、很现实的问题：

* 手写太机械；
* 反射太慢或不利于 AOT；
* 手工同步容易出错；
* 模板代码和业务代码长期分离后容易失真。

几个非常典型的场景：

| 场景 | Source Generator 能做什么 |
| --- | --- |
| `System.Text.Json` | 生成序列化元数据，减少反射，提高性能 |
| DI 自动注册 | 扫描标记类型并生成注册代码 |
| DTO / Mapper | 根据特性生成映射器或 DTO |
| `INotifyPropertyChanged` | 根据字段或属性生成通知逻辑 |
| Native AOT | 生成静态可见代码，减少运行时反射依赖 |
| SDK/客户端生成 | 根据协议或元数据生成强类型代码 |

一句话总结：

* 运行时反射更灵活；
* 编译时源生成器更静态、更快、更适合类型安全和 AOT。

### Source Generator 和反射、Expression Tree、Source Generator 之外的代码生成，有什么区别？

这几个东西经常被混在一起。

#### 和反射的区别

* 反射是在运行时读取类型信息；
* 源生成器是在编译时分析代码并输出静态代码。

如果你的需求是“运行时再决定做什么”，反射仍然有价值。

如果你的需求是“编译时就知道规则，想生成固定代码”，源生成器通常更合适。

#### 和表达式树的区别

* 表达式树主要解决“把代码表示成对象结构，并在运行时分析或拼接”；
* 源生成器主要解决“在编译时生成新的源码”。

两者都和“代码生成”有关，但时间点完全不同。

#### 和 T4 / 手工脚本生成的区别

* T4、脚本、命令行模板常常是编译流程之外的外部步骤；
* 源生成器直接集成进 Roslyn 编译流程里。

这带来的优势很直接：

* 不容易忘记重新生成；
* IDE 体验更统一；
* 类型系统和错误提示更自然。

### 源生成器的工作方式

如果只记一条主线，记这个就够了：

```text
源码 -> 语法树 -> 语义分析 -> Generator 执行 -> AddSource -> 合并编译
```

源生成器能拿到的核心信息包括：

* `SyntaxTree`：语法树
* `SemanticModel`：语义模型
* `Compilation`：当前项目编译上下文
* `INamedTypeSymbol` / `ISymbol`：类型、方法、属性等符号信息

这些信息足够你回答很多问题：

* 哪些类带了某个特性？
* 某个属性是什么类型？
* 这个类是不是 `partial`？
* 这个命名空间是什么？
* 这个接口是否被实现？

然后你就可以决定生成什么代码。

### 两种源生成器：`ISourceGenerator` 和 `IIncrementalGenerator`

这是今天写源生成器最重要的一个区分。

#### 1. 传统生成器：`ISourceGenerator`

接口大致长这样：

```csharp
public interface ISourceGenerator
{
    void Initialize(GeneratorInitializationContext context);
    void Execute(GeneratorExecutionContext context);
}
```

它的特点：

* 模型直观；
* 容易理解；
* 适合教学和简单场景；
* 但每次编译往往是全量思维，容易重复计算。

#### 2. 增量生成器：`IIncrementalGenerator`

接口大致长这样：

```csharp
public interface IIncrementalGenerator
{
    void Initialize(IncrementalGeneratorInitializationContext context);
}
```

它的特点：

* 通过增量管道组织输入和输出；
* 只有相关输入发生变化时才重新计算；
* 更适合真实项目和大型代码库；
* 编译性能通常更好。

今天的务实建议非常明确：

* 想理解原理，可以先看 `ISourceGenerator`；
* 真正在项目里写新生成器，优先考虑 `IIncrementalGenerator`。

### 一个最小可运行示例：生成 HelloWorld 类

先看最小版本，这样最容易建立直觉。

#### 第一步：创建生成器项目

通常创建一个类库项目：

```bash
dotnet new classlib -n MySourceGenerator
```

项目文件可以这样配置：

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.11.0" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

之所以通常选 `netstandard2.0`，是因为它对生成器项目的兼容性最好。

#### 第二步：写一个最简单的生成器

```csharp
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.Text;

[Generator]
public sealed class HelloWorldGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
    }

    public void Execute(GeneratorExecutionContext context)
    {
        const string source = """
namespace Generated;

public static class HelloWorld
{
    public static string SayHello() => "Hello from Source Generator";
}
""";

        context.AddSource(
            "HelloWorld.g.cs",
            SourceText.From(source, Encoding.UTF8));
    }
}
```

这里最关键的一行是：

```csharp
context.AddSource("HelloWorld.g.cs", ...);
```

它的意思不是“写磁盘文件”，而是“把这段源码加入当前编译”。

#### 第三步：在业务项目里引用它

业务项目里引用生成器时，重点不是普通程序集引用，而是作为 analyzer 引入：

```xml
<ItemGroup>
  <ProjectReference Include="..\MySourceGenerator\MySourceGenerator.csproj"
                    OutputItemType="Analyzer"
                    ReferenceOutputAssembly="false" />
</ItemGroup>
```

然后你就可以在业务代码里直接使用生成出来的类型：

```csharp
Console.WriteLine(Generated.HelloWorld.SayHello());
```

### 为什么很多生成器都要求 `partial`？

这是非常常见的模式。

例如你会看到用户代码写成：

```csharp
[AutoNotify]
public partial class Person
{
    private string _name = string.Empty;
}
```

生成器再额外生成：

```csharp
public partial class Person
{
    public string Name
    {
        get => _name;
        set
        {
            _name = value;
            OnPropertyChanged(nameof(Name));
        }
    }
}
```

之所以用 `partial`，是因为：

* 手写代码和生成代码本质上是同一个类型的不同部分；
* 生成器不能直接改你原来的类；
* 所以最自然的办法就是生成另一个 `partial` 片段。

这也是大多数源生成器设计的基本套路。

### 从“写死生成”走向“按规则生成”

真正有意义的生成器，当然不是每次都无脑输出一个固定类，而是：

* 扫描代码中感兴趣的目标；
* 分析它们的符号信息；
* 再按规则生成对应代码。

例如：扫描带 `[GenerateToString]` 特性的类。

用户代码：

```csharp
[GenerateToString]
public partial class User
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}
```

生成器逻辑大致要做这些事：

* 找到所有类声明；
* 判断类上是否有这个特性；
* 获取属性列表；
* 生成 `ToString()` 或辅助方法。

这时候，语法树和语义模型就派上用场了。

### 传统生成器的典型写法

先看 `ISyntaxReceiver` 版本，因为它最容易理解。

```csharp
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp.Syntax;

[Generator]
public sealed class DemoGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
        context.RegisterForSyntaxNotifications(() => new SyntaxReceiver());
    }

    public void Execute(GeneratorExecutionContext context)
    {
        if (context.SyntaxReceiver is not SyntaxReceiver receiver)
        {
            return;
        }

        foreach (var classNode in receiver.CandidateClasses)
        {
            var model = context.Compilation.GetSemanticModel(classNode.SyntaxTree);
            var symbol = model.GetDeclaredSymbol(classNode);

            if (symbol is null)
            {
                continue;
            }

            // 这里可以继续判断特性、命名空间、成员等
        }
    }
}

internal sealed class SyntaxReceiver : ISyntaxReceiver
{
    public List<ClassDeclarationSyntax> CandidateClasses { get; } = new();

    public void OnVisitSyntaxNode(SyntaxNode syntaxNode)
    {
        if (syntaxNode is ClassDeclarationSyntax classNode)
        {
            CandidateClasses.Add(classNode);
        }
    }
}
```

这里的逻辑可以拆成两步：

* `SyntaxReceiver` 先粗筛：哪些语法节点值得关注；
* `Execute` 再结合语义模型细筛：这些类到底是不是目标类型。

这是传统生成器最经典的写法。

### 增量生成器为什么更值得用？

因为真实项目里，编译性能很重要。

如果每次小改一个文件，你的生成器都全量重新扫描和全量重新拼接代码，IDE 体验会明显变差。

增量生成器的核心思路是：

* 把“发现目标”
* “提取信息”
* “生成代码”

拆成一条增量数据管道。

只有输入变化的部分，才会重新计算。

### 一个增量生成器的最小示例

```csharp
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Text;

[Generator]
public sealed class ClassNameListGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var classDeclarations = context.SyntaxProvider.CreateSyntaxProvider(
            predicate: static (node, _) => node is ClassDeclarationSyntax,
            transform: static (ctx, _) => (ClassDeclarationSyntax)ctx.Node);

        context.RegisterSourceOutput(classDeclarations, static (spc, classNode) =>
        {
            var className = classNode.Identifier.Text;

            var source = $$"""
namespace Generated;

public static class {{className}}Info
{
    public const string Name = "{{className}}";
}
""";

            spc.AddSource($"{className}.Info.g.cs", SourceText.From(source, Encoding.UTF8));
        });
    }
}
```

虽然这个例子很简单，但它已经体现了增量生成器的基本风格：

* 先声明输入源；
* 再定义转换过程；
* 最后注册输出。

### 一个更接近实战的例子：根据特性生成方法

下面这个示例会稍微更真实一些。

目标是：

* 让用户给类打一个 `[GenerateGreeting]` 特性；
* 生成器自动为这个类补一个 `SayHello()` 方法。

#### 用户侧代码

```csharp
namespace Demo;

[GenerateGreeting]
public partial class UserService
{
}
```

#### 特性定义

```csharp
using System;

[AttributeUsage(AttributeTargets.Class)]
public sealed class GenerateGreetingAttribute : Attribute
{
}
```

#### 增量生成器

```csharp
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Text;

[Generator]
public sealed class GreetingGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var candidates = context.SyntaxProvider.CreateSyntaxProvider(
            predicate: static (node, _) => node is ClassDeclarationSyntax cds && cds.AttributeLists.Count > 0,
            transform: static (ctx, _) =>
            {
                var classNode = (ClassDeclarationSyntax)ctx.Node;
                var symbol = ctx.SemanticModel.GetDeclaredSymbol(classNode) as INamedTypeSymbol;
                return symbol;
            })
            .Where(static symbol => symbol is not null);

        context.RegisterSourceOutput(candidates, static (spc, symbol) =>
        {
            if (symbol is null)
            {
                return;
            }

            var hasAttribute = symbol.GetAttributes()
                .Any(a => a.AttributeClass?.Name == "GenerateGreetingAttribute");

            if (!hasAttribute)
            {
                return;
            }

            var namespaceName = symbol.ContainingNamespace.IsGlobalNamespace
                ? null
                : symbol.ContainingNamespace.ToDisplayString();

            var source = $$"""
{{(namespaceName is null ? "" : $"namespace {namespaceName};")}}

public partial class {{symbol.Name}}
{
    public string SayHello()
    {
        return "Hello from generated code";
    }
}
""";

            spc.AddSource($"{symbol.Name}.Greeting.g.cs", SourceText.From(source, Encoding.UTF8));
        });
    }
}
```

这个例子虽然简单，但已经足够说明大部分业务生成器的核心套路：

* 通过特性声明意图；
* 生成器扫描这些特性；
* 再生成 `partial` 代码补到目标类型上。

#### 这段增量生成器代码，逐行到底在做什么？

很多人第一次看增量生成器，不是卡在“概念”，而是卡在这几个 API 名字：

* `CreateSyntaxProvider`
* `predicate`
* `transform`
* `RegisterSourceOutput`

它们看起来很像编译器黑话，但其实这段代码做的事情非常朴素：

> 从所有语法节点里找出“带特性的类”，拿到它们的类型符号，再判断是不是带了目标特性，如果是，就生成代码。

可以把整段逻辑先压缩成一条流水线：

```text
所有语法节点
-> 粗筛出“带特性的类”
-> 把类语法节点转成类型符号
-> 过滤空值
-> 检查是否真的带目标特性
-> 生成对应的 partial 代码
```

下面按执行顺序逐行拆。

#### `using` 部分

```csharp
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Text;
```

作用分别是：

* `System.Text`：主要为了 `Encoding.UTF8`
* `Microsoft.CodeAnalysis`：Roslyn 核心 API，比如 `IIncrementalGenerator`、`INamedTypeSymbol`
* `Microsoft.CodeAnalysis.CSharp.Syntax`：C# 语法节点类型，比如 `ClassDeclarationSyntax`
* `Microsoft.CodeAnalysis.Text`：`SourceText`，用于把字符串包装成编译器接受的源码对象

#### `[Generator]`

```csharp
[Generator]
```

这个特性告诉 Roslyn：

* 这个类是一个源生成器
* 编译器需要在编译阶段加载并执行它

没有这个标记，编译器不会把它当作生成器。

#### 类声明

```csharp
public sealed class GreetingGenerator : IIncrementalGenerator
```

这里有两个重点：

* 这是一个源生成器
* 它使用的是增量模型 `IIncrementalGenerator`

这意味着它不是每次编译都用“全量扫描 -> 全量生成”的思路，而是把处理过程拆成若干增量步骤，只在输入变化时重新计算对应部分。

#### `Initialize(...)`

```csharp
public void Initialize(IncrementalGeneratorInitializationContext context)
```

增量生成器只需要实现这一个方法。

但这里要注意，它不是“直接开始生成代码”的方法，而更像是：

* 定义输入从哪里来
* 定义输入如何筛选和转换
* 定义最后如何产出生成代码

也就是说，`Initialize` 本质上是在搭一条增量处理管道。

### 第 1 步：建立候选输入 `candidates`

```csharp
var candidates = context.SyntaxProvider.CreateSyntaxProvider(
```

这行可以理解成：

* 从编译器能看到的所有语法节点中，构建一个“候选数据流”
* 之后这条数据流会不断产出可能和生成逻辑有关的节点

这里的 `candidates` 不是最终结果，而是后续生成代码要用的一批候选对象。

#### `predicate`

```csharp
predicate: static (node, _) => node is ClassDeclarationSyntax cds && cds.AttributeLists.Count > 0,
```

这一行是在做“粗筛”。

它的意思是：

* 如果当前语法节点是一个类声明 `ClassDeclarationSyntax`
* 并且这个类上写了特性列表
* 那它就是候选项

注意，这里并没有判断：

* 它是不是带了 `GenerateGreetingAttribute`

这里只是先把完全不可能相关的节点排掉，比如：

* 方法
* 属性
* 接口
* 没有任何特性的类

这样做的原因很简单：

* `predicate` 阶段要尽量便宜
* 它适合做快速语法级筛选
* 不适合在这里做太重的语义分析

再看这个 `static`：

```csharp
static (node, _) => ...
```

它表示这个 lambda 不捕获外部变量，在增量生成器里这样写更常见，也更利于性能。

第二个参数 `_` 这里没用，所以直接忽略。

#### `transform`

```csharp
transform: static (ctx, _) =>
{
    var classNode = (ClassDeclarationSyntax)ctx.Node;
    var symbol = ctx.SemanticModel.GetDeclaredSymbol(classNode) as INamedTypeSymbol;
    return symbol;
})
```

这一步是在做“从语法节点到语义符号”的转换。

前面的 `predicate` 只是告诉我们：

* 这个节点看起来像一个“带特性的类”

但这里才真正把它变成可以深入分析的类型符号。

先看第一行：

```csharp
var classNode = (ClassDeclarationSyntax)ctx.Node;
```

因为前面的 `predicate` 已经保证当前节点是类声明，所以这里可以安全转成 `ClassDeclarationSyntax`。

你可以把 `classNode` 理解成：

* 代码长相层面的类节点

它知道：

* 这是一个类
* 类名是什么
* 写了哪些特性语法

但它还不是“真正的类型定义对象”。

再看第二行：

```csharp
var symbol = ctx.SemanticModel.GetDeclaredSymbol(classNode) as INamedTypeSymbol;
```

这行非常关键。

它做的事是：

* 根据语义模型，把类语法节点转换成类型符号 `INamedTypeSymbol`

为什么一定要转成 `symbol`？

因为很多真正重要的信息，语法节点本身并不适合直接判断，例如：

* 这个类完整命名空间是什么
* 它身上的特性到底是什么真实类型
* 它有哪些属性、方法、基类、接口
* 它是不是泛型类

这些通常都更适合从 `INamedTypeSymbol` 读取。

所以到这一步，生成器已经从“看代码长相”升级到“理解代码语义”了。

最后：

```csharp
return symbol;
```

意味着后续增量管道里流动的，不再是类语法节点，而是类的类型符号。

#### `.Where(...)`

```csharp
.Where(static symbol => symbol is not null);
```

这是在做空值过滤。

因为 `GetDeclaredSymbol(...)` 理论上可能拿不到结果，所以这里把空值剔掉。

经过这一步之后，`candidates` 可以理解成：

* 一串非空的候选类类型符号

### 第 2 步：把候选项变成真正的源代码输出

```csharp
context.RegisterSourceOutput(candidates, static (spc, symbol) =>
```

这一行的意思是：

* 针对 `candidates` 这条增量数据流
* 给它注册一个最终输出动作
* 当有候选项进入这一步时，就执行这里面的生成逻辑

这里两个参数的含义：

* `spc`：`SourceProductionContext`，主要用来 `AddSource`
* `symbol`：当前这一项候选类的类型符号

换句话说，这块就是“真正生成代码”的地方。

#### 防御式判空

```csharp
if (symbol is null)
{
    return;
}
```

严格说，前面 `.Where(...)` 已经做过空值过滤了，这里再判一次更多是防御式写法。

不是必须，但写上也没问题。

#### 判断目标特性

```csharp
var hasAttribute = symbol.GetAttributes()
    .Any(a => a.AttributeClass?.Name == "GenerateGreetingAttribute");
```

这一段的作用是：

* 读取当前类上的所有特性
* 判断是否真的带了目标特性 `GenerateGreetingAttribute`

这里为什么不在前面的 `predicate` 就直接判断？

因为前面的 `predicate` 是语法级快速筛选，只知道：

* 它写了特性

但不知道：

* 这个特性在语义上究竟是不是 `GenerateGreetingAttribute`

真正判断特性类型，更适合在拿到 `symbol` 之后做。

这也是增量生成器很常见的套路：

* 先便宜地粗筛
* 再在语义层面精筛

#### 如果不是目标类，就直接跳过

```csharp
if (!hasAttribute)
{
    return;
}
```

这很简单：

* 不是我们要处理的类
* 就不要生成代码

也就是说，虽然 `candidates` 是“带特性的类”，但只有其中真正带目标特性的类才会继续往下走。

### 第 3 步：通常接下来会做什么？

虽然示例后半段代码你已经能看到，但从逻辑上可以直接总结成 3 件事：

#### 1. 读取类的命名空间和名称

通常会有类似代码：

```csharp
var namespaceName = symbol.ContainingNamespace.IsGlobalNamespace
    ? null
    : symbol.ContainingNamespace.ToDisplayString();
```

作用是：

* 让生成代码回到原来的命名空间里
* 避免用户代码和生成代码命名空间对不上

类名一般直接用：

```csharp
symbol.Name
```

#### 2. 拼出源代码字符串

通常会有类似：

```csharp
var source = $$"""
namespace Demo;

public partial class UserService
{
    public string SayHello()
    {
        return "Hello from generated code";
    }
}
""";
```

这一步就是把分析结果转成最终要加入编译的 `.cs` 内容。

为什么通常写成 `partial class`？

因为源生成器不能修改你已有的类，只能再补一份同名 `partial` 类代码进去。

#### 3. 把源码交给编译器

最后一般是：

```csharp
spc.AddSource(
    $"{symbol.Name}.Greeting.g.cs",
    SourceText.From(source, Encoding.UTF8));
```

这一步的意思不是“往磁盘写文件”，而是：

* 把这段源码加入当前编译
* 让编译器把它和手写代码一起编译

其中：

* `${symbol.Name}.Greeting.g.cs` 是 hint name，用来标识这份生成文件
* `SourceText.From(...)` 是把字符串包装成源码对象

### 为什么这段代码体现了“增量”？

因为它不是简单的“每次都全量扫描 + 全量生成”，而是把流程拆成了可缓存、可复用的几个阶段：

* 语法粗筛
* 语义转换
* 空值过滤
* 最终输出

一旦输入没有变化，很多步骤就不需要重新完整执行。

这就是增量生成器相对于传统生成器最有价值的地方：

* 对大型项目更友好
* 对 IDE 更友好
* 对频繁增量编译更友好

### 再用一句人话总结这段代码

如果把这段增量生成器翻译成最直白的话，它做的事情就是：

1. 先从所有语法节点里，挑出“带特性的类”
2. 把这些类转换成真正可分析的类型符号
3. 再判断它们是不是带了 `GenerateGreetingAttribute`
4. 如果是，就为它们生成一份 `partial` 代码
5. 最后把生成代码交给编译器一起编译

当你这样理解之后，`CreateSyntaxProvider` 和 `RegisterSourceOutput` 就没那么神秘了，它们本质上只是：

* 一个负责建立输入管道
* 一个负责注册最终输出

中间再加上筛选和转换而已。

### 源生成器特别适合哪些场景？

如果你在项目里遇到这些问题，源生成器往往值得考虑。

#### 1. 大量重复模板代码

例如：

* DTO
* 配置绑定辅助类
* API 客户端
* 事件包装器

#### 2. 运行时反射太多

例如：

* 序列化
* DI 自动发现
* 元数据访问器
* AOT 场景下的反射替代

#### 3. 规则明确、编译期就能知道

如果规则在编译时就能确定，那就非常适合源生成器。

反过来说，如果必须等到运行时才知道输入，源生成器就不一定适合。

### 源生成器的几个真实优点

#### 1. 降低运行时反射开销

这是最常见的收益。

特别是序列化、注册、元数据访问等路径。

#### 2. 生成代码同样受编译器保护

它不是字符串模板吐出来就完事，而是最终真的参与编译。

所以：

* 有类型检查；
* 有编译错误；
* 有 IDE 补全；
* 有重构联动的基础。

#### 3. 更适合 AOT 和裁剪场景

因为很多依赖反射的动态行为，在 AOT / trimming 场景下天然更脆。

源生成器生成的是静态代码，这方面通常更友好。

### 也别把源生成器想得太万能

它确实很有用，但不是什么问题都应该上。

#### 1. 它不能修改现有代码

你只能新增代码，不能直接把用户手写的方法改掉。

#### 2. 它的调试和维护成本不低

相比普通业务代码，生成器有更强的工具链和 Roslyn 依赖。

如果只是为了省 20 行样板代码，未必划算。

#### 3. 不是所有问题都适合编译期解决

如果信息只有运行时才知道，那源生成器帮不上忙。

#### 4. 生成代码本身也需要设计质量

糟糕的生成器会制造：

* 难读的 `.g.cs`
* 不稳定的文件命名
* 重复生成
* IDE 卡顿
* 增量失效

所以写生成器不是“只会拼字符串”就够了。

### 调试和查看生成代码

这是写源生成器时几乎必备的技能。

#### 1. 到 `obj` 目录看生成结果

生成代码通常可以在类似目录里看到：

```text
obj/Debug/net8.0/generated/
```

不同 SDK 和 IDE 展示方式略有差异，但本质都是中间生成文件。

#### 2. 文件名建议统一 `.g.cs`

例如：

* `UserService.Greeting.g.cs`
* `Person.AutoNotify.g.cs`
* `JsonContext.g.cs`

这样最容易区分手写代码和生成代码。

#### 3. 生成代码要尽量稳定

所谓稳定，主要指：

* 同样输入生成同样输出；
* 文件名可预测；
* 不要每次编译都乱改格式和顺序。

否则会严重影响调试体验。

### 写源生成器时最容易踩的坑

#### 1. 忘记让目标类型加 `partial`

如果你要补的是同一个类或结构体，通常就必须是 `partial`。

#### 2. 只看语法，不看语义

只靠字符串匹配类名、属性名，通常不够稳。

很多时候你最终需要的是 `ISymbol`，而不是纯语法节点。

#### 3. 重复生成同一个文件

`AddSource` 的 hint name 要稳定且唯一。

否则就会冲突。

#### 4. 生成器做了太多无谓工作

这是传统生成器最容易出现的问题。

如果项目变大，IDE 性能会明显受影响。

所以再次强调：

* 新项目优先增量生成器。

#### 5. 把所有逻辑都塞进字符串拼接

业务简单时这样还行；
一旦代码模板复杂，最好抽出：

* 模型层
* 渲染层
* 命名规则
* 公共帮助方法

否则生成器自己会很快变得不可维护。

### 一套比较务实的建议

如果你打算在项目里真正使用源生成器，下面这些建议会比较有用：

* 小型演示先用 `ISourceGenerator` 理解原理；
* 真实项目优先 `IIncrementalGenerator`；
* 通过特性或约定声明“生成意图”；
* 生成代码尽量走 `partial` 扩展，而不是奇怪的旁路类型；
* 生成文件名统一、稳定、可预测；
* 对高频编译场景，尽量减少全量扫描和重复字符串拼接；
* 把“编译期适合解决的问题”和“运行时才知道的问题”分清楚。

### 总结

源生成器的本质，不是“自动造代码这么简单”，而是把一部分原本放在运行时、手工维护、或外部脚本里的工作，前移到编译期来做。

你可以这样理解它：

* 反射是在运行时读元数据；
* 表达式树是在运行时拼代码结构；
* 源生成器是在编译时直接生成新的源码。

在今天的 `.NET` 项目里，尤其涉及这些场景时，源生成器非常值得掌握：

* 编译期自动补代码；
* 减少反射；
* 提升 AOT 兼容性；
* 降低样板代码；
* 做框架或基础设施扩展。

如果你把它当成“Roslyn 里的编译期自动化工具”，通常就不会理解偏。 
