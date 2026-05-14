# 列使用历史记录

查询优化器在生成新的执行计划时，会跟踪 `WHERE` 子句中引用了哪些列。然后，这些信息会不时地存储在数据字典表 `col_usage$` 中。⁷ 通过如下查询（可在脚本 `col_usage.sql` 中找到），可以知道哪些列在 `WHERE` 子句中被引用过，以及用于何种谓词。`timestamp` 列指示上次使用的时间。其他列是使用次数（注意，不是执行次数）的计数。从未在 `WHERE` 子句中被引用的列在表 `col_usage$` 中没有行，因此输出中除 `name` 外的所有列均为 `NULL`。

```sql
SQL> SELECT c.name, cu.timestamp,
  2         cu.equality_preds AS equality, cu.equijoin_preds AS equijoin,
  3         cu.nonequijoin_preds AS noneequijoin, cu.range_preds AS range,
  4         cu.like_preds AS "LIKE",cu.null_preds AS "NULL"
  5  FROM sys.col$ c, sys.col_usage$ cu, sys.obj$ o, sys.user$ u
  6  WHERE c.obj# = cu.obj# (+)
  7  AND c.intcol# = cu.intcol# (+)
  8  AND c.obj# = o.obj#
  9  AND o.owner# = u.user#
 10  AND o.name = 'T'
 11  AND u.name = user
 12  ORDER BY c.col#;
```

```
NAME       TIMESTAMP EQUALITY EQUIJOIN NONEEQUIJOIN  RANGE   LIKE   NULL
---------- --------- -------- -------- ------------ ------ ------ ------
ID         14-MAY-07       14       17            0      0      0      0
VAL1       14-MAY-07       15        0            0      0      0      0
VAL2
VAL3       14-MAY-07        7       18            0      0      0      0
PAD        13-MAY-07        0        7            0      1      0      0
```

`dbms_stats` 包使用此信息来确定哪些列适合创建直方图。

