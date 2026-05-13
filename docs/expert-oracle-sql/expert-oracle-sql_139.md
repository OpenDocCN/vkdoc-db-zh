# 支付模式概述

绝大多数工资支付在每月 20 号进行（若 20 号为周末则提前至周五）。对于错误、调整及高管薪酬等少数例外情况，支付日期可能稍晚，但占比极小。如果`PAYMENT_DATE`列上没有直方图，正常支付日的选择率（selectivity）会过低。由于`PAYMENTS`表存储了一年的数据，我们的直方图应显示大约 1/12 的支付发生在 12 个标准发薪日，并且年内其他日期的选择率应非常小。

## PAYGRADE 与 PAYMENT_DATE 关联分析

实际上，我们虚构公司中最资深的两位员工（`PAYGRADE`为 1）在每月最后一个工作日领取薪酬，而其他所有人都在 20 号左右领取。如果处理这两位`PAYGRADE`为 1 的高管的特殊代码中，包含同时基于`PAYGRADE`和`PAYMENT_DATE`的谓词条件，那么 CBO（基于成本的优化器）如果假设大多数支付发生在标准发薪日，就不太可能选择合适的执行计划。

我们可以为`PAYGRADE`和`PAYMENT_DATE`列设置列组统计信息，以显示这两位`PAYGRADE`为 1 的员工在其特殊发薪日各有一次支付。我们也可以为可能发生例外的日期设置非零选择率，这样列组统计信息仍然会被使用。

## JOB_DESCRIPTION 列分析

共有 13 种职位描述和 10 个薪酬等级，但并非有 130 种组合。我们可能会考虑为`PAYGRADE`和`JOB_DESCRIPTION`设置列组统计信息，但只要应用程序从不使用独立于`PAYGRADE`的`JOB_DESCRIPTION`谓词，就存在一个更简单的方法：我们可以直接告诉 CBO `JOB_DESCRIPTION`只有两个值。这种“善意的谎言”在您有大量相关列，并且只想阻止 CBO 过度降低基数估计时很有用。一旦我们构造好示例数据，您就会看到它的效果。

## 构建样本数据

现在我们已经识别了问题列，并决定了希望 CBO 如何处理它们，接下来就可以构造符合我们模型的一些数据了。

Listing 20-4 创建了一个表 `SAMPLE_PAYMENTS`，它包含了我们感兴趣的`PAYMENTS`表的列子集，即`PAYGRADE`、`PAYMENT_DATE`和`JOB_DESCRIPTION`。

Listing 20-4 还展示了一个过程 `TSTATS.AMEND_TIME_BASED_STATISTICS`，用于填充`SAMPLE_PAYMENTS`。一旦`SAMPLE_PAYMENTS`被数据填充，我们就收集统计信息，然后将列统计信息从`SAMPLE_PAYMENTS`复制到真实的`PAYMENTS`表。

**Listing 20-4. 创建并填充 SAMPLE_PAYMENTS 表**

```
CREATE TABLE sample_payments
(
   paygrade          INTEGER
  ,payment_date      DATE
  ,job_description   CHAR (20)
);

CREATE OR REPLACE PACKAGE tstats
AS
-- Other procedures cut
     PROCEDURE amend_time_based_statistics (
      effective_date    DATE DEFAULT SYSDATE);
-- Other procedures cut
END tstats;
/

CREATE OR REPLACE PACKAGE BODY tstats AS
  PROCEDURE amend_time_based_statistics (
      effective_date    DATE DEFAULT SYSDATE)
   IS
      distcnt   NUMBER;
      density   NUMBER;
      nullcnt   NUMBER;
      srec      DBMS_STATS.statrec;
      avgclen   NUMBER;
   BEGIN
      --
      -- Step 1: Remove data from previous run
      --
      DELETE FROM sample_payments;

      --
      -- Step 2:  Add data for standard pay for standard employees
      --
      INSERT INTO sample_payments (paygrade, payment_date, job_description)
         WITH payment_dates
              AS (    SELECT ADD_MONTHS (TRUNC (effective_date, 'MM') + 19
                                        ,1 - ROWNUM)
                                standard_paydate
                        FROM DUAL
                  CONNECT BY LEVEL <= 12)
             ,paygrades
              AS (    SELECT ROWNUM + 1 paygrade
                        FROM DUAL
                  CONNECT BY LEVEL <= 9)
             ,multiplier
              AS (    SELECT ROWNUM rid
                        FROM DUAL
                  CONNECT BY LEVEL <= 100)
         SELECT paygrade
               ,CASE MOD (standard_paydate - DATE '1001-01-06', 7)
                   WHEN 5 THEN standard_paydate - 1
                   WHEN 6 THEN standard_paydate - 2
                   ELSE standard_paydate
                END
                   payment_date
               ,'AAA' job_description
           FROM paygrades, payment_dates, multiplier;

      --
      -- Step 3:  Add data for paygrade 1
      --
      INSERT INTO sample_payments (paygrade, payment_date, job_description)
         WITH payment_dates
              AS (    SELECT ADD_MONTHS (LAST_DAY (TRUNC (effective_date))
                                        ,1 - ROWNUM)
                                standard_paydate
                        FROM DUAL
                  CONNECT BY LEVEL <= 12)
         SELECT 1 paygrade
               ,CASE MOD (standard_paydate - DATE '1001-01-06', 7)
                   WHEN 5 THEN standard_paydate - 1
                   WHEN 6 THEN standard_paydate - 2
                   ELSE standard_paydate
                END
                   payment_dates
               ,'zzz' job_description
           FROM payment_dates;

      --
      -- Step 4:  Add rows for exceptions.
      --
      INSERT INTO sample_payments (paygrade, payment_date, job_description)
         WITH payment_dates
              AS (    SELECT ADD_MONTHS (TRUNC (effective_date, 'MM') + 19
                                        ,1 - ROWNUM)
                                standard_paydate
                        FROM DUAL
                  CONNECT BY LEVEL <= 12)
             ,paygrades
              AS (    SELECT ROWNUM + 1 paygrade
                        FROM DUAL
                  CONNECT BY LEVEL <= 7)
         SELECT paygrade
               ,CASE MOD (standard_paydate - DATE '1001-01-06', 7)
                   WHEN 5 THEN standard_paydate - 2 + paygrade
                   WHEN 6 THEN standard_paydate - 3 + paygrade
                   ELSE standard_paydate - 1 + paygrade
                END
                   payment_date
               ,'AAA' job_description
           FROM paygrades, payment_dates;

      --
      -- Step 5:  Gather statistics for SAMPLE_PAYMENTS
      --
      DBMS_STATS.gather_table_stats (
         ownname      => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
        ,tabname      => 'SAMPLE_PAYMENTS'
        ,method_opt   =>    'FOR COLUMNS SIZE 1 JOB_DESCRIPTION '
                         || 'FOR COLUMNS SIZE 254 PAYGRADE,PAYMENT_DATE, '
                         || '(PAYGRADE,PAYMENT_DATE)');

      --
      -- Step 6:  Copy column statistics from SAMPLE_PAYMENTS to PAYMENTS
      --
      FOR r IN (SELECT column_name, histogram
                  FROM all_tab_cols
                 WHERE table_name = 'SAMPLE_PAYMENTS')
      LOOP
         DBMS_STATS.get_column_stats (
            ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
           ,tabname   => 'SAMPLE_PAYMENTS'
           ,colname   => r.column_name
           ,distcnt   => distcnt
           ,density   => density
           ,nullcnt   => nullcnt
           ,srec      => srec
           ,avgclen   => avgclen);
```


## 使用伪造数据生成稳定统计信息

Listing 20-4 中的代码需要注意以下几点：

*   `SAMPLE_PAYMENTS` 表中的行数不需要与 `PAYMENTS` 表一样多。我们只需要足够的行来演示想要的数据分布。事实上，`SAMPLE_PAYMENTS` 包含 10,896 行，而 `PAYMENTS` 当前包含 120,000 行。
*   `SAMPLE_PAYMENTS` 表并未包含 `PAYMENTS` 中的所有列，仅包含需要特殊处理的列。
*   从 `SAMPLE_PAYMENTS` 中删除历史数据后，我们为薪资等级 2 到 9 与 12 个标准发薪日的每种组合添加 100 行。我们为 `JOB_DESCRIPTION` 指定一个低值。
*   然后，我们添加高管薪资的行——每个高管发薪日仅一行。我们为 `JOB_DESCRIPTION` 指定第二个高值。
*   最后添加的一组行是用于特殊条目——标准发薪日后一周内的每一天一行。我们为 `PAYGRADE` 指定一个任意值（不是 1）。
*   当我们收集统计信息时，重要的是**不**为 `JOB_DESCRIPTION` 收集直方图，因为我们为该列指定的值不是真实的。我们对其余列收集直方图。注意 `METHOD_OPT` 语法的一个有趣特性：我们可以通过这种方式隐式创建扩展统计信息！
*   然后，我们将为 `SAMPLE_PAYMENTS` 收集的列统计信息复制到 `PAYMENTS` 中。

你可能想知道我们在这里实现了什么。所有这些操作的关键点在于，每个月我们都可以重新生成 `SAMPLE_PAYMENTS` 中的数据，并且可以 100% 确定数据的**唯一**差异是 `PAYMENT_DATE` 的值。其他一切都将保持不变。这意味着一旦我们将统计信息复制到 `PAYMENTS`，执行计划将保持不变。Listing 20-5 展示了这在实践中的工作方式。

**Listing 20-5. 展示伪造数据生成的统计信息在实际中的应用**

```sql
DECLARE
   extension_name     all_stat_extensions.extension_name%TYPE;
   extension_exists   EXCEPTION;
   PRAGMA EXCEPTION_INIT (extension_exists, -20007);
BEGIN
   extension_name :=
      DBMS_STATS.create_extended_stats (
         ownname     => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
        ,tabname     => 'PAYMENTS'
        ,extension   => '(PAYGRADE,PAYMENT_DATE)');
EXCEPTION
   WHEN extension_exists
   THEN
      NULL;
END;
/

EXEC tstats.amend_time_based_statistics(date '2014-05-01');

ALTER SESSION SET statistics_level='ALL';

SELECT COUNT (*)
  FROM payments
 WHERE paygrade = 1 AND payment_date = DATE '2014-04-30';

SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC IOSTATS LAST'));

| Id  | Operation                            | Name         | Starts | E-Rows | A-Rows |

|   0 | SELECT STATEMENT                     |              |      1 |        |      1 |
|   1 |  SORT AGGREGATE                      |              |      1 |      1 |      1 |
|*  2 |   TABLE ACCESS BY INDEX ROWID BATCHED| PAYMENTS     |      1 |     11 |      2 |
|*  3 |    INDEX RANGE SCAN                  | PAYMENTS_IX1 |      1 |    132 |     24 |

SELECT COUNT (*)
  FROM payments
 WHERE paygrade = 10 AND payment_date = DATE '2014-04-18';

SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC IOSTATS LAST'));

| Id  | Operation          | Name     | Starts | E-Rows | A-Rows |

|   0 | SELECT STATEMENT   |          |      1 |        |      1 |
|   1 |  SORT AGGREGATE    |          |      1 |      1 |      1 |
|*  2 |   TABLE ACCESS FULL| PAYMENTS |      1 |   1101 |   4525 |

SELECT COUNT (*)
  FROM payments
 WHERE paygrade = 4 AND payment_date = DATE '2014-04-18';

SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC IOSTATS LAST'));

| Id  | Operation          | Name     | Starts | E-Rows | A-Rows |

|   0 | SELECT STATEMENT   |          |      1 |        |      1 |
|   1 |  SORT AGGREGATE    |          |      1 |      1 |      1 |
|*  2 |   TABLE ACCESS FULL| PAYMENTS |      1 |   1101 |     28 |

SELECT COUNT (*)
  FROM payments
 WHERE paygrade = 8 AND job_description = 'JUNIOR TECHNICIAN';

SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC IOSTATS LAST'));

| Id  | Operation         | Name         | Starts | E-Rows | A-Rows |

|   0 | SELECT STATEMENT  |              |      1 |        |      1 |
|   1 |  SORT AGGREGATE   |              |      1 |      1 |      1 |
|*  2 |   INDEX RANGE SCAN| PAYMENTS_IX1 |      1 |   6674 |   7716 |

SELECT COUNT (*)
  FROM payments
 WHERE employee_id = 101;

SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC IOSTATS LAST'));

| Id  | Operation          | Name     | Starts | E-Rows | A-Rows |

|   0 | SELECT STATEMENT   |          |      1 |        |      1 |
|   1 |  SORT AGGREGATE    |          |      1 |      1 |      1 |
|*  2 |   TABLE ACCESS FULL| PAYMENTS |      1 |     12 |     12 |

EXEC tstats.amend_time_based_statistics(date '2015-05-01');

SELECT COUNT (*)
  FROM payments
 WHERE paygrade = 1 AND payment_date = DATE '2015-04-30';

SELECT * FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC +ROWS'));

| Id  | Operation                            | Name         | Rows  |

|   0 | SELECT STATEMENT                     |              |       |
|   1 |  SORT AGGREGATE                      |              |     1 |
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED| PAYMENTS     |    11 |
|   3 |    INDEX RANGE SCAN                  | PAYMENTS_IX1 |   132 |

SELECT COUNT (*)
  FROM payments
 WHERE paygrade = 10 AND payment_date = DATE '2015-04-20';

SELECT * FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC +ROWS'));

| Id  | Operation          | Name     | Rows  |

|   0 | SELECT STATEMENT   |          |       |
|   1 |  SORT AGGREGATE    |          |     1 |
|   2 |   TABLE ACCESS FULL| PAYMENTS |  1101 |

SELECT COUNT (*)
  FROM payments
 WHERE paygrade = 4 AND payment_date = DATE '2015-04-20';

SELECT * FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC +ROWS'));

| Id  | Operation          | Name     | Rows  |

|   0 | SELECT STATEMENT   |          |       |
|   1 |  SORT AGGREGATE    |          |     1 |
|   2 |   TABLE ACCESS FULL| PAYMENTS |  1101 |

SELECT COUNT (*)
  FROM payments
 WHERE paygrade = 8 AND job_description = 'JUNIOR TECHNICIAN';

SELECT * FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC +ROWS'));

| Id  | Operation         | Name         | Rows  |

|   0 | SELECT STATEMENT  |              |       |
|   1 |  SORT AGGREGATE   |              |     1 |
|   2 |   INDEX RANGE SCAN| PAYMENTS_IX1 |  6674 |

SELECT COUNT (*)
  FROM payments
 WHERE employee_id = 101;

SELECT * FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC +ROWS'));

| Id  | Operation          | Name     | Rows  |

|   0 | SELECT STATEMENT   |          |       |
|   1 |  SORT AGGREGATE    |          |     1 |
|   2 |   TABLE ACCESS FULL| PAYMENTS |    12 |

```

Listing 20-5 首先展示了为 `PAYMENTS` 表的两个列 `PAYGRADE` 和 `PAYMENT_DATE` 创建列组扩展统计信息。这是一项一次性的任务，并且与调用 `TSTATS.AMEND_TIME_BASED_STATISTICS` 不同，它不会每个月都执行。


通常情况下，`TSTATS.AMEND_TIME_BASED_STATISTICS`会使用当前日期生成样本数据，但出于演示目的，清单 20-5 首先指定了 2014 年 5 月 1 日。清单 20-5 中的第一个查询涉及`PAYGRADE` 1。输入到`COUNT`函数的有 24 行，但估计的基数过高，为 132。尽管如此，仍然选择了正确的索引访问方法。如果高选择性存在其他问题，可能需要进一步调整样本数据。

清单 20-5 中的第二个和第三个查询涉及标准薪资等级和 4 月 18 日的标准发薪日（2014 年 4 月 20 日是星期日）。CBO 为这两个查询提供了相同的 1,101 行估计值。使用了全表扫描，这似乎是合理的。对于`PAYGRADE` 10，基数估计过低，而对于`PAYGRADE` 4，则估计过高。然而，我们得到了我们想要的一致估计和一致的执行计划。

清单 20-5 中的第四个查询涉及`PAYGRADE`和`JOB_DESCRIPTION`。尽管我们没有创建显示`PAYGRADE` 8 的员工只有两个`JOB_DESCRIPTION`值的列组统计信息，但对 6,674 行的估计与`PAYMENTS`表中实际的 7,716 匹配行相当接近。这是因为从`SAMPLE_PAYMENTS`复制的统计信息使 CBO 相信整个表中只有两个`JOB_DESCRIPTION`值！如果多个列相互关联，通过这种方式修改列统计信息以反映显著减少的不同值数量，通常可以避免 CBO 低估基数。

清单 20-5 中的下一个查询涉及`EMPLOYEE_ID`上的谓词。我们的样本数据不包含此列，因此`PAYMENTS`表中`EMPLOYEE_ID`列的统计信息未被替换。CBO 基于从`PAYMENTS`表收集的统计信息进行了基数估计。

假设时间流逝。清单 20-5 第二次调用`TSTATS.AMEND_TIME_BASED_STATISTICS`，时间为 2015 年 5 月 1 日，模拟时间的推移。我们可以看到，2015 年新日期的基数估计与 2014 年的相同；我没有显示实际的行数，因为在现实生活中并未更新`PAYMENTS`表。

**管理列统计总结**

我们可能希望修改收集的列统计信息以帮助 CBO 产生更准确的基数估计，和/或随时间稳定这些基数估计，原因有很多。我们可能从收集的列统计信息开始，只是移除`HIGH_VALUE`和`LOW_VALUE`部分。这是最常见的方法。为了处理更复杂的问题，我们可以伪造统计信息，甚至在伪造的数据上收集统计信息。关键是，无论遇到什么问题，我们都能找到一种方法，避免对应用程序使用的表重复收集统计信息。

**统计与分区**

在第 9 章中，我解释了使用分区级统计信息的一些原因，并指出，在我看来，分区级统计信息往往弊大于利。我知道你们中的许多人可能仍需要对此点有所信服。我也确信你们中的一些人会非常正确地担心，从使用分区级统计信息转变为全局统计信息是一个巨大的变化，可能对某些已建立的应用程序带来不必要的风险。让我们来看看这个被高估的过程`DBMS_STATS.COPY_TABLE_STATS`，目的是使这些问题更清晰。

**DBMS_STATS.COPY_TABLE_STATS 的迷思**

`DBMS_STATS.COPY_TABLE_STATS`在 11gR1 中首次被记录，但实际上在 10.2.0.4 及 10gR2 的更高版本中已得到支持。该过程的名称有些不恰当，因为它并非用于复制表统计信息本身，至少不是全局表统计信息。该过程实际上用于将分区统计信息从一个分区复制到同一表的另一个分区，或者将子分区统计信息从一个子分区复制到同一表的另一个子分区。在讨论其缺点之前，让我从解释其预期工作方式开始。

**DBMS_STATS.COPY_TABLE_STATS 的预期工作方式**

使用`DBMS_STATS.COPY_TABLE_STATS`相比收集分区级统计信息有三个关键优势。第一个优势是复制统计信息比收集统计信息快得多。第二个优势是分区统计信息只能在分区填充数据后才能收集，而你可能需要在所有数据加载完毕之前查询该分区。第三个优势，也是最初最吸引我的，是复制统计信息可以最大限度地减少执行计划变更的风险。清单 20-6 展示了`DBMS_STATS.COPY_TABLE_STATS`背后的基本原理。

清单 20-6. 使用 `DBMS_STATS.COPY_TABLE_STATS` 的初始尝试

```sql
CREATE TABLE statement_part
(
   transaction_date_time
  ,transaction_date
  ,posting_date
  ,description
  ,transaction_amount
  ,product_category
  ,customer_category
)
PARTITION BY RANGE
   (transaction_date)
   (
      PARTITION p1 VALUES LESS THAN (DATE '2013-01-05')
     ,PARTITION p2 VALUES LESS THAN (DATE '2013-01-11')
     ,PARTITION p3 VALUES LESS THAN (maxvalue))
PCTFREE 99
PCTUSED 1
AS
       SELECT   TIMESTAMP '2013-01-01 12:00:00.00 -05:00'
              + NUMTODSINTERVAL (TRUNC ( (ROWNUM - 1) / 50), 'DAY')
             ,DATE '2013-01-01' + TRUNC ( (ROWNUM - 1) / 50)
             ,DATE '2013-01-01' + TRUNC ( (ROWNUM - 1) / 50) + MOD (ROWNUM, 3)
             ,DECODE (MOD (ROWNUM, 4)
                     ,0, 'Flight'
                     ,1, 'Meal'
                     ,2, 'Taxi'
                     ,'Deliveries')
             ,DECODE (MOD (ROWNUM, 4)
                     ,0, 200 + (30 * ROWNUM)
                     ,1, 20 + ROWNUM
                     ,2, 5 + MOD (ROWNUM, 30)
                     ,8)
             ,TRUNC ( (ROWNUM - 1) / 50) + 1
             ,MOD ( (ROWNUM - 1), 50) + 1
         FROM DUAL
   CONNECT BY LEVEL <= 500;

CREATE INDEX statement_part_ix1
   ON statement_part (transaction_date)
   LOCAL
   PCTFREE 99;

BEGIN
   DBMS_STATS.delete_table_stats (
      ownname           => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname           => 'STATEMENT_PART'
     ,cascade_parts     => TRUE
     ,cascade_indexes   => TRUE
     ,cascade_columns   => TRUE);

DBMS_STATS.gather_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,partname      => 'P1'
     ,granularity   => 'PARTITION'
     ,method_opt    => 'FOR ALL COLUMNS SIZE 1');

DBMS_STATS.copy_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,srcpartname   => 'P1'
     ,dstpartname   => 'P2');

DBMS_STATS.copy_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,srcpartname   => 'P1'
     ,dstpartname   => 'P3');
END;
/

--
-- 我们可以看到 NUM_ROWS 统计信息已从 P1 复制到其他分区并聚合到全局统计信息中。
--

SELECT partition_name, num_rows
  FROM all_tab_statistics
 WHERE owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
       AND table_name = 'STATEMENT_PART';

PARTITION_NAME    NUM_ROWS
```


