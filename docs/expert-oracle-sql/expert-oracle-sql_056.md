# 代码清单 8-7 第 22 行的 `MERGE` 提示展示了应用于查询块 `SEL$1` 的简单视图合并查询转换的用法。我们将在第 13 章介绍 CBO 可用的大部分查询转换功能，而语句中使用的所有非强制性查询转换都将在执行计划大纲中通过诸如 `MERGE` 之类的提示反映出来。

## 注释

代码清单 8-7 第 13 行的 `NO_ACCESS` 提示对我来说有点神秘。它记录了 CBO 没有合并某个查询块这一事实。如果你在代码中添加它，它实际上并不会产生任何效果；即使你从大纲中移除它，似乎也无关紧要。它看起来只是一个注释。

## 最终状态优化提示

代码清单 8-7 第 3 至 12 行、第 14 和 15 行的提示看起来`几乎`像是 SQL 语句中常见的内嵌提示。它们控制连接顺序、连接方法和访问方法。我说`几乎`，是因为它们包含了全局提示语法和完全限定的对象别名，这些虽然合法，但在 SQL 语句的源代码中很少见到。我们将在本章后面更详细地探讨全局提示。

现在我们已经对代码清单 8-7 中的所有大纲提示进行了分类。让我们继续看`DBMS_XPLAN`输出的其他部分。

## 被窥探的绑定变量

当调用 `DBMS_XPLAN.DISPLAY_CURSOR` 时，`PEEKED_BINDS` 部分会紧跟在 `OUTLINE` 部分之后。代码清单 8-6 无法用于演示 `PEEKED_BINDS`，原因有二：
*   该语句没有任何绑定变量。
*   该语句使用了数据库链接，因此在 12cR1 之前的版本中，不会执行任何绑定变量窥探。

因此，为了演示 `PEEKED_BINDS` 部分，我将使用代码清单 8-9 中所示的脚本。

代码清单 8-9. 展示被窥探绑定变量的本地数据库查询

```
DECLARE
   dummy   NUMBER := 0;
BEGIN
   FOR r IN (WITH q1 AS (SELECT /*+ TAG1*/
                               * FROM t1)
             SELECT *
               FROM (SELECT *
                       FROM q1, t2
                     WHERE q1.c1 = t2.c2 AND c1 > dummy
                     UNION ALL
                     SELECT *
                       FROM t3, t4
                      WHERE t3.c3 = t4.c4 AND c3 > dummy)
                   ,t3
              WHERE c1 = c3)
   LOOP
      NULL;
   END LOOP;
END;
/

SET LINES 200 PAGES 0

SELECT p.*
  FROM v$sql s
      ,TABLE (
          DBMS_XPLAN.display_cursor (s.sql_id
                                    ,s.child_number
                                    ,'BASIC +PEEKED_BINDS')) p
 WHERE sql_text LIKE 'WITH%SELECT /*+ TAG1*/%';

EXPLAINED SQL STATEMENT:

WITH Q1 AS (SELECT /*+ TAG1 */ * FROM T1) SELECT * FROM (SELECT * FROM
Q1, T2 WHERE Q1.C1 = T2.C2 AND C1 > :B1 UNION ALL SELECT * FROM T3, T4
WHERE T3.C3 = T4.C4 AND C3 > :B1 ) ,T3 WHERE C1 = C3

Plan hash value: 2648033210

| Id  | Operation             | Name |

|   0 | SELECT STATEMENT      |      |
|   1 |  HASH JOIN            |      |
|   2 |   TABLE ACCESS FULL   | T3   |
|   3 |   VIEW                |      |
|   4 |    UNION-ALL          |      |
|   5 |     HASH JOIN         |      |
|   6 |      TABLE ACCESS FULL| T1   |
|   7 |      TABLE ACCESS FULL| T2   |
|   8 |     HASH JOIN         |      |
|   9 |      TABLE ACCESS FULL| T3   |
|  10 |      TABLE ACCESS FULL| T4   |

Peeked Binds (identified by position):

1 - :B1 (NUMBER): 0
   2 - :B1 (NUMBER, Primary=1)
```

与本章前面那些暗示先调用 `EXPLAIN PLAN` 然后调用 `DBMS_XPLAN.DISPLAY` 的示例不同，代码清单 8-9 实际运行了一个语句，然后调用 `DBMS_XPLAN.DISPLAY_CURSOR` 来生成输出。你会看到我在语句的注释中包含了字符串 `TAG1`。这个注释对 PL/SQL 编译器来说像是一个提示（当然它并不是），因此注释没有被移除。运行语句并丢弃返回的行后，我使用 `left lateral join` 将 `DBMS_XPLAN.DISPLAY_CURSOR` 与 `V$SQL` 连接起来以识别 `SQL_ID`。我们将在第 11 章讨论连接方法时介绍左横向连接。调用 `DBMS_XPLAN.DISPLAY_CURSOR` 的输出也显示在代码清单 8-9 中。

我们可以看到，当查询被解析时，会查看绑定变量 `:B1` 的值，因为该值可能会影响执行计划的选择。该绑定变量是 PL/SQL 由于我命名为 `DUMMY` 的 PL/SQL 变量的使用而添加的。`DUMMY` 被使用了两次，但不需要窥探它的值两次。这就是输出中 `Primary=1` 部分的含义——它表示第二次使用的绑定变量的值取自第一次。

值得再次强调的是，绑定变量只在语句被解析时被窥探。除少数例外情况（如自适应游标共享），使用不同绑定变量值对同一语句的第二次及后续执行将不会导致更多的窥探，并且此类调用也不会更新 `DBMS_XPLAN.DISPLAY_CURSOR` 的输出。

## 谓词信息

`DBMS_XPLAN` 函数调用的 `PREDICATE` 部分出现在任何 `PEEKED_BINDS` 部分之后。我们已经在第 3 章介绍过过滤谓词，因为 `DBMS_XPLAN` 函数的默认输出级别是 `TYPICAL`，而 `PREDICATE` 部分包含在 `TYPICAL` 级别中。为完整起见，代码清单 8-10 展示了代码清单 8-6 中语句的过滤谓词。

代码清单 8-10. 代码清单 8-6 中查询的谓词信息

```
Predicate Information (identified by operation id):

4 - access("C1"="C3")
   8 - access("T1"."C1"="T2"."C2")
  11 - access("T3"."C3"="T4"."C4")
```

此例中的访问谓词与三个 `HASH JOIN` 操作相关。

## 列投影

`DBMS_XPLAN` 输出的下一个显示部分展示了由各个行源操作返回的列。这些列可能与程序员指定的不同，因为在某些情况下 CBO 可以判定它们不是必需的，然后将其消除。代码清单 8-11 显示了代码清单 8-6 中语句的列投影数据。

代码清单 8-11. 代码清单 8-6 中查询的列投影信息

```
Column Projection Information (identified by operation id):

1 - SYSDEF[4], SYSDEF[16336], SYSDEF[1], SYSDEF[112], SYSDEF[16336]
   2 - (#keys=3) COUNT(*)[22]
   3 - (#keys=0) COUNT(*)[22]
   4 - (#keys=1) "C3"[NUMBER,22], "C1"[NUMBER,22]
   5 - (rowset=200) "C3"[NUMBER,22]
   6 - "C1"[NUMBER,22]
   7 - STRDEF[22]
   8 - (#keys=1) "T1"."C1"[NUMBER,22], "T2"."C2"[NUMBER,22]
   9 - "T1"."C1"[NUMBER,22]
  10 - (rowset=200) "T2"."C2"[NUMBER,22]
  11 - (#keys=1) "T3"."C3"[NUMBER,22], "T4"."C4"[NUMBER,22]
  12 - (rowset=200) "T3"."C3"[NUMBER,22]
  13 - (rowset=200) "T4"."C4"[NUMBER,22]
```



本节在你评估内存需求时可能很有用，因为你不仅能看到列的名称，还能看到数据类型和列的大小。操作 4、8 和 11 包含文本 `(#keys=1)`。这表示连接中使用了一列。连接或排序中使用的列总是列在首位，例如，操作 11 在哈希连接中使用了 `T3.C3`。

我发现操作 3 的关联输出很有趣。文本 `(#keys=0)` 表明排序中没有使用任何列。正如我在第 3 章中解释的那样，使用 `SORT AGGREGATE` 时并不存在排序！你还会看到操作 7（即 `UNION-ALL` 运算符）将其标识符名称显示为 `STRDEF`。我不知道 `STRDEF` 代表什么，但它用于内部生成的列名。

![image](img/sq.jpg) **注意** 集合查询块中的 `ORDER BY` 子句必须指定位置或别名，而不是显式表达式。清单 8-6 中加粗的注释行给出了一个示例。

## 远程 SQL

由于清单 8-6 使用了数据库链接，因此需要将部分语句发送到链接的远端进行处理。清单 8-12 显示了此 SQL。

清单 8-12。清单 8-6 中查询的远程 SQL 信息

```
远程 SQL 信息（通过操作 ID 标识）：

9 - SELECT /*+ */ "C1" FROM "BOOK"."T1" "T1" (accessing 'LOOPBACK' )
```

你会看到，发送到“远程”数据库的查询文本已被修改，以引用标识符、指定模式（在我的例子中是 `BOOK`）并添加对象别名。如果原始语句中提供了提示，那么相应的子集也会包含在发送到“远程”数据库的查询中。当然，这个远程 SQL 语句会有自己的执行计划，但如果你需要检查这个计划，你必须在远程数据库上单独进行。

## 自适应计划

自适应执行计划是 `12cR1` 的新功能，可能是该版本新自适应特性中最著名的。除了一条注释外，只有当 `ADAPTIVE` 被显式添加到 `DBMS_XPLAN` 调用的格式参数中时，才会显示有关执行计划调整的信息。与文档描述相反，`ADAPTIVE` 是一种细粒度控制，而非格式级别。

自适应计划的思想如下：你有两个可能的计划。你从其中一个开始，如果那个计划效果不佳，你就在中途切换到另一个。虽然我们将在本章后面讨论并行查询时提到的 `混合并行查询分发` 与此非常相似，但截至 `12.1.0.1`，唯一真正自适应的机制是表连接。

自适应连接的工作方式如下：你从其中一个连接表中找到一些匹配的行，并查看得到多少行。如果得到的行数很少，你可以使用嵌套循环连接访问第二个被连接的表。如果看起来会得到很多行，那么在某个时刻，你会放弃嵌套循环连接，转而开始使用哈希连接。

我们将在第 11 章讨论嵌套循环连接和哈希连接的细节，但现在我们只需要专注于理解执行计划试图告诉我们的信息。

并非所有连接都是自适应的。事实上，大多数都不是，因此在我的演示中，我不得不依赖 `SQL 调优指南` 中提供的例子。清单 8-13 展示了我对 `SQL 调优指南` 中自适应连接示例的改编。

清单 8-13。自适应执行计划的使用

```
ALTER SYSTEM FLUSH SHARED_POOL;

SET LINES 200 PAGES 0
VARIABLE b1 NUMBER;
EXEC :b1 := 15;

SELECT product_name
  FROM oe.order_items o, oe.product_information p
 WHERE unit_price = :b1 AND o.quantity > 1 AND p.product_id = o.product_id;

SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (format => 'TYPICAL +ADAPTIVE'));

EXEC :b1 := 1000;

SELECT product_name
  FROM oe.order_items o, oe.product_information p
 WHERE unit_price = :b1 AND o.quantity > 1 AND p.product_id = o.product_id;

SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (format => 'TYPICAL'));

EXEC :b1 := 15;

SELECT product_name
  FROM oe.order_items o, oe.product_information p
 WHERE unit_price = :b1 AND o.quantity > 1 AND p.product_id = o.product_id;

SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (format => 'TYPICAL'));
```

在查看清单 8-13 代码的输出之前，让我们先看看它的作用。清单 8-13 首先刷新共享池以清除任何先前的运行，然后设置一个 `SQL*Plus` 变量 `B1`，该变量初始化为 15，并用作基于 `OE` 示例模式的查询中的绑定变量。查询返回后，使用 `DBMS_XPLAN.DISPLAY_CURSOR` 显示执行计划。清单 8-13 在调用 `DBMS_XPLAN.DISPLAY_CURSOR` 时指定了 `ADAPTIVE`，以显示我们感兴趣的执行计划部分。

然后，`SQL*Plus` 变量 `B1` 的值从 15 改为 1000，查询第二次运行。最后，`B1` 被改回 15，查询第三次也是最后一次运行。在查询第二次和第三次运行后，我们在调用 `DBMS_XPLAN.DISPLAY_CURSOR` 时不再列出执行计划的自适应部分。

清单 8-14 显示了清单 18-13 的输出。

清单 8-14。自适应查询多次运行的执行计划

```
System altered.
 PL/SQL procedure successfully completed.
Screws <B.28.S>
<cut>
Screws <B.28.S>

13 rows selected.
SQL_ID  g4hyzd4v4ggm7, child number 0

SELECT product_name   FROM oe.order_items o, oe.product_information p
WHERE unit_price = :b1 AND o.quantity > 1 AND p.product_id =
o.product_id

Plan hash value: 1553478007

|   Id  | Operation                     | Name                   | Rows  | Time     |

|     0 | SELECT STATEMENT              |                        |       |          |
|  *  1 |  HASH JOIN                    |                        |     4 | 00:00:01 |
|-    2 |   NESTED LOOPS                |                        |       |          |
|-    3 |    NESTED LOOPS               |                        |     4 | 00:00:01 |
|-    4 |     STATISTICS COLLECTOR      |                        |       |          |
|  *  5 |      TABLE ACCESS FULL        | ORDER_ITEMS            |     4 | 00:00:01 |
|- *  6 |     INDEX UNIQUE SCAN         | PRODUCT_INFORMATION_PK |     1 |          |
|-    7 |    TABLE ACCESS BY INDEX ROWID| PRODUCT_INFORMATION    |     1 | 00:00:01 |
|     8 |   TABLE ACCESS FULL           | PRODUCT_INFORMATION    |     1 | 00:00:01 |

Predicate Information(identified by operation id):

1 - access("P"."PRODUCT_ID"="O"."PRODUCT_ID")
   5 - filter(("UNIT_PRICE"=:B1 AND "O"."QUANTITY">1))
   6 - access("P"."PRODUCT_ID"="O"."PRODUCT_ID")

Note

- this is an adaptive plan (rows marked '-' are inactive)

33 rows selected.
 PL/SQL procedure successfullycompleted.
no rows selected.
SQL_ID  g4hyzd4v4ggm7, child number 1

SELECT product_name   FROM oe.order_items o, oe.product_information p
WHERE unit_price = :b1 AND o.quantity > 1 AND p.product_id =
o.product_id

Plan hash value: 1255158658

| Id  | Operation                    | Name                   | Rows  | Time     |
```



