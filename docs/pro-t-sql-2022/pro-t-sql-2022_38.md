# 分区切换

当将数据从非分区表切换到分区表的某个分区时，必须首先验证非分区表中的所有数据是否能够存在于分区表的对应分区中。为此，需要向非分区表添加一个检查约束。上面的检查约束确认了 `dbo.CustomerOrderActive` 表中的所有数据都是在 2022 年 1 月 1 日之前创建的。这与 `dbo.CustomerOrderActive` 表中第一个分区的日期范围相同。

> **注意**
> 如果由于存在超出分区日期范围的数据而导致创建检查约束失败，你将无法使用分区切换将数据从非分区表移动到分区表。同样，如果你只想将非分区表中的部分数据移动到分区表，或者想将非分区表中的所有数据移动到目标表中一个非空的分区，你也将无法使用分区切换。

一旦添加了检查约束，就可以使用以下 T-SQL（参考 清单 17-14）将非分区表 `dbo.CustomerOrderActive` 中的所有数据切换到 `dbo.CustomerOrderHistory` 表的第一个分区。

```sql
ALTER TABLE dbo.CustomerOrderActive
SWITCH TO dbo.CustomerOrderHistory PARTITION 1;
```
**清单 17-14**
**从非分区表切换到分区表的分区切换**

与之前的操作一样，一旦分区成功切换，你就可以执行以下 T-SQL（参考 清单 17-15）来确认每个受影响表中的记录数。

```sql
SELECT COUNT(*)
FROM dbo.CustomerOrderActive;
SELECT COUNT(*)
FROM dbo.CustomerOrderHistory;
```
**清单 17-15**
**非分区表和分区表中的记录数**

这些查询的结果符合预期。`dbo.CustomerOrderActive` 表中剩余 0 条记录，而 `dbo.CustomerOrderNonPartition` 表中有 802,821 条记录，如表 17-4 所示。

**表 17-4**
**每个表的记录数统计**

| 表名 | 记录数 |
| --- | --- |
| `dbo.CustomerOrderActive` | 0 |
| `dbo.CustomerOrderNonPartition` | 802,821 |

这确认了所有记录已从 `dbo.CustomerOrderActive` 切换到 `dbo.CustomerOrderNonPartition`。

你将切换分区以重置演示。在将分区切换回去之前，你可以运行一个额外的查询来验证 `dbo.CustomerOrderNonPartition` 中包含这些记录的分区。清单 17-16 展示了用于验证这些结果的查询。

```sql
SELECT tbl.[name] AS TableName,
sch.[name] AS PartitionScheme,
fnc.[name] AS PartitionFunction,
prt.partition_number,
fnc.[type_desc],
rng.boundary_id,
rng.[value] AS BoundaryValue,
prt.[rows]
FROM sys.tables tbl
INNER JOIN sys.indexes idx
ON tbl.[object_id] = idx.[object_id]
INNER JOIN sys.partitions prt
ON idx.[object_id] = prt.[object_id]
AND idx.index_id = prt.index_id
INNER JOIN sys.partition_schemes AS sch
ON idx.data_space_id = sch.data_space_id
INNER JOIN sys.partition_functions AS fnc
ON sch.function_id = fnc.function_id
LEFT JOIN sys.partition_range_values AS rng
ON fnc.function_id = rng.function_id
AND rng.boundary_id = prt.partition_number
WHERE tbl.[name] = 'CustomerOrderHistory'
AND idx.[type] <= 1
ORDER BY prt.partition_number;
```
**清单 17-16**
**用于查看 CustomerOrderHistory 表分区信息的查询**

运行此 T-SQL 代码后，你会发现所有 802,821 条记录都按预期在第一个分区中。这表明你已成功将数据从非分区表切换到分区表。如果你想将分区切换回去，需要运行 清单 17-17 中所示的 T-SQL。

```sql
ALTER TABLE dbo.CustomerOrderActive
DROP CONSTRAINT CK_MaxDateCreated;
ALTER TABLE dbo.CustomerOrderHistory
SWITCH PARTITION 1 TO dbo.CustomerOrderActive;
```
**清单 17-17**
**从当前分区表切换回原始非分区表**

在将分区切换回去之前，你需要移除 `dbo.CustomerOrderActive` 表上的检查约束，以便表结构与 `dbo.CustomerOrderHistory` 匹配。如果表结构不匹配，你将无法切换分区。这一要求是分区切换比将数据插入目标表快得多的原因之一。

移除此约束后，你就可以成功地将数据从 `dbo.CustomerOrderHistory` 切换回 `dbo.CustomerOrderActive`。

如果你重新运行了 清单 17-9 和 清单 17-10 中的 T-SQL，你将把数据移回 `dbo.CustomerOrderHistory` 表。将数据从 `dbo.CustomerOrderActive` 切换到 `dbo.CustomerOrderHistory` 后，`dbo.CustomerOrderHistory` 表中将有 802,821 条记录。完成后，我们就可以开始在两个分区表之间实现分区切换了。在开始分区切换过程之前，你应该执行 清单 17-18 来确认每个表中的记录数。

```sql
SELECT COUNT(*)
FROM dbo.CustomerOrderHistory;
SELECT COUNT(*)
FROM dbo.CustomerOrderHistoryArchive;
```
**清单 17-18**
**分区表中的记录数**

上面的查询结果显示了 `dbo.CustomerOrderHistory` 和 `dbo.CustomerOrderHistoryArchive` 表的行数。这些行数统计如表 17-5 所示。

**表 17-5**
**每个表的记录数统计**

| 表名 | 记录数 |
| --- | --- |
| `dbo.CustomerOrderHistory` | 802,821 |
| `dbo.CustomerOrderHistoryArchive` | 0 |

你可以确认 `dbo.CustomerOrderHistory` 表有 802,821 条记录，而 `dbo.CustomerOrderHistoryArchive` 表有 0 条记录。既然知道了切换分区前的记录数，你就可以按照 清单 17-19 所示，将 `dbo.CustomerOrderHistory` 的第一个分区切换到 `dbo.CustomerOrderHistoryArchive` 的第一个分区。

```sql
ALTER TABLE dbo.CustomerOrderHistory SWITCH PARTITION 1
TO dbo.CustomerOrderHistoryArchive PARTITION 1;
```
**清单 17-19**
**从分区表切换到分区表**

此分区切换完成后，你可以再次验证 `dbo.CustomerOrderHistory` 和 `dbo.CustomerOrderHistoryArchive` 中的行数。清单 17-20 展示了用于验证这些行数的 T-SQL。

```sql
SELECT COUNT(*)
FROM dbo.CustomerOrderHistory;
SELECT COUNT(*)
FROM dbo.CustomerOrderHistoryArchive;
```
**清单 17-20**
**分区表中的记录数**

这些查询的结果显示，`dbo.CustomerOrderHistory` 表中没有剩余记录。表 17-6 也显示所有 802,821 条记录现在都在 `dbo.CustomerOrderHistoryArchive` 表中。

**表 17-6**
**每个表的记录数统计**

| 表名 | 记录数 |
| --- | --- |
| `dbo.CustomerOrderHistory` | 0 |
| `dbo.CustomerOrderHistoryArchive` | 802,821 |

此表表明分区已成功切换。如果你希望将分区切换回原始分区表，可以执行 清单 17-21 中的 T-SQL。



### 在两个分区表之间切换分区

```sql
ALTER TABLE dbo.CustomerOrderHistoryArchive SWITCH PARTITION 1
TO dbo.CustomerOrderHistory PARTITION 1;
```
*清单 17-21：从分区表切换回分区表*

此 T-SQL 语句将把 `dbo.CustomerOrderHistoryArchive` 表中第一个分区的所有数据移回 `dbo.CustomerOrderHistory` 表的第一个分区。清单 17-22 展示了关于 `dbo.CustomerOrderHistoryArchive` 表分区的信息。

```sql
SELECT tbl.[name] AS TableName,
sch.[name] AS PartitionScheme,
fnc.[name] AS PartitionFunction,
prt.partition_number,
fnc.[type_desc],
rng.boundary_id,
rng.[value] AS BoundaryValue,
prt.[rows]
FROM sys.tables tbl
INNER JOIN sys.indexes idx
ON tbl.[object_id] = idx.[object_id]
INNER JOIN sys.partitions prt
ON idx.[object_id] = prt.[object_id]
AND idx.index_id = prt.index_id
INNER JOIN sys.partition_schemes AS sch
ON idx.data_space_id = sch.data_space_id
INNER JOIN sys.partition_functions AS fnc
ON sch.function_id = fnc.function_id
LEFT JOIN sys.partition_range_values AS rng
ON fnc.function_id = rng.function_id
AND rng.boundary_id = prt.partition_number
WHERE tbl.[name] = 'CustomerOrderHistoryArchive'
AND idx.[type] <= 1
ORDER BY prt.partition_number;
```
*清单 17-22：用于查看 CustomerOrderHistoryArchive 表分区信息的查询*

查看此信息后，您就可以确认已成功地在两个分区表之间切换了分区。

本节介绍了如何在两个非分区表、一个非分区表与一个分区表以及两个分区表之间切换分区。虽然这些过程分别都很有用，但结合它们以适应您的业务需求可能会更好。在下一节中，您将通过一个示例来了解如何跨多个表切换分区，以帮助您完成数据保留和归档流程。下一节中的演示将应用这些概念，使您能够拥有一个用于活动应用程序数据的表、一个用于报表的表、一个用于存储合规性数据的表以及一个用于快速从数据库中删除数据的表。

### 滑动窗口分区

本示例的目标是展示如何在多个表（包括分区表和非分区表）之间切换分区。在此示例中，您将在几个表中使用一组预先加载的小型样本数据，如表 17-7 所示。

*表 17-7：用于管理客户订单的各表记录数*

| 表名 | 记录数 |
| --- | --- |
| `dbo.CustomerOrderActive` | 1 |
| `dbo.CustomerOrderHistory` | 3 |
| `dbo.CustomerOrderHistoryArchive` | 12 |
| `dbo.CustomerOrderPurge` | 0 |

`dbo.CustomerOrderActive` 表中的单条记录对应 2023 年 4 月。`dbo.CustomerOrderHistory` 表中的三条记录分别对应 2023 年 1 月、2 月和 3 月。在 `dbo.CustomerHistory` 表中，有 12 条记录，分别对应过去 4 到 15 个月中每一个月的数据。

此过程假定您处于 2023 年 5 月初。此时，您将准备把 `dbo.CustomerOrderActive` 表的数据更新为 2023 年 5 月。按照 `dbo.CustomerOrderHistory` 表存储最近三个月历史数据的模式，此表将被更新为保存 2023 年 2 月至 2023 年 4 月的数据，而 `dbo.CustomerOrderHistoryArchive` 表将包含从 2022 年 2 月到 2023 年 1 月的数据。

在此示例中，首先将 `dbo.CustomerOrderHistoryArchive` 表中最旧的一个月数据移动到 `dbo.CustomerOrderPurge` 表中。在此之前，您可以通过执行清单 17-23 中的 T-SQL 来验证 `dbo.CustomerOrderHistoryArchive` 表每个分区的行数。

```sql
SELECT SCHEMA_NAME(tbl.[schema_id]) AS SchemaName,
tbl.[name] AS TableName,
pt.partition_number AS PartitionNumber,
pt.[rows] AS NumberRecords,
rv.[value] AS BoundaryValue,
CASE WHEN ISNULL(rv.[value], rv2.[value]) IS NULL
THEN 'N/A'
ELSE
CASE WHEN fnc.boundary_value_on_right = 0
AND rv2.[value] IS NULL THEN '>='
WHEN fnc.boundary_value_on_right = 0
THEN '>'
ELSE '>='
END + ' '
+ ISNULL(CONVERT(varchar(64), rv2.value),
'Min Value') + ' ' +
CASE fnc.boundary_value_on_right
WHEN 1 THEN 'and <'
ELSE 'and <='
END
+ ' ' + ISNULL(CONVERT(varchar(64), rv.value),
'Max Value')
END AS TextComparison
FROM sys.tables AS tbl
INNER JOIN sys.indexes AS idx
ON tbl.[object_id] = idx.[object_id]
INNER JOIN sys.partitions AS pt
ON idx.[object_id] = pt.[object_id]
AND idx.index_id = pt.index_id
INNER JOIN sys.partition_schemes AS pch
ON idx.data_space_id = pch.data_space_id
INNER JOIN sys.partition_functions AS fnc
ON pch.function_id = fnc.function_id
LEFT JOIN sys.partition_range_values AS rng
ON fnc.function_id = rng.function_id
AND rng.boundary_id = pt.partition_number
LEFT JOIN sys.partition_range_values AS rv
ON fnc.function_id = rv.function_id
AND pt.partition_number = rv.boundary_id
LEFT JOIN sys.partition_range_values AS rv2
ON fnc.function_id = rv2.function_id
AND pt.partition_number - 1= rv2.boundary_id
WHERE tbl.[name] = 'CustomerOrderHistoryArchive'
AND idx.[type] <= 1
ORDER BY tbl.[name], pt.partition_number;
```
*清单 17-23：合并超过 15 个月的客户订单分区*

此查询不仅显示了切换分区所需的分区号，还显示了行数和分区逻辑。对于此表，表 17-8 显示了此查询的结果。

*表 17-8：分区切换前的 CustomerOrderHistoryArchive 表*



