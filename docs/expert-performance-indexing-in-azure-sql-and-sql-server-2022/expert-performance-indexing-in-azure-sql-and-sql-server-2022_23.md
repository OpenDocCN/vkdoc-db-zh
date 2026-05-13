# 数据库索引与存储结构

## 索引基础概念

### 数据类型与存储

数据库处理多种数据类型，包括结构化数据、半结构化数据和非结构化数据。大型对象（`LOBs`）的存储涉及特殊的页面类型。`varbinary(max)` 数据是一种常见的大型对象存储形式。

数据库的物理存储单元是**页**（`Pages`）。页是数据存储的基本单位，大小通常为 `8 KB`。页的类型包括数据页、索引页、`LOB` 页、`GAM` 页、`SGAM` 页、`PFS` 页、`DCM` 页和 `BCM` 页等。页的分配与组织构成了 `B-tree` 结构和堆结构。

区（`Extents`）是页的集合，用于管理空间分配。区分为混合区和统一区。

### 索引结构

索引是提升查询性能的关键物理设计结构（`PDSs`）。主要的索引类型包括：
- **堆**（`Heaps`）：无聚集索引的表。
- **聚集索引**（`Clustered indexes`）：将数据行存储在叶子层。
- **非聚集索引**（`Non-clustered indexes`）：包含指向数据行的指针。
- **列存储索引**（`Columnstore index`）：为数据分析工作负载优化。
- **全文索引**（`Full-text index`）：用于全文搜索（`FTS`）。
- **空间索引**（`Spatial index`）：用于地理空间数据类型，如 `GEOMETRY` 和 `GEography`。
- **哈希索引**（`Hash index`）：内存优化表中使用。
- **筛选索引**（`Filtered indexes`）：包含 `WHERE` 子句的索引。
- **包含列索引**（`Included columns`）：在非聚集索引中添加非键列。
- **索引视图**（`Indexed views`）：物化的视图。

## 索引类型与特性

### 聚集索引
聚集索引决定了表中数据的物理存储顺序。每个表只能有一个聚集索引。其叶子节点包含数据行本身。设计时需考虑聚集键（`clustering key`）的选择，例如使用 `GUID` 模式（属性为 `ever-increasing`）或标识列（`identity column`）模式。聚集索引有助于实现页的预排序（`pre-sorted pages`）。

### 非聚集索引
非聚集索引包含索引键值和指向数据行的指针（`Forwarded record pointer`）。其结构包含一个背部指针（`Back pointer`）。非聚集索引支持多种模式：
- **覆盖索引模式**（`Covering index pattern`）：通过包含列（`INCLUDE` 子句）避免键查找（`Key lookup`）。
- **多列模式**（`Multiple column pattern`）：在多个列上创建索引。
- **筛选索引模式**（`Filtered indexes pattern`）：使用 `WHERE` 子句索引部分数据。
- **索引交集模式**（`Index intersection pattern`）：使用多个索引的交集。
- **搜索列模式**（`Search columns pattern`）：针对 `WHERE` 子句中的列创建索引。

非聚集索引的设计需要权衡，例如与聚集索引的对比（`*vs*. covering index patterns`）。

### 列存储索引
列存储索引（`Columnstore structures`）适用于数据仓库和分析查询。它分为聚集列存储索引（`clustered`）和非聚集列存储索引（`non-clustered`）。其优势在于高效的压缩（`row/page/columnstore compression`）和批量加载（`minimally logged bulk load process`）。核心结构包括：
- **行组**（`Row groups`）：数据的逻辑分段。
- **删除位图**（`Delete bitmap`）：跟踪已删除的行。
- **`deltastore`**：用于存储新插入或更新的行，直到它们被压缩到列存储中。

### 其他索引特性
- **填充因子**（`Fill factor`）：控制页填充的程度，以减少页拆分（`Page splits`）。
- **分区**（`Partitioning`）：将大表或索引分成更小的部分。
- **唯一索引**（`Unique index`）：确保索引键的唯一性。
- **主键**（`Primary keys`）：通常是聚集索引的候选键。

## 索引操作与维护

### 创建与修改
索引的定义语言（`DDL`）包括：
- `CREATE INDEX` 语句
- `ALTER INDEX` 语句
- `DROP INDEX` 语句

创建索引时可以使用 `SORT_IN_TEMPDB` 选项。可恢复索引操作（`resumable index operations`）可通过 `sys.index_resumable_operations` 监控。

### 碎片整理
索引碎片（`Fragmentation`）会影响 `I/O` 性能。碎片类型包括内部碎片和外部碎片。维护操作包括：
- **重新组织**（`Reorganize`）：在线操作，清理页碎片。
- **重新生成**（`Rebuild`）：可以联机或脱机进行，创建新的索引副本。在维护计划中，可以使用 `Reorganize Index Task` 和 `Rebuild Index Task`。

### 填充操作
大容量加载（`Bulk-loading`）操作（如 `BULK_LOGGED` 恢复模型下的操作）可能产生碎片。数据加载和更新（`UPDATE operation`）操作也会导致页拆分和转发记录（`Forwarded record process`）。

## 统计与监控

### 统计信息
统计信息（`Statistics`）是查询优化器的依据。包括列级统计信息（`Column-level statistics`）和索引统计信息。统计信息包含直方图（`Histogram`）、密度向量（`Density vector`）等。
- **创建与更新**：可以自动创建（`auto-update`）或手动更新（`Update Statistics` 命令）。对于内存优化表（`Memory-optimized tables`），统计信息需要单独管理。
- **查看命令**：`DBCC SHOW_STATISTICS`。
- **相关函数**：`STATISTICS_DATE` 函数。
- **相关 `DMO`**：`sys.dm_db_stats_properties`, `sys.dm_db_stats_histogram`, `sys.dm_db_incremental_stats_properties`。

### 系统视图与动态管理对象（`DMOs`）
监控和发现索引及统计信息依赖于系统视图和 `DMOs`。

**关键目录视图**：
- `sys.indexes`
- `sys.index_columns`
- `sys.columns`
- `sys.dm_db_index_physical_stats`
- `sys.dm_db_index_usage_stats`
- `sys.dm_db_index_operational_stats`
- `sys.foreign_key_columns`
- `sys.dm_db_missing_index_details`
- `sys.dm_db_stats_properties`

**操作统计信息**：通过 `sys.dm_db_index_operational_stats` 可以查看 `I/O` 统计信息、闩锁争用（`latch contention`）、锁争用（`locking contention`）和 `LOB` 访问情况。关键列包括 `leaf_ghost_count`, `forwarded_fetch_count`, `range_scan_count`, `singleton_lookup_count`, `page_lock_count`, `row_lock_count`, `page_latch_wait_count` 等。

**使用情况统计信息**：通过 `sys.dm_db_index_usage_stats` 可以查看用户查找（`user_lookups`）、用户扫描（`user_scans`）、用户搜索（`user_seeks`）等操作，帮助识别未使用的索引。

### 性能监控
监控索引和查询性能涉及多种 `DMVs` 和性能计数器（`Performance counters`）。

**关键性能计数器**（通过 `sys.dm_os_performance_counters` 或 `Performance Monitor`）：
- `Page Life Expectancy/sec (PLE)`
- `Buffer cache hit ratio`
- `Page lookups/sec`
- `Page reads/sec`
- `Page writes/sec`
- `Full Scans/sec`
- `Index Searches/sec`
- `Page Splits/sec`
- `Lock Waits/sec`
- `Lock Wait Time`
- `Deadlocks/sec`

**等待统计信息**：通过 `sys.dm_os_wait_stats` 分析性能瓶颈，常见等待类型包括 `PAGEIOLATCH_*`, `LCK_M_*`, `CXPACKET`, `CXCONSUMER` 等。

**碎片统计信息**：使用 `sys.dm_db_index_physical_stats` 获取碎片化统计信息，列包括 `avg_fragmentation_in_percent`, `fragment_count`, `avg_fragment_size_in_pages`。

## 最佳实践与工具

### 索引设计最佳实践
- 分析当前工作负载（`current workload`）。
- 合理设置填充因子（`fill factors`）。
- 考虑数据库级和索引级设置。
- 审视主键和索引策略。

### 工具
- **数据库引擎优化顾问（`DTA`）**：用于分析工作负载并提供索引和统计信息建议。
- **查询存储（`Query Store`）**：跟踪查询执行历史、等待统计信息和性能变化，用于识别回归和优化查询。
- **扩展事件（`Extended Events`）**：用于生产环境下的活动监控，比 `SQL Trace` 更轻量高效。一个关键会话示例是 `Waitstats` 分析。
- **`SQL Trace`**：传统跟踪工具，使用 `sp_trace_set`, `sp_trace_setfilter`, `sp_trace_setstatus` 等存储过程管理。
- **`T-SQL` 脚本**：广泛用于索引分析、维护和监控。例如，查询 `sys.dm_db_index_physical_stats` 来分析碎片，或运行 `DBCC IND` 和 `DBCC PAGE` 来深入检查页结构。

### 重要命令与函数
- `DBCC` 命令：`DBCC SHOW_STATISTICS`, `DBCC IND`, `DBCC PAGE`, `DBCC EXTENTINFO`。
- 统计信息函数：`STATISTICS_DATE`。
- 新序列函数：`NEWID()`, `NEWSEQUENTIALID()`。
- 全文属性函数：`FULLTEXTCATALOGPROPERTY`。
- 空间方法：`MakeValid()`, `STDistance()`。
- 智能查询处理（`IQP`）相关功能。

### 最小日志记录与恢复
`BULK_LOGGED` 恢复模型允许最小日志记录（`Minimally Logged (ML)`）操作，可以提升大容量加载性能，但会增加事务日志备份的大小。

### 压缩技术
页压缩（`Page compression`）和行压缩（`Row compression`）可以减少存储空间占用，提升 `I/O` 性能，但会增加 `CPU` 开销。压缩通过字典编码（`Dictionary encoding`）和前缀压缩（`Prefix compression`）实现。
