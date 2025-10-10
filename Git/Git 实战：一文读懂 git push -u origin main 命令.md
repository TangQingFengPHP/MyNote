### 简介

命令的完整语法结构

```shell
git push -u origin main
```

其实等价于：

```shell
git push --set-upstream origin main
```

分为三个部分：

| 部分       | 含义                                                    |
| ---------- | ------------------------------------------------------- |
| `git push` | 推送（push）本地提交到远程仓库                          |
| `origin`   | 远程仓库名称（默认是 `origin`，指克隆时的默认远程）     |
| `main`     | 要推送的本地分支（同时也是远程分支名）现代 Git 默认分支为 main，早期为 master                  |
| `-u`       | 建立“跟踪关系（upstream）”，使以后的 push/pull 简化命令 |

### 执行时到底发生了什么？

假设你本地有一个分支 `main`，远程仓库也叫 `origin`。

执行：

```shell
git push -u origin main
```

输出示例：

```text
枚举对象中: 5, 完成.
对象计数中: 100% (5/5), 完成.
使用 4 个线程进行压缩
压缩对象中: 100% (3/3), 完成.
写入对象中: 100% (5/5), 536 bytes | 536.00 KiB/s, 完成.
总共 5（差异 0），复用 0（差异 0）
To https://github.com/user/repo.git
 * [new branch]      main -> main
分支 'main' 设置为跟踪来自 'origin' 的远程分支 'main'。
```

设置后的状态

```shell
# 再次查看分支信息
git branch -vv
# * main 123abcd [origin/main] 初始提交
```

Git 内部会做以下几件事：

1. 推送提交（ `push commits` ）

    * 把本地 `main` 分支中的提交对象（`commit`）上传到远程仓库。

    * 如果远程仓库还没有 `main` 分支，则会新建一个 `main` 分支。

2. 推送分支引用（`refs update`）

    * 远程仓库的 `refs/heads/main` 指向你推送的最新 `commit`。

3. 建立 `upstream` 跟踪关系

    * `Git` 会在本地的配置文件 `.git/config` 中写入类似内容：

    ```ini
        [branch "main"]
        remote = origin
        merge = refs/heads/main
    ```

    * 这表示：

        * 你的本地 `main` 分支的上游（upstream）是远程的 `origin/main`。

        * 以后可以直接用 `git pull` 或 `git push`（无需再写远程名和分支名）。

### 底层原理

#### 检查远程仓库：

* `Git` 通过 `git remote` 配置（`.git/config` 中的 `[remote "origin"]`）获取远程 URL（如 `https://github.com/user/repo.git`）。

* 如果未添加远程，使用 `git remote add` 先添加。

#### 打包和传输对象：

* `Git` 扫描本地分支 `main` 的提交历史，计算需要推送的对象（`commits、trees、blobs`）。

* 使用 `pack` 协议打包对象（压缩后传输），避免传输冗余数据（通过内容寻址的 `SHA-1` 哈希去重）。

* 传输到远程仓库的 `refs/heads/main`（远程分支引用）。

#### 更新远程引用：

* 远程仓库更新其 `main` 分支指针，指向最新提交。

* 如果远程分支不存在，创建新分支。

#### 设置上游跟踪：

* `-u` 选项修改本地 `.git/config`，添加 `[branch "main"]` 部分：

    * `remote = origin`：指定远程仓库。

    * `merge = refs/heads/main`：指定远程分支。

* 这允许 `Git` 在后续操作中自动推断上游（如 `git status` 显示“ahead of origin/main by 2 commits”）。

#### 协议与认证：

* 默认使用 `HTTPS` 协议（需输入用户名/密码或 token）；`SSH` 协议（`git@github.com:user/repo.git`）使用密钥认证。

* 推送后，`Git` 更新本地 `refs/remotes/origin/main`（远程分支的本地镜像）。

### 后续操作的简化效果

#### 首次推送（需要 -u）

```shell
git push -u origin main
```

建立跟踪关系。

以后推送就可以直接写：

```shell
git push
```

`Git` 会自动知道推向 `origin/main`。

拉取也可以直接写：

```shell
git pull
```

`Git` 会自动从 `origin/main` 拉取。

### 常见场景说明

| 场景                       | 命令                                | 说明                                |
| -------------------------- | ----------------------------------- | ----------------------------------- |
| **第一次推送新建分支**     | `git push -u origin dev`            | 如果远程没有 `dev` 分支，会自动创建 |
| **已建立跟踪关系后推送**   | `git push`                          | 自动推送到上次的远程分支            |
| **修改默认远程分支**       | `git branch -u origin/other-branch` | 改变当前分支的 upstream             |
| **查看当前分支的跟踪关系** | `git branch -vv`                    | 显示每个分支的上游分支及状态        |
| **取消跟踪关系**           | `git branch --unset-upstream`       | 移除当前分支的 upstream 设置        |

### 实际应用场景

#### 场景 1：新仓库首次推送

```shell
# 初始化新项目
mkdir my-project
cd my-project
git init
git add .
git commit -m "Initial commit"

# 添加远程仓库
git remote add origin https://github.com/user/repo.git

# 推送并设置跟踪（关键步骤！）
git push -u origin main
```

#### 场景 2：创建新分支并推送

```shell
# 创建并切换到新功能分支
git checkout -b feature/new-feature

# 进行一些开发工作...
git add .
git commit -m "Add new feature"

# 推送新分支并设置上游
git push -u origin feature/new-feature
```

#### 场景 3：修复分支上游设置

如果分支已经存在但没有设置上游：

```shell
# 查看当前分支状态
git status
# 分支基于 'origin/main'，但上游信息已丢失。

# 修复上游设置
git push -u origin current-branch-name
```

### 相关命令和替代写法

#### 查看配置验证

执行：

```shell
git branch -vv
```

输出示例：

```css
* main  a1b2c3d [origin/main] Initial commit
```

说明当前分支 `main` 跟踪远程分支 `origin/main`。

或者查看配置：

```shell
git config --local -l
```

可以看到类似：

```ini
branch.main.remote=origin
branch.main.merge=refs/heads/main
```

#### 手动设置上游分支（不推送）

```shell
# 设置现有分支的上游
git branch --set-upstream-to=origin/main main
```

#### 不同的分支名称

```shell
# 如果默认分支是 master
git push -u origin master

# 推送其他分支
git push -u origin develop
git push -u origin feature/login
```


### 与其他参数的区别与组合

| 命令                                  | 作用                                             |
| ------------------------------------- | ------------------------------------------------ |
| `git push origin main`                | 只推送，不建立跟踪关系（常用于已存在的远程分支） |
| `git push -u origin main`             | 推送并建立跟踪关系（推荐首次推送使用）           |
| `git push --set-upstream origin main` | 与 `-u` 等价，更易理解                           |
| `git push -f origin main`             | 强制推送（慎用，会覆盖远程提交）                 |
| `git push origin main:remote-name`    | 推送本地 main 到远程的 remote-name 分支          |

### 注意事项与常见坑

#### 忘记加 -u 并不是问题

后续可以手动补：

```shell
git branch -u origin/main
```

或者再次使用 `git push -u origin main`

#### 不同远程仓库可以存在多个（不仅是 origin）

例如：

```shell
git remote add backup https://github.com/xxx/backup.git
git push -u backup main
```

则会在本地配置 `branch.main.remote=backup`

#### 强制推送谨慎使用

* 如果使用 `-f` 或 `--force`，会覆盖远程历史

* 建议使用：

```shell
git push --force-with-lease
```

这样会在远程没有被别人改动的情况下才强制。

#### 远程分支删除

```shell
git push origin --delete branch_name
```