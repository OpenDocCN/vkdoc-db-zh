# 第 17 章：查询重新编译

本章涵盖以下主题：
*   重新编译的优点与缺点
*   如何识别导致重新编译的语句
*   如何分析重新编译的原因
*   必要时避免重新编译的方法

## 重新编译的优点与缺点

查询的重新编译可能有益也可能有害。有时，为查询考虑一个新的处理策略而非重用现有计划可能是有益的，特别是当表中的数据分布（或相应的统计信息）已更改，或表中添加了新索引时。SQL Server 2014 中的重新编译是在语句级别进行的。这增加了过程中可能发生的重新编译总次数，但总体上减少了一般重新编译的影响和开销。语句级重新编译降低了开销，因为它们只重新编译单个语句，而不是过程中的所有语句，而 SQL Server 2000 中的重新编译会导致整个过程被反复重新编译。尽管重新编译的足迹变小了，但在实际情况下，仍需尽可能减少和控制它。

为了理解重新编译现有计划有时为何有益，假设您需要从 `Production.WorkOrder` 表中检索一些信息。存储过程可能如下所示：

```sql
IF (SELECT OBJECT_ID('dbo.WorkOrder')) IS NOT NULL
    DROP PROCEDURE dbo.WorkOrder;
GO
CREATE PROCEDURE dbo.WorkOrder
AS
    SELECT wo.WorkOrderID, wo.ProductID, wo.StockedQty
    FROM Production.WorkOrder AS wo
    WHERE wo.StockedQty BETWEEN 500 AND 700;
```

根据当前索引，作为存储过程计划一部分的 `SELECT` 语句的执行计划会扫描索引 `PK_WorkOrder_WorkOrderID`，如图 17-1 所示。

**图 17-1.** 存储过程的执行计划

此计划保存在过程缓存中，以便在存储过程重新执行时可以重用。但是，如果按如下所示在表上添加了一个新索引，那么现有计划将不再是执行查询的最高效处理策略。

```sql
CREATE INDEX IX_Test ON Production.WorkOrder(StockedQty, ProductID);
```

在这种情况下，花费额外的 CPU 周期来重新编译存储过程以生成更好的执行计划是有益的。

由于索引 `IX_Test` 可以作为 `SELECT` 语句的覆盖索引，通过使用索引 `IX_Test` 代替扫描 `PK_WorkOrder_WorkOrderID`，可以避免书签查找的成本。SQL Server 会自动检测到此更改，并重新编译现有计划以考虑使用新索引的优势。这导致了存储过程（执行时）的一个新执行计划，如图 17-2 所示。

**图 17-2.** 存储过程的新执行计划

SQL Server 会自动检测需要重新编译现有计划的条件。SQL Server 在确定现有计划何时需要重新编译时遵循特定规则。如果查询的特定实现符合重新编译的规则（执行计划老化退出、`SET` 选项更改等），那么每次该语句满足重新编译要求时，它都将被重新编译，并且 SQL Server 可能会也可能不会生成更好的执行计划。

为了观察实际效果，您需要一个不同的存储过程。以下过程返回 `WorkOrder` 表中的所有行：

```sql
IF (SELECT OBJECT_ID('dbo.WorkOrderAll')) IS NOT NULL
    DROP PROCEDURE dbo.WorkOrderAll;
GO
CREATE PROCEDURE dbo.WorkOrderAll
AS
    SELECT *
    FROM Production.WorkOrder AS wo;
```

在执行此过程之前，请删除索引 `IX_Test`。

```sql
DROP INDEX Production.WorkOrder.IX_Test;
```



