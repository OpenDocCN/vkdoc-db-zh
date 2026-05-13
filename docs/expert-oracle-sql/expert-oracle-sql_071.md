# 索引访问成本与统计

共有 50 个不同的 `CUSTOMER_CATEGORY` 值，因此当我们按一个 `CUSTOMER_CATEGORY` 值选择行时，只得到 10 行，而不是按特定 `PRODUCT_CATEGORY` 选择时得到的 50 行。仅基于选择性参数，您可能会认为，如果 CBO 使用索引访问 50 行，它也会使用索引访问 10 行。然而，由于特定 `CUSTOMER_CATEGORY` 的选择行分散在表中，聚簇因子更高，现在 CBO 估计全表扫描比索引访问更便宜。Listing 9-3 中报告的全表扫描成本为 `7`。当我们使用提示强制使用索引时，通过索引访问表的成本计算为选择性 (`1/50`) 乘以 `STATEMENT_I_CC` 索引的聚簇因子 (`500`)。这得出成本 `10`，再加上索引访问本身的成本 `1`，显示的总成本为 `11`。由于通过索引访问表的估计总成本是 `11`，而全表扫描的成本是 `5`，未加提示的选择特定 `CUSTOMER_CATEGORY` 行会使用全表扫描。

## 通过多列索引的嵌套循环访问

存在一种晦涩的情况，其中索引范围扫描的成本计算方式与我上面描述的完全不同。该情况涉及多列索引上的嵌套循环，其中索引列的值之间存在强相关性，并且所有索引列都存在等式谓词。这种晦涩的情况通过 `DISTINCT_KEYS` 的低值来识别，其计算只是简单地将 `BLEVEL` 和 `AVG_LEAF_BLOCKS_PER_KEY` 相加作为索引访问成本；表访问成本是 `AVG_DATA_BLOCKS_PER_KEY`。我提到这种情况只是为了避免让您产生 `DISTINCT_KEYS`、`AVG_LEAF_BLOCKS_PER_KEY` 和 `AVG_DATA_BLOCKS_PER_KEY` 索引统计信息未被使用的印象。有关此晦涩情况的更多信息，请参见 Jonathan Lewis (2006) 的《Cost-Based Oracle》的第 11 章。

## 函数基础索引与 TIMESTAMP WITH TIME ZONE

可以创建涉及表中列的表达式（而不仅仅是列本身）的一个或多个表达式的索引。此类索引被称为*函数基础索引*。可能令你们中的一些人感到惊讶的是，任何在 `TIMESTAMP WITH TIME ZONE` 类型的列上创建索引的尝试都会创建一个函数基础索引！这是因为 `TIMESTAMP WITH TIME ZONE` 类型的列在索引时以 Listing 9-4 所示的方式进行转换。

### Listing 9-4. 一个涉及 TIMESTAMP WITH TIME ZONE 的函数基础索引

```
SELECT transaction_date_time
  FROM statement t
 WHERE transaction_date_time = TIMESTAMP '2013-01-02 12:00:00.00 -05:00';

| Id  | Operation                           | Name                |
|---|---|---|
|   0 | SELECT STATEMENT                    |                     |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| STATEMENT           |
|   2 |   INDEX RANGE SCAN                  | STATEMENT_I_TRAN_DT |

Predicate Information (identified by operation id):

2 - access(SYS_EXTRACT_UTC("TRANSACTION_DATE_TIME")=TIMESTAMP'
              2013-01-02 17:00:00.000000000')
```

如果您查看 Listing 9-4 中查询执行计划的过滤谓词，您将看到谓词已更改，包含了一个函数 `SYS_EXTRACT_UTC`，该函数将 `TIMESTAMP WITH TIME ZONE` 数据类型转换为表示*协调世界时* (UTC) 的 `TIMESTAMP` 数据类型²。字面量中原始时区比 UTC 早五小时，这一事实反映在执行计划中显示的谓词的 `TIMESTAMP` 字面量中。进行此转换的原因是，两个涉及不同时区但实际上是同时发生的 `TIMESTAMP WITH TIME ZONE` 值被认为是相等的。请注意，由于函数基础索引排除了原始时区，即使选择列表中没有其他列，也需要访问表本身以检索时区。

![image](img/sq.jpg) **注意** 由于所有涉及 `TIMESTAMP WITH TIME ZONE` 类型列的谓词都被转换为使用 `SYS_EXTRACT_UTC` 函数，因此无法使用 `TIMESTAMP WITH TIME ZONE` 类型的对象列上的列统计信息来确定基数！

#### 位图索引

位图索引的根块和分支块的结构与 B-tree 索引相同，但叶块中的条目是指向表中多个行的位图。因此，位图索引的 `NUM_ROWS` 统计信息（在这种情况下应解释为“索引条目数”）通常远小于表中的行数，对于小表将等于该列的不同值的数量。`CLUSTERING_FACTOR` 在位图索引中未使用，并被任意设置为 `NUM_ROWS` 的副本。

