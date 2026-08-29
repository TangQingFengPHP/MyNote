# 别只会 malloc：Zig Allocator、所有权与内存生命周期实战

在 Zig 里，动态内存不是一句 `malloc` 就结束了。内存从哪里来、由谁释放、什么时候失效，都需要在代码里说清楚。

Allocator 就是这套规则的入口。它不是一块内存，而是一组“申请、调整、释放内存”的操作，以及这些操作背后的上下文。换一个 Allocator，同一段业务代码就可以使用堆内存、栈上的固定缓冲区、Arena，甚至自定义内存池。

本文使用 Zig 0.16 语法，示例都围绕可运行代码展开。Zig 0.16 已将旧版本常见的 GeneralPurposeAllocator 更名为 DebugAllocator，文中的通用堆示例统一使用当前名称。

## 1. 为什么需要 Allocator

数组长度在编译期确定时，不需要动态分配：

~~~zig
var scores: [3]u32 = .{ 80, 90, 100 };
~~~

如果元素个数运行时才知道，就需要一块可以动态申请和扩容的内存：

~~~zig
fn makeBytes(allocator: std.mem.Allocator, count: usize) ![]u8 {
    return try allocator.alloc(u8, count);
}
~~~

函数没有偷偷使用某个全局堆，而是明确接收一个 Allocator。调用方可以决定内存策略：测试时使用 `std.testing.allocator`，普通程序使用 GPA，短生命周期数据使用 Arena，资源极少的场景使用固定缓冲区。

这种写法还有一个直接好处：内存依赖可以沿着函数参数传递，测试和替换都比较容易。

## 2. `std.mem.Allocator` 到底是什么

最常见的类型写法是：

~~~zig
const allocator: std.mem.Allocator = ...;
~~~

它代表一套统一的内存操作接口。接口背后可以连接不同实现，但业务代码只需要调用统一方法：

| 方法 | 返回值 | 典型用途 |
| --- | --- | --- |
| `alloc(T, n)` | `![]T` | 申请 n 个 T |
| `create(T)` | `!*T` | 申请一个 T |
| `dupe(T, slice)` | `![]T` | 复制一份切片 |
| `realloc(slice, n)` | `![]T` | 调整切片容量 |
| `free(slice)` | `void` | 释放 alloc、dupe 或 realloc 得到的内存 |
| `destroy(ptr)` | `void` | 释放 create 得到的对象 |

Allocator 的错误类型通常只有 `error.OutOfMemory`。这不是异常，而是普通的 Zig error union，可以用 `try` 向上传递。

## 3. `alloc` 与 `free`：申请一段连续内存

`alloc(u8, 16)` 表示申请 16 个 `u8`，结果是一个长度为 16 的切片：

~~~zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const buffer = try allocator.alloc(u8, 16);
    defer allocator.free(buffer);

    @memset(buffer, 0);
    buffer[0] = 'Z';
    std.debug.print("first = {c}, length = {}\n", .{ buffer[0], buffer.len });
}
~~~

`alloc` 只负责拿到内存，不负责初始化内容。上面的 `@memset` 把内容清零；如果后续代码会完整覆盖所有元素，也可以直接写入。

`free` 必须使用与申请时兼容的 Allocator，并且通常应该紧跟在申请之后用 `defer` 注册。这样即使中途 `return` 或 `try` 失败，也不会忘记释放。

## 4. `create` 与 `destroy`：申请一个对象

数组适合存放一批元素，单个结构体则可以使用 `create`：

~~~zig
const Point = struct {
    x: i32,
    y: i32,
};

fn makePoint(allocator: std.mem.Allocator, x: i32, y: i32) !*Point {
    const point = try allocator.create(Point);
    point.* = .{ .x = x, .y = y };
    return point;
}

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}){};
    defer _ = gpa.deinit();

    const point = try makePoint(gpa.allocator(), 3, 4);
    defer gpa.allocator().destroy(point);
    std.debug.print("({}, {})\n", .{ point.x, point.y });
}
~~~

`create(Point)` 返回 `*Point`，也就是指向一个 Point 的指针；`destroy` 是它的配对释放操作。结构体字段通过 `point.*` 写入，因为 `point` 本身是指针。

## 5. `dupe`：复制切片并获得所有权

切片只是一对“指针加长度”，不自动拥有底层内存。需要长期保存一份字符串时，应复制内容：

~~~zig
fn copyName(allocator: std.mem.Allocator, source: []const u8) ![]u8 {
    return try allocator.dupe(u8, source);
}

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const name = try copyName(allocator, "zig");
    defer allocator.free(name);
    std.debug.print("name = {s}\n", .{name});
}
~~~

返回的 `name` 是一份新内存，和原来的字符串没有共享存储。`dupe` 常用于把函数参数、临时缓冲区或外部数据保存到结构体里。

如果 C API 要求以 `\\0` 结尾，可以使用 `dupeZ`，其结果是带 sentinel 的切片：

~~~zig
const c_name = try allocator.dupeZ(u8, "zig");
defer allocator.free(c_name);
~~~

## 6. `realloc`：扩容时切片可能搬家

动态数组扩容时，原地址不一定还能继续使用。Allocator 可能原地扩大，也可能申请新区域、复制旧数据、释放旧区域。因此必须接住返回值：

~~~zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var text = try allocator.alloc(u8, 4);
    defer allocator.free(text);
    @memcpy(text, "Zig!");

    text = try allocator.realloc(text, 8);
    @memcpy(text[4..], "lang");
    std.debug.print("{s}\n", .{text});
}
~~~

关键点是 `text = try allocator.realloc(text, 8)`。如果只调用 `realloc` 而不保存结果，扩容后的地址可能丢失；如果先手动 `free(text)`，再继续使用旧切片，则会产生释放后使用。

## 7. 用 `defer` 和 `errdefer` 管好生命周期

`defer` 在当前作用域退出时执行，适合正常路径和错误路径都需要执行的清理：

~~~zig
const data = try allocator.alloc(u8, 32);
defer allocator.free(data);
~~~

`errdefer` 只在函数返回错误时执行，适合“函数成功返回后，所有权已经交给调用方”的场景：

~~~zig
const Pair = struct {
    left: []u8,
    right: []u8,

    fn deinit(self: Pair, allocator: std.mem.Allocator) void {
        allocator.free(self.left);
        allocator.free(self.right);
    }
};

fn makePair(allocator: std.mem.Allocator) !Pair {
    const left = try allocator.alloc(u8, 8);
    errdefer allocator.free(left);

    const right = try allocator.alloc(u8, 8);
    errdefer allocator.free(right);

    return .{ .left = left, .right = right };
}
~~~

两次申请都成功时，`return` 会把两段内存的所有权交给 Pair，`errdefer` 不执行。第二次申请失败时，第一段内存会被回滚，避免部分初始化造成泄漏。

## 8. 一个完整的拥有内存的类型

把 Allocator 和数据放进结构体，是封装所有权的常见方式：

~~~zig
const Message = struct {
    allocator: std.mem.Allocator,
    text: []u8,

    fn init(allocator: std.mem.Allocator, source: []const u8) !Message {
        const text = try allocator.dupe(u8, source);
        return .{ .allocator = allocator, .text = text };
    }

    fn deinit(self: *Message) void {
        self.allocator.free(self.text);
        self.* = undefined;
    }
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var message = try Message.init(allocator, "hello allocator");
    defer message.deinit();
    std.debug.print("{s}\n", .{message.text});
}
~~~

这个模式把规则写在类型附近：`Message.init` 申请内存，`Message.deinit` 释放内存。调用方只需要保证每个成功初始化的 Message 最终调用一次 `deinit`。

## 9. DebugAllocator：通用堆分配器

DebugAllocator 适合普通程序中的长期对象、动态数组和复杂数据结构：

~~~zig
var gpa = std.heap.DebugAllocator(.{}){};
defer {
    const status = gpa.deinit();
    if (status == .leak) @panic("memory leak detected");
}
const allocator = gpa.allocator();
~~~

Debug 构建下，DebugAllocator 可以帮助发现泄漏、重复释放和部分内存使用错误。生产构建是否保留同样的检查，取决于 Zig 版本和配置，不能把调试检查当成业务逻辑的一部分。旧版 Zig 中，这个角色通常由 GeneralPurposeAllocator 承担。

## 10. ArenaAllocator：批量释放

一批对象拥有相同生命周期时，逐个释放很麻烦。Arena 把多次申请集中管理，最后一次 `deinit` 全部释放：

~~~zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}){};
    defer _ = gpa.deinit();

    var arena = std.heap.ArenaAllocator.init(gpa.allocator());
    defer arena.deinit();
    const allocator = arena.allocator();

    const first = try allocator.dupe(u8, "first");
    const second = try allocator.dupe(u8, "second");
    std.debug.print("{s}, {s}\n", .{ first, second });
}
~~~

Arena 很适合解析一份配置、构建一棵语法树、处理一次请求等场景。Arena 中的内存不应在 `arena.deinit()` 之后继续使用。单独调用 `free` 通常不会带来有意义的回收，重点是批量结束生命周期。

## 11. FixedBufferAllocator：不走堆的固定缓冲区

固定缓冲区把存储空间放在调用方提供的数组中：

~~~zig
const std = @import("std");

pub fn main() !void {
    var storage: [64]u8 = undefined;
    var fixed = std.heap.FixedBufferAllocator.init(&storage);
    const allocator = fixed.allocator();

    const text = try allocator.dupe(u8, "stored in a fixed buffer");
    std.debug.print("{s}\n", .{text});
}
~~~

申请失败时返回 `error.OutOfMemory`。这类分配器适合协议包、嵌入式程序、临时格式化缓冲区和需要明确内存上限的代码。返回的数据不能超过 `storage` 的生命周期，也不能在 `storage` 离开作用域后继续使用。

## 12. 其他常见分配器

`std.heap.page_allocator` 直接以页为单位向操作系统申请内存，适合少量、大块、生命周期较长的分配；小对象频繁申请通常不如 GPA 合适。

`std.heap.c_allocator` 使用 C 的 malloc/free，主要用于和 C 库或已有 C 接口对接。使用前需要考虑链接方式、跨边界释放规则和对齐要求。

`std.testing.allocator` 只适合测试。它会记录分配情况，测试结束时可以发现泄漏：

~~~zig
test "owned copy" {
    const copy = try std.testing.allocator.dupe(u8, "zig");
    defer std.testing.allocator.free(copy);
}
~~~

## 13. Allocator 与 `std.ArrayList`

ArrayList 是“可增长的连续数组”，底层扩容需要 Allocator。在 Zig 0.16 中，未托管形式的 ArrayList 不保存 Allocator，因此调用扩容和释放方法时显式传入：

~~~zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var numbers: std.ArrayList(i32) = .empty;
    defer numbers.deinit(allocator);

    try numbers.append(allocator, 10);
    try numbers.append(allocator, 20);
    try numbers.append(allocator, 30);

    for (numbers.items) |number| {
        std.debug.print("{} ", .{number});
    }
    std.debug.print("\n", .{});
}
~~~

`numbers.items` 是当前有效元素的切片。扩容可能搬动底层数组，所以不要长期保存元素指针；追加操作后，之前保存的指针可能已经失效。

## 14. `[]const T` 是切片，不等于“传入一个指针”

`[]const Command` 表示一段连续的 Command 集合，包含地址和长度：

~~~zig
fn runAll(commands: []const Command) void {
    for (commands) |command| {
        _ = command;
    }
}
~~~

它确实携带指向首元素的地址，但不只是一根指针；长度信息也在其中。`const` 表示不能通过这份切片修改元素，不表示底层内存一定拥有，也不表示内存一定来自 Allocator。

例如，字符串字面量、栈数组、堆分配结果都可以转换成 `[]const u8`。是否需要释放，取决于底层内存的所有权，而不是切片类型本身。

## 15. 一个动态文本缓冲区 Demo

下面的例子实现一个最小的可增长文本缓冲区，展示申请、扩容、复制和释放的完整流程：

~~~zig
const std = @import("std");

const TextBuffer = struct {
    allocator: std.mem.Allocator,
    data: []u8,
    len: usize = 0,

    fn init(allocator: std.mem.Allocator, capacity: usize) !TextBuffer {
        return .{
            .allocator = allocator,
            .data = try allocator.alloc(u8, capacity),
        };
    }

    fn deinit(self: *TextBuffer) void {
        self.allocator.free(self.data);
        self.* = undefined;
    }

    fn append(self: *TextBuffer, text: []const u8) !void {
        const required = self.len + text.len;
        if (required > self.data.len) {
            var capacity = self.data.len * 2;
            if (capacity < required) capacity = required;
            self.data = try self.allocator.realloc(self.data, capacity);
        }
        @memcpy(self.data[self.len..][0..text.len], text);
        self.len = required;
    }

    fn slice(self: *const TextBuffer) []const u8 {
        return self.data[0..self.len];
    }
};

pub fn main() !void {
    var gpa = std.heap.DebugAllocator(.{}){};
    defer _ = gpa.deinit();

    var buffer = try TextBuffer.init(gpa.allocator(), 4);
    defer buffer.deinit();
    try buffer.append("Zig ");
    try buffer.append("Allocator");
    std.debug.print("{s}\n", .{buffer.slice()});
}
~~~

这里的 `TextBuffer` 拥有 `data`。`append` 可能改变 `data` 的地址，但外部只通过 `slice()` 读取当前内容；`deinit` 统一回收底层存储。

## 16. 选择分配器的简单规则

| 场景 | 优先考虑 |
| --- | --- |
| 普通程序、对象生命周期复杂 | `DebugAllocator`（旧版为 `GeneralPurposeAllocator`） |
| 一批数据一起结束 | `ArenaAllocator` |
| 内存上限明确、不想使用堆 | `FixedBufferAllocator` |
| 测试中的分配检查 | `std.testing.allocator` |
| 和 C 库交互 | `c_allocator` |
| 少量大块、页级管理 | `page_allocator` |

分配器的选择，本质上是在选择生命周期、性能、内存上限和诊断能力。业务函数通常只接收 `std.mem.Allocator`，不要把某种具体分配器写死在每个函数内部。

## 17. 最容易踩的坑

1. 申请后没有释放。使用 `defer allocator.free(...)` 或 `defer allocator.destroy(...)` 注册清理。
2. 把借用切片当成拥有内存的切片。需要保存时使用 `dupe` 复制。
3. `realloc` 后继续使用旧切片或旧指针。扩容后重新获取地址。
4. `errdefer` 注册得太晚。每次成功申请后，应该立即注册对应回滚动作。
5. 在 Arena 释放后继续访问其中的数据。
6. 把固定缓冲区返回到更长生命周期。返回值不能超过 backing buffer 的生命周期。
7. 用错误的 Allocator 释放内存。申请和释放必须遵守同一套分配器契约。
8. 依赖 GPA 的调试检查代替正确的所有权设计。检测工具只能帮助定位问题，不能修复生命周期混乱。

## 总结

Allocator 解决的不是“怎样调用 malloc”，而是“内存由谁提供、怎样扩容、谁拥有、何时释放”。

记住几组配对关系即可建立基本框架：

- `alloc` 对应 `free`
- `create` 对应 `destroy`
- `dupe` 对应 `free`
- `realloc` 返回的新切片必须接住
- `ArenaAllocator` 的资源通常在 `deinit` 时批量释放
- `[]const T` 只描述访问方式，不代表所有权

当所有权和生命周期被写进类型、函数参数与 `defer` 中，Zig 的手动内存管理就不再是到处寻找 `free`，而是一套可以检查、测试和替换的明确规则。
