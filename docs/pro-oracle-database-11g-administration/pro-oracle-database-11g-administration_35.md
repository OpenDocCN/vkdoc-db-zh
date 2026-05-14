# 显示最耗内存的进程

```bash
alias topm="ps -e -o pmem,pid,user,tty,args | sort -n -k 1 -r | head"
```

```bash
alias sqlp='sqlplus "/ as sysdba"'
alias shutdb='echo "shutdown immediate;" | sqlp'
alias startdb='echo "startup;" | sqlp'
```

## `dba_fcns`

使用此脚本来存储有助于您在数据库环境中导航和操作的操作系统函数。函数通常比别名具有更多的功能。您可以根据需要使用的函数数量和复杂性发挥相当的创造力。关键在于，无论您登录到哪个数据库服务器，都希望有一套一致且标准的函数可供调用。

例如，您可能经常需要导航到数据库警报日志所在的目录。

如果您在包含多个 Oracle 版本的环境中工作，警报日志的位置会因数据库版本而异。下一个脚本中包含一个函数，用于确定您是否处于 Oracle Database 11g 环境中（或不是），并导航到警报日志的适当位置。您可以通过手动运行脚本来建立这些函数，像这样：`$ . dba_fcns`

现在您就有了一个名为 `bdump` 的函数可以执行：

```bash
$ bdump
```

对于这个环境，该函数将我的目录更改为：

```
/oracle/app/oracle/diag/rdbms/o11r2/O11R2/trace
```

将此脚本放置在 `HOME/bin` 这样的目录中。通常，您会通过启动脚本（将在后面的“组织脚本”部分介绍）在登录服务器时自动调用此脚本。以下是一些您可以使用的典型函数：

## 第三章 ■ 配置高效环境

```bash
#-----------------------------------------------------------#
# 以排序列表显示环境变量
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
# 查找当前目录下最大的文件
function flf {
  find . -ls | sort -nrk7 | head -10
}

#-----------------------------------------------------------#
# 查找当前目录下占用空间最大的目录
function fld {
  du -S . | sort -nr | head -10
}

#-----------------------------------------------------------#
# cd 到 bdump 目录
function bdump {
  echo $ORACLE_HOME | grep 11 >/dev/null
  if [ $? -eq 0 ]; then
    lower_sid=$(echo $ORACLE_SID | tr '[:upper:]' '[:lower:]')
    cd $ORACLE_BASE/diag/rdbms/$lower_sid/$ORACLE_SID/trace
  else
    cd $ORACLE_BASE/admin/$ORACLE_SID/bdash
  fi
} # bdump

#-----------------------------------------------------------#
# 查看警报日志
function valert {
  echo $ORACLE_HOME | grep 11 >/dev/null
  if [ $? -eq 0 ]; then
    lower_sid=$(echo $ORACLE_SID | tr '[:upper:]' '[:lower:]')
    view $ORACLE_BASE/diag/rdbms/$lower_sid/$ORACLE_SID/trace/alert_$ORACLE_SID.log
  else
    view $ORACLE_BASE/admin/$ORACLE_SID/bdash/alert_$ORACLE_SID.log
  fi
} # valert

#-----------------------------------------------------------#
```

## `tbsp_chk.bsh`

此脚本用于检查是否有任何表空间超过了特定的已满阈值。将此脚本存储在 `HOME/bin` 这样的目录中。请确保修改脚本，使其包含适用于您环境的用户名、密码和电子邮件地址。

## 第三章 ■ 配置高效环境

您还需要建立所需的操作系统变量，如 `ORACLE_SID` 和 `ORACLE_HOME`。您可以将这些变量硬编码到脚本中，也可以调用一个为您设置这些变量的脚本。下一个脚本调用了一个名为 `oraset` 的脚本来建立操作系统变量。有关此脚本的详细信息请参见第二章。您不必使用此脚本——关键在于为您的环境建立一种一致且可重复的方式来设置操作系统变量。

您可以从命令行运行此脚本。例如，我传入了数据库名称 (`O11R2`)，并希望查看哪些表空间剩余空间不足 20%：

```bash
$ tbsp_chk.bsh O11R2 20
```

输出表明此数据库的两个表空间剩余空间不足 20%：

```
space not okay
0 % free UNDOTBS1, 17 % free SYSAUX,
```

以下是 `tbsp_chk.bsh` 脚本的内容：

```bash
#!/bin/bash
#
if [ $# -ne 2 ]; then
  echo "Usage: $0 SID threshold"
  exit 1
fi
# 可以硬编码操作系统变量，或者从脚本中 source 它们。
# 有关 oraset 脚本的详细信息请参见第 2 章
# source oracle 操作系统变量
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
     AND x.tablespace_name NOT LIKE 'UNDO%'
   GROUP BY x.tablespace_name) f,
  (SELECT SUM(bytes) bytes, tablespace_name
   FROM dba_data_files
   GROUP BY tablespace_name) a
WHERE a.tablespace_name = f.tablespace_name
  AND (f.bytes/a.bytes)*100 <= $2
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

## 第三章 ■ 配置高效环境

通常，您会从 `cron` 这样的调度工具中定期自动运行此类脚本。以下是一个典型的 `cron` 条目，每小时运行一次该脚本：

```bash
# 表空间检查
2 * * * * /orahome/bin/tbsp_chk.bsh INVPRD 10 1>/orahome/bin/log/tbsp_chk.log 2>&1
```

此 `cron` 条目运行作业并将任何信息输出存储在 `tbsp_chk.log` 文件中。

## `conn.bsh`

如果数据库连接出现问题，您需要收到警报。此脚本检查是否可以建立与数据库的连接。如果无法建立连接，则会发送一封电子邮件。

将此脚本放置在 `HOME/bin` 这样的目录中。请确保修改脚本，使其包含适用于您环境的用户名、密码和电子邮件地址。

您还需要建立所需的操作系统变量，如 `ORACLE_SID` 和 `ORACLE_HOME`。您可以将这些变量硬编码到脚本中，也可以调用一个为您设置这些变量的脚本。与上一个脚本类似，此脚本调用了一个名为 `oraset` 的脚本来建立操作系统变量。（参见第 2 章。）

该脚本需要将 `ORACLE_SID` 传递给它。例如：

```bash
$ conn.bsh INVPRD
```

如果脚本能够建立与数据库的连接，则显示以下消息：

```
success
db ok
```

以下是 `conn.bsh` 脚本的内容：

```bash
#!/bin/bash
if [ $# -ne 1 ]; then
  echo "Usage: $0 SID"
  exit 1
fi
# 可以硬编码操作系统变量，或者从脚本中 source 它们。
# 有关 oraset 脚本的详细信息请参见第 2 章
# source oracle 操作系统变量
. /var/opt/oracle/oraset $1
#
echo "select 'success' from dual;" | sqlplus -s darl/foo@INVPRD | grep success
if [[ $? -ne 0 ]]; then
  echo "problem with $1" | mailx -s "db problem" dkuhn@sun.com
else
  echo "db ok"
fi
#
exit 0
```

此脚本通常通过 `cron` 这样的工具实现自动化。以下是一个典型的 `cron` 条目：


