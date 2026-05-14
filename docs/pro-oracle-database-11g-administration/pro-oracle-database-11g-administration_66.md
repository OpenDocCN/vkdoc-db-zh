# 截断大型日志文件

有时，日志文件可能变得非常大，并通过填满关键挂载点而导致问题。`listener.log` 将记录有关数据库传入连接的信息。对于活跃系统，此文件可能迅速增长到数 GB。对于我的大多数环境，`listener.log` 文件中的信息无需出于任何原因保留。如果出现 Oracle Net 连接问题，则可以检查该文件以帮助进行故障排除。

`listener.log` 文件是正在写入的，因此您不应该直接删除它。如果您删除了该文件，监听器进程不会重新创建该文件并再次开始写入。您必须停止并重新启动监听器才能恢复其对 `listener.log` 文件的写入。但是，您可以清空或截断 `listener.log` 文件。在 Linux/Unix 环境中，这通过以下技术完成：
```
$ cat /dev/null >listener.log
```

前面的命令用 `/dev/null`（Linux/Unix 系统上的一个默认文件，内容为空）的内容替换 `listener.log` 文件的内容。上一行代码的结果是 `listener.log` 文件被截断，监听器可以继续主动向其中写入。

**清单 21–5.** 截断默认 `listener.log` 的脚本
```
#!/bin/bash
#
if [ $# -ne 1 ]; then
    echo "Usage: $0 SID"
    exit 1
fi
# 关于设置操作系统变量的详细信息请参见第 2 章
# 使用 oraset 脚本加载 oracle 操作系统变量
. /var/opt/oracle/oraset $1
#
MAILX='/bin/mailx'
MAIL_LIST='dkuhn@sun.com'
#
if [ -f $TNS_ADMIN/../log/listener.log ]; then
    cat /dev/null > $TNS_ADMIN/../log/listener.log
fi
if [ $? -ne 0 ]; then
    echo "trunc list. problem" | $MAILX -s "trunc list. problem $1" $MAIL_LIST
else
    echo "no problem..."
fi
# 一个命名的监听器日志文件
if [ -f $TNS_ADMIN/../log/appinvprd.log ]; then
    cat /dev/null > $TNS_ADMIN/../log/appinvprd.log
fi
if [ $? -ne 0 ]; then
    echo "trunc list. problem" | $MAILX -s "trunc list. problem $1" $MAIL_LIST
else
    echo "no problem..."
fi
#
exit 0
```

以下的 `cron` 条目每月运行一次前述脚本：
```
# 每月截断日志文件一次。
30 6 1 * * /orahome/oracle/bin/trunc_log.bsh DWREP
1>/orahome/oracle/bin/log/trunc_log.log 2>&1
```
> （注意：此 `cron` 表条目被分成两行以适应页面。在 `cron` 表中，它需要位于一行。）

