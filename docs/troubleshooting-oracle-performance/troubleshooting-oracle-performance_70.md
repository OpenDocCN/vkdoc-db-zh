# 约束的创建与验证

当创建或验证约束（如外键和检查约束）时，必须验证表中已存储的数据。为此，数据库引擎执行递归查询。例如，假设你执行以下 SQL 语句：

```sql
ALTER TABLE t ADD CONSTRAINT t_id_nn CHECK (id IS NOT NULL)
```

递归地，数据库引擎会执行类似以下的查询来验证表中存储的数据（注意，如果查询不返回任何行，则数据是有效的）：

```sql
SELECT rowid
FROM t
WHERE NOT (id IS NOT NULL)
```

因此，如果表 `t` 的并行度为 2 或更高，数据库引擎将并行执行该查询。

***

**注意**：定义在表级别的并行度用于递归查询，无论会话级别的并行查询和 DDL 语句是启用、强制还是禁用。换句话说，SQL 语句 `ALTER SESSION ... PARALLEL` 对递归查询没有影响。

***

当你定义主键约束时，数据库引擎无法并行创建索引。为了避免此限制，你必须在定义约束之前创建（唯一）索引。以下 SQL 语句展示了一个示例：

```sql
CREATE UNIQUE index t_pk ON t (id) PARALLEL 2

ALTER TABLE t ADD CONSTRAINT t_pk PRIMARY KEY (id)
```



###### 何时使用

仅当满足两个条件时才应使用并行处理。第一，当有大量可用资源（CPU、内存和 I/O 带宽）时。请记住，并行处理的目的在于，将通常由单个进程（因此是单个 CPU）执行的工作分配给多个进程（因此是多个 CPU），以缩短响应时间。第二，当 SQL 语句串行执行需要超过十几秒时；否则，初始化和终止并行环境（主要是从属进程和表队列）所需的时间和资源，可能高于并行化本身所节省的时间。实际限制取决于可用资源的数量。因此，在某些情况下，只有那些执行时间超过几分钟甚至更长的 SQL 语句，才适合作为并行执行的候选。必须强调的是，如果这两个条件不满足，性能反而可能下降，而非提升。

如果并行处理被普遍用于许多 SQL 语句，则应在表和/或索引级别设置并行度。反之，如果它仅用于特定的批处理或报告，则通常最好在会话级别或通过提示来启用它。

###### 陷阱与误区

理解提示 `parallel` 和 `parallel_index` **并不强制**查询优化器使用并行处理，这一点非常重要。相反，它们会覆盖在表或索引级别定义的并行度。这种改变进而允许查询优化器考虑使用指定并行度的并行处理。这意味着，查询优化器会同时考虑使用和不使用并行处理的执行计划，并像往常一样，选择成本较低的那个。让我通过基于脚本 `px_dop2.sql` 的一个例子来强调这一点。如下面的 SQL 语句所示，与全表扫描相关的成本随并行度成比例地降低：

```sql
SQL> EXPLAIN PLAN SET STATEMENT_ID 'dop1' FOR
  2 SELECT /*+ full(t) parallel(t 1) */ * FROM t WHERE id > 90000;

SQL> EXPLAIN PLAN SET STATEMENT_ID 'dop2' FOR
  2 SELECT /*+ full(t) parallel(t 2) */ * FROM t WHERE id > 90000;

SQL> EXPLAIN PLAN SET STATEMENT_ID 'dop3' FOR
  2 SELECT /*+ full(t) parallel(t 3) */ * FROM t WHERE id > 90000;

SQL> EXPLAIN PLAN SET STATEMENT_ID 'dop4' FOR
  2 SELECT /*+ full(t) parallel(t 4) */ * FROM t WHERE id > 90000;

SQL> SELECT statement_id, cost
  2 FROM plan_table
  3 WHERE id = 0
  4 ORDER BY statement_id;

STATEMENT_ID  COST
------------ -----
dop1           430
dop2           238
dop3           159
dop4           119
```

如果 SQL 语句在没有提示的情况下执行，且并行度设置为 1，查询优化器会选择索引范围扫描。请注意，与此执行计划相关的成本（178）低于并行度最多为 2 时的全表扫描成本。相比之下，当并行度等于或高于 3 时，全表扫描的成本更低。

```sql
SQL> SELECT * FROM t WHERE id > 9000;

--------------------------------------------------------
| Id | Operation                   | Name | Cost (%CPU)|
--------------------------------------------------------
|   0| SELECT STATEMENT            |      |   178   (0)|
|   1|  TABLE ACCESS BY INDEX ROWID| T    |   178   (0)|
|   2|   INDEX RANGE SCAN          | I    |    24   (0)|
--------------------------------------------------------
```

现在，让我们看看当 SQL 语句中只添加了 `parallel` 提示时会发生什么，换句话说，当没有使用访问路径提示时。发生的情况是，当并行度设置为 2 时，查询优化器选择串行索引范围扫描；而当并行度设置为 3 时，则选择并行全表扫描。基本上，`parallel` 和 `parallel_index` 提示仅仅是允许查询优化器考虑并行处理；它们并不强制并行处理。

```sql
SQL> SELECT /*+ parallel(t 2) */ * FROM t WHERE id > 90000;

--------------------------------------------------------
| Id | Operation                   | Name | Cost (%CPU)|
--------------------------------------------------------
|   0| SELECT STATEMENT            |      |   178   (0)|
|   1|  TABLE ACCESS BY INDEX ROWID| T    |   178   (0)|
|   2|   INDEX RANGE SCAN          | I    |    24   (0)|
--------------------------------------------------------

SQL> SELECT /*+ parallel(t 3) */ * FROM t WHERE id > 90000;

-----------------------------------------------------
| Id | Operation             | Name    | Cost (%CPU)|
-----------------------------------------------------
|   0| SELECT STATEMENT      |         |   159   (0)|
|   1|  PX COORDINATOR       |         |            |
|   2|   PX SEND QC (RANDOM) | :TQ10000|   159   (0)|
|   3|    PX BLOCK ITERATOR  |         |   159   (0)|
|   4|     TABLE ACCESS FULL | T       |   159   (0)|
-----------------------------------------------------
```


