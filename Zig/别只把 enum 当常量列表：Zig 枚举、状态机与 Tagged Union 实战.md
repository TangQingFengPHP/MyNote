# 别只把 enum 当常量列表：Zig 枚举、状态机与 Tagged Union 实战

枚举最简单的用途，是给一组固定选项起名字：订单有 `pending`、`paid`、`shipped`，日志有 `info`、`warn`、`error`。

但 Zig 的 `enum` 不止是“几个整数常量”。它有强类型约束，可以和 `switch` 做穷尽检查，可以指定底层整数类型，可以定义方法，还能和 `union` 组合成携带不同数据的 Tagged Union。状态机、协议解析、命令分发和错误分类，都适合用这套能力表达。

本文按 Zig 0.16.0 语法编写，示例可以直接保存为 `.zig` 文件运行。

## 一、最基础的 enum

```zig
const std = @import("std");

const Color = enum {
    red,
    green,
    blue,
};

pub fn main() void {
    const color1 = Color.red;
    const color2: Color = .green;

    std.debug.print("{} {}\n", .{ color1, color2 });
}
```

`Color` 是一个枚举类型，`Color.red` 是其中一个枚举值。已知目标类型时，可以省略枚举名，写成 `.green`：

```zig
const color: Color = .blue;
```

这种写法叫枚举字面量。`.blue` 只有放在能够推导出 `Color` 的上下文里才成立；单独写 `const color = .blue`，编译器没有足够信息判断它属于哪个枚举。

枚举是强类型。下面两个 `red` 虽然名字相同，但属于不同类型：

```zig
const Color = enum { red, green };
const Status = enum { red, green };

const color = Color.red;
// const status: Status = color; // 编译错误，Color 和 Status 不是同一种类型
```

这能避免不同业务含义的整数或标签意外混用。

## 二、枚举与 `switch`：让状态处理完整

枚举最常和 `switch` 一起使用：

```zig
const std = @import("std");

const Status = enum {
    pending,
    running,
    finished,
};

fn statusText(status: Status) []const u8 {
    return switch (status) {
        .pending => "等待中",
        .running => "运行中",
        .finished => "已完成",
    };
}

pub fn main() void {
    std.debug.print("{s}\n", .{statusText(.running)});
}
```

这里没有 `else`，但三个枚举成员全部处理了，所以 `switch` 是完整的。以后给 `Status` 增加一个 `.failed`，这个函数会立刻出现编译错误，提醒补上处理逻辑。

这比一串整数判断更可靠：

```zig
// 不推荐：魔术数字没有业务含义
// if (status_code == 2) { ... }
```

多个分支可以合并：

```zig
fn isFinished(status: Status) bool {
    return switch (status) {
        .finished => true,
        .pending, .running => false,
    };
}
```

如果确实需要兜底，也可以使用 `else`：

```zig
fn shortText(status: Status) []const u8 {
    return switch (status) {
        .finished => "done",
        else => "not done",
    };
}
```

对于普通穷尽枚举，`else` 会让新增枚举成员时失去一部分编译期提醒。状态规则比较重要时，优先逐项列出分支。

## 三、枚举的底层整数值

默认情况下，枚举成员从 `0` 开始递增：

```zig
const Level = enum(u8) {
    low,    // 0
    normal, // 1
    high,   // 2
};
```

指定底层类型后，可以把枚举转换成对应的整数：

```zig
const std = @import("std");

const HttpStatus = enum(u16) {
    ok = 200,
    not_found = 404,
    internal_error = 500,
};

pub fn main() void {
    const code: u16 = @intFromEnum(HttpStatus.not_found);
    std.debug.print("{}\n", .{code});
}
```

`@intFromEnum` 读取枚举的整数标签。它适合协议编码、日志输出和与外部数值对接，但不应该让业务代码到处依赖数字本身。

也可以从整数转回枚举：

```zig
const raw: u16 = 404;
const status: HttpStatus = @enumFromInt(raw);
```

`@enumFromInt` 的目标类型来自左侧的 `HttpStatus`。旧版本资料中常见的 `@enumFromInt(HttpStatus, raw)` 是旧写法，当前 Zig 语法不需要把类型作为第一个参数传入。

对普通穷尽枚举来说，整数必须对应已经声明的成员。非法值在编译期可能直接报错，运行时则会触发安全检查错误：

```zig
const raw: u16 = 401;
// const status: HttpStatus = @enumFromInt(raw); // 运行时非法枚举值
```

如果输入来自网络、文件或用户，不能直接信任整数。更稳妥的方式是先用 `switch` 或范围判断完成业务校验，再转换。

## 四、跳号和显式标签值

枚举成员可以只给部分值赋值，后续未指定的成员会从前一个值继续递增：

```zig
const Token = enum(u8) {
    identifier = 1,
    number = 10,
    string, // 11
    keyword = 20,
    operator, // 21
};
```

显式数值适合协议码、系统调用编号或稳定的持久化格式。普通业务状态通常不需要手动编号，因为业务关心的是标签，不是底层整数。

## 五、`@tagName`：把枚举标签变成字符串

调试日志或展示文本经常需要枚举成员名称，可以使用 `@tagName`：

```zig
const std = @import("std");

const Direction = enum {
    north,
    south,
    east,
    west,
};

pub fn main() void {
    const direction = Direction.east;
    std.debug.print("方向：{s}\n", .{@tagName(direction)});
}
```

`@tagName` 返回以 `0` 结尾的编译期字符串，适合直接打印。它返回的是标签名 `east`，不是业务翻译文本。如果界面需要“东”，仍然适合通过 `switch` 映射：

```zig
fn directionText(direction: Direction) []const u8 {
    return switch (direction) {
        .north => "北",
        .south => "南",
        .east => "东",
        .west => "西",
    };
}
```

## 六、枚举也可以定义方法

枚举内部的方法，本质上仍然是带命名空间的函数：

```zig
const std = @import("std");

const TrafficLight = enum {
    red,
    yellow,
    green,

    pub fn duration(self: TrafficLight) u32 {
        return switch (self) {
            .red => 30,
            .yellow => 5,
            .green => 25,
        };
    }

    pub fn next(self: TrafficLight) TrafficLight {
        return switch (self) {
            .red => .green,
            .green => .yellow,
            .yellow => .red,
        };
    }
};

pub fn main() void {
    var light: TrafficLight = .red;
    std.debug.print("当前={}秒\n", .{light.duration()});

    light = light.next();
    std.debug.print("下一个：{s}\n", .{@tagName(light)});
}
```

`light.duration()` 只是 `TrafficLight.duration(light)` 的点号调用形式。方法适合放置与枚举状态紧密相关的转换规则，例如“下一个状态”“是否终态”“对应超时时间”等。

## 七、完整实战：用 enum 写一个任务状态机

状态机的核心是：当前状态 + 输入事件 = 下一个状态。把状态和事件分别定义成枚举，再用 `switch` 写转换规则，结构会非常清楚。

```zig
const std = @import("std");

const TaskState = enum {
    todo,
    running,
    success,
    failed,
};

const TaskEvent = enum {
    start,
    complete,
    fail,
    retry,
};

const Task = struct {
    state: TaskState = .todo,

    fn dispatch(self: *Task, event: TaskEvent) !void {
        self.state = switch (self.state) {
            .todo => switch (event) {
                .start => .running,
                else => return error.InvalidEvent,
            },
            .running => switch (event) {
                .complete => .success,
                .fail => .failed,
                else => return error.InvalidEvent,
            },
            .failed => switch (event) {
                .retry => .running,
                else => return error.InvalidEvent,
            },
            .success => return error.TaskAlreadyFinished,
        };
    }

    fn print(self: *const Task) void {
        std.debug.print("状态：{s}\n", .{@tagName(self.state)});
    }
};

pub fn main() !void {
    var task = Task{};
    task.print();

    try task.dispatch(.start);
    task.print();

    try task.dispatch(.complete);
    task.print();

    if (task.dispatch(.retry)) |_| {
        unreachable;
    } else |err| {
        std.debug.print("事件被拒绝：{}\n", .{err});
    }
}
```

运行结果：

```text
状态：todo
状态：running
状态：success
事件被拒绝：error.TaskAlreadyFinished
```

这个 demo 中的 `TaskState` 只负责描述状态，`TaskEvent` 只负责描述外部事件，`Task.dispatch` 负责维护状态转移规则。非法转移通过错误返回，不需要用注释约定“这个状态不能调用某个方法”。

## 八、`union(enum)`：枚举标签也可以携带数据

普通 `enum` 只能表示“当前是哪一个标签”，不能携带额外数据。例如，文件操作结果可能是成功并带有文件大小，也可能失败并带有错误信息。这时可以使用 Tagged Union：

```zig
const Result = union(enum) {
    ok: usize,
    not_found: []const u8,
    permission_denied: []const u8,
};
```

创建值：

```zig
const success: Result = .{ .ok = 1024 };
const missing: Result = .{ .not_found = "config.json" };
```

使用 `switch` 时，可以同时匹配标签并取出载荷：

```zig
const std = @import("std");

const Result = union(enum) {
    ok: usize,
    not_found: []const u8,
    permission_denied: []const u8,
};

fn printResult(result: Result) void {
    switch (result) {
        .ok => |size| std.debug.print("读取成功，大小={}\n", .{size}),
        .not_found => |path| std.debug.print("文件不存在：{s}\n", .{path}),
        .permission_denied => |path| std.debug.print("没有权限：{s}\n", .{path}),
    }
}

pub fn main() void {
    printResult(.{ .ok = 1024 });
    printResult(.{ .not_found = "config.json" });
}
```

`union(enum)` 内部有一个隐藏的枚举标签，用来记录当前激活的是哪个字段；同时只能有一个载荷处于有效状态。`switch` 会检查所有变体，避免读取错误的载荷。

普通 `union` 没有这个标签，代码必须自行记录当前激活字段；只有 Tagged Union 才适合直接用 `switch` 安全分支处理。

### 自定义 Tagged Union 的标签类型

`union(enum)` 让 Zig 自动生成标签类型，也可以显式提供一个枚举：

```zig
const EventTag = enum {
    text,
    number,
};

const Event = union(EventTag) {
    text: []const u8,
    number: i64,
};
```

这种写法适合多个类型需要共享同一套标签，或者需要明确访问标签类型的场景。大多数业务代码使用 `union(enum)` 就足够了。

## 九、非穷尽枚举：允许出现未命名值

普通枚举必须覆盖所有可能的值。与外部系统交互时，底层整数可能包含当前版本还不认识的值，这时可以使用非穷尽枚举：

```zig
const DeviceState = enum(u8) {
    offline = 0,
    online = 1,
    _,
};
```

末尾的 `_` 表示还有未命名的合法值。整数转换：

```zig
const raw: u8 = 200;
const state: DeviceState = @enumFromInt(raw);
```

对于非穷尽枚举，`@enumFromInt` 可以产生未命名值。`switch` 中可以使用 `_` 处理未知值，同时仍然逐项列出已知成员：

```zig
fn stateText(state: DeviceState) []const u8 {
    return switch (state) {
        .offline => "离线",
        .online => "在线",
        _ => "未知状态",
    };
}
```

`else` 也可以兜底，但 `_` 更适合非穷尽枚举：它表示“其他标签”，同时能让编译器继续检查已知标签是否遗漏。

## 十、C 互操作：`extern enum`

普通 Zig 枚举默认不保证 C ABI 兼容。需要导出到 C 或接收 C API 的枚举时，使用 `extern enum` 并明确指定底层类型：

```zig
const Signal = extern enum(c_int) {
    interrupt = 2,
    kill = 9,
};
```

`extern enum` 适合 ABI 场景，不是普通枚举的“增强版本”。业务代码优先使用普通 `enum`，只有跨语言边界需要稳定布局时才使用 `extern enum`。

## 十一、枚举与错误集不是同一种类型

错误集的写法和枚举很像：

```zig
const ParseError = error{
    InvalidNumber,
    UnexpectedEnd,
};
```

错误集也可以使用 `switch`：

```zig
fn printError(err: ParseError) void {
    switch (err) {
        .InvalidNumber => std.debug.print("数字格式错误\n", .{}),
        .UnexpectedEnd => std.debug.print("输入提前结束\n", .{}),
    }
}
```

但错误集不是普通 `enum`。错误值使用 `error.Name` 产生，函数返回错误时通常写成 `ParseError!T` 或 `!T`；普通枚举则使用 `Type.member` 或上下文中的 `.member`。两者都适合表示有限集合，但用途和类型系统语义不同。

## 十二、通过 `@typeInfo` 做编译期反射

需要在编译期遍历所有枚举成员时，可以使用 `@typeInfo`：

```zig
const std = @import("std");

const Color = enum {
    red,
    green,
    blue,
};

pub fn main() void {
    const info = @typeInfo(Color).@"enum";

    inline for (info.fields) |field| {
        std.debug.print("{s}\n", .{field.name});
    }
}
```

`inline for` 会在编译期展开循环，适合生成注册表、命令帮助信息或测试。只需要打印当前值时，`@tagName` 更简单；只有需要遍历整个枚举定义时，才有必要使用 `@typeInfo`。

## 十三、枚举和位标志不是一回事

枚举表示“多个选项中当前只有一个”：

```zig
const Mode = enum(u8) {
    read,
    write,
};
```

权限、功能开关这类场景往往允许多个选项同时存在，不应该用普通枚举：

```zig
const Permissions = packed struct(u8) {
    read: bool = false,
    write: bool = false,
    execute: bool = false,
    reserved: u5 = 0,
};

const permissions = Permissions{
    .read = true,
    .write = true,
};
```

这里 `read` 和 `write` 可以同时为 `true`，所以它是位标志集合，不是枚举。单选使用 `enum`，多选使用 `packed struct` 或专门的位运算封装，语义会更准确。

## 十四、常见错误

### 1. 直接把枚举和整数比较

```zig
const State = enum(u8) { idle, running };
const state = State.idle;

// if (state == 0) {} // 类型不匹配
if (@intFromEnum(state) == 0) {
    // 只有明确需要比较协议数值时，才转换为整数
}
```

业务判断更建议写成 `state == .idle`，这样不会依赖底层编号。

### 2. 用非法整数强行转换

```zig
const State = enum(u8) { idle, running };
const raw: u8 = 99;
// const state: State = @enumFromInt(raw); // 非法枚举值
```

外部输入先校验，再转换；无法列举所有外部值时，考虑非穷尽枚举或保留原始整数。

### 3. `switch` 漏掉成员

普通枚举的 `switch` 要么覆盖所有成员，要么提供 `else`。状态转移逻辑最好不要随意用 `else`，否则新状态可能静默落入旧的兜底分支。

### 4. 用 enum 模拟多选

`enum` 同一时间只有一个值。权限、功能开关、多个过滤条件同时启用时，应使用位标志集合。

### 5. 把 `union(enum)` 当成普通 enum

普通枚举只有标签；`union(enum)` 还有与标签对应的载荷。创建 Tagged Union 时需要为带载荷的字段提供值：

```zig
const value: union(enum) {
    none,
    number: i32,
} = .{ .number = 42 };
```

## 十五、选择哪一种写法

- 一组固定的单选状态：普通 `enum`。
- 协议或数据库需要稳定数值：`enum(u8)`、`enum(u16)` 等显式底层类型。
- 输入可能包含未识别标签：非穷尽枚举 `enum(u8) { known, _ }`。
- C ABI 交互：`extern enum(c_int)` 或目标 ABI 要求的整数类型。
- 一个状态需要携带不同类型数据：`union(enum)`。
- 多个开关可以同时打开：`packed struct` 或位标志封装。
- 错误分类：错误集 `error{ ... }`，不要和普通 enum 混用。

枚举真正有价值的地方，不是把数字换成了单词，而是把“允许出现哪些状态”写进了类型。再配合 `switch` 的完整性检查和 `union(enum)` 的载荷，状态变化、协议消息和业务分支都能在编译期获得更清晰的约束。

## 参考资料

- [Zig 0.16.0 Language Reference：enum](https://ziglang.org/documentation/0.16.0/#enum)
- [Zig 0.16.0 Language Reference：Enum Literals](https://ziglang.org/documentation/0.16.0/#Enum-Literals)
- [Zig 0.16.0 Language Reference：Non-exhaustive enum](https://ziglang.org/documentation/0.16.0/#Non-exhaustive-enum)
- [Zig 0.16.0 Language Reference：Tagged union](https://ziglang.org/documentation/0.16.0/#Tagged-union)
- [Zig 0.16.0 Language Reference：@enumFromInt](https://ziglang.org/documentation/0.16.0/#@enumFromInt)
