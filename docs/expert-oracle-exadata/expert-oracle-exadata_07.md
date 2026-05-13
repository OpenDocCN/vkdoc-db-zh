# SQL 执行计划与列投影分析

## 使用`+PROJECTION`获取列投影信息

```sql
SQL> select * from table(dbms_xplan.display_cursor(null,null,’+projection’));

PLAN_TABLE_OUTPUT
-------------------------------------
SQL_ID  69y720khfvjq4, child number 1
-------------------------------------
select /*+ gather_plan_statistics */  count(s.prod_id),
avg(amount_sold) from sales_nonpart s, products p where p.prod_id =
s.prod_id and s.time_id = DATE ’2013-12-01’ and s.tax_country = ’DE’

Plan hash value: 754104813

--------------------------------------------------------------------------------------------
| Id  | Operation                   | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |              |       |       |  198K (100)|          |
|   1 |  SORT AGGREGATE             |              |     1 |    22 |            |          |
|*  2 |   HASH JOIN                 |              |   149 |  3278 |   198K  (2)| 00:00:08 |
|   3 |    TABLE ACCESS STORAGE FULL| PRODUCTS     |    72 |   288 |     3   (0)| 00:00:01 |
|*  4 |    TABLE ACCESS STORAGE FULL| SALES_NONPART|   229 |  4122 |   198K  (2)| 00:00:08 |
--------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("P"."PROD_ID"="S"."PROD_ID")
   4 - storage(("S"."TIME_ID"=TO_DATE(’ 2013-12-01 00:00:00’, ’syyyy-mm-dd
              hh24:mi:ss’) AND "S"."TAX_COUNTRY"=’DE’))
       filter(("S"."TIME_ID"=TO_DATE(’ 2013-12-01 00:00:00’, ’syyyy-mm-dd
              hh24:mi:ss’) AND "S"."TAX_COUNTRY"=’DE’))

Column Projection Information (identified by operation id):
-----------------------------------------------------------
   1 - (#keys=0) COUNT(*)[22], SUM("AMOUNT_SOLD")[22]
   2 - (#keys=1; rowset=200) "AMOUNT_SOLD"[NUMBER,22]
   3 - (rowset=200) "P"."PROD_ID"[NUMBER,22]
   4 - (rowset=200) "S"."PROD_ID"[NUMBER,22], "AMOUNT_SOLD"[NUMBER,22]

Note
-----
   - statistics feedback used for this statement
39 rows selected.
```

```sql
SQL> select projection from v$sql_plan
  2  where projection is not null
  3  and sql_id = ’69y720khfvjq4’
  4  and child_number = 1;

PROJECTION
----------------------------------------------------------------
(#keys=0) COUNT(*)[22], SUM("AMOUNT_SOLD")[22]
(#keys=1; rowset=200) "AMOUNT_SOLD"[NUMBER,22]
(rowset=200) "P"."PROD_ID"[NUMBER,22]
(rowset=200) "S"."PROD_ID"[NUMBER,22], "AMOUNT_SOLD"[NUMBER,22]

Elapsed: 00:00:00.00
```

如你所见，计划输出显示了投影信息，但前提是调用`DBMS_XPLAN`包时使用了`+PROJECTION`参数。另请注意，`PROJECTION`部分列出了两个表的`PROD_ID`列，但并非`WHERE`子句中的所有列都被包含。这一点在操作 ID 4 的谓词输出中变得非常明显。尽管查询通过指定`TIME_ID`和`TAX_COUNTRY`缩小了结果集，但这些列并未出现在`PROJECTION`的任何地方。只有那些需要返回给数据库的列才会被列出。另请注意，投影信息并非 Exadata 独有，而是数据库代码的通用部分。

`V$SQL`系列视图包含定义通过卸载可能节省的数据量（`IO_CELL_OFFLOAD_ELIGIBLE_BYTES`）以及存储服务器实际返回的数据量（`IO_INTERCONNECT_BYTES`, `IO_CELL_OFFLOAD_RETURNED_BYTES`）的列。请注意，这些列是该语句所有执行的累计值。本书中将大量使用`V$SQL`中的这些列，因为它们是卸载处理的关键指标。这里有一个快速演示，表明投影确实会影响返回到数据库服务器的数据量，并且选择更少的列会导致更少的数据传输：

```sql
SQL> select /* single-col-test */ avg(prod_id) from sales;

AVG(PROD_ID)
------------
   80.0035113

Elapsed: 00:00:14.12

SQL> select /* multi-col-test */ avg(prod_id), sum(cust_id), sum(channel_id) from sales;

AVG(PROD_ID) SUM(CUST_ID) SUM(CHANNEL_ID)
------------ ------------ ---------------
   80.0035113   4.7901E+15      1354989738

Elapsed: 00:00:25.89

SQL> select sql_id, sql_text from v$sql where regexp_like(sql_text,’(single|multi)-col-test’);

SQL_ID        SQL_TEXT
------------- ----------------------------------------
8m5zmxka24vyk select /* multi-col-test */ avg(prod_id)
0563r8vdy9t2y select /* single-col-test */ avg(prod_id

SQL> select SQL_ID, IO_CELL_OFFLOAD_ELIGIBLE_BYTES eligible,
  2  IO_INTERCONNECT_BYTES actual,
  3  100*(IO_CELL_OFFLOAD_ELIGIBLE_BYTES-IO_INTERCONNECT_BYTES)
  4  /IO_CELL_OFFLOAD_ELIGIBLE_BYTES "IO_SAVED_%", sql_text
  5  from v$sql where SQL_ID in (’8m5zmxka24vyk’, ’0563r8vdy9t2y’);

SQL_ID          ELIGIBLE     ACTUAL IO_SAVED_% SQL_TEXT
------------- ---------- ---------- ---------- ----------------------------------------
8m5zmxka24vyk 1.6272E+10 4760099744 70.7475353 select /* multi-col-test */ avg(prod_id)
0563r8vdy9t2y 1.6272E+10 3108328256 80.8982443 select /* single-col-test */ avg(prod_id

SQL> @fsx4
Enter value for sql_text: %col-test%
Enter value for sql_id:

SQL_ID       CHILD OFFLOAD IO_SAVED_%  AVG_ETIME SQL_TEXT
------------- ------ ------- ---------- ---------- ----------------------------------------
0563r8vdy9t2y       0 Yes       80.90      14.11 select /* single-col-test */ avg(prod_id
8m5zmxka24vyk       0 Yes       70.75      25.89 select /* multi-col-test */ avg(prod_id)
```

请注意，额外的列导致了查询完成所需的额外时间，并且`V$SQL`中的列验证了必须传输的数据量增加。你还可以初步了解修改版`fsx.sql`脚本的输出，本章稍后将更详细地讨论该脚本。现在，请接受它向我们展示了语句是否被卸载这一事实。



#### 谓词过滤

三大智能扫描优化中的第二项是**谓词过滤**。这个术语指的是 Exadata 仅将数据库层感兴趣的行返回的能力。由于与 Exadata 存储单元交互时使用的 iDB 协议在其请求中包含了谓词信息，谓词过滤是通过在返回数据之前在存储层执行标准的过滤操作来实现的。在使用非 Exadata 存储的数据库上，过滤总是在数据库服务器上完成。这通常意味着大量最终将被丢弃的记录会被返回到数据库层。在存储层过滤这些行可以显著减少必须传输到数据库层的数据量。虽然这种优化也节省了数据库服务器上的 CPU 使用，但最大的优势通常是缩短了数据传输所需的时间。

以下是一个示例：
```
SQL> alter session set cell_offload_processing=false;

Session altered.

Elapsed: 00:00:00.01
SQL> select count(*) from sales;

  COUNT(*)
----------
294575180

Elapsed: 00:00:23.17
SQL> alter session set cell_offload_processing=true;

Session altered.

Elapsed: 00:00:00.00
SQL> select count(*) from sales;

  COUNT(*)
----------
294575180

Elapsed: 00:00:05.68
SQL> -- 禁用存储索引
SQL> alter session set "_kcfis_storageidx_disabled"=true;

System altered.

Elapsed: 00:00:00.12
SQL> select count(*) from sales where quantity_sold = 1;

  COUNT(*)
----------
   3006298

Elapsed: 00:00:02.78
```

首先，使用`CELL_OFFLOAD_PROCESSING`参数完全禁用了卸载功能，然后执行了一个没有`WHERE`子句（“谓词”）的查询。虽然没有卸载功能的益处，但由于完全从智能闪存缓存读取，这个查询花了大约 23 秒。以下证明数据完全来自智能闪存缓存，使用了你将在第 11 章中找到详细说明的`mystats`脚本。在这个案例中，“optimized”关键字表示使用了闪存缓存：
```
STAT    cell flash cache read hits                    51,483
...
STAT    physical read IO requests                     51,483
STAT    physical read bytes                   16,364,175,360
STAT    physical read requests optimized              51,483
STAT    physical read total IO requests               51,483
STAT    physical read total bytes             16,364,175,360
STAT    physical read total bytes optimized   16,364,175,360
```

接下来，启用了卸载功能，并重新执行了相同的查询。这一次，耗时仅为约六秒。节省的大约 18 秒完全归功于列投影（因为没有用于过滤的`WHERE`子句，没有其他优化可以发挥作用）。通过设置隐藏参数`_KCFIS_STORAGEIDX_DISABLED`为`TRUE`（下一节有更多介绍）来禁用存储索引，并添加了`WHERE`子句后，执行时间减少到约两秒。这额外节省的大约三秒钟要归功于谓词过滤。请注意，在示例中必须禁用存储索引，以确保查询性能的提升完全归因于谓词过滤，没有其他性能增强（即存储索引）的干扰。

### 存储索引与区域映射表

存储索引为智能扫描提供了第三级优化。存储索引是存储单元上的内存结构，为每个 1MB 的磁盘存储单元维护表中最多八列的最小值和最大值。它们在段被查询后在存储单元上透明地创建和维护。存储索引与大多数智能扫描优化略有不同。存储索引的目标不是减少传输回数据库层的数据量。事实上，无论在给定查询中是否使用它们，返回到数据库层的数据量都保持不变。相反，存储索引旨在消除存储服务器本身从磁盘读取数据所花费的时间。可以将此功能视为一个预过滤器。由于智能扫描将查询谓词传递给存储服务器，并且存储索引包含了每个 1MB 存储区域中最多八列的最小/最大值映射，任何不可能包含匹配行（因为它位于存储索引存储的最小和最大值之外）的区域都可以在不被读取的情况下被排除。你也可以将存储索引视为一种替代分区机制。磁盘 I/O 的消除方式类似于分区消除。如果一个分区不包含任何感兴趣的记录，该分区的块就不会被读取。类似地，如果一个存储区域不包含任何感兴趣的记录，则无需读取该存储区域。

存储索引并非在所有情况下都能使用，而且几乎无法影响它们何时或如何被使用。但是，在适当的情况下，这种优化技术的效果可能令人震惊。一如既往，最好用一个例子来展示：
```
SQL> -- 禁用存储索引
SQL> alter session set "_kcfis_storageidx_disabled"=true;

System altered.

Elapsed: 00:00:00.11
SQL> select count(*) from bigtab where id = 8000000;

COUNT(*)
----------
32

Elapsed: 00:00:22.02
SQL> -- 重新启用存储索引
SQL> alter session set "_kcfis_storageidx_disabled"=false;

System altered.

Elapsed: 00:00:00.01
SQL> select count(*) from bigtab where id = 8000000;

COUNT(*)
----------
32

Elapsed: 00:00:00.54
```

在这个例子中，再次使用前述参数`_KCFIS_STORAGEIDX_DISABLED`故意禁用了存储索引，以提醒你仅使用列投影和谓词过滤读取所有行所需的耗时。请记住，尽管在这种情况下返回到数据库层的数据量非常小，但存储服务器仍然必须读取包含`BIGTAB`表数据的所有块，然后必须检查每一行是否匹配`WHERE`子句。这就是 22 秒中大部分时间花费的地方。在重新启用存储索引并重新执行查询后，执行时间减少到约 0.05 秒。执行时间的减少是存储索引被用来避免几乎所有磁盘 I/O 以及过滤这些记录所花费时间的结果。

从 Exadata 版本 12.1.2.1.0 和数据库 12.1.0.2 开始，Oracle 引入了在存储服务器的存储索引中保留列最小/最大值的全局最小/最大值的能力。其思想是`min()`和`max()`函数可以直接获取保留的全局值，而无需访问存储索引来计算该值。这应该对使用`min()`和`max()`函数的查询（例如填充有这些值的仪表板的分析工作负载）有益。不幸的是，在撰写本文时，没有额外的工具可以表明使用了缓存的最小或最大值。你只能看到众所周知的统计信息“cell physical IO bytes saved by storage index”发生了变化。



重申一下，列投影和谓词过滤（以及大多数其他 Smart Scan 优化）通过减少传输回数据库服务器的数据量（从而缩短数据传输时间）来提升性能。存储索引则通过消除从磁盘读取数据以及在存储服务器上过滤数据所花费的时间来提高性能。关于存储索引的更多细节将在第 4 章中详细阐述。

## 区域映射是 Oracle 12.1.0.2 中的新特性

区域映射是 Oracle 12.1.0.2 中的新特性，其概念上与存储索引相似。不同之处在于，区域映射让用户对要监控的段拥有更多控制权。当我们首次听说区域映射时，我们感到非常兴奋，因为这一特性可以看作是将存储索引（需要 Exadata 存储单元）移植到了非 Exadata 平台上。遗憾的是，区域映射的使用仅限于 Exadata，这使其吸引力大打折扣。在 Oracle 术语中，“区域”指的是磁盘上表的一个区域，通常大约包含 1024 个数据块。与存储索引一样，区域映射允许 Oracle 跳过磁盘上与查询无关的区域。区域映射与刚刚讨论的存储索引之间的最大区别在于，后者是在存储服务器上维护的，而区域映射则是在数据库管理员的控制下在数据库级别创建和维护的。存储索引驻留在存储单元上，数据库管理员对其控制权很小。区域映射与物化视图非常相似，但不需要物化视图日志。创建区域映射时，需要决定如何刷新它以防止其变得过时。区域映射信息也存储在数据库的数据字典中。当存在可用的区域映射且可以使用时，你会在执行计划中看到它被用作过滤器。

```sql
SQL> -- 关闭存储索引，否则可能干扰结果
SQL> alter session set "_kcfis_storageidx_disabled" = true;

会话已更改。

已用时间: 00:00:00.00
SQL> select /*+ gather_plan_statistics zmap_example_001 */ count(*)
  2  from T1_ORDER_BY_ID where id = 121;

  COUNT(*)
----------
        16

已用时间: 00:00:02.65
SQL> select * from table(dbms_xplan.display_cursor);

PLAN_TABLE_OUTPUT
-------------------------------------
SQL_ID  avt7474pb4m1m, child number 0
-------------------------------------
select /*+ gather_plan_statistics zmap_example_001 */ count(*) from
T1_ORDER_BY_ID where id = 121

Plan hash value: 775109614

--------------------------------------------------------------------------------------
| Id  | Operation                      | Name           | Rows  | Bytes | Cost (%CPU)|
--------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |                |       |       |   723K(100)|
|   1 |  SORT AGGREGATE                |                |     1 |     5 |            |
|*  2 |   TABLE ACCESS STORAGE FULL    |                |       |       |            |
|     |    WITH ZONEMAP                | T1_ORDER_BY_ID |    16 |    80 |   723K  (1)|
--------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - storage("ID"=121)
       filter((SYS_ZMAP_FILTER('/* ZM_PRUNING */ SELECT "ZONE_ID$", CASE WHEN
              BITAND(zm."ZONE_STATE$",1)=1 THEN 1 ELSE
              CASE WHEN (zm."MIN_1_ID" > :1 OR zm."MAX_1_ID" < :2)
              THEN 3 ELSE 2 END END FROM "MARTIN"."T1_ORDER_BY_ID_ZMAP" zm
              WHERE zm."ZONE_LEVEL$"=0 ORDER BY
              zm."ZONE_ID$"',SYS_OP_ZONE_ID(ROWID),121,121)<3 AND "ID"=121))

已选择 24 行。
```

当你查看该语句的 SQL 跟踪时，你会看到在作为 `SYS` 执行的递归 SQL 语句中，会为符合修剪条件的区域查询区域映射。跟踪到的语句看起来与执行计划中报告的过滤器完全一样。


### 简单连接（Bloom 过滤器）

在某些情况下，连接处理也可以被下推到存储层完成。这种下推的连接是通过创建一种叫做 Bloom 过滤器（Bloom filter）的结构来实现的。Bloom 过滤器由来已久，自 Oracle 数据库 10g 第 2 版起就已被 Oracle 使用。因此，它们并非 Exadata 特有。Oracle 使用它们的主要方式之一是减少并行查询从属进程（parallel query slaves）之间的流量。从 Oracle 11.2.0.4 和 12c 开始，你也可以在串行查询处理中使用 Bloom 过滤器。

Bloom 过滤器的优势在于，相对于它们所代表的数据集，其尺寸非常小。然而，这是有代价的——它们可能返回假阳性（false positives）。也就是说，本不应包含在目标结果集中的行，偶尔可能会通过 Bloom 过滤器检查。因此，在 Bloom 过滤器之后必须应用一个额外的过滤器，以确保消除任何假阳性。从 Exadata 的角度来看，关于 Bloom 过滤器一个有趣的事实是，它们可以被传递到存储服务器并在那里进行评估，从而有效地将一个连接（join）转换为一个过滤器（filter）。这种技术可以显著减少必须传回数据库服务器的数据量。下面的例子演示了这一点：

```
SQL> show parameter bloom

PARAMETER_NAME                     TYPE        VALUE
----------------------------------- ----------- ------
_bloom_predicate_offload           boolean     FALSE

SQL> show parameter kcfis

PARAMETER_NAME                     TYPE        VALUE
----------------------------------- ----------- ------
_kcfis_storageidx_disabled         boolean     TRUE

SQL> select /* bloom0015 */ * from customers c, orders o
  2  where o.customer_id = c.customer_id and c.cust_email = 'user@example.com';

no rows selected

Elapsed: 00:00:13.57

SQL> alter session set "_bloom_predicate_offload" = true;

Session altered.

SQL> select /* bloom0015 */ * from customers c, orders o
  2  where o.customer_id = c.customer_id and c.cust_email = 'user@example.com';

no rows selected

Elapsed: 00:00:02.56

SQL> @fsx
Enter value for sql_text: %bloom0015%
Enter value for sql_id:

SQL_ID        CHILD  PLAN_HASH  EXECS  AVG_ETIME AVG_PX OFFLOAD IO_SAVED_% SQL_TEXT
------------- ------ ---------- ------ ------ ------- ---------- --------------
5n3np45j2x9vn      0  576684111      1      13.56      0 Yes          54.26 select /* bloom0015
5n3np45j2x9vn      1 2651416178      1       2.56      0 Yes          99.98 select /* bloom0015

SQL> --fetch first explain plan, without bloom filter
SQL> select * from table(
  2   dbms_xplan.display_cursor('5n3np45j2x9vn',0,'BASIC +predicate +partition'));

PLAN_TABLE_OUTPUT
------------------------------------------------------------------------
EXPLAINED SQL STATEMENT:
------------------------------------------------------------------------
select /* bloom0015 */ * from customers c, orders o where o.customer_id
= c.customer_id and c.cust_email = 'user@example.com'

Plan hash value: 576684111

-------------------------------------------------------------------------------
| Id  | Operation                   | Name      | Pstart| Pstop |
-------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |           |       |       |
|*  1 |  HASH JOIN                  |           |       |       |
|   2 |   PARTITION HASH ALL        |           |     1 |    32 |
|*  3 |    TABLE ACCESS STORAGE FULL| CUSTOMERS |     1 |    32 |
|   4 |   PARTITION HASH ALL        |           |     1 |    32 |
|   5 |    TABLE ACCESS STORAGE FULL| ORDERS    |     1 |    32 |
-------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - access("O"."CUSTOMER_ID"="C"."CUSTOMER_ID")
   3 - storage("C"."CUST_EMAIL"='user@example.com')
       filter("C"."CUST_EMAIL"='user@example.com')

25 rows selected.

Elapsed: 00:00:00.01

SQL> --fetch second explain plan, with bloom filter
SQL> select * from table(
  2  dbms_xplan.display_cursor('5n3np45j2x9vn',1,'BASIC +predicate +partition'));

PLAN_TABLE_OUTPUT
------------------------------------------------------------------------
EXPLAINED SQL STATEMENT:
------------------------------------------------------------------------
select /* bloom0015 */ * from customers c, orders o where o.customer_id
= c.customer_id and c.cust_email = 'user@example.com'

Plan hash value: 2651416178

---------------------------------------------------------------------------------
| Id  | Operation                    | Name      | Pstart| Pstop |
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |           |       |       |
|*  1 |  HASH JOIN                   |           |       |       |
|   2 |   JOIN FILTER CREATE         | :BF0000   |       |       |
|   3 |    PARTITION HASH ALL        |           |     1 |    32 |
|*  4 |     TABLE ACCESS STORAGE FULL| CUSTOMERS |     1 |    32 |
|   5 |   JOIN FILTER USE            | :BF0000   |       |       |
|   6 |    PARTITION HASH ALL        |           |     1 |    32 |
|*  7 |     TABLE ACCESS STORAGE FULL| ORDERS    |     1 |    32 |
---------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - access("O"."CUSTOMER_ID"="C"."CUSTOMER_ID")
   4 - storage("C"."CUST_EMAIL"='user@example.com')
       filter("C"."CUST_EMAIL"='user@example.com')
   7 - storage(`SYS_OP_BLOOM_FILTER`(:BF0000,"O"."CUSTOMER_ID"))
       filter(`SYS_OP_BLOOM_FILTER`(:BF0000,"O"."CUSTOMER_ID"))

29 rows selected.
```

在这个示例中，隐藏参数`_BLOOM_PREDICATE_OFFLOAD`（在 11.2 中曾命名为`_BLOOM_PREDICATE_PUSHDOWN_TO_STORAGE`）被用于对比。注意，测试查询在使用 Bloom 过滤器时运行了大约 2 秒，而没有使用时则用了 14 秒。同时请注意，两次查询都发生了数据下推（offloaded）。如果你仔细查看执行计划的谓词信息（predicate information），你会看到第二次运行（对应子游标 1）时，`SYS_OP_BLOOM_FILTER(:BF0000,"O"."CUSTOMER_ID")`谓词是在存储服务器上执行的。使用 Bloom 过滤器的查询运行得更快，因为存储服务器能够预先连接表，从而消除了大量原本需要传回数据库服务器的数据。Oracle 数据库引擎不仅限于使用单个 Bloom 过滤器，根据查询的复杂程度，可能会有多个。

> 注意
> 内存聚合（In-Memory Aggregation）功能（需要内存选件）提供了另一种类似于 Bloom 过滤器的优化。使用它，你可以受益于下推作为转换一部分的键向量（key vectors）。

## Exadata 优化：函数卸载与 HCC 压缩

### 函数卸载

Oracle 对结构化查询语言（SQL）的实现包含许多内置函数。这些函数可以直接在 SQL 语句中使用。大致可分为两类：单行函数和多行函数。单行函数为查询表的每一行返回一个结果行。这些单行函数可以进一步细分为以下几类：

*   数字函数（`SIN`, `COS`, `FLOOR`, `MOD`, `LOG`, ...）
*   字符函数（`CHR`, `LPAD`, `REPLACE`, `TRIM`, `UPPER`, `LENGTH`, ...）
*   日期时间函数（`ADD_MONTHS`, `TO_CHAR`, `TRUNC`, ...）
*   转换函数（`CAST`, `HEXTORAW`, `TO_CHAR`, `TO_DATE`, ...）

几乎所有这些单行函数都可以被卸载到 Exadata 存储。第二类主要的 SQL 函数对一组行进行操作。在这个多行函数类别中有两个子类：

*   聚合函数（`AVG`, `COUNT`, `SUM`, ...）
*   分析函数（`AVG`, `COUNT`, `DENSE_RANK`, `LAG`, ...）

这些函数返回单行（聚合函数）或多行（分析函数）。请注意，有些函数被重载，同时属于这两组。这些函数都不能卸载到 Exadata，这是合理的，因为许多函数需要访问整组行——这是单个存储单元不具备的。

还有一些额外的函数不完全属于上述任何分组。这些函数可能被卸载到存储单元，也可能不被卸载。例如，`DECODE`和`NVL`是可卸载的，但大多数 XML 函数则不是。一些数据挖掘函数是可卸载的，但有些不是。同时请记住，可卸载函数列表可能随着新版本的发布而变化。特定版本可卸载函数的权威列表包含在`V$SQLFN_METADATA`中。例如，在 11.2.0.3 版本中，923 个 SQL 函数中有 393 个是可卸载的。

```sql
SQL> select count(*), offloadable from v$sqlfn_metadata group by rollup(offloadable);
COUNT(*) OFF
----------- ---
       530 NO
       393 YES
       923
```

在写作时的当前版本 12.1.0.2 中，函数数量有所增加：

```sql
SQL> select count(*), offloadable from v$sqlfn_metadata group by rollup(offloadable);
COUNT(*) OFF
---------- ---
       615 NO
       418 YES
      1033
```

卸载函数确实允许存储单元完成一些通常由数据库服务器 CPU 完成的工作。然而，CPU 使用率的节省通常是一个相对次要的改进。主要的收益通常来自于限制传回数据库服务器的数据量。能够评估`WHERE`子句中的函数，使得存储单元可以仅将感兴趣的行发送回数据库层。因此，与大多数卸载一样，此优化的主要目标是减少存储层和数据库层之间的流量。如果一个函数已被卸载到存储服务器，你可以在`DBMS_XPLAN.DISPLAY_CURSOR`输出的谓词信息中看到这一点，如下所示：

```sql
SQL> select count(*) from sales where lower(tax_country) = 'ir';
COUNT(*)
----------
    435700

SQL> select * from table(dbms_xplan.display_cursor(null, null, 'BASIC +predicate'));
PLAN_TABLE_OUTPUT
------------------------------------------------------------------------
EXPLAINED SQL STATEMENT:
------------------------------------------------------------------------
select count(*) from sales where lower(tax_country) = 'ir'

Plan hash value: 3519235612

------------------------------------------------
| Id  | Operation                   | Name  |
------------------------------------------------
|   0 | SELECT STATEMENT            |       |
|   1 |  SORT AGGREGATE             |       |
|   2 |   PARTITION RANGE ALL       |       |
|*  3 |    TABLE ACCESS STORAGE FULL| SALES |
------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   3 - storage(LOWER("TAX_COUNTRY")='ir')
       filter(LOWER("TAX_COUNTRY")='ir')
```

这与在 select-list 中引用函数的情况不同：

```sql
SQL> select lower(tax_country) from sales where rownum < 11;
LOW
---
zl
uw
...
bg

10 rows selected.

SQL> select * from table(dbms_xplan.display_cursor(null, null, 'BASIC +predicate'));
PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------
EXPLAINED SQL STATEMENT:
--------------------------------------------------------------------------
select lower(tax_country) from sales where rownum < 11

Plan hash value: 807288713

------------------------------------------------------------
| Id  | Operation                         | Name  |
------------------------------------------------------------
|   0 | SELECT STATEMENT                  |       |
|*  1 |  COUNT STOPKEY                    |       |
|   2 |   PARTITION RANGE ALL             |       |
|   3 |    TABLE ACCESS STORAGE FULL FIRST ROWS| SALES |
------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter(ROWNUM<11)
```

如你所见，谓词信息中没有提到`to_lower()`函数。不过，在这种情况下，你仍然可以从列投影中受益。

#### 压缩/解压缩

Exadata 的一个备受关注的特性是 HCC。在 Smart Scan 操作期间，Exadata 会卸载对 HCC 格式存储数据的解压。也就是说，当通过 Smart Scan 访问压缩数据时，感兴趣的列会在存储单元上被解压。这种解压对于过滤并非必需，因此只有将返回给数据库层的数据才会被解压。请注意，所有的压缩操作都是在数据库层完成的。当数据不是通过 Smart Scan 访问，或者存储单元非常繁忙时，解压也可能在数据库层完成。为简化起见，表 2-1 显示了工作在何处完成。

表 2-1.
HCC 压缩/解压卸载

| 操作             | 数据库服务器                     | 存储服务器             |
| ---------------- | -------------------------------- | ---------------------- |
| 压缩（Compress） | 总是                             | 从不                   |
| 解压（Decompress）| 非 Smart Scan；在单元太忙时可帮忙 | Smart Scan             |

在存储层解压数据与大多数其他 Smart Scan 优化的主题背道而驰。那些优化大多旨在减少需要传输回数据库服务器的数据量。由于解压是一项 CPU 密集型任务，特别是在使用更高级别的压缩时，因此决定尽可能在存储服务器上进行解压。然而，这个决定并非一成不变，因为在某些情况下，可能有足够的 CPU 资源可用，使得在数据库服务器上解压数据成为一个有吸引力的选择。（也就是说，在某些情况下，减少需要传输的数据量可能比减少数据库服务器的 CPU 消耗更重要。）事实上，从`cellsrv`版本 11.2.2.3.1 开始，当存储单元繁忙时，Exadata 确实有能力将压缩数据返回给数据库服务器。第 3 章更详细地讨论了 HCC，并将提供此类情况的示例。

有一个隐藏参数控制是否完全卸载解压。不幸的是，它并不能简单地在存储层和数据库层之间来回移动解压操作。如果将`_CELL_OFFLOAD_HYBRIDCOLUMNAR`参数设置为`FALSE`，Smart Scan 将完全禁用对 HCC 数据的处理。

#### 加密/解密

加密和解密的处理方式与 HCC 数据的压缩和解压缩非常相似。加密总是在数据库层完成，而解密可由存储服务器或数据库服务器执行。当通过智能扫描访问加密数据时，解密在存储服务器上进行；否则，解密在数据库服务器上进行。请注意，从 X2 Exadata 这一代开始，存储服务器中的 Intel Xeon 芯片已内置基于硅片的加密能力。现代 Intel 芯片包含一套特殊指令集（`Intel AES-NI`），可有效为执行加密或解密的进程提供硬件加速。需要注意的是，需要 Oracle Database 11.2.0.2 或更高版本才能利用这套新指令集。

加密与 HCC 压缩能够良好协同工作。由于压缩先执行，对 HCC 数据进行加密和解密的进程所需完成的工作量便得以减少。请注意，`CELL_OFFLOAD_DECRYPTION` 参数控制此行为，并且与隐藏参数 `_CELL_OFFLOAD_HYBRIDCOLUMNAR` 类似，将该参数设置为 `FALSE` 会完全禁用对加密数据的智能扫描，这同时也禁用了存储层的解密。

#### 虚拟列

虚拟列提供了定义伪列的能力，这些伪列可基于表中的其他列计算得出，而无需实际存储计算后的值。虚拟列可用作分区键、用于约束中或建立索引。也可以收集它们的列级统计信息。由于虚拟列的值并未实际存储，因此在访问时必须即时计算。在 Exadata 平台之外，数据库会话必须自行计算这些值。而在 Exadata 上，通过智能扫描进行段访问时，这些计算可以被卸载执行：

```sql
SQL> alter table bigtab add idn1 generated always as (id + n1);

Table altered.

SQL> select column_name, data_type, data_default
  2  from user_tab_columns where table_name = 'BIGTAB';

COLUMN_NAME          DATA_TYPE            DATA_DEFAULT
-------------------- -------------------- ------------
IDN1                 NUMBER               "ID"+"N1"
ID                   NUMBER
V1                   VARCHAR2
N1                   NUMBER
N2                   NUMBER
N_256K               NUMBER
N_128K               NUMBER
N_8K                 NUMBER
PADDING              VARCHAR2

9 rows selected.
```

现在，您可以查询包含虚拟列的表。为了演示卸载的效果，首先需要一个随机值。在此数据集中，`ID` 和 `N1` 的组合应具有较好的唯一性：

```sql
SQL> select /*+ gather_plan_statistics virtual001 */ id, n1, idn1 from bigtab where rownum < 11;

        ID         N1       IDN1
---------- ---------- ----------
   1161826       1826    1163652
   1161827       1827    1163654
   1161828       1828    1163656
   1161829       1829    1163658
   1161830       1830    1163660
   1161831       1831    1163662
   1161832       1832    1163664
   1161833       1833    1163666
   1161834       1834    1163668
   1161835       1835    1163670

10 rows selected.
```

以下是卸载计算如何使执行时间受益的演示：

```sql
SQL> select /* gather_plan_statistics virtual0002 */ count(*)
  2  from bigtab where idn1 = 1163652;

  COUNT(*)
----------
        64

Elapsed: 00:00:06.78

SQL> @fsx4
Enter value for sql_text: %virtual0002%
Enter value for sql_id:

SQL_ID         CHILD OFFLOAD IO_SAVED_%  AVG_ETIME SQL_TEXT
------------- ------ ------- ---------- ---------- ----------------------------------------
8dyyd6kzycztq      0 Yes          99.99       6.77 select /* virtual0002 */ count(*) from b

Elapsed: 00:00:00.39

SQL> @dplan
Copy and paste SQL_ID and CHILD_NO from results above
Enter value for sql_id: 8dyyd6kzycztq
Enter value for child_no:

PLAN_TABLE_OUTPUT
-------------------------------------
SQL_ID  8dyyd6kzycztq, child number 0
-------------------------------------
select /* virtual0002 */ count(*) from bigtab where idn1 = 1163652

Plan hash value: 2140185107

-------------------------------------------------------------------------------------
| Id  | Operation                   | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |        |       |       |  2780K(100)|          |
|   1 |  SORT AGGREGATE             |        |     1 |    13 |            |          |
|*  2 |   TABLE ACCESS STORAGE FULL | BIGTAB |  2560K|    31M|  2780K  (1)| 00:01:49 |
-------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - storage("ID"+"N1"=1163652)
       filter("ID"+"N1"=1163652)
```

根据上次执行时 `V$SQL` 中的关键列计算，节省的 I/O 量为 99.99%，该查询耗时 6.78 秒完成。

与 Oracle 世界中的许多功能一样，存在一个参数可影响会话行为。在下一个示例中，将使用相关的下划线参数在存储服务器级别禁用虚拟列处理。这样做是为了模拟相同查询在非 Exadata 平台上的运行方式：

```sql
SQL> alter session set "_cell_offload_virtual_columns"=false;

Session altered.

Elapsed: 00:00:00.00

SQL> select /* virtual0002 */ count(*)
  2  from bigtab where idn1 = 1163652;

  COUNT(*)
----------
        64

Elapsed: 00:00:23.13
```

当计算节点必须评估表达式时，执行时间明显增加。比较两条语句的执行情况可以看出这一点：

```sql
SQL> @fsx4
Enter value for sql_text: %virtual0002%
Enter value for sql_id:

SQL_ID         CHILD OFFLOAD IO_SAVED_%  AVG_ETIME SQL_TEXT
------------- ------ ------- ---------- ---------- ----------------------------------------
8dyyd6kzycztq      0 Yes          99.99       6.77 select /* virtual0002 */ count(*) from b
8dyyd6kzycztq      1 Yes          94.18      23.13 select /* virtual0002 */ count(*) from b

2 rows selected.
```

您还会注意到，在显示第一个子游标对应的执行计划时，谓词部分缺少了 `storage` 关键字：

```sql
SQL> @dplan
Copy and paste SQL_ID and CHILD_NO from results above
Enter value for sql_id: 8dyyd6kzycztq
Enter value for child_no: 1

PLAN_TABLE_OUTPUT
-------------------------------------
SQL_ID  8dyyd6kzycztq, child number 1
-------------------------------------
select /* virtual0002 */ count(*) from bigtab where idn1 = 1163652

Plan hash value: 2140185107

-------------------------------------------------------------------------------------
| Id  | Operation                   | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |        |       |       |  2780K(100)|          |
|   1 |  SORT AGGREGATE             |        |     1 |    13 |            |          |
|*  2 |   TABLE ACCESS STORAGE FULL | BIGTAB |    64 |   832 |  2780K  (1)| 00:01:49 |
-------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - filter("ID"+"N1"=1163652)

Note
-----
   - statistics feedback used for this statement
```

如果您在 `WHERE` 子句中使用虚拟列，那么您无疑能从 Exadata 平台获益。


### 对 LOB 卸载的支持

随着 Exadata 软件 12.1.1.1.1 和 RDBMS 12.1.0.2 的引入，针对定义为安全文件的内联 LOB 的查询也可以被卸载了。根据相关文档集，`like` 和 `regexp_like` 可以被卸载。为了演示这一新功能，创建了一个新表 `LOBOFFLOAD`，并填充了 1600 万行数据。这应能确保它被考虑用于智能扫描。以下是关于 LOB 列的关键信息：

```
SQL> select table_name, column_name, segment_name, securefile, in_row
  2  from user_lobs where table_name = 'LOBOFFLOAD';

TABLE_NAME              COLUMN_NAME              SEGMENT_NAME                      SEC IN_
---------------------   ---------------------   ---------------------------       --- ---
LOBOFFLOAD              COMMENTS                SYS_LOB0000096135C00002$$         YES YES
```

该表试图模拟一种常见的应用技术，即在表中定义一个 CLOB 字段，用于输入与记录相关的额外非结构化信息。只要这不违背数据模型中的约束条件，并且存储的纯粹是信息性的、在任何形式的处理中都不需要的信息，这应该就是可以的。以下是在 12c 中的示例：

```
SQL> select /*+ monitor loboffload001 */ count(*) from loboffload where comments like '%GOOD%';

  COUNT(*)
----------
     15840

Elapsed: 00:00:02.93

SQL> @fsx4.sql
Enter value for sql_text: %loboffload001%
Enter value for sql_id:

SQL_ID         CHILD OFFLOAD IO_SAVED_%  AVG_ETIME SQL_TEXT
------------- ------ ------- ---------- ---------- -----------------------------------------
18479dnagkkyu      0 Yes          98.94       2.93 select /*+ monitor loboffload001 */ count
```

正如你在脚本输出中看到的（我们稍后将详细讨论），该查询已被卸载。在 11.2.0.3 版本中重现此测试用例时，情况则不同：

```
SQL> select /*+ monitor loboffload001 */ count(*) from loboffload where comments like '%GOOD%';

   COUNT(*)
-----------
      15840

Elapsed: 00:01:34.04

SQL> @fsx4.sql
Enter value for sql_text: %loboffload001%
Enter value for sql_id:

SQL_ID         CHILD OFFLOAD IO_SAVED_%  AVG_ETIME SQL_TEXT
------------- ------ ------- ---------- ---------- -----------------------------------------
18479dnagkkyu      0 No             .00      94.04 select /*+ monitor loboffload001 */ count
```

与第一个示例不同，在 11.2.0.3 上执行的第二个查询未被卸载。由于段大小的原因，它使用了直接路径读取，但与第一个示例不同，这些读取并未转变为智能扫描。

### JSON 支持与卸载

随着 Oracle 12.1.0.2 的推出，数据库层添加了 JSON 支持。如果你使用的是 Exadata 12.1.0.2.1.0 或更高版本，就可以从卸载部分这些运算符中受益。正如你在函数卸载部分看到的，你可以查询 `v$sqlfn_metadata` 以了解函数是否支持卸载。以下是检查 JSON 相关函数及其卸载支持情况的结果：

```
SQL> select count(*), name, offloadable from v$sqlfn_metadata
  2  where name like '%JSON%' group by name, offloadable
  3  order by offloadable, name;

  COUNT(*) NAME                       OFF
---------- ------------------------------ ---
         1 JSON_ARRAY                    NO
         1 JSON_ARRAYAGG                 NO
         1 JSON_EQUAL                    NO
         1 JSON_OBJECT                   NO
         1 JSON_OBJECTAGG                NO
         1 JSON_QUERY                    NO
         1 JSON_SERIALIZE                NO
         1 JSON_TEXTCONTAINS2            NO
         1 JSON_VALUE                    NO
         2 JSON                          YES
         1 JSON_EXISTS                   YES
         1 JSON_QUERY                    YES
         1 JSON_VALUE                    YES

13 rows selected.
```

根据 Oracle 文档，12.1.0.2.1 的用户也能从卸载 `XMLExists` 和 `XMLCast` 操作中受益。

#### 数据挖掘模型评分

一些数据模型评分函数可以被卸载。一般来说，这种优化旨在减少传输到数据库层的数据量，而不是纯粹的 CPU 卸载。与其他函数卸载一样，你可以通过查询 `V$SQLFN_METADATA` 来验证哪些数据挖掘函数可以卸载。输出如下：

```
SQL> select distinct name, version, offloadable
  2  from V$SQLFN_METADATA
  3  where name like 'PREDICT%'
  4  order by 1,2
  5  /

NAME                           VERSION      OFF
------------------------------ ------------ ---
PREDICTION                     V10R2 Oracle YES
PREDICTION_BOUNDS              V11R1 Oracle NO
PREDICTION_COST                V10R2 Oracle YES
PREDICTION_DETAILS             V10R2 Oracle NO
PREDICTION_PROBABILITY         V10R2 Oracle YES
PREDICTION_SET                 V10R2 Oracle NO

6 rows selected.
```

如你所见，部分函数支持卸载，部分则不支持。支持卸载的函数可以被存储单元用于谓词过滤。以下是一个示例查询，它应该只返回符合 `WHERE` 子句中指定的评分要求的记录：

```
SQL> select cust_id
  2  from customers
  3  where region = 'US'
  4  and prediction_probability(churnmod,'Y' using *) > 0.8
  5  /
```

此优化旨在卸载 CPU 使用率并减少传输的数据量。然而，在能够减少返回到数据库层的数据量的情况下，它最为有益，如前面的示例所示。

### 非智能扫描卸载

有一些优化与查询处理无关。由于这些不是本章的重点，我们将仅简要提及。

#### 智能/快速文件创建

这个优化的名称有些误导性。它实际上是一种旨在加速块初始化的优化。每当分配块时，数据库都必须对其进行初始化。这种活动在创建表空间时会发生，但当因任何其他原因添加或扩展文件时也会发生。在非 Exadata 存储上，这些情况要求数据库服务器格式化每个块，然后将其写回磁盘。所有这些读写操作会在数据库服务器和存储单元之间产生大量流量。正如你现在所知，消除层间流量是 Exadata 的主要目标。正如你可能想象的那样，这种完全不必要的流量已被消除。

此过程已得到进一步改进。从 Oracle Exadata 11.2.3.3.0（这是作者们最喜爱的 Exadata 版本之一）开始，Oracle 引入了快速数据文件创建。通过使用一个巧妙的技巧，可以进一步减少初始化数据文件所需的时间。你在上一段中读到的第一个优化是将清零数据文件的任务委派给存储单元，这本身就被证明非常有效。下一个合乎逻辑的步骤，也是通过快速文件创建获得的功能，就是仅将元数据写入回写闪存缓存，从而消除实际格式化块的过程。如果存储单元中启用了 WBFC，默认情况下将使用快速数据文件创建。你可以在第 5 章中阅读更多关于 Exadata 智能闪存缓存的信息。


#### RMAN 增量备份

Exada 通过提高块更改跟踪的精细度来加速增量备份。在非 Exadata 平台上，块更改是按块组进行跟踪的；而在 Exadata 上，更改是针对单个块进行跟踪的。这可以显著减少需要备份的块数量，从而得到更小的备份大小、更少的 I/O 带宽，并缩短完成增量备份所需的时间。可以通过将 `_DISABLE_CELL_OPTIMIZED_BACKUPS` 参数设置为 `TRUE` 来禁用此功能。此优化在第 10 章中有更详细的介绍。

#### RMAN 恢复

此优化加速了在存储节点上从备份恢复时的文件初始化部分。尽管从备份恢复数据库并不常见，但此优化也有助于加速环境克隆。该优化降低了数据库服务器上的 CPU 使用率，并减少了两层架构之间的流量。如果将 `_CELL_FAST_FILE_RESTORE` 参数设置为 `FALSE`，则将禁用此行为。此优化同样在第 10 章中有所介绍。

## Smart Scan 前置条件

并非在 Exadata 上运行的每个查询都会发生 Smart Scan。必须满足三个基本要求才能发生 Smart Scan：

*   必须对段（表、分区、物化视图等）进行全扫描。
*   扫描必须使用 Oracle 的直接路径读取机制。
*   对象必须存储在 Exadata 存储上。

这些要求的存在有一个简单的解释。Oracle 是一个 C 程序。执行 Smart Scan 的函数 (`kcfis_read`) 由直接路径读取函数 (`kcbldrget`) 调用，而后者又由某个全扫描函数调用。事情就是这么简单。如果没有从全扫描到直接读取的代码路径，就无法到达 `kcfis_read` 函数。当然，存储层也必须运行 Oracle 软件，以便处理所有数据文件均位于 Exadata 上的 Smart Scan。我们将依次讨论这些要求。

### 全扫描

为了使查询能够利用 Exadata 的卸载功能，优化器必须决定对语句执行全表扫描或快速全索引扫描。在此上下文中，这些术语的使用有些泛化。全段扫描也是直接路径读取的前提条件。如前所述，除非做出了直接路径读取的决定，否则不会有 Smart Scan。

一般来说，（快速）全扫描对应于执行计划中的 `TABLE ACCESS FULL` 和 `INDEX FAST FULL SCAN` 操作。在 Exadata 上，这些熟悉的操作名称略有更改，以表明它们正在访问 Exadata 存储。新的操作名称是 `TABLE ACCESS STORAGE FULL` 和 `INDEX STORAGE FAST FULL SCAN`。

确定是否发生了全扫描通常相当简单，但你可能需要查看不止一个地方。开始调查最简单的方法是在查询执行完毕后立即调用 `DBMS_XPLAN.DISPLAY_CURSOR()`：

```
SQL> select count(*) from bigtab;

  COUNT(*)
----------
 256000000

Elapsed: 00:00:07.98

SQL> select * from table(dbms_xplan.display_cursor(null, null));

PLAN_TABLE_OUTPUT
-------------------------------------
SQL_ID  8c9rzdry8yahs, child number 0
-------------------------------------
select count(*) from bigtab
Plan hash value: 2140185107
-----------------------------------------------------------------------------
| Id  | Operation                  | Name   | Rows  | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT           |        |       |  2779K(100)|          |
|   1 |  SORT AGGREGATE            |        |     1 |            |          |
|   2 |   TABLE ACCESS STORAGE FULL | BIGTAB |   256M|  2779K  (1)| 00:01:49 |
-----------------------------------------------------------------------------
```

或者，你可以使用 `fsx.sql` 脚本从共享池中定位查询的 SQL 文本，并使用 `SQL_ID` 和游标子编号调用 `DISPLAY_CURSOR()` 函数。`dplan.sql` 脚本是执行此操作的便捷方式。

请注意，这些操作还有一些细微变体，例如 `MAT_VIEW ACCESS STORAGE FULL`，它也适用于物化视图的 Smart Scan。然而，你应该意识到，你的执行计划显示 `TABLE ACCESS STORAGE FULL` 操作并不意味着你的查询是通过 Smart Scan 执行的。它仅仅意味着满足了这个特定的前提条件。在本章后面，你将读到关于如何验证语句是否确实通过 Smart Scan 卸载执行的方法。


#### 直接路径读取

除了需要全表扫描操作外，智能扫描还要求读取操作通过 Oracle 的直接路径读取机制执行。直接路径读取机制由来已久。传统上，这种读取机制由并行查询服务器进程使用。因为并行查询最初预期用于访问非常大量的数据（通常大到无法放入 Oracle 缓冲区缓存），所以决定并行服务器应直接将数据读入它们自己的内存（也称为程序全局区域或 PGA）。直接路径读取机制完全绕过了将数据块放入缓冲区缓存的标准 Oracle 缓存机制。它依赖于快速对象检查点操作，在“舀取”多块读取之前将脏缓冲区刷新到磁盘。这对于非常大的数据集来说是一件非常好的事情，因为它消除了预期无益的额外工作（缓存可能不会被重用的全表扫描数据），并防止它们将其他数据从缓存中冲刷出去。此外，还消除了硬盘随机寻道的固有延迟。不将读取的缓冲区插入缓冲区缓存也消除大量潜在的 CPU 开销。

这种情况一直持续到 Oracle 11g，那时非并行查询也开始使用直接路径读取。这在当时有点令人惊讶！

因此，智能扫描并不需要并行执行。为串行查询引入直接路径读取无疑有利于通过智能扫描读取数据的 Exadata 方式。之前提到过，`kcfis`（内核缓存文件智能存储）函数深藏在`kcbldrget`（内核缓存块直接读取获取）函数之下。因此，只有在使用直接路径读取机制时，才能执行智能扫描。

串行查询并非总是使用智能扫描——那将极其低效。建立直接路径读取，尤其是在集群环境中，可能是一项耗时的任务。因此，只有在条件合适时才会建立和使用直接路径读取。

一个隐藏参数`_SERIAL_DIRECT_READ`控制此功能。当此参数设置为其默认值(`AUTO`)时，Oracle 会自动决定是否对非并行扫描使用直接路径读取。该计算基于多个因素，包括对象的大小、缓冲区缓存的大小以及对象已有多少块被缓存在缓冲区缓存中。还有一个隐藏参数(`_SMALL_TABLE_THRESHOLD`)在决定表必须多大才会被考虑用于串行直接路径读取中起作用。确定在非并行扫描上是否使用直接路径读取机制的算法并未公开。稍加挖掘，你可以发掘出部分决策过程。在数据库的近期版本中，你可以跟踪一个名为`NSMTIO`的 RDBMS 内核工具。可以调用低级的`oradebug`实用程序来显示数据库中可跟踪的组件，其中一个顶级组件名为 KXD——Exadata 特定内核模块(kxd)：

```sql
SQL> oradebug doc component kxd
KXD                        Exadata specific Kernel modules (kxd)
KXDAM                      Exadata Disk Auto Manage (kxdam)
KCFIS                      Exadata Predicate Push (kcfis)
NSMTIO                     Trace Non Smart I/O (nsmtio)
KXDBIO                     Exadata Block level Intelligent Operations (kxdbio)
KXDRS                      Exadata Resilvering Layer (kxdrs)
KXDOFL                     Exadata Offload (kxdofl)
KXDMISC                    Exadata Misc (kxdmisc)
KXDCM                      Exadata Metrics Fixed Table Callbacks (kxdcm)
KXDBC                      Exadata Backup Compression for Backup Appliance (kxdbc)
```

跟踪`KXD.*`从研究角度看相当有趣，但由于它可能生成巨大的跟踪文件，绝不应在实验室环境之外进行。`NSMTIO`子组件包含有关直接路径读取决策的有趣信息。这里显示的第一个跟踪是关于一个变成了智能扫描的直接路径读取：

```sql
SQL> select value from v$diag_info
  2  where name like 'Default%';
VALUE
------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/dbm01/dbm011/trace/dbm011_ora_32020.trc

SQL> alter session set events 'trace[nsmtio]';
Session altered.
Elapsed: 00:00:00.00

SQL> select count(*) from bigtab;
  COUNT(*)
----------
 256000000
Elapsed: 00:00:08.10

SQL> !cat /u01/app/oracle/diag/rdbms/dbm01/dbm011/trace/dbm011_ora_32020.trc
NSMTIO: kcbism: islarge 1 next 0 nblks 10250504 type 3, bpid 65535, kcbisdbfc 0 kcbnhl 262144 kcbstt 44648 keep_nb 0 kcbnbh 2232432 kcbnwp 3
NSMTIO: kcbism: islarge 1 next 0 nblks 10250504 type 2, bpid 3, kcbisdbfc 0 kcbnhl 262144 kcbstt 44648 keep_nb 0 kcbnbh 2232432 kcbnwp 3
NSMTIO: kcbimd: nblks 10250504 kcbstt 44648 kcbpnb 223243 kcbisdbfc 3 is_medium 0
NSMTIO: kcbivlo: nblks 10250504 vlot 500 pnb 2232432 kcbisdbfc 0 is_large 0
NSMTIO: qertbFetch:[MTT < OBJECT_SIZE < VLOT]:
Checking cost to read from caches(local/remote) and checking storage reduction factors (OLTP/EHCC Comp)
NSMTIO: kcbdpc:DirectRead: tsn: 7, objd: 34422, objn: 20491 ckpt: 1, nblks: 10250504, ntcache: 2173249, ntdist:2173249
Direct Path for pdb 0 tsn 7  objd 34422 objn 20491
Direct Path 1 ckpt 1, nblks 10250504 ntcache 2173249 ntdist 2173249
Direct Path mndb 0 tdiob 6 txiob 0 tciob 43
Direct path diomrc 128 dios 2 kcbisdbfc 0
NSMTIO: Additional Info: VLOT=11162160
Object# = 34422, Object_Size = 10250504 blocks
SqlId = 8c9rzdry8yahs, plan_hash_value = 2140185107, Partition# = 0
```

`BIGTAB`相对较大，有 10250504 个块。早期的 Exadata 软件版本会执行段头的单块读取来确定对象大小。自 11.2.0.2 起，隐藏参数`_direct_read_decision_statistics_driven`被设置为`TRUE`，这意味着将改为查阅字典统计信息。有 2173249 个块被缓存在缓冲区缓存中，这似乎在这里没有起作用。然而，如果有太多块被缓存，则可能会选择带缓冲的访问路径，而不是直接路径读取。

表访问函数(`qertbFetch`)报告该对象大于 MTT（中等表阈值）而小于 VLOT（非常大对象阈值）。幸运的是，这里显示了相关语句的`SQL_ID`和计划哈希值，以及分区信息。

不幸的是，中等表阈值在区间定义[`MTT < OBJECT_SIZE < VLOT`]中有点误导。MTT 计算为`_small_table_threshold`（STT）的五倍，乍一看似乎是开始考虑直接路径读取的分界点。这对于早期的 11.2 版本可能是正确的。在 11.2.0.3 及更高版本（包括 12c）中的测试表明，即使段只比 STT 大一点，也符合直接路径读取的条件。此时的决策基于缓冲区缓存中的块数（在 RAC 中会考虑远程和本地）及其类型。这在跟踪中由“checking cost to read from caches (local/remote) and checking storage reduction factors...”这一行指示。

另一方面，如果没有直接路径读取，对于一个非常小的表，你会看到类似这样的内容：

```text
NSMTIO: kcbism: islarge 0 next 0 nblks 4 type 3, bpid 65535, kcbisdbfc 0 kcbnhl 262144 kcbstt 48117 keep_nb 0 kcbnbh 2405898 kcbnwp 3
NSMTIO: kcbism: islarge 0 next 0 nblks 4 type 2, bpid 3, kcbisdbfc 0 kcbnhl 262144 kcbstt 48117 keep_nb 0 kcbnbh 2405898 kcbnwp 3
NSMTIO: qertbFetch:NoDirectRead :[- STT < OBJECT_SIZE < MTT]:Obect’s size: 4 (blocks), Threshold: MTT(240589 blocks),
```



