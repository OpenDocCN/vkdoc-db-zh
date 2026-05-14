# 管理数据库环境的脚本和技巧

## 查找占用磁盘空间的大目录

```bash
# find largest directories consuming space below this point
function fld {
du -S . | sort -nr | head -10
}
```

## 切换到数据库诊断目录

```bash
#-----------------------------------------------------------#
function bdump {
if [ $ORACLE_SID = "o18c" ]; then
cd /u01/app/oracle/diag/rdbms/o18c/o18c/trace
elif [ $ORACLE_SID = "CDB1" ]; then
cd /u01/app/oracle/diag/rdbms/cdb1/CDB1/trace
elif [ $ORACLE_SID = "rcat" ]; then
cd /u01/app/oracle/diag/rdbms/rcat/rcat/trace
fi
pwd
} # bdump
```

## 判断命令类型

如果不确定一个快捷方式是别名还是函数，可以使用 `type` 命令来验证命令的来源。下面的例子验证了 `bdump` 是一个函数：

```
$ type bdump
```

## 快速重新运行命令

当数据库服务器出现问题时，你需要能够快速地从操作系统提示符运行命令。你可能遇到了某种性能问题，需要运行命令导航到包含日志文件的目录，或者你想随时显示消耗资源最多的进程。在这些情况下，你不想浪费时间重新输入命令序列。

Bash shell 的一个省时特性是它提供了多种编辑和重新运行先前执行过的命令的方法。以下列出了可用于操作先前输入命令的几种选项：

*   使用向上 (`↑`) 和向下 (`↓`) 箭头键滚动
*   使用 `Ctrl+P` 和 `Ctrl+N`
*   列出命令历史
*   反向搜索
*   设置命令编辑器

以下部分将简要描述这些技术。

### 使用上下箭头键滚动

你可以使用向上箭头键在最近的命令历史中向上滚动。当你滚动浏览之前运行过的命令时，可以通过按 `Enter` 或 `Return` 键来重新运行所需的命令。如果你想编辑一个命令，可以使用退格键删除字符，或者使用左箭头键在命令文本中导航到所需位置。在命令堆栈中向上滚动后，可以使用向下箭头键向下滚动回之前查看过的命令。

> **注意**
> 如果你熟悉 Windows，滚动命令堆栈类似于使用 `DOSKEY` 实用程序。

### 使用 `Ctrl+P` 和 `Ctrl+N`

`Ctrl+P` 键击（同时按下 `Ctrl` 和 `P` 键）会显示你之前输入的命令。如果你多次按了 `Ctrl+P`，可以通过按 `Ctrl+N`（同时按下 `Ctrl` 和 `N` 键）在命令堆栈中向下滚动。

### 列出命令历史

你可以使用 `history` 命令来显示用户先前输入的命令：

```
$ history
```

根据之前执行的命令数量，你可能会看到一个很长的堆栈。你可以通过提供一个数字来限制输出最后 n 条命令。例如，以下查询列出了最后运行的五条命令：

```
$ history 5
```

示例输出如下：

```
273  cd -
274  grep -i ora alert.log
275  ssh -Y -l oracle 65.217.177.98
276  pwd
277  history 5
```

要运行输出中列出的先前命令，使用感叹号 (`!`)（有时称为 bang）后跟历史记录编号。在这个例子中，要运行第 276 行的 `pwd` 命令，使用 `!276`，如下所示：

```
$ !276
```

要运行你最后执行的命令，使用 `!!`，如下所示：

```
$ !!
```

### 反向搜索

按 `Ctrl+R`，你会看到 Bash shell 的反向搜索工具：

```
$ (reverse-i-search)`':
```

在 `reverse-i-search` 提示符下，随着你输入每个字母，工具会自动搜索先前运行过的、包含与你输入字符串相似文本的命令。一旦出现你想要的匹配命令，你可以通过按 `Enter` 或 `Return` 键重新运行该命令。要查看所有匹配某个字符串的命令，可以重复按 `Ctrl+R`。要退出反向搜索，按 `Ctrl+C`。

### 设置命令编辑器

你可以使用 `set -o` 命令将你的命令行编辑器设置为 `vi` 或 `emacs`。此示例将命令行编辑器设置为 `vi`：

```
$ set -o vi
```

现在，当你按 `Esc+K` 时，你就进入了一种可以使用 `vi` 命令搜索先前输入的命令堆栈的模式。例如，如果你想在命令堆栈中向上滚动，可以使用 `K` 键；同样，你可以使用 `J` 键向下滚动。在此模式下，你可以使用斜杠 (`/`) 键，然后键入一个字符串，以便在整个命令堆栈中搜索。

> **提示**
> 在尝试使用命令编辑器功能之前，请确保你非常熟悉 `vi` 或 `emacs` 编辑器。

下面是一个简短的例子来说明这个功能的强大。假设你知道大约一小时前运行过 `ls -altr` 命令。你想再次运行它，但这次不带 `r`（反向排序）选项。要进入命令堆栈，按 `Esc+K`：

```
$ Esc+K
```

你现在应该看到最后执行的命令。要在命令堆栈中搜索 `ls` 命令，输入 `/ls`，然后按 `Enter` 或 `Return`：

```
$ /ls
```

最近执行的 `ls` 命令会出现在提示符处：

```
$ ls -altr
```

要移除 `r` 选项，使用右箭头键将光标移动到屏幕上的 `r` 上，然后按 `X` 将 `r` 从命令末尾移除。编辑完命令后，按 `Enter` 或 `Return` 键执行它。

## 开发标准脚本

我曾在一个地方工作，那里的数据库管理团队开发了数百个脚本和实用程序来帮助管理环境。有一家公司有一小队 DBA，他们的工作职责就是维护这些环境脚本。我认为这有点过度了。我倾向于使用一小套专注于特定任务的脚本，每个脚本通常不超过 50 行。如果你开发了一个其他 DBA 无法理解或维护的脚本，那么它就失去了其有效性。此外，如果你必须执行一个命令超过几次，就应该创建一个脚本来执行它。如果它已经成为一项标准的、定期的检查或任务，则可以使用该脚本来自动化该过程。

这些脚本很方便，可以放入自动运行的作业中，或在需要快速完成任务的故障排除期间使用。还有其他工具也可以维护数据库，并提供针对多个数据库的主动警报和监控，而不是一次只在一个数据库上运行脚本。

> **注意**
> 本章中的所有脚本都可以从 Apress 网站的源代码/下载区下载。

本节包含几个简短的 shell 函数、shell 脚本和 SQL 脚本，它们可以帮助你管理数据库环境。这绝不是一个完整的脚本列表，而是为你提供一个可以在此基础上构建的起点。每个小节的标题就是一个脚本的名称。

> **注意**
> 在尝试运行 shell 脚本之前，请确保它是可执行的。使用 `chmod` 命令来实现这一点：`chmod 750 <script>`

## `dba_setup` 脚本

通常，你会为每个数据库服务器建立一组相同的操作系统变量和别名。在服务器之间导航时，你应该以一致且可重复的方式设置这些变量和别名。这样做有助于你（或你的团队）在每种环境中高效工作。例如，当你处理几十台不同的服务器时，将操作系统提示符设置为一致的方式非常有用。这有助于你快速识别你所在的机器、登录的操作系统用户等等。

一种技术是将这些标准设置存储在一个脚本中，然后在登录到服务器时自动执行该脚本。我通常创建一个名为 `dba_setup` 的脚本来设置这些操作系统变量和别名。你可以将此脚本放在像 `HOME/bin` 这样的目录中，并通过启动脚本自动执行它（参见本章后面的“组织脚本”部分）。下面是一个典型的 `dba_setup` 脚本的内容：

```bash

```



## 设置提示符

```
PS1='[\h:\u:${ORACLE_SID}]$ '
```

```
export EDITOR=vi
export VISUAL=$EDITOR
export SQLPATH=$HOME/scripts
set -o vi
```

## 仅列出目录

```
alias lsd="ls -p | grep /"
```

## 显示消耗 CPU 最多的进程

```
alias topc="ps -e -o pcpu,pid,user,tty,args | sort -n -k 1 -r | head"
```

## 显示消耗内存最多的进程

```
alias topm="ps -e -o pmem,pid,user,tty,args | sort -n -k 1 -r | head"
```

```
alias sqlp='sqlplus "/ as sysdba"'
alias shutdb='echo "shutdown immediate;" | sqlp'
alias startdb='echo "startup;" | sqlp'
```

## dba_fcns

使用此脚本来存储有助于你在数据库环境中导航和操作的操作系统函数。函数通常比别名具有更多功能。你可以相当有创意地使用数量众多且复杂的函数。其目的是，无论你登录到哪个数据库服务器，都能调用一套一致且标准的函数。

将此脚本放在诸如 `HOME/bin` 的目录中。通常，你会在通过启动脚本登录到服务器时自动调用此脚本（参见本章后面的“组织脚本”部分）。以下是一些你可以使用的典型函数：

```
-----------------------------------------------------------#

# 以排序列表形式显示环境变量
function envs {
if test -z "$1"
then /bin/env | /bin/sort
else /bin/env | /bin/sort | /bin/grep -i $1
fi
} # envs
#-----------------------------------------------------------#

# 登录到 sqlplus
function sp {
time sqlplus "/ as sysdba"
} # sp
#-----------------------------------------------------------#

# 查找当前位置下最大的文件
function flf {
find . -ls | sort -nrk7 | head -10
}
#-----------------------------------------------------------#

# 查找当前位置下占用空间最大的目录
function fld {
du -S . | sort -nr | head -10
}
#-----------------------------------------------------------#

# 切换到包含告警日志文件的目录
function bdump {
cd /u01/app/oracle/diag/rdbms/o18c/o18c/trace
} # bdump
#-----------------------------------------------------------#
```

## tbsp_chk.bsh

此脚本用于检查是否有任何表空间超过了特定的容量阈值。将此脚本存储在诸如 `HOME/bin` 的目录中。确保修改脚本以包含适用于你环境的正确用户名、密码和电子邮件地址。你还需要建立所需的操作系统变量，如 `ORACLE_SID` 和 `ORACLE_HOME`。你可以将这些变量硬编码到脚本中，也可以调用一个为你提供变量源的脚本。下一个脚本调用一个名为 `oraset` 的脚本来设置操作系统变量（有关此脚本的详细信息，请参见第 2 章）。你不必使用此脚本——其目的是为你的环境建立一种一致且可重复的方式来设置操作系统变量。

你可以从命令行运行此脚本。在此示例中，我传递了数据库名称 (o18c) 并希望查看哪些表空间剩余空间不足 20%：

```
$ tbsp_chk.bsh o18c 20
```

输出表明此数据库有两个表空间的剩余空间不足 20%：

```
space not okay
0 % free UNDOTBS1, 17 % free SYSAUX,
```

以下是 `tbsp_chk.bsh` 脚本的内容：

```
#!/bin/bash
#
if [ $# -ne 2 ]; then
echo "Usage: $0 SID threshold"
exit 1
fi

# 或者硬编码操作系统变量，或者从脚本中获取它们。

# 有关使用 oraset 来源 oracle 操作系统变量的详细信息，请参见第 2 章
. /var/opt/oracle/oraset $1
#
crit_var=$(
sqlplus -s <<EOF
system/foo
SET HEAD OFF TERM OFF FEED OFF VERIFY OFF
COL pct_free FORMAT 999
SELECT (f.bytes/a.bytes)*100 pct_free,'% free',a.tablespace_name||','
FROM
(SELECT NVL(SUM(bytes),0) bytes, x.tablespace_name
FROM dba_free_space y, dba_tablespaces x
WHERE x.tablespace_name = y.tablespace_name(+)
AND x.contents != 'TEMPORARY' AND x.status != 'READ ONLY'
AND x.tablespace_name  NOT LIKE 'UNDO%'
GROUP BY x.tablespace_name) f,
(SELECT SUM(bytes) bytes, tablespace_name
FROM dba_data_files
GROUP BY tablespace_name) a
WHERE a.tablespace_name = f.tablespace_name
AND  (f.bytes/a.bytes)*100 <= $2
ORDER BY 1;
EXIT;
EOF)
if [ "$crit_var" = "" ]; then
echo "space okay"
else
echo "space not okay"
echo $crit_var
echo $crit_var | mailx -s "tbsp getting full on $1" dkuhn@gmail.com
fi
exit 0
```

通常，你会通过调度工具（如 `cron`）自动定期运行此类脚本。以下是一个典型的 `cron` 条目，它每小时运行一次该脚本：

```
# 表空间检查
2 * * * * /orahome/bin/tbsp_chk.bsh INVPRD 10 1>/orahome/bin/log/tbsp_chk.log 2>&1
```

此 `cron` 条目运行作业并将任何信息输出存储在 `tbsp_chk.log` 文件中。

在 Oracle Database 12c 可插拔数据库环境中，从根容器运行 `tbsp_chk.bsh` 时，你需要引用 `CDB_*` 视图而不是 `DBA_*` 视图，以便脚本能正确报告容器数据库内所有可插拔数据库的空间情况。你还应该考虑在查询中添加 `NAME` 和 `CON_ID`，以便查看哪个可插拔数据库可能存在空间问题；例如，

```
SELECT a.name, (f.bytes/a.bytes)*100 pct_free,'% free',a.tablespace_name||','
FROM
(SELECT c.name, NVL(SUM(bytes),0) bytes, x.tablespace_name
FROM cdb_free_space y, cdb_tablespaces x, v$containers c
WHERE x.tablespace_name = y.tablespace_name(+)
AND x.contents != 'TEMPORARY' AND x.status != 'READ ONLY'
AND x.tablespace_name  NOT LIKE 'UNDO%'
AND x.con_id = y.con_id
AND x.con_id = c.con_id
GROUP BY c.name, x.tablespace_name) f,
(SELECT c.name, SUM(d.bytes) bytes, d.tablespace_name
FROM cdb_data_files d, v$containers c
WHERE d.con_id = c.con_id
GROUP BY c.name, tablespace_name) a
WHERE a.tablespace_name = f.tablespace_name
AND  (f.bytes/a.bytes)*100 <= 50
AND   a.name NOT IN ('PDB$SEED')
AND   a.name = f.name
ORDER BY 1;
```

## conn.bsh

如果连接数据库时出现问题，你需要被提醒。此脚本检查是否可以建立到数据库的连接。如果无法建立连接，则会发送电子邮件。将此脚本放在诸如 `HOME/bin` 的目录中。确保修改脚本以包含适用于你环境的正确用户名、密码和电子邮件地址。你还需要建立所需的操作系统变量，如 `ORACLE_SID` 和 `ORACLE_HOME`。你可以将这些变量硬编码到脚本中，也可以调用一个为你提供变量源的脚本。与上一个脚本类似，此脚本调用一个名为 `oraset` 的脚本来设置操作系统变量（参见第 2 章）。

此脚本需要将 `ORACLE_SID` 传递给它；例如，

```
$ conn.bsh INVPRD
```

如果脚本能够建立到数据库的连接，则显示以下消息：

```
success
db ok
```

以下是 `conn.bsh` 脚本的内容：

```
#!/bin/bash
if [ $# -ne 1 ]; then
echo "Usage: $0 SID"
exit 1
fi

# 或者硬编码操作系统变量，或者从脚本中获取它们。

# 有关使用 oraset 脚本来源操作系统变量的详细信息，请参见第 2 章
. /etc/oraset $1
#
echo "select 'success' from dual;" | sqlplus -s system/foo@o18c | grep success
if [[ $? -ne 0 ]]; then
echo "problem with $1" | mailx -s "db problem" dkuhn@gmail.com
else
echo "db ok"
fi
#
exit 0
```

此脚本通常通过 `cron` 等实用程序实现自动化。以下是一个典型的 `cron` 条目：



