# 第 15 章 ■ 分区技术

## 15-9\. 应用程序控制的分区

### 问题

在一种罕见的情况下，你可能希望向表中插入记录的应用程序能够显式地控制数据插入到哪个分区。

### 解决方案

如果你使用的是 Oracle Database 11g 或更高版本，可以使用 `PARTITION BY SYSTEM` 子句，允许 `INSERT` 语句指定要插入数据的分区。下面的示例创建了一个包含三个分区的系统分区表：

```sql
create table apps
(app_id number
,app_amnt number)
partition by system
(partition p1
,partition p2
,partition p3);
```

向此表插入数据时，必须指定一个分区。以下代码将一条记录插入分区 `P1`：

```sql
insert into apps partition(p1) values(1,100);
```

当执行更新或删除操作时，如果没有指定分区，Oracle 将扫描系统分区表的所有分区以查找相关行。因此，为了避免性能低下，在更新和删除时你应该指定一个分区。

### 工作原理

自 Oracle Database 11g 起，你可以创建系统分区。系统分区表适用于需要显式控制记录插入到哪个分区的*特殊情况*。这允许你的应用程序代码来管理记录在分区间的分布。我们建议，只有在无法使用 Oracle 的其他分区机制来满足你的业务需求时，才使用此功能。

## 15-10\. 使用表空间配置分区

### 问题

你想将表的每个分区放置在各自的表空间中。这将便于执行诸如独立备份和恢复各个分区的操作。

### 解决方案

如果在创建分区表或索引时没有指定表空间，分区将在你的默认表空间中创建。如果你希望将每个分区放在自己的表空间中，请使用 `TABLESPACE` 子句。当然，在 `CREATE TABLE` 语句中引用表空间之前，你必须预先创建它们：

```sql
create table f_regs
(reg_count number
,d_date_id number
)
partition by range (d_date_id)(
partition p_2009 values less than (20100101)
tablespace p_2009_tbsp,
partition p_2010 values less than (20110101)
tablespace p_2010_tbsp,
partition p_max values less than (maxvalue)
tablespace p_max_tbsp
);
```



