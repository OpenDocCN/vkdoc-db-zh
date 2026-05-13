# 第 15 章 ■ 分区

## 15-14. 交换分区

交换分区是一项强大的功能。它允许你将现有表的分区提取出来，使其成为一个独立的表；同时，也能将一个独立的表并入成为分区表的一部分。当你交换一个分区时，Oracle 只是简单地更新数据字典中的条目来执行交换操作。

当你使用 `WITHOUT VALIDATION` 子句来交换分区时，你是在指示 Oracle 不验证传入的分区（或子分区）中的行是否是所定义范围的有效条目。

这样做的好处是使得交换操作非常快速，因为 Oracle 只是通过更新数据字典中的指针来执行交换操作。如果你使用了 `WITHOUT VALIDATION`，你需要确保你的数据是准确的。

如果你为分区表定义了主键，那么被交换的表也必须定义有相同的主键结构。如果存在主键，`WITHOUT VALIDATION` 子句不会阻止 Oracle 强制执行唯一性约束。

##### 15-15. 重命名分区

### 问题

你想要重命名一个分区，使其符合你的命名标准。

### 解决方案

你可以重命名表分区和索引分区。此示例使用 `ALTER TABLE` 语句来重命名一个表分区：

```sql
alter table f_regs rename partition reg_p_1 to reg_part_1;
```

下一行代码使用 `ALTER INDEX` 语句来重命名一个索引分区：

```sql
alter index f_reg_dates_fk1 rename partition reg_p_1 to reg_part_1;
```

### 工作原理

有时需要重命名一个表分区或索引分区。例如，你可能想在删除一个分区之前先重命名它。此外，你可能希望重命名对象以使它们符合标准。在这些场景中，使用相应的 `ALTER TABLE` 或 `ALTER INDEX` 语句即可完成。你可以查询数据字典来验证有关已重命名对象的信息。此查询显示分区表的名称：

```sql
select table_name,partition_name,tablespace_name from user_tab_partitions;
```

类似地，此查询显示分区索引的信息：

```sql
select index_name,partition_name,status,high_value,tablespace_name
from user_ind_partitions;
```

##### 15-16. 拆分分区

### 问题

你已识别出一个包含过多行的分区。你想将该分区拆分成两个分区。

### 解决方案

使用 `ALTER TABLE...SPLIT PARTITION` 语句来拆分一个现有分区。以下示例拆分了一个范围分区表中的分区：

```sql
alter table f_regs split partition p2010 at (20100601)
into (partition p2010_a, partition p2010) update indexes;
```

如果你没有指定 `UPDATE INDEXES`，你将需要重建与被拆分分区相关联的所有局部索引以及任何全局索引。你可以使用以下 SQL 验证索引的状态：

```sql
select index_name, partition_name, status from user_ind_partitions;
```

下一个示例拆分一个列表分区。首先显示 `CREATE TABLE` 语句，以便你能看到列表分区最初是如何定义的：

```sql
create table f_sales
(reg_sales number
,d_date_id number
,state_code varchar2(20)
)
partition by list (state_code)
( partition reg_west values ('AZ','CA','CO','MT','OR','ID','UT','NV')
,partition reg_mid values ('IA','KS','MI','MN','MO','NE','OH','ND')
,partition reg_rest values (default)
);
```

接下来，拆分 `REG_MID` 分区：

```sql
alter table f_sales split partition reg_mid values ('IA','KS','MI','MN') into (partition reg_mid_a,
partition reg_mid_b);
```

`REG_MID_A` 分区现在包含值 `IA`、`KS`、`MI` 和 `MN`，而 `REG_MID_B` 则被分配了剩余的 `MO`、`NE`、`OH` 和 `ND` 值。

### 工作原理

拆分分区操作允许你从一个单独的分区创建出两个新的分区。每个新分区都有自己的段、物理属性和区。与原始分区关联的段会被删除。



##### 15-17. 合并分区

## 问题

你有两个分区，其中包含的数据不足以证明值得为它们保留单独的分区。你想将这两个分区合并为一个。

## 解决方案

使用 `ALTER TABLE...MERGE PARTITIONS` 语句来合并分区。此示例将 `REG_P_1` 分区合并到 `REG_P_2` 分区中：

```sql
alter table f_regs merge partitions reg_p_1, reg_p_2 into partition reg_p_2;
```

在此示例中，分区是按日期范围组织的。合并后的分区将被定义为接受两个被合并分区中范围最高的行。任何本地索引也会被合并到新的单个分区中。

请注意，合并分区会使与被合并分区关联的任何本地索引失效。此外，表中存在的任何全局索引的所有分区都将被标记为不可用。你可以通过查询数据字典来验证分区索引的状态：

```sql
select index_name, partition_name, tablespace_name, high_value, status
from user_ind_partitions
order by 1,2;
```

以下是一些示例输出，展示了分区合并后全局索引和本地索引的样子：

```
INDEX_NAME       PARTITION_NAME   TABLESPACE_NAME  HIGH_VALUE   STATUS
---------------- ---------------- ---------------- ------------ ----------
F_GLO_IDX1       SYS_P680         IDX1                          UNUSABLE
F_GLO_IDX1       SYS_P681         IDX1                          UNUSABLE
F_GLO_IDX1       SYS_P682         IDX1                          UNUSABLE
F_LOC_FK1        REG_P_2          USERS            20110101     UNUSABLE
F_LOC_FK1        REG_P_3          TBSP3            20120101     USABLE
```

合并分区时，你可以使用 `ALTER TABLE` 语句的 `UPDATE INDEXES` 子句来指示 Oracle 自动重建任何关联的索引：

```sql
alter table f_regs merge partitions reg_p_1, reg_p_2 into partition reg_p_2
tablespace tbsp2
update indexes;
```

请记住，当你使用 `UPDATE INDEXES` 子句时，合并操作将花费更长时间。如果你想最小化合并操作时间，请不要使用此子句。相反，手动重建与合并分区关联的本地索引：

```sql
alter table f_regs modify partition reg_p_2 rebuild unusable local indexes;
```

你可以使用 `ALTER INDEX...REBUILD PARTITION` 语句来重建全局索引的每个分区：

```sql
alter index f_glo_idx1 rebuild partition sys_p680;
alter index f_glo_idx1 rebuild partition sys_p681;
alter index f_glo_idx1 rebuild partition sys_p682;
```

## 原理说明

你可以使用 `ALTER TABLE...MERGE PARTITIONS` 语句来合并两个或多个分区。你合并到的目标分区名称可以是你要合并的其中一个分区的名称，也可以是一个全新的名称。

合并分区时，请确保重建与被合并分区关联的任何索引。你可以重建或重新创建任何失效的索引。重建索引可能是最简单的方法，因为你不需要重新创建索引所需的 DDL。

在合并两个（或更多）分区之前，请确保合并后的分区在其表空间中有足够的空间来容纳所有合并后的行。如果没有足够的空间，你将收到表空间无法扩展到必要大小的错误。

##### 15-18. 删除分区

## 问题

你有一些过时的数据。它包含在一个分区内，而你想删除该分区。

## 解决方案

首先确定要删除的分区的名称。运行以下查询以列出当前连接用户特定表的分区：

```sql
select segment_name, segment_type, partition_name
from user_segments
where segment_name = upper('&table_name');
```

接下来，使用 `ALTER TABLE...DROP PARTITION` 语句从表中删除一个分区。此示例从 `F_SALES` 表中删除 `P_2008` 分区：

```sql
alter table f_sales drop partition p_2008;
```

你应该会看到以下消息：

`Table altered.`

如果要删除子分区，请使用 `DROP SUBPARTITION` 子句：

```sql
alter table f_sales drop subpartition p2_south;
```



你可以查询 `USER_TAB_SUBPARTITIONS` 来确认子分区是否已被删除。

### 注意
Oracle 不允许你删除复合分区表中的所有子分区。每个分区必须至少保留一个子分区。

### 工作原理
当你删除一个分区时，没有恢复删除的操作。因此，在删除分区前，请确保你处于正确的环境并且确实需要删除它。如果你需要保留待删除分区中的数据，可以将该分区合并到另一个分区，而不是直接删除它。

你不能从哈希分区表中删除分区。对于哈希分区表，你必须合并分区来移除一个。同样，你不能显式地从引用分区表中删除分区。当父表的分区被删除时，它也会从相应的子引用分区表中删除。

## 15-19. 从分区中删除行
### 问题
你想高效地删除一个分区中的所有记录。

### 解决方案
首先，确定你想要删除记录的分区的名称。

```sql
select segment_name, segment_type, partition_name
from user_segments
where partition_name is not null;
```

使用 `ALTER TABLE...TRUNCATE PARTITION` 语句来删除分区中的所有记录。此示例截断了 `F_SALES` 表的 `P_2008` 分区：

```sql
alter table f_sales truncate partition p_2008;
```

你应该会看到以下消息：

```
Table truncated.
```

在此场景下，该消息并不意味着整个表被截断了。它只是确认指定的分区已被截断。

### 工作原理
截断分区是快速删除大量数据的一种高效方法。然而，当你截断一个分区时，没有回滚机制。截断操作会从分区中永久删除数据。

如果你需要能够回滚事务的选项，请使用 `DELETE` 语句：
```sql
delete from f_sales partition(p2008);
```

这种方法的缺点是，如果你有数百万条记录，`DELETE` 操作可能会运行很长时间。此外，对于大量记录，`DELETE` 会生成大量的回滚信息。这可能会给竞争资源的其他 SQL 语句带来性能问题。

##### 15-20. 为分区生成统计信息
### 问题
你刚刚将数据加载到单个分区中，需要生成统计信息以反映新插入的数据。

### 解决方案
使用 `EXECUTE` 语句运行 `DBMS_STATS` 包来为特定分区生成统计信息。

在此示例中，所有者是 `STAR`，表是 `F_SALES`，正在分析的分区是 `P_2009`：
```sql
exec dbms_stats.gather_table_stats(ownname=>'STAR',-
                                   tabname=>'F_SALES',-
                                   partname=>'P_2009');
```

如果你正在处理一个大型分区，你可能需要指定百分比抽样大小、并行度，并为任何索引也生成统计信息：
```sql
exec dbms_stats.gather_table_stats(ownname=>'STAR',-
                                   tabname=>'F_SALES',-
                                   partname=>'P_2009',-
                                   estimate_percent=>dbms_stats.auto_sample_size,-
                                   degree=>dbms_stats.auto_degree,-
                                   cascade=>true);
```

### 工作原理
对于分区表，你可以在单个分区或整个表上生成统计信息。我们建议每当分区内有大量数据发生变化时，就生成新的统计信息。你需要足够了解你的表和数据，以确定是否需要生成新的统计信息。

##### 15-21. 创建映射到分区的索引（本地索引）
### 问题
你发现查询分区表时经常在 `WHERE` 子句中使用分区键。你想提高查询的性能。

### 解决方案
以下是创建简单分区表的 SQL：
```sql
create table f_regs(
   reg_count number
  ,d_date_id number
)
partition by range(d_date_id)
   (partition p2009 values less than (20100101) tablespace tbsp1
   ,partition p2010 values less than (20110101) tablespace tbsp2
);
```

接下来，使用 `CREATE INDEX` 语句的 `LOCAL` 子句在分区表上创建本地索引：此示例在 `F_REGS` 表的 `D_DATE_ID` 列上创建了一个本地索引：
```sql
create index f_regs_dates_fk1 on f_regs(d_date_id) local;
```

运行以下查询以查看有关分区索引的信息：
```sql
select index_name, table_name, partitioning_type
from user_part_indexes;
```
以下是一些示例输出：

```
INDEX_NAME                     TABLE_NAME           PARTITI
------------------------------ -------------------- -------
F_REGS_DATES_FK1               F_REGS               RANGE
```

现在查询 `USER_IND_PARTITIONS` 表以查看有关本地分区索引的信息：
```sql
select index_name, partition_name, tablespace_name
from user_ind_partitions;
```
注意，已为表的每个分区创建了一个索引分区，并且索引是在与表分区相同的表空间中创建的：

```
INDEX_NAME           PARTITION_NAME       TABLESPACE_NAME
-------------------- -------------------- --------------------
F_REGS_DATES_FK1     P2009                TBSP1
F_REGS_DATES_FK1     P2010                TBSP2
```

如果你想让本地索引分区创建在与表分区不同的表空间（或多个表空间）中，请在创建索引时指定这些表空间：
```sql
create index f_regs_dates_fk1 on f_regs(d_date_id) local
   (partition p2009 tablespace idx1
   ,partition p2010 tablespace idx2);
```
如果在构建本地分区索引时指定分区信息，则分区数必须与构建索引的表上的分区数相匹配。

### 工作原理
本地分区索引的分区方式与其所基于的分区表的分区方式完全相同。每个表分区将有一个对应的索引，该索引仅包含该表分区的 `ROWID` 值和索引键值。换句话说，本地分区索引中的 `ROWID` 只指向其对应表分区中的行。

Oracle 保持本地索引分区与表分区同步。你不能显式地从本地索引中添加或删除分区。当你添加或删除表分区时，Oracle 会自动为本地索引执行相应的工作。Oracle 管理本地索引分区，无论本地索引是如何分配到表空间的。

本地索引在数据仓库和决策支持系统中很常见。如果你经常使用分区列进行查询，本地索引是合适的。这使得 Oracle 可以使用适当的索引和表分区来快速检索数据。

有两种类型的本地索引：本地*前缀*索引和本地*非前缀*索引。本地前缀索引是指索引的最左列与表分区键匹配的索引。解决方案部分中的示例是一个本地前缀索引，因为其最左列 (`D_DATE_ID`) 也是表的分区键。

非前缀本地索引是指其最左列与用于对相应表进行分区的分区键不匹配的索引。例如，这是一个本地非前缀索引：
```sql
create index f_regs_idx1 on f_regs(reg_count) local;
```
该索引按 `REG_COUNT` 列分区，这不是表的分区键，因此是非前缀索引。你可以通过查询 `USER_PART_INDEXES` 中的 `ALIGNMENT` 列来验证索引是否被视为前缀索引：
```sql
select index_name,table_name,alignment,locality
from user_part_indexes;
```
以下是一些示例输出：

```
INDEX_NAME           TABLE_NAME           ALIGNMENT     LOCALITY
-------------------- -------------------- ------------ ----------
F_REGS_DATES_FK1     F_REGS               PREFIXED      LOCAL
F_REGS_IDX1          F_REGS               NON_PREFIXED  LOCAL
```

#### 第 15 章：分区

你可能会问，为什么还要区分前缀索引和非前缀索引？本地的非前缀索引意味着索引定义中没有将分区键作为前导列。这可能会带来性能问题，因为使用非前缀索引进行范围扫描可能需要搜索每一个索引分区。如果分区数量很多，这可能导致性能低下。

[www.it-ebooks.info](http://www.it-ebooks.info/)

你可以通过将分区键列包含在索引定义的前导列中，将所有本地索引创建为前缀索引。例如，我们可以将 `F_REGS_IDX1` 索引创建为前缀索引，如下所示：
```
create index f_regs_idx2 on f_regs(d_date_id,reg_count) local;
```

前缀索引一定比非前缀索引更好吗？这取决于你如何查询表。你需要为你使用的查询生成执行计划（explain plans），并检查前缀索引是否比非前缀索引能更好地利用分区裁剪（partition pruning，即消除需要搜索的分区）。

另外请记住，多列的前缀本地索引会比非前缀本地索引消耗更多的空间和资源。

##### 15-22. 创建具有自身分区方案的索引（全局索引）

### 问题

你通过一个非分区键列来查询分区表。你希望通过在该列上创建索引来提高性能。

### 解决方案

你可以创建一个基于范围分区的全局索引或基于哈希分区的全局索引。使用关键字 `GLOBAL` 来指定索引将采用与其对应表不同的分区策略来构建。在创建基于范围分区的全局索引时，必须始终指定 `MAXVALUE`。以下示例创建了一个基于范围的全局索引：
```
create index f_sales_gidx1 on f_sales(sales_id)
global partition by range(sales_id)
(partition pg1 values less than (100)
,partition pg2 values less than (200)
,partition pg3 values less than (maxvalue));
```

另一种类型的全局分区索引是基于哈希的。此示例创建了一个基于哈希分区的全局索引：
```
create index f_count_idx1 on f_regs(reg_count)
global partition by hash(reg_count) partitions 3;
```

### 工作原理

与其基础表分区方式不同的索引被称为**全局索引**。全局索引中的一条记录可以指向其基础表的任何分区。全局索引可以在任何类型的分区表上创建。全局索引本身可以是基于范围或基于哈希进行分区的。

[www.it-ebooks.info](http://www.it-ebooks.info/)

通常，全局索引比本地索引更难维护。我们建议你尽量避免使用全局索引，并尽可能使用本地索引。全局索引没有自动维护机制（而本地索引有）。对于全局索引，你需要负责添加和删除索引分区。此外，对基础分区表的许多维护操作通常要求重建全局索引分区。以下对堆组织表的操作将使全局索引失效：

*   `ADD`（`HASH`）
*   `COALESCE`（`HASH`）
*   `DROP`
*   `EXCHANGE`
*   `MERGE`
*   `MOVE`
*   `SPLIT`
*   `TRUNCATE`

在执行维护操作时，请考虑使用 `UPDATE INDEXES` 子句。这将使全局索引在操作期间保持可用，并消除了重建的需要。使用 `UPDATE INDEXES` 的缺点是，由于在操作过程中需要维护索引，维护操作将花费更长时间。

全局索引对于通过索引检索少量行的查询很有用。在这些情况下，Oracle 可以消除（裁剪）任何不必要的索引分区并高效地检索数据。例如，全局范围分区索引在需要高效访问单个记录的 OLTP 环境中非常有用。

[www.it-ebooks.info](http://www.it-ebooks.info/)

[www.it-ebooks.info](http://www.it-ebooks.info/)

