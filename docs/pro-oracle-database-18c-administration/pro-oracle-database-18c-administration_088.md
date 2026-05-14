# 查看消耗 Undo 空间的 SQL

有时，一段代码未能正确提交，这会导致大量空间被分配在 undo 表空间中且永不释放。迟早，你会遇到 `ORA-30036` 错误，表明表空间无法扩展。通常，第一次出现空间相关错误时，你可以简单地增加与 undo 表空间关联的某个数据文件的大小。
然而，如果某个 SQL 语句持续运行并填满了新添加的空间，那么问题很可能出在一个写得不好的应用程序上。例如，开发者可能没有在代码中包含适当的 `commit` 语句。

在这种情况下，识别哪些用户正在消耗 undo 表空间中的空间是很有帮助的。运行此查询可以报告基于每个用户分配的空间的基本信息：

```sql
SQL> select s.sid, s.serial#, s.osuser, s.logon_time
       ,s.status, s.machine
       ,t.used_ublk, t.used_ublk*16384/1024/1024 undo_usage_mb
  from v$session     s
      ,v$transaction t
 where t.addr = s.taddr;
```

如果你想查看与消耗 undo 空间的用户相关联的 SQL 语句，则可以连接到 `V$SQL`，如下所示：

```sql
SQL> select s.sid, s.serial#, s.osuser, s.logon_time, s.status
       ,s.machine, t.used_ublk
       ,t.used_ublk*16384/1024/1024 undo_usage_mb
       ,q.sql_text
  from v$session     s
      ,v$transaction t
      ,v$sql         q
 where t.addr = s.taddr
   and s.sql_id = q.sql_id;
```

如果需要更多信息，例如回滚段的名称和状态，可以运行连接到 `V$ROLLNAME` 和 `V$ROLLSTAT` 视图的查询，像这样：

```sql
SQL> select s.sid, s.serial#, s.username, s.program
       ,r.name undo_name, rs.status
       ,rs.rssize/1024/1024 redo_size_mb
       ,rs.extents
  from v$session     s
      ,v$transaction t
      ,v$rollname    r
      ,v$rollstat    rs
 where s.taddr = t.addr
   and t.xidusn  = r.usn
   and r.usn     = rs.usn;
```

之前的查询使你能够精确定位哪些用户负责在 undo 表空间内分配的空间。当代码未在适当时候提交并过度消耗 undo 空间时，这尤其有用。

