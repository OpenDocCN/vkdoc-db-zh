# 并行执行

## 插入操作的并行化
仅操作 2 到 5 并行执行。查询协调器扫描表 `t1`，并使用循环（round-robin）方法将其内容分发给从属进程。由于查询协调器与多个从属进程通信，因此操作之间的关系为串行到并行（`S->P`）。一组从属进程接收数据并并行执行插入操作（操作 `LOAD AS SELECT`）。

```sql
CREATE TABLE t2 PARALLEL 2 AS SELECT /*+ no_parallel(t1) */ * FROM t1
```
```
-------------------------------------------------------------------
|Id | Operation                 | Name   |  TQ  |IN-OUT|PQ Distrib |
-------------------------------------------------------------------
|  0| CREATE TABLE STATEMENT    |        |      |      |           |
|  1|  PX COORDINATOR           |        |      |      |           |
|  2|   PX SEND QC (RANDOM)     |:TQ10001| Q1,01| P->S | QC (RAND) |
|  3|    LOAD AS SELECT         |        | Q1,01| PCWP |           |
|  4|     BUFFER SORT           |        | Q1,01| PCWC |           |
|  5|      PX RECEIVE           |        | Q1,01| PCWP |           |
|  6|       PX SEND ROUND-ROBIN |:TQ10000|      | S->P | RND-ROBIN |
|  7|        TABLE ACCESS FULL  | T1     |      |      |           |
-------------------------------------------------------------------
```

## 查询操作的并行化
仅操作 3 到 5 并行执行。从属进程基于块范围粒度并行扫描表 `t1`，并将其内容发送给查询协调器，这是并行到串行（`P->S`）关系的原因。查询协调器执行插入操作（操作 `LOAD AS SELECT`）。

```sql
CREATE TABLE t2 NOPARALLEL AS SELECT /*+ parallel(t1 2) */ * FROM t1
```
```
-----------------------------------------------------------------
| Id | Operation               | Name   |  TQ  |IN-OUT|PQ Distrib|
-----------------------------------------------------------------
|   0| CREATE TABLE STATEMENT  |        |      |      |          |
|   1|  LOAD AS SELECT         |        |      |      |          |
|   2|   PX COORDINATOR        |        |      |      |          |
|   3|    PX SEND QC (RANDOM)  |:TQ10000| Q1,00| P->S | QC (RAND)|
|   4|     PX BLOCK ITERATOR   |        | Q1,00| PCWC |          |
|   5|      TABLE ACCESS FULL  | T1     | Q1,00| PCWP |          |
-----------------------------------------------------------------
```

## 两个操作的并行化
从属进程基于块范围粒度并行扫描表 `t1`，并直接将他们获得的数据插入到目标表中。需要强调两点。首先，查询协调器不直接参与数据处理。其次，数据不通过表队列发送（除了操作 2 发送给查询协调器的少量可忽略信息外，没有发生通信）。

```sql
CREATE TABLE t2 PARALLEL 2 AS SELECT /*+ parallel(t1 2) */ * FROM t1
```
```
-----------------------------------------------------------------
| Id | Operation               | Name   |  TQ |IN-OUT| PQ Distrib|
-----------------------------------------------------------------
|   0| CREATE TABLE STATEMENT  |        |     |      |           |
|   1|  PX COORDINATOR         |        |     |      |           |
|   2|   PX SEND QC (RANDOM)   |:TQ10000|Q1,00| P->S | QC (RAND) |
|   3|    LOAD AS SELECT       |        |Q1,00| PCWP |           |
|   4|     PX BLOCK ITERATOR   |        |Q1,00| PCWC |           |
|   5|      TABLE ACCESS FULL  | T1     |Q1,00| PCWP |           |
-----------------------------------------------------------------
```

