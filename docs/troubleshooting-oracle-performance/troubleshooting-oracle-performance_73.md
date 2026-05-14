# 附录 A：脚本文件

| **文件名** | **描述** |
| --- | --- |
| `clustering_factor.sql` | 此脚本创建了一个函数，用于说明如何计算聚簇因子。 |
| `comparing_object_statistics.sql` | 此脚本展示了如何将当前的对象统计信息与历史中待处理的以及备份表中存储的对象统计信息进行比较。 |
| `cpu_cost_column_access.sql` | 此脚本展示了查询优化器在访问列时估算的 CPU 成本，该成本取决于列在表中的位置。 |
| `dbms_stats_job_10g.sql` | 此脚本展示了在创建 10*g*数据库期间安装和调度的、旨在自动收集对象统计信息的实际作业配置。 |
| `dbms_stats_job_11g.sql` | 此脚本展示了在创建 11*g*数据库期间安装和调度的、旨在自动收集对象统计信息的实际作业配置。 |
| `delete_histogram.sql` | 此脚本展示了如何在不修改其他统计信息的情况下删除单个直方图。 |
| `lock_statistics.sql` | 此脚本展示了被锁定的对象统计信息的工作方式和行为。 |
| `mreadtim_lt_sreadtim.sql` | 此脚本展示了当`mreadtim`小于或等于`sreadtim`时查询优化器执行的修正。 |
| `object_statistics.sql` | 此脚本提供了所有对象统计信息的概览。 |
| `pending_object_statistics.sql` | 此脚本展示了如何在发布新的对象统计信息之前，使用待定统计信息对其进行测试。 |
| `system_stats_history_job.sql` | 此脚本可用于创建一个表和一个作业，以存储多日来负载统计信息的演变情况。 |
| `system_stats_history.sql` | 此脚本用于从由`system_stats_history_job.sql`脚本创建的历史表中提取负载统计信息。输出可导入电子表格`system_stats_history.xls`。 |
| `system_stats_history.xls` | 此 Excel 电子表格可用于计算平均值，并绘制图表以显示通过`system_stats_history.sql`脚本提取的负载统计信息的趋势。 |

## 第五章：配置查询优化器

表 A-4 中列出的文件可供第五章下载使用。

### 表 A-4. 第五章文件

| **文件名** | **描述** |
| --- | --- |
| `assess_dbfmbrc.sql` | 此脚本用于测试初始化参数`db_file_multiblock_read_count`取不同值时多块读取的性能。其目的是确定能提供最佳性能的该参数值。 |
| `bug5015557.sql` | 此脚本展示了初始化参数`optimizer_features_enable`不仅会禁用常规特性，也会禁用错误修复。 |
| `dynamic_sampling_levels.sql` | 此脚本展示了利用动态采样的查询示例，采样级别从 1 到 4。 |
| `optimizer_index_caching.sql` | 此脚本展示了初始化参数`optimizer_index_caching`的工作方式和缺点。 |
| `optimizer_index_cost_adj.sql` | 此脚本展示了设置初始化参数`optimizer_index_cost_adj`的缺点。 |
| `optimizer_secure_view_merging.sql` | 此脚本展示了初始化参数`optimizer_secure_view_merging`的工作方式和缺点。 |

## 第六章：执行计划

表 A-5 中列出的文件可供第六章下载使用。

### 表 A-5. 第六章文件

| **文件名** | **描述** |
| --- | --- |
| `dbms_xplan_output.sql` | 此脚本生成一个示例输出，包含`dbms_xplan`函数提供的主要信息。 |
| `display.sql` | 此脚本展示了如何在`dbms_xplan`包中使用`display`函数的示例。 |
| `display_awr.sql` | 此脚本展示了如何在`dbms_xplan`包中使用`display_awr`函数的示例。 |
| `display_cursor.sql` | 此脚本展示了如何在`dbms_xplan`包中使用`display_cursor`函数的示例。 |
| `display_cursor_9i.sql` | 此脚本显示存储在库缓存中的游标的执行计划。游标通过地址、哈希值和子编号标识。 |
| `execution_plans.sql` | 此脚本展示了执行计划所包含的不同类型的操作。 |
| `parent_vs_child_cursors.sql` | 此脚本展示了父游标与其子游标之间的关系。 |
| `restriction_not_recognized.sql` | 此脚本生成的输出用于展示如何通过检查实际基数来识别低效的执行计划。 |
| `wrong_estimations.sql` | 此脚本生成的输出用于展示如何通过观察错误的估算来识别低效的执行计划。 |

## 第七章：SQL 调优技术

表 A-6 中列出的文件可供第七章下载使用。

### 表 A-6. 第七章文件



| 文件名 | 描述 |
| --- | --- |
| `all_rows.sql` | 此脚本展示了如何通过 SQL 配置文件将优化器模式从 `rule` 切换到 `all_rows`。 |
| `baseline_automatic.sql` | 此脚本展示了查询优化器如何自动捕获 SQL 计划基线。 |
| `baseline_from_sqlarea1.sql` | 此脚本展示了如何从库缓存手动加载 SQL 计划基线。光标通过与之关联的 SQL 语句文本来标识。 |
| `baseline_from_sqlarea2.sql` | 此脚本展示了如何从库缓存手动加载 SQL 计划基线。光标通过与之关联的 SQL 语句的 SQL 标识符来标识。 |
| `baseline_from_sqlarea3.sql` | 此脚本展示了如何在不更改应用程序代码的情况下对其进行调优。这里使用了 SQL 计划基线来实现这一目的。 |
| `baseline_from_sqlset.sql` | 此脚本展示了如何从 SQL 调优集手动加载 SQL 计划基线。 |
| `baseline_upgrade_10g.sql` | 此脚本展示了如何在 Oracle Database 10*g* 上创建和导出 SQL 调优集。它与 `baseline_upgrade_11g.sql` 脚本一起使用，用于展示在升级到 Oracle Database 11*g* 时如何稳定执行计划。 |
| `baseline_upgrade_11g.sql` | 此脚本展示了如何将 SQL 调优集导入并加载到 SQL 计划基线中。它与 `baseline_upgrade_10g.sql` 脚本一起使用，用于展示在从 Oracle Database 10*g* 升级到 Oracle Database 11*g* 时如何稳定执行计划。 |
| `clone_baseline.sql` | 此脚本展示了如何在两个数据库之间迁移 SQL 计划基线。 |
| `clone_sql_profile.sql` | 此脚本展示了如何创建 SQL 配置文件的副本。 |
| `depts_wo_emps.sql` | 此脚本用于生成示例中的执行计划，这些示例在“更改访问结构”一节中引用。 |
| `exec_env_trigger.sql` | 此脚本创建一个配置表和一个数据库触发器，以在会话级别控制执行环境。 |
| `first_rows.sql` | 此脚本展示了如何通过 SQL 配置文件将优化器模式从 `all_rows` 切换到 `first_rows`。 |
| `object_stats.sql` | 此脚本展示了如何通过 SQL 配置文件向查询优化器提供对象统计信息。 |
| `opt_estimate.sql` | 此脚本展示了如何通过 SQL 配置文件增强查询优化器执行的基数估计。 |
| `outline_editing.sql` | 此脚本展示了如何手动编辑存储的概要。 |
| `outline_edit_tables.sql` | 此脚本创建编辑私有概要所需的工作表和公共同义词。 |
| `outline_from_sqlarea.sql` | 此脚本展示了如何通过引用共享池中的光标手动创建存储的概要。 |
| `outline_from_text.sql` | 此脚本展示了如何手动创建存储的概要以及如何管理和使用它。 |
| `outline_with_ffs.sql` | 此脚本测试存储的概要是否能够覆盖初始化参数 `optimizer_features_enable` 的设置。 |
| `outline_with_hj.sql` | 此脚本测试存储的概要是否能够覆盖初始化参数 `hash_join_enabled` 的设置。 |
| `outline_with_rewrite.sql` | 此脚本测试存储的概要是否能够覆盖初始化参数 `query_rewrite_enabled` 的设置。 |
| `outline_with_star.sql` | 此脚本测试存储的概要是否能够覆盖初始化参数 `star_transformation_enabled` 的设置。 |
| `tune_last_statement.sql` | 此脚本用于指示 SQL 调优顾问分析当前会话执行的最后一条 SQL 语句。分析完成后，将显示分析报告。 |

### 第 8 章

第 8 章可供下载的文件列于表 A-7。

**表 A-7.** 第 8 章的文件

| 文件名 | 描述 |
| --- | --- |
| `bind_variables.sql` | 此脚本展示了绑定变量如何以及何时导致光标共享。 |
| `bind_variables_peeking.sql` | 此脚本展示了绑定变量窥探的利弊。 |
| `lifecycle.sql` | 此脚本展示了隐式和显式光标管理之间的区别。 |
| `long_parse.sql` | 此脚本用于执行持续约一秒钟的解析。它还展示了如何创建存储概要以避免如此长的解析。 |
| `long_parse.zip` | 此压缩包包含由 `long_parse.sql` 脚本执行生成的两个跟踪文件。对于每个跟踪文件，还提供了由 TKPROF 和 TVD$XTAT 生成的输出文件。 |
| `ParsingTest1.c`、`ParsingTest2.c` 和 `ParsingTest3.c` | 这些文件分别包含测试用例 1、2 和 3 的 C (OCI) 实现。 |
| `ParsingTest1.cs` 和 `ParsingTest2.cs` | 这些文件分别包含测试用例 1 和 2 的 C# (ODP.NET) 实现。 |
| `ParsingTest1.java`、`ParsingTest2.java` 和 `ParsingTest3.java` | 这些文件分别包含测试用例 1、2 和 3 的 Java 实现。 |
| `ParsingTest1.sql`、`ParsingTest2.sql` 和 `ParsingTest3.sql` | 这些脚本分别提供测试用例 1、2 和 3 的 PL/SQL 实现。 |
| `ParsingTest1.zip`、`ParsingTest2.zip` 和 `ParsingTest3.zip` | 这些压缩包包含由测试用例 1、2 和 3 的 Java 实现执行生成的多个跟踪文件。对于每个跟踪文件，还提供了由 TKPROF 和 TVD$XTAT 生成的输出文件。 |

### 第 9 章

第 9 章可供下载的文件列于表 A-8。

**表 A-8.** 第 9 章的文件

| 文件名 | 描述 |
| --- | --- |
| `access_structures_1.sql` | 此脚本比较了不同访问结构在读取单行时的性能。它用于生成图 9-3 中的插图。 |
| `access_structures_1000.sql` | 此脚本比较了不同访问结构在读取数千行时的性能。它用于生成图 9-4 中的插图。 |
| `conditions.sql` | 此脚本展示了如何使用 B 树和位图索引来应用多种类型的条件。 |
| `fbi.sql` | 此脚本展示了函数索引的一个示例。 |
| `full_scan_hwm.sql` | 此脚本展示了全表扫描会读取高水位线以下的所有块。 |
| `index_full_scan.sql` | 此脚本展示了全索引扫描的示例。 |
| `iot_guess.sql` | 此脚本展示了陈旧的猜测对逻辑读的影响。 |
| `linguistic_index.sql` | 此脚本展示了语言索引的一个示例。 |
| `pruning_composite.sql` | 此脚本展示了应用于复合分区表的分区裁剪的几个示例。 |
| `pruning_hash.sql` | 此脚本展示了应用于哈希分区表的分区裁剪的几个示例。 |
| `pruning_list.sql` | 此脚本展示了应用于列表分区表的分区裁剪的几个示例。 |
| `pruning_range.sql` | 此脚本展示了应用于范围分区表的分区裁剪的几个示例。 |
| `read_consistency.sql` | 此脚本展示了由于读一致性，逻辑读的数量可能会如何变化。 |
| `row_prefetching.sql` | 此脚本展示了由于行预取，逻辑读的数量可能会如何变化。 |

### 第 10 章



## 第十章

表 A-9 中列出的文件可供第十章下载使用。

### 表 A-9. 第十章相关文件

| **文件名** | **描述** |
| --- | --- |
| `block_prefetching.sql` | 此脚本展示了数据块和索引块的块预取。 |
| `hash_join.sql` | 此脚本提供了数个哈希连接的示例。 |
| `join_elimination.sql` | 此脚本提供了一个连接消除的示例。 |
| `join_trees.sql` | 此脚本为每种类型的连接树提供了一个示例。 |
| `join_types.sql` | 此脚本为每种类型的连接提供了一个示例。 |
| `nested_loops_join.sql` | 此脚本提供了数个嵌套循环连接的示例。 |
| `merge_join.sql` | 此脚本提供了数个合并连接的示例。 |
| `outer_join.sql` | 此脚本提供了数个外连接的示例。 |
| `outer_to_inner.sql` | 此脚本提供了一个将外连接转换为内连接的示例。 |
| `pwj.sql` | 此脚本提供了数个分区智能连接的示例。`pwj_performance.sql` 此脚本用于比较不同分区智能连接的性能。它被用于生成图 10-15 中的图表。 |
| `star_transformation.sql` | 此脚本提供了数个星型转换的示例。 |
| `query_unnesting.sql` | 此脚本提供了数个子查询非嵌套的示例。 |

## 第十一章

表 A-10 中列出的文件可供第十一章下载使用。

### 表 A-10. 第十一章相关文件

| **文件名** | **描述** |
| --- | --- |
| `array_interface.sql, array_interface.c, ArrayInterface.cs`, 和 `ArrayInterface.java` | 这些脚本提供了使用 PL/SQL、OCI、JDBC 和 ODP.NET 实现数组接口的示例。 |
| `ArrayInterfacePerf.java` | 此脚本展示了数组接口能大幅提高大批量加载的响应时间。它被用于生成图 11-13。 |
| `atomic_refresh.sql` | 此脚本可用于重现 Oracle9*i* 中的 bug 3168840。该错误会导致在刷新单个物化视图时刷新操作无法正确进行。 |
| `dpi.sql` | 此脚本展示了与缓冲区缓存利用、重做和撤销生成以及触发器和外键支持相关的直接路径插入行为。 |
| `dpi_performance.sql` | 此脚本用于比较直接路径插入与常规插入。它被用于生成图 11-10 和图 11-11。 |
| `makefile.mk` | 这是我用来编译示例中给出的 C 程序的 makefile。 |
| `mv.sql` | 此脚本展示了物化视图的基本概念。 |
| `mv_refresh_log.sql` | 此脚本展示了基于物化视图日志的快速刷新如何工作。 |
| `mv_refresh_pct.sql` | 此脚本展示了基于分区更改跟踪的快速刷新如何工作。 |
| `mv_rewrite.sql` | 此脚本展示了数个查询重写的示例。 |
| `px_ddl.sql` | 此脚本展示了数个并行 DDL 语句的示例。 |
| `px_dml.sql` | 此脚本展示了数个并行 DML 语句的示例。 |
| `px_dop1.sql` | 此脚本展示了初始化参数 `parallel_min_percent` 的影响。 |
| `px_dop2.sql` | 此脚本展示了提示不会强制查询优化器使用并行处理，它们只是覆盖默认的并行度。 |
| `px_query.sql` | 此脚本展示了数个并行查询的示例。 |
| `px_tqstat.sql` | 此脚本展示了动态性能视图 `v$pq_tqstat` 显示何种信息。 |
| `result_cache_query.sql` | 此脚本展示了一个利用服务器结果缓存的查询示例。 |
| `result_cache_plsql.sql` | 此脚本展示了一个实现 PL/SQL 函数结果缓存的 PL/SQL 函数示例。 |
| `row_prefetching.sql, row_prefetching.c, RowPrefetching.cs, RowPrefetching.java` | 这些脚本提供了使用 PL/SQL、OCI、JDBC 和 ODP.NET 实现行预取的示例。 |
| `RowPrefetchingPerf.java` | 此脚本展示了行预取能大幅提高检索大量行的查询的响应时间。它被用于生成图 11-12。 |

### 第十二章

表 A-11 中列出的文件可供第十二章下载使用。

### 表 A-11. 第十二章相关文件

| **文件名** | **描述** |
| --- | --- |
| `buffer_busy_waits.sql` | 此脚本展示了一个导致大量 `buffer busy waits` 事件的处理示例。 |
| `buffer_busy_waits.zip` | 此文件包含“块争用”一节中使用的跟踪文件以及 TKPROF 和 TVD$XTAT 的输出。 |
| `column_order.sql` | 此脚本展示了列在行中的位置决定了访问它所需的处理量。该脚本用于生成图 12-2 中表示的值。 |
| `data_compression.sql` | 此脚本展示了由于数据压缩，I/O 密集型处理的性能可能会得到提升。 |
| `reverse_index.sql` | 此脚本展示了反向索引上的范围扫描不能用于应用基于范围条件的限制。 |
| `wrong_datatype.sql` | 此脚本展示了使用错误数据类型会对查询优化器的决策产生负面影响。 |

* * *

1. 参见 [`http://www.centos.org`](http://www.centos.org) 获取更多信息。



该请求被拒绝，因为它被认为是高风险。



### A

`accept_sql_profile procedure`,, `2nd`
`接受 SQL 配置文件过程`,, `第 2 处`
`验收测试`,
`访问路径`
`访问路径的提示`,
`识别/解决低效访问路径`, –`第 2 处`
`强选择性、SQL 语句与访问路径`, –`第 2 处`
`弱选择性、SQL 语句与访问路径`, –`第 2 处`
`访问谓词`,
`更改访问结构`,
`操作名称属性`,
`addBatch 方法`,
`地址列`,
`管理 SQL 管理对象权限`,, `第 2 处`
`高级队列进程 (Qnnn)`,
`顾问权限`,
`聚合`,, `第 2 处`
`全行优化`,
`all_rows 提示`,
`Allround Automations 的 PL/SQLDeveloper`,
`更改任意大纲权限`,
`更改任意 SQL 配置文件权限`,
`ALTER INDEX RENAME 语句`,
`ALTER INDEX 语句、锁定对象统计信息与`,
`ALTER MATERIALIZED VIEW 语句`,, `第 2 处`
`ALTER OUTLINE 语句`,
`ALTER SESSION 语句`
`初始化参数与`,
`会话级执行环境更改与`,
`ALTER SYSTEM 语句、初始化参数与`,
`ALTER TABLE 语句、数据压缩与`,
`alter_sql_plan_baseline 过程`,
`alter_sql_profile 过程`,
`分析引擎 (JProbe)`,
`分析路线图`,
`ANALYZE INDEX 语句`,
`ANALYZE 语句`, `第 2 处`
`ANALYZE TABLE... 语句`,
`AND-EQUAL 执行计划操作`,
`ANSI 连接语法`,, `第 2 处`
`反连接子查询，转换为常规连接`,
`反连接`,, `第 2 处`
`Apache 日志服务项目`,
`APIs (应用程序编程接口)`, –`第 2 处`
`应用程序基准测试`,
`应用程序代码`
`分析路线图与`,
`应用程序代码的插桩`,
`应用程序代码的性能剖析分析`, –`第 2 处`
`应用程序设计`,
`应用监控工具`,
`应用程序编程接口 (APIs)`, –`第 2 处`
`多层架构`,
`A-Rows 列`,, `第 2 处`, `第 3 处`
`数组接口`, –`第 2 处`
`ArrayBindCount 属性`,
`到达率`,
`ASM 相关进程 (Onnn)`,
`ASSOCIATE STATISTICS 语句`,
`自动诊断库`,
`自动优化器`,
`自动工作负载库 (AWR)`,, `第 2 处`, `第 3 处`
`aux_stats$`,
`AWR (自动工作负载库)`,, `第 2 处`, `第 3 处`


### B

`background_dump_dest 参数`, #chapter03.html#page_70, 2nd

`备份表`, #chapter04.html#page_109, 2nd
- `创建/删除`, #chapter04.html#page_166
- `对象统计信息与`, #chapter04.html#page_146, 2nd
- `系统统计信息与`, #chapter04.html#page_119

`向后兼容性`, `ANALYZE 语句与`, #chapter04.html#page_109

`基础表`, #chapter11.html#page_461

`basicfile 存储方法`, #chapter12.html#page_534

`批量更新`, #chapter11.html#page_524

`基准测试`, `应用/合成`, #chapter04.html#page_111

`最佳实践`
- 关于 `绑定变量`, #chapter02.html#page_30
- 关于 `数据类型选择`, #chapter12.html#page_533 –2nd
- 关于通过 `dbms_stats 包` 进行 `对象统计信息收集`, #chapter04.html#page_163

`二进制比较`, #chapter09.html#page_392

`BINARY_DOUBLE 数据类型`, #chapter12.html#page_533

`BINARY_FLOAT 数据类型`, #chapter12.html#page_533

`绑定变量`
- `优点/缺点`, #chapter02.html#page_22 –2nd
- `API 与`, #chapter08.html#page_329
- `最佳实践`, #chapter02.html#page_30
- `EXPLAIN PLAN 语句与`, #chapter06.html#page_198
- `逐步提升`, #chapter02.html#page_23
- `窥视与`, #chapter02.html#page_25, 2nd
- `预处理语句与`, #chapter08.html#page_317, 2nd

`位串数据类型`, #chapter12.html#page_534

`BITMAP AND`/`BITMAP OR`/`BITMAP MINUS` 执行计划操作, #chapter06.html#page_226

`BITMAP CONVERSION FROM ROWIDS` 操作, #chapter09.html#page_402

`BITMAP CONVERSION TO ROWIDS` 操作, #chapter09.html#page_382

`BITMAP INDEX RANGE SCAN` 操作, #chapter09.html#page_386

`BITMAP INDEX SINGLE VALUE` 操作, #chapter09.html#page_382, 2nd

`位图索引`, #chapter09.html#page_371, 2nd –3rd
- `对比 B 树索引`, #chapter09.html#page_378
- `复合`, #chapter09.html#page_400
- `压缩与`, #chapter09.html#page_398
- `等值条件与`, #chapter09.html#page_382
- `IS NULL 条件与`, #chapter09.html#page_384
- `最小/最大函数与`, #chapter09.html#page_389
- `范围条件与`, #chapter09.html#page_386

`位图连接索引`, #chapter10.html#page_455

`BITMAP KEY ITERATION` 执行计划操作, #chapter06.html#page_227

`bitmap_merge_area_size 参数`, #chapter05.html#page_190, 2nd

`BLOB 数据类型`, #chapter12.html#page_534

`块争用`, #chapter12.html#page_539 –2nd
- `识别`, #chapter12.html#page_540
- `解决`, #chapter12.html#page_543 –2nd

`块预取`, #chapter10.html#page_422

`块范围粒度`, #chapter11.html#page_491

`阻塞/非阻塞` 执行计划操作, #chapter06.html#page_223, 2nd

`块`. *参见* `数据块`

`bloom-filter 裁剪`, #chapter09.html#page_362

`分支块`, #chapter04.html#page_133

`行的广播分布`, #chapter11.html#page_493

`B 树索引`, #chapter09.html#page_369, 2nd –3rd
- `对比 位图索引`, #chapter09.html#page_378
- `位图执行计划`, #chapter09.html#page_402
- `复合`, #chapter09.html#page_395 –2nd
- `压缩与`, #chapter09.html#page_398
- `等值条件与`, #chapter09.html#page_381
- `IS NULL 条件与`, #chapter09.html#page_383
- `最小/最大函数与`, #chapter09.html#page_389
- `范围条件与`, #chapter09.html#page_384

`错误 3168840`, #chapter11.html#page_474

`错误 5759631`, #chapter07.html#page_283

`构建输入`, #chapter10.html#page_434

`BULK COLLECT 语句`, #chapter11.html#page_519

`灌木树`, #chapter10.html#page_412

`性能优化的业务视角`, #chapter01.html#page_10


