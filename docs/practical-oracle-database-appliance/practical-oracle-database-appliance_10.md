# 6. 监控 Oracle 数据库设备

摘要

Oracle 数据库设备可以被视为一个整体或两个独立的单元，这取决于您希望如何监控它。实际上，Oracle 数据库设备是一种通过缩小物理硬件占用空间来解决企业问题的机器，同时提高了整体可用性。Oracle 数据库设备可用于帮助组织实现其架构的可扩展性。当使用新硬件扩展架构时，尤其是在使用 Oracle 数据库设备这样的工程化系统时，监控就成为了一个问题。如何监控 Oracle 数据库设备？

监控 Oracle 数据库设备与监控任何其他数据库环境非常相似；不同之处在于您有一个包含两台物理服务器的单一机箱。如前面章节所讨论的，Oracle 数据库设备可以配置为关联数据库，运行在独立模式（企业版）、Oracle RAC One 或 Oracle 真应用集群（RAC）模式。这三种配置都可以通过几种不同的方式进行监控。在本章中，我们将了解如何为 Oracle 11g 第 2 版数据库在 Oracle 数据库设备上配置、验证、重新配置、启动、停止和取消配置一个名为 Database Control 的实用程序。这是您用来控制设备的工具。之后，我们还将了解如何针对设备部署 Oracle 企业管理器代理。

注意

Oracle 数据库 12c 尚未获得 Oracle 数据库设备的认证。在 Oracle 数据库设备上使用 Oracle 数据库 12c 的企业管理器（EM）Express 尚未经过验证。

## 配置 Database Control

Database Control 是 Oracle 数据库 11g 的 Oracle 企业管理器的轻量级版本。Database Control 提供的功能可帮助数据库管理员和开发人员识别并解决可能与数据库中当前和持续处理相关的性能和管理问题。

配置 Database Control 有三种方式：在安装过程中、使用数据库配置助手（DBCA）以及从命令行使用企业管理器配置助手（EMCA）。出于本章的目的，让我们重点介绍如何使用 EMCA 工具配置 Database Control。

### 使用企业管理器配置助手（EMCA）

与 DBA 从命令行执行的任何操作一样，您需要为要配置的数据库设置 Oracle Home。在 Oracle 数据库设备上，这就像运行 `oraenv` 命令设置环境一样简单。清单 6-1 展示了如何设置环境以及如何在设置后检查环境。

**清单 6-1. 设置 Oracle 环境并检查它**

```bash
$ . oraenv
ORACLE_SID = [oracle] ? dboda
The Oracle base has been set to /u01/app/oracle
$ env | grep ORA
ORACLE_SID=dboda
ORACLE_BASE=/u01/app/oracle
ORACLE_HOME=/u01/app/oracle/product/11.2.0.3/dbhome_1
```

现在 Oracle 环境已设置，您可以从 Oracle Home 下的 `bin` 目录或当前所在位置运行 `emca` 实用程序。清单 6-2 说明了如何从 Oracle Home `bin` 目录运行 `emca` 实用程序。

**清单 6-2. 从 bin 目录执行 EMCA**

```bash
$ cd $ORACLE_HOME/bin
$ ./emca
```

运行 `emca` 命令时，可以在命令行上传递可选参数以帮助配置 Database Control。表 6-1 将帮助您理解这些参数是什么。企业管理器配置助手（EMCA）命令遵循一个简单的形式，如清单 6-3 所述。

**清单 6-3. EMCA 命令模式**

```bash
./emca [operation] [mode] [flags] [parameters]
```

要提供更多信息，可以使用 `-help` 选项来列出与 `emca` 实用程序相关的所有参数。清单 6-4 提供了调出帮助选项所需的语法。

**清单 6-4. EMCA 帮助选项**

```bash
$ ./emca -help
```

现在您对 `emca` 实用程序有了基本的了解，您将在 Oracle 数据库设备上配置 Database Control。在许多情况下，您将针对真实应用集群配置 Database Control。在配置 Database Control 之前，您需要更改要处理的数据库的 `job_queue_processes`。清单 6-5 显示了更改 `job_queue_processes` 参数所需的 `alter system` 命令。为此，您需要在命令行上指定几个额外的参数。清单 6-5 中的示例将允许的作业队列进程数增加到大于零的值。这有助于后续配置 Database Control。

**清单 6-5. 更改数据库的 job_queue_processes**

```sql
SQL> alter system set job_queue_processes = 10 scope=both sid='*';
```

要在 Oracle 数据库设备上为集群数据库配置 Database Control，命令很简单，如清单 6-6 所示。再次注意，清单 6-6 展示了如何从 Oracle Home `bin` 目录运行 `emca` 实用程序。

**清单 6-6. 为 RAC 数据库配置 Database Control**

```bash
$ cd $ORACLE_HOME/bin
$ ./emca -config dbcontrol db -repos create -cluster
```

配置启动后，`emca` 将要求您提供一系列输入。提供所需的输入。`emca` 正在寻找的参数可以作为选项在命令行上传递，也可以在参数文件中提供。表 6-1 提供了可以在命令行或参数文件中传递的参数列表。

表 6-1. EMCA 命令行参数



| 参数 | 描述 |
| --- | --- |
| `-respFile` | 指定一个输入文件的路径，该文件列出了 EMCA 在执行其配置操作时要使用的参数。更多信息，请参见“为 EMCA 参数使用输入文件”。 |
| `-SID` | 数据库系统标识符。 |
| `-PORT` | 为数据库提供服务的监听器的端口号。 |
| `-ORACLE_HOME` | 数据库的 Oracle 主目录，以绝对路径表示。 |
| `-ORACLE_HOSTNAME` | 本地数据库主机名。 |
| `-LISTENER_OH` | 运行监听器的 Oracle 主目录。如果监听器运行的 Oracle 主目录与数据库运行的主目录不同，则必须指定 `LISTENER_OH` 参数。 |
| `-HOST_USER` | 主机系统用户名（用于自动备份）。 |
| `-HOST_USER_PWD` | 主机系统用户密码（用于自动备份）。 |
| `-BACKUP_SCHEDULE` | 每日自动备份的时间表，格式为“HH:MM”。 |
| `-EMAIL_ADDRESS` | 用于通知的电子邮件地址。 |
| `-MAIL_SERVER_NAME` | 用于通知的发送邮件 (SMTP) 服务器。 |
| `-ASM_OH` | Oracle ASM 的 Oracle 主目录。 |
| `-ASM_SID` | Oracle ASM 实例的系统标识符。 |
| `-ASM_PORT` | 为 Oracle ASM 实例提供服务的监听器的端口号。 |
| `-ASM_USER_ROLE` | 用于连接到 Oracle ASM 实例的用户角色。 |
| `-ASM_USER_NAME` | 用于连接到 Oracle ASM 实例的用户名。 |
| `-ASM_USER_PWD` | 用于连接到 Oracle ASM 实例的密码。 |
| `-DBSNMP_PWD` | `DBSNMP` 用户的密码。 |
| `-SYSMAN_PWD` | `SYSMAN` 用户的密码。 |
| `-SYS_PWD` | `SYS` 用户的密码。 |
| `-SRC_OH` | 要升级或还原其 Enterprise Manager 配置的数据库的 Oracle 主目录。 |
| `-DBCONTROL_HTTP_PORT` | 使用此参数指定您在 Web 浏览器中显示数据库控制台的端口。更多信息，请参见“指定数据库控制使用的端口”。 |
| `-AGENT_PORT` | 使用此参数指定数据库控制的 Management Agent 端口。更多信息，请参见“指定数据库控制使用的端口”。 |
| `-RMI_PORT` | 使用此参数指定数据库控制的 RMI 端口。更多信息，请参见“指定数据库控制使用的端口”。 |
| `-JMS_PORT` | 使用此参数指定数据库控制的 JMS 端口。更多信息，请参见“指定数据库控制使用的端口”。 |
| `-CLUSTER_NAME` | 集群名称（用于集群数据库）。 |
| `-DB_UNIQUE_NAME` | 数据库唯一名称（用于集群数据库）。 |
| `-SERVICE_NAME` | 数据库服务名称（用于集群数据库）。 |
| `-EM_NODE` | 要运行数据库控制台的节点（用于集群数据库）。更多信息，请参见“在 Oracle RAC 中使用 EMCA”。 |
| `-EM_NODE_LIST` | 以逗号分隔的节点列表，用于仅代理配置，将数据上传到 `-EM_NODE`。更多信息，请参见“在 Oracle RAC 中使用 EMCA”。 |
| `-EM_SWLIB_STAGE_LOC` | 软件库位置。 |
| `-PORTS_FILE` | 指定要使用的端口的静态文件的路径。默认值是 `:${ORACLE_HOME}/install/staticports.ini`。 |

### 使用参数文件

使用 `emca` 配置数据库控制时，有很多标志、选项和参数可用于特定的配置目的。为了将输入量保持在最低限度并使配置更容易，可以使用参数文件。

要使用参数文件，请在运行 `emca` 时在命令行中包含 `-repFile` 命令行参数。清单 6-7 显示了与 `emca` 命令一起使用的参数文件的内容。

**清单 6-7. 参数文件内容**

```
PORT=1521
SID=patty
DBSNMP_PWD=<pass>
SYSMAN_PWD=<pass>
```

创建参数文件后，清单 6-8 向您展示了如何从命令行引用它。请关注 `–repfile` 参数。

**清单 6-8. 使用参数文件的 EMCA**

```
$cd $ORACLE_HOME/bin
$ ./emca -config dbcontrol db -repos create -repFile input_path_to_file
```

清单 6-8 中列出的命令向您展示了执行 `emca` 实用程序时如何指定响应文件。使用响应文件使数据库控制的配置变得容易得多，并为将来的配置提供了一种可重复的方法。

### 控制数据库控制

有时，您需要检查数据库控制是否正在运行。Enterprise Manager 控制 (`emctl`) 命令用于从命令行检查数据库控制。使用 `emctl` 命令，您可以检查状态，以及停止和启动数据库控制。

### 数据库控制的状态

配置数据库控制后，可以使用清单 6-9 中的命令检查状态。清单 6-9 还提供了成功状态检查的输出示例。

**清单 6-9. 数据库控制的状态**

```
$ cd $ORACLE_HOME/bin
$ ./emctl status dbconsole
```

命令的输出如下：

```
[oracle@patty bin]$ ./emctl status dbconsole
Oracle Enterprise Manager 11g Database Control Release 11.2.0.3.0
Copyright (c) 1996, 2011 Oracle Corporation. All rights reserved.
https://patty.enkitec.com:1158/em/console/aboutApplication
Oracle Enterprise Manager 11g is running.
------------------------------------------------------------------
Logs are generated in directory /u01/app/oracle/product/11.2.0.3/dbhome_1/patty_dboda/sysman/log
```

### 停止数据库控制

有时您需要停止数据库控制。这些情况包括当您需要进行维护、重新配置或计划数据库停机时。要停止数据库控制，您需要使用 `emctl` 命令，如清单 6-10 所示。

**清单 6-10. 停止数据库控制**

```
$ cd $ORACLE_HOME/bin
$ ./emctl stop dbconsole
```

当您停止数据库控制时，最佳实践通常是之后检查状态以验证数据库控制确实已停止。清单 6-11 显示了一个 `stop` 命令，后跟一个状态检查以验证停止是否确实发生。

**清单 6-11. 停止和状态示例**

```
[oracle@patty bin]$ ./emctl stop dbconsole
Oracle Enterprise Manager 11g Database Control Release 11.2.0.3.0
Copyright (c) 1996, 2011 Oracle Corporation. All rights reserved.
https://patty.enkitec.com:1158/em/console/aboutApplication
Stopping Oracle Enterprise Manager 11g Database Control ...
... Stopped.
[oracle@patty bin]$ ./emctl status dbconsole
Oracle Enterprise Manager 11g Database Control Release 11.2.0.3.0
Copyright (c) 1996, 2011 Oracle Corporation. All rights reserved.
https://patty.enkitec.com:1158/em/console/aboutApplication
Oracle Enterprise Manager 11g is not running.
```

## 取消配置数据库控制



### 启动数据库控制

如果数据库控制已关闭，您需要将其启动才能监控服务器上运行的数据库实例。启动数据库控制与停止操作一样简单。要启动数据库控制，您需要使用 `emctl` 命令的 `start` 选项。清单 6-12 进行了说明。

### 清单 6-12. 启动数据库控制

```
$ cd $ORACLE_HOME/bin
$ ./emctl stop dbconsole
```

同样，检查状态以验证数据库控制是否成功启动是一个好习惯。清单 6-13 展示了一个 `start` 命令及其后的状态检查。

### 清单 6-13. 启动与状态检查示例

```
[oracle@patty bin]$ ./emctl start dbconsole
Oracle Enterprise Manager 11g Database Control Release 11.2.0.3.0
Copyright (c) 1996, 2011 Oracle Corporation.  All rights reserved.
https://patty.enkitec.com:1158/em/console/aboutApplication
Starting Oracle Enterprise Manager 11g Database Control .... started.
------------------------------------------------------------------
Logs are generated in directory /u01/app/oracle/product/11.2.0.3/dbhome_1/patty_dboda/sysman/log
[oracle@patty bin]$ ./emctl status dbconsole
Oracle Enterprise Manager 11g Database Control Release 11.2.0.3.0
Copyright (c) 1996, 2011 Oracle Corporation.  All rights reserved.
https://patty.enkitec.com:1158/em/console/aboutApplication
Oracle Enterprise Manager 11g is running.
------------------------------------------------------------------
Logs are generated in directory /u01/app/oracle/product/11.2.0.3/dbhome_1/patty_dboda/sysman/log
```

至此，您已经了解了如何检查状态、如何停止以及如何启动数据库控制。在某些环境中，有时您需要创建或重新创建数据库控制以便对其进行重新配置。在下一节中，您将了解如何重新配置数据库控制。

