# 推荐一个 Zig Web 工程骨架：wing-app

最近看到朋友写的一个 Zig Web 项目：[dacheng-zig/wing-app](https://github.com/dacheng-zig/wing-app)。

它不是那种只跑一个 `Hello, world!` 的玩具 demo，而是一个可以直接 clone 下来改业务的 Web 服务骨架。对于想尝试 Zig 写后端、又不想从 HTTP 监听、路由、中间件、数据库连接、认证、接口文档这些基础设施一点点搭的人来说，`wing-app` 是一个很好的起点。

一句话概括：它把 Zig 生态里的 `zio`、`talon`、`wing` 和 `mantle` 组合起来，做成了一个偏工程化的 Web 应用模板。

## 它解决的不是“能不能写 Web”

很多语言或框架的早期示例，都停留在“启动服务，返回字符串”这个阶段。

这当然重要，但真正写业务服务时，问题马上会变成：

- 路由怎么组织？
- controller、service、repository 怎么分层？
- 配置从哪里来？
- 数据库连接池谁持有？
- 业务错误怎么统一映射成 HTTP 状态码？
- 登录态怎么做？
- API 文档能不能自动生成？
- 项目变大以后，目录还能不能看懂？

`wing-app` 比较有意思的一点是，它没有只展示框架 API，而是直接把这些工程问题放进了项目结构里。

它的 README 里给出的定位很明确：这是一个基于 `wing` Web 框架的 minimal, clone-and-go skeleton。也就是说，你可以 fork 或 clone 它，改包名，删掉示例业务，然后开始写自己的路由、handler 和状态。

## 技术栈：zio + talon + wing + mantle

`wing-app` 当前的技术栈大致是：

- `zio`：协程运行时
- `talon`：HTTP 引擎
- `wing`：Web 框架，负责路由、中间件、Context、extractor 等
- `mantle`：纯 Zig MySQL driver，用于数据库访问

README 里有一个很关键的描述：`wing` 的框架魔法在 comptime 解决，稳定请求路径上没有反射，也不需要堆分配。

这很 Zig。

Zig 写 Web 最吸引人的地方，不只是“语法不同”，而是它可以把很多动态框架运行时才做的事前移到编译期。handler 的函数签名、请求提取、状态投影、响应类型，都可以变成编译期可检查的结构。

比如一个用户详情接口可以写成这样：

```zig
pub fn show(
    ctx: *Ctx,
    svc: *UserService,
    path: wing.Path(struct { id: u64 }),
) anyerror!wing.Json(User) {
    const user = try svc.get(ctx.arena, path.value.id);
    return .{ .value = user };
}
```

这里能看出几个设计取向：

- `wing.Path(struct { id: u64 })` 明确表达路径参数
- `*UserService` 从 `AppState` 按类型投影出来
- 返回值是 `wing.Json(User)`，响应类型也是显式的
- controller 只做 HTTP 绑定和响应组织，业务逻辑在 service

如果你写惯了 Spring、ASP.NET Core、NestJS 或 axum，这种结构会很熟悉。但它背后是 Zig 的 comptime 和显式依赖，而不是一个黑盒 DI 容器。

## 项目结构很“后端工程”

`wing-app` 的目录不是随手堆的，它采用了经典分层：

```text
src/
├── main.zig
├── server.zig
├── app.zig
├── state.zig
├── config/
├── db/
├── routes/
├── controllers/
├── services/
├── repositories/
├── models/
├── middleware/
├── views/
├── auth/
├── user/
├── docs/
├── openapi/
└── trace/
```

README 和 `docs/architecture.md` 里都强调了请求流：

```text
routes -> controller -> service -> repository
```

每层只做自己的事：

- `routes` 负责 URL 到功能模块的组合
- `controllers` 负责请求绑定、调用 service、组织响应
- `services` 负责业务规则，不感知 HTTP
- `repositories` 负责数据访问
- `models` 放实体和 DTO
- `state.zig` 负责把 config、database、repositories、services 装配到一起

这个结构不新奇，但很实用。

我一直觉得，一个 Web 框架要想真正落地，不能只告诉你“怎么注册路由”，还要告诉你“代码长大以后应该放在哪里”。`wing-app` 在这点上是加分的。

## 没有 DI 容器，但依赖是清楚的

`wing-app` 使用 `AppState` 显式聚合应用依赖。

例如配置、数据库、用户服务、认证服务、session 服务等，都挂在统一的应用状态里。handler 需要什么，就在函数参数里声明对应类型的指针，框架在编译期根据类型完成投影。

这种方式有两个好处：

第一，依赖关系是显式的。你看 handler 签名，就知道它需要什么。

第二，没有运行时 DI 容器那类隐式魔法。装配点在 `state.zig`，出问题更容易定位。

代价也很清楚：顶层字段类型需要保持可区分。如果有两个同类型依赖，就需要包一层不同的 struct 来消歧。

我喜欢这种取舍，因为它符合 Zig 的气质：少一点隐式，多一点可读的结构。

## 数据库不是假数据：已经接了 MySQL

很多后端骨架项目喜欢先用内存数组演示 CRUD，最后真正接数据库时又是另一套故事。

`wing-app` 不是这样。它已经通过 `mantle` 接入 MySQL，并且有连接池、迁移和 repository 层。

当前项目里有这些迁移文件：

- `001_create_users.sql`
- `002_create_roles.sql`
- `003_create_sessions.sql`

应用启动时会管理自己的表结构，但不会自动创建 database。README 里也说明了需要提前创建数据库，例如：

```bash
mysql -uroot -proot -e "CREATE DATABASE IF NOT EXISTS wing_app;"
```

数据库配置来自环境变量：

- `DB_HOST`
- `DB_PORT`
- `DB_USER`
- `DB_PASSWORD`
- `DB_NAME`
- `DB_POOL_SIZE`

默认值是本地 MySQL、`root/root`、数据库 `wing_app`、连接池大小 `8`。

这说明作者不是只想做一个“看起来能跑”的 demo，而是在搭一个真实服务会用到的基础结构。

## 认证和授权已经有雏形

项目里还有一个完整的 `auth` 模块，包含：

- login
- logout
- me
- session repository
- role repository
- password support
- auth extractor
- authorizer

登录后会创建服务端 session，并通过 cookie 返回 `session_id`。cookie 带有 `HttpOnly`、`Secure`、`SameSite=Strict` 等属性。

接口层也能通过类型表达“这个接口需要登录”。

比如 `logout` 和 `me` 的 handler 参数里声明 `Auth`，这就不是靠注释约定，而是通过函数签名表达认证要求：

```zig
pub fn me(ctx: *Ctx, auth: Auth) anyerror!wing.Json(Profile) {
    const user = try ctx.state.users.get(ctx.arena, auth.principal.id);
    return .{ .value = .{
        .id = user.id,
        .name = user.name,
        .roles = auth.principal.roles,
    } };
}
```

这类设计很适合 Zig：把约束放到类型和编译期能理解的位置，而不是散落在注释和运行时判断里。

## OpenAPI 文档是一个亮点

我觉得 `wing-app` 里最值得单独拿出来说的，是它内置了一套 OpenAPI 生成能力。

根据文档，只要把 feature 路由从 `wing.Router` 换成 `openapi.Router`，并在路由注册时补一个文档参数，就可以生成 OpenAPI 3.1 文档。

它提供：

- `GET /openapi.json`：生成的 OpenAPI 文档
- `GET /docs`：Scalar 渲染的交互式接口文档页
- `zig build openapi`：不启动服务器、不连数据库，直接把 spec 输出到 stdout

更重要的是，schema 来自 handler 签名。

也就是说：

- `wing.Path(...)` 变成 path 参数
- `wing.Query(...)` 变成 query 参数
- `wing.Json(T)` 入参变成 request body
- `wing.Json(T)` 返回变成 200 响应
- `wing.Created(T)` 返回变成 201 响应
- `Auth`、`OptionalAuth`、`Require(...)` 会体现在 security 里

这点非常实用。

很多项目的接口文档腐烂，根源就是代码和文档是两份真相。`wing-app` 的思路是把 handler 签名作为单一真相源，文档从代码推导出来。

当然它现在也有一些限制，比如错误响应还没有完整进入文档、角色要求暂时不会自动写进 spec、Scalar 页面依赖 CDN。但作为一个 Zig Web 骨架，这个完成度已经很值得关注。

## 中间件和错误处理也比较克制

`app.zig` 里定义了应用的请求管线：

```zig
pub const App = wing.App(AppState, .{
    request_scope,
    wing.middleware.logger,
    wing.middleware.recoverWith(errorStatus),
    wing.middleware.route_match,
});
```

错误到 HTTP 状态码的映射也集中在这里，比如：

- `error.InvalidName` -> 400
- `error.InvalidCredentials` -> 400
- `error.UsernameTaken` -> 409
- 其他错误走 wing 默认映射

这种集中处理比每个 controller 里手写 status code 更容易维护。

另外 README 里提到，中间件顺序会在 comptime 做检查，例如某些中间件必须跟在 route match 之后。对 Zig 来说，这种“能编译期失败就不要拖到运行时”的设计非常自然。

## 适合什么人看？

我觉得 `wing-app` 很适合三类人：

第一类，是正在关注 Zig 后端生态的人。

你可以通过它看到 Zig 写 Web 服务时，路由、状态、数据库、认证、文档这些模块可以怎样组合起来。

第二类，是想写一个真实 Zig Web 项目的人。

相比从零开始搭 HTTP server，这个项目已经给了比较完整的工程骨架。你可以先按它的分层方式写业务，再根据项目规模调整成 feature-first 的模块化结构。

第三类，是框架作者或基础设施爱好者。

`wing-app` 不是只展示业务代码，它还展示了一个 Zig Web 框架应该如何承接真实工程需求：typed extractor、typed response、comptime 路由装配、OpenAPI 推导、状态投影、认证 extractor 等。

## 也要注意：它还处在早期阶段

推荐归推荐，但也要说清楚边界。

`wing-app` 当前依赖本地 sibling 目录里的 `wing` 和 `mantle`：

```zig
.dependencies = .{
    .wing = .{ .path = "../wing" },
    .mantle = .{ .path = "../mantle" },
}
```

这说明它还处在 pre-release 协同开发阶段。等 `wing`、`talon`、`mantle` 发布正式 tag 后，才会更适合作为稳定依赖通过 URL 和 hash 锁定。

另外它要求 Zig `0.16.0`，这本身也意味着你需要跟上 Zig 版本变化。

所以我的建议是：

- 学习和研究：很适合
- 写内部工具或实验性服务：可以尝试
- 直接上严肃生产：需要你自己评估依赖稳定性、测试覆盖、部署环境和安全边界

## 为什么我愿意推荐它？

因为它展示了一种我很喜欢的方向：用 Zig 写 Web，不是把其他语言的框架照搬一遍，而是利用 Zig 自己的能力重新组织 Web 工程。

它强调：

- 显式依赖
- 类型化请求提取
- 类型化响应
- 编译期装配
- 清楚的分层边界
- 数据库和认证这些真实服务绕不开的问题
- 自动生成 OpenAPI 文档

这些东西合在一起，才像一个“可以继续长大”的 Web 项目。

如果你对 Zig 后端开发感兴趣，可以去看看这个仓库：

[https://github.com/dacheng-zig/wing-app](https://github.com/dacheng-zig/wing-app)

可以先从 README、`docs/architecture.md` 和 `docs/openapi-user-guide.md` 看起，再顺着 `src/routes/routes.zig`、`src/state.zig`、`src/user/`、`src/auth/` 读代码。

即使你暂时不准备用 Zig 写生产服务，这个项目也值得一看。它很好地回答了一个问题：当 Zig 进入 Web 后端领域时，一个工程化的应用骨架可以长成什么样。

# 性能对比

## .net Minimal API

![alt text](/images/zig/.net.png)

## zig wing-app API

![alt text](/images/zig/zig.png)

在我的 Mac M1 上性能测试：
.net 能跑到5856，而 zig 能跑到6W+，10X 的差距，确实夸张。






