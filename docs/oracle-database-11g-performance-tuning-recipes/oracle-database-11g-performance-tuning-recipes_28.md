# 发现 (1)

-----------------------------

1.  SQL 执行计划的结构已发生变化。

## 变更前执行计划

-----------------------------

| ID | 操作 | 名称 | 行数 | 字节数 | 代价 | 时间 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 0 | SELECT STATEMENT | | 16777222 | 671088880 | 16793338 | |
| 1 | NESTED LOOPS | | 16777222 | 671088880 | 16793338 | |
| 2 | PARTITION RANGE ALL | | 16777222 | 335544440 | 16116 | |
| 3 | TABLE ACCESS FULL | EMPPART | 16777222 | 335544440 | 16116 | |
| *5 | INDEX UNIQUE SCAN | PK_DEPT | 1 | | | |

## 变更后执行计划

-----------------------------

| ID | 操作 | 名称 | 行数 | 字节数 | 代价 | 时间 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 0 | SELECT STATEMENT | | 16777222 | 352321662 | 16812553 | 6:02:31 |
| 1 | NESTED LOOPS | | | | | |
| 2 | NESTED LOOPS | | 16777222 | 352321662 | 16812553 | 56:02:31 |
| 3 | PARTITION RANGE ALL | | 16777222 | 150994998 | 29013 | 00:05:49 |
| 4 | TABLE ACCESS FULL | EMPPART | 16777222 | 150994998 | 29013 | 00:05:49 |
| *5 | INDEX UNIQUE SCAN | PK_DEPT | 1 | | 0 | 00:00:01 |
| 6 | TABLE ACCESS BY INDEX ROWID | DEPT | 1 | 12 | 1 | 00:00:01 |

在上述示例输出中，我们可以看到变更前后的执行统计信息，以及变更前后的执行计划。我们还能看到估计的工作负载影响百分比和 SQL 影响百分比，这对于快速浏览所做的系统变更是否有重大影响非常有用。在此示例中，我们看到将会有 1%的变化，并且通过查看执行计划，我们发现差异可以忽略不计。因此，对于上述查询，数据库升级基本上将对我们查询的性能产生最小或没有影响。如果您看到影响百分比达到 10%或更高，可能意味着需要在将变更应用到生产环境之前进行更多分析和调整，以主动调整查询或系统。为了获得准确的比较结果，还建议在执行使用 `DBMS_SQLPA` 的分析之前，导出生产统计信息并将其导入到您的测试环境中。

![images](img/square.jpg) **提示** 在 SQL Plus 中，记得设置 `SET LONG` 和 `SET LONG CHUNKSIZE`，以便正确显示输出内容。


#### 工作原理

SQL 性能分析器与`DBMS_SQLPA`包可用于分析 SQL 工作负载，该工作负载可定义为以下任何一种：

*   一条 SQL 语句
*   缓存中存储的 SQL ID
*   SQL 调优集（有关 SQL 调优集的信息，请参见第 11 章）
*   基于自动工作负载 repository (AWR) 快照的 SQL ID（更多信息，请参见第 4 章）

通常，收集一系列 SQL 语句的信息比收集单个语句的信息更容易。通过自动工作负载 repository (AWR) 快照或 SQL 调优集来获取信息是为一系列语句收集信息的最简单方法。AWR 快照将包含基于特定时间段的信息，而 SQL 调优集将包含针对特定目标 SQL 语句集的信息。考虑进行“前后”性能分析的一些可能的关键原因包括：

*   初始化参数更改
*   数据库升级
*   硬件更改
*   操作系统更改
*   应用程序模式对象添加或更改
*   SQL 基线或配置文件的实施

有大量信息可供比较。在报告分析中收集的信息时，仅显示因系统更改而受到不利影响的 SQL 语句的输出可能是有益的。例如，您可能希望将`REPORT_ANALYSIS_TASK`函数显示的信息范围缩小，仅显示以下类型的 SQL 语句信息：

*   那些性能退化的语句
*   那些执行计划发生更改的语句
*   那些 SQL 语句中出现错误的语句

在收集每个任务的信息之前，刷新`shared_pool`和/或`buffer_cache`可能是有益的，这将有助于为比较获得最佳信息。分析任务的信息存储在数据字典中。您可以引用任何以“`DBA_ADVISOR`”为前缀的数据字典视图，以获取有关您创建的性能分析任务、执行的操作以及执行统计信息、执行计划和报告信息的信息。有关`DBMS_SQLPA`包的完整说明，请参阅适用于您数据库版本的 Oracle PL/SQL Packages and Types Reference，该包在 Oracle 11g 版本中是新增的。

![images](img/square.jpg) **注意** 执行使用`DBMS_SQLPA`的分析任务需要`ADVISOR`系统权限。

## 第 10 章

## 追踪 SQL 执行

追踪会话活动是大多数 SQL 性能调优工作的核心。Oracle 提供了一套丰富的工具来追踪 SQL 活动。本章介绍 Oracle SQL 追踪功能，并向您展示如何在您的环境中设置 SQL 追踪。Oracle 提供了许多“事件”来帮助您执行各种类型的追踪。

尽管有几种追踪方法可用，但 Oracle 现在建议您对大多数类型的追踪使用`DBMS_MONITOR`包。本章包含几个说明如何使用此包生成追踪的配方。此外，我们还展示了如何通过设置各种 Oracle 事件来追踪会话，这通常是 Oracle 支持人员所要求的。您将学习如何追踪单个 SQL 语句、一个会话以及整个实例，以及如何追踪并行查询。有配方展示了如何追踪另一个用户的会话以及如何使用触发器来启动会话追踪。您还将学习如何追踪 Oracle 优化器的执行路径。

Oracle 提供了`TKPROF`实用程序以及可免费下载的分析器 Oracle Trace Analyzer。本章展示了如何使用这两种分析器来分析您生成的原始追踪文件。

### 10-1. 准备环境

#### 问题

您希望确保数据库已为追踪 SQL 会话正确设置。

#### 解决方案

在开始追踪 SQL 语句之前，您必须做三件事：

1.  启用定时统计信息收集。
2.  指定追踪转储文件的目标位置。
3.  调整追踪转储文件大小。

您可以通过将`timed_statistics`参数设置为`true`来启用定时统计信息的收集。首先检查此参数的当前值：

```sql
SQL> sho parameter statistics
NAME                                 TYPE        VALUE
------------------------------------ ----------- -----------
...
statistics_level                     string      TYPICAL
timed_statistics                     boolean     TRUE
SQL>
```

如果`timed_statistics`参数的值为`false`，请使用以下语句将其设置为`true`：

```sql
SQL> alter system set timed_statistics=true scope=both;
System altered.
SQL>
```

您也可以使用以下语句在会话级别设置此参数：

```sql
SQL> alter session set timed_statistics=true
```

您可以使用以下命令找到追踪目录的位置（在 Oracle Database 11g 之前的版本中称为用户转储目录）：

```sql
SQL> select name,value from v$diag_info
  2* where name='Diag Trace'
SQL> /
NAME                                      VALUE
-----------------------------    ----------------------------------------
Diag Trace                       c:\app\ora\diag\rdbms\orcl1\orcl1\trace
SQL>
```

在 Oracle Database 11g 中，`max_dump_file_size`参数的默认值为`unlimited`，您可以通过发出以下命令进行验证：

```sql
SQL> sho parameter dump
NAME                                 TYPE        VALUE
------------------------------------ ----------- ----------
...
max_dump_file_size                   string      unlimited
```

无限制的转储文件大小意味着文件可以增长到操作系统允许的大小。


