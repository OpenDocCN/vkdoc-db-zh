# 数据库自动化与 Oracle Scheduler

自动化在以下场景中发挥作用：当这些脚本（即代码块）成为不需要数据库管理员（DBA）干预即可运行的过程的一部分时。任务由按计划运行的进程或工作流定期执行，这些进程或工作流无需直接交互即可运行必要的脚本。例行维护作业通常是最容易自动化的工作，并且会产生一个成功完成或可能失败的检查结果以供验证。有时会变得复杂的是那些需要更改或理解故障原因以进行纠正，且无需人工交互即可继续的过程。Oracle 18c 数据库已发展成为支持数据库自我修复、调整和打补丁的系统。Oracle 18c 在 Oracle Cloud 中提供了自治数据库。这些任务按预期发生，以便 DBA 可以专注于其他领域，并在数据集成、开发和数据策略方面提供咨询。

自动化例行任务使 DBA 能够更有效和高效。自动化环境本质上比手动管理系统运行更流畅、更高效。从脚本自动运行的 DBA 作业每次执行相同的命令集，因此不易出现人为错误和失误。使用本地部署的 Oracle 18c 也允许将许多任务以及应用程序和其他维护作业所需的少量脚本自动化。本章将介绍两种调度实用程序：

*   Oracle Scheduler

*   Linux/Unix `cron` 实用程序

当然，也可以选择使用替代`cron`实用程序的企业解决方案。这允许对作业进行集中管理。由于这里的选项很多，本章将不涵盖使用企业工具的内容，但在`cron`和 Oracle 实用程序中调度的脚本和作业也可以利用企业工具。本章首先详细介绍 Oracle Scheduler 实用程序的基本方面。只要安装了 Oracle 数据库，就可以使用这个调度程序。Oracle Scheduler 可用于在各种配置中调度作业。

本章还介绍了如何使用 Linux/Unix `cron`调度工具。
在 Linux/Unix 环境中，DBA 经常使用`cron`调度实用程序来自动运行作业。`cron`实用程序无处不在，易于实现和使用。如果你是一名 Oracle DBA，你必须熟悉`cron`，因为迟早你会发现自己身处一个严重依赖此工具来自动化数据库作业的环境中。其他环境可能不允许访问`cron`调度，因此，本章中自动化的作业策略很重要，并且可以基于策略（可能是安全策略）在另一个工具中实现。

本章最后几节将向你展示如何实施许多现实世界的 DBA 作业，例如自动启动/停止数据库、监控和操作系统文件维护。你应该能够扩展这些脚本以满足你环境的自动化要求。

**注意**
Enterprise Manager Grid/Cloud Control 也可用于调度和管理自动化作业。如果你在使用 Enterprise Manager 的环境中工作，那么使用此工具来自动化你的环境是合适的。

## 使用 Oracle Scheduler 自动化作业

Oracle Scheduler 是一个提供自动化作业调度方法的工具。Oracle Scheduler 通过`DBMS_SCHEDULER`内部 PL/SQL 包实现。Oracle Scheduler 为调度作业提供了一套复杂的功能。`DBMS_JOB`是用于在数据库上运行作业的较旧的软件包。它们都针对数据库调度和执行作业，但调度器提供了额外的功能，例如日志记录、基于权限的模型、更详细的调度以及调度信息的存储。有了这些功能，使用`DBMS_SCHEDULER`比使用`DBMS_JOBS`更有利。本章接下来的几节将介绍使用 Oracle Scheduler 自动化具有简单需求的作业的基础知识。

**提示**
`DBMS_SCHEDULER`包中目前有超过 70 个可用的过程和函数。有关完整详细信息，请参阅《Oracle Database PL/SQL Packages and Types Reference Guide》，该指南可从 Oracle 网站的技术网络区域（ [`http://otn.oracle.com`](http://otn.oracle.com) ）下载。

### 创建和调度作业

本节中的示例展示了如何使用`DBMS_SCHEDULER`来每天运行一个操作系统 shell 脚本。首先，创建一个包含 RMAN 备份命令的 shell 脚本。在此示例中，shell 脚本名为`rmanback.bsh`，位于`/orahome/oracle/bin`目录下。该 shell 脚本还假定存在一个`/orahome/oracle/bin/log`目录可用。以下是 shell 脚本内容：

```bash
#!/bin/bash
# source oracle OS variables; see chapter 2 for an example of oraset script
. /etc/oraset o18c
rman target / <<EOF
spool log to '/orahome/oracle/bin/log/rmanback.log'
backup database;
spool log off;
EOF
exit 0
```

接下来，使用`DBMS_SCHEDULER`包的`CREATE_JOB`过程来创建一个每日作业。然后，以`SYS`或具有`CREATE`和`ALTER JOB`权限的`SYSBACKUP`用户身份连接，并执行以下命令：

```sql
SQL> BEGIN
DBMS_SCHEDULER.CREATE_JOB(
job_name => 'RMAN_BACKUP',
job_type => 'EXECUTABLE',
job_action => '/orahome/oracle/bin/rmanback.bsh',
repeat_interval => 'FREQ=DAILY;BYHOUR=9;BYMINUTE=35',
start_date => to_date('17-01-2018','dd-mm-yyyy'),
job_class => '"DEFAULT_JOB_CLASS"',
auto_drop => FALSE,
comments => 'RMAN backup job',
enabled => TRUE);
END;
/
```

在之前的代码中，`JOB_TYPE`参数可以是以下类型之一：`STORED_PROCEDURE`、`PLSQL_BLOCK`、`EXTERNAL_SCRIPT`、`SQL_SCRIPT`或`EXECUTABLE`。在此示例中，执行的是一个外部 shell 脚本，因此作业类型为`EXTERNAL`。

`REPEAT_INTERVAL`参数设置为`FREQ=DAILY;BYHOUR=9;BYMINUTE=35`。这指示作业在每天上午 9:35 运行。`CREATE_JOB`的`REPEAT_INTERVAL`参数能够实现复杂的日历频率。例如，它支持各种每年、每月、每周、每天、每小时、每分钟和每秒钟的调度计划。《Oracle Database PL/SQL Packages and Types Reference Guide》中仅`REPEAT_INTERVAL`参数的语法细节就占了好几页。

`JOB_CLASS`参数指定将作业分配到哪个作业类。通常，你会创建一个作业类并将作业分配给该类，然后该作业将继承该特定类的属性。例如，你可能希望特定类中的所有作业具有相同的日志记录级别或以相同的方式清除日志文件。如果你尚未创建任何作业类，则可以使用默认的作业类。前面的示例使用了默认的作业类。


