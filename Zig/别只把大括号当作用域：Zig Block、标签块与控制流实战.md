# 别只把大括号当作用域：Zig Block、标签块与控制流实战

在 Zig 里，大括号不只是把几行代码包起来。

大括号可以创建新的作用域，也可以作为一个表达式产生结果；加上标签后，还能像一个小型的提前返回点，让多步计算保持在同一段代码里完成。while、for、switch 和 defer，也都和 block 有直接关系。

这也是 Zig 代码里经常出现下面写法的原因：

~~~zig
const result = calculate: {
    // 多步计算
    break :calculate 42;
};
~~~

这不是特殊语法拼凑出来的技巧，而是 Zig 对 block 的正常设计。

本文按 Zig 0.16.0 语法编写，示例可以保存为 main.zig 后运行。

## 一、Block 到底是什么

最普通的 block 是一对大括号：

~~~zig
{
    const width: i32 = 12;
    const height: i32 = 8;
    _ = width * height;
}
~~~

这段代码创建了一个新的作用域。width 和 height 只能在大括号内部使用，离开大括号后就失效：

~~~zig
const std = @import("std");

pub fn main() void {
    {
        const message = "inside";
        std.debug.print("{s}\n", .{message});
    }

    // message 在这里不存在
}
~~~

block 的价值不只是减少变量的可见范围。临时变量、资源清理逻辑和多步计算，都可以集中在一个 block 中，避免把中间状态暴露到更大的函数作用域。

## 二、需要结果时使用标签 Block

普通 block 主要负责创建作用域。需要把多步计算的结果交给外部表达式时，应给 block 加上标签，再用 break 返回值：

~~~zig
{
    const width: i32 = 12;
    const height: i32 = 8;
    _ = width * height;
}
~~~

上面只是普通 block，最后一行带分号，block 只执行计算，不向外部提供 area。真正返回 area 的写法是：

~~~zig
const area: i32 = calculate: {
    const width: i32 = 12;
    const height: i32 = 8;
    break :calculate width * height;
};
~~~

标签 block 的结果由 break :calculate 后面的表达式提供。不能把普通 block 的最后一行无分号表达式当成返回值：

~~~zig
const area = {
    const width: i32 = 12;
    const height: i32 = 8;
    _ = width * height;
};
~~~

这个 block 的结果是 void，不能赋给要求 i32 的变量。可以把规则记成一句话：

> 只需要作用域时使用普通 block；需要局部结果时使用标签 block 和 break。

空 block 的结果是 void：

~~~zig
{
    // 空 block 的结果是 void
}
~~~

实际代码中，空 block 更常见于只为了创建作用域，或者作为 if、switch 分支的占位内容。

## 三、为什么需要标签块

普通 block 只能创建作用域；多分支、多步骤计算需要把结果交给外部时，应使用标签 block。

例如检查一个端口号：

~~~zig
const port: u16 = check: {
    const candidate: i32 = 8080;

    if (candidate < 1 or candidate > 65535) {
        break :check 0;
    }

    break :check @intCast(candidate);
};
~~~

标签块的结构如下：

~~~zig
label: {
    // ...
    break :label value;
}
~~~

前面的 label 是标签名，后面的 break :label 表示离开这个指定的 block，并把 value 作为 block 的结果。

标签名可以按业务含义命名：

~~~zig
const username = validate: {
    const input = "zig";

    if (input.len == 0) {
        break :validate "guest";
    }

    break :validate input;
};
~~~

标签并不代表变量，也不是字符串。它只是一个控制流目标，作用范围限定在当前函数中对应的 block、循环或 switch。

## 四、标签块适合处理多步计算

下面的例子计算订单最终价格。任一步骤失败，都从 price block 返回一个默认值：

~~~zig
const std = @import("std");

fn finalPrice(is_member: bool, amount: i32) i32 {
    const price: i32 = calculate: {
        if (amount <= 0) {
            break :calculate 0;
        }

        var result = amount;

        if (is_member) {
            result -= @divTrunc(result, 10);
        }

        if (result < 100) {
            result += 5;
        }

        break :calculate result;
    };

    std.debug.print("price = {}\n", .{price});
    return price;
}

pub fn main() void {
    _ = finalPrice(true, 120);
    _ = finalPrice(false, 80);
    _ = finalPrice(true, 0);
}
~~~

输出：

~~~text
price = 108
price = 85
price = 0
~~~

这段代码也可以拆成多个 return，但 return 会离开整个函数。break :calculate 只离开计算 block，后面的函数逻辑仍然可以继续执行。

## 五、break、return 和 continue 的区别

三个关键字的离开范围不同：

~~~text
break :label value    离开指定的标签块、循环或 switch，并产生结果
return value          离开整个函数，并把结果交给调用方
continue :label       跳到指定循环的下一轮
~~~

看一个嵌套例子：

~~~zig
const result: i32 = outer: {
    {
        break :outer 100;
    }

    // 不会执行
    break :outer 200;
};
~~~

内层普通 block 没有标签，但 break :outer 仍然可以直接跳到外层标签块。

标签名称必须匹配目标：

~~~zig
const result = work: {
    break :work 10;
};
~~~

break :other 不存在对应标签时会编译失败。

标签块的所有出口必须得到兼容的结果类型：

~~~zig
const value: i32 = choose: {
    if (true) {
        break :choose 10;
    }

    break :choose 20;
};
~~~

如果一个出口返回整数，另一个出口返回字符串，block 就无法推导出单一结果类型。

## 六、作用域、变量生命周期与 defer

defer 会在离开当前 block 时执行：

~~~zig
const std = @import("std");

pub fn main() void {
    {
        defer std.debug.print("离开 inner block\n", .{});
        std.debug.print("执行 inner block\n", .{});
    }

    std.debug.print("继续执行 main\n", .{});
}
~~~

输出顺序：

~~~text
执行 inner block
离开 inner block
继续执行 main
~~~

即使通过 break 离开标签块，defer 仍然会执行：

~~~zig
const std = @import("std");

pub fn main() void {
    const value: i32 = result: {
        defer std.debug.print("清理临时状态\n", .{});

        if (true) {
            break :result 42;
        }

        break :result 0;
    };

    std.debug.print("value = {}\n", .{value});
}
~~~

输出：

~~~text
清理临时状态
value = 42
~~~

同一个作用域有多个 defer 时，后写的先执行：

~~~zig
{
    defer std.debug.print("first\n", .{});
    defer std.debug.print("second\n", .{});
}
~~~

输出是 second、first。这和栈的后进先出规则一致。

资源申请和资源释放很适合放在同一个 block：

~~~zig
fn loadData() []const u8 {
    const data = load: {
        const buffer = "temporary data";
        defer std.debug.print("释放 buffer\n", .{});

        break :load buffer;
    };

    return data;
}
~~~

需要注意，返回的数据不能引用已经离开的局部变量。上面的 buffer 是字符串字面量切片，存活期足够长；如果是局部数组，返回指向该数组的切片就会产生生命周期问题。

## 七、Block 和 if、switch 的关系

if 和 switch 的分支本身就是 block：

~~~zig
const score: i32 = 86;

const level = if (score >= 90)
    "A"
else if (score >= 60)
    "B"
else
    "C";
~~~

分支内容复杂时，可以写成带标签的 block：

~~~zig
const level = if (score >= 60) blk: {
    const passed = true;
    break :blk if (passed) "pass" else "retry";
} else "fail";
~~~

switch 也可以直接产生结果：

~~~zig
const Command = enum {
    start,
    status,
    restart,
};

const command: Command = .restart;

const message = switch (command) {
    .start => "starting",
    .status => "running",
    .restart => "restarting",
};
~~~

switch 的分支需要多步处理时，可使用带标签的 switch：

~~~zig
const std = @import("std");

const Command = enum {
    start,
    status,
    restart,
};

pub fn main() void {
    var command = Command.start;

    const message = process: switch (command) {
        .start => {
            command = .status;
            break :process "started";
        },
        .status => "running",
        .restart => "restarting",
    };

    std.debug.print("{s}, next = {s}\n", .{ message, @tagName(command) });
}
~~~

这里的 process 标签属于 switch 表达式。break :process 返回 switch 的结果，并结束整个 switch。

## 八、循环也是可以产生结果的 Block

while 和 for 都可以作为表达式使用。循环正常结束时，可以由 else 分支提供结果：

~~~zig
const std = @import("std");

pub fn main() void {
    var n: i32 = 3;

    const result = while (n > 0) : (n -= 1) {
        std.debug.print("n = {}\n", .{n});
    } else 0;

    std.debug.print("result = {}\n", .{result});
}
~~~

如果循环通过 break 退出，结果来自 break；如果自然结束，结果来自 else：

~~~zig
const std = @import("std");

fn findFirst(values: []const i32, target: i32) ?i32 {
    const result: ?i32 = search: for (values, 0..) |value, index| {
        if (value == target) {
            break :search @intCast(index);
        }
    } else null;

    return result;
}

pub fn main() void {
    const values = [_]i32{ 4, 8, 15, 16, 23, 42 };

    if (findFirst(&values, 15)) |index| {
        std.debug.print("found at {}\n", .{index});
    } else {
        std.debug.print("not found\n", .{});
    }
}
~~~

这段代码有两条路径：

~~~text
找到 target  -> break :search index
遍历结束未找到 -> else null
~~~

相比先声明一个 found 变量，再在循环中修改 found，循环表达式把“找到时返回什么”和“找不到时返回什么”放到了同一个位置。

嵌套循环中，标签可以明确跳出哪一层：

~~~zig
const std = @import("std");

pub fn main() void {
    const matrix = [_][3]i32{
        .{ 1, 2, 3 },
        .{ 4, 5, 6 },
        .{ 7, 8, 9 },
    };

    const found = search: for (matrix, 0..) |row, row_index| {
        for (row, 0..) |value, column_index| {
            if (value == 5) {
                break :search .{ row_index, column_index };
            }
        }
    } else null;

    if (found) |position| {
        std.debug.print("row = {}, column = {}\n", .{ position[0], position[1] });
    }
}
~~~

## 九、错误处理中的 Block

标签块很适合把“校验、转换、返回默认值”放在一起。

下面的函数解析一个简单端口号：

~~~zig
const std = @import("std");

const ParseError = error{
    Empty,
    InvalidNumber,
    OutOfRange,
};

fn parsePort(text: []const u8) ParseError!u16 {
    const port: u16 = parse: {
        if (text.len == 0) {
            return error.Empty;
        }

        const number = std.fmt.parseInt(u32, text, 10) catch {
            return error.InvalidNumber;
        };

        if (number == 0 or number > 65535) {
            return error.OutOfRange;
        }

        break :parse @intCast(number);
    };

    return port;
}

pub fn main() void {
    const samples = [_][]const u8{ "8080", "", "abc", "70000" };

    for (samples) |sample| {
        const port = parsePort(sample) catch |err| {
            std.debug.print("{s}: error.{s}\n", .{ sample, @errorName(err) });
            continue;
        };

        std.debug.print("{s}: port = {}\n", .{ sample, port });
    }
}
~~~

这里的 return 会退出 parsePort 函数，因为错误需要直接交给调用方；成功路径则用 break :parse 返回 u16。两种控制流的作用范围不同，但可以放在同一个 block 中配合使用。

## 十、comptime Block

前面的大多数 block 在运行时执行。加上 comptime 后，block 会在编译阶段执行：

~~~zig
const Config = struct {
    max_connections: usize = 100,
};

comptime {
    if (@sizeOf(Config) == 0) {
        @compileError("Config cannot be empty");
    }
}
~~~

comptime block 常用于：

- 检查类型是否满足约定；
- 检查配置值是否合法；
- 生成或初始化编译期数据；
- 让错误更早暴露在编译阶段。

comptime 只改变执行时机，不会改变 block 的作用域规则。block 内声明的局部名称，仍然不能在 block 外使用。

## 十一、常见报错与排查方式

### 1. 把普通 block 当成结果表达式

~~~zig
const value: i32 = {
    10;
};
~~~

这里的 block 不是返回 i32 的标签 block。改成下面这样：

~~~zig
const value: i32 = result: {
    break :result 10;
};
~~~

如果只是创建作用域，则不要把 block 赋给变量：

~~~zig
{
    const value: i32 = 10;
    _ = value;
}
~~~

### 2. 在没有标签的 block 中使用带标签 break

~~~zig
{
    break :result 10;
}
~~~

这里没有 result 标签。需要把目标写出来：

~~~zig
result: {
    break :result 10;
}
~~~

### 3. 把 break 当作函数 return

break 只能离开当前可匹配的 block、循环或 switch。需要离开函数时，应使用 return。

### 4. 标签块的分支类型不同

~~~zig
const value = result: {
    if (condition) {
        break :result 1;
    }

    break :result "one";
};
~~~

两个出口分别是整数和字符串，无法得到统一结果类型。统一为同一类型，或改用 union、optional、error union 表达多种结果。

### 5. 变量越过作用域使用

~~~zig
const slice = {
    var buffer = [_]u8{ 'a', 'b', 'c' };
    buffer[0..]
};
~~~

这种写法把局部数组的切片带出了作用域，属于危险的生命周期设计。返回拥有更长生命周期的数据，或把 buffer 放到调用方管理的存储中。

## 十二、一个完整示例：读取配置并生成结果

下面的例子把 block、标签块、optional、循环和 defer 放在一个小程序中：

~~~zig
const std = @import("std");

const Config = struct {
    retries: u8,
    timeout_ms: u32,
};

fn loadConfig(input: []const u8) ?Config {
    const config: ?Config = parse: {
        defer std.debug.print("配置解析结束\n", .{});

        if (input.len == 0) {
            break :parse null;
        }

        if (input[0] != 'o') {
            break :parse null;
        }

        break :parse .{
            .retries = 3,
            .timeout_ms = 1000,
        };
    };

    return config;
}

pub fn main() void {
    const inputs = [_][]const u8{ "ok", "", "bad" };

    for (inputs) |input| {
        if (loadConfig(input)) |config| {
            std.debug.print(
                "input={s}, retries={}, timeout={}ms\n",
                .{ input, config.retries, config.timeout_ms },
            );
        } else {
            std.debug.print("input={s}, invalid\n", .{input});
        }
    }
}
~~~

执行顺序可以拆成三步：

1. loadConfig 创建 parse 标签块；
2. 输入不合法时 break :parse null；
3. 输入合法时 break :parse Config，并在离开 block 前执行 defer。

这种写法适合短小、局部、具有多个提前结束条件的计算。若 block 继续变长，或者需要独立测试，拆成普通函数通常更清楚。

## 十三、什么时候使用 Block

适合使用 block 的场景：

- 临时变量只在一小段逻辑中有效；
- 一个值需要多步计算后产生；
- 多个条件分支都要返回同一个结果；
- 循环需要“找到即返回，结束则给默认值”；
- 资源或临时状态需要在离开作用域时清理；
- 需要在编译期完成检查。

不适合强行使用 block 的场景：

- block 已经包含大量业务逻辑；
- 标签名称需要来回跳转才能理解；
- block 里混合了多个互不相关的任务；
- 普通函数已经能清楚表达输入、输出和错误。

判断标准很简单：block 应该帮助代码收拢局部逻辑，而不是把整个函数压缩成一个巨大的跳转区域。

## 总结

Zig Block 可以同时承担四个角色：

~~~text
普通 block       限制变量作用域
标签 block       用 break :label 产生一个计算结果
comptime block   在编译阶段执行局部逻辑
~~~

掌握下面几条规则，绝大多数 block 代码都能读懂：

1. 大括号会创建新的作用域；
2. 需要局部结果时，给 block 加标签并使用 break :label value；
3. 标签块的所有出口必须得到兼容的结果类型；
4. return 离开函数，break 离开指定 block、循环或 switch；
5. defer 在离开当前作用域时执行，并且后进先出；
6. while、for、switch 也可以作为表达式产生结果；
7. block 的结果类型必须统一，局部变量不能越过生命周期使用。

大括号看起来普通，真正有价值的地方在于：作用域、结果和控制流可以放在同一个局部结构中表达。

参考：

- Zig 0.16.0 Language Reference：Blocks、defer、while、for
- Zig 0.16.0 Language Reference：Labeled break、Labeled switch
