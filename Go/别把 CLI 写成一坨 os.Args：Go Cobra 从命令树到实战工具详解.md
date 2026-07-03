### 简介

Go 标准库当然可以写命令行工具：

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	if len(os.Args) < 2 {
		fmt.Println("缺少命令")
		return
	}

	switch os.Args[1] {
	case "version":
		fmt.Println("v1.0.0")
	default:
		fmt.Println("未知命令：", os.Args[1])
	}
}
```

小工具这样写没问题。一旦命令变成这样，代码很快会乱：

```bash
todoctl add "写周报" --priority high --tag work
todoctl list --done=false --limit 20
todoctl done 3
todoctl completion zsh
```

这时需要处理的东西不只是 `os.Args`：

```text
子命令怎么组织？
全局参数和局部参数怎么区分？
必填参数怎么提示？
参数不合法时怎么返回错误？
help、version、completion 怎么生成？
命令层和业务代码怎么分开？
```

Cobra 解决的正是这些问题。

一句话概括：

```text
Cobra 把命令行工具组织成一棵命令树，每个节点负责一类动作，每个动作再配自己的参数、校验和执行逻辑。
```

### Cobra 适合解决什么问题

Cobra 是 Go 里常用的 CLI 框架，包名是：

```text
github.com/spf13/cobra
```

很多命令行程序都会采用类似的结构：

```bash
kubectl get pods
kubectl describe pod nginx
docker image ls
docker container stop app
```

这种结构不是单个命令，而是一棵树：

```text
app
├── user
│   ├── add
│   └── list
├── server
│   ├── start
│   └── stop
└── version
```

Cobra 的核心价值就是把这棵树建起来，并提供配套能力：

* 子命令和多级子命令
* 本地 flag 和持久 flag
* 位置参数校验
* `--help`、`--version`、usage 文案
* `RunE` 错误返回
* 命令执行前后的钩子
* shell 自动补全脚本
* 和 Viper 等配置库集成

### Command、Args、Flag 三个核心概念

Cobra 里最重要的类型是 `cobra.Command`。

```go
var rootCmd = &cobra.Command{
	Use:   "todoctl",
	Short: "一个待办事项命令行工具",
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("hello cobra")
	},
}
```

一个命令主要由三部分组成：

```text
Command：命令本身，比如 add、list、done
Args：位置参数，比如 add 后面的任务标题
Flag：选项参数，比如 --priority high、--done=false
```

位置参数不带 `-`：

```bash
todoctl add "写周报"
```

这里的 `"写周报"` 就是 Args。

flag 带 `-` 或 `--`：

```bash
todoctl add "写周报" --priority high -t work
```

这里的 `--priority` 和 `-t` 就是 Flag。

### 安装方式

项目里只需要 Cobra 库：

```bash
go get github.com/spf13/cobra@latest
```

如果想用脚手架生成目录，也可以安装 `cobra-cli`：

```bash
go install github.com/spf13/cobra-cli@latest
```

脚手架不是必需品。理解 Cobra 本身时，手写一个小项目更清楚。

### 第一个最小示例

先看一个单文件版本：

```go
package main

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
)

func main() {
	rootCmd := &cobra.Command{
		Use:   "hello",
		Short: "第一个 Cobra 程序",
		Run: func(cmd *cobra.Command, args []string) {
			fmt.Println("Hello Cobra")
		},
	}

	if err := rootCmd.Execute(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

运行：

```bash
go run main.go
```

输出：

```text
Hello Cobra
```

查看帮助：

```bash
go run main.go --help
```

Cobra 会根据 `Use`、`Short`、flag、子命令自动生成帮助内容。

### Use、Short、Long、Example 怎么写

`cobra.Command` 里常用的描述字段有四个：

```go
var addCmd = &cobra.Command{
	Use:   "add <title>",
	Short: "新增待办事项",
	Long: `新增一条待办事项。

标题来自位置参数，优先级、标签、完成状态通过 flag 控制。`,
	Example: `  todoctl add "写周报"
  todoctl add "修复支付回调" --priority high --tag backend`,
}
```

这些字段不是摆设，都会出现在 `--help` 里。

常见写法：

```text
Use：写命令名和参数形状，比如 add <title>
Short：一句话说明，出现在命令列表里
Long：较长说明，出现在当前命令 help 里
Example：给出真实命令，减少使用成本
```

### Run 和 RunE 的区别

`Run` 不返回错误：

```go
Run: func(cmd *cobra.Command, args []string) {
	fmt.Println("执行成功")
}
```

`RunE` 返回错误：

```go
RunE: func(cmd *cobra.Command, args []string) error {
	if len(args) == 0 {
		return fmt.Errorf("缺少任务标题")
	}
	return nil
}
```

工程代码更推荐 `RunE`。

原因很简单：命令执行失败时，错误可以统一交给 `Execute` 处理，业务函数里不需要到处 `os.Exit`。

```go
func Execute() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

命令层返回错误，入口层决定退出码，这样结构更干净。

### 本地 Flag 和持久 Flag

Cobra 有两类常见 flag。

本地 flag 只对当前命令生效：

```go
addCmd.Flags().StringVarP(&priority, "priority", "p", "normal", "任务优先级")
```

持久 flag 对当前命令以及所有子命令生效：

```go
rootCmd.PersistentFlags().StringVar(&dataFile, "data", "tasks.json", "数据文件路径")
```

例子：

```bash
todoctl --data ./tmp/tasks.json add "写周报"
todoctl --data ./tmp/tasks.json list
todoctl --data ./tmp/tasks.json done 1
```

`--data` 是全局配置，适合放在 root 的 `PersistentFlags` 里。

`--priority` 只和新增任务有关，适合放在 `add` 命令的 `Flags` 里。

### 参数校验 Args

位置参数校验通过 `Args` 字段完成。

常见内置校验：

```go
Args: cobra.NoArgs
Args: cobra.ExactArgs(1)
Args: cobra.MinimumNArgs(1)
Args: cobra.MaximumNArgs(3)
Args: cobra.RangeArgs(1, 3)
```

新增任务至少需要一个标题：

```go
Args: cobra.MinimumNArgs(1)
```

完成任务必须传一个 ID：

```go
Args: cobra.ExactArgs(1)
```

也可以写自定义校验：

```go
Args: func(cmd *cobra.Command, args []string) error {
	if len(args) != 1 {
		return fmt.Errorf("需要传入任务 ID")
	}
	if _, err := strconv.Atoi(args[0]); err != nil {
		return fmt.Errorf("任务 ID 必须是数字：%s", args[0])
	}
	return nil
}
```

### 实战项目：todoctl 待办事项工具

下面写一个可运行的 `todoctl`。

功能：

```bash
todoctl add "写周报" --priority high --tag work
todoctl add "修复登录 bug" -p high -t backend -t bug
todoctl list
todoctl list --done=false
todoctl done 1
todoctl completion zsh
```

数据保存到本地 JSON 文件，方便观察完整流程。

项目结构：

```text
todoctl/
├── cmd/
│   ├── add.go
│   ├── completion.go
│   ├── done.go
│   ├── list.go
│   └── root.go
├── internal/
│   └── task/
│       └── store.go
├── go.mod
└── main.go
```

初始化项目：

```bash
mkdir todoctl
cd todoctl
go mod init todoctl
go get github.com/spf13/cobra@latest
```

### main.go：程序入口

`main.go` 只做一件事：执行根命令。

```go
package main

import "todoctl/cmd"

func main() {
	cmd.Execute()
}
```

入口保持薄一点，后面扩展命令时不会影响启动逻辑。

### internal/task/store.go：业务和存储

命令层不直接操作 JSON 细节，业务代码放到 `internal/task`。

```go
package task

import (
	"encoding/json"
	"errors"
	"fmt"
	"os"
	"slices"
	"strings"
	"time"
)

type Task struct {
	ID        int       `json:"id"`
	Title     string    `json:"title"`
	Priority  string    `json:"priority"`
	Tags      []string  `json:"tags"`
	Done      bool      `json:"done"`
	CreatedAt time.Time `json:"created_at"`
	DoneAt    time.Time `json:"done_at,omitempty"`
}

type Store struct {
	File string
}

type AddInput struct {
	Title    string
	Priority string
	Tags     []string
}

var allowedPriorities = []string{"low", "normal", "high"}

func (s Store) Load() ([]Task, error) {
	data, err := os.ReadFile(s.File)
	if err != nil {
		if errors.Is(err, os.ErrNotExist) {
			return []Task{}, nil
		}
		return nil, fmt.Errorf("读取任务文件失败：%w", err)
	}

	var tasks []Task
	if err := json.Unmarshal(data, &tasks); err != nil {
		return nil, fmt.Errorf("解析任务文件失败：%w", err)
	}
	return tasks, nil
}

func (s Store) Save(tasks []Task) error {
	data, err := json.MarshalIndent(tasks, "", "  ")
	if err != nil {
		return fmt.Errorf("序列化任务失败：%w", err)
	}

	if err := os.WriteFile(s.File, data, 0o644); err != nil {
		return fmt.Errorf("写入任务文件失败：%w", err)
	}
	return nil
}

func (s Store) Add(input AddInput) (Task, error) {
	title := strings.TrimSpace(input.Title)
	if title == "" {
		return Task{}, fmt.Errorf("任务标题不能为空")
	}

	priority := strings.TrimSpace(input.Priority)
	if priority == "" {
		priority = "normal"
	}
	if !slices.Contains(allowedPriorities, priority) {
		return Task{}, fmt.Errorf("不支持的优先级：%s", priority)
	}

	tasks, err := s.Load()
	if err != nil {
		return Task{}, err
	}

	task := Task{
		ID:        nextID(tasks),
		Title:     title,
		Priority:  priority,
		Tags:      cleanTags(input.Tags),
		CreatedAt: time.Now(),
	}

	tasks = append(tasks, task)
	if err := s.Save(tasks); err != nil {
		return Task{}, err
	}
	return task, nil
}

func (s Store) MarkDone(id int) (Task, error) {
	tasks, err := s.Load()
	if err != nil {
		return Task{}, err
	}

	for i := range tasks {
		if tasks[i].ID == id {
			tasks[i].Done = true
			tasks[i].DoneAt = time.Now()
			if err := s.Save(tasks); err != nil {
				return Task{}, err
			}
			return tasks[i], nil
		}
	}

	return Task{}, fmt.Errorf("任务不存在：%d", id)
}

func nextID(tasks []Task) int {
	id := 1
	for _, task := range tasks {
		if task.ID >= id {
			id = task.ID + 1
		}
	}
	return id
}

func cleanTags(tags []string) []string {
	result := make([]string, 0, len(tags))
	for _, tag := range tags {
		tag = strings.TrimSpace(tag)
		if tag != "" {
			result = append(result, tag)
		}
	}
	return result
}
```

这里有几个细节：

* `Store` 只关心数据文件路径
* `AddInput` 是新增任务的输入结构
* `Load` 遇到文件不存在时返回空切片
* 业务错误用 `fmt.Errorf` 返回给命令层
* 命令层不会知道 JSON 怎么读写

### cmd/root.go：根命令和全局参数

根命令负责挂载全局参数、版本、错误处理策略。

```go
package cmd

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
)

var (
	dataFile string
	verbose  bool
)

var rootCmd = &cobra.Command{
	Use:           "todoctl",
	Short:         "管理本地待办事项",
	Long:          "todoctl 是一个基于 Cobra 的本地待办事项命令行工具。",
	Version:       "v1.0.0",
	SilenceUsage:  true,
	SilenceErrors: true,
	RunE: func(cmd *cobra.Command, args []string) error {
		return cmd.Help()
	},
	PersistentPreRun: func(cmd *cobra.Command, args []string) {
		if verbose {
			fmt.Fprintf(os.Stderr, "data file: %s\n", dataFile)
		}
	},
}

func Execute() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}

func init() {
	rootCmd.PersistentFlags().StringVar(&dataFile, "data", "tasks.json", "数据文件路径")
	rootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "输出调试信息")
}
```

`SilenceUsage` 和 `SilenceErrors` 很实用。

默认情况下，Cobra 可能在错误时自动打印 usage 或错误文本。项目里通常希望自己控制输出，所以根命令里直接关掉自动输出。

```go
SilenceUsage:  true,
SilenceErrors: true,
```

`Version` 会让根命令自动支持：

```bash
todoctl --version
```

### cmd/add.go：新增任务

新增任务用到了位置参数、本地 flag、必填校验、`RunE`。

```go
package cmd

import (
	"fmt"
	"strings"

	"todoctl/internal/task"

	"github.com/spf13/cobra"
)

var (
	addPriority string
	addTags     []string
)

var addCmd = &cobra.Command{
	Use:   "add <title>",
	Short: "新增待办事项",
	Long:  "新增一条待办事项，标题来自位置参数，优先级和标签来自 flag。",
	Example: `  todoctl add "写周报"
  todoctl add "修复登录 bug" --priority high --tag backend --tag bug`,
	Args: cobra.MinimumNArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		store := task.Store{File: dataFile}
		created, err := store.Add(task.AddInput{
			Title:    strings.Join(args, " "),
			Priority: addPriority,
			Tags:     addTags,
		})
		if err != nil {
			return err
		}

		fmt.Printf("已新增 #%d [%s] %s\n", created.ID, created.Priority, created.Title)
		return nil
	},
}

func init() {
	rootCmd.AddCommand(addCmd)

	addCmd.Flags().StringVarP(&addPriority, "priority", "p", "normal", "任务优先级：low、normal、high")
	addCmd.Flags().StringSliceVarP(&addTags, "tag", "t", nil, "任务标签，可多次传入")
}
```

运行：

```bash
go run . add "写周报" --priority high --tag work
```

输出：

```text
已新增 #1 [high] 写周报
```

再添加一条：

```bash
go run . add "修复登录 bug" -p high -t backend -t bug
```

输出：

```text
已新增 #2 [high] 修复登录 bug
```

`StringSliceVarP` 支持逗号和值重复两种写法：

```bash
todoctl add "整理文档" -t doc -t work
todoctl add "整理文档" -t doc,work
```

### cmd/list.go：列表查询

列表命令展示本地 flag 的另一个用法：过滤和分页。

```go
package cmd

import (
	"fmt"
	"os"
	"strings"
	"text/tabwriter"

	"todoctl/internal/task"

	"github.com/spf13/cobra"
)

var (
	listDone     string
	listLimit    int
	listPriority string
)

var listCmd = &cobra.Command{
	Use:   "list",
	Short: "查看待办事项列表",
	Args:  cobra.NoArgs,
	RunE: func(cmd *cobra.Command, args []string) error {
		store := task.Store{File: dataFile}
		tasks, err := store.Load()
		if err != nil {
			return err
		}

		rows := filterTasks(tasks)
		if listLimit > 0 && len(rows) > listLimit {
			rows = rows[:listLimit]
		}
		if len(rows) == 0 {
			fmt.Println("暂无任务")
			return nil
		}

		w := tabwriter.NewWriter(os.Stdout, 0, 0, 2, ' ', 0)
		fmt.Fprintln(w, "ID\t状态\t优先级\t标题\t标签")
		for _, item := range rows {
			status := "待办"
			if item.Done {
				status = "完成"
			}
			fmt.Fprintf(w, "%d\t%s\t%s\t%s\t%s\n",
				item.ID,
				status,
				item.Priority,
				item.Title,
				strings.Join(item.Tags, ","),
			)
		}
		return w.Flush()
	},
}

func init() {
	rootCmd.AddCommand(listCmd)

	listCmd.Flags().StringVar(&listDone, "done", "", "按完成状态过滤：true、false")
	listCmd.Flags().StringVarP(&listPriority, "priority", "p", "", "按优先级过滤")
	listCmd.Flags().IntVarP(&listLimit, "limit", "n", 0, "最多显示条数")
}

func filterTasks(tasks []task.Task) []task.Task {
	rows := make([]task.Task, 0, len(tasks))
	for _, item := range tasks {
		if listDone == "true" && !item.Done {
			continue
		}
		if listDone == "false" && item.Done {
			continue
		}
		if listPriority != "" && item.Priority != listPriority {
			continue
		}
		rows = append(rows, item)
	}
	return rows
}
```

运行：

```bash
go run . list
```

输出类似：

```text
ID  状态  优先级  标题           标签
1   待办  high    写周报         work
2   待办  high    修复登录 bug   backend,bug
```

只看未完成任务：

```bash
go run . list --done=false
```

只看高优先级任务：

```bash
go run . list --priority high
```

### cmd/done.go：完成任务

完成任务需要一个数字 ID。这里用自定义 `Args` 校验，让错误更明确。

```go
package cmd

import (
	"fmt"
	"strconv"

	"todoctl/internal/task"

	"github.com/spf13/cobra"
)

var doneCmd = &cobra.Command{
	Use:   "done <id>",
	Short: "标记任务为完成",
	Args: func(cmd *cobra.Command, args []string) error {
		if len(args) != 1 {
			return fmt.Errorf("需要传入一个任务 ID")
		}
		if _, err := strconv.Atoi(args[0]); err != nil {
			return fmt.Errorf("任务 ID 必须是数字：%s", args[0])
		}
		return nil
	},
	RunE: func(cmd *cobra.Command, args []string) error {
		id, _ := strconv.Atoi(args[0])

		store := task.Store{File: dataFile}
		updated, err := store.MarkDone(id)
		if err != nil {
			return err
		}

		fmt.Printf("已完成 #%d %s\n", updated.ID, updated.Title)
		return nil
	},
}

func init() {
	rootCmd.AddCommand(doneCmd)
}
```

运行：

```bash
go run . done 1
```

输出：

```text
已完成 #1 写周报
```

再次查看：

```bash
go run . list
```

输出类似：

```text
ID  状态  优先级  标题           标签
1   完成  high    写周报         work
2   待办  high    修复登录 bug   backend,bug
```

参数错误时：

```bash
go run . done abc
```

输出：

```text
任务 ID 必须是数字：abc
```

### cmd/completion.go：生成自动补全脚本

Cobra 支持生成 bash、zsh、fish、powershell 补全脚本。

```go
package cmd

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
)

var completionCmd = &cobra.Command{
	Use:   "completion [bash|zsh|fish|powershell]",
	Short: "生成 shell 自动补全脚本",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		switch args[0] {
		case "bash":
			return rootCmd.GenBashCompletion(os.Stdout)
		case "zsh":
			return rootCmd.GenZshCompletion(os.Stdout)
		case "fish":
			return rootCmd.GenFishCompletion(os.Stdout, true)
		case "powershell":
			return rootCmd.GenPowerShellCompletion(os.Stdout)
		default:
			return fmt.Errorf("不支持的 shell：%s", args[0])
		}
	},
}

func init() {
	rootCmd.AddCommand(completionCmd)
}
```

生成 zsh 补全：

```bash
go run . completion zsh > _todoctl
```

实际安装路径和 shell 配置有关，命令本身只负责把脚本输出到标准输出。

### 运行完整 demo

完整项目写好后，可以按下面流程验证：

```bash
go run . --help
go run . --version
go run . add "写周报" --priority high --tag work
go run . add "修复登录 bug" -p high -t backend -t bug
go run . list
go run . done 1
go run . list --done=true
go run . completion bash
```

也可以编译成二进制：

```bash
go build -o todoctl .
./todoctl add "打包发布" -p normal
./todoctl list
```

### 命令树怎么组织

Cobra 命令靠 `AddCommand` 组织父子关系。

```go
rootCmd.AddCommand(addCmd)
rootCmd.AddCommand(listCmd)
rootCmd.AddCommand(doneCmd)
rootCmd.AddCommand(completionCmd)
```

如果有更复杂的业务模块，可以继续嵌套：

```go
rootCmd.AddCommand(userCmd)
userCmd.AddCommand(userCreateCmd)
userCmd.AddCommand(userListCmd)
```

对应命令：

```bash
app user create
app user list
```

父命令通常只做分组，不一定执行具体业务。

```go
var userCmd = &cobra.Command{
	Use:   "user",
	Short: "用户管理",
	RunE: func(cmd *cobra.Command, args []string) error {
		return cmd.Help()
	},
}
```

### flag 读取方式：绑定变量和临时读取

flag 有两种常见读取方式。

第一种是绑定变量：

```go
var port int

serveCmd.Flags().IntVarP(&port, "port", "p", 8080, "监听端口")
```

执行时直接用变量：

```go
fmt.Println(port)
```

第二种是在 `RunE` 里读取：

```go
port, err := cmd.Flags().GetInt("port")
if err != nil {
	return err
}
fmt.Println(port)
```

简单命令常用绑定变量，动态命令或测试场景常用 `GetXxx`。

注意持久 flag 的读取方式：

```go
value, err := cmd.Flags().GetString("data")
```

Cobra 会把继承来的持久 flag 合并到当前命令可见的 flag 集合中，所以子命令里也能这样读取。

### 必填 flag

有些参数适合作为 flag，但必须传。

比如发布命令必须带镜像名：

```go
var image string

deployCmd.Flags().StringVar(&image, "image", "", "镜像名称")
_ = deployCmd.MarkFlagRequired("image")
```

运行：

```bash
app deploy
```

会提示：

```text
required flag(s) "image" not set
```

`MarkFlagRequired` 返回错误，示例里用 `_ =` 是为了简化。实际项目可以在初始化阶段检查这个错误。

### PreRun、Run、PostRun 生命周期

Cobra 支持命令执行前后钩子：

```go
var serveCmd = &cobra.Command{
	Use: "serve",
	PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
		fmt.Println("初始化配置")
		return nil
	},
	PreRunE: func(cmd *cobra.Command, args []string) error {
		fmt.Println("检查端口")
		return nil
	},
	RunE: func(cmd *cobra.Command, args []string) error {
		fmt.Println("启动服务")
		return nil
	},
	PostRunE: func(cmd *cobra.Command, args []string) error {
		fmt.Println("释放资源")
		return nil
	},
}
```

常见用途：

```text
PersistentPreRunE：读取配置、初始化日志、创建客户端
PreRunE：检查当前命令自己的前置条件
RunE：执行主逻辑
PostRunE：清理临时资源、输出统计信息
```

不要把耗时初始化放在 `init()` 里。`init()` 适合注册命令和 flag，真正需要执行的动作放到钩子或 `RunE`。

### 错误处理怎么放

推荐结构：

```go
RunE: func(cmd *cobra.Command, args []string) error {
	result, err := service.DoSomething()
	if err != nil {
		return err
	}
	fmt.Println(result)
	return nil
}
```

不推荐在每个命令里直接退出：

```go
Run: func(cmd *cobra.Command, args []string) {
	if err := service.DoSomething(); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}
```

原因：

```text
RunE 方便测试
RunE 方便统一错误输出
RunE 方便上层决定退出码
RunE 避免业务层直接调用 os.Exit
```

入口层统一处理：

```go
func Execute() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

### 命令层不要塞太多业务

命令层适合做这些事：

```text
解析 flag
校验 Args
组装输入结构
调用 service
把结果打印成 CLI 输出
```

业务层适合做这些事：

```text
读写文件
访问数据库
调用 HTTP API
执行领域规则
返回结构化结果和错误
```

对比一下：

```go
RunE: func(cmd *cobra.Command, args []string) error {
	store := task.Store{File: dataFile}
	created, err := store.Add(task.AddInput{
		Title:    strings.Join(args, " "),
		Priority: addPriority,
		Tags:     addTags,
	})
	if err != nil {
		return err
	}
	fmt.Printf("已新增 #%d [%s] %s\n", created.ID, created.Priority, created.Title)
	return nil
}
```

这段命令代码只负责“把命令行输入转成业务输入”，没有直接读写 JSON。

### Cobra 和 Viper 怎么搭配

Cobra 负责命令和参数，Viper 负责配置。

典型场景：

```text
flag：--port 8080
环境变量：APP_PORT=8080
配置文件：config.yaml
默认值：8080
```

简化示例：

```go
package main

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
	"github.com/spf13/viper"
)

var cfgFile string

func main() {
	rootCmd := &cobra.Command{
		Use:          "server",
		SilenceUsage: true,
		PreRunE: func(cmd *cobra.Command, args []string) error {
			if cfgFile != "" {
				viper.SetConfigFile(cfgFile)
			} else {
				viper.SetConfigName("config")
				viper.SetConfigType("yaml")
				viper.AddConfigPath(".")
			}
			viper.SetEnvPrefix("APP")
			viper.AutomaticEnv()
			return viper.ReadInConfig()
		},
		Run: func(cmd *cobra.Command, args []string) {
			fmt.Println("port:", viper.GetInt("port"))
		},
	}

	rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "配置文件路径")
	rootCmd.Flags().Int("port", 8080, "监听端口")
	_ = viper.BindPFlag("port", rootCmd.Flags().Lookup("port"))

	if err := rootCmd.Execute(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

配置文件：

```yaml
port: 9090
```

运行：

```bash
go run . --config config.yaml
```

Viper 不是 Cobra 必需组件。只有配置来源变多时，才有必要引入。

### 测试 Cobra 命令

Cobra 命令可以直接测试。关键是给命令设置参数和输出缓冲区。

```go
package cmd

import (
	"bytes"
	"testing"

	"github.com/spf13/cobra"
)

func TestVersionCommand(t *testing.T) {
	cmd := &cobra.Command{
		Use:     "app",
		Version: "v1.0.0",
	}

	var out bytes.Buffer
	cmd.SetOut(&out)
	cmd.SetErr(&out)
	cmd.SetArgs([]string{"--version"})

	if err := cmd.Execute(); err != nil {
		t.Fatal(err)
	}
	if got := out.String(); got != "app version v1.0.0\n" {
		t.Fatalf("unexpected output: %q", got)
	}
}
```

业务命令测试时，可以把数据文件指向临时目录：

```go
dataFile = t.TempDir() + "/tasks.json"
```

这样测试不会污染本地文件。

### 常见错误一：把 flag 和 args 混在一起

这个命令：

```bash
todoctl add "写周报" --priority high
```

拆开看：

```text
add：命令
"写周报"：位置参数 args
--priority high：flag
```

标题这种核心对象通常适合做位置参数。

筛选、开关、配置这类修饰信息通常适合做 flag。

### 常见错误二：所有命令共用一堆全局变量

可以用全局变量绑定 flag，但不要把所有业务状态都堆进去。

适合全局变量的内容：

```text
flag 绑定变量
rootCmd、addCmd 等命令对象
版本号
```

不适合全局变量的内容：

```text
数据库连接
请求上下文
中间计算结果
业务缓存
```

后者更适合放在 service、client、store 里，在 `RunE` 或钩子里创建和传递。

### 常见错误三：父命令没有行为

父命令只做分组时，直接运行可能没有输出。

更友好的做法是显示帮助：

```go
var userCmd = &cobra.Command{
	Use:   "user",
	Short: "用户管理",
	RunE: func(cmd *cobra.Command, args []string) error {
		return cmd.Help()
	},
}
```

这样执行：

```bash
app user
```

会看到 `user` 下面有哪些子命令。

### 常见错误四：在 init 里做真实业务

`init()` 适合做注册：

```go
func init() {
	rootCmd.AddCommand(addCmd)
	addCmd.Flags().StringVarP(&addPriority, "priority", "p", "normal", "任务优先级")
}
```

不适合做这些事：

```text
读取配置文件
连接数据库
请求远程接口
创建大对象缓存
```

这些动作会在包加载时发生，测试和命令补全都可能被拖慢。更合适的位置是 `PersistentPreRunE`、`PreRunE` 或 `RunE`。

### 常见错误五：错误时打印一大段 usage

参数错了就打印完整帮助，很容易淹没真正的错误。

根命令可以这样设置：

```go
SilenceUsage:  true,
SilenceErrors: true,
```

然后统一输出错误：

```go
if err := rootCmd.Execute(); err != nil {
	fmt.Fprintln(os.Stderr, err)
	os.Exit(1)
}
```

需要帮助时，使用者主动执行：

```bash
todoctl add --help
```

### 工程实践建议

Cobra 项目可以按这个思路组织：

```text
cmd/
  root.go          根命令、全局 flag、版本、Execute
  add.go           add 子命令
  list.go          list 子命令
  completion.go    补全命令
internal/
  service/         业务服务
  repository/      数据访问
  config/          配置加载
main.go            程序入口
```

几个实用原则：

* 命令文件按子命令拆分
* `cmd` 包只做命令行适配
* 复杂逻辑放到 `internal`
* 业务函数返回 `error`，命令层用 `RunE` 传递
* 全局配置放 `PersistentFlags`
* 单命令配置放 `Flags`
* 位置参数用 `Args` 明确校验
* 父命令只分组时返回 `cmd.Help()`
* 需要配置文件时再引入 Viper

### 什么时候不需要 Cobra

不是所有命令行程序都需要 Cobra。

下面这类场景，用标准库可能更轻：

```text
只有一两个参数
没有子命令
不会长期维护
只是一次性脚本
```

标准库 `flag` 足够处理简单参数。

Cobra 更适合这些场景：

```text
有多个子命令
需要清晰 help
需要补全脚本
需要版本命令
需要复杂参数校验
命令行工具会长期扩展
```

### 总结

Cobra 的重点不是“省几行参数解析代码”，而是让 CLI 程序有清晰结构。

可以把它理解成三层：

```text
命令树：root、子命令、多级子命令
输入层：args、flags、校验、help、completion
执行层：RunE、钩子、错误返回、业务调用
```

写小工具时，`os.Args` 或标准库 `flag` 很顺手。

写可维护的 CLI 应用时，Cobra 更像一个骨架：命令怎么挂、参数怎么收、错误怎么返回、帮助怎么生成，都有固定位置。

真正好维护的 Cobra 项目，一般不是命令写得多炫，而是边界清楚：

```text
cmd 负责命令行输入输出
internal 负责真实业务
RunE 负责把两边接起来
```

抓住这条线，Cobra 就不会变成另一坨复杂代码。
