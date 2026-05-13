# 创建对象统计信息

在少数情况下，特别是在通过索引访问表时，需要同时获取多个行源操作的估计值。我们将在接下来讨论*聚簇因子*索引统计信息时，将此特例纳入考量。

成本优化器从数据字典中读取对象统计信息，但首先需要将这些统计信息存入数据字典。使用 `DBMS_STATS` 包创建或更新对象统计信息有四种不同的方式：*收集*、*导入*、*转移*和*设置*统计信息。此外：

*   默认情况下，当索引被创建或重建时，会为其生成对象统计信息。
*   默认情况下，自数据库版本 12cR1 起，当使用 `CREATE TABLE ... AS SELECT` 选项创建表或使用 `INSERT` 语句批量加载空表时，会生成表及其关联列的对象统计信息。

让我逐一介绍这些选项。

## 收集对象统计信息

收集统计信息是迄今为止创建或更新统计信息最常用的方式，对于应用对象，需要使用 `DBMS_STATS` 包中四种不同的过程之一：

*   `DBMS_STATS.GATHER_DATABASE_STATS` 收集整个数据库的统计信息。
*   `DBMS_STATS.GATHER_SCHEMA_STATS` 收集指定模式内对象的统计信息。
*   `DBMS_STATS.GATHER_TABLE_STATS` 收集指定表、表分区或表子分区的统计信息。
    *   为分区表收集统计信息时，可以在同一次调用中收集所有分区和子分区（如适用）的统计信息。
    *   无论是为表、分区还是子分区收集统计信息，默认行为都是在同一次调用中收集所有关联索引的统计信息。
    *   除了表统计信息外，此过程还会同时收集表中部分或所有列的统计信息；没有单独收集列统计信息的选项。
*   `DBMS_STATS.GATHER_INDEX_STATS` 收集指定索引、索引分区或索引子分区的统计信息。与 `DBMS_STATS.GATHER_TABLE_STATS` 一样，可以在同一次调用中收集全局、分区和子分区级别的统计信息。

![image](img/sq.jpg) **注意** 数据字典中的对象也需要对象统计信息。这些统计信息使用 `DBMS_STATS.GATHER_DICTIONARY_STATS` 和 `DBMS_STATS.GATHER_FIXED_OBJECT_STATS` 过程获取。

`DBMS_STATS` 中的所有统计信息收集过程都以类似方式工作：使用递归 SQL 语句读取被分析的对象。并非需要读取对象中的所有块。默认情况下，会对数据进行随机抽样，直到 `DBMS_STATS` 认为额外的抽样不会对计算出的统计信息产生实质性影响为止。自 11gR1 版本以来，在绝大多数情况下，这种确定何时停止的逻辑非常有效。

![image](img/sq.jpg) **注意** 在某些情况下，例如当您有一个列在 2,000,000 行中具有相同值，而在另外 2 行中具有第二个值时，随机抽样很可能过早地确定表中所有行的该列值都相同。在这种情况下，您应该直接设置列统计信息以反映现实情况！

`DBMS_STATS` 统计信息收集过程有许多可选参数。例如，您可以指示仅在统计信息缺失或被视为*陈旧*时才收集对象统计信息。您还可以指定以并行方式执行统计信息收集。更多详情请参阅 *PL/SQL Packages and Types Reference* 手册。

您应该注意，在首次安装 Oracle 数据库时，会自动创建一个用于收集整个数据库统计信息的作业。我怀疑这个作业的存在是为了迎合那些从不考虑对象统计信息的为数不少的客户。对于大多数大型系统，需要定制此过程，您应仔细审查此作业。根据版本不同，该作业的调度方式也多种多样，因此更多详情请参阅《SQL Tuning Guide》（对于 11g 或更早版本，则参阅《Performance Tuning Guide》）。

在绝大多数情况下，应该收集对象的*初始*统计信息集。这是唯一实用的方法。另一方面，正如我在第 6 章中所解释的，应尽量减少在生产系统上收集统计信息。现在我将解释一种可以完全避免在生产系统上收集统计信息的方法。

## 导出与导入统计信息

如果您是烹饪节目的粉丝，您会熟悉像“这是我预先准备好的一个”这样的表达方式。烹饪需要时间，有时还会出错。因此，在电视烹饪节目中，为了节省时间，通常会展示准备餐点的初始阶段，然后直接使用预先准备好的菜肴跳到结尾。

在某些方面，收集统计信息就像烹饪：需要一些时间，一个小失误就可能毁掉一切。好消息是，DBA 们像电视厨师一样，通常可以通过将预先制作好的统计信息导入数据库来节省时间并避免风险。使用预制统计信息是我在第 6 章介绍的 TSTATS 部署方法中的几个关键概念之一，但在许多其他场景中，使用预制的对象统计信息也很有用。一个例子是冗长的数据转换、迁移或升级过程。这些过程通常在紧张的维护窗口内执行，并使用与生产数据即使不完全相同也非常相似的数据进行多次演练。从这些测试数据中得出的对象统计信息很可能适用于生产系统。清单 9-1 演示了此技术。

清单 9-1. 使用 DBMS_STATS 导出和导入过程将统计信息从一个系统复制到另一个系统

```
CREATE TABLE statement
(
   transaction_date_time   TIMESTAMP WITH TIME ZONE
  ,transaction_date        DATE
  ,posting_date            DATE
  ,posting_delay           AS (posting_date - transaction_date)
  ,description             VARCHAR2 (30)
  ,transaction_amount      NUMBER
  ,amount_category         AS (CASE WHEN transaction_amount < 10 THEN 'LOW'
                                    WHEN transaction_amount < 100 THEN 'MEDIUM' ELSE 'HIGH'
                               END)
  ,product_category        NUMBER
  ,customer_category       NUMBER
)
PCTFREE 80
PCTUSED 10;
```



```sql
INSERT INTO statement (transaction_date_time
                      ,transaction_date
                      ,posting_date
                      ,description
                      ,transaction_amount
                      ,product_category
                      ,customer_category)
       SELECT   TIMESTAMP '2013-01-01 12:00:00.00 -05:00'
              + NUMTODSINTERVAL (TRUNC ( (ROWNUM - 1) / 50), 'DAY')
             ,DATE '2013-01-01' + TRUNC ( (ROWNUM - 1) / 50)
             ,DATE '2013-01-01' + TRUNC ( (ROWNUM - 1) / 50) + MOD (ROWNUM, 3)
                 posting_date
             ,DECODE (MOD (ROWNUM, 4)
                     ,0, 'Flight'
                     ,1, 'Meal'
                     ,2, 'Taxi'
                     ,'Deliveries')
             ,DECODE (MOD (ROWNUM, 4)
                     ,0, 200 + (30 * ROWNUM)
                     ,1, 20 + ROWNUM
                     ,2, 5 + MOD (ROWNUM, 30)
                     ,8)
             ,TRUNC ( (ROWNUM - 1) / 50) + 1
             ,MOD ( (ROWNUM - 1), 50) + 1
         FROM DUAL
   CONNECT BY LEVEL <= 500;
```

```sql
CREATE INDEX statement_i_tran_dt
   ON statement (transaction_date_time);
```

```sql
CREATE INDEX statement_i_pc
   ON statement (product_category);
```

```sql
CREATE INDEX statement_i_cc
   ON statement (customer_category);
```

```sql
BEGIN
   DBMS_STATS.gather_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT'
     ,partname      => NULL
     ,granularity   => 'ALL'
     ,method_opt    => 'FOR ALL COLUMNS SIZE 1'
     ,cascade       => FALSE);
END;
/
```

```sql
BEGIN
   DBMS_STATS.create_stat_table (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,stattab   => 'CH9_STATS');

   DBMS_STATS.export_table_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'STATEMENT'
     ,statown   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,stattab   => 'CH9_STATS');
END;
/
```
-- 移动到目标系统

```sql
BEGIN
   DBMS_STATS.delete_table_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'STATEMENT');

   DBMS_STATS.import_table_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'STATEMENT'
     ,statown   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,stattab   => 'CH9_STATS');
END;
/
```

*   清单 9-1 创建了一个名为 `STATEMENT` 的表，该表将构成本章其他演示的基础。`STATEMENT` 表被加载了数据，创建了一些索引，然后收集了统计信息。在这个阶段，你必须想象这个表位于源系统上，即我上面描述的场景中的测试系统。
*   统计信息复制过程的第一阶段是创建一个常规的堆表来存放导出的统计信息。该表必须具有特定的格式，并通过调用 `DBMS_STATS.CREATE_STAT_TABLE` 来创建。我将该堆表命名为 `CH9_STATS`，但你也可以使用任何其他名称。
*   下一步是导出统计信息。我使用 `DBMS_STATS.EXPORT_TABLE_STATS` 完成了这一步骤，该过程默认导出表、其列和相关索引的统计信息。正如你可能想象的那样，此过程还有用于数据库和模式等的变体。
*   下一个步骤（未展示）涉及将堆表移动到目标系统，即我上面描述的场景中的生产系统。这可以通过使用数据库链接或 Data Pump 实用程序来完成。然而，我更倾向于的方法是生成一个 SQL Loader 脚本或一组 `INSERT` 语句。这样可以将该脚本与其它项目脚本一起签入源代码控制系统以供存档。
*   现在，你必须想象 清单 9-1 中的剩余代码正在目标系统上运行。良好的做法是先使用 `DBMS_STATS.DELETE_TABLE_STATS` 或 `DBMS_STATS.DELETE_SCHEMA_STATS` 删除目标表上任何现有的统计信息。否则，你最终可能会得到一套混合的统计信息。
*   然后执行对 `DBMS_STATS.IMPORT_TABLE_STATS` 的调用，以将对象统计信息从 `CH9_STATS` 堆表加载到数据字典中，随后 CBO 就可以在涉及 `STATEMENT` 表的 SQL 语句中使用这些统计信息。

如果存在任何疑问，那么这些导出/导入操作所需的时间与所涉及对象的大小无关。执行导出/导入操作所需的时间纯粹取决于所涉及对象的数量，对于大多数数据库而言，导入统计信息是一项最多在几分钟内即可完成的任务。

### 传输统计信息

Oracle 数据库 12cR1 引入了一种替代导出/导入方法来复制统计信息从一个数据库到另一个数据库的方案。`DBMS_STATS.TRANSFER_STATS` 过程允许复制对象统计信息，而无需创建堆表，例如 清单 9-1 中使用的 `CH9_STATS`。清单 9-2 展示了如何直接使用数据库链接将特定表的统计信息从一个数据库传输到另一个数据库。

清单 9-2。使用数据库链接在数据库之间传输统计信息

```sql
BEGIN
   DBMS_STATS.transfer_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'STATEMENT'
     ,dblink    => 'DBLINK_NAME');
END;
/
```

如果你想尝试此操作，需要将 `DBLINK_NAME` 参数值替换为目标数据库上指向源数据库的有效数据库链接的名称。表 `STATEMENT` 的统计信息将直接从一个系统传输到另一个系统。

也许我有些守旧，但在我看来，如果你的统计信息重要到需要复制，那么它们也值得备份。基于这个前提，清单 9-1 中的方法优于 清单 9-2 中的方法。不过话又说回来，也许老狗学不了新把戏。

### 直接设置对象统计信息

尽管绝大多数对象统计信息应该通过收集获得，但在少数情况下需要直接设置统计信息。以下是其中一些情况：



