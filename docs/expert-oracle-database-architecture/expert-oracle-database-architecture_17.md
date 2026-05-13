# 4. 内存结构

在本章中，我们将探讨 Oracle 的三大内存结构：

*   系统全局区 (SGA)：这是一个大型的共享内存段，几乎所有的 Oracle 进程在某个时刻都会访问它。
*   进程（或程序）全局区 (PGA)：这是单个进程或线程私有的内存；其他进程/线程无法访问。
*   用户全局区 (UGA)：这是与你的会话相关联的内存。根据你是使用共享服务器（它将在 SGA 中）还是专用服务器（它将在 PGA 中）连接到数据库，它位于 SGA 或 PGA 中。

当我们讨论 Oracle 中的内存管理时，有五种模式需要研究：

*   自动内存管理 (AMM)：对于 SGA 和 PGA，DBA 只需设置一个参数——`MEMORY_TARGET` 参数——让数据库决定如何确定所有内存区域的大小。
*   自动共享内存管理 (ASMM)：对于 SGA，DBA 为 SGA 设置一个目标大小（通过 `SGA_TARGET`）。
*   手动共享内存管理：对于 SGA，DBA 手动调整 SGA 各个内存区域的大小（通过 `DB_CACHE_SIZE`、`SHARED_POOL_SIZE` 等）。
*   自动 PGA 内存管理：对于 PGA，DBA 为 PGA 设置一个目标大小（通过 `PGA_AGGREGATE_TARGET`）。
*   手动 PGA 内存管理：对于 PGA，DBA 手动调整 PGA 各个内存区域的大小（通过 `SORT_AREA_SIZE`、`HASH_AREA_SIZE` 等）。Oracle 强烈建议不要使用此方法，但我们将进行讨论，以便为其他内存管理概念奠定基础。

本章旨在帮助你精通 Oracle 内存的各个方面，以便你就如何启用内存管理做出明智的决策。我们将首先讨论 PGA 和 UGA 内存管理（先是手动，然后是自动）来探讨所有方法。然后我们将转向 SGA，同样先看手动再看自动内存管理方法。最后，我们将探讨如何使用单个参数来管理内存，以同时控制 SGA 和 PGA 区域。

## 进程全局区和用户全局区

PGA 是特定于进程的一块内存。换句话说，它是特定于单个操作系统进程或线程的内存。系统中的任何其他进程或线程都无法访问此内存。它通常通过 C 运行时调用 `malloc()` 或 `memmap()` 分配，并且可以在运行时增长（甚至缩小）。PGA 从不在 Oracle 的 SGA 中分配；它总是由进程或线程在本地分配——PGA 中的 P 代表进程或程序；它是不共享的。

UGA 实际上是你的会话状态。这是你的会话必须始终能够访问的内存。UGA 的位置取决于你如何连接到 Oracle。如果你通过共享服务器连接，UGA 必须存储在每个共享服务器进程都能访问的内存结构中——那就是 SGA。这样，你的会话可以使用任何一个共享服务器，因为它们中的任何一个都可以读写你会话的数据。另一方面，如果你使用的是专用服务器连接，则无需普遍访问你的会话状态，UGA 实际上就等同于 PGA；事实上，它将包含在你的专用服务器的 PGA 中。当你查看系统统计信息时，你会发现 UGA 在专用服务器模式下报告在 PGA 中（PGA 将大于或等于所使用的 UGA 内存；PGA 内存大小将包括 UGA 大小）。因此，PGA 包含进程内存，并且可能包含 UGA。PGA 内存的其他区域通常用于内存中的排序、位图合并和哈希运算。可以安全地说，除了 UGA 内存之外，这些是迄今为止 PGA 的最大贡献者。

管理 PGA 中的内存有两种方式：手动和自动。不应使用手动方法（除非你使用的是旧版本的 Oracle 且别无选择）。自动 PGA 内存管理是推荐的技术，你应该使用它。自动方法在管理内存方面更简单、更高效。在这两种情况下，内存的分配和使用方式有很大不同，因此我们将依次讨论每种方法。


### 手动 PGA 内存管理

我能想到的适合手动 PGA 内存管理的场景只有少数几种。一种是你运行的是旧版 Oracle 且无法升级。另一种情况可能是你运行大型批处理作业，且这些作业在实例中是唯一活动时，手动 PGA 管理可能会带来性能优势。因此，你可能会发现自己处于使用此类 PGA 内存管理的环境中。除了这些场景，应避免使用手动管理方式。

在手动 PGA 内存管理中，除了会话为 PL/SQL 表和其他变量分配的内存外，以下是对你的 PGA 大小影响最大的参数：

*   `SORT_AREA_SIZE`：在信息交换到磁盘（使用分配给用户的临时表空间中的磁盘空间）之前，用于排序信息的 RAM 总量。
*   `SORT_AREA_RETAINED_SIZE`：排序完成后，用于保存已排序数据的内存量。也就是说，如果 `SORT_AREA_SIZE` 为 512KB，而 `SORT_AREA_RETAINED_SIZE` 为 256KB，你的服务器进程在查询的初始处理阶段会使用最多 512KB 的内存来排序数据。排序完成后，排序区会“缩小”到 256KB，任何无法容纳在这 256KB 内的已排序数据将被写入临时表空间。
*   `HASH_AREA_SIZE`：你的服务器进程可用于在内存中存储哈希表的内存量。这些结构在哈希联接期间使用，通常是在联接一个大型集合与另一个集合时。两个集合中较小的一个会被哈希到内存中，任何无法容纳在哈希区域内的数据将由联接键存储在临时表空间中。

> 注意：在使用上表中的参数之前，你必须将 `WORKAREA_SIZE_POLICY` 参数设置为 `MANUAL`。

这些参数控制了在使用磁盘上的临时表空间之前，Oracle 将在内存中用于排序或哈希数据的空间量，以及排序完成后将保留多少该内存段。计算出的 `SORT_AREA_SIZE` 值（`SORT_AREA_SIZE` 减去 `SORT_AREA_RETAINED_SIZE`）通常从你的 PGA 中分配，而 `SORT_AREA_RETAINED_SIZE` 值将在你的 UGA 中。

以下是关于使用 `*_AREA_SIZE` 参数需要记住的重要事项：

*   这些参数控制单个 `SORT`、`HASH` 或 `BITMAP MERGE` 操作使用的最大内存量。
*   单个查询可能有许多使用此内存的操作在进行，并且可能创建多个排序/哈希区。请记住，你可能同时打开多个游标，每个游标都有自己的 `SORT_AREA_RETAINED` 需求。因此，如果你将排序区大小设置为 10MB，你的会话中可能会使用 10、100、1000 MB 或更多的 RAM。这些设置不是会话限制；相反，它们是对单个操作的限制，而你的会话在一个查询中可能有多个排序操作，或者打开多个需要排序的查询。
*   这些区域的内存是按需分配的。如果你像我们那样将排序区大小设置为 1GB，并不意味着你会分配 1GB 的 RAM。这只意味着你授予了 Oracle 进程为排序/哈希操作分配那么多内存的权限。

既然我们已经简要回顾了手动 PGA 内存管理，接下来让我们看看你应该使用的——自动 PGA 内存管理。

### 自动 PGA 内存管理

在几乎所有情况下，你都应该使用自动 PGA 内存管理。自动 PGA 内存管理的整个目标是在不使用超过你期望的更多 RAM 的同时，最大化 RAM 的使用。你可以通过两种方式启用 PGA 的自动管理：

*   将 `MEMORY_TARGET` 设置为零，然后将 `PGA_AGGREGATE_TARGET` 设置为非零值。`PGA_AGGREGATE_TARGET` 参数控制实例应为所有用于排序或哈希数据的工作区分配的总内存。其默认值因版本而异，可能由 DBCA 等各种工具设置。在此模式下，`WORKAREA_SIZE_POLICY` 被设置为 `AUTO`（这是其默认值）。
*   通过将 `MEMORY_TARGET` 设置为非零值来使用 AMM 功能，并将 `PGA_AGGREGATE_TARGET` 保持为零。这实际上让 Oracle 管理分配给 PGA 的内存。但是，如果你在使用 HugePages 的 Linux 环境中，则不应使用 AMM 方法来管理内存（更多内容见本章“系统全局区（SGA）内存管理”一节）。

以下小节将讨论前面提到的这两种技术。

#### 设置 PGA_AGGREGATE_TARGET

在我过去几年工作过的大多数数据库中，都使用了自动 PGA 内存管理和自动 SGA 内存管理。对于我的测试数据库，自动 PGA 内存管理和自动 SGA 内存管理启用如下（你应该根据工作负载和可用的物理内存量，为你自己的环境使用合适的内存大小）：

```sql
$ sqlplus / as sysdba
SQL> alter system set memory_target=0 scope=spfile;
SQL> alter system set pga_aggregate_target=300M scope=spfile;
SQL> alter system set sga_target=1500M scope=spfile;
```

然后重新启动实例以使参数生效（此处使用 startup force，它会以 abort 方式关闭并重启实例）：

```sql
SQL> startup force;
```

你不必同时启用 PGA 和 SGA 内存管理（如前面的示例所示）。你可以启用一个进行自动管理，而将另一个保留为手动管理。我通常不会那样实现，但你可以这样做。

此外，我工作过的一些地方也设置了 `PGA_AGGREGATE_LIMIT` 参数。在大多数情况下，你不需要设置此参数，因为它会默认为一个合理的值。如果出于某种原因你需要更多控制，那么可以随意设置它。请记住，如果你将此参数设置得过低，你会收到 `ORA-00093` 错误，并且你的实例将无法启动。在这种情况下，你需要创建一个基于文本的 `init.ora` 文件，重新启动你的实例，并重新创建你的 `spfile`（有关如何执行此操作的详细信息，请参见第 3 章）。

#### 设置 MEMORY_TARGET

PGA 的自动内存管理启用如下（根据你的环境调整内存大小）：

```sql
$ sqlplus / as sysdba
SQL> alter system set memory_target=1500M scope=spfile;
SQL> alter system set pga_aggregate_target=0 scope=spfile;
SQL> alter system set sga_target=0 scope=spfile;
```

此时，你可以重新启动实例以使参数生效。如果你想让 Oracle 为 `SGA_TARGET` 和 `PGA_AGGREGATE_TARGET` 推荐最小使用值，你可以将它们设置为非零值（只要它们的总和小于 `MEMORY_TARGET` 的值）：

```sql
SQL> alter system set sga_target=500M scope=spfile;
SQL> alter system set pga_aggregate_target=400M scope=spfile;
```

> 注意：对于可插拔数据库，此 `PGA_AGGREGATE_TARGET` 参数是可选的。当在可插拔数据库中设置此参数时，它指定了该可插拔数据库的目标聚合 PGA 大小。

既然我们已经介绍了如何启用自动 PGA 内存管理，接下来让我们看看 PGA 内存是如何分配的。



## 确定内存的分配方式

经常出现的问题是“这个内存是如何分配的？”以及“我的会话将使用多少 RAM？”。这些问题很难回答，原因很简单：自动内存分配方案的底层算法没有公开文档，并且可能随着版本更新而改变。当使用以“A”（代表自动）开头的设置时，你会失去一定程度的控制权，因为底层算法决定了如何做以及如何控制。

我们可以根据 Oracle Support notes 147806.1 和 223730.1 中的信息做出一些观察：

*   `PGA_AGGREGATE_TARGET` 是一个目标上限值。*它不是在数据库启动时预分配的值*。你可以通过将 `PGA_AGGREGATE_TARGET` 设置为远高于服务器物理内存量的值来观察这一点。你不会因此看到任何大量的内存分配（有一个注意事项：如果你设置了 `MEMORY_TARGET`，然后将 `PGA_AGGREGATE_TARGET` 设置为大于 `MEMORY_TARGET` 的值，实例启动时 Oracle 会抛出 `ORA-00838` 错误，并阻止你启动实例）。
*   对于给定会话可用的 PGA 内存量源自 `PGA_AGGREGATE_TARGET` 的设置。用于确定进程使用最大大小的算法因数据库版本而异。分配给进程的 PGA 内存量通常是可用内存量和竞争空间的进程数量的函数。
*   随着实例工作负载的增加（更多并发查询、并发用户），分配给工作区的 PGA 内存量将会减少。数据库会尝试将所有 PGA 分配的总和保持在 `PGA_AGGREGATE_TARGET` 设置的阈值以下。这类似于让 DBA 全天坐在控制台前，根据数据库中执行的工作量来设置 `SORT_AREA_SIZE` 和 `HASH_AREA_SIZE` 参数。我们很快将在测试中直接观察到这种行为。

那么，我们如何观察分配给我们的会话的不同工作区大小呢？通过运行一些测试脚本来观察会话使用的内存以及对临时表空间执行的 I/O 量。我在一个具有四个 CPU 和专用服务器连接的 Oracle Linux 机器上进行了以下测试。我们首先创建一个表来保存我们想要监控的指标（以下代码放在一个名为 `stats.sql` 的文件中）：

```
$ sqlplus eoda/foo@PDB1
SQL> create table sess_stats
as
select name, value, 0 active
from
(
select a.name, b.value
from v$statname a, v$sesstat b
where a.statistic# = b.statistic#
and b.sid = (select sid from v$mystat where rownum=1)
and (a.name like '%ga %'
or a.name like '%direct temp%')
union all
select 'total: ' || a.name, sum(b.value)
from v$statname a, v$sesstat b, v$session c
where a.statistic# = b.statistic#
and (a.name like '%ga %'
or a.name like '%direct temp%')
and b.sid = c.sid
and c.username is not null
group by 'total: ' || a.name
);
Table created.
```

该表中我们将用于指标的列表示：

*   `NAME`: 我们正在收集的统计信息的名称（来自 `V$SESSTAT` 的当前会话的 `PGA` 和 `UGA` 信息，以及数据库实例的所有内存信息以及临时表空间写入）。
*   `VALUE`: 给定指标的值。
*   `ACTIVE`: 实例中正在工作的其他会话数。在我们开始之前，我们假设一个“空闲”的实例；目前我们是唯一的用户会话，因此值为零。

```
接下来，创建表 T 如下：
SQL> create table t as select * from all_objects;
Table created.
SQL> exec dbms_stats.gather_table_stats( user, 'T' );
```

然后我在一个交互式会话中运行了以下 SQL*Plus 脚本（存储在名为 `single_load.sql` 的文件中）。

```
set echo on
declare
l_first_time boolean default true;
begin
for x in ( select * from t order by 1, 2, 3, 4 )
loop
if ( l_first_time )
then
insert into sess_stats
( name, value, active )
select name, value,
(select count(*)
from v$session
where status = 'ACTIVE'
and username is not null)
from
(
select a.name, b.value
from v$statname a, v$sesstat b
where a.statistic# = b.statistic#
and b.sid = (select sid from v$mystat where rownum=1)
and (a.name like '%ga %'
or a.name like '%direct temp%')
union all
select 'total: ' || a.name, sum(b.value)
from v$statname a, v$sesstat b, v$session c
where a.statistic# = b.statistic#
and (a.name like '%ga %'
or a.name like '%direct temp%')
and b.sid = c.sid
and c.username is not null
group by 'total: ' || a.name
);
l_first_time := false;
end if;
end loop;
end;
/
commit;
```

该脚本使用自动 PGA 内存管理对大表 `T` 进行排序。然后，对于该会话，它捕获所有 `PGA/UGA` 内存设置以及排序到磁盘的活动。此外，`UNION ALL` 添加了关于相同内容的系统级指标（总 PGA 内存、总 UGA 内存等）。我在使用以下初始化设置启动的数据库上运行了该脚本：

```
memory_target=0
pga_aggregate_target=300m
sga_target=1500m
```

这些设置显示我正在使用自动 `PGA` 内存管理，`PGA_AGGREGATE_TARGET` 为 300MB，意味着我希望 Oracle 使用最多约 300MB 的 `PGA` 内存进行排序。

我设置了另一个脚本，以便在其他会话中运行，以在机器上生成大型排序负载。该脚本循环并使用内置包 `DBMS_ALERT` 来查看是否应继续处理。如果应该，它将运行相同的大型查询，对整个 `T` 表进行排序。当模拟完成时，一个会话可以向所有排序进程（负载生成器）发出“停止”并退出的信号。以下是用于执行排序的脚本（存储在名为 `gen_load.sql` 的文件中）：

```
declare
l_msg   long;
l_status number;
begin
dbms_alert.register( 'WAITING' );
for i in 1 .. 999999 loop
dbms_application_info.set_client_info( i );
dbms_alert.waitone( 'WAITING', l_msg, l_status, 0 );
exit when l_status = 0;
for x in ( select * from t order by 1, 2, 3, 4 )
loop
null;
end loop;
end loop;
end;
/
exit
```

以下是用于停止这些进程运行的脚本（存储在名为 `stop.sql` 的文件中）：

```
begin
dbms_alert.signal( 'WAITING', '' );
commit;
end;
/
```

为了观察分配给我正在测量的会话的不同内存量，我最初在隔离状态下运行 `SELECT`——作为唯一的会话。我捕获了统计信息并将其保存到 `SESS_STATS` 表中，以及活动会话的数量。然后，我向系统添加了 25 个会话（即，我在 25 个新会话中运行了前面的基准测试脚本（`gen_load.sql`），其中包含 `for i in 1 .. 999999 loop`）。我等待了很短的时间——一分钟让系统适应这个新负载——然后我创建了一个新会话，并运行了之前的单个排序查询，在循环中第一次捕获了指标。我重复这样做，最多达到 300 个并发用户。

提示

在本书的 GitHub 源代码站点上，你可以下载此实验使用的脚本。在 `ch04` 目录中，`run.sql` 脚本自动化了本节描述的测试。

应该注意的是，我在这里要求数据库实例做了一件不可能的事情。在 300 个用户的情况下，仅仅让他们全部登录（更不用说实际做任何工作！），我们就非常接近 `PGA_AGGREGATE_TARGET` 设置了！这强调了 `PGA_AGGREGATE_TARGET` 正如其名：只是一个目标，而非指令。出于各种原因，我们可能并且将会超过这个值。

现在我们准备报告发现的结果；由于篇幅原因，我们将输出停止在 275 个用户——因为数据开始变得相当重复：



```
SQL> column active format 999
SQL> column pga format 999.9
SQL> column "tot PGA" format 999.9
SQL> column pga_diff format 999.99
SQL> column "temp write" format 9,999
SQL> column "tot writes temp" format 99,999,999
SQL> column writes_diff format 9,999,999
SQL> select active,
pga,
"tot PGA",
"tot PGA"-lag( "tot PGA" ) over (order by active) pga_diff,
"temp write",
"tot writes temp",
"tot writes temp"-lag( "tot writes temp" ) over (order by active) writes_diff
from (
select *
from (
select active,
name,
case when name like '%ga mem%' then round(value/1024/1024,1) else value end val
from sess_stats
where active < 275
)
pivot ( max(val) for name in  (
'session pga memory' as "PGA",
'total: session pga memory' as "tot PGA",
'physical writes direct temporary tablespace' as "temp write",
'total: physical writes direct temporary tablespace' as "tot writes temp"
) )
)
order by active
/
ACTIVE    PGA tot PGA PGA_DIFF temp write tot writes temp WRITES_DIFF
------ ------ ------- -------- ---------- --------------- -----------
0    3.5     7.6                   0               0
1   15.2    19.5    11.90          0               0           0
26   15.2   195.6   176.10          0         243,387     243,387
51    7.7   292.7    97.10      1,045         518,246     274,859
76    5.2   188.7  -104.00      3,066         941,324     423,078
101    5.2   232.6    43.90      6,323       1,834,035     892,711
126    5.2   291.8    59.20      6,351       3,021,485   1,187,450
151    5.1   345.0    53.20      6,326       4,783,879   1,762,394
177    5.0   403.3    58.30      6,321       8,603,295   3,819,416
201    5.2   453.2    49.90      6,327      12,848,568   4,245,273
226    4.8   507.5    54.30      6,333      15,225,399   2,376,831
251    5.1   562.2    54.70      6,315      17,579,502   2,354,103
12 rows selected.
```

## 查询分析

在分析结果之前，先看看我用于报告的查询。我的查询使用了一个名为`pivot`的功能来对结果集进行透视。以下是该 SQL 查询第 11 到 22 行（不使用`pivot`功能）的另一种写法：

```
select active,
max( decode(name,'session pga memory',val) ) pga,
max( decode(name,'total: session pga memory',val) ) as "tot PGA",
max( decode(name,
'physical writes direct temporary tablespace',
val) ) as "temp write",
max( decode(name,
'total: physical writes direct temporary tablespace',
val) ) as "tot writes temp"
from (
select active,
name,
case when name like '%ga mem%' then round(value/1024/1024,1) else value end val
from sess_stats
where active < 275
)
group by active
))
```

这部分查询从指标表中检索了活动会话数少于 275 条的记录，将内存（UGA/PGA 内存）指标从字节转换为兆字节，然后对四个关键指标进行了透视——将行转换为列。一旦我们将这四个指标整合到单条记录中，我们就使用了分析函数（特别是`LAG()`函数）为每一行添加了先前观察到的总 PGA 和总临时 I/O，以便于查看这些值的增量差异。

回到数据本身——正如你所看到的，当只有几个活动会话时，排序完全在内存中进行。对于 1 到略少于 50 的活动会话数，我可以完全在内存中排序。然而，当我有 50 个用户登录并积极进行排序时，数据库开始限制我一次可以使用的内存量。可能需要几分钟时间，PGA 的使用量才会回落到可接受的限制内（300MB 的请求），但在这些低并发用户级别下，最终还是会回落。我们观察的会话分配的 PGA 内存从 15.2MB 下降到 7.7MB，最终稳定在约 5.2MB 左右（请记住，那部分 PGA 中有些并非用于工作区（排序）分配，而是用于其他操作；仅仅是登录行为就创建了 0.5MB 的 PGA 分配）。系统使用的总 PGA 一直保持在可容忍的范围内，直到大约 126 个用户左右。此时，我开始经常性地超出`PGA_AGGREGATE_TARGET`，并一直持续到测试结束。我给这个数据库实例分配了一项不可能完成的任务；拥有 126 个用户（大多数在执行 PL/SQL）以及他们都在请求的排序操作，根本无法放入我设定的 300MB RAM 中。这根本无法实现。因此，每个会话都尽可能使用最少的内存，但又必须分配其所需的内存量。到我完成这个测试时，活动会话总共使用了大约 560MB 的 PGA 内存——这是它们能使用的最小量了。

自动 PGA 内存管理正是为了让一个小规模的用户群在资源可用时能尽可能多地使用 RAM 而设计的。在这种模式下，随着负载增加，它会随时间推移逐渐减少分配；随着负载降低，它会随时间推移增加分配给单个操作的 RAM 量。


## 使用 PGA_AGGREGATE_TARGET 控制内存分配

之前，我提到“理论上”我们可以使用 `PGA_AGGREGATE_TARGET` 来控制实例使用的 PGA 内存总量。但从上一个例子中我们看到，这并非一个硬性限制。实例会试图保持在 `PGA_AGGREGATE_TARGET` 的范围内，但如果无法做到，它不会停止处理；而只会被迫超出该阈值。

这个限制之所以是“理论”上的，另一个原因是工作区虽然是 PGA 内存的主要贡献者，但并非唯一贡献者。许多因素都会影响 PGA 内存分配，而只有工作区是受数据库实例控制的。如果你创建并执行了一个 `PL/SQL` 代码块，在专用服务器模式下（`UGA` 位于 `PGA` 中）用大量数据填充一个大数组，`Oracle` 除了允许你这样做之外，别无他法。

考虑下面这个简单的例子。我们将在服务器中创建一个可以存储一些持久（全局）数据的包：

```
$ sqlplus eoda/foo@PDB1
SQL> create or replace package demo_pkg
as
type array is table of char(2000) index by binary_integer;
g_data array;
end;
/
Package created.
```

现在，我们将测量当前会话在 `PGA`/`UGA` 中使用的内存量（本例中我使用了专用服务器，因此 `UGA` 是 `PGA` 内存的一个子集）：

```
SQL> select a.name, to_char(b.value, '999,999,999') bytes,
to_char(round(b.value/1024/1024,1), '99,999.9' ) mbytes
from v$statname a, v$mystat b
where a.statistic# = b.statistic#
and a.name like '%ga memory%';
NAME                           BYTES        MBYTES
------------------------------ ------------ ---------
session uga memory                3,796,448       3.6
session uga memory max            3,796,448       3.6
session pga memory                6,169,184       5.9
session pga memory max            6,169,184       5.9
```

最初，我们的会话使用了大约 6MB 的 `PGA` 内存（这是编译 `PL/SQL` 包、运行此查询等操作的结果）。现在，我们将使用相同的 300MB `PGA_AGGREGATE_TARGET` 再次运行对 `T` 的查询（这是在最近重启且其他方面空闲的实例中完成的；目前只有我们一个会话需要内存）：

```
SQL> set autotrace traceonly statistics;
SQL> select * from t order by 1,2,3,4;
72616 rows selected.
Statistics

148  recursive calls
0  db block gets
1599  consistent gets
1328  physical reads
0  redo size
4705439  bytes sent via SQL*Net to client
49873  bytes received via SQL*Net from client
4480  SQL*Net roundtrips to/from client
10  sorts (memory)
0  sorts (disk)
67180  rows processed
SQL> set autotrace off
```

如你所见，排序完全是内存中完成的。事实上，如果我们查看会话的 `PGA`/`UGA` 使用情况，就能看到使用了多少内存：

```
SQL> select a.name, to_char(b.value, '999,999,999') bytes,
to_char(round(b.value/1024/1024,1), '99,999.9' ) mbytes
from v$statname a, v$mystat b
where a.statistic# = b.statistic#
and a.name like '%ga memory%';
NAME                           BYTES        MBYTES
------------------------------ ------------ ---------
session uga memory                3,927,424       3.7
session uga memory max           16,022,584      15.3
session pga memory                5,644,896       5.4
session pga memory max           17,965,664      17.1
```

我们看到使用了大约 17MB 的 `RAM`。现在，我们将用数据填充包中的那个 `CHAR` 数组（`CHAR` 数据类型是空格填充的，因此这些数组元素中的每一个都正好是 2000 个字符长）：

```
SQL> begin
for i in 1 .. 200000
loop
demo_pkg.g_data(i) := 'x';
end loop;
end;
/
PL/SQL procedure successfully completed.
```

如果我们随后测量会话当前的 `PGA` 使用率，会发现大致如下：

```
SQL> select a.name, to_char(b.value, '999,999,999') bytes,
to_char(round(b.value/1024/1024,1), '99,999.9' ) mbytes
from v$statname a, v$mystat b
where a.statistic# = b.statistic#
and a.name like '%ga memory%';
NAME                           BYTES        MBYTES
------------------------------ ------------ ---------
session uga memory              478,432,768     456.3
session uga memory max          478,432,768     456.3
session pga memory              480,256,608     458.0
session pga memory max          480,256,608     458.0
```

现在，这是在 `PGA` 中分配的、实例本身无法控制的内存。我们已经*仅在这个单一会话中*就超过了为整个实例设置的 `PGA_AGGREGATE_TARGET` —— 数据库对此简直无能为力。如果它对此采取任何措施，那只能是拒绝我们的请求，而它只有在操作系统报告没有更多内存可分配时（`ORA-04030`）才会这样做。如果我们愿意，我们可以在该数组中分配更多空间并放入更多数据，而实例只能照办。

然而，实例知道我们做了什么。它不会忽略它无法控制的内存；它只是认识到内存正在被使用，并相应地缩减为工作区分配的内存大小。因此，如果我们重新运行相同的排序查询，会发现这次我们排序到磁盘了——实例没有给我们大约 12MB 的 `RAM` 来在内存中执行此操作，因为我们已经超出了 `PGA_AGGREGATE_TARGET`：

```
SQL> set autotrace traceonly statistics;
SQL> select * from t order by 1,2,3,4;
67180 rows selected.
Statistics

12  recursive calls
4  db block gets
1323  consistent gets
1418  physical reads
0  redo size
4705439  bytes sent via SQL*Net to client
49652  bytes received via SQL*Net from client
4480  SQL*Net roundtrips to/from client
0  sorts (memory)
1  sorts (disk)
67180  rows processed
SQL> set autotrace off
```

因此，由于部分 `PGA` 内存不在 `Oracle` 的控制之下，仅仅通过在我们的 `PL/SQL` 代码中分配大量非常大的数据结构，就很容易超出 `PGA_AGGREGATE_TARGET`。我绝不是在建议你这样做。我只是指出，`PGA_AGGREGATE_TARGET` 更像一个请求，而非硬性限制。

### PGA 与 UGA 总结

到目前为止，我们已经研究了两种内存结构：`PGA` 和 `UGA`。你现在应该理解，`PGA` 对一个进程是私有的。它是 `Oracle` 专用或共享服务器需要独立于会话而拥有的一组变量。`PGA` 是一个内存“堆”，可以在其中分配其他结构。`UGA` 也是一个内存堆，可以在其中定义各种会话特定的结构。当你使用专用服务器连接到 `Oracle` 时，`UGA` 是从 `PGA` 分配的；而在共享服务器连接下，则是从 `SGA` 分配的。这意味着在使用共享服务器时，你必须将 `SGA` 的大型池调整得足够大，以容纳所有可能同时连接到你的数据库的用户。因此，支持共享服务器连接的数据库的 `SGA` 通常比配置相似的、仅使用专用服务器模式的数据库的 `SGA` 要大得多。接下来，我们将更详细地介绍 `SGA`。


