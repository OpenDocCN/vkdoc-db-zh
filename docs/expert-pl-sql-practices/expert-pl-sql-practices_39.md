# 扩展统计

统计信息是基于函数的索引的优势之一。这些信息对于 CBO（基于成本的优化器）来说可能非常宝贵（即使它选择不使用该索引，也可以利用这些统计信息来决定连接或谓词的顺序）。然而，为 PL/SQL 函数谓词创建索引可能并不总是可行。在这种情况下，Oracle 数据库 11g 提供了一种替代方案，称为**扩展统计**。

扩展统计可以在无需支持性索引的情况下，基于表达式创建。此类统计信息被称为**扩展**，它们相对于基于函数的索引有一个额外好处：可应用于查询中的所有 PL/SQL 函数表达式（即不仅限于谓词中的表达式）。

为了演示这一点，我重新执行了按客户分类的销售报告，并启用了扩展统计。清单 9-46 重温了未使用扩展统计时的查询，以及来自清单 9-5 的执行计划和函数调用输出。

## 清单 9-46. 在投影的 PL/SQL 函数上使用扩展统计（无统计信息的执行计划，来自清单 9-5）

```sql
SQL> SELECT t.calendar_year
  2  ,      format_customer_name(
  3            c.cust_first_name, c.cust_last_name
  4            )                 AS cust_name
  5  ,      SUM(s.quantity_sold) AS qty_sold
  6  ,      SUM(s.amount_sold)   AS amt_sold
  7  FROM   sales     s
  8  ,      customers c
  9  ,      times     t
 10  WHERE  s.cust_id = c.cust_id
 11  AND    s.time_id = t.time_id
 12  GROUP  BY
 13         t.calendar_year
 14  ,      format_customer_name(
 15            c.cust_first_name, c.cust_last_name
 16            )
 17  ;

11604 rows selected.
```

耗时: 00:00:06.94

```
执行计划
----------------------------------------------------------
Plan hash value: 3113689673

-------------------------------------------------------------
| Id  | Operation                      | Name      | Rows  |
-------------------------------------------------------------
|   0 | SELECT STATEMENT               |           |   918K|
|   1 |  HASH GROUP BY                 |           |   918K|
|*  2 |   HASH JOIN                    |           |   918K|
|   3 |    PART JOIN FILTER CREATE     | :BF0000   |  1826 |
|   4 |     TABLE ACCESS FULL          | TIMES     |  1826 |
|*  5 |    HASH JOIN                   |           |   918K|
|   6 |     TABLE ACCESS FULL          | CUSTOMERS | 55500 |
|   7 |     PARTITION RANGE JOIN-FILTER|           |   918K|
|   8 |      TABLE ACCESS FULL         | SALES     |   918K|
-------------------------------------------------------------

统计信息
----------------------------------------------------------
       3189  一致读
          0  物理读

SQL> exec counter.show('Function calls');
```

函数调用次数: 918843

```
PL/SQL 过程已成功完成。
```

如你所见，这执行了超过 918,000 次函数调用，并在大约 7 秒内完成。在清单 9-47 中，我使用了`DBMS_STATS`在 PL/SQL 函数调用上生成扩展统计，并展示了重新执行查询后的执行计划。

## 清单 9-47. 生成并使用扩展统计

```sql
SQL> BEGIN
  2     DBMS_STATS.GATHER_TABLE_STATS(
  3        ownname    => USER,
  4        tabname    => 'CUSTOMERS',
  5        method_opt => 'FOR COLUMNS (format_customer_name(cust_first_name,cust_last_name)) SIZE AUTO'
  6        );
  7  END;
  8  /

PL/SQL 过程已成功完成。

SQL> SELECT t.calendar_year...

11604 rows selected.
```

耗时: 00:00:00.90

```
执行计划
----------------------------------------------------------
Plan hash value: 833790846

---------------------------------------------------------------
| Id  | Operation                        | Name      | Rows  |
---------------------------------------------------------------
|   0 | SELECT STATEMENT                 |           | 19803 |
|   1 |  HASH GROUP BY                   |           | 19803 |
|*  2 |   HASH JOIN                      |           | 24958 |
|   3 |    VIEW                          | VW_GBC_9  | 24958 |
|   4 |     HASH GROUP BY                |           | 24958 |
|*  5 |      HASH JOIN                   |           |   918K|
|   6 |       PART JOIN FILTER CREATE    | :BF0000   |  1826 |
|   7 |        TABLE ACCESS FULL         | TIMES     |  1826 |
|   8 |       PARTITION RANGE JOIN-FILTER|           |   918K|
|   9 |        TABLE ACCESS FULL         | SALES     |   918K|
|  10 |    TABLE ACCESS FULL             | CUSTOMERS | 55500 |
---------------------------------------------------------------

SQL> exec counter.show('Extended stats');
```

扩展统计调用次数: 237917

```
PL/SQL 过程已成功完成。
```

得益于 PL/SQL 函数的扩展统计信息，CBO 做出了明智的决策来重写查询并预先对 SALES 和 TIMES 数据进行分组。这体现在执行计划的步骤 3 中。由于这种预分组减少了结果集，PL/SQL 函数调用次数减少到 238,000 次以下。总体而言，对响应时间的影响是显著的。

扩展统计是前面演示的`NO_MERGE`提示的一个良好替代方案。若没有扩展统计，清单 9-48 中显示的查询会被 CBO 合并到主查询块中，导致 918,000 次函数执行。幸运的是，生成扩展统计可以很好地阻止这种转换，同样取得了良好效果。

## 清单 9-48. 扩展统计及对基于成本的查询转换的影响

```sql
SQL> SELECT t.calendar_year
  2  ,      c.cust_name
  3  ,      SUM(s.quantity_sold) AS qty_sold
  4  ,      SUM(s.amount_sold)   AS amt_sold
  5  FROM   sales     s
  6  ,      (
  7         SELECT cust_id
  8         ,      format_customer_name (
  9                   cust_first_name, cust_last_name
 10                   ) AS cust_name
 11         FROM   customers
 12        )          c
 13  ,      times     t
 14  WHERE  s.cust_id = c.cust_id
 15  AND    s.time_id = t.time_id
 16  GROUP  BY
 17         t.calendar_year
 18  ,      c.cust_name
 19  ;

11604 rows selected.
```

耗时: 00:00:00.94

```
执行计划
----------------------------------------------------------
Plan hash value: 833790846

---------------------------------------------------------------
| Id  | Operation                        | Name      | Rows  |
---------------------------------------------------------------
|   0 | SELECT STATEMENT                 |           | 19803 |
|   1 |  HASH GROUP BY                   |           | 19803 |
|*  2 |   HASH JOIN                      |           | 24958 |
|   3 |    VIEW                          | VW_GBC_9  | 24958 |
|   4 |     HASH GROUP BY                |           | 24958 |
|*  5 |      HASH JOIN                   |           |   918K|
|   6 |       PART JOIN FILTER CREATE    | :BF0000   |  1826 |
|   7 |        TABLE ACCESS FULL         | TIMES     |  1826 |
|   8 |       PARTITION RANGE JOIN-FILTER|           |   918K|
|   9 |        TABLE ACCESS FULL         | SALES     |   918K|
|  10 |    TABLE ACCESS FULL             | CUSTOMERS | 55500 |
---------------------------------------------------------------

SQL> exec counter.show('Extended stats + inline view');
```

扩展统计 + 内联视图调用次数: 17897



有趣的是，对于这个重新排列的查询，CBO 生成了相同的执行计划，尽管函数执行次数已从 234,000 次大幅下降到仅 18,000 次。这是函数执行具有不可预测性的又一个绝佳例证——我得到了相同的执行计划，但函数调用次数却存在超过一个数量级的差异。尽管如此，这些清单清晰地突显了针对 PL/SQL 函数的统计信息对于 CBO 而言是多么重要，以及扩展统计信息是一种为其提供信息的绝佳机制。

![images](img/square.jpg) 注意：扩展统计信息（或简称扩展）是通过系统生成的虚拟列（类似于基于函数的索引）实现的。要研究此功能的更多细节，请查询数据字典视图，如 `USER_TAB_COLS` 和 `USER_STAT_EXTENSIONS`。

## 默认统计信息

与扩展统计信息不同，默认统计信息是由用户生成的，并且仅适用于谓词中的 PL/SQL 函数。此外，CBO 不会使用默认统计信息来指导基于成本的查询转换，但如果你的 WHERE 子句中有多个函数调用，它将使用它们来确定谓词的排序。

默认统计信息使用 `ASSOCIATE STATISTICS` SQL 命令提供，通过这些命令，你可以为你的 PL/SQL 函数定义选择性、CPU 和 I/O 成本的统计信息（从而改进 CBO 否则会采用的默认值）。

在清单 9-19 中，当我引用查询中的两个函数（它们被恰当地命名为 `QUICK_FUNCTION` 和 `SLOW_FUNCTION`）时，我强调了默认谓词排序的影响。清单 9-49 展示了应用默认统计信息如何确保 CBO 以最高效的顺序应用函数谓词。

**清单 9-49.** 为 PL/SQL 函数谓词设置默认统计信息
```sql
SQL> ASSOCIATE STATISTICS WITH FUNCTIONS quick_function DEFAULT SELECTIVITY 0.1;

Statistics associated.

SQL> ASSOCIATE STATISTICS WITH FUNCTIONS slow_function DEFAULT SELECTIVITY 50;

Statistics associated.

SQL> SELECT *
  2  FROM   thousand_rows
  3  WHERE  slow_function(n1) = 0
  4  AND    quick_function(n1) = 0;

1 row selected.

Elapsed: 00:00:00.02
Execution Plan
----------------------------------------------------------
Plan hash value: 4012467810

-----------------------------------------------------------------------------------
| Id  | Operation         | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |               |     1 |    55 |     5   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| THOUSAND_ROWS |     1 |    55 |     5   (0)| 00:00:01 |
-----------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter("QUICK_FUNCTION"("N1")=0 AND "SLOW_FUNCTION"("N1")=0)
```

由于与 `SLOW_FUNCTION` 和 `QUICK_FUNCTION` 相关的选择性统计信息，你可以看到 CBO 已选择重新排序谓词，效果良好。我已告知 Oracle 数据库，针对 `QUICK_FUNCTION` 的相等性谓词只有千分之一的几率为真，而 `SLOW_FUNCTION` 谓词则有二分之一的几率为真。显然，先应用 `QUICK_FUNCTION` 以尽早减少数据源是有意义的。清单 9-50 中 10053 跟踪文件的摘录显示了 CBO 对此查询的处理过程。

**清单 9-50.** 默认选择性统计信息的 10053 跟踪文件摘录
```text
***************************************
BASE STATISTICAL INFORMATION
***********************
Table Stats::
  Table: THOUSAND_ROWS  Alias: THOUSAND_ROWS
    #Rows: 1000  #Blks:  12  AvgRowLen:  55.00  ChainCnt:  0.00
Access path analysis for THOUSAND_ROWS
***************************************
SINGLE TABLE ACCESS PATH
  Single Table Cardinality Estimation for THOUSAND_ROWS[THOUSAND_ROWS]
  No statistics type defined for function SLOW_FUNCTION
  No default cost defined for function SLOW_FUNCTION
  No statistics type defined for function SLOW_FUNCTION
  Default selectivity for function SLOW_FUNCTION: 50.00000000%
  No statistics type defined for function QUICK_FUNCTION
  No default cost defined for function QUICK_FUNCTION
  No statistics type defined for function QUICK_FUNCTION
  Default selectivity for function QUICK_FUNCTION: 0.10000000%
  Table: THOUSAND_ROWS  Alias: THOUSAND_ROWS
    Card: Original: 1000.000000  Rounded: 1  Computed: 0.50  Non Adjusted: 0.50
  Access Path: TableScan
    Cost:  5.10  Resp: 5.10  Degree: 0
      Cost_io: 5.00  Cost_cpu: 3288527
      Resp_io: 5.00  Resp_cpu: 3288527
  Best:: AccessPath: TableScan
         Cost: 5.10  Degree: 1  Resp: 5.10  Card: 0.50  Bytes: 0
```

`ASSOCIATE STATISTICS` 命令也可用于为 PL/SQL 函数提供成本信息，这与上述选择性统计信息具有类似的影响。清单 9-51 展示了如何为 `GET_RATE` PL/SQL 函数设置默认成本统计信息：

**清单 9-51.** 为 PL/SQL 函数设置默认成本统计信息
```sql
SQL> ASSOCIATE STATISTICS WITH FUNCTIONS get_rate DEFAULT COST (403416, 2, 0);

Statistics associated.
```

我按如下方式计算这些成本统计信息（按参数顺序）：

*   CPU 成本 (403416)：由 `DBMS_ODCI.ESTIMATE_CPU_UNITS(<ms>)` 函数计算，其中 `<ms>` 是执行一次 `GET_RATE` 函数所需的毫秒数（由分层分析器报告）。
*   I/O 成本 (2)：执行一次 `GET_RATE` 的逻辑和物理 I/O 总和（由 Autotrace 报告）。
*   网络成本 (0)：此功能尚未实现，因此可以保留为 0。

可以在单个 `ASSOCIATE STATISTICS` 命令中同时设置选择性和成本，并且在使用此策略时同时提供两者是有意义的。



### 可扩展优化器

我将通过简要概述可扩展优化器，来结束本次向 CBO 提供统计信息的讲解。此功能（Oracle Data Cartridge 工具集的一部分）将默认统计信息提升到了一个新的阶段，它通过一种对象类型来关联选择率和成本，而不是像前面提到的那些硬编码的默认值。

要用许多页的篇幅来描述可扩展优化器是可能的，因此我只提供对该技术的概览，向您展示如何构建一个用于估算您的 PL/SQL 函数统计信息的方法，该方法能够适应您的数据模式。

请看代码清单 9-52。我有一个简单的函数，用于返回大写的产品代码。在没有统计信息的情况下，CBO 会假定任何使用此函数的谓词都具有 1% 的选择率，如下所示。

**代码清单 9-52.** PL/SQL 函数谓词的默认选择率

```sql
SQL> CREATE FUNCTION format_prod_category(
  2                    p_prod_category IN VARCHAR2
  3                    ) RETURN VARCHAR2 DETERMINISTIC IS
  4  BEGIN
  5     RETURN UPPER(p_prod_category);
  6  END format_prod_category;
  7  /

Function created.

SQL> SELECT *
  2  FROM   products
  3  WHERE  format_prod_category(prod_category) = 'SOFTWARE/OTHER';

Execution Plan
----------------------------------------------------------
Plan hash value: 1954719464

------------------------------------------------
| Id  | Operation         | Name     | Rows  |
------------------------------------------------
|   0 | SELECT STATEMENT  |          |     1 |
|*  1 |  TABLE ACCESS FULL| PRODUCTS |     1 |
------------------------------------------------
```

CBO 估计此谓词的基数仅为 1 行，但我确切地知道，Software/Other 类别有更多的行，正如代码清单 9-53 中显示的数据概况所示。

**代码清单 9-53.** PRODUCTS 表的数据概况

```sql
SQL> SELECT prod_category
  2  ,      c                                           AS num_rows
  3  ,      ROUND(RATIO_TO_REPORT(c) OVER () * 100, 1) AS selectivity
  4  FROM   (
  5         SELECT prod_category
  6         ,      COUNT(*) AS c
  7         FROM   products
  8         GROUP  BY
  9                prod_category
 10        )
 11  ORDER  BY
 12         num_rows;

PROD_CATEGORY                     NUM_ROWS SELECTIVITY
-------------------------------- ---------- -----------
Hardware                                 2         2.8
Photo                                   10        13.9
Electronics                             13        18.1
Peripherals and Accessories             21        29.2
Software/Other                          26        36.1
```

您可以看到此数据中的选择率范围很广，因此默认的统计信息解决方案不够灵活，无法覆盖所有类别。因此，您可以使用可扩展优化器为此函数谓词构建一个动态统计信息解析器。如前所述，这是使用一个对象类型实现的，其规范在代码清单 9-54 中提供。

**代码清单 9-54.** 为可扩展优化器创建统计信息类型

```sql
SQL> CREATE TYPE prod_stats_ot AS OBJECT (
  2  
  3     dummy_attribute NUMBER,
  4
  5     STATIC FUNCTION ODCIGetInterfaces (
  6                      p_interfaces OUT SYS.ODCIObjectList
  7                      ) RETURN NUMBER,
  8  
  9     STATIC FUNCTION ODCIStatsSelectivity (
 10                      p_pred_info       IN  SYS.ODCIPredInfo,
 11                      p_selectivity     OUT NUMBER,
 12                      p_args            IN  SYS.ODCIArgDescList,
 13                      p_start           IN  VARCHAR2,
 14                      p_stop            IN  VARCHAR2,
 15                      p_prod_category   IN  VARCHAR2,
 16                      p_env             IN  SYS.ODCIEnv
 17                      ) RETURN NUMBER,
 18  
 19     STATIC FUNCTION ODCIStatsFunctionCost (
 20                      p_func_info       IN  SYS.ODCIFuncInfo,
 21                      p_cost            OUT SYS.ODCICost,
 22                      p_args            IN  SYS.ODCIArgDescList,
 23                      p_prod_category   IN  VARCHAR2,
 24                      p_env             IN  SYS.ODCIEnv
 25                      ) RETURN NUMBER
 26  );
 27  /

Type created.
```

可扩展优化器使用定义良好的接口方法，代码清单 9-54 使用了生成选择率和成本统计信息所需的三种方法。您必须使用 Oracle 规定的确切方法名称，就像我这样。

参数数据类型和顺序也由可扩展优化器规定，但您可以选择自己的参数名称，只有一个显著的例外。您必须在类型方法中使用与最终将与统计信息类型关联的 PL/SQL 函数中相同的参数名称。在本例中，我为其构建此统计信息类型的 `FORMAT_PROD_CATEGORY` 函数只有一个名为 `p_prod_category` 的参数（因此我在相关方法中包含了它）。

类型主体实现了统计信息类型，可以包含您喜欢的任何逻辑，以使 CBO 能够确定关联的 PL/SQL 函数的选择率和成本。`PROD_STATS_OT.ODCIStatsSelectivity` 方法的类型主体在代码清单 9-55 中提供（其余方法可从 Apress 网站获取）。

**代码清单 9-55.** 为可扩展优化器创建统计信息类型（类型主体节选）

```sql
SQL> CREATE TYPE BODY prod_stats_ot AS
  2  
  3     STATIC FUNCTION ODCIGetInterfaces ...
<snip>
 13     STATIC FUNCTION ODCIStatsSelectivity (
 14                      p_pred_info         IN  SYS.ODCIPredInfo,
 15                      p_selectivity       OUT NUMBER,
 16                      p_args              IN  SYS.ODCIArgDescList,
 17                      p_start             IN  VARCHAR2,
 18                      p_stop              IN  VARCHAR2,
 19                      p_prod_category     IN  VARCHAR2,
 20                      p_env               IN  SYS.ODCIEnv
 21                      ) RETURN NUMBER IS
 22     BEGIN
 23  
 24        /* Calculate selectivity of predicate... */
 25        SELECT (COUNT(CASE
 26                         WHEN UPPER(prod_category) = p_start
 27                         THEN 0
 28                      END) / COUNT(*)) * 100 AS selectivity
 29        INTO   p_selectivity
 30        FROM   sh.products;
 31  
 32        RETURN ODCIConst.success;
 33     END ODCIStatsSelectivity;
 34
 35     STATIC FUNCTION ODCIStatsFunctionCost ...
<snip>
 76  
 77  END;
 78  /
```



正如其名称所示，`ODCIStatsSelectivity` 方法用于计算给定值集合对应 PL/SQL 函数谓词的选择性。它如何实现这一点才是有趣的部分。想象一下，我有这样一个谓词：`WHERE format_prod_category(p.prod_category) = 'SOFTWARE/OTHER'`。当 CBO 优化此谓词时，它会调用 `ODCIStatsSelectivity` 方法，并通过 `p_start` 和 `p_stop` 参数将值 'SOFTWARE/OTHER' 传递给统计方法。（如果您有一个范围谓词，`p_start` 和 `p_stop` 将分别包含下界和上界。）因此，这意味着我可以专门计算 'SOFTWARE/OTHER' 在 `PRODUCTS` 表中出现的次数来确定选择性，如上所述。

## 清单 9-56. 关联并使用统计类型以进行动态 PL/SQL 函数统计

一旦创建了统计类型，就可以将其与 PL/SQL 函数关联起来。清单 9-56 展示了关联统计类型的语法以及两个显示其动态特性的小查询。

```sql
SQL> ASSOCIATE STATISTICS WITH FUNCTIONS format_prod_category USING prod_stats_OT;

Statistics associated.

SQL> SELECT *
  2  FROM   products
  3  WHERE  format_prod_category(prod_category) = 'SOFTWARE/OTHER';

Execution Plan
----------------------------------------------------------
Plan hash value: 1954719464

-----------------------------------------------
| Id  | Operation         | Name     | Rows  |
-----------------------------------------------
|   0 | SELECT STATEMENT  |          |    26 |
|*  1 |  TABLE ACCESS FULL| PRODUCTS |    26 |
-----------------------------------------------

SQL> SELECT *
  2  FROM   products
  3  WHERE  format_prod_category(prod_category) = 'HARDWARE';

Execution Plan
----------------------------------------------------------
Plan hash value: 1954719464

-----------------------------------------------
| Id  | Operation         | Name     | Rows  |
-----------------------------------------------
|   0 | SELECT STATEMENT  |          |     2 |
|*  1 |  TABLE ACCESS FULL| PRODUCTS |     2 |
-----------------------------------------------
```

您可以看到，通过使用可扩展优化器，我为 CBO 提供了关于我的 PL/SQL 函数的准确且动态的统计信息（在此示例中，这些信息与 `PRODUCTS` 表的数据分布完全匹配）。请注意，`ODCI*` 方法在 SQL 优化阶段（即“硬解析”）被调用*一次*，并且也支持绑定变量（只要启用了绑定变量窥探）。然而，您应该考虑执行统计方法所需的时间，并确保它们带来的性能提升，或者您多次重用共享游标所带来的好处，足以弥补这部分开销。

![images](img/square.jpg) **注意** 如果您想了解 CBO 如何调用统计类型方法来提取选择性和成本，可以运行 10053 跟踪并查看生成的跟踪文件。

### 调优 PL/SQL

我已经描述了许多提高使用 PL/SQL 函数的查询性能的技术。然而，除非您打算消除或大幅减少 PL/SQL 函数调用，否则您还应该考虑对函数本身进行调优。

您有多种调优选项可供选择（尤其是在 Oracle 数据库的后期版本中），例如原生编译、子程序内联、新的整数数据类型、数组提取、关联数组缓存等等。对于关键查询和/或频繁执行的 PL/SQL 函数，您应该发现，通过使用大量可用的 PL/SQL 调优技术中的一些，可以减少它们的执行时间。

话虽如此，我在此仅演示一种涉及数组缓存的调优技术，因为它与我之前演示的 Result Cache 选项密切相关，并且可能产生显著效果。

## 用户定义的 PL/SQL 会话缓存

我之前提到过，在关联数组中缓存查找数据是当内置功能（跨会话 PL/SQL 函数结果缓存）不可用时的一种替代方案。有几种方法可以实现这一点，在清单 9-57 中，我提供了其中一种方法。我为汇率数据创建了一个 PL/SQL 数组缓存（使用私有的全局关联数组），并提供了一个单独的 `GET_RATES` 函数来加载和访问缓存的数据。

### 清单 9-57. 用于汇率的用户定义 PL/SQL 缓存

```sql
SQL> CREATE OR REPLACE PACKAGE BODY rates_pkg AS
  2  
  4     /* 缓存的索引子类型... */
  5     SUBTYPE key_st IS VARCHAR2(128);
  7  
  8     /* 汇率缓存... */
  9     TYPE rates_aat IS TABLE OF rates.exchange_rate%TYPE
 10        INDEX BY key_st;
 11     rates_cache rates_aat;
 13  
 14     /* 支持缓存的函数... */
 15     FUNCTION get_rate (
 16              p_rate_date IN rates.rate_date%TYPE,
 17              p_from_ccy  IN rates.base_ccy%TYPE,
 18              p_to_ccy    IN rates.target_ccy%TYPE
 19              ) RETURN rates.exchange_rate%TYPE IS
 21  
 22        v_rate rates.exchange_rate%TYPE;
 23        v_key  key_st := TO_CHAR(p_rate_date, 'YYYYMMDD')
 24                         || '~' || p_from_ccy || '~' || p_to_ccy;
 26  
 27     BEGIN
 29  
 30        IF rates_cache.EXISTS(v_key) THEN
 32  
 33           /* 缓存命中... */
 34           v_rate := rates_cache(v_key);
 36  
 37        ELSE
 39  
 40           /* 缓存未命中。获取并缓存... */
 41           SELECT exchange_rate INTO v_rate
 42           FROM   rates
 43           WHERE  rate_date  = p_rate_date
 44           AND    base_ccy   = p_from_ccy
 45           AND    target_ccy = p_to_ccy;
 46           rates_cache(v_key) := v_rate;
 48
 49        END IF;
 51
 52        RETURN v_rate;
 54
 55     END get_rate;
 57
 58  END rates_pkg;
 59  /
```

我省略了包规范（它只有 `GET_RATES` 函数签名），只列出了包体。为了使示例简短，我省略了处理 `NO_DATA_FOUND` 等情况所需的异常处理。关于该实现需要注意的其他一些要点如下：

> *第 7-9 行：* 我创建了一个私有的全局关联数组类型和变量来存储缓存的数据。
> 
> *第 19-20 行：* 数组的索引是 `RATES` 表主键的字符串表示形式。为了节省空间，我直接在函数的声明部分分配了这个键，但最佳实践是将所有赋值操作放在 PL/SQL 程序的可执行块中执行。
> 
> *第 24-27 行：* 我测试汇率是否已经缓存。如果是，我直接从缓存中返回它。
> 
> *第 32-37 行：* 如果汇率不在缓存中，我从 `RATES` 表中获取它并将其添加到缓存中。我决定只缓存必要的内容，以减少潜在的 PGA 内存占用（作为替代方案，您可能更愿意在实例化时将小型查找表预加载到关联数组中）。

## 清单 9-58. 使用用户定义的 PL/SQL 会话缓存函数的查询性能

在清单 9-58 中，我重复了在清单 9-40（演示 Result Cache 时）中使用的销售报告，但将对结果缓存的 `GET_RATES` 函数的调用替换为我的新用户定义缓存等效函数。



`SQL> SELECT t.calendar_year`
`  2  ,      p.prod_name`
`  3  ,      SUM(s.amount_sold)                               AS amt_sold_usd`
`  4  ,      SUM(s.amount_sold *`
`  5             rates_pkg.get_rate(s.time_id, 'USD', 'GBP')) AS amt_sold_gbp`
`  6  FROM   sales     s`
`  7  ,      products  p`
`  8  ,      times     t`
`  9  WHERE  s.prod_id = p.prod_id`
` 10  AND    s.time_id = t.time_id`
` 11  GROUP  BY`
` 12         t.calendar_year`
` 13  ,      p.prod_name`
` 14  ;`

`272 rows selected.`
`Elapsed: 00:00:11.93`

`Statistics`
`----------------------------------------------------------`
`       1591  recursive calls`
`          0  db block gets`
`       4729  consistent gets`
`          0  physical reads`
`          0  redo size`
`      16688  bytes sent via SQL*Net to client`
`        437  bytes received via SQL*Net from client`
`          2  SQL*Net roundtrips to/from client`
`          9  sorts (memory)`
`          0  sorts (disk)`
`        272  rows processed`

与结果缓存示例类似，用户定义的缓存显著降低了查询的逻辑 I/O 和运行时，使其降至可接受的水平。实际上，用户定义会话缓存的统计数据与结果缓存相似（缓存未命中次数相同），尽管用户缓存整体上稍慢一些。清单 9-59 展示了该查询的层级分析器报告，并说明了（相对于结果缓存示例）额外时间花费在何处。

***清单 9-59.** 使用用户定义的 PL/SQL 会话缓存函数的查询性能（层级分析器报告）*

```
FUNCTION                                  LINE#      CALLS SUB_ELA_US FUNC_ELA_US
------------------------------------ ---------- ---------- ---------- -----------
__plsql_vm                                    0     918848    7734487     2202214
RATES_PKG.GET_RATE                           12     918843    5531933     5434184
RATES_PKG.__static_sql_exec_line32           32       1460      97749       97749
__anonymous_block                             0          3        328         270
DBMS_OUTPUT.GET_LINES                       180          2         58          58
RATES_PKG.__pkg_init                          0          1         12          12
DBMS_HPROF.STOP_PROFILING                    59          1          0           0
```

从`rates`函数内部 SQL 的调用情况可以看出，有 1,460 次缓存未命中，这与之前的标量子查询缓存和结果缓存示例相同。这大大减少了对`RATES`表的查找次数。但是，请记住，结果缓存也消除了对`GET_RATES` PL/SQL 函数本身的调用。在这方面，用户定义的缓存效果较差，因为对`RATES_PKG.GET_RATES`函数的调用根本没有减少；事实上，函数调用占用了全部的额外运行时。

通过使用您自己的 PL/SQL 数组缓存（正如我所做的），您本质上是在优化函数，而不是减少或消除其使用。尽管如此，用户定义的 PL/SQL 会话缓存仍是一种有用的调优技术，用于降低嵌入 PL/SQL 函数中的 SQL 的成本。

![images](img/square.jpg) `注意` 如果您选择自己的 PL/SQL 数组缓存而非结果缓存，您应当意识到三个潜在的缺点。首先，每个数组缓存仅对单个会话可见，不会被共享。其次，每个会话都需要缓存其自己的数据副本，这会消耗私有的 PGA 内存（相反，结果缓存将其结果的单个共享副本存储在 SGA 中）。第三点，也是最关键的一点是，如果您缓存的是经常更新的表的数据，您将需要某种形式的缓存管理，而这将难以实现。结果缓存“开箱即用”地提供了此功能，但在您的 PL/SQL 程序中，您将没有这种便利。

### 总结

作为本章的结尾，我演示了在将 PL/SQL 函数设计到您的应用程序和查询中时应考虑的一系列成本。此外，我还演示了您可以用来消除、减少或优化 SQL 语句中 PL/SQL 函数使用的各种技术。

## 第 10 章

### 选择正确的游标

**作者：梅兰妮·卡弗里**

任何曾编写过执行循环逻辑的 PL/SQL 函数或过程的人都深知选择恰当游标类型的痛苦——或者说选择*错误*游标类型的痛苦。本章旨在教导您如何根据具体的编程情境选择合适的游标类型。选择错误的游标类型可能导致您的用户、同事或管理者（或所有人）对您满足业务需求的技术能力失去信心。选择错误的游标类型还可能导致花费大量时间调试生产环境中的系统减速，而您可能认为最糟糕的情况是收入减少。鉴于这些潜在的陷阱和后果，每位 PL/SQL 程序员都应努力为所要解决的每个具体技术问题选择最适合的游标类型。

本章重点介绍四种游标类型——并非因为它们是您唯一可用的四种游标类型，而是因为它们是大多数 PL/SQL 程序员通常实现的*最常见*类型。针对特定的业务需求正确实现它们，是拥有高性能和可扩展 PL/SQL 代码的关键。本章讨论的四种游标类型是

*   显式游标
*   隐式游标
*   静态 `REF` 游标
*   动态 `REF` 游标

您编写的任何用于程序化获取记录集以进行处理（无论是单个处理还是批量处理）的 PL/SQL 程序的目标，都是选择一种能让您和 Oracle 数据库以最少的工作量获得正确答案的游标类型。道理真的就这么简单。当然，在许多情况下，您*可以*使用多种类型的游标。但您*应该*吗？这是您每次编写 PL/SQL 程序时都需要问自己的问题。明智地选择。首先要弄清楚您试图回答的业务问题是什么，然后根据每个具体情况选择最佳的编程工具来快速、正确地回答它。



### 显式游标

毫无疑问，在任何 PL/SQL 程序中，最常用的游标类型是**显式游标**。每个人在初次学习 PL/SQL 语言时都会学习显式游标，大多数 PL/SQL 程序员会立刻被其吸引，因为他们觉得显式游标能提供更多对处理过程的程序化控制。程序员（至少是新手）都热衷于控制；他们的印象往往是，如果不能控制程序的每个方面，程序就无法正确执行。

显式游标常被称为 `open`、`fetch`、`close` 游标，因为它们需要使用这些关键字：`OPEN`、`FETCH` 和 `CLOSE`。如果你编写一个显式游标，就必须显式地打开游标、从游标中提取数据并关闭游标。到目前为止，这听起来还不算太糟，甚至可能让追求控制的程序员感到安心。然而，重要的是不仅要弄清楚在哪些情况下合理使用这类游标，还要明确在哪些情况下它们可能弊大于利。请看下面清单 10-1 中的显式游标示例：

***清单 10-1.** 用于仅获取一个值的显式游标*

```sql
CREATE FUNCTION f_get_name (ip_emp_id in number ) RETURN VARCHAR2
AS
CURSOR c IS SELECT ename FROM emp WHERE emp_id = f_get_name.ip_emp_id;
lv_ename emp.ename%TYPE;
BEGIN
OPEN c;
FETCH c INTO lv_ename;
CLOSE c;
RETURN lv_ename;
END;
```

![images](img/square.jpg) `注意` 本章中参数名和变量名的命名约定如下：`ip_` 表示 `输入参数`，`op_` 表示 `输出参数`，`lv_` 表示 `局部变量`，`gv_` 表示 `全局变量`。当在函数或过程中引用输入和输出参数时，会加上函数或过程名作为前缀，以避免混淆并说明作用域。在上面的示例中，即清单 10-1，`ip_emp_id` 被引用为 `f_get_name.ip_emp_id`。

乍一看，这个函数看起来就像任何典型的 `get` 函数（其唯一目的是获取一行甚至一个值）。业务需求显然是根据输入的员工 ID `emp_id`，从 `emp` 表中获取至少一个（至多一个）员工姓名 `ename`。游标被打开；从打开的游标中提取单个值并放入变量 `lv_ename`；游标被关闭；最后函数将存储在 `lv_ename` 变量中的值返回给调用程序。那么，为什么这种游标可能不适合这类业务需求呢？因为函数 `f_get_name` 是一个潜在的错误。

在提取操作中，如果未能提取到任何数据，`lv_ename` 中最终会是什么？你将无法知道是否收到了值。此外，如果数据有问题怎么办？如果对于你输入的 `ip_emp_id` 值，最终返回了多行数据怎么办？清单 10-2 展示了这个显式游标的一个更正确的版本：

***清单 10-2.** 用于仅获取一个值的更正确的显式游标 (11.2.0.1)*

```sql
CREATE FUNCTION f_get_name (ip_emp_id IN NUMBER) RETURN VARCHAR2
AS
CURSOR c IS SELECT ename FROM emp WHERE emp_id = f_get_name.ip_emp_id;
lv_ename emp.ename%TYPE;
BEGIN
   OPEN c;
   FETCH c INTO lv_ename;
      IF (SQL%NOTFOUND) THEN
         RAISE NO_DATA_FOUND;
      ENDIF;
   FETCH c INTO lv_ename;
      IF (SQL%FOUND) THEN
         RAISE TOO_MANY_ROWS;
      ENDIF;
   CLOSE c;
   RETURN lv_ename;
END;
```

然而，正如你所看到的，这变得非常复杂。这就是为什么在编写简单的 `get` 函数时，显式游标是一个糟糕的选择。很快就显而易见，你原本简单的函数突然变得不那么简单了。问题有两个方面：

*   许多人只做打开/提取/关闭操作就完事了。这是一个潜在的错误。
*   另一方面，如果你正确编码，你需要写的代码*远远*多于必要的代码。更多的代码 = 更多的错误。仅仅因为你可以使用显式游标对之前的业务问题得出正确答案，并不意味着你应该这样做。在“隐式游标”一节中，你将看到使用隐式游标解决相同业务问题的示例。

#### 显式游标的剖析

了解显式游标的工作方式可以帮助你决定它是否适合你的业务功能。前面提到显式游标具有语法：`open`、`fetch`、`close`。那么，这是什么意思？每一步实际发生了什么？下面概述了每一步执行的进程或程序功能：

1.  `OPEN`：此步骤初始化游标并标识结果集。注意，它实际上不必*组装*结果集。它只是设置结果集将“依据”的时间点。
2.  `FETCH`：此步骤重复执行，直到所有行都被检索出来（除非你使用 `BULK COLLECT`（稍后描述），它一次性提取所有记录）。
3.  `CLOSE`：此步骤在最后一行处理完毕后释放游标。

此外，如清单 10-1 所示，显式游标必须在从中提取数据之前*声明*。显式游标必须始终在过程或函数（或包，因为它可以在包规范或包体中全局声明）的声明部分中声明，然后才能实际调用。然而，使其成为*显式*的是你必须显式地打开/提取/关闭它。请注意，你可以声明一个游标然后隐式地使用它。声明游标并不会使其成为显式游标；关键在于你如何编写后续与之交互的代码。在“隐式游标”部分，你会看到使用隐式游标时，这个声明步骤不是必需的。隐式游标*可以*被声明，但与显式游标不同，它们*不必*被声明。

![images](img/square.jpg) `注意` `REF` 游标（稍后讨论）不是用实际的 `SELECT` 语句显式声明的。`SELECT` 语句只有在它们被显式打开时才会发挥作用。

使用游标涉及的细节很多，尤其是如果你碰巧有多个游标打开并处于提取记录的过程中。考虑到显式游标所经过的工作量（如前面三步概述所示），只在绝对必要时使用显式游标非常重要。那么，在哪些情况下可能需要显式游标呢？使用显式游标最明显的原因是当有要求批量处理记录时。



