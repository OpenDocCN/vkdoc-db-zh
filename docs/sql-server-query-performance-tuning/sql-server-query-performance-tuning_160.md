# 第 19 章 ■ 减少查询资源使用

## 减少日志磁盘写入

--插入 10000 行数据
```sql
DECLARE @Count INT = 1;

WHILE @Count <= 10000
BEGIN
    INSERT INTO dbo.Test1 (C1)
    VALUES (@Count % 256);
    SET @Count = @Count + 1;
END
```

由于每次执行`INSERT`语句本身都是原子性的，SQL Server 会对每次`INSERT`语式的执行都写入事务日志。

减少日志磁盘写入次数的一个简单方法是将操作查询包含在一个显式事务中。
```sql
DECLARE @Count INT = 1;

DBCC SQLPERF(LOGSPACE);

BEGIN TRANSACTION

WHILE @Count <= 10000
BEGIN
    INSERT INTO dbo.Test1 (C1)
    VALUES (@Count % 256);
    SET @Count = @Count + 1;
END

COMMIT

DBCC SQLPERF(LOGSPACE);
```

定义的事务范围（在`BEGIN TRANSACTION`和`COMMIT`命令对之间）将原子性的范围扩展到事务内包含的多个`INSERT`语句。这减少了日志磁盘写入的次数，并提高了数据库功能的性能。为了验证这个理论，请在每个`WHILE`循环前后运行以下 T-SQL 命令。
```sql
DBCC SQLPERF(LOGSPACE);
```

这将向你显示日志空间的使用百分比。在我的数据库上运行第一组插入时，日志使用率从 2.6%上升到 29%。运行第二组插入时，日志增长了约 6%。

最佳方法是处理数据集而不是单个行。`WHILE`循环可能本身就是一种成本高昂的操作，类似于游标（关于游标的更多细节见第 22 章）。因此，运行一个避免使用`WHILE`循环，而是采用基于集合的方法的查询会更好。
```sql
DECLARE @Count INT = 1;

BEGIN TRANSACTION

WHILE @Count <= 10000
BEGIN
    INSERT INTO dbo.Test1 (C1)
    VALUES (@Count % 256);
    SET @Count = @Count + 1;
END

COMMIT
```

在前后运行`DBCC SQLPERF()`函数执行此查询显示，日志内已用空间的增长不到 4%，并且它在 41 毫秒内运行完成，而`WHILE`循环则需要超过 2 秒。

然而，需要注意的一个问题是，在事务中包含过多的数据操作查询会增加事务的持续时间。在此期间，所有试图访问事务中涉及的资源的其他查询都会被阻塞。长事务还会导致回滚持续时间和恢复期间的恢复时间增加。

### 减少锁开销

默认情况下，所有四种 SQL 语句（`SELECT`、`INSERT`、`UPDATE`和`DELETE`）都使用数据库锁将其工作与其他 SQL 语句的工作隔离开来。这种锁管理为查询增加了性能开销。通过请求更少的锁可以提高查询的性能。推而广之，其他查询的性能也会得到改善，因为它们需要等待获取自身锁的时间更短。

默认情况下，SQL Server 可以提供行级锁。对于处理大量行的查询，为所有单独的行请求行锁会给锁管理过程增加显著的开销。你可以通过降低锁的粒度（例如页级或表级）来减少这种锁开销。SQL Server 会动态考虑锁开销来执行锁升级。因此，通常不需要手动升级锁级别。但是，如果需要，可以使用锁提示以编程方式控制查询的并发性，如下所示：
```sql
SELECT * FROM <TableName> WITH(PAGLOCK) --使用页级锁
```

类似地，默认情况下，SQL Server 除了对`INSERT`、`UPDATE`和`DELETE`语句使用锁外，也对`SELECT`语句使用锁。这允许`SELECT`语句读取未被修改的数据。在某些情况下，数据可能相当静态，不会经历太多修改。在这种情况下，你可以通过以下方式之一来减少`SELECT`语句的锁开销：
- 将数据库标记为 `READONLY`（只读）。
  ```sql
  ALTER DATABASE <DatabaseName> SET READ_ONLY
  ```



