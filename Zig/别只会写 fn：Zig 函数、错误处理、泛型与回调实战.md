# 别只会写 fn：Zig 函数、错误处理、泛型与回调实战

函数不只是把几行代码起一个名字。在 Zig 里，函数签名还会直接说明参数类型、返回类型、是否可能失败、是否需要编译期参数，以及是否允许修改外部数据。

函数和指针、错误联合、struct、enum、comptime 组合起来，就能覆盖大部分业务代码中的计算、校验、状态修改和策略分发。

本文按 Zig 0.16.0 语法编写，示例可以保存为 main.zig 后运行。

## 一、函数的基本形状

~~~zig
const std = @import("std");

fn add(a: i32, b: i32) i32 {
    return a + b;
}

fn logMessage(message: []const u8) void {
    std.debug.print("[LOG] {s}\n", .{message});
}

pub fn main() void {
    const sum = add(10, 20);
    std.debug.print("sum = {}\n", .{sum});
    logMessage("hello zig");
}
~~~

函数声明可以拆成四部分：

~~~text
fn add(a: i32, b: i32) i32
│  │   │                  │
│  │   │                  └ 返回类型
│  │   └ 参数及参数类型
│  └ 函数名
└ 函数声明
~~~

Zig 通常要求直接写出返回类型。无返回值函数使用 void，调用方不需要从函数体中猜测结果。

## 二、参数默认是只读的

普通参数不能在函数内部重新赋值：

~~~zig
fn bad(value: i32) void {
    // value += 1; // 编译错误：参数绑定不可修改
}
~~~

需要修改调用方的数据时，传入指针：

~~~zig
const std = @import("std");

fn increment(value: *i32) void {
    value.* += 1;
}

pub fn main() void {
    var number: i32 = 10;
    increment(&number);
    std.debug.print("number = {}\n", .{number});
}
~~~

这里的 &number 取得地址，value.* 访问指针指向的数据。函数签名中的 *i32 已经表达出“这个函数可能修改外部整数”。

切片也遵循可写和只读的区别：

~~~zig
fn fillFirst(buffer: []u8, value: u8) void {
    if (buffer.len > 0) {
        buffer[0] = value;
    }
}
~~~

[]u8 表示元素可写，[]const u8 表示元素只读。切片参数本身是一个小的描述结构，包含数据指针和长度。

## 三、没有默认参数，也没有函数重载

下面的写法不是 Zig 支持的默认参数：

~~~zig
// fn greet(name: []const u8 = "world") void {}
~~~

可选参数使用 optional：

~~~zig
const std = @import("std");

fn greet(name: ?[]const u8) void {
    const actual_name = name orelse "world";
    std.debug.print("hello, {s}\n", .{actual_name});
}

pub fn main() void {
    greet(null);
    greet("zig");
}
~~~

参数较多时，使用 struct 更清楚：

~~~zig
const PrintOptions = struct {
    prefix: []const u8 = "",
    newline: bool = true,
};

fn printText(text: []const u8, options: PrintOptions) void {
    std.debug.print("{s}{s}", .{ options.prefix, text });
    if (options.newline) std.debug.print("\n", .{});
}
~~~

Zig 也不允许定义两个同名但参数不同的函数。需要不同类型时，可以使用不同函数名、anytype、comptime T: type，或者使用 union 表达多种输入。

## 四、返回值：值、结构体和 optional

单个结果直接返回：

~~~zig
fn square(value: i32) i32 {
    return value * value;
}
~~~

多个有关联的结果，通常使用结构体：

~~~zig
const std = @import("std");

const DivResult = struct {
    quotient: i32,
    remainder: i32,
};

fn divMod(a: i32, b: i32) DivResult {
    return .{
        .quotient = @divTrunc(a, b),
        .remainder = @rem(a, b),
    };
}

pub fn main() void {
    const result = divMod(17, 5);
    std.debug.print("quotient = {}, remainder = {}\n", .{
        result.quotient,
        result.remainder,
    });
}
~~~

也可以返回匿名结构体：

~~~zig
fn bounds() struct { min: i32, max: i32 } {
    return .{ .min = 10, .max = 90 };
}
~~~

多个函数需要共享结果类型时，命名结构体更容易维护。

optional 表示“可能没有值”：

~~~zig
fn findEven(values: []const i32) ?i32 {
    for (values) |value| {
        if (@rem(value, 2) == 0) return value;
    }
    return null;
}
~~~

查找不到通常适合返回 null；解析失败、权限不足或除数为零，则更适合返回错误。

## 五、错误联合：把失败路径写进签名

返回类型前加 !，表示成功值或错误：

~~~zig
const ParseError = error{
    EmptyInput,
    InvalidNumber,
    OutOfRange,
};

fn parsePercent(text: []const u8) ParseError!u8 {
    if (text.len == 0) return error.EmptyInput;

    const value = std.fmt.parseInt(u16, text, 10) catch {
        return error.InvalidNumber;
    };

    if (value > 100) return error.OutOfRange;
    return @intCast(value);
}
~~~

ParseError!u8 可以理解为：

~~~text
成功：u8
失败：ParseError 中的某个错误
~~~

### try：把错误交给上层

~~~zig
const std = @import("std");

fn readPort(text: []const u8) !u16 {
    return std.fmt.parseInt(u16, text, 10);
}

fn startServer(text: []const u8) !void {
    const port = try readPort(text);
    std.debug.print("server port = {}\n", .{port});
}

pub fn main() !void {
    try startServer("8080");
}
~~~

try 成功时取出 u16，失败时立即让当前函数返回错误。因此 startServer 也必须声明 !void。

### catch：在当前位置处理错误

~~~zig
const std = @import("std");

fn parseCount(text: []const u8) !u32 {
    return std.fmt.parseInt(u32, text, 10);
}

pub fn main() void {
    const count = parseCount("abc") catch |err| {
        std.debug.print("parse failed: {s}\n", .{@errorName(err)});
        return;
    };

    std.debug.print("count = {}\n", .{count});
}
~~~

catch 也可以直接提供默认值：

~~~zig
const count = parseCount("abc") catch 0;
~~~

错误处理不应无条件吞掉所有错误。可以把可恢复错误转成默认值，把真正的系统错误继续向上返回。

## 六、comptime 参数就是 Zig 泛型的基础

类型只能在编译期使用，所以泛型类型参数要写 comptime：

~~~zig
const std = @import("std");

fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

pub fn main() void {
    const int_max = max(i32, 10, 20);
    const float_max = max(f64, 3.5, 2.8);
    std.debug.print("int = {}, float = {d:.1}\n", .{
        int_max,
        float_max,
    });
}
~~~

T 是编译期类型值，调用 max(i32, ...) 时，编译器可以生成适用于 i32 的版本；调用 max(f64, ...) 时，又会得到适用于 f64 的版本。

anytype 让编译器从调用参数推导类型：

~~~zig
fn double(value: anytype) @TypeOf(value) {
    return value + value;
}

pub fn main() void {
    const a = double(@as(i32, 10));
    const b = double(@as(f64, 2.5));
    std.debug.print("a = {}, b = {d:.1}\n", .{ a, b });
}
~~~

两种泛型写法的差别：

~~~text
comptime T: type   类型作为显式参数，适合类型反射和构造新类型
anytype            从实参推导类型，适合简单的通用操作
~~~

comptime 参数必须接收编译期已知的值：

~~~zig
fn arrayOf(comptime T: type, comptime length: usize) [length]T {
    return [_]T{0} ** length;
}

pub fn main() void {
    const values = arrayOf(u8, 4);
    std.debug.print("{any}\n", .{values});
}
~~~

length 出现在返回类型 [length]T 中，所以不能使用运行时才知道的长度。运行时长度数据应使用 slice，并由调用方提供存储。

### Zig 泛型为什么不需要 Java、.NET 那样的约束

Java 和 .NET 的泛型通常先声明类型约束，再按照约束提供的能力编写泛型代码。

例如 C# 中可以使用 where 声明约束：

~~~text
T 必须实现某个接口
泛型函数只能使用接口中保证存在的成员
~~~

Zig 采用的是另一种方式：编译期鸭子类型。函数体直接使用需要的操作，具体调用时再检查传入类型是否支持这个操作。

~~~zig
fn double(comptime T: type, value: T) T {
    return value + value;
}

const a = double(i32, 10);       // 正确
const b = double(f64, 2.5);      // 正确
// const c = double(bool, true); // 编译错误：bool 不支持 +
~~~

这个函数没有写“ T 必须实现 Add ”，但函数体中的 value + value 已经形成了隐式约束：

~~~text
函数体使用了什么操作
传入类型就必须支持什么操作
具体调用时完成类型检查
~~~

因此，comptime T: type 只表示 T 是编译期类型参数，并不表示 T 自动拥有某种能力。T 是否支持加法，仍然由 value + value 这一行决定。

Java、.NET 和 Zig 的差别可以概括为：

~~~text
Java、.NET：
先声明泛型类型必须满足的接口或基类
再按照约束检查泛型函数

Zig：
直接在泛型函数中使用操作
每次实例化时检查具体类型
~~~

Zig 也可以手动增加更明确的限制：

~~~zig
fn doubleNumber(value: anytype) @TypeOf(value) {
    const T = @TypeOf(value);

    comptime {
        switch (@typeInfo(T)) {
            .int, .float, .comptime_int, .comptime_float => {},
            else => @compileError("doubleNumber 只支持数字类型"),
        }
    }

    return value + value;
}
~~~

这里同时存在两层检查：

1. @typeInfo 和 @compileError 提供自定义类型限制及更清楚的错误信息；
2. value + value 仍然由 Zig 编译器执行正常的操作检查。

这种设计减少了 trait 或 interface 的额外声明，但公共库通常仍然需要通过 @typeInfo、@hasDecl 和 @compileError 提供明确的使用约定。泛型函数越复杂，手动检查越有价值。

## 七、普通函数也能在编译期执行

Zig 没有必须写成 comptime fn 的函数声明。普通函数只要调用环境满足条件，就能在编译期执行：

~~~zig
fn factorial(n: u32) u32 {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

const CompileTimeValue: u32 = comptime factorial(6);
~~~

comptime 要求后面的表达式必须在编译期完成。递归本身不需要 comptime，只有明确要求编译期求值时才需要 comptime 调用。

同一个函数可以同时用于编译期和运行时：

~~~zig
const std = @import("std");

fn fibonacci(n: u32) u32 {
    if (n < 2) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

pub fn main() void {
    const compile_value = comptime fibonacci(10);
    const runtime_value = fibonacci(10);
    std.debug.print("compile = {}, runtime = {}\n", .{
        compile_value,
        runtime_value,
    });
}
~~~

## 八、inline fn 不等于编译期函数

inline fn 要求编译器把函数调用展开到调用位置：

~~~zig
inline fn square(value: i32) i32 {
    return value * value;
}
~~~

inline 主要影响调用展开方式，不代表函数只能在编译期执行，也不代表运行时参数会自动变成编译期常量。函数体很小且调用非常频繁时才考虑 inline；滥用可能增加代码体积。

需要控制某次调用时，可以使用 @call：

~~~zig
const result = @call(.never_inline, square, .{5});
~~~

@call 可以表达自动选择、强制内联、禁止内联或编译期调用等要求。

## 九、函数值、函数指针和回调

函数可以作为参数传递：

~~~zig
const std = @import("std");

const BinaryOp = *const fn (i32, i32) i32;

fn add(a: i32, b: i32) i32 {
    return a + b;
}

fn multiply(a: i32, b: i32) i32 {
    return a * b;
}

fn calculate(a: i32, b: i32, operation: BinaryOp) i32 {
    return operation(a, b);
}

pub fn main() void {
    const sum = calculate(3, 4, add);
    const product = calculate(3, 4, multiply);
    std.debug.print("sum = {}, product = {}\n", .{ sum, product });
}
~~~

这里的 calculate 不关心具体执行加法还是乘法，只依赖 BinaryOp 规定的调用接口。

函数类型和函数指针可以这样区分：

~~~text
fn (i32, i32) i32       函数类型
*const fn (...) i32     指向函数的指针类型
~~~

函数指针可以放进数组，形成策略表：

~~~zig
const Operation = *const fn (i32, i32) i32;

const operations = [_]Operation{ add, multiply };

for (operations) |operation| {
    std.debug.print("{}\n", .{operation(8, 3)});
}
~~~

这类写法适合命令分发、事件处理器、排序规则和插件策略。

Zig 没有传统的闭包语法，函数不能直接捕获外层函数的局部变量。需要保存“函数 + 环境”时，把环境放进 struct：

~~~zig
const Counter = struct {
    step: i32,

    fn add(self: *const Counter, value: i32) i32 {
        return value + self.step;
    }
};

pub fn main() void {
    const counter = Counter{ .step = 10 };
    std.debug.print("{}\n", .{counter.add(5)});
}
~~~

## 十、struct 函数就是方法

Zig 没有 class 和隐式 this，但 struct 可以声明函数：

~~~zig
const Account = struct {
    balance: i64,

    fn deposit(self: *Account, amount: i64) void {
        self.balance += amount;
    }

    fn canWithdraw(self: *const Account, amount: i64) bool {
        return amount >= 0 and amount <= self.balance;
    }
};

pub fn main() void {
    var account = Account{ .balance = 100 };
    account.deposit(50);
    std.debug.print("balance = {}\n", .{account.balance});
    std.debug.print("can withdraw = {}\n", .{
        account.canWithdraw(120),
    });
}
~~~

account.deposit(50) 本质上仍然是一个函数调用，account 作为第一个参数 self 传入。需要修改字段时使用 *Account，只读访问时使用 *const Account。

函数不能直接嵌套在另一个函数体中：

~~~zig
fn outer(value: i32) i32 {
    const Local = struct {
        fn double(input: i32) i32 {
            return input * 2;
        }
    };

    return Local.double(value);
}
~~~

局部辅助函数可以放进函数体内的匿名 struct。需要外部状态时，状态应显式作为参数传递，或放入 struct 字段。

## 十一、noreturn、pub 和 C ABI

noreturn 表示函数不会返回到调用位置：

~~~zig
const std = @import("std");

fn fail(message: []const u8) noreturn {
    std.debug.panic("{s}", .{message});
}
~~~

常见的 noreturn 函数包括 panic、无限循环和直接退出进程的函数。noreturn 还能帮助编译器判断分支类型。

跨模块访问函数时，需要 pub：

~~~zig
// math.zig
pub fn add(a: i32, b: i32) i32 {
    return a + b;
}
~~~

~~~zig
// main.zig
const math = @import("math.zig");

pub fn main() void {
    std.debug.print("{}\n", .{math.add(2, 3)});
}
~~~

pub 只决定 Zig 模块可见性。需要导出给链接器使用时，使用 export：

~~~zig
export fn exportedAdd(a: i32, b: i32) i32 {
    return a + b;
}
~~~

与 C 函数交互时，可以指定 C 调用约定：

~~~zig
fn cCompatible(value: i32) callconv(.c) i32 {
    return value * 2;
}
~~~

调用约定属于函数类型的一部分，函数指针也必须匹配。

## 十二、完整 Demo：计算器与错误处理

~~~zig
const std = @import("std");

const MathError = error{
    DivisionByZero,
};

const Operation = enum {
    add,
    subtract,
    multiply,
    divide,
};

const Calculator = struct {
    last_result: i32 = 0,

    fn execute(
        self: *Calculator,
        operation: Operation,
        a: i32,
        b: i32,
    ) MathError!i32 {
        const result = switch (operation) {
            .add => a + b,
            .subtract => a - b,
            .multiply => a * b,
            .divide => if (b == 0)
                return error.DivisionByZero
            else
                @divTrunc(a, b),
        };

        self.last_result = result;
        return result;
    }
};

pub fn main() void {
    var calculator = Calculator{};

    const jobs = [_]struct {
        operation: Operation,
        a: i32,
        b: i32,
    }{
        .{ .operation = .add, .a = 8, .b = 3 },
        .{ .operation = .multiply, .a = 8, .b = 3 },
        .{ .operation = .divide, .a = 8, .b = 0 },
    };

    for (jobs) |job| {
        const result = calculator.execute(
            job.operation,
            job.a,
            job.b,
        ) catch |err| {
            std.debug.print("error: {s}\n", .{@errorName(err)});
            continue;
        };

        std.debug.print("result = {}, last = {}\n", .{
            result,
            calculator.last_result,
        });
    }
}
~~~

执行流程：

~~~text
jobs 提供输入
    ↓
Calculator.execute 通过 self 修改状态
    ↓
switch 根据 enum 选择运算
    ↓
除数为 0 时返回 error.DivisionByZero
    ↓
调用方用 catch 处理错误
~~~

这个例子里，数据、状态、错误和控制流都通过函数参数及返回类型表达，没有隐藏的异常机制。

## 十三、常见报错

### 1. 忘记处理错误联合

~~~zig
fn load() !u32 {
    return 10;
}

fn run() void {
    // const value = load(); // 错误：错误联合没有处理
    const value = load() catch 0;
    _ = value;
}
~~~

根据业务选择 try、catch 或明确的错误分支。

### 2. 把 comptime 参数当运行时参数

数组类型 [length]u8 需要编译期长度：

~~~zig
fn make(comptime length: usize) [length]u8 {
    return [_]u8{0} ** length;
}
~~~

运行时长度应改成 []u8，并由调用方提供数组或分配内存。

### 3. 把 inline 当成编译期计算

inline 主要负责调用展开，编译期求值使用 comptime：

~~~zig
const value = comptime factorial(5);
~~~

### 4. 直接修改普通参数

普通参数不可重新赋值。需要修改外部变量时，传入 *T，并在调用方使用 var 变量取地址。

## 十四、函数设计的实用判断

~~~text
是否修改外部数据？       使用 *T 或 *const T
是否可能失败？           使用 !T
是否可能没有结果？       使用 ?T
是否需要不同类型复用？   使用 anytype 或 comptime T: type
是否需要替换执行策略？   使用函数类型或函数指针
是否属于某个数据对象？   放进 struct 作为方法
是否需要 C ABI？         使用 callconv、export 或 extern
~~~

函数体很长时，优先拆出命名函数。函数指针和 comptime 都是工具，不需要为了展示特性而使用。

## 总结

Zig 函数的核心价值在于接口透明：

~~~text
fn foo(a: T) R          普通函数
fn foo() !R             可能失败
fn foo() ?R             可能没有结果
fn foo(*T) void         通过指针修改数据
fn foo(comptime T: type) 泛型或编译期类型参数
*const fn (...) R       函数指针
struct.fn(self: *T) R   结构体方法
fn foo() noreturn       永不返回
~~~

掌握下面几条规则，函数相关代码就有了清晰的阅读入口：

1. 参数和返回类型写在函数签名中；
2. 普通参数不可直接修改，修改外部数据需要指针；
3. Zig 没有默认参数和函数重载；
4. !T 表示错误联合，调用时必须用 try 或 catch 处理；
5. ?T 表示可选值，和错误联合表达不同情况；
6. comptime 参数必须在编译期确定，常用于泛型；
7. inline 负责调用展开，comptime 负责编译期求值；
8. struct 函数通过显式 self 参数模拟方法；
9. 函数指针适合回调和策略表；
10. 函数接口越明确，调用链越容易维护。

参考：

- Zig 0.16.0 Language Reference：Functions、comptime、Errors
- Zig 0.16.0 Language Reference：Function Types、inline fn、@call
