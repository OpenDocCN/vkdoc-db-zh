# 第 13 章 ■ 索引碎片

`CREATE TABLE dbo.Test1`
(C1 INT,
C2 INT,
C3 INT,
c4 CHAR(2000)
) ;

`CREATE CLUSTERED INDEX i1 ON dbo.Test1 (C1)` ;

`WITH Nums`
AS (`SELECT 1 AS n`
`UNION ALL`
`SELECT n + 1`

[www.it-ebooks.info](http://www.it-ebooks.info/)

![](img/index-258_1.jpg)

![](img/index-258_2.jpg)

`FROM Nums`
`WHERE n < 21`
)
`INSERT INTO dbo.Test1`
(C1, C2, C3, c4)
`SELECT n,`
n,
n,
'a'
`FROM Nums` ;

`WITH Nums`
AS (`SELECT 1 AS n`
`UNION ALL`
`SELECT n + 1`
`FROM Nums`
`WHERE n < 21`
)
`INSERT INTO dbo.Test1`
(C1, C2, C3, c4)
`SELECT 41 - n,`
n,
n,
'a'
`FROM Nums` ;

如果你查看当前的碎片情况，你会发现它同时存在**内部碎片**和**外部碎片**（图 13-15）。

图 13-15. 内部和外部碎片

你可以使用 `ALTER INDEX REBUILD` 语句来对聚集索引（或表）进行碎片整理。

```
ALTER INDEX i1 ON dbo.Test1 REBUILD;
```

图 13-16 展示了针对 `sys.dm_db_index_physical_stats` 执行标准 `SELECT` 语句的输出结果。

图 13-16. 通过 `ALTER INDEX REBUILD` 解决的碎片问题

[www.it-ebooks.info](http://www.it-ebooks.info/)

