# 把代码搬到编译期：Zig comptime、泛型与类型反射实战

很多语言把编译期限制在常量折叠、宏或代码生成器上。Zig 更进一步：普通 Zig 函数、循环、条件判断、类型操作和校验逻辑，都可以在编译阶段执行。

这套能力由 comptime 提供。它不只是提前计算一个数字，还可以用来生成类型、实现泛型、遍历结构体字段、检查配置，并把错误提前到编译阶段。

本文按 Zig 0.16.0 语法编写，示例可以保存为 main.zig 后运行。

## 一、运行时和编译时

普通代码通常在程序启动后执行：

~~~zig
fn getValue() u32 {
    return 10;
}

const value = getValue();
~~~

如果函数依赖用户输入、文件、网络或当前时间，只能等程序运行后才能得到结果。

comptime 则要求表达式在编译阶段完成：

~~~zig
const value = comptime 10 + 20;
~~~

编译过程可以理解成：

~~~text
源码
  ↓
执行 comptime 表达式
  ↓
得到编译期结果 30
  ↓
生成机器码
  ↓
运行程序
~~~

运行时不再需要执行 10 + 20 这段计算。

## 二、const 不等于 comptime

const 表示绑定不能被重新赋值：

~~~zig
const value: u32 = 10;
// value = 20; // 不允许修改 const 绑定
~~~

comptime 表示值必须在编译阶段确定：

~~~zig
const value: u32 = comptime 10 + 20;
~~~

const 变量也可能来自运行时：

~~~zig
fn readValue() u32 {
    return 42;
}

pub fn main() void {
    const value = readValue();
    _ = value;
}
~~~

value 不能修改，但 readValue 仍然是在运行时调用。

可以这样区分：

~~~text
const       绑定不可修改
comptime    值必须在编译期确定
~~~

很多简单的 const 表达式本来就能被编译器优化，但只有写出 comptime，才能明确要求编译器必须在编译期完成。

## 三、最简单的 comptime 表达式

~~~zig
const std = @import("std");

fn factorial(n: u32) u32 {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

pub fn main() void {
    const a = comptime 10 + 20;
    const b = comptime factorial(6);
    std.debug.print("a = {}, b = {}\n", .{ a, b });
}
~~~

普通函数不需要额外声明成“编译期函数”。只要调用时的参数和执行环境满足编译期求值条件，就可以通过 comptime 调用。

同一个函数可以分别用于编译期和运行时：

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

## 四、comptime 参数

函数参数前加 comptime，表示调用时必须传入编译期已知的值：

~~~zig
fn twice(comptime number: u32) u32 {
    return number * 2;
}

const value = twice(10);
~~~

10 是编译期常量，所以调用合法。

运行时变量不能传入 comptime 参数：

~~~zig
fn twice(comptime number: u32) u32 {
    return number * 2;
}

fn readNumber() u32 {
    return 10;
}

pub fn main() void {
    const runtime_number = readNumber();
    // const value = twice(runtime_number); // 编译错误
}
~~~

即使 runtime_number 的实际结果碰巧是 10，变量来源仍然是运行时调用，不能当成编译期值。

comptime 参数也可以是字符串、枚举、数组或结构体配置：

~~~zig
fn makeTag(comptime name: []const u8) []const u8 {
    return "app." ++ name;
}

const tag = makeTag("server");
~~~

## 五、comptime T: type：类型也是编译期值

Zig 中，类型本身可以作为值传递，但类型只能在编译期使用：

~~~zig
const IntegerType: type = u32;
const value: IntegerType = 100;
~~~

因此，下面的写法很常见：

~~~zig
fn double(comptime T: type, value: T) T {
    return value + value;
}

const a = double(i32, 10);
const b = double(f64, 2.5);
~~~

comptime T: type 可以拆成：

~~~text
comptime   T 必须在编译期确定
T          参数名
type       T 保存的是一个类型
~~~

调用 double(i32, 10) 时，T 等于 i32；调用 double(f64, 2.5) 时，T 等于 f64。

这就是 Zig 泛型的基础：

~~~zig
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

const int_value = max(i32, 10, 20);
const float_value = max(f64, 2.5, 3.5);
~~~

如果传入 bool，函数体中的 > 不适用于 bool，编译器会报告错误。comptime T: type 不自动提供 trait 约束，函数体中的实际操作就是隐式约束。

## 六、anytype 和 comptime T 的区别

anytype 让编译器从实参推导类型：

~~~zig
fn double(value: anytype) @TypeOf(value) {
    return value + value;
}

const a = double(@as(i32, 10));
const b = double(@as(f64, 2.5));
~~~

comptime T: type 则把类型作为显式参数传入：

~~~zig
fn doubleTyped(comptime T: type, value: T) T {
    return value + value;
}

const a = doubleTyped(i32, 10);
const b = doubleTyped(f64, 2.5);
~~~

两种写法可以这样理解：

~~~text
anytype
    类型从实参表达式自动推导

comptime T: type
    类型作为编译期参数显式传入
~~~

需要访问类型本身、创建新类型或做类型反射时，通常使用 comptime T: type。只需要对不同类型执行同一套操作时，anytype 更简洁。

## 七、根据 comptime 参数生成类型

comptime 函数可以返回一个根据参数决定的类型：

~~~zig
fn Array(comptime T: type, comptime length: usize) type {
    return [length]T;
}

pub fn main() void {
    var numbers: Array(u32, 4) = .{ 10, 20, 30, 40 };
    numbers[0] += 1;
}
~~~

Array(u32, 4) 返回 [4]u32 类型，Array(f64, 2) 则返回 [2]f64 类型。

length 必须 comptime，因为它出现在类型 [length]T 中：

~~~zig
fn makeArray(comptime length: usize) [length]u8 {
    return [_]u8{0} ** length;
}
~~~

运行时长度不能直接作为数组类型长度。运行时长度的数据应使用切片：

~~~zig
fn fill(buffer: []u8, value: u8) void {
    for (buffer) |*item| {
        item.* = value;
    }
}
~~~

## 八、comptime Block

复杂的编译期逻辑可以放入 comptime block：

~~~zig
const std = @import("std");

const table = blk: {
    var result: [5]u32 = undefined;
    var i: usize = 0;

    while (i < result.len) : (i += 1) {
        result[i] = @intCast(i * i);
    }

    break :blk result;
};

pub fn main() void {
    std.debug.print("{any}\n", .{table});
}
~~~

break :blk 将计算结果返回给 block 外部的 table。

上面的 table 位于文件顶层。顶层 const 的初始化本身已经在编译期求值，因此不能再写成 comptime blk，否则会得到 redundant comptime 错误。函数内部的局部表达式则可以显式使用 comptime：

~~~zig
fn makeValue() u32 {
    return comptime blk: {
        var value: u32 = 10;
        value += 20;
        break :blk value;
    };
}
~~~

comptime block 内部可以使用 if、while、for、switch、函数调用、数组构造和结构体构造，前提是所有操作都能在编译期完成。

## 九、comptime var 和 inline 循环

comptime var 表示变量本身必须在编译期读写：

~~~zig
const std = @import("std");

pub fn main() void {
    comptime var sum: u32 = 0;

    inline while (sum < 5) : (sum += 1) {}

    std.debug.print("sum = {}\n", .{sum});
}
~~~

更常见的场景是使用 inline for 生成固定数组：

~~~zig
fn squareTable(comptime length: usize) [length]u32 {
    var table: [length]u32 = undefined;

    inline for (0..length) |index| {
        table[index] = @intCast(index * index);
    }

    return table;
}

pub fn main() void {
    const table = squareTable(6);
    _ = table;
}
~~~

普通 for 和 inline for 的区别：

~~~text
for         通常保留运行时循环
inline for  展开已知范围的循环体
~~~

inline 展开不是免费的。循环次数很大时，生成的代码会变大，编译时间也可能增加。

## 十、编译期反射：@typeInfo

@typeInfo(T) 返回 T 的类型信息。Zig 0.16 中，类型信息使用 tagged union 表示：

~~~zig
const std = @import("std");

const User = struct {
    id: u64,
    name: []const u8,
    enabled: bool,
};

fn printTypeInfo(comptime T: type) void {
    const info = @typeInfo(T);

    switch (info) {
        .@"struct" => |struct_info| {
            std.debug.print("struct: {s}\n", .{@typeName(T)});

            inline for (struct_info.fields) |field| {
                std.debug.print("field {s}: {s}\n", .{
                    field.name,
                    @typeName(field.type),
                });
            }
        },
        else => @compileError("只支持 struct"),
    }
}

pub fn main() void {
    printTypeInfo(User);
}
~~~

@typeInfo 常用于：

- 自动生成序列化和反序列化代码；
- 检查结构体字段；
- 判断泛型参数是否为整数、指针、数组或 enum；
- 根据字段类型选择不同实现；
- 为框架类型生成注册表。

### @TypeOf 和 @typeInfo 的区别

这两个内置函数经常连续出现，但职责完全不同：

~~~text
@TypeOf     这个值是什么类型？
@typeInfo   这个类型有哪些结构信息？
~~~

@TypeOf 的输入是表达式或值，返回结果是一个 type：

~~~zig
const std = @import("std");

pub fn main() void {
    const value: u32 = 100;
    const T = @TypeOf(value);

    std.debug.print("{s}\n", .{@typeName(T)});
}
~~~

执行流程：

~~~text
value
  ↓ @TypeOf(value)
u32
~~~

@typeInfo 的输入是类型，返回的是描述该类型的元信息：

~~~zig
const std = @import("std");

fn inspect(comptime T: type) void {
    switch (@typeInfo(T)) {
        .int => |info| {
            std.debug.print("bits = {}\n", .{info.bits});
        },
        else => {},
    }
}

pub fn main() void {
    inspect(u32);
}
~~~

u32 经过 @typeInfo 后，可以继续读取它的整数位数、是否有符号等信息。

两者的对比：

~~~text
输入：
@TypeOf(value)       一个表达式或值
@typeInfo(T)          一个类型

输出：
@TypeOf(value)       type
@typeInfo(T)          类型元信息 tagged union

用途：
@TypeOf              获取类型、推导返回类型
@typeInfo             拆解类型结构、编译期反射
~~~

泛型反射中，两者通常连起来使用：

~~~zig
fn inspectValue(value: anytype) void {
    const T = @TypeOf(value);
    const info = @typeInfo(T);
    _ = info;
}
~~~

第一步从 value 得到类型 T，第二步再分析 T 属于整数、浮点数、struct、指针还是其他类型。

@TypeOf 更像“取得类型标签”，@typeInfo 更像“打开类型说明书”。前者回答类型是什么，后者回答类型由哪些部分组成。

## 十一、编译期校验：@compileError

@compileError 会主动终止编译，并输出自定义错误信息：

~~~zig
fn assertPositive(comptime value: i32) void {
    if (value <= 0) {
        @compileError("value 必须大于 0");
    }
}

pub fn main() void {
    assertPositive(10);
    // assertPositive(-1); // 编译阶段失败
}
~~~

配置校验也适合放到编译期：

~~~zig
const Config = struct {
    max_clients: usize,
    timeout_ms: u32,
};

fn validateConfig(comptime settings: Config) Config {
    if (settings.max_clients == 0) {
        @compileError("max_clients 不能为 0");
    }

    if (settings.timeout_ms < 100) {
        @compileError("timeout_ms 不能小于 100");
    }

    return settings;
}

const config = validateConfig(.{
    .max_clients = 100,
    .timeout_ms = 1000,
});
~~~

配置错误会在编译阶段报告，不需要等程序启动后才发现。

## 十二、编译期泛型容器

comptime 函数可以返回一个根据类型参数生成的 struct：

~~~zig
const std = @import("std");

fn Box(comptime T: type) type {
    return struct {
        const Self = @This();
        value: T,

        pub fn init(value: T) Self {
            return .{ .value = value };
        }

        pub fn get(self: Self) T {
            return self.value;
        }
    };
}

pub fn main() void {
    const IntBox = Box(i32);
    const FloatBox = Box(f64);

    const int_box = IntBox.init(42);
    const float_box = FloatBox.init(3.14);

    std.debug.print("{} {d:.2}\n", .{
        int_box.get(),
        float_box.get(),
    });
}
~~~

Box(i32) 和 Box(f64) 是两个不同的具体类型。编译器会根据 T 分析 struct 中的字段和函数。

## 十三、完整 Demo：编译期配置和命令表

~~~zig
const std = @import("std");

const Config = struct {
    max_clients: usize,
    timeout_ms: u32,
};

fn validateConfig(comptime settings: Config) Config {
    if (settings.max_clients == 0) {
        @compileError("max_clients 不能为 0");
    }

    if (settings.timeout_ms < 100) {
        @compileError("timeout_ms 太小");
    }

    return settings;
}

const config = validateConfig(.{
    .max_clients = 256,
    .timeout_ms = 1000,
});

const Command = enum {
    start,
    stop,
    status,
};

const CommandInfo = struct {
    command: Command,
    name: []const u8,
};

const commands = [_]CommandInfo{
    .{ .command = .start, .name = "start" },
    .{ .command = .stop, .name = "stop" },
    .{ .command = .status, .name = "status" },
};

fn commandName(comptime command: Command) []const u8 {
    inline for (commands) |item| {
        if (item.command == command) {
            return item.name;
        }
    }

    unreachable;
}

fn printStructFields(comptime T: type) void {
    const info = @typeInfo(T);

    switch (info) {
        .@"struct" => |struct_info| {
            inline for (struct_info.fields) |field| {
                std.debug.print("field: {s}\n", .{field.name});
            }
        },
        else => @compileError("T 必须是 struct"),
    }
}

pub fn main() void {
    std.debug.print(
        "max_clients = {}, timeout = {}ms\n",
        .{ config.max_clients, config.timeout_ms },
    );

    std.debug.print("start = {s}\n", .{commandName(.start)});
    std.debug.print("status = {s}\n", .{commandName(.status)});

    printStructFields(Config);
}
~~~

这个 Demo 中：

~~~text
validateConfig     编译期检查配置
commands           固定表直接进入程序
commandName        编译期遍历命令表
printStructFields  编译期读取 struct 字段
~~~

配置改错时，编译器会直接停止；命令名称和字段信息不需要运行时反射。

## 十四、comptime 的限制

comptime 不是另一个可以访问所有系统资源的运行时环境。编译阶段没有普通程序运行时的外部输入，因此下列操作不能直接依赖运行时数据：

- 用户输入；
- 网络请求；
- 系统调用；
- 运行时文件内容；
- 当前时间；
- 运行时才知道的数组长度。

例如：

~~~zig
fn readValue(input: u32) u32 {
    return input;
}

pub fn main() void {
    var runtime_value: u32 = 10;
    _ = &runtime_value;
    // const value = comptime readValue(runtime_value); // 依赖运行时值时会失败
}
~~~

关键不是标准库函数名称，而是这次调用能否在编译期完成，并且不能依赖运行时数据或运行时副作用。

编译期也会消耗编译时间和内存。复杂递归、超大数组、数量很多的类型实例，都可能让编译变慢或导致生成代码变大。

## 十五、inline 和 comptime 不是一回事

inline 主要控制代码展开，comptime 主要控制执行时机：

~~~text
comptime expr       强制表达式在编译期求值
comptime 参数       要求调用实参在编译期确定
comptime block      强制 block 在编译期执行
inline for          展开已知范围的循环
inline fn           要求函数调用尝试内联
~~~

inline fn 不等于编译期函数：

~~~zig
inline fn square(value: i32) i32 {
    return value * value;
}
~~~

square 仍然可以接收运行时的 i32。需要明确编译期计算时，使用：

~~~zig
const result = comptime square(5);
~~~

性能优化不应只依赖 inline。comptime 是语言层面的编译期约束，inline 主要影响函数调用的生成方式。

## 十六、常见报错与排查方式

### 1. 运行时变量传给 comptime 参数

~~~zig
fn makeArray(comptime length: usize) [length]u8 {
    return [_]u8{0} ** length;
}

fn getLength() usize {
    return 4;
}

pub fn main() void {
    const length = getLength();
    // const result = makeArray(length); // length 不是编译期值
}
~~~

固定长度数组需要 comptime 长度；运行时长度改用 slice。

### 2. 把 const 当成 comptime

~~~zig
const value = readValue();
// const result = comptime value + 1; // value 可能来自运行时
~~~

const 只表示不可修改，不能保证来源一定是编译期。

### 3. 类型反射使用旧版字段名

Zig 0.16 的 @typeInfo 标签使用当前版本语法：

~~~zig
switch (@typeInfo(T)) {
    .@"struct" => |info| {
        _ = info;
    },
    else => {},
}
~~~

旧资料中的 .Struct、.Int 等大小写写法不能直接照搬到 Zig 0.16。

### 4. 过度使用 comptime

以下情况通常保留运行时实现更合适：

- 数据来自用户或网络；
- 循环次数很大；
- 类型实例数量很多；
- 编译速度比少量运行时计算更重要。

## 十七、什么时候使用 comptime

适合使用 comptime 的场景：

- 固定配置的合法性检查；
- 固定长度数组和矩阵；
- 泛型容器；
- 序列化器和反序列化器；
- struct 字段反射；
- 编译期生成查找表；
- 编译期选择不同实现；
- 将不可能的配置直接变成编译错误。

不适合使用 comptime 的场景：

- 处理运行时输入；
- 替代正常业务流程；
- 为了少量运行时计算而生成巨大代码；
- 把复杂逻辑全部堆进一个 comptime block。

判断标准可以归纳为：

~~~text
编译期已知 + 逻辑规模可控 + 结果适合固化
                    ↓
                 适合 comptime
~~~

## 总结

comptime 可以看成 Zig 内置的一套编译期执行环境：

~~~text
comptime expr           编译期执行表达式
comptime 参数           编译期确定调用参数
comptime T: type        编译期传入类型
comptime block          编译期执行复杂逻辑
comptime var            编译期读写变量
inline for / while      展开已知范围的循环
@typeInfo               获取类型元信息
@compileError           编译期主动报错
~~~

掌握下面几条规则，comptime 代码就有了清晰的阅读入口：

1. const 表示不可修改，comptime 表示必须在编译期确定；
2. 普通函数满足条件时，也可以在编译期调用；
3. comptime 参数必须接收编译期已知的值；
4. comptime T: type 是 Zig 泛型和类型工厂的基础；
5. @typeInfo 可以在编译期读取类型信息；
6. @compileError 可以把配置和类型错误提前到编译阶段；
7. inline 负责展开，comptime 负责编译期求值；
8. 编译期代码不能依赖运行时输入和外部副作用；
9. 复杂 comptime 逻辑可能增加编译时间和代码体积；
10. 只有固定、可验证、适合固化的逻辑才值得搬到编译期。

参考：

- Zig 0.16.0 Language Reference：comptime、Compile-Time Parameters、Generic Data Structures
- Zig 0.16.0 Language Reference：@typeInfo、@compileError、inline for
