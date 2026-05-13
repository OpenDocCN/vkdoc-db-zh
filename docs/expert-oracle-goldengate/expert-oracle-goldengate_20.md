# 定期清除旧的 Trail 文件

旧的 trail 文件会累积并导致磁盘空间问题。Trail 文件通常可以在几天后清除，但根据您的要求和可用磁盘空间量，它们可以保留更长或更短的时间。您可以在 Manager 参数文件中指定自动清除标准，如下一个示例所示。请记住，在 Manager 中指定的清除标准适用于在其下运行的任何 Extract 和 Replicat。这使您能够集中管理清除标准。

以下示例指示 Manager 清除超过两天的任何 trail 文件。`UseCheckPoints` 选项告诉 Manager 在文件被处理完毕*之后*再清除它们，如下例所示。

```
PurgeOldExtracts dirdat/*, UseCheckPoints, MinKeepDays 2
```

接下来让我们看一些自动启动 GoldenGate 进程的技巧。

#### 自动启动进程

使用 `AUTOSTART` Manager 参数，可以在 Manager 启动时自动启动您的 Extract 和 Replicat 进程。另一个参数 `AUTORESTART` 也可用，但应谨慎使用。有时进程失败是由于临时资源问题或网络问题，只需重新启动即可。`AUTORESTART` 会自动*尝试*重新启动任何失败的进程。为了确保您不会无意中忽视任何问题，您可能更希望让 Extract 或 Replicat 进程失败，然后手动解决问题，而不是使用 `AUTORESTART`。


### 性能

本节涵盖了一些关于如何使您的 GoldenGate 配置达到最佳性能的提示和技巧。在项目初期遵循这些建议将有助于确保您日后不会遇到性能问题。

#### 运行性能测试

您应始终通过执行或模拟执行实际生产处理工作负载的测试来检验您的 GoldenGate 配置的性能。这是防范未来问题的最佳保障。理想情况下，测试应在与生产系统硬件和软件相同的系统上进行。此外，您应使用实际生产数据的副本或至少是一个典型的生产工作负载下的生产数据的合理子集进行测试。

在测试期间，您可以捕获 GoldenGate 性能的统计数据并测量延迟，以确定配置是否能处理负载。您还可以收集服务器、网络和数据库的统计数据，并对其进行审查以发现潜在的性能问题。

您可以检查延迟并判断其是否过高而无法满足业务需求。例如，假设在测试期间，您在一个关键的 Replicat 上遇到了 15 分钟的峰值延迟。您的业务要求规定，对于此 Replicat，任何超过十分钟的延迟都是严重问题。在这种情况下，需要进行额外的调整、升级硬件或更改设计以降低延迟。

#### 限制 Extract 的数量

您应尝试使用单个 Local Extract 尽可能多地提取数据。Extract 进程通常不是性能瓶颈，因为它们只是读取数据。性能延迟通常发生在处理更改的 Replicat 上，而非 Extract。此外，如果需要过滤数据，请使用数据泵创建单独的流发送给 Replicat。

例如，假设您正在从 `CUSTOMER` 表中提取数据，并复制到两个不同的数据库：一个用于报告，另一个用于高可用性。`CUSTOMER` 表应**仅从源数据库提取一次**，然后您可以使用两个数据泵来过滤数据并将其发送到各自不同的目标数据库。

#### 为数据泵使用 Passthru 模式

如果数据泵仅是将数据传递到目标系统而不进行任何过滤，请使用数据泵 Extract 的 `PASSTHRU` 参数。这会使数据泵跳过从数据库或定义文件读取任何数据定义，从而提高性能。请注意，要使用 `PASSTHRU`，被复制的表必须完全相同。

#### 使用并行 Replicat

为了提高性能，您可以将 Replicat 拆分为并行处理组。以下是拆分 Replicat 时的一些考虑因素：

*   您不应将涉及参照完整性约束的表拆分到不同的 Replicat 中。
*   一种方法是，为每个模式使用不同的 Replicat。如果模式很大，则将其拆分为多个 Replicat。
*   高容量表可以拆分为它们自己的 Replicat。如果需要进一步分离，您可以使用 GoldenGate 的 `RANGE` 函数，基于表内的键范围将这些高容量表拆分到单独的 Replicat 中。例如，`Orders` 表可以根据 `ORDERID` 键列拆分为两个 Replicat，如下例所示。

```
-- 在第一个 Replicat 参数文件中
MAP SCHEMA_OWNER.ORDERS, TARGET SCHEMA_OWNER.ORDERS_AVAILABILITY, COLMAP (USEDEFAULTS), FILTER (@RANGE (1,2));

-- 在第二个 Replicat 参数文件中
MAP SCHEMA_OWNER.ORDERS, TARGET SCHEMA_OWNER.ORDERS_AVAILABILITY, COLMAP (USEDEFAULTS), FILTER (@RANGE (2,2));
```

#### 使用最快的可用存储

如果性能是关注点，您应始终为 GoldenGate 跟踪文件使用最快的可用存储。为了获得最佳性能，请将跟踪文件存储在独立挂载点的专用磁盘上。跟踪文件的大小应根据捕获的数据量进行适当调整。有关磁盘要求和基于您事务日志活动的容量计算公式，请参阅 第 2 章 "安装"。

#### 调优数据库

通常，GoldenGate 性能问题并非真正由 GoldenGate 引起，而是底层 DBMS 存在问题。在开始复制之前，您应确保数据库的性能是足够的。然后，在复制运行期间，执行 DBMS 性能监控报告（例如 Oracle 的 AWR），并确认没有突出的性能问题。如果可能，请确保您的数据库表拥有最新的性能统计信息，并在 GoldenGate 使用的关键列上建有适当的索引。

### 总结

您可以使用以下问题清单来确认您是否充分利用了本章介绍的提示和技巧：

*   需求与规划
    *   您是否清楚地理解了业务目标，并且这些目标已得到利益相关者的批准？
    *   您是否牢固理解了需求？
    *   您是否确定了适当的复制拓扑结构？
*   安装与设置
    *   您是否创建了专用的 GoldenGate 数据库和操作系统用户？
    *   您是否对 GoldenGate 参数文件中的密码进行了加密？
    *   您是否创建了专用的 GoldenGate 安装目录？
    *   您是否使用了数据库检查点表？
    *   您是否验证了源和目标数据库的字符集？
    *   您是否遵循了 GoldenGate 组件的命名标准？
    *   您是否使用了数据泵？
*   管理与监控
    *   您是否掌握了 `GGSCI` 命令快捷方式？
    *   您是否为命令使用了 `OBEY` 文件？
    *   您是否为您的 Extract 和 Replicat 生成了临时报告统计信息？
    *   您是否使用了废弃文件？
    *   您是否报告了 Extract 和 Replicat 的运行状况？
    *   您是否清除了旧的跟踪文件以防止磁盘被填满？
    *   您是否设置了 Extract 和 Replicat 自动启动？
*   性能
    *   您是否运行了足够的性能测试？
    *   您是否限制了本地 Extract 的数量？
    *   您是否为数据泵使用了 passthru 模式？
    *   如果需要，您是否设置了并行 Replicat？
    *   您是否为您的跟踪文件使用了尽可能最快的存储？
    *   您是否调优了底层的源和目标数据库？

您应在 GoldenGate 项目期间审查此清单。让您的答案引导您实施可能遗漏的最佳实践。实施这些实践通常会带来高效运行的 GoldenGate 环境。

## 附录

## Oracle GoldenGate 管理员的附加技术资源

既然您已经阅读完本书，我们希望您在掌握 Oracle GoldenGate 的征途上奠定了坚实的基础。为助您一臂之力，我们为您提供了一份技术资源列表和命令技巧，可作为您使用该产品时的参考指南。本附录包含以下三个部分：

*   第一部分为您提供了一个参考文献列表，以供进一步阅读和研究本书中讨论的 Oracle GoldenGate 主题。
*   第二部分为您提供了一个常用 Oracle GoldenGate 命令的列表，包含语法和使用示例。
*   第三部分包含了一系列有用的 `Logdump` 命令和语法，用于排查 Oracle GoldenGate 环境问题。

### Oracle GoldenGate 命令快速指南

对于刚接触 Oracle GoldenGate 的新手以及经验丰富的从业者来说，Oracle GoldenGate 命令宛如一片需要掌握的复杂丛林。本节向您展示常用命令，以助您完成 Oracle GoldenGate 中的各项任务。当您需要查询某个命令的语法时，它也可作为参考。



#### ADD

`ADD`命令的语法为：`ADD EXTRACT|REPLICAT`

在 GoldenGate 软件命令接口（GGSCI）中执行的`ADD`命令，是向 Oracle GoldenGate 添加新配置的方法。您可以使用它来添加新的 Extract、Replicat 或数据泵组，也可以在源或目标系统上为 Oracle GoldenGate 配置添加本地或远程跟踪文件。

示例：

```bash
ADD EXTRACT extora, RMTHOST prod, MGRPORT 7800, RMTNAME prodfin
ADD REPLICAT repora, EXTTRAIL d:\ggs\dirdat\rt
```

#### GGSCI

`GGSCI`是进入命令行界面的命令，用于执行 Oracle GoldenGate 的大多数管理任务。最简单的形式是，您打开一个 shell 提示符，从 Oracle GoldenGate 基本安装目录中键入`GGSCI`。您还可以为 Windows 或 UNIX/Linux 设置`path`变量作为直接链接或别名，以简化`GGSCI`命令的使用。

示例：

```bash
C:\ggs\mssqlserver>ggsci
```

```text
Oracle GoldenGate Command Interpreter for ODBC
Version 11.1.1.0.0 Build 078
Windows (optimized), Microsoft SQL Server on Jul 28 2010 18:55:52

Copyright (C) 1995, 2010, Oracle and/or its affiliates. All rights reserved.
```

#### HELP

`HELP`命令提供语法帮助，用于查找特定 Oracle GoldenGate 命令的参数和详细信息。最基本的形式是，登录 GGSCI 界面后，从 GGSCI 命令 shell 提示符键入`HELP`命令，如下例所示：

```text
GGSCI (oracledba) 1> help

GGSCI Command Summary:

Object:           Command:
SUBDIRS           CREATE
ER                INFO, KILL, LAG, SEND, STATUS, START, STATS, STOP
EXTRACT           ADD, ALTER, CLEANUP, DELETE, INFO, KILL,
                  LAG, SEND, START, STATS, STATUS, STOP
EXTTRAIL          ADD, ALTER, DELETE, INFO
GGSEVT            VIEW
MANAGER           INFO, SEND, START, STOP, STATUS
MARKER            INFO
PARAMS            EDIT, VIEW
REPLICAT          ADD, ALTER, CLEANUP, DELETE, INFO, KILL, LAG, SEND,
                  START, STATS, STATUS, STOP
REPORT            VIEW
RMTTRAIL          ADD, ALTER, DELETE, INFO
TRACETABLE        ADD, DELETE, INFO
TRANDATA          ADD, DELETE, INFO
CHECKPOINTTABLE   ADD, DELETE, CLEANUP, INFO

Commands without an object:
(Database)       DBLOGIN, LIST TABLES, ENCRYPT PASSWORD
(DDL)            DUMPDDL
(Miscellaneous)  FC, HELP, HISTORY, INFO ALL, OBEY, SET EDITOR, SHELL,
                 SHOW, VERSIONS, ! (note: you must type the word
                 COMMAND after the ! to display the ! help topic.)
                 i.e.: GGSCI (sys1)> help ! command
For help on a specific command, type HELP <command> <object>.
```

例如：`HELP ADD REPLICAT`

在下面的示例中，您需要查找`ADD EXTRACT`命令的语法和描述：

```text
GGSCI (oracledba) 2> help add extract

ADD EXTRACT

Use ADD EXTRACT to create an Extract group. Unless a SOURCEISTABLE task
or an alias Extract is specified, ADD EXTRACT creates checkpoints so
that processing continuity is maintained from run to run. Review the
Oracle GoldenGate Windows and UNIX Administrator s Guide before
creating an Extract group.
```

#### INFO

`INFO`命令提供当前 Oracle GoldenGate 配置的状态信息。一些示例如下。

`INFO ALL`显示 GoldenGate 处理的完整概览：

```text
GGSCI (oracledba) 11> info all allprocesses

Program     Status      Group       Lag         Time Since Chkpt

MANAGER     RUNNING
EXTRACT     STOPPED     EATAA       00:00:00    1435:43:49

GGSCI (oracledba) 12> info all

Program     Status      Group       Lag         Time Since Chkpt

MANAGER     RUNNING
EXTRACT     STOPPED     EATAA       00:00:00     1435:43:52
```

`INFO EXTRACT {extract process group name}`显示 Extract 的状态报告，如下例所示：

```text
GGSCI (oracledba) 15> info extract eataa

EXTRACT    EATAA     Initialized   2011-03-15 21:50   Status STOPPED
Checkpoint Lag       00:00:00 (updated 1435:45:44 ago)
Log Read Checkpoint  Oracle Redo Logs
                     2011-03-15 21:50:47  Seqno 0, RBA 0
```

您也可以使用`*`通配符来显示所有 Extract：

```text
GGSCI (oracledba) 14> info extract *

EXTRACT    EATAA     Initialized   2011-03-15 21:50   Status STOPPED
Checkpoint Lag       00:00:00 (updated 1435:45:09 ago)
Log Read Checkpoint  Oracle Redo Logs
                     2011-03-15 21:50:47  Seqno 0, RBA 0
```

要查看本地和远程跟踪文件的状态和详细信息，您可以输入`INFO EXTTRAIL *`命令：

```text
GGSCI (oracledba) 16> info exttrail *

       Extract Trail: AA
             Extract: EATAA
               Seqno: 0
                 RBA: 0
           File Size: 10M
```

#### SEND

GGSCI 命令`SEND`用于更新消息传递工具以刷新状态报告实用程序。此外，您可以使用它向 Oracle GoldenGate 处理组发送处理指令——例如，指示 Extract 或 Replicat 执行特定任务。

示例：

```bash
GGSCI (oracledba) 37> send mgr, getlag
```

#### STATUS

`STATUS`命令允许您检查当前 Oracle GoldenGate 环境中处理组（如 Replicat 和 Manager）的当前状态。

示例：

```bash
GGSCI (oracledba) 40> status mgr
```

```text
Manager is running (IP port oracledba.50001).
```

您可以在 Oracle GoldenGate 文档中在线提供的参考指南中找到完整的命令列表。

### 用于故障排除的 Logdump 命令和语法

Oracle GoldenGate Logdump 实用程序是分析和故障排除的瑞士军刀。您应该理解并学会使用它，因为它是您工具库中的一个强大工具。通过使用 Logdump，您可以查看跟踪文件中存在的事务数据，识别丢失的事务，并复制问题（例如重复和丢失的数据事务）。让我们看看一些与 Logdump 一起使用的常见技术和命令。

#### 访问 Logdump 实用程序

Logdump 是一个操作系统实用程序，可从 GGSCI 界面外部访问。要启动新的 Logdump 会话，请导航到 Oracle GoldenGate 基本安装目录，并键入命令`logdump`，如下所示：

```bash
C:\ggs\mssqlserver>logdump
```

```text
Oracle GoldenGate Log File Dump Utility
Version 11.1.1.0.0 Build 078

Copyright (C) 1995, 2010, Oracle and/or its affiliates. All rights reserved.

Logdump 8 >
```

默认情况下，Logdump 本身不执行任何操作。它会打开一个唯一的提示会话，等待您的命令来执行任务。

#### 获取 Logdump 语法帮助

Logdump 包含数十个命令，对新的甚至有经验的 Oracle GoldenGate 管理员都构成挑战。要获取其神秘语法的帮助，您可以输入 help 命令，如下所示：

```text
Logdump 7 >help
```



`FC [<num> | <string>]`      - 编辑上一条命令
`HISTORY`                   - 列出之前的命令
`OPEN | FROM`  `<filename>`   - 打开一个日志文件
`RECORD | REC`              - 显示审计记录
`NEXT [ <count> ]`          - 显示下一条数据记录
`SKIP [ <count> ] [FILTER]` - 跳过 `<count>` 条记录
     `FILTER`               - 在跳转时应用筛选器
`COUNT`                     - 统计文件中的记录数
      `[START[time] <timestr>,]`
      `[END[time] <timestr>,]`
      `[INT[erval] <minutes>,]`
      `[LOG[trail] <wildcard-template>,]`
      `[FILE <wildcard-template>,]`
      `[DETAIL ]`
       `<timestr>` 格式为
         `[[yy]yy-mm-dd] [hh[:mm][:ss]]`
`POSITION [ <rba> | FIRST | LAST | EOF ]` - 设置文件中的位置
         `REVerse | FORward`              - 设置读取方向
`RECLEN [ <size> ]`  - 设置最大输出长度
`EXIT | QUIT`        - 退出程序
`FILES | FI | DIR`   - 显示文件名
`ENV`                - 显示当前设置
`VOLUME | VOL | V`   - 更改默认卷
`DEBUG`              - 进入调试器
`GHDR  ON | OFF`     - 切换 GHDR 显示
`DETAIL ON | OFF | DATA` - 切换详细数据显示
`RECLEN <nnn>`        - 设置数据显示长度
`SCANFORHEADER (SFH)  [PREV]`  - 搜索头的开始
`SCANFORTYPE   (SFT)` - 查找下一条 `<TYPE>` 类型的记录
      `<typename> | <typenumber>`
      `[,<filename-template>]`
`SCANFORRBA    (SFR)` - 查找下一条具有 `<SYSKEY>` 的记录
      `<syskey>`                - syskey = -1 扫描下一条记录
      `,<filename-template>`
`SCANFORTIME  (SFTS)` - 查找具有时间戳的下一条记录
      `<date-time string>`
      `[,<filename-template>]`
         `<date-time string>` 格式为
           `[[yy]yy-mm-dd] [hh[:mm][:ss]]`
`SCANFORENDTRANS (SFET)` - 查找当前事务的结尾
`SCANFORNEXTTRANS (SFNT)` - 查找下一个事务的开头
`SHOW <option>`       - 显示内部信息
      `[OPEN]`        - 列出已打开的文件
      `[TIME]`        - 以各种格式打印当前时间
      `[ENV]`         - 显示当前环境
      `[RECTYPE]`     - 显示记录类型列表
      `[FILTER]`      - 显示活动的筛选器项
`BIO  <option>`       - 设置大块 I/O 信息
      `[ON]`          - 启用大块 I/O（默认）
      `[OFF]`         - 禁用大块 I/O
      `[BLOCK <nnnn>]`- 设置大块 I/O 大小
`TIMEOFFSET <option>` - 设置与 GMT 的时间偏移量
      `[LOCAL]`            - 使用本地时间
      `[GMT]`              - 使用 GMT 时间
      `[GMT +/- hh[:mm]]`  - 与 GMT 的偏移 +/- 
`FILTER SHOW`
`FILTER ENABLE | ON`   - 启用筛选
`FILTER DISABLE | OFF` - 禁用筛选
`FILTER CLEAR [ <filterid> | <ALL> ]`
`FILTER MATCH     ANY | ALL`
`FILTER [INClude | EXCLude] <filter options>`
   `<filter options>` 包括
       `RECTYPE  <type number | type name>`
       `STRING [BOTH] /<text>/ [<column range>]`
       `HEX      <hex string>  [<column range>]`
       `TRANSID  <TMF transaction identifier>`
       `FILENAME <filename template>`
       `PROCESS  <processname template>`
       `INT16    <16-bit integer>`
       `INT32    <32-bit integer>`
       `INT64    <64-bit integer>`
       `STARTTIME <date-time string>`
       `ENDTIME   <date-time string>`
       `SYSKEY   [<comparison>] <32/64-bit syskey>`
       `SYSKEYLEN [<comparison>] [<value>]`
       `TRANSIND [<comparison>] <nn>`
       `UNDOFLAG [<comparison>] <nn>`
       `RECLEN   [<comparison>] <nn>`
       `AUDITRBA [<comparison>] <nnnnnnnn>`
       `ANSINAME <ansi table name>`
       `GGSTOKEN <tokenname> [<comparison>] [<tokenvalue>]`
       `USERTOKEN <tokenname> [<comparison>] [<tokenvalue>]`
       `CSN | LogCSN [<comparison>] [<value>]`
   `<column range>`
       `<start column>:<end column>`，例如 `0:231`
   `<comparison>`
       `=, ==, !=, <>, <, >, <=, >=  EQ, GT, LE, GE, LE, NE`
`X <program> [string]`  - 执行 `<program>`
`TRANSHIST nnnn`        - 设置事务历史记录大小
`TRANSRECLIMIT nnnn`    - 设置低记录数阈值
`TRANSBYTELIMIT nnnn`   - 设置低字节数阈值
`LOG {STOP} | { [TO] <filename> }` - 写入会话日志
`BEGIN <date-time>`     - 使用时间戳设置下一个读取位置
`SAVEFILECOMMENT on | OFF`  - 切换保存文件中的注释记录
`SAVE <savefilename> [!] <options>`  - 将数据写入保存文件
   `<options>` 包括
   `nnn RECORDS | nnn BYTES`
   `[NOCOMMENT]`  - 抑制注释头/尾记录（默认）
   `[COMMENT]`    - 插入注释头/尾记录
   `[OLDFORMAT]`  - 强制使用旧格式记录
   `[NEWFORMAT]`  - 强制使用新格式记录
   `[TRUNCATE ]`  - 清除现有保存文件的数据
   `[EXT ( <pri>, <sec> [,<max>])]` - NSK 上保存文件范围的大小
   `[MEGabytes <nnnn>]`             - 用于范围大小计算
   `[TRANSIND <nnn>]`               - 设置 transind 字段
   `[COMMITTS <nnn>]`               - 设置 committs 字段
`USERTOKEN     on  | OFF | detail`  - 显示用户令牌信息
`HEADERTOKEN   on  | OFF | detail`  - 显示头令牌信息
`GGSTOKEN      on  | OFF | detail`  - 显示 GGS 令牌信息
`FILEHEADER    on  | OFF | detail`  - 显示文件头内容
`ASCIIHEADER   ON  | off`           - 切换头字符集
`EBCDICHEADER  on  | OFF`           - 切换头字符集
`ASCIIDATA     ON  | on`            - 切换用户数据字符集
`EBCDICDATA    on  | OFF`           - 切换用户数据字符集
`ASCIIDUMP     ON  | off`           - 切换十六进制/ASCII 显示的字符集
`EBCDICDUMP    on  | OFF`           - 切换十六进制/ASCII 显示的字符集
`TRAILFORMAT   old | new`           - 强制指定日志类型
`PRINTMXCOLUMNINFO  on | OFF`       - 切换 SQL/MX 列信息显示
`TMFBEFOREIMAGE     on | OFF`       - 切换 TMF 前映像的显示
`FLOAT  <value>`                    - 解释一个浮点数
       `[FORMAT <specifier>]`       - sprintf 格式，默认为 `%f`

`Logdump 8 >`

#### HISTORY

`HISTORY` 命令显示在 Logdump 会话中最近执行的所有活动，如下例所示：

```
Logdump 8 >history
1> ghr on
2> help
3> open dirdata/aa
4> open aa
5> open c:\ggs_src\trails\aa
6> host
7> help
8> history
```


#### 使用 Logdump 打开 GoldenGate 跟踪文件

Oracle GoldenGate 中 `Logdump` 最常见的用途是打开跟踪文件以进行分析和故障排查。启动 `Logdump` 会话后，您需要像下面这样发出 `OPEN` 命令，指定路径名和跟踪文件名：

```
Logdump 1>open ./dirdat/aa00001
Current LogTrail is /ggs/orasrc/dirdat/aa00001
```

打开跟踪文件后，您需要启用一些有用的功能以充分利用 `Logdump` 工具。第一条命令是 `GHDR ON`。它会为 Oracle GoldenGate 跟踪文件中的事务显示记录头：

```
Logdump 2 >ghdr on
```

接下来，您需要切换选项以查看跟踪文件数据的十六进制和 ASCII 值。为此，请执行详细数据命令：

```
Logdump 3 >detail data
```

现在，您可以在 `Logdump` 中使用 `next` 命令移动到跟踪文件中的下一条记录。当您发出 `next` 命令时，`Logdump` 会向前移动一条记录：

```
Logdump 5>next
___________________________________________________________________
Hdr-Ind    :     E  (x45)     Partition  :     . (x00)
UndoFlag   :     . (x00)     BeforeAfter:     A  (x41)
RecLength  :     0  (x0000)   IO Time    : 2011/05/04 21:22:17.611.797
IOType     :   151  (x97)     OrigNode   :     0  (x00)
TransInd   :     . (x03)     FormatType :     R  (x52)
SyskeyLen  :     0  (x00)     Incomplete :     . (x00)
AuditRBA   :          0       AuditPos   : 0
Continued  :     N  (x00)     RecCount   :     0  (x00)

2010/08/04 21:22:17.611.797 RestartOK            Len     0 RBA 936
Name:
After  Image:                                             Partition 0   G  s
```

在这里，您可以检查事务在 `Logdump` 中显示的前后映像。

有关在 Oracle GoldenGate 中使用 `Logdump` 工具的更多详细信息，请参阅《Oracle GoldenGate 故障排查指南》，其中包含所有 `Logdump` 命令、语法和用法的列表。该指南可在以下网址在线获取：适用于 10.x 版本的 [`http://download.oracle.com/docs/cd/E15881_01/doc.104/gg_troubleshooting_v104.pdf`](http://download.oracle.com/docs/cd/E15881_01/doc.104/gg_troubleshooting_v104.pdf) 以及适用于 11.1 版本的 [`http://download.oracle.com/docs/cd/E18101_01/doc.1111/e17792.pdf`](http://download.oracle.com/docs/cd/E18101_01/doc.1111/e17792.pdf)。

## 索引

### ![images](img/square.jpg)A

`Abend` 故障，Replicat 进程，236–237

`access.log` 文件，193

活动-活动复制，43，106，129

`ADD` 命令，293

`ADD EXTRACT` 命令，67，229，236

`ADD REPLICAT` 命令，91

`ADD RMTTRAIL` 命令，67，121

`ADD TRACETABLE` 命令，109

`ADD TRANDATA` 命令，55–56，239

Administrator 组件，Director，190

Administrator 角色名，Tomcat Web 服务器管理工具屏幕，187

高级复制，2–3

`代理`

C 代码，173

Java，173

Veridata，173

`警报`

检查点延迟，213

事件文本，213

设置，214

使用 Director 应用程序设置，212–214

别名 Extract，294

`ALTARCHIVELOGDEST <路径名>` 参数，154，235

`alter extract begin [now]|[timestamp]` 命令，156

`ALTER EXTRACT` 命令，`GGSCI`，229

`ALTER EXTTRAIL` 命令，`GGSCI`，222

`ALTER REPLICAT <组>, EXTTRAIL <跟踪文件名>` 命令，`GGSCI`，236

`ALTER RMTTRAIL` 命令，`GGSCI`，222

`ALTER TABLE`，57

`AMERICAN_AMERICA.AL32UTF8`，61

Apache Tomcat Web 服务器端口，186

`APPEND` 参数，89

`AQ (Oracle 高级队列)`，3

`架构`，33–47

`组件`，34–40

捕获进程，35–36

Collector 进程，38

数据泵，36–38

数据库，34–35，39–40

投递，39

本地 Extract 进程，35–36

Manager 进程，40

网络，38

Replicat 进程，39–39

工具和实用程序，45–47

`DEFGEN`，46

Director，46–47

`GGSCI`，46

`Logdump`，46

反向器，46

Veridata，46

拓扑和用例，40–45

典型流程，33–34

归档日志，检查，157–158

归档，数据库，235

`ASM (自动存储管理)`，其配置选项，102–103

指定用户，102

更新监听器，102

更新 `TNSNAMES.ORA` 文件，102–103

`ASMUser` 参数，103

`ASSUMETARGETDEFS` 参数，配置

用于初始加载 Replicat 文件，73

用于 Replicat 进程，80

星号，268

属性，在 Director 应用程序中查找，209–210

自动密钥序列，109

自动进程，启动和重启，90

自动存储管理。*参见* `ASM`，其配置选项

自动化，监控，162–169

磁盘空间，168–169

延迟脚本，163–167

内存和 CPU 脚本，168

进程脚本，162–163

`AUTORESTART` 参数，90，287

`AUTOSTART` 参数，90


### ![images](img/square.jpg)B

`bad data`, 174
`bad parameter errors`, 230
`baselines of performance`, 129–131
`basic replication`, 2
`BATCHSQL` 参数, 146–149
`BEGIN NOW` 命令, 63, 67
`bidirectional replication`, 42–43, 106–110
- 排除其事务, 108–109
- 处理其冲突解决, 109–110
`broadcast replication`, 44
`built-in editor`, 使用其修改 `Manager` 参数文件, 193
`business objectives`, 279–280
`by table`, 158

### ![images](img/square.jpg)C

`capture process`, 35–36
`cascade-delete constraints`, 禁用, 57–58
`C-Code agent`, 173
`CDC` (Change Data Capture), 56
`Central Processing Unit (CPU) scripts`, 监控, 168
`certification matrix`, Oracle Support, 112
`c:\ggate\sql2008` 目录, 282
`C:/ggs_src` 目录, 19
`C:/ggs_trgt` 目录, 19
`Change Capture process`, 为复制配置, 121–124
- IBM DB2 UDB 到 Oracle, 124
- MySQL 到 Oracle, 122
- Sybase 到 Oracle, 123
- Teradata 到 Oracle, 123
`Change Data Capture (CDC)`, 56
`change_date` 列, 182
`character fields`, 对其使用 `RTRIM`, 181
`character sets`
- 其配置问题, 239
- 验证, 283
`Check Connection` 选项, 在 `Director Admin Tool` 中, 193
`checkemployee.empduplicate = 0`, 110
`checkemployee.empduplicate > 0`, 110
`CHECKPARAMS` 参数, 230
`checkpoint` 表, 90–92, 236–237, 271, 282
`checkpoints`, 39
`chmod` 命令, 229
`chown` 命令, 229
`Client` 组件, `Director`, 192
`client installation requirements`, 针对 Oracle GoldenGate Director 11g, 26
`Clustered environments`, 18
`Collector process`, 38
`COLS` 列表, 95
`COLS` 参数, 95
`COLSEXCEPT` 参数, 96
`Column Mapping` 屏幕, 173
`columns`
- 比较不同类型的列, 182
- 排除列, 180
- 过滤列, 95–96
- 映射列, 97–99
  - 生成数据定义文件, 97–98
  - 转换列, 98–99
- 缺失的列, 239
`command line`, `Vericom` 界面的, 185–186
`commands`
- `Logdump` 实用程序, 296–300
  - 访问, 296
  - 获取语法帮助, 296–299
  - `HISTORY` 命令, 299
  - 用其打开跟踪文件, 299–300
  - 快速指南, 292–300
  - `ADD` 命令, 293
  - `GGSCI` 命令, 293
  - `HELP` 命令, 293–294
  - `INFO` 命令, 294–295
  - `SEND` 命令, 295
  - `STATUS` 命令, 295
`compare formats`, 比较不同格式, 182
`compare methods`, 180
`Compare Pair Configuration` 屏幕, 181–182
`Compare Pair Mapping` 屏幕, 178
`Compare Pair Reports`, 184



### 对比对

177–178

### 对比

*   不同列类型的比较和对比格式：182
*   大表的增量数据对比：181–182
*   实时复制数据的对比：182
*   使用 Veridata 工具进行对比：173–179

### 对比对

177–178

*   数据库连接：175–176
*   组：177
*   作业：178–179
*   配置文件：179
*   表和数据脚本：176–177

### 组件

34–40

*   捕获（Capture）进程：35–36
*   `Collector` 进程：38
*   数据泵（Data Pump）：36–38
*   数据库
    *   源端：34–35
    *   目标端：39–40
*   投递（Delivery）：39
*   `Local Extract` 进程：35–36
*   `Manager` 进程：40
*   网络：38
*   `Replicat` 进程：39
*   跟踪文件（Trails）
    *   远程跟踪文件：38–39
    *   源端跟踪文件：36

### `COMPRESS` 选项

151

### 压缩的行哈希数据

171

### `compressupdates` 参数

209

### 配置

*   用于 ASM：102–103
*   字符集的配置问题：239
*   DDL 复制：103–106
*   配置问题：226–234
    *   数据库可用性：227–228
    *   不正确的软件版本：227
    *   缺失的进程组：228
    *   缺失的跟踪文件：228
    *   网络问题：231–232
    *   操作系统问题：230–231
    *   参数文件问题：229–230
*   用于 RAC：100–102

### `config.xml` 文件

191

### `Confirm Out of Sync` 步骤

已禁用：180

### 确认对比

173

### `Confirmation Compares` 配置文件设置

180

### `Confirmed-Out-Of-Sync (COOS)` 报告

174

### 冲突解决

双向复制的处理：109–110

### 连接配置

176

### `Connection Configuration` 选项

176

### 数据库连接

175–176

### 约束

*   级联删除（Cascade-Delete），已禁用：57–58
*   键约束，表缺失：239

### 重做日志的消耗速率

158–159

### `convchk` 实用程序

237–238

### `COOS (Confirmed-Out-Of-Sync)` 报告

174

### 损坏的检查点错误

237

### CPU（中央处理器）脚本

监控：168

### `CREATE TABLE AS SELECT` 选项

113

### 数据库权限与凭据

27

### 跨数据库对比

174

### CSI（客户支持标识符）账户

292

### `CUSTOMER` 表

288

### 迁移割接

274–276



### ![images](img/square.jpg)D

### 数据同步问题

- 238–239
- 传输问题，232–234

### 数据时效性

- 50
- 128
- 280

### 数据定义语言（DDL）复制

- 103–106

### 数据定义文件，生成

- 97–98

### 数据过滤

- 列，95–96
- 行，96
- 表，94–95

### 数据映射，列

- 97–99
- 生成数据定义文件，97–98
- 转换，98–99

### 数据迁移项目

- 175

### Data Pump 进程

- 204–206

### 数据泵

- 36–38
- 65–70
- 283
- 添加，67
- 配置，65–67
  - `Extract` 参数，66
  - `PASSTHRU` 参数，66–67
  - `RMTHOST` 参数，67
  - `RMTTRAIL` 参数，67
  - `TABLE` 参数，67
- 为零停机迁移配置，269–270
- 灾难恢复复制，246–247
- 相关错误，236
- 回退，为零停机迁移配置，272–273
- `Passthru` 模式，288
- 启动与停止，68
- 验证，68–70

### 数据需求

- 50
- 280

### 数据脚本

- 176–177

### 数据源，Director 应用程序

- 192–193

### 数据验证

- 271

### 数据量

- 50
- 128
- 280

### 数据库组件，Director Server 及

- 190

### 数据库连接

- 175–176

### 数据库错误消息

- 225

### 数据库管理系统

- 参见 `DBMS`；`Sybase DBMS`

### 数据库迁移项目

- 175

### 数据库性能统计信息

- 130

### 数据库权限

- 与凭据，授予 Oracle GoldenGate Director Server 模式，27
- Oracle GoldenGate `Veridata`，28–30

### 数据库服务器版本，Oracle GoldenGate 支持的

- 16

### 数据库排序方法

- 181

### 数据库级补充日志记录

- 启用，54–55
- 验证，54

### 数据库

- 配置
  - Microsoft SQL Server 到 Oracle 复制，113–114
  - 源 Oracle 系统，114
- 连接到，100–101
- 相关问题
  - 可用性，227–228
  - 字符集配置问题，239
  - 数据泵错误，236



### 数据同步与 Oracle GoldenGate 组件

### 数据同步问题
* 238–239

### Extract 进程
* 235

### 提取失败
* 239–240

### 缺少列的错误
* 239

### Replicat 进程
* 236–238

### 表缺少主键约束
* 239

### 源端
* 34–35, 235

### 目标端
* 39–40

### 调优
* 152, 289–290

### 数据泵 Extract 组
* 247, 254

### 添加
* 273

### 启动
* 276, 278

### 停止
* 275, 277

### `DBLOGIN`命令
* 55, 222, 235

### DBMS（数据库管理系统）

### 配置选项
* 99–106

### 用于 ASM
* 102–103

### DDL 复制
* 103–106

### 用于 RAC
* 100–102

### 实用程序，用于加载
* 76

### DBMS 厂商加载实用程序
* 270

### `DBOPTIONS`参数
* 57

### DDL（数据定义语言）复制
* 103–106

### `ddl_setup.sql`
* 104

### 创建专用安装目录
* 282

### 创建专用用户
* 281

### `DEFERREFCONST`选项
* 57

### `defgen`命令
* 98

### `DEFGEN`实用程序
* 46, 207

### 定义文件

### 生成
* 207–209

### 将源传输到目标 Microsoft SQL Server 系统
* 114

### 在 Oracle 源系统上执行定义生成器
* 114

### `DELETE REPLICAT`命令，GGSCI
* 222

### 投递
* 39

### 桌面客户端，Director 组件
* 189

### 详细信息，添加和监控
* 156–157

### `detail data`命令，Logdump
* 300

### Detail 特性，Extract
* 206

### `DETAIL`选项
* 64, 69, 82

### `INFO EXTRACT <group>`命令
* 228

### `INFO EXTRACT`命令
* 233

### `INFO REPLICAT <group>`命令
* 228

### `INFO REPLICAT`命令
* 233

### `START EXTRACT <extract name>`命令
* 219

### `DetailReportView`角色名称，Tomcat Web 服务器管理工具屏幕
* 187

### 诊断，报告，在没有诊断信息的情况下处理进程失败
* 221

### `dirdat`目录
* 62, 90

### `dirdef`目录
* 98

### Director 11g，Oracle GoldenGate
* `参见` Oracle GoldenGate Director 11g

### Director 管理工具
* 193

### Director 管理员组件
* 190

### Director 应用程序
* 189–214

### 高级映射
* 210–212

### 警报
* 212–214

### 更改 Extract 和 Replicat 对象的运行选项
* 206

### 更改跟踪文件大小
* 206



### 目录索引

### Director 组件
- 组件构成，189–192
- Director 管理员组件，190
- Director 客户端组件，192
- Director 服务器与数据库组件，190
- Director Web 组件，191

### Director 相关项
- GGSCI 实例，190
- Director 客户端组件，192
- Director 组件，189–190
- Director 环境，13
- Director 监控代理，190
- Director 仓库数据库，190
- Director 服务器与数据库组件，190
- Director 工具，46–47
- Director Web 组件，191

### Data Pump 与数据源
- Data Pump 进程，204–206
- 数据源，192–193
  - 提取 tranlogoptions，206–207
  - 在其中查找参数或属性，209–210
  - 生成定义文件，207–209
- 初始加载任务，195–200
- 单向复制，200–203

### 目录与路径
- 目录，安装，282
- diroby 目录，63, 81, 285
- dirprm 子目录，59, 77, 91, 229
- dirrpt 目录，133

### 灾难恢复复制
- 灾难恢复复制，241–262
  - 用于计划内切换，251–256
  - 前提条件，241–242
  - 要求，242–243
  - 设置，244–251
    - data pump，246–247
    - 本地 Extract，245–246
    - Replicat，247–248
    - 备用 data pump，249–250
    - 备用 Extract，248–249
    - 备用 Replicat，250–251
  - 拓扑结构，243
  - 用于计划外故障切换，256

### 丢弃文件与参数
- 丢弃文件，223–225, 286
  - 丢失，224
  - 未打开，225
  - 过大，224
- `DISCARDFILE` 参数，89, 223–226
- `DISCARDROLLOVER` 参数，89, 224

### 磁盘相关
- 磁盘
  - 要求
    - 在 Microsoft Windows 和 UNIX 操作系统上安装 Oracle GoldenGate for Teradata，22
    - Oracle GoldenGate Veridata，28–29
  - 空间，17, 168–169
  - 存储，149–150

### 其他术语
- 分布式处理，1–2
- DNS (域名服务器)，231
- `DOWNCRITICAL` 参数，286
- 下载软件，5–12
  - 从 Oracle e-delivery，5–8
  - 从 OTN，9–12
- `DOWNREPORT` 参数，286
- `DYNAMICPORTLIST` 参数，232



### E

`编辑数据库连接设置`，`比较值时截断尾部空格`选项，181

`编辑参数`功能，209

`EDIT PARAMS`参数文件名，59，77

编辑器，使用编辑器修改 Manager 参数文件，193

`EM`缩写，268

加密

密码，92，282

跟踪文件，93–94

`ENCRYPTTRAIL`参数，93

`env`命令，230

`env|grep ORA`命令，231

环境，12–14

错误日志，162–223

错误消息 501，226

错误

缺少列，239

Replicat 进程，236–238

事件日志，162

事件查看器，Windows，116

退出代码，186

`Extract`和`Replicat`参数文件，286

`Extract`检查点，232–234

`Extract`组，268

添加，63

启动，277

启动和停止，63

验证，63–65

`Extract`层，174

`Extract`对象，为其修改`RUN`选项，206

`Extract`参数

为数据泵配置，66

为本地`Extract`配置，60

文件，71，85，225

`Extract`进程，54–65

添加和监控详细信息，156–157

配置本地`Extract`，59–62
- `Extract`参数，60
- `EXTTRAIL`参数，62
- `SETENV`参数，61
- `TABLE`参数，62
- `USERID`参数，61–62

禁用触发器和级联删除约束，57–58

编辑`Run`选项，206

无法访问数据库归档和重做日志时出错，235

`Extract`组
- 添加，63
- 启动和停止，63
- 验证，63–65

故障，219–221

首次任务，200

`GoldenGate`，191

监控的重要性，155

本地，配置，59–62

并行
- 通过表过滤实现，135–141
- 概述，134–135
- 使用键范围，141–146

由`Extract`生成报告，86–89
- `REPORT`参数，88–89
- `REPORTCOUNT`参数，88
- `REPORTROLLOVER`参数，89

源数据库问题，235

补充日志
- 数据库级，54–55
- 表级，55–57

验证`Manager`状态，58–59

`Extract`报告文件，231

`Extract` `TABLE`参数，230

限制`Extract`数量，288

`Extracts`和`Replicats`，287

`EXTTRAIL`参数，62，81

`EXTTRAILSOURCE`参数，67

### F

故障

提取，239–240

`Replicat`进程
- `Abend`故障，236–237
- 大型事务上，238

回退方案
- 数据泵，为零停机迁移配置，272–273
- 本地`Extract`，为零停机迁移配置，271–272
- 迁移，276–278
- `Replicat`进程，为零停机迁移配置，273–274

提取故障，239–240

`FETCHOPTIONS NOUSESNAPSHOT`参数，240

文件大小字段，222

文件访问问题，231

文件
- 定义，生成，207–209
- 废弃文件，223–225
    - 缺失，224
    - 未打开，225
    - 过大，224
- 参数文件，配置问题，229–230
- 跟踪文件
    - 更改大小，206
    - 缺失，228
    - 使用`Logdump`实用程序打开，299–300
    - 问题，221–223

文件系统目录，38

`FILTER`选项，Oracle GoldenGate `Replicat`参数，222

`FILTER`参数，96

过滤
- 列，95–96
- 行，96
- 表，94–95，135–141

已完成作业面板，`Veridata`主屏幕，180，183

`first_change#`，157

`FORMAT RELEASE`选项，62

`FORMATASCII`参数，238

`FORMATSQL`参数，238

`FORMATXML`参数，238

`FUNCTIONSTACKSIZE`参数，230–231


### `G`

`trail`文件的`生成速率`，160–162
`getlag`命令，159–160
`GETREPLICATES`参数，108
`gger.ckpt`表，108
`gger/ggs/dirprm`目录，59，77
`gger/ggs/ggserr.log`目录，64，82
`gger/ggs/ora10`目录，282
`GGSCI` (GoldenGate 软件命令行接口)，46，189–190
`GGSCI`命令，16，121，284–285，293
`ggserr.log`文件，64，68，133，162，213，223
`GGS.GET_LAG`，163
`GHDR ON`命令，Logdump，299
`GLOBALS`文件，91，103，271
`Go to Compare Pair Configuration ...`链接，178
`GoldenGate Director 管理工具`，192
`GoldenGate Director`，定义，189
`GoldenGate GGSCI 管理器进程`，189
`GoldenGate 管理器进程`，49
`GoldenGate RANGE 函数`，288
`GoldenGate 速率`，监控，158–159
`GoldenGate 参考指南`，60
`GoldenGate 复制环境`，51
`GoldenGate 软件命令行接口 (GGSCI)`，46，189–190
```
GRANT DBADM ON DATABASE TO USER <ggs_user>
```, 24
`图形用户界面 (GUIs)`，使用其修改`Manager`参数文件，194–195
`组配置`屏幕，177–178
`Group`参数，71
`组`，177–228
`GROUPTRANSOPS`参数，149–150
`GUIs (图形用户界面)`，使用其修改`Manager`参数文件，194–195

### `H`

`HANDLECOLLISIONS`参数，53，77，79–80，83，247，273
`HELP ADD EXTRACT`，68
`Help`命令，46
`HELP`命令，293–294
`异构复制`，111–125
`Microsoft SQL Server 到 Oracle`，111–118
`数据库配置`，113–118
`初始数据加载完成`，113
`准备环境`，113
`源数据库配置`，114
`示例 Microsoft SQL Server 数据库`，118–124
`验证操作准备情况`，124–125
`HISTORY`命令，285，299
`主机可观察`选项，在`Director 管理工具`中，193
`主机名`，231
`HP OpenView`，234


### ![images](img/square.jpg) I

IBM DB2 UDB（通用数据库）到 Oracle 的复制，124
版本 8.x，124
版本 9.x，124
在 Windows 和 UNIX 上，Oracle GoldenGate 的安装，23–24

`IFCONFIG` 命令，231
`IGNOREREPLICATES` 参数，108
不兼容的记录错误，Replicat 进程，238
增量比较，174
增量数据，针对大表，比较，181–182
进行中状态，182
进行中的事务，235
`INFO <extract name>, DETAIL` 命令，GGSCI，220
`INFO ALL` 命令，218–219，228，294
`info all` GGSCI 提示符命令，156
`INFO` 命令，64，68–69，82，268，271，294–295
`info er *` 命令，284
`INFO EXTRACT <group>` 命令，228–229
`INFO EXTRACT` 命令，63，68，233–234，269，295
`INFO EXTRACTS` 命令，232
`INFO EXTTRAIL *` 命令，GGSCI，222
`INFO EXTTRAIL <trail file name>` 命令，GGSCI，228
`INFO MANAGER` 命令，59
`INFO MGR` 命令，59，231
`info rep *` 命令，284
`INFO REPLICAT <group>` 命令，GGSCI，228
`INFO REPLICAT` 命令，82，233–234，236，248，271
`INFO RMTTRAIL *` 命令，GGSCI，222，236
初始比较，173
`Initial Compares` 配置文件设置，180
初始加载
添加 Replicat 组，74
完成，Microsoft SQL Server 到 Oracle 复制，113
配置 Replicat 参数文件，72–74
`ASSUMETARGETDEFS` 参数，73
`MAP` 参数，73–74
`REPLICAT` 参数，72
`SETENV` 参数，73
`USERID` 参数，73
Extract 参数文件
添加，71
配置，71
先决条件，70–71
启动，74
验证，74–75
`Initial Load` 任务，195–200
安装，5–31，281–283
检查点表，282
创建专用目录，282
创建专用用户，281
数据泵，283
制定命名标准，283
下载软件，5–12
从 Oracle e-delivery，5–8
从 OTN，9–12
加密密码，282
环境，12–14
安装说明，14–15
Oracle GoldenGate，16–24
用于 Windows 和 UNIX 上的 IBM DB2 UDB，23–24
在 Linux 和 UNIX 操作系统上，20
用于 Windows 上的 Microsoft SQL Server，21
在 Microsoft Windows 上，18–20
以及 Oracle RAC 注意事项，21
要求，16–18
用于 Windows 和 UNIX 上的 Sybase，23
用于 Windows 和 UNIX 操作系统上的 Teradata，21–22
Oracle GoldenGate Director 11g，24–26
Oracle GoldenGate Director Server，27
Oracle GoldenGate Veridata，27–31
代理，28–29
系统要求，29–31
验证字符集，283
同步状态，182
集成复制，44–45
临时统计信息，生成，286
`IOSTAT` 命令，150
IP 地址，231
`IPCONFIG` 命令，Windows，231

### ![images](img/square.jpg) J

Java 代理，173
作业，178–179

### ![images](img/square.jpg) K

键约束，缺少表，239
键标识符，109
键范围，使用并行 Extract 和 Replicat 进程，141–146
`KEYCOLS` 选项，238–239
`KEYCOLS` 参数，239



### L

L 缩写, 268

`l2` 跟踪文件, 136

`l3` 跟踪文件, 136, 140

`lag` 命令, 130

`lag` 脚本, 监控, 163–167

`LAGCRITICAL` 参数, 286

`LAGINFO` 参数, 286

### 延迟

各组的监控, 159–162

`getlag` 命令, 159–160

在 `ggserr.log` 文件中记录延迟状态, 162

跟踪文件生成速率, 160–162

`Write Checkpoint` 命令, 160

在 `ggserr.log` 文件中记录状态, 162

`LD_LIBRARY_PATH` 变量, 230

`LHREMD1` 抽取组, 60

`LHREMP2` 本地抽取, 246

`LIMITROWS` 选项, 239

### Linux

GoldenGate 进程故障, 222

安装 Oracle GoldenGate, 20

`listener.ora` 文件, 102

`listener.ora` 网络配置, 220

监听器, 更新, 102

字面量比较, 180

### 加载, 70–75

使用 DBMS 实用程序, 76

初始

添加 Replicat 组, 74

配置 Replicat 参数文件, 72–74

Extract 参数文件, 71

前提条件, 70–71

启动, 74

验证, 74–75

### 本地 Extract

配置, 59–62

`Extract` 参数, 60

`EXTTRAIL` 参数, 62

`SETENV` 参数, 61

`TABLE` 参数, 62

`USERID` 参数, 61–62

为零停机迁移配置, 267–268

灾难恢复复制, 245–246

回退, 为零停机迁移配置, 271–272

进程, 35–36

停止, 274, 276

`本地 Extract LHREMD1`, 142

`本地 Extract` 跟踪文件名, 268

`本地 Extract` 跟踪 `l1`, 139

`本地 Extract` 跟踪 `l2`, 144–145

基于日志的, 2

### Logdump 实用程序, 用于故障排除的命令和语法, 296–300

访问 `Logdump` 实用程序, 296

获取 `Logdump` 语法帮助, 296–299

`HISTORY` 命令, 299

使用 `Logdump` 实用程序打开跟踪文件, 299–300

### 日志记录, 补充

数据库级, 54–55

表级, 55–57

### 日志

检查

归档日志, 157–158

当前在线重做日志, 157

事件和错误日志, 162

重做日志, 监控消耗速率, 158–159

低延迟, 109

`LSNRCTL` 命令, 229



### M

![images](img/square.jpg) 管理与监控，284–287

自动启动进程，287

丢弃文件，286

生成中期统计信息，286

`GGSCI` 命令快捷方式，284–285

`OBEY` 文件，285–286

定期清除旧的跟踪文件，287

定期报告进程健康状况，286

`Manager` 参数文件，修改，193–195

使用内置编辑器，193

使用图形用户界面，194

`Manager` 进程，40, 193, 219, 228

`Manager` 状态，验证，58–59

`MANAGESECONDARYTRUNCATIONPOINT`，62

`Manual Mapping` 选项卡，`Compare Pair Mapping` 屏幕，178

`MAP` 参数，配置

用于初始加载 `Replicat` 文件，73–74

用于 `Replicat` 进程，80–81

`MAP` 语句，96, 110, 222, 239

映射

高级，210–212

列，97–98

规则，197

`marker_setup.sql` 脚本，103

`MAX` 选项，223

`MAXDISCARDRECS` 参数，224

最大阈值，155

`MAXTRANSOPS` 参数，238

`MEGABYTES` 选项，`DISCARDFILE` 参数，222, 225

内存

要求

Oracle GoldenGate，16

Oracle GoldenGate Veridata，28–29

脚本，监控，168

`MGRPORT` 参数，231

Microsoft SQL Server 2000，112

Microsoft SQL Server 2005，112

Microsoft SQL Server 2008，112

Microsoft SQL Server 数据库

到 Oracle 的复制，111–118

数据库配置，113–118

初始数据加载完成，113

准备环境，113

示例，异构复制，118–124

将源定义文件传输到目标，114

Windows 上的 Microsoft SQL Server，安装 Oracle GoldenGate，21

Microsoft Windows，安装 Oracle GoldenGate，18–20

IBM DB2 UDB，23–24

集群环境的要求，18

Sybase DBMS，23

Teradata RDBMS 平台，21–22

迁移，零停机，263–278

`MIN` 选项，223

缺少列错误，239

监控

自动化，162–169

检查磁盘空间，168–169

检查内存和 CPU 的脚本，168

检查进程的脚本，162–163

监控延迟的脚本，163–167

设计策略，153–155

监控 `Extract` 进程的重要性，155

最大阈值，155

磁盘空间，168–169

管理与，284–287

自动启动进程，287

丢弃文件，286

生成中期统计信息，286

`GGSCI` 命令快捷方式，284–285

`OBEY` 文件，285–286

定期清除旧的跟踪文件，287

定期报告进程健康状况，286

内存和 CPU 脚本，168

监控延迟脚本，163–167

进程脚本，162–163

进程，155–162

所有运行中的，156

检查日志，157–158

`Extract` 进程详情，156–157

事件和错误日志，162

`GoldenGate` 速率和重做日志消耗速率，158–159

每个组的延迟，159–162

`msconfig` 命令，117

`MSSQL2K8` 环境，14

并行 `Replicats`，41, 43

MySQL 5.x，122

MySQL RDBMS（关系数据库管理系统），到 Oracle 的复制，122

`MYSQLS` 环境，14

### N

![images](img/square.jpg) 命名标准，制定，283

`netstat` 命令，130, 150

`NETSTAT` 命令，232

网络诊断工具，234

网络性能统计，130

网络要求，51, 281

网络，38

配置问题，231–234

安装 Oracle GoldenGate 的要求，17

调优，150–151

`New Data Source`，120

新数据库，捕获其更改，275

`next` 命令，`Logdump`，299

`NLS_LANG` 变量，61, 73, 78, 239

`No Dynamic ports available` 错误，234

`nocompressupdates` 参数，209

节点，同步，100

`NODYNAMICRESOLUTION` 参数，156

`NOMANAGESECONDARYTRUNCATIONPOINT` 参数，62

`NSort` 排序方法，180


### ![images](img/square.jpg)O

`OBEY command`， 56

`obey filename command`， 63， 81

`OBEY files`， 285–286

`Observer role`，*在 Director Admin Tool 中*， 193

已弃用 `Replicat group`， 222

`OL11SRC` 环境， 13

`OL11TRG` 环境， 13

单向复制， 40–42， 52， 200–203

联机重做日志， 当前， 157

`OOS (不同步) 报告`， 173

`OPEN command`， Logdump， 299

操作系统
配置问题， 230–231
安装 Oracle GoldenGate 的要求， 18

Oracle `11g` 网络监听器， 220

Oracle 高级队列 (`AQ`)， 3

Oracle 电子交付，从中下载软件， 5–8

Oracle GoldenGate Director `11g`， 24–27

Oracle GoldenGate Director Server， 安装， 27

Oracle GoldenGate， 安装， 16–24
用于 Windows 和 UNIX 上的 IBM DB2 UDB， 23
在 Linux 和 UNIX 上， 20
用于 Windows 上的 Microsoft SQL Server， 21
在 Microsoft Windows 上， 18–20
以及 Oracle `RAC` 注意事项， 21
要求， 16–18
用于 Windows 和 UNIX 上的 Sybase， 23
用于 Windows 和 UNIX 上的 Teradata， 21–22

Oracle GoldenGate Manager 用户账户， 222

Oracle GoldenGate 根本原因分析， 217

Oracle GoldenGate 软件， 6

Oracle GoldenGate 故障排除指南， 300

Oracle GoldenGate 用户， 228

Oracle GoldenGate Veridata， 安装， 27–31
代理， 28–29
系统要求， 29–31

Oracle 物化视图， 174

Oracle 协议网络错误， 220

Oracle `RAC (真正应用集群)`， 21

Oracle 支持认证矩阵， 112

Oracle 支持说明 `965703.1`， 237

Oracle 支持说明 `966097.1`， 232

Oracle 技术网络 (`OTN`) 站点， 9–12， 291–292

`ORACLE_HOME`， 61， 73， 231

`ORACLE_SID`， 61， 73， 231

操作系统调度程序， 178

`OTN (Oracle 技术网络)` 站点， 9–12， 291–292

`ourceDB`， 61

`不同步 (OOS) 报告`， 173

`owner` 字段，*在 Director Admin Tool 中*， 193

`owner` 角色，*在 Director Admin Tool 中*， 193


### P

并行文件，`Replicat`参数，288

并行 Replicat 组，142

参数文件编辑器，189

参数文件，229–230, 267

在 Director 应用程序中查找参数，209–210

`PARAMS`参数，229

`passthru`模式，36

`PASSTHRU`模式，68

`Passthru`模式，用于数据泵，288

`PASSTHRU`参数，66–67, 236, 288

密码，加密，92, 282

`PATH`变量，230

`PAYROLL`数据库，266

性能，287–290
创建基线，129–131
定义需求，128–129
评估当前性能，131–132
最快的可用存储，288
限制抽取进程数量，288
并行 Replicat 参数文件，288
用于数据泵的 Passthru 模式，288
运行性能测试，287
调整数据库，289–290
Veridata 工具的性能，180–182
比较方法，180
比较大表的增量数据，181–182
禁用“确认不同步”步骤，180
排除列，180
增加线程数，180
字符字段上的`RTRIM`，181
统计信息，183–185
调整配置文件设置，180

持续不同步状态，182

`PING <主机名>`命令，232

`ping`命令，219

计划切换，用于此场景的灾难恢复复制，251–256

平台，支持 Oracle GoldenGate Director 11g 的平台，25

`PowerUser`角色名称，Tomcat Web 服务器管理工具屏幕，187

`PR`缩写，266, 268

先决条件，用于灾难恢复复制，241–242

预览选项卡，比较对映射屏幕，178

主抽取，35

进程故障，218–221
抽取进程，219–221
无报告诊断信息，221

进程组，缺失，228

进程脚本，监控，162–163

`<进程> paramfile <路径名>.prm`语法，221

进程
自动启动，287
监控，155–162
所有运行中的进程，156
检查日志，157–158
抽取进程的详细信息，156–157
事件和错误日志，162
GoldenGate 速率和重做日志消耗速率，158–159
每个组的延迟，159–162
定期报告健康状况，286
使用`TLTRACE`参数进行跟踪，225

处理延迟，130

处理速率，130

`.profile`启动文件，Oracle GoldenGate 主目录，230

配置文件，179–180

`ps -ef|grep smon`命令，228

`PURGE`选项，`DISCARDFILE`参数，225

`PURGEOLDEXTRACTS`参数，89

`purgeoldextracts`参数，195

`PURGEOLDEXTRACTS`参数，222–223

清除，222–223

### Q

`QUOTA UNLIMITED`权限，27



### R

### RAC（真正应用集群）

配置选项，参见第 100 页至第 102 页
连接到数据库，参见第 100 页至第 101 页
定义线程，参见第 101 页至第 102 页
同步节点，参见第 100 页

### RANGE 过滤

参见第 142 页

### RATE 信息

参见第 88 页

### RBA（相对字节地址）

参见第 224 页、第 234 页

### 真正应用集群

配置选项。*参见* RAC，配置选项

### 真正应用集群（Oracle RAC）

参见第 21 页

### 记录

报告丢弃的记录，参见第 89 页

### 重做日志

当 `Extract` 进程无法访问时报错，参见第 235 页
监控消耗率，参见第 158 页至第 159 页
联机、当前重做日志，参见第 157 页

### 参考资料

扩展阅读，参见第 291 页至第 292 页

### 关系数据库管理系统

*参见* Teradata RDBMS

### 关系数据库管理系统（MySQL RDBMS）

到 Oracle 的复制，参见第 122 页

### 相对字节地址 (RBA)

参见第 224 页、第 234 页

### 远程目录

参见第 34 页、第 38 页至第 39 页

### Replicat

灾难恢复复制，参见第 247 页至第 248 页
组，参见第 74 页、第 271 页、第 274 页、第 283 页、第 287 页
启动，参见第 276 页、第 278 页
停止，参见第 275 页、第 277 页

### `REPLICAT` 关键字

参见第 59 页

### `Replicat MAP` 参数

参见第 230 页

### `Replicat` 对象

更改其运行选项，参见第 206 页

### `REPLICAT` 参数

参见第 72 页、第 79 页、第 231 页

### Replicat 参数

参见第 270 页、第 273 页、第 286 页
为 `Replicat` 进程配置，参见第 78 页
文件，参见第 72 页至第 74 页、第 225 页、第 288 页

### Replicat 进程

参见第 39 页、第 76 页至第 83 页
`Abend` 失败，参见第 236 页至第 237 页
添加，参见第 81 页
配置，参见第 77 页至第 81 页
`ASSUMETARGETDEFS` 参数，参见第 80 页
`HANDLECOLLISIONS` 参数，参见第 79 页至第 80 页
`MAP` 参数，参见第 80 页至第 81 页
`Replicat` 参数，参见第 78 页
`SETENV` 参数，参见第 78 页
`USERID` 参数，参见第 79 页
用于零停机迁移，参见第 270 页至第 271 页
编辑运行选项，参见第 206 页
错误，参见第 236 页
大型事务失败，参见第 238 页
回退方案，为零停机迁移配置，参见第 273 页至第 274 页
`GoldenGate`，参见第 191 页
不兼容记录错误，参见第 238 页
并行
通过表过滤实现，参见第 135 页至第 141 页
概述，参见第 134 页至第 135 页
使用键范围，参见第 141 页至第 146 页
按 `Replicat` 报告，参见第 86 页至第 89 页



### 复制

### 高级
*   2–3

### 基础
*   2

### 双向
*   42–43, 106–110
    *   排除的事务
    *   冲突解决处理

### 广播
*   44

### 数据，实时
*   182

### 数据过滤
### 列
*   95–96
### 行
*   96
### 表
*   94–95

### 数据映射，列
*   97–98

### 数据泵
*   65–70
    *   添加
    *   配置
    *   启动与停止
    *   验证

### DBMS 配置选项
*   99–106
    *   针对 ASM
    *   DDL 复制
    *   针对 RAC

### 增强配置
*   85–92
    *   自动进程启动与重启
    *   检查点表
    *   清理旧的跟踪文件
    *   报告

### Extract 进程
*   54–65
    *   配置本地 Extract
    *   禁用触发器和级联删除约束
    *   Extract 组
    *   补充日志记录
    *   验证 Manager 状态

### 和 GoldenGate
*   4

### 异构的。*参见* 异构复制

### 集成
*   44–45

### 加载
*   70–75
    *   使用 DBMS 实用工具
        *   初始加载

### 单向
*   40–42, 200–203

### 概述
*   49–54
    *   基本步骤
    *   单向复制拓扑
    *   设置前提条件
    *   要求

### Replicat 进程
*   76–83
    *   添加
    *   配置
    *   启动与停止
    *   验证

### 安全
*   92–94

### 服务器
*   22

### Streams 复制
*   3–4

### 拓扑，单向
*   52

### 用于零停机迁移
*   263–278
    *   切换
    *   回退
    *   前提条件
    *   要求
    *   设置
    *   拓扑

### 无报告的进程故障诊断
*   221

### 报告链接
*   184

### REPORT 参数
*   88–89

### REPORTCOUNT 参数
*   88

### 报告
### 丢弃的记录
*   89
### 由 Extract 和 Replicat 进程生成
*   86–89
    *   `REPORT`参数
    *   `REPORTCOUNT`参数
    *   `REPORTROLLOVER`参数

### REPORTRATE MIN 命令
*   130

### REPORTROLLOVER 参数
*   89

### ReportViewer 角色名称，Tomcat Web 服务器管理工具屏幕
*   187

### 存储库，Veridata
*   173

### 要求
### 用于灾难恢复复制
*   242–243
### 和规划
*   279–281
    *   确定拓扑
    *   了解业务目标
    *   理解需求

### RESETREPORTSTATS 选项
*   89

### Reverse 实用工具
*   46

### RHREMD1.defs 文件
*   98

### 右修剪 (RTRIM)，在字符字段上
*   181

### RMTHOST 参数
*   230–231
    *   为数据泵配置
    *   调优

### RMTTASK 参数
*   71

### RMTTRAIL 参数
*   67, 230

### 基于角色的安全
*   174, 186–188

### 行哈希值
*   173

### 行，过滤
*   96

### RTRIM (右修剪)，在字符字段上
*   181

### 运行作业按钮，Veridata 主屏幕
*   179

### 运行选项，更改 Extract 和 Replicat 对象的选项
*   206

### 运行/执行作业选项卡，Veridata 主屏幕
*   180

### 正在运行的作业面板
*   176, 180

### 运行中状态
*   68



### ![images](img/square.jpg) S 部分

SCOTT 用户模式, 113

SCOTT.DEPT 表, 113

SCOTT.EMP 表, 113

脚本, 数据, 176–177

### **安全性**

加密, 92–94

密码, 92

跟踪文件, 93–94

基于角色的, 186–188

安全性要求, 50, 281

安全性选项卡, 119

选择性比较, 174

`SEND`命令, 79, 295

`SEND EXTRACT`命令, 235

序列号(Seqno), 155, 157–158, 161

序列号(Sequence#), 157–158, 160–161

服务器和数据库组件，Director, 190

服务器组件，Veridata, 172

服务器进程统计信息, 130

### **服务器**

数据库，Oracle GoldenGate 支持的版本, 16

复制, 22

`server.xml`文件, 186

服务级别协议(SLA), 180

`SET EDITOR`命令, 59, 77

`SETENV`参数，配置

用于初始加载 Replicat 文件, 73

用于本地 Extract, 61

用于 Replicat 进程, 78

### **安装设置**, 281–283

检查点表, 282

创建专用安装目录, 282

创建专用用户, 281

数据泵, 283

制定命名规范, 283

密码加密, 282

验证字符集, 283

shell 命令, 221

SLA(服务级别协议), 180

软件要求

用于 Oracle GoldenGate Director 11g, 25–26

用于 Oracle GoldenGate Veridata, 29–30

软件版本不正确, 227

数据库端排序选项, 180

服务器端排序选项, 180

源端和目标端配置, 50, 280

源数据库, 34–35, 235

源端跟踪文件, 34, 36

`SOURCEDB`参数, 55, 61, 73, 79, 236

`SOURCEDEFS`参数, 98

`SOURCEISTABLE`选项, 71

`SOURCEISTABLE`任务, 294

源到目标的 DDL 活动, 239

源到目标的 DML 活动, 239

`SPECIALRUN`参数, 74

SQL DML 变更，来自 PAYROLL 模式, 267

SQL 执行参数编辑器, 211

SQL 权限, 276

Microsoft SQL Server 2000, 112

Microsoft SQL Server 2005, 112

Microsoft SQL Server 2008, 112

Microsoft SQL Server 数据库, 118, 121


### SQL 语句和命令

SQL 语句，236

`SQL> ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;` 命令，55

`SQL> SELECT SUPPLEMENTAL_LOG_DATA_MIN FROM V$DATABASE;` 命令，54

`SQLEXEC` 语句，108

### 参数和实用程序

`STATOPTIONS` 参数，89

`srvctl` 实用程序，101

### 进程状态和命令

停滞的 `Replicat` 进程，236

`START` 命令，228

`START EXTRACT <extract name>` 命令，219

`STATS` 命令，69，83，248，268–269，271

`stats extract tablename` 命令，141，146

`stats replicat tablename` 命令，141，146

`STATS RHREMD1` 命令，130

`STATUS` 命令，295

`STATUS EXTRACT` 命令，226

`Status of ABENDED`，269，271

`Status of RUNNING`，269，271

`Status of STOPPED`，269，271

`STOP EXTRACT` 命令，68

`Stop Job` 按钮，`Veridata Main` 屏幕，179

`STOP` 状态，164

### 灾难恢复复制

备用数据泵，灾难恢复复制，249–250

备用 `Extract`，灾难恢复复制，248–249

备用 `Replicat`，灾难恢复复制，250–251

统计信息，临时，286

### 存储

磁盘，调优，149–150

最快可用，288

流复制，3–4

### 补充日志记录

数据库级别

启用，54–55

验证，54

表级别，启用，55–57

`SUPPLEMENTAL_LOG_DATA`，54

`SUPPLEMENTAL_LOG_DATA_MIN`，54

`SUPPRESSTRIGGERS` 选项，57

### 数据库系统

`Sybase DBMS` (数据库管理系统)

到 Oracle 的复制，123

在 Windows 和 UNIX 上，Oracle GoldenGate 的安装，23

数据同步，238–239

语法错误，在参数文件中，230

### 系统要求

适用于 Oracle GoldenGate

数据库服务器版本，16

磁盘空间，17

内存，16

适用于 Microsoft Windows 集群环境，18

网络，17

操作系统，18

适用于 Windows 和 UNIX 上的 Teradata，22

适用于 Oracle GoldenGate Director 11g，25–26

客户端安装要求，26

软件要求，25–26

支持的平台，25

Web 客户端安装指南，26

适用于 Oracle GoldenGate Veridata，29–31

数据库权限，30

磁盘要求，29

内存要求，29

软件要求，29–30

Web 要求，30–31

适用于 Oracle GoldenGate Veridata 代理，28


### T

![images](img/square.jpg)

表 EMPLOYEES，52

表过滤，用于实现并行的 Extract 和 Replicat 过程，135–141

`TABLE 参数`
- 为数据泵配置，67
- 为本地 Extract 配置，62

`TABLE` 语句，61，96，239

`TableExclude` 参数，268

表键约束，239

启用表级补充日志，55–57

表，176–177
- 检查点，282
- 比较大表的增量数据，181–182
- 过滤，94–95
- 缺失键约束，239

`tail` 命令，223

`tar` 命令，20

目标数据库，39–40

目标定义文件，236

`TargetDB`，73，78

TCP/IP 错误，231

TCP/IP 网络连接，38

TCP/IP 网络延迟，221

TCP/IP 套接字缓冲区大小，38

TDE（透明数据加密），281

Teradata 12.0，123

Teradata 13.0，123

Teradata RDBMS（关系数据库管理系统）
- Oracle GoldenGate 的安装，21–22
- 向 Oracle 复制，123

性能测试，287

`THREADOPTIONS IOLATENCY`，100

`THREADOPTIONS MAXCOMMITPROPAGATIONDELAY`，100

`THREADOPTIONS` 参数，100

线程
- 定义，101–102
- 增加数量，180

最大阈值，155

`Time Since Chkpt`，156

`TLTRACE` 参数，用于进程跟踪，225

`tnsnames.ora` 文件，100，102

`TNSNAMES.ORA` 文件，更新，102–103

`tnsnames.ora` 网络配置，220

`TNSPING` 命令，229

Tomcat Web 服务器管理工具屏幕，186–187

工具与实用程序，45–47
- `DEFGEN`，46
- `Director`，46–47
- `GGSCI`，46
- `Logdump`，46
- `Reverse`，46
- `Veridata`，46

`top` 命令，130

`top -U gger`，130

拓扑结构，40–45
- 确定，281
- 灾难恢复复制的拓扑，243
- 单向复制，52

`TOTALSONLY *` 命令，130

跟踪命令，225–226
- 使用 `TLTRACE` 参数进行进程跟踪，225
- `TRACE` 参数，226
- 故障排除案例研究，226

`TRACE` 参数，226

`TRACE2` 参数，226

`TRACETABLE` 参数，109

跟踪文件
- 更改大小，206



加密, 93–94

生成速率, 160–162

缺失, 228

监控大小, 206

使用 `Logdump` 工具打开, 299–300

问题, 221–223
未清空, 221–222
未滚动, 222
清除问题, 222–223
清除旧项, 89–90, 287
序列号, 224

跟踪文件
远程, 38–39
源, 36

`TRANLOG` 参数, 63

`tranlogoptions`, 提取, 206–207

`TranLogOptions ASMUser` 参数, 102

事务, 为双向复制排除, 108–109

传输, 数据的, 232–234

转换列, 98–99

`TRANSMEMORY` 参数, 209

透明数据加密 (`TDE`), 281

基于触发器的, 2

触发器, 禁用, 57–58

故障排查, 217–240

常见问题与解决方案, 217–218

配置问题, 226–234
数据库可用性, 227–228
软件版本不正确, 227
进程组缺失, 228
跟踪文件缺失, 228
网络问题, 231–232
操作系统问题, 230–231
参数文件问题, 229–230

数据库问题
字符集配置问题, 239
数据泵错误, 236
数据同步问题, 238–239
`Extract` 进程, 235
获取失败, 239–240
缺少列错误, 239
`Replicat` 进程, 236–238
表缺少键约束, 239

废弃文件, 223–225
缺失, 224
未打开, 225
过大, 224

错误日志分析, 223

`Logdump` 工具, 296–300
访问, 296
获取语法帮助, 296–299
`HISTORY` 命令, 299
使用其打开跟踪文件, 299–300

进程失败, 218–221
`Extract` 进程, 219–221
无报告诊断, 221

跟踪命令, 225–226
使用 `TLTRACE` 参数进行进程跟踪, 225
`TRACE` 参数, 226

故障排查案例研究, 226

跟踪文件问题, 221–223
未清空, 221–222
未滚动, 222
清除问题, 222

`比较值时截断尾随空格` 选项, 181

性能调优, 127–152
数据库, 289–290
设计并实施解决方案, 134–152
`BATCHSQL` 参数, 146–149
数据库, 152
磁盘存储, 149–150
`GROUPTRANSOPS` 参数, 149
网络, 150–151
并行 `Extract` 和 `Replicat` 进程, 134–135
`RMTHOST` 参数, 151

确定问题, 132–133

方法论, 127–128

性能
创建基线, 129–131
定义需求, 128–129
评估当前状况, 131–132

配置文件设置, 180

两阶段确认, 174

通用数据库。*参见* `IBM DB2 UDB`

`UNIX` 操作系统, Oracle GoldenGate 的安装

`IBM DB2 UDB`, 23–24

`Sybase` DBMS, 23

`Teradata` RDBMS 平台, 21–22

非计划故障切换, 其灾难恢复复制, 256

用例, 40–45

`USECHECKPOINTS` 选项, 90, 223

`用户定义` 选项, `列映射` 屏幕, 180

`USERID` 参数, 配置
用于初始加载 `Replicat` 文件, 73
用于本地 `Extract`, 61–62
用于 `Replicat` 进程, 79

用户
专用用户, 创建, 281
指定, 102

实用程序, 工具与, 45–47
`DEFGEN`, 46
`Director`, 46–47
`GGSCI`, 46
`Logdump`, 46
`Reverse`, 46
`Veridata`, 46

`UTL_FILE`, 103

```
-v 命令, GGSCI
```
227



### V

- `V$TRANSACTION` 动态性能视图，235
- `var/log/messages`，162
- `var/log/messages` 目录，213
- `vericom` 命令行作业，178
- `Vericom` 接口，173，185–186
- `vericon.exe -job job_big_table` 命令，179
- `Veridata` 工具，46，171–188
    - 优点，174
    - 比较
        - `Compare Pairs`，177–178
        - 数据库连接，175–176
        - 不同列类型和比较格式，182
        - 组，177
        - 作业，178–179
        - 配置文件，179
        - 实时复制数据，182
        - 表和数据脚本，176–177
    - 组件，171–173
        - `Vericom` 接口，173
        - `Veridata Agent`、`Java agent` 和 `C-Code agent`，173
        - `Veridata Repository`，173
        - `Veridata Server`，172
        - `Veridata Web` 工具，173
    - 主屏幕，179
    - 性能
        - 比较方法，180
        - 比较大表的增量数据，181–182
        - 禁用 `Confirm Out of Sync` 步骤，180
        - 排除列，180
        - 增加线程数，180
        - 字符字段上的 `RTRIM`，181
        - 统计信息，183–185
        - 调优配置文件设置，180
    - 基于角色的安全性，186–188
        - `Vericom` 接口命令行，185–186
- `<Veridata_install_dir>/server/web/conf` 目录，186
- 版本，软件，227
- `VIEW GGSEVT` 命令，223，226
- `VIEW PARAMS <group>` 命令，231
- `VIEW REPORT <group>` 命令，235
- `VIEW REPORT` 命令，219
- `-w` 参数，186

### W

- `WARNLOGTRANS` 参数，235
- `Web` 客户端安装指南，适用于 `Oracle GoldenGate Director 11g`，26
- `Web` 组件，`Director`，191
- `Web` 要求，`Oracle GoldenGate Veridata`，30–31
- `Web` 工具，`Veridata`，173
- `WebLogic` 域，190
- `WebLogic` 端口，191
- `WHERE` 子句，96，181
- `WIN11SRC` 环境，13–14
- `WIN11TRG` 环境，13–14
- `Windows`，`Microsoft`。参见 `Microsoft Windows`
- `Windows` 事件日志，213
- `Windows` 事件查看器，116
- `Windows Explorer` 图形界面，229
- `Windows` 服务，115
- `-wp` 参数，186
- `Write Checkpoint` 命令，160

### X, Y, Z

- 零停机数据库升级，41
- 零停机迁移，用于复制的，263–278
    - 切换，274–276
    - 回退，276–278
    - 先决条件，263–264
    - 要求，264–265
    - 设置，266–274
    - 拓扑，265–266
