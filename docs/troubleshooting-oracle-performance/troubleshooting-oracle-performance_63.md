# 刷新组



## `dbms_refresh` 包

`dbms_refresh` 包用于管理**刷新组**。一个刷新组就是一个或多个物化视图的集合。使用 `dbms_refresh` 包中的 `refresh` 过程执行的刷新是在单个事务中完成的（`atomic_refresh` 被设置为 `TRUE`）。如果需要在多个物化视图之间保证一致性，这种行为是必要的。这也意味着，要么组内包含的所有物化视图都成功刷新，要么整个刷新操作被回滚。

### 使用物化视图日志的快速刷新

在快速刷新期间，会重用容器表的内容，并且仅将修改从基表传播到容器表。显然，只有在数据库引擎知道这些修改的情况下才能进行传播。为此，您必须在每个基表上创建一个**物化视图日志**以启用快速刷新（下一节将讨论的分区变更跟踪快速刷新是一个例外）。例如，物化视图 `sales_mv` 需要在表 `sales`、`customers` 和 `products` 上创建物化视图日志，以便能够进行快速刷新。简单来说，物化视图日志是一个由数据库引擎自动维护的表，用于跟踪基表上发生的修改。

除了物化视图日志，直接路径插入还会使用一个内部日志表。您无需创建它，因为它在数据库创建时会自动安装。要显示其内容，您可以查询视图 `all_sumdelta`。

在最简单的情况下，您可以使用如下 SQL 语句创建物化视图日志（此示例基于脚本 `mv_refresh_log.sql`）。请注意，`WITH ROWID` 子句用于指定如何在物化视图日志中识别行，即如何识别物化视图日志中每一行所跟踪修改的基表行。也可以创建使用主键或对象 ID 识别行的物化视图日志。但是，出于本章的目的，行必须通过其 rowid 来识别。

```sql
SQL> CREATE MATERIALIZED VIEW LOG ON sales WITH ROWID;
SQL> CREATE MATERIALIZED VIEW LOG ON customers WITH ROWID;
SQL> CREATE MATERIALIZED VIEW LOG ON products WITH ROWID;
```

正如有一个关联的容器表对应物化视图，每个物化视图日志也有一个关联的表，用于记录对基表的修改。以下查询展示了如何显示其名称：

```sql
SQL> SELECT master, log_table
  2  FROM user_mview_logs
  3  WHERE master IN ('SALES', 'CUSTOMERS', 'PRODUCTS');

MASTER      LOG_TABLE
----------  ----------------
CUSTOMERS   MLOG$_CUSTOMERS
PRODUCTS    MLOG$_PRODUCTS
SALES       MLOG$_SALES
```

对于大多数物化视图来说，这样一个基础的物化视图日志不足以支持快速刷新。还有额外的条件需要满足。由于这些条件与物化视图关联的查询以及 Oracle 数据库引擎的版本密切相关，因此我不会提供一个列表（可在《数据仓库指南》手册中找到），而是向您展示如何针对特定情况找出这些要求。为此，您可以使用与找出支持的查询重写模式相同的方法（参见前面的“查询重写”部分）。换句话说，您可以使用 `dbms_mview` 包中的 `explain_mview` 过程，如下例所示：

```sql
SQL> execute dbms_mview.explain_mview(mv => 'sales_mv', stmt_id => '42')
```

该过程的输出在表 `mv_capabilities_table` 中提供。要查看物化视图是否可以进行快速刷新，您可以使用如下查询。请注意，在输出中，`possible` 列始终设置为 `N`。这意味着无法进行快速刷新。此外，`msgtxt` 和 `related_text` 列指出了问题的原因。

```sql
SQL> SELECT capability_name, possible, msgtxt, related_text
  2  FROM mv_capabilities_table
  3  WHERE statement_id = '42'
  4  AND capability_name LIKE 'REFRESH_FAST_AFTER%'
  5  ORDER BY seq;

CAPABILITY_NAME               POSSIBLE MSGTXT                           RELATED_TEXT
--------------------------    -------- -------------------------------  -------------
REFRESH_FAST_AFTER_INSERT     N        mv log must have new values      SH.PRODUCTS
REFRESH_FAST_AFTER_INSERT     N        mv log does not have all         SH.PRODUCTS
                                         necessary columns
REFRESH_FAST_AFTER_INSERT     N        mv log must have new values      SH.CUSTOMERS
REFRESH_FAST_AFTER_INSERT     N        mv log does not have all         SH.CUSTOMERS
                                         necessary columns
REFRESH_FAST_AFTER_INSERT     N        mv log must have new values      SH.SALES
REFRESH_FAST_AFTER_INSERT     N        mv log does not have all         SH.SALES
                                         necessary columns
REFRESH_FAST_AFTER_ONETAB_DML N        SUM(expr) without COUNT(expr)    AMOUNT_SOLD
REFRESH_FAST_AFTER_ONETAB_DML N        SUM(expr) without COUNT(expr)    QUANTITY_SOLD
REFRESH_FAST_AFTER_ONETAB_DML N        see the reason why REFRESH_FAST
                                         _AFTER_INSERT is disabled
REFRESH_FAST_AFTER_ONETAB_DML N        COUNT(*) is not present in the
                                         select list
REFRESH_FAST_AFTER_ONETAB_DML N        SUM(expr) without COUNT(expr)
REFRESH_FAST_AFTER_ANY_DML    N        mv log does not have sequence  # SH.PRODUCTS
REFRESH_FAST_AFTER_ANY_DML    N        mv log does not have sequence  # SH.CUSTOMERS
REFRESH_FAST_AFTER_ANY_DML    N        mv log does not have sequence  # SH.SALES
REFRESH_FAST_AFTER_ANY_DML    N        see the reason why REFRESH_FAST
                                         _AFTER_ONETAB_DML is disabled
```

有些问题与物化视图日志有关，另一些则与物化视图有关。简而言之，数据库引擎需要更多信息来执行快速刷新。

要解决与物化视图日志相关的问题，您必须在 `CREATE MATERIALIZED VIEW LOG` 语句中添加一些选项。对于问题 `mv log does not have all necessary columns`，您必须指定物化视图中引用的每个列都存储在物化视图日志中。对于问题 `mv log must have new values`，您需要添加 `INCLUDING NEW VALUES` 子句。使用此选项，物化视图日志将存储旧值和新值（默认情况下仅存储旧值）。对于问题 `mv log does not have sequence`，必须添加 `SEQUENCE` 子句。使用此选项，一个序列号将与存储在物化视图日志中的每一行相关联。以下是重新定义后的物化视图日志：

```sql
SQL> CREATE MATERIALIZED VIEW LOG ON sales WITH ROWID, SEQUENCE
   2 (cust_id, prod_id, quantity_sold, amount_sold) INCLUDING NEW VALUES;

SQL> CREATE MATERIALIZED VIEW LOG ON customers WITH ROWID, SEQUENCE
   2 (cust_id, country_id) INCLUDING NEW VALUES;

SQL> CREATE MATERIALIZED VIEW LOG ON products WITH ROWID, SEQUENCE
   2 (prod_id, prod_category) INCLUDING NEW VALUES;
```

为了解决物化视图的问题，在创建物化视图时，必须基于 `count` 函数添加一些新列。以下 SQL 语句展示了包含新列的定义：



