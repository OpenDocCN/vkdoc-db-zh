# 配置和实现监听器

在安装了二进制文件并创建了数据库之后，您需要使数据库能够接受远程客户端连接。这通过配置和启动 Oracle 监听器来实现。顾名思义，监听器是数据库服务器上“监听”来自远程客户端连接请求的进程。如果数据库服务器上没有启动监听器，则无法从远程客户端连接。
监听器可以包含在数据库主目录或网格主目录中。监听器只需要存在一次，因为只能有一个活动的网格主目录，并且可能存在多个数据库主目录。这是一个管理和维护监听器的地方。将监听器作为网格环境的一部分，允许其补丁更新独立于数据库，并与网格作为基础设施补丁过程的一部分进行。同时，在维护监听器时，请注意设置正确的 `ORACLE_HOME`，以确保监听器在所需的环境中运行。接下来的两种方法展示了在数据库中配置监听器，但同样可以轻松地应用于网格环境。
设置监听器有两种方法：Oracle Net 配置助手 (`netca`) 或手动配置 `listener.ora` 文件。

## 使用网络配置助手实现监听器

`netca` 实用程序可帮助您完成实现监听器的各个方面。您可以在图形模式或静默模式下运行 `netca` 工具。在图形模式下使用 `netca` 简单直观。要在图形模式下使用 `netca`，请确保安装了适当的 X 软件，然后发出 `xhost +` 命令，并检查您的 `DISPLAY` 变量是否已设置；例如，

```
$ xhost +
$ echo $DISPLAY
:0.0
```

现在可以运行 `netca` 实用程序：

```
$ netca
```

接下来，您将被引导通过几个屏幕，从中可以选择监听器名称、所需端口等选项。

您也可以使用响应文件以静默模式运行 `netca` 实用程序。此模式允许您编写脚本并确保创建和实施监听器时的可重复性。首先，在包含 Oracle 安装介质的目录结构中找到默认的监听器响应文件：

```
$  find . -name "netca.rsp"
./18.0.0.0/database/response/netca.rsp
```

现在，复制该文件以便修改：

```
$ cp 18.0.0.0/database/response/netca.rsp mynet.rsp
```

如果要更改默认名称或其他属性，请使用 `vi` 等操作系统实用程序编辑 `mynet.rsp` 文件：

```
$ vi mynet.rsp
```

在此示例中，我没有修改 `mynet.rsp` 文件中的任何值。换句话说，我使用了响应文件中已包含的所有默认值。接下来，以静默模式运行 `netca` 实用程序：

```
$ netca -silent -responsefile /home/oracle/orainst/mynet.rsp
```

该实用程序会在 `ORACLE_HOME/network/admin` 目录中创建 `listener.ora` 和 `sqlnet.ora` 文件，并启动一个默认监听器。

## 手动配置监听器

在设置新环境时，手动配置监听器是一个两步过程：

1.  配置 `listener.ora` 文件。
2.  启动监听器。

`listener.ora` 文件默认位于 `ORACLE_HOME/network/admin` 目录中。这也应该是 `TNS_ADMIN` 操作系统变量设置指向的目录。手动配置监听器和更新 `listener.ora` 文件时，请注意括号和任何特殊字符，配置错误的 `listener.ora` 将导致无法启动，而同一端口上运行的另一个监听器是首先需要检查的地方。

以下是一个 `listener.ora` 文件示例，其中包含一个数据库的网络配置信息：

```
LISTENER =
(DESCRIPTION_LIST =
(DESCRIPTION =
(ADDRESS_LIST =
(ADDRESS = (PROTOCOL = TCP)(HOST = oracle18c)(PORT = 1521))
)
)
)
SID_LIST_LISTENER =
(SID_LIST =
(SID_DESC =
(GLOBAL_DBNAME = o18c)
(ORACLE_HOME = /u01/app/oracle/product/18.0.0.0/db_1)
(SID_NAME = o18c)
)
)
```

此代码清单包含两个部分。第一部分定义了监听器名称和服务；在此示例中，监听器名称为 `LISTENER`。第二部分定义了监听器正在监听传入连接（到数据库）的 SID 列表。SID 列表名称的格式为 `SID_LIST_<监听器名称>`。监听器的名称必须出现在 SID 列表名称中。此示例中的 SID 列表名称为 `SID_LIST_LISTENER`。
此外，您不必在上面的代码清单中显式指定 `SID_LIST_LISTENER` 部分（第二部分）。这是因为进程监控器（`PMON`）后台进程会自动将任何正在运行的数据库作为服务注册到监听器；这称为动态注册。然而，一些 DBA 偏好显式列出应注册到监听器的数据库，因此包含第二部分；这称为静态注册。

在 `listener.ora` 文件就位后，您可以使用 `lsnrctl` 实用程序启动监听器后台进程：

```
$ lsnrctl start
```

您应该会看到信息性消息，例如：

```
Listening Endpoints Summary...
(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST = oracle18c)(PORT=1521)))
Services Summary...
Service "o18c" has 1 instance(s).
```



