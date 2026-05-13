# 第二部分 使用 Oracle 数据库

## 7. 恢复与备份

到目前为止，本书中我们已经创建了虚拟机，安装了 Oracle 数据库软件，并创建了第一个数据库。现在系统已运行，我们需要首先开始保护我们的数据库。请记住第 1 章的内容：数据库管理员是数据的守护者。保护数据并确保其安全是你的责任。本章和接下来的一章对于数据守护者至关重要。本章我们将讨论 Oracle 的备份与恢复。下一章，我们将讨论数据库安全。

### 启动 Oracle 监听程序

如果我们希望用户能够从数据库服务器外部连接，就需要运行一个 Oracle 监听程序。监听程序的工作是协调传入的连接请求。如果你还记得本章前面图 6-7 的内容，我们在创建数据库时选择不创建监听程序。我们其实不必创建监听程序，只需启动它即可。

由于这不是一个生产就绪的服务器，仅仅是我们的测试平台，在启动监听程序之前，我们必须执行一个简单的任务。我们的测试平台不会被网络上任何服务器名称所知，因此也不会在网络 DNS 服务器中注册。为了解决这个问题，我们必须将主机名添加到虚拟机的 `/etc/hosts` 文件中。在服务器上，打开一个终端窗口。由于我们是以 oracle 用户身份登录的，我们输入 `su` 以 root 用户身份连接，并提供 root 密码。然后执行清单 6-2 中的命令。

```
echo "127.0.0.1 dbamentor dbamentor.localdomain" >> /etc/hosts
清单 6-2
添加主机条目
```

上面的命令将我们的主机名添加到列表中。输入 `exit` 退出 root 提示符。回到 oracle 提示符后，执行

```
lsnrctl start
```

这将启动监听程序，默认端口通常就足够了。等待几分钟，然后检查监听程序的状态。我们可以在清单 6-3 中看到 `lsnrctl status` 命令及其输出。

```
[oracle@dbamentor ~]$ lsnrctl status
LSNRCTL for Linux: Version 12.2.0.1.0 - Production on 10-JUN-2018 23:34:03
Copyright (c) 1991, 2016, Oracle.  All rights reserved.
Connecting to (ADDRESS=(PROTOCOL=tcp)(HOST=)(PORT=1521))
STATUS of the LISTENER

Alias                    LISTENER
Version                  TNSLSNR for Linux: Version 12.2.0.1.0 - Production
Start Date               10-JUN-2018 23:29:22
Uptime                   0 days 0 hr. 4 min. 40 sec
Trace Level              off
Security                 ON: Local OS Authentication
SNMP                     OFF
Listener Log File        /u01/app/oracle/diag/tnslsnr/dbamentor/listener/alert/log.xml
Listening Endpoints Summary...
(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=dbamentor)(PORT=1521)))
Services Summary...
Service "orcl" has 1 instance(s).
Instance "orcl", status READY, has 1 handler(s) for this service...
Service "orclXDB" has 1 instance(s).
Instance "orcl", status READY, has 1 handler(s) for this service...
The command completed successfully
清单 6-3
监听程序状态
```

从输出中我们可以看到，我们的数据库已经联系到监听程序，并已准备好接受传入连接。实例 `orcl` 在上面的输出中状态为 READY。

### 执行我们的首次查询

现在数据库已运行，让我们执行几个查询。首先，我们需要设置环境以便能够连接到该数据库。Oracle 包含一个脚本来帮助我们。在终端窗口中，输入 `. oraenv`（确保点号和 `oraenv` 之间有一个空格）。该脚本会询问我们想要哪个数据库，我们回复 `orcl`，如清单 6-4 所示。

```
[oracle@dbamentor ~]$ . oraenv
ORACLE_SID = [oracle] ? orcl
The Oracle base has been set to /u01/app/oracle
清单 6-4
设置我们的环境
```

如果这台服务器上运行了多个 Oracle 数据库，我们可以使用此方法来切换我们想要与之交互的数据库。`oraenv` 脚本使用 `/etc/oratab` 文件来了解我们数据库的 Oracle 主目录。如果我们查看该文件，里面只有一条非注释行：

```
orcl:/u01/app/oracle/product/12.2.0.1:N
```

当我们回复 `orcl` 时，`oraenv` 脚本通过读取此文件确定了 Oracle 主目录是 `/u01/app/oracle/product/12.2.0.1`。

环境设置好后，我们就可以启动 SQL*Plus 并连接到数据库了。在命令提示符下输入 `sqlplus system`，并在要求时提供用户密码。连接后，让我们通过查询其名称来验证是否连接到了我们的数据库，如清单 6-5 所示。

```
SQL> select name from v$database;
NAME

ORCL
清单 6-5
数据库名称
```

接下来，我们将在清单 6-6 中查询实例名称。

```
SQL> select instance_name,status from v$instance;
INSTANCE_NAME        STATUS
----------------                        ------------
orcl                 OPEN
清单 6-6
实例名称
```

实例名称与数据库名称匹配，这是符合预期的。如果这是一个 Oracle RAC 数据库，实例名称会在数据库名称后附加一个数字，例如 `orcl1`。

我们在本章要做的最后一件事是关闭实例然后再重新启动它。我们当前的用户 `SYSTEM` 没有执行这些操作的适当权限，因此我们将以 `SYS` 身份连接。由于我们就在数据库服务器本地，我们可以直接使用 `/ as sysdba` 进行连接，如清单 6-7 所示。这会将我们连接到 `SYS` 用户而无需密码。

```
SQL> connect / as sysdba
Connected.
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup
ORACLE instance started.
Total System Global Area 1660944384 bytes
Fixed Size                  8621376 bytes
Variable Size            1056965312 bytes
Database Buffers          587202560 bytes
Redo Buffers                8155136 bytes
Database mounted.
Database opened.
清单 6-7
关闭与启动
```

每次关闭虚拟机时，我们都需要先关闭 Oracle 实例。每次启动虚拟机时，我们都需要启动监听程序和 Oracle 实例。

在 Windows 上，由 DBCA 创建的服务会自动为您处理关闭/启动。在 Linux 和 Unix 上，您可以使用 `$ORACLE_HOME/bin` 中的 `dbshut` 和 `dbstart` 脚本来分别帮助自动执行关闭和启动。然而，在测试平台中我很少使用这些脚本，因为我在尝试新事物时往往会弄坏数据库。我个人的偏好是避免在启动虚拟机时启动一个损坏的数据库。您的看法可能不同，因此您可以选择在您的测试平台中自动化启动和关闭。

## 继续前进

在本章中，我们创建了第一个数据库。本章完成了我们测试平台的初始设置。我们甚至对数据库运行了一些查询，以确保它启动并运行良好。

在下一章，我们将学习一些关于如何备份数据库的基础知识。记住，我们希望成为数据守护者并保护数据库。备份与恢复是我们的第一道防线。


## 恢复先于备份

我几乎每天都能看到这种情况。数据库管理员讨论备份与恢复的顺序是错的。我见过这类问题：“备份数据库的最佳方式是什么？”“如何备份一个非常大的数据库？”“Oracle 首选的备份工具是什么？”“我应该执行增量备份吗？”每一个问题都在讨论备份，却丝毫未涉及恢复。甚至“备份与恢复”(`backup and recovery`)这个术语在我看来都是颠倒的。如果每个立志成为数据库管理员的人都将术语改为“恢复与备份”(`recovery and backup`)，那会更好，因为这才是我们思考问题应有的顺序。

### 小贴士

在设计备份方案之前，先定义你的恢复要求。

在我的职业生涯中，我一直大力倡导，在考虑备份之前先思考恢复要求。如今，我看到其他数据库管理员也开始效仿，我们一小群人正在改变这种态度。作为本书的读者，你现在已经加入了这场“先讨论恢复，再讨论备份”的运动。

备份只有一个且唯一的目的：能够在发生故障后，让数据库重新运行并恢复业务。重要的不是执行备份这个动作本身，而是从故障中**恢复和还原**这一关键任务。太多数据库管理员先实施备份方案，然后要么希望它能让自己从故障中恢复，要么盲目相信它未来一定能起作用。在这两种情况下，数据库管理员都犯了关键的判断错误。本书的读者将会明白，在实施备份方案之前，他们必须了解自己的恢复要求。请帮忙传播这个理念：是**恢复要求驱动备份方案**，而不是反过来。

大多数 Oracle 数据库管理员会告诉你使用 Oracle 恢复管理器(`RMAN`)来备份和还原你的数据库。他们会告诉你每周或每月执行完全备份，如果你的数据库很大或交易量很高，则在中间执行增量备份。表面上，这似乎是很好的建议，并且可能满足你绝大多数需求。但如果你的管理层告诉你，需要确保数据库在一小时内恢复运行，而你的数据库有 100TB 大，那会怎样？`RMAN`将很难在如此短的时间内还原这么大的数据库。这并不是说`RMAN`不好。`RMAN`是一个优秀的产品，如果可能，你应该使用它。`RMAN`是 Oracle 首选的备份与恢复工具。就像任何工具一样，使用它有优点也有缺点。有时，你的恢复要求决定了要使用不同的工具。与其思考你将使用哪个工具进行备份和恢复，不如让我们首先定义恢复要求。

## 恢复要求

在设计备份方案之前，请将你的恢复要求记录成文。这通常需要数据库管理员与管理层紧密合作，而管理层又可能需要与组织紧密合作。请回想第 2 章，我们讨论了了解你的受众，并考虑了一些与不同技能背景的人沟通的技巧。收集恢复要求正是你沟通技巧派上用场的时候。

恢复要求需要被记录下来，并用于制定服务级别协议(`SLA`)。`SLA`应包含业界所知的恢复时间目标(`RTO`)，即衡量你的数据库需要在故障后多快恢复运行的指标。你的恢复要求还需要指明可接受的数据丢失级别，称为恢复点目标(`RPO`)，通常以一段时间来表示。我需要恢复每一个事务吗？在这种情况下，`RPO`为零。或者，组织能否承受丢失一周的数据，或者介于两者之间的某个值？数据库管理员需要知道这两个指标，才能设计出满足`SLA`要求的备份策略。

`RTO`或数据丢失量`RPO`越低，实施方案的成本就越高。例如，如果你的组织可以承受七天的停机时间，并且可以丢失一个月的数据，那么你可以每月将备份写到异地磁带上。你会有充足的时间把那盘磁带带回数据中心并还原你的数据库。如果你的要求是零数据丢失，并且数据库必须在五分钟内恢复运行，那么`RMAN`或基于磁盘的快照将无法帮助你。你将需要实施`Data Guard`，并在另一个数据中心拥有一个事务一致的数据库副本。第一种情况成本更低，更容易实施。第二种情况则昂贵得多。

组织正在为你需要实施的灾难恢复解决方案付费。既然他们必须为数据库管理员你所设计的方案买单，那么你和组织双方对所有相关方的期望都有坚实的理解，这是你的责任。这就是为什么`SLA`如此重要。

`SLA`不仅仅是一份让所有相关方理解期望的文件。它也是数据库管理员与业务部门之间的一份合同。如果数据库管理员没有实施满足`SLA`的备份与恢复策略，那么该数据库管理员应被组织解雇。如果组织没有为满足`SLA`所需的资源付费，数据库管理员应对此进行记录，以便在无法满足协议条款时免于承担责任。



### 建议

与管理层协作，为公司数据库恢复制定服务级别协议。

该 SLA 应为企业中的每个数据库包含以下项目：

*   恢复时间目标，即数据库必须恢复运行的时间范围。
*   恢复点目标，即可接受的数据丢失量。
*   适用于安置数据库的数据中心的灾难场景列表，例如火灾、洪水、飓风、龙卷风或人为错误。
*   为备份数据库可接受的停机时间（如有）。
*   该数据库的任何特殊需求。

生产数据库的 SLA 条款通常与非生产数据库大不相同。许多数据库管理员仅为生产数据库创建 SLA，但你的非生产数据库也应该有 SLA。开发和测试数据库通常可以承受更长的停机时间并丢失更多数据，这需要被记录在案。请记住，虽然你组织的最终用户依赖你的生产数据库，但你的开发和测试数据库对某些人（通常是内部员工）而言也被视为生产环境。如果你的开发数据库宕机，应用开发人员可能无法工作。如果测试数据库宕机，质量保证人员可能无法工作。

当今的 IT 基础设施有助于满足非生产 SLA 要求。例如，利用云计算，可以轻松丢弃一个损坏的开发测试数据库并快速启动另一个，但你可能会经历完全的数据丢失。使用`Oracle Multitenant`，你可以为不同目的克隆数据库。例如，我有一个每周的“黄金镜像”，它是生产数据库的一个副本，挂载在一个`Oracle Multitenant`数据库中。所有的开发和测试数据库都是从这些黄金镜像克隆而来的。如果一个开发数据库损坏到无法使用，我们只需销毁它并从黄金镜像重新克隆。本段的全部重点是，技术变革可以帮助满足 SLA 要求。云计算、多租户、虚拟化和容器等新技术可以协助数据库管理员。

一旦 SLA 创建完毕且所有相关方都已签署，数据库管理员就需要仔细审阅它并设计恢复测试计划。恢复测试计划确保 SLA 中的所有要求都被考虑到，并且数据库管理员已验证可以按要求缓解灾难。恢复测试计划是一个检查清单，列出了根据 SLA 必须满足的项目。

请注意，在本节中，我们只讨论恢复这一半。在分析恢复需求时，我们从不考虑备份。SLA 中永远不应包含任何关于备份的内容，只应涉及恢复。如果你的 SLA 说明了要使用任何特定的备份方法，请将该条款从文档中删除。我见过太多 SLA 例如写道“每周执行完整备份，每日执行增量备份”——这种语言不属于 SLA。SLA 不应限制备份解决方案。相反，它应有助于制定备份解决方案。经过测试，数据库管理员可能会发现备份解决方案无法满足 SLA 的恢复要求，因此数据库管理员可能需要采用另一种备份解决方案。如果备份解决方案不在 SLA 中，这样做会更容易。

### 建议

只有在 SLA 完成之后，才能创建备份解决方案。

既然数据库管理员知道了需要从哪些场景中恢复、恢复速度要求以及可接受的最大数据丢失量，他们最终能够设计备份解决方案了。`Oracle`数据库管理员很可能会发现，使用`RMAN`每周执行一次完整备份、每日执行增量备份就足够了。数据库管理员也可能会发现需要实施不止一种备份解决方案。例如，数据库管理员可能需要能够定期刷新开发中的某个模式。此需求将属于前面列表中最后一个要点提到的“特殊需求”。数据库管理员可以选择使用`RMAN`备份来满足大部分恢复需求，但也利用数据库导出来满足这一特殊需求。或者，数据库管理员可以决定在`Multitenant`环境中使用`RMAN`备份，该环境为开发数据库创建克隆。

备份解决方案设计并实施后，数据库管理员针对所有要求进行恢复测试至关重要。数据库管理员应使用源自 SLA 的恢复测试计划，以确保覆盖所有 SLA 条款。如果某个恢复要求无法满足，数据库管理员需要弄清楚如何处理该要求。是需要修改现有的备份解决方案来处理这个额外要求并同时处理其他所有要求，还是需要数据库管理员实施另一个工具来满足该要求？

数据库管理员应每年与管理层一起审查 SLA。所有相关方都需要确保协议条款在接下来的一年仍然有效。事物随时间变化，在创建 SLA 时有效的内容可能不再有效。

数据库管理员也应至少每年练习一次恢复测试计划。这确保过去一年的技术变化不会影响满足 SLA 的能力。年度恢复测试应验证每个 SLA 要求是否仍然得到满足。如果数据库管理员有上面提到的恢复要求检查清单，他们可以在这里使用它。执行定期的恢复测试也能使数据库管理员的技能保持最新。作为数据库管理员，最糟糕的感觉之一莫过于所有人都在等待你将生产数据库恢复上线，而你却不记得该怎么做了。年度审查至关重要。理想情况下，应每半年甚至每季度进行一次审查。如果你的公司因业务或监管要求而接受审计，则需要进行多次恢复测试。

### 建议

至少每年审查一次 SLA 并演练恢复测试计划。



## 备份与恢复文档

Oracle 在《**数据库备份与恢复用户指南**》中详细记录了如何执行备份和恢复操作。每位 Oracle 数据库管理员都应至少从头到尾通读此指南两遍。要访问《**数据库备份与恢复用户指南**》，请使用网页浏览器访问 `https://docs.oracle.com` 并点击 `Database` 按钮。然后点击 `Oracle Database` 按钮，点击下拉菜单并选择您的数据库版本。每个版本通常会引入一些变更并为备份和恢复提供新功能。当新版本发布时，Oracle DBA 阅读此文档至关重要。选择好版本后，点击 `Topics` 部分中的 `Administration` 链接。向下滚动一点，您会看到一个标题为 `Backup and Recovery` 的部分。此部分应包含两份手册：《**数据库备份与恢复参考**》和《**数据库备份与恢复用户指南**》。本章将涵盖 Oracle 12.2 和 18c 两个数据库版本。

《**数据库备份与恢复参考**》包含了关于 `RMAN` 备份命令的所有详细信息。您会遇到许多 `RMAN` 命令示例。有时，某个示例可能与您想执行的操作很相似但略有不同。《**备份与恢复参考**》详细说明了每个命令的功能并列出了其所有选项。请花时间熟悉这份手册。目的不是要记住手册中的所有内容，而是要对其包含的内容有个概念。将来某天您可能需要在其中查找某些内容。

《**数据库备份与恢复用户指南**》是主要的阅读材料。这份文档包含大量关于如何使用 Oracle 的实用程序来备份和恢复数据库的信息，包括 `RMAN`、`闪回数据库` 以及 `用户管理的备份与恢复`。

强烈建议本书的所有读者都阅读《**备份与恢复用户指南**》。本章涵盖的信息量远不及 Oracle 在该指南中所提供的。当您阅读该手册时，不仅会学到知识，还要留意其中的内容及其位置。注意目录页，以了解文档的组织结构。和所有 Oracle 文档一样，您将来某个时候会需要参考它。

在《**备份与恢复用户指南**》的开头部分，您可能会发现一个陈述，那些没有读过本书的人可能不会多加思考。在指南的第 1 章中写道：“作为备份管理员，您的主要工作是进行备份和监控备份以保护数据。”本书的读者知道，主要工作是让我们的数据库启动并运行，以及从故障中恢复。备份并非您的主要工作；从故障中恢复才是。备份是达成此目的的手段。

## 备份类型

Oracle 数据库在讨论备份类型时有两种不同的分类。第一种分类是备份的“温度”。第二种分类是备份的管理类型。

### 热备份与冷备份

备份可以是 `热` 备份或 `冷` 备份。温度用于表示在执行备份时，数据库是打开并可供业务使用（`热`）还是处于关闭状态（`冷`）。在当今 `24×7` 的环境中，生产数据库很少使用 `冷` 备份。`冷` 备份非常好，因为它们最容易实现，也最容易从恢复。然而，由于对持续可用性的需求，我们经常需要依赖 `热` 备份。

数据库由磁盘上的文件组成。备份数据库时，该文件的副本会被写入备份目标位置。文件越大，数据库越大，完成备份所需的时间就越长。数据库的问题在于它是一个活生生的、不断变化的实体。事务持续发生并修改数据库的数据文件。备份开始后但尚未完成前，某个事务可能会修改已经备份过的文件，以及尚未备份的文件。这将导致备份不一致。我们可用的一种处理机制是停止所有事务，即执行 `冷` 备份。使用 `冷` 备份，我们知道所有文件在某个特定时间点上事务是一致的。

如果我们需要执行 `热` 备份，就需要另一种机制来解决在将数据文件复制到备份目标位置期间发生的任何事务不一致问题。在 Oracle 中，`热` 备份要求我们将数据库配置为 `归档日志` 模式。在 `归档日志` 模式下，Oracle 数据库会在磁盘上的一个特殊位置维护事务日志的副本。我们执行 `热` 备份。当从该 `热` 备份恢复数据库时，Oracle 使用日志文件中的事务来解决任何不一致。`热` 备份让我们能够在保持数据库开放供业务使用的同时保护它，但代价是需要维护事务日志。



### RMAN 与用户管理备份

Oracle 备份有两种管理方式。我们可以让恢复管理器来管理流程，也可以完全交由用户自行处理。这两种方式都在《*数据库备份与恢复用户指南*》中有详细记录。

过去，数据库管理员总是亲自管理备份及后续的恢复操作。当灾难发生时，DBA 需要分析问题，评估需要恢复的内容，并执行正确的步骤以使数据库重新开放业务。丢失控制文件后，DBA 需要执行的操作与丢失数据文件后的操作截然不同。DBA 需要精通各种灾难场景及相应的恢复步骤。

数据库管理员负责编写自己的备份脚本。一旦 DBA 编写出一个适用于大多数情况的备份脚本，就会尽可能地用它来备份许多不同的数据库。热备份和冷备份的脚本完全不同。如果数据库处于归档日志模式，DBA 还需要另一个脚本来管理归档日志目标，以防其被填满。

DBA 需要密切关注备份和恢复的每一个细节。因此，过程中出现人为错误的可能性很高。在你需要从灾难中恢复时才发现备份脚本中存在人为错误，那将是一场噩梦。用户管理的备份与恢复是一项耗时且脆弱的活动。

Oracle 公司随 Oracle 8.0 版本引入了恢复管理器（Recovery Manager）。有了 RMAN，数据库管理员无需再了解所有细节。RMAN 会为你打理一切。在 RMAN 中，恢复丢失的控制文件或数据文件的操作是相同的。你只需告诉 RMAN 恢复所有内容。RMAN 会找出问题所在，并为你重新整合数据。如果你使用企业管理器并安排数据库备份，它会在后台为你执行 RMAN 命令。自推出以来，RMAN 多年来不断发展壮大，现已具备许多你可能会喜欢的功能。希望你能明白为什么 RMAN 是 Oracle 首选的备份和恢复实用工具。

有了 RMAN，用户管理的备份是否就完全没有必要了呢？许多数据库管理员会告诉你始终使用 RMAN，但我偶尔仍然发现用户管理备份的用武之地。我采用用户管理备份的两个场景是：对小型测试数据库进行快速简便的备份，以及备份那些可以利用基于硬件的磁盘快照功能的超大型数据库。

本章稍后，我们将对我们上一章创建的测试数据库执行一次用户管理的备份。该过程超级简单且非常直观，你无需记住任何 RMAN 命令。

所有这些听起来可能像是我反对 RMAN，但事实并非如此。RMAN 是一款出色的工具，只要它能满足你的恢复需求，我推荐使用它。我反对的是，在人们不了解具体需要满足哪些恢复需求的情况下，就声称要使用 RMAN，或者说它是首选备份工具。

请记住，恢复需求决定了备份解决方案。如果我的需求是在十分钟内恢复一个 200TB 的数据库，那么用 RMAN 祝你好运。对于如此庞大的数据库，该工具根本无法实现那么快的备份或恢复。但许多磁盘子系统允许存储管理员对磁盘上的文件进行快照。我们可以制定一种用户管理的备份策略：将数据库置于备份模式，进行快照，然后结束备份模式。整个过程可以在几分钟内完成。要恢复时，我们关闭数据库，并让存储管理员恢复到一个好的快照状态。即使是 200TB 的数据库，这个过程也可以在几分钟内完成。数据库管理员必须编写备份脚本，并与存储管理员协作，双方都需要处理所有细节。然而，有些恢复需求决定了必须采用 RMAN 无法处理的解决方案。RMAN 几乎能满足所有恢复需求，但并非全部。请记录下这些恢复需求，然后选择备份工具。

### 逻辑备份

太多数据库管理员仅依赖*逻辑备份*来保护其数据库，在大多数情况下，这是不正确的备份解决方案，正如本节所讨论的。逻辑备份可以满足某些恢复需求。然而，它们很少能满足生产数据库的恢复需求。

什么是逻辑备份？它是使用类似数据泵的`expdp`这样的导出实用工具进行的备份。RMAN 和用户管理的备份是*物理备份*，因为它们将数据文件复制到备份目标位置。备份时，数据库中的文件与备份目标位置中的内容是完全相同的。随着时间的推移，数据库文件会发生变化，并与备份副本产生差异。我需要指出的是，RMAN 的物理备份通常在比文件级别更小的数据块级别上操作。这种技术差异不会改变逻辑备份与物理备份之间的本质区别。

逻辑备份读取表的内容，并将表中的数据写入转储文件。该转储文件不像数据库中的任何文件。相反，转储文件包含表的结构及其数据本身。物理备份和逻辑备份之间最大的区别在于，对于逻辑备份，你永远无法前滚（*rolling forward*）自备份以来发生的任何更改。而对于物理备份，你可以恢复数据库，并且如果你拥有事务日志，就可以将数据库带到比备份时间点更晚的状态，这称为*前滚*。前滚可以减少甚至消除任何数据丢失。因为逻辑备份无法前滚，根据其陈旧程度以及自执行逻辑备份以来数据变化的多少，它可能会遭受大量的数据丢失。

我经常听到初级数据库管理员询问关于从故障中恢复数据库的问题，而我最初的回应通常是询问有关数据库如何备份的更多细节。需要帮助的 DBA 回应说他们使用数据泵进行备份。此时，我就知道这位数据库管理员几乎没有考虑过他们的恢复需求。他们只是实施了他们认为能覆盖需求的备份解决方案，而十有八九，他们会得到一个惨痛的教训。这并不是说逻辑备份不好。它们确实有其用途，但逻辑备份很少是生产数据库的唯一备份选项。

逻辑备份非常适合将数据从一个数据库复制到另一个数据库。RMAN 在某种程度上也能做到这一点。使用数据泵，我可以只导出一个方案（schema）并导入到另一个数据库中。我也可以导出一组表。逻辑备份也可用于获取数据的某个时间点快照。当我需要保留某个方案的内容时，我经常会使用数据泵，通常是在我销毁整个方案之前。我会将转储文件复制到磁带介质或其他异地存储中。多年后，当有人说他们需要这些数据时，恢复用户管理或 RMAN 备份可能会很困难，因为我需要与备份时完全相同的数据库版本。而将旧版本的数据泵导出文件导入到较新的 Oracle 版本中通常没有问题。逻辑备份有其用途。物理备份也有其用途。数据库管理员需要了解两者的区别，以及每种方式对恢复能力的影响。然后，DBA 将利用最合适的方法。


## 还原与恢复

我们经常互换使用 `restore` 和 `recover` 这两个术语，认为它们是同一回事。在 Oracle 的术语中，它们具有截然不同的含义。注意这些区别非常重要。

当你 `restore` 一个 Oracle 数据库时，你只是简单地将数据库的文件放回磁盘。还原阶段将文件从备份位置复制到服务器上 Oracle 可以访问的某个位置。通常，我们会尝试还原到完全相同的位置，但如果那个磁盘设备损坏了，我们可能需要将文件还原到服务器上的不同位置。

请记住，数据库是一个动态的实体。事务不断对数据进行更改。`recovery` 阶段将确保所有已提交事务的更改都已写入数据文件，并且任何未提交的事务都会被回滚。无论你执行的是热备份还是冷备份，恢复阶段都是必需的。

## 我们的冷备份

既然我们已经创建了测试数据库，现在是时候备份它了。因为这是一个测试环境，我可能会在尝试新概念时破坏某些东西。能够回到一个已知状态是很好的。这个数据库没有任何生产用途，我可以丢失其中的所有数据。毕竟，它只是一个测试库。我不需要任何快速恢复，因此我的 RTO 可以相当大。现在我了解了简单的恢复需求，可以设计备份策略了。对于我的测试环境，我通常对数据库进行一次冷的、用户管理的备份。

在对数据库执行用户管理的备份时，我们需要备份控制文件、在线重做日志和数据文件。控制文件包含所有其他文件位置的主目录。在线重做日志包含事务日志。数据文件包含数据库中的数据。我们可以查询 `V$CONTROLFILE`、`V$LOGFILE` 和 `V$DATAFILE` 来了解这些文件的名称和位置。清单 7-1 显示了获取这些文件位置的基本查询。

```sql
SQL> select name from v$controlfile;
NAME
------------------------------------------------------
/u01/app/oracle/oradata/orcl/control01.ctl
/u01/app/oracle/oradata/orcl/control02.ctl
SQL> select member from v$logfile;
MEMBER
------------------------------------------------------
/u01/app/oracle/oradata/orcl/redo01.log
/u01/app/oracle/oradata/orcl/redo02.log
/u01/app/oracle/oradata/orcl/redo03.log
SQL> select name from v$datafile;
NAME
------------------------------------------------------
/u01/app/oracle/oradata/orcl/system01.dbf
/u01/app/oracle/oradata/orcl/sysaux01.dbf
/u01/app/oracle/oradata/orcl/undotbs01.dbf
/u01/app/oracle/oradata/orcl/users01.dbf
```

清单 7-1
用于冷备份的文件

因为这是一个测试环境，所有文件都在同一个位置。让我们创建一个备份目录，关闭实例，备份数据库，然后启动实例。我们可以在不离开 `SQL*Plus` 的情况下完成这些操作，只需在所有主机命令前加上感叹号。清单 7-2 显示了执行冷备份的步骤。

```sql
SQL> !mkdir /u01/app/oracle/oradata/db_backup
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> !cp /u01/app/oracle/oradata/orcl/* /u01/app/oracle/oradata/db_backup/.
SQL> startup
ORACLE instance started.
Total System Global Area 1660944384 bytes
Fixed Size                  8621376 bytes
Variable Size            1056965312 bytes
Database Buffers          587202560 bytes
Redo Buffers                8155136 bytes
Database mounted.
Database opened.
```

清单 7-2
冷备份

我们的备份完成了。无论我们对测试数据库造成何种损坏，我们总能回退到执行此备份的那一天。如清单 7-2 所示，对于我们的测试环境，冷备份非常简单。我们创建了一个备份目标目录，关闭了实例，将文件复制到备份目标目录，然后启动了实例。

要从故障中恢复，我们需要做的就是关闭实例，将文件复制回来，然后再次启动它。就是这么简单，这就是为什么我喜欢为测试环境使用用户管理的冷备份。对于这个测试环境，用户管理的冷备份易于理解和执行。如果你的数据库更复杂，文件位于不同的位置，那么备份和恢复可能会更复杂。

让我们关闭实例并删除一个数据文件，然后尝试启动实例。我们正在模拟数据文件丢失的情况。数据文件丢失的原因可能是人为意外删除，或者磁盘上该文件所在位置有坏块。清单 7-3 显示了当我们尝试启动一个缺少数据库文件的实例时会发生什么。

```sql
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> !rm /u01/app/oracle/oradata/orcl/users01.dbf
SQL> startup
ORACLE instance started.
Total System Global Area 1660944384 bytes
Fixed Size                  8621376 bytes
Variable Size            1056965312 bytes
Database Buffers          587202560 bytes
Redo Buffers                8155136 bytes
Database mounted.
ORA-01157: cannot identify/lock data file 4 - see DBWR trace file
ORA-01110: data file 4: '/u01/app/oracle/oradata/orcl/users01.dbf'
```

清单 7-3
模拟数据文件丢失

该文件因某种原因丢失了。当实例启动时，它无法找到该文件，因此给了我们 ORA-1157 错误。为了让系统重新运行起来，我们将中止并关闭实例，还原我们的文件，然后启动系统。我们可以在清单 7-4 中看到这些步骤。

```sql
SQL> shutdown abort
ORACLE instance shut down.
SQL> !cp /u01/app/oracle/oradata/db_backup/* /u01/app/oracle/oradata/orcl/.
SQL> startup
ORACLE instance started.
Total System Global Area 1660944384 bytes
Fixed Size                  8621376 bytes
Variable Size            1056965312 bytes
Database Buffers          587202560 bytes
Redo Buffers                8155136 bytes
Database mounted.
Database opened.
```

清单 7-4
从丢失的数据文件中恢复

清单中的最后一行显示数据库现在已打开可供使用。

此时，我们现在可以开始玩“假设”游戏，并真正开始了解更多关于 Oracle 数据库的知识。我们有了测试环境，可以随心所欲地进行尝试。在后面的章节中，我们将更详细地讨论测试环境和测试用例，但在这里我们也可以做一点。鼓励你花些时间寻找以下问题的答案：

*   如果我丢失了一个控制文件会怎样？
*   如果我丢失了一个在线重做日志文件会怎样？
*   如果我只还原丢失的文件而不还原整个数据库会怎样？

培养提出这类问题的好奇心。当你心中有一个问题时，就去你的测试环境看看会发生什么。如果你彻底搞砸了，你总可以从你的冷备份中恢复。记住，像这样的书不会教你所有东西，所以要通过提问来学习更多。

## 归档日志模式

正如我在本章前面提到的，如果你想执行热备份，就需要将数据库配置为归档日志模式。当事务发生时，Oracle 会将事务的更改写入在线重做日志。当事务完成时，Oracle 会写入一个标记，表明事务是已提交还是已回滚。Oracle 数据库中只有少数几个在线重做日志组。如果你回顾清单 7-1，你会看到我们的测试数据库有三个在线重做日志组。Oracle 会先写入第一个，然后转到第二个，再转到第三个。当第三个在线重做日志组写满时，Oracle 会回到第一个组并覆盖其内容。由于在线重做日志是以这种循环方式使用的，我们会随着时间的推移丢失事务历史记录。

大多数 Oracle 数据库有三到五个在线重做日志组。一个好的经验法则是使在线重做日志组足够大，以便我们每小时使用三到四个组。这意味着我们将每小时覆盖一次事务日志。

为了保存我们的事务历史记录，我们需要让 Oracle 归档在线重做日志。因此，我们开启归档日志模式。一旦 Oracle 切换到下一个在线重做日志组，它将归档前一个组。要开启归档日志模式，我们将定义归档目标。然后我们关闭实例，以装载模式启动它，并启动归档日志模式。最后，我们打开数据库供业务使用。这个过程确实需要停机时间，因此你可能需要在维护窗口期间执行。为数据库配置归档日志模式的步骤如清单 7-5 所示。

```
SQL> !mkdir /u01/app/oracle/oradata/arch
SQL> alter system set log_archive_dest_1='location=/u01/app/oracle/oradata/arch/' scope=spfile;
System altered.
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup mount
ORACLE instance started.
Total System Global Area 1660944384 bytes
Fixed Size                  8621376 bytes
Variable Size            1056965312 bytes
Database Buffers          587202560 bytes
Redo Buffers                8155136 bytes
Database mounted.
SQL> alter database archivelog;
Database altered.
SQL> alter database open;
Database altered.
SQL> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            /u01/app/oracle/oradata/arch/
Oldest online log sequence     15
Next log sequence to archive   17
Current log sequence           17
Listing 7-5
归档日志模式
```

清单 7-5 中的最后一个命令显示归档日志模式已启用。

数据库处于归档日志模式后，我们的事务就能安全地保存，不会被在线重做日志覆盖。我们可以从之前的备份中恢复，并使用归档的重做日志进行前滚。你还需要备份归档的重做日志。至少，你应该在数据库服务器上保留足够的归档重做日志，以覆盖自上次数据库备份以来的时间段。如果你执行每周完整备份，你应该在磁盘上保留七天的归档重做日志。然而，你也应该备份这些归档重做日志，这样数据库服务器的磁盘就不是它们唯一存储的地方。

应该指出的是，启用归档日志模式是有成本的。当归档日志模式开启时，Oracle 需要在在线重做日志写满后读取它并将其复制到归档日志目标。这将需要在数据库服务器上使用额外的资源，即 CPU 和磁盘 I/O。通常，好处大于成本，但数据库管理员需要意识到这一点。

如果我们不管理归档日志目标，它最终会被填满。当它被填满并且 Oracle 无法归档任何重做日志时，数据库中的事务将戛然而止。对于我的测试平台，我通常使用类似清单 7-6 所示的`RMAN`命令删除所有超过一天的归档重做日志。

```
[oracle@dbamentor ~]$ rman
Recovery Manager: Release 12.2.0.1.0 - Production on Tue Jun 19 13:10:46 2018
Copyright (c) 1982, 2017, Oracle and/or its affiliates. All rights reserved.
RMAN> connect target /
connected to target database: ORCL (DBID=1506090724)
RMAN> run { delete archivelog all completed before 'sysdate-1'; }
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=63 device type=DISK
List of Archived Log Copies for database with db_unique_name ORCL
=====================================================================
Key    Thrd Seq     S Low Time
------ ---- ------- - ---------
1      1    17      A 18-JUN-18
Name: /u01/app/oracle/oradata/arch/1_17_978309988.dbf
2      1    18      A 19-JUN-18
Name: /u01/app/oracle/oradata/arch/1_18_978309988.dbf
3      1    19      A 19-JUN-18
Name: /u01/app/oracle/oradata/arch/1_19_978309988.dbf
4      1    20      A 19-JUN-18
Name: /u01/app/oracle/oradata/arch/1_20_978309988.dbf
5      1    21      A 19-JUN-18
Name: /u01/app/oracle/oradata/arch/1_21_978309988.dbf
Do you really want to delete the above objects (enter YES or NO)? YES
deleted archived log
archived log file name=/u01/app/oracle/oradata/arch/1_17_978309988.dbf RECID=1 STAMP=979218471
deleted archived log
archived log file name=/u01/app/oracle/oradata/arch/1_18_978309988.dbf RECID=2 STAMP=979218632
deleted archived log
archived log file name=/u01/app/oracle/oradata/arch/1_19_978309988.dbf RECID=3 STAMP=979218638
deleted archived log
archived log file name=/u01/app/oracle/oradata/arch/1_20_978309988.dbf RECID=4 STAMP=979218638
deleted archived log
archived log file name=/u01/app/oracle/oradata/arch/1_21_978309988.dbf RECID=5 STAMP=979218641
Deleted 5 objects
Listing 7-6
清理归档重做日志
```

`RMAN`找到了所有超过 24 小时的归档重做日志，并从目标中删除了它们。然而，这只对测试平台有效。对于正常操作，你会希望使用`RMAN`或其他例程来备份你的归档重做日志以及数据库的其余部分。然后让`RMAN`或其他例程删除任何已备份的归档重做日志。这样，你就拥有了备份和事务，并且在需要恢复时可以最大限度地减少数据损失。

许多数据库管理员总是将生产数据库置于归档日志模式。但对于可以轻松重新创建丢失数据并且我可以承担冷备份停机时间的大型数据仓库，我将偏离这一惯例。一些数据库管理员也希望他们的测试数据库处于归档日志模式。理由是归档事务的行为确实给数据库带来了一定程度的开销。在某些情况下，这可能会导致性能问题。如果我们将更改首先应用到测试数据库，我们就有很好的机会在更改影响生产之前发现任何性能问题，并且我们可以找出处理方法，以免生产用户受到负面影响。


## 我们的热备份

现在我们的测试数据库已处于归档日志模式，可以执行热备份了。这次，我们将使用 RMAN 来执行备份。首先，我们需要配置 RMAN，让其知道备份文件的存放位置。在清单 7-7 中，我们将调用 RMAN 工具并配置备份目标位置。

```
[oracle@dbamentor ~]$ rman
Recovery Manager: Release 12.2.0.1.0 - Production on Tue Jun 19 13:18:06 2018
Copyright (c) 1982, 2017, Oracle and/or its affiliates. All rights reserved.
RMAN> connect target /
connected to target database: ORCL (DBID=1506090724)
RMAN> configure default device type to disk;
using target database control file instead of recovery catalog
new RMAN configuration parameters:
CONFIGURE DEFAULT DEVICE TYPE TO DISK;
new RMAN configuration parameters are successfully stored
RMAN> configure controlfile autobackup on;
new RMAN configuration parameters:
CONFIGURE CONTROLFILE AUTOBACKUP ON;
new RMAN configuration parameters are successfully stored
RMAN> configure channel device type disk format '/u01/app/oracle/oradata/db_backup/backup%d_%u_%s_%p';
new RMAN configuration parameters:
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT  '/u01/app/oracle/oradata/db_backup/backup%d_%u_%s_%p';
new RMAN configuration parameters are successfully stored
Listing 7-7
RMAN Configuration
```

我们指示 RMAN 将备份写入磁盘，并希望控制文件能够自动备份。最后一条命令指定了磁盘位置和文件命名格式。这个配置可以在你的 RMAN 脚本中轻松覆盖。备份到磁盘因其速度优势而非常普遍。我通常会将备份写入专门为该目的保留的磁盘。如果需要将备份写入磁带，另一个作业会将备份从磁盘复制到磁带。如果需要将备份异地存放，我会使用像 `rsync` 或 `robocopy` 这样的工具。当然，RMAN 也具备直接写入磁带介质的能力。

设置好这些默认值后，我们的 RMAN 数据库备份命令就变得简单多了。在清单 7-8 中，可以看到备份数据库及其归档重做日志只需一个简单的命令。

```
RMAN> run { backup database plus archivelog; }
Starting backup at 19-JUN-18
current log archived
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=71 device type=DISK
channel ORA_DISK_1: starting archived log backup set
channel ORA_DISK_1: specifying archived log(s) in backup set
input archived log thread=1 sequence=22 RECID=6 STAMP=979219448
input archived log thread=1 sequence=23 RECID=7 STAMP=979219451
input archived log thread=1 sequence=24 RECID=8 STAMP=979219492
channel ORA_DISK_1: starting piece 1 at 19-JUN-18
channel ORA_DISK_1: finished piece 1 at 19-JUN-18
piece handle=/u01/app/oracle/oradata/db_backup/backupORCL_1kt5rd15_52_1 tag=TAG20180619T132453 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 19-JUN-18
Starting backup at 19-JUN-18
using channel ORA_DISK_1
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00001 name=/u01/app/oracle/oradata/orcl/system01.dbf
input datafile file number=00002 name=/u01/app/oracle/oradata/orcl/sysaux01.dbf
input datafile file number=00003 name=/u01/app/oracle/oradata/orcl/undotbs01.dbf
input datafile file number=00004 name=/u01/app/oracle/oradata/orcl/users01.dbf
channel ORA_DISK_1: starting piece 1 at 19-JUN-18
channel ORA_DISK_1: finished piece 1 at 19-JUN-18
piece handle=/u01/app/oracle/oradata/db_backup/backupORCL_1lt5rd16_53_1 tag=TAG20180619T132454 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:15
Finished backup at 19-JUN-18
Starting backup at 19-JUN-18
current log archived
using channel ORA_DISK_1
channel ORA_DISK_1: starting archived log backup set
channel ORA_DISK_1: specifying archived log(s) in backup set
input archived log thread=1 sequence=25 RECID=9 STAMP=979219509
channel ORA_DISK_1: starting piece 1 at 19-JUN-18
channel ORA_DISK_1: finished piece 1 at 19-JUN-18
piece handle=/u01/app/oracle/oradata/db_backup/backupORCL_1mt5rd1l_54_1 tag=TAG20180619T132509 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 19-JUN-18
Starting Control File and SPFILE Autobackup at 19-JUN-18
piece handle=/u01/app/oracle/product/12.2.0.1/dbs/c-1506090724-20180619-00 comment=NONE
Finished Control File and SPFILE Autobackup at 19-JUN-18
Listing 7-8
RMAN Hot Backup
```

RMAN 命令就是简单的 `backup database plus archivelog`。一条命令搞定一切。如果你仔细阅读 RMAN 的输出，可以看到 RMAN 在读取数据库的文件，甚至备份了归档重做日志。最后，RMAN 还备份了控制文件和参数文件。

如果我们的测试环境不进行实际测试，那就没什么价值了。所以让我们运行之前做过的相同测试。我关闭了实例，然后删除了 `users01.dbf` 数据文件，模拟文件丢失。现在我将使用 RMAN 进行恢复。它会找出缺失的文件并按顺序将其全部恢复。还记得之前关于 `restore` 和 `recovery` 区别的讨论吗？这些区别现在很重要，因为我们首先需要告诉 RMAN 恢复（`restore`）数据库，然后应用恢复（`recovery`）。在清单 7-9 中，操作顺序是：以 `mount` 模式启动实例、恢复数据库、恢复数据库，然后打开数据库对外服务。

```
[oracle@dbamentor orcl]$ rman
Recovery Manager: Release 12.2.0.1.0 - Production on Tue Jun 19 13:29:38 2018
Copyright (c) 1982, 2017, Oracle and/or its affiliates.  All rights reserved.
RMAN> connect target /
connected to target database (not started)
RMAN> run {
2> startup mount;
3> restore database;
4> recover database;
5> alter database open;
6> }
Oracle instance started
database mounted
Total System Global Area    1660944384 bytes
Fixed Size                    8621376 bytes
Variable Size              1056965312 bytes
Database Buffers            587202560 bytes
Redo Buffers                  8155136 bytes
Starting restore at 19-JUN-18
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=34 device type=DISK
channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00001 to /u01/app/oracle/oradata/orcl/system01.dbf
channel ORA_DISK_1: restoring datafile 00002 to /u01/app/oracle/oradata/orcl/sysaux01.dbf
channel ORA_DISK_1: restoring datafile 00003 to /u01/app/oracle/oradata/orcl/undotbs01.dbf
channel ORA_DISK_1: restoring datafile 00004 to /u01/app/oracle/oradata/orcl/users01.dbf
channel ORA_DISK_1: reading from backup piece /u01/app/oracle/oradata/db_backup/backupORCL_1lt5rd16_53_1
channel ORA_DISK_1: piece handle=/u01/app/oracle/oradata/db_backup/backupORCL_1lt5rd16_53_1 tag=TAG20180619T132454
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:15
Finished restore at 19-JUN-18
Starting recover at 19-JUN-18
using channel ORA_DISK_1
starting media recovery
media recovery complete, elapsed time: 00:00:00
Finished recover at 19-JUN-18
Statement processed
Listing 7-9
RMAN Recovery from Missing Data File
```

RMAN 为我们理清了所有头绪，并在很短的时间内让一切恢复了秩序。

RMAN 拥有大量功能，还有很多东西需要学习。再次强调，阅读 `Database Backup and Recovery User’s Guide` 以了解 RMAN 所有的功能是非常重要的。本章实在无法涵盖其所有内容。



## 深入探索

当你看到本章标题时，这很可能与你预期的不同。我本可以像许多其他书籍那样来写本章：“这是备份数据库的步骤。这是恢复数据库的步骤。”本书的要点是让你超越程序性步骤去思考，因为这才是真正拓展你数据库管理员职业生涯的唯一途径。在本章中，我们确实提出了一个问题：确定在丢失数据文件后如何使数据库恢复运行。我们还探讨了丢失控制文件或在线重做日志会发生什么。希望你能回去进行更多实验。最坏的情况是你彻底搞乱了你的测试数据库且无法使其再次运行，但这没关系，因为你知道如何从那次冷备份中恢复并重新运行起来。

本章首先提出了一个令人信服的论点，即数据库管理员甚至应在考虑备份之前就先处理恢复需求。应制定服务级别协议，并由此衍生出一份测试文档。只有到那时，才设计备份方案并随后进行测试，以确保其满足所有要求。我鼓励你加入日益壮大的数据库专业人士行列，他们主张先明确恢复需求，再考虑备份。

随后，本章为理解不同类型的备份及其使用场景奠定了基础。我们分别使用 `RMAN` 和用户管理命令在测试环境中执行了热备份和冷备份。我们并未止步于此，而是稍作实验，因为这就是我们的学习方式。永不停止提问，永不停止实验。在阅读本章时，你是否想到了我们未涉及的其他问题？把这些问题写下来，看看你是否能找到答案。

下一章将聚焦于数据库安全。如果我们锁定数据库，并确保使用专门为满足我们需求而定制的备份来保护它，那么我们就朝着成为公司所需的数据守护者迈出了重要一步。

