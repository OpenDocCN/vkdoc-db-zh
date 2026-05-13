# 单元智能表扫描

`单元智能表扫描`事件是 Oracle 用来统计因卸载操作而进行全表扫描所花费的等待时间。它是 Exadata 平台上用于报表类工作负载的最重要事件。其存在与否可以用来验证一个语句是否从卸载操作中获益。正如第 2 章所讨论的，卸载仅在 Oracle 能够进行直接路径读取时发生。因此，在 Exadata 的大多数情况下，此事件取代了`直接路径读取`事件。与非 Exadata 平台上的直接路径读取一样，数据直接返回到数据库服务器上请求进程的`PGA`（无论是用户的影子进程还是并行从属进程）。数据块不会返回到`缓冲区缓存`。

### 事件含义

尽管通过 InfiniBand 网络执行读取的机制与非 Exadata 平台上的普通读取非常不同，但驱动智能扫描的代码路径实际上与非 Exadata 平台上的直接路径读取非常相似。主要区别在于，每个对存储单元的请求都包含对语句元数据的引用，在 Exadata 的情况下，这包括谓词和要返回的列列表等信息。由于存储单元可以访问这些信息，它们可以在将数据返回给请求进程之前应用过滤器并进行列投影。这些优化在每次请求一组数据块时都会应用。数据库服务器上请求数据的进程可以访问`ASM`区映射，因此可以从每个存储单元请求所需的分配单元（`AUs`）。在没有存储索引段的情况下，存储单元读取请求的`AU`，并应用谓词过滤器等任务。如果有任何行满足过滤条件，单元格便将投影的列返回给请求进程。然后，该进程请求下一个`AU`，并重复整个例程，直到扫描完所有数据。因此，在大型扫描中，此事件会反复出现。重要的是，与其他一些与 I/O 相关的事件不同，单个智能扫描事件不能用于推导 I/O 性能。您需要使用`OEM 12c`或`cellcli`命令来访问单元格上的指标信息。我们将在第 12 章中更详细地讨论这一点。

## 注意

列投影是智能扫描提供的主要优化之一。这个功能有些被误解。它不仅仅是将选择列表中的列传回数据库服务器；它还传回`WHERE`子句中涉及的一些列。旧版本的`cellsrv`会将`WHERE`子句中指定的所有列传回数据库层。后续版本已纠正此行为，只包含涉及连接谓词的列。例如，您可以在`DBMS_XPLAN.DISPLAY_CURSOR`的输出中看到投影的列。

与所有等待事件一样，`单元智能表扫描`将被记录在 SQL 跟踪中。但对于 Exadata，您可能需要进一步调查，因为跟踪文件不仅限于包含您设置的单个事件的信息。可能不立即显而易见的是，您可以将多个事件的输出组合到一个跟踪文件中。对于智能扫描，您可以包括`LIBCELL`客户端库（主要用于系统间通信）、Exadata 智能扫描层以及 SQL 跟踪的跟踪信息，以获得正在发生的一切的综合视图。考虑以下启用大量跟踪的代码片段。您通常不需要跟踪如此多的信息，但这对于研究非常有用！不用说，您不应在生产环境中启用这些事件。这仅适用于生产环境和灾难恢复环境之外的开发系统。跟踪中的信息量可能非常大，填满`/u01`挂载点并实际上导致服务中断。

```
SQL> select value from v$diag_info where name like 'Default%'

VALUE
----------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/db12c/db12c1/trace/db12c1_ora_11916.trc

SQL> alter session set events 'trace[LIBCELL.Client_Library.*] disk=highest';
Session altered.

SQL> alter session set events 'sql_trace level 8';
Session altered.

SQL> alter session set events 'trace[KXD.*] disk=highest';
Session altered.

SQL> select count(*) from t1;

COUNT(*)
----------
33554432
```


Oracle 已经非常友好地记录了许多可追踪的组件，因此您实际上可以看到代码前缀并将其映射到各自的代码层。获取神奇文档的命令是 `oradebug`。使用 `oradebug doc event name` 和 `oradebug doc component` 可以获取有关可追踪内容的更多信息，如下面这个 LIBCELL 的示例所示。

```
SQL> oradebug doc component libcell

Components in library LIBCELL:
--------------------------
Client_Library               Client Library
Disk_Layer                   Disk Layer
Network_Layer                Network Layer
IPC_Layer                    IPC Layer
```

上面代码示例中的星号（`*`）包含了所有子层，而无需显式指定。请注意，由此产生的跟踪信息量很可能非常巨大，并可能在数据库挂载点导致空间问题。但我们不要离题——回到 SQL 跟踪。当仅跟踪 SQL 时，跟踪文件显示以下几行：

```
=====================
PARSING IN CURSOR #140...784 len=23 dep=0 uid=198 oct=3 lid=198 tim=3471433479182
hv=4235652837 ad='808033f0' sqlid='5bc0v4my7dvr5'
select count(*) from t1
END OF STMT
PARSE #140...784:c=0,e=138,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,plh=3724264953,tim=3471433479181
EXEC #140...784:c=0,e=51,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,plh=3724264953,tim=3471433479280
WAIT #140...784: nam='SQL*Net message to client' ela= 4 driver id=1650815232 #bytes=1 p3=0
obj#=-1 tim=3471433479330
WAIT #140...784: nam='reliable message' ela= 968 channel context=10126085744 channel
handle=10164814784 broadcast message=10178993320 obj#=-1 tim=3471433480522
WAIT #140...784: nam='enq: KO - fast object checkpoint' ela= 194 name|mode=1263468550 2=65606
0=1 obj#=-1 tim=3471433480789
WAIT #140...784: nam='enq: KO - fast object checkpoint' ela= 125 name|mode=1263468545 2=65606
0=2 obj#=-1 tim=3471433480986
WAIT #140...784: nam='Disk file operations I/O' ela= 6 FileOperation=2 fileno=7 filetype=2
obj#=-1 tim=3471433481040
WAIT #140...784: nam='cell smart table scan' ela= 145 cellhash#=3249924569 p2=0 p3=0 obj#=61471
tim=3471433501731
WAIT #140...784: nam='cell smart table scan' ela= 149 cellhash#=674246789 p2=0 p3=0 obj#=61471
tim=3471433511233
WAIT #140...784: nam='cell smart table scan' ela= 143 cellhash#=822451848 p2=0 p3=0 obj#=61471
tim=3471433516295
WAIT #140...784: nam='cell smart table scan' ela= 244 cellhash#=3249924569 p2=0 p3=0 obj#=61471
tim=3471433561706
WAIT #140...784: nam='cell smart table scan' ela= 399 cellhash#=674246789 p2=0 p3=0
...
```

不幸的是，Oracle 在 11g 生命周期的某个地方改变了游标名称的格式。这会导致上面的输出中跟踪行换行。这就是为什么完整的游标标识符被截断了。

跟踪文件的这一部分还显示了 `enq: KO - fast object checkpoint` 事件，该事件用于确保在开始扫描之前，被扫描对象的所有脏块都已刷新到磁盘。顺便说一句，`direct path read` 事件在 Exadata 平台上并没有被完全消除。实际上，可以使用一个提示（hint）来禁用卸载（offloading），并观察同一语句在没有卸载的情况下是如何运行的：

```
=====================
PARSING IN CURSOR #140...520 len=75 dep=0 uid=198 oct=3 lid=198 tim=3471777663904 hv=2068810426 ad='9114cd20' sqlid='44xptp5xnz2pu'
select /*+ opt_param('cell_offload_processing','false') */
count(*) from t1
END OF STMT
PARSE #140...520:c=2000,e=1381,p=0,cr=0,cu=0,mis=1,r=0,dep=0,og=1,plh=3724264953,
tim=3471777663903
EXEC #140...520:c=0,e=41,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,plh=3724264953,tim=3471777663997
WAIT #140...520: nam='SQL*Net message to client' ela= 4 driver id=1650815232 #bytes=1 p3=0
obj#=61471 tim=3471777664057
WAIT #140...520: nam='enq: KO - fast object checkpoint' ela= 228 name|mode=1263468550 2=65677
0=2 obj#=61471 tim=3471777664460
WAIT #140...520: nam='reliable message' ela= 1036 channel context=10126085744 channel
handle=10164814784 broadcast message=10179010104 obj#=61471 tim=3471777665585
WAIT #140...520: nam='enq: KO - fast object checkpoint' ela= 134 name|mode=1263468550 2=65677
0=1 obj#=61471 tim=3471777665771
WAIT #140...520: nam='enq: KO - fast object checkpoint' ela= 126 name|mode=1263468545 2=65677
0=2 obj#=61471 tim=3471777665950
WAIT #140...520: nam='direct path read' ela= 807 file number=7 first dba=43778051 block cnt=13
obj#=61471 tim=3471777667104
WAIT #140...520: nam='direct path read' ela= 714 file number=7 first dba=43778081 block cnt=15
obj#=61471 tim=3471777668112
WAIT #140...520: nam='direct path read' ela= 83 file number=7 first dba=43778097 block cnt=15
obj#=61471 tim=3471777668322
WAIT #140...520: nam='direct path read' ela= 617 file number=7 first dba=43778113 block cnt=15
obj#=61471 tim=3471777669050
WAIT #140...520: nam='direct path read' ela= 389 file number=7 first dba=43778129 block cnt=15
obj#=61471 tim=3471777669549
```

请注意，我们仍然有用于刷新脏块的 `enq: KO – fast object checkpoint` 事件。因此，很明显，`cell smart table scan` 事件取代了这个事件。

### 参数

此事件的参数信息量不大。仅提供被扫描表的对象 ID 和单元哈希号：

*   `P1` - 单元哈希号
*   `P2` - 未使用
*   `P3` - 未使用
*   `obj#` - 被扫描的表段的数据对象 ID

您应该使用跟踪中报告的 `obj#` 在 `DBA_OBJECTS` 中查找表名。请记住，智能扫描（Smart Scan）在段级别操作——您需要查询 `data_object_id` 而不是 `object_id`。您会注意到 `direct path read` 事件（`cell smart table scan` 所取代的事件）提供了更多信息，包括文件号、文件中的偏移量（`first dba`）以及读取的连续块数（`block cnt`）。另一方面，对于 `direct path read` 事件，没有指示读取请求如何路由到各个单元。

许多等待事件中报告的单元哈希号可以在 `V$CELL` 视图中找到。该视图只有两列：`CELL_PATH` 和 `CELL_HASHVAL`。`CELL_PATH` 列实际上包含存储单元的 IP 地址。如果使用 KXD 和 LIBCELL 组件进行跟踪，您将在每个等待事件前后看到更多信息。这些额外的跟踪信息非常适合让您对智能扫描处理有极其深入的了解。完整解释这些内容超出了本章的范围；请参阅第 2 章。

与某些非 Exadata I/O 事件不同，智能扫描事件并不真正适合用于计时 I/O 完成。请回忆第 2 章的内容，智能扫描是针对单元中的各个线程异步触发的——每个都是独立的逻辑工作单元。Oracle 软件可能会按与执行顺序不同的顺序读取这些 I/O 事件的结果。这一点，以及智能扫描处理的复杂性，使得无法使用此事件来推导 I/O 延迟。

### cell smart index scan

当执行被卸载的快速全索引扫描时，时间会被计入 `cell smart index scan` 事件。此事件类似于 `cell smart table scan`，不同之处在于被扫描的对象是一个索引。不要将此索引访问路径与任何其他可用的索引访问路径（例如索引唯一、范围或全扫描）混淆。后者表示单块 I/O 调用，根据定义不能被卸载。单元智能索引扫描等待取代了 `direct path read` 事件，并将数据直接返回到请求进程的 PGA，而不是缓冲区缓存。


### 事件含义

在我们观察到的系统上，这个事件并不常见，可能出于以下几个原因：

*   Exadata 非常擅长执行全表扫描，因此迁移到该平台时倾向于删除大量索引。这可能包括为了查询优化目的而将索引设为不可见。
*   在索引扫描中执行直接路径读取的频率不如在表扫描中高。Oracle 11.2 的一个重要变化是它在串行表扫描中执行直接路径读取的激进程度。这项增强很可能是为了允许 Exadata 执行更多智能全表扫描而特别推进的，但无论如何，若没有此功能，则只有并行表扫描才能利用智能扫描。同样的增强也适用于索引快速完全扫描。也就是说，它们也可以通过串行直接路径读取来执行。然而，控制其发生时机的算法似乎不太可能对索引使用此技术（可能是因为索引可能比表小得多）。

此外，只有索引的快速完全扫描有资格进行智能扫描（范围扫描和全扫描没有资格）。由于这些问题，`cell smart index scans`事件相比`cell smart table scans`事件相当罕见。当然，可以通过提示（如`parallel_index`）或为特定索引设置大于 1 的并行度来鼓励使用此功能。以下是显示该事件的 10046 跟踪文件摘录：

```
=====================
PARSING IN CURSOR #140694139370464 len=100 dep=1 uid=198 oct=3 lid=198 tim=3551928848463
hv=3104270335 ad='85f450b8' sqlid='fg97a72whftzz'
select /*+ monitor gather_plan_statistics parallel_index(t1) */
/* ffs_test_001 */
count(id) from t1
END OF STMT
PARSE #140694139370464:c=0,e=114,p=0,cr=0,cu=0,mis=0,r=0,dep=1,og=1,plh=3371189879,
tim=3551928848462
WAIT #140694139370464: nam=' cell smart index scan ' ela= 173 cellhash#=674246789 p2=0 p3=0
obj#=78100 tim=3551928872228
WAIT #140694139370464: nam=' cell smart index scan ' ela= 127 cellhash#=674246789 p2=0 p3=0
obj#=78100 tim=3551928872442
WAIT #140694139370464: nam=' cell smart index scan ' ela= 8888 cellhash#=674246789 p2=0 p3=0
obj#=78100 tim=3551928881675
WAIT #140694139370464: nam=' cell smart index scan ' ela= 142 cellhash#=822451848 p2=0 p3=0
obj#=78100 tim=3551928900699
WAIT #140694139370464: nam=' cell smart index scan ' ela= 400 cellhash#=822451848 p2=0 p3=0
obj#=78100 tim=3551928901202
```

请注意，此跟踪文件是由并行从属进程之一产生的，而非请求进程。为清晰起见，部分 PX 消息已被移除。当禁用下推时，相同语句产生的跟踪文件应看起来更熟悉。以下是另一个摘录，同样来自查询从属进程：

```
=====================
PARSING IN CURSOR #13...92 len=158 dep=1 uid=198 oct=3 lid=198 tim=3553633447559 hv=3263861856 ad='9180bab0' sqlid='35yb4ug18p530'
select /*+ opt_param('cell_offload_processing','false') monitor
gather_plan_statistics parallel_index(t1) */
/* ffs_test_002 */
count(id) from t1
END OF STMT
PARSE #13...92:c=0,e=152,p=0,cr=0,cu=0,mis=0,r=0,dep=1,og=1,plh=3371189879,tim=3553633447557
WAIT #13...92: nam='PX Deq: Execution Msg' ela= 9671 sleeptime/senderid=268566527 passes=1
p3=9216340064 obj#=-1 tim=3553633458202
WAIT #13...92: nam=' direct path read ' ela= 5827 file number=7 first dba=39308443 block cnt=128 obj#=78100 tim=3553633464791
WAIT #13...92: nam='PX Deq: Execution Msg' ela= 62 sleeptime/senderid=268566527 passes=1
p3=9216340064 obj#=78100 tim=3553633481394
WAIT #13...92: nam=' direct path read ' ela= 2748 file number=7 first dba=39370251 block cnt=128
obj#=78100 tim=3553633484390
WAIT #13...92: nam='PX Deq: Execution Msg' ela= 64 sleeptime/senderid=268566527 passes=1
p3=9216340064 obj#=78100 tim=3553633497671
WAIT #13...92: nam=' direct path read ' ela= 4665 file number=7 first dba=39357143 block cnt=128
obj#=78100 tim=3553633502565
WAIT #13...92: nam='PX Deq: Execution Msg' ela= 70 sleeptime/senderid=268566527 passes=1
p3=9216340064 obj#=78100 tim=3553633518439
WAIT #13...92: nam=' direct path read ' ela= 3001 file number=7 first dba=39337739 block cnt=128
obj#=78100 tim=3553633521674
```

将此与前一个示例（其中存储 cell 下推了扫描）进行比较。在前一个示例中，`cell smart index scan`事件取代了直接路径读取。在启动直接路径读取之前用于刷新脏块的`enq: KO – fast object checkpoint`事件仍然存在。但在此摘录中未显示它们，因为它们发生在查询协调器进程中，而非并行从属进程中。

### 参数

与`cell smart table scan`事件类似，`cell smart index scan`的参数不包含很多细节。仅提供了存储 cell 哈希号和被扫描段的对象 ID：

*   `P1` - 存储 cell 哈希号
*   `P2` - 未使用
*   `P3` - 未使用
*   `obj#` - 被扫描索引的对象号

### cell single block physical read

此事件等同于非 Exadata 平台上使用的`db file sequential read`事件。单块读取最常用于索引访问路径（包括索引块读取和通过索引查找得到的 rowid 进行的表块读取）。它们也可用于各种其他适合读取单个块的操作。


