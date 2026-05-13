# 第十四章

## 最终思考

你现在读到了本书的最后一章，到目前为止已经阅读和学习了很多内容。本书涵盖了许多材料，你对 Exadata 是什么以及它能做什么有了更好的认识。你也对该系统及其管理方式有了扎实的操作知识。起初这可能是一项令人望而生畏的任务，但通过阅读本书并运行示例，你已经获得了对 Exadata 的理解，这将为你继续作为 Exadata DBA 或 DMA 的职业生涯提供良好的服务。

## 你已走得很远，朝圣者

当你开始阅读时，你可能刚刚被告知 Exadata 即将到来，并且你将与之合作。很可能你对 Exadata 如何配置或它如何提供大家都在谈论的卓越性能并没有真正的概念。存储索引和智能扫描是陌生的概念，该系统的架构也是你前所未见的。从那时起，你已经走了很长的路。

让我们回顾一下我们涵盖的领域，并重点介绍我们介绍过的一些材料。

![image](img/sq.jpg) **注意** 这不会是我们之前章节中所呈现的所有主题和材料的详尽列表。某些领域的信息可能比其他领域更详细，但请了解本章并非对全文的全面总结。

我们很恰当地从开头开始，讨论了 Exadata 系统的各种配置。我们这样做是为了向你介绍当前可用各系统上的硬件，并让你熟悉每种配置上的可用资源。我们还简要介绍了 Exadata 的历史，以及它如何发展以应对不同类型的工作负载，包括 OLTP。

这段旅程的下一站将你带到了智能扫描和卸载，这是 Exadata 两个非常重要的方面。

有人说智能扫描是 Exadata 的命脉，我们同意。正如你所了解到的，智能扫描提供了一种“分而治之”的方法来处理查询和语句，它允许存储单元执行在传统硬件配置中必须由数据库服务器执行的任务。你还了解到，卸载提供了更强大的工具，使数据处理更加高效。列投影和谓词过滤通过将数据块处理转移到存储层，使存储单元仅返回查询请求的数据。回想一下，列投影允许 Exadata 仅返回选择列表中请求的列以及连接条件中指定的任何列。这通过消除数据库服务器执行传统数据块处理和过滤的需要，大大减少了工作量。这也减少了传输到数据库服务器的数据量，这是 Exadata 比传统配置系统更高效的一种方式。

### 须知事项

每个 ERP 数据库都必须从自己的 Oracle Home 运行。不允许在 ERP 数据库之间共享 Oracle Home。这是一项长期的 Oracle 标准。此外，每个数据库都必须有一个专用的监听器。

监听器的问题可能是最具挑战性的。首先，你必须决定 SCAN 监听器是否可以简单地留在 1521 端口，还是必须移动。虽然你可以使用 SCAN 监听器，但在开发/测试集群上这样做并不实际，因为你很可能运行着多个 ERP 数据库。每个数据库将需要自己的监听器端口，因此也需要自己的监听器。为了保持一致，我们发现最好不要让任何数据库使用 SCAN 监听器作为主监听器。

尽管如此，你仍然必须将数据库注册到本地监听器，以便 `dbca` 能够工作。如果它没有在本地监听器中注册，当你尝试运行 `dbca` 时会收到错误。实际上，每个数据库都应该用为该数据库唯一的命名监听器和端口以及默认监听器进行注册。请记住，`dbca` 用于多种用途，包括向数据库添加实例、向 Grid Control OMS 注册数据库以及配置 Database Vault——仅举几例。

务必配置你的 `$ORACLE_HOME/network/admin`，使得后续在数据库层运行 Autoconfig 不会损害你的 `TNS` 配置。务必将配置充分告知任何可能运行 Autoconfig 的人。

设置环境时，切勿使用 Autoconfig 生成的 `TNS_ADMIN` 设置。Autoconfig 生成的 `env` 文件没问题，但 `TNS_ADMIN` 变量必须被取消设置（unset）。

正确克隆每个 `ORACLE_HOME`，使其在中央清单中注册。如果不遵循克隆过程，那么 Home 将不会被注册，你将无法为其应用季度补丁包。

有经验的 ERP DBA 知道 Autoconfig 会在 `$ORACLE_HOME` 下创建一个名为 `<context>.env` 的 `env` 文件。使用我们一直使用的名称，在节点 1 上我们会有 `erpdev1_exadb01.env`，在节点 2 上是 `erpdev2_exadb02.env`，等等。然而，`env` 文件会将 `TNS_ADMIN` 设置为 `$ORACLE_HOME/network/admin/<context>`——出于我们先前讨论的原因，我们不使用它。你必须通过从 Autoconfig 生成的 `env` 文件中注释掉该行，或者通过创建另一个 `env` 文件来调用该文件并在其中包含 `unset TNS_ADMIN` 语句，来做出调整。我建议不要修改生成的 `env` 文件，因为它会在任何 Autoconfig 运行时被覆盖，因此你每次都需要记住进行调整。

创建一个名为 `<db>.env` 的文件，并将其放在路径中的某个目录下，例如 `/usr/local/bin`。我们的数据库名称是 `erpdev`，所以我们会是

```
$ cat erpdev.env
.  /u01/app/oracle/product/11.2.0.3/erpdevdb/erpdev1_exadb01.env
unset TNS_ADMIN
echo;
echo "********************************************************"
echo "ORACLE_HOME...$ORACLE_HOME"
echo "ORACLE_SID....$ORACLE_SID"
echo "********************************************************"
echo;
```

然后，为了进一步简化操作，在 `.bash_profile` 中添加一个别名定义：

```
alias erpdev='. /usr/local/bin/erpdev.env'
```

然后，为某个数据库设置环境就变得轻而易举了。

```
$ erpdev

********************************************************
ORACLE_HOME... /u01/app/oracle/product/11.2.0.3/erpdevdb
ORACLE_SID....erpdev1
********************************************************
```

## “我的 Oracle 支持”说明

任何使用过 Oracle 产品的人都应该熟悉“My Oracle Support”，这是 Oracle 的网站，包含其知识库、补丁下载和服务请求系统。你也可能听到它被称为 `Metalink`，因为那是它多年来的名字。现在，人们通常称其为“MOS”。无论你怎么称呼它，它都是一个宝贵的资源。

以下是可以在那里找到的相关说明列表。只需搜索参考 ID，该说明就应该在搜索结果中排名第一。

*   将 Oracle 11g Release 2 真正应用集群与 Oracle 电子商务套件 Release 11i 配合使用 [ID 823586.1]
*   关于应用程序 11i 数据库 11g 的导出/导入说明 [ID 557738.1]
*   Oracle EBS 11i 与 Oracle 数据库 11gR2 (11.2.0) 的互操作性说明 [ID 881505.1]
*   安装后如何修改 SCAN 设置或 SCAN 监听器端口 [ID 972500.1]

![image](img/frontdot.jpg)



谓词过滤通过仅返回基于所提供谓词感兴趣行的数据，进一步提升了此性能。执行智能扫描所需的其他条件，包括直接路径读取以及全表或索引扫描，之前已经介绍过。执行计划也得到了解释，以便您了解哪些信息能表明确实使用了智能扫描，同时还提供了其他指标以进一步证明智能扫描的使用。这些额外指标包括谓词信息中的 `storage` 关键字，以及 `V$SQL`/`GV$SQL` 视图对中的两列：`io_cell_offload_eligible_bytes` 和 `io_cell_offload_returned_bytes`。以下示例（首次出现在第 2 章）展示了这些列值如何提供智能扫描执行信息。

```sql
SQL>select  sql_id,
  2         io_cell_offload_eligible_bytes qualifying,
  3         io_cell_offload_eligible_bytes - io_cell_offload_returned_bytes actual,
  4         round(((io_cell_offload_eligible_bytes -
io_cell_offload_returned_bytes)/io_cell_offload_eligible_bytes)*100, 2) io_saved_pct,
  5         sql_text
  6  from v$sql
  7  where io_cell_offload_returned_bytes> 0
  8  and instr(sql_text, 'emp') > 0
  9  and parsing_schema_name = 'BING';

SQL_ID        QUALIFYING     ACTUAL IO_SAVED_PCT SQL_TEXT
------------- ---------- ---------- ------------ -------------------------------------
gfjb8dpxvpuv6  185081856   42510928        22.97 select * from emp where empid = 7934

SQL>
```

我们还讨论了 Bloom 过滤器及其在 Exadata 中如何改进连接处理。Bloom 过滤器是限定连接卸载过程的一部分，其使用使连接更加高效。您可以通过合格查询的执行计划看到 Bloom 过滤器正在使用，如下例所示（同样首次在第 2 章中提供）。

```sql
Execution Plan

Plan hash value: 2313925751

Predicate Information (identified by operation id):

7 - access("ED"."EMPID"="E"."EMPID")
  15 - access("D"."DEPTNUM"=20)
  17 - storage("ED"."EMPDEPT"=20)
       filter("ED"."EMPDEPT"=20)
  19 - storage(SYS_OP_BLOOM_FILTER(:BF0000,"E"."EMPID"))
       filter(SYS_OP_BLOOM_FILTER(:BF0000,"E"."EMPID"))

Note

- dynamic sampling used for this statement (level=2)

Statistics

60  recursive calls
    174  db block gets
  40753  consistent gets
  17710  physical reads
   2128  redo size
9437983  bytes sent via SQL*Net to client
 183850  bytes received via SQL*Net from client
  16668  SQL*Net roundtrips to/from client
      6  sorts (memory)
      0  sorts (disk)
 250000  rows processed

SQL>
```

执行计划输出中存在 `SYS_OP_BLOOM_FILTER` 函数表明该连接已被卸载。

如果函数在 `V$SQLFN_METADATA` 视图中找到，它们也可以被卸载；未在此列表中找到的函数将使查询或语句无法执行智能扫描。虚拟列也可以被卸载，这使得定义了虚拟列的表有资格执行智能扫描。

存储索引在第 3 章中讨论过，它们可能是 Exadata 中最令人困惑的方面，因为这种索引用于告诉 Oracle *不要* 在哪里查找数据。存储索引旨在辅助卸载处理，通过消除表的 1MB 数据段，可以显著减少 Oracle 读取的数据量。您了解到存储索引包含最少量的数据。您还了解到，尽管存储索引很小，但它无疑非常强大。它用于跳过未找到所需数据的 1MB 段，由此产生的节省可能是巨大的。当然，有其利也有其弊，存储索引可能提供误报，导致 Oracle 读取它不需要的 1MB 段，因为所需值落在存储索引记录的最小/最大范围内，即使该值实际上并不在该表的 1MB 段中。为了刷新您的记忆，以下示例说明了这一点。

```sql
SQL> insert /*+ append */
  2  into chicken_hr_tab (chicken_name, talent_cd, retired, retire_dt, suitable_for_frying, fry_dt)
  3  select
  4  chicken_name, talent_cd, retired, retire_dt, suitable_for_frying, fry_dt from chicken_hr_tab2
  5  where talent_cd in (3,5);

1048576 rows created.

Elapsed: 00:01:05.10
SQL> commit;

Commit complete.

Elapsed: 00:00:00.01
SQL> insert /*+ append */
  2  into chicken_hr_tab (chicken_name, talent_cd, retired, retire_dt, suitable_for_frying, fry_dt)
  3  select
  4  chicken_name, talent_cd, retired, retire_dt, suitable_for_frying, fry_dt from chicken_hr_tab2
  5  where talent_cd not in (3,5);

37748736 rows created.

Elapsed: 00:38:09.12
SQL> commit;

Commit complete.

Elapsed: 00:00:00.00
SQL>
SQL> exec dbms_stats.gather_table_stats(user, 'CHICKEN_TALENT_TAB', cascade=>true, estimate_percent=>null);

PL/SQL procedure successfully completed.

Elapsed: 00:00:00.46
SQL> exec dbms_stats.gather_table_stats(user, 'CHICKEN_HR_TAB', cascade=>true, estimate_percent=>null);

PL/SQL procedure successfully completed.

Elapsed: 00:00:31.66
SQL>
SQL> set timing on
SQL>
SQL> connect bing/#########
Connected.
SQL> alter session set parallel_force_local=true;

Session altered.

Elapsed: 00:00:00.00
SQL> alter session set parallel_min_time_threshold=2;

Session altered.

Elapsed: 00:00:00.00
SQL> alter session set parallel_degree_policy=manual;

Session altered.

Elapsed: 00:00:00.00
SQL>
SQL>
SQL> set timing on
SQL>
SQL> select /*+ parallel(4) */
  2  chicken_id
  3  from chicken_hr_tab
  4  where talent_cd = 4;

CHICKEN_ID

...

4718592 rows selected.

Elapsed: 00:03:17.92
SQL>
SQL> select *
  2  from v$mystat
  3  where statistic# = (select statistic# from v$statname where name = 'cell physical IO bytes saved by storage index');

SID      STATISTIC#           VALUE
--------------- --------------- ---------------
            915             247               0

Elapsed: 00:00:00.01
SQL>
```

我们认为，为了如此高的效率，这是一个很小的代价。

第 4 章将我们引向智能闪存缓存，它是 Exadata 一个非常多功能的特性，因为它可以用作直读缓存、写回缓存，甚至可以配置为 ASM 可用的闪存盘。除了此功能范围外，智能闪存缓存还有一部分配置为智能闪存日志，以使重做日志处理更加高效。日志写入既通过日志组写入磁盘，也写入智能闪存日志，首先完成的写入会向 Oracle 发出信号，表明日志处理已成功完成。这使得 Oracle 能够以比单独使用重做日志更快的速率继续事务处理。


## 第 5 章

第 5 章 带我们进入了并行处理的领域，在传统配置的系统中，这一领域可能既有益处也有阻碍。Oracle 11.2.0.x 在并行语句的实现和执行方面提供了改进，这些改进适用于运行此 Oracle 版本的任何 Exadata 或非 Exadata 配置。正是 Exadata 最初是为数据仓库工作负载而设计这一事实——这类工作负载依赖于并行处理来加速工作流程——使得它成为并行处理的理想选择。

需要某些配置设置才能启用自动并行处理功能，而 Exadata 要实际根据这些配置设置采取行动，一项非常重要的任务——I/O 校准——是必需的。I/O 校准是一个资源密集型过程，不应在系统高负载期间运行。为此提供了一个打包过程 `dbms_resource_manager.calibrate_io`。一旦 I/O 校准完成，您将需要验证表 5-1 和表 5-2 中的参数和设置。

一旦您将所有设置就绪，就让 Oracle 接管控制权；然后用户就能享受到这一改进的并行处理功能带来的好处。其中一个好处是并行语句排队，Oracle 会推迟执行并行化语句，直到有足够的资源可用。查询 `V$SQL_MONITOR` 视图（或 `GV$SQL_MONITOR` 以查看集群中所有节点的语句），其状态为 `QUEUED`，将提供一个列表，显示那些正在等待执行完毕的并行语句释放出足够资源的语句。只要您正确设置了资源限制，这就能防止系统过载，相比早期的并行自适应多用户机制是一个巨大的进步。

另一个好处是，Oracle 将根据可用资源动态设置并行度（DOP）。需要注意的一个重要事项是，一旦 DOP 由 Oracle 设置，在执行期间就无法更改，即使有额外的资源变得可用也不行。DOP 是基于给定语句的估计串行执行时间来计算的。参数 `parallel_min_time_threshold` 设置了串行执行时间的阈值，这将成为决定是否进行并行执行的关键因素。请记住，您可以通过将隐藏参数 `_parallel_statement_queuing` 设置为 FALSE 来控制是否启用排队。

并行执行的另一个方面是内存中并行执行。一方面，这可能是理想的行为，因为它消除了磁 I/O 和数据库服务器与存储单元之间的流量，并且内存的延迟远低于磁盘访问。另一方面，当此功能激活时，智能扫描优化不再可用，因为磁 I/O 被消除了。由于智能扫描优化被消除，数据库服务器必须执行所有存储单元本应执行的读取处理，这增加了 CPU 使用率。我们还注意到，在我们管理过的任何 Exadata 系统上，我们尚未经历过内存中并行执行。

## 第 6 章

接下来是第 6 章关于压缩的内容。Exadata 提供了在任何非 Exadata 平台上都不可用的压缩选项，特别是混合列压缩（HCC）选项。这些选项包括 QUERY HIGH、QUERY LOW、ARCHIVE HIGH 和 ARCHIVE LOW。这些选项可以显著减小所选表的大小；然而，压缩表应该处于非活动状态，或者在正常工作时间之外进行批处理，包括重新压缩。如果 HCC 用于频繁更新的表，压缩级别会自动更改为 OLTP，原本节省的空间就丢失了。使用 HCC 的另一个方面是灾难恢复。如果恢复方法只是将最后一个良好的备份恢复到非 Exadata 服务器或不使用 Exadata 存储的服务器，则恢复完成后数据库可能无法使用，因为 HCC 在非 Exadata 存储上不受支持。这种情况可以通过解压缩表来纠正，但纠正时，如果目标服务器的表空间是基于 Exadata 数据库的压缩大小创建的，您可能会遇到空间问题。

第 6 章中的表 6-3 和表 6-4 列出了各种压缩方法（包括 OLTP）的估计和实际压缩比率。这些表格不会在此处重现，但如果您正在考虑使用某种压缩方法来节省空间，再次查看它们是个好主意。

## 第 7 章

第 7 章涵盖了 Exadata 特有的等待事件。这些等待事件报告与单元相关的等待信息。其中七个等待事件在 I/O 等待类别下收集等待数据，而 `cell statistics gather` 事件似乎被错误地包含在此类别中。`cell single block physical read` 和 `cell multiblock physical read` 事件基本上分别取代了旧的 `db file sequential read` 和 `db file scattered read` 事件。

RMAN 也有 Exadata 特定的等待事件：`cell smart incremental backup` 和 `cell smart restore from backup`。第一个等待事件针对增量 Level 1 备份收集时间，第二个则累积 RMAN 恢复过程中经历的等待时间。由于 Level 0 增量本质上是一个完整备份，这些备份经历的等待时间不会被记录。

## 第 8 章

第 8 章涵盖了 Exadata 特有的性能计数器和指标。该章的目的是概述有哪些指标和计数器可用、它们的含义以及您应该在何时考虑使用它们。

单元指标提供了对语句性能以及 Exadata 如何执行语句的洞察。动态计数器如 `cell blocks processed by data layer` 和 `cell blocks processed by index layer` 显示了存储单元在处理过程中的效率。每次存储单元能够在无需将数据传回数据库层的情况下完成数据层和索引层处理时，这些计数器就会增加。存储单元将数据传回数据库服务器的两个原因是：需要撤销块的一致性读处理（常规块 I/O）以及跨存储单元的链式行处理。虽然第一个条件无法完全控制（事务可能非常大，超过了自动块清理的阈值），但第二个问题，链式行，是可以解决并可能纠正的。


`cell num fast response sessions` 和 `cell num fast response sessions continuing to smart scan` 这两个计数器揭示了 Oracle 选择推迟智能扫描而转为常规块 I/O 的次数，这是为了以最少的操作返回所请求的数据（第一个列出的计数器），以及在快速响应会话未能返回所需数据后，Oracle 实际上启动智能扫描的次数。通过展示因为小型常规块 I/O 操作返回了所请求数据而避免智能扫描的频率，这些计数器可以帮助您洞察提交到 Exadata 数据库的语句的性质。

`V$SQL` 也提供了可用于确定语句效率的数据，体现在 `IO_CELL_OFFLOAD_ELIGIBLE_BYTES` 和 `IO_CELL_OFFLOAD_RETURNED_BYTES` 这两列中。这两列可用于计算给定查询通过智能扫描实现的节省百分比。

第 9 章 深入探讨了存储单元监控，这是 Exadata 非常重要的一个方面，因为某些指标不会传回数据库服务器。除了 “root” 账户外，还有两个可用账户，即 `cellmonitor` 和 `celladmin`。使用哪一个取决于需要完成的任务。

`cellmonitor` 账户可以访问单元指标和计数器，并能生成监控报告。它在操作系统（O/S）级别的访问权限有限，无法执行来自 `cellcli`（单元命令行界面）的任何管理命令或功能。基本上，`LIST` 命令对 `cellmonitor` 是可用的，这些命令足以监控单元并验证其运行正常。

正如您所预期的那样，`celladmin` 账户拥有更大的权限。除了能像 `cellmonitor` 一样生成报告外，它还能够执行一系列 `ALTER`、`ASSIGN`、`DROP`、`EXPORT` 和 `IMPORT` 命令。

除了 `cellcli`，还可以使用 `cellsrvstat` 以及从集群中任何节点使用 `dcli` 来监控存储单元。`cellcli` 工具也可以直接从命令行运行，只需将希望执行的命令传递给它即可，如下例所示。

```
[celladmin@myexa1cel03 ∼]$ cellcli -e "list flashcache detail"
         name:                   myexa1cel03_FLASHCACHE
         cellDisk:               FD_00_myexa1cel03,FD_11_myexa1cel03,FD_02_myexa1cel03,FD_09_
myexa1cel03,FD_06_myexa1cel03,FD_14_myexa1cel03,FD_15_myexa1cel03,FD_03_myexa1cel03,FD_05_
myexa1cel03,FD_10_myexa1cel03,FD_07_myexa1cel03,FD_01_myexa1cel03,FD_13_myexa1cel03,FD_04_
myexa1cel03,FD_08_myexa1cel03,FD_12_myexa1cel03
         creationTime:           2012-08-28T14:15:50-05:00
         degradedCelldisks:
         effectiveCacheSize:     364.75G
         id:                     95f4e303-516f-441c-8d12-1795f5024c70
         size:                   364.75G
         status:                 normal
[celladmin@myexa1cel03 ∼]$
```

使用 `dcli` 从任一数据库节点，您可以查询所有存储单元或其子集。有两个选项控制轮询的单元数量：`-g` 选项，用于向 `dcli` 传递一个包含所有存储单元名称的组文件；以及 `-c` 选项，您可以在命令行上指定想要获取信息的存储单元列表。配置 Exadata 时，会创建一个名为 `cell_group` 的组文件（以及其他文件），可用于轮询所有可用的存储单元，如下所示：

```
[oracle@myexa1db01] $ dcli -g cell_group cellcli -e "list flashcache detail"
myexa1cel01: name:                       myexa1cel01_FLASHCACHE
myexa1cel01: cellDisk:                   FD_07_myexa1cel01,FD_12_myexa1cel01,FD_15_
myexa1cel01,FD_13_myexa1cel01,FD_04_myexa1cel01,FD_14_myexa1cel01,FD_00_myexa1cel01,FD_10_
myexa1cel01,FD_03_myexa1cel01,FD_09_myexa1cel01,FD_02_myexa1cel01,FD_08_myexa1cel01,FD_01_
myexa1cel01,FD_05_myexa1cel01,FD_11_myexa1cel01,FD_06_myexa1cel01
myexa1cel01: creationTime:               2013-03-16T12:16:39-05:00
myexa1cel01: degradedCelldisks:
myexa1cel01: effectiveCacheSize:         364.75G
myexa1cel01: id:                         3dfc24a5-2591-43d3-aa34-72379abdf3b3
myexa1cel01: size:                       364.75G
myexa1cel01: status:                     normal
myexa1cel02: name:                       myexa1cel02_FLASHCACHE
myexa1cel02: cellDisk:                   FD_04_myexa1cel02,FD_15_myexa1cel02,FD_02_
myexa1cel02,FD_11_myexa1cel02,FD_05_myexa1cel02,FD_01_myexa1cel02,FD_08_myexa1cel02,FD_14_
myexa1cel02,FD_13_myexa1cel02,FD_07_myexa1cel02,FD_03_myexa1cel02,FD_09_myexa1cel02,FD_12_
myexa1cel02,FD_00_myexa1cel02,FD_06_myexa1cel02,FD_10_myexa1cel02
myexa1cel02: creationTime:               2013-03-16T12:49:04-05:00
myexa1cel02: degradedCelldisks:
myexa1cel02: effectiveCacheSize:         364.75G
myexa1cel02: id:                         a450958c-5f6d-4b27-a70c-a3877430b82c
myexa1cel02: size:                       364.75G
myexa1cel02: status:                     normal
myexa1cel03: name:                       myexa1cel03_FLASHCACHE
myexa1cel03: cellDisk:                   FD_00_myexa1cel03,FD_11_myexa1cel03,FD_02_
myexa1cel03,FD_09_myexa1cel03,FD_06_myexa1cel03,FD_14_myexa1cel03,FD_15_myexa1cel03,FD_03_
myexa1cel03,FD_05_myexa1cel03,FD_10_myexa1cel03,FD_07_myexa1cel03,FD_01_myexa1cel03,FD_13_
myexa1cel03,FD_04_myexa1cel03,FD_08_myexa1cel03,FD_12_myexa1cel03
myexa1cel03: creationTime:               2012-08-28T14:15:50-05:00
myexa1cel03: degradedCelldisks:
myexa1cel03: effectiveCacheSize:         364.75G
myexa1cel03: id:                         95f4e303-516f-441c-8d12-1795f5024c70
myexa1cel03: size:                       364.75G
myexa1cel03: status:                     normal
myexa1cel04: name:                       myexa1cel04_FLASHCACHE
myexa1cel04: cellDisk:                   FD_08_myexa1cel04,FD_10_myexa1cel04,FD_00_
myexa1cel04,FD_12_myexa1cel04,FD_03_myexa1cel04,FD_02_myexa1cel04,FD_05_myexa1cel04,FD_01_
myexa1cel04,FD_13_myexa1cel04,FD_04_myexa1cel04,FD_11_myexa1cel04,FD_15_myexa1cel04,FD_07_
myexa1cel04,FD_14_myexa1cel04,FD_09_myexa1cel04,FD_06_myexa1cel04
myexa1cel04: creationTime:               2013-07-09T17:33:53-05:00
myexa1cel04: degradedCelldisks:
myexa1cel04: effectiveCacheSize:         1488.75G
myexa1cel04: id:                         7af2354f-1e3b-4932-b2be-4c57a1c03f33
myexa1cel04: size:                       1488.75G
myexa1cel04: status:                     normal
myexa1cel05: name:                       myexa1cel05_FLASHCACHE
myexa1cel05: cellDisk:                   FD_11_myexa1cel05,FD_03_myexa1cel05,FD_15_
myexa1cel05,FD_13_myexa1cel05,FD_08_myexa1cel05,FD_10_myexa1cel05,FD_00_myexa1cel05,FD_14_
myexa1cel05,FD_04_myexa1cel05,FD_06_myexa1cel05,FD_07_myexa1cel05,FD_05_myexa1cel05,FD_12_
myexa1cel05,FD_09_myexa1cel05,FD_02_myexa1cel05,FD_01_myexa1cel05
myexa1cel05: creationTime:               2013-07-09T17:33:53-05:00
myexa1cel05: degradedCelldisks:
myexa1cel05: effectiveCacheSize:         1488.75G
myexa1cel05: id:                         8a380bf9-06c3-445e-8081-cff72d49bfe6
myexa1cel05: size:                       1488.75G
myexa1cel05: status:                     normal
[oracle@myexa1db01]$
```

如果您怀疑只有某些特定单元导致了问题，可以指定轮询这些单元，如下例所示。



[oracle@myexa1db01]$ dcli -c myexa1cel02,myexa1cel05 cellcli -e "list flashcache detail"
myexa1cel02: name:                       myexa1cel02_FLASHCACHE
myexa1cel02: cellDisk:                   FD_04_myexa1cel02,FD_15_myexa1cel02,FD_02_
myexa1cel02,FD_11_myexa1cel02,FD_05_myexa1cel02,FD_01_myexa1cel02,FD_08_myexa1cel02,FD_14_
myexa1cel02,FD_13_myexa1cel02,FD_07_myexa1cel02,FD_03_myexa1cel02,FD_09_myexa1cel02,FD_12_
myexa1cel02,FD_00_myexa1cel02,FD_06_myexa1cel02,FD_10_myexa1cel02
myexa1cel02: creationTime:               2013-03-16T12:49:04-05:00
myexa1cel02: degradedCelldisks:
myexa1cel02: effectiveCacheSize:         364.75G
myexa1cel02: id:                         a450958c-5f6d-4b27-a70c-a3877430b82c
myexa1cel02: size:                       364.75G
myexa1cel02: status:                     normal
myexa1cel05: name:                       myexa1cel05_FLASHCACHE
myexa1cel05: cellDisk:                   FD_11_myexa1cel05,FD_03_myexa1cel05,FD_15_
myexa1cel05,FD_13_myexa1cel05,FD_08_myexa1cel05,FD_10_myexa1cel05,FD_00_myexa1cel05,FD_14_
myexa1cel05,FD_04_myexa1cel05,FD_06_myexa1cel05,FD_07_myexa1cel05,FD_05_myexa1cel05,FD_12_
myexa1cel05,FD_09_myexa1cel05,FD_02_myexa1cel05,FD_01_myexa1cel05
myexa1cel05: creationTime:               2013-07-09T17:33:53-05:00
myexa1cel05: degradedCelldisks:
myexa1cel05: effectiveCacheSize:         1488.75G
myexa1cel05: id:                         8a380bf9-06c3-445e-8081-cff72d49bfe6
myexa1cel05: size:                       1488.75G
myexa1cel05: status:                     normal
[oracle@myexa1db01]$

使用 `dcli`，您可以将输出重定向到一个文件。设置一个脚本来定期执行此监控，并将输出写入带日期的日志文件，是通过 `cron` 调度此监控的好方法。

延续监控的主题，第 10 章 讨论了在数据库级别和存储单元级别监控 Exadata 的各种方法，并包括了实时 SQL 监控的讨论。涵盖了 GUI 和脚本方法。

建立基线是必须的，以确保您实施的任何监控过程都能提供有用且可用的信息。没有基线，您充其量只是在试图击中一个移动的目标。基线不需要在性能“良好”时建立；它只需要被创建，为所有后续的监控数据提供一个参考点。

在我们看来，Exadata 的首选 GUI 是 Oracle Enterprise Manager 12c (OEM12c)，并安装了 Diagnostic and Tuning Pack。以这种方式配置的 OEM12c 可以生成实时 SQL 监控报告。Oracle 将自动监控并行运行的 SQL 语句，也会监控合并 I/O 和 CPU 时间超过五秒的序列化语句。

Oracle 提供了 `GV$SQL`、`GV$SQLSTATS` 和 `GV$SQL_MONITOR` 视图（以及其他视图），允许您从 SQL*Plus 内部生成实时 SQL 监控报告。还提供了 `DBMS_SQLTUNE.REPORT_SQL_MONITOR` 过程，该过程可以为给定的 `sql_id` 生成 HTML 报告。下面是如何调用此过程的示例。

```
select dbms_sqltune.report_sql_monitor(session_id=>&sessid, report_level=>'ALL',type=>'HTML') from dual;
```

`TYPE` 参数可以有三个值之一：TEXT、HTML，或者，如果您的服务器上有活动的互联网连接，则可以是 ACTIVE。第三个值生成的 HTML 报告与 OEM12c 中的屏幕非常相似。

您可以安装 Exadata Storage Server 的 System Monitoring 插件，该插件配置 OEM12c 访问存储单元，返回指标数据，以便可以从与数据库服务器相同的位置进行监控。如果您无法访问 OEM12c，或者未安装 Exadata 存储插件，您仍然可以从命令行监控存储单元。第 9 章 和 第 10 章 都提供了使用命令行实用程序监控存储单元的方法。请参考这些章节以获取更多详细信息和示例。

第 11 章 介绍了存储重新配置，我们认为无论您是否需要在两个主要磁盘组之间重新分配存储，都需要讨论这个主题。正如所讨论的，只需要三个步骤即可删除现有的存储配置。重建该配置则需要一个包含磁盘分区、用户帐户创建、grid disk 创建以及在存储单元和数据库服务器上重新创建存储单元初始化文件的 26 步过程。

准备工作是成功更改 `+DATA_<system name>` 和 `+RECO_<system name>` 磁盘组之间存储分布的关键。作为此过程的一部分，“oracle” 操作系统账户在集群中所有可用的数据库服务器上被删除并重新创建。作为删除 “oracle” 操作系统账户的一部分，“oracle” 在所有可用数据库服务器上的操作系统主目录将被删除并重新创建。在执行任何实际的存储重新配置步骤之前，保留这些主目录是必要的。

一旦存储重新配置步骤成功完成，就需要恢复 “oracle” 操作系统用户主目录，并重新安装和重新配置在重新配置过程中可能丢失的任何 OEM 代理。如果您已将 SCAN 监听器的端口从默认值更改，则可能还需要重新配置它。

理解您需要将数据库迁移到 Exadata 平台，第 12 章 通过提供物理和逻辑迁移方法来解决该任务。请记住，并非所有系统在内存值的写入方式上都是相同的。一些被认为是大端（big-endian），另一些是小端（little-endian）。一个操作系统是其中之一；没有其他选择。

物理方法包括 RMAN、物理备用数据库和可传输表空间。RMAN 可以通过将数据文件转换为正确的字节序格式，让您从不同的平台迁移数据库。当操作系统的字节序设计不同时，RMAN 也与可传输表空间方法一起使用。

逻辑方法包括导出和导入、数据库链接以及复制，无论是使用 Streams 还是使用 Golden Gate。请记住，复制对要迁移的数据类型施加了一些限制。物理迁移方法可能是最佳的。

对象在迁移过程中可能会失效。在检查目标数据库中的无效对象时，生成源数据库中无效对象的列表作为参考是一个好习惯。应在迁移后运行 `$ORACLE_HOME/rdbms/admin/utlrp.sql` 脚本以编译无效对象。

将 Oracle ERP 数据库迁移到 Exadata 可能带来独特的问题。第 13 章 讨论了这些迁移。每个 ERP 数据库必须在其自己的 `Oracle Home` 中运行。不允许 ERP 数据库之间共享 `Oracle Home`。这是一项长期存在的 Oracle 标准。此外，每个数据库都必须有一个专用的监听器。



## 监听器问题可能是最具挑战性的。

首先，你必须决定 SCAN 监听器是继续使用 1521 端口还是必须更换。虽然你可以使用 SCAN 监听器，但在开发/测试集群上这样做并不实际，因为那里很可能运行着多个 ERP 数据库。每个数据库都需要自己的监听器端口，因此也需要自己的监听器。为了保持一致，我们发现最好的做法是不让任何数据库将 SCAN 监听器作为主监听器。

你需要将数据库注册到 SCAN 监听器，这样 `dbca` 命令才能工作。每个 ERP 数据库既会注册到一个专属于该数据库的唯一监听器和端口，也会注册到 SCAN 监听器本身。请记住，`dbca` 用途广泛，包括向数据库添加实例、向 Grid Control OMS 注册数据库以及配置 Database Vault 等等——仅举几例。

务必配置好你的 `$ORACLE_HOME/network/admin`，以确保后续在数据库层运行 Autoconfig 时不会破坏你的 TNS 配置。请将配置详情充分告知任何可能运行 Autoconfig 的人。

正确克隆每个 `ORACLE_HOME`，使其在中央清单中注册。如果未遵循克隆流程，该 Home 将不会被注册，你也无法为其应用季度补丁包。

## 成为 DMA 与否

Exadata 是一种供 DBA 管理的不同系统。在此环境中，某些任务（例如运行 `exachk` 脚本）需要 root 操作系统权限。该脚本可以由系统管理员运行，如果你作为 DBA 管理 Exadata，情况通常如此。然而，相对于 Exadata 出现了一个新的角色，即数据库机器管理员，或 DMA。让我们看看成为 DMA 的真正含义。

除了通常的 DBA 技能外，DMA 还必须熟悉并能够理解在指定系统上的以下管理和监控命令。

在计算节点（数据库节点）上：

*   Linux: `top`, `mpstat`, `vmstat`, `iostat`, `fdisk`, `ustat`, `sar`, `sysinfo`
*   Exadata: `dcli`
*   ASM: `asmcmd`, `asmca`
*   Clusterware: `crsctl`, `srvctl`

在存储服务器/Cell 上：

*   Linux: `top`, `mpstat`, `vmstat`, `iostat`, `fdisk`, `ustat`, `sar`, `sysinfo`
*   Cell 管理: `cellcli`, `cellsrvstat`

成为 DMA 还包括一些与成为 DBA 无关的其他职责领域。表 14-1 总结了 DMA 的职责领域。

**表 14-1. DMA 职责**

| 技能 | 百分比 |
| --- | --- |
| 系统管理员 | 15 |
| 存储管理员 |   0 |
| 网络管理员 |   5 |
| 数据库管理员 | 60 |
| Cell 管理员 | 20 |

“百分比”列表示整个 Exadata 系统需要此知识的比例，如你所见，如果你曾是 11g RAC 管理员，那么你已具备成为 DMA 所需技能的 60%。成为 DMA 所需的其余技能并不难以学习和掌握。我们已经介绍了你需要的 Cell 管理员命令（`cellcli`、`dcli`），这为你提供了 80% 的技能集。你可能需要的网络命令包括 `ifconfig`、`iwconfig`、`netstat`、`ping`、`traceroute` 和 `tracepath`。在某些时候，你可能还需要 `ifup` 和 `ifdown` 来启用或禁用网络接口，尽管使用这些命令不会是常规操作。以下示例展示了如何启用 `eth0` 接口。

```
# ifup eth0
```

成为一名 DMA 似乎是一项艰巨的任务，但实际上并没有那么难。这确实需要一种稍微不同的心态，因为你现在审视和管理的是整个系统，而不仅仅是数据库。你的 Exadata 系统仍然需要专门的系统管理员和网络管理员，因为作为 DMA，你不会负责这些资源的配置，也不会负责打补丁和固件升级。DMA 本质上是通过承担这些资源日常提供的任务来协助这些专门的管理员。

## 不懂，可以查

IT 世界变化非常迅速，因此不能期望你无所不知。然而，你可以知道在哪里找到所需的信息。以下资源可协助你作为 DBA 或 DMA 在 Exadata 上的旅程。

*   `http://tahiti.oracle.com`—在线 Oracle 文档站点
*   `www.oracle.com/us/products/database/exadata/overview/index.html`—Oracle 技术网站 Exadata 资源
*   `www.tldp.org/LDP/GNU-Linux-Tools-Summary/html/c8319.htm`—Linux 网络命令参考
*   `www.yolinux.com/TUTORIALS/LinuxTutorialSysAdmin.html`—Linux 系统管理教程
*   `https://support.oracle.com`—My Oracle Support 网站
*   `http://blog.tanelpoder.com/category/exadata/`—Tanel Poder 的 Exadata 博客，来自 Exadata 性能调优专家
*   `http://kevinclosson.wordpress.com/kevin-closson-index/exadata-posts/`—Kevin Closson 的 Exadata 文章，关于存储层的绝佳资源
*   `http://arup.blogspot.com/`—Arup Nanda 的博客，你可以在其中搜索与 Exadata 相关的文章
*   `www.enkitec.com/`—Enkitec 博客，来自一家专注于 Exadata 并主办年度 Exadata 主题会议的公司
*   `http://jonathanlewis.wordpress.com/?s=exadata`—Jonathan Lewis 的优秀 Exadata 文章

这些是我们遇到问题时经常查阅的资源。你可能会找到其他你喜欢的，但这些是很好的起点。

## 需知事项

本章并非 Exadata 的速成课程；其目的是回顾我们已涵盖的领域，为你这段旅程带来一些视角。当你开始这段探索之旅时，Exadata 可能只是一个笼罩在神秘面纱中、被人们以敬畏语气提及的名字。随着你通读本书，你迈出的每一步都将 Exadata 带出迷雾，并希望通过事实和实例取代炒作和谣言，使其变得清晰，以便你现在能更好地承担起 Exadata DBA 或 DMA 的职责。

