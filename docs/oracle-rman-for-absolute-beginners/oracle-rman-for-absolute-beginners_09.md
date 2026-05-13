# Oracle 环境变量设置与用户认证

## 一、 关键环境变量
`ORACLE_SID`（系统标识符）变量定义了您将要连接的数据库的默认名称。`ORACLE_SID` 也用于确定参数文件的默认名称，即 `init<ORACLE_SID>.ora` 或 `spfile<ORACLE_SID>.ora`。默认情况下，Oracle 在 Linux/Unix 系统上会在 `ORACLE_HOME/dbs` 目录中查找这些初始化文件，在 Windows 系统上则在 `ORACLE_HOME\database` 目录中查找。初始化文件包含管理数据库各个方面的参数，例如为数据库分配多少内存、最大连接数等。

`LD_LIBRARY_PATH` 变量非常重要，因为它指定了在 Linux/Unix 系统上搜索库文件的位置。该变量的值通常设置为包含 `ORACLE_HOME/lib`。

`PATH` 变量指定了当您在操作系统提示符下键入命令时默认查找的目录。在几乎所有情况下，`ORACLE_HOME/bin`（Oracle 二进制文件的位置）都必须包含在您的 `PATH` 变量中。

您可以手动设置这些变量，也可以使用 Oracle 提供的标准脚本来设置。

## 二、 手动设置变量
在 Linux/Unix 中，当您使用 Bourne、Bash 或 Korn shell 时，可以使用以下 `export` 命令从操作系统命令行手动设置操作系统变量：

```bash
$ export ORACLE_HOME=/orahome/app/oracle/product/12.1.0.1/db_1
$ export ORACLE_SID=O12C
$ export LD_LIBRARY_PATH=/usr/lib:$ORACLE_HOME/lib
$ export PATH=$ORACLE_HOME/bin:$PATH
```

请注意，上面的命令是针对我特定的开发环境的；您需要调整它们以匹配您环境中使用的 Oracle 主目录和数据库名称。

对于 C 或 `tcsh` shell，请使用 `setenv` 命令设置变量：

```bash
$ setenv ORACLE_HOME <path>
$ setenv ORACLE_SID <sid>
$ setenv LD_LIBRARY_PATH <path>
$ setenv PATH <path>
```

DBA 设置这些变量的另一种方式是将之前的 `export` 或 `setenv` 命令放入 Linux/Unix 启动文件中，例如 `.bash_profile`、`.bashrc` 或 `.profile`。这样，变量会在登录时自动设置。

然而，手动设置操作系统变量（无论是从命令行还是通过在启动文件中硬编码值）并不是实例化这些变量的最佳方式。例如，如果您在一台服务器上有多个具有多个 Oracle 主目录的数据库，手动设置这些变量很快就会变得难以管理且不易维护。

## 三、 使用 Oracle 脚本
设置操作系统变量的一个好方法是使用一个脚本，该脚本利用一个包含服务器上所有 Oracle 数据库名称及其关联 Oracle 主目录的文件。这种方法灵活且易于维护。例如，如果某个数据库的 Oracle 主目录发生变化（例如，升级后），您只需在服务器上修改一个文件，而不必去查找硬编码在脚本中的 Oracle 主目录变量。

Oracle 提供了一种自动设置所需操作系统变量的机制。此方法依赖于两个文件：`oratab` 和 `oraenv`。

### 3.1 理解 oratab
您可以将 `oratab` 文件中的条目视为一台服务器上安装的数据库及其对应 Oracle 主目录的注册表。当您安装 Oracle 软件时，`oratab` 文件会自动为您创建。在 Linux 服务器上，`oratab` 通常位于 `/etc` 目录中。在 Solaris 服务器上，`oratab` 文件位于 `/var/opt/oracle` 目录中。如果由于某种原因 `oratab` 文件未自动创建，您可以手动创建它（使用文本编辑器）。

`oratab` 文件在 Linux/Unix 环境中用于以下目的：

*   自动设置所需的操作系统变量
*   自动启动和停止服务器上的 Oracle 数据库

`oratab` 文件有三列，格式如下：

```
<database_sid>:<oracle_home_dir>:Y|N
```

`Y` 或 `N` 表示您是否希望 Oracle 在服务器重启时自动重启；`Y` 表示是，`N` 表示否（自动重启功能需要本书未涉及的额外任务）。

`oratab` 文件中的注释以井号（`#`）开头。以下是一个典型的 `oratab` 文件条目：

```
O12C:/orahome/app/oracle/product/12.1.0.1/db_1:N
ORA12CR1:/orahome/app/oracle/product/12.1.0.1/db_1:N
```

上一行中的数据库名称是 `O12C` 和 `ORA12CR1`。每个数据库的 Oracle 主目录路径紧接在行中（与数据库名用冒号 [`:`] 分隔）。

多个 Oracle 提供的实用程序使用 `oratab` 文件：

*   `oraenv` 使用 `oratab` 来设置操作系统变量。
*   `dbstart` 使用它在服务器重启时自动启动数据库（如果 `oratab` 中的第三个字段是 `Y`）。
*   `dbshut` 使用它在服务器重启时自动停止数据库（如果 `oratab` 中的第三个字段是 `Y`）。

`oraenv` 工具将在下一节讨论。

### 3.2 使用 oraenv
如果您没有为 Oracle 环境正确设置所需的操作系统变量，那么诸如 SQL*Plus、RMAN、Data Pump 等实用程序将无法正常工作。`oraenv` 实用程序可自动在 Oracle 数据库服务器上设置所需的操作系统变量（如 `ORACLE_HOME`、`ORACLE_SID` 和 `PATH`）。此实用程序用于 Bash、Korn 和 Bourne shell 环境（如果您在 C shell 环境中，有一个对应的 `coraenv` 实用程序）。

`oraenv` 实用程序位于 `ORACLE_HOME/bin` 目录中。您必须先导航到您的 `ORACLE_HOME/bin` 目录（您需要修改以下路径以匹配您的环境）：

```bash
$ cd /orahome/app/oracle/product/12.1.0.1/db_1/bin
```

然后您可以手动运行 `oraenv`，像这样：

```bash
$ . ./oraenv
```

系统将提示您输入 `ORACLE_SID`（并且如果 `ORACLE_SID` 不在 `oratab` 文件中，您还将被提示输入 `ORACLE_HOME` 值）：

```
ORACLE_SID = [oracle] ?
ORACLE_HOME = [/home/oracle] ?
```

您也可以在运行 `oraenv` 实用程序之前设置操作系统变量，以非交互方式运行它。这在您不希望提示输入的脚本编写中很有用：

```bash
$ export ORACLE_SID=O12C
$ export ORACLE_HOME=/orahome/app/oracle/product/12.1.0.1/db_1
$ export ORAENV_ASK=NO
$ cd /orahome/app/oracle/product/12.1.0.1/db_1/bin
$ . ./oraenv
```

![Image](img/sq.jpg) **注意** 在 Windows 操作系统中，变量在注册表中设置。

您可以使用 `echo` 命令验证操作系统变量设置，例如：

```bash
$ echo $ORACLE_SID
O12C

$ echo $ORACLE_HOME
/orahome/app/oracle/product/12.1.0.1/db_1
```

## 四、 连接到数据库
建立操作系统变量后，您需要使用适当的权限连接到数据库。您可以通过两种方式之一完成此操作：使用操作系统认证或使用密码文件。

### 4.1 使用操作系统认证
在连接到 Oracle 数据库之前，您需要设置正确的操作系统变量（如前一节所述）。此外，如果您想以特权用户身份连接到 Oracle，那么您还必须有权访问特权操作系统账户或特权数据库用户。以特权用户身份连接允许您执行管理任务，例如启动和停止数据库。您可以使用操作系统认证或密码文件以特权用户身份连接到数据库。

特权用户的概念对于 RMAN 备份和恢复也很重要。RMAN 使用操作系统认证和密码文件来允许特权用户建立特权数据库会话（通过 `rman` 实用程序）。只有特权账户才允许备份、恢复和恢复数据库。

如果您的 Linux/Unix 账户是 `dba` 组的成员（您的公司可能使用不同的组名，但 `dba` 是最常见的），那么凭借登录到您的 Linux/Unix 账户，您就可以通过 SQL*Plus 以所需的特权连接到您的数据库。



