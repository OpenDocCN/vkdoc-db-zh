# ls /var/spool/cron
oracle root
```

`cron` 后台进程大部分时间处于空闲状态。它每分钟唤醒一次，并检查 `/etc/crontab`、`/etc/cron.d` 以及用户 cron 表文件，以确定是否有任何需要执行的任务。

表 21–1 总结了 `cron` 使用的各种文件和目录的用途。了解这些文件和目录将有助于你排查问题并更详细地理解 `cron`。

### **表 21–1. cron 工具使用的文件和目录描述**

| 文件 | 用途 |
| :--- | :--- |
| `/etc/init.d/crond` | 在系统启动时启动 cron 守护进程。 |
| `/var/log/cron` | 与 cron 进程相关的系统消息。对于故障排除很有用。 |
| `/var/spool/cron/<用户名>` | 用户的 crontab 文件存储在 `/var/spool/cron` 目录中。 |
| `/etc/cron.allow` | 指定可以创建 cron 表的用户。 |
| `/etc/cron.deny` | 指定不允许创建 cron 表的用户。 |
| `/etc/crontab` | 系统 cron 表，其命令用于运行位于以下目录中的脚本：`/etc/cron.hourly`、`/etc/cron.daily`、`/etc/cron.weekly` 和 `/etc/cron.monthly`。 |
| `/etc/cron.d` | 一个目录，包含需要以非每小时、每天、每周或每月周期运行的任务的 cron 表。 |
| `/etc/cron.hourly` | 包含需要每小时运行的系统脚本的目录。 |
| `/etc/cron.daily` | 包含需要每天运行的系统脚本的目录。 |
| `/etc/cron.weekly` | 包含需要每周运行的系统脚本的目录。 |
| `/etc/cron.monthly` | 包含需要每月运行的系统脚本的目录。 |

## **启用对 cron 的访问权限**

有时，系统管理员在设置新系统时，默认不会为所有用户启用 `cron` 功能。要验证你是否有权限访问 `cron`，请键入以下命令：

```bash
$ crontab -e
```

如果你收到以下错误消息，则表示你没有权限：`You (oracle) are not allowed to use this program (crontab)`

要以 `root` 用户身份启用 `cron` 访问权限，可以使用 `echo` 命令将 `oracle` 用户添加到 `/etc/cron.allow` 文件中：

```bash
