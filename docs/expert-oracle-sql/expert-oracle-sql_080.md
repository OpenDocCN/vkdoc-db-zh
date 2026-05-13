# CBO 如何使用分区级统计信息

现在我们已经了解了什么是分区级统计信息以及如何获取它们，接下来该看看 CBO 如何使用它们了。许多人似乎将 CBO 对分区级统计信息的使用与运行时引擎的分区消除（partition elimination）概念混淆了。清单 9-17 到 9-20 应该有助于澄清这一点。

## 运行时引擎的分区消除与 CBO 的分区统计信息

**[清单 9-17]**. 运行时引擎的分区消除与 CBO 的分区统计信息

```sql
SELECT *
  FROM statement_part
 WHERE     transaction_date = DATE '2013-01-01'
       AND posting_date = DATE '2013-01-01';

| Id  | Operation              | Name           | Rows  | Bytes | Time     | Pstart| Pstop |
|---  |---                     |---             |---    |---    |---       |---    |---    |
|   0 | SELECT STATEMENT       |                |     8 |   424 | 00:00:01 |       |       |
|   1 |  PARTITION RANGE SINGLE|                |     8 |   424 | 00:00:01 |     1 |     1 |
|*  2 |   TABLE ACCESS FULL    | STATEMENT_PART |     8 |   424 | 00:00:01 |     1 |     1 |

SELECT *
  FROM statement_part
 WHERE     transaction_date = DATE '2013-01-06'
       AND posting_date = DATE '2013-01-06';

| Id  | Operation              | Name           | Rows  | Bytes | Time     | Pstart| Pstop |
|---  |---                     |---             |---    |---    |---       |---    |---    |
|   0 | SELECT STATEMENT       |                |     6 |   324 | 00:00:01 |       |       |
|   1 |  PARTITION RANGE SINGLE|                |     6 |   324 | 00:00:01 |     2 |     2 |
|*  2 |   TABLE ACCESS FULL    | STATEMENT_PART |     6 |   324 | 00:00:01 |     2 |     2 |
```

由于查询中的谓词 `TRANSACTION_DATE`，CBO 精确地知道将访问哪个分区。在第一个查询中，CBO 知道只会引用第一个分区 `P1`。CBO 通过在操作表的 `PSTART`（分区起始）和 `PSTOP`（分区终止）列中列出数字 1 来告诉我们这一点。因此，CBO 可以在其计算中使用 `P1` 的分区级统计信息。`P1` 的分区级统计信息表明：

*   分区 `P1` 中有 300 行。
*   分区 `P1` 中 `TRANSACTION_DATE` 有六个不同的值，`POSTING_DATE` 也有六个不同的值。
*   因此，我们得到的估计基数（cardinality）为 300/6/6 = 8.33 行（四舍五入为 8）。

清单 9-17 中的第二个查询，CBO 已知它使用第二个分区，因此使用了 `P2` 的统计信息：

*   分区 `P2` 中有 200 行。
*   分区 `P2` 中 `TRANSACTION_DATE` 有四个不同的值，`POSTING_DATE` 有八个不同的值。
*   因此，我们得到的估计基数为 200/4/8 = 6.25 行（四舍五入为 6）。

两个查询不同的基数估计清楚地证明，CBO 在计算中使用了分区级统计信息。清单 9-18 向我们展示了第三个查询，它与 清单 9-17 中的查询仅有细微差别，但这个差别至关重要。

## 运行时引擎的分区消除与 CBO 对全局统计信息的使用

**[清单 9-18]**. 运行时引擎的分区消除与 CBO 对全局统计信息的使用

```sql
SELECT *
  FROM statement_part
 WHERE     transaction_date = (SELECT DATE '2013-01-01' FROM DUAL)
       AND posting_date = DATE '2013-01-01';

| Id  | Operation              | Name           | Rows  | Bytes | Time     | Pstart| Pstop |
|---  |---                     |---             |---    |---    |---       |---    |---    |
|   0 | SELECT STATEMENT       |                |     4 |   220 | 00:00:01 |       |       |
|   1 |  PARTITION RANGE SINGLE|                |     4 |   220 | 00:00:01 |   KEY |   KEY |
|*  2 |   TABLE ACCESS FULL    | STATEMENT_PART |     4 |   220 | 00:00:01 |   KEY |   KEY |
|   3 |    FAST DUAL           |                |     1 |       | 00:00:01 |       |       |
```

清单 9-18 没有使用 `TRANSACTION_DATE` 值的字面量。它使用了一个表达式。这个简单的改变意味着 CBO 无法追踪将引用哪个分区，这一事实在操作表的 `PSTART` 和 `PSTOP` 列中使用了“`KEY`”一词中变得清晰。由于这种不确定性，CBO 回退到使用全局统计信息，如下所示：

*   根据全局统计信息，`STATEMENT_PART` 中有 500 行。
*   根据全局统计信息，`STATEMENT_PART` 中 `TRANSACTION_DATE` 有 10 个不同的值。
*   根据全局统计信息，`STATEMENT_PART` 中 `POSTING_DATE` 有 12 个不同的值。
*   因此，我们得到的估计基数为 500/10/12 = 4.17 行（四舍五入为 4）。

我想绝对明确一点：CBO *确实* 知道只会引用一个分区。这就是为什么它能够指定 `PARTITION RANGE SINGLE` 操作。运行时引擎将能够将其访问限制在一个分区。然而，运行时引擎将使用 *哪个* 分区，直到实际运行那个小的子查询之前仍然是未知的。因此，即使在运行时只访问一个分区，CBO 也需要使用全局统计信息来估计基数，因为它不知道使用哪一组分区级统计信息。

## 引用多个分区时对全局统计信息的使用

前两个清单涵盖了仅访问单个分区的查询。清单 9-19 展示了一个引用多个分区的查询：

**[清单 9-19]**. 引用多个分区时对全局统计信息的使用

```sql
SELECT *
  FROM statement_part
 WHERE     transaction_date IN (DATE '2013-01-04', DATE '2013-01-05')
       AND posting_date = DATE '2013-01-05';

| Id  | Operation              | Name           | Rows  | Bytes | Time     | Pstart| Pstop |
|---  |---                     |---             |---    |---    |---       |---    |---    |
|   0 | SELECT STATEMENT       |                |     8 |   440 | 00:00:01 |       |       |
|   1 |  PARTITION RANGE INLIST|                |     8 |   440 | 00:00:01 |KEY(I) |KEY(I) |
|*  2 |   TABLE ACCESS FULL    | STATEMENT_PART |     8 |   440 | 00:00:01 |KEY(I) |KEY(I) |
```

清单 9-19 引用了不同分区中 `TRANSACTION_DATE` 的两个值。CBO 知道每个 inlist 值使用的是哪个分区，因此理论上 CBO 可以使用分区 `P1` 的统计信息来估计 `TRANSACTION_DATE` 为 1 月 4 日的行数，并使用 `P2` 的统计信息来估计 `TRANSACTION_DATE` 为 1 月 5 日的行数。这将产生 14 (8+6) 的基数估计。然而，CBO 被设计为一次只使用一组统计信息，因此对两个日期都使用了全局统计信息。因此基数为 8 (4+4)。注意操作表中的 `PSTART` 和 `PSTOP` 列的值为 `KEY (I)`，表示存在 inlist。

## 不存在分区级统计信息时

最后一个原因可能导致使用全局统计信息，那就是分区级统计信息不存在！为了完整起见，清单 9-20 演示了这种情况。

**[清单 9-20]**. 删除分区级统计信息

```sql
BEGIN
   DBMS_STATS.delete_table_stats (
      ownname    => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname    => 'STATEMENT_PART'
     ,partname   => 'P1');
END;
/

SELECT *
  FROM statement_part
 WHERE     transaction_date = DATE '2013-01-01'
       AND posting_date = DATE '2013-01-01';

| Id  | Operation              | Name           | Rows  | Bytes | Time     | Pstart| Pstop |
|---  |---                     |---             |---    |---    |---       |---    |---    |
|   0 | SELECT STATEMENT       |                |     4 |   220 | 00:00:01 |       |       |
|   1 |  PARTITION RANGE SINGLE|                |     4 |   220 | 00:00:01 |     1 |     1 |
|*  2 |   TABLE ACCESS FULL    | STATEMENT_PART |     4 |   220 | 00:00:01 |     1 |     1 |
```



清单 9-20 显示了与清单 9-17 中相同的查询的执行计划，不同之处在于分区级统计信息已被删除。我们可以看到，基于成本的优化器（CBO）仍然知道要引用的具体分区，但其基数估计为 4，这表明我们像在清单 9-18 中一样，使用了全局统计信息作为估计的基础。

## 为何需要分区级统计信息

既然我已经解释了分区级统计信息是什么，现在可以探讨为何首先需要分区级统计信息。需要明确的是：分区级统计信息并非总是必需的。CBO 通常仅凭全局统计信息就能制定出相当合理的计划。人们之所以费心处理分区级统计信息，主要有三个原因：

*   各分区之间的数据量差异很大，因此在查询不同分区时可能需要不同的执行计划。
*   收集全局统计信息耗时过长。
*   分区级统计信息似乎有助于 CBO 选择性能更优的执行计划。

让我们逐一审视这些原因。

### 分区大小的偏斜

当使用分区选项来实现《*VLDB 与分区指南*》所称的*信息生命周期管理（ILM）* 时，表通常按 `DATE` 类型的列进行分区。其思路是，可以通过删除或导出整个分区来轻松删除或归档旧数据，而无需删除单个行。当以这种方式将分区用于 ILM 时，通常也会带来性能收益，因为查询往往会在分区列上指定谓词，从而可以进行分区消除。ILM 可能是所有分区选项中最常见的用途。

当分区用于 ILM 时，各分区的大小通常相似。然而，当表以其他方式分区时，例如按 `PRODUCT` 或 `CUSTOMER` 分区，则可能出现显著的偏斜，因为例如，贵公司的生存可能依赖于某个特别成功的产品或某个大客户。即使表按日期分区，也可能以某些子分区远大于其他子分区的方式进行子分区。

在极端情况下，您可能需要为小分区制定一种执行计划，而为大分区制定另一种计划。在这些情况下，让 CBO 理解全表扫描对于一个分区来说会非常快，而对于另一个分区则会非常慢，可能是有用的。

分区级统计信息几乎肯定是为了应对此类场景而发明的。尽管如此，虽然并非闻所未闻，但此类情况相当罕见。使用分区级统计信息的通常原因是为了减少统计信息收集时间或提高执行计划的质量。

### 减少收集统计信息的时间

正如我在第 6 章中所解释的，除非您采用类似 `TSTATS` 的部署方法，或者至少将统计信息保持在相对最新的状态，否则您的执行计划将会恶化。无论您的表是否分区，这一点都适用。

假设一个表基于名为 `BUSINESS_DATE` 的列按日分区用于 ILM。

*   每个工作日，您都会向当天的分区加载大量数据，随后查询加载的数据。该应用程序在周末不使用。
*   您保留两周的数据在线。每个周末，您为下一周的五个工作日创建五个新分区，并删除前一周的五个分区。因此，例如，在 2015 年 3 月 14 日星期六，您会创建 2015 年 3 月 16 日星期一至 2015 年 3 月 20 日星期五的分区，并删除 2015 年 3 月 2 日至 2015 年 3 月 6 日的分区，留下 2015 年 3 月 9 日星期一至 2015 年 3 月 13 日星期五的分区在线。执行分区维护后，您会为整个表收集统计信息。

收集统计信息后，您可能会祈求好运，希望可以撑到下一个周末，因为在工作日收集统计信息实在太耗时了。如果是这样，您将遇到以下两个问题之一：

*   如果在收集统计信息时未显式设置 `GRANULARITY` 参数，它将默认为 `ALL`。当 `GRANULARITY` 设置为 `ALL` 时，将为所有分区（包括空分区）收集统计信息。在星期一查询已加载的数据时，CBO 将使用指示星期一分区为空的分区级统计信息，并几乎肯定会得出一个非常糟糕的执行计划。
*   如果您仔细地将 `GRANULARITY` 参数设置为 `GLOBAL`，那么将不会为空分区创建分区级统计信息。您也可以通过在创建新的空分区之前收集统计信息来实现相同的效果。在星期一，您的执行计划可能是合理的，但到了星期五，执行计划将会改变。这是因为 CBO 将查看 `BUSINESS_DATE` 的 `LOW_VALUE` 和 `HIGH_VALUE` 列统计信息。在 2015 年 3 月 21 日星期五，统计信息将显示 `BUSINESS_DATE` 的最低值为 2015 年 3 月 9 日星期一，最大值为 2015 年 3 月 14 日星期五。CBO 将假设表中不可能有任何 2015 年 3 月 21 日星期五的数据。您将遇到我在第 6 章中演示的过时统计信息问题。

必须采取一些措施来确保在工作日期间不会出现意外的执行计划变更。许多人选择在每天数据加载之后、查询之前收集统计信息。这种统计信息收集活动可能会延迟查询的执行，并且在某些情况下，统计信息收集可能是夜间批处理运行耗时的最大贡献者。

为了减轻统计信息收集造成的延迟影响，在这些情况下，通常只为刚加载的分区收集统计信息，并指定 `GRANULARITY` 为 `PARTITION`。只要应用程序中的查询编写时包含了 `BUSINESS_DATE` 上的等式谓词，以这种方式收集分区级统计信息将防止与过时统计信息相关的意外执行计划变更。

我解释的语气可能让您得出结论，我认为仅仅为了减少统计信息收集时间而使用分区级统计信息这一常见做法并不合适。如果这样，您的结论是正确的，并且在第 20 章中我们将深入探讨这一点，因为它是 `TSTATS` 方法的关键要素。我对人们使用分区级统计信息的第三个也是最后一个原因同样持悲观看法。现在让我们来看看它。

### 分区级统计信息带来更优的执行计划

由于分区级统计信息特定于某个分区，直观上，CBO 使用它们可能比使用全局统计信息做出更准确的估计。确实，如果我们查看清单 9-20 中使用全局统计信息的执行计划，其基数估计为 4。这低于清单 9-17 中相同查询使用分区级统计信息时的基数估计 8。由于该查询实际返回的行数为 18，我们可以看到，基于分区级统计信息的估计虽然不完美，但更准确。

即使不一定理解所涉及的所有计算机科学知识，许多人也在现实中经历过基于全局统计信息的执行计划质量不佳的问题，并因此对分区级统计信息产生了不合理的偏爱。但让我们先从这个问题中退后一步。



## CBO 基数估计的问题与解决方案

在分区表 `STATEMENT_PART` 上查询时，CBO 估算基数所面临的问题，与在 `STATEMENT` 上查询时面临的问题相比，有什么不同？答案是没有区别。我们当初是如何处理在 `STATEMENT` 表上查询的基数误差的？答案是我们创建了扩展统计。我们可以对 `STATEMENT_PART` 做同样的事情，如 Listing 9-21 所示。

### 在分区表上创建扩展全局统计

Listing 9-21. 在分区表上创建扩展全局统计

```
BEGIN
   DBMS_STATS.delete_table_stats (
      ownname         => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname         => 'STATEMENT_PART'
     ,partname        => NULL
     ,cascade_parts   => TRUE);

   DBMS_STATS.set_table_prefs (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'STATEMENT_PART'
     ,pname     => 'INCREMENTAL'
     ,pvalue    => 'FALSE');
END;
/

DECLARE
   extension_name   all_tab_cols.column_name%TYPE;
BEGIN
   extension_name :=
      DBMS_STATS.create_extended_stats (
         ownname     => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
        ,tabname     => 'STATEMENT_PART'
        ,extension   => '(TRANSACTION_DATE,POSTING_DATE)');

   DBMS_STATS.gather_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,partname      => NULL
     ,granularity   => 'GLOBAL'
     ,method_opt    => 'FOR ALL COLUMNS SIZE 1'
     ,cascade       => FALSE);
END;
/

SELECT *
  FROM statement_part
 WHERE     transaction_date = DATE '2013-01-01'
       AND posting_date = DATE '2013-01-01';

| Id  | Operation              | Name           | Rows  | Time     | Pstart| Pstop |
|---|---|---|---|---|---|---|
|   0 | SELECT STATEMENT       |                |    17 | 00:00:01 |       |       |
|   1 |  PARTITION RANGE SINGLE|                |    17 | 00:00:01 |     1 |     1 |
|*  2 |   TABLE ACCESS FULL    | STATEMENT_PART |    17 | 00:00:01 |     1 |     1 |
```

为避免任何歧义，Listing 9-21 首先删除了 `STATEMENT_PART` 的所有统计信息，并将 `INCREMENTAL` 统计收集首选项重置为 `FALSE`。我明确地将 `PARTNAME` 指定为 `NULL`，表示针对整个表，并将 `CASCADE_PARTS` 指定为 `TRUE`，以确保所有底层分区的统计信息也被删除。这两个设置是默认值，但我想表达得非常清楚。

Listing 9-21 接着以与 Listing 9-11 为 `STATEMENT` 创建统计信息相同的方式，为 `STATEMENT_PART` 创建了多列扩展统计信息，然后仅收集 *全局* 统计信息。现在，当我们查看查询的执行计划时，可以看到基数估计为 17，这几乎是完全正确的，并且比使用非扩展的分区级统计信息所得到的结果要好得多。

![image](img/sq.jpg) **提示** 使用分区级统计信息常常会掩盖对包含分区列的多列创建扩展统计信息的需求。

分区级统计信息比扩展全局统计信息使用得更为频繁的一个原因是，扩展统计信息是 11gR1 版本才引入的新特性，而旧有的习惯很难改变。人们尚未完全意识到扩展统计信息带来的所有好处。

### 统计信息与分区总结

在本节中，我探讨了分区表可能存在的不同级别的统计信息。我解释了如何收集分区级统计信息，CBO 如何使用它们，以及为什么分区级统计信息如此流行。尽管对于分区大小不均匀的表格，分区级统计信息有其罕见且合理的用途，但在更多情况下，它们被用于我认为不恰当的目的，我们将在第 20 章中回到这个讨论。

### 恢复统计信息

在第 6 章中，我曾表示收集统计信息就像玩俄罗斯轮盘赌。那么，如果你“中弹”了怎么办？如果在收集统计信息后，执行计划突然“翻转”，性能急剧恶化怎么办？好消息是，你可以快速恢复之前的统计信息作为短期措施，以便有时间找出问题所在并实施永久解决方案。Listing 9-22 是一个有点刻意的恢复统计信息的演示。

### 使用 DBMS_STATS.RESTORE_TABLE_STATS 恢复表统计信息

Listing 9-22. 使用 `DBMS_STATS.RESTORE_TABLE_STATS` 恢复表统计信息

```
SET SERVEROUT ON

DECLARE
   original_stats_time   DATE;
   num_rows              all_tab_statistics.num_rows%TYPE;
   sowner                all_tab_statistics.owner%TYPE;
BEGIN
   SELECT owner, last_analyzed
     INTO sowner, original_stats_time
     FROM all_tab_statistics
    WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
          AND table_name = 'STATEMENT_PART'
          AND partition_name IS NULL;

   DELETE FROM statement_part;

   DBMS_STATS.gather_table_stats (ownname       => sowner
                                 ,tabname       => 'STATEMENT_PART'
                                 ,partname      => NULL
                                 ,GRANULARITY   => 'GLOBAL'
                                 ,method_opt    => 'FOR ALL COLUMNS SIZE 1'
                                 ,cascade       => FALSE);

   SELECT num_rows
     INTO num_rows
     FROM all_tab_statistics
    WHERE     owner = sowner
          AND table_name = 'STATEMENT_PART'
          AND partition_name IS NULL;

   DBMS_OUTPUT.put_line (
      'After deletion and gathering num_rows is: ' || num_rows);

   DBMS_STATS.restore_table_stats(
      ownname           => sowner
     ,tabname           => 'STATEMENT_PART'
     ,as_of_timestamp   => original_stats_time + 1 / 86400
     ,no_invalidate     => FALSE);

   SELECT num_rows
     INTO num_rows
     FROM all_tab_statistics
    WHERE     owner = sowner
          AND table_name = 'STATEMENT_PART'
          AND partition_name IS NULL;

   DBMS_OUTPUT.put_line (
      'After restoring earlier statistics num_rows is: ' || num_rows);

   INSERT INTO statement_part
      SELECT * FROM statement;

   COMMIT;
END;
/

-- Output

After deletion and gathering num_rows is: 0
After restoring earlier statistics num_rows is: 500
```

Listing 9-22 首先确定 `STATEMENT_PART` 的统计信息最后一次收集的时间。然后删除表中的行并收集统计信息。输出显示统计信息现在正确地反映了表中没有行的事实。接着，我们调用 `DBMS_STATS.RESTORE_TABLE_STATS` 来恢复之前的统计信息。该过程包含一个 `AS_OF_TIMESTAMP` 参数。此参数用于标识要恢复哪一组统计信息。为了让这个示例可复现，Listing 9-22 指定了一个比统计信息最初收集时间晚一秒的时间。在恢复统计信息时，请牢记以下几点：



-  有数个用于恢复统计信息的过程，包括 `DBMS_STATS.RESTORE_SCHEMA_STATS`。
-  使用 `DBMS_STATS.SET_xxx_STATS` 过程设置的用户统计信息**不会**被恢复。因此，例如，任何手工制作的直方图在统计信息恢复后都需要重新应用。
-  虽然这通常是默认行为，但良好的做法是通过使用 `NO_INVALIDATE => FALSE` 参数设置来显式地使共享池中的任何不良执行计划失效，如 清单 9-22 所执行的那样。
-  视图 `DBA_OPTSTAT_OPERATIONS` 提供了收集和恢复操作的历史记录。
-  默认情况下，被取代的统计信息会保留 31 天。这可以通过函数 `DBMS_STATS.GET_STATS_HISTORY_RETENTION` 和过程 `DBMS_STATS.ALTER_STATS_HISTORY_RETENTION` 进行管理。

一旦恢复了统计信息，你可能就会想阻止某个好心人在两分钟后又重新收集它们。让我们看看可以做些什么。

## 锁定统计信息

你可能有很多理由想要阻止人们在一个或多个表上收集统计信息。也许这些表已经通过 `DBMS_STATS.SET_xxx_STATS` 过程进行了大量手工修改。也许收集统计信息是一项耗时的活动，而你已知这是不必要的，或者在一段时间内都不会必要。也许，正如上一节讨论的，你已经恢复了统计信息，并且知道重新收集会导致问题。也许一个小但正在增长的表的统计信息是从测试系统导入的，反映了未来的大数据量，而你想确保这些导入的统计信息不会被反映当前低行数的统计信息所覆盖。

无论出于何种原因想要阻止收集统计信息，你都可以通过调用以下过程之一来实现：`DBMS_STATS.LOCK_SCHEMA_STATS`、`DBMS_STATS.LOCK_TABLE_STATS` 和 `DBMS_STATS.LOCK_PARTITION_STATS`。使用这些例程时请记住以下几点：

-  一个坚决的用户总是可以通过使用 `FORCE` 参数来对锁定的对象进行收集或设置统计信息。覆盖统计信息锁不需要额外的权限。
-  如果你锁定一个或多个表的统计信息，那么为一个模式收集统计信息时通常会避开这些被锁定的表（除非 `FORCE` 被设置为 `TRUE`）。
-  有些人会锁定那些已变为只读的表分区的统计信息。这可能发生在 ILM 环境中的历史分区上。锁定只读分区可以在 `GRANULARITY` 设置为 `ALL` 时，减少收集统计信息所需的时间。
-  当你锁定一个表的统计信息时，你同时锁定了该表所有分区、列和索引的统计信息。你不能只锁定一个索引的统计信息。
-  如前所述，当一个表的统计信息被锁定时，在创建或重建该表上的索引时不会生成统计信息，在批量加载数据到该表时也不会生成统计信息。

在 TSTATS 环境中，你不希望在生产环境中收集任何应用程序对象的统计信息，因此最佳实践是锁定应用程序模式中所有表的统计信息。

## 待定统计信息

如果你怀疑一个直方图能解决你的生产问题，但又没有一个专供个人使用的合适测试系统，该怎么办？是否有办法在不影响其他用户的情况下，调查统计信息变更的影响？

为应对这些情况，开发了待定统计信息的概念。通常，当统计信息被收集、导入或设置时，它们会立即*发布*，这意味着所有用户都可以看到并利用它们。然而，也可以先生成统计信息，稍后再发布。已生成但尚未发布的统计信息被称为*待定*统计信息。清单 9-23 展示了待定统计信息的工作原理。

清单 9-23. 使用待定统计信息测试直方图的效果

```
001 BEGIN
002   2     DBMS_STATS.set_table_prefs (
003   3        ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
004   4       ,tabname   => 'STATEMENT_PART'
005   5       ,pname     => 'PUBLISH'
006   6       ,pvalue    => 'TRUE');
007   7  END;
008   8  /

010 PL/SQL procedure successfully completed.

013 BEGIN
014   2     DBMS_STATS.gather_table_stats (
015   3        ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
016   4       ,tabname       => 'STATEMENT_PART'
017   5       ,partname      => NULL
018   6       ,GRANULARITY   => 'GLOBAL'
019   7       ,method_opt    => 'FOR ALL COLUMNS SIZE 1'
020   8       ,cascade       => FALSE);
021   9  END;
022  10  /

024 PL/SQL procedure successfully completed.

027 BEGIN
028   2     DBMS_STATS.set_table_prefs (
029   3        ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
030   4       ,tabname   => 'STATEMENT_PART'
031   5       ,pname     => 'PUBLISH'
032   6       ,pvalue    => 'FALSE');
033   7  END;
034   8  /

036 PL/SQL procedure successfully completed.

039 DECLARE
040   2     srec   DBMS_STATS.statrec;
041   3  BEGIN
042   4     FOR r
043   5        IN (SELECT *
044   6              FROM all_tab_cols
045   7             WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
046   8                   AND table_name = 'STATEMENT_PART'
047   9                   AND column_name = 'TRANSACTION_AMOUNT')
048  10     LOOP
049  11        srec.epc := 3;
050  12        srec.bkvals := DBMS_STATS.numarray (600, 400, 600);
051  13        DBMS_STATS.prepare_column_values (srec
052  13                                         ,DBMS_STATS.numarray (-1e7, 8, 1e7));
053  15        DBMS_STATS.set_column_stats (ownname   => r.owner
054  15                                    ,tabname   => r.table_name
055  16                                    ,colname   => r.column_name
056  17                                    ,distcnt   => 3
057  18                                    ,density   => 1 / 250
058  19                                    ,nullcnt   => 0
059  20                                    ,srec      => srec
060  21                                    ,avgclen   => r.avg_col_len);
061  22     END LOOP;
062  23  END;
063  24  /

065 PL/SQL procedure successfully completed.

068 SET AUTOTRACE OFF

070 SELECT endpoint_number, endpoint_value
071   2    FROM all_tab_histograms
072   3   WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
073   4         AND table_name = 'STATEMENT_PART'
074   5         AND column_name = 'TRANSACTION_AMOUNT';

076 ENDPOINT_NUMBER ENDPOINT_VALUE
077 --------------- --------------
078               0              5
079               1          15200

081 2 rows selected.

084 SELECT endpoint_number, endpoint_value
085   2    FROM all_tab_histgrm_pending_stats
086   3   WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
087   4         AND table_name = 'STATEMENT_PART'
088   5         AND column_name = 'TRANSACTION_AMOUNT';

090 ENDPOINT_NUMBER ENDPOINT_VALUE
091 --------------- --------------
092             600      -10000000
093            1000              8
094            1600       10000000

096 3 rows selected.

099 SET AUTOTRACE TRACEONLY EXPLAIN

101 SELECT *
102   2    FROM statement_part
103   3   WHERE transaction_amount = 8;

105 ---------------------------------------------------------------------------------
106 | Id  | Operation           | Name           | Rows  | Time     | Pstart| Pstop |
107 ---------------------------------------------------------------------------------
108 |   0 | SELECT STATEMENT    |                |     2 | 00:00:01 |       |       |
109 |   1 |  PARTITION RANGE ALL|                |     2 | 00:00:01 |     1 |     2 |
110 |*  2 |   TABLE ACCESS FULL | STATEMENT_PART |     2 | 00:00:01 |     1 |     2 |
111 ---------------------------------------------------------------------------------
```



