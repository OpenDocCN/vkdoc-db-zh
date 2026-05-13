# PL/SQL 函数与统计信息

为什么代价优化器（CBO）难以处理 PL/SQL 函数？答案在于统计信息。你知道，CBO 依赖高质量的统计信息来做出合理的优化决策。然而，当涉及到 PL/SQL 函数时，CBO 几乎总是没有可用的统计信息，只能诉诸于默认的启发式方法。例如，当遇到 `WHERE plsql_function(expression-involving-some-column) = value` 这种形式的谓词时，CBO 会默认将该谓词的选择率设定为仅 1%（由于没有该函数的统计信息，它别无选择）。

在代码清单 9-17 中，我通过一个小表（包含 1000 行）、一个 PL/SQL 函数以及一个启用了 CBO 跟踪（10053 事件）的简单查询，来演示这种启发式方法。

## 代码清单 9-17. PL/SQL 函数谓词的默认选择率

```sql
SQL> CREATE TABLE thousand_rows
  2  AS
  3     SELECT ROWNUM           AS n1
  4     ,      RPAD('x',50,'x') AS v1
  5     FROM   dual
  6     CONNECT BY ROWNUM <= 1000;

表已创建。

SQL> BEGIN
  2     DBMS_STATS.GATHER_TABLE_STATS(user, 'THOUSAND_ROWS');
  3  END;
  4  /

PL/SQL 过程已成功完成。

SQL> CREATE FUNCTION plsql_function (
  2                  p_id IN INTEGER
  3                  ) RETURN INTEGER AS
  4  BEGIN
  5     RETURN 0;
  6  END plsql_function;
  7  /

函数已创建。

SQL> ALTER SESSION SET EVENTS '10053 trace name context forever, level 1';

会话已更改。

SQL> set autotrace traceonly explain

SQL> SELECT *
  2  FROM   thousand_rows
  3  WHERE  plsql_function(n1) = 0;

执行计划
----------------------------------------------------------
Plan hash value: 4012467810

-----------------------------------------------------------------------------------
| Id  | Operation         | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |               |    10 |   550 |     5   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| THOUSAND_ROWS |    10 |   550 |     5   (0)| 00:00:01 |
-----------------------------------------------------------------------------------

谓词信息 (通过操作标示符识别):
---------------------------------------------------

   1 - filter("PLSQL_FUNCTION"("N1")=0)

SQL> ALTER SESSION SET EVENTS '10053 trace name context off';

会话已更改。
```

由于缺乏 PL/SQL 函数的统计信息，CBO 默认使用 1% 的选择率来确定该谓词的基数。你可以从上面执行计划中显示的基数（行数）看出这一点。代码清单 9-18 中 10053 跟踪文件的节选也展示了 CBO 的计算过程。

## 代码清单 9-18. PL/SQL 函数谓词的默认选择率（10053 跟踪文件）

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
  `No statistics type defined for function PLSQL_FUNCTION`
  `No default cost defined for function PLSQL_FUNCTION`
  `No statistics type defined for function PLSQL_FUNCTION`
  `No default selectivity defined for function PLSQL_FUNCTION`
  Table: THOUSAND_ROWS  Alias: THOUSAND_ROWS
  Card: Original: 1000.000000  Rounded: 10  `Computed: 10.00`  Non Adjusted: 10.00
  Access Path: TableScan
    Cost:  5.10  Resp: 5.10  Degree: 0
      Cost_io: 5.00  Cost_cpu: 3285657
      Resp_io: 5.00  Resp_cpu: 3285657
  Best:: AccessPath: TableScan
```

我在这个跟踪文件中高亮了一些重要信息。它指出，对于 PL/SQL 函数的成本或选择率，没有定义默认的统计信息或统计信息类型。这向你表明，为 PL/SQL 函数提供统计信息是可能的；我将在后文演示如何做到这一点。




### 谓词顺序

当 PL/SQL 函数的统计信息缺失时，由于谓词编码顺序的原因，可能会引发其他潜在问题。例如，如果你编写了一个包含两个 PL/SQL 函数谓词的 WHERE 子句，在缺乏统计信息的情况下，函数将仅仅按照它们在 SQL 中出现的顺序执行。

我在代码清单 9-19 中展示了这一点，其中有一个 SQL 语句，它通过两个 PL/SQL 函数进行过滤：`SLOW_FUNCTION`和`QUICK_FUNCTION`。顾名思义，一个函数比另一个成本更高，因此我使用 Autotrace 和 SQL*Plus 计时器，以不同的谓词顺序执行了两次该 SQL 语句来比较结果。

***代码清单 9-19.** PL/SQL 函数与谓词顺序*

```sql
SQL> CREATE FUNCTION quick_function (
  2                  p_id IN INTEGER
  3                  ) RETURN INTEGER AS
  4  BEGIN
  5     RETURN MOD(p_id, 1000);
  6  END quick_function;
  7  /

Function created.

SQL> CREATE FUNCTION slow_function (
  2                  p_id IN INTEGER
  3                  ) RETURN INTEGER AS
  4  BEGIN
  5     DBMS_LOCK.SLEEP(0.005);
  6     RETURN MOD(p_id, 2);
  7  END slow_function;
  8  /

Function created.

SQL> set autotrace traceonly

SQL> SELECT *
  2  FROM   thousand_rows
  3  WHERE  quick_function(n1) = 0
  4  AND    slow_function(n1) = 0;

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

SQL> SELECT *
  2  FROM   thousand_rows
  3  WHERE  slow_function(n1) = 0
  4  AND    quick_function(n1) = 0;

1 row selected.

Elapsed: 00:00:11.09

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
   1 - filter("SLOW_FUNCTION"("N1")=0 AND "QUICK_FUNCTION"("N1")=0)
```

我移除了 Autotrace 统计信息，但保留了 Explain Plan 部分，以展示每个 SQL 语句中 PL/SQL 函数的执行顺序。你可以看到，这与它们被编码的顺序相匹配，并且这对两个语句的相对性能产生了显著影响。这就是关键所在。CBO 不知道`SLOW_FUNCTION`具有较低的选择性，或者它比`QUICK_FUNCTION`执行时间更长。在缺乏所提供的统计信息的情况下，它怎么可能知道呢？CBO 没有理由重新排序谓词，因为它没有这样做的依据。

好消息是，有几种方法可以为 PL/SQL 函数生成或设置统计信息来缓解此问题。这些方法在标题为*“协助 CBO.”*的章节中进行了描述。

#### 读一致性陷阱

代码清单 9-13 清楚地展示了将 SQL 查找放入 PL/SQL 函数中的资源成本（并且这些成本很容易量化）。然而，使用这种设计还存在另一个重要的陷阱，尽管它更难量化其对你的应用程序可能产生的潜在影响。那就是，PL/SQL 函数内部的 SQL 与调用该函数的主 SQL 语句不是读一致的。

来自代码清单 9-12 的 TKProf 报告（如代码清单 9-16 所示）证明，PL/SQL 函数内部的 SQL 被视为一个单独的语句。这意味着在 READ COMMITTED 隔离级别下，它有自己的读一致性，而不是调用它的 SQL 语句的读一致性。

在代码清单 9-20 到 9-23 中，我演示了这个读一致性陷阱。我有两个会话；第一个（*会话一*）执行一个长时间运行的查询，该查询包含对 PL/SQL 查找函数的调用。第二个会话（*会话二*）在随机时间点更新了一条查找记录，但重要的是，这是在第一个会话的查询仍在运行时发生的。

我将从代码清单 9-20 中的 PL/SQL 查找函数开始。

***代码清单 9-20.** 读一致性陷阱（创建 PL/SQL 函数）*

```sql
SQL> CREATE FUNCTION get_customer_name (
  2                  p_cust_id IN customers.cust_id%TYPE
  3                  ) RETURN VARCHAR2 AS
  4     v_customer_name VARCHAR2(200);
  5  BEGIN
  6     SELECT cust_first_name || ' ' || cust_last_name
  7     INTO   v_customer_name
  8     FROM   customers
  9     WHERE  cust_id = p_cust_id;
 10     RETURN v_customer_name;
 11  END get_customer_name;
 12  /

Function created.
```

你可以看到，该查找函数通过主键获取客户名称。此外，我创建了一个小的`SLEEPER`函数，以便减慢*会话一*的查询速度，模拟一个长时间运行的 SQL 语句（该函数只是调用`DBMS_LOCK.SLEEP`并返回常数 0）。

继续这个例子，*会话一* 发出了代码清单 9-21 中所示的查询。

***代码清单 9-21.** 读一致性陷阱（会话一的长时间运行查询）*

**`会话一`**
```sql
22:47:23 SQL> SELECT s.cust_id
22:47:23   2  ,      get_customer_name(s.cust_id) AS cust_name
22:47:23   3  ,      sleeper(1)                   AS sleep_time
22:47:23   4  FROM   sales s
22:47:23   5  WHERE  ROWNUM <= 10;
```

请注意*会话一*发出此 SQL 的时间。返回 10 行结果集且每行睡眠 1 秒意味着这个查询应该在 10 秒多一点的时间内完成。当它在执行时，*会话二*更新了*会话一*正在重复查询的一个客户名称，如代码清单 9-22 所示。

***代码清单 9-22.** 读一致性陷阱（会话二的客户更新）*

**`会话二`**
```sql
22:47:26 SQL> UPDATE customers
22:47:26   2  SET    cust_last_name = 'Smith'
22:47:26   3  WHERE  cust_id = 2;

1 row updated.

22:47:26 SQL> COMMIT;

Commit complete.
```

再次注意此更新被提交的时间（大约在*会话一*的长时间运行查询开始后 3 秒）。代码清单 9-23 显示了这对*会话一*报告的影响。

***代码清单 9-23.** 读一致性陷阱（会话一的结果集）*



`会话一`
-----------

`客户 ID 客户名称                      睡眠时间`
`---------- ------------------------------ ----------`
`         2 Anne Koch                               1`
`         2 Anne Koch                               1`
`         2 Anne Koch                               1`
`         2 Anne Koch                               1`
`         2 Anne Smith                              1`
`         2 Anne Smith                              1`
`         2 Anne Smith                              1`
`         2 Anne Smith                              1`
`         2 Anne Smith                              1`
`         2 Anne Smith                              1`

`已选择 10 行。`

`已用时间: 00:00:10.02`

大约在`会话一`的结果集进行到一半时，`ID=2`的客户名称发生了变化，这是因为查询函数在主查询的数据读一致性映像之外运行。因此，报告本身就是不一致的。

如果你使用此类`PL/SQL`函数，这意味着什么？显然，每次在查询中包含该函数时，你都面临结果集不一致的风险，特别是当你的用户可能随时更新查询表时。当然，你需要在自身应用模式的背景下评估这种风险。收到不一致报告的用户自然会质疑他们所看到的内容，甚至更糟，会质疑你的应用程序。

不幸的是，保护自己免受此影响的选项都不太令人满意。除了移除所有`PL/SQL`查询函数，你还有几种选择，例如：

*   使用`SERIALIZABLE`（可串行化）隔离级别。
*   使用`SET TRANSACTION READ ONLY`（设置事务为只读）。
*   让你的`PL/SQL`函数使用给定的`SCN`（系统变更号）进行闪回查询，以匹配主查询的开始。
*   让你的读取者通过`FOR UPDATE`查询来阻塞写入者（即使这样，也只有在你的主查询中其他地方直接访问了查询表时才有效。此时，你首先需要质疑是否需要你的`PL/SQL`查询函数！）

这些选项中的任何一个都能解决问题，但更有可能因为严重限制应用程序的并发性和性能而给你带来更大的麻烦。

### 其他问题

我已经比较详细地描述了在`SQL`中使用`PL/SQL`函数时需要注意的主要问题。不用说，还有其他问题，也许你已经遇到了一个我没想到的特定极端情况。我将在本节最后简要提一下你在某些阶段可能遇到的其他问题。

#### 并行查询

可以启用`PL/SQL`函数以配合 Oracle 数据库的并行查询选项（`PQ`）工作。虽然许多`PL/SQL`函数可以由`PQ`从属进程执行而不出问题，但有时引用未显式启用并行的`PL/SQL`函数可能会禁用`PQ`。如果你在并行化某个`SQL`语句时遇到麻烦，并且该语句包含`PL/SQL`函数，请检查该函数是否已启用并行。假设你的`PQ`提示、会话`PQ`设置和`PQ`系统参数对于并行查询是有效的，但它就是“不起作用”，那么请重新创建你的`PL/SQL`函数，包含`PARALLEL_ENABLE`子句，然后再试一次。请注意，要使`PL/SQL`函数支持并行，它不应引用`PQ`从属进程可能无法访问的会话状态变量（例如包状态）。

#### NO_DATA_FOUND（未找到数据）

我在前面的代码清单 9-12 的注释中已经描述过这一点，但为了完整性，值得重申。在从`SQL`语句调用的`PL/SQL`函数中引发的`NO_DATA_FOUND`异常**不会**传播该异常！相反，函数将返回`NULL`。

### 降低 PL/SQL 函数的成本

我刚刚用了几页的篇幅描述了在使用`PL/SQL`函数时可能遇到的许多困难、成本和陷阱。然而，这是一本面向`PL/SQL`专业人士的书，因此我将在本章的剩余部分描述一系列技术，以降低使用`PL/SQL`函数的影响。

#### 换个角度看问题

在我继续之前，我认为给出一个客观的视角很重要。在 Oracle 开发和性能的世界里，没有什么东西是绝对的，而且幸运的是，有时基本实现的`PL/SQL`函数就绰绰有余了。代码清单 9-24 和 9-25 展示了这样一个例子。我将创建一个`isNumber`检查器来验证传入的原始数据，并比较`PL/SQL`函数实现与纯`SQL`版本的实现。为此，我创建了一个包含七个字段的外部表（全部为`VARCHAR2(30)`类型），将`SALES`表中的数据转储到一个平面文件，并将每 10,000 个`AMOUNT_SOLD`值中的一个修改为包含非数字字符（`AMOUNT_SOLD`映射到外部表中的`FIELD_07`）。最后，我创建了如代码清单 9-24 所示的`IS_NUMBER`函数，用于验证字符串是否确实为数字格式。

**代码清单 9-24.** 验证数字数据（`IS_NUMBER`函数）

```sql
SQL> CREATE FUNCTION is_number (
  2                    p_str IN VARCHAR2
  3                    ) RETURN NUMBER IS
  4       n NUMBER;
  5    BEGIN
  6       n := TO_NUMBER(p_str);
  7       RETURN 1;
  8    EXCEPTION
  9       WHEN VALUE_ERROR THEN
 10          RETURN 0;
 11    END;
 12    /

函数已创建。
```

在代码清单 9-25 中，我使用了两种替代方法来验证`FIELD_07`中的数据是否为数字格式。首先，我使用了`IS_NUMBER` `PL/SQL`函数（如果输入值是数字，则返回 1）。其次，我使用了`SQL`内置的正则表达式，该示例取自`OTN`（Oracle 技术网）论坛的一个帖子。我使用`Autotrace`来抑制输出，并使用`SQL*Plus`计时器来记录时间。

**代码清单 9-25.** 验证数字数据（`PL/SQL`函数 vs. `SQL`内置函数）

```sql
SQL> SELECT *
  2  FROM   external_table
  3  WHERE  is_number(field_07) = 1;

已选择 918752 行。

已用时间: 00:00:10.19

SQL> SELECT *
  2  FROM   external_table
  3  WHERE  REGEXP_LIKE(
  4             field_07,
  5             '^( *)(\+|-)?((\d*[.]?\d+)|(d+[.]?\d*)){1}(e(\+|-)?\d+)?(f|d)?$',
  6             'i');

已选择 918752 行。

已用时间: 00:00:28.25
```

你可以看到`PL/SQL`函数的实现比正则表达式快得多（几乎快三倍）。当反过来查找少量非数字记录时，效果仍然更好。正则表达式非常消耗`CPU`；比较`V$SQL`中两个`SQL`语句的`CPU`时间，或者使用`DBMS_UTILITY.GET_CPU_TIME`跟踪这两个查询，你就能亲眼看到差异。

这个例子表明，在`SQL`语句中使用`PL/SQL`函数的有效性不能一概而论地否定。你的需求、你对所开发系统的了解以及你的测试将帮助你判断一个`PL/SQL`函数是否可行。如果可行，那么我在本章剩余部分描述的技术应该能帮助你从实现中获得更多收益。




#### 使用 SQL 替代方案

然而，PL/SQL 函数比纯 SQL 表达式或内置函数更快的情况相当罕见。你应该考虑是否可能或值得将应用程序中一些关键的（使用了 PL/SQL 函数的）查询更改为使用纯 SQL 替代方案。如果你正在开发新应用程序，那么你有一个很好的机会从一开始就内置高性能。

当然，这里需要考虑一些因素，如果以下任何一项为真，你可能会决定倾向于使用某些 PL/SQL 函数而不是 SQL 实现：

*   一个 PL/SQL 函数并不被频繁使用。
*   一个 PL/SQL 函数过于复杂，难以用 SQL 轻松表达。
*   一个 SQL 语句并未展现出我描述的任何问题。
*   你需要修改的 SQL 量太大，或者回归测试的成本过高。

我将演示使用 SQL 带来的性能优势，并提供一些你可以用来重构那些关键 SQL 语句（甚至可以考虑作为新应用程序的标准）的替代技术。

##### 使用 SQL

在清单 9-12 中，我运行了一个销售报告，该报告导致 `GET_RATE` PL/SQL 函数被执行了超过 918,000 次。我演示了同一报告的纯 SQL 实现（即与 `RATES` 表的一个简单连接）将该查询的运行时间从 66 秒减少到了仅 1 秒（逻辑 I/O 也大幅减少）。纯 SQL 技术对于使用更快的 PL/SQL 函数（例如那些不包含任何 SQL 的函数）的查询的运行时间也能产生影响。

SQL 是一种极其灵活且功能丰富的语言，用于快速处理数据集。假设性能是你的目标，你可以使用诸如分析函数、子查询分解以及内置函数的广泛运用等 SQL 技术来将你的 PL/SQL 函数重构为 SQL（有关 SQL 创新用法的示例，请访问 OTN SQL 和 PL/SQL 论坛或 [`http://oraqa.com/`](http://oraqa.com/)）。

##### 使用视图

支持保留 PL/SQL 函数的一个论点是，它们封装并集中了应用程序逻辑，防止相同的规则在代码库中扩散。也存在其他的封装方法，例如使用视图，它额外提供了 SQL 性能的优势。在清单 9-26 中，我通过将按日历年和产品的销售报告从使用 `GET_RATE` PL/SQL 函数改为使用视图进行了转换，其中逻辑被封装在 SQL 表达式中。

**清单 9-26.** 在视图中封装函数逻辑

```sql
SQL> CREATE VIEW sales_rates_view
  2  AS
  3     SELECT s.*
  4     ,      s.amount_sold * (SELECT r.exchange_rate
  5                             FROM   rates r
  6                             WHERE  r.base_ccy   = 'USD'
  7                             AND    r.target_ccy = 'GBP'
  8                             AND    r.rate_date  = s.time_id) AS amount_sold_gbp
  9     FROM   sales     s;

视图已创建。
```

通过这个视图，我将 `GET_RATE` PL/SQL 函数转换成了一个标量子查询查找。整个汇率转换现在被投影为单个列。我使用了标量子查询而不是与 `RATES` 的简单连接，以利用视图的一个称为“列消除”的优化。有了这个优化，标量子查询只会在查询中引用 `AMOUNT_SOLD_GBP` 列时才会执行。这意味着不需要转换后汇率的查询也可以利用这个视图而没有性能损失，从而扩大了其重用范围。使用此视图时销售报告的计时和统计信息如清单 9-27 所示。

**清单 9-27.** 利用视图中的列消除（未使用列消除）

```sql
SQL> SELECT t.calendar_year
  2  ,      p.prod_name
  3  ,      SUM(s.quantity_sold)   AS qty_sold
  4  ,      SUM(s.amount_sold)     AS amt_sold_usd
  5  ,      SUM(s.amount_sold_gbp) AS amt_sold_gbp
  6  FROM   sales_rates_view s
  7  ,      times            t
  8  ,      products         p
  9  WHERE  s.time_id    = t.time_id
 10  AND    s.prod_id    = p.prod_id
 11  GROUP  BY
         t.calendar_year
 12  ,      p.prod_name;

已选择 272 行。

已用时间: 00:00:01.29

执行计划
----------------------------------------------------------
Plan hash value: 625253124

------------------------------------------------------------
| Id  | Operation                     | Name     | Rows  |
------------------------------------------------------------
|   0 | SELECT STATEMENT              |          |   252 |
|*  1 |  INDEX UNIQUE SCAN            | RATES_PK |     1 |
|   2 |  HASH GROUP BY                |          |   252 |
|*  3 |   HASH JOIN                   |          |   918K|
|   4 |    PART JOIN FILTER CREATE    | :BF0000  |  1826 |
|   5 |     TABLE ACCESS FULL         | TIMES    |  1826 |
|*  6 |    HASH JOIN                  |          |   918K|
|   7 |     TABLE ACCESS FULL         | PRODUCTS |    72 |
|   8 |     PARTITION RANGE JOIN-FILTER|          |   918K|
|   9 |      TABLE ACCESS FULL        | SALES    |   918K|
------------------------------------------------------------

统计信息
----------------------------------------------------------
          8  递归调用
          0  数据库块读取
      10639  一致性读取
          0  物理读取
          0  重做大小
      17832  通过 SQL*Net 发送给客户端的字节数
        437  通过 SQL*Net 从客户端接收的字节数
          2  SQL*Net 往返次数
          0  内存排序
          0  磁盘排序
        272  处理的行数
```


### 性能对比与列消除

我已高亮了输出的关键部分。与简单地连接到 `RATES` 表相比，这里的逻辑 I/O 略有增加，但与 PL/SQL 函数产生的 I/O 相比，这微不足道。销售报告仍然比使用 PL/SQL 函数的版本快得多，并且业务逻辑已被封装。

清单 9-28 演示了列消除。我已从报告中排除了 `AMOUNT_SOLD_GBP` 列，并再次捕获了统计信息。

### 利用视图中的列消除

```sql
SQL> SELECT t.calendar_year
  2  ,      p.prod_name
  3  ,      SUM(s.quantity_sold)   AS qty_sold
  4  ,      SUM(s.amount_sold)     AS amt_sold_usd
  5  FROM   sales_rates_view s
  6  ,      times            t
  7  ,      products         p
  8  WHERE  s.time_id    = t.time_id
  9  AND    s.prod_id    = p.prod_id
 10  GROUP  BY
         t.calendar_year
 11  ,      p.prod_name;

272 rows selected.
```

`Elapsed: 00:00:00.61`

```sql
Statistics
----------------------------------------------------------
         8  recursive calls
         0  db block gets
      1736  consistent gets
         0  physical reads
         0  redo size
     11785  bytes sent via SQL*Net to client
       437  bytes received via SQL*Net from client
         2  SQL*Net roundtrips to/from client
         0  sorts (memory)
         0  sorts (disk)
       272  rows processed
```

这次，查询的完成时间缩短了一半，逻辑 I/O 再次减少。这是因为在运行时，视图投影中已消除了对 `RATES` 表的查找。清单 9-29 中的执行计划和投影清晰地证明了这一点。

### 利用视图中的列消除（执行计划）

```sql
PLAN_TABLE_OUTPUT
------------------------------------------------------------------------
Plan hash value: 171093611

-----------------------------------------------------
| Id  | Operation                        | Name     |
-----------------------------------------------------
|   0 | SELECT STATEMENT                 |          |
|   1 |  HASH GROUP BY                   |          |
|   2 |   HASH JOIN                      |          |
|   3 |    TABLE ACCESS FULL             | PRODUCTS |
|   4 |    VIEW                          | VW_GBC_9 |
|   5 |     HASH GROUP BY                |          |
|   6 |      HASH JOIN                   |          |
|   7 |       PART JOIN FILTER CREATE    | :BF0000  |
|   8 |        TABLE ACCESS FULL         | TIMES    |
|   9 |       PARTITION RANGE JOIN-FILTER|          |
|  10 |        TABLE ACCESS FULL         | SALES    |
-----------------------------------------------------

Column Projection Information (identified by operation id):
-----------------------------------------------------------

   1 - "ITEM_4"[NUMBER,22], "P"."PROD_NAME"[VARCHAR2,50],
       SUM("ITEM_2")[22], SUM("ITEM_3")[22]
   2 - (#keys=1) "P"."PROD_NAME"[VARCHAR2,50], "ITEM_4"[NUMBER,22],
       "ITEM_2"[NUMBER,22], "ITEM_3"[NUMBER,22]
   3 - "P"."PROD_ID"[NUMBER,22], "P"."PROD_NAME"[VARCHAR2,50]
   4 - "ITEM_1"[NUMBER,22], "ITEM_2"[NUMBER,22], "ITEM_3"[NUMBER,22],
       "ITEM_4"[NUMBER,22]
   5 - "T"."CALENDAR_YEAR"[NUMBER,22], "S"."PROD_ID"[NUMBER,22],
       SUM("S"."AMOUNT_SOLD")[22], SUM("S"."QUANTITY_SOLD")[22]
   6 - (#keys=1) "T"."CALENDAR_YEAR"[NUMBER,22],
       "S"."PROD_ID"[NUMBER,22], "S"."AMOUNT_SOLD"[NUMBER,22],
       "S"."QUANTITY_SOLD"[NUMBER,22]
   7 - "T"."TIME_ID"[DATE,7], "T"."TIME_ID"[DATE,7],
       "T"."CALENDAR_YEAR"[NUMBER,22]
   8 - "T"."TIME_ID"[DATE,7], "T"."CALENDAR_YEAR"[NUMBER,22]
   9 - "S"."PROD_ID"[NUMBER,22], "S"."TIME_ID"[DATE,7],
       "S"."QUANTITY_SOLD"[NUMBER,22], "S"."AMOUNT_SOLD"[NUMBER,22]
  10 - "S"."PROD_ID"[NUMBER,22], "S"."TIME_ID"[DATE,7],
       "S"."QUANTITY_SOLD"[NUMBER,22], "S"."AMOUNT_SOLD"[NUMBER,22]
```

这突显了一个事实：视图可以是 PL/SQL 函数的一种更高性能的替代方案，而不会损害封装和集中化的需求。作为最后的考虑，你可以重命名基表，并允许视图采用其名称。通过这些操作，你将业务逻辑列“插入”到了 `SALES` “表”中，并且你的 DML 语句不会受到影响，如 清单 9-30 所示。

### 通过带有标量子查询的视图对表执行 DML

```sql
SQL> RENAME sales TO sales_t;

Table renamed.

SQL> CREATE VIEW sales
  2  AS
     SELECT s.prod_id, s.cust_id, s.time_id, s.channel_id,
            s.promo_id, s.quantity_sold, s.amount_sold,
            s.amount_sold * (SELECT r.exchange_rate
                             FROM   rates r
                             WHERE  r.base_ccy   = 'USD'
                             AND    r.target_ccy = 'GBP'
                             AND    r.rate_date  = s.time_id) AS amount_sold_gbp
     FROM   sales_t  s;

View created.

SQL> INSERT INTO sales
     ( prod_id, cust_id, time_id, channel_id, promo_id,
       quantity_sold, amount_sold )
  2  VALUES
     ( 13, 987, DATE '1998-01-10', 3, 999, 1, 1232.16 );

1 row created.

SQL> UPDATE sales
  2  SET    amount_sold = 10000
  3  WHERE  ROWNUM = 1;

1 row updated.

SQL> DELETE
  2  FROM   sales
  3  WHERE  amount_sold = 10000;

1 row deleted.
```

要使用此方法，需要考虑任何现有的代码库，例如 `%ROWTYPE` 声明、基于 PL/SQL 记录的 DML、未限定的 `INSERT` 或 `SELECT` 列表（即那些没有显式列引用的列表）等。幸运的是，视图中的派生列无法被修改，正如本节最后一个示例在 清单 9-31 中演示的那样。

### 通过带有标量子查询的视图对表执行 DML

```sql
SQL> INSERT INTO sales
     ( prod_id, cust_id, time_id, channel_id, promo_id,
       quantity_sold, amount_sold, amount_sold_gbp )
  2  VALUES
     ( 13, 987, DATE '1998-01-10', 3, 999, 1, 1232.16, 1000 );
     quantity_sold, amount_sold, amount_sold_gbp )
                         *
ERROR at line 3:
ORA-01733: virtual column not allowed here
```


##### 使用虚拟列

为了总结本节关于使用 SQL 表达式替代 PL/SQL 函数的内容，我将演示如何用虚拟列替换 PL/SQL 函数。

虚拟列是 Oracle Database 11g 的一项功能，它们是作为列元数据存储在表中的表达式（本身不占用任何存储空间）。它们在逻辑上类似于视图中的列，但灵活性要高得多，因为可以为其收集统计信息或创建索引。虚拟列是封装简单业务规则的绝佳工具，而且由于它们作为元数据存储在表旁，因此也是数据逻辑的自我文档化声明。

为了演示虚拟列作为 PL/SQL 函数的替代方案，我转换了清单 9-6 中的销售报告。回顾一下，该清单使用了 `FORMAT_CUSTOMER_NAME` 函数来格式化客户名称，执行时间接近 7 秒（而纯 SQL 查询只需 1 秒）。为了开始转换，我向 `CUSTOMERS` 表添加了一个虚拟列，如清单 9-32 所示。

**清单 9-32.** 将 PL/SQL 函数转换为虚拟列（语法）

```
SQL> ALTER TABLE customers ADD
  2  ( cust_name VARCHAR2(100) GENERATED ALWAYS AS (cust_first_name||' '||cust_last_name) )
  3  ;

Table altered.
```

`GENERATED ALWAYS AS (expression)` 语法是虚拟列特有的（我省略了可选的 `VIRTUAL` 关键字以使显示保持在一行）。`CUSTOMERS` 表现在如清单 9-33 所示。

**清单 9-33.** 将 PL/SQL 函数转换为虚拟列（表描述）

```
SQL> SELECT column_name
  2  ,      data_type
  3  ,      data_default
  4  FROM   user_tab_columns
  5  WHERE  table_name = 'CUSTOMERS'
  6  ORDER  BY
  7         column_id;

COLUMN_NAME                    DATA_TYPE       DATA_DEFAULT
------------------------------ --------------- -----------------------------------------
CUST_ID                        NUMBER
CUST_FIRST_NAME                VARCHAR2
CUST_LAST_NAME                 VARCHAR2
CUST_GENDER                    CHAR
<...output removed...>
CUST_VALID                     VARCHAR2
CUST_NAME                      VARCHAR2        "CUST_FIRST_NAME"||' '||"CUST_LAST_NAME"
```

在 SQL 语句中使用虚拟列与使用任何其他表、视图或其他列没有区别。转换后的销售报告如清单 9-34 所示。

**清单 9-34.** 将 PL/SQL 函数转换为虚拟列（用法）

```
SQL> SELECT t.calendar_year
  2  ,      c.cust_name
  3  ,      SUM(s.quantity_sold) AS qty_sold
  4  ,      SUM(s.amount_sold)   AS amt_sold
  5  FROM   sales     s
  6  ,      customers c
  7  ,      times     t
  8  WHERE  s.cust_id = c.cust_id
  9  AND    s.time_id = t.time_id
 10  GROUP  BY
 11         t.calendar_year
 12  ,      c.cust_name;

11604 rows selected.

Elapsed: 00:00:01.20
```

从这个查询的执行时间可以看出，通过将客户名称格式化移至虚拟列，我消除了 PL/SQL 函数的开销，同时将业务逻辑的封装性与它所应用的实际数据保留在了一起。

#### 减少执行次数

在本章前面，我描述了无法可靠预测一条 SQL 语句将调用 PL/SQL 函数的次数这一事实。此外，每次执行除了 PL/SQL 函数本身要完成的工作外，还会带来上下文切换的开销。不过，有一些技术可以用来可靠地*减少*函数调用，接下来我将演示其中一些。

##### 使用 SQL 提示进行预/后计算

我在清单 9-8 到清单 9-10 中给出了一个 CBO（基于成本的优化器）进行查询转换的例子，在该例中，我试图在内联视图中预计算格式化的客户名称，但因视图合并而被否定了。结果是 918,000 次 PL/SQL 函数执行（即在较大的 `SALES` 表中每行执行一次），而不是预期的 55,500 次调用（即在 `CUSTOMERS` 表中每行执行一次）。对于某些查询，这种数量级甚至更多的函数执行可能是灾难性的。

我在清单 9-35 中重复了客户销售报告。然而，这次我使用了几个提示来确保在内联视图中对客户名称的预计算能够“坚持”，并且 CBO 不会合并查询块。

**清单 9-35.** 通过 SQL 减少函数调用

```
SQL> SELECT /*+ NO_MERGE(@customers) */
  2         t.calendar_year
  3  ,      c.cust_name
  4  ,      SUM(s.quantity_sold) AS qty_sold
  5  ,      SUM(s.amount_sold)   AS amt_sold
  6  FROM   sales     s
  7  ,     (
  8         SELECT /*+ QB_NAME(customers) */
  9                cust_id
 10         ,      format_customer_name (
 11                   cust_first_name, cust_last_name
 12                   ) AS cust_name
 13         FROM   customers
 14        )          c
 15  ,      times     t
 16  WHERE  s.cust_id = c.cust_id
 17  AND    s.time_id = t.time_id
 18  GROUP  BY
 19         t.calendar_year
 20  ,      c.cust_name
 21  ;

11604 rows selected.

Elapsed: 00:00:01.49

SQL> exec counter.show('Function calls');

Function calls: 55500
```

你可以看到，这次我使用了提示来指示 CBO 不要合并我的内联视图。目的是确保我的 PL/SQL 函数只在 `CUSTOMERS` 数据集上执行，而不是在更大的 `SALES` 集上执行。我通过使用 `NO_MERGE` 提示，并采用 Oracle Database 10g 引入的查询块命名语法来实现了这一点。`QB_NAME` 提示用于标记内联视图查询块；因此，我可以从主查询块引用该内联视图，正如你在 `NO_MERGE` 提示中看到的那样。

然而，对于这个查询，我还可以更进一步。在这个特定例子中，`FORMAT_CUSTOMER_NAME` 函数只需要应用到最终的 11,604 行结果集上（远少于 55,500 条客户记录）。因此，我可以使用与之前相同的提示对整个结果集进行预分组，但只在最后阶段才调用 PL/SQL 函数。清单 9-36 展示了这一做法的影响。

**清单 9-36.** 通过 SQL 减少函数调用（重构后的查询）

```
SQL> SELECT /*+ NO_MERGE(@pregroup) */
  2         calendar_year
  3  ,      format_customer_name(
  4            cust_first_name, cust_last_name
  5            ) AS cust_name
  6  ,      qty_sold
  7  ,      amt_sold
  8  FROM  (
  9           SELECT /*+ QB_NAME(pregroup) */
 10                  t.calendar_year
 11           ,      c.cust_first_name
 12           ,      c.cust_last_name
 13           ,      SUM(s.quantity_sold) AS qty_sold
 14           ,      SUM(s.amount_sold)   AS amt_sold
 15           FROM   sales     s
 16           ,      customers c
 17           ,      times     t
 18           WHERE  s.cust_id = c.cust_id
 19           AND    s.time_id = t.time_id
 20           GROUP  BY
 21                  t.calendar_year
 22           ,      c.cust_first_name
 23           ,      c.cust_last_name
 24        );
11604 rows selected.

Elapsed: 00:00:01.14

SQL> exec counter.show('Function calls');

Function calls: 11604
```

这效果更好，因为我设法在调用 PL/SQL 函数之前对所有数据进行了预聚合，将其调用次数减少到 11,604 次，并在报告上节省了更多时间。


这项技术可以应用于那些必须尽可能减少`PL/SQL`函数调用的特定场景。例如，通过对之前带有`GET_RATE`函数的销售报告采用此技术，我成功地将 918,000 次函数执行减少到仅 36,000 次，从而使总体响应时间从 66 秒大幅缩短至 4 秒。

然而，一种更策略性的减少函数调用的方法是利用缓存方案，我将在下文进行描述。

### 通过缓存减少函数调用

根据您使用的 Oracle 数据库版本，有几种缓存数据的选项。我将简要描述两种您可能希望研究并/或应用到您应用程序中的、用于减少函数调用的缓存技术。它们是：

*   标量子查询缓存
*   跨会话`PL/SQL`函数结果缓存

### 标量子查询缓存

此缓存功能是一项旨在减少嵌入在标量子查询中的`SQL`语句或`PL/SQL`函数执行次数的内部优化。尽管无法可靠预测此内部缓存的效率，但它可以在一定程度上用于减少`PL/SQL`函数调用。在清单 9-37 中，我将包含`GET_RATES`查找函数的销售报告进行了转换，以利用标量子查询缓存。

**清单 9-37.** 使用标量子查询缓存减少`PL/SQL`函数调用

```sql
SQL> SELECT t.calendar_year
  2  ,      p.prod_name
  3  ,      SUM(s.amount_sold)                                             AS amt_sold_usd
  4  ,      SUM(s.amount_sold * (SELECT get_rate(s.time_id, 'USD', 'GBP')
  5                                FROM   dual))                           AS amt_sold_gbp
  6  FROM   sales     s
  7  ,      products  p
  8  ,      times     t
  9  WHERE  s.prod_id = p.prod_id
 10  AND    s.time_id = t.time_id
 11  GROUP  BY
 12         t.calendar_year
 13  ,      p.prod_name
 14  ;

272 rows selected.

Elapsed: 00:00:02.54

Statistics
----------------------------------------------------------
      19451  recursive calls
          0  db block gets
      40634  consistent gets
          0  physical reads
          0  redo size
      16688  bytes sent via SQL*Net to client
        437  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
        272  rows processed

SQL> exec counter.show('Function calls');

Function calls: 19450
```

通过将对`GET_RATE`的调用封装在标量子查询中，我成功地将查询的耗时从 66 秒减少到不到 3 秒。计数器显示`PL/SQL`函数调用次数已降至仅 19,450 次（从超过 918,000 次），从而在时间和逻辑 I/O 上实现了巨大的节省（缓存阻止了超过 180 万次的一致性读取）。

如前所述，缓存的效率取决于多种因素，包括输入数据的顺序。考虑到这一点，我知道我的`SALES`数据中`GET_RATE`函数的输入值范围相当小，因此在清单 9-38 中，我尝试通过排序输入到标量子查询的数据来进一步减少函数调用。

**清单 9-38.** 使用标量子查询缓存减少`PL/SQL`函数调用（排序数据的效果）

```sql
SQL> SELECT /*+ NO_MERGE(@inner) */
  2         calendar_year
  3  ,      prod_name
  4  ,      SUM(amount_sold)                                              AS amt_sold_usd
  5  ,      SUM(amount_sold * (SELECT get_rate(time_id, 'USD', 'GBP')
  6                              FROM   dual))                            AS amt_sold_gbp
  7  FROM  (
  8         SELECT /*+ QB_NAME(inner) NO_ELIMINATE_OBY */
  9                t.calendar_year
 10         ,      s.time_id
 11         ,      p.prod_name
 12         ,      s.amount_sold
 13         FROM   sales     s
 14         ,      products  p
 15         ,      times     t
 16         WHERE  s.prod_id = p.prod_id
 17         AND    s.time_id = t.time_id
 18         ORDER  BY
 19                s.time_id
 20        )
 21  GROUP  BY
 22         calendar_year
 23  ,      prod_name
 24  ;

272 rows selected.

Elapsed: 00:00:02.14

Statistics
----------------------------------------------------------
       1461  recursive calls
          0  db block gets
       4662  consistent gets
          8  physical reads
          0  redo size
      16688  bytes sent via SQL*Net to client
        437  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          2  sorts (memory)
          0  sorts (disk)
        272  rows processed

SQL> exec counter.show('Function calls: ordered inputs');

Function calls: ordered inputs: 1460
```

通过排序输入到标量子查询缓存的数据，我将`PL/SQL`函数调用次数进一步减少到仅 1,460 次（从而将逻辑 I/O 降低了一个数量级，并略微缩短了总体耗时）。这表明，对输入到子查询缓存的调用进行排序（或聚类）可以影响其效率。在我的案例中，通过进一步减少`PL/SQL`函数调用所实现的节省，“抵消”了预排序`SALES`数据的成本。



