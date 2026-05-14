# 14. 日志记录

在你与 SQL Server 打交道期间，会收到来自公司的各种各样的需求。其中一些需求可能涉及更改业务逻辑或添加新功能。有时候，公司可能还想追踪过去发生的事情。有多种第三方工具有助于跟踪性能及与数据库维护相关的其他功能。然而，你的业务团队可能更关注于跟踪数据变更或了解问题出在哪里。

这种与数据修改或错误处理相关的日志记录可以通过 T-SQL 来实现。在记录这些类型的更改时，你有多种选项可供选择。这些选项包括从最简略的活动日志到记录最细粒度的活动日志。你还需要考虑这些信息将来将如何被访问和使用。这将帮助你以一种对未来组织有益的方式记录信息。这也可以防止你记录下最终从未被使用的数据。

## 数据修改

本节提供如何跟踪数据库内数据变更的示例。本节涵盖了设置变更跟踪以及使用变更跟踪时可以收集的信息类型。还有一部分内容关于启用变更跟踪以及访问从变更跟踪生成的数据。

选择跟踪数据修改的方法需要了解你的组织需要跟踪哪些类型的信息。你可能只想在数据发生变更时记录信息。根据你的业务场景，你可能不仅需要知道数据何时变更，还需要知道具体什么发生了变化。关于如何跟踪这些变更也有几个选项。你可以选择让 SQL Server 为你跟踪这些变更，也可以创建数据库对象来为你记录这些信息。

在实施任何类型的日志记录时，需要考虑的事情之一是性能开销。在跟踪数据修改时，这一点可能更为关键。你需要确保选择的日志记录方法既能跟踪必要的变更，又具有可预期的性能影响。正如你可能预料到的，你实施的日志记录越详细，性能开销就越大。

在本章讨论的所有选项中，**变更跟踪**的开销最小。如果你希望记录更详细的信息并且愿意增加资源消耗，你可能需要考虑**变更数据捕获**。另一种跟踪变更的实现方式涉及使用数据库触发器。虽然使用数据库触发器可以让你精细地调整记录内容和记录方式，但这可能会以更高的资源消耗为代价。

根据你想记录的数据类型，一个中间选项可能是使用 **SQL Server 审核**。SQL Server 审核可用于跟踪服务器或数据库级别的活动。在记录与应用程序相关的变更的上下文中，你应该专注于数据库审核。SQL Server 审核中的数据库审核操作可以跟踪与所有类型数据活动相关的更改，包括访问或修改数据。SQL Server 使用扩展事件来监控审核活动。可检索的数据包括操作发生的时间、触发审核的用户信息以及受影响的对象。也可能记录审核操作发生时所执行的语句。

关于数据修改，另一个可用的最简略日志记录活动涉及记录记录最后一次更改的时间，并递增该数据记录发生的更改次数。这种类型的跟踪由 SQL Server 内置的**变更跟踪**功能处理，该功能将自动记录表中数据的更改，并在自定义表对象中报告这些更改。要使用变更跟踪，你首先需要在数据库上启用它。在本书中，我们一直使用 `Menu` 数据库。你需要运行清单 14-1 所示的 T-SQL 代码来在 `Menu` 数据库上启用变更跟踪。

```
ALTER DATABASE OutdoorRecreation
SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 2 DAYS, AUTO_CLEANUP = ON);
```
清单 14-1
在 OutdoorRecreation 数据库上启用变更跟踪

要在 `Menu` 数据库上启用变更跟踪，你需要指定数据库名称并表明你希望开启变更跟踪。前面 T-SQL 代码中的最后一行是可选的。这些值表示你希望保留更改记录的时间，以及是否应自动清理保留的历史记录。


一旦您在 `Menu` 数据中启用了变更跟踪，就可以配置表以使用变更跟踪。您要实施变更跟踪的表必须具有主键。如果该表没有主键，您必须先添加一个，然后才能在该表上启用变更跟踪。在此情况下，您将在 `dbo.CustomerOrder` 表上启用变更跟踪；请参考代码清单 14-2。

```sql
ALTER TABLE dbo.CustomerOrder
ENABLE CHANGE_TRACKING
WITH (TRACK_COLUMNS_UPDATED = ON);
```

**代码清单 14-2**
在 `dbo.CustomerOrder` 表上启用变更跟踪

现在，`dbo.CustomerOrder` 表已启用变更跟踪。变更跟踪可以记录整个数据行已更改，也可以指定特定列已更改。当您跟踪列更改时，SQL Server 会记录发生了更改以及哪些特定列已更改。如果您没有为 `TRACK_COLUMNS_UPDATED` 指定值，SQL Server 将使用默认值 `OFF`，并且只会告诉您某行已添加到表中、从表中删除或在表内被修改，而不会提供更多细节。

设置好变更跟踪后，您可能想确定有哪些信息可用或已更改。虽然 SQL Server 会跟踪 `dbo.CustomerOrder` 表中的更改，但这些信息无法从您可以在对象资源管理器中找到的表中获得。相反，您需要访问与 `dbo.CustomerOrder` 表关联的 `CHANGETABLE`。一旦启用了变更跟踪，记录就会被初始化，并可以在 `CHANGETABLE` 中找到。代码清单 14-3 中的查询展示了如何查找这些已初始化的记录。

```sql
SELECT ord.CustomerOrderID,
ord.OrderNumber,
ord.OrderDate,
chng.CustomerOrderID,
chng.SYS_CHANGE_VERSION,
chng.SYS_CHANGE_CONTEXT
FROM dbo.CustomerOrder AS ord
CROSS APPLY CHANGETABLE
(VERSION CustomerOrder,
(CustomerOrderID), (ord.CustomerOrderID)) AS chng;
```

**代码清单 14-3**
查询已初始化的记录

此查询返回来自 `dbo.CustomerOrder` 表和 `CHANGETABLE` 的信息。表 14-1 显示了已初始化记录的示例。

**表 14-1**
已初始化的记录

| CustomerOrderID | SYS_CHANGE_VERSION | SYS_CHANGE_CONTEXT |
| --- | --- | --- |
| 1 | NULL | NULL |
| 2 | NULL | NULL |
| 3 | NULL | NULL |
| 4 | NULL | NULL |

表 14-1 中的记录来自 `CHANGETABLE`。这里有一列是 `CustomerOrderID`（表的主键），以及用于跟踪更改信息的列。由于没有任何数据记录被修改，所有值均为 NULL。

您可以修改 `dbo.CustomerOrder` 表中的记录。在代码清单 14-4 中，您更新了 `DateModified`。

```sql
UPDATE dbo.CustomerOrder
SET ShipDate = SYSDATETIME()
WHERE CustomerOrderID = 3;
```

**代码清单 14-4**
更新 `DateModified`

既然记录已被修改，您就能更好地了解在涉及变更跟踪时信息是如何存储的。如果您再次运行代码清单 14-3 中的查询，将得到表 14-2 所示的结果。

**表 14-2**
更改 `ShipDate` 后的记录

| CustomerOrderID | SYS_CHANGE_VERSION | SYS_CHANGE_CONTEXT |
| --- | --- | --- |
| 1 | NULL | NULL |
| 2 | NULL | NULL |
| 3 | 1 | NULL |
| 4 | NULL | NULL |

该表显示 `SYS_CHANGE_VERSION` 已从初始值 NULL 更改为 1。这表明自变更跟踪最初初始化以来，`CustomerOrderID` 为 3 的记录已被修改。

如果您想查找有关哪些列被更改的更多信息，代码清单 14-5 展示了查找自上次自动清理以来 `dbo.CustomerOrder` 发生更改的信息所需的查询。

```sql
SELECT CustomerOrderID,
SYS_CHANGE_OPERATION AS ChangeOperation,
CHANGE_TRACKING_IS_COLUMN_IN_MASK
(COLUMNPROPERTY
(OBJECT_ID('CustomerOrder'), 'OrderNumer', 'ColumnId'),
SYS_CHANGE_COLUMNS) AS OrderNumberChange,
CHANGE_TRACKING_IS_COLUMN_IN_MASK
(COLUMNPROPERTY
(OBJECT_ID('CustomerOrder'), 'OrderDate', 'ColumnId'),
SYS_CHANGE_COLUMNS) AS OrderDateChange,
CHANGE_TRACKING_IS_COLUMN_IN_MASK
(COLUMNPROPERTY
(OBJECT_ID('CustomerOrder'), 'ShipDate', 'ColumnId'),
SYS_CHANGE_COLUMNS) AS ShipDateChange,
CHANGE_TRACKING_IS_COLUMN_IN_MASK
(COLUMNPROPERTY
(OBJECT_ID('CustomerOrder'), 'DateCreated', 'ColumnId'),
SYS_CHANGE_COLUMNS) AS CreatedChange,
CHANGE_TRACKING_IS_COLUMN_IN_MASK
(COLUMNPROPERTY
(OBJECT_ID('CustomerOrder'), 'DateModified', 'ColumnId'),
SYS_CHANGE_COLUMNS) AS ModifiedChange,
CHANGE_TRACKING_IS_COLUMN_IN_MASK
(COLUMNPROPERTY
(OBJECT_ID('CustomerOrder'), 'DateDisabled', 'ColumnId'),
SYS_CHANGE_COLUMNS) AS DisabledChange,
SYS_CHANGE_CONTEXT
FROM CHANGETABLE
(CHANGES dbo.CustomerOrder,0) as ChngTbl
ORDER BY SYS_CHANGE_VERSION;
```

**代码清单 14-5**
查找已更改的记录

需要此版本信息来查找此表上更改的状态。代码清单 14-5 中的查询展示了您可用于访问此信息的一种方法。表 14-3 显示了代码清单 14-5 中查询的结果。

**表 14-3**
变更跟踪结果集

| CustomerOrderID | ChangeOperation | OrderNumberChange | OrderDateChange | ShipDateChange | CreatedChange | ModifiedChange |
| --- | --- | --- | --- | --- | --- | --- |
| 3 | U | False | False | True | False | False |

这些结果记录自代码清单 14-4 中发出的更新。

受影响的 `CustomerOrderID` 列在表 14-3 的第一列。这与代码清单 14-4 中更新的 `CustomerOrderID` 相匹配。`ChangeOperation` 列出为 `U`，表示更新。这也与代码清单 14-4 中的 DML 操作相匹配。表 14-3 中的最后五列是使用特定函数填充的，用于揭示变更跟踪表中指示的列。如果列被修改，该函数返回 `True` 值；如果数据未被修改，则返回 `False` 值。在代码清单 14-4 的情况下，唯一更新的列是 `DateModified`。这与表 14-3 中的结果相匹配。`OrderNumberChange`、`OrderDateChange`、`CreatedChange` 和 `DisabledChange` 列均为 `False`。这些值未被更改。然而，`ModifiedChange` 列为 `True`。这表明引用的列 `DateModified` 已被更改。

使用变更跟踪时，在一致性方面存在一些潜在问题。存储数据不影响数据的一致性。数据一致性受影响是数据检索过程的一部分。您可以尽一切可能确保访问变更跟踪表中最新的数据。这包括检查变更跟踪中最后同步的版本，并确认该版本仍然可用。但是，如果变更跟踪中的版本早于保留期，则这些数据可能在检索之前就被清理掉了。使用变更跟踪时，您还可能遇到在最后一次同步之后发生数据修改的情况。这可能导致返回额外的版本或修改的记录。这些情况中的任何一种都可能影响变更跟踪返回数据的一致性。最小化一致性问题的最佳实践是使用快照隔离级别。



