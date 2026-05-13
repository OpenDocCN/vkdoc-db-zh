# 引言

| 标识 | 操作                     | 名称         | 行数 | 成本 (%CPU) |
|----|-------------------------------|--------------|------|-------------|
| 0  | SELECT STATEMENT              |              |      | 1000M       | 1740K (1)   |
| 1  | NESTED LOOPS                  |              |      |             |
| 2  | NESTED LOOPS                  |              | 1000M| 1740K (1)   |
| 3  | INDEX FULL SCAN               | CUSTOMERS_PK | 55500| 116 (0)     |
| 4  | PARTITION RANGE ALL           |              |      |             |
| 5  | BITMAP CONVERSION TO ROWIDS   |              |      |             |
| 6  | BITMAP INDEX SINGLE VALUE     | SALES_CUST_BIX |    |             |
| 7  | TABLE ACCESS BY LOCAL INDEX ROWID | SALES     | 18018| 1740K (1)   |

对普通读者来说，`SH.SALES` 与 `SH.CUSTOMERS` 的连接似乎没有意义，但现在我们获得了一组按 `CUST_ID` 排序的完整客户数据。维度表主键索引上的 `INDEX FULL SCAN` 以正确的顺序返回客户，对于每个索引条目，我们都可以从 `SH.SALES` 表中选取所需的行。得到的执行计划仍然远非最优。我们必须为每个 `CUST_ID` 访问表中的所有分区，但性能提升可能足以打消你在 `SH.SALES` 上创建全局索引并因此带来维护问题的念头。

