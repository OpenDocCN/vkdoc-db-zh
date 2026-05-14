# Oracle Scheduler 任务管理与配置

## 提示

为本地和远程外部作业设置凭据。Oracle 可以使用默认用户，但出于安全策略和捕获作业执行情况的考虑，最好创建一个用户来验证外部作业。`DBMS_CREDENTIAL` 存储用户详细信息，并且可以存储 Windows 域用户（例如服务帐户）来执行这些作业。凭据由 `SYS` 拥有，也可以使用 `DBMS_CREDENTIAL` 进行管理。

在此示例中，`AUTO_DROP` 参数被设置为 `FALSE`。这指示 Oracle 调度程序在作业运行后不要自动删除它（默认值为 `TRUE`）。

## 查看作业详情

要查看作业的配置方式，请查询 `DBA_SCHEDULER_JOBS` 视图。此查询选择 `RMAN_BACKUP` 作业的信息：

```sql
SQL> SELECT job_name
,last_start_date
,last_run_duration
,next_run_date
,repeat_interval
FROM dba_scheduler_jobs
WHERE job_name='RMAN_BACKUP';
```

每次作业运行时，其执行记录都会记录在数据字典中。要检查作业执行的状态，请查询 `DBA_SCHEDULER_JOB_LOG` 视图。对于作业每次运行，都应有一个条目：

```sql
SQL> SELECT job_name
,log_date
,operation
,status
FROM dba_scheduler_job_log
WHERE job_name='RMAN_BACKUP';
```

## 修改作业日志历史记录

默认情况下，Oracle 调度程序保留 30 天的日志历史记录。你可以通过 `SET_SCHEDULER_ATTRIBUTE` 过程修改默认的保留期。例如，此命令将默认天数更改为 15：

```sql
SQL> exec dbms_scheduler.set_scheduler_attribute('log_history',15);
```

要完全清除日志历史记录的内容，请使用 `PURGE_LOG` 过程：

```sql
SQL> exec dbms_scheduler.purge_log();
```

## 修改作业

你可以通过 `SET_ATTRIBUTE` 过程修改作业的各种属性。此示例将 `RMAN_BACKUP` 作业修改为每周一运行：

```sql
SQL> BEGIN
dbms_scheduler.set_attribute(
name=>'RMAN_BACKUP'
,attribute=>'repeat_interval'
,value=>'freq=weekly; byday=mon');
END;
/
```

你可以通过从 `DBA_SCHEDULER_JOBS` 视图中选择 `REPEAT_INTERVAL` 列来验证更改。以下是 `RMAN_BACKUP` 作业的 `REPEAT_INTERVAL` 列现在的显示内容：

```
21-JAN-18 12.00.00.200000 AM -07:00 freq=weekly; byday=mon
```

从先前的输出中，你可以看到作业将在下一个星期一运行，并且由于没有指定 `BYHOUR` 和 `BYMINUTE` 选项（在修改作业时），作业被安排在默认时间 12:00 AM 运行。

## 停止作业

如果某个作业运行时间异常长，你可能希望中止它。使用 `STOP_JOB` 过程来停止当前正在运行的作业。此示例在 `RMAN_BACKUP` 作业运行时将其停止：

```sql
SQL> exec dbms_scheduler.stop_job(job_name=>'RMAN_BACKUP');
```

`DBA_SCHEDULER_JOB_LOG` 的 `STATUS` 列将为使用 `STOP_JOB` 过程停止的作业显示 `STOPPED`。

## 禁用作业

你可能希望临时禁用一个运行不正确的作业。在排除故障时，你需要确保该作业不会运行。使用 `DISABLE` 过程来禁用作业：

```sql
SQL> exec dbms_scheduler.disable('RMAN_BACKUP');
```

如果作业当前正在运行，请考虑先停止作业或使用 `DISABLE` 过程的 `FORCE` 选项：

```sql
SQL> exec dbms_scheduler.disable(name=>'RMAN_BACKUP',force=>true);
```

## 启用作业

你可以通过 `DBMS_SCHEDULER` 包的 `ENABLE` 过程来启用先前禁用的作业。此示例重新启用 `RMAN_BACKUP` 作业：

```sql
SQL> exec dbms_scheduler.enable(name=>'RMAN_BACKUP');
```

## 提示

你可以通过从 `DBA_SCHEDULER_JOBS` 视图中选择 `ENABLED` 列来检查作业是被禁用还是已启用。

## 复制作业

如果你想克隆一个当前的作业，可以使用 `COPY_JOB` 过程来完成。该过程接受两个参数：旧作业名和新作业名。以下是一个复制作业的示例，其中 `RMAN_BACKUP` 是先前创建的作业，`RMAN_NEW_BACK` 是将要创建的新作业：

```sql
begin
dbms_scheduler.copy_job('RMAN_BACKUP','RMAN_NEW_BACK');
end;
/
```

复制的作业将被创建但处于禁用状态。在运行之前，你必须先启用该作业（参见前一节的示例）。

## 手动运行作业

你可以在其常规计划之外手动运行作业。你可能想这样做以测试作业，确保其正常工作。使用 `RUN_JOB` 过程手动启动作业。此示例手动运行先前创建的 `RMAN_BACKUP` 作业：

```sql
SQL> BEGIN
DBMS_SCHEDULER.RUN_JOB(
JOB_NAME => 'RMAN_BACKUP',
USE_CURRENT_SESSION => FALSE);
END;
/
```

`USE_CURRENT_SESSION` 参数指示 Oracle 调度程序是否以当前用户身份运行作业（或不）。值为 `FALSE` 指示调度程序以最初创建和安排作业的用户身份运行作业。

## 删除作业

如果你不再需要某个作业，应将其从调度程序中删除。使用 `DROP_JOB` 过程永久删除作业。此示例移除 `RMAN_BACKUP` 作业：

```sql
SQL> BEGIN
dbms_scheduler.drop_job(job_name=>'RMAN_BACKUP');
END;
/
```

该代码将删除作业，并从 `DBA_SCHEDULER_JOBS` 视图中移除有关已删除作业的任何信息。

