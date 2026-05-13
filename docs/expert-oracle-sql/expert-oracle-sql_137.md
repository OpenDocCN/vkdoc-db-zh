# SQL 代码清单与解释

```sql
INSERT INTO payments (payment_id
                     ,employee_id
                     ,special_flag
                     ,paygrade
                     ,payment_date
                     ,job_description)
   WITH standard_payment_dates
        AS (    SELECT ADD_MONTHS (DATE '2014-01-20', ROWNUM - 1)
                          standard_paydate
                      ,LAST_DAY (ADD_MONTHS (DATE '2014-01-20', ROWNUM - 1))
                          last_day_month
                  FROM DUAL
            CONNECT BY LEVEL <= 12)
       ,employees
        AS (    SELECT ROWNUM employee_id, TRUNC (LOG (2.6, ROWNUM)) + 1 paygrade
                  FROM DUAL
            CONNECT BY LEVEL <= 10000)
       ,q1
        AS (SELECT ROWNUM payment_id
                  ,employee_id
                  ,DECODE (MOD (ROWNUM, 100), 0, 'Y', NULL) special_flag
                  ,paygrade
                  --
                  -- The calculation in the next few lines to determine what day of the week
                  -- the 20th of the month falls on does not use the TO_CHAR function
                  -- as the results of this function depend on NLS settings!
                  --
                   ,CASE
                      WHEN MOD (ROWNUM, 100) = 0
                      THEN
                         standard_paydate + MOD (ROWNUM, 7)
                      WHEN     paygrade = 1
                           AND MOD (last_day_month - DATE '1001-01-06', 7) =

THEN
                         last_day_month - 1
                      WHEN     paygrade = 1
                           AND MOD (last_day_month - DATE '1001-01-06', 7) =

THEN
                         last_day_month - 2
                      WHEN paygrade = 1
                      THEN
                         last_day_month
                      WHEN MOD (standard_paydate - DATE '1001-01-06', 7) = 5
                      THEN
                         standard_paydate - 1
                      WHEN MOD (standard_paydate - DATE '1001-01-06', 7) = 6
                      THEN
                         standard_paydate - 2
                      ELSE
                         standard_paydate
                   END
                      paydate
                  ,DECODE (
                      paygrade
                     ,1, 'SENIOR EXECUTIVE'
                     ,2, 'JUNIOR EXECUTIVE'
                     ,3, 'SENIOR DEPARTMENT HEAD'
                     ,4, 'JUNIOR DEPARTMENT HEAD'
                     ,5, 'SENIOR MANAGER'
                     ,6, DECODE (MOD (ROWNUM, 3)
                                ,0, 'JUNIOR MANAGER'
                                ,1, 'SENIOR TECHNICIAN'
                                ,'SENIOR SUPERVISOR')
                     ,7, DECODE (MOD (ROWNUM, 2)
                                ,0, 'SENIOR TECHNICIAN'
                                ,'SENIOR SUPERVISOR')
                     ,8, DECODE (MOD (ROWNUM, 2)
                                ,0, 'JUNIOR TECHNICIAN'
                                ,'JUNIOR SUPERVISOR')
                     ,9, 'ANCILLORY STAFF'
                     ,10, DECODE (MOD (ROWNUM, 2)
                                 ,0, 'INTERN'
                                 ,'CASUAL WORKER'))
                      job_description
              FROM standard_payment_dates, employees)
     SELECT *
       FROM q1
   ORDER BY paydate;

BEGIN
   DBMS_STATS.gather_table_stats (
      ownname      => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname      => 'PAYMENTS'
     ,method_opt   => 'FOR ALL COLUMNS SIZE 1');
END;
/

CREATE UNIQUE INDEX payments_pk
   ON payments (payment_id);

ALTER TABLE payments
   ADD  CONSTRAINT payments_pk PRIMARY KEY (payment_id);

CREATE INDEX payments_ix1
   ON payments (paygrade, job_description);

SELECT *
  FROM payments
 WHERE special_flag = 'Y';

| Id  | Operation         | Name     | Rows   |
| --- | ----------------- | -------- | ------ |
|   0 | SELECT STATEMENT  |          |   1200 |
|   1 |  TABLE ACCESS FULL| PAYMENTS |   1200 |

EXEC tstats.adjust_column_stats_v1(p_table_name=>'PAYMENTS');

SELECT *
  FROM payments
 WHERE special_flag = 'Y';

| Id  | Operation         | Name     | Rows  |
| --- | ----------------- | ------- | ----- |
|   0 | SELECT STATEMENT  |          |     1 |
|   1 |  TABLE ACCESS FULL| PAYMENTS |     1 |
```

## 对 Listing 20-2 的解释

Listing 20-2 创建了一个名为 `PAYMENTS` 的表，我将在本章中用它来说明几个要点。`PAYMENTS` 表包含一个 `SPECIAL_FLAG` 列，其值为 `Y` 或 `NULL`。在收集统计信息后，我们可以立即看到，对于在 `SPECIAL_FLAG` 上包含等式谓词的查询，其估计的基数是绝对准确的。然而，一旦调用过程 `TSTATS.ADJUST_COLUMN_STATS_V1` 来移除 `HIGH_VALUE` 和 `LOW_VALUE` 统计列，我们就会看到 CBO 感到困惑，并将基数减少到 0（在显示中四舍五入为 1，通常如此）。这可能是一个错误，因为在 12.1.0.1 中无法观察到此行为。

解决此问题的一种方法是让统计调整过程避免处理 `NUM_DISTINCT` 为 1 的列。这在 Listing 20-2 所示的示例中肯定有效。然而，一个唯一值的列也可能是一个基于时间的列，如果是这样，保留 `HIGH_VALUE` 和 `LOW_VALUE` 会产生问题，正如我们稍后在查看 `DBMS_STATS.COPY_TABLE_STATS` 时将要展示的那样。

处理 `NUM_DISTINCT` 为 1 的另一种方法是更改 `NUM_DISTINCT` 的值！Listing 20-3 展示了过程 `TSTATS.ADJUST_COLUMN_STATS_V2`，它解决了这个问题。

## Listing 20-3. 调整 `NUM_DISTINCT` 和 `DENSITY` 以修复基数问题

```sql
CREATE OR REPLACE PACKAGE tstats
AS
-- Other procedures not shown
   PROCEDURE adjust_column_stats_v2 (
      p_owner all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name all_tab_col_statistics.table_name%TYPE);
-- Other procedures not shown
END tstats;
/

CREATE OR REPLACE PACKAGE BODY tstats
AS
-- Other procedures not shown
  PROCEDURE adjust_column_stats_v2 (
      p_owner all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name all_tab_col_statistics.table_name%TYPE)
   AS
      CURSOR c1
      IS
         SELECT *
           FROM all_tab_col_statistics
          WHERE     owner = p_owner
                AND table_name = p_table_name
                AND last_analyzed IS NOT NULL;

      v_num_distinct all_tab_col_statistics.num_distinct%TYPE;
   BEGIN
      FOR r IN c1
      LOOP
         DBMS_STATS.delete_column_stats (ownname         => r.owner
                                        ,tabname         => r.table_name
                                        ,colname         => r.column_name
                                        ,cascade_parts   => TRUE
                                        ,no_invalidate   => TRUE
                                        ,force           => TRUE);

         IF r.num_distinct = 1
         THEN
            v_num_distinct := 1 + 1e-14;
         ELSE
            v_num_distinct := r.num_distinct;
         END IF;
```



