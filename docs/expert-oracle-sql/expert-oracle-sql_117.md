# 多列索引的优化策略

## 查询执行计划分析

| Id | Operation | Name | Cost (%CPU) |
|---|---|---|---|
| 0 | SELECT STATEMENT | | 4 (0) |
| 1 | NESTED LOOPS | | 4 (0) |
| 2 | TABLE ACCESS BY INDEX ROWID | ORDERS | 1 (0) |
| *3 | INDEX UNIQUE SCAN | ORDER_PK | 0 (0) |
| 4 | TABLE ACCESS BY INDEX ROWID BATCHED | ORDER_ITEMS | 3 (0) |
| *5 | INDEX RANGE SCAN | ITEM_ORDER_IX | 1 (0) |

```sql
SELECT /*+ index(i order_items_pk) */ *
  FROM oe.orders JOIN oe.order_items i USING (order_id)
 WHERE order_id = 2400;
```

| Id | Operation | Name | Cost (%CPU) |
|---|---|---|---|
| 0 | SELECT STATEMENT | | 4 (0) |
| 1 | NESTED LOOPS | | 4 (0) |
| 2 | TABLE ACCESS BY INDEX ROWID | ORDERS | 1 (0) |
| *3 | INDEX UNIQUE SCAN | ORDER_PK | 0 (0) |
| 4 | TABLE ACCESS BY INDEX ROWID BATCHED | ORDER_ITEMS | 3 (0) |
| *5 | INDEX RANGE SCAN | ORDER_ITEMS_PK | 1 (0) |

清单 15-1 中的第一个查询展示了人们认为`ITEM_ORDER_IX`索引设计所针对的那种查询：对特定`ORDER_ID`的查询。然而，该清单中的第二个查询使用了一个提示来强制使用支持主键约束的多列索引。强制主键的索引名为`ORDER_ITEMS_PK`。`ORDER_ITEMS_PK`索引是一个多列唯一索引，其前导列是`ORDER_ID`，而报告估算的使用`ORDER_ITEMS_PK`的成本与使用`ITEM_ORDER_IX`的成本相同：我们不需要两个索引。

当然，`ITEM_ORDER_IX`索引比`ORDER_ITEMS_PK`索引略小，因此实际上使用`ITEM_ORDER_IX`可能比使用`ORDER_ITEMS_PK`带来轻微的性能提升。然而，在实践中，维护`ITEM_ORDER_IX`的 DML 开销将远远超过该索引为查询带来的好处，如果这是一个真实的表，我会删除单列的`ITEM_ORDER_IX`索引。

![image](img/sq.jpg) **注意** 如果删除了`ITEM_ORDER_IX`列，当删除`ORDERS`表中的行时，仍然不会有表锁。这是因为`ORDER_ID`是`ORDER_ITEMS_PK`的前导列，因此`ORDER_ITEMS_PK`可以防止表锁。

## 多列非唯一索引的正确使用

如果你需要在一个列上创建非唯一 B-tree 索引，那么几乎总是有意义添加额外的列到索引中，以帮助那些使用涉及多列谓词的查询。这是因为向索引添加一两个列通常对 DML 或查询增加的开销相对较小，但添加额外的索引对 DML 来说非常昂贵。清单 15-2 展示了在`OE.CUSTOMERS`表上使用一个潜在合法的索引。

清单 15-2. 多列索引的正确使用

```sql
SELECT *
  FROM oe.customers
 WHERE UPPER (cust_last_name) = 'KANTH';

| Id | Operation | Name | Rows | Cost (%CPU) |
|---|---|---|---|---|
| 0 | SELECT STATEMENT | | 2 | 4 (0) |
| 1 | TABLE ACCESS BY INDEX ROWID BATCHED | CUSTOMERS | 2 | 4 (0) |
| *2 | INDEX RANGE SCAN | CUST_UPPER_NAME_IX | 2 | 2 (0) |

Predicate Information (identified by operation id):
2 - access(UPPER("CUST_LAST_NAME")='KANTH')
```

```sql
SELECT *
  FROM oe.customers
 WHERE     UPPER (cust_first_name) = 'MALCOLM'
       AND UPPER (cust_last_name) = 'KANTH';

| Id | Operation | Name | Rows | Cost (%CPU) |
|---|---|---|---|---|
| 0 | SELECT STATEMENT | | 1 | 2 (0) |
| 1 | TABLE ACCESS BY INDEX ROWID BATCHED | CUSTOMERS | 1 | 2 (0) |
| *2 | INDEX RANGE SCAN | CUST_UPPER_NAME_IX | 1 | 1 (0) |

Predicate Information (identified by operation id):
2 - access(UPPER("CUST_LAST_NAME")='KANTH' AND
              UPPER("CUST_FIRST_NAME")='MALCOLM')
```

```sql
SELECT *
  FROM oe.customers
 WHERE UPPER (cust_first_name) = 'MALCOLM';

| Id | Operation | Name | Rows | Cost (%CPU) |
|---|---|---|---|---|
| 0 | SELECT STATEMENT | | 2 | 5 (0) |
| *1 | TABLE ACCESS FULL | CUSTOMERS | 2 | 5 (0) |

Predicate Information (identified by operation id):
1 - filter(UPPER("CUST_FIRST_NAME")='MALCOLM')
```

`CUST_UPPER_NAME_IX`是一个基于函数的多列索引，包含两个表达式：`UPPER (CUST_LAST_NAME)`和`UPPER (CUST_FIRST_NAME)`。清单 15-2 表明，当查询只使用第一个表达式的谓词或使用两个表达式的谓词时，都可以使用`INDEX RANGE SCAN`。但是当查询中只使用第二个表达式时，只有昂贵且不合适的`INDEX SKIP SCAN`可用，因此 CBO 选择进行全表扫描。

让我以一个提醒来结束本节：如果你的查询只从索引中选择列，则根本不需要访问表。这意味着，即使某些查询没有包含被添加列的谓词，通过向索引添加列，这些查询的性能也可能得到提升。

## 多列索引中的列顺序

正如清单 15-2 所示，确保常用的列或表达式出现在索引的前导边缘非常重要。但是，如果你的查询包含了指定索引中所有列的谓词，那么索引中列的顺序是否重要呢？一般来说，答案是否定的：无论索引中列的顺序如何，你的索引范围扫描都将扫描完全相同数量的索引条目。然而，除了那些只指定部分索引列的查询外，在确定索引的最佳列顺序时还有另外三个考虑因素：

*   如果你只想压缩索引列的一个子集，那么这些列需要出现在索引的前导边缘。我将在本章后面介绍索引压缩。
*   如果查询例行地使用索引中的多个列对数据进行排序，那么在索引中正确排序列可能会避免显式排序。我将在第 19 章中演示如何使用索引来避免排序。
*   从理论上讲，如果数据是以有序或半有序的方式加载的，那么按照与表数据排序相同的方式来排列索引中的列可能会改善索引聚簇因子（从而改善查询性能）。我必须说，我个人从未遇到过在选择索引列顺序时聚簇因子是一个需要关注的问题的情况。

![image](img/sq.jpg) **注意** 《性能调优指南》指出，你应该“将最具选择性的列放在最前面”。这是一个著名的文档错误，截至 12cR1 仍未纠正。

#### 位图索引

到目前为止，我们对索引策略的讨论明确或隐含地都限制在 B-tree 索引上。位图索引往往非常小巧，是解决一个重要问题的一种方法。

[内容]
清单 15-2 表明，如果查询不包含对索引首列的谓词，通常无法高效利用多列索引。假设你有一个名为 `PROBLEM_TABLE` 的表。你有几十个对该表的常用查询，每个查询都指定了来自约六列中的两到三列的一个子集，但没有哪一列是在所有查询中都用到的。你该创建哪些索引？正是这种情况，导致一些开发人员为每个列子集创建数十个索引。这不是个好主意。

如果你是批量加载数据，那么位图索引可以解决这个问题：你只需在 `PROBLEM_TABLE` 上创建六个单列位图索引，而我们在 清单 10-18 中展示的 `INDEX COMBINE` 操作就可以用于在访问表之前，识别出所需的行子集。重要的是，不要试图在 DML 操作本身期间维护你的位图索引。清单 15-3 展示了在使用位图索引时加载数据的正确方法。

清单 15-3. 将数据加载到包含位图索引的分区表中

```sql
CREATE TABLE statement_part_ch15
PARTITION BY RANGE
   (transaction_date)
   (
      PARTITION p1 VALUES LESS THAN (DATE '2013-01-05')
     ,PARTITION p2 VALUES LESS THAN (DATE '2013-01-11')
     ,PARTITION p3 VALUES LESS THAN (DATE '2013-01-12')
     ,PARTITION p4 VALUES LESS THAN (maxvalue))
AS
   SELECT transaction_date_time
         ,transaction_date
         ,posting_date
         ,description
         ,transaction_amount
         ,product_category
         ,customer_category
     FROM statement;

CREATE BITMAP INDEX statement_part_pc_bix
   ON statement_part_ch15 (product_category)
   LOCAL;

CREATE BITMAP INDEX statement_part_cc_bix
   ON statement_part_ch15 (customer_category)
   LOCAL;

ALTER TABLE statement_part_ch15 MODIFY PARTITION FOR (DATE '2013-01-11') UNUSABLE LOCAL INDEXES;

INSERT INTO statement_part_ch15 (transaction_date_time
                                ,transaction_date
                                ,posting_date
                                ,description
                                ,transaction_amount
                                ,product_category
                                ,customer_category)
   SELECT DATE '2013-01-11'
         ,DATE '2013-01-11'
         ,DATE '2013-01-11'
         ,description
         ,transaction_amount
         ,product_category
         ,customer_category
     FROM statement_part_ch15;

ALTER TABLE statement_part_ch15 MODIFY PARTITION FOR (DATE '2013-01-11')
            REBUILD UNUSABLE LOCAL INDEXES;
```

清单 15-3 使用我们在第 9 章创建的 `STATEMENT` 表中的数据，创建了一个分区表 `STATEMENT_PART_CH15`。在将数据加载到之前为空的、`TRANSACTION_DATE` 为 2013 年 1 月 11 日的分区之前，该分区的索引被标记为不可用。然后，在不维护索引的情况下插入数据，最后，重建该分区的索引（特别是位图索引）。一旦你的位图索引重建完毕，它们就可以投入使用了。

我想强调一下，位图索引只有在你先加载数据、再重建索引、最后查询数据的情况下才具有实用价值。这在数据仓库应用中通常非常实用，但是对*可用的*位图索引进行 DML 操作开销很大，并发性也很差。这意味着位图索引通常对于大多数其他类型的应用程序来说是不切实际的。

### 位图索引的替代方案

让我们回到那个 `PROBLEM_TABLE`，它的六列被各种问题查询的谓词所使用。假设现在并发的 DML 和查询使得位图索引不切实际。那该怎么办？当我们单独考虑每个查询时，最好的做法是创建一个多列索引，包含该查询中的所有列或表达式。但正如我已经强调过的，这种针对每个查询的自动化反应重复进行，可能会导致索引泛滥，从而严重降低 DML 性能。那么，有替代方案吗？

你可以考虑创建六个单列 B 树索引，因为正如我在清单 10-18 中所示，`INDEX COMBINE` 操作既可以用于位图索引，也可以用于 B 树索引。然而，将 B 树索引条目转换为位图的相关操作可能代价高昂，因此六个单列 B 树索引不太可能是个好主意。你也可以只创建一个包含所有六列的大索引。这样，如果某个特定查询未使用索引的前导列，则可以使用 `INDEX FULL SCAN` 或 `INDEX SKIP SCAN` 作为替代。

归根结底，你可能不得不接受这样一个事实：对于某些查询，即使你知道使用索引的嵌套循环连接会使查询性能更好，你也只能忍受全表扫描和哈希连接。

如果你真的非常需要优化，我将在第 19 章向你展示如何重写 SQL 来解决这个问题。

### 索引组织表

在我看来，索引组织表（IOT）在以下两个条件都满足时使用效果最佳：

-   你的表（或表分区）非常小（小于 50MB）或非常窄（主键至少占行大小的 10%）
-   除了支持主键的那个索引外，你不需要任何其他索引

如果这两个条件都满足，你就可以将表的所有列存储在索引中，从而完全避免对实际表的需要。

可以在 IOT 上创建额外的索引，但正如我在第 3 章中简要提到的，IOT 上的二级索引保存了对所引用行所在位置的“猜测”，因此当“猜测”过时时，它们可能需要频繁重建。一些专家比我更看好 IOT。Martin Widlake 是 IOT 的忠实粉丝。你可以从他的这篇博文开始阅读更多内容：`http://mwidlake.wordpress.com/2011/07/18/index-organized-tables-the-basics/`。

### 管理争用

Oracle 数据库架构的一大优点是写入器不会阻塞读取器。唉，但写入器会阻塞其他写入器！因此，本节讨论由于多个进程并发地对同一数据库对象执行 DML 而可能出现的性能问题。

#### 序列争用

你应该努力设计你的应用程序，使得序列生成的数字出现间隔不是问题，并且如果数字从序列中乱序生成也不会发生特别糟糕的事情。这种努力在 RAC 环境中尤其重要。一旦你对应用程序的稳定性有信心，就可以修改序列并指定 `CACHE` 和 `NOORDER` 属性。这将确保在 RAC 集群中分配序列号时，节点间的通信最小化。

### 热块问题

当多个进程并发地向表中插入行时，无论你是使用自动段空间管理（`ASSM`）还是手动段空间管理（`MSSM`），这些行通常都会被插入到表中的不同块中。然而，多个进程经常试图同时更新同一个索引块。当索引的前导列是时间戳或从序列生成的数字时，这种情况尤其明显。


如果你尝试访问一个缓冲区，而它正被另一个会话更新，你就必须等待。在 Oracle 9i 及更早版本中，你总是会看到这种等待表现为一个“`buffer busy waits`”事件，但从 10.1 及以后版本开始，第二个等待事件被分离出来。“`read by other session`”等待事件意味着你试图访问的缓冲区正被另一个会话更新，并且该块正由那个会话从磁盘读入缓冲区缓存。“`buffer busy waits`”事件和“`read by other session`”等待事件都是对数据块争用的迹象，尽管当有多个会话并发读取同一个块时，你也可能看到“`read by other session`”等待事件。

当多个进程并发尝试更新同一个块时，该块被称为`热块`。热块并非总是出现在索引中，但大多数情况下是。让我们通过看一个特定于索引的解决方案来开始我们对热块争用的讨论。

## 反向键索引

理论上，这个想法简单而优雅。想象你有一个在列`C1`上的索引`I1`，并且你通过从一个序列获取值来填充`C1`的值。三个进程可能从序列中生成值 123,456、123,457 和 123,458，然后将行插入到你的表中。在传统索引中，索引条目将是连续的，并且很可能这些进程会争用以更新同一个索引叶块。如果你使用`REVERSE`关键字创建索引，索引条目中的字节将在插入前被反转。你可以这样想象：索引条目 654,321、754,321 和 854,321。如你所见，这些索引条目甚至不再接近连续，它们将被插入到不同的块中。这并非完全准确：是索引条目中的字节被反转，而不是数字中的位数，但概念是一样的。这看起来几乎像变魔术一样，你似乎瞬间解决了你的争用问题。

现实情况并不像最初看起来那么优雅。有两个大问题。第一个问题是`INDEX RANGE SCAN`（索引范围扫描）被限制在`C1`的特定值。如果你有一个带有谓词`WHERE C1 BETWEEN 123456 AND 123458`的查询，CBO 将无法使用范围扫描来找到这三个索引条目，因为它们分散在索引各处。

反向键索引的第二个问题是，热块问题现在变成了冷块问题。在反转索引之前，进程之间可能存在争用，但至少只需要将一个索引块读入 SGA。当你反转索引时，索引条目将被插入到各处，除非索引的所有叶块都能被缓存在 SGA 中，否则索引的磁盘读取急剧增加的风险是真实存在的。

我要感谢 Oracle 的“真实世界性能”团队，他们让我了解到一个可能更优的热索引块问题解决方案。那就是全局分区索引。

#### 全局分区索引

无论底层表是否分区，你都可以以一种完全独立于底层表任何分区策略的方式来分区索引。这样的索引被称为`全局分区索引`。清单 15-4 展示了如何做到这一点。

**清单 15-4. 创建一个全局分区索引**

```sql
CREATE TABLE global_part_index_test
(
   c1   INTEGER
  ,c2   VARCHAR2 (50)
  ,c3   VARCHAR2 (50)
);

CREATE UNIQUE INDEX gpi_ix
   ON global_part_index_test (c1)
   GLOBAL PARTITION BY HASH (c1)
      PARTITIONS 32;

ALTER TABLE global_part_index_test ADD CONSTRAINT global_part_index_test_pk PRIMARY KEY (c1);

SELECT *
  FROM global_part_index_test
 WHERE c1 BETWEEN 123456 AND 123458;

| Id | Operation                            | Name                   |

|  0 | SELECT STATEMENT                     |                        |
|  1 |  PARTITION HASH ALL                  |                        |
|  2 |   TABLE ACCESS BY INDEX ROWID BATCHED| GLOBAL_PART_INDEX_TEST |
|  3 |    INDEX RANGE SCAN                  | GPI_IX                 |
```

清单 15-4 创建了用于支持主键约束的索引，将其作为全局哈希分区索引。对索引的插入将分散在 32 个分区中，从而显著降低争用水平，就像反向键索引一样。但与我们的反向键索引不同，我们没有失去对多个`C1`值执行范围扫描的能力。从执行计划的第 1 行可以看到，运行时引擎需要检查所有 32 个分区，因此索引范围扫描比未分区的索引成本更高，但至少它们是可行的。

我们也解决了冷块问题。想象一下，向一个大型表插入 1,024 行数据，`C1`值从 1 到 1,024。如果`C1`是用反向键索引索引的，我们很可能会将索引条目插入到 1,024 个不同的块中。另一方面，假设我们能在单个块中存储超过 32 个索引条目，那么当`C1`像清单 15-4 中那样被索引时，我们的 1,024 个索引条目将仅存储在 32 个块中。这是因为一个哈希分区的`C1`值（大约 32 个）在该分区内将是连续的！所以，正如 Graham Wood 所说，我们的块是“温的”，并且这 32 个块很可能可以保持在缓冲区缓存中。

全局分区索引有缺点吗？是的。它们难以维护——甚至比在分区表上维护未分区的全局索引更困难。

想象你有一个 1TB 的表。你创建了五个全局哈希分区索引，每个索引使用 32 个哈希分区。现在你升级了硬件，并使用数据泵导出/导入将 1TB 表迁移到新机器上，然后尝试重建索引。你需要扫描整个 1TB 表来重建每个索引分区，无论表上有什么分区策略。那就是 5 x 32 = 160 次对你的 1TB 表的全表扫描！

解决重建问题的一个方案是，按哈希对表进行（子）分区，然后使用本地索引。对表进行（子）分区还有一个额外好处，就是降低表本身发生争用的风险。

## 初始 ITL 条目

当事务第一次更新一个块时，它会使用块中感兴趣事务列表（ITL）里的一个条目。如果一个块中的所有 ITL 条目都被未提交的事务占用，并且块中有空间添加新条目，那么就会添加一个新的 ITL 条目。一个 ITL 条目消耗 24 字节，因此如果你预分配 100 个条目，在一个 8K 的块中大约会浪费四分之一的空间。默认情况下，表块的初始 ITL 槽数量为 1，索引块为 2。如果你看到大量名为“`enq: TX - allocate ITL entry`”的等待事件，那么等待进程由于块内空间不足而无法分配新的 ITL 条目。解决此问题的直接方法是增加索引或表中块最初分配的 ITL 条目数量。要将索引`I1`的初始 ITL 条目数从两个增加到五个，语法是`ALTER INDEX I1 INITRANS 5`，但请记住以下几点：

*   对`INITRANS`的更改不会影响索引或表中已格式化的块。你可能需要重建索引或移动表。
*   解决了 ITL 争用问题，并不意味着你不会再遇到其他争用问题。



如果您遇到 ITL（兴趣事务列表）条目争用问题，可能更好的方法是避免问题而非解决它：使用反向键或全局哈希分区索引可能同时消除您的 ITL 问题，因为并发添加的新索引条目很可能位于不同的块，因此也位于不同的 ITL 中。现在让我们看看另一种减少块争用的方法。

## 分散数据

仅仅因为两个会话试图更新同一个表块，并不意味着它们试图更新该块中的同一行。同样，仅仅因为两个会话试图更新同一个索引块，也不意味着它们试图更新同一个索引条目。如果您的数据库对象很小，您可以尝试减少每个表块的行数或每个索引块的索引条目数以缓解争用。

假设您有 50 个会话，每个会话都频繁更新表中一个特定于会话的小行。如果不做特殊处理，所有 50 行最终都会位于同一个块中，您可能会看到大量的`缓冲区忙等待`事件。由于表非常小，您或许可以负担得起将行分散开，每行一个块以避免争用。实际上有几种方法可以实现这一点。清单 15-5 展示了其中一种方法。

### 清单 15-5. 确保每个块只有一行

```sql
CREATE TABLE process_status
(
   process_id                 INTEGER PRIMARY KEY
  ,status                     INTEGER
  ,last_job_id                INTEGER
  ,last_job_completion_time   DATE
  ,filler_1                   CHAR (100) DEFAULT RPAD (' ', 100)
)
PCTFREE 99
PCTUSED 1;

INSERT INTO process_status (process_id)
       SELECT ROWNUM
         FROM DUAL
   CONNECT BY LEVEL <= 50;

BEGIN
   DBMS_STATS.gather_table_stats (
      ownname     => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'PROCESS_STATUS');
END;
/

UPDATE process_status
     SET status = 1, last_job_id = 1, last_job_completion_time = SYSDATE
 WHERE process_id = 1;
```

```
| 编号 | 操作                 | 名称             |
|---|---|---|
|   0 | UPDATE STATEMENT   |                |
|   1 |  UPDATE            | PROCESS_STATUS |
|   2 |   INDEX UNIQUE SCAN| SYS_C005053    |
```

清单 15-5 创建了一个名为`PROCESS_STATUS`的表，该表存储 50 行数据，每行对应一个应用程序进程。我们尝试减少`PROCESS_STATUS`表每个块中的行数，方法是将`PCTFREE`设置为 99，这意味着一旦 8K 块的 1%被使用，就不再向表中插入行。但我们的行长度小于 81 字节，因此我们需要添加一个额外的列，我称之为`FILLER_1`。现在，当我们插入 50 行时，它们都将进入不同的块。任何针对单行的`UPDATE`语句都将通过索引找到专用的块，并且不会与更新表中其他行的其他会话发生争用。

### 分区

专家之间轻松辩论的一个热门话题是，分区选项的主要优势是与管理简便性相关，还是与性能提升相关。当然，分区既提供了管理方面的优势，也提供了性能方面的优势。我们已经讨论了全局哈希分区索引和哈希分区表如何帮助缓解热点块争用。现在让我们看看分区带来的其他一些与性能相关的优势。

### 对分区或子分区的全表扫描

想象一下，您的表存储了 10 个工作日的 100,000 行数据，总计 1,000,000 行。假设每个块大约可以容纳 100 行，那么您的表总共有大约 10,000 个块。让我们以乐观的态度看待这个问题，并假设所有行都按业务日期完美聚簇，使得特定业务日期的所有行都位于 10,000 个块中的 1,000 个块中。现在假设您想要读取特定业务日期的所有 10,000 行。为简化起见，我们假设缓冲区缓存中没有任何数据。您认为索引访问还是全表扫描更好？

实际上，两种选择都不太理想。如果您使用全表扫描，那么您将读取并丢弃 10,000 个块中的 9,000 个块。如果您使用索引读取表块，那么您只会访问所需的 1,000 个块，但它们将通过 1,000 次单块读取来读取。

我们真正想要的是，通过多块读取读取选定业务日期的 1,000 个块，而不必理会其他 9,000 个块。正如我将在第 19 章中展示的，可以重写 SQL 来实现这一点，但这非常棘手。如果您按业务日期对表进行分区，您将有 10 个表段，每个段大小为 1,000 个块，而不是一个单独的 10,000 块的段。在分区消除之后，您的查询可以对单个 1,000 块的段执行一次全表扫描。如果这 1,000 个块已经在缓冲区缓存中，那么通过索引访问块与通过全表扫描访问块之间可能不会有巨大的性能差异。但如果选定的 1,000 个块都需要从磁盘读取，那么全表扫描将使用多块读取这一事实意味着它将比使用索引快得多。

因此，分区使全表扫描更具吸引力，这反过来可能使一些原本需要的索引变得多余。一举两得！

#### 分区智能连接

如果您连接两个以相同方式分区的表，并且连接谓词中包含了分区列，那么您可以执行所谓的*完全分区智能连接*。清单 15-6 展示了一个正在运行的完全分区智能连接。

#### 清单 15-6. 串行执行的完全分区智能连接

```sql
CREATE TABLE orders_part
(
   order_id           INTEGER NOT NULL
  ,order_date         DATE NOT NULL
  ,customer_name      VARCHAR2 (50)
  ,delivery_address   VARCHAR2 (100)
)
PARTITION BY LIST
   (order_date)
   SUBPARTITION BY HASH (order_id)
      SUBPARTITIONS 16
   (
      PARTITION p1 VALUES (DATE '2014-04-01')
     ,PARTITION p2 VALUES (DATE '2014-04-02'));

CREATE UNIQUE INDEX orders_part_pk
   ON orders_part (order_date, order_id)
   LOCAL;

ALTER TABLE orders_part
   ADD CONSTRAINT orders_part_pk PRIMARY KEY
  (order_date,order_id);

CREATE TABLE order_items_part
(
   order_id        INTEGER NOT NULL
  ,order_item_id   INTEGER NOT NULL
  ,order_date      DATE NOT NULL
  ,product_id      INTEGER
  ,quantity        INTEGER
  ,price           NUMBER
  ,CONSTRAINT order_items_part_fk FOREIGN KEY
      (order_date, order_id)
       REFERENCES orders_part (order_date, order_id)
)
PARTITION BY REFERENCE (order_items_part_fk);

CREATE UNIQUE INDEX order_items_part_pk
   ON order_items_part (order_date, order_id, order_item_id)
   COMPRESS 1
   LOCAL;

ALTER TABLE order_items_part
   ADD CONSTRAINT order_items_part_pk PRIMARY KEY
  (order_date,order_id,order_item_id);

SELECT /*+ leading(o i) use_hash(i) no_swap_join_inputs(i) */
       *
  FROM orders_part o JOIN order_items_part i USING (order_id);
```

```
| 编号 | 操作                 | 名称             |
```



