# 数据泵高级特性

## 压缩特性

使用命令 `dmp directory=dp_dir full=y \ compression=all compression_algorithm=MEDIUM`。

`COMPRESSION_ALGORITHM` 参数在磁盘空间不足或通过网络连接导出时特别有用，因为它减少了需要传输的字节数。

![Image](img/sq.jpg)

**注意**
`COMPRESSION_ALGORITHM` 参数需要 Oracle Advanced Compression 选项的许可。

### 导入时更改表压缩特性

从 Oracle 12c 开始，您可以在导入表时更改其压缩特性。此示例将作业中导入的所有表的压缩特性更改为 `COMPRESS FOR OLTP`。由于此示例中的命令需要引号，因此将其放在参数文件中，如下所示：

```
userid=mv_maint/foo dumpfile=inv.dmp directory=dp_dir transform=table_compression_clause:"COMPRESS FOR OLTP"
```

假设参数文件名为 `imp.par`。现在可以按如下方式调用：

```
$ impdp parfile=imp.par
```

导入作业中包含的所有表都将创建为 `COMPRESS FOR OLTP`，并且数据在加载时被压缩。

![Image](img/sq.jpg)

**注意**
表级压缩（用于 OLTP）需要 Oracle Advanced Compression 选项的许可。

## 数据加密

数据泵转储文件的一个潜在安全问题是，任何对输出文件有操作系统访问权限的人都可以在文件中搜索字符串。在 Linux/Unix 系统上，您可以使用 `strings` 命令执行此操作：

```
$ strings inv.dmp | grep -i secret
```

这是此特定转储文件的输出：

```
Secret Data<
top secret data<
corporate secret data<
```

此命令允许您查看转储文件的内容，因为数据是常规文本且未加密。如果您需要保护数据安全，可以使用数据泵的加密功能。

此示例使用 `ENCRYPTION` 参数来保护输出中的所有数据和元数据：

```
$ expdp mv_maint/foo encryption=all directory=dp_dir dumpfile=inv.dmp
```

要使此命令生效，您的数据库必须有一个已打开的加密钱包。有关如何创建和打开钱包的更多详细信息，请参阅《Oracle Advanced Security Administrator’s Guide》，可从 Oracle 网站 (`http://otn.oracle.com`) 的 Technology Network 区域下载。

![Image](img/sq.jpg)

**注意**
数据泵 `ENCRYPTION` 参数要求您使用 Oracle 11g 或更高版本的企业版，并且还需要 Oracle Advanced Security 选项的许可。

`ENCRYPTION` 参数接受以下选项：

*   `ALL`
*   `DATA_ONLY`
*   `ENCRYPTED_COLUMNS_ONLY`
*   `METADATA_ONLY`
*   `NONE`

`ALL` 选项为数据和元数据启用加密。`DATA_ONLY` 选项仅加密数据。`ENCRYPTED_COLUMNS_ONLY` 选项指定仅在数据库中加密的列才以加密格式写入转储文件。`METADATA_ONLY` 选项仅加密导出文件中的元数据。

## 将视图导出为表

从 Oracle 12c 开始，您可以导出视图并在之后将其作为表导入。如果您需要将视图中包含的数据复制到历史报告数据库，则可能需要这样做。

使用 `VIEWS_AS_TABLES` 参数将视图导出到表结构中。此参数的语法如下：

```
VIEWS_AS_TABLES=[schema_name.]view_name[:template_table_name]
```

以下是一个示例：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=v.dmp \
views_as_tables=sales_rockies
```

现在，转储文件可用于将名为 `SALES_ROCKIES` 的表导入不同的方案或数据库。

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=v.dmp
```

如果您只想导入在导出期间从视图创建的表，可以按如下方式进行：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=v.dmp tables=sales_rockies
```

该表将具有与视图定义相同的列和数据类型。该表还将包含与导出时从视图中选择的内容相匹配的数据行。

## 导入时禁用重做日志记录

从 Oracle 12c 开始，您可以指定对象以 nologging 方式加载重做。这是通过 `DISABLE_ARCHIVE_LOGGING` 参数实现的：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp \
transform=disable_archive_logging:Y
```

在执行导入期间，对象的日志记录属性被设置为 `NO`；导入后，日志记录属性被设置回其原始值。对于数据泵可以使用直接路径执行的操作（例如向表中插入数据），这可以减少导入期间生成的重做量。

### 交互式命令模式

数据泵提供了一个交互式命令模式，允许您监控数据泵作业的状态并动态修改许多作业特征。交互式命令模式对于长时间运行的数据泵操作最为有用。在此模式下，您还可以停止、重新启动或终止当前正在运行的作业。以下部分将讨论这些活动。

### 进入交互式命令模式

有两种方式可以访问交互式命令模式提示符：

*   在通过 `expdp` 或 `impdp` 启动的数据泵作业中按 Ctrl+C。
*   使用 `ATTACH` 参数附加到当前正在运行的作业。

当您从命令行运行数据泵作业时，您将处于命令行模式。随着作业的进行，您应该会看到输出显示在您的终端上。如果要退出命令行模式，请按 Ctrl+C。这将使您进入交互式命令界面模式。对于导出作业，提示符为：

```
Export>
```

输入 `HELP` 命令以查看可用的导出交互式命令（参见 表 8-1）：

```
Export> help
```

**表 8-1**. 导出交互式命令

| 命令 | 描述 |
| --- | --- |
| `ADD_FILE` | 向导出转储集添加文件 |
| `CONTINUE_CLIENT` | 继续交互式客户端模式 |
| `EXIT_CLIENT` | 退出客户端会话并返回到操作系统提示符；让当前作业继续运行 |
| `FILESIZE` | 为任何后续创建的转储文件定义文件大小 |
| `HELP` | 显示交互式导出命令 |
| `KILL_JOB` | 终止当前作业 |
| `PARALLEL` | 增加或减少并行度 |
| `REUSE_DUMPFILES` | 如果转储文件存在则覆盖它（默认为 `N`） |
| `START_JOB` | 重新启动已附加的作业 |
| `STATUS` | 显示当前附加作业的状态 |
| `STOP_JOB [=IMMEDIATE]` | 停止作业处理（您可以稍后重新启动它）。使用 `IMMEDIATE` 参数可快速停止作业，但可能有一些未完成的任务。 |

输入 `EXIT` 以离开交互式命令模式：

```
Export> exit
```

您现在应该处于操作系统提示符下。

您可以在导出或导入作业中按 Ctrl+C。对于导入作业，交互式命令模式提示符为：

```
Import>
```

要查看所有可用命令，请输入 `HELP`：

```
Import> help
```

交互式命令模式的导入命令总结在 表 8-2 中。

**表 8-2**. 导入交互式命令

| 命令 | 描述 |
| --- | --- |
| `CONTINUE_CLIENT` | 继续交互式日志记录模式 |
| `EXIT_CLIENT` | 退出客户端会话并返回到操作系统提示符。让当前作业继续运行 |
| `HELP` | 显示可用的交互式命令 |
| `KILL_JOB` | 终止客户端当前连接到的作业 |
| `PARALLEL` | 增加或减少并行度 |
| `START_JOB` | 重新启动先前停止的作业 |


`START_JOB=SKIP_CURRENT` 将重启作业并跳过作业停止时正处于活动状态的任何操作。
`STATUS` 用于指定监控作业状态的频率。默认模式为 0；在此模式下，客户端会在作业状态变化可用时立即报告。
`STOP_JOB [=IMMEDIATE]` 用于停止作业处理（稍后可重启）。使用 `IMMEDIATE` 参数可快速停止作业，但可能会有未完成的任务。

键入 `EXIT` 以离开 Data Pump 状态工具：
```
Import> exit
```
此时您应处于操作系统提示符下。

## 连接到正在运行的作业

Data Pump 的一个强大功能是，您可以连接到当前正在运行的作业并查看其进度和状态。如果您拥有 DBA 权限，即使您不是作业所有者，也可以连接。您可以通过 `ATTACH` 参数连接到导入或导出作业。

在连接到作业之前，您必须首先确定 Data Pump 作业名称（以及所有者名称，如果您不是作业所有者）。运行以下 SQL 查询以显示当前正在运行的作业：
```
SQL> select owner_name, operation, job_name, state from dba_datapump_jobs;
```
以下是一些示例输出：
```
OWNER_NAME      OPERATION       JOB_NAME               STATE
--------------- --------------- -------------------- --------------------
MV_MAINT        EXPORT          SYS_EXPORT_SCHEMA_01  EXECUTING
```
在此示例中，`MV_MAINT` 用户可以直接连接到该导出作业，如下所示：
```
$ expdp mv_maint/foo attach=sys_export_schema_01
```
如果您不是作业所有者，则通过指定所有者名称和作业名称来连接到作业：
```
$ expdp system/foobar attach=mv_maint.sys_export_schema_01
```
现在您应该会看到 Data Pump 命令行提示符：
```
Export>
```
键入 `STATUS` 以查看当前连接作业的状态：
```
Export> status
```

## 停止和重启作业

如果您有一个当前正在运行的 Data Pump 作业希望暂时停止，可以先连接到交互式命令模式来实现。您可能希望停止作业以解决空间问题或性能问题，待问题解决后再重启作业。以下示例连接到一个导入作业：
```
$ impdp mv_maint/foo attach=sys_import_table_01
```
现在，使用 `STOP_JOB` 参数停止作业：
```
Import> stop_job
```
您应该会看到以下输出：
```
Are you sure you wish to stop this job ([yes]/no):
```
键入 `YES` 以继续停止作业。您还可以指定立即停止作业：
```
Import> stop_job=immediate
```
当您使用 `IMMEDIATE` 选项停止作业时，与该作业关联的一些任务可能未完成。要重启作业，请连接到交互式命令模式，并发出 `START_JOB` 命令：
```
Import> start_job
```
如果您希望继续将作业输出记录到您的终端，请发出 `CONTINUE_CLIENT` 命令：
```
Import> continue_client
```

## 终止 Data Pump 作业

您可以指示 Data Pump 永久终止一个导出或导入作业。首先，以交互式命令模式连接到作业，然后发出 `KILL_JOB` 命令：
```
Import> kill_job
```
系统将提示您以下输出：
```
Are you sure you wish to stop this job ([yes]/no):
```
键入 `YES` 以永久终止该作业。Data Pump 会立即终止作业，并从运行导出或导入的用户中删除相关的状态表。

## 监控 Data Pump 作业

当您有长时间运行的 Data Pump 作业时，应不时检查作业状态，以确保其未失败、未被挂起等等。

有几种方法可以监控 Data Pump 作业的状态：
* 屏幕输出
* Data Pump 日志文件
* 查询数据字典视图
* 数据库告警日志
* 查询状态表
* 交互式命令模式状态
* 使用进程状态 (`ps`) 操作系统工具

最直观的监控方法是查看 Data Pump 在作业运行时显示在屏幕上的状态。如果您已从命令模式断开连接，则状态将不再显示在屏幕上。在这种情况下，您必须使用另一种技术来监控 Data Pump 作业。

### Data Pump 日志文件

默认情况下，Data Pump 会为每个作业生成一个日志文件。启动 Data Pump 作业时，最好为该作业指定一个特定的日志文件名：
```
$ impdp mv_maint/foo directory=dp_dir dumpfile=archive.dmp logfile=archive.log
```
此作业会创建一个名为 `archive.log` 的文件，该文件放置在数据库对象 `DP` 引用的目录中。如果您没有显式命名日志文件，Data Pump 导入会创建一个名为 `import.log` 的文件，而 Data Pump 导出会创建一个名为 `export.log` 的文件。

![Image](img/sq.jpg) `Note`  该日志文件包含的信息与您运行 Data Pump 作业时在屏幕上以交互方式看到的信息相同。

### 数据字典视图

快速判断 Data Pump 作业是否正在运行的一种方法是检查 `DBA_DATAPUMP_JOBS` 视图，查看是否有状态为 `EXECUTING` 的作业在运行：
```
select job_name, operation, job_mode, state from dba_datapump_jobs;
```
以下是一些示例输出：
```
JOB_NAME                  OPERATION            JOB_MODE   STATE
------------------------- -------------------- ---------- ---------------
SYS_IMPORT_TABLE_04       IMPORT               TABLE      EXECUTING
SYS_IMPORT_FULL_02        IMPORT               FULL       NOT RUNNING
```
您还可以通过以下查询 `DBA_DATAPUMP_SESSIONS` 视图来获取会话信息：
```
select sid, serial#, username, process, program from v$session s,
     dba_datapump_sessions d where s.saddr = d.saddr;
```
以下是一些示例输出，显示正在使用多个 Data Pump 会话：
```
      SID    SERIAL#  USERNAME           PROCESS         PROGRAM
---------- ---------- -------------------- --------------- ----------------------
      1049       6451 STAGING            11306           oracle@xengdb (DM00)
      1058      33126 STAGING            11338           oracle@xengdb (DW01)
      1048      50508 STAGING            11396           oracle@xengdb (DW02)
```

### 数据库告警日志

如果某个作业花费的时间比您预期的长得多，请检查数据库告警日志中是否有类似以下的消息：
```
statement in resumable session 'SYS_IMPORT_SCHEMA_02.1' was suspended due to
ORA-01652: unable to extend temp segment by 64 in tablespace REG_TBSP_3
```
此消息表明一个 Data Pump 导入作业被暂停，正在等待向 `REG_TBSP_3` 表空间添加空间。向表空间添加空间后，Data Pump 作业会自动恢复处理。默认情况下，Data Pump 作业会等待 2 小时以添加空间。

![Image](img/sq.jpg) `Note`  除了写入告警日志外，对于每个 Data Pump 作业，Oracle 会在 `ADR_HOME/trace` 目录中创建一个跟踪文件。该文件包含会话 ID 和作业开始时间等信息。跟踪文件的命名格式如下：`<SID>_dm00_<process_ID>.trc`。

### 状态表

每次启动 Data Pump 作业时，都会在运行作业的用户帐户中自动创建一个状态表。对于导出作业，表名取决于您运行的导出作业类型。表的命名格式为 `SYS_<OPERATION>_<JOB_MODE>_NN`，其中 `OPERATION` 是 `EXPORT` 或 `IMPORT`。`JOB_MODE` 可以是 `FULL`、`SCHEMA`、`TABLE`、`TABLESPACE` 等。



以下是查询状态表以获取当前正在运行的作业详情的示例：

```sql
select name, object_name, total_bytes/1024/1024 t_m_bytes ,job_mode ,state ,to_char(last_update, 'dd-mon-yy hh24:mi') from SYS_EXPORT_TABLE_01 where state='EXECUTING';
```

## 交互式命令模式状态

验证数据泵是否正在运行作业的快速方法是以交互式命令模式连接并发出 `STATUS` 命令；例如，

```bash
$ impdp mv_maint/foo attach=SYS_IMPORT_TABLE_04
Import> status
```

以下是一些示例输出：

```
Job: SYS_IMPORT_TABLE_04
  Operation: IMPORT
  Mode: TABLE
  State: EXECUTING
  Bytes Processed: 0
  Current Parallelism: 4
```

你应该看到一个 `EXECUTING` 状态，这表示作业正在主动运行。输出中需要检查的其他项目包括处理的对象和字节数。这些数字应随着作业的进行而增加。

## 操作系统实用程序

你可以使用操作系统实用程序 `ps` 来显示服务器上正在运行的作业。例如，你可以搜索主进程和工作进程，如下所示：

```bash
$ ps -ef | egrep 'ora_dm|ora_dw' | grep -v egrep
```

以下是一些示例输出：

```
oracle 29871   717   5 08:26:39 ?          11:42 ora_dw01_STAGE
oracle 29848   717   0 08:26:33 ?           0:08 ora_dm00_STAGE
oracle 29979   717   0 08:27:09 ?           0:04 ora_dw02_STAGE
```

如果你多次运行此命令，你应该会看到一个或多个当前作业的处理时间（第七列）在增加。这是数据泵仍在执行并工作的良好指标。

### 数据泵传统模式

本章最后介绍此功能，但它非常实用，尤其如果你是一位老派的 DBA。自 Oracle 11g Release 2 起，数据泵允许你在调用数据泵作业时使用旧的 `exp` 和 `imp` 实用程序参数。这被称为传统模式，是一项很棒的功能。

要使用传统模式的数据泵，你无需做任何特殊操作。一旦数据泵检测到传统参数，它会尝试将其作为来自旧 `exp`/`imp` 实用程序的参数进行处理。你甚至可以将旧的传统参数与新参数混合使用；例如，

```bash
$ expdp mv_maint/foo consistent=y tables=inv directory=dp_dir
```

在输出中，数据泵会指出它遇到了传统参数，并为你提供该参数在数据泵语法中的翻译结果。对于前面的命令，以下是数据泵会话的输出，显示了 `consistent=y` 参数被翻译成的内容：

```
Legacy Mode Parameter: "consistent=TRUE"
Location: Command Line, Replaced with: "flashback_time=TO_TIMESTAMP('2014-01-25 19:31:54', 'YYYY-MM-DD HH24:MI:SS')"
```

此功能可能极其方便，特别是当你非常熟悉旧的传统语法并想知道它如何在数据泵中实现时。

我建议你尽可能使用较新的数据泵语法。然而，你可能会遇到这样的情况：你有传统的 `exp`/`imp` 作业，并希望保持脚本原样运行，无需修改。

![Image](img/sq.jpg) **注意** 当数据泵在传统模式下运行时，它不会创建旧的 `exp`-/`imp` 格式的文件。数据泵总是创建数据泵文件，并且只能读取数据泵文件。

## 数据泵与 exp 实用程序的映射

如果你习惯旧的 `exp`/`imp` 参数，最初可能会对某些语法含义感到困惑。然而，在使用数据泵之后，你会发现新语法相当容易记忆和使用。表 8-3 描述了传统导出参数如何映射到数据泵导出。

表 8-3. 旧导出参数到数据泵的映射

| 原始 exp 参数 | 类似的数据泵 expdp 参数 |
| --- | --- |
| `BUFFER` | 不适用 |
| `COMPRESS` | `TRANSFORM` |
| `CONSISTENT` | `FLASHBACK_SCN` 或 `FLASHBACK_TIME` |
| `CONSTRAINTS` | `EXCLUDE=CONSTRAINTS` |
| `DIRECT` | 不适用；数据泵尽可能自动使用直接路径。 |
| `FEEDBACK` | 客户端输出中的 `STATUS` |
| `FILE` | 数据库目录对象和 `DUMPFILE` |
| `GRANTS` | `EXCLUDE=GRANT` |
| `INDEXES` | `INCLUDE=INDEXES`, `INCLUDE=INDEXES` |
| `LOG` | 数据库目录对象和 `LOGFILE` |
| `OBJECT_CONSISTENT` | 不适用 |
| `OWNER` | `SCHEMAS` |
| `RECORDLENGTH` | 不适用 |
| `RESUMABLE` | 不适用；数据泵自动提供此功能。 |
| `RESUMABLE_NAME` | 不适用 |
| `RESUMABLE_TIMEOUT` | 不适用 |
| `ROWS` | `CONTENT=ALL` |
| `STATISTICS` | 不适用；数据泵导出总是导出表的统计信息。 |
| `TABLESPACES` | `TRANSPORT_TABLESPACES` |
| `TRANSPORT_TABLESPACE` | `TRANSPORT_TABLESPACES` |
| `TRIGGERS` | `EXCLUDE=TRIGGER` |
| `TTS_FULL_CHECK` | `TRANSPORT_FULL_CHECK` |
| `VOLSIZE` | 不适用；数据泵不支持磁带设备。 |

在许多情况下，并非一一对应。通常，数据泵会自动提供旧版实用程序中需要参数才能使用的功能。例如，过去必须指定 `DIRECT=Y` 才能获得直接路径导出，而数据泵会在可能时自动使用直接路径。

## 数据泵与 imp 实用程序的映射

与数据泵导出类似，数据泵导入通常也与旧版实用程序参数没有一一对应的映射。数据泵导入自动提供了旧 `imp` 实用程序的许多功能。例如，不需要 `COMMIT=Y`，因为数据泵导入在每个表导入后会自动提交。表 8-4 描述了传统导入参数如何映射到数据泵导入。

表 8-4. 旧导入参数到数据泵的映射

| 原始 imp 参数 | 类似的数据泵 impdp 参数 |
| --- | --- |
| `BUFFER` | 不适用 |
| `CHARSET` | 不适用 |
| `COMMIT` | 不适用；数据泵导入在每个表导出后会自动提交。 |
| `COMPILE` | 不适用；数据泵导入在过程创建后编译它们。 |
| `CONSTRAINTS` | `EXCLUDE=CONSTRAINT` |
| `DATAFILES` | `TRANSPORT_DATAFILES` |
| `DESTROY` | `REUSE_DATAFILES=y` |
| `FEEDBACK` | 客户端输出中的 `STATUS` |
| `FILE` | 数据库目录对象和 `DUMPFILE` |
| `FILESIZE` | 不适用 |
| `FROMUSER` | `REMAP_SCHEMA` |
| `GRANTS` | `EXCLUDE=OBJECT_GRANT` |
| `IGNORE` | `TABLE_EXISTS_ACTION`，可选值为 `APPEND`、`REPLACE`、`SKIP` 或 `TRUNCATE` |
| `INDEXES` | `EXCLUDE=INDEXES` |
| `INDEXFILE` | `SQLFILE` |
| `LOG` | 数据库目录对象和 `LOGFILE` |
| `RECORDLENGTH` | 不适用 |
| `RESUMABLE` | 不适用；此功能自动提供。 |
| `RESUMABLE_NAME` | 不适用 |
| `RESUMABLE_TIMEOUT` | 不适用 |
| `ROWS=N` | `CONTENT`，可选值为 `METADATA_ONLY` 或 `ALL` |
| `SHOW` | `SQLFILE` |
| `STATISTICS` | 不适用 |
| `STREAMS_CONFIGURATION` | 不适用 |
| `STREAMS_INSTANTIATION` | 不适用 |
| `TABLESPACES` | `TRANSPORT_TABLESPACES` |
| `TOID_NOVALIDATE` | 不适用 |
| `TOUSER` | `REMAP_SCHEMA` |
| `TRANSPORT_TABLESPACE` | `TRANSPORT_TABLESPACES` |
| `TTS_OWNERS` | 不适用 |
| `VOLSIZE` | 不适用；数据泵不支持磁带设备。 |

### 总结

数据泵是一个功能极其强大且丰富的工具。如果你使用数据泵不多，那么我建议你花些时间重读本章并实践其中的示例。此工具极大地简化了将用户和数据从一个环境迁移到另一个环境等任务。你可以导出和导入用户的子集，通过 SQL 和 PL/SQL 过滤和重映射数据，重命名用户和表空间，压缩、加密和并行化，所有这些都只需一个命令。它确实如此强大。



数据库管理员有时会坚持使用旧的`exp`/`imp`实用程序，因为那是他们所熟悉的（我偶尔也会犯这样的错误）。如果你运行的是 Oracle 11g 第 2 版，你可以直接在命令行使用旧的`exp`/`imp`参数和选项。Data Pump 会将这些参数实时转换为 Data Pump 特有的语法。这个特性很好地促进了从旧工具到新工具的迁移。作为参考，我还提供了旧`exp`/`imp`语法与 Data Pump 命令之间的映射关系。

## 索引

- ![images](img/sq.jpg)  A
    - 活动的联机重做日志组
    - 归档日志模式
    - 架构决策
    - 备份
    - 数据库
    - 损坏的控制文件
    - 恢复数据文件
    - 描述
    - 不完全恢复
    - 在加载模式下
    - 介质恢复
    - 联机
    - RECOVER 语句
    - 恢复与还原
    - 还原控制文件
    - 重做文件位置
    - 用户定义的磁盘位置
    - 归档重做日志
    - 备份
    - 目标与文件格式
    - 保留策略
- ![images](img/sq.jpg)  B
    - 备份控制文件
    - 备份, RMAN
        - 归档重做日志
        - 自动备份, 数据库文件
        - `CATALOG` 命令
        - 控制文件
        - 数据库级别
        - 数据文件, 并行度
        - FRA
        - 离线/不可访问的文件
        - 可插拔数据库
        - RMAN 完全备份
        - 根容器
        - spfile
        - 表空间
        - `vs.` 映像副本
    - 块变化跟踪
    - 块级损坏
- ![images](img/sq.jpg)  C
    - 冷备份策略
        - 归档日志模式数据库
        - 非归档日志模式数据库
        - 优势
        - 业务需求
        - 创建, 文件副本
        - 数据文件位置
        - 位置与名称, 数据库文件
        - 联机重做日志
        - `OPEN RESETLOGS` 子句
        - 还原
        - 编写脚本
        - 关闭, 数据库
    - 控制文件管理
        - 自动备份
        - 后台进程
        - 数据文件
        - 信息类型
        - [init.



