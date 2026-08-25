# 别把错误当异常：Zig error set、error union 与 errdefer 实战

很多语言把失败交给异常机制：函数执行到一半抛出异常，调用方再用 try/catch 接住。

Zig 选择了另一条路线：错误是语言中的一种值，函数签名直接写出“可能失败”，调用方必须明确决定继续传播、提供默认值，还是按错误类型处理。

这一套机制由几个部分组成：

~~~text
error set       定义可能出现的错误名称
error union     表示成功值或错误
try             失败时继续向上返回
catch           在当前位置处理错误
errdefer        只在错误退出时执行清理
~~~

本文按 Zig 0.16.0 语法编写，示例可以保存为 main.zig 后运行。

## 一、错误不是异常，而是一个错误值

最简单的错误值写法：

~~~zig
const err = error.InvalidInput;
~~~

InvalidInput 是错误名称，error.InvalidInput 是一个具体错误值。

一组相关错误可以声明成 error set：

~~~zig
const ParseError = error{
    EmptyInput,
    InvalidNumber,
    OutOfRange,
};
~~~

ParseError 表示一组允许出现的错误：

~~~text
error.EmptyInput
error.InvalidNumber
error.OutOfRange
~~~

错误名称必须在编译期确定，不能在运行时拼接出一个新的错误名称：

~~~zig
// 错误名称不是字符串，不能动态创建：
// const name = "InvalidInput";
// const err = error.{name};
~~~

错误值可以比较：

~~~zig
const err = error.InvalidNumber;

if (err == error.InvalidNumber) {
    // 处理数字格式错误
}
~~~

error set 看起来像 enum，但它不是普通 enum。错误集合用于描述失败路径，错误值还可以和普通结果组合成 error union。

## 二、error union：成功值或错误

函数可能失败时，在返回类型前加 !：

~~~zig
const std = @import("std");

fn parseNumber(text: []const u8) !i32 {
    if (text.len == 0) {
        return error.EmptyInput;
    }

    return std.fmt.parseInt(i32, text, 10);
}
~~~

!i32 表示：

~~~text
成功：i32
失败：某个错误
~~~

也可以显式写出错误集合：

~~~zig
const std = @import("std");

const ParseError = error{
    EmptyInput,
    InvalidNumber,
};

fn parseNumber(text: []const u8) ParseError!i32 {
    if (text.len == 0) {
        return error.EmptyInput;
    }

    return std.fmt.parseInt(i32, text, 10) catch {
        return error.InvalidNumber;
    };
}
~~~

两种写法的区别：

~~~text
!i32              错误集合由编译器推导
ParseError!i32    明确指定错误集合
~~~

公共函数或业务边界通常适合显式写错误集合。内部小函数可以使用 !T，让编译器根据 return 的错误值推导。

## 三、try：成功继续，失败向上返回

try 是最常用的错误传播方式：

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

下面这行：

~~~zig
const port = try readPort(text);
~~~

可以近似理解为：

~~~zig
const port = readPort(text) catch |err| return err;
~~~

两条路径如下：

~~~text
readPort 成功 -> 取出 u16，继续执行
readPort 失败 -> 当前函数立即返回同一个错误
~~~

因此 startServer 也必须返回 !void。错误不会被自动吞掉，也不会凭空跳到某个全局异常处理器。

## 四、catch：在当前位置处理错误

catch 可以提供一个默认值：

~~~zig
const value = parseNumber("abc") catch 0;
~~~

解析成功时 value 是解析结果，解析失败时 value 是 0。

catch 右边的结果类型必须和成功值兼容，或者是 noreturn：

~~~zig
const value = parseNumber("abc") catch {
    return;
};
~~~

catch 代码块没有提供 i32，因为 return 会离开函数，所以它的类型是 noreturn，能够匹配这个位置。

需要查看具体错误时，捕获错误变量：

~~~zig
const std = @import("std");

pub fn main() void {
    const value = parseNumber("abc") catch |err| {
        std.debug.print("parse failed: {s}\n", .{@errorName(err)});
        return;
    };

    std.debug.print("value = {}\n", .{value});
}
~~~

catch 也可以把错误继续向上返回：

~~~zig
const value = parseNumber(text) catch |err| {
    logError(err);
    return err;
};
~~~

## 五、按错误类型分别处理

错误联合可以使用 if 解构：

~~~zig
const std = @import("std");

pub fn main() !void {
    const result = parseNumber("42");

    if (result) |value| {
        std.debug.print("success: {}\n", .{value});
    } else |err| {
        std.debug.print("failure: {s}\n", .{@errorName(err)});
        return err;
    }
}
~~~

错误较多时，使用 switch：

~~~zig
const std = @import("std");

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

fn printPercent(text: []const u8) void {
    if (parsePercent(text)) |value| {
        std.debug.print("percent = {}%\n", .{value});
    } else |err| switch (err) {
        error.EmptyInput => std.debug.print("输入为空\n", .{}),
        error.InvalidNumber => std.debug.print("不是有效数字\n", .{}),
        error.OutOfRange => std.debug.print("数字超过 100\n", .{}),
    }
}
~~~

错误集合是已知的，switch 可以帮助发现遗漏的错误分支。也可以使用 else 统一处理剩余错误：

~~~zig
if (parsePercent(text)) |value| {
    useValue(value);
} else |err| switch (err) {
    error.EmptyInput => useDefault(),
    else => return err,
}
~~~

## 六、!void：只表示可能失败

有些操作成功时不需要返回数据，但仍然可能失败：

~~~zig
fn saveConfig(path: []const u8) !void {
    if (path.len == 0) {
        return error.EmptyPath;
    }

    // 写入文件
}
~~~

调用时：

~~~zig
try saveConfig("config.txt");
~~~

!void 的含义是：

~~~text
成功：void
失败：error
~~~

文件关闭、网络发送、配置保存等操作经常使用 !void。

## 七、错误集合的合并、子集与 anyerror

错误集合可以使用 || 合并：

~~~zig
const FileError = error{
    NotFound,
    PermissionDenied,
};

const NetworkError = error{
    Timeout,
    ConnectionRefused,
};

const AppError = FileError || NetworkError;
~~~

AppError 包含四种错误。

子集错误可以自动转换为更大的错误集合：

~~~zig
const FileError = error{
    NotFound,
    PermissionDenied,
};

const SmallFileError = error{
    NotFound,
};

fn readSmallFile() SmallFileError!void {
    return error.NotFound;
}

fn loadFile() FileError!void {
    return readSmallFile();
}
~~~

SmallFileError 是 FileError 的子集，因此可以向更大的错误集合转换。反方向不成立，因为 FileError 中还可能包含 PermissionDenied。

anyerror 表示整个编译单元中的全局错误集合：

~~~zig
fn unsafeApi() anyerror!void {
    return error.SomeFailure;
}
~~~

anyerror 很方便，但会隐藏函数的具体失败范围。公共 API 通常优先使用明确的错误集合，只有确实需要接收任意错误时才使用 anyerror。

## 八、错误映射：把底层错误转换成业务错误

底层库的错误不一定适合直接暴露到业务层：

~~~zig
const ConfigError = error{
    InvalidFormat,
    MissingValue,
};

fn loadPort(text: []const u8) ConfigError!u16 {
    if (text.len == 0) {
        return error.MissingValue;
    }

    const value = std.fmt.parseInt(u16, text, 10) catch {
        return error.InvalidFormat;
    };

    return value;
}
~~~

这里把 parseInt 可能返回的底层错误统一映射成 InvalidFormat。这样上层不需要知道标准库内部的具体错误名称。

错误映射适合放在边界位置：

~~~text
标准库或系统调用
        ↓
模块内部错误
        ↓
业务层错误
        ↓
用户提示或日志
~~~

不要为了保留每一个底层错误而让整个业务层暴露一大串无关错误。

## 九、defer 和 errdefer：清理资源

defer 无论函数正常返回还是错误返回，都会在离开当前作用域时执行：

~~~zig
const std = @import("std");

fn work() !void {
    defer std.debug.print("always cleanup\n", .{});

    if (true) {
        return error.Failed;
    }
}
~~~

errdefer 只在错误退出时执行：

~~~zig
const std = @import("std");

fn work(should_fail: bool) !void {
    errdefer std.debug.print("rollback\n", .{});

    if (should_fail) {
        return error.Failed;
    }

    std.debug.print("success\n", .{});
}

pub fn main() void {
    work(false) catch {};
    work(true) catch {};
}
~~~

输出：

~~~text
success
rollback
~~~

典型用途是：先申请资源，后续步骤失败时自动回滚：

~~~zig
fn createResource() !void {
    const resource = try allocateResource();
    errdefer releaseResource(resource);

    try initializeResource(resource);
}
~~~

成功返回时，resource 交给调用方；初始化失败时，errdefer 自动释放 resource。

defer 和 errdefer 可以组合：

~~~zig
fn openAndProcess() !void {
    const file = try openFile();
    defer closeFile(file);

    const buffer = try allocateBuffer();
    errdefer freeBuffer(buffer);

    try process(file, buffer);
}
~~~

这里 file 无论成功还是失败都要关闭，buffer 只在后续流程失败时释放。

errdefer 可以捕获导致错误退出的错误值：

~~~zig
fn operation() !void {
    errdefer |err| {
        std.debug.print("rollback because: {s}\n", .{
            @errorName(err),
        });
    }

    return error.Failed;
}
~~~

多个 defer 按后进先出顺序执行，errdefer 也遵循相同的作用域规则。

## 十、错误联合不是带 payload 的错误

错误本身只表示错误名称，不携带业务数据：

~~~zig
const err = error.OutOfRange;
~~~

下面这种写法不是 Zig 的错误集合语法：

~~~zig
// error{OutOfRange: i32} // 错误集合成员不能声明数据类型
~~~

需要携带额外数据时，使用结构体、union 或 tagged union：

~~~zig
const ValidationResult = union(enum) {
    ok: u32,
    invalid: struct {
        value: i32,
        reason: []const u8,
    },
};
~~~

如果结果同时需要“成功值、错误名称和错误信息”，可以设计成：

~~~zig
const Result = union(enum) {
    success: u32,
    failure: struct {
        code: error{Invalid, TooLarge},
        message: []const u8,
    },
};
~~~

error union 适合表达“成功或失败”；union(enum) 适合表达带不同 payload 的多种状态。两者解决的问题不同。

## 十一、错误不是 optional

optional 和 error union 的语义不同：

~~~zig
fn findUser(id: u32) ?User {
    // 找不到用户，返回 null
}

fn loadUser(id: u32) !User {
    // 数据库连接失败、权限不足等情况返回错误
}
~~~

可以这样区分：

~~~text
?T       结果可能不存在
!T       操作可能失败
!?T      操作可能失败，成功时结果也可能不存在
~~~

例如数据库查询：

~~~zig
fn findUser(id: u32) !?User {
    // 连接失败 -> error
    // 查询成功但没有记录 -> null
    // 查询到记录 -> User
}
~~~

不要用 null 代替所有错误，否则“没有数据”和“系统故障”会混在一起。

## 十二、catch unreachable 的边界

明确知道某个调用不可能失败时，可以使用 catch unreachable：

~~~zig
const number = std.fmt.parseInt(u32, "123", 10) catch unreachable;
~~~

如果运行时实际出现错误，Debug 和 ReleaseSafe 模式会触发安全检查失败。

catch unreachable 适合编译期固定且已经验证过的输入：

~~~zig
const port: u16 = std.fmt.parseInt(u16, "8080", 10) catch unreachable;
~~~

不适合用于网络请求、文件读取、用户输入等外部数据：

~~~zig
// 不推荐：
// const data = readFile(path) catch unreachable;
~~~

外部数据随时可能失败。此时应使用 try、catch 或明确的错误分支。

## 十三、错误回溯

Debug 构建中，错误从底层一路通过 try 传播到 main 时，Zig 可以记录 error return trace。

~~~zig
fn readConfig() !void {
    return error.FileNotFound;
}

fn start() !void {
    try readConfig();
}

pub fn main() !void {
    try start();
}
~~~

当错误最终返回到 main，Debug 模式的错误回溯可以显示错误经过的函数路径。它和普通崩溃栈不同：

~~~text
普通栈回溯：程序崩溃时正在执行哪些函数
错误回溯：错误值经过哪些函数传播到当前地点
~~~

Release 模式下错误回溯支持和默认行为可能不同，调试阶段优先使用 Debug 构建定位错误传播链。

## 十四、完整 Demo：配置加载流程

下面的示例模拟配置读取、解析、错误映射、资源清理和错误处理：

~~~zig
const std = @import("std");

const ConfigError = error{
    EmptyInput,
    InvalidPort,
    PortOutOfRange,
};

fn parsePort(text: []const u8) ConfigError!u16 {
    if (text.len == 0) {
        return error.EmptyInput;
    }

    const value = std.fmt.parseInt(u32, text, 10) catch {
        return error.InvalidPort;
    };

    if (value == 0 or value > 65535) {
        return error.PortOutOfRange;
    }

    return @intCast(value);
}

fn loadConfig(text: []const u8) ConfigError!u16 {
    std.debug.print("load config\n", .{});
    defer std.debug.print("leave loadConfig\n", .{});
    errdefer std.debug.print("rollback config state\n", .{});

    const port = try parsePort(text);
    return port;
}

fn printResult(text: []const u8) void {
    const port = loadConfig(text) catch |err| {
        switch (err) {
            error.EmptyInput => std.debug.print("配置为空\n", .{}),
            error.InvalidPort => std.debug.print("端口格式错误\n", .{}),
            error.PortOutOfRange => std.debug.print("端口超出范围\n", .{}),
        }
        return;
    };

    std.debug.print("server port = {}\n", .{port});
}

pub fn main() void {
    printResult("8080");
    printResult("");
    printResult("abc");
    printResult("70000");
}
~~~

执行时可以观察到：

~~~text
正常返回：defer 执行，errdefer 不执行
错误返回：errdefer 先执行，defer 也执行
~~~

这个结构适合扩展到真实场景：

~~~text
读取配置
    ↓ try
解析端口
    ↓ error mapping
转换成业务错误
    ↓ catch
展示错误或使用降级配置
~~~

## 十五、常见报错与排查方式

### 1. 直接丢弃错误联合

~~~zig
fn read() !u32 {
    return 10;
}

fn run() void {
    // read(); // 编译错误：错误没有被处理
    _ = read() catch {};
}
~~~

如果确实不关心错误，需要明确写出 catch。不能让错误静默消失。

### 2. try 所在函数没有返回错误

~~~zig
fn caller() void {
    // const value = try read(); // 编译错误
}
~~~

改成：

~~~zig
fn caller() !void {
    const value = try read();
    _ = value;
}
~~~

或者在当前位置 catch。

### 3. 使用错误集合之外的错误

~~~zig
const SmallError = error{OnlyOne};

fn bad() SmallError!void {
    // return error.Other; // 不属于 SmallError
}
~~~

显式错误集合可以帮助限制模块对外暴露的错误范围。

### 4. 用 orelse 处理错误

orelse 处理 optional：

~~~zig
const value: ?u32 = null;
const result = value orelse 0;
~~~

错误联合应使用 try 或 catch。

### 5. 滥用 catch unreachable

只有在失败确实代表程序逻辑错误，并且输入已经被可靠验证时，才适合使用 catch unreachable。

## 十六、错误处理的设计建议

~~~text
底层函数：返回具体错误
模块边界：映射成模块错误
业务层：决定传播、降级或展示
资源管理：申请后立刻写 defer 或 errdefer
~~~

几个实用原则：

- 外部输入不要使用 catch unreachable；
- 资源申请后尽快安排清理逻辑；
- 可恢复错误和系统故障分开处理；
- 公共函数优先写明确的错误集合；
- 不要用 anyerror 隐藏所有可能错误；
- 不要用 null 掩盖真正的系统失败；
- 错误信息需要额外数据时，使用 struct 或 tagged union；
- 错误映射放在模块边界，避免底层实现细节泄露到业务层。

## 总结

Zig 的错误处理可以归纳为：

~~~text
error{A, B}       一组错误值
ErrorSet!T        明确的错误集合 + 成功值
!T                推导错误集合 + 成功值
try expr          错误向上返回，成功取出值
expr catch value  出错时提供默认值
catch |err|       捕获并处理具体错误
defer             无论如何执行清理
errdefer          错误退出时执行回滚
@errorName(err)   获取错误名称
~~~

掌握下面几条规则，错误处理代码就有了清晰的阅读入口：

1. 错误是值，不是异常对象；
2. error set 描述可能出现的错误名称；
3. error union 表示成功值或错误；
4. try 负责传播，catch 负责处理；
5. errdefer 专门处理错误路径上的资源回滚；
6. 错误集合可以合并，子集可以转换为超集；
7. anyerror 虽然方便，但会隐藏错误范围；
8. 错误不携带 payload，额外数据使用 struct 或 tagged union；
9. optional 表示结果不存在，error union 表示操作失败；
10. 外部输入不应轻易使用 catch unreachable。

参考：

- Zig 0.16.0 Language Reference：Error Set、Error Union、try、catch、errdefer
- Zig 0.16.0 Language Reference：Error Return Traces
