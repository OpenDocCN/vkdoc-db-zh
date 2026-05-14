# 自定义命令提示符

## 使用 Bash 变量

以下表格列出了一些可用于自定义命令提示符的 Bash shell 转义序列：

| 序列 | 描述 |
| :--- | :--- |
| `\v` | Bash shell 的版本 |
| `\V` | Bash shell 的发布版本 |
| `\w` | 当前工作目录 |
| `\W` | 当前工作目录的基本名称（非完整路径） |
| `\!` | 命令的历史编号 |
| `\$` | 如果有效用户标识符（`UID`）为 0，则显示 `#`；否则显示 `$` |

可用的命令提示符变量因操作系统（OS）和 Shell 而异。例如，在 Korn shell 环境中，`hostname` 变量会在 OS 提示符中显示服务器名称：

```
$ export PS1="[`hostname`]$ "
```

如果想在该字符串中包含 `ORACLE_SID` 变量，可以按如下方式设置：

```
$ export PS1=[`hostname`':"${ORACLE_SID}"]$ '
```

尽量不要在 OS 提示符中显示过多信息。信息过多会限制在单行输入和查看命令的能力。根据经验，至少应在 OS 提示符中显示服务器名称和数据库名称。拥有这些信息可以避免误以为身处一个环境而实际上在另一个环境中的错误。

## 自定义 SQL 提示符

DBA 经常使用 SQL*Plus 来执行日常管理任务。通常，你会在包含多个数据库的服务器上工作。显然，每个数据库包含多个用户账户。连接到数据库时，可以运行以下命令来验证用户名、数据库连接和主机名等信息：

```
SQL> show user;
SQL> select name from v$database;
```

这对于验证开发环境与生产环境账户以保持它们分离非常有用。使用 SQL 提示符提供快速的视觉提示，结合查询数据库，可以确保任何查询、更改等操作都在正确的环境中执行。

一个更有效的方法是设置 SQL 提示符来显示用户名和 SID；例如：

```
SQL> SET SQLPROMPT '&_USER.@&_CONNECT_IDENTIFIER.> '
```

一个更高效的方法是让 SQL*Plus 在登录时自动运行 `SET SQLPROMPT` 命令。请按照以下步骤完全自动化此过程：

1.  创建一个名为 `login.sql` 的文件，并在其中放入 `SET SQLPROMPT` 命令。
2.  设置你的 `SQLPATH` 操作系统变量以包含 `login.sql` 文件所在的目录位置。在此示例中，`SQLPATH` 操作系统变量在 `.bashrc` 操作系统文件中设置，该文件在每次登录或启动新 shell 时执行。条目如下：

```
    export SQLPATH=$HOME/scripts
    ```

3.  在 `$HOME/scripts` 目录中创建一个名为 `login.sql` 的文件。将以下行放入该文件：

```
    SET SQLPROMPT '&_USER.@&_CONNECT_IDENTIFIER.> '
    ```

4.  要查看结果，可以注销后重新登录服务器，或直接 source `.bashrc` 文件：

```
    $ . ./.bashrc
    ```

现在，登录到 SQL。以下是 SQL*Plus 提示符的示例：

```
SYS@devdb1>
```

如果连接到不同的用户，提示符中应有所反映：

```
SQL> conn system/foo
```

SQL*Plus 提示符现在显示：

```
SYSTEM@devdb1>
```

设置 SQL 提示符是提醒自己当前连接的环境和用户的简单方法。这有助于防止意外在错误环境中执行 SQL 语句。最不希望发生的事情就是以为在开发环境中，却发现在连接生产环境时运行了删除对象的脚本。

表 3-2 包含可用来自定义提示符的 SQL*Plus 变量的完整列表。

### 表 3-2 预定义的 SQL*Plus 变量

| 变量 | 描述 |
| :--- | :--- |
| `_CONNECT_IDENTIFIER` | 连接标识符，例如 Oracle SID |
| `_DATE` | 当前日期 |
| `_EDITOR` | SQL `EDIT` 命令使用的编辑器 |
| `_O_VERSION` | Oracle 版本 |
| `_O_RELEASE` | Oracle 发布版本 |
| `_PRIVILEGE` | 当前连接会话的权限级别 |
| `_SQLPLUS_RELEASE` | SQL*Plus 发布编号 |
| `_USER` | 当前连接的用户 |

## 为常用命令创建快捷方式

在 Linux/Unix 环境中，可以使用两种常见方法为其他命令创建快捷方式：为经常重复的命令创建别名，以及使用函数为命令组创建快捷方式。以下部分描述了部署这两种技术的方法。

### 使用别名

别名是一种简单的机制，用于创建一小段文本，该文本将执行其他 shell 命令。通用语法如下：

```
$ alias 别名='命令'
```

例如，遇到数据库问题时，创建一个运行 `cd` 命令的别名将你置于包含数据库警报日志的目录中通常很有用。此示例创建一个别名（命名为 `bdump`），将当前工作目录更改为警报日志所在的位置：

```
$ alias bdump='cd /u01/app/oracle/diag/rdbms/o18c/o18c/trace'
```

现在，无需输入 `cd` 命令及冗长（且容易忘记）的目录路径，只需键入 `bdump`，即可进入指定目录：

```
$ bdump
$ pwd
/u01/app/oracle/diag/rdbms/o18c/o18c/trace
```

上述技术允许你高效准确地导航到目标目录。当你在多台服务器上管理许多不同的数据库时，这尤其方便。你只需设置一套标准的别名，以便更高效地导航和工作。

要显示所有已定义的别名，请使用不带参数的 `alias` 命令：

```
$ alias
```

下面列出了一些你可以使用的常见别名定义示例：

```
alias l.='ls -d .*'
alias ll='ls -l'
alias lsd='ls -altr | grep ^d'
alias sqlp='sqlplus "/ as sysdba"'
alias shutdb='echo "shutdown immediate;" | sqlp'
alias startdb='echo "startup;" | sqlp'
```

如果想从当前环境中删除别名定义，请使用 `unalias` 命令。以下示例移除了 `lsd` 的别名：

```
$ unalias lsd
```

## 定位警报日志

在 Oracle Database 11g 及更高版本中，警报日志目录路径具有以下结构：

```
ORACLE_BASE/diag/rdbms/数据库唯一名称/SID/trace
```

通常（但并非总是）数据库唯一名称（`db_unique_name`）与实例名称（`instance_name`）相同。在 Data Guard 环境中，数据库唯一名称通常与实例名称不同。你可以使用以下查询验证目录路径：

```
SQL> select value from v$diag_info where name = 'Diag Trace';
```

警报日志的名称遵循以下格式：

```
alert_<SID>.log
```

你也可以从操作系统定位警报日志（无论数据库是否已启动），使用以下操作系统命令：

```
$ cd $ORACLE_BASE
$ find . -name alert_<ORACLE_SID>.log
```

在上面的 `find` 命令中，你需要将 `<ORACLE_SID>` 值替换为你的数据库名称。

### 使用函数

与别名类似，你也可以使用函数来形成命令快捷方式。函数的通用定义语法如下：

```
$ function 函数名 {
shell 命令
}
```

例如，以下代码行创建了一个简单的函数（命名为 `bdump`），该函数根据传入的数据库名称更改当前工作目录：

```
function bdump {
if [ "$1" = "engdev" ]; then
cd /orahome/app/oracle/diag/rdbms/engdev/ENGDEV/trace
elif [ "$1" = "stage" ]; then
cd /orahome/app/oracle/diag/rdbms/stage/STAGE/trace
fi
echo "Changing directories to $1 Diag Trace directory"
pwd
}
```

你现在可以在命令行键入 `bdump` 后跟数据库名称，以将工作目录更改为 Oracle 后台转储目录：

```
$ bdump stage
Changing directories to stage Diag Trace directory
/orahome/app/oracle/diag/rdbms/stage/STAGE/trace
```

使用函数通常比使用别名更可取。函数比别名更强大，因为它们具备诸如能够在命令行上操作传入的参数、允许多行代码以及因此实现更复杂功能等特性。

DBA 通常通过在 `HOME/.bashrc` 文件中设置函数来创建函数。管理函数的一种更好方法是创建一个专门存储函数代码的文件，然后从 `.bashrc` 文件中调用该文件。将特殊用途的文件存放在您为此创建的目录中也是更好的做法。例如，在 `HOME` 下创建一个名为 `bin` 的目录。然后，在 `bin` 目录中创建一个名为 `dba_fcns` 的文件，并将您的函数代码放入其中。现在，从 `.bashrc` 文件中调用 `dba_fcns` 文件。以下是在 `.bashrc` 文件中的一个条目示例：

```
. $HOME/bin/dba_fcns
```

接下来列出的是您可以使用的函数类型的一小部分示例：

```
# 显示环境变量并按排序列表输出
function envs {
if test -z "$1"
then /bin/env | /bin/sort
else /bin/env | /bin/sort | /bin/grep -i $1
fi
} # envs
#-----------------------------------------------------------#

# 查找当前位置下最大的文件
function flf {
find . -ls | sort -nrk7 | head -10
}
#-----------------------------------------------------------#
```


