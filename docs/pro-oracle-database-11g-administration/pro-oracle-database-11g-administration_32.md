# 第三章 配置高效环境

> **提示**：在数据库服务器上工作时，始终使用一种操作系统 shell。我建议您使用 Bash shell；它包含了来自其他 shell（Korn 和 C）的所有最有用的功能，并且还具有增加其易用性的附加功能。

## 自定义操作系统命令提示符

通常，DBA 与多台服务器和多个数据库一起工作。在这些情况下，您可能在屏幕上打开了许多终端会话。您可以运行以下类型的命令来标识您当前的工作环境：

```bash
$ hostname -a
$ id
$ who am i
$ echo $ORACLE_SID
$ pwd
```

为了避免混淆您正在使用的服务器，通常希望配置您的命令提示符以显示有关其环境的信息，例如计算机名和数据库 `SID`。在此示例中，命令提示符名称被自定义为包含主机名、用户和 `Oracle SID`：

```bash
$ PS1='[\h:\u:${ORACLE_SID}]$ '
```

`\h` 指定主机名。`\u` 指定当前操作系统用户。`$ORACLE_SID` 包含 `Oracle` 实例标识符的当前设置。以下是此示例中命令提示符现在的样子：

```
[ora03:oracle:devdb1]$
```

命令提示符包含有关环境的三个重要信息：服务器名、操作系统用户名和数据库名称。当您在多个环境之间导航时，设置命令提示符对于跟踪您所在的位置和所处的环境来说是一个非常宝贵的工具。

如果您希望在登录时自动配置操作系统提示符，那么您需要在启动文件中设置它。在 Bash shell 环境中，通常使用 `.bashrc` 文件。此文件通常位于您的 `HOME` 目录中。将以下代码行放入 `.bashrc` 中：

```bash
PS1='[\h:\u:${ORACLE_SID}]$ '
```

当您将此代码行放入启动文件时，那么每当您登录服务器时，操作系统提示符都会自动为您设置。在其他 shell 中，例如 Korn shell，`.profile` 文件是启动文件。

根据您的个人偏好，您可能希望修改命令提示符以满足您的特定需求。例如，许多 DBA 喜欢在命令提示符中显示当前工作目录。要显示当前工作目录信息，请添加 `\w` 变量：

```bash
$ PS1='[\h:\u:\w:${ORACLE_SID}]$ '
```

正如您可以想象的那样，可用于命令提示符中显示的信息有多种选项。以下是另一种流行的格式：

```bash
$ PS1='[\u@${ORACLE_SID}@\h:\w]$ '
```

**表 3–1** 列出了许多可用于自定义操作系统命令提示符的 Bash shell 变量。

**表 3–1. 用于自定义命令提示符的 Bash Shell 反斜杠转义变量**
| **变量** | **描述** |
| :--- | :--- |
| `\a` | ASCII 响铃字符 |
| `\d` | 日期，格式为“星期几 月份 日期” |
| `\h` | 主机名 |
| `\e` | ASCII 转义字符 |
| `\j` | shell 管理的作业数 |
| `\l` | shell 的终端设备的基本名称 |
| `\n` | 换行符 |
| `\r` | 回车符 |
| `\s` | shell 的名称 |
| `\t` | 时间，24 小时制 HH:MM:SS 格式 |
| `\T` | 时间，12 小时制 HH:MM:SS 格式 |
| `\@` | 时间，12 小时制 a.m./p.m. 格式 |
| `\A` | 时间，24 小时制 HH:MM 格式 |
| `\u` | 当前用户的用户名 |
| `\v` | Bash shell 的版本 |
| `\V` | Bash shell 的发布版本 |
| `\w` | 当前工作目录 |
| `\W` | 当前工作目录的基本名称 |
| `\!` | 命令的历史编号 |
| `\$` | 如果有效 UID 为 0，则显示 `#`；否则，显示 `$` |

可用于命令提示符的变量因操作系统和 shell 而异。例如，在 Korn shell 环境中，主机名变量在操作系统提示符中显示服务器名称：

```bash
$ export PS1="[`hostnamè]$ "
```

## 自定义 SQL 提示符

DBA 经常使用 `SQL*Plus` 来执行日常管理任务。通常，您将在包含多个数据库的服务器上工作。显然，每个数据库包含多个用户账户。连接到数据库时，您可以运行以下命令来验证您的用户名和数据库连接：

```sql
SQL> show user;
SQL> select name from v$database;
```

确定用户名和 `SID` 的更有效方法是设置 `SQL` 提示符以显示该信息。例如：

```sql
SQL> SET SQLPROMPT '&_USER.@&_CONNECT_IDENTIFIER.> '
```

配置 `SQL` 提示符的更有效方法是让您在登录 `SQL*Plus` 时自动运行 `SET SQLPROMPT` 命令。请按照以下步骤完全自动化此操作：
1.  创建一个名为 `login.sql` 的文件，并在其中放入 `SET SQLPROMPT` 命令。
2.  设置您的 `SQLPATH` 操作系统变量以包含 `login.sql` 的目录位置。在此示例中，`SQLPATH` 操作系统变量在 `.bashrc` 操作系统文件中设置，该文件在每次登录或启动新 shell 时执行。以下是条目：

```bash
export SQLPATH=$HOME/scripts
```
3.  在 `HOME/scripts` 目录中创建一个名为 `login.sql` 的文件。在文件中放入以下行：

```sql
SET SQLPROMPT '&_USER.@&_CONNECT_IDENTIFIER.> '
```
4.  要查看结果，您可以注销并重新登录到服务器，或者直接 source `.bashrc` 文件：

```bash
$ . ./.bashrc
```

现在，登录到 SQL。以下是 `SQL*Plus` 提示符的示例：

```
SYS@devdb1>
```

如果您连接到不同的用户，这应该反映在提示符中：

```sql
SQL> conn system/foo
```

`SQL*Plus` 提示符现在显示为

```
SYSTEM@devdb1
```

设置 `SQL` 提示符是提醒您当前连接的环境和用户的简便方法。这将有助于防止您在错误的环境中意外运行 SQL 语句。您最不希望发生的事情是以为自己在开发环境中，然后发现您在连接生产环境时运行了删除对象的脚本。

**表 3–2** 包含可用于自定义提示符的 `SQL*Plus` 变量的完整列表。

**表 3–2. 预定义的 SQL*Plus 变量**
| **变量** | **描述** |
| :--- | :--- |
| `_CONNECT_IDENTIFIER` | 连接标识符，例如 Oracle SID |
| `_DATE` | 当前日期 |
| `_EDITOR` | `SQL EDIT` 命令使用的编辑器 |
| `_O_VERSION` | Oracle 版本 |
| `_O_RELEASE` | Oracle 发布版本 |
| `_PRIVILEGE` | 当前连接会话的权限级别 |
| `_SQLPLUS_RELEASE` | `SQL*Plus` 版本号 |
| `_USER` | 当前连接的用户 |

## 为常用命令创建快捷方式

在 Linux/Unix 环境中，您可以使用两种常用方法为其他命令创建快捷方式：为经常重复的命令创建别名，以及使用函数为一组命令创建快捷方式。以下部分描述了您可以使用这两种技术的方法。

### 使用别名

您经常需要导航到服务器上的各个目录。例如，其中一个位置是数据库后台进程日志记录目录。要导航到此目录，您必须键入类似以下内容：

```bash
$ cd /ora01/app/oracle/admin/O10R24/bdump
```

您可以使用 `alias` 命令创建一个快捷方式来完成相同的任务。*别名* 是创建一小段文本以执行其他 shell 命令的简单机制。此示例创建一个名为 `bdump` 的别名，该别名根据 `ORACLE_SID` 变量的值更改目录到后台位置：

```bash
$ alias bdump='cd /ora01/app/oracle/admin/$ORACLE_SID/bdump'
```

现在您可以键入 `bdump`，这与将当前工作目录更改为 Oracle 后台转储目录相同。

要显示所有已定义的别名，请使用不带参数的 `alias` 命令：

```bash
$ alias
```

接下来列出了一些可以使用的常见别名定义示例：

```bash
alias l.='ls -d .*'
alias ll='ls -l'
alias lsd='ls -altr | grep ^d'
alias bdump='cd /ora01/app/oracle/admin/$ORACLE_SID/bdump'
alias sqlp='sqlplus "/ as sysdba"'
alias shutdb='echo "shutdown immediate;" | sqlp'
alias startdb='echo "startup;" | sqlp'
```

如果您想从当前环境中删除别名定义，请使用 `unalias` 命令。以下示例删除了 `lsd` 的别名：

```bash
$ unalias lsd
```

### 使用函数

您也可以使用函数创建命令快捷方式。以下代码行创建了一个名为 `bdump` 的简单函数：

```bash
$ function bdump { cd /ora01/app/oracle/admin/$ORACLE_SID/bdump; }
```

您现在可以在命令行键入 `bdump` 以将工作目录更改为 Oracle 后台转储目录。

使用函数通常比使用别名更可取。函数比别名更强大，因为它们具有诸如能够在命令行上传递的参数上操作以及允许复杂编码等功能。

为了演示函数的强大功能，请考虑这样一种场景：您在服务器上安装了不同版本的数据库，并且后台转储目标根据版本而异。使用函数，您可以构建逻辑来检查您使用的是哪个版本的数据库并相应地导航：

```bash
function bdump {
echo $ORACLE_HOME | grep 11 >/dev/null
if [ $? -eq 0 ]; then
lower_sid=$(echo $ORACLE_SID | tr '[:upper:]' '[:lower:]')
cd $ORACLE_BASE/diag/rdbms/$lower_sid/$ORACLE_SID/trace
else
cd $ORACLE_BASE/admin/$ORACLE_SID/bdump
fi
} # bdump
```

前面的函数代码行很难用别名复制。使用函数，您可以根据需要编写任意多的逻辑，并在需要时传入变量。

DBA 通常通过在 `HOME/.bashrc` 文件中设置函数来建立函数。管理函数的更好方法是创建一个仅存储函数代码的文件，并从 `.bashrc` 文件中调用该文件。

将特殊用途的文件存储在您为这些文件创建的目录中也更好。例如，在 `HOME` 下创建一个名为 `bin` 的目录；在 `bin` 目录中，创建一个名为 `dba_fcns` 的文件，并将您的函数代码放入其中。现在，从 `.bashrc` 文件调用 `dba_fcns` 文件。以下是 `.bashrc` 文件中的条目示例：

```bash
. $HOME/bin/dba_fcns
```

接下来列出了一些您可以使用的函数类型的小样本：

```bash
#-----------------------------------------------------------#
```


## 查找当前目录下的最大文件

```
function flf {
find . -ls | sort -nrk7 | head -10
}
```

## 查找当前目录下占用空间最大的目录

```
function fld {
du -S . | sort -nr | head -10
}
```


