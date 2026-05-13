# SQL 聚合函数详解

## 聚合行为基础

以下是创建测试表`T1`并执行一系列查询的 SQL 代码：
```sql
CREATE TABLE t1
AS
       SELECT DATE '2013-12-31' + ROWNUM transaction_date
             ,MOD (ROWNUM, 4) + 1 channel_id
             ,MOD (ROWNUM, 5) + 1 cust_id
             ,DECODE (ROWNUM
                     ,1, 4
                     ,ROWNUM * ROWNUM + DECODE (MOD (ROWNUM, 7), 3, 3000, 0))
                 sales_amount
         FROM DUAL
   CONNECT BY LEVEL <= 100;

-- 下一个查询中不包含聚合操作

SELECT * FROM t1;

-- 下一个查询中精确返回一行

SELECT COUNT (*) cnt FROM t1;

-- 下一个查询中返回 0 行或多行

SELECT channel_id
    FROM t1
GROUP BY channel_id;

-- 下一个查询中最多返回一行

SELECT 1 non_empty_flag
  FROM t1
HAVING COUNT (*) > 0;

-- 下一个查询中返回 0 行或多行

SELECT channel_id, COUNT (*) cnt
    FROM t1
GROUP BY channel_id
  HAVING COUNT (*) < 2;
```

清单 7-1 创建了表`T1`并随后执行了数个查询。

-   清单 7-1 中的第一个查询不包含聚合函数和`GROUP BY`子句，因此不会发生聚合。
-   第二个查询包含了`COUNT(*)`聚合函数。由于未指定`GROUP BY`子句，整个表被聚合并精确返回一行。需要说明的是，如果`T1`为空，该查询仍将返回一行，且`CNT`的值为 0。
-   第三个查询包含了一个`GROUP BY`子句，因此即使语句中没有出现聚合函数，聚合操作仍然会发生。`GROUP BY`运算符会为`GROUP BY`子句中指定的表达式的每一个不同值返回一行。因此在这种情况下，会为`CHANNEL_ID`的每一个不同值返回一行，当表为空时则不返回行。
-   清单 7-1 中的第四个查询表明，触发聚合并不需要将聚合函数放在选择列表中。`HAVING`子句在聚合**之后**应用过滤条件，这与在聚合**之前**过滤的`WHERE`子句不同。因此，在这种情况下，如果`T1`为空，`COUNT(*)`的值将为 0，查询将返回 0 行。如果表中有行，查询将返回一行。
-   当然，聚合最常见的用法是`GROUP BY`子句与聚合函数结合使用。清单 7-1 中最后一个查询的`GROUP BY`子句确保只返回在`T1`中至少出现过一次的`CHANNEL_ID`值，因此`CNT`的值不可能为 0。`HAVING`子句会移除所有`COUNT(*) >= 2`的行，所以如果这个最终查询返回任何行，在所有情况下`CNT`的值都将为 1！

## 排序聚合函数

许多聚合函数需要排序才能有效运行。清单 7-2 展示了如何使用`MEDIAN`聚合函数。

清单 7-2. 使用排序聚合函数
```sql
 SELECT channel_id, MEDIAN (sales_amount) med, COUNT (*) cnt
    FROM t1
GROUP BY channel_id;
```

```
| Id  | Operation          |

|   0 | SELECT STATEMENT   |
|   1 |  SORT GROUP BY     |
|   2 |   TABLE ACCESS FULL|
```

![image](img/sq.jpg) `Note` 奇数个数字的中位数是排序后列表中的中间那个数。对于偶数个数值，它是两个中间数字的平均值（均值）。因此，2, 4, 6, 8, 100, 102 的中位数是 7（6 和 8 的平均值）。

从清单 7-2 开始，我将使用`DBMS_XPLAN.DISPLAY`生成的格式展示 SQL 语句的执行计划，而不使用`EXPLAIN PLAN`语句，也不展示对`DBMS_XPLAN.DISPLAY`函数的显式调用。此外，我将只展示执行计划的相关部分。这些措施纯粹是为了节省空间。可下载的材料并未以这种方式缩写。

清单 7-2 将我们在清单 7-1 中创建的表`T1`的 100 行数据进行聚合，返回四行，每行对应一个不同的`CHANNEL_ID`值。这些行中的`MED`值代表每个组内 25 行数据的中位数。该语句的基本执行计划也显示在清单 7-2 中，操作 1 就是用于实现此目的的排序。

![image](img/sq.jpg) `Note` 说实话，清单 7-2 中实际上执行了六次排序！第一次是按`C1`排序以创建四个组。然后，每个组按照`MEDIAN`函数的要求按`SALES_AMOUNT`排序。如果你在清单 7-2 查询前后检查“*sorts (rows)*”统计信息，你会发现它增加了 200，是表中行数的两倍；每一行被排序了两次。并且如果使用了多个具有不同排序顺序的聚合函数，那么每个组可能会被排序多次。

可用于检查该统计信息的代码是：
```sql
SELECT VALUE FROM v$mystat NATURAL JOIN v$statname WHERE name = 'sorts (rows)';
```

### 非排序聚合函数

并非所有聚合函数都需要排序。我使用术语“*非排序聚合函数*”（NSAF）来指代不需要排序的聚合函数。NSAF 的一个例子是`COUNT`。`COUNT`不需要排序：随着每一行的处理，`COUNT`只是简单地增加一个运行中的计数。其他 NSAF，如`SUM`、`AVG`和`STDDEV`，也是通过维护一种或另一种形式的运行计数来实现的。

当查询块中同时出现 NSAF 和`GROUP BY`子句时，NSAF 需要为每个组维护运行计数。Oracle 数据库 10g 引入了一项称为*哈希聚合*的功能，它允许 NSAF 通过在哈希表中查找来找到每个组的计数，而不是从排序列表中查找。让我们看一下清单 7-3，它展示了该功能的实际应用。

清单 7-3. 针对 NSAF 的哈希聚合
```sql
 SELECT channel_id
        ,AVG (sales_amount)
        ,SUM (sales_amount)
        ,STDDEV (sales_amount)
        ,COUNT (*)
    FROM t1
GROUP BY channel_id;
```

```
| Id  | Operation          | Name |

|   0 | SELECT STATEMENT   |      |
|   1 |  HASH GROUP BY     |      |
|   2 |   TABLE ACCESS FULL| T1   |
```

哈希聚合要求查询块中使用*所有*聚合函数都是 NSAF；清单 7-3 中的所有函数都是 NSAF，因此使用了操作`HASH GROUP BY`来避免任何排序。

![image](img/sq.jpg) `Note` 并非所有 NSAF 都能利用哈希聚合。不能利用的包括`COLLECT`、`FIRST`、`LAST`、`RANK`、`SYS_XMLAGG`和`XMLAGG`。这些函数在使用`GROUP BY`子句时使用`SORT GROUP BY`操作，在未指定`GROUP BY`子句时使用非排序的`SORT AGGREGATE`操作。此限制可能是由于聚合值的可变大小造成的。

仅仅因为 CBO 可以使用哈希聚合，并不意味着它一定会使用。当被分组的数据已经排序或无论如何都需要排序时，将不会使用哈希聚合。清单 7-4 提供了两个例子。

清单 7-4. 针对 NSAF 的 SORT GROUP BY
```sql
 SELECT channel_id, AVG (sales_amount) mean
    FROM t1
   WHERE channel_id IS NOT NULL
GROUP BY channel_id
ORDER BY channel_id;

CREATE INDEX t1_i1
   ON t1 (channel_id);

SELECT channel_id, AVG (sales_amount) mean
    FROM t1
   WHERE channel_id IS NOT NULL
GROUP BY channel_id;

DROP INDEX t1_i1;
```

```
| Id  | Operation          | Name |

|   0 | SELECT STATEMENT   |      |
|   1 |  SORT GROUP BY     |      |
|*  2 |   TABLE ACCESS FULL| T1   |

| Id  | Operation                    | Name  |
```


| Id | Operation | Name |
|---|---|--- |
| 0 | `SELECT STATEMENT` | |
| 1 | `SORT GROUP BY NOSORT` | |
| 2 | `TABLE ACCESS BY INDEX ROWID` | `T1` |
| *3 | `INDEX FULL SCAN` | `T1_I1` |

`清单 7-4` 中的第一个查询包含一个 `ORDER BY` 子句。在这种情况下，排序无法避免，因此哈希聚合没有任何优势。

> **注意**：你可能会认为，先执行哈希聚合，再对聚合后的行进行排序会更高效。毕竟，如果你有 1,000,000 行数据和 10 个分组，难道对聚合后的 10 行进行排序不比对 1,000,000 条未聚合的行排序更好吗？实际上，在这种情况下，排序工作区中只会保留 10 行数据；非全聚合函数（NSAF）会像哈希聚合一样，为每个分组维护其运行时的累计值。

在执行 `清单 7-4` 中的第二个查询之前，我们在 `CHANNEL_ID` 上创建了一个索引。该索引会以排序后的顺序返回数据，因此我们既不需要显式排序，也不需要哈希表，聚合是按分组依次处理的。请注意，此时操作 1 的名称现在是自相矛盾的：`SORT GROUP BY NOSORT`。显然，没有发生排序操作。

## 分析函数

既然我们已经完全理解了什么是聚合函数以及它们是如何处理的，现在可以转向更复杂的概念——分析函数。分析函数有潜力使 SQL 语句易于阅读且性能良好。但如果使用不当，它们也可能使 SQL 语句难以理解且运行缓慢。

### 分析的概念

我们说过，聚合函数会丢弃被聚合的原始行。如果这不是我们想要的结果，会发生什么？还记得 `清单 3-1` 吗？那个查询列出了 `SCOTT.EMP` 表中的每一行，以及关于该员工所在部门的聚合数据。我们使用子查询来实现这一点。`清单 7-5` 是使用 `清单 7-1` 中的表 `T1` 和 `COUNT` 函数的另一个相同技术的示例。

**`清单 7-5`**：使用子查询实现分析功能

```sql
SELECT outer.*
      , (SELECT COUNT (*)
           FROM t1 inner
          WHERE inner.channel_id = outer.channel_id)
          cnt
  FROM t1 outer;

| Id | Operation | Name |
|---|---|--- |
| 0 | `SELECT STATEMENT` | |
| 1 | `SORT GROUP BY` | |
| *2 | `TABLE ACCESS FULL` | `T1` |
| 3 | `TABLE ACCESS FULL` | `T1` |
```

`清单 7-5` 中的查询列出了 `T1` 表中的 100 行，并附加了一个额外的列，其中包含对应 `CHANNEL_ID` 值的匹配行数。该查询的执行计划似乎表明我们将对 `T1` 表中的 100 行中的每一行执行一次子查询。然而，与 `清单 3-1` 一样，标量子查询缓存意味着我们只执行四次子查询，每个 `CHANNEL_ID` 的不同值执行一次。尽管如此，当我们将主查询中的操作 3 加入计算时，在整个查询过程中仍然会对 `T1` 执行五次全表扫描。

子查询也可以用来表示排序。`清单 7-6` 展示了如何为排序列表添加行号和排名。

**`清单 7-6`**：使用 `ROWNUM` 和子查询对输出进行编号

```sql
WITH q1
     AS (  SELECT *
             FROM t1
         ORDER BY transaction_date)
  SELECT ROWNUM rn
        , (SELECT COUNT (*)+1
             FROM q1 inner
            WHERE inner.sales_amount < outer.sales_amount)
            rank_sales, outer.*
    FROM q1 outer
ORDER BY sales_amount;

-- 输出的前几行
RN RANK_SALES TRANSACTI CHANNEL_ID CUST_ID SALES_AMOUNT
---------- ---------- --------- ---------- ---------- ------------
         1          1 01-JAN-14          2          2            4
         2          1 02-JAN-14          3          3            4
         4          3 04-JAN-14          1          5           16
         5          4 05-JAN-14          2          1           25

| Id | Operation | Name |
|---|---|--- |
| 0 | `SELECT STATEMENT` | |
| 1 | `SORT AGGREGATE` | |
| *2 | `VIEW` | |
| 3 | `TABLE ACCESS FULL` | `SYS_TEMP_0FD9D664B_479D89AC` |
| 4 | `TEMP TABLE TRANSFORMATION` | |
| 5 | `LOAD AS SELECT` | `SYS_TEMP_0FD9D664B_479D89AC` |
| 6 | `SORT ORDER BY` | |
| 7 | `TABLE ACCESS FULL` | `T1` |
| 8 | `SORT ORDER BY` | |
| 9 | `COUNT` | |
| 10 | `VIEW` | |
| 11 | `TABLE ACCESS FULL` | `SYS_TEMP_0FD9D664B_479D89AC` |
```

`清单 7-6` 中的因子化子查询 `Q1` 对表中的行进行排序，然后我们使用 `ROWNUM` 伪列来标识每行在排序列表中的位置。`RANK_SALES` 列与 `RN` 略有不同，因为具有相同销售额的行（如前两行）被赋予相同的位置。该语句的执行计划远非直截了当。肯定有一种更简单、更高效的方法！确实有；分析函数为许多低效的子查询提供了解决方案。

> **注意**：所有分析函数都有一个涉及子查询的等效表述。有时分析函数比子查询性能更好，有时子查询比分析函数性能更好。

### 用作分析的聚合函数

大多数（但并非全部）聚合函数也可以用作分析函数。`清单 7-7` 重写了 `清单 7-5`，这次将 `COUNT` 作为分析函数而不是聚合函数使用。

**`清单 7-7`**：聚合函数用作分析函数

```sql
SELECT t1.*, COUNT (sales_amount) OVER (PARTITION BY channel_id) cnt FROM t1;

| Id | Operation | Name |
|---|---|--- |
| 0 | `SELECT STATEMENT` | |
| 1 | `WINDOW SORT` | |
| 2 | `TABLE ACCESS FULL` | `T1` |
```

`清单 7-7` 中 `OVER` 子句的存在表明 `COUNT` 函数被用作分析函数而不是聚合函数。该查询返回 `T1` 的所有 100 行，并添加了列 `CNT`，就像在 `清单 7-5` 中一样。尽管 `清单 7-5` 和 `清单 7-7` 返回的行是相同的，但查询的执行方式却大不相同，而且返回的行顺序很可能也不同。

`清单 7-7` 中显示的执行计划包含了分析函数特有的 `WINDOW SORT` 操作。与可能执行多次排序的 `SORT GROUP BY` 操作不同，`WINDOW SORT` 操作只对其从子操作接收的行执行一次排序。在 `清单 7-7` 的情况下，行按照包含 `CHANNEL_ID` 和 `SALES_AMOUNT` 列的复合键排序，这隐式地为每个 `CHANNEL_ID` 值创建了一个分组。然后，`COUNT` 函数分两个阶段处理每个分组：


*   第一阶段，对组进行处理，以使用 `COUNT` 函数的聚合变体来计算该组的 `COUNT` 值。
*   第二阶段，输出组中的每一行，同时为每一行提供一个额外的列，其中包含相同的计算计数值。

术语 `PARTITION BY` 在分析函数中的作用，相当于聚合查询中的 `GROUP BY` 子句；如果省略 `PARTITION BY` 关键字，分析函数将处理整个结果集。

清单 7-8 展示了一个处理整个结果集的简单分析函数示例。

清单 7-8. 在分析函数中使用非序列聚合函数

```sql
SELECT t1.*, AVG (sales_amount) OVER ()average_sales_amount FROM t1;

| Id  | Operation          | Name |

|   0 | SELECT STATEMENT   |      |
|   1 |  WINDOW BUFFER     |      |
|   2 |   TABLE ACCESS FULL| T1   |
```

清单 7-8 中的查询添加了一个列 `AVERAGE_SALES_AMOUNT`，该列在 `T1` 表的所有 100 行中具有相同的值。该值是整个表中 `SALES_AMOUNT` 的平均值。请注意，因为 `AVG` 是一个非序列聚合函数，并且没有 `PARTITION BY` 子句，所以根本不需要排序。因此，执行计划使用了 `WINDOW BUFFER` 操作而不是 `WINDOW SORT`。

与聚合一样，当使用多个函数时，可能需要多次排序。然而，在分析函数的情况下，每次排序都有自己的操作。比较一下 清单 7-9 中两个查询的执行计划。

清单 7-9. 比较聚合和分析函数的多次排序

```sql
SELECT MEDIAN (channel_id) med_channel_id
      ,MEDIAN (sales_amount) med_sales_amount
  FROM t1;

| Id  | Operation          | Name |

|   0 | SELECT STATEMENT   |      |
|   1 |  SORT GROUP BY     |      |
|   2 |   TABLE ACCESS FULL| T1   |

SELECT t1.*
      ,MEDIAN (channel_id) OVER ()med_channel_id
      ,MEDIAN (sales_amount) OVER ()med_sales_amount
  FROM t1;

| Id  | Operation           |

|   0 | SELECT STATEMENT    |
|   1 |  WINDOW SORT        |
|   2 |   WINDOW SORT       |
|   3 |    TABLE ACCESS FULL|
```

清单 7-9 中的第一个查询返回一行，其中包含 `CHANNEL_ID` 和 `SALES_AMOUNT` 的中值。每个聚合函数调用都需要自己的排序：第一次按 `CHANNEL_ID` 排序，第二次按 `SALES_AMOUNT` 排序。然而，这个细节被 `SORT GROUP BY` 操作掩盖了。第二个查询涉及分析函数调用，返回 `T1` 中的所有 100 行。这第二个查询也需要两次排序。但是，在这种情况下，这两次排序在执行计划中各自都有自己的 `WINDOW SORT` 操作。

## 纯分析函数

虽然几乎所有的聚合函数都可以作为分析函数运行，但有许多分析函数没有对应的聚合函数。例如，清单 7-10 以一种更易读且更高效的方式重写了 清单 7-6。

清单 7-10. 引入 ROW_NUMBER 和 RANK 分析函数

```sql
 SELECT ROW_NUMBER () OVER (ORDER BY sales_amount) rn
        ,RANK () OVER (ORDER BY sales_amount) rank_sales
        ,t1.*
    FROM t1
ORDER BY sales_amount;

| Id  | Operation          |

|   0 | SELECT STATEMENT   |
|   1 |  WINDOW SORT       |
|   2 |   TABLE ACCESS FULL|
```

清单 7-10 中查询的输出与清单 7-6 中的输出相同。`WINDOW SORT` 操作按 `SALES_AMOUNT` 对行进行排序，然后 `ROW_NUMBER` 分析函数从排序后的缓冲区读取行，并添加一个指示其在列表中位置的新列。`RANK` 函数与 `ROW_NUMBER` 类似，只是 `SALES_AMOUNT` 值相等的行被赋予相同的排名。请注意，`ORDER BY` 子句不会导致额外的开销，因为数据已经排序。但这种行为不能保证，并且万一 CBO 在将来的版本中做了不同的事情，如果排序很重要，最好添加 `ORDER BY` 子句。

## 分析子句

我们已经看到分析函数包含关键字 `OVER`，后跟一对括号。在这些括号内，最多有三个可选子句。它们是 `query partition` 子句、`order by` 子句和 `windowing` 子句。如果存在多个子句，则必须按照规定的顺序书写。让我们更详细地看看这些子句中的每一个。

### 查询分区子句

此子句使用关键字 `PARTITION BY` 将行分成组，然后独立地对每个组运行分析函数。

![image](img/sq.jpg) **注意**  分析函数中使用的关键字 `PARTITION BY` 与允许将索引和表拆分为多个段的许可功能无关。分析函数中的术语 `PARTITION BY` 与我们第一章中讨论的分区外连接概念之间也没有任何关系。

我们在清单 7-7 中看到了一个查询分区子句的例子。清单 7-11 给我们提供了另一个例子，它是对清单 7-8 的一个轻微修改。

清单 7-11. 带 PARTITION BY 子句的 NSAF 分析函数

```sql
SELECT t1.*
      ,AVG (sales_amount) OVER (PARTITION BY channel_id) average_sales_amount
  FROM t1;

| Id  | Operation          |

|   0 | SELECT STATEMENT   |
|   1 |  WINDOW SORT       |
|   2 |   TABLE ACCESS FULL|
```

清单 7-8 中的查询与 清单 7-11 中的查询之间的唯一区别是添加了查询分区子句。因此，在清单 7-8 中，`CHANNEL_ID` 为 1 且 `SALES_AMOUNT` 为 16 的行的 `AVERAGE_SALES_AMOUNT` 值为 3803.53，就像每一行一样。`T1` 表中所有 100 行的 `SALES_AMOUNT` 的平均值是 3803.53。另一方面，包含 `PARTITION BY` 子句的查询在 `CHANNEL_ID` 为 1 且 `SALES_AMOUNT` 为 16 时返回的 `AVERAGE_SALES_AMOUNT` 值为 3896。这是 `CHANNEL_ID` 为 1 的 25 行的平均值。

你会注意到 清单 7-11 中的执行计划与 清单 7-8 中的不同。清单 7-11 中的查询分区子句要求按 `CHANNEL_ID` 对行进行排序，即使 `AVG` 函数是一个非序列聚合函数。

### ORDER BY 子句

当 `MEDIAN` 函数用作分析函数时，行的排序顺序由 `MEDIAN` 函数的参数决定，在这种情况下，`order by` 子句是多余的，并且是非法的。然而，我们已经在清单 7-10 中看到了一个合法使用 `order by` 子句的例子，该清单引入了 `ROW_NUMBER` 分析函数。清单 7-12 显示了 `order by` 子句的另一个用法。

清单 7-12. NTILE 分析函数

```sql
SELECT t1.*, NTILE (4) OVER (ORDER BY sales_amount) nt FROM t1;
```



