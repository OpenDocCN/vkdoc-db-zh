# 组织脚本

当您拥有一组脚本和实用程序时，您应该对它们进行组织，以便为每个数据库服务器一致地实施。它们应成为您安装 Oracle 二进制文件后步骤的一部分。
这些脚本不仅能够作为此过程的一部分一致地部署，还可以用于测试数据库的安装和设置。请按照以下步骤为环境中的每个数据库服务器实施上述 DBA 实用程序：

1.  创建用于存储脚本的操作系统目录。
2.  将您的脚本和实用程序复制到步骤 1 中创建的目录。
3.  配置您的启动文件以初始化环境。

这些步骤在以下部分中有详细说明。

## 步骤 1. 创建目录

在每个数据库服务器上创建一组标准目录来存储您的自定义脚本。`oracle`用户`HOME`目录下的一个目录通常是一个不错的位置。我通常创建以下三个目录：

*   `HOME/bin`。用于存放以自动方式（例如从`cron`）运行的 Shell 脚本的标准位置。
*   `HOME/bin/log`。用于存放由计划 Shell 脚本生成的日志文件的标准位置。
*   `HOME/scripts`。用于存储 SQL 脚本的标准位置。

您可以使用`mkdir`命令创建上述目录，如下所示：

```
$ mkdir -p $HOME/bin/log
$ mkdir $HOME/scripts
```

脚本放置在何处或目录如何命名并不重要，只要您有一个标准位置，这样当您在服务器之间导航时，总能在相同位置找到相同的文件即可。换句话说，标准本身是什么并不重要，重要的是您有一个标准。

## 步骤 2. 将文件复制到目录

将您的实用程序和脚本放入相应的目录。将以下文件复制到`HOME/bin`目录：

```
dba_setup
dba_fcns
tbsp_chk.bsh
conn.bsh
filesp.bsh
```

将以下 SQL 脚本放入`HOME/scripts`目录：

```
login.sql
top.sql
lock.sql
users.sql
```

## 步骤 3. 配置启动文件

将以下代码放入`.bashrc`文件或您使用的 Shell 对应的等效启动文件中（对于 Korn shell 是`.profile`）。以下是如何配置`.bashrc`文件的示例：

```

```



```
# 引入全局定义
if [ -f /etc/bashrc ]; then
. /etc/bashrc
fi
#
# 引入 Oracle 数据库操作系统变量
. /etc/oraset 
#
```


The request was rejected because it was considered high risk


