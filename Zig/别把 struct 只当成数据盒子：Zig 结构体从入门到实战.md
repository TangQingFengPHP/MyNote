# 别把 struct 只当成数据盒子：Zig 结构体从入门到实战

在 Zig 里，`struct` 不只是把几个变量放在一起的“数据盒子”。字段、默认值、方法、嵌套类型、泛型、错误处理和内存布局，都可以围绕一个结构体组织起来。

这也是 Zig 代码经常把业务对象写成 `struct` 的原因：数据放在字段里，操作数据的函数放在类型内部，内存由调用方明确管理，最终形成一组边界清楚的代码。

本文按 Zig 0.16.0 语法编写，从最基础的初始化开始，逐步讲到方法、指针、组合、泛型、`extern struct`、`packed struct` 和堆内存，并提供可以直接运行的 demo。

## 一、先看一个最小结构体

```zig
const std = @import("std");

const Person = struct {
    name: []const u8,
    age: u8,
};

pub fn main() void {
    const person = Person{
        .name = "Alice",
        .age = 30,
    };

    std.debug.print("{s}，{} 岁\n", .{ person.name, person.age });
}
```

`Person` 是一个类型，`person` 是这个类型的一个值。字段使用 `.字段名 = 值` 初始化，访问字段直接使用点号：

```zig
person.name
person.age
```

结构体实例默认是值。用 `const` 声明后不能修改字段，用 `var` 声明后才可以修改：

```zig
var person = Person{ .name = "Alice", .age = 30 };
person.age += 1;
```

如果把 `var` 改成 `const`，`person.age += 1` 会在编译阶段报错。这个规则和指针文章中的可变性规则是一致的：修改数据之前，数据本身必须处于可变存储中。

## 二、初始化：完整写法、类型推导与默认值

### 1. 完整初始化

没有默认值的字段必须全部提供：

```zig
const User = struct {
    id: u64,
    username: []const u8,
    enabled: bool,
};

const user = User{
    .id = 1,
    .username = "admin",
    .enabled = true,
};
```

结构体字面量前的类型名可以省略，让左侧类型负责推导：

```zig
const user: User = .{
    .id = 2,
    .username = "guest",
    .enabled = false,
};
```

`.{ ... }` 不是一个没有类型的神秘对象，它必须依赖上下文推导出具体类型。下面这种写法就不完整：

```zig
// const user = .{ .id = 1, .username = "admin" }; // 缺少目标类型
```

### 2. 字段默认值

字段可以写默认值，初始化时就能省略这些字段：

```zig
const ServerConfig = struct {
    host: []const u8 = "127.0.0.1",
    port: u16 = 8080,
    log_enabled: bool = true,
};

const local = ServerConfig{};
const test_server = ServerConfig{
    .port = 9000,
    .log_enabled = false,
};
```

`local` 使用全部默认值，`test_server` 只覆盖端口和日志开关，`host` 仍然是 `127.0.0.1`。

默认值适合表达“省略字段也一定合理”的情况。例如日志开关默认为 `true` 没有问题；但如果结构体要求 `minimum <= maximum`，直接给两个字段设置默认值可能掩盖无效状态。数据有复杂约束时，更适合提供 `init` 方法统一校验。

### 3. 用 `init` 表达初始化规则

Zig 没有内置构造函数，也没有 `new` 关键字。通常把普通函数放在结构体内部，并命名为 `init`：

```zig
const User = struct {
    name: []const u8,
    age: u8,

    pub fn init(name: []const u8) User {
        return .{
            .name = name,
            .age = 0,
        };
    }
};

const user = User.init("Alice");
```

`init` 只是 Zig 社区常用的命名约定，并没有特殊语法含义。返回值是一个普通结构体值，是否放到堆上由调用方决定。

## 三、结构体方法：把操作放回数据旁边

结构体内部可以定义函数。第一个参数通常叫 `self`，它不是关键字，只是一个约定名称。

```zig
const Rectangle = struct {
    width: u32,
    height: u32,

    fn area(self: Rectangle) u32 {
        return self.width * self.height;
    }
};

const rect = Rectangle{ .width = 10, .height = 5 };
const result = rect.area();
```

调用 `rect.area()` 时，实际上就是把 `rect` 作为第一个参数传给 `Rectangle.area`。

### `self` 的三种常见写法

#### `self: Rectangle`：按值传递

```zig
fn renamed(self: Person) Person {
    return .{ .name = "Bob", .age = self.age };
}
```

函数拿到的是一份值。字段不能直接修改；即使先复制到 `var`，修改的也只是副本。小型、只读结构体可以采用这种写法。

#### `self: *Rectangle`：修改原实例

```zig
const Rectangle = struct {
    width: u32,
    height: u32,

    fn scale(self: *Rectangle, factor: u32) void {
        self.width *= factor;
        self.height *= factor;
    }
};

var rect = Rectangle{ .width = 10, .height = 5 };
rect.scale(2);
```

`rect.scale(2)` 会自动把 `&rect` 传给方法。因为 `self` 是 `*Rectangle`，方法可以修改原来的 `rect`。

通过结构体指针访问字段时，Zig 会自动解引用一层，因此可以写 `self.width`，不必写成 `self.*.width`。两种写法表达的是同一件事。

#### `self: *const Rectangle`：只读且避免复制

```zig
fn printArea(self: *const Rectangle) void {
    std.debug.print("area={}\n", .{self.width * self.height});
}
```

这种写法不会复制整个结构体，也不允许修改字段，适合大型结构体的只读方法。

## 四、一个完整 demo：带校验、转账和错误处理的账户模型

下面这个例子把字段默认值、`init`、指针方法、只读方法、枚举和错误联合放在一起。保存为 `account.zig` 后，可以执行：

```bash
zig run account.zig
```

```zig
const std = @import("std");

const AccountKind = enum {
    checking,
    savings,
};

const Account = struct {
    id: u32,
    owner: []const u8,
    balance: i64 = 0,
    kind: AccountKind = .checking,

    pub fn init(id: u32, owner: []const u8) Account {
        return .{
            .id = id,
            .owner = owner,
        };
    }

    pub fn deposit(self: *Account, amount: i64) void {
        self.balance += amount;
    }

    pub fn withdraw(self: *Account, amount: i64) !void {
        if (amount < 0) return error.InvalidAmount;
        if (self.balance < amount) return error.InsufficientFunds;
        self.balance -= amount;
    }

    pub fn transfer(self: *Account, target: *Account, amount: i64) !void {
        try self.withdraw(amount);
        target.deposit(amount);
    }

    pub fn show(self: *const Account) void {
        const kind_name = switch (self.kind) {
            .checking => "活期",
            .savings => "储蓄",
        };

        std.debug.print(
            "账户 {} | {s} | {s} | 余额 {}\n",
            .{ self.id, self.owner, kind_name, self.balance },
        );
    }
};

pub fn main() !void {
    var alice = Account.init(1001, "Alice");
    var bob = Account{
        .id = 1002,
        .owner = "Bob",
        .kind = .savings,
    };

    alice.deposit(1000);
    try alice.transfer(&bob, 300);

    alice.show();
    bob.show();

    if (alice.withdraw(10_000)) |_| {
        unreachable;
    } else |err| {
        std.debug.print("取款失败：{}\n", .{err});
    }
}
```

运行结果类似：

```text
账户 1001 | Alice | 活期 | 余额 700
账户 1002 | Bob | 储蓄 | 余额 300
取款失败：error.InsufficientFunds
```

这个例子里，`Account` 没有隐藏状态，也没有隐式分配内存。余额变化必须经过 `deposit`、`withdraw` 或 `transfer`，业务规则集中在结构体方法中，调用处只负责组织流程。

### `enum` 和 `error` 从哪里来

账户类型由这段代码定义：

```zig
const AccountKind = enum {
    checking,
    savings,
};
```

因此，下面两种写法才是有效值：

```zig
const a = Account{ .id = 1, .owner = "A", .kind = .checking };
const b = Account{ .id = 2, .owner = "B", .kind = .savings };
```

如果枚举只写成：

```zig
const AccountKind = enum { checking };
```

再使用 `.savings` 就会出现 `enum ... has no member named 'savings'`。修复方式是在枚举中补上 `savings`，或者把使用处统一改为已经声明的成员名称。

`error.InvalidAmount` 和 `error.InsufficientFunds` 不是标准库里预先存在的变量，而是 Zig 的错误值。原 demo 使用了错误集合自动推导：

```zig
pub fn withdraw(self: *Account, amount: i64) !void {
    if (amount < 0) return error.InvalidAmount;
    if (self.balance < amount) return error.InsufficientFunds;
    self.balance -= amount;
}
```

返回类型中的 `!void` 表示“成功时返回 `void`，失败时返回一个错误”。函数体里出现的 `error.InvalidAmount` 和 `error.InsufficientFunds` 会被 Zig 收集到该函数的错误集合中，因此不需要另外声明。

也可以显式声明错误集合，让错误名称集中展示：

```zig
const AccountError = error{
    InvalidAmount,
    InsufficientFunds,
};

fn withdraw(self: *Account, amount: i64) AccountError!void {
    if (amount < 0) return error.InvalidAmount;
    if (self.balance < amount) return error.InsufficientFunds;
    self.balance -= amount;
}
```

两种写法的运行效果相同。小 demo 使用自动推导更简洁，公共模块或错误类型较多的项目通常会显式声明错误集合。

## 五、结构体是值类型：什么时候传值，什么时候传指针

函数参数默认按值传递：

```zig
const Counter = struct {
    value: u32,
};

fn changeCopy(counter: Counter) void {
    var copy = counter;
    copy.value = 100;
}

fn changeOriginal(counter: *Counter) void {
    counter.value = 100;
}

var counter = Counter{ .value = 1 };
changeCopy(counter);
// counter.value 仍然是 1

changeOriginal(&counter);
// counter.value 变成 100
```

选型可以简单归纳为：

- 只读且结构体很小：传 `T`。
- 只读但结构体较大：传 `*const T`。
- 方法需要修改实例：传 `*T`。
- 需要表达“可能不存在”：传 `?*T` 或返回 `?T`，根据所有权和数据大小决定。

传指针不会自动延长对象生命周期。指针指向的结构体必须在使用期间一直有效，不能返回局部变量的地址：

```zig
// 错误示意：函数返回后 value 的生命周期已经结束
// fn bad() *i32 {
//     var value: i32 = 10;
//     return &value;
// }
```

## 六、嵌套结构体：用组合组织复杂数据

结构体可以包含另一个结构体，初始化时可以使用类型名，也可以使用 `.{}` 简写：

```zig
const Address = struct {
    city: []const u8,
    zip: u32,
};

const Profile = struct {
    name: []const u8,
    address: Address,
};

const profile = Profile{
    .name = "Alice",
    .address = .{
        .city = "Shanghai",
        .zip = 200000,
    },
};
```

这种组合方式比把所有字段平铺在一个大结构体里更容易维护。地址有自己的字段和方法，用户资料只负责组合它。

结构体内部也可以定义类型：

```zig
const HttpRequest = struct {
    const Headers = struct {
        content_type: []const u8 = "text/plain",
    };

    method: []const u8,
    headers: Headers = .{},
};

const request = HttpRequest{ .method = "GET" };
```

`HttpRequest.Headers` 是嵌套类型的访问方式。类型和数据放在一起，适合表达只服务于某个外部结构体的辅助模型。

## 七、自引用结构体：链表为什么需要 `?*Node`

结构体不能直接包含自身：

```zig
// 错误：Node 里面再放一个 Node，会导致类型无限展开
// const Node = struct { next: Node };
```

但可以包含指向自身的指针，因为指针大小是固定的：

```zig
const std = @import("std");

const Node = struct {
    value: i32,
    next: ?*Node = null,
};

pub fn main() void {
    var third = Node{ .value = 30 };
    var second = Node{ .value = 20, .next = &third };
    var first = Node{ .value = 10, .next = &second };

    var current: ?*Node = &first;
    while (current) |node| {
        std.debug.print("{} -> ", .{node.value});
        current = node.next;
    }
    std.debug.print("null\n", .{});
}
```

输出：

```text
10 -> 20 -> 30 -> null
```

`next` 使用 `?*Node`，因为最后一个节点没有下一个节点。这里的节点都在 `main` 的作用域中，遍历期间一直有效；如果节点由 allocator 创建，则还需要在释放头节点前逐个释放后续节点。

## 八、`@This()`：泛型结构体里引用当前类型

匿名结构体没有一个可以直接书写的类型名，`@This()` 可以得到当前结构体类型：

```zig
fn Box(comptime T: type) type {
    return struct {
        const Self = @This();

        value: T,

        fn get(self: *const Self) T {
            return self.value;
        }
    };
}

const IntBox = Box(i32);
const box = IntBox{ .value = 42 };
```

`Self` 只是一个类型别名，方便方法参数书写。泛型结构体的关键不在于特殊的 `class` 语法，而在于 comptime 函数根据类型参数返回一个新的具体结构体类型。

## 九、泛型结构体 demo：固定容量栈

下面实现一个编译期确定元素类型和容量的栈。它不需要 allocator，也不会发生运行时扩容：

```zig
const std = @import("std");

fn Stack(comptime T: type, comptime capacity: usize) type {
    return struct {
        const Self = @This();

        items: [capacity]T = undefined,
        len: usize = 0,

        fn push(self: *Self, value: T) !void {
            if (self.len == capacity) return error.StackFull;
            self.items[self.len] = value;
            self.len += 1;
        }

        fn pop(self: *Self) ?T {
            if (self.len == 0) return null;
            self.len -= 1;
            return self.items[self.len];
        }
    };
}

pub fn main() !void {
    const IntStack = Stack(i32, 3);
    var stack = IntStack{};

    try stack.push(10);
    try stack.push(20);
    try stack.push(30);

    while (stack.pop()) |value| {
        std.debug.print("{} ", .{value});
    }
    std.debug.print("\n", .{});
}
```

输出为 `30 20 10`。`Stack(i32, 3)` 在编译期生成一个具体类型，`items` 的类型就是 `[3]i32`。如果改成 `Stack([]const u8, 8)`，同一份结构体逻辑就能保存字符串切片。

## 十、堆上创建结构体：`create` 对应 `destroy`

局部变量适合短生命周期对象，需要动态创建时可以使用 allocator：

```zig
const std = @import("std");

const Point = struct {
    x: i32,
    y: i32,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    const allocator = gpa.allocator();
    const point = try allocator.create(Point);
    defer allocator.destroy(point);

    point.* = .{ .x = 10, .y = 20 };
    point.x += 5;

    std.debug.print("({}, {})\n", .{ point.x, point.y });
}
```

`allocator.create(Point)` 返回 `*Point`，分配的是一个 `Point` 对象；`allocator.destroy(point)` 释放这个对象。`defer` 可以保证离开 `main` 时执行释放，即使中途发生错误也不会漏掉这一步。

如果结构体内部还持有 `allocator.dupe`、`alloc` 等得到的动态内存，通常需要额外提供 `deinit` 方法，先释放字段，再释放结构体本身：

```zig
const Document = struct {
    allocator: std.mem.Allocator,
    title: []u8,

    fn deinit(self: *Document) void {
        self.allocator.free(self.title);
    }
};
```

结构体不自动拥有字段，也不会自动调用析构逻辑。资源的创建和释放必须通过代码明确表达。

## 十一、普通 struct、`extern struct` 与 `packed struct`

这三种写法都叫结构体，但解决的问题不同。

### 1. 普通 `struct`

```zig
const Message = struct {
    code: u16,
    length: usize,
};
```

这是业务代码的默认选择。字段可以有方法、默认值和嵌套类型，但不要把字段顺序或填充字节当成 ABI 契约。普通结构体的布局由 Zig 编译器管理，不保证和 C 结构体一致。

### 2. `extern struct`：匹配 C ABI

需要和 C 函数、C 动态库或操作系统 ABI 交互时，使用 `extern struct`：

```zig
const CPoint = extern struct {
    x: i32,
    y: i32,
};
```

`extern struct` 的内存布局匹配目标平台的 C ABI。它不是“更快的普通 struct”，也不是所有结构体都应该使用的写法；只有布局必须和外部 ABI 对齐时才需要它。

### 3. `packed struct`：按位表达字段

标志位、协议头和硬件寄存器经常需要按 bit 保存：

```zig
const Flags = packed struct(u8) {
    enabled: bool,
    readonly: bool,
    level: u3,
    reserved: u3,
};

pub fn main() void {
    const flags = Flags{
        .enabled = true,
        .readonly = false,
        .level = 5,
        .reserved = 0,
    };

    const raw: u8 = @bitCast(flags);
    std.debug.print("0x{X}\n", .{raw});
}
```

字段总宽度是 `1 + 1 + 3 + 3 = 8` 位，正好对应 `u8`。`packed struct` 的布局有明确规则，但字段可能不是字节对齐的，访问复杂字段时需要更谨慎。它适合位级协议和硬件寄存器，不适合替代普通业务结构体。

Zig 0.16.0 支持像上面这样显式写出 `packed struct(u8)`。显式 backing integer 能让位宽和外部表示更加清楚，尤其是在导出或 ABI 场景中。

## 十二、大小、对齐和布局检查

`@sizeOf(T)` 可以得到类型大小，`@alignOf(T)` 可以得到类型对齐要求：

```zig
const std = @import("std");

const Normal = struct {
    a: u8,
    b: u64,
};

const Bits = packed struct(u8) {
    first: u4,
    second: u4,
};

pub fn main() void {
    std.debug.print("Normal size={} align={}\n", .{
        @sizeOf(Normal),
        @alignOf(Normal),
    });
    std.debug.print("Bits size={} align={}\n", .{
        @sizeOf(Bits),
        @alignOf(Bits),
    });
}
```

普通结构体可能因为对齐要求产生 padding，因此字段类型排列会影响大小，但普通布局不能拿来做跨语言协议约定。跨语言场景应使用 `extern struct`，位级场景应使用 `packed struct`。

## 十三、结构体与访问控制：`pub` 修饰的是声明

结构体内部除了字段，还可以放常量、类型和函数：

```zig
const Parser = struct {
    const version = "1.0";

    pub fn parse(input: []const u8) usize {
        return input.len;
    }
};

const size = Parser.parse("hello");
```

`pub` 主要用于让声明可以从其他文件模块访问。未使用 `pub` 的声明默认只在当前文件或当前容器可见。字段本身通常通过结构体实例访问，真正需要隐藏实现细节时，可以把内部状态放入私有声明、使用不透明类型，或通过模块 API 约束访问路径。

## 十四、常见误区

### 误区一：把 Zig struct 当成 C# class

结构体没有继承体系，也没有自动构造和析构。方法只是结构体内部的函数，资源释放也不会自动发生。组合、函数参数和显式 allocator 管理通常比模拟 class 层级更符合 Zig 的写法。

### 误区二：以为默认值等于构造函数校验

字段默认值只是在初始化时补字段，不会自动检查多个字段之间的关系。涉及不变量时，应使用 `init` 或返回错误的初始化函数：

```zig
const Range = struct {
    start: i32,
    end: i32,

    fn init(start: i32, end: i32) !Range {
        if (start > end) return error.InvalidRange;
        return .{ .start = start, .end = end };
    }
};
```

### 误区三：普通 struct 直接传给 C

普通 `struct` 的布局不是 C ABI 契约。需要 C 兼容时使用 `extern struct`，需要按 bit 解释时使用 `packed struct`。

### 误区四：拿着结构体指针忘记生命周期

`*T` 只是地址，不是所有权。栈变量离开作用域后，指向它的指针不能继续使用；allocator 分配的结构体必须调用对应的 `destroy`。结构体字段里的切片也可能指向外部存储，释放结构体不等于释放切片背后的数据。

## 十五、结构体选型口诀

- 一组业务字段：普通 `struct`。
- 字段之间需要规则：普通 `struct` + `init` + 方法。
- 方法要修改实例：`self: *Self`。
- 只读访问且不想复制大对象：`self: *const Self`。
- 可复用的数据结构：`fn Type(comptime T: type) type`。
- 和 C ABI 对接：`extern struct`。
- 标志位、寄存器、位级协议：`packed struct`。
- 动态生命周期：allocator `create` 对应 `destroy`，内部动态字段通过 `deinit` 释放。

理解 `struct` 的关键，不是记住多少 API，而是分清三层职责：字段描述数据，方法维护数据规则，调用方负责对象生命周期。三层边界明确后，Zig 里的结构体既可以是简单的配置模型，也可以成长为带错误处理、泛型能力和资源管理的完整模块。

## 参考资料

- [Zig 0.16.0 Language Reference：struct](https://ziglang.org/documentation/0.16.0/#struct)
- [Zig 0.16.0 Language Reference：Default Field Values](https://ziglang.org/documentation/0.16.0/#Default-Field-Values)
- [Zig 0.16.0 Language Reference：extern struct](https://ziglang.org/documentation/0.16.0/#extern-struct)
- [Zig 0.16.0 Language Reference：packed struct](https://ziglang.org/documentation/0.16.0/#packed-struct)
