### 简介

`keyof` 是 TypeScript 里的类型操作符，用来提取一个对象类型的所有 key，并组成一个联合类型。

先看一个普通对象类型：

```ts
interface User {
  id: number;
  name: string;
  age: number;
}
```

使用 `keyof`：

```ts
type UserKey = keyof User;
```

得到的结果相当于：

```ts
type UserKey = "id" | "name" | "age";
```

一句话概括：

```text
keyof 负责把对象类型的 key 提取成联合类型。
```

它经常出现在这些场景里：

* 根据字段名安全取值
* 根据字段名安全赋值
* 表格列配置
* 表单字段校验
* 接口字段映射
* `Pick`、`Omit`、`Partial`、`Record` 等工具类型源码
* `keyof typeof` 从配置对象里反推出 key 类型

`keyof` 看起来只是一个小语法，但它是 TypeScript 类型系统里非常核心的一块。

### 基本语法

```ts
keyof T
```

含义：

```text
获取 T 类型上的所有属性名，并组成联合类型。
```

示例：

```ts
type Product = {
  id: number;
  title: string;
  price: number;
  enabled: boolean;
};

type ProductKey = keyof Product;
```

`ProductKey` 等价于：

```ts
type ProductKey = "id" | "title" | "price" | "enabled";
```

所以变量只能赋这些值：

```ts
let key: ProductKey;

key = "id";
key = "title";
key = "price";
key = "enabled";

key = "name";
// Type '"name"' is not assignable to type 'keyof Product'.
```

### keyof 和 Object.keys 的区别

`keyof` 很像类型层面的 `Object.keys()`，但两者不是一回事。

```ts
const user = {
  id: 1,
  name: "张三"
};

console.log(Object.keys(user));
```

运行结果：

```ts
["id", "name"]
```

`Object.keys()` 是运行时方法，代码执行时才会拿到对象真实 key。

`keyof` 是类型操作符，只在 TypeScript 编译阶段生效。

```ts
type User = {
  id: number;
  name: string;
};

type UserKey = keyof User;
```

编译成 JavaScript 后，`UserKey` 不存在。

对比：

| 对比项 | keyof | Object.keys() |
| --- | --- | --- |
| 所在阶段 | 编译时 | 运行时 |
| 作用对象 | 类型 | 真实对象 |
| 返回结果 | 联合类型 | 字符串数组 |
| 是否生成 JS 代码 | 不生成 | 生成 |

简单理解：

```text
keyof 处理类型，Object.keys() 处理数据。
```

### 为什么需要 keyof

在 JavaScript 里，动态取属性很常见：

```ts
function getValue(obj: any, key: string) {
  return obj[key];
}
```

这种写法的问题是：`key` 可以传任何字符串。

```ts
const user = {
  id: 1,
  name: "张三"
};

getValue(user, "name");
getValue(user, "xxx");
```

`"xxx"` 根本不是 `user` 的属性，但类型检查不会拦住。

加上 `keyof` 后：

```ts
type User = {
  id: number;
  name: string;
};

function getValue(obj: User, key: keyof User) {
  return obj[key];
}

const user: User = {
  id: 1,
  name: "张三"
};

getValue(user, "name");
getValue(user, "xxx");
// Argument of type '"xxx"' is not assignable to parameter of type 'keyof User'.
```

这样就把“属性名不能乱传”这件事交给了类型系统。

### keyof + 泛型：安全取值

实际项目里，通常不会只给某一个类型写取值函数，而是写成泛型。

```ts
function getValue<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

拆开看：

```ts
T
```

表示对象类型。

```ts
K extends keyof T
```

表示 `K` 必须是 `T` 的某个 key。

```ts
T[K]
```

表示这个 key 对应的 value 类型。

使用示例：

```ts
const user = {
  id: 1,
  name: "张三",
  age: 18
};

const id = getValue(user, "id");
const name = getValue(user, "name");
const age = getValue(user, "age");
```

TypeScript 能自动推导出：

```ts
const id: number
const name: string
const age: number
```

如果传不存在的 key：

```ts
getValue(user, "email");
// Argument of type '"email"' is not assignable to parameter of type '"id" | "name" | "age"'.
```

这是 `keyof` 最经典的用法。

### keyof + 泛型：安全赋值

取值可以安全，赋值也可以安全。

```ts
function setValue<T, K extends keyof T>(
  obj: T,
  key: K,
  value: T[K]
) {
  obj[key] = value;
}
```

使用：

```ts
const user = {
  id: 1,
  name: "张三",
  age: 18
};

setValue(user, "name", "李四");
setValue(user, "age", 20);
```

传错值类型会报错：

```ts
setValue(user, "age", "20");
// Argument of type 'string' is not assignable to parameter of type 'number'.
```

这里的重点是 `T[K]`。它会根据传入的 key 自动找到对应的 value 类型。

### keyof typeof

`keyof` 后面接的是类型，不是值。

```ts
const statusText = {
  pending: "待支付",
  paid: "已支付",
  cancelled: "已取消"
};

type Status = keyof statusText;
// Cannot find name 'statusText'.
```

因为 `statusText` 是一个变量，不是类型。

如果想从变量反推出 key 类型，需要先用 `typeof` 获取变量的类型，再用 `keyof` 提取 key。

```ts
const statusText = {
  pending: "待支付",
  paid: "已支付",
  cancelled: "已取消"
};

type Status = keyof typeof statusText;
```

`Status` 等价于：

```ts
type Status = "pending" | "paid" | "cancelled";
```

这就是 `keyof typeof`。

拆开看：

```text
typeof statusText：拿到变量 statusText 的类型
keyof typeof statusText：拿到这个类型的所有 key
```

这种写法很适合配置表。

```ts
const themeConfig = {
  light: { label: "浅色模式", background: "#ffffff" },
  dark: { label: "深色模式", background: "#141414" },
  system: { label: "跟随系统", background: "var(--page-bg)" }
} as const;

type ThemeName = keyof typeof themeConfig;

function getTheme(name: ThemeName) {
  return themeConfig[name];
}

getTheme("dark");
getTheme("blue");
// Argument of type '"blue"' is not assignable to parameter of type '"light" | "dark" | "system"'.
```

这里的 `as const` 会让对象里的值保持更窄的字面量类型，常量配置里很常见。

### keyof 和索引访问类型

`keyof` 经常和索引访问类型一起出现。

```ts
type User = {
  id: number;
  name: string;
  age: number;
};

type UserKey = keyof User;
```

`UserKey` 是：

```ts
"id" | "name" | "age"
```

再看：

```ts
type UserValue = User[UserKey];
```

等价于：

```ts
type UserValue = User["id" | "name" | "age"];
```

最后得到：

```ts
type UserValue = number | string;
```

也可以只取单个字段类型：

```ts
type UserName = User["name"];
// string

type UserAge = User["age"];
// number
```

这种写法常用于从接口类型里提取字段类型。

```ts
type ApiResponse = {
  code: number;
  message: string;
  data: {
    list: Array<{
      id: number;
      title: string;
    }>;
    total: number;
  };
};

type ListItem = ApiResponse["data"]["list"][number];
```

`ListItem` 等价于：

```ts
type ListItem = {
  id: number;
  title: string;
};
```

### keyof 和映射类型

很多 TypeScript 内置工具类型都离不开 `keyof`。

例如 `Partial` 的核心逻辑：

```ts
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};
```

意思是：

```text
遍历 T 的每一个 key，把每个属性变成可选属性。
```

使用：

```ts
type User = {
  id: number;
  name: string;
  age: number;
};

type PartialUser = MyPartial<User>;
```

等价于：

```ts
type PartialUser = {
  id?: number;
  name?: string;
  age?: number;
};
```

再看 `Readonly`：

```ts
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};
```

使用：

```ts
type ReadonlyUser = MyReadonly<User>;
```

等价于：

```ts
type ReadonlyUser = {
  readonly id: number;
  readonly name: string;
  readonly age: number;
};
```

这就是 `keyof` 和 `in` 的组合：

```text
keyof 负责拿到 key 集合
in 负责遍历 key 集合
T[K] 负责拿到每个 key 对应的 value 类型
```

### keyof 和 Pick

`Pick<T, K>` 表示从 `T` 里挑选一部分属性。

它的核心实现大概是：

```ts
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};
```

使用：

```ts
type User = {
  id: number;
  name: string;
  email: string;
  password: string;
};

type PublicUser = MyPick<User, "id" | "name" | "email">;
```

等价于：

```ts
type PublicUser = {
  id: number;
  name: string;
  email: string;
};
```

如果挑了不存在的字段：

```ts
type ErrorUser = MyPick<User, "id" | "nickname">;
// Type '"id" | "nickname"' does not satisfy the constraint 'keyof User'.
```

`K extends keyof T` 的作用就是限制只能挑已有字段。

### keyof 和 Record

`Record` 常和 `keyof` 搭配，用来把某个类型的字段变成另一个映射表。

```ts
type User = {
  id: number;
  name: string;
  email: string;
};

type UserColumnText = Record<keyof User, string>;
```

等价于：

```ts
type UserColumnText = {
  id: string;
  name: string;
  email: string;
};
```

使用：

```ts
const userColumnText: UserColumnText = {
  id: "用户 ID",
  name: "姓名",
  email: "邮箱"
};
```

如果 `User` 新增一个字段：

```ts
type User = {
  id: number;
  name: string;
  email: string;
  phone: string;
};
```

`userColumnText` 会提示补充 `phone`。这在表格列名、导出字段、表单标签里很实用。

### keyof 和数组、元组

数组也是对象，所以 `keyof` 也能作用在数组上。

```ts
type ArrayKeys = keyof string[];
```

结果会包含：

```ts
number | "length" | "push" | "pop" | "map" | ...
```

因为数组可以通过数字下标访问，也有 `length`、`push`、`map` 等属性和方法。

元组也类似：

```ts
type Tuple = [string, number];

type TupleKeys = keyof Tuple;
```

结果会包含：

```ts
number | "0" | "1" | "length" | ...
```

实际业务里，`keyof` 更多用于普通对象类型。数组和元组里的 `keyof` 了解即可，没必要强行使用。

### keyof 和索引签名

索引签名会影响 `keyof` 的结果。

```ts
type StringMap = {
  [key: string]: boolean;
};

type StringMapKey = keyof StringMap;
```

结果是：

```ts
type StringMapKey = string | number;
```

为什么会有 `number`？

因为 JavaScript 对象里的数字 key 会被转成字符串。

```ts
const map = {
  1: true
};

console.log(map[1]);
console.log(map["1"]);
```

这两种访问方式都能拿到值。

数字索引签名：

```ts
type NumberMap = {
  [key: number]: boolean;
};

type NumberMapKey = keyof NumberMap;
```

结果是：

```ts
type NumberMapKey = number;
```

### keyof any、unknown、object

`keyof any` 是：

```ts
type AnyKey = keyof any;
// string | number | symbol
```

原因是 JavaScript 对象属性名只能是字符串、数字或 `symbol`。

`Record` 的源码里就能看到它：

```ts
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
```

`keyof unknown` 是：

```ts
type UnknownKey = keyof unknown;
// never
```

因为 `unknown` 表示完全未知，不能确定它有哪些属性。

`keyof object` 也是：

```ts
type ObjectKey = keyof object;
// never
```

`object` 只表示非原始值，并不表示它一定有某个已知属性。

### keyof 联合类型和交叉类型

这一点容易误解。

先看联合类型：

```ts
type A = {
  id: number;
  name: string;
};

type B = {
  id: number;
  enabled: boolean;
};

type Keys = keyof (A | B);
```

`Keys` 的结果是：

```ts
type Keys = "id";
```

因为 `A | B` 表示值可能是 `A`，也可能是 `B`。只有 `id` 是两边都确定存在的 key，所以只能安全访问 `id`。

再看交叉类型：

```ts
type Keys = keyof (A & B);
```

结果是：

```ts
type Keys = "id" | "name" | "enabled";
```

因为 `A & B` 表示同时拥有 `A` 和 `B` 的属性。

简单记：

```text
keyof (A | B)：取共同 key
keyof (A & B)：取合并后的 key
```

### 实战一：表格列配置

后台管理系统里经常有表格：

```ts
type User = {
  id: number;
  name: string;
  email: string;
  status: "enabled" | "disabled";
  createdAt: string;
};
```

列配置如果直接写字符串，容易写错：

```ts
const columns = [
  { key: "name", title: "姓名" },
  { key: "emial", title: "邮箱" }
];
```

`"emial"` 是拼写错误，但普通字符串不会报错。

可以用 `keyof` 限制：

```ts
type TableColumn<T> = {
  key: keyof T;
  title: string;
  width?: number;
};

const columns: TableColumn<User>[] = [
  { key: "id", title: "用户 ID", width: 100 },
  { key: "name", title: "姓名" },
  { key: "email", title: "邮箱" },
  { key: "status", title: "状态" },
  { key: "createdAt", title: "创建时间" }
];
```

写错字段会报错：

```ts
const columns: TableColumn<User>[] = [
  { key: "emial", title: "邮箱" }
  // Type '"emial"' is not assignable to type 'keyof User'.
];
```

渲染表格时：

```ts
function renderCell<T>(row: T, column: TableColumn<T>) {
  return row[column.key];
}
```

这样列配置和数据字段就绑在一起了。

### 实战二：表单字段校验

表单字段很适合用 `keyof`。

```ts
type LoginForm = {
  username: string;
  password: string;
  captcha: string;
  remember: boolean;
};
```

定义校验规则：

```ts
type Validator<T, K extends keyof T> = (value: T[K], values: T) => string | null;

type ValidationRules<T> = {
  [K in keyof T]?: Validator<T, K>;
};
```

使用：

```ts
const loginRules: ValidationRules<LoginForm> = {
  username: value => value ? null : "请输入用户名",
  password: value => value.length >= 6 ? null : "密码至少 6 位",
  captcha: value => value ? null : "请输入验证码"
};
```

这里每个字段的 `value` 类型都是准确的。

```ts
const loginRules: ValidationRules<LoginForm> = {
  remember: value => value ? null : "请确认记住登录状态"
};
```

`remember` 的 `value` 会被推导成 `boolean`。

写不存在的字段会报错：

```ts
const loginRules: ValidationRules<LoginForm> = {
  email: value => null
  // Object literal may only specify known properties.
};
```

实现校验函数：

```ts
function validateForm<T extends object>(
  values: T,
  rules: ValidationRules<T>
): Partial<Record<keyof T, string>> {
  const errors: Partial<Record<keyof T, string>> = {};

  for (const key of Object.keys(rules) as Array<keyof T>) {
    const rule = rules[key] as
      | ((value: T[typeof key], values: T) => string | null)
      | undefined;

    if (!rule) {
      continue;
    }

    const error = rule(values[key], values);

    if (error) {
      errors[key] = error;
    }
  }

  return errors;
}
```

使用：

```ts
const form: LoginForm = {
  username: "",
  password: "123",
  captcha: "",
  remember: false
};

const errors = validateForm(form, loginRules);

console.log(errors);
```

输出类似：

```ts
{
  username: "请输入用户名",
  password: "密码至少 6 位",
  captcha: "请输入验证码"
}
```

### 实战三：接口字段映射

接口字段经常要映射成表格列名、导出列名、表单 label。

```ts
type Product = {
  id: number;
  name: string;
  price: number;
  stock: number;
  status: "on" | "off";
};
```

字段文案：

```ts
const productFieldText: Record<keyof Product, string> = {
  id: "商品 ID",
  name: "商品名称",
  price: "价格",
  stock: "库存",
  status: "状态"
};
```

如果 `Product` 新增字段：

```ts
type Product = {
  id: number;
  name: string;
  price: number;
  stock: number;
  status: "on" | "off";
  category: string;
};
```

`productFieldText` 会提示补充 `category`。

导出 CSV 时可以利用这个映射：

```ts
function exportRows<T extends object>(
  rows: T[],
  fieldText: Record<keyof T, string>
) {
  const keys = Object.keys(fieldText) as Array<keyof T>;
  const header = keys.map(key => fieldText[key]);
  const body = rows.map(row => keys.map(key => String(row[key] ?? "")));

  return [header, ...body];
}

const csvRows = exportRows<Product>(
  [
    { id: 1, name: "键盘", price: 399, stock: 100, status: "on" },
    { id: 2, name: "鼠标", price: 159, stock: 80, status: "off" }
  ],
  productFieldText
);
```

这样导出字段和类型字段能保持同步。

### 实战四：安全 pick 函数

`pick` 的作用是从对象里取一部分字段。

```ts
function pick<T extends object, K extends keyof T>(
  obj: T,
  keys: K[]
): Pick<T, K> {
  const result = {} as Pick<T, K>;

  for (const key of keys) {
    result[key] = obj[key];
  }

  return result;
}
```

使用：

```ts
const user = {
  id: 1,
  name: "张三",
  email: "zhangsan@example.com",
  password: "123456"
};

const publicUser = pick(user, ["id", "name", "email"]);
```

`publicUser` 的类型是：

```ts
{
  id: number;
  name: string;
  email: string;
}
```

传不存在的 key 会报错：

```ts
pick(user, ["id", "nickname"]);
// Type '"nickname"' is not assignable to type '"id" | "name" | "email" | "password"'.
```

### 实战五：安全排序字段

列表页经常有排序字段，例如按价格、库存、创建时间排序。

```ts
type Product = {
  id: number;
  name: string;
  price: number;
  stock: number;
  createdAt: string;
};
```

如果所有字段都允许排序，可以直接用：

```ts
type SortField = keyof Product;
```

但实际项目里，不是所有字段都适合排序。可以用 `Pick` 先筛选，再 `keyof`：

```ts
type SortableProduct = Pick<Product, "price" | "stock" | "createdAt">;
type SortField = keyof SortableProduct;
```

排序参数：

```ts
type SortOrder = "asc" | "desc";

type SortQuery = {
  field: SortField;
  order: SortOrder;
};
```

使用：

```ts
const query: SortQuery = {
  field: "price",
  order: "desc"
};
```

错误字段会被拦住：

```ts
const query: SortQuery = {
  field: "name",
  order: "asc"
  // Type '"name"' is not assignable to type '"price" | "stock" | "createdAt"'.
};
```

### 实战六：配置对象反推类型

有些数据本身就是配置对象，没必要先手写类型，再写配置。

```ts
const routeConfig = {
  dashboard: {
    title: "工作台",
    path: "/dashboard",
    needLogin: true
  },
  article: {
    title: "文章管理",
    path: "/article",
    needLogin: true
  },
  login: {
    title: "登录",
    path: "/login",
    needLogin: false
  }
} as const;
```

从配置对象反推出路由名：

```ts
type RouteName = keyof typeof routeConfig;
```

使用：

```ts
function getRoute(name: RouteName) {
  return routeConfig[name];
}

getRoute("dashboard");
getRoute("setting");
// Argument of type '"setting"' is not assignable to parameter of type '"dashboard" | "article" | "login"'.
```

获取配置里的某个字段类型：

```ts
type RouteConfig = typeof routeConfig;
type RouteItem = RouteConfig[RouteName];
```

这种写法适合路由配置、权限配置、状态文案配置、主题配置。

### Object.keys 的类型问题

`Object.keys()` 返回的是 `string[]`，不是 `Array<keyof T>`。

```ts
const user = {
  id: 1,
  name: "张三"
};

const keys = Object.keys(user);
```

`keys` 的类型是：

```ts
string[]
```

所以这样写可能报错：

```ts
for (const key of Object.keys(user)) {
  console.log(user[key]);
  // Element implicitly has an 'any' type because expression of type 'string' can't be used to index ...
}
```

常见处理方式是写一个小工具函数：

```ts
function objectKeys<T extends object>(obj: T) {
  return Object.keys(obj) as Array<keyof T>;
}
```

使用：

```ts
for (const key of objectKeys(user)) {
  console.log(key, user[key]);
}
```

需要注意：这个断言只适合普通对象。因为运行时对象可能有额外属性，类型系统无法 100% 保证 `Object.keys()` 一定只返回 `keyof T`。

### 常见坑

#### keyof 不会读取运行时对象

`keyof` 只看类型，不看运行时数据。

```ts
type User = {
  id: number;
  name: string;
};

type UserKey = keyof User;
```

这里的 `UserKey` 只存在于类型系统里。运行时没有 `UserKey` 这个变量。

#### keyof 后面不能直接接变量

```ts
const user = {
  id: 1,
  name: "张三"
};

type UserKey = keyof user;
// Cannot find name 'user'.
```

正确写法：

```ts
type UserKey = keyof typeof user;
```

#### keyof string 不是字符串的索引

```ts
type StringKey = keyof string;
```

结果不是任意字符串，而是字符串类型上的属性和方法，例如：

```ts
"length" | "toString" | "charAt" | ...
```

所以不要把 `keyof string` 理解成 `string`。

#### keyof 联合类型只保留共同 key

```ts
type A = { a: string; common: number };
type B = { b: string; common: number };

type Keys = keyof (A | B);
// "common"
```

如果需要拿到所有 key，可以用分布式条件类型：

```ts
type KeysOfUnion<T> = T extends T ? keyof T : never;

type AllKeys = KeysOfUnion<A | B>;
// "a" | "b" | "common"
```

这属于进阶写法，只有处理联合类型工具时才常用。

#### keyof 类型太宽时意义会变弱

```ts
type AnyObject = Record<string, unknown>;

type AnyObjectKey = keyof AnyObject;
// string
```

如果对象本身就是任意字符串 key，`keyof` 得到的也只是宽泛的 `string`，类型约束会变弱。

更推荐在配置、状态、字段名这类场景里使用有限 key：

```ts
type StatusMap = {
  pending: string;
  paid: string;
  cancelled: string;
};

type Status = keyof StatusMap;
// "pending" | "paid" | "cancelled"
```

### 使用建议

`keyof` 适合这些场景：

* 约束动态访问对象属性的 key
* 根据接口字段生成表格列配置
* 根据表单字段生成校验规则
* 根据接口字段生成导出列名
* 从配置对象反推出 key 类型
* 编写 `Pick`、`Omit`、`Partial`、`Readonly` 这类工具类型
* 和 `Record` 搭配维护字段映射表

不太适合这些场景：

* 普通字符串处理
* 运行时枚举对象真实 key
* 没有固定字段结构的宽泛对象
* 为了类型技巧强行包装简单代码

### 总结

`keyof` 的核心就是：

```text
把对象类型的 key 提取成联合类型。
```

最常见的写法是：

```ts
function getValue<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

再配合 `typeof`：

```ts
const statusText = {
  pending: "待支付",
  paid: "已支付",
  cancelled: "已取消"
} as const;

type Status = keyof typeof statusText;
```

`keyof` 的价值不是让代码看起来更复杂，而是把散落的字符串 key 收进类型系统里。字段写错、字段遗漏、字段类型不匹配，都可以更早暴露出来。
