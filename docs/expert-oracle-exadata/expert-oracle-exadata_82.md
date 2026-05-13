# 17. 摒弃一些我们曾自以为知道的东西

## 摘要

希望本章有助于缓解许多人在首次为其 Exadata 打补丁时所感到的焦虑。与其他任何事情一样，随着补丁应用次数的增加，这个过程会越来越熟悉。尽管在 Exadata 的早期阶段，补丁发布是**成败难料的尝试**，但在过去几年里，补丁版本已经稳定得多。这可归因于多种因素——平台的成熟、更好的测试以及不断增长的客户群。应用 Exadata 补丁时，最好的建议是多次阅读文档并保持**十足的耐心**。在应用 QDPE 时仔细查看 `OPatch` 日志，会揭示出先前安装的补丁被回滚时的警告。在检查 Exadata 存储服务器补丁的内部情况时，你可能会看到看似错误的信息。请相信 `patchmgr` 脚本能够筛选出哪些可以忽略，哪些是实际错误。如果确实发生故障，`patchmgr` 会给出适当的警告，表明发生了故障。与其他任何软件版本一样，在补丁发布后等待一小段时间再应用是个好主意。该平台能够为堆栈中的每个组件提供一套统一的补丁，这是其他厂商在 Oracle 数据库方面无法提供的。

Oracle 在 Exadata 上运行时，可以做一些与在非 Exadata 平台上运行时非常不同的事情。Exadata 提供的优化旨在采取一种不同于 Oracle 传统遵循的方法。这种改变意味着你需要以不同的思路来处理某些问题。这并不是说一切都不同——恰恰相反。事实上，大多数基本原则保持不变。毕竟，在 Exadata 上运行的数据库软件与在其他平台上运行的是相同的。但有些事情就是不一样。正如你在第 13 章读到的，你可以直接将你的数据库 1:1 地部署到 Exadata 上。根据部署类型，这可能是可以的。然而，如果你有兴趣从投资中获取最大收益，你可能需要退一步，审视一下你可以做些什么来进一步优化 Exadata 上的数据库。由于与标准非 Exadata 部署相比，Exadata 上有一些不同之处，因此本章值得在你迁移的第二阶段阅读。在本章中，我们将重点讨论在 Exadata 上运行数据库时，应如何改变我们的方法。

## 两种系统的故事

我们思考运行在 Exadata 上的系统的方式，在很大程度上取决于执行的工作负载。面向在线事务处理（OLTP）的工作负载倾向于让我们专注于使用 `Smart Flash Cache` 来加速小型物理读取。但坦白说，这类工作负载无法利用 Exadata 提供的大部分性能优势。面向数据仓库（DW）的工作负载倾向于抓住一切机会利用 `Smart Scans`，并尝试使用所有可用资源（例如，存储和数据库层上的 CPU 资源）。这是 Exadata 内置优势能产生最大影响的地方。不幸的是，大多数系统都同时表现出 DW 和 OLTP 工作负载的特征。这些“混合工作负载”是最困难的，而且具有讽刺意味的是，也是最常见的。它们之所以最常见，是因为很难找到一个面向 OLTP 的系统不包含某些产生长时间运行、对吞吐量敏感的查询的报告组件。同样常见的是，面向 DW 的工作负载具有类似 OLTP 的缓慢滴入式数据流或类似的抽取、加载、转换（`ELT`）处理需求。这些组合系统需要最困难的思考过程，因为根据具体问题，你必须不断重置管理性能的方法。它们也很困难，因为 DW 工作负载通常受控制吞吐量的数据流动态限制，而 OLTP 系统通常受延迟问题限制。因此，对于混合工作负载，你基本上需要训练自己评估每种情况，并将其归类为延迟敏感型或吞吐量敏感型。这种对工作负载特征的评估应在开始任何分析之前完成。

如果你认为那是一个有趣的挑战，那么第三个变量也可能被引入。如今部署的 Exadata 系统很少只托管一个应用程序。虽然在前两代产品推出时，将关键应用程序放置在 Exadata 上相当常见，但 `Smart Flash Cache` 以及磁盘容量的增加，加上 Intel 提供的日益增长的 CPU 算力，使得 Exadata 成为一个有趣的整合平台。这一点已被许多用户认识到，因此，你经常需要在同一硬件上安排多个数据库的需求——这些数据库可能具有混合的工作负载需求——而不是一个具有混合工作负载需求的数据库。值得庆幸的是，Oracle 提供了工具来处理这种情况。

## 面向 OLTP 的工作负载

尽管关于在 Exadata 上运行 OLTP 工作负载没有太多可说的，但对此类系统仍有几点需要牢记。由于 Exadata 运行标准的 Oracle 数据库软件，因此你无需大幅调整基本方法。



### Exadata 智能闪存缓存（ESFC）

在处理 `OLTP` 工作负载时，Exadata 的关键组件是 Exadata 智能闪存缓存（`ESFC`），它可以显著减少小规模读取的磁盘访问时间。因此，验证 `ESFC` 是否正常工作至关重要。对于此类工作负载，您还应该预期大部分物理 `I/O` 操作是由 `ESFC` 满足的。通过查看平均单块读取时间可以相当容易地推断出这一点。如果是由闪存缓存满足的，单块读取应大约需要 0.5 毫秒。相比之下，如果是由实际磁盘读取满足的，单块读取平均大约需要 5 毫秒。标准的 `AWR` 报告会提供平均值和等待事件的直方图。如果平均单块读取时间远高于 1 毫秒范围，您应该寻找系统性问题——例如闪存卡未工作或关键表被定义为永不缓存（使用 `CELL_FLASH_CACHE NONE` 语法）。还应检查 `I/O` 资源管理。还应使用直方图来验证平均值是否掩盖了大量的异常值。以下是用于检查闪存卡状态的 `cellcli` 语法：

```
CellCLI> list flashcache detail

      name:                   dm01cel03_FLASHCACHE
      cellDisk:               FD_00_dm01cel03,FD_01_dm01cel03,FD_02_dm01cel03,
                              FD_03_dm01cel03,FD_04_dm01cel03,FD_05_dm01cel03,
                              FD_06_dm01cel03,FD_07_dm01cel03,FD_08_dm01cel03,
                              FD_09_dm01cel03,FD_10_dm01cel03,FD_11_dm01cel03,
                              FD_12_dm01cel03,FD_13_dm01cel03,FD_14_dm01cel03,FD_15_dm01cel03
      creationTime:           2010-03-22T17:39:46-05:00
      id:                     850be784-714c-4445-91a8-d3c961ad924b
      size:                   365.25G
      status:                 critical
```

请注意，此单元上的 `status` 属性是 `critical`。正如您可能预料的那样，这不是一件好事。在这个特定系统上，闪存缓存基本上已经自行禁用。我们注意到它是因为单块读取时间已经变慢。此示例来自早期版本的 `cellsrv`。后期版本包含更多信息。以下是 `cellsrv` 12.1.2.1.0 的一个示例：

```
CellCLI> list flashcache detail

         name:                   enkx4cel01_FLASHCACHE
         cellDisk:               FD_04_enkx4cel01,FD_06_enkx4cel01,FD_11_enkx4cel01,
                                 FD_02_enkx4cel01,FD_13_enkx4cel01,FD_12_enkx4cel01,
                                 FD_00_enkx4cel01,FD_14_enkx4cel01,FD_03_enkx4cel01,
                                 FD_09_enkx4cel01,FD_10_enkx4cel01,FD_15_enkx4cel01,
                                 FD_08_enkx4cel01,FD_07_enkx4cel01,FD_01_enkx4cel01,
                                 FD_05_enkx4cel01
         creationTime:           2015-01-19T21:33:37-06:00
         degradedCelldisks:
         effectiveCacheSize:     5.8193359375T
         id:                     3d415a32-f404-4a27-b9f2-f6a0ace2cee2
         size:                   5.8193359375T
         status:                 normal
```

注意新属性 `degradedCelldisks`。还要注意，此单元上的闪存缓存显示 `status` 为 `normal`。监控存储软件行为将在第 12 章中更详细地介绍。

#### 可扩展性

处理 `OLTP` 工作负载时要记住的另一点是，Exadata 平台提供了卓越的可扩展性。从半机架升级到全机架会使数据库层和存储层的 CPU 数量都增加一倍。`ESFC` 的数量以及可用内存也增加一倍。这使得 Exadata 对于许多系统能够以近乎线性的方式进行扩展。

为了说明这一点，我们可以分享一个小轶事。在研讨会上介绍 Exadata 智能扫描功能时，我们过去常常针对一个禁用了所有 Exadata 优化的表运行查询，最初是在 X2-2 四分之一机架上进行纯硬盘访问。当 X4-2 推出时，我们有幸使用了一个 X4-2 半机架。抛开 CPU 差异不谈，使用三个单元与七个单元之间的差异是惊人的。再加上表的大部分内容通过 `ESFC` 提供服务，您可以想象响应时间会急剧下降。为了回到我们习惯的耗时水平，我们不得不显著增加表的大小。这只是 Exadata 在不需要对应用程序本身进行任何更改的情况下扩展得非常好的一个例子。

### 写密集型 OLTP 工作负载

写密集型工作负载是面向 `OLTP` 的系统的一个子集。有些系统只是不断地执行单行插入然后提交，或者换一种说法，采用“逐行”（slow-by-slow (row-by-row)）的方法。这些系统通常受到提交速度的限制，而提交速度通常取决于写入日志文件的速度。这是 Exadata 在默认直写模式下运行 `ESFC` 时与其他平台处于相对公平竞争环境的一个领域。对于因写入操作而遇到瓶颈的系统，没有重大增强功能能使 Exadata 的运行速度快上几个数量级。闪存日志记录可以在这里提供帮助。从 `cellsrv` 11.2.2.4 开始，Exadata 会同时将重做日志写入磁盘和闪存设备。最先完成的那个“胜出”，允许日志写入器继续处理，而另一个写入则继续完成。此外，随着 X5 硬件代的推出，磁盘控制器上的缓存已翻倍至 1GB。

智能闪存日志记录并非万能药，对于这些类型的系统来说，已经建立的调优方法——例如最小化提交——要合适得多。将逐行处理改为基于集合的处理所能带来的性能提升会让您感到惊讶，甚至无需考虑硬件！许多网站，最著名的可能是 Tom Kyte 的 Ask Tom，都有大量参考资料，展示每次完成某些工作后就进行逐行处理并提交的方式对性能是次优的。

日志写入器并不是唯一渴望低延迟 `I/O` 性能的组件。在 Exadata 版本 11.2.3.2.1 之前，使用直写模式的智能闪存缓存是唯一可用的选项。从 11.2.3.2.1 开始，可以使用回写模式的闪存缓存。顺便说一下，切换到回写模式并不会消除对闪存日志的需求——切换后您仍然会发现它被定义。启用回写模式可以提高写密集型工作负载的性能，正如您在第 5 章中看到的那样。但是，您应该使用前述第 5 章中关于闪存缓存的信息，仔细评估使用回写缓存是否值得。

### 面向数据仓库的工作负载

Exadata 最初设计用于加速针对海量数据的长时间运行查询。因此，数据仓库是我们需要改变一些基本思维模式的地方，这不足为奇。当然，主要技术之一是不断寻找机会让 Exadata 优化发挥作用。这意味着确保应用程序能够利用智能扫描。


#### 启用智能扫描

在处理面向数据仓库（DW）的工作负载时，需要牢记的最重要的概念是：长运行语句通常应当被卸载执行。以下是需要遵循的步骤：

确定是否正在使用智能扫描。如果未使用，请进行调整以确保其被使用。

这些要点看似如此显然，以至于几乎无需赘述。Exadata 中内置的大部分优化功能仅在启用智能扫描时才生效。你需要做出的首要思维转变之一，就是训练自己持续思考智能扫描是否被恰当地使用。这意味着你需要深入了解哪些语句（或语句的哪些部分）可以被卸载，并能够判断语句是否正在被卸载。本书已广泛涵盖了智能扫描的要求以及一些可用于验证其是否在执行的技术。但为了避免遗漏，这里再次说明。

本质上，智能扫描的发生需要满足两个主要前提条件。第一，优化器必须选择对表或物化视图进行全表扫描，或者优化器必须选择对索引进行快速全索引扫描。请注意，智能扫描不仅限于查询甚至子查询。当会影响大量行时，优化器也可能选择对 `DELETE` 和 `UPDATE` 语句使用全扫描。但是，如果你的应用程序正在这样做，你或许应该考虑修改它，采用类似截断后重建的方式。俗话说得好，“视情况而定”。

智能扫描的第二个要求是，扫描必须使用直接路径读取机制来执行。请注意，在描述第二个要求时，有意未提及优化器。这是因为优化器并不决定是否使用直接路径读取。这是一个在执行计划确定后做出的启发式决策。因此，它不会被 `explain plan` 或其他性能相关工具直接显示。在实践中，这意味着验证第一个要求是否满足很容易，但验证第二个要求则更具挑战性。

在大多数 Exadata 实施中，相当高比例的长运行查询会被卸载。你可以通过选择所有平均运行时间超过某秒数，或平均逻辑 I/O 值大于某个合理值的语句（从 `v$sql` 中，或者如果你有相应许可的话，从 AWR 表 `DBA_HIST_SQLSTAT` 中），来检查你的长运行 SQL 语句中有多少百分比被卸载了。实际上，逻辑 I/O 是一个更好的度量标准，因为一些被卸载的语句可能运行得非常快，可能无法满足你的最小时间标准，这会让你对被卸载语句的百分比产生扭曲的看法。以下是一个示例（请注意，脚本包含在在线代码仓库中）：

```
SQL> @offload_percent

Enter value for sql_text:

Enter value for min_etime:

Enter value for min_avg_lio: 500000

TOTAL  OFFLOADED OFFLOADED_%
-------------------------------
13         11      84.62%

SQL> /

Enter value for sql_text: SELECT%

Enter value for min_etime:

Enter value for min_avg_lio: 500000

TOTAL  OFFLOADED OFFLOADED_%
-------------------------------
11         11        100%
```

此列表使用了 `offload_percent.sql` 脚本，该脚本计算当前位于共享池中且已被卸载的语句的百分比。它最初用于评估所有逻辑 I/O 超过 500,000 的语句。第二次运行时，调查范围仅限于以 `SELECT` 单词开头的语句。在下一个列表中，你可以看到另一个脚本 (`fsxo.sql`) 的输出，该脚本允许你查看实际贡献于上一列表中计算的 `OFFLOAD_%` 的语句：

```
SQL> @fsxo

Enter value for sql_text:

Enter value for sql_id:

Enter value for min_etime:

Enter value for min_avg_lio: 500000

Enter value for offloaded:

SQL_ID          EXECS  AVG_ETIME OFFLOAD IO_SAVED_% SQL_TEXT
----------------------------------------------------------------------------
0bvt5z48t18by      1        .10 Yes         100.00 select count(*) from skew3 whe
0jytfr1y0jdr1      1        .09 Yes         100.00 select count(*) from skew3 whe
12pkwt8zjdhbx      1        .09 Yes         100.00 select count(*) from skew3 whe
2zbt555tg123s      2       4.37 Yes          71.85 select /*+ parallel (a 8) */ a
412n404yughsy      1        .09 Yes         100.00 select count(*) from skew3 whe
5zruc4v6y32f9      5      51.13 No             .00 DECLARE job BINARY_INTEGER :=
6dx247rvykr72      1        .10 Yes         100.00 select count(*) from skew3 whe
6uutdmqr72smc      2      32.83 Yes          71.85 select /* avgskew3.sql */ avg(
7y09dtyuc4dbh      1       2.87 Yes          71.83 select avg(pk_col) from kso.sk
b6usrg82hwsa3      5      83.81 No             .00 call dbms_stats.gather_databas
fvx3v0wpvxvwt      1      11.05 Yes          99.99 select count(*) from skew3 whe
gcq9a53z7szjt      1        .09 Yes         100.00 select count(*) from skew3 whe
gs35v5t21d9yf      1       8.02 Yes          99.99 select count(*) from skew3 whe
13 rows selected.
```

`fsxo.sql` 脚本提供了与 `offload_percent.sql` 脚本相同的限制因素，即最小平均耗时和最小平均逻辑 I/O。它还允许你选择性地将语句限制为仅那些被卸载的或未被卸载的。请参考脚本获取更多详细信息，并请记住这些技术也可以应用于 AWR 记录的数据，以便进行历史分析。

在下一节中，我们将讨论一些可能使你启用智能扫描的努力复杂化的问题。

### 可能削弱智能扫描的问题

有几种常见的编码“技术”要么会完全禁用智能扫描，要么会导致其效果远低于应有水平。其中一些技术只是不良实践，无论你是否在 Exadata 平台上。另一些在非 Exadata 平台上不会带来如此显著的性能损失，但在 Exadata 上运行时，它们会阻止存储软件发挥其全部潜能。当开发工作位于非 Exadata 平台上时，这是一个常见的观察结果。这样的系统使得开发高性能应用程序变得异常困难。确保应用程序在非 Exadata 系统上运行良好，很可能使用的技术并不适合在 Exadata 平台上获得最佳性能，因此一旦迁移到 Exadata 上，就需要额外的工作来优化代码。

本书通篇已经讨论了许多此类问题。这里有几个你应该牢记的问题，因为它们在 Exadata 平台上具有根本不同的行为。

#### WHERE 子句中的函数

Oracle 提供了大量可直接在 SQL 语句中使用的函数。正如 第 2 章 所讨论的，并非所有这些函数都是**可下推的**。了解哪些函数**不可下推**非常重要，因为如果在 `WHERE` 子句中使用这些函数，会禁用谓词过滤，而这原本可能极大地减少需要传回数据库层的数据量。显然，自定义编写的 PL/SQL 函数也属于“不可下推”函数的范畴。

这个问题在某种程度上有些反直觉，因为在数据仓库系统中你通常无论如何都会进行全表扫描。在非 Exadata 平台上，对于通过全表扫描执行的语句，在其 `WHERE` 子句中使用函数不会对必须返回的数据量造成太大影响，因为数据库本来就需要将表的所有数据块返回给数据库服务器。然而，在 Exadata 上，使用一个可能禁用谓词过滤的函数可能会导致巨大的性能损失。顺便提一句，在非 Exadata 平台上，在 `WHERE` 子句中使用自定义 PL/SQL 函数通常也是个坏主意，因为处理每一行的 PL/SQL 都需要额外的 CPU，这与 Oracle 提供的、基于 C 代码的优化函数不同。

**注意**

你可以查询 `V$SQLFN_METADATA` 来查看哪些函数是可下推的。

此外，“可下推的”函数也可能带来巨大的性能损失。这里有一个非常简单的示例，展示了在 `WHERE` 子句中使用可下推函数的负面影响：

```
SQL> select /* example001 */ count(*) from SALES where prod_id < 1;

COUNT(*)
----------
0

Elapsed: 00:00:00.75

SQL> select /* example001 */ count(*) from SALES where abs(prod_id) < 1;

COUNT(*)
----------
0

Elapsed: 00:00:34.26

SQL> @fsx4
Enter value for sql_text: %example001%
Enter value for sql_id:

SQL_ID          CHILD OFFLOAD IO_SAVED_%  AVG_ETIME SQL_TEXT
------------- ------ ------- ---------- ---------- ----------------------------------------
7ttw461bngzn0      0 Yes         100.00        .75 select /* example001 */ count(*) from SA
fktc9145xy6qg      0 Yes          99.98      33.81 select /* example001 */ count(*) from SA

Elapsed: 00:00:00.08

SQL> select name, offloadable from v$sqlfn_metadata
  2  where name = 'ABS';

NAME                       OFF
------------------------------ ---
ABS                        YES
ABS                        YES
ABS                        YES

3 rows selected.
```

`ABS()` 是一个可下推函数，但在此特定语句的 `WHERE` 子句中使用时，却导致了性能的大幅下降。如果你已经在自己的环境中跟随此示例操作，可能已经很清楚原因了。解决方案如下：

```
SQL> select name,value from v$statname natural join v$mystat
  2  where name like '%storage index%';

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
cell physical IO bytes saved by storage index                              0

SQL> set timing on
SQL> select /* example001 */ count(*) from SALES where abs(prod_id) < 1;

COUNT(*)
----------
0

Elapsed: 00:00:33.90

SQL> select name,value from v$statname natural join v$mystat where name like '%storage index%';

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
cell physical IO bytes saved by storage index                              0

Elapsed: 00:00:00.00

SQL> select /* example001 */ count(*) from SALES where prod_id < 1;

COUNT(*)
----------
0

Elapsed: 00:00:00.76

SQL> select name,value from v$statname natural join v$mystat where name like '%storage index%';

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
cell physical IO bytes saved by storage index                   1.4517E+11

Elapsed: 00:00:00.00
```

就像普通索引一样，存储索引也会被函数禁用。这并不太令人惊讶，但它同样很容易被忽视。当我们看到全表扫描时，我们已经习惯于不去担心 `WHERE` 子句中可能禁用索引的函数。但 Exadata 的情况不同。


#### 链式行

这是一个非常宽泛的概括，但基本上任何 Oracle 处理操作如果需要额外读取一个数据块才能完成一行数据的提取，就会导致 Exadata 存储软件回退到块传输模式或直通模式。你在前面的章节中多个地方会读到相关内容——特别是第 11 章提供了大部分细节。一个简单的例子是链式行，但还有其他情况也可能导致 Oracle 回退到直通模式。在实践中，这意味着某些在非 Exadata 平台上只会引起轻微延迟的操作，在 Exadata 上可能对性能产生更严重的影响。该问题的主要诊断症状是存在大量单块读等待事件与单元智能扫描等待事件组合出现。在这种情况下，你可能会发现，在更彻底地解决问题之前，对于有问题的语句，暂时不使用卸载功能可能是更好的补救措施。下面是一个示例，展示了当首次执行从包含链式行的表中进行选择时，Oracle 的时间花费情况。此示例是特意设计来夸大问题并使其可复现的，如关于该表的字典信息所示。

```
SQL> select num_rows,chain_cnt,avg_row_len from tabs where table_name = 'CHAINS';

NUM_ROWS  CHAIN_CNT AVG_ROW_LEN
---------- ----------- -----------
600000     600000       20005
1 row selected.

SQL> select segment_name,partition_name,round(bytes/power(1024,3),2) gb
2   from user_segments
3  where segment_name = 'CHAINS';

SEGMENT_NAME                   PARTITION_NAME                             GB
------------------------------ ------------------------------ ----------
CHAINS                                                     12.75

SQL> alter system flush buffer_cache;
System altered.

SQL> select /*+ gather_plan_statistics monitor */ avg(length(col2)) from chains;
AVG(LENGTH(C))
--------------
3990
1 row selected.
```

为了调查执行时间花费在哪里，可以使用以下命令：

```
SQL> select * from table(dbms_xplan.display_cursor);

PLAN_TABLE_OUTPUT
---------------------------------------------------------------------------------------------
SQL_ID  6xpsmzknmkutw, child number 0
-------------------------------------
select /*+ gather_plan_statistics monitor */ avg(length(col2)) from chains
Plan hash value: 1270987893
-------------------------------------------------------------------------------------
| Id  | Operation                  | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT           |        |       |       |   450K(100)|          |
|   1 |  SORT AGGREGATE            |        |     1 |  3990 |            |          |
|   2 |   TABLE ACCESS STORAGE FULL| CHAINS |   600K|  2283M|   450K  (1)| 00:00:18 |
-------------------------------------------------------------------------------------
```

请注意，上述输出显示的是查询预估的时间和返回行数。如果你想要实际的统计数据，可以使用 SQL Monitor（前提是你有相应许可），或者在`DBMS_XPLAN.DISPLAY_CURSOR`中提供`ALLSTATS LAST`作为格式参数。其他有用的工具包括 session snapper 以及第 2 章中提到的`fsx`系列脚本。查询活动会话历史（这同样需要许可）也能提供关于当前正在发生的事情的有趣见解，但请确保你进行了适当的筛选。最终，一个跟踪文件将揭示发生的每一次等待。处理原始跟踪文件后，收集到了以下信息。由于跟踪文件中条目数量庞大，实际处理时间被延长：

`SQL ID: 6xpsmzknmkutw Plan Hash: 1270987893`



