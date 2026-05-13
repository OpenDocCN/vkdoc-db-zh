# 跨会话 PL/SQL 函数结果缓存

跨会话 PL/SQL 函数结果缓存是在 Oracle 数据库 11*g*中引入的。结果缓存的原理很简单。首先，一个 PL/SQL 函数被标记为需要缓存（使用`RESULT_CACHE`指令）。之后，每次使用新参数调用该函数时，Oracle 数据库会执行该函数，将返回值添加到结果缓存中，然后将结果返回给调用上下文。如果调用重复发生，Oracle 数据库会从缓存中检索结果，而不是重新执行函数。在某些情况下，这种缓存行为可以带来显著的性能提升。它的另一个好处是几乎不需要重构代码（实际上只需要对函数做很小的改动）。

我将通过`GET_RATE`函数来演示结果缓存的效果。首先，我需要用缓存指令重新编译该函数，如清单 9-39 所示。

**清单 9-39. 为 PL/SQL 函数结果缓存准备函数**

```sql
SQL> CREATE OR REPLACE FUNCTION get_rate(
  2                     p_rate_date IN rates.rate_date%TYPE,
  3                     p_from_ccy  IN rates.base_ccy%TYPE,
  4                     p_to_ccy    IN rates.target_ccy%TYPE
  5                     ) RETURN rates.exchange_rate%TYPE
6 RESULT_CACHE RELIES_ON (rates) AS
  7     v_rate rates.exchange_rate%TYPE;
  8  BEGIN
     <...省略...>
 15  END get_rate;
 16  /

函数已创建。
```

如上所示，我添加了两个专门用于结果缓存的语法要素。`RESULT_CACHE`关键字不言自明，它告诉 Oracle 数据库为该函数启用结果缓存。`RELIES_ON (rates)`语法仅在 Oracle 数据库 11*g*第 1 版中需要，它声明了该函数依赖于`RATES`表中的数据（Oracle 数据库 11*g*第 2 版会自动识别此类依赖关系）。因此，任何针对`RATES`的事务都会导致 Oracle 数据库使缓存的结果失效，并暂停缓存该函数，直到事务完成（此时缓存将开始新的刷新周期）。

既然我已经为这个函数启用了缓存，我可以重新执行最初的销售查询了，如清单 9-40 所示。

**清单 9-40. 使用结果缓存的 PL/SQL 函数的查询性能**

```sql
SQL> SELECT t.calendar_year
  2  ,      p.prod_name
  3  ,      SUM(s.amount_sold)                     AS amt_sold_usd
  4  ,      SUM(s.amount_sold *
  5              get_rate(s.time_id, 'USD', 'GBP')) AS amt_sold_gbp
  6  FROM   sales     s
  7  ,      products  p
  8  ,      times     t
  9  WHERE  s.prod_id = p.prod_id
 10  AND    s.time_id = t.time_id
 11  GROUP  BY
         t.calendar_year
 12  ,      p.prod_name
 13  ;

已选择 272 行。

已用时间: 00:00:08.74

统计信息
----------------------------------------------------------
       1481  递归调用
          0  数据库块获取
       4683  一致性读取
          0  物理读取
          0  重做大小
      16688  通过 SQL*Net 发送给客户端的字节数
        437  通过 SQL*Net 从客户端接收的字节数
          2  SQL*Net 往返通信次数
          0  内存排序
          0  磁盘排序
        272  处理的行数
```

通过缓存函数结果，该查询的执行时间减少了 57 秒。这虽然不如纯 SQL 实现（耗时 1 秒）或标量子查询版本（耗时 2 秒）那么快，但考虑到无需重写 SQL 即可实现，且行为可预测，你可能会认为这是一个可接受的性能提升。

原查询的另一个问题是封装的`RATES`查找所产生的 I/O。使用结果缓存后，逻辑 I/O 已降至更可接受的水平。

为了量化函数执行次数的减少，我使用 PL/SQL 分层分析器跟踪了该查询。分析报告见清单 9-41。

**清单 9-41. 使用结果缓存 PL/SQL 函数查询的 PL/SQL 分层分析器会话报告**

```sql
FUNCTION                                      LINE#      CALLS SUB_ELA_US FUNC_ELA_US
------------------------------------ ---------- ---------- ---------- -----------
__plsql_vm                                        0     918846    3777356     3651587
GET_RATE                                          1       1460     125433       25731
GET_RATE.__static_sql_exec_line9                  9       1460      99702       99702
__anonymous_block                                 0          3        336         276
DBMS_OUTPUT.GET_LINES                           180          2         60          57
DBMS_OUTPUT.GET_LINE                            129          2          3           3
DBMS_HPROF.STOP_PROFILING                        59          1          0           0
```

你可以看到，启用了缓存的`GET_RATE`函数仅执行了 1,460 次（与最佳标量子查询缓存示例的结果相同）。有趣的是，使用结果缓存*并不能*减少切换到 PL/SQL 虚拟机的开销，因此 Oracle 数据库需要为查询需要处理的 918,000 行数据中的每一行，都将控制权传递给 PL/SQL 引擎一次。因此，将纯 PL/SQL 函数（即没有嵌入任何 SQL 的函数）转换为使用结果缓存通常得不偿失。首先，上下文切换没有减少；其次，你可能会发现，将一个函数调用重定向到结果缓存并获取保护它的`RC`锁存器，所花费的时间可能比函数本身的执行时间还要长。

最后，Oracle 数据库还为结果缓存提供了几个性能视图。例如，`V$RESULT_CACHE_STATISTICS`视图可以提供有关缓存效率的有用概览。清单 9-40 是在空缓存下运行的，因此清单 9-42 中的统计信息完全归因于此查询。

**清单 9-42. 使用 V$RESULT_CACHE_STATISTICS 调查结果缓存命中情况**

```sql
SQL> SELECT name, value
  2  FROM   v$result_cache_statistics
  3  WHERE  name IN ('Find Count','Create Count Success');

NAME                    VALUE
----------------------- ----------
Create Count Success    1460
Find Count              917383
```

这些统计信息展示了一个非常高效的缓存。`GET_RATE`函数仅执行了 1,460 次就加载了缓存（由`Create Count Success`统计信息证明，该统计信息也与清单 9-41 分层分析器报告中的`GET_RATE`统计信息相对应）。一旦缓存，结果就被大量重用（`Find Count`统计信息显示缓存的结果被使用了超过 917,000 次）。如果此时我重复执行销售查询，PL/SQL 函数将根本不会执行（假设缓存的结果没有失效），响应时间将进一步减少，尽管幅度不大。

**注意** 如果你运行的不是 Oracle 数据库 11*g*，你可以通过使用关联数组创建自己的基于会话的 PL/SQL 缓存来模拟结果缓存（在一定程度上）。虽然这不会以相同的方式减少函数执行次数，但它将显著减少函数内嵌入的 SQL 查找的影响。请参阅本章末尾的*“调整 PL/SQL”*部分，其中有一个示例将`GET_RATE`函数从结果缓存版本转换为用户定义的数组缓存实现。但是，请务必考虑这种方法的潜在缺点，如该部分所述。


##### 确定性函数

如前所述，Oracle 数据库针对确定性函数有一种内部优化机制，这有时能使其成为减少 PL/SQL 函数调用的一个有用特性。在清单 9-43 中，我展示了将`GET_CUSTOMER_NAME`函数声明为确定性（即在函数规范中使用`DETERMINISTIC`关键字）并运行清单 9-5 中按客户销售报告的一个变体所产生的效果。

**清单 9-43.** 声明`DETERMINISTIC`函数的效果

```
SQL> CREATE OR REPLACE FUNCTION format_customer_name (
  2                             p_first_name IN VARCHAR2,
  3                             p_last_name  IN VARCHAR2
  4                             ) RETURN VARCHAR2 DETERMINISTIC AS
<snip>
  9 /

Function created.

SQL> SELECT /*+ NO_MERGE(@inner) */
  2         calendar_year
  3  ,      format_customer_name(
  4            cust_first_name, cust_last_name
  5            )                              AS cust_name
  6  ,      SUM(quantity_sold)                AS qty_sold
  7  ,      SUM(amount_sold)                  AS amt_sold
  8  FROM  (
  9         SELECT /*+
 10                    QB_NAME(inner)
 11                    NO_ELIMINATE_OBY
 12                 */
 13                t.calendar_year
 14  ,      c.cust_first_name
 15  ,      c.cust_last_name
 16  ,      s.quantity_sold
 17  ,      s.amount_sold
 18  FROM   sales     s
 19  ,      customers c
 20  ,      times     t
 21  WHERE  s.cust_id = c.cust_id
 22  AND    s.time_id = t.time_id
 23  ORDER  BY
 24         c.cust_first_name
 25  ,      c.cust_last_name
 26        )
 27  GROUP  BY
 28         calendar_year
 29  ,      format_customer_name(
 30            cust_first_name, cust_last_name
 31            )
 32  ;

11604 rows selected.

Elapsed: 00:00:07.83

Statistics
----------------------------------------------------------
       3189  consistent gets
          0  physical reads

SQL> exec counter.show('Deterministic function calls');

Deterministic function calls: 912708
```

从结果中可以看到，PL/SQL 函数调用次数的减少幅度非常小（比清单 9-5 中的原始查询大约减少了 6,000 次），因此效果并不显著（这个报告实际上比原始版本更慢）。事实上，在这个例子中，为了*哪怕一点点*从确定性函数优化中受益，我需要对输入数据进行预排序，以确保传入函数的客户名称是聚集在一起的。在这种情况下，排序数据的成本超过了减少 6,000 次`FORMAT_CUSTOMER_NAME`调用带来的微小收益，导致我的报告运行得更慢了。

这并不是说针对确定性函数的优化不起作用。如前所述，有许多因素会影响其效率。在某些情况下，它可能非常有效，特别是当你的结果集包含非常少的不同函数输入时。例如，`SALES`表中大约有 3,700 个不同的客户，因此我简化了我的销售查询，将`SALES`与`CUSTOMERS`连接，在内联视图中对数据进行排序，并在没有聚合的情况下调用`FORMAT_CUSTOMER_NAME`函数。使用这个简化的查询，对于 918,000 行数据，PL/SQL 函数仅被执行了 36,000 次。当然，这比标准的销售报告要好，但与我展示的其他替代技术相比，这是一种相当受限的优化。

![images](img/square.jpg) **注意** 就其本质而言，返回随机数据或执行 SQL 查找的 PL/SQL 函数不是确定性的。除非函数确实是确定性的，否则不要试图将其声明为确定性，否则你可能会面临查询返回错误结果的风险。

### 协助基于成本的优化器

如前所述，有几种方式可能导致 PL/SQL 函数干扰基于成本的优化器，这主要是由于缺乏统计信息和依赖于默认值所致。有一系列方法可用于改进 CBO 对包含 PL/SQL 函数的 SQL 的处理，我将演示以下技术：

*   基于函数的索引
*   扩展统计信息
*   默认统计信息
*   可扩展优化器


