### 简介

看 `.NET IL` 时，最容易卡住的地方通常不是指令名字，而是执行方式。

例如这几行 IL：

```il
ldarg.0
ldarg.1
add
ret
```

如果按 C# 的写法去看，会觉得很别扭：

```text
没有变量名
没有 a + b 这种表达式
add 也没有写操作数
ret 到底返回什么
```

原因很简单：

```text
.NET IL 是栈机模型。
```

一句话概括：

```text
栈机执行模型就是：指令把数据压入计算栈，再从栈顶弹出数据完成计算，结果再压回栈顶。
```

所以看 IL 的关键不是先背指令表，而是先学会盯住一个东西：

```text
Evaluation Stack
```

中文一般叫：

```text
计算栈
求值栈
操作数栈
```

### 栈机到底是什么？

普通 C# 代码看起来像这样：

```csharp
int result = a + b;
```

表达式里直接写了两个操作数：

```text
a
b
```

但 IL 的 `add` 指令不直接写操作数。

它默认从计算栈顶拿数据：

```text
栈顶弹出 b
栈顶弹出 a
计算 a + b
把结果压回栈顶
```

例如：

```text
初始栈：[]

ldarg.0  -> [a]
ldarg.1  -> [a, b]
add      -> [a + b]
ret      -> 返回 a + b
```

这就是栈机的核心思路。

### 栈机和寄存器机有什么区别？

现代 CPU 更接近寄存器机。

寄存器机的指令通常会显式写出操作数位置：

```asm
mov eax, 10
add eax, 20
```

这里的 `eax` 就是寄存器。

栈机不这样写。

栈机的指令更像：

```text
push 10
push 20
add
```

`add` 不写“加谁和谁”，因为默认就是操作栈顶两个值。

对比一下：

| 对比项 | 栈机 | 寄存器机 |
| --- | --- | --- |
| 操作数位置 | 隐式在栈顶 | 显式在寄存器或内存 |
| 指令形式 | 短一些 | 操作数更明确 |
| 编译器生成 | 相对简单 | 要考虑寄存器分配 |
| 验证类型安全 | 更容易 | 更复杂 |
| 最终执行 | 仍会被 JIT 转成本机代码 | CPU 直接执行 |

.NET IL 采用栈机模型，不代表 CPU 最终也按 IL 栈机执行。

真实流程是：

```text
C# 源码
  |
  v
C# 编译器生成 IL
  |
  v
CLR 加载程序集
  |
  v
JIT 把 IL 编译成本机代码
  |
  v
CPU 执行机器码
```

所以栈机模型主要是 IL 这一层的执行抽象。

### 三个容易混淆的“栈”

讲栈机时，经常会把几个概念混在一起。

### 1. 调用栈

调用栈记录方法调用关系。

例如：

```text
Main
  -> CreateOrder
      -> ValidateOrder
```

每调用一个方法，就会出现一个栈帧。

栈帧里会有：

* 参数
* 局部变量
* 返回地址
* 当前方法执行状态

### 2. 计算栈

计算栈是 IL 指令执行时用来放临时值的地方。

例如：

```il
ldc.i4.s 10
ldc.i4.s 20
add
```

执行过程：

```text
[] -> [10] -> [10, 20] -> [30]
```

计算栈属于当前方法的执行过程。

### 3. 托管堆

托管堆存放对象。

例如：

```csharp
var user = new User();
```

`User` 对象在托管堆上，局部变量里通常保存的是对象引用。

三者关系可以简化成：

```text
调用栈：记录方法怎么一层层调用
计算栈：记录当前方法里 IL 指令的临时操作数
托管堆：存放对象实例
```

### 第一个例子：return a + b

C# 代码：

```csharp
public static int Add(int a, int b)
{
    return a + b;
}
```

对应 IL 大致是：

```il
ldarg.0
ldarg.1
add
ret
```

逐步看栈：

| 指令 | 说明 | 执行后计算栈 |
| --- | --- | --- |
| 初始 | 空栈 | `[]` |
| `ldarg.0` | 加载参数 `a` | `[a]` |
| `ldarg.1` | 加载参数 `b` | `[a, b]` |
| `add` | 弹出两个值，相加后压栈 | `[a + b]` |
| `ret` | 返回栈顶值 | `[]` |

如果调用：

```csharp
Add(10, 20)
```

栈变化就是：

```text
[]
[10]
[10, 20]
[30]
return 30
```

### 第二个例子：局部变量怎么参与？

C# 代码：

```csharp
public static int Calc(int a, int b)
{
    int sum = a + b;
    return sum * 2;
}
```

IL 大致是：

```il
.locals init ([0] int32 sum)

ldarg.0
ldarg.1
add
stloc.0

ldloc.0
ldc.i4.2
mul
ret
```

这里出现了两个新东西：

```text
stloc.0：把栈顶值弹出，存到第 0 个局部变量
ldloc.0：把第 0 个局部变量加载到栈顶
```

假设调用：

```csharp
Calc(3, 4)
```

执行过程：

| 指令 | 计算栈 | 局部变量 |
| --- | --- | --- |
| 初始 | `[]` | `sum = ?` |
| `ldarg.0` | `[3]` | `sum = ?` |
| `ldarg.1` | `[3, 4]` | `sum = ?` |
| `add` | `[7]` | `sum = ?` |
| `stloc.0` | `[]` | `sum = 7` |
| `ldloc.0` | `[7]` | `sum = 7` |
| `ldc.i4.2` | `[7, 2]` | `sum = 7` |
| `mul` | `[14]` | `sum = 7` |
| `ret` | `[]` | 返回 `14` |

可以看到：

```text
局部变量不是计算栈本身。
局部变量需要通过 ldloc 加载到栈上，才能参与计算。
```

### 第三个例子：方法调用怎么走栈？

C# 代码：

```csharp
public static int Double(int value)
{
    return value * 2;
}

public static int Run()
{
    return Double(10);
}
```

`Run` 的 IL 大致是：

```il
ldc.i4.s 10
call int32 Demo::Double(int32)
ret
```

栈变化：

| 指令 | 说明 | 计算栈 |
| --- | --- | --- |
| 初始 | 空栈 | `[]` |
| `ldc.i4.s 10` | 压入常量 10 | `[10]` |
| `call Double` | 弹出参数，调用方法，把返回值压栈 | `[20]` |
| `ret` | 返回栈顶值 | `[]` |

方法调用的关键点：

```text
调用前：参数按顺序压栈
调用时：call 消耗参数
调用后：如果有返回值，返回值会压回栈顶
```

### 实例方法里的 ldarg.0 是什么？

静态方法里：

```text
ldarg.0 = 第一个参数
ldarg.1 = 第二个参数
```

实例方法里：

```text
ldarg.0 = this
ldarg.1 = 第一个业务参数
ldarg.2 = 第二个业务参数
```

例如：

```csharp
public class Counter
{
    private int _value;

    public void Add(int amount)
    {
        _value += amount;
    }
}
```

`Add` 方法里要访问字段 `_value`，就得先加载 `this`：

```il
ldarg.0
ldarg.0
ldfld int32 Counter::_value
ldarg.1
add
stfld int32 Counter::_value
ret
```

这里的栈变化可以拆成：

```text
ldarg.0       -> [this]
ldarg.0       -> [this, this]
ldfld _value  -> [this, 当前_value]
ldarg.1       -> [this, 当前_value, amount]
add           -> [this, 新_value]
stfld _value  -> []
```

`stfld` 需要两个东西：

```text
对象引用
要写入的新值
```

所以前面要先把 `this` 留在栈里。

### 装箱也是栈操作

C# 代码：

```csharp
int number = 123;
object value = number;
```

IL 大致是：

```il
ldc.i4.s 123
stloc.0
ldloc.0
box int32
stloc.1
```

栈变化：

| 指令 | 说明 | 计算栈 |
| --- | --- | --- |
| `ldc.i4.s 123` | 压入整数 | `[123]` |
| `stloc.0` | 保存到 `number` | `[]` |
| `ldloc.0` | 重新加载 `number` | `[123]` |
| `box int32` | 装箱成对象引用 | `[object]` |
| `stloc.1` | 保存到 `value` | `[]` |

`box` 做的事情不是简单改类型名，而是在托管堆上创建一个对象，把值复制进去，然后把对象引用压回计算栈。

### ret 返回什么？

`ret` 不写返回值变量。

返回值来自栈顶。

例如返回 `int` 的方法，执行到 `ret` 前，计算栈顶必须有一个 `int32`。

```il
ldc.i4.s 100
ret
```

表示：

```csharp
return 100;
```

如果方法返回 `void`，执行到 `ret` 前，计算栈通常应该是空的。

```il
call void [System.Console]System.Console::WriteLine(string)
ret
```

这也是 IL 验证要检查的重要内容之一：

```text
返回值类型和栈顶类型必须匹配。
```

### .maxstack 是什么？

IL 方法里经常能看到：

```il
.maxstack 2
```

它表示这个方法执行过程中，计算栈最多需要多深。

例如：

```il
ldarg.0
ldarg.1
add
ret
```

最大深度是 2：

```text
ldarg.0 -> [a]       深度 1
ldarg.1 -> [a, b]    深度 2
add     -> [a + b]   深度 1
```

所以 `.maxstack 2` 就够了。

现代工具和运行时通常能帮忙计算或验证，但理解 `.maxstack` 对手写 Emit 很有帮助。

### 可运行 Demo：写一个小型栈机解释器

下面写一个小型栈机，模拟几条 IL 指令。

支持这些指令：

```text
ldarg
ldc
add
sub
mul
stloc
ldloc
ret
```

创建项目：

```bash
mkdir StackMachineDemo
cd StackMachineDemo

dotnet new console
```

修改 `Program.cs`：

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

Instruction[] program =
[
    new(OpCode.LdArg, 0),
    new(OpCode.LdArg, 1),
    new(OpCode.Add),
    new(OpCode.StLoc, 0),
    new(OpCode.LdLoc, 0),
    new(OpCode.Ldc, 2),
    new(OpCode.Mul),
    new(OpCode.Ret)
];

var machine = new MiniStackMachine(argumentCount: 2, localCount: 1);

int result = machine.Execute(program, [3, 4]);

Console.WriteLine($"执行结果：{result}");

public enum OpCode
{
    LdArg,
    Ldc,
    Add,
    Sub,
    Mul,
    StLoc,
    LdLoc,
    Ret
}

public readonly record struct Instruction(OpCode Code, int Operand = 0);

public sealed class MiniStackMachine
{
    private readonly int[] _arguments;
    private readonly int[] _locals;
    private readonly Stack<int> _stack = new();

    public MiniStackMachine(int argumentCount, int localCount)
    {
        _arguments = new int[argumentCount];
        _locals = new int[localCount];
    }

    public int Execute(Instruction[] instructions, int[] arguments)
    {
        if (arguments.Length != _arguments.Length)
        {
            throw new ArgumentException("参数数量不匹配");
        }

        arguments.CopyTo(_arguments, 0);

        for (int ip = 0; ip < instructions.Length; ip++)
        {
            Instruction instruction = instructions[ip];

            Console.WriteLine($"执行前：{FormatStack(),-14} 指令：{FormatInstruction(instruction)}");

            switch (instruction.Code)
            {
                case OpCode.LdArg:
                    _stack.Push(_arguments[instruction.Operand]);
                    break;

                case OpCode.Ldc:
                    _stack.Push(instruction.Operand);
                    break;

                case OpCode.Add:
                {
                    int right = _stack.Pop();
                    int left = _stack.Pop();
                    _stack.Push(left + right);
                    break;
                }

                case OpCode.Sub:
                {
                    int right = _stack.Pop();
                    int left = _stack.Pop();
                    _stack.Push(left - right);
                    break;
                }

                case OpCode.Mul:
                {
                    int right = _stack.Pop();
                    int left = _stack.Pop();
                    _stack.Push(left * right);
                    break;
                }

                case OpCode.StLoc:
                    _locals[instruction.Operand] = _stack.Pop();
                    break;

                case OpCode.LdLoc:
                    _stack.Push(_locals[instruction.Operand]);
                    break;

                case OpCode.Ret:
                {
                    int result = _stack.Pop();
                    Console.WriteLine($"执行后：{FormatStack(),-14} 返回：{result}");
                    return result;
                }

                default:
                    throw new NotSupportedException($"不支持的指令：{instruction.Code}");
            }

            Console.WriteLine($"执行后：{FormatStack()}");
        }

        throw new InvalidOperationException("缺少 ret 指令");
    }

    private string FormatStack()
    {
        return "[" + string.Join(", ", _stack.Reverse()) + "]";
    }

    private static string FormatInstruction(Instruction instruction)
    {
        return instruction.Code switch
        {
            OpCode.LdArg => $"ldarg.{instruction.Operand}",
            OpCode.Ldc => $"ldc.i4 {instruction.Operand}",
            OpCode.StLoc => $"stloc.{instruction.Operand}",
            OpCode.LdLoc => $"ldloc.{instruction.Operand}",
            OpCode.Add => "add",
            OpCode.Sub => "sub",
            OpCode.Mul => "mul",
            OpCode.Ret => "ret",
            _ => instruction.Code.ToString()
        };
    }
}
```

运行：

```bash
dotnet run
```

输出类似：

```text
执行前：[]             指令：ldarg.0
执行后：[3]
执行前：[3]            指令：ldarg.1
执行后：[3, 4]
执行前：[3, 4]         指令：add
执行后：[7]
执行前：[7]            指令：stloc.0
执行后：[]
执行前：[]             指令：ldloc.0
执行后：[7]
执行前：[7]            指令：ldc.i4 2
执行后：[7, 2]
执行前：[7, 2]         指令：mul
执行后：[14]
执行前：[14]           指令：ret
执行后：[]             返回：14
执行结果：14
```

这个小 Demo 模拟的就是：

```csharp
int sum = a + b;
return sum * 2;
```

### 用 DynamicMethod 生成真实 IL

小型栈机是为了理解模型。

下面用真实 `DynamicMethod` 生成同样的逻辑：

```csharp
using System;
using System.Reflection.Emit;

Func<int, int, int> calc = CreateCalc();

Console.WriteLine(calc(3, 4));

static Func<int, int, int> CreateCalc()
{
    DynamicMethod method = new(
        name: "Calc",
        returnType: typeof(int),
        parameterTypes: new[] { typeof(int), typeof(int) });

    ILGenerator il = method.GetILGenerator();

    LocalBuilder sum = il.DeclareLocal(typeof(int));

    il.Emit(OpCodes.Ldarg_0);
    il.Emit(OpCodes.Ldarg_1);
    il.Emit(OpCodes.Add);
    il.Emit(OpCodes.Stloc, sum);

    il.Emit(OpCodes.Ldloc, sum);
    il.Emit(OpCodes.Ldc_I4_2);
    il.Emit(OpCodes.Mul);
    il.Emit(OpCodes.Ret);

    return method.CreateDelegate<Func<int, int, int>>();
}
```

输出：

```text
14
```

这段 Emit 和小型栈机 Demo 是同一套思路：

```text
ldarg.0
ldarg.1
add
stloc
ldloc
ldc.i4.2
mul
ret
```

区别只是：

* 小型栈机用 C# 模拟执行
* `DynamicMethod` 生成真正 CLR 可以执行的 IL

### 栈不平衡会怎样？

栈机有一个硬规则：

```text
每条指令执行前，栈上必须有足够的操作数。
方法返回时，栈状态必须和返回类型匹配。
```

例如这段 IL 思路就是错的：

```il
ldarg.0
add
ret
```

`add` 需要两个值，但栈里只有一个值。

再比如返回 `int` 的方法：

```il
ret
```

如果 `ret` 前栈是空的，也不合法。

小型栈机里也能看到类似错误。

把前面的程序改成：

```csharp
Instruction[] badProgram =
[
    new(OpCode.LdArg, 0),
    new(OpCode.Add),
    new(OpCode.Ret)
];
```

执行到 `Add` 时，`_stack.Pop()` 会因为栈里不够两个值而失败。

真实 CLR 在加载、验证或 JIT 阶段也会检查类似问题。

### 常见指令按栈行为分类

| 指令 | 栈行为 | 说明 |
| --- | --- | --- |
| `ldarg.*` | 压栈 | 加载参数 |
| `ldloc.*` | 压栈 | 加载局部变量 |
| `ldc.i4.*` | 压栈 | 加载整数常量 |
| `ldstr` | 压栈 | 加载字符串引用 |
| `stloc.*` | 弹栈 | 保存到局部变量 |
| `stfld` | 弹栈 | 写字段 |
| `add` | 弹两个，压一个 | 加法 |
| `sub` | 弹两个，压一个 | 减法 |
| `mul` | 弹两个，压一个 | 乘法 |
| `call` | 弹参数，可能压返回值 | 调用方法 |
| `callvirt` | 弹对象引用和参数，可能压返回值 | 调用实例方法 |
| `box` | 弹值类型，压对象引用 | 装箱 |
| `newobj` | 弹构造参数，压对象引用 | 创建对象 |
| `ret` | 弹返回值或空返回 | 方法返回 |

看 IL 时，可以把每条指令都翻译成：

```text
它压了什么？
它弹了什么？
它留下了什么？
```

### 为什么 C# 编译器喜欢这种模型？

栈模型有几个好处。

### 1. IL 更接近语言无关格式

.NET 不只支持 C#。

还有 F#、VB.NET 等语言。

这些语言都可以编译到 IL。

栈机指令不绑定具体 CPU 寄存器，更适合作为统一中间语言。

### 2. 编译器生成更简单

如果中间语言直接暴露寄存器，编译器就要处理寄存器分配。

栈机模型更像表达式求值：

```text
先把操作数放到栈上
再执行操作
结果留在栈上
```

这对前端语言编译器更友好。

### 3. 验证更容易

CLR 可以检查：

* 某条指令执行前栈深度够不够
* 栈顶类型对不对
* 方法返回值类型是否匹配
* 分支跳转后的栈状态是否一致
* 是否把错误类型传给方法调用

例如 `add` 需要数值类型。

如果栈顶是两个对象引用，就不符合要求。

### 4. JIT 还有优化空间

栈机只是 IL 层的模型。

JIT 编译成本机代码时，会把这些栈操作映射到寄存器、内存和机器指令上。

也就是说，最终 CPU 执行时，不一定真的按“压栈、弹栈”一步步运行。

### 栈机模型和性能有什么关系？

栈机模型本身不是性能问题的直接答案。

因为最终会经过 JIT 优化。

但理解它有助于识别这些行为：

* 装箱和拆箱
* 局部变量读写
* 方法调用参数压栈
* 虚调用和非虚调用
* 结构体复制
* `try/finally` 展开方式
* `async/await` 状态机生成方式

例如看到：

```il
box int32
```

就能知道这里发生了装箱。

看到：

```il
callvirt
```

就能进一步判断是否涉及虚调用或空引用检查。

### 常见误区

### 1. 计算栈不是托管堆

计算栈只是临时操作数栈。

对象实例仍然在托管堆上。

计算栈里经常保存的是对象引用，而不是完整对象。

### 2. 局部变量不等于计算栈

局部变量有自己的槽位。

要参与计算，需要先用 `ldloc` 压入计算栈。

要保存结果，需要用 `stloc` 从计算栈弹出。

### 3. ret 不是返回某个变量名

`ret` 返回的是栈顶值。

源码里写的是：

```csharp
return result;
```

IL 里通常是：

```il
ldloc.0
ret
```

也就是先把 `result` 加载到栈顶，再返回。

### 4. 栈机模型不等于 CPU 最终执行方式

IL 是栈机。

JIT 后的机器码会使用真实 CPU 的寄存器和内存。

所以不要把 IL 栈变化直接等同于最终机器码开销。

### 5. callvirt 不一定只表示虚调用

C# 编译器经常对普通实例方法也生成 `callvirt`。

原因之一是它可以顺带做空引用检查。

所以看到 `callvirt` 时，不能直接断定一定发生了多态虚派发。

### 实战建议

看 IL 时可以按这套顺序来：

```text
先找方法签名
再看参数和局部变量
逐行跟踪计算栈
遇到 call 看它消耗哪些参数
遇到 ret 看栈顶是什么
遇到 box / unbox 看是否有值类型转换
遇到 branch 看不同路径的栈状态是否一致
```

如果只是为了日常业务开发，不需要背所有 IL 指令。

但掌握栈机模型后，再看这些内容会清楚很多：

* IL 中间码
* Reflection.Emit
* 表达式树编译
* JIT 优化
* 装箱拆箱
* async/await 状态机
* 动态代理和 ORM 底层实现

### 总结

.NET IL 的栈机执行模型可以用一句话记住：

```text
数据先压栈，指令从栈顶取数据，结果再压回栈顶。
```

理解这件事后，很多 IL 指令都会变得直观：

* `ldarg` 是把参数压栈
* `ldloc` 是把局部变量压栈
* `stloc` 是把栈顶保存到局部变量
* `add` 是弹出两个值再压回一个结果
* `call` 是弹出参数并压回返回值
* `ret` 是返回栈顶值

栈机模型不是为了替代 C# 编程，而是为了看懂 C# 编译后的真实形态。

看懂这层模型后，IL、Emit、JIT、装箱、动态代理这些内容都会更容易串起来。
