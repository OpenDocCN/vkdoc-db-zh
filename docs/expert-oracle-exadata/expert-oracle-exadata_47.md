# Exadata 等待事件

## 单元单块物理读

### 事件含义

以下查询输出展示了在一个活跃的生产系统上导致 `cell single block physical read` 等待事件的操作：

```sql
SQL> select event, operation,   count(*) from (
  2   select sql_id, event, sql_plan_operation||' '||sql_plan_options operation
  3     from DBA_HIST_ACTIVE_SESS_HISTORY
  4     where event like 'cell single%')
  5     group by operation, event
  6     order by 1,2,3
  7   /
EVENT                             OPERATION                                          COUNT(*)
--------------------------------- ------------------------------------------------ ----------
cell single block physical read                                                      13321
 CREATE TABLE STATEMENT                            35
 DDL STATEMENT                                   118
 DELETE                                          269
 FIXED TABLE FIXED INDEX                            3
 FOR UPDATE                                         2
 HASH JOIN                                          4
 HASH JOIN RIGHT OUTER                               8
 INDEX FULL SCAN                                   9283
 INDEX FULL SCAN (MIN/MAX)                            1
 INDEX RANGE SCAN                                  2763
 INDEX STORAGE FAST FULL SCAN                          6
 INDEX STORAGE SAMPLE FAST FULL SCAN                   13
 INDEX UNIQUE SCAN                                   1676
 INSERT STATEMENT                                   1181
 LOAD AS SELECT                                       6
 LOAD TABLE CONVENTIONAL                              92
 MERGE                                             106
 SELECT STATEMENT                                    41
 SORT ORDER BY                                        6
 TABLE ACCESS BY GLOBAL INDEX ROWID                 10638
 TABLE ACCESS BY INDEX ROWID                         8714
 TABLE ACCESS BY LOCAL INDEX ROWID                  10446
 TABLE ACCESS CLUSTER                                  12
 TABLE ACCESS STORAGE FULL                            776
 TABLE ACCESS STORAGE SAMPLE                           40
 UPDATE                                            8116
```

如你所见，通过索引进行行访问是产生此事件的最常见操作。你还应该注意，Exadata 在每个存储单元上提供了大量的闪存缓存。因此，物理读（包括多块和单块读）比大多数基于磁盘的存储系统要快得多，即使没有进行下推处理。

以下摘录自 AWR 报告，显示了该实例的单块读直方图：

```
% of Waits
-----------------------------------------
Total
Event                         Waits <1ms <2ms <4ms <8ms <16ms <32ms <=1s  >1s
-------------------------- ----- ---- ---- ---- ---- ----- ----- ---- ----
cell single block physical 2940K 94.4  3.2   .3   .6    .9    .5   .2   .0
```

请注意，大约 95% 的 `cell single block physical read` 事件耗时小于 1 毫秒。这在我们观察过的几个生产系统中具有相当的代表性。

### 参数

`cell single block physical read` 事件比大多数单元事件提供了更多信息。这些参数允许你准确判断读取了哪个对象，并提供存储该块的磁盘和单元信息：

*   `P1` - 单元哈希值
*   `P2` - 磁盘哈希值
*   `P3` - 读取操作期间传递的总字节数（假设块大小为 8K，则始终为 8192）
*   `obj#` - 正在读取的对象的对象号

## 单元多块物理读

这是另一个重新命名的事件。它等同于以前名称不太清晰的 `db file scattered read` 事件。在非 Exadata 平台上，Oracle Database 11gR2 和 12c 仍然在向操作系统发出连续的多块缓冲区读时使用 `db file scattered read` 事件。旧事件名称中的“分散”反映了从磁盘读取的数据在缓冲区缓存中的存储方式：分散各处。它并非反映数据从磁盘读取的方式（实际上是连续的）。

### 事件含义

此事件通常与全表扫描和快速全索引扫描一起使用，尽管它也可用于许多其他操作。在 Exadata 平台上，新名称比旧名称更具描述性。对于报告型工作负载，这种等待事件在 Exadata 平台上远不如在非 Exadata 平台上普遍，因为 Exadata 使用具有自身等待事件（`cell smart table scan` 和 `cell smart index scan`）的智能扫描来处理许多全扫描操作。

Exadata 平台上的 `cell multiblock physical read` 事件用于低于串行直接路径读取阈值的表的串行全扫描操作。也就是说，你最常在此事件用于对相对较小的表进行全扫描时看到它。另一方面，如果你的查询没有进行下推处理（例如由于非常大的缓冲区缓存），你也会看到大量此类事件。在这些情况下，启动智能扫描或直接路径读取之前的阈值非常高。

此事件也用于不以直接路径读取执行的快速全索引扫描。以下查询输出展示了一个活跃生产系统上导致 `cell multiblock physical read` 等待事件的操作：

```
EVENT                             OPERATION                                          COUNT(*)
-------------------------------- ------------------------------------------------ ----------
cell multiblock physical read                                                      764
                                 DDL STATEMENT                                        28
                                 INDEX FAST FULL SCAN                                  2
                                 INDEX STORAGE FAST FULL SCAN                        657
                                 INDEX STORAGE SAMPLE FAST FULL SCAN                 133
                                 TABLE ACCESS BY INDEX ROWID                          74
                                 TABLE ACCESS BY LOCAL INDEX ROWID                   1428
                                 TABLE ACCESS STORAGE FULL                           5046
                                 TABLE ACCESS STORAGE SAMPLE                          916
```

### 参数

`cell multiblock physical read` 事件也比大多数单元事件提供了更多信息。以下列表中的参数允许你判断读取了哪个对象，并识别存储块的磁盘和单元。传递的总字节数应是块大小的倍数：

*   `P1` - 单元哈希值
*   `P2` - 磁盘哈希值
*   `P3` - 读取操作期间传递的总字节数
*   `obj#` - 正在读取的对象的对象号

## 单元块列表物理读

此事件替代了非 Exadata 平台上的 `db file parallel read` 事件。看起来开发人员借此机会重命名了一些与磁盘操作相关的事件，这就是其中之一。新名称实际上比之前的名称更具描述性，因为该等待事件与并行查询或并行 DML 毫无关系。


### 事件含义

此事件用于多块读取非连续块。这在异步 I/O 下更为高效，而 Exadata 默认启用异步 I/O。多个操作可能触发此事件。最常见的是索引范围扫描、索引唯一扫描以及通过索引 ROWID 访问表。触发该事件的最常见原因是索引预提取。以下查询输出显示了在 Exadata 系统上导致此等待事件的操作：

```
SQL> select event, operation, count(*) from (
  2  select sql_id, event, sql_plan_operation||' '||sql_plan_options operation
  3    from DBA_HIST_ACTIVE_SESS_HISTORY
  4    where event like 'cell list%')
  5    group by operation, event
  6    order by 1,2,3
 7 /

EVENT                                    OPERATION                                 COUNT(*)
---------------------------------------- ----------------------------------- -------------
cell list of blocks physical read                                                         2
                                         INDEX RANGE SCAN                              156
                                         INDEX STORAGE FAST FULL SCAN                    1
                                         INDEX UNIQUE SCAN                              66
                                         TABLE ACCESS BY GLOBAL INDEX ROWID             90
                                         TABLE ACCESS BY INDEX ROWID                  1273
                                         TABLE ACCESS BY LOCAL INDEX ROWID            2593
                                         TABLE ACCESS STORAGE FULL                      20
                                         TABLE ACCESS STORAGE SAMPLE                     1
```

如您所见，这些事件绝大多数由索引访问路径引发。顺便提一下，在非 Exadata 平台上，非连续多块读取仍计入旧的 `db file parallel read` 事件。在某些操作中，此旧等待事件名称也可能出现在 Exadata 平台上。

### 参数

`cell list of blocks physical read` 事件比大多数单元事件提供更多信息。以下参数允许您准确识别读取的对象以及存储该块的磁盘和单元：

*   P1 - 单元哈希值
*   P2 - 磁盘哈希值
*   P3 - 读取的块数
*   obj# - 被读取对象的对象号

### cell smart file creation

Exadata 拥有一种优化技术，允许存储单元在创建或扩展数据文件时初始化块。这发生在创建表空间或手动向表空间添加数据文件时。但是，在 DML 操作期间自动扩展数据文件时也可能发生。

#### Event Meaning

您已在本章前面读到，此事件在 User I/O 类类似乎不合时宜。然而，如果它是由 DML 操作引发的，那么将其归入此类别是合理的。无论如何，将块格式化操作 offload（卸载）消除了数据库服务器的 CPU 使用和 I/O，并将其转移到存储层。当这种情况发生时，时间被计入 `cell smart file creation` 事件。此事件取代了在非 Exadata 平台上仍使用的 `Data file init write` 事件。以下是一个繁忙生产系统的查询输出，显示了产生此事件的操作：

```
SYS@EXDB1> select event, operation, count(*) from (
  2  select sql_id, event, sql_plan_operation||' '||sql_plan_options operation
  3    from DBA_HIST_ACTIVE_SESS_HISTORY
  4    where event like 'cell smart file%')
  5    group by operation, event
  6    order by 1,2,3
 7 /

EVENT                      OPERATION                  COUNT(*)
-------------------------- ------------------------- --------
cell smart file creation                                35
                             DELETE                       3
                             INDEX BUILD NON UNIQUE       5
                             LOAD AS SELECT               3
                             LOAD TABLE CONVENTIONAL      1
                             UPDATE                       1
```

您会注意到，在这个特定系统上，`DELETE` 语句偶尔也会生成 `cell smart file creation` 事件。`DELETE` 可能导致此事件可能有点令人惊讶。但请记住，此事件实际上是在为一段执行块格式化的代码计时，而不是为文件创建计时。

#### Parameters

此事件唯一值得关注的参数是 `P1`，它显示生成此事件时正在访问哪个单元：

*   `P1` - 单元哈希值
*   `P2` - 未使用
*   `P3` - 未使用

### cell statistics gather

`cell statistics gather` 事件记录从各种 `V$` 和 `X$` 表中读取所花费的时间。尽管该事件被归类在 User I/O 类别中，但它并非指读写磁盘意义上的 I/O。

#### Event Meaning

当会话从 `V$CELL` 系列视图以及同一类别中的少数其他 `X$` 表中读取时，时间会计入此事件。我们认为该事件被错误分类，实际上并不属于 User I/O 类别。以下是一个典型示例：

```
PARSING IN CURSOR #140003020408112 len=79 dep=0 uid=0 oct=3 lid=0 tim=4163271218132 hv=2136254865 ad='a2c23650' sqlid='fcjwbgdzp9acj'
select count(cell_name),cell_name from V$CELL_THREAD_HISTORY group by cell_name
END OF STMT
PARSE #14...12:c=17997,e=23189,p=5,cr=99,cu=0,mis=1,r=0,dep=0,og=1,plh=3108513074,tim=4
WAIT #14...12: nam='Disk file operations I/O' ela= 34 FileOperation=8 fileno=0
  filetype=8 obj#=262 tim=4163271218207
EXEC #14...12:c=0,e=54,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,plh=3108513074,tim=4...
WAIT #14...12: nam='SQL*Net message to client' ela= 2 driver id=1650815232 #bytes=1 p3=0
   obj#=262 tim=4163271218304
WAIT #14...12: nam=' cell statistics gather ' ela= 162 cellhash#=0 p2=0 p3=0 obj#=262 tim=4163
WAIT #14...12: nam=' cell statistics gather ' ela= 248 cellhash#=0 p2=0 p3=0 obj#=262 tim=4163
WAIT #14...12: nam=' cell statistics gather ' ela= 236 cellhash#=0 p2=0 p3=0 obj#=262 tim=4163
WAIT #14...12: nam=' cell statistics gather ' ela= 250 cellhash#=0 p2=0 p3=0 obj#=262 tim=4163
```

注意

您可以在第 11 章阅读更多关于 `V$CELL%` 系列视图的信息。

#### Parameters

此事件的参数不提供额外信息。实际上，此事件的参数值甚至没有被设置。参数定义如下：

*   P1 - 单元哈希值（始终为 0）
*   P2 - 未使用
*   P3 - 未使用

### User/IO 类中的次要事件

正如您在引言中所读到的，Oracle 12.1.0.2 是长久以来首个引入新的 Exadata 相关等待事件的版本。再次总结一下，这些事件是：

*   cell list of blocks read request
*   cell multi-block read request
*   cell physical read no I/O
*   cell single-block read request

尽管在准备本章时非常谨慎，但我们无法找到一种方法来为它们生成等待，但有两个例外：`cell single block read request` 和 `cell multi-block read request` 已被发现是执行 `DBMS_STATS.GATHER_SCHEMA_STATS` 的一部分。相应的 `SQL_ID` 明确表明这些事件是在统计信息收集作业期间触发的。`TOP_LEVEL_SQL_ID` 可用于确认这一点。



