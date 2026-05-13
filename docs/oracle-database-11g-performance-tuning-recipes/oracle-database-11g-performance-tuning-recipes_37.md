# Oracle 数据库统计收集优化技术

## 增量统计收集工作原理

增量统计收集功能默认是禁用的。您可以通过设置 `INCREMENTAL` 首选项来启用该功能。在我们的示例中，展示了如何在表级别设置 `INCREMENTAL` 首选项，但您也可以在模式（schema）或数据库级别进行设置。

在处理分区表时，优化器会同时使用全局统计信息（整个表的统计信息）和单个分区的统计信息来选择最优的执行计划。默认情况下，当分区的数据发生变化后，数据库会使用一种两阶段扫描技术来维护准确的表统计信息。在这种两阶段技术下，数据库将执行以下操作：

*   在第一阶段，扫描整个表以收集全局统计信息。
*   在第二阶段，扫描发生变更的分区，以收集分区统计信息。

例如，当您作为夜间批处理作业的一部分向分区加载（或从中删除）数据时，数据库将扫描该分区以收集分区级别的统计信息。此外，它还会扫描整个表以收集表级别的全局统计信息。数据库不仅扫描发生变更的分区，还会扫描表中的所有其他分区。可以看出，每次分区数据变更都进行全表扫描是一个开销很大的过程，尤其是在处理超大型表时。

一旦您开启了增量统计收集功能，Oracle 将采用一种高效得多的技术来维护分区表的统计信息。当分区的数据发生变化时，数据库仅收集该分区的统计信息，并在不扫描任何其他分区的情况下推导出全局表统计信息。数据库是如何在不扫描全表的情况下维护全局统计信息的呢？Oracle 可以从分区级统计信息推导出一些全局统计信息——例如，通过将每个分区中的行数相加来推导出行总数。对于推导不同值的数量（NDV），Oracle 使用一种称为“概要”（synopsis）的结构，它类似于列中 NDV 的样本。Oracle 通过合并所有分区概要来推导全局 NDV。总之，当您实施增量统计收集时，Oracle 会跳过默认的全表扫描来收集表统计信息，而是执行以下操作：

1.  收集您加载数据的分区的统计信息，并为该分区创建概要。
2.  通过合并所有分区级别的概要来创建一个全局概要。
3.  基于分区级别的统计信息和全局概要计算全局统计信息。

增量统计收集效率极高，尤其是在处理大型分区表时，特别是当您频繁向一个或多个空分区加载数据时（这在许多数据仓库中是常见情况），您必须考虑使用此功能。

## 大表的并发统计收集

### 问题

您希望利用多处理器环境来最小化收集统计信息所需的时间。

### 解决方案

在 Oracle Database 11g Release 2 (11.2.0.2) 中，您可以指定 *并发统计信息收集模式*，以便同时收集多个表以及一个表内的多个分区（和子分区）的统计信息。通过这样做，您可以充分利用多处理器环境，更快地完成统计信息收集过程。

默认情况下，并发统计收集是禁用的。您可以通过执行 `SET_GLOBAL_PREFS` 过程来启用它。请按照以下步骤启用并发统计收集。

1.  将 `job_queue_processes` 参数至少设置为 4。
    ```
    SQL> alter system set job_queue_processes=4;
    ```
    如果您不打算使用并行执行来收集统计信息（请参见下一节），但希望充分利用系统资源，则必须将 `job_queue_processes` 参数设置为服务器 CPU 核心数的两倍。

2.  启用并发统计收集。
    ```
    SQL> begin
         dbms_stats.set_global_prefs('CONCURRENT','TRUE');
       end;
       /
    ```
请确保执行此命令的用户具有 `CREATE JOB`、`MANAGE SCHEDULER` 和 `MANAGE ANY QUEUE` 权限。


## 工作原理

并发统计信息收集的目标是缩短大表和分区的统计信息收集时间。启用并发统计信息收集后，Oracle 会利用数据库的作业调度器和高级队列功能创建多个并发统计信息收集作业。参数 `job_queue_processes` 决定了并发统计信息收集作业的最大数量。在 RAC 环境中，您必须在每个节点上设置此参数。并发统计信息收集的工作方式在某种程度上取决于统计信息收集的级别（是否在表级别），如下所述。

如果您执行 `DBMS_STATS.GATHER_TABLE_STATS` 过程来收集分区表的统计信息，Oracle 将为表中的每个分区（和子分区）创建一个单独的统计信息收集作业。调度器会根据系统容量来决定并发运行的作业数量以及需要排队的作业数量。

![images](img/square.jpg) 注意：`job_queue_processes` 参数的值决定了并发统计信息收集作业的最大数量。

如果您执行 `GATHER_DATABASE_STATS`、`GATHER_SCHEMA_STATS` 或 `GATHER_DICTIONARY_STATS` 过程，Oracle 会为每个表以及分区表中的每个分区创建一个单独的统计信息收集作业。为了防止潜在的死锁问题，Oracle 不会同时处理多个分区表。Oracle 会为每个分区表创建一个协调器作业，以管理该分区的统计信息收集作业。每个作业要么是一个表分区级作业的协调器（如果该表是分区表），要么是一个实际的统计信息收集作业。如果您有多个分区表，数据库会将除一个以外的所有分区表放入队列；在完成对一个分区表的统计信息收集后，它会将另一个分区表的作业出队并启动。这种队列行为不适用于非分区表。

### 使用并行执行策略

如果您正在为极大的表收集统计信息，可以启用各个统计信息收集作业的并行执行。为此，您必须禁用 `parallel_adaptive_multi_user` 初始化参数，如下所示：

```
SQL> alter system set parallel_adaptive_multi_user=false;
```

虽然不是必需的，Oracle 还建议您通过激活资源管理器、创建一个临时资源计划，并为消费者组 `OTHER_GROUPS` 启用队列来启用并行语句排队。以下是一个简单的示例，展示了如何创建临时资源计划并启用资源管理器：

```
begin
  dbms_resource_manager.create_pending_area();
  dbms_resource_manager.create_plan('parallel_test', 'parallel_test');
  dbms_resource_manager.create_plan_directive(
        'parallel_test',
        'OTHER_GROUPS',
        'OTHER_GROUPS directive for parallel test',
        parallel_target_percentage => 90);
  dbms_resource_manager.submit_pending_area();
end;
/
ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'parallel_test' SID='*';
```

### 监控并发统计信息收集作业

使用 `DBA_SCHEDULER_JOBS` 视图来监控并发统计信息收集作业。您可以通过执行以下语句查看数据库中的所有并发统计信息收集作业：

```
SQL> select job_name,state,comments
     from dba_scheduler_jobs
     where job_class like 'CONC%';
```

如果您想将输出限制为当前正在执行的作业，请在上述查询中添加 "`and state='RUNNING'`" 这一行。类似地，您可以添加 "`and state='SCHEDULED'`" 来仅查看正在等待运行的已计划作业。您可以通过执行以下查询来检查当前正在执行的统计信息收集作业的已用时间：

```
SQL> select job_name,elapsed_time
     from dba_scheduler_running_jobs
     where job_name like 'ST$%';
```

