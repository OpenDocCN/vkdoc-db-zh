# 索引的创建与重建

你可以并行创建和重建索引。为此，两组从属进程协同工作。第一组读取要索引的数据。第二组对从第一组接收的数据进行排序并构建索引。以下 SQL 语句是一个示例。请注意第一组如何执行操作 6 到 8（`Q1,00`），第二组执行操作 2 到 5（`Q1,01`）。数据使用范围方法在两组之间分发，并且具有并行到并行（`P->P`）的关系。

```sql
CREATE INDEX i1 ON t1 (id) PARALLEL 4
```
```
-------------------------------------------------------------------
| Id | Operation                 | Name   | TQ  |IN-OUT|PQ Distrib |
-------------------------------------------------------------------
|   0| CREATE INDEX STATEMENT    |        |     |      |           |
|   1|  PX COORDINATOR           |        |     |      |           |
|   2|   PX SEND QC (ORDER)      |:TQ10001|Q1,01| P->S | QC (ORDER)|
|   3|    INDEX BUILD NON UNIQUE | I1     |Q1,01| PCWP |           |
|   4|     SORT CREATE INDEX     |        |Q1,01| PCWP |           |
|   5|      PX RECEIVE           |        |Q1,01| PCWP |           |
|   6|       PX SEND RANGE       |:TQ10000|Q1,00| P->P | RANGE     |
|   7|        PX BLOCK ITERATOR  |        |Q1,00| PCWC |           |
|   8|         TABLE ACCESS FULL | T1     |Q1,00| PCWP |           |
-------------------------------------------------------------------
```

重建索引会得到非常相似的执行计划：

```sql
ALTER INDEX i1 REBUILD PARALLEL 4
```
```
----------------------------------------------------------------------
| Id | Operation                    | Name   | TQ  |IN-OUT|PQ Distrib |
----------------------------------------------------------------------
|   0| ALTER INDEX STATEMENT        |        |     |      |           |
|   1|  PX COORDINATOR              |        |     |      |           |
|   2|   PX SEND QC (ORDER)         |:TQ10001|Q1,01| P->S | QC (ORDER)|
|   3|    INDEX BUILD NON UNIQUE    |I1      |Q1,01| PCWP |           |
|   4|     SORT CREATE INDEX        |        |Q1,01| PCWP |           |
|   5|      PX RECEIVE              |        |Q1,01| PCWP |           |
|   6|       PX SEND RANGE          |:TQ10000|Q1,00| P->P | RANGE     |
|   7|        PX BLOCK ITERATOR     |        |Q1,00| PCWC |           |
|   8|         INDEX FAST FULL SCAN |I1      |Q1,00| PCWP |           |
----------------------------------------------------------------------
```

