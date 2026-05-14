# 17. 长期数据管理

在上一章中，我介绍了分区表和分区视图的概念。本章继续讨论分区，重点讲解如何使用分区切换。第一节介绍分区切换的概念。第二节指导你运用已学到的关于分区切换的所有知识，来管理数据保留和归档。

在前几章中，我们探讨了分区表和分区视图在提升查询性能方面的应用。本章将转向数据生命周期管理，介绍一个强大的工具——分区切换，它能极大地简化数据的归档和清除过程。

## 分区切换简介

分区切换是一种高效的数据管理操作，它能够将整个分区（或非分区表）作为一个单元快速地移入或移出分区表。此操作本质上是元数据的更改，因此速度极快，且通常只需极少的系统资源。

其核心思想是将数据存储与数据管理分离。你可以通过创建 `staging` 表（暂存表）来准备新数据，或通过将旧数据移动到归档表中来清理生产数据，然后通过 `ALTER TABLE ... SWITCH` 语句在瞬间完成“切换”，而无需实际移动大量数据行。

### 使用分区切换进行数据管理

通过一个具体的例子来理解如何利用分区切换进行数据管理。假设你有一个按月分区的销售表 `Sales`，并且希望将超过 24 个月的旧数据归档。

1.  **准备归档表**：首先，创建一个与 `Sales` 表结构相同的非分区表 `SalesArchive`。确保其文件组与要移出的分区所在文件组一致，并且没有任何索引或约束（或与原分区完全相同）。
    ```sql
    CREATE TABLE SalesArchive
    (
        SaleID INT,
        SaleDate DATE,
        Amount DECIMAL(10,2)
        -- ... 其他列
    ) ON [ArchiveFG];
    ```

2.  **切换分区**：使用 `ALTER TABLE ... SWITCH PARTITION ... TO` 语句将指定的分区切换到归档表。
    ```sql
    ALTER TABLE Sales
    SWITCH PARTITION 1 TO SalesArchive;
    ```
    执行后，原 `Sales` 表第一个分区的所有数据行会“变成” `SalesArchive` 表的内容，而 `Sales` 表的第一个分区变为空。这个操作几乎是瞬间完成的。

3.  **处理空分区**：切换后，`Sales` 表留下的空分区可以被移除（`ALTER PARTITION FUNCTION ... MERGE RANGE`），为未来的新数据腾出空间，或者用于接收新数据。

### 将非分区表切换到分区表

分区切换不仅可以从分区表中移出数据，还可以将准备好的非分区表（暂存表）切换为分区表的一个新分区。这在批量加载新数据时非常有用，因为它可以避免直接在大型生产表上执行缓慢的 `INSERT` 操作。

流程如下：
1.  在一个单独的文件组上创建并准备 `Staging` 表，填入新数据。
2.  确保 `Staging` 表的结构与目标分区表兼容。
3.  将 `Staging` 表切换为目标分区表的一个新分区。
    ```sql
    ALTER TABLE Staging
    SWITCH TO Sales PARTITION 5;
    ```
    此操作后，`Staging` 表变为空，并成为 `Sales` 表的一部分，而加载过程对用户的影响微乎其微。

### 注意事项

-   **兼容性检查**：源表和目标表/分区的架构必须完全兼容，包括列、数据类型、索引、约束、压缩设置等。不匹配的操作将失败。
-   **对齐要求**：分区表的分区函数和分区方案必须已正确定义，目标分区必须存在且为空（移出时），或者源表必须与目标分区架构对齐（移入时）。
-   **文件组**：源表和目标表通常需要位于相同的文件组或文件组组中。

分区切换是管理大数据量生命周期的关键技术。它使得按时间窗口（如每月、每季度）归档历史数据或加载新数据变得高效而简单，从而在保持系统性能的同时，满足数据保留策略的要求。结合分区视图，你可以构建出既灵活又高效的数据存储架构。


## 数据保留与归档

长期的数据管理应能让你高效地存储数据并快速删除数据。本节介绍如何将待删除的数据交换到一个新表中，以便将其移除。表的最后两个分区将被合并，这样最旧的数据仍保留在最后一个分区中。最后，新分区将被拆分，以便新数据可以保存到最新的分区中。整个过程被称为*滑动窗口分区*。如果你决定实现此过程之一，最好使用 SQL Server 代理作业将此过程自动化。

让我们熟悉一下你将用于此过程的表的 T-SQL。第一个表是 `dbo.CustomerOrderActive`。此表用于根据业务规则保存活动订单。清单 17-1 中的表用于保存当月的客户订单。

```sql
CREATE TABLE dbo.CustomerOrderActive
(
CustomerOrderHistoryID         BIGINT             NOT NULL,
CustomerOrderID                INT                NOT NULL,
CustomerOrderHistoryStatusID   TINYINT            NOT NULL,
DateCreated                    DATETIME2(2)       NOT NULL,
DateModified                   DATETIME2(2)           NULL,
CONSTRAINT PK_CustomerOrderActive_CustomerOrderHistoryID
PRIMARY KEY (CustomerOrderHistoryID),
CONSTRAINT FK_CustomerOrderActive_CustomerOrderID
FOREIGN KEY (CustomerOrderID)
REFERENCES dbo.CustomerOrder(CustomerOrderID),
CONSTRAINT FK_CustomerOrderActive_CustomerOrderHistoryStatusID
FOREIGN KEY (CustomerOrderHistoryStatusID)
REFERENCES
dbo.CustomerOrderHistoryStatus(CustomerOrderHistoryStatusID)
);
```

`清单 17-1`
当月客户订单

预期所有订单都将在三个月内完成。因此，最近三个月的活动预计最为频繁。

第二个表用于保存过去 12 个月的客户历史记录。此数据用于业务内的报告和分析。在清单 17-2 中，T-SQL 显示该表已分区。

```sql
CREATE TABLE dbo.CustomerOrderHistory
(
CustomerOrderHistoryID        BIGINT    IDENTITY(1,1)    NOT NULL,
CustomerOrderID               INT                        NOT NULL,
CustomerOrderHistoryStatusID  TINYINT                    NOT NULL,
DateCreated                   DATETIME2(2)               NOT NULL,
DateModified                  DATETIME2(2)                   NULL,
CONSTRAINT PK_CustomerOrderHistory_CustomerOrderHistoryID
PRIMARY KEY NONCLUSTERED
(CustomerOrderHistoryID, DateCreated),
CONSTRAINT FK_CustomerOrderHistory_CustomerOrder
FOREIGN KEY (CustomerOrderID)
REFERENCES dbo.CustomerOrder(CustomerOrderID),
CONSTRAINT FK_CustomerOrderHistory_CustomerOrderHistoryStatus
FOREIGN KEY (CustomerOrderHistoryStatusID)
REFERENCES dbo.CustomerOrderHistoryStatus
(CustomerOrderHistoryStatusID)
) ON CustomerOrderHistoryRange (DateCreated);
```

`清单 17-2`
过去 12 个月客户订单的分区表

第三个表是一个用于保存法规目的所需存档数据的表，在本例中是超过一年的客户订单信息。15 个月后，数据将从此表中删除。此表也是分区的，如下方清单 17-3 所示。

```sql
CREATE TABLE dbo.CustomerOrderHistoryArchive
(
CustomerOrderHistoryID        BIGINT     IDENTITY(1,1)    NOT NULL,
CustomerOrderID               INT                         NOT NULL,
CustomerOrderHistoryStatusID  TINYINT                     NOT NULL,
DateCreated                   DATETIME2(2)                NOT NULL,
DateModified                  DATETIME2(2)                    NULL,
CONSTRAINT PK_CustomerOrderHistoryArchive_CustomerOrderHistoryID
PRIMARY KEY NONCLUSTERED
(CustomerOrderHistoryID, DateCreated),
CONSTRAINT FK_CustomerOrderHistoryArchive_CustomerOrder
FOREIGN KEY (CustomerOrderID)
REFERENCES dbo.CustomerOrder(CustomerOrderID),
CONSTRAINT FK_CustomerOrderHistoryArchive_CustomerOrderHistoryStatus
FOREIGN KEY (CustomerOrderHistoryStatusID)
REFERENCES dbo.CustomerOrderHistoryStatus
(CustomerOrderHistoryStatusID)
) ON CustomerOrderHistoryArchiveRange (DateCreated);
```

`清单 17-3`
用于归档过去 15 个月客户订单的分区表

最后一个表用于临时保存要清除的数据。此表必须与其他表具有相同的格式，如清单 17-4 所示。

```sql
CREATE TABLE dbo.CustomerOrderPurge
(
CustomerOrderHistoryID        BIGINT    IDENTITY(1,1)   NOT NULL,
CustomerOrderID               INT                       NOT NULL,
CustomerOrderHistoryStatusID  TINYINT                   NOT NULL,
DateCreated                   DATETIME2(2)              NOT NULL,
DateModified                  DATETIME2(2)                  NULL,
CONSTRAINT PK_CustomerOrderPurge_CustomerOrderHistoryID
PRIMARY KEY NONCLUSTERED
(CustomerOrderHistoryID, DateCreated),
CONSTRAINT FK_CustomerOrderPurge_CustomerOrder
FOREIGN KEY (CustomerOrderID)
REFERENCES dbo.CustomerOrder(CustomerOrderID),
CONSTRAINT FK_CustomerOrderPurge_CustomerOrderHistoryStatus
FOREIGN KEY (CustomerOrderHistoryStatusID)
REFERENCES dbo.CustomerOrderHistoryStatus
(CustomerOrderHistoryStatusID)
);
```

`清单 17-4`
用于清除超过 15 个月的客户订单的表

现在你已经创建了表，接下来是在月底移动这些分区的过程。



## 切换分区

在第 18 章中，我介绍了 SQL Server 中分区的概念。本节将探讨创建分区后管理分区的过程。第一个示例将演示如何在两个**非分区**表之间使用分区切换。第二个示例将展示如何将数据从非分区表切换到分区表。最后一个示例将展示如何在分区表之间切换分区。

### 在非分区表之间切换

对于第一个示例，你将使用清单 17-5 中的 T-SQL 代码创建一个非分区表。

```sql
CREATE TABLE dbo.CustomerOrderNonParitition(
    CustomerOrderHistoryID         BIGINT         NOT NULL,
    CustomerOrderID                INT            NOT NULL,
    CustomerOrderHistoryStatusID   TINYINY        NOT NULL,
    DateCreated                    DATETIME2(2)   NOT NULL,
    DateModified                   DATETIME2(2)   NULL,
    CONSTRAINT [PK_CustomerOrderNonPartition]
    PRIMARY KEY CLUSTERED ([CustomerOrderHistoryID], [DateCreated])
);
-- 清单 17-5
-- 创建一个非分区表
```

这是一个新的空表，其架构与表 `dbo.CustomerOrderActive` 相同。这两个表都未进行分区。在将数据从一个表切换到另一个表之前，你应该获取每个表中的记录数。这是因为非分区表被视为由单个分区组成的表。清单 17-6 中的查询获取了 `dbo.CustomerOrderActive` 和 `dbo.CustomerOrderNonPartition` 表的记录数。

```sql
SELECT COUNT(*)
FROM dbo.CustomerOrderActive;
SELECT COUNT(*)
FROM dbo.CustomerOrderNonParitition;
-- 清单 17-6
-- 获取源表和目标表的记录数
```

上述查询的输出已格式化为表 17-1，以显示每个表的客户记录数。

表 17-1

每个表的记录数

| 表名 | 记录数 |
| --- | --- |
| `dbo.CustomerOrderActive` | 802,821 |
| `dbo.CustomerOrderNonPartition` | 0 |

在分区切换之前，`dbo.CustomerOrder` 表有 802,821 条记录，而 `dbo.CustomerOrderNonPartition` 表有 0 条记录。清单 17-7 中的 T-SQL 展示了如何将数据切换到另一个非分区表。

```sql
ALTER TABLE dbo.CustomerOrderActive
SWITCH TO dbo.CustomerOrderNonParitition;
-- 清单 17-7
-- 在非分区表之间移动数据
```

将分区（在此例中是整个表）从 `dbo.CustomerOrderActive` 切换到 `dbo.CustomerOrderNonPartition` 后，你可以运行查询来检查两个表的行数，如清单 17-8 所示。

```sql
SELECT COUNT(*)
FROM dbo.CustomerOrderActive;
SELECT COUNT(*)
FROM dbo.CustomerOrderNonParitition;
-- 清单 17-8
-- 非分区表中的记录数
```

运行此 T-SQL 后，你将获得两个表的记录数，如表 17-2 所示。

表 17-2

每个表的记录数

| 表名 | 记录数 |
| --- | --- |
| `dbo.CustomerOrderActive` | 0 |
| `dbo.CustomerOrderNonPartition` | 802,821 |

该表显示 `dbo.CustomerOrderActive` 中没有行，而 `dbo.CustomerOrderNonPartition` 中有 802,821 行。这符合预期，因为 `dbo.CustomerOrderActive` 的整个表已切换到 `dbo.CustomerOrderNonPartition`。如果你想将表恢复到分区切换之前的状态，可以执行清单 17-9 中的 T-SQL，将数据从 `dbo.CustomerOrderNonParition` 切换回 `dbo.CustomerOrderActive`。

```sql
ALTER TABLE dbo.CustomerOrderNonParitition
SWITCH TO dbo.CustomerOrderActive;
-- 清单 17-9
-- 将数据移回原始表
```

现在分区已切换回来，所有记录再次位于 `dbo.CustomerOrderActive` 中。如果在 `dbo.CustomerOrderActive` 表中写入了数据，你将无法将数据切换回原始表。目标表必须为空。清单 17-10 中有一个查询，你可以执行它来确认 `dbo.CustomerOrderNonPartition` 没有任何分区。

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
WHERE tbl.[name] = 'CustomerOrderNonParitition'
    AND idx.[type] <= 1
ORDER BY prt.partition_number;
-- 清单 17-10
-- 查询以查看 CustomerOrderNonPartition 表的分区信息
```

### 从非分区表切换到分区表

以上示例展示了如何在两个非分区表之间进行分区切换。然而，更可能的情况是，你会希望至少对一个分区表使用分区切换。

接下来，你将通过一个示例来学习如何从非分区表切换到分区表。作为复习，清单 17-11 展示了如何创建分区函数和分区方案。

```sql
CREATE PARTITION FUNCTION CustomerOrderHistoryFunc(DATETIME2(2))
AS RANGE RIGHT FOR VALUES
(
    '2022-01-01',
    '2022-02-01',
    '2022-03-01'
);
CREATE PARTITION SCHEME CustomerOrderHistoryRange
AS PARTITION CustomerOrderHistoryFunc TO
(
    [PRIMARY],
    [PRIMARY],
    CustomerOrderHistory202202,
    CustomerOrderHistory202203
);
-- 清单 17-11
-- 创建分区函数和分区方案
```

切换分区时需要注意的一个重要点是：被切换出的源表和切换入的目标表必须位于**相同的文件组**中。对于你的示例，`dbo.CustomerOrderActive` 位于 PRIMARY 文件组中。因此，如果你要将其切换到 `dbo.CustomerOrderHistory` 的第一个分区，那么第一个分区也必须位于 PRIMARY 文件组中。否则，你将需要把 `dbo.CustomerOrderActive` 移动到与 `dbo.CustomerOrderHistory` 中第一个分区相同的文件组。

在将记录从 `dbo.CustomerActive` 切换到 `dbo.CustomerOrderHistory` 之前，你可以使用清单 17-12 中的查询来检查记录数。

```sql
SELECT COUNT(*)
FROM dbo.CustomerOrderActive;
SELECT COUNT(*)
FROM dbo.CustomerOrderHistory;
-- 清单 17-12
-- 非分区表和分区表中的记录数
```

这两个查询的结果显示在表 17-3 中。

表 17-3

每个表的记录数

| 表名 | 记录数 |
| --- | --- |
| `dbo.CustomerOrderActive` | 802,821 |
| `dbo.CustomerOrderNonPartition` | 0 |

记录数显示 `dbo.CustomerOrderActive` 表中有 802,821 条记录，而 `dbo.CustomerOrderNonPartition` 中有零条记录。

在尝试切换分区之前，你需要确认 `dbo.CustomerOrderActive` 中的所有记录都存在于你将在 `dbo.CustomerOrderNonPartition` 中使用的分区内。为此，你需要向 `dbo.CustomerOrderActive` 添加检查约束。清单 17-13 显示了用于添加检查约束的 T-SQL。



