# 第 19 章

## 减少查询资源使用

在前一章中，你专注于以适当利用索引和统计信息的方式编写查询。在本章中，你将确保以不会不恰当地消耗资源的方式编写查询。编写查询有一些方法可以避免使用内存、CPU 和 I/O，也有一些方法会使查询使用的这些资源超出实际所需。我将介绍多种机制，以确保由你控制的查询能够最优地使用资源。

在本章中，我将涵盖以下主题：

• 资源密集度较低的查询设计
• 有效利用过程缓存的查询设计
• 减少网络开销的查询设计
• 降低查询事务成本的技术

## 避免资源密集型查询

许多数据库功能可以使用各种查询技术来实现。你应采取的方法是使用对资源友好且基于集合的查询技术。以下是几种可用于减少查询资源占用的技术：

• 避免数据类型转换。
• 使用 `EXISTS` 而非 `COUNT(*)` 来验证数据存在性。
• 使用 `UNION ALL` 而非 `UNION`。
• 为聚合和排序操作使用索引。
• 避免在批处理查询中使用局部变量。
• 命名存储过程时要谨慎。

我将在后续小节中更详细地介绍这些要点。

### 避免数据类型转换

在某些情况下（定义见联机丛书中庞大的数据类型转换表），`SQL Server` 允许将具有不同但兼容数据类型的值/常量与列的数据进行比较。`SQL Server` 会自动将数据从一种数据类型转换为另一种数据类型。这个过程被称为*隐式数据类型转换*。

尽管隐式转换很有用，但它给查询优化器增加了额外开销。为了提升性能，请使用与所比较列具有相同数据类型的值/常量。

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 19 章 ■ 减少查询资源使用

要理解隐式数据类型转换如何影响性能，请考虑以下示例：

```
IF EXISTS ( SELECT *
FROM sys.objects
WHERE object_id = OBJECT_ID(N'dbo.Test1') )
DROP TABLE dbo.Test1;

CREATE TABLE dbo.Test1 (
Id INT IDENTITY(1,1),
MyKey VARCHAR(50),
MyValue VARCHAR(50));

CREATE UNIQUE CLUSTERED INDEX Test1PrimaryKey ON dbo.Test1 ([Id] ASC);

CREATE UNIQUE NONCLUSTERED INDEX TestIndex ON dbo.Test1 (MyKey);
GO

SELECT TOP 10000
IDENTITY( INT,1,1 ) AS n
INTO #Tally
FROM Master.dbo.syscolumns scl,
Master.dbo.syscolumns sc2;

INSERT INTO dbo.Test1
(MyKey,
MyValue)
SELECT TOP 10000
'UniqueKey' + CAST(n AS VARCHAR),
'Description'
FROM #Tally;

DROP TABLE #Tally;

SELECT t.MyValue
FROM dbo.Test1 AS t
WHERE t.MyKey = 'UniqueKey333';

SELECT t.MyValue
FROM dbo.Test1 AS t
WHERE t.MyKey = N'UniqueKey333';
```

在创建表 `Test1`、在其上创建几个索引并插入一些数据后，定义了两个查询。



