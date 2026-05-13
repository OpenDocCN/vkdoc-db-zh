# 第 11 章 集中化审计数据

## 清单 11-8. 用于收集审计数据的 SQL Server 代理作业

```sql
DECLARE @CentralServerName varchar(100);
DECLARE @AuditFilePath varchar(250);

/* 只修改这两个变量 */
SET @CentralServerName = 'yourcentralservername';
SET @AuditFilePath = 'e:\sqlaudit\*.sqlaudit';

/*
除非你的 SQL Server 版本低于 2019，否则不要修改此处以下的任何内容
*/

DROP TABLE IF EXISTS ##tempvariables;

DECLARE @sql varchar(max);

SET @sql = N'INSERT INTO ' + @CentralServerName + '.Auditing.dbo.
AuditChanges
SELECT DATEADD(mi, DATEPART(TZ, SYSDATETIMEOFFSET()), event_time) as
event_time, statement,
server_instance_name, database_name, schema_name, session_server_
principal_name,
server_principal_name, object_Name, file_name, client_ip,
application_name,
host_name, succeeded, aa.name AS audit_action
FROM sys.fn_get_audit_file ('''+@AuditFilePath+''',default,
default) af
INNER JOIN sys.dm_audit_actions aa
ON aa.action_id = af.action_id
第 11 章 集中化审计数据
WHERE DATEADD(mi, DATEPART(TZ, SYSDATETIMEOFFSET()), event_time) >
DATEADD(HOUR, -4, GETDATE());';

SELECT @CentralServerName AS servername,
@AuditFilePath as auditfilepath, @sql as stepsql
into ##tempvariables;

USE [msdb]
GO

BEGIN TRANSACTION
DECLARE @ReturnCode INT
SELECT @ReturnCode = 0

IF NOT EXISTS (SELECT name FROM msdb.dbo.syscategories WHERE
name=N'[Uncategorized (Local)]' AND category_class=1)
BEGIN
EXEC @ReturnCode = msdb.dbo.sp_add_category @class=N'JOB', @type=N'LOCAL',
@name=N'[Uncategorized (Local)]'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
END

DECLARE @jobId BINARY(16)
EXEC @ReturnCode = msdb.dbo.sp_add_job @job_name=N'Audit Changes
Collection',
@enabled=1,
@notify_level_eventlog=0,
@notify_level_email=0,
@notify_level_netsend=0,
@notify_level_page=0,
@delete_level=0,
@description=N'No description available.',
@category_name=N'[Uncategorized (Local)]',
@owner_login_name=N'sa', @job_id = @jobId OUTPUT
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback

DECLARE @stepsql varchar(max);
SET @stepsql = (SELECT stepsql from ##tempvariables);

EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId,
@step_name=N'audit sql server changes',
第 11 章 集中化审计数据
@step_id=1,
@cmdexec_success_code=0,
@on_success_action=1,
@on_success_step_id=0,
@on_fail_action=2,
@on_fail_step_id=0,
@retry_attempts=3,
@retry_interval=3,
@os_run_priority=0, @subsystem=N'TSQL',
@command=@stepsql,
@database_name=N'master',
@flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback

EXEC @ReturnCode = msdb.dbo.sp_update_job @job_id = @jobId,
@start_step_id = 1
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback

EXEC @ReturnCode = msdb.dbo.sp_add_jobschedule @job_id=@jobId,
@name=N'every 4 hours get audit into from sqlaudit files on disk',
@enabled=1,
@freq_type=4,
@freq_interval=1,
@freq_subday_type=8,
@freq_subday_interval=4,
@freq_relative_interval=0,
@freq_recurrence_factor=0,
@active_start_date=20190812,
@active_end_date=99991231,
@active_start_time=0,
@active_end_time=235959,
@schedule_uid=N'c68b91ed-4f7f-4fe4-874a-670982cb20cb'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback

EXEC @ReturnCode = msdb.dbo.sp_add_jobserver @job_id = @jobId, @server_name =
N'(local)'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback

COMMIT TRANSACTION
第 11 章 集中化审计数据
GOTO EndSave

QuitWithRollback:
IF (@@TRANCOUNT > 0) ROLLBACK TRANSACTION

EndSave:
GO
```

请确保在清单 11-8 的脚本顶部设置你的`@CentralServerName`和`@AuditFilePath`变量。对于 SQL Server 2019，无需更改`@sql`变量。此作业将从一台服务器收集审计数据，并每四小时发送到中央服务器一次。

要在 SQL Server 2017 中查询审计数据，请修改`@sql`变量并删除`host_name`列，因为该列在 2017 版本中不可用。要在 SQL Server 2016 中查询审计数据……



对于 SQL Server 2017 之前的版本，需要修改 `@sql` 变量，并移除 `host_name`、`application_name` 和 `client_ip` 列，因为这些列在 2017 之前的版本中不可用。

您还需要一个 SQL Server Agent 作业来清理集中审计数据库中的审计数据，以确保仅保留所需的审计数据量，而不是永久保留。我将其设置为 30 天。您可以选择最适合您需求的时间范围。清单 11-9 提供了创建此清理代理作业的脚本。

**清单 11-9.** 用于清理审计数据的 SQL Agent 作业

```sql
USE [msdb]
GO
BEGIN TRANSACTION
DECLARE @ReturnCode INT
SELECT @ReturnCode = 0
/****** Object:  JobCategory [[Uncategorized (Local)]]    Script Date: 5/8/2022 1:53:20 PM ******/
IF NOT EXISTS (SELECT name FROM msdb.dbo.syscategories WHERE name=N'[Uncategorized (Local)]' AND category_class=1)
BEGIN
EXEC @ReturnCode = msdb.dbo.sp_add_category @class=N'JOB', @type=N'LOCAL', @name=N'[Uncategorized (Local)]'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
END
DECLARE @jobId BINARY(16)
EXEC @ReturnCode = msdb.dbo.sp_add_job @job_name=N'Audit Retention Cleanup', @enabled=1, @notify_level_eventlog=0, @notify_level_email=0, @notify_level_netsend=0, @notify_level_page=0, @delete_level=0, @description=N'No description available.', @category_name=N'[Uncategorized (Local)]', @owner_login_name=N'sa', @job_id = @jobId OUTPUT
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
/****** Object:  Step [only retain 30 days of audit data]    Script Date: 5/8/2022 1:53:20 PM ******/
EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId, @step_name=N'only retain 30 days of audit data', @step_id=1, @cmdexec_success_code=0, @on_success_action=1, @on_success_step_id=0, @on_fail_action=2, @on_fail_step_id=0, @retry_attempts=0, @retry_interval=0, @os_run_priority=0, @subsystem=N'TSQL', @command=N'USE [Auditing];
DELETE FROM [dbo].[AuditChanges]
WHERE event_time <= getdate()-30;', @database_name=N'master', @flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_update_job @job_id = @jobId, @start_step_id = 1
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_add_jobschedule @job_id=@jobId, @name=N'nightly audit tables retention cleanup', @enabled=1, @freq_type=4, @freq_interval=1, @freq_subday_type=1, @freq_subday_interval=0, @freq_relative_interval=0, @freq_recurrence_factor=0, @active_start_date=20190903, @active_end_date=99991231, @active_start_time=230000, @active_end_time=235959, @schedule_uid=N'39b449aa-2c4d-4d05-8a5f-c79598ef52bb'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_add_jobserver @job_id = @jobId, @server_name = N'(local)'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
COMMIT TRANSACTION
GOTO EndSave
QuitWithRollback:
IF (@@TRANCOUNT > 0) ROLLBACK TRANSACTION
EndSave:
GO
```

一旦将审计数据集中，查询和报告就会容易得多。在下一章中，我将展示如何使用 SQL Server Agent 作业和 PowerShell 对审计数据进行报告。

