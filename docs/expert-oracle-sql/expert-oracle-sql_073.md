# 列统计信息

*   **NUM_DISTINCT.** 此统计信息*有时*用于确定选择率。如果对一个有五个不同值的列使用等值谓词，选择率将是 1/5 或 20%。另请参见`DENSITY`统计信息。
*   **DENSITY.** *当不使用直方图时*，此统计信息是`NUM_DISTINCT`的倒数，并*有时*用于确定选择率。如果对一个有五个不同值的列使用等值谓词，`DENSITY`列将反映 1/5 或 0.2 的选择率。
*   **NUM_NULLS.** 此列统计信息用于帮助确定基数。当你有这样的谓词时，例如 `<object_column> IS NULL` 或 `<object_column> IS NOT NULL`，该统计信息显然非常宝贵。在后一种情况下，选择的行数是对象表的`NUM_ROWS`统计信息值减去对象列的`NUM_NULLS`统计信息值。对于对象列，非空值的行数也是计算相等、不等和范围谓词基数的基础。
*   **LOW_VALUE 和 HIGH_VALUE.** 这些列统计信息的主要目的是处理范围谓词，例如 `TRANSACTION_DATE` < `DATE '2013-02-11'`。由于在清单 9-1 中创建的`STATEMENT`表里`TRANSACTION_DATE`的最大值已知是 2013 年 1 月 10 日，因此该谓词的选择率估计为 100%。我将在第 20 章中回到`LOW_VALUE`和`HIGH_VALUE`列统计信息进行更详细的解释。
*   **AVG_COL_LEN.** 此统计信息用于确定操作返回的字节数。例如，如果你从一个表中选择五列，那么估计该操作每行返回的字节数将是这五列的`AVG_COL_LEN`统计信息的总和，再加上每行的可变大小开销。

对列统计信息的这个解释似乎遗漏了很多内容。如果你从一个典型应用中随机挑选几条 SQL 语句，并尝试仅基于对象统计信息和我目前的解释来重现 CBO 的基数计算，你可能会在很少的情况下成功。但首先，让我们看看清单 9-5 并过一个非常简单的例子。

## 清单 9-5. 基于列统计信息的简单基数和字节计算

```sql
SELECT num_rows
      ,column_name
      ,num_nulls
      ,avg_col_len
      ,num_distinct
      ,ROUND (density, 3) density
  FROM all_tab_col_statistics c, all_tab_statistics t
 WHERE     t.owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
       AND t.table_name = 'STATEMENT'
       AND c.owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
       AND c.table_name = 'STATEMENT'
       AND c.column_name IN ('TRANSACTION_DATE'
                            ,'DESCRIPTION'
                            ,'POSTING_DATE'
                            ,'POSTING_DELAY');

SELECT *
  FROM statement
 WHERE description = 'Flight' AND posting_delay = 0;
```

```
NUM_ROWS  COLUMN_NAME        NUM_NULLS   AVG_COL_LEN  NUM_DISTINCT DENSITY
500       TRANSACTION_DATE   0           8            10           0.1
500       POSTING_DATE       0           8            12           0.083
500       POSTING_DELAY      0           3            3            0.333
500       DESCRIPTION        0           7            4            0.25
```

```
| Id  | Operation         | Name      | Rows  | Bytes | Time     |
|---|---|---|---|---|---|
|   0 | SELECT STATEMENT  |           |    42 |  2310 | 00:00:01 |
|   1 |  TABLE ACCESS FULL| STATEMENT |    42 |  2310 | 00:00:01 |
```

清单 9-5 首先从统计视图中选择几个相关的列统计信息，并查看基于清单 9-1 中创建的表的简单单表选择语句的执行计划。

*   第一个选择谓词是在`DESCRIPTION`列上的等值操作，`DESCRIPTION`对象列的`NUM_DISTINCT`列统计信息为 4（`DENSITY`为 1/4）。
*   第二个选择谓词是在`POSTING_DELAY`列上的等值操作，`POSTING_DELAY`对象列的`NUM_DISTINCT`列统计信息为 3（`DENSITY`为 1/3）。
*   `STATEMENT`表的`NUM_ROWS`统计信息为 500，`DESCRIPTION`和`POSTING_DELAY`列的`NUM_NULLS`统计信息为 0，因此我们从中进行选择的估计行数是 500。
*   给定 1/4 和 1/3 的选择率，从 500 行中，该语句的估计基数是 500/4/3 = 41.67，四舍五入后这就是`DBMS_XPLAN`显示的内容。

你可能会说，这都很好，但真实的查询在选择列表和谓词中包含表达式和函数调用。清单 9-6 只是稍微复杂一点，但 CBO 的估计开始变得不那么科学，也不那么准确了。

## 清单 9-6. 缺少列统计信息时的 CBO 计算

```sql
SELECT *
  FROM statement
 WHERE SUBSTR (description, 1, 1) = 'F';
```

```
| Id  | Operation         | Name      | Rows  | Bytes | Time     |
|---|---|---|---|---|---|
|   0 | SELECT STATEMENT  |           |     5 |   275 | 00:00:01 |
|   1 |  TABLE ACCESS FULL| STATEMENT |     5 |   275 | 00:00:01 |
```

这次我们的谓词涉及一个函数调用。我们没有可用的列统计信息，所以 CBO 必须选择一个任意的选择率。当面对一个等值谓词且没有可用于估计选择率的统计数据时，CBO 就直接选择 1%！硬编码的！因此，我们估计的基数是 500/100 = 5。如果你运行这个查询，实际会得到 125 行，所以估计值相差了 25 倍。

公平地说，我对 CBO 基数估计算法的解释是异常简化的，但这并不影响观点的有效性：即在没有有意义的输入数据时，CBO 通常会做出任意的估计。

