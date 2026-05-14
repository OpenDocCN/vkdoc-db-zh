# 数据库分区管理操作示例

| 分区号 | 记录数 | 边界值 | 文本比较 |
| --- | --- | --- | --- |
| 1 | 0 | 2022-01-01 | >= 最小值且 < 2022 年 1 月 1 日 12:00AM |
| 2 | 1 | 2022-02-01 | >= 2022 年 1 月 1 日 12:00AM 且 < 2022 年 2 月 1 日 12:00AM |
| 3 | 1 | 2022-03-01 | >= 2022 年 2 月 1 日 12:00AM 且 < 2022 年 3 月 1 日 12:00AM |
| 4 | 1 | 2022-04-01 | >= 2022 年 3 月 1 日 12:00AM 且 < 2022 年 4 月 1 日 12:00AM |
| 5 | 1 | 2022-05-01 | >= 2022 年 4 月 1 日 12:00AM 且 < 2022 年 5 月 1 日 12:00AM |
| 6 | 1 | 2022-06-01 | >= 2022 年 5 月 1 日 12:00AM 且 < 2022 年 6 月 1 日 12:00AM |
| 7 | 1 | 2022-07-01 | >= 2022 年 6 月 1 日 12:00AM 且 < 2022 年 7 月 1 日 12:00AM |
| 8 | 1 | 2022-08-01 | >= 2022 年 7 月 1 日 12:00AM 且 < 2022 年 8 月 1 日 12:00AM |
| 9 | 1 | 2022-09-01 | >= 2022 年 8 月 1 日 12:00AM 且 < 2022 年 9 月 1 日 12:00AM |
| 10 | 1 | 2022-10-01 | >= 2022 年 9 月 1 日 12:00AM 且 < 2022 年 10 月 1 日 12:00AM |
| 11 | 1 | 2022-11-01 | >= 2022 年 10 月 1 日 12:00AM 且 < 2022 年 11 月 1 日 12:00AM |
| 12 | 1 | 2022-12-01 | >= 2022 年 11 月 1 日 12:00AM 且 < 2022 年 12 月 1 日 12:00AM |
| 13 | 0 |   | >= 2022 年 12 月 1 日 12:00AM 且 < 最大值 |

这表明您有两个空分区。第一个空分区用于存放 2022 年 1 月 1 日之前的数据。最后一个分区用于存放 2022 年 12 月 1 日当天或之后的数据。

第一步是将 2022 年 1 月的分区合并到 2022 年 2 月的分区中，如代码清单 17-24 所示。

```
ALTER PARTITION FUNCTION CustomerOrderHistoryArchiveFunc()
MERGE RANGE ('2022-01-01');
代码清单 17-24
合并超过 15 个月的客户订单分区
```

这将把存放 2022 年 1 月 1 日之前数据的最旧分区合并到存放 2022 年 2 月 1 日之前数据的分区中。分区将更新为如表 17-9 所示。

### 表 17-9：合并分区后的 CustomerOrderHistoryArchive

| 分区号 | 记录数 | 边界值 | 文本比较 |
| --- | --- | --- | --- |
| 1 | 1 | 2022-01-01 | >= 最小值且 < 2022 年 2 月 1 日 12:00AM |
| 2 | 1 | 2022-02-01 | >= 2022 年 2 月 1 日 12:00AM 且 < 2022 年 3 月 1 日 12:00AM |
| 3 | 1 | 2022-03-01 | >= 2022 年 3 月 1 日 12:00AM 且 < 2022 年 4 月 1 日 12:00AM |
| 4 | 1 | 2022-04-01 | >= 2022 年 4 月 1 日 12:00AM 且 < 2022 年 5 月 1 日 12:00AM |
| 5 | 1 | 2022-05-01 | >= 2022 年 5 月 1 日 12:00AM 且 < 2022 年 6 月 1 日 12:00AM |
| 6 | 1 | 2022-06-01 | >= 2022 年 6 月 1 日 12:00AM 且 < 2022 年 7 月 1 日 12:00AM |
| 7 | 1 | 2022-07-01 | >= 2022 年 7 月 1 日 12:00AM 且 < 2022 年 8 月 1 日 12:00AM |
| 8 | 1 | 2022-08-01 | >= 2022 年 8 月 1 日 12:00AM 且 < 2022 年 9 月 1 日 12:00AM |
| 9 | 1 | 2022-09-01 | >= 2022 年 9 月 1 日 12:00AM 且 < 2022 年 10 月 1 日 12:00AM |
| 10 | 1 | 2022-10-01 | >= 2022 年 10 月 1 日 12:00AM 且 < 2022 年 11 月 1 日 12:00AM |
| 11 | 1 | 2022-11-01 | >= 2022 年 11 月 1 日 12:00AM 且 < 2022 年 12 月 1 日 12:00AM |
| 12 | 0 |   | >= 2022 年 12 月 1 日 12:00AM 且 < 最大值 |

合并结果显示，第一个分区现在用于存放 2022 年 2 月 1 日之前的所有数据。该表还显示此分区中有一条记录。根据表 17-8 和表 17-9，可以推断这条记录是 2022 年 2 月某个时间的数据。

既然已经合并了分区，就可以将 `dbo.CustomerOrderHistoryArchive` 的最后一个分区切换到 `dbo.CustomerOrderPurge`，如代码清单 17-25 所示。

```
ALTER TABLE dbo.CustomerOrderHistoryArchive
SWITCH PARTITION 1 TO dbo.CustomerOrderPurge;
代码清单 17-25
切换出超过 60 个月的客户订单分区
```

预期是 `dbo.CustomerOrderHistoryArchive` 分区 1 中的所有数据现在都已进入 `dbo.CustomerOrderPurge`。您可以通过像之前在表 17-8 中那样获取每个表的行数来验证这一点。查询每个表的行数得到如表 17-10 所示的结果。

### 表 17-10：管理客户订单的各表记录数

| 表名 | 记录数 |
| --- | --- |
| `dbo.CustomerOrderActive` | 1 |
| `dbo.CustomerOrderHistory` | 3 |
| `dbo.CustomerOrderHistoryArchive` | 11 |
| `dbo.CustomerOrderPurge` | 1 |

这些记录数也表明，`dbo.CustomerOrderHistoryArchive` 中 2022 年 1 月的行已被切换到 `dbo.CustomerOrderPurge`。在清除数据之前，您可以再次查看 `dbo.CustomerOrderHistoryArchive` 的分区信息，如表 17-11 所示。

### 表 17-11：切换分区后的 CustomerOrderHistoryArchive

| 分区号 | 记录数 | 边界值 | 文本比较 |
| --- | --- | --- | --- |
| 1 | 0 | 2022-01-01 | >= 最小值且 < 2022 年 2 月 1 日 12:00AM |
| 2 | 1 | 2022-02-01 | >= 2022 年 2 月 1 日 12:00AM 且 < 2022 年 3 月 1 日 12:00AM |
| 3 | 1 | 2022-03-01 | >= 2022 年 3 月 1 日 12:00AM 且 < 2022 年 4 月 1 日 12:00AM |
| 4 | 1 | 2022-04-01 | >= 2022 年 4 月 1 日 12:00AM 且 < 2022 年 5 月 1 日 12:00AM |
| 5 | 1 | 2022-05-01 | >= 2022 年 5 月 1 日 12:00AM 且 < 2022 年 6 月 1 日 12:00AM |
| 6 | 1 | 2022-06-01 | >= 2022 年 6 月 1 日 12:00AM 且 < 2022 年 7 月 1 日 12:00AM |
| 7 | 1 | 2022-07-01 | >= 2022 年 7 月 1 日 12:00AM 且 < 2022 年 8 月 1 日 12:00AM |
| 8 | 1 | 2022-08-01 | >= 2022 年 8 月 1 日 12:00AM 且 < 2022 年 9 月 1 日 12:00AM |
| 9 | 1 | 2022-09-01 | >= 2022 年 9 月 1 日 12:00AM 且 < 2022 年 10 月 1 日 12:00AM |
| 10 | 1 | 2022-10-01 | >= 2022 年 10 月 1 日 12:00AM 且 < 2022 年 11 月 1 日 12:00AM |
| 11 | 1 | 2022-11-01 | >= 2022 年 11 月 1 日 12:00AM 且 < 2022 年 12 月 1 日 12:00AM |
| 12 | 0 |   | >= 2022 年 12 月 1 日 12:00AM 且 < 最大值 |

这也证实，第一个分区中的大部分数据已被切换到 `dbo.CustomerOrderPurge`。

一旦数据从 `dbo.CustomerOrderHistoryArchive` 切换到 `dbo.CustomerOrderPurge`，就可以从 `dbo.CustomerOrderPurge` 中移除这些数据。最快的方法是使用 `TRUNCATE`，如代码清单 17-26 所示。

```
TRUNCATE TABLE dbo.CustomerOrderPurge;
代码清单 17-26
截断表
```

将数据从 `dbo.CustomerOrderPurge` 清除后，您可以继续在剩余表之间切换分区的过程。

下一步是更新 `dbo.CustomerOrderHistoryArchive` 中的分区。您需要为将从 `dbo.CustomerOrderHistory` 切换出来的新数据准备该表。在您的示例中，您需要拆分 `dbo.CustomerOrderHistoryArchive` 中的最后一个分区，以便为 2023 年 1 月的数据创建一个单独的分区。代码清单 17-27 显示了拆分最新分区的 T-SQL。

```
ALTER PARTITION FUNCTION CustomerOrderHistoryArchiveFunc()
SPLIT RANGE ('2023-01-01');
代码清单 17-27
拆分超过四个月的客户订单分区
```

此分区拆分将在该表中创建 13 个可用分区。这类似于您在开始此过程之前的分区方式。然而，在您的情况下，分区已向后移动了一个月，如表 17-12 所示。

### 切换分区后的 CustomerOrderHistoryArchive

（注：此处原文未提供完整表格内容，故仅保留标题）



| 分区编号 | 记录数 | 边界值 | 文本比较 |
| --- | --- | --- | --- |
| 1 | 0 | 2022-02-01 | >= 最小值 且 < 2022 年 2 月 1 日 12:00AM |
| 2 | 1 | 2022-03-01 | >= 2022 年 2 月 1 日 12:00AM 且 < 2022 年 3 月 1 日 12:00AM |
| 3 | 1 | 2022-04-01 | >= 2022 年 3 月 1 日 12:00AM 且 < 2022 年 4 月 1 日 12:00AM |
| 4 | 1 | 2022-05-01 | >= 2022 年 4 月 1 日 12:00AM 且 < 2022 年 5 月 1 日 12:00AM |
| 5 | 1 | 2022-06-01 | >= 2022 年 5 月 1 日 12:00AM 且 < 2022 年 6 月 1 日 12:00AM |
| 6 | 1 | 2022-07-01 | >= 2022 年 6 月 1 日 12:00AM 且 < 2022 年 7 月 1 日 12:00AM |
| 7 | 1 | 2022-08-01 | >= 2022 年 7 月 1 日 12:00AM 且 < 2022 年 8 月 1 日 12:00AM |
| 8 | 1 | 2022-09-01 | >= 2022 年 8 月 1 日 12:00AM 且 < 2022 年 9 月 1 日 12:00AM |
| 9 | 1 | 2022-10-01 | >= 2022 年 9 月 1 日 12:00AM 且 < 2022 年 10 月 1 日 12:00AM |
| 10 | 1 | 2022-11-01 00:00:00.000 | >= 2022 年 10 月 1 日 12:00AM 且 < 2022 年 11 月 1 日 12:00AM |
| 11 | 1 | 2022-12-01 00:00:00.000 | >= 2022 年 11 月 1 日 12:00AM 且 < 2022 年 12 月 1 日 12:00AM |
| 12 | 1 | 2023-01-01 00:00:00.000 | >= 2022 年 12 月 1 日 12:00AM 且 < 2023 年 1 月 1 日 12:00AM |
| 13 | 0 |   | >= 2023 年 1 月 1 日 12:00AM 且 < 最大值 |

你已成功将 2022 年 1 月的分区从 `dbo.CustomerOrderHistoryArchive` 中切换出，并将 2023 年 1 月的分区切换进来。你是通过将分区从分区表切换到非分区表来完成此操作的。

整体流程是：你将一个旧分区从已分区的 `dbo.CustomerOrderHistoryArchive` 表切换到了未分区的 `dbo.CustomerOrderHistoryPurge` 表。然后，你截断了 `dbo.CustomerOrderHistoryPurge` 表以删除最旧的数据。将分区从 `dbo.CustomerOrderHistoryArchive` 切换出后，该表剩下 11 个分区而不是 12 个。为了恢复 `dbo.CustomerOrderHistoryArchive` 中的 12 个分区，你拆分了最新的分区。接着，你将最旧的分区从 `dbo.CustomerOrderHistory` 移动到了 `dbo.CustomerOrderHistoryArchive`。`dbo.CustomerOrderHistoryArchive` 中最旧的数据被移入了刚刚拆分出的分区。由于 `dbo.CustomerOrderHistory` 现在只有两个分区，你需要拆分最近的分区，以便该表再次拥有三个分区。

现在分区切换已完成，你可以为过去一至三个月的客户订单重复此过程。在本示例中，你将在两个分区表之间切换分区。通过核实表 17-13 中的表行数，`dbo.CustomerOrderHistoryArchive` 中只有 11 条记录。

**表 17-13**
**管理客户订单的各表记录数统计**

| 表名 | 记录数 |
| --- | --- |
| `dbo.CustomerOrderActive` | 1 |
| `dbo.CustomerOrderHistory` | 3 |
| `dbo.CustomerOrderHistoryArchive` | 11 |
| `dbo.CustomerOrderPurge` | 0 |

由于此表应包含过去四至十二个月的数据，因此存在数据缺失。根据上面的表 17-12，你知道 2023 年 1 月之后数据的最后一个分区是空的。然而，要使此表分区完整，你需要将 2023 年 1 月的数据从 `dbo.CustomerOrderHistory` 切换到 `dbo.CustomerOrderHistoryArchive` 的最后一个分区中。

由于你是在两个分区之间切换，因此需要确定要从 `dbo.CustomerOrderHistory` 切换出的分区。执行清单 17-28 中的查询将显示 `dbo.CustomerOrderHistory` 的分区信息。

**清单 17-28**
**合并超过 15 个月的客户订单分区**

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
WHERE tbl.[name] = 'CustomerOrderHistory'
AND idx.[type] <= 1
ORDER BY tbl.[name], pt.partition_number;
```

运行此查询可以获得表 `dbo.CustomerOrderHistory` 的分区数量、记录数量和边界信息。此查询的结果如表 17-14 所示。

**表 17-14**
**分区切换前的 CustomerOrderHistory**

| 分区编号 | 记录数 | 边界值 | 文本比较 |
| --- | --- | --- | --- |
| 1 | 0 | 2023-01-01 | >= 最小值 且 < 2023 年 1 月 1 日 12:00AM |
| 2 | 1 | 2023-02-01 | >= 2023 年 1 月 1 日 12:00AM 且 < 2023 年 2 月 1 日 12:00AM |
| 3 | 1 | 2023-03-01 | >= 2023 年 2 月 1 日 12:00AM 且 < 2023 年 3 月 1 日 12:00AM |
| 4 | 1 |   | >= 2023 年 3 月 1 日 12:00AM 且 < 最大值 |

此表显示，分区 2 包含了你需要从 `dbo.CustomerOrderHistory` 获取的记录。你可以使用清单 17-29 中的 T-SQL 将这些记录切换出去。

**清单 17-29**
**切换出超过四个月的客户订单分区**

```sql
ALTER TABLE dbo.CustomerOrderHistory SWITCH PARTITION 2
TO dbo.CustomerOrderHistoryArchive PARTITION 13;
```

你将 `dbo.CustomerOrderHistory` 第二个分区中的 2023 年 1 月数据切换到了 `dbo.CustomerOrderHistoryArchive` 的最后一个分区，该分区允许存放 2023 年 1 月及之后的所有记录。这也是验证每个表行数的好时机。表 17-15 显示了将数据从 `dbo.CustomerOrderHistory` 切换到 `dbo.CustomerOrderHistoryArchive` 后的行数。

**表 17-15**
**管理客户订单的各表记录数统计**

| 表名 | 记录数 |
| --- | --- |
| `dbo.CustomerOrderActive` | 1 |
| `dbo.CustomerOrderHistory` | 2 |
| `dbo.CustomerOrderHistoryArchive` | 12 |
| `dbo.CustomerOrderPurge` | 0 |

这证实你已成功在两个表之间切换了 2023 年 1 月的记录。

既然已将数据从 `dbo.CustomerOrderHistory` 切换出，现在是时候更新此表的分区了。清单 17-30 显示了更新分区所需的 T-SQL，以便范围从 2023 年 2 月开始。

**清单 17-30**
**合并 dbo.CustomerOrderHistory 的前两个分区**

```sql
ALTER PARTITION FUNCTION CustomerOrderHistoryFunc()
MERGE RANGE ('2023-01-01');
```



### 分区维护操作

此代码将表中的第一个和第二个分区合并为一个分区。这会将分区总数从四个减少到三个。在您的情况下，这将更新第一个分区以包含 2023 年 2 月之前的所有数据。要完成为该表准备新月份数据的操作，您还需要按清单 17-31 所示拆分最后一个分区。

```
ALTER PARTITION FUNCTION CustomerOrderHistoryFunc()
SPLIT RANGE ('2023-04-01');
Listing 17-31
Spliting the Last Partition for dbo.CustomerOrderHistory
```

执行此 T-SQL 后，表中将再次有四个分区。倒数第二个分区将用于存放 2023 年 3 月的数据。最后一个分区将用于存放 2023 年 4 月 1 日或之后的数据。更新后的分区情况如表 17-16 所示。

表 17-16 CustomerOrderHistory 在分区切换前

| Partition Number | Record Count | Boundary Value | Text Comparison |
| --- | --- | --- | --- |
| 1 | 0 | 2023-02-01 | >= Min Value and < Feb  1 2023 12:00AM |
| 2 | 1 | 2023-03-01 | >= Feb  1 2023 12:00AM and < Mar  1 2023 12:00AM |
| 3 | 1 | 2023-04-01 | >= Mar  1 2023 12:00AM and < Apr  1 2023 12:00AM |
| 4 | 0 |   | >= Apr  1 2023 12:00AM and < Max Value |

您已成功准备好 `dbo.CustomerOrderHistory` 表，以便从 `dbo.CustomerOrderActive` 切换入 2023 年 4 月的数据。

### 检查约束与数据切换

在您从非分区表 `dbo.CustomerOrderActive` 切换数据之前，需要向该表添加检查约束。这允许 SQL Server 确保从 `dbo.CustomerOrderActive` 切换到 `dbo.CustomerOrderHistory` 第四个分区的数据能够成功切换。清单 17-32 显示了创建所需检查约束的 T-SQL。

```
ALTER TABLE dbo.CustomerOrderActive
WITH CHECK ADD CONSTRAINT CK_CustomerOrderActive3_MinDateCreated
CHECK (DateCreated IS NOT NULL AND DateCreated >= '2023-04-01');
ALTER TABLE dbo.CustomerOrderActive
WITH CHECK ADD CONSTRAINT CK_CustomerOrderActive3_MaxDateCreated
CHECK (DateCreated IS NOT NULL AND DateCreated < '2023-05-01');
Listing 17-32
Creating Check Constraints on the Non-Partitioned Table
```

此步骤假设您之前未创建检查约束。由于性能限制，建议更新检查约束范围，而不是删除约束。在您完成分区切换后，我将讨论该步骤。

一旦添加了这些检查约束，您就可以按清单 17-33 所示，将数据从 `dbo.CustomerOrderActive` 切换到 `dbo.CustomerOrderHistory`。

```
ALTER TABLE dbo.CustomerOrderActive
SWITCH TO dbo.CustomerOrderHistory PARTITION 4;
Listing 17-33
Partition Switch Out Last Month’s Customer Orders
```

此 T-SQL 将 `dbo.CustomerOrderActive` 中的所有数据移动到 `dbo.CustomerOrderHistory` 的第四个分区。表 17-17 显示了将数据从 `dbo.CustomerOrderActive` 切换出后的记录数。

表 17-17 管理客户订单的表记录数统计

| Table Name | Record Count |
| --- | --- |
| `dbo.CustomerOrderActive` | 0 |
| `dbo.CustomerOrderHistory` | 3 |
| `dbo.CustomerOrderHistoryArchive` | 12 |
| `dbo.CustomerOrderPurge` | 0 |

您可以通过执行清单 17-28 中的代码来验证 `dbo.CustomerOrderHistory` 中行的分布情况。分区分布情况可在下面的表 17-18 中找到。

表 17-18 CustomerOrderHistory 在分区切换后

| Partition Number | Record Count | Boundary Value | Text Comparison |
| --- | --- | --- | --- |
| 1 | 0 | 2023-02-01 | >= Min Value and < Feb  1 2023 12:00AM |
| 2 | 1 | 2023-03-01 | >= Feb  1 2023 12:00AM and < Mar  1 2023 12:00AM |
| 3 | 1 | 2023-04-01 | >= Mar  1 2023 12:00AM and < Apr  1 2023 12:00AM |
| 4 | 1 |   | >= Apr  1 2023 12:00AM and < Max Value |

表 `dbo.CustomerOrderHistory` 已更新为包含过去三个月的数据。您已完成所有必要的分区切换，但仍需对表 `dbo.CustomerOrderActive` 进行一些更新，以便它可以收集 2023 年 5 月的数据。

### 更新检查约束与创建视图

由于您使用的是 T-SQL，需要删除 `dbo.CustomerOrderActive` 上现有的检查约束，然后创建新约束。由于表是空的，此过程将相对较快。清单 17-34 显示了可用于为 2023 年 5 月更新检查约束的代码。

```
ALTER TABLE dbo.CustomerOrderActive
DROP CONSTRAINT CK_CustomerOrderActive_MinDateCreated;
ALTER TABLE dbo.CustomerOrderActive
DROP CK_CustomerOrderActive_MaxDateCreated;
ALTER TABLE dbo.CustomerOrderActive
WITH CHECK ADD CONSTRAINT CK_CustomerOrderActive_MinDateCreated
CHECK (DateCreated IS NOT NULL AND DateCreated >= '2023-05-01');
ALTER TABLE dbo.CustomerOrderActive
WITH CHECK ADD CONSTRAINT CK_CustomerOrderActive_MaxDateCreated
CHECK (DateCreated IS NOT NULL AND DateCreated < '2023-06-01');
Listing 17-34
Updating Check Constraints on dbo.CustomerOrderActive
```

此代码更新表，使其只允许创建日期为 2023 年 5 月的记录被导入表中。在表为空时添加约束，不要在一个月后执行窗口滑动代码时尝试添加。这可以节省大量时间，而不是在表已满后添加检查约束。

既然您已将所有表更新为包含正确的数据，您还可以使用分区视图来访问这些表中的数据。可能存在业务原因，使得应用程序或用户只想访问当前月份的订单以及过去三个月的订单。请参阅清单 17-35。

```
CREATE VIEW dbo.vwCustomerOrderHistoryRecent
AS
-- Select data from current read/write table
SELECT CustomerOrderHistoryID,
CustomerOrderID,
CustomerOrderHistoryStatusID,
DateCreated,
DateModified
FROM dbo.CustomerOrderActive
UNION ALL
-- Select data from historicial table
SELECT CustomerOrderHistoryID,
CustomerOrderID,
CustomerOrderHistoryStatusID,
DateCreated,
DateModified
FROM dbo.CustomerOrderHistory;
Listing 17-35
Partitioned View for Recent Orders, Current Month, and Past Three Months
```

既然您已更新了分区，此视图将返回 2023 年 2 月至 2023 年 5 月的结果。如果您发现需要访问所有数据的情况，那么可以创建另一个分区视图以方便访问所有可用订单。在清单 17-36 中，该视图合并了来自 `dbo.CustomerOrderActive`、`dbo.CustomerOrderHistory` 和 `dbo.CustomerOrderHistoryArchive` 的所有订单信息。

```
CREATE VIEW dbo.vwCustomerOrderHistoryAll
AS
-- Select data from current read/write table
SELECT CustomerOrderHistoryID,
CustomerOrderID,
CustomerOrderHistoryStatusID,
DateCreated,
DateModified
FROM dbo.CustomerOrderActive
UNION ALL
-- Select data from historicial table
SELECT CustomerOrderHistoryID,
CustomerOrderID,
CustomerOrderHistoryStatusID,
DateCreated,
DateModified
FROM dbo.CustomerOrderHistory
UNION ALL
-- Select data from archive table
SELECT CustomerOrderHistoryID,
CustomerOrderID,
CustomerOrderHistoryStatusID,
DateCreated,
DateModified
FROM dbo.CustomerOrderHistoryArchive;
Listing 17-36
Partitioned View for All Orders
```

此视图显示从 2022 年 2 月到 2023 年 5 月的任何客户订单。这适用于数据库中存在的任何订单。



## 数据管理总结

上一节介绍了如何结合分区切换来管理长期数据。通常，应用程序不需要访问与报表相同的数据。创建独立的表可能有助于长期数据管理。

本章继续讨论第 18 章引入的分区概念。在本章中，我介绍了如何在非分区表、分区表以及两者组合之间切换分区。您利用这些知识建立了一种管理数据归档的方法。这使您能够管理数据访问，并可能通过让应用程序使用数据量更小的表来提升性能。至此，关于构建可扩展 T-SQL 以有效管理数据的部分就结束了。下一节将介绍如何增强数据的安全性。

