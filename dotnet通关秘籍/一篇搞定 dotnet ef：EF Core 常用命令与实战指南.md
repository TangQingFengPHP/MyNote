### 基础知识

| 项目          | 说明                                                                                                                                          |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **EF Core**   | .NET 的 ORM 框架，支持 Code First、Database First。                                                                                           |
| **dotnet ef** | 一个 **CLI 工具**，用于管理 EF Core 迁移、数据库操作。                                                                                        |
| **安装方式**  | 通常安装在项目中（推荐）：<br>`dotnet add package Microsoft.EntityFrameworkCore.Design`<br>全局工具：`dotnet tool install --global dotnet-ef` |
| **使用位置**  | 在包含 `*.csproj` 的目录执行，或者使用 `--project` 指定项目路径。                                                                             |

> 在执行任何 dotnet ef 命令前，需要在 csproj 中包含 Microsoft.EntityFrameworkCore.Design 包。

### 命令总览

| 命令                           | 作用                         | 典型场景                 |
| ------------------------------ | ---------------------------- | ------------------------ |
| `dotnet ef migrations add`     | 添加新的数据库迁移           | 增加或修改实体模型后     |
| `dotnet ef migrations list`    | 查看已有迁移                 | 确认数据库版本           |
| `dotnet ef migrations remove`  | 删除最新迁移                 | 回滚刚添加但未应用的迁移 |
| `dotnet ef migrations script`  | 生成 SQL 脚本                | 手工执行迁移             |
| `dotnet ef database update`    | 更新数据库到指定迁移         | 同步数据库与模型         |
| `dotnet ef database drop`      | 删除数据库                   | 重置开发环境             |
| `dotnet ef dbcontext info`     | 查看 DbContext 信息          | 调试上下文配置           |
| `dotnet ef dbcontext list`     | 列出可用 DbContext           | 多上下文项目             |
| `dotnet ef dbcontext scaffold` | 根据现有数据库生成实体       | Database First 逆向工程  |
| `dotnet ef dbcontext optimize` | 预生成模型快照以提升启动速度 | 高性能场景               |

### 常用命令详解 + 示例

#### 创建迁移

```shell
dotnet ef migrations add InitialCreate
```

在 `Migrations`/ 目录下生成：

* `YYYYMMDDHHMMSS_InitialCreate.cs`：迁移代码

* `AppDbContextModelSnapshot.cs`：模型快照文件

常用参数：

| 参数                | 说明                             | 示例                            |
| ------------------- | -------------------------------- | ------------------------------- |
| `--project`         | 指定启动项目                     | `--project ./MyApp`             |
| `--startup-project` | 指定包含 `Program.cs` 的启动项目 | `--startup-project ./MyApp.Web` |
| `--context`         | 指定 DbContext                   | `--context AppDbContext`        |
| `--output-dir`      | 指定迁移文件夹                   | `--output-dir Data/Migrations`  |


> 在多项目架构中（如分离 `DAL`/启动项目），务必同时指定 `--project` 和 `--startup-project`。

#### 应用迁移

```shell
dotnet ef database update
```

* 将数据库更新到最新迁移。

* 如果数据库不存在，会自动创建。

可指定版本：

```shell
dotnet ef database update InitialCreate
```

将数据库回滚到指定迁移（或升级到某个中间版本）。

#### 查看迁移

```shell
dotnet ef migrations list
```

输出：

```
20250922121212_InitialCreate
20250923104530_AddUserTable
```

#### 删除迁移

```shell
dotnet ef migrations remove
```

* 删除最新迁移文件。

* 仅限未执行 `database update` 的迁移。

#### 生成 SQL 脚本

```shell
dotnet ef migrations script
```

* 生成从初始数据库到最新迁移的 `SQL` 脚本。

* 可指定起止迁移：

```shell
dotnet ef migrations script InitialCreate AddUserTable -o update.sql
```

#### 删除数据库

```shell
dotnet ef database drop
```

* 交互式确认后删除数据库。

* 可加 `--force` 跳过确认。

#### 查看 DbContext 信息

```shell
dotnet ef dbcontext info
```

输出数据库提供程序、连接字符串等信息。

#### 列出 DbContext

```shell
dotnet ef dbcontext list
```

用于多上下文项目，可快速确认可用的 `DbContext`。

#### 数据库逆向生成实体 (Database First)

```shell
dotnet ef dbcontext scaffold \
"Server=localhost;Database=MyDb;User Id=sa;Password=Passw0rd;" \
Microsoft.EntityFrameworkCore.SqlServer \
--output-dir Models --context MyDbContext
```

常用参数：

| 参数                   | 说明                                       |
| ---------------------- | ------------------------------------------ |
| `--schema`             | 指定数据库模式                             |
| `--table`              | 指定表（可多次指定）                       |
| `--context-dir`        | 指定 DbContext 文件夹                      |
| `--force`              | 覆盖已有文件                               |
| `--use-database-names` | 保留数据库原始命名（不做 PascalCase 转换） |

#### 模型预编译优化

`EF Core 6+` 提供：

```shell
dotnet ef dbcontext optimize
```

* 在编译时预生成模型元数据，提升启动速度。

* 适合大型数据库或高性能场景。

### 开发常用流程示例

#### Code First 开发流程

```shell
# 1. 添加初始迁移
dotnet ef migrations add InitialCreate

# 2. 创建/更新数据库
dotnet ef database update

# 3. 修改实体模型后生成新迁移
dotnet ef migrations add AddUserTable

# 4. 应用到数据库
dotnet ef database update
```

#### Database First 开发流程

```shell
# 从现有数据库生成实体与上下文
dotnet ef dbcontext scaffold \
"Server=localhost;Database=MyDb;User Id=sa;Password=Passw0rd;" \
Microsoft.EntityFrameworkCore.SqlServer \
--output-dir Models
```

### 常见问题与技巧

| 问题                        | 解决方案                                                                  |
| --------------------------- | ------------------------------------------------------------------------- |
| **找不到 `dotnet ef` 命令** | `dotnet tool install --global dotnet-ef`                                  |
| **提示找不到 DbContext**    | 检查 `--startup-project` 或 `--context` 是否指定正确                      |
| **跨项目调用失败**          | 使用 `--project` 指定包含迁移的类库项目，`--startup-project` 指定启动项目 |
| **数据库连接不生效**        | 检查 `Program.cs` 中 `UseSqlServer/UseNpgsql` 配置                        |
| **生产环境手动部署**        | 使用 `dotnet ef migrations script` 生成 SQL 并在 DBA 审核后执行           |

### 总结

| 命令                           | 场景              |
| ------------------------------ | ----------------- |
| `dotnet ef migrations add`     | **创建迁移**      |
| `dotnet ef database update`    | **同步数据库**    |
| `dotnet ef migrations list`    | **查看迁移历史**  |
| `dotnet ef dbcontext scaffold` | **逆向工程**      |
| `dotnet ef migrations script`  | **生成 SQL 脚本** |
