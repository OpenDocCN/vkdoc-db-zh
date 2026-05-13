# Exadata 智能扫描深度解析

`_object_statistics: enabled, Sage: enabled,`
`Direct Read for serial qry: enabled(::::kctfsage::), Ascending SCN table scan: FALSE`
`flashback_table_scan: FALSE, Row Versions Query: FALSE`
`SqlId: 71acyavyyg1dg, plan_hash_value: 2604480108, Object#: 25576, Parition#: 0`
`DW_scan: disabled`

在此跟踪信息中，您可以看到对象太小——只有四个块。由于段小于 `_small_table_threshold`，因此未选择直接路径读取。跟踪的最后一个参数也很有趣：`DW_SCAN` 与自动大表缓存（ABTC）相关，而与将查询卸载到存储服务器无关。

还有另一种可以识别的情况。它与 VLOT（非常大对象阈值）有关。您可以在第一个 `NSMTIO` 列表中看到一个引用，其中 `BIGTAB` 小于该阈值。VLOT 默认为 500，即缓冲区缓存大小的五倍。第一个 `NSMTIO` 跟踪中提供的附加信息显示 VLOT 为 11162160。实例的缓冲区缓存大小约为 20GB，即 2232432 个缓冲区。实例缓冲区缓存中的当前缓冲区数量可以从 `v$db_cache_advise` 中检索，如下所示：

```sql
SQL> select block_size,size_for_estimate,buffers_for_estimate
  2  from v$db_cache_advice where size_factor = 1 and name = 'DEFAULT';

BLOCK_SIZE SIZE_FOR_ESTIMATE BUFFERS_FOR_ESTIMATE
---------- ----------------- --------------------
      8192             20608              2232432
```

将 2232432 乘以 5 返回 11162160；请记住，`db_block_buffers` 是以块而非字节为单位进行测量的。

虽然执行串行直接路径读取的能力已经存在一段时间，但自 Oracle 11g 以来才变得相对常见。Oracle Database 11gR2 对用于确定是否对非并行扫描使用直接路径读取的计算进行了修改。对算法的新修改使得直接路径读取机制比以前版本更有可能发生。这可能是由于 Exadata 的智能扫描优化以及希望尽可能触发这些优化而采取的措施。该算法在非 Exadata 平台上可能有些过于激进。

## Exadata 存储

当然，要进行智能扫描，数据必须存储在 Exadata 存储上。可以在 Exadata 数据库服务器上创建访问非 Exadata 存储的 ASM 磁盘组。当然，任何访问使用这些非 Exadata 磁盘组定义的对象的 SQL 语句都将不符合卸载条件。

虽然不常见，但也可以使用 Exadata 和非 Exadata 存储的组合创建 ASM 磁盘组。由于无法将光纤通道主机总线适配器放入 Exadata 计算节点，因此网络连接存储是唯一的选择。随着 NAS 解决方案（如 ZFS 存储设备）的推出，将较冷的数据移动到更便宜的存储并通过 `dNFS` 访问变得越来越普遍。我们在第 3 章中，在自动数据优化（ADO）的上下文中讨论了此场景。

针对段驻留在混合存储上的对象的查询不符合卸载条件。实际上，ASM 磁盘组有一个属性（`cell.smart_scan_capable`）指定磁盘组是否能够处理智能扫描。在将非 Exadata 存储分配给 ASM 磁盘组之前，必须将此属性设置为 `FALSE`。

字典视图 `DBA_TABLESPACES` 有一个名为 `PREDICATE_EVALUATION` 的属性，您也可以查询。以下是针对我们的 X4-2 半机架实验室数据库查询的输出：

```sql
SQL> select tablespace_name, bigfile, predicate_evaluation
  2  from dba_tablespaces;

TABLESPACE_NAME                BIG PREDICA
------------------------------ --- -------
SYSTEM                         NO  STORAGE
SYSAUX                         NO  STORAGE
UNDOTBS1                       YES STORAGE
TEMP                           YES STORAGE
UNDOTBS2                       YES STORAGE
UNDOTBS3                       YES STORAGE
UNDOTBS4                       YES STORAGE
USERS                          NO  STORAGE
SOE                            YES STORAGE
SH                             YES STORAGE
```

## 智能扫描禁用器

在某些情况下，智能扫描实际上被禁用。简单的情况是它们在代码中尚未启用，因此智能扫描根本无法发生。还有其他情况是 Oracle 开始走智能扫描路径，但存储软件决定或被迫恢复为块传输模式。通常，此决策是逐块做出的。智能扫描禁用器的完整列表可在 Exadata 文档集中找到，幸运的是，在撰写本文时该文档是公开可用的。请参阅《存储服务器软件用户指南》第 7 章中的“将 SQL EXPLAIN PLAN 命令与 Oracle Exadata 存储服务器软件结合使用”部分。您可能需要时不时地参考它，因为 Oracle 不断增强软件，当前的限制可能会在未来的版本中解除。

### 完全不可用

在讨论智能扫描优化时，您了解了启用智能扫描必须满足的先决条件。然而，即使满足这些条件，也有一些情况会阻止智能扫描。以下是其他一些与特定优化无关，但根本无法使用智能扫描的情况：

*   在集群表或索引组织表（IOT）上
*   查询扫描行外 LOB 或 LONG 数据类型
*   在启用了 `ROWDEPENDENCIES` 的表上
*   查询包含 `flashback_query_clause` 的情况
*   无法对反向键索引卸载查询的情况
*   在非 Exadata 存储上查询数据的情况

您在前面的部分还看到了一些影响智能扫描行为的参数。如果您将 `CELL_OFFLOAD_PROCESSING` 设置为 `FALSE` 或将 `_SERIAL_DIRECT_READ` 设置为 `never`，那么根据定义，您无法拥有智能扫描。



### 回退至块传输模式

存在一些使用了智能扫描（Smart Scans）的场景，但由于各种原因，`cellsrv` 会回退到块传输模式。这是一个非常复杂的话题，我们曾犹豫是否要在关于卸载（offloading）的入门章节中包含它。但由于它是一个基本概念，我们决定在此简要讨论。关于此主题的更多细节，请参阅第 11 章。

到目前为止，本章将智能扫描描述为一种避免将大量数据传输到数据库层的手段，其方式是直接将预过滤的数据返回给`PGA`。主要工作由存储单元（storage cells）承担——存储单元越多，扫描执行得就越快。你的查询即使只返回表中 2%的数据，也并不意味着你可以避免扫描全部数据，正如你可以在`V$SQL`以及本书后续将介绍的其他地方所看到的那样。请记住，存储单元彼此之间完全独立运行；换句话说，它们在查询处理期间**从不**进行通信。查询处理期间的通信仅限于存储服务器与计算节点（如果查询在集群中并行处理，则是多个节点）之间的信息交换。在此上下文中，另一个重要信息是智能扫描只会返回一致读（consistent reads），而不是当前块（current blocks）。

偶尔，智能扫描可以选择（或被强制）将完整的块返回给`SGA`。基本上，任何导致 Oracle 需要读取另一个块来完成/回滚记录到某个快照`SCN`（系统更改号）的情况，都会导致此行为发生。链接行（chained row）是另一个，也许是最简单的例子。当 Oracle 遇到链接行时，该行的头部将包含一个指向包含第二行片（row piece）的块的指针。由于存储单元彼此之间不直接通信，并且链接的块很可能不在同一个存储单元上，`cellsrv` 便直接传输整个块，并由数据库层来处理它。

在这个非常简单的情况下，智能扫描会暂停片刻，并有效地执行一次单块读取，这又促使另一次单块读取以获取额外的行片。请记住，这是一个非常简单的情况。

当 Oracle 必须处理读一致性问题时，也会出现同样的行为。例如，如果 Oracle 注意到一个块比当前查询的`SCN`“更新”，那么查找该块适当年龄版本的过程就交由数据库层处理。这实际上会暂停智能扫描处理，同时数据库执行其传统的读一致性处理。延迟块清理（Delayed block cleanout）是另一个可能需要暂停智能扫描的类似情况。

> 注意
>
> 本节篇幅太短，无法完整恰当地传达全貌；这些场景的复杂程度远超我们在入门章节想要涵盖的范围。所有细节都可以在第 11 章中找到。

那么，这真的重要吗？你为什么应该关心？答案当然是：视情况而定。在大多数情况下，你可能无需担心。Oracle 保证读取是一致的，即使在使用智能扫描时也是如此。一些优化措施，例如第 11 章中讨论的提交缓存（commit cache），有助于加速处理。从应用的角度来看，无论是否使用智能扫描，Oracle 的行为都完全相同，这一点非常重要。Exadata 并非一个高度专业的分析引擎。它使用的仍是与其他所有人完全相同的数据库软件。如果结果是正确的，且性能没有受到严重影响，那么 Oracle 可能在进行智能扫描的同时做一些单块读取这件事就无需过多担心，而且在大多数情况下也不会有影响。然而，在某些情况下，选择执行智能扫描然后回退到块传输模式，从性能角度来看可能会很痛苦。正是在这些情况下，理解底层发生的事情才变得重要。再次说明，你可以在第 11 章中找到关于此问题的更多信息。

### 跳过部分卸载处理

我们将要简要提及的另一个非常复杂的行为是 `cellsrv` 拒绝执行某些正常卸载处理的能力。这样做可能是为了避免使存储单元上的 CPU 资源过载。这种行为的一个很好的例子发生在解压缩 `HCC`（混合列压缩）数据时。解压缩是一项极其消耗 CPU 的任务，特别是对于较高的压缩级别。自 Exadata 存储软件 11.2.2.3.0 及更高版本起，当存储单元上的 CPU 非常繁忙时，`cellsrv` 可以选择跳过对部分数据的解压缩步骤。这通过强制数据库主机进行解压缩，有效地将部分工作负载移回数据库层。

### 静默跳过卸载处理

有时，Exadata 软件不得不回退到所谓的 `passthrough mode`（直通模式）。这可能是一个令人担忧的问题，因为它并不总是明显发生了这种情况，尤其是在 11g Release 2 中。通过一个例子可以最好地解释这个问题。以下查询通常执行时间很短：

```
SQL> select count(*) from bigtab where id = 80000;

  COUNT(*)
----------
        32
Elapsed: 00:00:00.83
```

假设该语句突然需要 25 秒才能执行。系统化的方法是检查执行计划是否改变、统计数据是否更新、数据量是否变化等等。但这次什么都没变（这次是真的没变）。该语句在不到一秒内执行时是被卸载到存储单元上的，现在检查你也可以看到等待事件也指示了卸载。如果你有使用 `ASH`（活动会话历史）的许可，你可以使用一个非常基础的查询来验证这一点：

```
SQL> select count(*), event, session_state from v$active_session_history
  2  where sql_id = '0pmmwn5xq8h9a' group by event, session_state;

  COUNT(*) EVENT                        SESSION
---------- ---------------------------- -------
        28                               ON CPU
        46 cell smart table scan        WAITING
```

有趣的是，正如 `Cell Smart Table Scan` 事件的存在所表明的，查询是被卸载处理的。“为什么这么慢？”这个问题的答案，必定在别处。有点超前地说，它在于会话统计信息中。使用第 11 章中描述的工具 `snapper` 或 `mystats`，你可以发现存在大量的 `passthrough`（直通）操作：

```
Type    Statistic Name                                                   Value
------  ----------------------------------------------------------------  ----------------
STAT    cell num bytes in passthru during predicate offload             28,004,319,232
STAT    cell num smart IO sessions using passthru mode due to cellsrv                1
STAT    cell physical IO bytes eligible for predicate offload           83,886,137,344
STAT    cell physical IO bytes saved by storage index                   51,698,524,160
STAT    cell physical IO interconnect bytes returned by smart scan      28,004,930,160
```

`直通模式` 意味着存储单元仍然执行部分智能扫描，但它们不应用谓词过滤，而是将整个块传递给 `RDBMS` 层。你可以在第 11 章中阅读更多关于 `直通模式` 的内容。


## 如何验证智能扫描正在发生

关于 Exadata，你需要学习的最重要的内容之一就是如何识别一个查询是否能够利用智能扫描。有趣的是，由 `DBMS_XPLAN` 包生成的普通执行计划输出**不会**显示是否使用了智能扫描。下面是一个示例：

```
PLAN_TABLE_OUTPUT
-------------------------------------
SQL_ID  2y17pb7bnmpt0, child number 0
-------------------------------------
select count(*) from bigtab where id = 17000

Plan hash value: 2140185107

-------------------------------------------------------------------------------------
| Id  | Operation                  | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT           |        |       |       |  2779K(100)|          |
|   1 |  SORT AGGREGATE            |        |     1 |     6 |            |          |
|*  2 |   TABLE ACCESS STORAGE FULL| BIGTAB |    32 |   192 |  2779K  (1)| 00:01:49 |
-------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - storage("ID"=17000)
       filter("ID"=17000)
```

注意，优化器选择了一个 `TABLE ACCESS STORAGE FULL` 操作，并且谓词部分显示了一个与计划第 2 步关联的 `storage()` 谓词。这两个特征都表明**可能**发生了智能扫描，但都不能提供确凿的验证。实际上，这个清单中的语句**并没有**使用智能扫描执行。如果你想知道原因，那是因为我们在执行查询前，在会话中将 `_serial_direct_read` 设置为了 `never`。

执行计划不显示是否进行了智能扫描，这一点有些令人沮丧。不过，有几种技术可以用来解决这个问题。接下来的几节将介绍一些实用的技术。请注意，关于分析智能扫描是否发生以及其效率如何的主题，在第 10 章和第 11 章中有更详细的介绍。

### 10046 跟踪

确定是否使用了智能扫描最直接的方法之一是对目标语句启用 10046 跟踪。不幸的是，这种方法有点繁琐，并且不允许你对过去的执行情况进行调查。尽管如此，跟踪是一种相当可靠的方法来验证是否使用了智能扫描。如果使用了智能扫描，跟踪文件中将包含 `CELL SMART TABLE SCAN` 或 `CELL SMART INDEX SCAN` 事件。以下是为前述语句收集的跟踪文件摘录（已重新格式化以提高可读性）：

```
PARSING IN CURSOR #1..4 len=44 dep=0 uid=65 oct=3 lid=65 tim=1625363834946
  hv=3611940640 ad='5e7a2e420' sqlid='2y17pb7bnmpt0'
WAIT #139856525281664: nam='cell single block physical read' ela= 1237 ...
WAIT #139856525281664: nam='cell single block physical read' ela= 651 ...
WAIT #139856525281664: nam='cell single block physical read' ela= 598 ...
...
WAIT #139856525281664: nam='cell multiblock physical read' ela= 1189 ...
WAIT #139856525281664: nam='cell single block physical read' ela= 552 ...
WAIT #139856525281664: nam='cell multiblock physical read' ela= 596 ...
WAIT #139856525281664: nam='cell multiblock physical read' ela= 612 ...
WAIT #139856525281664: nam='cell multiblock physical read' ela= 607 ...
WAIT #139856525281664: nam='cell multiblock physical read' ela= 632 ...
WAIT #139856525281664: nam='cell multiblock physical read' ela= 618 ...
[...]
```

请注意，此部分跟踪文件中记录的事件是单块读和多块读。Oracle 借此机会将 `db file sequential read` 和 `db file scattered read` 等待事件重命名为更不易混淆的 `cell single-block read` 和 `cell multi-block read`。下面是一个显示智能扫描的示例：

```
PARSING IN CURSOR #139856525283104 len=44 dep=0 uid=65 oct=3 lid=65 tim=1625653524727
  hv=3611940640 ad='5e7a2e420' sqlid='2y17pb7bnmpt0'
select count(*) from bigtab where id = 17000
END OF STMT
PARSE #139856525283104:c=0,e=117,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,plh=2140185107,...
EXEC #139856525283104:c=0,e=55,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,plh=2140185107,...
WAIT #139856525283104: nam='SQL*Net message to client' ela= 3 ...
WAIT #139856525283104: nam='reliable message' ela= 1049 channel context=26855200120 ...
WAIT #139856525283104: nam='enq: KO - fast object checkpoint' ela= 298 ...
WAIT #139856525283104: nam='enq: KO - fast object checkpoint' ela= 156 ...
WAIT #139856525283104: nam='cell smart table scan' ela= 151 ...
WAIT #139856525283104: nam='cell smart table scan' ela= 168 ...
WAIT #139856525283104: nam='cell smart table scan' ela= 153 ...
WAIT #139856525283104: nam='cell smart table scan' ela= 269 ...
WAIT #139856525283104: nam='cell smart table scan' ela= 209 ...
WAIT #139856525283104: nam='cell smart table scan' ela= 231 ...
WAIT #139856525283104: nam='cell smart table scan' ela= 9 ...
[...]
```

在第二个例子中，你可以看到许多 `cell smart table scan` 事件，这表明处理已经下推到了存储层。

### 会话性能统计

另一种方法是查看一些性能视图，例如 `V$SESSSTAT` 和 `V$MYSTAT`。这种方法常常被忽视，但正如你在关于直通模式章节中所见，它非常有用。要调查当前正在执行 SQL 语句的会话中发生了什么，一个极好的方法是使用 Tanel Poder 的 `Snapper` 脚本。它提供了一个很好的方式来查看语句运行时产生了哪些等待事件。此外，它还可以捕获在观察 SQL 语句期间会话计数器的变化。`Snapper` 专注于**当前正在执行**的 SQL 语句；它不是用来回溯历史数据的。

只要你能在调查的语句执行期间访问系统，性能统计就是一个可靠的数据源。以下是一个使用 `V$MYSTATS` 的例子，它只是 `V$SESSSTAT` 的一个版本，将数据限制为当前会话。在这个例子中，重点是 `cell scans` 统计信息，当某个段上发生智能表扫描时，该计数器会增加：

```
SQL> @mystat
Enter value for name: cell scans
NAME                       VALUE
--------------------------------
cell scans                      0
Elapsed: 00:00:00.04

SQL> select count(*) from bigtab where id = 17001;
  COUNT(*)
----------
        32
Elapsed: 00:00:00.44

SQL> @mystat
Enter value for name: cell scans
NAME                       VALUE
--------------------------------
cell scans                      1
Elapsed: 00:00:00.02

SQL>
```

如你所见，该查询触发了会话计数器的递增。可以肯定地说，在两次执行 `mystats` 脚本之间发生了一次智能扫描。

**注意**

请不要混淆此脚本与本章中提到的另一个名为 `mystats` 的脚本。`mystat` 脚本从 `v$mystat` 中选择数据，并打印给定会话计数器的当前值。而 `mystats`（由 Adrian Billington 编写，可从 `oracle-developer.net` 获取）计算在执行 SQL 语句期间会话计数器的变化，类似于默认模式下的 `Snapper`，但它是从开始到结束进行计算的。

关于会话计数器还有很多内容可讲，我们在第 11 章中会进行介绍。


## 可卸载字节数

判断一条语句是否使用了智能扫描（Smart Scan）还有另一个线索。正如你在前面章节中看到的，`V$SQL`系列视图包含一个名为`IO_CELL_OFFLOAD_ELIGIBLE_BYTES`的列，它显示了有资格进行卸载的字节数。该列可以用作判断语句是否使用了智能扫描的指标。该列似乎只有在使用了智能扫描时才会被设置为大于 0 的值。你可以利用这一观察结果编写一个简短的脚本（`fsx.sql`），该脚本根据`V$SQL`中该列的值是否大于 0 返回`YES`或`NO`。该脚本的输出对于书籍格式来说有点过宽，因此在示例中提供了几个精简版本。当然，所有版本都可以在线上代码仓库中找到。你在前面的几个章节中已经看到了该脚本的实际运行。为方便起见，将脚本及其使用示例展示如下：

```sql
> !cat fsx.sql

----------------------------------------------------------------------------------------
--
-- File name:   fsx.sql
--
-- Purpose:     Find SQL and report whether it was Offloaded and % of I/O saved.
--
-- Usage:       This scripts prompts for two values.
--
--              sql_text: a piece of a SQL statement like %select col1, col2 from skew%
--
--              sql_id: the sql_id of the statement if you know it (leave blank to ignore)
--
-- Description:
--
--              This script can be used to locate statements in the shared pool and
--              determine whether they have been executed via Smart Scans.
--
--              It is based on the observation that the IO_CELL_OFFLOAD_ELIGIBLE_BYTES
--              column in V$SQL is only greater than 0 when a statement is executed
--              using a Smart Scan. The IO_SAVED_% column attempts to show the ratio of
--              of data received from the storage cells to the actual amount of data
--              that would have had to be retrieved on non-Exadata storage. Note that
--              as of 11.2.0.2, there are issues calculating this value with some queries.
--
--              Note that the AVG_ETIME will not be acurate for parallel queries. The
--              ELAPSED_TIME column contains the sum of all parallel slaves. So the
--              script divides the value by the number of PX slaves used which gives an
--              approximation.
--
--              Note also that if parallel slaves are spread across multiple nodes on
--              a RAC database the PX_SERVERS_EXECUTIONS column will not be set.
--
---------------------------------------------------------------------------------------
set pagesize 999
set lines 190
col sql_text format a70 trunc
col child format 99999
col execs format 9,999
col avg_etime format 99,999.99
col "IO_SAVED_%" format 999.99
col avg_px format 999
col offload for a7
select sql_id, child_number child, plan_hash_value plan_hash, executions execs,
(elapsed_time/1000000)/decode(nvl(executions,0),0,1,executions)/
decode(px_servers_executions,0,1,px_servers_executions/
decode(nvl(executions,0),0,1,executions)) avg_etime,
px_servers_executions/decode(nvl(executions,0),0,1,executions) avg_px,
decode(IO_CELL_OFFLOAD_ELIGIBLE_BYTES,0,'No','Yes') Offload,
decode(IO_CELL_OFFLOAD_ELIGIBLE_BYTES,0,0,
100*(IO_CELL_OFFLOAD_ELIGIBLE_BYTES-IO_INTERCONNECT_BYTES)
/decode(IO_CELL_OFFLOAD_ELIGIBLE_BYTES,0,1,IO_CELL_OFFLOAD_ELIGIBLE_BYTES))
"IO_SAVED_%", sql_text
from v$sql s
where upper(sql_text) like upper(nvl('&sql_text',sql_text))
and sql_text not like 'BEGIN :sql_text := %'
and sql_text not like '%IO_CELL_OFFLOAD_ELIGIBLE_BYTES%'
and sql_id like nvl('&sql_id',sql_id)
order by 1, 2, 3
/
```



在 `fsx` 脚本中，可以看到 `OFFLOAD` 列实际上是一个 `DECODE` 函数，用于检查 `IO_CELL_OFFLOAD_ELIGIBLE_BYTES` 列是否等于 0。`IO_SAVED_%` 列是通过 `IO_INTERCONNECT_BYTES` 字段计算得出的，它试图显示有多少数据被返回给了数据库服务器。

该脚本可用于多种实用目的。作者主要用它来查找共享池中 SQL 语句的 `SQL_ID` 和子游标号。在此示例中，它用于判断一条语句是否已被卸载：

```sql
SQL> select /*+ gather_plan_statistics fsx-example-002 */
  2  avg(id) from bigtab where id between 1000 and 50000;

AVG(ID)
----------
     25500

Elapsed: 00:00:00.64

SQL> alter session set cell_offload_processing=false;

Session altered.

Elapsed: 00:00:00.00

SQL> select /*+ gather_plan_statistics fsx-example-002 */
  2  avg(id) from bigtab where id between 1000 and 50000;

AVG(ID)
----------
     25500

Elapsed: 00:00:53.88

SQL> @fsx4
Enter value for sql_text: %fsx-example-002%
Enter value for sql_id:

SQL_ID          CHILD OFFLOAD IO_SAVED_%  AVG_ETIME SQL_TEXT
------------- ------ ------- ---------- ---------- ----------------------------------------
cj0p52wha5wb8      0 Yes          99.97        .63 select /*+ gather_plan_statistics fsx-ex
cj0p52wha5wb8      1 No             .00      53.88 select /*+ gather_plan_statistics fsx-ex

2 rows selected.
```

执行时间已经能大致说明语句是否被卸载，但如果是事后才介入分析，`fsx` 脚本的输出清晰地显示子游标号 (`child_number`) 为 1 的语句未被卸载。在此示例中，创建了一个新的子游标，这一点非常重要。当将 `CELL_OFFLOAD_PROCESSING` 设置为 `FALSE` 时，优化器由于不匹配而创建了一个新的子游标。创建子游标的原因可以在 `v$sql_shared_cursor` 中找到。该视图包含一长串标志，可用于识别子游标之间的差异，但在 SQL*Plus 中非常难以阅读。Oracle 在 11.2.0.2 版本中添加了一个包含 XML 数据的 CLOB 列，使得差异更容易发现。利用上一个示例中的 SQL ID，可以演示其用法。请注意，我将 CLOB 转换为 XML 以增强可读性：

```sql
SQL> select xmltype(reason) from v$sql_shared_cursor
  2   where sql_id = 'cj0p52wha5wb8' and child_number = 0;

XMLTYPE(REASON)
---------------------------------------------------------------------------
<ChildNode>
  <ChildNumber>0</ChildNumber>
  <ID>3</ID>
  <reason>Optimizer mismatch(12)</reason>
  <size>2x356</size>
  <cell_offload_processing> true     false  </cell_offload_processing>
</ChildNode>
```

将 XML 输出翻译成简单的英文，可以看到存在一个优化器不匹配：参数 `cell_offload_processing` 从 `TRUE` 更改为 `FALSE`。

更改参数后并不总是会创建子游标。某些下划线参数（如 `_SERIAL_DIRECT_READ`）不会导致创建新的子游标。同一游标的某些执行可能会被卸载，而其他则不会。这可能相当令人困惑，尽管这种情况应该非常罕见！下面是一个演示效果的示例：

```sql
SQL> select /*+ gather_plan_statistics fsx-example-004 */ avg(id)
  2  from bigtab where id between 1000 and 50002;

AVG(ID)
----------
     25501

Elapsed: 00:00:00.68

SQL> alter session set "_serial_direct_read" = never;

Session altered.

Elapsed: 00:00:00.00

SQL> select /*+ gather_plan_statistics fsx-example-004 */ avg(id)
  2  from bigtab where id between 1000 and 50002;

AVG(ID)
----------
     25501

Elapsed: 00:04:50.32

SQL> SQL> alter session set "_serial_direct_read" = auto;

Session altered.

Elapsed: 00:00:00.00

SQL> select /*+ gather_plan_statistics fsx-example-004 */ avg(id)
  2  from bigtab where id between 1000 and 50002;

AVG(ID)
----------
     25501

Elapsed: 00:00:00.63

SQL> @fsx4
Enter value for sql_text: %fsx-example-004%
Enter value for sql_id:

SQL_ID          CHILD OFFLOAD IO_SAVED_%  AVG_ETIME SQL_TEXT
------------- ------ ------- ---------- ---------- ----------------------------------------
6xh6qwv302p13      0 Yes          55.17      97.21 select /*+ gather_plan_statistics fsx-ex
```

如你所见，有三次执行使用的是同一个子游标（没有创建新的子游标）。关于 I/O 节省和执行时间的统计信息现在几乎没有价值了：两次执行在不到一秒内完成，一次则花了近五分钟。这就是众所周知的平均值问题：它们掩盖了细节。


