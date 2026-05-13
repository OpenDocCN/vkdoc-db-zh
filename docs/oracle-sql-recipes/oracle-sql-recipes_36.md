# 第 15 章 ■ 分区

##### 15-12. 对现有表进行分区

### 问题

你有一个表已经变得很大，且当前未分区。你想知道是否可以对它进行分区。

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 解决方案

有几种方法可以将非分区表转换为分区表。本解决方案展示一种简单的方法，步骤如下：

1.  使用 `CREATE TABLE <新表名>...AS SELECT * FROM <旧表名>` 从旧表创建一个新的分区表。
2.  删除或重命名旧表。
3.  将步骤 1 中创建的表重命名为被删除的旧表名。

在此示例中，我们首先使用 `CREATE TABLE` 语句从旧的、名为 `F_REGS` 的非分区表创建一个名为 `F_REGS_NEW` 的新分区表：

```sql
create table f_regs_new
partition by range (d_date_id)
(partition p2008 values less than(20090101),
partition p2009 values less than(20100101),
partition pmax values less than(maxvalue)
)
nologging
as select * from f_regs;
```

现在，你可以删除（或重命名）旧的非分区表，并将新的分区表重命名为旧表名：

```sql
drop table f_regs purge;
rename f_regs_new to f_regs;
```

最后，为新表重建所有约束、授权、索引和统计信息。你现在应该拥有一个替换了旧的非分区表的新分区表。

如果原始表包含许多约束、授权和索引，对于最后一步，你可能希望使用 Data Pump `EXPDP` 或 `EXP` 导出不含数据的原始表，然后在创建新表后，使用 Data Pump `IMPDP` 或 `IMP` 在新表上创建约束、授权、索引和统计信息。

### 工作原理

将表从非分区转换为分区可能是一个耗时的过程。在解决方案部分，我们展示了一种简单的表转换方法。表 15-3 列出了从非分区表创建分区表的各种技术的优缺点。毫无疑问，你需要仔细规划并测试你的转换过程。

[www.it-ebooks.info](http://www.it-ebooks.info/)

**表 15-3. 转换非分区表的方法**

| **转换方法** | **优点** | **缺点** |
| :--- | :--- | :--- |
| `CREATE <新分区表> AS SELECT * FROM <旧表>` | 简单，可使用 `NOLOGGING` 和 `PARALLEL` 选项。直接路径加载。 | 需要同时容纳新旧表的空间。 |
| `INSERT /*+ APPEND */ INTO <新分区表> SELECT * FROM <旧表>` | 快速、简单。直接路径加载。 | 需要同时容纳新旧表的空间。 |
| Data Pump `EXPDP` 旧表，`IMPDP` 新表（或对于旧版 Oracle 使用 `EXP` 和 `IMP`） | 快速，所需空间较少。处理了授权、权限等。可按分区通过过滤条件进行加载。 | 因为需要使用工具而更复杂。 |
| 创建分区表 `<新分区表>`，与旧表 `<旧表>` 交换分区 | 潜在停机时间更少。 | 步骤多，复杂。 |
| 使用 `DBMS_REDEFINITION` 包（较新版本的 Oracle） | 在线转换现有表。 | 步骤多，复杂。 |
| 创建 CSV 文件或外部表，使用 SQL*Loader 加载 `<新分区表>` | 可按分区进行加载。 | 步骤多，复杂。 |

##### 15-13. 向分区表添加分区

### 问题

你有一个已经分区的表，但你想向该表添加另一个分区。

### 解决方案

解决方案取决于你实现的分区类型。对于范围分区表，如果表的最高界限未定义 `MAXVALUE`，你可以使用 `ALTER TABLE...ADD PARTITION` 语句向表的高端添加分区。此示例向范围分区表的高端添加一个分区：

```sql
alter table f_regs add partition p2011 values less than (20120101)
pctfree 5 pctused 95 tablespace p_tbsp;
```

如果你有一个高范围以 `MAXVALUE` 限定的范围分区表，则无法添加分区。在这种情况下，你必须拆分一个现有分区（详见配方 15-16）。

对于哈希分区表，使用 `ADD PARTITION` 子句添加分区，如下所示：

```sql
alter table browns add partition new_part tablespace p_tbsp update indexes;
```

如果你未指定 `UPDATE INDEXES` 子句，哈希分区表上新分区的任何全局索引和本地索引都需要重建。

对于列表分区表，只有在未定义 `DEFAULT` 分区的情况下，你才能添加新分区。下一个示例向列表分区表添加一个分区：

```sql
alter table f_sales add partition reg_east values('GA');
```

### 工作原理

向表添加分区后，务必检查索引以确保它们都仍处于可用状态 (`USABLE`)：

```sql
select index_name, partition_name, status from user_ind_partitions;
```

考虑在执行维护操作时使用 `ALTER TABLE` 语句的 `UPDATE INDEXES` 子句来自动重建索引。在某些情况下，你可能无法使用 `UPDATE INDEXES` 子句，而将不得不重建任何不可用的索引。我们强烈建议你始终在非生产数据库中测试维护操作，以确定任何不可预见的副作用。

##### 15-14. 将分区与现有表交换

### 问题

你已经将数据加载到一个临时表（staging table）中。现在你想将该表作为分区表的一个分区。

### 解决方案

使用分区交换功能将独立表与表分区交换。一个简单的示例将说明该过程。假设你创建了一个范围分区表，如下所示：

```sql
create table f_sales
(sales_amt number, d_date_id number)
partition by range (d_date_id)
(partition p_2007 values less than (20080101),
partition p_2008 values less than (20090101),
partition p_2009 values less than (20100101)
);
```

并且，在 `D_DATE_ID` 列上创建了一个位图索引：

```sql
create bitmap index d_date_id_fk1 on
f_sales(d_date_id) local;
```

现在向表中添加一个新分区以存储新数据：

```sql
alter table f_sales add partition p_2010
values less than(20110101);
```

接下来，创建一个临时表并插入属于新添加分区值范围的数据：

```sql
create table workpart(sales_amt number, d_date_id number);
insert into workpart values(100,20100101);
insert into workpart values(120,20100102);
```

在 `WORKPART` 表上创建一个与 `F_SALES` 上的位图索引结构匹配的位图索引：

```sql
create bitmap index d_date_id_fk2 on workpart(d_date_id);
```

现在交换 `WORKPART` 表与 `P_2010` 分区：

```sql
alter table f_sales exchange partition p_2010
with table workpart including indexes without validation;
```

快速查询 `F_SALES` 表可验证分区已成功交换：

```sql
select * from f_sales partition(p_2010);
```

输出如下：

```
SALES_AMT D_DATE_ID
---------- ----------
       100 20100101
       120 20100102
```

此查询将显示所有索引仍可用：

```sql
select index_name, partition_name, status from user_ind_partitions;
```

你还可以验证是否为新区创建了本地索引段：

```sql
select segment_name,segment_type,partition_name
from user_segments
where segment_name IN('F_SALES','D_DATE_ID_FK1');
```

输出如下：

```
SEGMENT_NAME       SEGMENT_TYPE       PARTITION_NAME
-------------------- -------------------- --------------------
D_DATE_ID_FK1       INDEX PARTITION    P_2007
D_DATE_ID_FK1       INDEX PARTITION    P_2008
D_DATE_ID_FK1       INDEX PARTITION    P_2009
D_DATE_ID_FK1       INDEX PARTITION    P_2010
F_SALES             TABLE PARTITION    P_2007
F_SALES             TABLE PARTITION    P_2008
F_SALES             TABLE PARTITION    P_2009
F_SALES             TABLE PARTITION    P_2010
```

### 工作原理


