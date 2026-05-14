# 通过 cron 实现作业自动化

`cron` 程序是一个作业调度实用工具，在 Linux/Unix 环境中无处不在。其名称来源于希腊语中的 "chrónos"（意为“时间”）。`cron`（极客们对“调度程序”的称呼）允许你安排脚本或命令在指定时间运行，并以设定的频率重复执行。

## cron 工作原理

当你的 Linux 服务器启动时，一个 `cron` 后台进程会自动启动，以管理所有系统中的 `cron` 作业。该 `cron` 后台进程也称为 `cron` 守护进程。此进程由 `/etc/init.d/crond` 脚本在系统启动时启动。你可以使用 `ps` 命令检查 `cron` 守护进程是否正在运行：

```
$ ps -ef | grep crond | grep -v grep
root      3081     1  0  2012 ?        00:00:18 crond
```

在 Linux 系统上，你也可以使用 `service` 命令检查 `cron` 守护进程是否在运行：

```
$ /sbin/service crond status
crond (pid  3081) is running...
```

`root` 用户在执行系统 `cron` 作业时会使用到若干文件和目录。`/etc/crontab` 文件包含了运行系统 `cron` 作业的命令。以下是 `/etc/crontab` 文件内容的典型示例：

```
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
HOME=/
# run-parts
01 * * * * root run-parts /etc/cron.hourly
02 4 * * * root run-parts /etc/cron.daily
22 4 * * 0 root run-parts /etc/cron.weekly
42 4 1 * * root run-parts /etc/cron.monthly
```

这个 `/etc/crontab` 文件使用 `run-parts` 实用工具来运行位于以下目录中的脚本：`/etc/cron.hourly`、`/etc/cron.daily`、`/etc/cron.weekly` 和 `/etc/cron.monthly`。如果有一个需要运行的系统实用工具，其执行频率不是每小时、每天、每周或每月一次，那么它可以被放置在 `/etc/cron.d` 目录中。

每个用户都可以创建一个 `crontab`（也称为 `cron` 表）文件。该文件包含了你希望在特定时间和间隔运行的程序列表。这个文件通常位于 `/var/spool/cron` 目录中。对于每个创建了 `cron` 表的用户，`/var/spool/cron` 目录中都会有一个以该用户名命名的文件。作为 `root` 用户，你可以列出该目录中的文件，如下所示：

```
# ls /var/spool/cron
oracle root
```

`cron` 后台进程大部分时间处于空闲状态。它每分钟唤醒一次，检查 `/etc/crontab`、`/etc/cron.d` 以及用户 `cron` 表文件，并确定是否有任何作业需要执行。

表 20-1 总结了 `cron` 所使用的各种文件和目录的用途。了解这些文件和目录将有助于你排查问题，并更好地理解 `cron`。

### 表 20-1 cron 实用工具使用的文件和目录描述

| 文件 | 用途 |
| :--- | :--- |
| `/etc/init.d/crond` | 在系统启动时启动 `cron` 守护进程 |
| `/var/log/cron` | 与 `cron` 进程相关的系统消息；对故障排除很有用 |
| `/var/spool/cron/<username>` | 用户的 `crontab` 文件存储在 `/var/spool/cron` 目录中 |
| `/etc/cron.allow` | 指定可以创建 `cron` 表的用户 |
| `/etc/cron.deny` | 指定不允许创建 `cron` 表的用户 |
| `/etc/crontab` | 系统 `cron` 表，其中包含运行以下目录中脚本的命令：`/etc/cron.hourly`、`/etc/cron.daily`、`/etc/cron.weekly` 和 `/etc/cron.monthly` |
| `/etc/cron.d` | 包含 `cron` 表的目录，用于需要以非每小时、每天、每周或每月频率运行的作业 |
| `/etc/cron.hourly` | 包含每小时运行一次系统脚本的目录 |
| `/etc/cron.daily` | 包含每天运行一次系统脚本的目录 |
| `/etc/cron.weekly` | 包含每周运行一次系统脚本的目录 |
| `/etc/cron.monthly` | 包含每月运行一次系统脚本的目录 |



