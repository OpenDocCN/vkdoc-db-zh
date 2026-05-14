# 第 20 章 ■ 阻塞与被阻塞进程

## 事务问题简介

在多用户环境中，并发执行事务可能导致数据不一致。例如，以下业务逻辑可能因并发插入而产生错误结果：

```sql
CREATE CLUSTERED INDEX i1 ON dbo.MyEmployees (GroupID);
```

```sql
--Group 10 中的 Employee 1
INSERT INTO dbo.MyEmployees
VALUES (1,10,1000),
--Group 10 中的 Employee 2
(2,10,1000),
--不同组的 Employees 3 & 4
(3,20,1000),
(4,9,1000);
```

上述业务功能可以如下实现：

```sql
DECLARE @Fund MONEY = 100,
    @Bonus MONEY,
    @NumberOfEmployees INT;

BEGIN TRAN PayBonus

SELECT @NumberOfEmployees = COUNT(*)
FROM dbo.MyEmployees
WHERE GroupID = 10;

/*允许事务 2 执行*/
WAITFOR DELAY '00:00:10';

IF @NumberOfEmployees > 0
BEGIN
    SET @Bonus = @Fund / @NumberOfEmployees;

    UPDATE dbo.MyEmployees
    SET Salary = Salary + @Bonus
    WHERE GroupID = 10;

    PRINT 'Fund balance =
' + CAST((@Fund - (@@ROWCOUNT * @Bonus)) AS VARCHAR(6)) + ' $';
END

COMMIT
```

您将看到返回的资金余额为 0 美元，因为更新成功完成。`PayBonus` 事务在单用户环境中运行良好。然而，在多用户环境中，存在问题。

考虑另一个事务，它添加一个新员工到 `GroupID = 10`，如下所示，并与 `PayBonus` 事务并发执行（在 `PayBonus` 事务开始后立即从第二个连接执行）：

```sql
BEGIN TRAN NewEmployee

INSERT INTO MyEmployees
VALUES (5, 10, 1000);

COMMIT
```

`PayBonus` 事务之后的资金余额将是 -$50！尽管新员工可能喜欢这样，但小组基金将出现赤字。这会导致数据逻辑状态的不一致。

## 使用可序列化隔离级别的解决方案

为防止这种数据不一致，应阻止对正在操作的数据集（或数据集）添加新员工。在讨论的五种隔离级别中，只有快照（Snapshot）隔离可以提供类似功能，因为事务不仅需要保护现有数据，还需要防止新数据进入数据集。可序列化（Serializable）隔离级别可以通过在受影响行及 `MyEmployees` 表上基于 `GroupID` 列的 `i1` 索引顺序确定的下一行上获取范围锁来提供这种隔离。因此，可以通过将事务隔离级别设置为 `Serializable` 来防止 `PayBonus` 事务的数据不一致。

请记住首先重新创建表。

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
GO

DECLARE @Fund MONEY = 100,
    @Bonus MONEY,
    @NumberOfEmployees INT;

BEGIN TRAN PayBonus

SELECT @NumberOfEmployees = COUNT(*)
FROM dbo.MyEmployees
WHERE GroupID = 10;

/*允许事务 2 执行*/
WAITFOR DELAY '00:00:10';

IF @NumberOfEmployees > 0
BEGIN
    SET @Bonus = @Fund / @NumberOfEmployees;

    UPDATE dbo.MyEmployees
    SET Salary = Salary + @Bonus
    WHERE GroupID = 10;

    PRINT 'Fund balance =
' + CAST((@Fund - (@@ROWCOUNT * @Bonus)) AS VARCHAR(6)) + ' $';
END

COMMIT
GO

--恢复为默认值
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
GO
```

可序列化隔离级别的效果也可以通过在查询级别使用 `SELECT` 语句上的 `HOLDLOCK` 锁定提示来实现，如下所示：

```sql
DECLARE @Fund MONEY = 100,
    @Bonus MONEY,
    @NumberOfEmployees INT;

BEGIN TRAN PayBonus

SELECT @NumberOfEmployees = COUNT(*)
FROM dbo.MyEmployees WITH (HOLDLOCK)
WHERE GroupID = 10;

/*允许事务 2 执行*/
WAITFOR DELAY '00:00:10';

IF @NumberOfEmployees > 0
BEGIN
    SET @Bonus = @Fund / @NumberOfEmployees

    UPDATE dbo.MyEmployees
    SET Salary = Salary + @Bonus
    WHERE GroupID = 10;

    PRINT 'Fund balance = ' + CAST((@Fund - (@@ROWCOUNT * @Bonus)) AS VARCHAR(6)) + ' $';
END

COMMIT
```

## 分析范围锁

您可以查询 `sys.dm_tran_locks` 来观察 `PayBonus` 事务获取的范围锁，如图 20-6 所示。

**图 20-6.** sys.dm_tran_locks 的输出，显示授予可序列化事务的范围锁

`sys.dm_tran_locks` 的输出显示，在三个索引行上获取了共享范围（RangeS-S）锁：`GroupID = 10` 中的第一个员工、`GroupID = 10` 中的第二个员工以及 `GroupID = 20` 中的第三个员工。

这些范围锁阻止了任何新员工进入 `GroupID = 10`。

刚刚展示的范围锁引入了一些有趣的副作用。

- 在此期间，无法添加任何 `GroupID` 在 10 到 20 之间（不包括 10 和 20）的新员工。例如，尝试添加一个 `GroupID` 为 15 的新员工将被 `PayBonus` 事务阻塞。

    ```sql
    BEGIN TRAN NewEmployee
    INSERT INTO dbo.MyEmployees
    VALUES (6, 15, 1000);
    COMMIT
    ```

- 如果 `PayBonus` 事务的数据集是索引排序中现有数据的最后一个集合，那么所需的数据集最后一行之后的范围锁将获取在表中最后一个可能的数据值上。

    要理解此行为，让我们删除 `GroupID > 10` 的员工，使 `GroupID = 10` 数据集成为聚簇索引（或表）中的最后一个数据集。

    ```sql
    DELETE dbo.MyEmployees
    WHERE GroupID > 10;
    ```

    再次运行更新奖金查询和新增员工查询。图 20-7 显示了 `PayBonus` 事务的 `sys.dm_tran_locks` 输出结果。

    **图 20-7.** sys.dm_tran_locks 的输出，显示授予可序列化事务的扩展范围锁

    如图 20-7 所示，聚簇索引中最后一行（`KEY = ffffffffffff`）上的范围锁将阻止添加所有大于或等于 10 的 `GroupID` 的新员工。您知道锁是在最后一行上，不是因为它在 `sys.dm_tran_locks` 的输出中以可见形式显示，而是因为您之前清空了直到该行的所有内容。例如，尝试添加一个 `GroupID = 999` 的新员工将被 `PayBonus` 事务阻塞。

    ```sql
    BEGIN TRAN NewEmployee
    INSERT INTO dbo.MyEmployees
    VALUES (7, 999, 1000);
    COMMIT
    ```

### 索引对可序列化隔离级别效果的影响

如果表在 `GroupID` 列（即 `WHERE` 子句中的列）上没有索引，您猜会发生什么？在您思考的时候，我将在不同列上重新创建带有聚簇索引的表。

```sql
IF (SELECT OBJECT_ID('dbo.MyEmployees')) IS NOT NULL
    DROP TABLE dbo.MyEmployees;
GO

CREATE TABLE dbo.MyEmployees
(EmployeeID INT,
GroupID INT,
Salary MONEY
);

CREATE CLUSTERED INDEX i1 ON dbo.MyEmployees (EmployeeID);

--Group 10 中的 Employee 1
INSERT INTO dbo.MyEmployees
VALUES (1,10,1000),
--Group 10 中的 Employee 2
(2,10,1000),
--不同组的 Employees 3 & 4
(3,20,1000),
(4,9,1000);
```

现在重新运行更新奖金查询和新增员工查询。图 20-8 显示了 `PayBonus` 事务的 `sys.dm_tran_locks` 输出结果。

**图 20-8.** sys.dm_tran_locks 的输出，显示授予可序列化事务的范围锁（WHERE 子句列上无索引）

如图 20-8 所示，新聚簇索引中最后一行（`KEY = ffffffffffff`）上的范围锁将阻止向表中添加任何新行。我将在本章后面的“索引对可序列化隔离级别效果的影响”一节中讨论这种广泛锁定背后的原因。

