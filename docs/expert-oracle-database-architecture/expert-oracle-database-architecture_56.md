# 临时表类型对比

现在，让我们来看看两种类型之间的差异：

```
SQL> insert into temp_table_session select * from scott.emp;
14 rows created.
SQL> insert into temp_table_transaction select * from scott.emp;
14 rows created.
```

我们刚刚向每个 `TEMP` 表中插入了 14 行数据，这意味着我们可以查看它们：

```
SQL> select session_cnt, transaction_cnt
from ( select count(*) session_cnt from temp_table_session ),
( select count(*) transaction_cnt from temp_table_transaction );
SESSION_CNT TRANSACTION_CNT
----------- ---------------
14              14
SQL> commit;
```

由于我们已经提交（commit），我们将看到基于会话（session-based）的行，但看不到基于事务（transaction-based）的行：

```
SQL> select session_cnt, transaction_cnt
from ( select count(*) session_cnt from temp_table_session ),
( select count(*) transaction_cnt from temp_table_transaction );
SESSION_CNT TRANSACTION_CNT
----------- ---------------
14               0
SQL> disconnect
SQL> connect eoda@PDB1
Enter password:
Connected.
```

由于我们已经开始了一个新的会话，我们将看不到任何表中的行：

```
SQL> select session_cnt, transaction_cnt
from ( select count(*) session_cnt from temp_table_session ),
( select count(*) transaction_cnt from temp_table_transaction );
SESSION_CNT TRANSACTION_CNT
----------- ---------------
0               0
```

你可以通过查询 `USER_TABLES` 视图的 `TEMPORARY` 和 `DURATION` 列来检查一个表是否被创建为临时表以及数据的持续时间（按会话或按事务）。默认的 `DURATION` 是 `SYS$TRANSACTION`（意味着 `ON COMMIT DELETE ROWS`）。以下是在本示例中这些值的显示：

```
SQL> select table_name, temporary, duration from user_tables;
TABLE_NAME                T DURATION
------------------------- - ---------------
TEMP_TABLE_TRANSACTION    Y SYS$TRANSACTION
TEMP_TABLE_SESSION        Y SYS$SESSION
```

