# 第 25 章：查询优化与执行

系统的可维护性是另一个非常重要的因素。您应该记录使用提示（hints）的情况，并定期重新评估它们是否仍然必要。数据量和数据分布的变化可能导致由提示强制的执行计划变得次优。例如，考虑一种情况，提示强制查询优化器使用嵌套循环联接。随着数据量和输入集的增长，这种联接类型的效率会越来越低。

强制查询优化器使用特定索引是另一个例子。在数据选择性变化的情况下，索引的选择可能变得低效，并且这会阻止查询优化器使用后来创建的其他索引。此外，如果您删除了提示中引用的索引或对其重命名，代码会出错，查询将无法执行。

作为一般规则，您应该仅在最后万不得已的情况下才使用提示。如果使用，请确保统计信息是最新的，并且在应用提示之前，查询无法再被优化、简化或重构。

在参数嗅探（parameter sniffing）的情况下，通常最好使用 `OPTIMIZE FOR` 提示或语句级重编译，而不是使用索引提示来强制使用特定索引。我们将在下一章更深入地讨论这些方法。

## INDEX 表提示

`INDEX` 可能是最常用的表提示之一。它强制查询优化器使用特定索引进行数据访问。它要求您指定索引的名称或 ID 作为参数。在大多数情况下，出于可维护性考虑，索引名称是更好的选择。然而，有两个例外情况，使用索引 ID 是更好的选择：强制聚集索引或堆表扫描。在这些情况下，您可以考虑分别使用 `1` 和 `0` 作为 ID。

SQL Server 可以对索引使用 `扫描（Scan）` 或 ` Seek ` 访问方法。代码清单 25-18 展示了一个使用 `INDEX` 提示的例子，它强制 SQL Server 在查询中使用 `IDX_Orders_OrderDate` 索引。

***代码清单 25-18.*** `INDEX` 查询提示

```sql
select OrderId, OrderDate, CustomerID, Total
from dbo.Orders with (Index = IDX_Orders_OrderDate)
where OrderDate between @StartDate and @EndDate
```

`INDEX` 查询提示的一个合法使用场景是，在无法进行正确的基数估计的情况下，强制 SQL Server 使用复合索引之一。考虑一个表存储属于不同账户的多台设备的位置信息，如代码清单 25-19 所示。我们假设 `DeviceId` 仅在单个账户内是唯一的。

***代码清单 25-19.*** 复合索引与不均匀数据分布：表创建

```sql
create table dbo.Locations
(
    AccountId int not null,
    DeviceId int not null,
    UtcTimeTag datetime2(0) not null,
    /* 其他列 */
);

create unique clustered index IDX_Locations_AccountId_UtcTimeTag_DeviceId
    on dbo.Locations(AccountId, UtcTimeTag, DeviceId);

create unique nonclustered index IDX_Locations_AccountId_DeviceId_UtcTimeTag
    on dbo.Locations(AccountId, DeviceId, UtcTimeTag);
```

在多租户系统中，数据分布非常不均匀是很常见的，有些账户拥有数百甚至数千台设备，而其他账户只有少数几台。假设我们希望为特定时间范围选择属于设备子集的数据，如代码清单 25-20 所示。

***代码清单 25-20.*** 复合索引与不均匀数据分布：查询

```sql
select DeviceId, UtcTimeTag /* 其他列 */
from dbo.Locations
where
    AccountId = @AccountID and
    UtcTimeTag between @StartTime and @StopTime and
    DeviceID in (select DeviceID from #ListOfDevices);
```

SQL Server 对于执行计划有两种不同的选择。第一种选择使用 `非聚集索引 Seek` 和 `键查找（key lookup）`，当您需要选择账户中非常小一部分设备的数据时，这种方式更好。在所有其他情况下，使用以 `AccountId` 和 `UtcTimeTag` 作为 Seek 谓词的 `聚集索引 Seek`，并对属于该账户的所有设备执行范围扫描，会更高效。

不幸的是，SQL Server 没有足够的数据来在任一情况下进行正确的基数估计。它可以根据任一索引的直方图来估计特定 `AccountID` 数据的选择性；然而，这对于估计设备列表的基数是不够的。

一种可能的解决方案是编写代码来计算 `#ListOfDevices` 表中的设备数量，并将其与每个账户的设备总数进行比较，然后根据比较结果使用 `INDEX` 提示强制 SQL Server 使用特定索引。

值得一提的是，这种系统设计并不理想。最好使 `DeviceId` 在全局范围内唯一，而不仅仅在账户范围内。这将允许您将 `DeviceId` 作为非聚集索引中最左边的列，这将帮助 SQL Server 基于设备列表进行基数估计。然而，这种方法仍然不会将时间参数纳入此类估计中。

## FORCE ORDER 提示

`FORCE ORDER` 查询提示保留查询中的联接顺序。指定此提示后，SQL Server 将始终按照查询 `from` 子句中列出的顺序联接表。但是，SQL Server 会在每种情况下选择开销最小的联接类型。

代码清单 25-21 展示了这种提示的一个例子。SQL Server 将按以下顺序执行联接：`((TableA join TableB) join TableC)`。

***代码清单 25-21.*** `FORCE ORDER` 提示

```sql
select /* 列 */
from
    TableA join TableB on TableA.ID = TableB.AID
    join TableC on TableB.ID = TableC.BID
option (force order)
```

## LOOP, MERGE, 和 HASH JOIN 提示

您可以在查询级别和单个联接级别使用 `LOOP`、`MERGE` 和 `HASH` 提示来指定联接类型。可以在查询提示中指定多个联接类型，并允许 SQL Server 选择开销最小的一个。如果同时指定了联接操作符提示和查询提示，则联接操作符提示优先级更高。最后，联接类型提示会以类似于 `FORCE ORDER` 提示的方式强制联接顺序。

代码清单 25-22 展示了使用联接类型提示的例子。SQL Server 将按以下顺序执行联接：`((TableA join TableB) join TableC)`。它将使用嵌套循环联接来联接 `TableA` 和 `TableB`，并使用嵌套循环或合并联接来联接 `TableC`。

***代码清单 25-22.*** 联接类型提示

```sql
select /* 列 */
from
    TableA inner loop join TableB on TableA.ID = TableB.AID
    join TableC on TableB.ID = TableC.BID
option (loop join, merge join)
```

## FORCESEEK/FORCESCAN 提示

`FORCESEEK` 提示阻止 SQL Server 使用 `索引扫描` 操作符。它可用于查询级别和单个表级别，如果需要，可以与 `INDEX` 提示结合使用。如果无法生成不含索引扫描的执行计划，SQL Server 将生成错误。您还可以为 `SEEK` 谓词指定一个可选的列列表。

相反的提示 `FORCESCAN` 阻止 SQL Server 使用 `索引 Seek` 操作符，并强制其扫描数据。这两个提示都是在 SQL Server 2008 SP1 中引入的。

## NOEXPAND/EXPAND VIEWS 提示

`NOEXPAND` 和 `EXPAND VIEWS` 提示控制 SQL Server 如何处理索引视图。此行为因版本而异。默认情况下，非企业版的 SQL Server 会将索引视图展开为其定义，即使查询中引用了视图，也不会使用其中的数据。您应指定 `NOEXPAND` 提示来避免这种情况。

**提示：** 如果数据库有可能迁移到非企业版 SQL Server，在查询中引用索引视图时，始终指定 `NOEXPAND` 提示。


`清单 25-23`展示了`NOEXPAND`和`INDEX`提示的示例，这些提示强制`SQL Server`使用在索引视图上创建的非聚集索引。

`清单 25-23.` `NOEXPAND`和`INDEX`提示

```
select CustomerID, ArticleId, TotalSales
from
dbo.vArticleSalesPerCustomer
with (NOEXPAND, Index=IDX_vArticleSalesPerCustomer_CustomerID)
where CustomerID = @CustomerID
```

或者，`EXPAND VIEWS`提示允许`SQL Server`在企业版中将索引视图展开为其定义。老实说，我想不出这种行为在哪些使用场景下是有益的。

## `FAST N`提示

`FAST N`提示告诉`SQL Server`生成一个以快速返回指定参数行数为目标的执行计划。即使与使用阻塞操作符的计划相比，该计划成本更高，它也可能生成一个包含非阻塞操作符的执行计划。

这种提示的一个可能使用场景是：某个应用程序在后台加载大量数据（可能在进行缓存），并希望尽快将数据的第一页显示给用户。`清单 25-24`展示了一个使用此类提示的查询示例。

`清单 25-24.` `FAST N`提示

```
select o.OrderId, OrderNumber, OrderData, CustomerId, CustomerName, OrderTotal
from dbo.vOrders
where OrderDate > @StartDate
order by OrderDate desc
option (FAST 50)
```

■ `注意`你可以在 [`technet.microsoft.com/en-us/library/ms181714.aspx`](http://technet.microsoft.com/en-us/library/ms181714.aspx) 查看查询和表提示的完整列表。

#### 总结

查询生命周期包含四个不同的阶段：解析、绑定、优化和执行。查询使用树状结构进行多次转换，从解析阶段的逻辑查询树开始，到优化后的执行计划结束。

查询优化分几个阶段进行。除了简单计划搜索外，`SQL Server`使用基于成本的模型，评估访问方法的成本、资源使用情况以及其他一些因素。

执行计划的质量很大程度上取决于输入数据的正确性。准确且最新的统计信息是改进基数估计的关键因素，它使得`SQL Server`能够生成高效的执行计划。然而，与任何模型一样，它也有局限性。在某些情况下，你需要重构、拆分和简化查询以克服这些限制。

