# 在索引上设置 NOLOGGING

使用 `NOLOGGING` 选项有两种方法。你已经见过一种——在 SQL 命令中嵌入 `NOLOGGING` 关键字。另一种方法涉及在段（索引或表）上设置 `NOLOGGING` 属性，它允许某些操作在 `NOLOGGING` 模式下隐式执行。例如，我可以将索引或表的默认模式更改为 `NOLOGGING`。这意味着对于该索引，后续的索引重建操作将不会被记录（索引不会产生重做日志；其他索引和表本身可能会产生，但这个索引不会）。使用我们刚刚创建的表 `T`，我们可以观察到：

```
$ sqlplus eoda/foo@PDB1
SQL> select log_mode, force_logging from v$database;
LOG_MODE     FORCE_LOGGING
------------ -----------------
ARCHIVELOG   NO
SQL> create index t_idx on t(object_name);
Index created.
SQL> variable redo number
SQL> exec :redo := get_stat_val( 'redo size' );
PL/SQL procedure successfully completed.
SQL> alter index t_idx rebuild;
Index altered.
SQL> exec dbms_output.put_line( (get_stat_val('redo size')-:redo)
|| ' bytes of redo generated...');
672264 bytes of redo generated...
PL/SQL procedure successfully completed.
```

注意

再次强调，此示例是在 `ARCHIVELOG` 模式数据库中执行的。在 `NOARCHIVELOG` 模式数据库中，你不会看到重做日志大小的差异，因为索引的 `CREATE` 和 `REBUILD` 操作在 `NOARCHIVELOG` 模式下不会被记录。

当索引处于 `LOGGING` 模式时（默认），重建它会产生大约 600KB 的重做日志。但是，我们可以更改索引：

```
SQL> alter index t_idx nologging;
Index altered.
SQL> exec :redo := get_stat_val( 'redo size' );
PL/SQL procedure successfully completed.
SQL> alter index t_idx rebuild;
Index altered.
SQL> exec dbms_output.put_line( (get_stat_val('redo size')-:redo)
|| ' bytes of redo generated...');
39352 bytes of redo generated...
PL/SQL procedure successfully completed.
```

现在它只产生了 39KB 的重做日志。但该索引现在是“无保护”的。如果它所在的数据库文件发生故障并需要从备份中恢复，我们将丢失该索引数据。理解这一点至关重要。该索引目前是不可恢复的——我们需要进行备份。或者，DBA 可以直接重新创建索引，因为我们也能够直接从表数据重新创建索引。

