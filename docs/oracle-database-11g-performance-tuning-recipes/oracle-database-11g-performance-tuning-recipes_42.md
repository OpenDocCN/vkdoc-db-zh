# 15-5. 并行创建表

## 问题

你需要从现有的大型表中快速创建一个新表，并希望利用多进程来帮助加速表的创建。

## 解决方案

如果你正在管理超大型数据库，或者需要重建大型表，并行 DDL 速度快，并且比运行并行 DML 更有优势。速度是选择使用并行 DDL 从现有大型表创建表的最大因素。在你的特定 DDL 命令中，有一个 `PARALLEL` 子句来决定是否并行执行操作。这是通过使用 `CREATE TABLE ... AS SELECT` 操作完成的：

```
CREATE TABLE EMP_COPY
PARALLEL(DEGREE 4)
AS
SELECT * FROM EMP;
```

```
------------------------------------------------------------------------
| Id  | Operation              | Name     |    TQ  |IN-OUT| PQ Distrib |
------------------------------------------------------------------------
|   0 | CREATE TABLE STATEMENT |          |        |      |            |
|   1 |  PX COORDINATOR        |          |        |      |            |
|   2 |   PX SEND QC (RANDOM)  | :TQ10000 |  Q1,00 | P->S | QC (RAND)  |
|   3 |    LOAD AS SELECT      | EMP_COPY |  Q1,00 | PCWP |            |
|   4 |     PX BLOCK ITERATOR  |          |  Q1,00 | PCWC |            |
|   5 |      TABLE ACCESS FULL | EMP      |  Q1,00 | PCWP |            |
------------------------------------------------------------------------
```



