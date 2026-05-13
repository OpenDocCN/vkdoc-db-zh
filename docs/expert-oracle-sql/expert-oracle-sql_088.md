# TABLE 和 XMLTABLE

SQL 语言参考手册中的函数列表包含`XMLTABLE`。由于即将明了的原因，我将不准确地将`XMLTABLE`称为一个“运算符”。`XMLTABLE`的一个兄弟是`TABLE`，但该“运算符”被奇怪地记录在手册中与`XMLTABLE`完全不同的部分。

`XMLTABLE`“运算符”有一个 Xquery“操作数”，并且该“操作”的结果是一个行源，可以像常规表或视图一样出现在`FROM`子句中。

我不是 XML 专家，但 Listing 10-24 是一个使用`XMLTABLE`的快速示例。

Listing 10-24. XMLTABLE 示例

```sql
CREATE TABLE xml_test (c1 XMLTYPE);

SELECT x.*
    FROM xml_test t
        ,XMLTABLE (
                  '//a'
                  PASSING t.c1
                  COLUMNS "a" CHAR (10) PATH '@path1'
                         ,"b" CHAR (50) PATH '@path2'
                         ) x;

| Id  | Operation          | Name     |

|   0 | SELECT STATEMENT   |          |
|   1 |  NESTED LOOPS      |          |
|   2 |   TABLE ACCESS FULL| XML_TEST |
|   3 |   XPATH EVALUATION|          |
```

`TABLE`“运算符”有一个“操作数”，该操作数是一个*嵌套表*。嵌套表操作数具有以下三种格式之一：

*   数据字典或经过`CAST`的 PL/SQL 嵌套表对象
*   返回嵌套表对象的函数。此类函数通常是管道函数
*   一个`CAST...MULTISET`表达式



这三种形式中的第二种最为常见，管道函数通常被称为 **“表函数”**。这种对“表函数”术语的常见用法，使得将 `TABLE` 和 `XMLTABLE` 关键字称为 **“操作符”** 而非函数更为方便。

在数据字典中使用嵌套表并不是一个很好的做法。这些表本身并非内联存储，因此你完全可以创建一个子表，并通过引用完整性约束将其链接到父表。这样，你至少可以有不访问父表而直接访问子数据的选项。

![image](img/sq.jpg) **注意** 我们将在第 13 章中看到，如今 CBO 可以执行表消除转换，从而消除任何对父表的不必要引用。

然而，如果你从前任那里继承了一些糟糕的嵌套表，清单 10-25 展示了如何查询它们。

清单 10-25. 查询嵌套表

```
CREATE TYPE order_item AS OBJECT
(
   product_name VARCHAR2 (50)
  ,quantity INTEGER
  ,price NUMBER (8, 2)
);
/

CREATE TYPE order_item_table AS TABLE OF order_item;
/

CREATE TABLE orders
(
   order_id      NUMBER PRIMARY KEY
  ,customer_id   INTEGER
  ,order_items   order_item_table
)
NESTED TABLE order_items
   STORE AS order_items_nt;

SELECT o.order_id
      ,o.customer_id
      ,oi.product_name
      ,oi.quantity
      ,oi.price
  FROM orders o, TABLE (o.order_items) oi;

| Id  | Operation                    | Name                    |
|---|---|---|
|   0 | SELECT STATEMENT             |                         |
|   1 |  NESTED LOOPS                |                         |
|   2 |   NESTED LOOPS               |                         |
|   3 |    TABLE ACCESS FULL         | ORDERS                  |
|   4 |    INDEX RANGE SCAN          | SYS_FK0000127743N00003$ |
|   5 |   TABLE ACCESS BY INDEX ROWID| ORDER_ITEMS_NT          |
```

如你所见，我们通过自动创建在隐藏列上的索引，建立了一个从新创建的父表 `ORDERS` 到嵌套表 `ORDER_ITEMS_NT` 的嵌套循环。其他的连接顺序和方法也是可能的。

![image](img/sq.jpg) **注意** 关于嵌套表中所有列的信息，包括列统计信息，可以在视图 `ALL_NESTED_TABLE_COLS` 中查看。该视图也显示了在其上构建索引的隐藏列 `NESTED_TABLE_ID`。

管道表函数是表操作符的一种更好的用途。清单 10-26 展示了一个常见例子。

清单 10-26. `DBMS_XPLAN.display` 示例

```
SELECT * FROM TABLE (DBMS_XPLAN.display);

| Id  | Operation                         | Name    |
|---|---|---|
|   0 | SELECT STATEMENT                  |         |
|   1 |  COLLECTION ITERATOR PICKLER FETCH| DISPLAY |
```

你可以看到 `COLLECTION ITERATOR PICKLER FETCH` 操作，它表示调用管道函数来检索行。实际上，这些行是批量处理的，因此你不会在检索每一行时都在调用 SQL 语句和管道函数之间进行上下文切换。

使用表操作符的第三种也是最后一种方式是结合子查询。清单 10-27 展示了一种可能不太熟悉的语法，它需要使用用户定义的类型才能工作。

清单 10-27. `CAST MULTISET` 示例

```
SELECT *
  FROM TABLE (CAST (MULTISET (SELECT 'CAMERA' product, 1 quantity, 1 price
                                 FROM DUAL
                           CONNECT BY LEVEL <= 3) AS order_item_table)) oi;

| Id  | Operation                          | Name |
|---|---|---|
|   0 | SELECT STATEMENT                   |      |
|   1 |  COLLECTION ITERATOR SUBQUERY FETCH|      |
|   2 |   COUNT                            |      |
|   3 |    CONNECT BY WITHOUT FILTERING    |      |
|   4 |     FAST DUAL                      |      |
```



这里我查询了 `DUAL` 表来生成一个包含三行的集合，然后使用我在清单 10-25 中创建的 `ORDER_ITEM_TYPE` 将结果强制转换为嵌套表。接着，`TABLE` 操作符允许我将此嵌套表视为行源。查询结果与最初从 `DUAL` 表 `SELECT` 的结果相同。

清单 10-27 中的执行计划展示了第 2、3、4 行的子查询，而第 1 行的 `COLLECTION ITERATOR SUBQUERY FETCH` 操作是由 `TABLE` 操作符生成的。

这一切看起来非常复杂且没有必要。为什么非要费尽周折将一堆行构建成嵌套表，然后再将它们展开回行呢？然而，正如我们将在第 17 章中看到的，将此技术与我之前提到的左外连接结合使用，在某些情况下会非常有用。

## 集群访问

为了本章内容的完整性，我需要提及集群（参见表 10-6）。

表 10-6. 集群访问操作

| 操作 | 提示 |
| --- | --- |
| `TABLE ACCESS HASH` | `HASH` |
| `TABLE ACCESS CLUSTER` | `CLUSTER` |

让我来讲解这个例子。清单 10-28 是本章的最后一个清单。

清单 10-28. 集群访问方法

```
CREATE CLUSTER cluster_hash
(
  ck                              INTEGER
)
HASHKEYS 3
HASH IS ck;

CREATE TABLE tch1
(
   ck   INTEGER
  ,c1   INTEGER
)
CLUSTER cluster_hash ( ck );

CREATE CLUSTER cluster_btree
(
  ck                              INTEGER,
  c1                              INTEGER
);

CREATE INDEX cluster_btree_ix
   ON CLUSTER cluster_btree;

CREATE TABLE tc2
(
   ck   INTEGER
  ,c1   INTEGER
)
CLUSTER cluster_btree ( ck, c1 );

CREATE TABLE tc3
(
   ck   INTEGER
  ,c1   INTEGER
)
CLUSTER cluster_btree ( ck, c1 );

SELECT /*+ hash(tch1) index(tc2) cluster(tc3)*/
            *
        FROM tch1, tc2, tc3
       WHERE     tch1.ck = 1
             AND tch1.ck = tc2.ck
             AND tch1.c1 = tc2.c1
             AND tc2.ck = tc3.ck
             AND tc2.c1 = tc3.c1;

| Id  | Operation              | Name             |

|   0 | SELECT STATEMENT       |                  |
|   1 |  NESTED LOOPS          |                  |
|   2 |   NESTED LOOPS         |                  |
|   3 |    TABLE ACCESS HASH   | TCH1             |
|   4 |    TABLE ACCESS CLUSTER| TC2              |
|   5 |     INDEX UNIQUE SCAN  | CLUSTER_BTREE_IX |
|   6 |   TABLE ACCESS CLUSTER | TC3              |
```

清单 10-28 首先创建了一个哈希集群、一个 B 树集群和三个关联表。表 `TCH1` 是在单表哈希集群中创建的，表 `TC2` 和 `TC3` 是在 B 树集群中创建的。

清单 10-28 中查询的执行计划首先访问表 `TCH1`。因为我们在查询中包含了谓词 `TCH1.CK = 1`，并且 `CK` 是我们的哈希集群键，所以我们能够直接定位到包含 `TCH1` 中匹配行的块，而无需使用索引。对于 `TCH1` 中的每一个匹配行，我们使用 `CK` 和 `C1` 的值来访问我们在 `CLUSTER_BTREE` 上的集群索引。集群索引的特殊之处在于，每个键值只有一个条目，这就是为什么我们在第 5 行得到一个 `INDEX UNIQUE SCAN` 操作，即使 `TC2` 中可能有许多行匹配从 `TCH1` 提取的特定 `CK` 和 `C1` 值。

由于 `CK` 和 `C1` 列都在 `CLUSTER_BTREE` 中定义，并且也被指定为 `TC2` 和 `TC3` 的连接条件，所以我们知道在 `TC3` 中需要的行与 `TC2` 中的行是同位的。实际上，第 5 行的单次索引条目查找帮助我们找到所有匹配我们 `TCH1` 行的、在 `TC2` 和 `TC3` 中我们需要的行。

这一切看起来相当酷。事实上，集群在数据字典中被广泛使用。然而，我个人从未参与过任何在数据字典之外使用任何形式集群的生产环境 Oracle 数据库。

那么为什么它们如此不受欢迎呢？嗯，在绝大多数情况下，通过索引访问数据的主要开销在于访问表中的数据，而不是在于执行索引查找。这是因为索引结构本身很可能被缓存，而表数据则很可能没有。这倾向于限制哈希集群的好处。哈希集群也有些难以管理，因为你必须大致知道每个特定哈希键会匹配多少行。一方面，如果你设置的这个值太小，哈希链就会变长。另一方面，如果你设置得太大，你的集群就会不必要地庞大，最终导致比你实际需要的更多逻辑和/或物理 I/O。无论你将 `HASHKEYS` 参数设置得过大还是过小，你原本期望获得的性能收益都会迅速消失。

但是普通的 B 树集群呢？你不必担心设置 `HASHKEYS`，而且你仍然可以获得集群的主要性能收益，不是吗？

想象一下，你有一个名为 `ORDERS` 的表，其主键是 `ORDER_ID`，还有一个 `ORDER_ITEMS` 表，其中有一个参照完整性约束将其链接到父表。如果能进行一次单块读取，同时获取 `ORDERS` 表中的一行及其在 `ORDER_ITEMS` 表中对应的行，岂不美哉？这肯定比分别从两个表中检索数据做两次 I/O 要好，对吧？

嗯，是的。然而，仅检索单个订单数据的查询无论如何都会很快完成，你可能注意不到性能的提升。另一方面，如果你正在运行一个查看大量订单及其相关明细项的报表，无论如何你都会读取大量块，所以数据如何分布在这些块中就无关紧要了。

你可能有一个拥有成百上千交互用户的系统，因此 `ORDERS` 和 `ORDER_ITEMS` 表可能非常庞大，无法缓存在 SGA 中，所以你确实希望将这些 I/O 减半以提高处理能力。

好的。不幸的是，这个场景有一个很大的问题：你无法对集群表进行分区。这是一个大的“陷阱”。分区为大型表提供了显著的管理和性能优势，很少有人愿意为了享受集群的好处而放弃所有这些优势。

但如果你因为某些原因（例如许可问题）无法使用分区，并且你经常执行我所描述的表之间的连接，那么集群可能看起来是个不错的选择。

即使在这些罕见的情况下，大多数人最终也会选择对数据进行反规范化或寻找其他变通方法，而不是使用像集群这样的东西——无论对错，它被视为一个巨大的未知领域。在本书中，我将不再提及集群。

## 总结

恐怕即使是对访问方法的冗长讨论也是不完整的。我们没有讨论 `LOB` 对象的访问、`REF` 数据类型的使用或域索引的使用。但即便如此，本章已经涵盖了你在商业应用中绝大多数 SQL 语句里可能遇到的所有访问方法。

既然已经介绍了在查询中访问单个行源的各种方法，现在该将我们的注意力转向将这些行源连接在一起这个主题了。继续前进到第 11 章。

