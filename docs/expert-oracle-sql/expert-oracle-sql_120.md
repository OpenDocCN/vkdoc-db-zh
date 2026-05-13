# 隐式数据类型转换

Oracle 数据库自动将数据从一种类型转换为另一种类型这一事实可能非常方便，特别是对于业务用户拼凑一个重要的即席查询而言，因为开发时间非常宝贵。然而，这种无声的转换可能会掩盖严重的性能问题。Listing 16-5 展示了隐式数据类型转换可能掩盖性能问题的一种方式。

Listing 16-5. 隐式数据类型转换

```
CREATE TABLE date_table
(
   mydate     DATE
  ,filler_1   CHAR (2000)
)
PCTFREE 0;

INSERT INTO date_table (mydate, filler_1)
       SELECT SYSDATE, RPAD ('x', 2000)
         FROM DUAL
   CONNECT BY LEVEL <= 1000;

BEGIN
   DBMS_STATS.gather_table_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'DATE_TABLE');
END;
/

CREATE INDEX date_index
   ON date_table (mydate);

SELECT mydate
  FROM date_table
 WHERE mydate = SYSTIMESTAMP;

| Id  | Operation         | Name       | Rows  | Cost (%CPU)|
|   0 | SELECT STATEMENT  |            |    10 |    70   (0)|
|*  1 |  TABLE ACCESS FULL| DATE_TABLE |    10 |    70   (0)|

Predicate Information (identified by operation id):
1 - filter(SYS_EXTRACT_UTC(INTERNAL_FUNCTION("MYDATE"))=SYS_EXTRACT_U
              TC(SYSTIMESTAMP(6)))

SELECT mydate
  FROM date_table
 WHERE mydate = SYSDATE;

| Id  | Operation        | Name       | Rows  | Cost (%CPU)|
|   0 | SELECT STATEMENT |            |     1 |     1   (0)|
|*  1 |  INDEX RANGE SCAN| DATE_INDEX |     1 |     1   (0)|

Predicate Information (identified by operation id):
1 - access("MYDATE"=SYSDATE@!)
```

Listing 16-5 创建了一个表 `DATE_TABLE`，其中包含一个类型为 `DATE` 的索引列。当我们尝试将该列与 `TIMESTAMP WITH TIMEZONE` 类型的值进行比较时，该列会被隐式转换为 `TIMESTAMP WITH TIMEZONE` 数据类型，从而无法使用索引。当我们的谓词指定了 `DATE` 类型的值时，索引就可以被使用。
这类编码问题可能看起来很晦涩；谁会用 `SYSTIMESTAMP` 而不是 `SYSDATE`？然而，当使用绑定变量时，这类问题实际上经常发生，特别是当同一个 PL/SQL 变量用于多个 SQL 语句时。同时请记住，SQL*Plus 的 `VARIABLE` 命令只支持有限的类型，而 `DATE` 并不包括在内。

