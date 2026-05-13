# 第十五章 ■ 分区

你还可以指定任何其他存储设置。下一个示例显式设置了 `PCTFREE`、`PCTUSED` 和 `NOLOGGING` 子句：

```sql
create table f_regs
(reg_count number
,d_date_id number
)
partition by range (d_date_id)(
partition p_2009 values less than (20100101)
tablespace p_2009_tbsp pctfree 5 pctused 90 nologging,
partition p_2010 values less than (20110101)
tablespace p_2010_tbsp pctfree 5 pctused 90 nologging,
partition p_max values less than (maxvalue)
tablespace p_max_tbsp pctfree 5 pctused 90 nologging
);
```

## 工作原理

将分区放置在独立表空间的一个优点是，你可以独立地备份和恢复分区。如果某个分区未被修改，你可以将其表空间更改为只读，并指示像 `Oracle Recovery Manager` (`RMAN`) 这样的实用程序跳过备份此类表空间，从而提高备份性能。还有其他优点。在各自的表空间中创建每个分区有助于将数据从 `OLTP` 数据库移动到 `DSS`（决策支持系统）数据库，并且它允许将特定的表空间及相应的数据文件放置在独立的存储设备上，以提高可扩展性和性能。

如果你已经创建了一个表，并希望将其分区移动到特定的表空间，请使用 `ALTER TABLE...MOVE PARTITION` 语句将表分区重新定位到不同的表空间。此示例将 `P_2009` 分区移动到 `INV_MGMT_DATA` 表空间：

```sql
alter table f_sales move partition p_2009 tablespace inv_mgmt_data;
```

当你将表分区移动到不同的表空间时，索引将失效。因此，你必须重建与已移动表分区关联的任何本地索引。

重建索引时，你可以将其保留在原处，也可以将其移动到不同的表空间。此示例重建 `D_DATE_ID_FK1` 索引的 `P_2009` 分区，并将其移动到 `INV_MGMT_INDEX` 表空间：

```sql
alter index d_date_id_fk1 rebuild partition p_2009 tablespace inv_mgmt_index;
```

当你将表分区移动到不同的表空间时，会导致表分区中每条记录的 `ROWID` 发生变化。由于常规索引在其结构中存储了表 `ROWID`，如果表分区移动，索引分区将失效。在这种情况下，你必须重建索引。

重建索引分区时，你可以选择将其移动到不同的表空间。

## 15-11\. 自动移动更新后的行

### 问题

你尝试将分区键更新为一个会导致该行属于不同分区的值：

```sql
update f_regs set d_date_id = 20100901 where d_date_id = 20090201;
```

你收到以下错误：

`ORA-14402: updating partition key column would cause a partition change`

你想知道是否有办法修改表，使得当分区键的更改需要将行移动到不同的分区时，Oracle 会直接移动该行。

### 解决方案

使用 `ALTER TABLE` 语句的 `ENABLE ROW MOVEMENT` 子句，以允许更新分区键（该更新会改变一个值所属的分区）。在此示例中，首先修改 `F_REGS` 表以启用行移动：

```sql
alter table f_regs enable row movement;
```

现在你应该能够将分区键更新为一个将行移动到不同段的值。你可以通过查询 `USER_TABLES` 视图的 `ROW_MOVEMENT` 列来验证行移动是否已启用：

```sql
select row_movement from user_tables where table_name='F_REGS';
```

你应该看到值为 `ENABLED`：

`ROW_MOVE`
`---------`
`ENABLED`

### 工作原理

在你能够将分区键更新为一个需要将行移动到不同分区的值之前，你需要启用行移动。这允许 Oracle 在物理上将行移动到不同的数据块。

要禁用行移动，请使用 `DISABLE ROW MOVEMENT` 子句：


