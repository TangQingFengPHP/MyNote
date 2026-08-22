# 一个值多种形态：Zig union、Tagged Union 与内存布局实战

当一个值可能是整数、字符串或一组结构化数据时，union 可以把这些可能性组织成一个类型。多个成员共享同一块存储，同一时刻只有一个成员有效。

Zig 中常见的 union 有四种：

~~~text
union { ... }          裸联合，没有自动标签
union(enum) { ... }    Tagged Union，带标签，业务代码首选
extern union { ... }   C ABI 联合
packed union { ... }   位级布局联合
~~~

其中最容易混淆的是：union 并不会默认带标签。需要安全判断当前成员时，应明确使用 union(enum)。

本文按 Zig 0.16.0 语法编写，示例可以直接保存为 .zig 文件运行。

## 一、union 与 struct 的区别

struct 的成员同时存在：

~~~zig
const StructData = struct {
    integer: i32,
    decimal: f32,
};
~~~

union 的成员共享存储：

~~~zig
const UnionData = union {
    integer: i32,
    decimal: f32,
};
~~~

可以粗略理解成：

~~~text
struct: integer + decimal
union:  integer 或 decimal
~~~

union 的大小通常由最大成员和对齐要求决定，不会同时保存所有成员的数据。

## 二、裸 union：共享内存但没有 tag

~~~zig
const std = @import("std");

const Value = union {
    integer: i32,
    decimal: f32,
};

pub fn main() void {
    var value: Value = .{ .integer = 100 };
    std.debug.print("{}\n", .{value.integer});

    value = .{ .decimal = 3.14 };
    std.debug.print("{d:.2}\n", .{value.decimal});
}
~~~

给 decimal 赋值后，之前的 integer 内容已经失效。裸 union 不记录当前激活成员，因此读取前必须由程序自行确认：

~~~zig
// 当前激活的是 integer，不能直接读取 value.decimal
~~~

安全模式下读取非当前成员可能触发运行时错误；关闭安全检查后可能得到无意义的数据。裸 union 适合已有外部 tag 的底层数据，业务代码通常不优先使用。

## 三、Tagged Union：业务代码首选

~~~zig
const Value = union(enum) {
    integer: i32,
    decimal: f64,
    text: []const u8,
    none,
};
~~~

Zig 会为 union(enum) 维护一个隐藏的枚举标签，记录当前激活的成员。

~~~zig
const a: Value = .{ .integer = 100 };
const b: Value = .{ .decimal = 3.14 };
const c: Value = .{ .text = "hello" };
const d: Value = .none;
~~~

不携带数据的成员可以省略 void：

~~~zig
const A = union(enum) {
    none: void,
};

const B = union(enum) {
    none,
};
~~~

## 四、switch 与 Tagged Union

Tagged Union 最常见的用法是配合 switch：

~~~zig
const std = @import("std");

const Value = union(enum) {
    integer: i32,
    decimal: f64,
    text: []const u8,
    none,
};

fn printValue(value: Value) void {
    switch (value) {
        .integer => |number| {
            std.debug.print("整数：{}\n", .{number});
        },
        .decimal => |number| {
            std.debug.print("小数：{d:.2}\n", .{number});
        },
        .text => |text| {
            std.debug.print("文本：{s}\n", .{text});
        },
        .none => {
            std.debug.print("空值\n", .{});
        },
    }
}

pub fn main() void {
    printValue(.{ .integer = 100 });
    printValue(.{ .decimal = 3.14 });
    printValue(.{ .text = "hello Zig" });
    printValue(.none);
}
~~~

分支中的捕获变量就是当前成员携带的载荷。switch 会检查所有 Tagged Union 成员，新增成员后，遗漏处理的函数会在编译阶段暴露。

## 五、如何修改 union 载荷

普通捕获用于读取：

~~~zig
switch (value) {
    .integer => |number| {
        _ = number;
    },
    else => {},
}
~~~

需要修改载荷时，使用指针捕获：

~~~zig
const Counter = union(enum) {
    integer: i32,
    decimal: f32,
};

pub fn main() void {
    var counter: Counter = .{ .integer = 10 };

    switch (counter) {
        .integer => |*number| number.* += 5,
        .decimal => |*number| number.* *= 2,
    }

    std.debug.print("{}\n", .{counter.integer});
}
~~~

counter 必须是 var。若写成 const counter，修改会报 cannot assign to constant。

修改逻辑放在函数中时，传入 union 指针：

~~~zig
fn increase(counter: *Counter) void {
    switch (counter.*) {
        .integer => |*number| number.* += 1,
        .decimal => |*number| number.* += 1,
    }
}

var counter: Counter = .{ .integer = 10 };
increase(&counter);
~~~

这里的关系是：var 提供可变存储，&counter 传入地址，counter.* 取出 union，指针捕获再修改载荷。

## 六、union 可以携带不同结构

~~~zig
const Event = union(enum) {
    mouse_click: struct {
        x: i32,
        y: i32,
        button: u8,
    },
    key_press: u32,
    quit,
};

fn describe(event: Event) void {
    switch (event) {
        .mouse_click => |click| {
            std.debug.print(
                "点击 x={}, y={}, button={}\n",
                .{ click.x, click.y, click.button },
            );
        },
        .key_press => |key| {
            std.debug.print("按键：{}\n", .{key});
        },
        .quit => {
            std.debug.print("退出\n", .{});
        },
    }
}
~~~

消息、事件、协议包和 AST 节点都适合这种建模方式。每种消息只声明自己需要的字段，不需要堆积一组互相无关的 optional 字段。

## 七、显式指定标签类型

union(enum) 会自动推导标签枚举，也可以显式提供：

~~~zig
const EventTag = enum {
    text,
    number,
};

const Event = union(EventTag) {
    text: []const u8,
    number: i64,
};

const event: Event = .{ .number = 42 };
const tag: EventTag = @as(EventTag, event);
~~~

显式标签类型适合多个类型共享同一组标签的场景。一般业务代码使用 union(enum) 更简洁。

获取当前标签名称：

~~~zig
std.debug.print("{s}\n", .{@tagName(event)});
~~~

## 八、四种 union 的区别

| 类型 | 是否保存 tag | 能否直接 switch | 典型用途 |
| --- | --- | --- | --- |
| union { ... } | 否 | 否 | 已有外部 tag 的底层数据 |
| union(enum) { ... } | 是 | 是 | 业务消息、状态、AST |
| extern union { ... } | 否 | 否 | C ABI、FFI |
| packed union { ... } | 否 | 否 | 位级数据、硬件和协议 |

需要安全判断当前成员时，使用 union(enum)。

## 九、extern union：C ABI 联合

extern union 的内存布局匹配目标平台的 C ABI：

~~~zig
const CValue = extern union {
    integer: i32,
    decimal: f32,
};
~~~

它不会自动记录当前成员。C API 通常会额外传入 tag：

~~~zig
const Kind = enum(c_int) {
    integer = 1,
    decimal = 2,
};

const CValue = extern union {
    integer: i32,
    decimal: f32,
};

fn printCValue(kind: Kind, value: CValue) void {
    switch (kind) {
        .integer => std.debug.print("{}\n", .{value.integer}),
        .decimal => std.debug.print("{d:.2}\n", .{value.decimal}),
    }
}
~~~

kind 和 value 必须来自同一套外部协议。extern union 只负责布局，不负责业务安全。

## 十、packed union：位级布局

packed union 用于需要明确位表示的底层数据，成员需要满足 packed union 的位宽规则：

~~~zig
const Bits = packed union(u16) {
    raw: u16,
    parts: packed struct {
        low: u8,
        high: u8,
    },
};

var bits: Bits = .{ .raw = 0x1234 };
std.debug.print("raw=0x{X}\n", .{bits.raw});
~~~

字节顺序和目标平台端序有关，不能把结果写死成只适用于小端机器的结论。packed union 适合寄存器、协议字段和位级重解释，不适合替代 Tagged Union。

## 十一、union 的大小与值语义

可以用 @sizeOf 和 @alignOf 查看布局：

~~~zig
const std = @import("std");

const Payload = union(enum) {
    small: u8,
    large: u64,
    text: []const u8,
};

pub fn main() void {
    std.debug.print("size={}\n", .{@sizeOf(Payload)});
    std.debug.print("align={}\n", .{@alignOf(Payload)});
}
~~~

裸 union 的数据区至少容纳最大成员；Tagged Union 还要保存 tag，因此通常会占用更多空间。最终大小受对齐和目标平台影响。

union 和 struct 一样是值类型：

~~~zig
const Value = union(enum) {
    integer: i32,
    text: []const u8,
};

var first: Value = .{ .integer = 10 };
var second = first;
second = .{ .integer = 20 };
~~~

修改 second 不会改变 first。需要修改原 union 时，传入 *Value：

~~~zig
fn setText(value: *Value) void {
    value.* = .{ .text = "changed" };
}
~~~

## 十二、完整实战：事件队列消息

~~~zig
const std = @import("std");

const Message = union(enum) {
    start: struct {
        worker_id: u32,
    },
    data: struct {
        value: i32,
    },
    stop,
};

fn handle(message: Message) void {
    switch (message) {
        .start => |info| {
            std.debug.print("启动 worker {}\n", .{info.worker_id});
        },
        .data => |info| {
            std.debug.print("收到数据 {}\n", .{info.value});
        },
        .stop => {
            std.debug.print("停止 worker\n", .{});
        },
    }
}

pub fn main() void {
    const messages = [_]Message{
        .{ .start = .{ .worker_id = 7 } },
        .{ .data = .{ .value = 100 } },
        .stop,
    };

    for (messages) |message| {
        handle(message);
    }
}
~~~

运行结果：

~~~text
启动 worker 7
收到数据 100
停止 worker
~~~

Tagged Union 比一个大 struct 加多个 optional 字段更准确：

~~~text
start  必须有 worker_id
data   必须有 value
stop   不携带数据
~~~

## 十三、完整实战：简单 AST 节点

~~~zig
const std = @import("std");

const Expr = union(enum) {
    number: i32,
    add: struct {
        left: *const Expr,
        right: *const Expr,
    },
    multiply: struct {
        left: *const Expr,
        right: *const Expr,
    },
};

fn evaluate(expr: *const Expr) i32 {
    return switch (expr.*) {
        .number => |number| number,
        .add => |node| evaluate(node.left) + evaluate(node.right),
        .multiply => |node| evaluate(node.left) * evaluate(node.right),
    };
}

pub fn main() void {
    const two = Expr{ .number = 2 };
    const three = Expr{ .number = 3 };
    const four = Expr{ .number = 4 };

    const product = Expr{
        .multiply = .{
            .left = &three,
            .right = &four,
        },
    };

    const expression = Expr{
        .add = .{
            .left = &two,
            .right = &product,
        },
    };

    std.debug.print("{}\n", .{evaluate(&expression)});
}
~~~

这棵树表达 2 + (3 * 4)，运行结果为 14。每一种节点类型都由 union 标签区分，递归计算只需要用 switch 覆盖全部变体。

## 十四、@unionInit

字段名在编译期已知时，可以使用 @unionInit：

~~~zig
const std = @import("std");

const Value = union(enum) {
    integer: i32,
    text: []const u8,
};

pub fn main() void {
    const field_name = "integer";
    const value = @unionInit(Value, field_name, 666);

    switch (value) {
        .integer => |number| std.debug.print("{}\n", .{number}),
        .text => |text| std.debug.print("{s}\n", .{text}),
    }
}
~~~

字段名来自运行时字符串时，不能直接使用 @unionInit，需要先映射到明确分支。

## 十五、常见错误与选型

1. 把 union 当成自动带 tag 的类型。需要 switch 解构时，使用 union(enum)。
2. 读取非当前成员。Tagged Union 当前是 integer 时，不要直接读取 text。
3. 用 const union 修改载荷。局部变量使用 var，函数参数使用 *Union。
4. 认为 extern union 会自动检查 tag。extern union 只保证 C ABI 布局。
5. 把 packed union 当作普通业务容器。位宽、端序和对齐都需要明确处理。

选型口诀：

- 业务数据多种形态：union(enum)；
- 需要 switch 和载荷解构：union(enum)；
- 外部 C API：extern union；
- 已经有独立 tag 的底层数据：裸 union；
- 位级重解释和寄存器：packed union。

union 的核心不是“节省内存”这么简单，而是把“一个值可能有哪些形态”写进类型。裸 union 只提供共享存储；Tagged Union 进一步提供当前标签；switch 再把标签和载荷安全地连接起来。

## 参考资料

- [Zig 0.16.0 Language Reference：union](https://ziglang.org/documentation/0.16.0/#union)
- [Zig 0.16.0 Language Reference：Tagged union](https://ziglang.org/documentation/0.16.0/#Tagged-union)
- [Zig 0.16.0 Language Reference：extern union](https://ziglang.org/documentation/0.16.0/#extern-union)
- [Zig 0.16.0 Language Reference：packed union](https://ziglang.org/documentation/0.16.0/#packed-union)
