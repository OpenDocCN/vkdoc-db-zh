# 第 6 章 XML 索引

#### 统计列

`sys.dm_db_column_store_row_group_physical_stats` 中的统计列提供了列存储索引的大量元数据，有助于理解其结构的构建方式。表 **5-26** 中定义的统计信息为管理列存储索引提供了必要的洞察。例如，`total_rows` 和 `deleted_rows` 可用于确定行组中仍然有效的部分。在某些情况下，对行组进行激进的修改可能会在表中留下空行组。当行组小于 `2²⁰` 行时，`state`、`trim_reason` 和 `transition_to_compressed` 信息可以帮助识别行组是如何被压缩的。例如，如果有大量由于 `BULKLOAD` 而关闭的小行组，重建这些行组或修改加载过程以尝试在每次插入操作中插入更多行可能是有价值的。

**表 5-26**

`sys.dm_db_column_store_row_group_physical_stats` 中的统计列

| 列名 | 数据类型 | 描述 |
| --- | --- | --- |
| `state` | `tinyint` | 与 `state_desc` 关联的 ID 号。 |
| `state_desc` | `nvarchar(60)` | 行组状态的描述，可以是 `INVISIBLE`、`OPEN`、`CLOSED`、`COMPRESSED` 或 `TOMBSTONE`。 |
| `total_rows` | `bigint` | 行组中行的完整计数，包括任何被标记为删除的行。 |
| `deleted_rows` | `bigint` | 行组中被标记为要删除的行数。 |
| `size_in_bytes` | `bigint` | 行组的大小（字节）。 |
| `trim_reason` | `tinyint` | 与 `trim_reason_desc` 关联的 ID 号。 |
| `trim_reason_desc` | `nvarchar(60)` | 描述压缩行组行数少于一百万行最大值的原因，可以是 `NO_TRIM`、`BULKLOAD`、`ROERG`、`DICTIONARY SIZE`、`MEMORY LIMITATION`、`RESIDUAL ROW GROUP`、`STATS MISMATCH` 或 `SPILLOVER`。 |
| `transition_to_compressed_state` | `tinyint` | 与 `transition_to_compressed_state_desc` 关联的 ID 号。 |
| `transition_to_compressed_state_desc` | `nvarchar(60)` | 描述行组如何从增量存储转换为行组，包括 `NOT APPLICABLE`、`INDEX BUILD`、`TUPLE MOVER`、`REORG NORMAL`、`REORG FORCED`、`BULKLOAD` 或 `MERGE`。 |
| `has_vertipaq_optimization` | `bit` | 布尔值，标识压缩过程中是否使用了 Vertipaq 优化。此功能大大提高了列存储压缩的效率。 |
| `generation` | `bigint` | 与此行组关联的行组代。 |
| `created_time` | `datetime2` | 创建此行组时的时钟时间。 |
| `closed_time` | `datetime2` | 关闭此行组时的时钟时间。 |

### 列存储操作统计信息

列存储索引操作统计信息由 DMO `sys.dm_db_column_store_row_group_operational_stats` 提供。此 DMO 也返回列存储索引中每个行组的一行。如果表已分区，则将返回每个分区的每个行组一行。

#### 头列

`sys.dm_db_column_store_row_group_operational_stats` 的头列类似于列存储物理统计 DMO 的头列。不同之处在于没有对增量存储的引用。此 DMO 返回列 `object_id`、`index_id`、`partition_number` 和 `row_group_id`。这些列在表 **5-27** 中定义。

**表 5-27**

`sys.dm_db_column_store_row_group_operational_stats` 中的头列

| 列名 | 数据类型 | 描述 |
| --- | --- | --- |
| `object_id` | `int` | 定义索引的表或视图的 ID |
| `index_id` | `int` | 索引的 ID |
| `partition_number` | `int` | 索引或堆内的 1 基分区号 |
| `row_group_id` | `bigint` | 行组的 ID |

#### 统计列

在表 **5-28** 中定义的列中，包含了关于行组扫描次数、删除位图被扫描次数以及列存储分区被扫描次数的详细信息。这些详细信息识别了与索引的其余部分相比，每个行组被访问的频率，以及删除位图被查询的频率，描绘了数据如何写入每个行组。

此外，还可以测量对行组可访问性产生负面影响的锁和阻塞事务，从而深入了解何时存在需要解决的潜在 I/O 或事务瓶颈。

**表 5-28**

`sys.dm_db_column_store_row_group_operational_stats` 中的统计列

| 列名 | 数据类型 | 描述 |
| --- | --- | --- |
| `scan_count` | `int` | 自上次 SQL 重启以来扫描行组的次数。 |
| `delete_buffer_scan_count` | `int` | 使用删除缓冲区确定此行组中已删除行的次数。这包括访问内存中的哈希表和底层的 B 树。 |
| `index_scan_count` | `int` | 扫描列存储索引分区的次数。对于分区中的所有行组，此值相同。 |
| `rowgroup_lock_count` | `bigint` | 自上次 SQL 重启以来，此行组锁请求的累计计数。 |
| `rowgroup_lock_wait_count` | `bigint` | 自上次 SQL 重启以来，数据库引擎在此行组锁上等待的累计次数。 |
| `rowgroup_lock_wait_in_ms` | `bigint` | 自上次 SQL 重启以来，数据库引擎在此行组锁上等待的累计毫秒数。 |

## 总结

本章广泛介绍了 SQL Server 中与索引相关的统计信息。这些信息为深入了解 SQL Server 中索引这一广泛主题打开了一扇窗口。在接下来的章节中，将利用这些信息，通过查看已捕获的统计数据并使用它们来改进索引创建、使用和维护的各个方面。

