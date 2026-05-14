# 通过网络连接到数据库

一旦监听器配置并启动后，你可以从 SQL*Plus 客户端测试远程连接，如下所示：

```bash
$ sqlplus user/pass@'server:port/service_name'
```

在下一行代码中，用户和密码是 `system/foo`，连接到 `oracle18c` 服务器、端口 1521 上名为 `o18c` 的数据库：

```bash
$ sqlplus system/foo@'oracle18c:1521/o18c'
```

这个示例演示了连接到数据库的所谓**简易连接命名方法**。它之所以“简易”，是因为它不依赖于任何设置文件或工具。你唯一需要知道的信息是用户名、密码、服务器、端口和服务名（SID）。

## 简易连接命名方法

另一种常见的连接方法是**本地命名**。此方法依赖于 `ORACLE_HOME/network/admin/tnsnames.ora` 文件中的连接信息。在此示例中，编辑了 `tnsnames.ora` 文件，并添加了以下透明网络底层（TNS）（Oracle 的网络架构）条目：

```text
o18c =
(DESCRIPTION =
(ADDRESS = (PROTOCOL = TCP)(HOST = oracle18c)(PORT = 1521))
(CONNECT_DATA = (SERVICE_NAME = o18c)))
```

现在，从操作系统命令行，你可以通过引用已放入 `tnsnames.ora` 文件的 `o18c` TNS 信息来建立连接：

```bash
$ sqlplus system/foo@o18c
```

## 本地命名方法

此连接方法被称为“本地”，是因为它依赖于本地客户端的 `tnsnames.ora` 文件副本来确定 Oracle Net 连接详情。默认情况下，SQL*Plus 会检查由 `TNS_ADMIN` 变量定义的目录中名为 `tnsnames.ora` 的文件。如果未找到，则会搜索由 `ORACLE_HOME/network/admin` 定义的目录。如果找到了 `tnsnames.ora` 文件，并且该文件包含在 SQL*Plus 连接字符串中指定的别名（本例中为 `o18c`），那么连接详情将从该 `tnsnames.ora` 文件的条目中获取。

Oracle 使用的其他连接命名方法包括外部命名和目录命名。更多详细信息，请参阅《Oracle Net Services 管理员指南》，该指南可以从 Oracle 网站（`http://otn.oracle.com`）的技术网络区域免费下载。

> **提示**
> 你可以使用 `netca` 实用程序来创建 `tnsnames.ora` 文件。启动该实用程序并选择“本地网络服务名配置”选项。系统将提示你输入 SID、主机名和端口等信息。

