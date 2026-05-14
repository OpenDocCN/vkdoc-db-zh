# 第 8 章 ■ 索引体系结构与行为

```sql
WITH Nums
AS (SELECT TOP (10000)
        ROW_NUMBER() OVER (ORDER BY (SELECT 1)) AS n
    FROM master.sys.all_columns ac1
    CROSS JOIN master.sys.all_columns ac2
)
INSERT INTO dbo.Test1
    (C1, C2, C3)
SELECT n,
    n,
    'C3'
FROM Nums;
```

运行一个 `UPDATE` 语句，如下所示：
```sql
UPDATE dbo.Test1
SET C1 = 1,
    C2 = 1
WHERE C2 = 1;
```

然后 `SET STATISTICS IO` 报告的逻辑读取次数如下：
```
Table 'Test1'. Scan count 1, logical reads 29
```

在列 `c1` 上添加一个索引，如下所示：
```sql
CREATE CLUSTERED INDEX iTest
ON dbo.Test1(C1);
```

对于相同的 `UPDATE` 语句，逻辑读取次数从 29 增加到 42，但还添加了一个工作表，额外增加了 5 次读取，总计 47：
```
Table 'Test1'. Scan count 1, logical reads 42
Table 'Worktable'. Scan count 1, logical reads 5
```

读取次数增加是因为需要重新排列数据，以便在聚集索引中按正确顺序存储，这增加了读取次数，超过了堆表仅将数据添加到现有存储末尾所需的次数。

尽管维护索引所需的开销量确实会因数据操作查询而增加，但请注意，SQL Server 必须先找到一行，然后才能更新或删除它；因此，对于带有必要 `WHERE` 子句的 `UPDATE` 和 `DELETE` 语句，索引可能是有用的。使用索引定位行的效率提高通常抵消了更新索引所需的额外开销，除非表有很多索引。

此外，绝大多数系统都是读取密集型的，这意味着检索的数据量远多于插入或修改的数据量。

为了理解索引如何有益于数据修改查询，让我们基于这个例子继续。在表 `t1` 上创建另一个索引。这次，在 `UPDATE` 语句的 `WHERE` 子句中引用的列 `c2` 上创建索引。
```sql
CREATE INDEX iTest2
ON dbo.Test1(C2);
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

