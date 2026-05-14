# 将操作系统进程映射到 SQL 语句

在识别操作系统进程时，查看哪些进程消耗最多的 CPU 是很有用的。如果资源消耗大户是数据库进程，将其映射到数据库作业或查询也很有用。要确定消耗最多 CPU 资源的进程 ID，可以使用类似 `ps` 的命令：

```bash
$ ps -e -o pcpu,pid,user,tty,args | sort -n -k 1 -r | head
```

这是一段输出示例：

```
14.6 24875 oracle   ?      oracleo18c (DESCRIPTION=(LOCAL=YES)(ADDRESS=...
0.8 21613 oracle   ?      ora_vktm_o18c
0.1 21679 oracle   ?      ora_mmon_o18c
```

从输出中可以看到，操作系统会话 24875 是 CPU 资源的最大消费者。输出还表明该进程与 `o12c` 数据库相关联。掌握这些信息后，登录到相应的数据库，并使用以下 SQL 语句来确定与操作系统进程 24875 关联的程序类型：

```sql
SQL> select
'USERNAME : ' || s.username|| chr(10) ||
'OSUSER   : ' || s.osuser  || chr(10) ||
'PROGRAM  : ' || s.program || chr(10) ||
'SPID     : ' || p.spid    || chr(10) ||
'SID      : ' || s.sid     || chr(10) ||
'SERIAL#  : ' || s.serial# || chr(10) ||
'MACHINE  : ' || s.machine || chr(10) ||
'TERMINAL : ' || s.terminal
from v$session s,
v$process p
where s.paddr = p.addr
and   p.spid  = &PID_FROM_OS;
```

运行该命令时，SQL*Plus 将提示您输入用于替代 `&PID_FROM_OS` 的值。在此示例中，您将输入 `24875`。以下是相关输出：

```
USERNAME : MV_MAINT
OSUSER   : oracle
PROGRAM  : sqlplus@speed2 (TNS V1-V3)
SPID     : 24875
SID      : 111
SERIAL#  : 899
MACHINE  : speed2
TERMINAL : pts/4
```

在此输出中，`PROGRAM` 值表明一个 SQL*Plus 会话是在服务器上消耗过多资源的程序。接下来，运行以下查询以显示与操作系统 PID 关联的 SQL 语句（在此示例中，服务器进程标识符 [SPID] 是 24875）：

```sql
SQL> select
'USERNAME : ' || s.username || chr(10) ||
'OSUSER   : ' || s.osuser   || chr(10) ||
'PROGRAM  : ' || s.program  || chr(10) ||
'SPID     : ' || p.spid     || chr(10) ||
'SID      : ' || s.sid      || chr(10) ||
'SERIAL#  : ' || s.serial#  || chr(10) ||
'MACHINE  : ' || s.machine  || chr(10) ||
'TERMINAL : ' || s.terminal || chr(10) ||
'SQL TEXT : ' || sa.sql_text
from v$process p,
v$session s,
v$sqlarea sa
where p.addr = s.paddr
and s.username is not null
and s.sql_address = sa.address(+)
and s.sql_hash_value = sa.hash_value(+)
and p.spid= &PID_FROM_OS;
```

结果在 `SQL TEXT` 列中显示了消耗资源的 SQL 语句。以下是输出的一段摘录：

```
USERNAME : MV_MAINT
OSUSER   : oracle
PROGRAM  : sqlplus@speed2 (TNS V1-V3)
SPID     : 24875
SID      : 111
SERIAL#  : 899
MACHINE  : speed2
TERMINAL : pts/4
SQL TEXT : select a.table_name from dba_tables a,dba_indexes b,dba_objects c...
```

当在一台服务器上运行多个数据库并且遇到服务器性能问题时，有时很难确定是哪个数据库及相关进程导致了问题。在这些情况下，您必须使用操作系统工具来识别系统上资源消耗最高的会话。

在 Linux/Unix 环境中，您可以使用 `ps`、`top` 或 `vmstat` 等工具来识别消耗最高的操作系统进程。`ps` 工具非常方便，因为它可以让您识别消耗最多 CPU 或内存的进程。前面的 `ps` 命令识别了消耗 CPU 最多的进程。这里，它被用来识别消耗 Oracle 内存最多的进程：

```bash
$ ps -e -o pmem,pid,user,tty,args | grep -i oracle | sort -n -k 1 -r | head
```

一旦确定了与数据库关联的消耗最高的进程，您就可以基于 SPID 查询数据字典视图，以识别数据库进程正在执行什么。

## OS Watcher

Oracle 提供了一系列用于收集和存储 CPU、内存、磁盘和网络使用情况指标的脚本。在 Linux/Unix 系统上，OS Watcher 工具套件自动化了统计信息的收集，使用的工具包括 `top`、`vmstat`、`iostat`、`mpstat`、`netstat` 和 `traceroute`。该工具还有一个可选的图形组件，用于直观地显示性能指标。

您可以从 Oracle 的 MOS 网站获取 OS Watcher。对于 Linux/Unix 版本，请参阅 MOS note 301137.1 或标题为“OS Watcher User Guide”的文档。有关 Windows 版本的 OS Watcher 的详细信息，请参阅 MOS note 433472.1。

## 查找资源密集型 SQL 语句

隔离性能不佳的查询的最佳方法之一是让用户或开发人员抱怨某个特定的 SQL 语句。在这种情况下，不需要进行任何侦探工作。您可以直接定位需要调整的 SQL 查询。

然而，在调查性能问题时，您并不经常有幸让人类明确告知从何处查看。有多种方法可用于确定数据库中哪些 SQL 语句消耗了最多的资源：

*   实时执行统计信息
*   近实时统计信息
*   Oracle 性能报告

这些技术将在接下来的几个部分中描述。

### 监控实时 SQL 执行统计信息

您可以使用以下查询从 `V$SQL_MONITOR` 中进行选择，以监控 SQL 查询的近实时资源消耗：

```sql
SQL> select * from (
select a.sid session_id, a.sql_id
,a.status
,a.cpu_time/1000000 cpu_sec
,a.buffer_gets, a.disk_reads
,b.sql_text sql_text
from v$sql_monitor a
,v$sql b
where a.sql_id = b.sql_id
order by a.cpu_time desc)
where rownum <=20;
```

该查询的输出不容易完整地放在一页上。以下是输出的子集：

```
SESSION_ID  SQL_ID         STATUS      CPU_SEC  BUFFER_GETS  DISK_READS
----------  -------------  ---------  --------  -----------  -----------   SQL_TEXT---------
139  d07nngmx93rq7  DONE       331.88    5708         3490 select  count(*)
130  9dtu8zn9yy4uc  EXECUTING  11.55     5710         248 select  task_name
```

在查询中，首先使用内联视图检索所有记录并按 `CPU_TIME` 降序组织它们。然后，外部查询使用 `ROWNUM` 伪列将结果集限制为前 20 行。

您可以修改前面的查询，以便按您选择的统计信息对结果进行排序，或者仅显示当前正在执行的查询。例如，下一个 SQL 语句监控当前正在执行的查询，并按磁盘读取次数排序：

```sql
SQL> select * from (
select a.sid session_id, a.sql_id, a.status
,a.cpu_time/1000000 cpu_sec
,a.buffer_gets, a.disk_reads
,substr(b.sql_text,1,35) sql_text
from v$sql_monitor a
,v$sql b
where a.sql_id = b.sql_id
and   a.status='EXECUTING'
order by a.disk_reads desc)
where rownum <=20;
```



