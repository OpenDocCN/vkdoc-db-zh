# 调查磁盘使用情况

要进一步诊断问题（例如磁盘空间不足），您需要直接登录到远程服务器。通常，您需要以 Oracle 软件的所有者身份（通常是 `oracle` 操作系统账户）登录。首次登录到服务器时，导致数据库挂起或出现问题的一个原因是挂载点已满。带有人类可读选项 `-h` 的 `df` 命令有助于验证磁盘使用情况：
```
$ df -h
```
任何已满的挂载点都需要进行调查。如果包含 `ORACLE_HOME` 的挂载点已满，那么在连接到数据库时会收到如下错误：
```
Linux Error: 28: No space left on device
```
要解决挂载点已满的问题，首先要识别可以移动或删除的文件。通常，从查找旧的跟踪文件开始；通常有一些旧文件可以安全删除。

## 定位警报日志和跟踪文件

默认的警报日志目录路径结构如下：
```
ORACLE_BASE/diag/rdbms/<DB_UNIQUE_NAME>/<INSTANCE_NAME>/trace
```
或者在 SQL*Plus 中查看目录：
```
SQL> show parameter background
```
> 注意
> 您可以通过设置 `DIAGNOSTIC_DEST` 初始化参数来覆盖警报日志的默认目录路径。

通常，`db_unique_name` 与 `instance_name` 相同。然而，在 RAC 和 Data Guard 环境中，`db_unique_name` 通常与 `instance_name` 不同。您可以使用以下查询验证目录路径：
```
SQL> select value from v$diag_info where name = 'Diag Trace';
```
警报日志的名称遵循以下格式：
```
alert_<ORACLE_SID>.log
```
您还可以通过操作系统命令从 OS 定位警报日志（无论数据库是否已启动）：
```
$ cd $ORACLE_BASE
$ find . -name alert_<ORACLE_SID>.log
```
在上面的 `find` 命令中，您需要将 `<ORACLE_SID>` 值替换为您的数据库名称。

如第 3 章所示，建议设置一个操作系统函数来帮助您导航到警报日志的位置。下面是一个这样的函数（您需要根据您的环境进行修改）：
```
function bdump {
if [ "$ORACLE_SID" = "O18C" ]; then
cd /orahome/app/oracle/diag/rdbms/o18c/O18C/trace
elif [ "$ORACLE_SID" = "O12c" ]; then
cd /orahome/app/oracle/diag/rdbms/o12c/O12C/trace  
elif [ "$ORACLE_SID" = "O11R2" ]; then
cd /orahome/app/oracle/diag/rdbms/o11r2/O11R2/trace
fi
}
```
现在您可以输入 `bdump`，然后将被置于包含数据库警报日志的工作目录中。找到正确的文件后，检查最近的条目是否有错误，然后在同一目录中查找跟踪文件：
```
$ ls -altr *.tr*
```
如果其中任何跟踪文件超过几天，请考虑移动或删除它们。

## 删除文件
不用说，在删除文件时要非常小心。在尝试解决问题时，最不想做的事情就是让情况变得更糟。意外删除一个关键文件可能是灾难性的。对于您识别为候选删除的任何文件，请考虑移动（而不是删除）它们。如果您有可用空间的挂载点，请将文件移过去，放置几天后再删除它们。

> 提示
> 考虑使用 Automatic Diagnostic Repository Command Interpreter (ADRCI) 实用程序来清除旧的跟踪文件。有关更多详细信息，请参阅《Oracle Database Utilities Guide》，可从 Oracle 技术网络区网站下载。

如果您已识别出可以删除的文件，请在实际删除它们之前先创建要删除的文件列表。至少在删除任何文件之前执行此操作：
```
$ ls -altr <filename>
```
查看 `ls` 命令返回的结果后，删除文件。此示例使用 Linux/Unix 的 `rm` 命令永久删除文件：
```
$ rm <filename>
```
您也可以根据文件年龄删除文件。例如，假设您确定任何超过 2 天的跟踪文件都可以安全删除。通常，`find` 命令与 `rm` 命令结合使用来完成此任务。在删除文件之前，首先显示 `find` 命令的结果：
```
$ find . -type f -mtime +2 -name "*.tr*"
```
如果您对文件列表满意，则添加 `rm` 命令来删除它们：
```
$ find . -type f -mtime +2 -name "*.tr*" | xargs rm
```
在上面的代码行中，`find` 命令的结果通过管道传递给 `xargs` 命令，该命令为 `find` 命令找到的每个文件执行 `rm` 命令。这是一种基于年龄删除文件的有效方法。但是，请务必确认您知道将要删除哪些文件。

另一个有时会占用大量空间的文件是 `listener.log` 文件。由于此文件由侦听器进程主动写入，因此您不能简单地删除它。在第 19 章中有一个关于此作业的示例，可以执行该作业或查看它是否正在运行。如果您需要保留此文件的内容以检查连接问题，请先将其复制到有可用磁盘空间的备份位置，然后截断该文件。在此示例中，`listener.log` 文件被复制到 `/u01/backups`，然后截断该文件，如下所示：
```
$ cp listener.log /u01/backups
```
接下来，使用 `cat` 命令将 `listener.log` 的内容替换为 `/dev/null` 文件（它包含零字节）：
```
$ cat /dev/null > listener.log
```
此文件需要替换为空内容（/dev/null），而不是删除文件并让系统创建新文件，因为此文件附加到一个进程（侦听器）。替换文件将允许写入操作在进程运行时继续到该文件。对于 `alert.log` 文件也需要执行相同的操作。

