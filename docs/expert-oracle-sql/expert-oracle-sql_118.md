# 执行计划与分区表连接分析

## 执行计划对比

以下展示了两个不同的执行计划，对应于不同的查询方式：

```
|   0 | SELECT STATEMENT         |                  |
|   1 |  HASH JOIN               |                  |
|   2 |   PARTITION LIST ALL     |                  |
|   3 |    PARTITION HASH ALL    |                  |
|   4 |     TABLE ACCESS FULL    | ORDERS_PART      |
|   5 |   PARTITION REFERENCE ALL|                  |
|   6 |    TABLE ACCESS FULL     | ORDER_ITEMS_PART |
```

第一个查询语句如下，它使用了提示（hints）来获取一个易于理解的典型执行计划：

```sql
SELECT /*+ leading(o i) use_hash(i) no_swap_join_inputs(i) */
       *
  FROM orders_part o JOIN order_items_part i USING (order_date, order_id);
```

第二个执行计划对应于仅使用 `order_id` 列进行连接的查询：

```
| Id  | Operation            | Name             |
|   0 | SELECT STATEMENT     |                  |
|   1 |  PARTITION LIST ALL  |                  |
|   2 |   PARTITION HASH ALL |                  |
|   3 |    HASH JOIN         |                  |
|   4 |     TABLE ACCESS FULL| ORDERS_PART      |
|   5 |     TABLE ACCESS FULL| ORDER_ITEMS_PART |
```

## 分区表连接分析

Listing 15-6 创建了两个表：`ORDERS_PART` 和 `ORDER_ITEMS_PART`。`ORDERS_PART` 表按 `ORDER_DATE` 列表分区，并按 `ORDER_ID` 哈希子分区。它包含两个 `ORDER_DATE` 值的分区，每个 `ORDER_DATE` 有 16 个子分区，总计 32 个子分区。主键本应仅为 `ORDER_ID`，但我们做了一个常见且实用的妥协：将 `ORDER_DATE` 添加到主键中，以便可以使用本地索引来强制执行约束。

`ORDER_ITEMS_PART` 表按引用分区，这意味着在创建时它有 32 个分区，每个分区对应 `ORDER_ITEMS` 的一个子分区。每当向 `ORDERS_PART` 添加或删除一个分区时，`ORDER_ITEMS_PART` 将相应地添加或删除 16 个分区。

当我们连接这两个表时会发生什么？Listing 15-6 中的第一个查询仅使用 `ORDER_ID` 列连接两个表。从执行计划的第 5 行可以看出，对于 `ORDERS_PART` 的每个子分区，都会扫描 `ORDER_ITEMS_PART` 的全部 32 个子分区。这是因为 CBO 不知道特定的 `ORDER_ID` 值只出现在 `ORDER_ITEMS_PART` 的一个分区中。

当我们把 `ORDER_DATE` 添加到连接列列表中时，可以看到 `PARTITION REFERENCE ALL` 操作消失了，这意味着对于 `ORDER_ORDERS_PART` 的每个子分区，只扫描 `ORDER_ITEMS_PART` 的一个分区。在 Listing 15-6 的后一个查询中，由于两个表中用于定义分区和子分区的所有列都出现在连接条件中，成功地在 `ORDER_ITEMS_PART` 上实现了分区消除。

![image](img/sq.jpg) **提示** 引用分区的一个假定好处是，你可以使用仅存在于父表中的列对子表进行分区。然而，Listing 15-6 表明，如果你试图通过从 `ORDER_ITEMS_PART` 中省略 `ORDER_DATE` 列来规范化设计，你将发现无法使用完全分区智能连接（full partition-wise joins）。用于分区或子分区父表的所有列都应存在于子表中。

那么我们获得了什么好处？如果我们根本没有对表进行分区，我们将得到一个涉及整个表的大哈希连接，而现在我们有 32 个小的哈希连接。关键在于，这些小的哈希连接更有可能完全在内存中完成，而大型哈希连接则更有可能溢出到磁盘。

到目前为止，我们只看了串行分区智能连接。当我们考虑并行化时，分区智能连接的真正优势才变得显而易见。

## 并行分区连接

我在 Listings 11-17 和 11-18 中分别给出了完全和部分分区智能连接的示例，但为了回顾，Listing 15-7 获取了 Listing 15-6 中的后一个查询并以并行方式运行它。

**Listing 15-7. 并行完全分区智能连接**

```sql
SELECT /*+ leading(o i) use_hash(i) no_swap_join_inputs(i)
           parallel(16) PQ_DISTRIBUTE(I NONE NONE)*/
       *
  FROM orders_part o JOIN order_items_part i USING (order_date,order_id);

| Id  | Operation               | Name             |
|   0 | SELECT STATEMENT        |                  |
|   1 |  PX COORDINATOR         |                  |
|   2 |   PX SEND QC (RANDOM)   | TQ10000:         |
|   3 |    PX PARTITION HASH ALL|                  |
|   4 |     HASH JOIN           |                  |
|   5 |      TABLE ACCESS FULL  | ORDERS_PART      |
|   6 |      TABLE ACCESS FULL  | ORDER_ITEMS_PART |
```

## 数据库反规范化策略

我并非逻辑数据库设计方面的权威专家，但我相信逻辑数据库设计应该是规范化的。然而，一旦你有了初始整洁、规范化的逻辑模型，作为物理数据库设计过程的一部分，你通常需要对其进行反规范化，以获得良好的性能。本节描述了一些你可能需要反规范化数据库的方法，从物化视图开始。

### 物化视图

创建物化视图有两个原因。一个原因是为了自动化将数据从一个数据库复制到另一个数据库。第二个原因是为了性能目的对数据进行反规范化。性能是我们这里关注的重点，主要有两类与性能相关的物化视图：

*   **物化连接视图** 存储两个或多个表连接的结果。
*   **物化汇总视图** 存储聚合的结果。

当然，你也可以创建一个既连接表又执行聚合的物化视图。如果相同的连接和/或聚合操作频繁执行，那么创建物化视图可能是有意义的，这样工作只需执行一次。物化视图提供了一些很好的功能：

*   **查询重写（Query rewrite）**。这是我在 Listing 13-30 中展示的一个优化器转换示例。你不需要重写你的查询来利用物化视图。
*   **物化视图日志（Materialized view logs）**。这些可用于跟踪物化视图所基于的表的变化，以便可以将更改增量地应用到物化视图。
*   **提交时刷新（Refresh on commit）**。使用此选项，每当对物化视图所基于的表进行更改时，物化视图都会使用物化视图日志以增量方式更新。

这只是物化视图功能的一个简介。关于这个复杂功能的完整教程，我推荐《数据仓库指南》。《数据仓库指南》还描述了 `DIMENSION` 数据库对象，它们用于支持行内聚合值。

### 手动聚合与连接表

实际上，物化视图对于许多应用程序来说可能过于复杂。你不一定希望处理发生或未按预期发生的查询重写：你可以直接编写查询来访问物化视图数据。你不一定希望担心物化视图日志和提交时刷新可能对 DML 性能产生的重大性能影响：你可以自己刷新物化视图。



如果那是你的立场，那么你根本不需要使用 Oracle 物化视图功能。例如，在数据仓库应用中，你可以在每日加载流程结束时自行生成聚合数据。你可以将聚合数据加载到常规表中，供在线时段内的查询使用。

你也不一定需要同时维护数据的规范化和反规范化副本。例如，你可以将维度表中常用的列存储在事实表中，从而消除连接操作。这种反规范化甚至可能避免对星型转换的需求。但如果星型转换的高成本或复杂性是你的主要顾虑，那么有一个潜在的更优解决方案。我将在接下来介绍该方案。

请记住，反规范化会带来数据库不一致的风险，你应该考虑是否需要构建一些保护措施（例如对账作业）来缓解该风险。

## 位图连接索引

位图连接索引实际上是一种非常特殊的反规范化形式。它们主要用于避免星型转换。我必须说，我个人最初觉得位图连接索引的概念很难理解。让我分阶段、慢慢地阐述这个概念。

常规的单列 B 树索引获取列的每个不同值，并为拥有该值的每一行创建一个索引条目。被索引表中的匹配行通过`ROWID`来标识。位图索引类似，只是匹配行通过位图而非`ROWID`列表来标识。基于函数的位图索引与常规位图索引类似，只是位图与表达式关联，而非与表中显式存储的值关联。与基于函数的位图索引类似，位图连接索引中的索引值并不显式地作为被索引表中的一列出现，而是间接地与索引值相关联。我想是时候举个例子了。请看 Listing 15-8，其中展示了我们如何使用位图连接索引来改进 Listing 13-35 中的查询。

Listing 15-8. 位图连接索引

```
CREATE TABLE sales_2
PARTITION BY RANGE (time_id)
   (PARTITION pdefault VALUES LESS THAN (maxvalue))
AS
   SELECT *
     FROM sh.sales
    WHERE time_id = DATE '2001-10-18';

CREATE TABLE customers_2
AS
   SELECT * FROM sh.customers;

CREATE TABLE products_2
AS
   SELECT * FROM sh.products;

ALTER TABLE products_2
   ADD CONSTRAINT products_2_pk PRIMARY KEY
 (prod_id) ENABLE VALIDATE;

ALTER TABLE customers_2
   ADD CONSTRAINT customers_2_pk PRIMARY KEY
 (cust_id) ENABLE VALIDATE;

BEGIN
   DBMS_STATS.gather_table_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'SALES_2');
   DBMS_STATS.gather_table_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'CUSTOMERS_2');
   DBMS_STATS.gather_table_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'PRODUCTS_2');
END;
/

CREATE BITMAP INDEX sales_2_cust_ln_bjx
   ON sales_2 (c.cust_last_name)
   FROM customers_2 c, sales_2 s
   WHERE c.cust_id = s.cust_id
   LOCAL;

CREATE BITMAP INDEX sales_prod_category_bjx
   ON sales_2 (p.prod_category)
   FROM products_2 p, sales_2 s
   WHERE s.prod_id = p.prod_id
   LOCAL;

SELECT prod_name
      ,cust_first_name
      ,time_id
      ,amount_sold
  FROM customers_2 c, products_2 p, sales_2 s
 WHERE     s.cust_id = c.cust_id
       AND s.prod_id = p.prod_id
       AND c.cust_last_name = 'Everett'
       AND p.prod_category = 'Electronics';

| Id  | Operation                                     | Name                    |
|---|---|---|
|   0 | SELECT STATEMENT                              |                         |
|   1 |  NESTED LOOPS                                 |                         |
|   2 |   NESTED LOOPS                                |                         |
|   3 |    HASH JOIN                                  |                         |
|   4 |     TABLE ACCESS FULL                         | PRODUCTS_2              |
|   5 |     PARTITION RANGE SINGLE                    |                         |
|   6 |      TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES_2                 |
|   7 |       BITMAP CONVERSION TO ROWIDS             |                         |
|   8 |        BITMAP AND                             |                         |
|   9 |         BITMAP INDEX SINGLE VALUE             | SALES_2_CUST_LN_BJX     |
|  10 |         BITMAP INDEX SINGLE VALUE             | SALES_PROD_CATEGORY_BJX |
|  11 |    INDEX UNIQUE SCAN                          | CUSTOMERS_2_PK          |
|  12 |   TABLE ACCESS BY INDEX ROWID                 | CUSTOMERS_2             |
```

Listing 15-8 创建了`SH.SALES`、`SH.CUSTOMERS`和`SH.PRODUCTS`表的副本，而不是对 Oracle 提供的示例模式进行 DDL 更改。在创建任何位图连接索引之前，我们需要在`CUSTOMERS_2`和`PRODUCTS_2`表上创建已启用、已验证的主键约束。示例模式中的约束并未经过验证。

我们创建的第一个位图索引是`SALES_2_CUST_LN_BJX`。此索引标识`SALES_2`表中与`CUSTOMERS_2`表中每个`CUST_LAST_NAME`值相关联的所有行。此位图索引使我们能够识别`SALES_2`中与姓氏为 Everett 的客户相关联的所有行，而无需访问`CUSTOMERS_2`表。类似地，`SALES_PROD_CATEGORY_BJX`使我们能够识别`SALES_2`中与“电子产品”类别中的产品相关联的行，而无需访问`PRODUCTS_2`表。通过组合这些索引，我们可以找到`SALES_2`中同时匹配两组谓词的确切行集。

尽管位图连接索引与基于函数的索引相似，但它们也与物化连接视图非常相似。你可以创建一个物化连接视图，保存来自`CUSTOMERS_2`的`CUST_LAST_NAME`值和来自`SALES_2`的相关联的`ROWID`，这样你基本上就得到了与位图连接索引相同的东西。然而，位图连接索引比物化连接视图紧凑得多，使用效率也更高。

位图连接索引的最大问题（与任何位图索引一样）是，对与该索引关联的表进行 DML 操作的效率低下。实际上，在维度表上进行并行 DML 操作会将位图连接索引标记为不可用。因此，位图连接索引实际上只在数据仓库环境中才有用。

### 压缩

压缩数据不是一种反规范化形式。除了性能变化之外，使用或不使用压缩对用户或 SQL 开发者没有可见影响。接下来让我们看看索引和表压缩。我将在稍后提到大对象（LOB）时介绍 LOB 压缩。

一般来说，索引压缩和表压缩都会使 DML 操作成本更高，而使查询成本更低。这可能听起来违反直觉，因为查询需要解压缩数据。然而，正如我们将要看到的，索引和表的解压缩过程都非常廉价，并且即使是适度的逻辑 I/O 减少也将远远超过解压缩的成本。另一方面，如果在数据加载过程中看到过多的 CPU 使用，你可能需要尝试关闭压缩功能，看看是否有所不同。

## 索引压缩



## 索引压缩

索引压缩是 Oracle 中应该被广泛使用，却常被遗忘的特性之一。索引压缩是在数据块级别实现的。基本上，Oracle 使用一种称为`prefix table`的结构来存储数据块中被压缩列的唯一值。现在，索引条目中只存储指向`prefix table`条目的指针，而不是列值本身。Oracle 允许你指定要压缩的列（如果有的话），但你的选择是受限的：语法允许你从引导列开始指定要压缩的列数。例如，你不能选择压缩第二列而不压缩第一列。

想象一下，在一个非压缩索引中，每个数据块有 200 行。首先，如果被压缩列的唯一值通常在每个数据块中只有一两个，那么你的`prefix table`将只包含一两个条目，你就能够在压缩索引的一个数据块中存储远多于 200 行的数据。但是，假设在 200 个未压缩行中，你的引导列有 150 个唯一值。现在索引压缩就会成为一个问题，因为`prefix table`中的 150 个条目将消耗掉数据块的大部分空间，所有额外的开销很可能意味着你将无法再在数据块中容纳所有 200 行。

![image](img/sq.jpg) **提示** `ANALYZE INDEX . . . VALIDATE STRUCTURE`命令可用于确定索引建议压缩的列数。建议值可以在`INDEX_STATS`表的`OPT_CMPR_COUNT`列中找到。

不要认为索引是唯一的，它就不能从压缩中受益。请注意，清单 15-6 中创建的`ORDER_ITEMS_PART_PK`索引在创建时指定了`COMPRESS 1`，如果每天有大量订单行项目，这是一个完全合适的做法，因为如果不压缩它，每个`ORDER_DATE`值都会有大量的索引条目。除非每个订单中有大量行项目，否则压缩`ORDER_ID`列可能并不值得。

请注意，索引压缩实际上并不是真正的压缩。它实际上是一种 Oracle 所谓的*去重*形式。因此，查询无需进行真正的解压缩工作。查询只需遵循一个指针来获取`prefix table`中的列值即可。

## 表压缩

表压缩有几种类型：

*   **基本表压缩**。这是一种仅针对直接路径插入和表移动压缩表数据的压缩变体，不需要额外许可。
*   **OLTP 表压缩**。在 11gR1 中，此功能被称为*所有操作压缩*。基本上它与基本压缩相同，但与基本压缩不同的是，数据在非直接路径`INSERT`操作时也会被压缩。OLTP 表压缩需要 Oracle 数据库产品的个人版或企业版。对于企业版，需要单独购买 Oracle Advanced Compression。
*   **混合列压缩**。这是一个完全不同的功能，特定于 Exadata 和 Oracle 提供的存储。这个主题超出了本书的范围。

与索引压缩类似，基本压缩和 OLTP 表压缩都基于数据块内去重的概念。换句话说，Oracle 试图在每个数据块中只存储每个重复值一次。表压缩的实现比索引压缩稍微复杂一些。Oracle 可以重新排序列，对列的组合进行去重等等。关于表压缩内部机制的详细描述，请参阅 Jonathan Lewis 的文章，从这里开始：`http://allthingsoracle.com/compression-oracle-basic-table-compression/`。以下是一些需要注意的要点：

*   当你使用基本压缩创建表时，`PCTFREE`默认为 0。当你使用 OLTP 压缩创建表时，`PCTFREE`的默认值保持为 10。
*   当你更新表中的一行时，更新后的值不会被重新压缩。这可能就是为什么 11gR1 中 OLTP 压缩的名称“所有操作压缩”被匆忙更改的原因！
*   对于插入后被修改的行，你可能应该谨慎使用基本或 OLTP 压缩。这意味着 OLTP 压缩最好用于通过非直接路径`INSERT`语句加载、从不修改、然后被重复读取的数据。因此，在创建使用 OLTP 压缩的表时，你可能应该覆盖`PCTFREE`的值。清单 15-9 显示了基本语法。

清单 15-9。使用 OLTP 压缩创建表并覆盖 PCTFREE

```sql
    CREATE TABLE order_items_compress
    (
       order_id        INTEGER NOT NULL
      ,order_item_id   INTEGER NOT NULL
      ,order_date      DATE NOT NULL
      ,product_id      INTEGER
      ,quantity        INTEGER
      ,price           NUMBER
      ,CONSTRAINT order_items_compress_fk FOREIGN KEY
          (order_date, order_id)
           REFERENCES orders_part (order_date, order_id)
    )
    COMPRESS FOR OLTP
    PCTFREE 0;
```

![image](img/sq.jpg) **提示** 无论你是否使用表压缩，只要你确定表中的行绝不会因为`UPDATE`或`MERGE`操作而增大，就应该将`PCTFREE`设置为 0。

## LOB

术语`LOB`代表大对象。不仅仅是 LOB 很大——这个主题也很大！事实上，Oracle 有一本完整的手册《SecureFiles and Large Objects Developer's Guide》，专门讨论这个主题。我在这里只挑选几个关键特性。

*   你的 LOB 应该在具有自动段空间管理的表空间中创建为 SecureFiles。根据手册，另一种类型的 LOB，BasicFiles，将在未来的 Oracle 数据库版本中被弃用。
*   利用 Advanced Compression 选项，你可以使用 LOB 去重和 LOB 压缩。
*   你可以选择为读取缓存 LOB，或者为读写都缓存。默认情况下，根本不会在 SGA 中缓存 LOB 数据。
*   只要你不缓存 LOB 数据，你就可以选择不记录 LOB 数据。如果你利用这个选项，`INSERT`操作的性能将会提高，但在介质恢复时你将丢失 LOB 数据。如果你可以重新加载数据，这可能没关系。

## 总结

本章涵盖了物理数据库设计中对 SQL 性能重要的一些特性，但它并不自称是一本全面的指南。然而，通常 SQL 性能问题最好通过物理数据库设计更改来解决，希望本章能帮助你识别这些情况。

如果你从本章只记住一件事，那就是：不要盲目地添加一个索引来解决单个查询的性能问题——至少在没有经过一些思考的情况下不要这样做。

## 第 16 章

![image](img/frontdot.jpg)

## 重写查询

在第 1 章的开头，我解释了假设 CBO 会得出最优执行计划，而不管你如何编写 SQL 语句，是天真的想法。到目前为止，在本书中你已经看到许多例子，其中构造不当的 SQL 语句会导致 CBO 选择不合适的执行计划，仅在上一章就有相当多的例子。本章和下一章将更详细地探讨重写 SQL 语句的主题，但我不会假装给你一份问题场景的完整列表。



事实上，与几年前若撰写此章所需篇幅相比，本章内容已大幅精简；因为随着优化器转换数量的增加，引发性能问题的 SQL 结构数量反而减少了。尽管如此，仍存在许多情况：一个 SQL 语句在随意浏览者看来完全合理，但为获得可接受的性能，却需要被重写。

如今，重写 SQL 语句最常见的原因之一是解决排序性能不佳的问题，而下一章将专门深入探讨该主题。然而，在深入这一复杂主题之前，让我们先看几个本书前文未曾讨论过的简单 SQL 重写示例。

## 在谓词中使用表达式

一般来说，应尽量避免在用于谓词的列上使用表达式。我们在清单 14-1 中已经看到，在谓词中使用函数可能导致 CBO 的基数估算错误，但这并非唯一问题。清单 16-1 展示了表达式如何影响分区消除。

清单 16-1. 谓词中的表达式与分区消除

```sql
SELECT COUNT (DISTINCT amount_sold)
  FROM sh.sales
 WHERE EXTRACT (YEAR FROM time_id) = 1998;

| Id  | Operation              | Name     | Cost (%CPU)| Pstart| Pstop |

|   0 | SELECT STATEMENT       |          |   528   (4)|       |       |
|   1 |  SORT AGGREGATE        |          |            |       |       |
|   2 |   VIEW                 | VW_DAG_0 |   528   (4)|       |       |
|   3 |    HASH GROUP BY       |          |   528   (4)|       |       |
|   4 |     PARTITION RANGE ALL|          |   528   (4)|     1 |    28 |
|   5 |      TABLE ACCESS FULL | SALES    |   528   (4)|     1 |    28 |

SELECT COUNT (DISTINCT amount_sold)
  FROM sh.sales
 WHERE time_id >=DATE '1998-01-01' AND time_id < DATE '1999-01-01';

| Id  | Operation                   | Name     | Cost (%CPU)| Pstart| Pstop |

|   0 | SELECT STATEMENT            |          |   118   (6)|       |       |
|   1 |  SORT AGGREGATE             |          |            |       |       |
|   2 |   VIEW                      | VW_DAG_0 |   118   (6)|       |       |
|   3 |    HASH GROUP BY            |          |   118   (6)|       |       |
|   4 |     PARTITION RANGE ITERATOR|          |   113   (2)|     5 |     8 |
|   5 |      TABLE ACCESS FULL      | SALES    |   113   (2)|     5 |     8 |
```

清单 16-1 中的两个查询产生相同结果：`SH.SALES`表中 1998 年的`AMOUNT_SOLD`不同值的数量。第一个查询使用`EXTRACT`函数识别每笔销售的年份。第二个查询使用两个谓词生成日期范围。我们可以看到，后者能够执行分区消除，因此仅访问了 1998 年的分区。而前者不得不访问表中的所有分区，因为对列值应用函数阻碍了分区消除。

如果没有直接方法避免在用于谓词的列上使用函数，可能需要创建虚拟列或添加基于函数的索引，如清单 15-2 所示。清单 16-2 展示了另一种方法。

清单 16-2. 当函数应用于列时使用连接

```sql
SELECT COUNT (DISTINCT amount_sold)
  FROM sh.sales
 WHERE EXTRACT (MONTH FROM time_id) = 10;

| Id  | Operation              | Name     | Cost (%CPU)| Pstart| Pstop |

|   0 | SELECT STATEMENT       |          |   528   (4)|       |       |
|   1 |  SORT AGGREGATE        |          |            |       |       |
|   2 |   VIEW                 | VW_DAG_0 |   528   (4)|       |       |
|   3 |    HASH GROUP BY       |          |   528   (4)|       |       |
|   4 |     PARTITION RANGE ALL|          |   528   (4)|     1 |    28 |
|   5 |      TABLE ACCESS FULL | SALES    |   528   (4)|     1 |    28 |

SELECT COUNT (DISTINCT amount_sold)
  FROM sh.sales s, sh.times t
 WHERE EXTRACT (MONTH FROM t.time_id) = 10 AND t.time_id = s.time_id;

| Id  | Operation                             | Name           | Cost (%CPU)| Pstart| Pstop |

|   0 | SELECT STATEMENT                      |                |   102   (1)|       |       |
|   1 |  SORT AGGREGATE                       |                |            |       |       |
|   2 |   VIEW                                | VW_DAG_0       |   102   (1)|       |       |
|   3 |    HASH GROUP BY                      |                |   102   (1)|       |       |
|   4 |     NESTED LOOPS                      |                |            |       |       |
|   5 |      NESTED LOOPS                     |                |   101   (0)|       |       |
|   6 |       INDEX FULL SCAN                 | TIMES_PK       |     0   (0)|       |       |
|   7 |       PARTITION RANGE ITERATOR        |                |            |   KEY |   KEY |
|   8 |        BITMAP CONVERSION TO ROWIDS    |                |            |       |       |
|   9 |         BITMAP INDEX SINGLE VALUE     | SALES_TIME_BIX |            |   KEY |   KEY |
|  10 |      TABLE ACCESS BY LOCAL INDEX ROWID| SALES          |   101   (0)|     1 |     1 |
```

清单 16-2 中的两个查询都计算`SH.SALES`表中十月份（任意年份）的`AMOUNT_SOLD`不同值的数量。第一个查询的执行计划访问了表中的所有行，因为与清单 16-1 中的第一个查询一样，无法进行分区消除。清单 16-2 中的第二个查询对`SH.TIMES`表的`TIMES_PK`索引执行了`INDEX FULL SCAN`。这个额外步骤增加了开销，但开销（单块访问）被超额补偿了，因为只访问了`SH.SALES`表中的匹配行。最终收益可能不明显，但你可以看到，在我的笔记本电脑上，CBO 估计第一个查询的成本为 528，第二个查询为 102。第二个查询实际上运行得也更快！

### 等式与不等式谓词

在谓词中使用不等式运算符有时是不可避免的，但其误用如此频繁，以至于 SQL 调优顾问会发出警告。清单 16-3 展示了一个典型例子。

清单 16-3. 不等式运算符的不当使用

```sql
CREATE TABLE mostly_boring
(
   primary_key_id   INTEGER PRIMARY KEY
  ,special_flag     CHAR (1)
  ,boring_field     CHAR (100) DEFAULT RPAD ('BORING', 100)
)
PCTFREE 0;

INSERT INTO mostly_boring (primary_key_id, special_flag)
       SELECT ROWNUM, DECODE (MOD (ROWNUM, 10000), 0, 'Y', 'N')
         FROM DUAL
   CONNECT BY LEVEL <= 100000;

BEGIN
   DBMS_STATS.gather_table_stats (
      ownname      => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname      => 'MOSTLY_BORING'
     ,method_opt   => 'FOR COLUMNS SPECIAL_FLAG SIZE 2');
END;
/

CREATE INDEX special_index
   ON mostly_boring (special_flag);

SELECT *
  FROM mostly_boring
 WHERE special_flag != 'N';

| Id  | Operation         | Name          | Rows  | Cost (%CPU)|

|   0 | SELECT STATEMENT  |               |    10 |   384   (1)|
|   1 |  TABLE ACCESS FULL| MOSTLY_BORING |    10 |   384   (1)|

SELECT *
  FROM mostly_boring
 WHERE special_flag = 'Y';

| Id  | Operation                           | Name          | Rows  | Cost (%CPU)|
```


