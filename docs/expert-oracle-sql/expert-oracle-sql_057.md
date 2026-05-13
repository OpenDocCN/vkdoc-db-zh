# Oracle 执行计划分析

## 第一个执行计划

```
|   0 | SELECT STATEMENT             |                        |       |          |
|   1 |  NESTED LOOPS                |                        |       |          |
|   2 |   NESTED LOOPS               |                        |    13 | 00:00:01 |
|*  3 |    TABLE ACCESS FULL         | ORDER_ITEMS            |     4 | 00:00:01 |
|*  4 |    INDEX UNIQUE SCAN         | PRODUCT_INFORMATION_PK |     1 |          |
|   5 |   TABLE ACCESS BY INDEX ROWID| PRODUCT_INFORMATION    |     4 | 00:00:01 |
```

## 谓词信息

谓词信息（通过操作 ID 识别）：

`3 - filter(("UNIT_PRICE"=:B1AND "O"."QUANTITY">1))`
`   4 - access("P"."PRODUCT_ID"="O"."PRODUCT_ID")`

## 注意事项

`Note`

- `statistics feedback used for this statement`
   - `this is an adaptive plan`

已选择 30 行。
PL/SQL 过程已成功完成。
Screws <B.28.S>
<cut>
Screws <B.28.S>

已选择 13 行。
`SQL_ID  g4hyzd4v4ggm7, child number 1`

`SELECT product_name   FROM oe.order_items o, oe.product_information p`
`WHERE unit_price = :b1 AND o.quantity > 1 AND p.product_id =`
`o.product_id`

`Plan hash value: 1255158658`

## 第二个执行计划

```
| Id  | Operation                    | Name                   | Rows  | Time     |
|   0 | SELECT STATEMENT             |                        |       |          |
|   1 |  NESTED LOOPS                |                        |       |          |
|   2 |   NESTED LOOPS               |                        |    13 | 00:00:01 |
|*  3 |    TABLE ACCESS FULL         | ORDER_ITEMS            |     4 | 00:00:01 |
|*  4 |    INDEX UNIQUE SCAN         | PRODUCT_INFORMATION_PK |     1 |          |
|   5 |   TABLE ACCESS BY INDEX ROWID| PRODUCT_INFORMATION    |     4 | 00:00:01 |
```

谓词信息（通过操作 ID 识别）：

`3 - filter(("UNIT_PRICE"=:B1 AND "O"."QUANTITY">1))`
`   4 - access("P"."PRODUCT_ID"="O"."PRODUCT_ID")`

`Note`

- `statistics feedback used for this statement`
   - `this is an adaptive plan`

已选择 30 行。

## 自适应执行计划讨论

第一个查询返回 13 行，超过了 CBO 估计的 4 行——通过仔细查看该语句的第一个子游标（child 0），可以看到使用了哈希连接。您可以通过在心理上移除显示被丢弃替代计划的带减号的行来看到这一点。当我们再次运行相同的 SQL 语句，这次使用一个导致没有行返回的绑定变量时，我们看到创建了一个新的子游标（child 1），并且使用了嵌套循环连接。这次我没有在`DBMS_XPLAN.DISPLAY_CURSOR`的格式参数中使用`ADAPTIVE`关键字，因此未使用的执行计划部分没有显示。

![image](img/sq.jpg) `Note` 我感谢 Randolf Geist，经过多次尝试，他说服我额外的子游标实际上是统计反馈的体现，而不是自适应计划。

这一切都非常令人印象深刻，但当我们第三次运行查询，使用原始绑定变量值时会发生什么？不幸的是，我们继续使用嵌套循环连接，没有切换回哈希连接！

我们已经看到 Oracle 数据库的一个重要特性：它适应一次但不再适应——绑定变量窥探。绑定变量窥探效果不佳，但希望自适应执行计划的这个相同缺点能很快得到修复。

## 结果缓存信息

在第 4 章中，我展示了运行时引擎如何利用结果缓存。如果结果缓存用于某个语句，那么所有`DBMS_XPLAN`函数的输出中都会放置一个显示其使用的部分。这个部分无法被抑制，但仅当与`DBMS_XPLAN.DISPLAY`一起使用时才包含有意义的信息，并且即使那样，也仅当级别为`TYPICAL`、`ALL`或`ADVANCED`时。清单 8-15 显示了一个简单示例。

清单 8-15. 来自简单语句的结果缓存信息

```
SET PAGES 0 LINES 300
EXPLAIN PLAN FOR SELECT /*+ RESULT_CACHE */ * FROM DUAL;

Explain complete.
```


