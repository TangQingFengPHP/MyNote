### 简介

Linux 上提到定时任务，最先想到的通常是 `cron`。

`cron` 足够简单，也足够稳定，但任务一旦涉及日志、启动依赖、超时控制、错过后补跑、运行用户和资源限制，单独一行 crontab 很快就会变得难以维护。

`systemd Timer` 提供了另一套方案：

```text
.timer 负责决定什么时候执行
.service 负责决定执行什么、以什么方式执行
```

例如，每天凌晨备份一次应用数据，可以拆成两个单元：

```text
myapp-backup.timer
        |
        | 到达触发时间
        v
myapp-backup.service
        |
        | 启动备份脚本
        v
/usr/local/sbin/myapp-backup.sh
```

这样不仅能定时执行，还能直接使用 `systemctl` 查看状态，使用 `journalctl` 查看日志，并继续使用 systemd 的权限、依赖、超时和资源控制能力。

一句话概括：

```text
systemd Timer 不是把 cron 表达式换一种写法，而是把定时任务纳入 systemd 的服务管理体系。
```

### systemd Timer 是什么？

Timer 是 systemd 的一种单元，文件名以 `.timer` 结尾。

Timer 本身通常不运行脚本。触发时间到达后，Timer 会激活另一个单元，最常见的目标是 `.service`。

假设存在下面两个文件：

```text
/etc/systemd/system/report.service
/etc/systemd/system/report.timer
```

当 `report.timer` 没有显式配置 `Unit=` 时，默认会触发同名的 `report.service`。

也可以明确指定其他服务：

```ini
[Timer]
OnCalendar=daily
Unit=generate-report.service
```

所以，“Timer 和 Service 必须同名”并不准确。

准确说法是：

```text
没有配置 Unit= 时，foo.timer 默认触发 foo.service。
```

同名是最省事、也最容易维护的做法，但不是硬性要求。

### Timer 和 cron 有什么区别？

| 对比项 | cron | systemd Timer |
| --- | --- | --- |
| 配置方式 | 一行 crontab | `.timer` 和 `.service` 单元 |
| 日历定时 | 支持 | 支持，语法更丰富 |
| 开机后延时 | 通常需要额外处理 | `OnBootSec=` |
| 按间隔循环 | 支持有限 | 多种单调时间触发器 |
| 关机期间错过后补跑 | 默认不支持 | `Persistent=true` |
| 随机延迟 | 通常需要 `sleep` | `RandomizedDelaySec=` |
| 日志 | 邮件、文件重定向或 syslog | 直接进入 journal |
| 超时和权限控制 | 需要脚本处理 | 交给 `.service` |
| 服务依赖 | 较弱 | 使用 systemd 依赖关系 |
| 查看下次执行时间 | 不够直观 | `systemctl list-timers` |

systemd Timer 也不是任何场景都比 cron 合适。

临时写一个简单的个人任务，cron 配置更短；服务器已经由 systemd 管理，并且任务需要日志、补跑、超时或依赖控制时，Timer 更容易维护。

### 最小 Demo：每分钟输出一次当前时间

先用一个最小示例跑通完整流程。

这个示例不写脚本，也不使用额外配置，只做一件事：

```text
每分钟执行一次 date 命令，并把输出写入 systemd 日志。
```

#### 创建 Service

创建 `/etc/systemd/system/show-time.service`：

```bash
sudo vim /etc/systemd/system/show-time.service
```

内容如下：

```ini
[Unit]
Description=Show current time

[Service]
Type=oneshot
ExecStart=/usr/bin/date
```

这里只需要关注两个配置：

* `Type=oneshot`：命令执行完成后，服务结束
* `ExecStart=/usr/bin/date`：指定需要执行的命令

#### 创建 Timer

创建 `/etc/systemd/system/show-time.timer`：

```bash
sudo vim /etc/systemd/system/show-time.timer
```

内容如下：

```ini
[Unit]
Description=Run show-time every minute

[Timer]
OnCalendar=minutely

[Install]
WantedBy=timers.target
```

`OnCalendar=minutely` 表示每分钟触发一次。

Timer 和 Service 都叫 `show-time`，所以 `show-time.timer` 会自动触发 `show-time.service`。

#### 启动 Timer

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now show-time.timer
```

查看下一次执行时间：

```bash
systemctl list-timers show-time.timer
```

不想等待下一分钟，可以手动执行一次 Service：

```bash
sudo systemctl start show-time.service
```

查看输出：

```bash
journalctl -u show-time.service -n 10 --no-pager
```

完整流程只有三步：

```text
创建 .service
创建 .timer
启动 .timer
```

理解这个最小示例后，再继续看各种时间规则和进阶配置。

### 两类时间：日历时间和单调时间

Timer 的时间规则主要分成两类。

#### 日历时间

日历时间关注钟表上的日期和时间，例如：

```text
每天 02:30
每周一 09:00
每月 1 日 00:00
工作日每小时一次
```

对应配置项是 `OnCalendar=`：

```ini
[Timer]
OnCalendar=*-*-* 02:30:00
```

这种方式和 cron 最接近，会受到系统日期、时区和夏令时影响。

#### 单调时间

单调时间不关心当前是几月几日，而是关心某个事件过去了多久，例如：

```text
开机 5 分钟后执行
Timer 启动 30 秒后执行
服务上次启动 1 小时后再次执行
服务上次结束 10 分钟后再次执行
```

常见配置如下：

```ini
[Timer]
OnBootSec=5min
OnUnitInactiveSec=1h
```

单调时钟不会受到手动修改系统时间的影响，更适合固定间隔任务。

### 常用配置：`.service` 文件负责什么？

定时任务真正执行的命令写在 `.service` 中。

一个最小的 Service 如下：

```ini
[Unit]
Description=Generate application report

[Service]
Type=oneshot
ExecStart=/usr/local/bin/generate-report.sh
```

`Type=oneshot` 表示启动命令执行完成后，服务就结束。备份、清理、同步和报表生成等一次性任务通常使用这个类型。

常用配置如下：

| 配置 | 作用 |
| --- | --- |
| `User=` | 指定任务使用哪个用户运行 |
| `Group=` | 指定运行组 |
| `WorkingDirectory=` | 指定工作目录 |
| `Environment=` | 设置环境变量 |
| `EnvironmentFile=` | 从文件读取环境变量 |
| `ExecStart=` | 指定执行命令 |
| `TimeoutStartSec=` | 限制任务最长执行时间 |
| `StandardOutput=journal` | 把标准输出写入 journal |
| `StandardError=journal` | 把标准错误写入 journal |

`ExecStart=` 不是交互式 Shell，下面这些内容不能想当然地直接使用：

```text
~
$HOME
*.log
命令1 | 命令2
命令 > 文件
```

管道、重定向、变量展开等 Shell 语法需要明确调用 Shell：

```ini
ExecStart=/bin/bash -c 'date >> /var/log/task.log'
```

复杂逻辑更适合放进独立脚本。单元文件只保留启动参数，排错和复用都会更简单。

### 常用配置：`.timer` 文件负责什么？

一个常见的 Timer 如下：

```ini
[Unit]
Description=Run application report every day

[Timer]
OnCalendar=*-*-* 02:30:00

[Install]
WantedBy=timers.target
```

各部分作用如下：

* `[Unit]` 保存描述、依赖等通用配置
* `[Timer]` 保存触发时间和 Timer 行为
* `[Install]` 决定执行 `systemctl enable` 时建立什么启动关系

`WantedBy=timers.target` 表示启用后，系统进入正常运行状态时会拉起这个 Timer。

注意：

```text
enable 负责设置开机自动启动
start 负责当前立即启动 Timer
```

因此最常见的命令是：

```bash
sudo systemctl enable --now report.timer
```

`--now` 会立即启动 Timer，但不会无条件立即执行 Service。Service 是否马上执行，仍由 Timer 的时间规则决定。

### OnCalendar 日历语法

`OnCalendar=` 的完整形式可以写成：

```text
星期 年-月-日 时:分:秒
```

没有用到的部分可以省略，常见关键字也可以直接使用。

| 配置 | 含义 |
| --- | --- |
| `OnCalendar=minutely` | 每分钟一次 |
| `OnCalendar=hourly` | 每小时一次 |
| `OnCalendar=daily` | 每天 00:00 |
| `OnCalendar=weekly` | 每周一 00:00 |
| `OnCalendar=monthly` | 每月 1 日 00:00 |
| `OnCalendar=yearly` | 每年 1 月 1 日 00:00 |
| `OnCalendar=*-*-* 02:30:00` | 每天 02:30 |
| `OnCalendar=Mon *-*-* 09:00:00` | 每周一 09:00 |
| `OnCalendar=Mon..Fri *-*-* 09:00:00` | 周一到周五 09:00 |
| `OnCalendar=*-*-01 00:00:00` | 每月 1 日 00:00 |
| `OnCalendar=*-01-01 00:00:00` | 每年 1 月 1 日 00:00 |
| `OnCalendar=*-*-* 09,15,21:00:00` | 每天 9、15、21 点 |
| `OnCalendar=*:0/5` | 每 5 分钟一次 |

Timer 还支持在表达式末尾指定时区：

```ini
OnCalendar=Mon..Fri *-*-* 09:00:00 Asia/Shanghai
```

日历表达式容易在通配符、步进值上写错。不要靠肉眼猜，直接让 systemd 解析：

```bash
systemd-analyze calendar '*:0/5'
```

输出中会包含规范化后的表达式、下一次触发时间以及距离触发还有多久。

查看连续多次触发时间：

```bash
systemd-analyze calendar --iterations=5 'Mon..Fri *-*-* 09:00:00'
```

这一步特别适合检查跨天、跨月和工作日规则。

### 单调时间触发器

常见的单调时间配置如下：

| 配置 | 起算点 |
| --- | --- |
| `OnActiveSec=30s` | Timer 被激活后 30 秒 |
| `OnBootSec=5min` | 系统启动后 5 分钟 |
| `OnStartupSec=5min` | systemd 服务管理器启动后 5 分钟 |
| `OnUnitActiveSec=1h` | 目标单元上次被激活后 1 小时 |
| `OnUnitInactiveSec=10min` | 目标单元上次进入 inactive 状态后 10 分钟 |

`OnUnitActiveSec=` 和 `OnUnitInactiveSec=` 经常被混淆。

假设任务每次运行需要 8 分钟：

```ini
OnUnitActiveSec=1h
```

间隔从服务开始运行时计算，理论上的两次启动时间相隔 1 小时。

```ini
OnUnitInactiveSec=1h
```

间隔从服务运行结束、进入 inactive 状态时计算，上一轮结束 1 小时后才会开始下一轮。

对于“每轮完成后休息一段时间”的采集或同步任务，`OnUnitInactiveSec=` 更符合直觉。

开机后先执行一次，之后每轮结束 30 分钟再执行，可以这样写：

```ini
[Timer]
OnBootSec=2min
OnUnitInactiveSec=30min
```

同一个 Timer 中可以配置多个触发器。多个触发器采用“任意一个到期就触发”的关系，不需要全部满足。

### 进阶配置：Persistent 错过任务后补跑

服务器每天 02:30 备份，但 02:00 到 04:00 处于关机状态，普通日历任务会错过这次执行。

加入下面的配置后，Timer 再次启动时会检查上次触发记录：

```ini
[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
```

如果发现关机期间至少错过了一次，会尽快补执行一次。

这里有两个限制：

* `Persistent=true` 只对 `OnCalendar=` 这类日历触发器生效
* 错过多次不会按次数全部补齐，只会触发一次

备份、账单生成和证书检查等任务通常适合开启；高频监控和临时状态采集未必需要补跑。

### 进阶配置：AccuracySec 精度不是越高越好

Timer 默认允许 systemd 在一定精度窗口内合并唤醒，以减少不必要的 CPU 唤醒和能耗。

```ini
AccuracySec=1min
```

这并不表示任务每分钟执行一次，而是表示触发时间允许在 1 分钟精度窗口内被安排。

确实需要接近指定秒执行时，可以缩小精度：

```ini
AccuracySec=1s
```

普通备份、清理任务没有必要追求毫秒或微秒精度。系统繁忙、服务依赖未满足或目标服务仍在运行时，即使精度设置很高，也不能保证程序准点开始执行。

### 进阶配置：RandomizedDelaySec 给任务加一点随机延迟

多台服务器都在整点备份，可能同时请求数据库、对象存储或监控接口，瞬间形成流量尖峰。

下面的配置会在原定时间后增加一个 0 到 15 分钟的随机延迟：

```ini
RandomizedDelaySec=15min
```

这个参数适合：

* 集群批量备份
* 软件更新检查
* 日志上传
* 监控数据上报
* 定时调用外部接口

随机延迟不是执行间隔。每天 02:30 的任务设置 15 分钟随机延迟后，实际启动时间会落在大约 02:30 到 02:45 之间。

### 一个容易忽略的规则：服务还在运行时不会再启动一份

假设 Timer 每 5 分钟触发一次，但任务每次需要 8 分钟。

到达下一次触发时间时，对应 Service 仍是 active 状态，systemd 不会再并发启动一个相同 Service，也不会为每次触发排队补跑。

这能避免同一个 oneshot 服务重叠运行，但也意味着高频触发可能被跳过。

需要严格消费每个时间点的任务时，应该把任务写入消息队列或任务表，再由常驻 Worker 消费，不能把 Timer 当成可靠队列。

### 完整 Demo：每天备份应用数据

下面搭建一个可直接运行的备份任务：

```text
每天 02:30 触发
最多随机延迟 15 分钟
关机错过后补跑一次
把 /srv/myapp/data 打包到 /var/backups/myapp
清理约 7 天前的旧备份
任务超过 30 分钟自动终止
执行日志进入 journal
```

#### 准备测试目录

```bash
sudo mkdir -p /srv/myapp/data
echo 'systemd timer demo' | sudo tee /srv/myapp/data/demo.txt
sudo install -d -m 0700 /var/backups/myapp
```

#### 编写备份脚本

创建 `/usr/local/sbin/myapp-backup.sh`：

```bash
sudo vim /usr/local/sbin/myapp-backup.sh
```

脚本内容如下：

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
umask 077

SOURCE_DIR="${SOURCE_DIR:-/srv/myapp/data}"
BACKUP_DIR="${BACKUP_DIR:-/var/backups/myapp}"
KEEP_DAYS="${KEEP_DAYS:-7}"

if [[ ! -d "$SOURCE_DIR" ]]; then
    echo "source directory does not exist: $SOURCE_DIR" >&2
    exit 1
fi

install -d -m 0700 "$BACKUP_DIR"

timestamp="$(date '+%Y%m%d-%H%M%S')"
archive="$BACKUP_DIR/myapp-$timestamp.tar.gz"
temp_file="$(mktemp "$BACKUP_DIR/.myapp-XXXXXX.tar.gz")"

cleanup() {
    rm -f "$temp_file"
}
trap cleanup EXIT

tar -C "$SOURCE_DIR" -czf "$temp_file" .
mv "$temp_file" "$archive"
trap - EXIT

find "$BACKUP_DIR" \
    -maxdepth 1 \
    -type f \
    -name 'myapp-*.tar.gz' \
    -mtime "+$KEEP_DAYS" \
    -delete

echo "backup completed: $archive"
```

增加执行权限：

```bash
sudo chmod 0755 /usr/local/sbin/myapp-backup.sh
```

脚本先写临时文件，压缩成功后再通过 `mv` 改成正式文件名。压缩中途失败时，不会留下一个看起来正常、实际已损坏的正式备份。

#### 创建 Service

创建 `/etc/systemd/system/myapp-backup.service`：

```bash
sudo vim /etc/systemd/system/myapp-backup.service
```

内容如下：

```ini
[Unit]
Description=Back up MyApp data
Documentation=man:systemd.service(5)
ConditionPathIsDirectory=/srv/myapp/data

[Service]
Type=oneshot
Environment=SOURCE_DIR=/srv/myapp/data
Environment=BACKUP_DIR=/var/backups/myapp
Environment=KEEP_DAYS=7
ExecStart=/usr/local/sbin/myapp-backup.sh
TimeoutStartSec=30min

User=root
Group=root
UMask=0077
Nice=10
IOSchedulingClass=best-effort
IOSchedulingPriority=7

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/backups/myapp

StandardOutput=journal
StandardError=journal
```

这里的安全配置把系统目录设为只读，只允许任务写入 `/var/backups/myapp`。源目录仍然可以读取。

如果实际数据位于 `/home` 下，`ProtectHome=true` 会阻止访问，需要删除这一项或按实际目录调整。

#### 创建 Timer

创建 `/etc/systemd/system/myapp-backup.timer`：

```bash
sudo vim /etc/systemd/system/myapp-backup.timer
```

内容如下：

```ini
[Unit]
Description=Run MyApp backup every day
Documentation=man:systemd.timer(5)

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
RandomizedDelaySec=15min
AccuracySec=1min

[Install]
WantedBy=timers.target
```

文件名都是 `myapp-backup`，因此不需要额外配置：

```ini
Unit=myapp-backup.service
```

#### 校验配置

先校验单元文件：

```bash
sudo systemd-analyze verify \
    /etc/systemd/system/myapp-backup.service \
    /etc/systemd/system/myapp-backup.timer
```

再校验日历表达式：

```bash
systemd-analyze calendar --iterations=3 '*-*-* 02:30:00'
```

没有错误后，让 systemd 重新读取单元文件：

```bash
sudo systemctl daemon-reload
```

#### 先手动测试 Service

不要等到凌晨才判断脚本能不能运行。直接手动启动一次 Service：

```bash
sudo systemctl start myapp-backup.service
```

查看执行结果：

```bash
systemctl status myapp-backup.service
sudo journalctl -u myapp-backup.service -n 50 --no-pager
sudo ls -lh /var/backups/myapp
```

oneshot 任务成功执行后变成 `inactive (dead)` 属于正常现象，不表示任务失败。重点查看 `Result=success`、进程退出码和 journal 日志。

#### 启用并启动 Timer

```bash
sudo systemctl enable --now myapp-backup.timer
```

查看 Timer：

```bash
systemctl status myapp-backup.timer
systemctl list-timers myapp-backup.timer
```

`list-timers` 常见列含义如下：

| 列 | 含义 |
| --- | --- |
| `NEXT` | 预计下次触发时间 |
| `LEFT` | 距离下次触发还有多久 |
| `LAST` | 上次触发时间 |
| `PASSED` | 距离上次触发过去了多久 |
| `UNIT` | Timer 单元 |
| `ACTIVATES` | 被触发的目标单元 |

### 常用管理命令

#### 查看所有 Timer

```bash
systemctl list-timers
systemctl list-timers --all
```

`--all` 会把当前未激活的 Timer 也列出来。

#### 启动、停止和重启

```bash
sudo systemctl start myapp-backup.timer
sudo systemctl stop myapp-backup.timer
sudo systemctl restart myapp-backup.timer
```

#### 设置或取消开机启动

```bash
sudo systemctl enable myapp-backup.timer
sudo systemctl disable myapp-backup.timer
```

停止当前 Timer 并取消开机启动：

```bash
sudo systemctl disable --now myapp-backup.timer
```

#### 手动执行一次任务

```bash
sudo systemctl start myapp-backup.service
```

启动 `.service` 是立即执行一次，启动 `.timer` 是开始等待触发时间，两者不要混淆。

#### 查看实际生效的配置

```bash
systemctl cat myapp-backup.timer
systemctl cat myapp-backup.service
```

查看 systemd 解析后的属性：

```bash
systemctl show myapp-backup.timer
systemctl show myapp-backup.service
```

#### 查看日志

```bash
sudo journalctl -u myapp-backup.service
sudo journalctl -u myapp-backup.service --since today
sudo journalctl -u myapp-backup.service -f
```

Timer 日志主要记录触发和状态变化：

```bash
sudo journalctl -u myapp-backup.timer
```

业务脚本的标准输出和报错通常在 Service 日志中，所以排错时应该优先查看 `.service`。

### 修改配置后怎样生效？

修改 `.service` 或 `.timer` 后，需要重新加载单元文件：

```bash
sudo systemctl daemon-reload
```

修改 Timer 时间规则后，再重启 Timer：

```bash
sudo systemctl restart myapp-backup.timer
```

如果只修改了外部脚本内容，通常不需要 `daemon-reload`，下次执行时会直接读取新脚本。

### 网络任务怎样配置？

备份上传、接口调用等任务可能依赖网络，可以在 Service 中声明：

```ini
[Unit]
Wants=network-online.target
After=network-online.target
```

`After=` 只表示启动顺序，不能保证外部网站一定可访问；`Wants=` 会尝试拉起网络在线目标，但最终仍需要脚本处理 DNS 失败、超时和重试。

Timer 只负责触发，不会因为 Service 返回非零退出码就自动反复重试。失败重试应该通过脚本、自定义 Service 策略或专门的任务队列明确实现。

### 普通用户也能创建 Timer

不需要 root 权限的个人任务可以使用用户级 Timer。

单元文件目录是：

```text
~/.config/systemd/user/
```

例如：

```text
~/.config/systemd/user/notes-sync.service
~/.config/systemd/user/notes-sync.timer
```

管理命令需要加 `--user`：

```bash
systemctl --user daemon-reload
systemctl --user enable --now notes-sync.timer
systemctl --user list-timers
journalctl --user -u notes-sync.service
```

用户退出登录后，用户级 systemd 管理器是否继续运行取决于系统配置。需要退出后继续执行时，可以由管理员开启 linger：

```bash
sudo loginctl enable-linger username
```

系统级 Timer 的 `[Service]` 中也可以使用 `User=appuser`，但这种方式和用户级 Timer 不是同一套管理范围。

### 常见问题排查

#### Timer 根本没有启动

```bash
systemctl status myapp-backup.timer
systemctl is-enabled myapp-backup.timer
systemctl list-timers --all
```

`enabled` 表示已经设置开机启动，`active` 才表示当前正在等待触发。只执行 `enable` 而没有重启服务器，也没有执行 `start`，Timer 当前可能仍未运行。

#### 修改文件后没有变化

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp-backup.timer
```

再用下面的命令确认 systemd 实际加载了什么：

```bash
systemctl cat myapp-backup.timer
```

#### 时间表达式写错

```bash
systemd-analyze calendar --iterations=5 '*:0/5'
```

不要仅根据 `systemctl status` 猜测，先确认表达式是否能解析、下一次触发是否符合预期。

#### Service 手动执行也失败

```bash
sudo systemctl start myapp-backup.service
systemctl status myapp-backup.service
sudo journalctl -u myapp-backup.service -e --no-pager
```

常见原因包括：

* `ExecStart=` 使用了相对路径
* 脚本没有执行权限
* 脚本依赖登录 Shell 中的 `PATH`
* 运行用户没有文件读写权限
* 安全加固项阻止了目录访问
* 命令返回了非零退出码
* SELinux 或 AppArmor 拒绝访问

定时任务中的命令尽量使用绝对路径。可以这样查询：

```bash
command -v tar
command -v curl
command -v mysqldump
```

#### Timer 正常，Service 却没有再次执行

先检查 Service 是否仍处于 active 状态：

```bash
systemctl status myapp-backup.service
```

相同 Service 还在运行时，后续 Timer 触发不会再启动一个副本。任务卡死时，需要结合日志、进程状态和 `TimeoutStartSec=` 排查。

#### `Persistent=true` 没有补跑

重点检查三项：

* 是否使用了 `OnCalendar=`
* Timer 以前是否启动过并留下触发时间记录
* 错过时间后 Timer 是否真的重新变成 active

`Persistent=true` 对 `OnBootSec=`、`OnUnitActiveSec=` 等单调触发器不起作用。

### 常用模板

一次性任务的 Service 模板：

```ini
[Unit]
Description=Run scheduled task

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/task.sh
TimeoutStartSec=10min
StandardOutput=journal
StandardError=journal
```

固定时间执行的 Timer 模板：

```ini
[Unit]
Description=Run scheduled task every day

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=10min
AccuracySec=1min

[Install]
WantedBy=timers.target
```

固定间隔执行的 Timer 模板：

```ini
[Unit]
Description=Run scheduled task periodically

[Timer]
OnBootSec=2min
OnUnitInactiveSec=30min

[Install]
WantedBy=timers.target
```

部署命令：

```bash
sudo systemd-analyze verify \
    /etc/systemd/system/task.service \
    /etc/systemd/system/task.timer

sudo systemctl daemon-reload
sudo systemctl start task.service
sudo systemctl enable --now task.timer
```

### 总结

systemd Timer 的核心并不复杂：

```text
.service 定义任务内容和运行方式
.timer 定义触发时间
```

日历任务使用 `OnCalendar=`，相对时间任务使用 `OnBootSec=`、`OnUnitActiveSec=` 或 `OnUnitInactiveSec=`。

生产环境中还需要重点关注：

* `Persistent=true` 是否需要补跑
* `RandomizedDelaySec=` 是否需要错峰
* `TimeoutStartSec=` 是否能防止任务长期卡住
* Service 的运行用户和目录权限是否正确
* 脚本是否使用绝对路径并正确返回退出码
* journal 中是否保留了足够的排错信息

从 cron 迁移到 systemd Timer 后，最大的变化不是定时语法，而是每个定时任务都变成了一个可查看状态、可追踪日志、可限制权限的系统服务。

### 参考资料

* [systemd.timer 官方手册](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html)
* [systemd.time 官方手册](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html)
* [systemd.service 官方手册](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
* [systemctl 官方手册](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
* [systemd-analyze 官方手册](https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html)
