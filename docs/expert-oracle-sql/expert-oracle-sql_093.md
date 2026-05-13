# 半连接

事实证明，优化器可以将一些你并非显式编写的连接构造转换为连接。我们将从半连接开始，然后介绍反连接。半连接有两种类型：标准半连接和接受空值的半连接。让我们先看看标准半连接。

## 标准半连接

考虑清单 11-12，它查找有客户的国家：

清单 11-12. 半连接

```
SELECT *                             SELECT *
  FROM sh.countries                     FROM sh.countries
 WHERE country_id IN (SELECT         WHERE country_id IN (SELECT /*+ NO_UNNEST */
country_id FROM sh.customers);                            country_id FROM sh.customers);

--------------------------------------       --------------------------------------
| Id| Operation          | Name      |       | Id| Operation          | Name      |
--------------------------------------       --------------------------------------
| 0 | SELECT STATEMENT   |           |       | 0 | SELECT STATEMENT   |           |
| 1 |  HASH JOIN SEMI    |           |       | 1 |  FILTER            |           |
| 2 |   TABLE ACCESS FULL| COUNTRIES |       | 2 |   TABLE ACCESS FULL| COUNTRIES |
| 3 |   TABLE ACCESS FULL| CUSTOMERS |       | 3 |   TABLE ACCESS FULL| CUSTOMERS |
--------------------------------------       --------------------------------------
```

当运行时引擎执行清单 11-12 左侧的执行计划时，它会先像常规哈希连接一样为`COUNTRIES`创建一个内存中的哈希簇，然后处理`CUSTOMERS`以查找匹配项。然而，当运行时引擎找到匹配项时，它会从内存中的哈希簇中删除该条目，这样同一个国家就不会再次返回。

将子查询更改为连接的过程称为`子查询展开`，它是优化器转换的一个例子。我们将在第 13 章中更多地讨论优化器转换，但这里先做个预览。像所有优化器转换一样，`子查询展开`由两个提示控制。左侧不需要`UNNEST`提示，因为 CBO 自己选择了这种转换。在清单 11-12 的右侧，通过`NO_UNNEST`提示显式地抑制了这种转换。没有`子查询展开`的帮助，`CUSTOMERS`表将被扫描 23 次，即`COUNTRIES`中的每一行扫描一次。

嵌套循环半连接、合并半连接和哈希半连接都是可能的。

*   半连接的哈希连接输入可以像外连接一样交换，产生`哈希右半连接`操作。
*   你可以使用`UNNEST`和`NO_UNNEST`提示来控制`子查询展开`。
*   将局部提示应用于涉及`子查询展开`的执行计划可能有点麻烦，如前面清单 8-17 所示。
*   当你使用以下任何构造时，可能会看到半连接：
    *   `IN ( ... )`
    *   `EXISTS ( ... )`
    *   `= ANY ( ... ), > ANY( ... ), < ANY ( ... )`
    *   `<= ANY (...), >=ANY ( ... )`
    *   `= SOME ( ... ), > SOME( ... ), < SOME ( ... )`
    *   `<= SOME (...), >=SOME ( ... )`

## 接受空值的半连接

12cR1 版本扩展了半连接展开，以包含像清单 11-13 中的查询：

清单 11-13. 12cR1 中的接受空值半连接

```
SELECT *
  FROM t1
 WHERE    t1.c1 IS NULL OR
       EXISTS
             (SELECT *
                FROM t2
               WHERE t1.c1 = t2.c2);
-----------------------------------  -----------------------------------
| Id  | Operation          | Name |  | Id  | Operation          | Name |
-----------------------------------  -----------------------------------
|   0 | SELECT STATEMENT   |      |  |   0 | SELECT STATEMENT   |      |
|*  1 |  FILTER            |      |  |*  1 |  HASH JOIN SEMI NA |      |
|   2 |   TABLE ACCESS FULL| T1   |  |   2 |   TABLE ACCESS FULL| T1   |
|*  3 |   TABLE ACCESS FULL| T2   |  |   3 |   TABLE ACCESS FULL| T2   |
-----------------------------------  -----------------------------------

Predicate Information                  Predicate Information
---------------------                  ----------------------

1 - filter("T1"."C1" IS NULL OR     1 - access("T1"."C1"="T2"."C2")
       EXISTS (SELECT 0
    FROM "T2" "T2" WHERE "T2"."C2"=:B1))
   3 - filter("T2"."C2"=:B1)
```

像清单 11-13 所示的查询很常见。其思路是，当我们找到与`T2.C2`匹配的项*或者*`T1.C1`为`NULL`时，返回来自`T1`的行。在 12cR1 之前，这样的构造会阻止`子查询展开`，从而产生清单 11-13 左侧的执行计划。在 12cR1 中，引入了`哈希半连接 NA`和`哈希右半连接 NA`操作，它们确保除了匹配`T2.C2`的行之外，还返回`T1.C1`为`NULL`的来自`T1`的行，从而产生了清单 11-13 右侧所示的执行计划。

## 反连接

你可能看到的另一种`子查询展开`称为`反连接`。`反连接`查找不匹配项而不是匹配项，这正是我们目前一直在做的。与半连接一样，反连接也有两种类型：标准反连接和空值感知反连接。

## 标准反连接

清单 11-14 在客户表中查找无效国家。

清单 11-14. 标准反连接

```
 SELECT /*+ leading(c)*/ *                SELECT *
    FROM sh.customers c                       FROM sh.customers c
WHERE country_id                              WHERE country_id  NOT IN (SELECT country_id
NOT IN (SELECT country_id                                               FROM sh.countries);
        FROM sh.countries);

-----------------------------------------       -------------------------------------------
| Id| Operation          | Name         |       | Id| Operation            | Name         |
-----------------------------------------       -------------------------------------------
| 0 | SELECT STATEMENT   |              |       | 0 | SELECT STATEMENT     |              |
| 1 |  NESTED LOOPS ANTI |              |       | 1 |  HASH JOIN RIGHT ANTI|              |
| 2 |   TABLE ACCESS FULL| CUSTOMERS    |       | 2 |   INDEX FULL SCAN    | COUNTRIES_PK |
| 3 |   INDEX UNIQUE SCAN| COUNTRIES_PK |       | 3 |   TABLE ACCESS FULL  | CUSTOMERS    |
-----------------------------------------       -------------------------------------------
```

为了演示目的，我再次为清单 11-14 的左侧添加了提示。对于`CUSTOMERS`中的每一行，我们查找匹配的`COUNTRIES`，但仅当*未*找到匹配项时才返回`CUSTOMERS`中的一行。

嵌套循环反连接、合并反连接和哈希反连接都是可能的。

*   像往常一样，哈希连接的输入可以交换。清单 11-14 右侧显示的未加提示的执行计划使用了输入交换后的哈希连接。
*   与半连接一样，可以使用`UNNEST`和`NO_UNNEST`提示来控制展开。

当你使用以下任何构造时，可能会看到反连接：



