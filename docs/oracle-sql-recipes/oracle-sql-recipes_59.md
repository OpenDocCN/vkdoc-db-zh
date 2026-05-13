# 序列管理

`create sequence inv2 start with 10000 maxvalue 1000000;`

要删除序列，请使用 `DROP SEQUENCE` 语句：

```sql
drop sequence inv;
```

如果您拥有 `DBA` 权限，可以查询 `DBA_SEQUENCES` 视图以显示数据库中所有序列的信息。要查看您自己模式拥有的序列，请查询 `USER_SEQUENCES` 视图：

```sql
select sequence_name, min_value, max_value, increment_by from user_sequences;
```

## 重置序列号

要重置序列号，您可以删除并以期望的起点重新创建该序列。以下代码删除并重新创建一个序列，使其从数字 1 开始：

```sql
drop sequence cia_seq;
create sequence cia_seq start with 1;
```

之前这种重置序列号的方法存在一个问题：如果有其他模式访问该序列，它们将需要被重新授予对该序列的 `select` 权限。这是因为当您删除一个对象时，与该对象相关的所有权限授予也会被删除。

除了删除并重建序列之外，另一种方法是先将序列的 `INCREMENT BY` 修改为比您想要重置的值小 1 的整数，然后再将序列的 `INCREMENT BY` 修改回 1。这有效地重置了序列，而无需删除并重建，也消除了重新授予 `select` 权限的需要。以下几行 SQL 代码展示了此技术：

```sql
UNDEFINE seq_name
UNDEFINE reset_to
PROMPT "sequence name"
ACCEPT '&&seq_name'
PROMPT "reset to value"
ACCEPT &&reset_to
COL seq_id NEW_VALUE hold_seq_id
COL min_id NEW_VALUE hold_min_id
--
SELECT &&reset_to - &&seq_name..nextval - 1 seq_id
FROM dual;
--
SELECT &&hold_seq_id - 1 min_id
FROM dual;
--
ALTER SEQUENCE &&seq_name INCREMENT BY &hold_seq_id MINVALUE &hold_min_id;
--
SELECT &&seq_name..nextval FROM dual;
--
ALTER SEQUENCE &&seq_name INCREMENT BY 1;
```

为确保序列已设置为您的期望值，请从中选择 `NEXTVAL`：

```sql
select &&seq_name..nextval from dual;
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

