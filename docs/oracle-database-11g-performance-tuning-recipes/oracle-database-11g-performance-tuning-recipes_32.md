# 解决方案

您可以通过使用 `DBMS_AUTO_TASK_ADMIN` 包中的 `enable` 过程来启用自动统计信息收集。可通过以下方式检查自动优化器统计信息收集任务的状态：

```sql
SQL> select client_name,status from dba_autotask_client;

CLIENT_NAME                                                          STATUS
---------------------------------------------------------------- --------
auto optimizer stats collection                                     DISABLED
auto space advisor                                                  ENABLED
sql tuning advisor                                                  ENABLED
SQL>
```

执行 `dbms_auto_task_admin.enable` 过程以启用自动统计信息收集任务：

```sql
SQL> begin dbms_auto_task_admin.enable(
  2  client_name=>'auto optimizer stats collection',
  3  operation=>NULL,
  4  window_name=>NULL);
  5  end;
  6  /

PL/SQL procedure successfully completed.
```

检查 `auto optimizer stats collection` 任务的状态：

```sql
SQL>  SELECT client_name,status from dba_autotask_client;

CLIENT_NAME                                                          STATUS
---------------------------------------------------------------- --------
auto optimizer stats collection                                     ENABLED
auto space advisor                                                  ENABLED
sql tuning advisor                                                  ENABLED

SQL>
```

您可以使用 `dbms_auto_task_admin.disable` 过程来禁用统计信息收集任务：

```sql
SQL> begin
  2  dbms_auto_task_admin.disable(
  3  client_name=> 'auto optimizer stats collection',
  4  operation=> NULL,
  5  window_name=> NULL);
  6  end;
  7  /

PL/SQL procedure successfully completed.

SQL>
```

