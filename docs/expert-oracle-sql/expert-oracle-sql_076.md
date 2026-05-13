# 扩展统计

与通过创建基于函数的索引而隐式创建的列类似，扩展统计是隐藏的虚拟列。在第 6 章中，我解释了扩展统计的主要目的，即让 CBO 理解列之间的相关性。现在让我使用在清单 9-1 中创建的 `STATEMENT` 表给您举个例子。清单 9-11 显示了一个查询的两个执行计划，一个是在为 `DESCRIPTION` 和 `AMOUNT_CATEGORY` 列创建扩展统计之前，另一个是之后。

清单 9-11. 多列扩展统计的使用

```
--
-- 没有扩展统计时的查询及相关执行计划
--
SELECT *
  FROM statement t
 WHERE     transaction_date = DATE '2013-01-02'
       AND posting_date = DATE '2013-01-02';

| Id  | Operation         | Name      | Rows  | Time     |

|   0 | SELECT STATEMENT  |           |     4 | 00:00:01 |
|*  1 |  TABLE ACCESS FULL| STATEMENT |     4 | 00:00:01 |

--
-- 现在我们为这两列收集扩展统计信息
--
DECLARE
   extension_name   all_tab_col_statistics.column_name%TYPE;
BEGIN
   extension_name :=
      DBMS_STATS.create_extended_stats (
         ownname     => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
        ,tabname     => 'STATEMENT'
        ,extension   => '(TRANSACTION_DATE,POSTING_DATE)');

   DBMS_STATS.gather_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT'
     ,partname      => NULL
     ,GRANULARITY   => 'GLOBAL'
     ,method_opt    => 'FOR ALL COLUMNS SIZE 1'
     ,cascade       => FALSE);
END;
/
--
-- 现在让我们看看该查询的新执行计划
--

| Id  | Operation         | Name      | Rows  | Time     |

|   0 | SELECT STATEMENT  |           |    17 | 00:00:01 |
|*  1 |  TABLE ACCESS FULL| STATEMENT |    17 | 00:00:01 |
```

清单 9-11 中的查询包含对 `TRANSACTION_DATE` 和 `POSTING_DATE` 列的谓词。`TRANSACTION_DATE` 有 `10` 个值，`POSTING_DATE` 有 `12` 个值，因此在没有扩展统计的情况下，CBO 假设查询将返回 `500/10/12` 行，大约是 `4`。实际上，`TRANSACTION_DATE` 和 `POSTING_DATE` 只有 `30` 种不同的组合，在收集了扩展统计之后，我们查询的执行计划显示基数为 `17`，这是 `500/30` 四舍五入的结果。

![image](img/sq.jpg) **注意**  我本可以将 `TRANSACTION_DATE` 列创建为一个虚拟列，由表达式 `TRUNC (TRANSACTION_DATE_TIME)` 派生而来。然而，像清单 9-11 中定义的这类扩展统计，不能基于虚拟列。

我说过，对于基数估算，我们不应该过度担心 `2` 倍或 `3` 倍的误差，而 `17` 和 `4` 之间的差异也不过如此。然而，在多个相关列上有多个谓词的情况相当常见，这类基数误差的累积效应可能是灾难性的。

![image](img/sq.jpg) **注意**  在第 6 章中，我提到当您开始分析一个新应用程序或现有应用程序的新主要版本的性能时，寻找改进大量语句执行计划的方法是个好主意。扩展统计就是这类变更的主要例子：如果您正确设置了扩展统计，CBO 可能会改进大量语句的执行计划。

还有第二种扩展统计可能很有用。这种扩展就像您通过基于函数的索引得到的隐藏虚拟列，但没有索引本身。清单 9-12 显示了一个例子。

清单 9-12. 为表达式创建扩展统计

```
SELECT *
  FROM statement
 WHERE CASE
          WHEN description <> 'Flight' AND transaction_amount > 100
          THEN
             'HIGH'
       END = 'HIGH';

| Id  | Operation         | Name      | Rows  | Time     |

|   0 | SELECT STATEMENT  |           |     5 | 00:00:01 |
|*  1 |  TABLE ACCESS FULL| STATEMENT |     5 | 00:00:01 |

Predicate Information (identified by operation id):

1 - filter(CASE  WHEN ("DESCRIPTION"<>'Flight' AND
              "TRANSACTION_AMOUNT">100) THEN 'HIGH' END ='HIGH')

DECLARE
   extension_name   all_tab_cols.column_name%TYPE;
BEGIN
   extension_name :=
      DBMS_STATS.create_extended_stats (
         ownname     => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
        ,tabname     => 'STATEMENT'
        ,extension   => q'[(CASE WHEN DESCRIPTION <> 'Flight'
                                      AND TRANSACTION_AMOUNT > 100
                                 THEN 'HIGH' END)]');

   DBMS_STATS.gather_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT'
     ,partname      => NULL
     ,method_opt    => 'FOR ALL COLUMNS SIZE 1'
     ,cascade       => FALSE);
END;
/

| Id  | Operation         | Name      | Rows  | Time     |

|   0 | SELECT STATEMENT  |           |   105 | 00:00:01 |
|*  1 |  TABLE ACCESS FULL| STATEMENT |   105 | 00:00:01 |

Predicate Information (identified by operation id):

1 - filter(CASE  WHEN ("DESCRIPTION"<>'Flight' AND
              "TRANSACTION_AMOUNT">100) THEN 'HIGH' END ='HIGH')
```

清单 9-12 中的查询列出了所有 `AMOUNT > 100` 且不是航班的交易。这个查询必须以特殊方式构造，以便我们能够利用扩展统计。在创建扩展统计之前，基于我们惯用的 `1%` 估算，估算的基数是 `5`。在为我们精心构造的表达式创建扩展统计，并随后收集表上的统计信息之后，我们可以看到 CBO 已经能够准确地估算出查询将返回的行数为 `105`。

