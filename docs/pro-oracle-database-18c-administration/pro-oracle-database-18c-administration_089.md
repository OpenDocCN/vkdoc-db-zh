# 处理临时表空间问题

临时表空间的问题相对容易发现。
例如，当临时表空间空间不足时，将抛出以下错误：

```sql
ORA-01652: unable to extend temp segment by 128 in tablespace TEMP
```

当你看到此错误时，你需要确定是临时表空间空间不足，还是一个罕见的失控 SQL 查询暂时消耗了异常大量的临时空间。这两个问题都将在以下部分讨论。

## 确定临时表空间大小是否正确

当进程耗尽可用内存并需要更多空间时，临时表空间被用作磁盘上的排序区。需要排序区的操作包括：

*   索引创建
*   SQL 排序操作
*   临时表和索引
*   临时 LOB
*   临时 B 树

没有确切的公式来确定你的临时表空间大小是否正确。它取决于查询的数量和类型、索引构建操作、并行操作以及你的内存排序空间（PGA）的大小。你必须在数据库有负载时监控你的临时表空间以建立其使用模式。由于 TEMP 表空间是临时文件，它们的处理方式与数据文件不同，详情在另一个视图中。运行以下查询以显示临时表空间中已分配和空闲的空间：

```sql
SQL> select tablespace_name
       ,tablespace_size/1024/1024 mb_size
       ,allocated_space/1024/1024 mb_alloc
       ,free_space/1024/1024      mb_free
  from dba_temp_free_space;
```

以下是一些示例输出：

```sql
TABLESPACE_NAME    MB_SIZE   MB_ALLOC    MB_FREE
--------------- ---------- ---------- ----------
TEMP                   200        200        170
```

如果 `FREE_SPACE` (`MB_FREE`) 值降至接近 0，说明数据库中有 SQL 操作消耗了大部分可用空间。`FREE_SPACE` (`MB_FREE`) 列是总的可用空间，包括当前已分配和可重用的空间。

如果使用量接近你当前的分配量，你可能需要向临时表空间数据文件分配更多空间。运行以下查询以查看临时数据文件名称和分配大小：

```sql
SQL> select name, bytes/1024/1024 mb_alloc from v$tempfile;
```

以下是一些典型输出：

```sql
NAME                                       MB_ALLOC
---------------------------------------- ----------
/u01/dbfile/o18c/temp01.dbf                     400
/u01/dbfile/o18c/temp02.dbf                     100
/u01/dbfile/o18c/temp03.dbf                     100
```

在首次创建数据库时，如果你对临时表空间的“正确”大小毫无概念，通常将其大小设置为约 20GB。如果我正在构建一个数据仓库类型的数据库，我可能会将临时表空间大小设置为约 80GB。你必须使用适当的 SQL 监控你的临时表空间，并根据需要调整大小。
你还可以创建多个 TEMP 表空间以供不同的应用程序和用户使用。如果某个应用程序似乎消耗了所有 TEMP 表空间，请创建另一个 TEMP 表空间，例如 TEMP2，并将其分配给该应用程序用户。这将隔离问题，直到可以研究并修复代码、排序和索引创建。

## 查看消耗临时空间的 SQL

当 Oracle 抛出 `ORA-01652: unable to extend temp` 错误时，这可能表明你的临时表空间太小。然而，如果 Oracle 因一次性事件（例如大型索引构建）而耗尽空间，它也可能抛出该错误。你必须决定是否需要为空间不足的一次性索引构建或消耗大量临时表空间中排序空间的查询添加空间。

要查看会话在临时表空间中使用的空间，请运行此查询：

```sql
SQL> SELECT s.sid, s.serial#, s.username
       ,p.spid, s.module, p.program
       ,SUM(su.blocks) * tbsp.block_size/1024/1024 mb_used
       ,su.tablespace
  FROM v$sort_usage    su
      ,v$session       s
      ,dba_tablespaces tbsp
      ,v$process       p
 WHERE su.session_addr = s.saddr
   AND su.tablespace   = tbsp.tablespace_name
   AND s.paddr         = p.addr
 GROUP BY
       s.sid, s.serial#, s.username, s.osuser, p.spid, s.module,
       p.program, tbsp.block_size, su.tablespace
 ORDER BY s.sid;
```

如果你确定需要添加空间，可以调整现有临时文件的大小或添加一个新的。要调整临时表空间临时文件的大小，请使用 `ALTER DATABASE TEMPFILE...RESIZE` 语句。以下命令将临时数据文件大小调整为 12GB：

```sql
SQL> alter database tempfile '/u01/dbfile/o18c/temp02.dbf' resize 12g;
```

你可以向临时表空间添加一个数据文件，如下所示：

```sql
SQL> alter tablespace temp add tempfile '/u02/dbfile/o18c/temp04.dbf' size 2g;
```


