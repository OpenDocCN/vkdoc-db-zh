# /ora01/app/oracle/product/10.2.0.4/db_1/root.sh

我通常接受默认设置。`root.sh`脚本运行后，您应已拥有一个良好的 Oracle 软件安装，并可以创建数据库（数据库创建将在第 2 章中介绍）。

#### Oracle Database 11g 场景

以下是随 Oracle Database `11g` release 2 在 Linux 服务器上提供的响应文件：
```
$ find . -name "*.rsp"
./response/db_install.rsp
./response/dbca.rsp
./response/netca.rsp
```

复制其中一个响应文件以便修改。本例将`db_install.rsp`文件复制到当前工作目录，并命名为`inst.rsp`：
```
$ cp response/db_install.rsp inst.rsp
```

请记住，响应文件的格式会因 Oracle 数据库版本不同而有较大差异。例如，从 Oracle Database `11g` release 1 到 Oracle Database `11g` release 2 就存在重大差异。安装新版本时，您必须检查响应文件并确定哪些参数必须设置。以下是 Oracle Database `11g` release 2 响应文件的部分内容（以下文件的前两行本应在一行内，但本书中将其放在两行以适应页面）：

```
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v11_2_0
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=ora03
UNIX_GROUP_NAME=dba
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=dba
INVENTORY_LOCATION=/ora01/orainst/11.2.0.1/database/stage/products.xml
SELECTED_LANGUAGES=en
ORACLE_HOME=/oracle/app/oracle/product/11.2.0/db_1
ORACLE_BASE=/oracle/app/oracle
DECLINE_SECURITY_UPDATES=true
oracle.install.db.InstallEdition=EE
oracle.install.db.isCustomInstall=true
```

请务必根据您的环境修改相应的参数。如果不确定`ORACLE_HOME`和`ORACLE_BASE`的值应设置为何值，请参阅本章第一节“理解最佳灵活架构（OFA）”，其中描述了 OFA 标准目录。

这些参数有时会因版本而异，存在一些特殊性。例如，在 Oracle Database `11g` release 2 中，如果您不想提供 My Oracle Support（MOS）登录信息，则需要设置以下参数：

`DECLINE_SECURITY_UPDATES=true`

如果您未将`DECLINE_SECURITY_UPDATES`设置为`TRUE`，则系统会要求您提供 MOS 登录信息。如果未提供，安装将无法成功。

配置好响应文件后，您可以以静默模式运行 Oracle 安装程序。请注意，您需要提供响应文件位置的完整目录路径：
```
$ ./runInstaller -ignoreSysPrereqs -force -silent -responseFile /ora01/orainst/11.2.0.1/database/inst.rsp
```
上一条命令输入在两行中。第一行通过`\`（反斜杠）字符延续到第二行。

如果安装过程中遇到错误，您可以查看相关的日志文件。每次尝试运行安装程序时，它都会创建一个包含时间戳的唯一名称的日志文件。日志文件创建在`oraInventory/logs`目录中。您可以在 OUI 写入日志文件时，将输出流式传输到屏幕上：
```
$ tail -f <日志文件名>
```
日志文件的命名类似于：
```
installActions2009-04-25_11–42-51AM.log
```

如果一切运行成功，输出中会通知您需要以 root 用户身份运行`root.sh`脚本：
```
#需要运行的 Root 脚本
/oracle/app/oracle/product/11.2.0/db_1/root.sh
```

以 root 操作系统用户身份运行`root.sh`脚本。之后，您应该能够创建 Oracle 数据库（数据库创建将在第 2 章中介绍）。

**注意** 在 Linux/Unix 平台上，`root.sh`脚本包含必须以 root 用户身份运行的命令。此脚本需要修改一些 Oracle 可执行文件（如`nmo`可执行文件）的所有者和权限。

某些版本的`root.sh`会提示您是否接受默认值。通常，接受默认值是合适的。

## 第 6 步：对任何问题进行故障排查

如果在使用响应文件时遇到错误，90%的情况下是由于响应文件中变量设置的方式有问题。请仔细检查这些变量，确保它们设置正确。此外，如果您未完全指定响应文件的命令行路径，您会收到如下错误：
```
OUI-10203: 未找到指定的响应文件 'enterprise.rsp'。
```

当响应文件的路径或名称指定错误时，会出现另一个常见错误：
```
OUI-10202: 此会话未指定响应文件。
```

如果您在响应文件的`FROM_LOCATION`变量中输入了错误的`products.xml`文件路径，您将收到以下错误消息：
```
OUI-10133: 无效的暂存区域
```

另外，在运行响应文件时，请确保提供正确的命令行语法。如果您错误地指定或拼写错了选项，可能会收到误导性的错误消息，例如`DISPLAY not set`。使用响应文件时，您无需设置`DISPLAY`变量。此消息令人困惑，因为在此场景中，错误是由错误指定的命令行选项引起的，与`DISPLAY`变量无关。请检查从命令行输入的所有选项，确保您没有拼写错误。

另一个常见问题发生在您指定了`ORACLE_HOME`，而静默安装程序认为给定的主目录已存在：
```
检查结果：失败 <<<<
问题：Oracle Database 11g Release 2 只能安装在新的 Oracle 主目录中。
建议：为此产品安装选择一个新的 Oracle 主目录。
```

检查您的`inventory.xml`文件（位于`oraInventory/ContentsXML`目录中），确保不存在与已存在的 Oracle 主目录名称冲突的情况。

当您对 Oracle 安装问题进行故障排查时，请记住安装程序使用两个关键文件来跟踪已安装的软件及其位置：`oraInst.loc`和`inventory.xml`。

表 1-1 描述了 Oracle 安装程序使用的文件。

**表 1-1.** 用于排查 Oracle 安装问题的有用文件

| 文件名 | 目录位置 | 内容 |
| :--- | :--- | :--- |
| `oraInst.loc` | 此文件的位置因操作系统而异。在 Linux 上，此文件位于`/etc`；在 Solaris 上，位于`/var/opt/oracle`。 | oraInventory 目录位置和安装操作系统组 |
| `inst.loc` (Windows 注册表) | `HKEY_LOCAL_MACHINE\Software\Oracle` | 库存信息 |
| `inventory.xml` | `oraInventory/ContentsXML/inventory.xml` | Oracle 主目录名称及相应的目录位置 |
| `.log` 文件 | `oraInventory/logs` | 安装日志文件，对于故障排查非常有用 |

### 使用现有安装的副本进行安装

DBA 有时会使用`tar`等实用程序，将 Oracle 二进制文件的现有安装复制到不同的服务器（或同一服务器上的不同位置）来安装 Oracle 软件。这种方法快速简便（特别是与下载并运行 Oracle 安装程序相比）。此技术允许 DBA 轻松地在多台服务器上安装 Oracle 软件，同时确保每个安装都相同。

使用现有二进制文件副本安装 Oracle 是一个两部分的过程：使用操作系统实用程序复制二进制文件。附加 Oracle 主目录。

这些步骤将在接下来的两个小节中详细介绍。

**提示** MOS 注释`300062.1`包含如何克隆现有 Oracle 安装的说明。

### 步骤 1. 使用操作系统实用程序复制二进制文件

