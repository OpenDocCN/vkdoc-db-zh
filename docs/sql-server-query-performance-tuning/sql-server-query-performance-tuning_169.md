# 第 20 章 ■ 阻塞与被阻塞的进程

键范围模式仅在隔离级别设置为**可序列化**时适用（你将在后面的“隔离级别”部分了解更多关于事务隔离级别的信息）。键范围锁应用于一系列或某个范围内的键值，这些键值在事务打开期间会被反复使用。在可序列化事务期间锁定一个范围，可确保其他行不会插入到该范围内，从而可能改变事务内的结果集。可以使用其他锁模式来锁定范围，这使得它更像是一种组合锁模式，而非一种独特独立的锁模式。要使键范围锁模式生效，必须使用索引来定义范围内的值。

### 锁兼容性

SQL Server 通过防止其他事务以不兼容的方式访问同一资源，来为事务提供隔离。但是，如果一个事务尝试对同一资源执行兼容的操作，那么，为了提高并发性，它不会被第一个事务阻塞。SQL Server 通过防止事务获取由另一事务持有的资源上的不兼容锁，来确保这种选择性阻塞。例如，一个事务在资源上获取的`(S)`锁，允许其他事务在同一资源上获取`(S)`锁。然而，一个事务在资源上获取的`(Sch-M)`锁会阻止其他事务在该资源上获取任何锁。

## 隔离级别

上一节介绍的锁模式有助于事务保护其数据一致性免受其他并发事务的影响。事务获得的数据保护或隔离程度不仅取决于锁模式，还取决于事务的隔离级别。这个级别会影响锁模式的行为。

例如，默认情况下，`(S)`锁在数据被读取后立即释放；它不会一直持有到事务结束。这种行为可能不适合某些应用功能。在这种情况下，你可以配置事务的隔离级别以达到所需的隔离程度。

SQL Server 实现了六种隔离级别，其中四种由 ISO 定义：

• `未提交读`

• `已提交读`

• `可重复读`

• `可序列化`

另外两个隔离级别提供**行版本控制**，这是一种机制，其中行的版本作为数据操作查询的一部分被创建。这个额外的行版本允许读取查询访问数据，而无需对其获取锁。另外两个隔离级别如下：

• `已提交读快照`（实际上是`已提交读`隔离的一部分）

• `快照`

四种 ISO 隔离级别按隔离程度递增的顺序列出。你可以使用`SET TRANSACTION ISOLATION LEVEL`语句在连接级别配置它们，或使用锁提示在查询级别配置。连接级别的隔离级别配置在通过`SET`语句重新配置隔离级别或连接关闭之前一直有效。所有隔离级别将在后续章节中解释。

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 未提交读

`未提交读`是四种隔离级别中最低的一级，它允许`SELECT`语句读取数据而无需请求`(S)`锁。由于`SELECT`语句不请求`(S)`锁，它既不会被`(X)`锁阻塞，也不会阻塞`(X)`锁。它允许`SELECT`语句在数据被修改时读取数据。这种数据读取被称为`脏读`。

假设你有一个应用程序，其中数据修改量极少，并且你的应用程序对其发出的用于读取数据的`SELECT`语句不需要太高的准确性。在这种情况下，你可以使用`未提交读`隔离级别，以避免其他数据修改活动阻塞`SELECT`语句。


你可以使用以下 `SET` 语句将数据库连接的隔离级别配置为 **读未提交** 隔离级别：

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED
```

你也可以使用 `NOLOCK` 锁定提示，在单个查询的基础上实现这种程度的隔离。

```sql
SELECT *
FROM Production.Product AS p WITH (NOLOCK);
```

此锁定提示的效果仅适用于该查询，并不会更改连接的隔离级别。

**读未提交**隔离级别避免了由 `SELECT` 语句引起的阻塞，但如果事务依赖于 `SELECT` 语句所读取数据的准确性，或者事务无法承受其他事务并发更改数据，则不应使用它。

理解什么是**脏读**非常重要。很多人认为这意味着，当某个字段的值正从 `Tusa` 更新为 `Tulsa` 时，查询仍然可以在提交前读取到旧值甚至是更新中的值。虽然这是事实，但可能发生的更严重的数据问题还不止于此。由于读取数据时不放置任何锁，索引可能会发生分裂。这可能导致查询返回额外或缺失的数据行。需要明确的是，在同时发生数据操作和数据读取的任何环境中使用 **读未提交**，都可能导致不可预期的行为。此隔离级别旨在用于主要侧重于报表和商业智能的系统，而非在线事务处理系统。

### 读已提交

**读已提交**隔离级别可以防止由 **读未提交** 隔离级别引起的脏读。这意味着在此隔离级别下，`SELECT` 语句会请求 `(S)` 锁。这是 SQL Server 的默认隔离级别。如果需要，你可以使用以下 `SET` 语句将连接的隔离级别更改为 **读已提交**：

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED
```

**读已提交**隔离级别在大多数情况下都适用，但由于 `SELECT` 语句获取的 `(S)` 锁不会一直持有到事务结束，它可能会导致不可重复读或幻读问题，如后续章节所述。

**读已提交**隔离级别的行为可以通过 `READ_COMMITTED_SNAPSHOT` 数据库选项来改变。当此选项设置为 `ON` 时，数据操作事务将使用行版本控制。这会对 `tempdb` 产生额外负载，因为在事务未提交期间，被更改行的先前版本会存储在那里。

这使得其他事务可以在无需对数据放置锁的情况下读取数据，从而在不引起像使用 `NOLOCK` 或 **读未提交** 时因页分裂而导致问题的前提下，提升系统中所有查询的速度和效率。

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 第 20 章 ■ 阻塞与被阻塞的进程

接下来，修改 `AdventureWorks2012` 数据库以开启 `READ_COMMITTED_SNAPSHOT`。

```sql
ALTER DATABASE AdventureWorks2012
SET READ_COMMITTED_SNAPSHOT ON;
```

现在设想一个业务场景。第一个连接和事务将从 `Production.Product` 表中提取数据，获取特定项目的颜色。

```sql
BEGIN TRANSACTION;
SELECT p.Color
FROM Production.Product AS p
WHERE p.ProductID = 711;
```

建立第二个连接并开启一个新事务，该事务将修改同一项目的颜色。

```sql
BEGIN TRANSACTION ;
UPDATE Production.Product
SET Color = 'Coyote'
WHERE ProductID = 711;

SELECT p.Color
FROM Production.Product AS p
WHERE p.ProductID = 711;
```

在更新颜色后运行 `SELECT` 语句，你可以看到颜色已被更新。但如果你切换回第一个连接并重新运行原始的 `SELECT` 语句（不要再次运行 `BEGIN TRAN` 语句），你仍然会看到颜色是 `Blue`。切换回第二个连接并完成该事务。

```sql
COMMIT TRANSACTION;
```

再次切换到第一个事务，提交该事务，然后重新运行原始的 `SELECT` 语句。



### 可重复读

`可重复读`隔离级别允许`SELECT`语句将其共享锁`(S) lock`保持到事务结束，从而防止其他事务在此期间修改数据。数据库功能可能在事务内部，基于该事务中`SELECT`语句读取的数据，实现一个逻辑决策。如果决策的结果取决于`SELECT`语句读取的数据，那么你应该考虑防止其他并发事务修改数据。例如，考虑以下两个事务：
*   **规范化价格**：对于`ProductID = 1`，如果`Price > 10`，则将价格降低 10。
*   **应用折扣**：对于`Price > 10`的产品，应用 40%的折扣。

现在考虑以下测试表：
```sql
IF (SELECT OBJECT_ID('dbo.MyProduct')
) IS NOT NULL
DROP TABLE dbo.MyProduct ;
GO
CREATE TABLE dbo.MyProduct
(ProductID INT,
Price MONEY
) ;
INSERT INTO dbo.MyProduct
VALUES (1, 15.0) ;
```
你可以这样编写这两个事务：
```sql
DECLARE @Price INT ;
BEGIN TRAN NormailizePrice
SELECT @Price = mp.Price
FROM dbo.MyProduct AS mp
WHERE mp.ProductID = 1 ;
/*Allow transaction 2 to execute*/
WAITFOR DELAY '00:00:10' ;
IF @Price > 10
UPDATE dbo.MyProduct
SET Price = Price - 10
WHERE ProductID = 1 ;
COMMIT

--Transaction 2 from Connection 2
BEGIN TRAN ApplyDiscount
UPDATE dbo.MyProduct
SET Price = Price * 0.6 --Discount = 40%
WHERE Price > 10 ;
COMMIT
```
表面上，前面的事务看起来没问题，在单用户环境中确实可以工作。但在多用户环境中，当多个事务可以并发执行时，这里就出现了问题！

为了找出问题，让我们按照以下顺序从不同连接执行这两个事务：1. 先启动事务 1。2. 在事务 1 启动后的十秒内启动事务 2。正如你可能猜到的，事务结束时，产品（`ProductID = 1`）的新价格将是`-1.0`。哎呀——这看起来你准备好要破产了！

问题出现的原因是，事务 2 被允许在事务 1 读取完数据并即将基于数据做出决策时修改数据。事务 1 需要比默认隔离级别（`已提交读`）提供的更高的隔离度。

作为解决方案，你希望在事务 1 处理数据时，防止事务 2 修改它。换句话说，为事务 1 提供稍后在事务中再次读取数据而不被他人修改的能力。这个特性被称为`可重复读`。考虑到上下文，解决方案的实现可能是显而易见的。重新创建示例表后，你可以这样写：
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ ;
GO
--Transaction 1 from Connection 1
DECLARE @Price INT ;
BEGIN TRAN NormalizePrice
SELECT @Price = Price
FROM dbo.MyProduct AS mp
WHERE mp.ProductID = 1 ;
/*Allow transaction 2 to execute*/
WAITFOR DELAY '00:00:10' ;
IF @Price > 10
UPDATE dbo.MyProduct
SET Price = Price - 10
WHERE ProductID = 1 ;
COMMIT
GO
SET TRANSACTION ISOLATION LEVEL READ COMMITTED --Back to default
GO
```

`注意`：
■
如果`tempdb`已满，使用行版本控制的数据修改将继续成功，但读取可能会失败，因为版本化行将不可用。如果你在数据库中启用任何类型的行版本控制隔离，你必须格外注意在`tempdb`中维护可用空间。

`www.it-ebooks.info`



