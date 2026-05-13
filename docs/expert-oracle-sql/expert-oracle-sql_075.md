# 手动创建直方图与虚拟列

## 手动创建频率直方图的注意事项

*   手动创建一个具有不合理最小值和最大值的频率直方图，意味着你不必担心新出现的列值高于之前的最大值或低于之前的最小值。然而，像 `TRANSACTION_AMOUNT > 15000` 这样的范围谓词可能会导致基数估计值过高。如果你使用范围谓词，你可能需要选择合理的最小值和最大值。
*   不要为 `srec.bkvals` 数组选择过小的值，否则可能会产生舍入误差。例如，不要将清单 9-8 中的 600、400 和 600 替换为 6、4 和 6。这会改变像 `TRANSACTION_AMOUNT > 8` 这类谓词的结果。
*   如果来自 `DENSITY` 的默认选择率高于为提供的端点指定的选择率，CBO（基于成本的优化器）会感到困惑。
*   对于手动创建的直方图，只有 `DENSITY` 统计量被用于计算列中未指定值的选择率。当直方图是自动生成（`ALL_TAB_COL_STATISTICS` 中的 `USER_STATS = 'NO'`）时，未指定值的选择率是最低提供端点值选择率的一半。
*   还有其他类型的直方图。这些包括 *基于高度的直方图*、*顶部频率直方图* 和 *混合直方图*。顶部频率和混合直方图首次出现在 12cR1 中。根据我的经验（可能并不典型），我在这里描述的手动创建的频率直方图是我唯一需要的类型。

## 查看直方图数据

当你使用表 9-1 中列出的视图之一来查看直方图时，你可能会暂时感到困惑。清单 9-9 显示了我在清单 9-8 中创建的 `TRANSACTION_AMOUNT` 列上的直方图：

**[清单 9-9]. 显示直方图数据**

```sql
SELECT table_name
      ,column_name
      ,endpoint_number
      ,endpoint_value
      ,endpoint_actual_value
  FROM all_tab_histograms
 WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
       AND table_name = 'STATEMENT'
       AND column_name = 'TRANSACTION_AMOUNT';

TABLE_NAME    COLUMN_NAME             ENDPOINT_NUMBER     ENDPOINT_VALUE     ENDPOINT_ACTUAL_VALUE
STATEMENT     TRANSACTION_AMOUNT      600                -10000000
STATEMENT     TRANSACTION_AMOUNT      1000                8
STATEMENT     TRANSACTION_AMOUNT      1600                10000000
```

需要警惕两点：

1.  `ENDPOINT_ACTUAL_VALUE` 仅对前六个字符由连续端点共享的字符列进行填充。
2.  `ENDPOINT_NUMBER` 是累积的。它代表了到该点为止的基数的运行总和。

## 虚拟列与隐藏列

我已经提到，基于函数的索引会创建一个隐藏的对象列，并且该隐藏的对象列可以像常规对象列一样拥有统计信息。但是因为该列是隐藏的，所以它不会被以 `SELECT * FROM` 开头的查询返回。这类隐藏列也被认为是 *虚拟列*，因为它们在表中没有物理表示。

Oracle 数据库 11gR1 引入了非隐藏虚拟列的概念。这些非隐藏的虚拟列是通过 `CREATE TABLE` 或 `ALTER TABLE` DDL 语句显式声明的，而不是作为基于函数的索引创建的一部分隐式声明的。从 CBO 的角度来看，虚拟列的处理方式几乎相同，无论它们是隐藏的还是非隐藏的。清单 9-10 展示了 CBO 如何使用在清单 9-1 中创建的 `STATEMENT` 表的虚拟列上的统计信息。

**[清单 9-10]. CBO 对虚拟列统计信息的使用**

```sql
--
-- 查询 1：使用显式声明的虚拟列
--
SELECT *
  FROM statement
 WHERE posting_delay = 1;

| Id  | Operation         | Name      | Rows  |

|   0 | SELECT STATEMENT  |           |   167 |
|*  1 |  TABLE ACCESS FULL| STATEMENT |   167 |

Predicate Information (identified by operation id):

1 - filter("POSTING_DELAY"=1)

--
-- 查询 2：使用与显式声明的虚拟列等效的表达式。
--
SELECT *
  FROM statement
 WHERE (posting_date - transaction_date) = 1;

| Id  | Operation         | Name      | Rows  |

|   0 | SELECT STATEMENT  |           |   167 |
|*  1 |  TABLE ACCESS FULL| STATEMENT |   167 |

Predicate Information (identified by operation id):

1 - filter("STATEMENT"."POSTING_DELAY"=1)
--
-- 查询 3：使用与虚拟列不完全相同的表达式。
--
SELECT *
  FROM statement
 WHERE (transaction_date - posting_date) = -1;

| Id  | Operation         | Name      | Rows  |

|   0 | SELECT STATEMENT  |           |     5 |
|*  1 |  TABLE ACCESS FULL| STATEMENT |     5 |

Predicate Information (identified by operation id):

1 - filter("TRANSACTION_DATE"-"POSTING_DATE"=(-1))

--
-- 查询 4：使用隐藏列
--
SELECT /*+ full(s) */ sys_nc00010$
  FROM statement s
 WHERE sys_nc00010$ = TIMESTAMP '2013-01-02 17:00:00.00';

| Id  | Operation         | Name      | Rows  |

|   0 | SELECT STATEMENT  |           |    50 |
|*  1 |  TABLE ACCESS FULL| STATEMENT |    50 |

Predicate Information (identified by operation id):

1 - filter(SYS_EXTRACT_UTC("TRANSACTION_DATE_TIME")=TIMESTAMP'
              2013-01-02 17:00:00.000000000')
--
-- 查询 5：使用与隐藏列等效的表达式。
--

SELECT SYS_EXTRACT_UTC (transaction_date_time)
  FROM statement s
 WHERE SYS_EXTRACT_UTC (transaction_date_time) =
          TIMESTAMP '2013-01-02 17:00:00.00';

| Id  | Operation        | Name                | Rows  |

|   0 | SELECT STATEMENT |                     |    50 |
|*  1 |  INDEX RANGE SCAN| STATEMENT_I_TRAN_DT |    50 |

Predicate Information (identified by operation id):

1 - access(SYS_EXTRACT_UTC("TRANSACTION_DATE_TIME")=TIMESTAMP'
              2013-01-02 17:00:00.000000000')
--
-- 查询 6：使用带时区的 timestamp 列
--

SELECT transaction_date_time
  FROM statement s
 WHERE transaction_date_time = TIMESTAMP '2013-01-02 12:00:00.00 -05:00';

| Id  | Operation                           | Name                | Rows  |

|   0 | SELECT STATEMENT                    |                     |    50 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| STATEMENT           |    50 |
|*  2 |   INDEX RANGE SCAN                  | STATEMENT_I_TRAN_DT |    50 |

Predicate Information (identified by operation id):

2 - access(SYS_EXTRACT_UTC("TRANSACTION_DATE_TIME")=TIMESTAMP'
              2013-01-02 17:00:00.000000000')
```

清单 9-10 中的第一个查询涉及对显式声明的虚拟列的谓词。可以使用该虚拟列的统计信息，CBO 将非空行数除以 `POSTING_DELAY` 的不同值的数量。结果是 50/3，四舍五入后得到 167。第二个查询使用了与虚拟列定义完全相同的表达式，同样使用了虚拟列统计信息，得出的基数估计值为 167。请注意，虚拟列的名称出现在第二个查询执行计划的谓词部分。第三个查询使用了与虚拟列定义相似但不完全相同的表达式，现在无法使用虚拟列统计信息，CBO 不得不退而求其次，使用其 1% 的选择率猜测。


当我们在隐藏列上使用谓词时，会看到类似的行为。清单 9-10 中的第四个查询有点不寻常，但完全合法地显式引用了为我们的基于函数的索引生成的隐藏列。CBO 能够利用表中该表达式存在十个不同值这一事实，将基数估算为 `500/10`，即 `50`。我强制使用了全表扫描来强调一个事实：尽管隐藏列纯粹是由于基于函数的索引而存在，但无论使用何种访问方法，都可以使用这些统计信息。清单 9-10 中的第五和第六个查询也利用了隐藏列上的统计信息。第五个查询使用了隐藏列的表达式，而第六个查询使用了 `TIMESTAMP WITH TIME ZONE` 列的名称。请注意，在最后一个查询中，必须访问表以检索索引中缺失的时区信息。对最后三个查询的最后一个观察是，执行计划的谓词部分没有提及隐藏列名，这是对显式声明的虚拟列的过滤谓词显示的一个合理变体。

