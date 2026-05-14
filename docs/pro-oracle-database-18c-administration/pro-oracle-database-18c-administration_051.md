# 设置 NLS_DATE_FORMAT

在运行任何 RMAN 作业之前，请设置操作系统变量 `NLS_DATE_FORMAT` 以包含时间（时、分、秒）组成部分；例如，

```bash
$ export NLS_DATE_FORMAT='dd-mon-yyyy hh24:mi:ss'
```

此外，如果一个 Shell 脚本调用了 RMAN，请将上述行直接放入 Shell 脚本中（示例请参见第 17 章末尾的 shell 脚本）：

```bash
NLS_DATE_FORMAT='dd-mon-yyyy hh24:mi:ss'
```

这确保了当 RMAN 显示日期时，其输出始终包含小时、分钟和秒钟。默认情况下，RMAN 的输出中只包含日期部分（`DD-MON-YY`）。例如，未设置 `NLS_DATE_FORMAT` 时，开始备份时 RMAN 显示如下：

```
Starting backup at 11-JAN-13
```

当你将操作系统变量 `NLS_DATE_FORMAT` 设置为包含时间部分后，输出将变为：

```
Starting backup at 11-jan-2013 16:43:04
```

在故障排除时，拥有时间部分至关重要，因为这样你可以确定一个命令运行了多久，或者一个命令在失败前运行了多长时间。Oracle 技术支持几乎总是要求你在捕获输出并发送给他们之前，将此变量设置为包含时间部分。

设置 `NLS_DATE_FORMAT` 的唯一缺点是，如果将其设置为 RMAN 无法识别的值，可能会导致连接问题。例如，这里 `NLS_DATE_FORMAT` 被设置为一个无效值：

```bash
$ export NLS_DATE_FORMAT='dd-mon-yyyy hh24:mi:sd'
$ rman target /
```

当设置为无效值时，登录 RMAN 会遇到此错误：

```
RMAN-03999: Oracle error occurred while converting a date: ORA-01821:
```

要取消设置 `NLS_DATE_FORMAT` 变量，请将其设置为空值，如下所示：

```bash
$ export NLS_DATE_FORMAT="
```

