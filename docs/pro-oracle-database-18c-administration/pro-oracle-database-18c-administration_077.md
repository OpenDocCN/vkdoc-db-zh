# 启用对 Cron 的访问权限

有时，系统管理员在设置新服务器时，默认不会为系统上的所有用户启用 `cron` 功能。要验证你是否拥有对 `cron` 的访问权限，请按如下方式调用该工具：

```
$ crontab -e
```

如果你收到以下错误信息，则表示你没有访问权限：

```
You (oracle) are not allowed to use this program (crontab)
```

要以 `root` 用户身份启用 cron 访问权限，可以使用 `echo` 命令将 `oracle` 添加到 `/etc/cron.allow` 文件中：

```
# echo oracle >> /etc/cron.allow
```

一旦 `oracle` 条目被添加到 `/etc/cron.allow` 文件中，你就可以使用 `crontab` 工具来安排任务了。

> **注意**
> 你也可以使用编辑工具（如 `vi`）向 `cron.allow` 文件中添加条目。

`root` 用户总是可以使用 `crontab` 工具安排任务。其他用户必须被列在 `/etc/cron.allow` 文件中。如果 `/etc/cron.allow` 文件不存在，那么操作系统用户则不能出现在 `/etc/cron.deny` 文件中。如果 `/etc/cron.allow` 和 `/etc/cron.deny` 文件都不存在，那么只有 `root` 用户可以访问 `crontab` 工具。（这些规则可能因你使用的 Linux/Unix 版本而略有不同。）

在某些 Unix 操作系统（如 Solaris）上，`cron.allow` 和 `cron.deny` 文件位于 `/etc/cron.d` 目录中。这些文件通常只能由 `root` 用户修改。

> **提示**
> 在某些 Linux/Unix 平台上，可以使用更新、更灵活的 `cron` 变体，例如 `anacron` 工具。使用 `man anacron` 命令可以查看该工具在你的系统上的实现细节。

