# source oracle OS variables; see chapter 2 for an example of oraset script
. /var/opt/oracle/oraset RMDB1
rman target / <<EOF
spool log to '/orahome/oracle/bin/log/rmanback.log'
backup database;
spool log off;
EOF
exit 0
```

在清单 21–2 中，你使用 `DBMS_SCHEDULER` 包的 `CREATE_JOB` 过程来创建一个作业。
以 SYS 用户身份运行（从 SQL*Plus）。

**清单 21–2.** 使用 `CREATE_JOB` 过程

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_JOB(
    job_name      => 'RMAN_BACKUP',
    job_type      => 'EXECUTABLE',
    job_action    => '/orahome/oracle/bin/rmanback.bsh',
    repeat_interval => 'FREQ=DAILY;BYHOUR=14;BYMINUTE=11',
    start_date    => to_date('21–OCT-10'),
    job_class     => '"DEFAULT_JOB_CLASS"',
    auto_drop     => FALSE,
    comments      => 'RMAN backup job',
    enabled       => TRUE);
END;
/
```

在清单 21–2 中，一些参数值得额外解释。`JOB_TYPE` 参数可以是以下类型之一：`STORED_PROCEDURE`、`PLSQL_BLOCK` 或 `EXTERNAL`。在这个例子中，执行的是一个外部 shell 脚本，因此作业类型是 `EXTERNAL`。如果你有一个想要运行的 PL/SQL 存储过程，则使用 `STORED_PROCEDURE` 类型。`PLSQL_BLOCK` 类型允许你直接运行一个匿名的 PL/SQL 块。

`REPEAT_INTERVAL` 参数被设置为 `FREQ=DAILY;BYHOUR=14;BYMINUTE=11`。这指示作业每天运行，在 14:00（24 小时制）过 11 分运行（换句话说，每天下午 2:11 运行）。`CREATE_JOB` 的 `REPEAT_INTERVAL` 参数能够实现复杂的日历频率。例如，它支持各种年度、月度、每周、每日、每小时、每分钟和每秒的计划。*Oracle Database PL/SQL Packages and Types Reference* 指南仅针对 `REPEAT_INTERVAL` 参数就包含数页的语法细节。

`JOB_CLASS` 参数指定将作业分配到哪个作业类。通常，你会创建一个作业类，以提供一种方法，让作业通过被分配到特定类来自动继承属性。例如，你可能希望某个特定类中的所有作业具有相同的日志记录级别或以相同的方式清除日志文件。如果你尚未创建任何作业类，可以使用默认的作业类。此示例使用了默认作业类。

在此示例中，`AUTO_DROP` 参数被设置为 `FALSE`。这指示 Oracle 调度器不要在作业运行后自动删除它。我希望这个作业持久存在并按照计划的频率运行。

## 查看作业详情

要查看作业配置的详细信息，请查询 `DBA_SCHEDULER_JOBS` 视图。清单 21–3 选择了 `RMAN_BACKUP` 作业的信息：

**清单 21–3.** `RMAN_BACKUP` 作业的信息

```sql
SELECT
  job_name
  ,last_start_date
  ,last_run_duration
  ,next_run_date
  ,repeat_interval
FROM dba_scheduler_jobs
WHERE job_name='RMAN_BACKUP';
```

以下是输出的一个片段（输出已换行以适应页面）：

```
JOB_NAME     LAST_START_DATE               LAST_RUN_DURATION   NEXT_RUN_DATE           REPEAT_INTERVAL
------------ ----------------------------- ------------------- ----------------------- -------------------
RMAN_BACKUP  21–OCT-10 02.12.59.257151 PM  +000000000 00:00:   22-OCT-10 02.11.0.3000  FREQ=DAILY;BYHOUR
             -06:00                        41.933585 -06:00    PM -06:00               =14;BYMINUTE=11
```

每次作业运行时，作业执行的记录都会被记录在数据字典中。要检查每次作业执行的状态，请查询 `DBA_SCHEDULER_JOB_LOG`。应该对作业的每次运行都有一条记录，如下所示：

```sql
SELECT
  job_name
  ,log_date
  ,operation
  ,status
FROM dba_scheduler_job_log
WHERE job_name='RMAN_BACKUP';
```

以下是一些示例输出：

```
JOB_NAME     LOG_DATE                              OPERATION STATUS
------------ ------------------------------------- --------- ----------
RMAN_BACKUP  21–OCT-10 02.13.41.196695 PM -06:00  RUN       SUCCEEDED
```

## 修改作业日志记录历史

默认情况下，Oracle 调度器保留 30 天的日志历史记录。你可以通过 `SET_SCHEDULER_ATTRIBUTE` 过程来修改默认的保留期限。例如，这会将默认天数更改为 15 天：


# 第 21 章 ■ 自动化作业

SQL> exec `dbms_scheduler.set_scheduler_attribute('log_history',15);`
要完全清除日志历史记录的内容，请使用 `PURGE_LOG` 过程：
SQL> exec `dbms_scheduler.purge_log();`

## 修改作业

你可以通过 `SET_ATTRIBUTE` 过程修改作业的各种属性。此示例将 `RMAN_BACKUP` 作业修改为每周一运行：

```
BEGIN
  dbms_scheduler.set_attribute(
    name=>'RMAN_BACKUP'
    ,attribute=>'repeat_interval'
    ,value=>'freq=weekly; byday=mon');
END;
/
```

对于这个特定示例，你可以通过从 `DBA_SCHEDULER_JOBS` 中选择 `REPEAT_INTERVAL` 列来验证更改。以下是运行上一节“查看作业详情”中的查询所产生的输出：

```
JOB_NAME    LAST_START_DATE          LAST_RUN_DURATION       NEXT_RUN_DATE           REPEAT_INTERVAL
----------- ----------------------- ----------------------- ----------------------- -----------------
RMAN_BACKUP 21-OCT-10 02.12.59.257 +000000000 00:00:41.93 25-OCT-10 12.00.00.000 freq=weekly; byday=mon
            06:00 PM - 06:00        3385 PM - 06:00         AM - 06:00
```

根据之前的输出，作业将在下一个周一运行，并且由于修改作业时没有指定 `BYHOUR` 和 `BYMINUTE` 选项，它现在被安排在默认时间 12:00 a.m. 运行。

## 停止作业

如果你有一个运行时间异常长的作业，你可能希望中止它。使用 `STOP_JOB` 过程来停止当前正在运行的作业。此示例在 `RMAN_BACKUP` 作业运行时停止它：

SQL> exec `dbms_scheduler.stop_job(job_name=>'RMAN_BACKUP');`

`DBA_SCHEDULER_JOB_LOG` 的 `STATUS` 列将为使用 `STOP_JOB` 过程停止的作业显示 `STOPPED`。

## 禁用作业

你可能希望暂时禁用一个作业，因为它运行不正确。你需要确保在排查问题期间该作业不会运行。使用 `DISABLE` 过程来禁用作业：
SQL> exec `dbms_scheduler.disable('RMAN_BACKUP');`

如果作业当前正在运行，请考虑先停止作业或使用 `DISABLE` 过程的 `FORCE` 选项：
SQL> exec `dbms_scheduler.disable(name=>'RMAN_BACKUP',force=>true);`

## 启用作业

你可以通过 `DBMS_SCHEDULER` 包的 `ENABLE` 过程来启用先前禁用的作业。此示例重新启用 `RMAN_BACKUP` 作业：
SQL> exec `dbms_scheduler.enable(name=>'RMAN_BACKUP');`

> **提示** 你可以通过选择 `DBA_SCHEDULER_JOBS` 的 `ENABLED` 列来检查作业是否已被禁用或启用。

## 复制作业

如果你有一个想要克隆的现有作业，你可以使用 `COPY_JOB` 过程来实现。此过程接受两个参数：旧作业名称和新作业名称。以下是一个复制作业的示例，其中 `RMAN_BACKUP` 是先前创建的作业，`RMAN_NEW_BACK` 是将要创建的新作业：

```
begin
  dbms_scheduler.copy_job('RMAN_BACKUP','RMAN_NEW_BACK');
end;
/
```

复制的作业将被创建但未启用。你必须首先启用该作业（参见本章前面的部分），它才会运行。

## 手动作业运行

你可以在其常规计划之外手动作业运行。你可能希望这样做来测试作业以确保其正常工作。使用 `RUN_JOB` 过程手动启动作业。此示例手动运行先前创建的 `RMAN_BACKUP` 作业：

```
BEGIN
  DBMS_SCHEDULER.RUN_JOB(
    JOB_NAME => 'RMAN_BACKUP',
    USE_CURRENT_SESSION => FALSE);
END;
/
```

`USE_CURRENT_SESSION` 参数指示 Oracle 调度程序是否以当前用户身份运行作业。`FALSE` 值表示以作业常规计划运行时所使用的用户身份运行该作业（异步方式）。

## 删除作业

如果你不再需要某个作业，应将其从调度程序中删除。使用 `DROP_JOB` 过程永久删除作业。此示例移除 `RMAN_BACKUP` 作业：

```
BEGIN
  dbms_scheduler.drop_job(job_name=>'RMAN_BACKUP');
END;
/
```

此代码将删除作业，并从 `DBA_SCHEDULER_JOBS` 视图中移除有关该已删除作业的任何信息。

## Oracle 调度程序 vs cron

数据库管理员经常争论是应该使用 Oracle 调度程序还是 Linux/Unix 的 `cron` 实用程序来安排和自动化任务。Oracle 调度程序相对于 `cron` 的优势包括：

- 可以使作业的执行依赖于另一作业的完成
- 强大的资源平衡和灵活的计划功能
- 可以基于数据库事件运行作业
- Oracle 调度程序 `DBMS_SCHEDULER` PL/SQL 包语法无论操作系统如何都保持一致
- 可以使用数据字典运行状态报告
- 如果在集群环境中工作，无需担心为集群中的每个节点同步多个 `cron` 表
- 可以通过企业管理器进行维护和监控

Oracle 调度程序通过 `DBMS_SCHEDULER` PL/SQL 包实现。使用此实用程序创建和维护作业相当容易（如本章前面所示）。虽然 Oracle 调度程序有许多优点，但许多 DBA 更喜欢使用 `cron` 等调度实用程序。使用 `cron` 的优势包括：

- 易于使用；简单、久经考验且可靠；创建和/或修改作业只需几秒钟
- 几乎在所有 Linux/Unix 机器上普遍可用；在很大程度上，无论 Linux/Unix 平台如何，其运行方式几乎相同（是的，存在细微差异）
- 数据库无关；独立于数据库运行，无论数据库供应商或数据库版本如何，其工作方式都相同
- 无论数据库是否可用，都能工作

前面的列表并非详尽无遗，但应该让你了解每种调度工具的用途。我倾向于使用 `cron`，但如果你需要更复杂的调度程序，那么可以考虑使用 Oracle 调度程序。本章接下来的几节将提供有关如何通过 `cron` 实现和调度自动化作业的信息。

## 通过 cron 自动化作业

`cron` 程序是 Linux/Unix 环境中普遍存在的作业调度实用程序。该工具的名称源自 *chronos*（希腊语中的时间）。`cron`（调度程序的行话）工具允许你安排脚本或命令在指定时间运行，并按指定的频率重复。

### cron 工作原理

当你的 Linux 服务器启动时，会自动启动一个 `cron` 后台进程来管理系统上的所有 `cron` 作业。此 `cron` 后台进程也称为 `cron` 守护进程。该进程在系统启动时通过 `/etc/init.d/crond` 脚本启动。你可以使用 `ps` 命令检查 `cron` 守护进程是否正在运行：

```
$ ps -ef | grep crond | grep -v grep
root      3049     1  0 Aug02 ?        00:00:00 crond
```

你也可以使用 `service` 命令检查 `cron` 守护进程是否正在运行：

```
$ /sbin/service crond status
crond (pid  3049) is running...
```

root 用户在执行系统 `cron` 作业时使用几个文件和目录。`/etc/crontab` 文件包含要运行的系统 `cron` 作业的命令。以下是 `/etc/crontab` 文件内容的典型列表：

```
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
HOME=/
```


# **run-parts**

```bash
01 * * * * root run-parts /etc/cron.hourly
02 4 * * * root run-parts /etc/cron.daily
22 4 * * 0 root run-parts /etc/cron.weekly
42 4 1 * * root run-parts /etc/cron.monthly
```

## **第 21 章 ■ 自动化任务**

这个 `/etc/crontab` 文件使用 `run-parts` 工具来运行位于以下目录中的脚本：`/etc/cron.hourly`、`/etc/cron.daily`、`/etc/cron.weekly` 和 `/etc/cron.monthly`。如果有一个需要运行的系统工具，其执行周期并非每小时、每天、每周或每月一次，那么可以将其放置在 `/etc/cron.d` 目录中。

每个用户都可以创建自己的 `crontab`（也称为 cron 表）文件。该文件包含了你希望在特定时间和间隔运行的程序列表。此文件通常位于 `/var/spool/cron` 目录中。对于每一个创建了 cron 表的用户，在 `/var/spool/cron` 目录下都会有一个以该用户名命名的文件。作为 `root` 用户，你可以列出该目录中的文件：

```bash
