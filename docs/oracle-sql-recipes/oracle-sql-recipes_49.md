# 全局临时表

全局临时表存储仅存在于会话或事务期间的会话私有数据。一旦创建了临时表，它就会一直存在，直到你将其删除为止。换句话说，临时表的定义是永久的；只是其中的数据是短暂的。

你可以通过查询 `DBA/ALL/USER_TABLES` 的 `TEMPORARY` 列来查看一个表是否是临时表：
```sql
select table_name, temporary from user_tables;
```
临时表在 `TEMPORARY` 列中标记为 `Y`。普通表在 `TEMPORARY` 列中标记为 `N`。

当你在临时表中创建记录时，空间是在你的默认临时表空间中分配的。你可以通过运行以下 SQL 语句来验证这一点：
```sql
select username, contents, segtype from v$sort_usage;
```
如果你正在处理大量行，并且在选择性地检索行时需要更好的性能，你可能需要考虑在临时表的相应列上创建索引：
```sql
create index temp_index on temp_output(temp_row);
```
使用 `DROP TABLE` 命令来删除临时表：
```sql
drop table temp_output;
```

## 临时表与重做

对全局临时表的数据块所做的更改不会生成重做数据。但是，针对临时表的事务会生成回滚数据。由于回滚数据会产生重做，因此与临时表的事务相关联会有一些重做数据。你可以通过启用统计跟踪（更多详情请参见技巧 19-7）并在向临时表插入记录时查看重做大小来验证这一点：
```sql
SQL> set autotrace on
```
接下来向临时表中插入几条记录：
```sql
insert into temp_output values(1);
insert into temp_output values(1);
```
以下是部分输出片段（仅显示重做大小）：`140 redo size`

临时表的重做负载小于普通表，因为生成的重做仅与临时表事务的回滚数据相关。

[www.it-ebooks.info](http://www.it-ebooks.info/)

