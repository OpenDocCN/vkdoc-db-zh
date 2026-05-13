# 10. Exadata 等待事件

Oracle 数据库是一段高度可监测的代码，并且已保持这种状态相当长一段时间。它通过使用等待事件来跟踪在离散操作上花费的时间量，除非相关会话正在使用 CPU。尽管数据库软件相当复杂，但等待事件分析允许性能分析师确定数据库将时间花在了哪里。许多困难的性能问题可以通过分析来自等待接口的数据来解决，最近还可以通过活动会话历史 (ASH) 来解决。Exadata 的引入导致了几个新等待事件的创建，以支持在该平台上执行的独特操作。本章将重点描述这些新事件以及它们如何与实际执行的活动相关联，同时将它们与非 Exadata 平台上数据库使用的等待事件进行对比。它还将描述一些并非 Exadata 特有但在 Exadata 平台上扮演重要角色的等待事件。

在一些罕见情况下，等待接口的粒度不够细，无法让您弄清楚数据库引擎将时间花在了什么上。如果您使用喜欢的搜索引擎搜索“等待接口不够”，您会找到关于该主题的博客文章，以及 Oracle 维护的会话统计信息如何提供有关会话所做工作的进一步见解。第 11 章将帮助您更好地理解会话统计信息。不过，在大多数情况下，分析等待事件足以对问题进行故障排除。给本章作者的一条非常有用的建议是：不要太早迷失在细节中！

等待事件实际上是一段被计时并被赋予名称的代码段。这些事件所涵盖的代码段通常是离散的操作系统调用，例如单独的 I/O 请求，但有些等待事件涵盖了相当大段的代码。这些事件甚至可能包含其他等待事件的代码段。多年来，等待事件的命名有些不一致，许多事件的名称略有误导性。尽管一些事件名称被公认为可能有误导性，但 Oracle 出于可以理解的原因一直不愿更改它们。Exadata 平台为重命名一些与 I/O 相关的事件提供了借口，并且，正如您稍后将看到的，开发人员抓住这个机会就这么做了。

等待事件在数据库的许多地方被外部化。查看等待事件最常见的方法是查询 V-dollar 视图、跟踪和 SQL Monitor。用于查看等待事件信息的常见 V-dollar 视图是 `V$SESSSION` 或 `V$SESSION_WAIT`。这些视图显示一个会话的当前等待事件，但不显示事件历史记录。10g 中引入的活动会话历史通过采样活动会话并记录相关的信息给性能分析师，从而改善了这一点，但它需要额外的许可证。请始终确保您所做的事情符合您的许可协议。

如果您想获取一个会话的所有等待记录，您可以选择启用 SQL 跟踪。成熟的 SQL 跟踪允许您记录针对数据库发出的 SQL 语句以及所有相关的等待事件。然后，位于数据库诊断目标中的原始跟踪文件会被后处理并转换为更易于人类阅读的格式。


除了 SQL 跟踪，另一种选择是使用 SQL 监控器。值得庆幸的是，在 Exadata 平台上，您不会遇到这个强大工具在技术上不可用的情况（许可问题另当别论）。SQL 监控器在 Oracle 11.1 版本中引入，而这是 Exadata 支持的最低版本。SQL 监控器允许您实时窥探单条 SQL 语句的执行过程。在 SQL 语句执行期间，您可以确切看到 RDBMS 引擎在 SQL 执行的哪个位置、哪个行源上花费时间，如果适用，还能看到等待事件。所有这些信息，甚至在语句仍在执行时就可获取！要能利用这一强大工具，您必须为数据库获得**诊断包和调优包**的许可。支撑 SQL 监控器的底层技术同样是活动会话历史（ASH），正如您刚刚读到的，它是在 Oracle 10g 中引入的。ASH 每秒对数据库活动进行采样，收集关于活动会话的信息，并以一秒为间隔存储约一小时。此后，信息被聚合，每第十个样本被保留在`SYSAUX`表空间的磁盘上。持久化部分被称为**活动负载仓库（AWR）**。AWR 与`STATSPACK`工具的相似之处仅在于它会随时间记录数据库活动。信息的保留期可由用户配置。存储 AWR 信息所需的空间量可能很大，但不应忘记，更多数据使您能更轻松地与过去事件进行比较。像所有事情一样，您需要在磁盘空间使用与将感知到的性能问题与过去事件进行比较的优势之间找到平衡。在我看来，默认的八天保留期远远不够，应该增加。

## Exadata 特有事件

实际上，不存在专属于 Exadata 平台的事件。等待事件是内置于数据库代码中的。由于 Exadata 上的计算节点运行的是标准的 Oracle 数据库软件，所有在调用 Exadata 特有功能时使用的等待事件，在非 Exadata 平台上的数据库中同样可用。但因为 Exadata 功能仅在 Exadata 平台上可用，所以在其他平台上，这些事件永远不会被分配时间。作为证明，请看以下示例，它首先在 Exadata 数据库一体机上，然后在非 Exadata 平台的标准 12c Release 1 数据库上，比较了`V$EVENT_NAME`（它暴露了有效的等待事件）中的事件：

`SQL> select count(1) from v$event_name;`

   `COUNT(1)`

`-----------`

       `1650`

`SQL> select count(1) from v$event_name@lab12c;`

   `COUNT(1)`

`-----------`

       `1650`

`SQL> select name from v$event_name`

  `2  minus`

  `3  select name from v$event_name@lab12c;`

`no rows selected`

因此，事件之间没有差异。这确实使得列出一个"Exadata 专属"事件列表有些困难。不过，事件名称是一个很好的起点。

### “cell”事件

《Oracle Exadata 存储服务器软件用户指南 12c Release 1》提供了一个等待事件表。所有这些事件都以单词"cell"开头。手册列出了十个这样的事件。其中一个（`cell interconnect retransmit during physical read`）实际上仍然不存在。

还有一批名称中包含"cell"的事件未包含在文档中。结合已记录和未记录的事件，就得到了完整的"cell"事件列表。这个列表将是本章的起点。您可以查询`V$EVENT_NAME`来获取该列表，您将得到以下结果。请注意，大多数事件属于某一 I/O 类。等待事件的数量在 12.1.0.1.x 中没有变化；与本章第一版中使用的 11.2.0.2 版本一样，仍然是 17 个事件。

```
SYS:db12c1> select name,wait_class
  2  from v$event_name
  3  where name like 'cell%'
  4  order by name;

NAME                                               WAIT_CLASS
-------------------------------------------------- --------------------
cell list of blocks physical read                  User I/O
cell manager cancel work request                   Other
cell manager closing cell                          System I/O
cell manager discovering disks                     System I/O
cell manager opening cell                          System I/O
cell multiblock physical read                      User I/O
cell single block physical read                    User I/O
cell smart file creation                           User I/O
cell smart flash unkeep                            Other
cell smart incremental backup                      System I/O
cell smart index scan                              User I/O
cell smart restore from backup                     System I/O
cell smart table scan                              User I/O
cell statistics gather                             User I/O
cell worker idle                                   Idle
cell worker online completion                      Other
cell worker retry                                  Other

17 rows selected.
```

Oracle 12.1.0.2 是第一个添加了新的 cell 相关等待事件的版本。这些事件如下：

*   cell external table Smart Scan
*   cell list of blocks read request
*   cell multi-block read request
*   cell physical read no I/O
*   cell single block read request

除了一个例外，其他事件都将在后面的章节中介绍：外部表扫描事件属于另一个 Oracle 产品，此处不作讨论。

接下来的章节将涵盖所有相关的 Exadata 等待事件，以及一些对 Exadata 具有特殊适用性的附加事件。


## 用户 I/O 类中的 Exadata 等待事件

#### 触发事件的执行计划步骤

首先，或许值得一看哪些操作（执行计划步骤）会导致“cell”等待事件的发生。以下查询来自一个运行在 Exadata 系统上的活跃生产系统，针对`DBA_HIST_ACTIVE_SESS_HISTORY`表，展示了 cell 事件以及引发它们的操作。当然，这个列表并非详尽无遗——并非每个事件在每个系统上都可见！

```
SQL> select event, operation,  count(*) from (
  2  select sql_id, event, sql_plan_operation||’ ’||sql_plan_options operation
  3    from DBA_HIST_ACTIVE_SESS_HISTORY
  4    where event like ’cell %’)
  5    group by operation, event
  6    order by 1,2,3
  7  /

EVENT                              OPERATION                                          COUNT(*)
---------------------------------- -------------------------------------------------- ----------
cell list of blocks physical read                                                     62
    DDL STATEMENT                                                                    2
    INDEX FAST FULL SCAN                                                             1
    INDEX RANGE SCAN                                                              3060
    INDEX STORAGE FAST FULL SCAN                                                     7
    INDEX STORAGE SAMPLE FAST FULL SCAN                                             10
    INDEX UNIQUE SCAN                                                             1580
    INSERT STATEMENT                                                                 6
    TABLE ACCESS BY GLOBAL INDEX ROWID                                             151
    TABLE ACCESS BY INDEX ROWID                                                   5458
    TABLE ACCESS BY LOCAL INDEX ROWID                                              131
    TABLE ACCESS STORAGE FULL                                                      183
    TABLE ACCESS STORAGE SAMPLE                                                      2
    TABLE ACCESS STORAGE SAMPLE BY ROWID RAN                                         1
cell multiblock physical read                                                      3220
    DDL STATEMENT                                                                  157
    INDEX FAST FULL SCAN                                                            94
    INDEX RANGE SCAN                                                                 2
    INDEX STORAGE FAST FULL SCAN                                                  6334
    INDEX STORAGE SAMPLE FAST FULL SCAN                                           429
    UNIQUE SCAN                                                                      2
    VIEW ACCESS STORAGE FULL                                                       634
    MAT_VIEW ACCESS STORAGE SAMPLE                                                 56
    TABLE ACCESS BY GLOBAL INDEX ROWID                                               5
    TABLE ACCESS BY INDEX ROWID                                                    484
    TABLE ACCESS BY LOCAL INDEX ROWID                                                 3
    TABLE ACCESS STORAGE FULL                                                    41559
    TABLE ACCESS STORAGE SAMPLE                                                   1763
    TABLE ACCESS STORAGE SAMPLE BY ROWID RAN                                         78
    UPDATE                                                                            4
cell single block physical read                                                  181186
    BUFFER SORT                                                                       1
    CREATE TABLE STATEMENT                                                           67
    DDL STATEMENT                                                                  985
    DELETE                                                                       11204
    DELETE STATEMENT                                                                 6
    FIXED TABLE FIXED INDEX                                                        352
    FOR UPDATE                                                                      27
    HASH GROUP BY                                                                     3
    HASH JOIN                                                                       14
    HASH JOIN RIGHT OUTER                                                             1
    INDEX BUILD NON UNIQUE                                                          80
    INDEX BUILD UNIQUE                                                                6
    INDEX FAST FULL SCAN                                                             9
    INDEX FULL SCAN                                                               1101
    INDEX RANGE SCAN                                                             17597
    INDEX RANGE SCAN (MIN/MAX)                                                       1
    INDEX RANGE SCAN DESCENDING                                                      6
    INDEX SKIP SCAN                                                                691
    INDEX STORAGE FAST FULL SCAN                                                   313
    INDEX STORAGE SAMPLE FAST FULL SCAN                                             72
    INDEX UNIQUE SCAN                                                            30901
    INSERT STATEMENT                                                              5174
    LOAD AS SELECT                                                                 120
    LOAD TABLE CONVENTIONAL                                                       5827
    MAT_VIEW ACCESS STORAGE FULL                                                     3
    MAT_VIEW ACCESS STORAGE SAMPLE                                                   1
    MERGE                                                                           12
    PX COORDINATOR                                                                   1
    SELECT STATEMENT                                                                978
    SORT CREATE INDEX                                                                1
    SORT GROUP BY                                                                     1
    SORT JOIN                                                                         5
    SORT ORDER BY                                                                     2
    TABLE ACCESS BY GLOBAL INDEX ROWID                                            5812
    TABLE ACCESS BY INDEX ROWID                                                   65799
    TABLE ACCESS BY LOCAL INDEX ROWID                                             4591
    TABLE ACCESS BY USER ROWID                                                      464
    TABLE ACCESS CLUSTER                                                            57
    TABLE ACCESS STORAGE FULL                                                     7168
    TABLE ACCESS STORAGE SAMPLE                                                    205
    TABLE ACCESS STORAGE SAMPLE BY ROWID RAN                                         24
    UNION-ALL                                                                        7
    UPDATE                                                                       89353
    UPDATE STATEMENT                                                               367
    WINDOW CHILD PUSHED RANK                                                         2
    WINDOW SORT                                                                      1
    WINDOW SORT PUSHED RANK                                                          1
cell smart file creation                                                           35
    DELETE                                                                            3
    INDEX BUILD NON UNIQUE                                                            5
    LOAD AS SELECT                                                                    3
    LOAD TABLE CONVENTIONAL                                                           1
    UPDATE                                                                            1
cell smart incremental backup                                                     714
cell smart index scan                                                              14
    INDEX STORAGE FAST FULL SCAN                                                    42
    INDEX STORAGE SAMPLE FAST FULL SCAN                                             32
cell smart table scan                                                             163
    MAT_VIEW ACCESS STORAGE FULL                                                     1
    TABLE ACCESS STORAGE FULL                                                    12504
```

同样，这个输出并未展示所有可能的组合，但它应该能让你了解事件的相对频率以及通常引发它们的操作。

## 用户 I/O 类中的 Exadata 等待事件

用户 I/O 类对于 Exadata 而言，无疑是其中最重要的一类。当然，该类别中最有趣的事件是那两个智能扫描事件（`cell smart table scan` 和 `cell smart index scan`）。这些事件记录了 Exadata 提供的主要查询下推优化所消耗的时间，这些优化主要包括谓词过滤、列投影和存储索引的使用。

用户 I/O 类还包含三个被描述为物理 I/O 事件的事件。这三个事件实际上衡量的是使用更为熟悉的多块和单块读取机制进行物理 I/O 所花费的时间——你在非 Exadata 平台上常见到的那些机制，只不过它们的名字被改得更有意义了一些。

最后，还有两个事件似乎并不真正属于用户 I/O 类别。一个与文件空间分配时块的初始化有关。另一个与从存储单元收集统计信息有关。

Oracle 12.1.2.1 和数据库 12.1.0.2 引入了三个与用户 I/O 相关的新次要 cell 事件，这些事件在 12.1.0.1 或 12.1.0.3 中不存在。接下来的几个章节将依次介绍每一个等待事件，首先从智能扫描事件开始。


