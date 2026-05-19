### 简介

`Record` 是 TypeScript 内置的工具类型，用来描述一种很常见的数据结构：

```text
一组 key，对应一组 value。
```

例如订单状态对应中文文案：

```ts
const orderStatusText = {
  pending: "待支付",
  paid: "已支付",
  shipped: "已发货",
  finished: "已完成"
};
```

这种对象在前端项目里很常见：

* 状态码映射文案
* 角色映射权限
* 路由映射菜单
* 表单字段映射错误信息
* 接口列表转成 `id -> 数据` 的字典
* 主题名映射主题配置

`Record` 的作用就是给这种“键值映射对象”加上清晰的类型约束。

一句话概括：

```text
Record<K, T> 表示：对象的 key 是 K，value 是 T。
```

### 基本语法

```ts
Record<K, T>
```

含义：

* `K`：对象 key 的类型
* `T`：对象 value 的类型

例如：

```ts
type UserAgeMap = Record<string, number>;

const userAges: UserAgeMap = {
  Tom: 18,
  Jack: 20
};
```

这段代码表示：

```text
key 必须是 string
value 必须是 number
```

所以这样写会报错：

```ts
const userAges: Record<string, number> = {
  Tom: 18,
  Jack: "20"
  // Type 'string' is not assignable to type 'number'.
};
```

### Record 的源码定义

TypeScript 里的 `Record` 定义大概是这样：

```ts
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
```

拆开看：

```ts
K extends keyof any
```

`keyof any` 等价于：

```ts
string | number | symbol
```

也就是说，`Record` 的 key 只能是可以作为对象属性名的类型。

再看这一段：

```ts
[P in K]: T
```

这是 TypeScript 的映射类型。意思是：

```text
遍历 K 中的每一个 key，并把每个 key 的 value 都设置成 T。
```

例如：

```ts
type Role = "admin" | "editor" | "guest";

type RoleText = Record<Role, string>;
```

等价于：

```ts
type RoleText = {
  admin: string;
  editor: string;
  guest: string;
};
```

### 最基础用法

#### 字符串 key

```ts
type ScoreMap = Record<string, number>;

const scores: ScoreMap = {
  math: 95,
  english: 88,
  history: 76
};
```

适合任意字符串作为 key 的字典对象。

#### 数字 key

```ts
type HttpStatusText = Record<number, string>;

const httpStatusText: HttpStatusText = {
  200: "成功",
  400: "请求错误",
  401: "未登录",
  500: "服务器错误"
};
```

需要注意：JavaScript 对象里的数字 key，运行时会转成字符串。

```ts
const map: Record<number, string> = {
  1: "one"
};

console.log(map[1]);
console.log(map["1"]);
```

上面两种访问方式都能拿到值。类型层面写的是 `number`，运行时对象属性名实际还是字符串。

#### 联合类型 key

`Record` 最好用的场景，是配合字面量联合类型。

```ts
type OrderStatus = "pending" | "paid" | "shipped" | "finished";

const orderStatusText: Record<OrderStatus, string> = {
  pending: "待支付",
  paid: "已支付",
  shipped: "已发货",
  finished: "已完成"
};
```

这种写法的好处很明显：少一个 key 会报错，多一个 key 也会报错。

```ts
const orderStatusText: Record<OrderStatus, string> = {
  pending: "待支付",
  paid: "已支付",
  shipped: "已发货"
  // Property 'finished' is missing.
};
```

```ts
const orderStatusText: Record<OrderStatus, string> = {
  pending: "待支付",
  paid: "已支付",
  shipped: "已发货",
  finished: "已完成",
  closed: "已关闭"
  // Object literal may only specify known properties.
};
```

这就是 `Record` 的核心价值：把“应该有哪些 key”写进类型系统里。

### Record 和普通对象类型

普通对象类型可以这样写：

```ts
type RoleText = {
  admin: string;
  editor: string;
  guest: string;
};
```

`Record` 写法：

```ts
type Role = "admin" | "editor" | "guest";
type RoleText = Record<Role, string>;
```

如果 key 很少，普通对象类型没问题。如果 key 来自联合类型、枚举、接口字段、路由名、状态码，`Record` 会更适合。

例如：

```ts
type Permission = "read" | "create" | "update" | "delete";

type PermissionText = Record<Permission, string>;
```

后续只要新增权限：

```ts
type Permission = "read" | "create" | "update" | "delete" | "export";
```

对应的映射对象就会提示补充 `export`。

### Record 和索引签名

索引签名写法：

```ts
type UserMap = {
  [key: string]: number;
};
```

`Record` 写法：

```ts
type UserMap = Record<string, number>;
```

这两种写法在 `string -> number` 这种场景下很接近。

区别主要在“有限 key”上。

```ts
type Field = "username" | "password" | "code";

type FormValue = Record<Field, string>;
```

这表示对象必须包含：

```text
username
password
code
```

索引签名表达的是“只要是字符串 key，value 都是某个类型”：

```ts
type FormValue = {
  [key: string]: string;
};
```

这种写法不关心具体有哪些字段，所以无法强制 `username`、`password`、`code` 必须存在。

对比总结：

| 写法 | 适合场景 |
| --- | --- |
| `{ [key: string]: T }` | 任意字符串 key 的字典 |
| `Record<string, T>` | 任意字符串 key 的字典，写法更简洁 |
| `Record<"a" | "b", T>` | 固定 key 集合，要求 key 必须完整 |

### Record 和 interface 怎么选

`Record` 适合“同一种 value 类型的映射对象”。

```ts
type StatusText = Record<"pending" | "paid", string>;
```

`interface` 或普通 `type` 更适合“一个业务实体”。

```ts
interface User {
  id: number;
  name: string;
  age: number;
  enabled: boolean;
}
```

因为 `User` 的每个字段含义不同，类型也可能不同。强行用 `Record` 会很别扭。

不推荐：

```ts
type User = Record<"id" | "name" | "age" | "enabled", string | number | boolean>;
```

这种写法会让每个字段都变成 `string | number | boolean`，字段类型失去了精确性。

推荐：

```ts
interface User {
  id: number;
  name: string;
  age: number;
  enabled: boolean;
}
```

简单判断：

```text
固定业务实体，用 interface 或普通 type。
键值映射字典，用 Record。
```

### Record 和 Map 的区别

`Record` 是 TypeScript 类型工具，编译后不会出现在 JavaScript 代码里。

`Map` 是 JavaScript 运行时数据结构，代码运行时真实存在。

| 对比项 | Record | Map |
| --- | --- | --- |
| 类型 | TypeScript 类型 | JavaScript 数据结构 |
| 是否存在于运行时 | 不存在 | 存在 |
| key 类型 | `string`、`number`、`symbol` 及其联合类型 | 几乎任意值，包括对象 |
| 访问方式 | `obj[key]` | `map.get(key)` |
| 适合场景 | 配置、状态映射、接口数据字典 | 频繁增删、复杂 key、需要保持插入顺序 |

`Record` 示例：

```ts
type Theme = "light" | "dark";

const themeText: Record<Theme, string> = {
  light: "浅色",
  dark: "深色"
};
```

`Map` 示例：

```ts
const domCache = new Map<HTMLElement, string>();

const element = document.querySelector("#app");

if (element) {
  domCache.set(element, "root");
}
```

如果 key 是对象、DOM、函数，使用 `Map` 更合适。如果 key 是状态、字段名、枚举值，`Record` 更直接。

### 实战一：订单状态映射

状态码转文案是 `Record` 最常见的用法。

```ts
type OrderStatus = "pending" | "paid" | "shipped" | "finished" | "cancelled";

const orderStatusText: Record<OrderStatus, string> = {
  pending: "待支付",
  paid: "已支付",
  shipped: "已发货",
  finished: "已完成",
  cancelled: "已取消"
};

function getOrderStatusText(status: OrderStatus) {
  return orderStatusText[status];
}

console.log(getOrderStatusText("paid"));
```

如果状态不只需要文案，还需要颜色、图标、是否允许操作，可以把 value 改成对象。

```ts
type OrderStatus = "pending" | "paid" | "shipped" | "finished" | "cancelled";

type OrderStatusMeta = {
  text: string;
  color: "gray" | "blue" | "orange" | "green" | "red";
  canCancel: boolean;
};

const orderStatusMeta: Record<OrderStatus, OrderStatusMeta> = {
  pending: { text: "待支付", color: "orange", canCancel: true },
  paid: { text: "已支付", color: "blue", canCancel: true },
  shipped: { text: "已发货", color: "blue", canCancel: false },
  finished: { text: "已完成", color: "green", canCancel: false },
  cancelled: { text: "已取消", color: "red", canCancel: false }
};

function getOrderActions(status: OrderStatus) {
  const meta = orderStatusMeta[status];

  return {
    label: meta.text,
    showCancelButton: meta.canCancel
  };
}
```

这种写法比到处写 `if...else` 更集中，也更容易检查遗漏。

### 实战二：表单错误信息

表单字段固定时，`Record` 可以强制每个字段都有对应的错误信息。

```ts
type LoginField = "username" | "password" | "captcha";

type LoginForm = Record<LoginField, string>;
type LoginErrors = Record<LoginField, string | null>;

const form: LoginForm = {
  username: "",
  password: "",
  captcha: ""
};

const errors: LoginErrors = {
  username: null,
  password: null,
  captcha: null
};

function validateLoginForm(values: LoginForm): LoginErrors {
  return {
    username: values.username ? null : "请输入用户名",
    password: values.password.length >= 6 ? null : "密码至少 6 位",
    captcha: values.captcha ? null : "请输入验证码"
  };
}
```

如果后面新增字段：

```ts
type LoginField = "username" | "password" | "captcha" | "remember";
```

`LoginForm` 和 `LoginErrors` 相关对象都会提示补齐 `remember`。

### 实战三：接口列表转字典

接口经常返回数组：

```ts
type User = {
  id: number;
  name: string;
  role: "admin" | "editor" | "viewer";
};

const users: User[] = [
  { id: 1, name: "张三", role: "admin" },
  { id: 2, name: "李四", role: "editor" },
  { id: 3, name: "王五", role: "viewer" }
];
```

如果页面里经常按 `id` 查用户，可以转成字典：

```ts
const userMap: Record<number, User> = {};

for (const user of users) {
  userMap[user.id] = user;
}

console.log(userMap[1].name);
```

也可以封装成通用函数：

```ts
function listToRecord<T, K extends string | number | symbol>(
  list: T[],
  getKey: (item: T) => K
): Record<K, T> {
  const result = {} as Record<K, T>;

  for (const item of list) {
    result[getKey(item)] = item;
  }

  return result;
}

const userMap = listToRecord(users, user => user.id);
```

这类函数在列表页、详情页、树形数据处理里很常见。

### 实战四：角色权限表

权限系统里经常需要维护“角色能访问哪些菜单”。

```ts
type Role = "admin" | "editor" | "viewer";

type MenuKey =
  | "dashboard"
  | "user"
  | "article"
  | "setting";

const roleMenus: Record<Role, MenuKey[]> = {
  admin: ["dashboard", "user", "article", "setting"],
  editor: ["dashboard", "article"],
  viewer: ["dashboard"]
};

function canVisit(role: Role, menu: MenuKey) {
  return roleMenus[role].includes(menu);
}

console.log(canVisit("editor", "article"));
console.log(canVisit("viewer", "setting"));
```

如果需要细到按钮级权限，可以嵌套 `Record`。

```ts
type Role = "admin" | "editor" | "viewer";
type Action = "view" | "create" | "update" | "delete";

const roleActions: Record<Role, Record<Action, boolean>> = {
  admin: {
    view: true,
    create: true,
    update: true,
    delete: true
  },
  editor: {
    view: true,
    create: true,
    update: true,
    delete: false
  },
  viewer: {
    view: true,
    create: false,
    update: false,
    delete: false
  }
};

function hasAction(role: Role, action: Action) {
  return roleActions[role][action];
}
```

嵌套 `Record` 不要套太深。超过两层时，通常可以拆成更明确的类型。

```ts
type ActionPermission = Record<Action, boolean>;
type RoleActionMap = Record<Role, ActionPermission>;
```

这样读起来更轻松。

### 实战五：主题配置

前端主题配置很适合用 `Record`。

```ts
type ThemeName = "light" | "dark" | "system";

type ThemeConfig = {
  label: string;
  background: string;
  text: string;
  border: string;
};

const themeConfig: Record<ThemeName, ThemeConfig> = {
  light: {
    label: "浅色模式",
    background: "#ffffff",
    text: "#1f2329",
    border: "#e5e6eb"
  },
  dark: {
    label: "深色模式",
    background: "#141414",
    text: "#f5f5f5",
    border: "#303030"
  },
  system: {
    label: "跟随系统",
    background: "var(--page-bg)",
    text: "var(--text-color)",
    border: "var(--border-color)"
  }
};

function getThemeConfig(name: ThemeName) {
  return themeConfig[name];
}
```

好处是新增主题时，配置对象会强制补齐。

### 配合 Partial

`Record<K, T>` 默认要求所有 key 都存在。

```ts
type Field = "username" | "password" | "captcha";

const errors: Record<Field, string> = {
  username: "请输入用户名",
  password: "请输入密码"
  // Property 'captcha' is missing.
};
```

如果只想保存有错误的字段，可以配合 `Partial`。

```ts
type Field = "username" | "password" | "captcha";

type FormErrors = Partial<Record<Field, string>>;

const errors: FormErrors = {
  username: "请输入用户名"
};
```

`Partial<Record<Field, string>>` 等价于：

```ts
type FormErrors = {
  username?: string;
  password?: string;
  captcha?: string;
};
```

这种写法适合“不是每个字段都有值”的场景。

### 配合 Readonly

如果映射表不应该被修改，可以加 `Readonly`。

```ts
type Status = "enabled" | "disabled";

const statusText: Readonly<Record<Status, string>> = {
  enabled: "启用",
  disabled: "禁用"
};

// statusText.enabled = "已启用";
// Cannot assign to 'enabled' because it is a read-only property.
```

`Readonly<Record<K, T>>` 常用于常量配置、状态文案、权限规则。

### 配合 Pick 和 Omit

`Record` 可以和 `keyof`、`Pick`、`Omit` 一起使用。

```ts
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

type PublicUserKey = keyof Omit<User, "password">;

type UserColumnText = Record<PublicUserKey, string>;

const userColumnText: UserColumnText = {
  id: "用户 ID",
  name: "姓名",
  email: "邮箱"
};
```

这类写法适合表格列名、表单标签、导出字段名。

### 配合 satisfies

`satisfies` 可以校验对象符合 `Record`，同时避免把整个变量直接写死成 `Record<...>` 类型。

```ts
type Status = "pending" | "paid" | "cancelled";

const statusConfig = {
  pending: { text: "待支付", color: "orange" },
  paid: { text: "已支付", color: "green" },
  cancelled: { text: "已取消", color: "red" }
} as const satisfies Record<Status, { text: string; color: string }>;
```

和直接标注类型相比：

```ts
const statusConfig: Record<Status, { text: string; color: string }> = {
  pending: { text: "待支付", color: "orange" },
  paid: { text: "已支付", color: "green" },
  cancelled: { text: "已取消", color: "red" }
};
```

`satisfies` 更适合配置表。它能检查 key 是否完整；配合 `as const` 时，还能保留对象里的字面量信息。

### 配合 enum

如果项目里使用 `enum`，也可以和 `Record` 搭配。

```ts
enum OrderStatus {
  Pending = "pending",
  Paid = "paid",
  Cancelled = "cancelled"
}

const orderStatusText: Record<OrderStatus, string> = {
  [OrderStatus.Pending]: "待支付",
  [OrderStatus.Paid]: "已支付",
  [OrderStatus.Cancelled]: "已取消"
};
```

如果使用字符串联合类型，也可以达到类似效果，而且生成的 JavaScript 更少。

```ts
type OrderStatus = "pending" | "paid" | "cancelled";
```

### 模板字符串 key

TypeScript 支持模板字符串类型，所以 `Record` 也可以约束具有固定格式的 key。

```ts
type EventName = `on${Capitalize<string>}`;

type EventHandlers = Record<EventName, () => void>;

const handlers: EventHandlers = {
  onClick: () => {},
  onMouseEnter: () => {}
};
```

不过这种写法的 key 范围可能很大。实际项目里更常见的是有限联合类型：

```ts
type ButtonEvent = "onClick" | "onFocus" | "onBlur";

type ButtonHandlers = Partial<Record<ButtonEvent, () => void>>;

const handlers: ButtonHandlers = {
  onClick: () => {}
};
```

### 完整 Demo：前端菜单权限和页面标题

下面用一个完整示例串起来：角色、菜单、权限、页面标题、菜单过滤。

```ts
type Role = "admin" | "editor" | "viewer";

type MenuKey =
  | "dashboard"
  | "user"
  | "article"
  | "setting";

type MenuItem = {
  key: MenuKey;
  title: string;
  path: string;
};

const menuTitle: Record<MenuKey, string> = {
  dashboard: "工作台",
  user: "用户管理",
  article: "文章管理",
  setting: "系统设置"
};

const menuPath: Record<MenuKey, string> = {
  dashboard: "/dashboard",
  user: "/user",
  article: "/article",
  setting: "/setting"
};

const roleMenus: Record<Role, MenuKey[]> = {
  admin: ["dashboard", "user", "article", "setting"],
  editor: ["dashboard", "article"],
  viewer: ["dashboard"]
};

function createMenuItem(key: MenuKey): MenuItem {
  return {
    key,
    title: menuTitle[key],
    path: menuPath[key]
  };
}

function getMenusByRole(role: Role): MenuItem[] {
  return roleMenus[role].map(createMenuItem);
}

console.log(getMenusByRole("editor"));
```

输出结果类似：

```ts
[
  { key: "dashboard", title: "工作台", path: "/dashboard" },
  { key: "article", title: "文章管理", path: "/article" }
]
```

如果新增一个菜单：

```ts
type MenuKey =
  | "dashboard"
  | "user"
  | "article"
  | "setting"
  | "report";
```

TypeScript 会提示下面这些地方需要补充：

* `menuTitle.report`
* `menuPath.report`

至于 `roleMenus` 是否加入 `report`，这属于业务规则。因为 `roleMenus` 的 value 是 `MenuKey[]`，类型系统不会强制每个角色都必须包含全部菜单。

这就是 `Record` 在项目里的实际价值：配置项之间不容易断掉。

### 常见坑

#### Record 不会生成运行时代码

`Record` 只是类型，编译成 JavaScript 后会被擦除。

```ts
type StatusText = Record<"open" | "closed", string>;
```

这段代码运行时不存在。不能把 `Record` 当成对象、函数或类来使用。

#### Record<string, any> 会削弱类型

```ts
type Data = Record<string, any>;
```

这种写法虽然方便，但 `any` 会绕过类型检查。更推荐把 value 类型写明确。

```ts
type ApiCache = Record<string, {
  data: unknown;
  expireAt: number;
}>;
```

如果值结构暂时不确定，优先考虑 `unknown`，再在使用时做判断。

#### Record<string, T> 不代表 key 一定存在

```ts
const users: Record<string, { name: string }> = {};

const user = users["1001"];

console.log(user.name);
```

类型上 `users["1001"]` 可能被认为是 `{ name: string }`，但运行时可能是 `undefined`。

开启 `noUncheckedIndexedAccess` 后，这类访问会更安全：

```ts
const user = users["1001"];

if (user) {
  console.log(user.name);
}
```

如果 key 是有限联合类型，并且对象完整，访问会更可靠。

```ts
type Status = "open" | "closed";

const text: Record<Status, string> = {
  open: "开启",
  closed: "关闭"
};

console.log(text.open);
```

#### 数字 key 运行时会变成字符串

```ts
const map: Record<number, string> = {
  1: "one"
};

console.log(Object.keys(map));
```

输出：

```ts
["1"]
```

如果真的需要保留数字 key 的语义，或者需要频繁增删，`Map<number, T>` 可能更合适。

#### 嵌套太深会难读

```ts
type PermissionMap = Record<string, Record<string, Record<string, boolean>>>;
```

这种类型虽然能写，但很难维护。更好的写法是拆开：

```ts
type ActionPermission = Record<string, boolean>;
type ResourcePermission = Record<string, ActionPermission>;
type PermissionMap = Record<string, ResourcePermission>;
```

或者直接定义更有业务含义的接口。

### 使用建议

`Record` 适合这些场景：

* 状态码映射文案
* 枚举值映射配置
* 字段名映射表单标签
* 角色映射权限
* 路由名映射菜单配置
* 接口数组转字典
* 主题名映射主题配置

不太适合这些场景：

* 字段类型各不相同的业务实体
* key 是对象、DOM、函数的映射
* 需要频繁增删并关心插入顺序的数据
* 深层复杂业务结构
* 大量使用 `Record<string, any>` 的宽泛对象

### 总结

`Record<K, T>` 的核心就是：

```text
用 K 约束对象有哪些 key，用 T 约束每个 key 对应的 value 类型。
```

最推荐的写法是：

```ts
type Status = "pending" | "success" | "error";

const statusText: Record<Status, string> = {
  pending: "处理中",
  success: "成功",
  error: "失败"
};
```

它比散落的 `if...else` 更集中，比 `Record<string, any>` 更安全，比普通索引签名更适合有限 key 集合。

项目里只要出现“某一组固定 key 对应某一类 value”的对象，就可以优先考虑 `Record`。
