# 设置与发布待处理统计信息

以下脚本展示了如何设置、测试并发布待处理的统计信息。首先执行命令：
```sql
ALTER SESSION SET optimizer_use_pending_statistics=TRUE;
```
会话已更改。

接着执行查询：
```sql
SELECT *
  FROM statement_part
 WHERE transaction_amount = 8;
```
执行计划如下：
```
---------------------------------------------------------------------------------
| Id  | Operation           | Name           | Rows  | Time     | Pstart| Pstop |
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |                |   125 | 00:00:01 |       |       |
|   1 |  PARTITION RANGE ALL|                |   125 | 00:00:01 |     1 |     2 |
|*  2 |   TABLE ACCESS FULL | STATEMENT_PART |   125 | 00:00:01 |     1 |     2 |
---------------------------------------------------------------------------------
```

然后发布待处理的统计信息：
```sql
BEGIN
    DBMS_STATS.publish_pending_stats (
       ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
      ,tabname   => 'STATEMENT_PART');
END;
/
```
PL/SQL 过程已成功完成。

之后禁用待处理统计信息的使用：
```sql
ALTER SESSION SET optimizer_use_pending_statistics=FALSE;
```
会话已更改。

再次运行查询以验证效果：
```sql
SELECT *
  FROM statement_part
 WHERE transaction_amount = 8;
```
执行计划显示：
```
---------------------------------------------------------------------------------
| Id  | Operation           | Name           | Rows  | Time     | Pstart| Pstop |
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |                |   125 | 00:00:01 |       |       |
|   1 |  PARTITION RANGE ALL|                |   125 | 00:00:01 |     1 |     2 |
|*  2 |   TABLE ACCESS FULL | STATEMENT_PART |   125 | 00:00:01 |     1 |     2 |
---------------------------------------------------------------------------------
```

关闭自动跟踪：
```sql
SET AUTOTRACE OFF
```

最后，查询 `ALL_TAB_HISTOGRAMS` 和 `ALL_TAB_HISTGRM_PENDING_STATS` 视图来观察直方图的状态变化。

查询 `ALL_TAB_HISTOGRAMS`：
```sql
SELECT endpoint_number, endpoint_value
  FROM all_tab_histograms
 WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
       AND table_name = 'STATEMENT_PART'
       AND column_name = 'TRANSACTION_AMOUNT';
```
结果如下：
```
ENDPOINT_NUMBER ENDPOINT_VALUE
--------------- --------------
           1000              8
           1600       10000000
            600      -10000000
```
已选择 3 行。

查询 `ALL_TAB_HISTGRM_PENDING_STATS`：
```sql
SELECT endpoint_number, endpoint_value
  FROM all_tab_histgrm_pending_stats
 WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
       AND table_name = 'STATEMENT_PART'
       AND column_name = 'TRANSACTION_AMOUNT';
```
未选择任何行。

