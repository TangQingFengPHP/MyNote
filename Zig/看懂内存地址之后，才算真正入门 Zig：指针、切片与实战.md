# 看懂内存地址之后，才算真正入门 Zig：指针、切片与实战

很多 Zig 代码第一次看起来有点“绕”，问题往往不在语法，而在指针、数组和切片没有分清。

`*T`、`*[N]T`、`[*]T`、`[]T` 只差几个符号，代表的能力却完全不同：有的只能指向一个值，有的可以移动，有的带长度，有的允许为空。类型写得准确，很多内存错误在编译阶段就会暴露；类型选错，代码即使能编译，也可能留下越界访问或悬空指针。

这篇文章从一个最小例子开始，再逐步扩展到数组、切片、链表、C 字符串和堆内存。示例按 Zig 0.16.0 语法编写。

## 先记住这几种类型

| 写法 | 含义 | 能否为空 | 是否带长度 | 常见用途 |
| --- | --- | --- | --- | --- |
| `*T` | 指向一个 `T` 的单项指针 | 否 | 否 | 修改变量、传递大结构体 |
| `*const T` | 指向一个只读 `T` 的单项指针 | 否 | 否 | 只读访问 |
| `*[N]T` | 指向一个长度为 `N` 的数组 | 否 | 是，编译期已知 | 操作固定大小数组 |
| `[*]T` | 指向未知数量元素的多项指针 | 否 | 否 | 底层内存、C 接口 |
| `[]T` | 带长度的切片 | 否 | 是，运行时已知 | 缓冲区、字符串、集合 |
| `?*T` | 可选单项指针 | 是 | 否 | 链表、查找结果 |
| `[*:0]const u8` | 以 `0` 结尾的指针 | 否 | 由哨兵结尾 | C 风格字符串 |

日常业务代码通常优先选择 `*T`、`*const T` 和 `[]const T`。`[*]T` 没有长度，安全边界需要额外维护；`[*c]T` 是 C 兼容指针，主要留给 C 头文件和 FFI 使用。

## 取地址与解引用

取地址使用 `&`，解引用使用 `.*`。这两个操作是指针最核心的入口。

```zig
const std = @import("std");

pub fn main() void {
    var score: i32 = 80;
    const score_ptr = &score;

    std.debug.print("修改前：{}\n", .{score_ptr.*});
    score_ptr.* += 5;
    std.debug.print("修改后：{}\n", .{score});
}
```

输出：

```text
修改前：80
修改后：85
```

`score_ptr.*` 不是复制出来的一份新数据，它仍然指向 `score` 原来的存储位置，所以通过它赋值，原变量也会改变。

这里有一个容易忽略的区别：`const score_ptr = &score` 只表示“指针变量本身不能改指向别处”，并不表示指向的数据只读。因为 `score` 是 `var`，`score_ptr` 的类型仍然是 `*i32`。如果写成：

```zig
const score: i32 = 80;
const score_ptr = &score; // *const i32
```

得到的才是只读指针，`score_ptr.* = 85` 会编译失败。可变指针可以自动转换为只读指针，反方向不允许。

## `*T` 只指向一个值

`*T` 是最常用的单项指针。函数需要修改调用方的变量时，传入 `*T` 就很自然：

```zig
const std = @import("std");

const User = struct {
    name: []const u8,
    score: u32,
};

fn levelUp(user: *User) void {
    user.score += 10;
}

pub fn main() void {
    var user = User{ .name = "Ming", .score = 90 };
    levelUp(&user);
    std.debug.print("{s}: {}\n", .{ user.name, user.score });
}
```

`user.score` 是 `user.*.score` 的简写。Zig 允许通过指针直接访问结构体字段，因此不必每次都手写 `.*`。

传指针也不只是为了修改数据。结构体很大时，传 `*const BigStruct` 可以避免复制整个结构体；不过指针带来的是“借用一段时间”，并不会自动延长被指向对象的生命周期。

## 数组指针、数组和切片到底差在哪里

看下面这组声明：

```zig
var numbers = [_]i32{ 10, 20, 30, 40 };

const array: [4]i32 = numbers; // 数组本体，长度是类型的一部分
const array_ptr: *[4]i32 = &numbers; // 指向整个数组
const first_ptr: *i32 = &numbers[0]; // 只指向第一个元素
const slice: []i32 = numbers[1..3]; // 指向第 2、3 个元素，长度为 2
```

它们不是同一种东西：

- `[4]i32` 本身包含 4 个 `i32`。
- `*[4]i32` 指向一个完整的 4 元素数组，`array_ptr.len` 在编译期就是 `4`。
- `*i32` 只承诺指向一个元素，不能写 `first_ptr + 1`。
- `[]i32` 是切片，内部可以理解为“指针 + 长度”，适合把一段数组传给函数。

数组指针可以索引和切片：

```zig
var values = [_]u8{ 1, 2, 3, 4, 5 };
const p: *[5]u8 = &values;

std.debug.print("len={} third={}\n", .{ p.len, p[2] });
const middle = p[1..4]; // { 2, 3, 4 }
```

切片访问会携带长度，在启用运行时安全检查的构建模式下，`middle[3]` 会触发越界错误，而不是悄悄读到旁边的内存。字符串字面量本质上也可以当作 `[]const u8` 使用：

```zig
fn printName(name: []const u8) void {
    std.debug.print("姓名：{s}，长度：{}\n", .{ name, name.len });
}

printName("Zig");
```

函数参数一般写成 `[]const T`，表示“需要一段只读元素，不关心底层到底是数组、数组指针还是另一段切片”。这比直接接收 `[*]const T` 更稳妥，因为调用方不需要另传一个长度。

## `[*]T`：能移动，但不知道边界

多项指针 `[*]T` 类似 C 里的 `T *`：支持索引、切片和指针加减，但自身不记录元素数量。

```zig
const std = @import("std");

fn doubleAll(ptr: [*]u8, count: usize) void {
    var i: usize = 0;
    while (i < count) : (i += 1) {
        ptr[i] *= 2;
    }
}

pub fn main() void {
    var data = [_]u8{ 1, 2, 3, 4 };
    doubleAll(&data, data.len);

    for (data) |value| {
        std.debug.print("{} ", .{value});
    }
    std.debug.print("\n", .{});
}
```

这个函数同时接收指针和长度，长度一旦传错，`ptr[i]` 就可能越过数组边界。改成切片后，接口更完整：

```zig
fn doubleAllSafe(data: []u8) void {
    for (data) |*value| {
        value.* *= 2;
    }
}
```

因此可以把选择规则记成一句话：知道长度就传切片；只有在系统调用、硬件缓冲区或 C API 明确要求裸指针时，才使用 `[*]T`。

## `?*T`：把“可能没有”写进类型

普通的 `*T` 不能赋值为 `null`。可能不存在的指针必须写成 `?*T`，使用前先解包：

```zig
const std = @import("std");

const Node = struct {
    value: i32,
    next: ?*Node,
};

fn find(head: ?*Node, target: i32) ?*Node {
    var current = head;
    while (current) |node| {
        if (node.value == target) return node;
        current = node.next;
    }
    return null;
}

pub fn main() void {
    var third = Node{ .value = 30, .next = null };
    var second = Node{ .value = 20, .next = &third };
    var first = Node{ .value = 10, .next = &second };

    if (find(&first, 20)) |node| {
        std.debug.print("找到：{}\n", .{node.value});
    } else {
        std.debug.print("没有找到\n", .{});
    }
}
```

`while (current) |node|` 每次循环都会把非空指针解包成 `node: *Node`，循环体内可以安全访问 `node.value`。这正是 Zig 把空指针风险放到类型系统里的体现。

## 哨兵指针与 C 字符串

C 字符串不是“带长度的字符串”，而是一串字符后面跟着一个 `0`。Zig 用 `[*:0]const u8` 表示这种以 `0` 结尾的多项指针：

```zig
const std = @import("std");

pub fn main() void {
    const c_string: [*:0]const u8 = "hello";
    const text: [:0]const u8 = std.mem.span(c_string);

    std.debug.print("{s}, len={}，结尾={}\n", .{
        text,
        text.len,
        text[text.len],
    });
}
```

这里的 `[:0]const u8` 是带长度的哨兵切片，`text.len` 位置保证存在 `0`。普通 `[]const u8` 不保证末尾有 `0`，所以不能直接当作 C API 所需的字符串传入。反过来，也不能把任意一段字节强行标成哨兵切片；Zig 会检查哨兵位置是否真的匹配。

## 堆内存中的指针：真正需要小心的是生命周期

指针本身不会拥有数据。用 allocator 分配出来的内存，必须在不再使用时释放；释放后仍然保留的指针就是悬空指针。

下面的例子分配一个整数数组，修改其中一个元素，再释放内存：

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    const allocator = gpa.allocator();
    const numbers = try allocator.alloc(i32, 3);
    defer allocator.free(numbers);

    numbers[0] = 10;
    numbers[1] = 20;
    numbers[2] = 30;

    const item: *i32 = &numbers[1];
    item.* += 5;
    std.debug.print("{}\n", .{numbers[1]}); // 25
}
```

`numbers` 是 `[]i32`，不是 `*i32`。它带有长度，适合遍历；`&numbers[1]` 才是指向其中一个元素的 `*i32`。`allocator.free(numbers)` 执行后，`item` 不能再使用。

一个实用判断是：指针指向的对象由谁创建，就由谁负责保证它的生命周期。函数返回局部变量地址是错误的，因为函数返回后局部变量已经失效：

```zig
// 错误示意：返回后局部变量 value 不再存在
// fn bad() *i32 {
//     var value: i32 = 10;
//     return &value;
// }
```

## 常见错误，放在编译器报错之前理解

### 把 Zig 解引用写成 C 语法

```zig
const value = ptr.*; // Zig
// const value = *ptr; // C 写法，Zig 中不对
```

### 把单项指针当成数组指针

```zig
const first: *i32 = &numbers[0];
// first[1] // 错误：*i32 只指向一个值

const all: *[3]i32 = &numbers;
const second = all[1]; // 正确
```

### 把可选指针直接解引用

```zig
var maybe: ?*i32 = null;
// maybe.* = 1; // 错误，必须先判断是否为空

if (maybe) |ptr| {
    ptr.* = 1;
}
```

### 忘记切片和裸指针的职责不同

`[*]T` 只提供“从某个地址开始的一串元素”，并不知道这一串有多长；`[]T` 额外携带长度，因而可以做边界检查。接口能使用切片时，不要为了看起来更底层而改成裸指针。

## 一套简单的选型口诀

- 修改一个变量：`*T`
- 只读一个变量：`*const T`
- 处理一段数据：`[]T` 或 `[]const T`
- 固定长度数组且需要整体操作：`*[N]T`
- 需要指针移动、对接底层 API：`[*]T`
- 查找结果可能不存在：`?*T`
- 调用 C 字符串接口：`[*:0]const u8` 或 `[:0]const u8`

指针并不等于“直接操作内存”。在 Zig 里，指针类型还描述了元素数量、可变性、是否为空、是否有哨兵和是否带长度。把这些信息写进类型，代码会更容易读，编译器也能替开发者提前拦住一批危险操作。

## 参考资料

- [Zig 0.16.0 Language Reference：Pointers](https://ziglang.org/documentation/0.16.0/#Pointers)
- [Zig 0.16.0 Language Reference：Slices](https://ziglang.org/documentation/0.16.0/#Slices)
- [Zig 0.16.0 Language Reference：Optional Pointers](https://ziglang.org/documentation/0.16.0/#Optional-Pointers)
