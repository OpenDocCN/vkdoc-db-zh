# 测试快速刷新

在重新定义物化视图日志和物化视图之后，使用过程`explain_mview`进行进一步分析表明，在所有情况下都可以进行快速刷新（`possible`列始终设置为`Y`）。那么，让我们通过向两个表插入数据，然后执行快速刷新来测试其速度：

```sql
SQL> CREATE MATERIALIZED VIEW sales_mv
  2  REFRESH FORCE ON DEMAND
  3  AS
  4  SELECT p.prod_category, c.country_id,
  5         sum(s.quantity_sold) AS quantity_sold,
  6         sum(s.amount_sold) AS amount_sold,
         count(*) AS count_star,
         count(s.quantity_sold) AS count_quantity_sold,
         count(s.amount_sold) AS count_amount_sold
 10  FROM sales s, customers c, products p
 11  WHERE s.cust_id = c.cust_id
 12  AND s.prod_id = p.prod_id
 13  GROUP BY p.prod_category, c.country_id;
```

```sql
SQL> INSERT INTO products
  2  SELECT 619, prod_name, prod_desc, prod_subcategory, prod_subcategory_id,
  3         prod_subcategory_desc, prod_category, prod_category_id,
  4         prod_category_desc, prod_weight_class, prod_unit_of_measure,
  5         prod_pack_size, supplier_id, prod_status, prod_list_price,
  6         prod_min_price, prod_total, prod_total_id, prod_src_id,
  7         prod_eff_from, prod_eff_to, prod_valid
  8  FROM products
  9  WHERE prod_id = 136;
```

```
已创建 1 行。
```

```sql
SQL> INSERT INTO sales
  2  SELECT 619, cust_id, time_id, channel_id, promo_id, quantity_sold, amount_sold
  3  FROM sales
  4  WHERE prod_id = 136;
```

```
已创建 710 行。
```

```sql
SQL> COMMIT;
```

```sql
SQL> execute dbms_mview.refresh(list => 'sh.sales_mv', method => 'f')
```

```
PL/SQL 过程已成功完成。

已用时间: 00:00:00.32
```

在这种情况下，快速刷新持续了 0.32 秒。这比完全刷新少了一个数量级以上，效果很好。如果你对快速刷新的性能不满意，应该使用 SQL 跟踪来调查为什么耗时过长。然后，通过应用第 9 章中描述的技术，你可能能够通过添加索引或对段进行分区来加速。

## 使用分区变更跟踪的快速刷新

存储历史数据的表通常按天、周或月进行范围分区。换句话说，分区是基于存储时间信息的列。因此，经常会新增分区，向其中加载数据，并删除较旧的分区（通常只保留特定数量的分区在线）。执行这些操作后，所有依赖的物化视图都会过期，因此应该刷新。

问题在于，使用物化视图日志的快速刷新（前一节描述的那些）无法在诸如`CREATE PARTITION`或`DROP PARTITION`之类的分区管理操作之后执行。如果启动此类刷新，会引发错误`ORA-32313: REFRESH FAST of "SH"."SALES_MV" unsupported after PMOPs`。当然，执行完全刷新总是可行的。但是，如果分区很多，而只有一两个被修改，刷新时间可能无法接受。

为了解决这个问题，可以使用带有分区变更跟踪（PCT）的快速刷新。其思想是数据库引擎能够跟踪分区级别的过期情况，而不仅仅是表级别。换句话说，它可以跳过所有未被更改的分区的刷新。要使用这种刷新方法，物化视图必须满足一些要求。基本上，数据库引擎必须能够将物化视图中存储的行映射到基表分区。如果物化视图包含以下内容之一，则可以实现：

*   分区键
*   Rowid
*   分区标记符
*   依赖于连接的表达式（仅限 Oracle Database 10*g* 及以上）

前两个应该很明了；让我们看第三个和第四个的例子。

分区标记符无非是由包`dbms_mview`中的函数`pmarker`生成的分区标识符。为了生成分区标记符，该函数使用作为参数传递的 rowid。以下示例基于脚本`mv_refresh_pct.sql`，展示了如何创建包含分区标记符的物化视图：

```sql
CREATE MATERIALIZED VIEW sales_mv
REFRESH FORCE ON DEMAND
AS
SELECT p.prod_category, c.country_id,
       sum(s.quantity_sold) AS quantity_sold,
       sum(s.amount_sold) AS amount_sold,
       count(*) AS count_star,
       count(s.quantity_sold) AS count_quantity_sold,
       count(s.amount_sold) AS count_amount_sold,
       dbms_mview.pmarker(s.rowid) AS pmarker
FROM sales s, customers c, products p
WHERE s.cust_id = c.cust_id
AND s.prod_id = p.prod_id
GROUP BY p.prod_category, c.country_id, dbms_mview.pmarker(s.rowid)
```

***

**注意** 由于函数`pmarker`是为每一行调用的，所以不要低估调用它所需的时间。在我的系统上，使用分区标记符创建物化视图所需的时间是不使用它的 2.5 倍。

***

当分区键用于连接另一个表时，物化视图包含一个*依赖于连接的表达式*。在本节使用的例子中，这意味着必须将与表`times`的连接添加到物化视图中。以下 SQL 语句提供了一个示例：

```sql
CREATE MATERIALIZED VIEW sales_mv
REFRESH FORCE ON DEMAND
AS
SELECT p.prod_category, c.country_id, t.fiscal_year,
       sum(s.quantity_sold) AS quantity_sold,
       sum(s.amount_sold) AS amount_sold,
       count(*) AS count_star,
       count(s.quantity_sold) AS count_quantity_sold,
       count(s.amount_sold) AS count_amount_sold
FROM sales s, customers c, products p, times t
WHERE s.cust_id = c.cust_id
AND s.prod_id = p.prod_id
AND s.time_id = t.time_id
GROUP BY p.prod_category, c.country_id, t.fiscal_year
```

有了分区标记符或依赖于连接的表达式后，使用包`dbms_mview`中的函数`explain_mview`进行的分析表明，基于分区变更跟踪的快速刷新是可能的。然而，它仅对在表`sales`上执行的修改有效：

```sql
SQL> SELECT capability_name, possible, msgtxt, related_text
  2  FROM mv_capabilities_table
  3  WHERE statement_id = '42'
  4  AND capability_name IN ('PCT_TABLE','REFRESH_FAST_PCT')
  5  ORDER BY seq;
```

```
CAPABILITY_NAME  POSSIBLE MSGTXT                        RELATED_TEXT
---------------- -------- ----------------------------- -------------
PCT_TABLE        Y                                      SALES
PCT_TABLE        N        关系不是分区表                 CUSTOMERS
PCT_TABLE        N        关系不是分区表                 PRODUCTS
REFRESH_FAST_PCT Y
```



###### 何时使用

物化视图是一种冗余访问结构。与所有冗余访问结构一样，它们对于高效访问数据很有用，但维护其最新状态会产生开销。如果将物化视图与索引进行比较，物化视图带来的性能提升和产生的开销都可能远高于索引。显然，这两个概念旨在解决不同的问题。简而言之，只有当改善数据访问带来的好处超过了管理数据冗余副本（当然也包括索引）的弊端时，才应该使用物化视图。

通常，我看到物化视图有两种用途：

*   用于提升大型聚合和/或连接操作的性能，这类操作的逻辑读次数与返回行数之间的比率非常高。
*   用于提升那些既无法通过全表扫描高效执行，也无法通过索引范围扫描高效执行的单表访问的性能。基本上，这些访问具有平均选择性，原本需要通过分区来优化，但如果无法利用分区（第 9 章讨论了何时无法利用），物化视图可能会有所帮助。

物化视图常用于数据仓库中以构建预存储的聚合。这有两个原因。首先，数据大多是只读的；因此，刷新物化视图的开销可以被最小化，并在数据库专用于修改表的特定时间窗口内完成。其次，在此类环境中，性能提升可能非常巨大。事实上，经常可以看到基于大型聚合或连接的查询，如果没有物化视图，处理它们所需的资源量是不可接受的。

尽管数据仓库是物化视图的主要应用环境，但我也在 OLTP 系统中成功实施过它们。这对于那些频繁被查询、但相比之下修改操作相对较少的表可能有益。在此类环境中，为了刷新物化视图，通常使用提交时快速刷新，以保证亚秒级的刷新时间和始终保持物化视图的新鲜度。

###### 陷阱与误区

你不应将新的连接语法（即 ANSI 连接语法）与物化视图一起使用。如果使用了，某些查询重写和快速刷新功能将不受支持。因此，我建议你在可能的情况下，直接避免使用该语法。请注意，官方文档并未提及这些限制。

由于快速刷新并不总是比完全刷新更快，因此不应在所有情况下都使用它。一个特定情况是当基础表中大量数据被修改时。此外，你不应低估在修改基础表时维护物化视图日志的开销。因此，你需要仔细评估使用快速刷新的利弊。

在创建物化视图日志时，你必须非常小心逗号的使用。你能找出以下 SQL 语句的问题所在吗？

```
SQL> CREATE MATERIALIZED VIEW LOG ON products WITH ROWID, SEQUENCE,
  2  (prod_id, prod_category) INCLUDING NEW VALUES;
CREATE MATERIALIZED VIEW LOG ON products WITH ROWID, SEQUENCE,
*
ERROR at line 1:
ORA-12026: invalid filter column detected
```

问题在于关键字 `SEQUENCE` 和过滤列表（即括号中的列列表）之间的逗号。如果存在这个逗号，则隐含了 `PRIMARY KEY` 选项，而在这种情况下不能指定该选项，因为主键（列 `prod_id`）已经在过滤列表中了。以下是正确的 SQL 语句。请注意，只删除了那个逗号。

```
SQL> CREATE MATERIALIZED VIEW LOG ON products WITH ROWID, SEQUENCE
  2  (prod_id, prod_category) INCLUDING NEW VALUES;

Materialized view log created.
```

##### 结果缓存

缓存是计算机系统中用于提高性能的最常用技术之一。硬件和软件都广泛使用它。Oracle 数据库引擎也不例外。例如，它在缓冲区缓存中缓存数据文件块，在字典缓存中缓存数据字典信息，在库缓存中缓存游标。从 Oracle Database 11*g 开始，还提供了*结果缓存*。

* * *

**注意** 结果缓存仅在企业版中可用。

* * *



###### 工作原理

Oracle 数据库引擎提供了三种结果缓存：

*   *服务器结果缓存*（亦称*查询结果缓存*）是一种服务器端缓存，用于存储查询结果集。
*   *PL/SQL 函数结果缓存*是一种服务器端缓存，用于存储 PL/SQL 函数的返回值。
*   *客户端结果缓存*是一种客户端缓存，用于存储查询结果集。

接下来的章节将描述这些缓存的工作原理，以及你需要做些什么来利用它们。请注意，默认情况下不使用结果缓存。

## 服务器结果缓存

服务器结果缓存用于避免重新执行查询。简而言之，首次执行查询时，其结果集会存储在共享池中。随后，对于同一查询的后续执行，结果集将直接从结果缓存中提供，而无需重新计算。请注意，只有当两个查询的文本完全相同时（允许空格和大小写差异），它们才被视为相同，因此可以使用同一缓存结果。此外，如果存在绑定变量，则所有绑定变量的值必须完全相同。这显然是必要的，因为绑定变量是传递给查询的输入参数，因此，对于不同的绑定变量值，结果集通常也不同。还需注意，结果缓存存储在共享池中，连接到给定实例的所有会话共享相同的缓存条目。

举个例子，让我们执行在物化视图部分已经使用过两次的查询。注意，查询中指定了提示 `result_cache` 以启用结果缓存。第一次执行耗时 1.78 秒。请注意，执行计划中的 `RESULT CACHE` 操作证实了该查询已启用结果缓存。然而，执行计划中的 `Starts` 列清楚地显示所有操作都至少执行了一次。所有操作都必须执行，因为这是查询的首次执行，因此结果缓存中尚未包含结果集。

```sql
SQL> SELECT /*+ result_cache */ 
  2         p.prod_category, c.country_id,
  3         sum(s.quantity_sold) AS quantity_sold,
  4         sum(s.amount_sold) AS amount_sold
  5  FROM sales s, customers c, products p
  6  WHERE s.cust_id = c.cust_id
  7  AND s.prod_id = p.prod_id
  8  GROUP BY p.prod_category, c.country_id
  9  ORDER BY p.prod_category, c.country_id;
```

`已用时间: 00:00:01.78`

```sql
--------------------------------------------------------------------------------
| Id | Operation                  | Name                       | Starts |A-Rows |
--------------------------------------------------------------------------------
|   1|   RESULT CACHE             | fxtfk2w8kgrhm6c5j1y73twrgp |      1|     81 |
|   2|    SORT GROUP BY           |                            |      1|     81 |
|*  3|     HASH JOIN              |                            |      1|    918K|
|   4|      TABLE ACCESS FULL     | PRODUCTS                   |      1|     72 |
|*  5|      HASH JOIN             |                            |      1|    918K|
|   6|       TABLE ACCESS FULL    | CUSTOMERS                  |      1|  55500 |
|   7|       PARTITION RANGE ALL  |                            |      1|    918K|
|   8|        TABLE ACCESS FULL   | SALES                      |     28|    918K|
--------------------------------------------------------------------------------
```

第二次执行耗时 0.07 秒。这次，执行计划中的 `Starts` 列显示，除 `RESULT CACHE` 外的所有操作都未执行。换句话说，查询的结果集是直接从结果缓存中提供的。

```sql
SQL> SELECT /*+ result_cache */ 
  2         p.prod_category, c.country_id,
  3         sum(s.quantity_sold) AS quantity_sold,
  4         sum(s.amount_sold) AS amount_sold
  5  FROM sales s, customers c, products p
  6  WHERE s.cust_id = c.cust_id
  7  AND s.prod_id = p.prod_id
  8  GROUP BY p.prod_category, c.country_id
  9  ORDER BY p.prod_category, c.country_id;
```

`已用时间: 00:00:00.07`

```sql
--------------------------------------------------------------------------------
| Id | Operation                  | Name                       | Starts |A-Rows |
--------------------------------------------------------------------------------
|   1|   RESULT CACHE             | fxtfk2w8kgrhm6c5j1y73twrgp |      1|     81 |
|   2|    SORT GROUP BY           |                            |      0|      0 |
|*  3|     HASH JOIN              |                            |      0|      0 |
|   4|      TABLE ACCESS FULL     | PRODUCTS                   |      0|      0 |
|*  5|      HASH JOIN             |                            |      0|      0 |
|   6|       TABLE ACCESS FULL    | CUSTOMERS                  |      0|      0 |
|   7|       PARTITION RANGE ALL  |                            |      0|      0 |
|   8|        TABLE ACCESS FULL   | SALES                      |      0|      0 |
--------------------------------------------------------------------------------
```

在执行计划中，有趣的是注意到与 `RESULT CACHE` 操作关联了一个名称，即 *缓存 ID*。如果你知道缓存 ID，就可以查询视图 `v$result_cache_objects` 来显示有关缓存数据的信息。以下查询显示了缓存的结果集是否已发布（换句话说，可供使用）、结果缓存创建的时间、构建它花费了多少时间（以百分之一秒为单位）、其中存储了多少行数据以及它被引用了多少次。其他提供结果缓存信息的视图有 `v$result_cache_dependency`、`v$result_cache_memory` 和 `v$result_cache_statistics`。

```sql
SQL> SELECT status, creation_timestamp, build_time, row_count, scan_count
  2  FROM v$result_cache_objects
  3  WHERE cache_id = 'fxtfk2w8kgrhm6c5j1y73twrgp';
```

```sql
STATUS    CREATION_TIMESTAMP   BUILD_TIME ROW_COUNT SCAN_COUNT
--------- -------------------- ---------- --------- ----------
Published 08.04.2008 23:51:38         168        81          1
```

为了保证结果的一致性（即无论结果是从缓存提供还是根据数据库内容计算，结果集都相同），每当查询引用的对象发生任何变化时，依赖于它的缓存条目都会失效（远程对象的例外情况稍后讨论）。即使没有实际变化发生，情况也是如此。例如，即使是一个 `SELECT FOR UPDATE`，紧接着执行一个 `COMMIT`，也会导致依赖于被选择表的缓存条目失效。

以下是控制服务器结果缓存的动态初始化参数：

*   `result_cache_max_size` 指定（以字节为单位）可用于结果缓存的共享池内存量。如果设置为 0，则禁用该功能。默认值大于 0，源自共享池的大小。内存分配是动态的，因此此初始化参数仅指定上限。你可以使用如下查询显示当前已分配的内存：

```sql
SQL> SELECT name, sum(bytes)
  2  FROM v$sgastat
  3  WHERE name LIKE 'Result Cache%'
  4  GROUP BY rollup(name);
```



