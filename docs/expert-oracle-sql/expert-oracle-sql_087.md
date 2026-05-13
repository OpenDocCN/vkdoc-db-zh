# 执行计划操作

| 0 | SELECT STATEMENT | |
| 1 | SORT AGGREGATE | |
| 2 | BITMAP CONVERSION COUNT | |
| 3 | BITMAP OR | |
| 4 | BITMAP MERGE | |
| 5 | BITMAP INDEX RANGE SCAN | T1_BIX2 |
| 6 | BITMAP AND | |
| 7 | BITMAP MERGE | |
| 8 | BITMAP INDEX RANGE SCAN | T1_BIX1 |
| 9 | BITMAP CONVERSION FROM ROWIDS | |
| 10 | INDEX RANGE SCAN | T1_I3 |
| 11 | BITMAP CONVERSION FROM ROWIDS | |
| 12 | SORT ORDER BY | |
| 13 | INDEX RANGE SCAN | T1_I1 |

此计划与前一个计划的主要区别在于第 2 行存在`BITMAP CONVERSION COUNT`操作，并且没有任何`BITMAP CONVERSION TO ROWIDS`操作。

位图操作的一个主要限制是我们将在讨论星型转换时详细研究（详见第 13 章），即这种机制无法处理包含子查询的“IN 列表”，正如 Listing 10-20 中尝试但失败的那样。

Listing 10-20. 针对 IN 列表的 INDEX COMBINE 尝试

```sql
SELECT /*+ index_combine(e) */
       /* THIS HINT INNEFFECTIVE IN THIS QUERY */
      employee_id
      ,first_name
      ,last_name
      ,email
  FROM hr.employees e
 WHERE    manager_id IN (SELECT employee_id
                           FROM hr.employees
                          WHERE salary > 14000)
       OR job_id = 'SH_CLERK';

| Id  | Operation                    | Name          |

|   0 | SELECT STATEMENT             |               |
|   1 |  FILTER                      |               |
|   2 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEES     |
|   3 |    INDEX FULL SCAN           | EMP_EMP_ID_PK |
|   4 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEES     |
|   5 |    INDEX UNIQUE SCAN         | EMP_EMP_ID_PK |
```

#### 全表扫描

现在我们来到执行计划中备受诟病的操作：全表扫描（full table scan）（参见表 10-5）。

表 10-5. 全表扫描操作

| Operation | Hint |
| --- | --- |
| `TABLE ACCESS FULL` | `FULL` |
| `TABLE ACCESS SAMPLE` | `FULL` |
| `TABLE ACCESS SAMPLE BY ROWID RANGE` | `FULL PARALLEL` |

虽然我们都称这个操作为全表扫描，但在执行计划显示中该操作的实际名称是`TABLE ACCESS FULL`。这两个术语都有些误导性，因为在分区表的情况下，被扫描的“表”实际上是一个单独的分区或子分区。也许为了避免过于具体的术语，它可能被称为“段全扫描”，但这又不够具体；毕竟我们这里讨论的不是索引。为了简单起见，在本节剩余部分我将假设我们查看的是非分区表；其机制不会改变。为了简洁，我也会使用缩写 FTS。

FTS 操作听起来足够简单：我们从头到尾读取表中的所有数据块。

然而，实际上它比这要复杂得多。我们需要：

*   访问段头以读取区映射并识别高水位标记（HWM）；
*   读取直到 HWM 的所有区中的所有数据；以及
*   重新读取段头以确认没有更多行。

即使我们只讨论串行（而非并行）操作，这仍然不是全部情况。一旦所有数据都返回给客户端，可能需要额外的 FETCH 操作来再次确认没有更多数据。这将导致第三次读取段头。

当我们处理大型甚至中等大小的表时，这些额外的段头读取通常不是主要因素，但对于一个微小的表，例如只有一个数据块，这些额外读取可能是个问题。如果这个小表被频繁访问，你可能需要创建索引，或者将表转换为索引组织表。

有些人会告诉你，调整性能不佳的 SQL 的关键是找出 FTS 并通过添加索引来消除它们。这是一种危险的不完整说法。确实，缺失的索引*可能*导致性能问题，并且这些性能问题可能通过执行计划中存在`TABLE FULL SCAN`操作被识别出来。然而，正如我在第 9 章介绍索引聚簇因子概念时所解释的那样，FTS 通常是访问表最有效的方式。如果你忘记了，可以回顾一下图 9-1。

Listing 10-21 展示了基本的 FTS 操作。

Listing 10-21. 全表扫描

```sql
SELECT /*+ full(t1) */
         * FROM t1;

| Id  | Operation         | Name |

|   0 | SELECT STATEMENT  |      |
|   1 |  TABLE ACCESS FULL| T1   |
```

Listing 10-22 展示了本节开头列出的第二个操作。与`INDEX FAST FULL SCAN`一样，使用样本会改变操作。

Listing 10-22. TABLE ACCESS SAMPLE

```sql
SELECT /*+ full(t1) */
            *
        FROM t1 SAMPLE (5);

| Id  | Operation           | Name |

|   0 | SELECT STATEMENT    |      |
|   1 |  TABLE ACCESS SAMPLE| T1   |
```

Listing 10-22 请求从表中抽取 5%的样本。最后的 FTS 变体在 Listing 10-23 中演示，展示了一个并行采样的查询。

Listing 10-23. 并行 TABLE ACCESS SAMPLE

```sql
SELECT /*+ full(t1) parallel */
            *
        FROM t1 SAMPLE (5);

| Id  | Operation                             | Name     |

|   0 | SELECT STATEMENT                      |          |
|   1 |  PX COORDINATOR                       |          |
|   2 |   PX SEND QC (RANDOM)                 | :TQ10000 |
|   3 |    PX BLOCK ITERATOR                  |          |
|   4 |     TABLE ACCESS SAMPLE BY ROWID RANGE| T1       |
```

