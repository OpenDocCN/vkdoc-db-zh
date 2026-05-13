# 第 15 章 分区

## MAXVALUE 参数与智能代理键

`MAXVALUE` 参数用于创建一个分区，以存储不符合其他已定义分区范围的行。您不必指定 `MAXVALUE`。但是，如果您尝试插入一个不符合现有分区范围的值，系统将抛出错误。

您可能已经注意到，在 `F_REGS` 表中，我们将 `D_DATE_ID` 列用作数字数据类型，而非日期数据类型。`D_DATE_ID` 是一个从 `D_DATES` 维度表指向 `F_REGS` 事实表的外键。数据仓库环境中采用的一种技术是在 `D_DATES` 维度的主键上使用**智能代理键**。对于日期使用智能键的主要原因在于物理数据库设计，特别是当您希望基于日期字段进行范围分区时。这允许您在外键数字（看起来像一个日期）上进行分区。

##### 15-3. 按列表分区

### 问题

您希望根据列表中的值（例如州代码列表）对表进行分区。

### 解决方案

使用 `CREATE TABLE` 语句的 `PARTITION BY LIST` 子句。此示例按 `STATE_CODE` 列进行分区：

```sql
create table f_sales
(
 reg_sales number
,d_date_id number
,state_code varchar2(20)
)
partition by list (state_code)
(
 partition reg_west values ('AZ','CA','CO','MT','OR','ID','UT','NV')
,partition reg_mid values ('IA','KS','MI','MN','MO','NE','OH','ND')
,partition reg_rest values (default)
);
```

### 工作原理

列表分区表的分区键只能是一列。使用 `DEFAULT` 列表来指定一个分区，用于存储其值不匹配列表中任何值的行。运行此 SQL 语句可查看每个分区的列表值：

```sql
select table_name, partition_name, high_value
from user_tab_partitions
order by 1;
```

`HIGH_VALUE` 列将显示为每个分区定义的列表值。此列的数据类型为 `LONG`。如果您使用 `SQL*Plus`，可能需要将 `LONG` 变量的值设置得比默认值（80 字节）更高，以显示该列的全部内容：

```sql
SQL> set long 1000
```

##### 15-4. 按哈希分区

### 问题

您有一个表，其中没有包含明显的列可以进行范围或列表分区。您希望根据表中的一个 ID 列将数据均匀地分散到各个分区中。

### 解决方案

使用 `CREATE TABLE` 语句的 `PARTITION BY HASH` 子句。此示例创建一个表，该表被划分为三个分区，每个分区位于自己的表空间中。

```sql
create table browns(
 brown_id number
,bear_name varchar2(30))
partition by hash(brown_id)
partitions 3
store in(tbsp1, tbsp2, tbsp3);
```

当然，您需要修改表空间名称等细节以匹配您环境中的名称。或者，您可以省略 `STORE IN` 子句，Oracle 会将所有分区放置在您的默认表空间中。如果您想同时为表空间和分区命名，可以按如下方式指定：

```sql
create table browns(
 brown_id number
,bear_name varchar2(30))
partition by hash(brown_id)
(
 partition p1 tablespace tbsp1
,partition p2 tablespace tbsp2
,partition p3 tablespace tbsp3
);
```

### 工作原理

哈希分区根据一种算法将行映射到分区，该算法会将数据均匀地分散到所有分区中。您无法控制哈希算法或 Oracle 分布数据的方式。

您指定所需的分区数量，Oracle 会根据哈希键列将数据均匀划分。

哈希分区具有一些有趣的性能影响。所有哈希键值相同的行将被插入到同一个分区中。这意味着插入操作特别高效，因为哈希算法确保了数据在各分区之间的均匀分布。此外，如果您通常针对特定的键值进行选择，Oracle 只需访问一个分区即可检索这些行。

[电子书下载](http://www.it-ebooks.info/)



但是，如果按值范围进行搜索，Oracle 很可能需要扫描每个分区来确定要检索的行。因此，范围搜索在哈希分区表中可能表现不佳。

##### 15-5. 以多种方式对表进行分区

### 问题

你有一个表，想按数字范围对其进行分区，但你还希望按区域列表对每个分区进行进一步细分。

### 解决方案

使用组合分区来指示 Oracle 以多种方式对表进行分区。以下示例首先按数字范围对表进行分区，然后通过列表分区进一步分发数据：

```sql
create table f_sales(
  sales_amnt number
 ,reg_code varchar2(3)
 ,d_date_id number
)
partition by range(d_date_id)
subpartition by list(reg_code)
(partition p2010 values less than (20100101)
 (subpartition p1_north values ('ID','OR')
  ,subpartition p1_south values ('AZ','NM')
 ),
 partition p2011 values less than (20110101)
 (subpartition p2_north values ('ID','OR')
  ,subpartition p2_south values ('AZ','NM')
 )
);
```

你可以通过运行以下查询来查看子分区信息：

```sql
select table_name, partitioning_type, subpartitioning_type
from user_part_tables;
```

以下是一些示例输出：

```
TABLE_NAME           PARTITIONING_TYPE    SUBPARTITIONING_TYPE
-------------------- -------------------- ---------------------
F_SALES              RANGE                LIST
```

运行下一个查询以查看有关子分区的信息：

```sql
select table_name, partition_name, subpartition_name
from user_tab_subpartitions;
```

以下是部分输出内容：

```
TABLE_NAME           PARTITION_NAME       SUBPARTITION_NAME
-------------------- -------------------- --------------------
F_SALES              P2010                P1_SOUTH
F_SALES              P2010                P1_NORTH
F_SALES              P2011                P2_SOUTH
F_SALES              P2011                P2_NORTH
```

第 15 章 ■ 分区

### 工作原理

Oracle 允许你组合两种分区策略，如下所示：

- 范围-范围分区
- 范围-哈希分区
- 范围-列表分区
- 列表-范围分区
- 列表-哈希分区
- 列表-列表分区

组合分区在数据划分方式上提供了极大的灵活性。你可以使用这些方法来进一步细化数据的分布和维护方式。

##### 15-6. 按需创建分区

### 问题

你有一个范围分区表，并希望当插入的值高于为最高范围定义的最高值时，Oracle 能够自动添加一个分区。

### 解决方案

如果你使用的是 Oracle Database `11g` 或更高版本，可以使用 `CREATE TABLE` 语句的 `INTERVAL` 子句来指示 Oracle 自动向范围分区表的高端添加分区。

此示例创建了一个表，该表最初有一个分区，其高值范围为 2010 年 1 月 1 日。

```sql
create table f_sales(
  sales_amt number
 ,d_date date
)
partition by range (d_date)
interval(numtoyminterval(1, 'YEAR'))
(partition p1 values less than (to_date('01-jan-2010','dd-mon-yyyy')));
```

此示例中的间隔为 1 年，由 `INTERVAL(NUMTOYMINTERVAL(1, 'YEAR'))` 子句定义。

如果向表中插入的记录的 `D_DATE` 值大于或等于 2010 年 1 月 1 日，Oracle 将自动在表的高端添加一个新分区。你可以通过运行此 SQL 语句来检查分区的详细信息：

```sql
select table_name, partition_name, partition_position, high_value
from user_tab_partitions order by table_name, partition_name;
```

以下是一些示例输出（`HIGH_VALUE` 列已被截短以便输出适应页面）：

```
TABLE_NAME           PARTITION_NAME       PARTITION_POSITION   HIGH_VALUE
----------           ------------         -------------------- ----------
F_SALES              P1                   1                    TO_DATE(' 2010-01-01 00:00:00',...
```

现在，我们插入一些高于最高分区高值的数据：

```sql
insert into f_sales values(1,sysdate+1000);
```

这是现在从 `USER_TAB_PARTITIONS` 选择后显示的输出：

```
TABLE_NAME           PARTITION_NAME       PARTITION_POSITION   HIGH_VALUE
----------           ------------         -------------------- ----------
```



