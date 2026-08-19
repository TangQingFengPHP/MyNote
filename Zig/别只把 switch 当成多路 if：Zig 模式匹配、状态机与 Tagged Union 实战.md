# 别只把 switch 当成多路 if：Zig 模式匹配、状态机与 Tagged Union 实战

很多语言里的 switch 只是把一串 if-else 换成了另一种写法。Zig 的 switch 更像一个带编译期检查的模式匹配工具：

- 可以作为语句执行逻辑，也可以作为表达式返回值；
- 枚举和错误集可以做穷尽检查；
- 支持多个值合并和闭区间范围匹配；
- 可以解构 Tagged Union，直接拿到载荷；
- 可以处理可选值；
- 支持带标签的 switch 和 continue，适合表达状态机；
- 不存在 C 语言那种隐式 fall-through。

本文按 Zig 0.16.0 语法编写，示例可以直接保存为 .zig 文件运行。

## 一、最基本的 switch

~~~zig
const std = @import("std");

pub fn main() void {
    const value: u8 = 5;

    switch (value) {
        5 => {
            std.debug.print("value is 5\n", .{});
        },
        else => {
            std.debug.print("other value\n", .{});
        },
    }
}
~~~

每个分支由两部分组成：

~~~text
匹配项 => 执行结果
~~~

分支之间使用逗号分隔。执行完匹配到的分支后，switch 自动结束，不会继续执行下面的分支。

Zig 不需要像 C 语言那样写 break。多个值需要执行同一段逻辑时，使用逗号合并：

~~~zig
switch (value) {
    1, 2, 3 => std.debug.print("small\n", .{}),
    else => std.debug.print("other\n", .{}),
}
~~~

## 二、switch 作为表达式返回值

~~~zig
const std = @import("std");

const Status = enum {
    success,
    failed,
};

pub fn main() void {
    const status: Status = .success;

    const message = switch (status) {
        .success => "操作成功",
        .failed => "操作失败",
    };

    std.debug.print("{s}\n", .{message});
}
~~~

表达式形式适合做映射：

~~~zig
const message = switch (code) {
    200 => "ok",
    404 => "not found",
    else => "unknown",
};
~~~

所有分支的结果必须能够转换成同一个类型。一个分支返回整数、另一个分支返回字符串，会导致类型推导失败。

分支可以是代码块，代码块最后的表达式作为该分支的结果：

~~~zig
const result = switch (value) {
    0 => {
        std.debug.print("zero\n", .{});
        0,
    },
    else => value * 2,
};
~~~

## 三、穷尽匹配：编译器帮忙发现漏分支

~~~zig
const Color = enum {
    red,
    green,
    blue,
};

fn colorText(color: Color) []const u8 {
    return switch (color) {
        .red => "红色",
        .green => "绿色",
        .blue => "蓝色",
    };
}
~~~

三个成员全部列出后，不需要 else。以后增加 yellow，colorText 会出现编译错误，提醒补充处理逻辑。

只关心某一个成员时，可以使用 else：

~~~zig
fn isWarm(color: Color) bool {
    return switch (color) {
        .red => true,
        else => false,
    };
}
~~~

状态转换和权限规则通常更适合显式列出全部状态；纯粹的分类判断才适合使用 else 统一兜底。

整数类型通常需要 else，因为 u8 有 256 种可能：

~~~zig
const result = switch (value: u8) {
    0...127 => "low",
    128...255 => "high",
};
~~~

当范围覆盖整个类型时，也可以省略 else。

## 四、多个值与范围匹配

~~~zig
const std = @import("std");

pub fn main() void {
    const score: u8 = 85;

    const grade = switch (score) {
        0...59 => "F",
        60...69 => "D",
        70...79 => "C",
        80...89 => "B",
        90...100 => "A",
        else => "无效分数",
    };

    std.debug.print("分数 {}，等级 {s}\n", .{ score, grade });
}
~~~

80...89 是闭区间，包含 80 和 89。

多个离散值可以合并：

~~~zig
const kind = switch (ch) {
    '0'...'9' => "数字",
    'a'...'z', 'A'...'Z' => "字母",
    else => "其他字符",
};
~~~

## 五、枚举 + switch：最常见的组合

~~~zig
const std = @import("std");

const OrderStatus = enum {
    created,
    paid,
    shipping,
    finished,
    cancelled,
};

fn statusText(status: OrderStatus) []const u8 {
    return switch (status) {
        .created => "待支付",
        .paid => "已支付",
        .shipping => "配送中",
        .finished => "已完成",
        .cancelled => "已取消",
    };
}

pub fn main() void {
    const status: OrderStatus = .shipping;
    std.debug.print("{s}\n", .{statusText(status)});
}
~~~

枚举成员可以使用 .shipping 简写，因为匹配对象已经是 OrderStatus。

多个状态共享逻辑时使用逗号：

~~~zig
fn isEndState(status: OrderStatus) bool {
    return switch (status) {
        .finished, .cancelled => true,
        .created, .paid, .shipping => false,
    };
}
~~~

## 六、switch 与可选值

可选值 ?T 有两个可能：一个真实的 T，或者 null。Zig 0.16.0 不能直接对可选值使用 switch，需要先用 if 解包：

~~~zig
const std = @import("std");

fn doubleOrDefault(value: ?i32) i32 {
    if (value) |number| {
        return number * 2;
    } else {
        return 0;
    }
}

pub fn main() void {
    std.debug.print("{}\n", .{doubleOrDefault(21)});
    std.debug.print("{}\n", .{doubleOrDefault(null)});
}
~~~

if (value) |number| 会把非空的 i32 解包出来；value 为 null 时执行 else 分支。可选值不能直接作为 switch 的操作数，这一点和 Tagged Union 不同。

## 七、Tagged Union：匹配标签并取出载荷

普通 enum 只能记录当前标签，Tagged Union 还可以保存对应数据：

~~~zig
const std = @import("std");

const Shape = union(enum) {
    circle: f32,
    rectangle: struct {
        width: f32,
        height: f32,
    },
    point,
};

fn area(shape: Shape) f32 {
    return switch (shape) {
        .circle => |radius| 3.14159 * radius * radius,
        .rectangle => |rect| rect.width * rect.height,
        .point => 0,
    };
}

pub fn main() void {
    const shape: Shape = .{
        .rectangle = .{
            .width = 4,
            .height = 3,
        },
    };

    std.debug.print("{d:.2}\n", .{area(shape)});
}
~~~

rectangle 分支中的 |rect| 同时完成匹配变体和取出载荷。

需要修改载荷时，对 union 值进行 switch，并使用指针捕获：

~~~zig
var shape: Shape = .{
    .rectangle = .{
        .width = 4,
        .height = 3,
    },
};

switch (shape) {
    .rectangle => |*rect| {
        rect.width *= 2;
        rect.height *= 2;
    },
    else => {},
}
~~~

|*rect| 是指针捕获，修改会作用于 shape 内部的原始载荷。

这里的 union 值必须来自可变存储。若写成 `const shape = ...`，或者把 `shape` 按值传入一个函数，载荷仍然是只读的，修改时会出现 `cannot assign to constant`。需要在函数中修改时，传入 Tagged Union 指针：

```zig
fn scaleRectangle(shape: *Shape) void {
    switch (shape.*) {
        .rectangle => |*rect| {
            rect.width *= 2;
            rect.height *= 2;
        },
        else => {},
    }
}

pub fn main() void {
    var shape: Shape = .{
        .rectangle = .{ .width = 4, .height = 3 },
    };

    scaleRectangle(&shape);
}
```

关键点是 `var shape`、`&shape` 和函数内部的 `shape.*`：前两步把可变地址传入，最后一步把指针还原为 union 值供 switch 匹配。

## 八、错误集与 switch

~~~zig
const std = @import("std");

const ParseError = error{
    InvalidNumber,
    UnexpectedEnd,
};

fn printError(err: ParseError) void {
    switch (err) {
        error.InvalidNumber => std.debug.print("数字格式错误\n", .{}),
        error.UnexpectedEnd => std.debug.print("输入提前结束\n", .{}),
    }
}

pub fn main() void {
    printError(error.InvalidNumber);
}
~~~

错误集是有限的错误值集合，但不是普通 enum。错误值使用 error.Name，错误联合通常写成 ErrorSet!T 或 !T。

## 九、标签 switch 与 continue

带标签的 switch 可以使用 continue 把新值重新送回 switch：

~~~zig
const std = @import("std");

pub fn main() void {
    const result = blk: switch (@as(u8, 1)) {
        1 => continue :blk 2,
        2 => continue :blk 3,
        3 => break :blk "完成",
        else => break :blk "未知状态",
    };

    std.debug.print("{s}\n", .{result});
}
~~~

正确的标签位置是 label: switch (value)，不是 switch :label (value)。

continue :label new_value 会让 switch 重新执行，并用 new_value 重新匹配；break :label result 则返回结果并结束带标签的 switch。

### 使用标签 switch 表达状态机

~~~zig
const std = @import("std");

pub fn main() void {
    var count: u8 = 0;

    const result = fsm: switch (@as(u8, 0)) {
        0 => {
            std.debug.print("初始化\n", .{});
            count += 1;
            if (count < 3) {
                continue :fsm 1;
            }
            break :fsm "完成";
        },
        1 => {
            std.debug.print("处理第 {} 次\n", .{count});
            count += 1;
            if (count < 3) {
                continue :fsm 0;
            }
            break :fsm "完成";
        },
        else => break :fsm "未知状态",
    };

    std.debug.print("结果：{s}\n", .{result});
}
~~~

这种写法适合有限步骤的解析器和紧凑状态转换。普通业务状态机使用 while + switch 往往更容易调试。

## 十、标签块：复杂分支返回结果

switch 的某个分支需要多条语句时，可以使用标签块：

~~~zig
const value: u64 = 101;

const result = switch (value) {
    101 => blk: {
        const base: u64 = 5;
        const extra: u64 = 2;
        break :blk base * extra + 1;
    },
    else => 0,
};
~~~

break :blk expression 会把结果返回给代码块，适合在 switch 表达式中承载局部计算。

## 十一、inline switch 与 comptime

inline 常用于编译期反射和泛型代码。下面的例子根据字段类型判断某个字段是否为 optional：

~~~zig
const std = @import("std");

const Config = struct {
    name: []const u8,
    port: ?u16,
};

fn fieldIsOptional(comptime T: type, comptime index: usize) bool {
    const fields = @typeInfo(T).@"struct".fields;
    const field_type = fields[index].type;

    return switch (@typeInfo(field_type)) {
        .optional => true,
        else => false,
    };
}

pub fn main() void {
    const result = fieldIsOptional(Config, 1);
    std.debug.print("{}\n", .{result});
}
~~~

普通业务代码不需要为了形式而使用 inline。只有分支本身依赖编译期类型信息时，inline 才有明显价值。

## 十二、完整实战 Demo：HTTP 状态码处理

外部 HTTP 服务可能返回当前代码没有命名的状态码。非穷尽枚举可以保留未知整数，再用 _ 兜底：

~~~zig
const std = @import("std");

const HttpStatus = enum(u16) {
    ok = 200,
    created = 201,
    bad_request = 400,
    not_found = 404,
    internal_error = 500,
    _,
};

fn statusText(code: u16) []const u8 {
    const status: HttpStatus = @enumFromInt(code);

    return switch (status) {
        .ok, .created => "成功",
        .bad_request => "客户端错误",
        .not_found => "资源不存在",
        .internal_error => "服务器错误",
        _ => "未知状态码",
    };
}

pub fn main() void {
    std.debug.print("{s}\n", .{statusText(200)});
    std.debug.print("{s}\n", .{statusText(404)});
    std.debug.print("{s}\n", .{statusText(999)});
}
~~~

输出：

~~~text
成功
资源不存在
未知状态码
~~~

这里的 _ 不是普通 enum 成员，而是非穷尽枚举允许的未知值。

## 十三、完整实战 Demo：命令分发器

~~~zig
const std = @import("std");

const Command = enum {
    start,
    stop,
    status,
    restart,
};

fn execute(command: Command) ![]const u8 {
    return switch (command) {
        .start => "服务已启动",
        .stop => "服务已停止",
        .status => "服务运行中",
        .restart => blk: {
            std.debug.print("正在重启...\n", .{});
            break :blk "服务已重启";
        },
    };
}

pub fn main() !void {
    const commands = [_]Command{ .start, .status, .restart };

    for (commands) |command| {
        const message = try execute(command);
        std.debug.print("{s}\n", .{message});
    }
}
~~~

新增命令成员后，execute 会因为没有处理新分支而编译失败，分发逻辑不会悄悄遗漏。

## 十四、常见误区

### 1. 期待 C 风格 fall-through

Zig 不会自动穿透到下一个分支。多个值走同一逻辑时，直接写：

~~~zig
1, 2, 3 => handleSmall(),
~~~

### 2. 把闭区间当成半开区间

80...89 包含 80 和 89 两个端点。如果需要半开区间，要拆成明确的条件或调整边界。

### 3. 忘记 switch 表达式的统一结果类型

所有分支都要能转换到同一个类型，不能一边返回整数、一边返回字符串。

### 4. 对普通 union 使用 switch

普通 union 没有标签，不能直接安全地使用 switch。需要根据当前变体匹配时，应使用 union(enum)。

### 5. 用值捕获修改原对象

值捕获拿到的是载荷值；需要修改原始载荷时，在 union 值的 switch 分支中使用 |*value|。

### 6. 对整数写大量离散 case

数字分类优先使用范围匹配。只有业务含义确实不同的离散值，才逐个列出。

## 十五、switch 的选型口诀

- 枚举状态：优先写完整分支，不急着加 else。
- 整数分类：使用 a...b 范围和 else。
- 多个值相同处理：用逗号合并。
- 需要结果：使用 switch 表达式。
- Tagged Union：用 |value| 解构载荷。
- 需要修改载荷：在 switch 分支中使用 |*value|。
- 可选值：使用 if 解包 null 和非空值；不要直接 switch ?T。
- 外部未知整数：使用非穷尽枚举和 _。
- 复杂单分支计算：使用标签块和 break :label。
- 状态反复跳转：谨慎使用标签 switch 的 continue。

switch 的真正价值，不是少写几行 if，而是把“输入有哪些可能、每种可能如何处理、结果是什么”集中写在一个可检查的表达式里。配合 enum，编译器可以发现遗漏状态；配合 Tagged Union，可以安全拆出不同载荷；配合范围和标签块，又能覆盖数字分类与状态机等底层场景。

## 参考资料

- [Zig 0.16.0 Language Reference：switch](https://ziglang.org/documentation/0.16.0/#switch)
- [Zig 0.16.0 Language Reference：Exhaustive Switching](https://ziglang.org/documentation/0.16.0/#Exhaustive-Switching)
- [Zig 0.16.0 Language Reference：Labeled switch](https://ziglang.org/documentation/0.16.0/#Labeled-switch)
- [Zig 0.16.0 Language Reference：Tagged union](https://ziglang.org/documentation/0.16.0/#Tagged-union)
