# 查询重写完整性参数与示例

为了在类似情况下利用通用查询重写，可以使用动态初始化参数`query_rewrite_integrity`。通过该参数，您可以指定是仅使用由数据库引擎验证过的强制约束，以及是否使用包含过时数据的物化视图。该参数可设置为以下三个值：

*   `enforced`：仅考虑包含新鲜数据的物化视图用于查询重写。此外，仅使用经过验证的约束进行通用查询重写。此为默认值。
*   `trusted`：仅考虑包含新鲜数据的物化视图用于查询重写。此外，启用`novalidate`并标记为`rely`的维度和约束会被信任用于通用查询重写。
*   `stale_tolerated`：所有现有的物化视图，包括那些包含过时数据的，都会被考虑用于查询重写。此外，启用`novalidate`并标记为`rely`的维度和约束会被信任用于通用查询重写。

## 通用查询重写示例

以下示例展示了如何在不验证约束的情况下使用通用查询重写。如示例所示，约束被标记为`rely`，完整性级别被设置为`trusted`。

```sql
ALTER TABLE sales MODIFY CONSTRAINT sales_customer_fk RELY;
ALTER SESSION SET query_rewrite_integrity = trusted;
SELECT upper(p.prod_category) AS prod_category,
       sum(s.amount_sold) AS amount_sold
FROM sales s, products p
WHERE s.prod_id = p.prod_id
GROUP BY p.prod_category
ORDER BY p.prod_category;
```

对应的执行计划输出如下：

```
--------------------------------------------------
| Id  | Operation                     | Name     |
--------------------------------------------------
|   1 |  SORT GROUP BY                |          |
|   2 |   MAT_VIEW REWRITE ACCESS FULL| SALES_MV |
--------------------------------------------------
```

## 使用过程分析查询重写失败原因

如果您的某个 SQL 语句没有使用查询重写，并且您不理解原因，可以使用`dbms_mview`包中的过程`explain_rewrite`来找出问题所在。以下 PL/SQL 块展示了如何使用它。请注意，参数`query`指定了应被重写的查询，参数`mv`指定了应用于重写的物化视图，参数`statement_id`指定了一个任意字符串，用于标识存储在输出表`rewrite_table`中的信息。

```sql
ALTER SESSION SET query_rewrite_integrity = enforced;
DECLARE
    l_query CLOB := 'SELECT upper(p.prod_category) AS prod_category,
                            sum(s.amount_sold) AS amount_sold
                     FROM sales s, products p
                     WHERE s.prod_id = p.prod_id
                     GROUP BY p.prod_category, p.prod_status
                     ORDER BY p.prod_category';
BEGIN
    dbms_mview.explain_rewrite(
        query        => l_query,
        mv           => 'sales_mv',
        statement_id => '42'
    );
END;
/
```

> **注意**
> 默认情况下，表`rewrite_table`不存在。您可以通过执行存储在`$ORACLE_HOME/rdbms/admin`下的脚本`utlxrw.sql`，在用于分析的模式中创建它。

过程执行后，`rewrite_table`表中的输出给出了查询重写未发生的原因。输出由记录在《错误消息》手册中的消息组成。

```sql
SELECT message
FROM rewrite_table
WHERE statement_id = '42';
```

对应的输出信息可能如下：

```
MESSAGE
------------------------------------------------------------------------
QSM-01150: query did not rewrite
QSM-01110: query rewrite not possible with materialized view SALES_MV
because it contains a join between tables (SALES and CUSTOMERS) that is
not present in the query and that potentially eliminates rows needed by
the query
QSM-01052: referential integrity constraint on table, SALES, not VALID
in ENFORCED integrity mode
```

## 理解物化视图的查询重写支持

同样重要的是，并非所有查询重写方法都适用于所有物化视图。某些物化视图仅支持完全文本匹配查询重写。其他则仅支持完全文本匹配和部分文本匹配查询重写。通常，随着物化视图复杂度（例如，使用集合运算符和层次查询等结构）的增加，支持高级查询重写方法的频率会降低。限制也取决于 Oracle 数据库引擎版本。因此，与其提供一个支持列表（此类列表已在《数据仓库指南》手册中提供），我将向您展示如何针对特定案例找出支持哪些查询重写方法。

为了说明，让我们使用以下 SQL 语句重新创建物化视图。请注意，与之前的示例相比，我只是在`GROUP BY`子句中增加了`p.prod_status`（实际上，执行这样的 SQL 语句通常是无意义的，但正如您稍后将看到的，这是部分停用查询重写的一种简单方法）。

```sql
CREATE MATERIALIZED VIEW sales_mv
ENABLE QUERY REWRITE
AS
SELECT p.prod_category, c.country_id,
       sum(s.quantity_sold) AS quantity_sold,
       sum(s.amount_sold) AS amount_sold
FROM sales s, customers c, products p
WHERE s.cust_id = c.cust_id
AND s.prod_id = p.prod_id
GROUP BY p.prod_category, c.country_id, p.prod_status;
```

要显示物化视图支持的查询重写方法，您可以查询视图`user_mviews`，如下例所示。在这种情况下，根据列`rewrite_enabled`，在物化视图级别启用了查询重写；根据列`rewrite_capability`，仅支持文本匹配查询重写（换句话说，不支持通用查询重写）。

```sql
SELECT rewrite_enabled, rewrite_capability
FROM user_mviews
WHERE mview_name = 'SALES_MV';
```

输出结果：

```
REWRITE_ENABLED REWRITE_CAPABILITY
--------------- ------------------
Y               TEXTMATCH
```

请注意，列`rewrite_capability`只能是以下值之一：`none`、`textmatch`或`general`。如果支持通用查询重写（因此也支持其他两种方法），则视图`user_mviews`提供的信息就足够了。然而，如本例所示，如果显示的值是`textmatch`，那么了解至少两件事将很有用。第一，支持两种文本匹配查询重写中的哪一种？仅支持完全文本匹配，还是也支持部分文本匹配？第二，为什么不支持通用查询重写？

## 使用 explain_mview 分析物化视图能力

要回答这些问题，您可以使用`dbms_mview`包中的过程`explain_mview`，如下例所示。请注意，参数`mv`指定了物化视图的名称，参数`stmt_id`指定了一个任意字符串，用于标识存储在输出表`mv_capabilities_table`中的信息。

```sql
dbms_mview.explain_mview(mv => 'sales_mv', stmt_id => '42')
```

> **注意**
> 默认情况下，表`mv_capabilities_table`不可用。您可以通过执行存储在`$ORACLE_HOME/rdbms/admin`下的脚本`utlxmv.sql`，在用于分析的模式中创建它。



过程的结果（见表 `mv_capabilities_table`）显示了物化视图 `sales_mv` 是否支持三种查询重写模式。如果不支持，`msgtxt` 列会指出特定查询重写模式不受支持的原因。在这种情况下，请注意问题是由至少一个仅在 `GROUP BY` 子句中引用的列引起的。检查 SQL 语句后，你可以立即识别出该列：`p.prod_status`。

```sql
SELECT capability_name, possible, msgtxt
  FROM mv_capabilities_table
 WHERE statement_id = '42'
   AND capability_name IN ('REWRITE_FULL_TEXT_MATCH',
                           'REWRITE_PARTIAL_TEXT_MATCH',
                           'REWRITE_GENERAL')
 ORDER BY seq;
```

```
CAPABILITY_NAME            POSSIBLE MSGTXT
-------------------------- --------
REWRITE_FULL_TEXT_MATCH    Y
REWRITE_PARTIAL_TEXT_MATCH N        grouping column omitted from SELECT list
REWRITE_GENERAL            N        grouping column omitted from SELECT list
```

