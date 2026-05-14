# Oracle 数据库执行计划的提取与查看

如果你使用的是 Oracle9i，`dbms_xplan`包可能不提供`display_cursor`函数。即使在这种情况下，我也不建议直接查询动态性能视图。我的建议是从库缓存中提取执行计划信息，并将其插入到一个计划表中。然后使用`dbms_xplan`包中的`display`函数来查询这个计划表。以下是一个示例：

```sql
SQL> SELECT address, hash_value, child_number, sql_text
  2  FROM v$sql
  3  WHERE sql_text LIKE '%online discount%' AND sql_text NOT LIKE '%v$sql%';

ADDRESS          HASH_VALUE CHILD_NUMBER SQL_TEXT
---------------- ---------- ------------ -------------------------------------
0000000055DCD888 4132422484            0 SELECT sum(amount_sold) FROM sales s,
                                         promotions p WHERE s.promo_id =
                                         p.promo_id AND promo_subcategory =
                                         'online discount'

SQL> DELETE plan_table;

SQL> INSERT INTO plan_table (operation, options, object_node, object_owner,
  2                          object_name, optimizer, search_columns, id,
  3                          parent_id, position, cost, cardinality, bytes,
  4                          other_tag, partition_start, partition_stop,
  5                          partition_id, other, distribution, cpu_cost,
  6                          io_cost, temp_space, access_predicates,
  7                          filter_predicates)
  8  SELECT operation, options, object_node, object_owner, object_name,
  9         optimizer, search_columns, id, parent_id, position, cost,
 10         cardinality, bytes, other_tag, partition_start, partition_stop,
 11         partition_id, other, distribution, cpu_cost, io_cost, temp_space,
 12         access_predicates, filter_predicates
 13  FROM v$sql_plan
 14  WHERE address = '0000000055DCD888'
 15  AND hash_value = 4132422484
 16  AND child_number = 0;

SQL> SELECT * FROM table(dbms_xplan.display);

PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------
--------------------------------------------------------------------------
| Id  | Operation             | Name        | Rows  | Bytes | Cost (%CPU)|
--------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |             |       |       |            |
|   1 |  SORT AGGREGATE       |             |     1 |    29 |            |
|*  2 |   HASH JOIN           |             | 64482 |  1826K|   426  (24)|
|*  3 |    TABLE ACCESS FULL  | PROMOTIONS  |    24 |   504 |     4  (25)|
|   4 |    PARTITION RANGE ALL|             |       |       |            |
|   5 |     TABLE ACCESS FULL | SALES       |  1016K|  7939K|   398  (19)|

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("PROMO_ID"="PROMO_ID")
   3 - filter("PROMO_SUBCATEGORY"='online discount')
```

为了利用这一技术而无需花时间编写长 SQL 语句，你可以使用脚本`display_cursor_9i.sql`。

## 自动工作负载库与 Statspack

当创建快照时，自动工作负载库（AWR）和 Statspack 能够收集执行计划。为了获取执行计划，会执行针对前一节所述动态性能视图的查询。一旦可用，执行计划可以在报告中显示，或者对于 AWR，可以通过 Enterprise Manager 显示。对于 AWR 和 Statspack，存储执行计划的仓库表与视图`v$sql_plan`基本相同。因此，前一节描述的技术同样适用于此。

### AWR AND STATSPACK

从 Oracle Database 10g 开始，用于存储性能相关信息的 AWR 仓库是自动安装的。在正常操作期间，数据库引擎不仅负责维护其内容，还负责利用它进行自我调整。其目的是保留过去几周数据库工作负载的历史记录。有关 AWR 的信息，请参阅《性能调优指南》手册的第 5 章。

AWR 的前身称为*Statspack*，既不会自动安装，也不会自动维护。它只是一个 DBA 可以在数据库中安装的外接组件。无论如何，其目的与 AWR 类似。有关 Statspack 的信息，请参阅 Oracle9i 手册《数据库性能调优指南与参考》的第 21 章。

存储在 AWR 中的执行计划可通过视图`dba_hist_sql_plan`获取。要查询它们，`dbms_xplan`包提供了`display_awr`函数。与该包的其他函数一样，它们的使用方法是直接的。以下查询是一个示例。注意，传递给`display_awr`函数的参数通过其`sql_id`来标识 SQL 语句。

```sql
SQL> SELECT * FROM table(dbms_xplan.display_awr('1hqjydsjbvmwq'));

PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------
SELECT SUM(AMOUNT_SOLD) FROM SALES S, PROMOTIONS P WHERE S.PROMO_ID =
P.PROMO_ID AND PROMO_SUBCATEGORY = 'online discount'

Plan hash value: 265338492

------------------------------------------------------------------------------------
| Id  | Operation             | Name       | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |            |       |       |   517 (100)|          |
|   1 |  SORT AGGREGATE       |            |     1 |    30 |            |          |
|   2 |   HASH JOIN           |            |   913K|    26M|     517 (4)| 00:00:07 |
|   3 |    TABLE ACCESS FULL  | PROMOTIONS |    23 |   483 |      17 (0)| 00:00:01 |
|   4 |    PARTITION RANGE ALL|            |   918K|  8075K|   494   (3)| 00:00:06 |
|   5 |     TABLE ACCESS FULL | SALES      |   918K|  8075K|   494   (3)| 00:00:06 |
```

`display_awr`函数不仅可以无参数使用。因此，在本章后面，我将详细介绍`dbms_xplan`包，探讨所有的可能性，包括对生成输出的描述。

当使用等于或大于 6 的级别创建快照时，Statspack 会将执行计划存储在`stats$sql_plan`仓库表中。不幸的是，`dbms_xplan`包没有提供查询它的功能。我建议将执行计划复制到一个计划表中，并使用`dbms_xplan`包中的`display`函数来显示它（参见前一节中关于 Oracle9i 中`v$sql_plan`视图的技术描述）。


## AWR 和 Statspack 报告

此外，对于 AWR 和 Statspack，Oracle 提供了有用的报告，用于突出显示特定 SQL 语句在一段时间内的执行计划变更和资源消耗变化。它们的脚本名称分别是`awrsqrpt.sql`和`sprepsql.sql`。你可以在`$ORACLE_HOME/rdbms/admin`目录下找到它们。请注意，AWR 的脚本仅在 Oracle Database 10g Release 2 及更高版本中可用。以下是脚本`awrsqrpt.sql`生成的输出摘录。根据输出，该 SQL 语句的执行计划在分析期间发生了变化。平均耗时从第一次的大约 1.5 毫秒（15,098/10）下降到第二次的大约 0.3 毫秒（2,780/10）。

```
SQL ID: 1hqjydsjbvmwq               DB/Inst: DBM11106/DBM11106   Snaps: 143-145
-> 1st Capture and Last Capture Snap IDs
   refer to Snapshot IDs witin the snapshot range
-> SELECT SUM(AMOUNT_SOLD) FROM SALES S, PROMOTIONS P WHERE S.PROMO_ID = ...

    Plan Hash           Total Elapsed                 1st Capture   Last Capture
#   Value                    Time(ms)    Executions       Snap ID        Snap ID
--- ---------------- ---------------- ------------- ------------- --------------
1   1279966040               15,098           10            144             145
2   265338492                 2,780           10            144             145
          -------------------------------------------------------------
Plan 1(PHV: 01279966040)
-----------------------

Execution Plan
------------------------------------------------------------------------------------
| Id  | Operation              | Name       | Rows | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |            |      |       |  4028 (100)|          |
|   1 |  SORT AGGREGATE        |            |    1 |    30 |            |          |
|   2 |   MERGE JOIN           |            |  913K|    26M|  4028   (2)| 00:00:49 |
|   3 |    SORT JOIN           |            |   23 |   483 |    18   (6)| 00:00:01 |
|   4 |     TABLE ACCESS FULL  | PROMOTIONS |   23 |   483 |    17   (0)| 00:00:01 |
|   5 |    SORT JOIN           |            |  918K|  8075K|  4010   (2)| 00:00:49 |
|   6 |     PARTITION RANGE ALL|            |  918K|  8075K|   494   (3)| 00:00:06 |
|   7 |      TABLE ACCESS FULL | SALES      |  918K|  8075K|   494   (3)| 00:00:06 |
------------------------------------------------------------------------------------

Plan 2(PHV: 265338492)
----------------------

Execution Plan
------------------------------------------------------------------------------------
| Id  | Operation              | Name       | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |            |       |       |   517 (100)|          |
|   1 |  SORT AGGREGATE        |            |     1 |    30 |            |          |
|   2 |   HASH JOIN            |            |   913K|    26M|   517   (4)| 00:00:07 |
|   3 |    TABLE ACCESS FULL   | PROMOTIONS |    23 |   483 |    17   (0)| 00:00:01 |
|   4 |    PARTITION RANGE ALL |            |   918K|  8075K|   494   (3)| 00:00:06 |
|   5 |     TABLE ACCESS FULL  | SALES      |   918K|  8075K|   494   (3)| 00:00:06 |
------------------------------------------------------------------------------------
```

### 追踪工具

有几种追踪工具可以提供关于执行计划的信息。不幸的是，除了 SQL 追踪（参见第 3 章），其他所有工具都未得到官方支持。无论如何，它们可能被证明是有用的，因此我将简要介绍其中两个。

#### 事件 10053

如果你因为查询优化器做出的决策而陷入严重困境，并且你想了解正在发生什么，事件 10053 可能会帮助你。不过，让我提醒你，阅读它生成的追踪文件并非易事。幸运的是，可能不需要经常这样做，并且只有在你真正对查询优化器的内部工作机制感兴趣时才需要这样做。

通常，你希望一次分析一个 SQL 语句。因此，为了生成追踪文件，通常将其嵌入以下两条 SQL 语句之间，从而启用和禁用事件 10053。只是要小心，追踪文件仅在执行硬解析时生成。

```sql
ALTER SESSION SET events '10053 trace name context forever'
ALTER SESSION SET events '10053 trace name context off'
```

当事件 10053 启用时，查询优化器会生成一个追踪文件，其中包含大量关于它所执行工作的信息。在其中，你会找到由初始化参数、系统统计信息和对象统计信息确定的执行环境，以及为找出最有效的执行计划而进行的估算。描述此事件生成的追踪文件的内容超出了本书的范围。如有必要，请参考以下资料：

*   Wolfgang Breitling 的论文 "A Look under the Hood of CBO: The 10053 Event"
*   Metalink 注释 "CASE STUDY: Analyzing 10053 Trace Files (338137.1)"
*   Jonathan Lewis 的著作 *Cost-Based Oracle Fundamentals* 的第 14 章

每个服务器进程将其解析的所有 SQL 语句的数据写入其自己的追踪文件中。这不仅意味着一个追踪文件可以包含多个 SQL 语句的信息，而且当在多个会话中启用该事件时，将使用多个追踪文件。有关追踪文件名称和位置的信息，请参阅第 3 章中的"查找追踪文件"部分。

#### 事件 10132

你可以使用事件 10132 来生成一个追踪文件，其中包含每次硬解析相关的执行计划。如果你想为特定模块或应用程序保留所有执行计划的历史记录，这可能很有用。以下示例显示了为每个 SQL 语句存储在追踪文件中的信息类型，主要是 SQL 语句及其执行计划（包括关于谓词的信息）。请注意，在此输出中，我在两个不同的地方剪掉了提供执行环境信息的长参数列表和错误修复列表。

```
sql_id=gbxvdrz7jvt80.
Current SQL statement for this session:
SELECT count(n) FROM t WHERE n BETWEEN 6 AND 19
```



