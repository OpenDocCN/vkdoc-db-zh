# SQL 监控与调优

如果您没有使用 `AWR`、`ADSM` 和 `ASH` 报告的许可，免费的 `Statspack` 工具可以帮助您识别性能不佳的 SQL 语句。以 `SYS` 用户身份运行以下脚本来安装 `Statspack`：

```
SQL> @?/rdbms/admin/spcreate.sql
```

此脚本创建拥有 `Statspack` 仓库的 `PERFSTAT` 用户。要启用自动收集 `Statspack` 统计信息，请运行此脚本：

```
SQL> @?/rdbms/admin/spauto.sql
```

收集到一些快照后，您可以以 `PERFSTAT` 用户身份运行以下脚本来创建 `Statspack` 报告：

```
SQL> @?/rdbms/admin/spreport.sql
```

报告创建后，查找标有“SQL ordered by CPU”的部分。以下是一些示例输出：

```
SQL ordered by CPU DB/Inst: DW11/DW11 Snaps: 11-14
-> Total DB CPU (s): 107
-> Captured SQL accounts for 246.0% of Total DB CPU
-> SQL reported below exceeded 1.0% of Total DB CPU

CPU CPU per Elapsd Old Time (s) Executions Exec (s) %Total Time (s) Buffer Gets Hash Value
---------- ----------- ---------- ------ ---------- --------------- ----------
254.95 4 63.74 238.1 249.74 12,811 2873951798
Module: SQL*Plus
select count(*) from dba_indexes, dba_tables
```

**提示** 查看 `ORACLE_HOME/rdbms/admin/spdoc.txt` 文件获取 `Statspack` 文档。

## 工作原理

Oracle 维护着大量的动态性能视图，用于跟踪和累积数据库性能指标。例如，如果您运行以下查询，您会注意到对于 Oracle Database 11g，有超过 400 个动态性能视图：

```
select count(*) from dictionary where table_name like 'V$%';
```

Oracle 性能工具依赖于从这些内部性能视图定期获取的快照。关于性能统计最有用的两个视图是 `V$SYSSTAT` 和 `V$SESSTAT` 视图。`V$SYSSTAT` 视图包含超过 400 种数据库统计信息。`V$SYSSTAT` 视图包含整个数据库的信息，而 `V$SESSTAT` 视图包含各个会话的统计信息。

`V$SYSSTAT` 和 `V$SESSTAT` 视图中的少数几个值包含资源的当前使用量。这些值是：
- `opened cursors current`
- `logons current`
- `session cursor cache current`
- `workarea memory allocated`

其余的值是累积的。`V$SYSSTAT` 中的值是自实例启动以来整个数据库的累积值。`V$SESSTAT` 中的值是自会话启动以来每个会话的累积值。一些更重要的与性能相关的累积值是：
- `CPU used`
- `consistent gets`
- `physical reads`
- `physical writes`

对于累积统计，衡量周期性使用情况的方法是在起始点记录一个统计值，然后在稍后时间点记录该值并捕获差值。这是 Oracle 性能工具（如 `AWR` 和 `Statspack`）所采用的方法。Oracle 会定期获取动态等待接口视图的快照，并将它们存储在仓库中。

您可以在 `AWR` 视图 `DBA_HIST_SQLSTAT` 中访问关于消耗资源最多的 SQL 语句的统计信息。还有另一个相关的视图 `DBA_HIST_SNAPSHOT`，它显示每个 `AWR` 快照的记录。类似地，`Statspack` 的历史性能统计数据存储在 `STATS$SQL_SUMMARY` 表中，而 `STATS$SNAPSHOT` 表包含每个 `Statspack` 快照的记录。

##### 19-6. 使用操作系统识别资源密集型查询

### 问题

您在一台服务器上运行着多个 Oracle、MySQL 和 PostgreSQL 数据库。服务器的性能似乎很迟缓。您想确定哪个操作系统会话消耗的资源最多，并判断该会话是否与某个数据库相关，并最终定位到具体的 SQL 查询。

### 解决方案

首先，使用操作系统实用程序来识别消耗资源最多的进程。此示例使用 Linux/Unix 的 `ps` 实用程序来识别消耗 CPU 最多的操作系统会话：

```
$ ps -e -o pcpu,pid,user,tty,args | sort -n -k 1 -r | head
```

以下是一些示例输出：

```
7.8 14028 oracle ? oracleDW11 (DESCRIPTION=(LOCAL=YES)(ADDRESS=(PROTOCOL=beq)))
0.1 17012 oracle ? ora_j003_SCDEV
0.1 17010 oracle ? ora_j002_SCDEV
```

从输出中看，操作系统会话 `14028` 消耗了最多的 CPU 资源，为 7.8%。在此示例中，`14028` 进程与 `DW11` 数据库相关。接下来，登录到相应的数据库，并使用以下 SQL 语句来确定与操作系统进程 `14028` 关联的是什么类型的程序：

```
select
'USERNAME : ' || s.username|| chr(10) ||
'OSUSER : ' || s.osuser || chr(10) ||
'PROGRAM : ' || s.program || chr(10) ||
'SPID : ' || p.spid || chr(10) ||
'SID : ' || s.sid || chr(10) ||
'SERIAL# : ' || s.serial# || chr(10) ||
'MACHINE : ' || s.machine || chr(10) ||
'TERMINAL : ' || s.terminal
from v$session s,
v$process p
where s.paddr = p.addr
and p.spid = '&PID_FROM_OS';
```

这是本示例的输出：

```
'USERNAME:'||S.USERNAME||CHR(10)||'OSUSER:'||S.OSUSER||CHR(10)||'PROGRAM:'
USERNAME : INV_MGMT
OSUSER : oracle
PROGRAM : sqlplus@rmougprd.rmoug.org (TNS V1-V3)
SPID : 14028
SID : 198
SERIAL# : 57
MACHINE : rmougprd.rmoug.org
TERMINAL :
```

在此输出中，`PROGRAM` 值为 `sqlplus@rmougprd.rmoug.org`。这表明一个 `SQL*Plus` 会话是在 `rmougprd.rmoug.org` 服务器上消耗过多资源的程序。

接下来运行以下查询，以显示与操作系统进程 ID（本示例中 `SPID` 为 `14028`）关联的 SQL 语句：

```
select
'USERNAME : ' || s.username || chr(10) ||
'OSUSER : ' || s.osuser || chr(10) ||
'PROGRAM : ' || s.program || chr(10) ||
'SPID : ' || p.spid || chr(10) ||
'SID : ' || s.sid || chr(10) ||
'SERIAL# : ' || s.serial# || chr(10) ||
'MACHINE : ' || s.machine || chr(10) ||
'TERMINAL : ' || s.terminal || chr(10) ||
'SQL TEXT : ' || q.sql_text
from v$session s
,v$process p
,v$sql q
where s.paddr = p.addr
and p.spid = '&PID_FROM_OS'
and s.sql_id = q.sql_id;
```

结果显示消耗资源的 SQL 语句，作为 `SQL TEXT` 列的一部分输出：

```
'USERNAME:'||S.USERNAME||CHR(10)||'OSUSER:'||S.OSUSER||CHR(10)||'PROGRAM:'
USERNAME : INV_MGMT
OSUSER : oracle
PROGRAM : sqlplus@rmougprd.rmoug.org (TNS V1-V3)
SPID : 14028
SID : 198
SERIAL# : 57
MACHINE : rmougprd.rmoug.org
TERMINAL :
SQL TEXT : select count(*) ,object_name from dba_objects,dba_segments 470
```

### 工作原理

当您在一台服务器上运行多个数据库并遇到服务器性能问题时，有时很难精确定位是哪个数据库及其关联进程导致了问题。在这种情况下，您必须使用操作系统工具来识别系统上消耗资源最多的会话。

在 Linux 或 Unix 环境中，您可以使用 `ps`、`top` 或 `vmstat` 等实用程序来识别消耗资源最多的操作系统进程。`ps` 实用程序很方便，因为它可以让您识别消耗最多 CPU 或内存的进程。在解决方案部分，我们使用了 `ps` 命令来识别 CPU 密集型查询。这里我们用它来识别 Oracle 内存使用最多的进程：

```
$ ps -e -o pmem,pid,user,tty,args | grep -i oracle | sort -n -k 1 -r | head
```

一旦您识别出一个与数据库相关的、消耗资源最多的进程，您就可以基于服务器进程 ID 查询数据字典视图，以识别数据库进程正在执行什么。

`OS Watcher`


