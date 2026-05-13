# 启用 ILM 实现存储分层

您已经了解到，Oracle 允许您根据所谓的 ILM 策略，自动将“冷”数据从第 1 层存储移动到更低层存储。这些策略可以在创建表时分配给表，或者事后加装。策略可以在行或段级别定义。

首先，需要一个数据移动的目标位置。在本例中，它是一个名为 `ILM_COMPRESS` 的表空间。源表空间名称是 `MARTIN_BIGFILE`。目前，如对 `DBA_TAB_PARTITIONS` 的查询所示，表 `ADODEMO` 的每个段都位于该表空间上：

```
TABLESPACE_NAME                PARTITION_NAME
------------------------------ --------------
MARTIN_BIGFILE                 P_MANUAL
MARTIN_BIGFILE                 SYS_P796
MARTIN_BIGFILE                 SYS_P797
MARTIN_BIGFILE                 SYS_P798
MARTIN_BIGFILE                 SYS_P799
MARTIN_BIGFILE                 SYS_P800
MARTIN_BIGFILE                 SYS_P801
MARTIN_BIGFILE                 SYS_P802
```

现在，您可以向最旧的分区添加 ILM 策略。

```
SQL> alter table ADODEMO modify partition SYS_P797
  2  ilm add policy tier to ILM_COMPRESS;
Table altered.
SQL> select policy_name,object_name,subobject_name,enabled from user_ilmobjects;
POLICY_NAME                    OBJECT_NAME                    SUBOBJECT_NAME                 ENA
------------------------------ ------------------------------ ------------------------------ ---
P1                             ADODEMO                        SYS_P797                       YES
1 row selected.
```

您刚刚看到了与信息生命周期管理相关的一个字典视图的实际应用：`USER_ILMOBJECTS`。此视图列出了所有附加了 ILM 策略的对象。在示例表发生任何操作之前，需要满足某些条件。您可以在视图 `DBA_ILMPARAMETERS` 中找到当前定义的 ILM 参数：

```
SQL> select * from DBA_ILMPARAMETERS;
NAME                   VALUE
-------------------- ----------
ENABLED                      1
RETENTION TIME              30
JOB LIMIT                    2
EXECUTION MODE               2
EXECUTION INTERVAL          15
TBS PERCENT USED            85
TBS PERCENT FREE            25
POLICY TIME                  0
8 rows selected.
```

关于存储分层，最初只有一个参数值得关注。`TBS PERCENT USED` 是一个阈值，指示何时应实施“数据移动”策略。翻译成英文就是，当段所在的表空间使用率超过 85% 时，可能会发生数据移动。继续上面的示例，该表空间使用率已超过此阈值，这应该会触发数据移动。

```
Tablespace                    Size (MB)  Free (MB)     % Free     % Used
------------------------------ ---------- ---------- ---------- ----------
MARTIN_BIGFILE                   209920  6403.4375          3         97
```

ILM 策略的评估作为夜间维护作业窗口的一部分执行。对于大多数系统，除非计划程序窗口已被更改，否则这应该是晚上 10 点。如果您比较着急，可以通过调用 `DBMS_ILM` 包中的 `EXECUTE_ILM` 过程来加快开发环境上的处理速度。其参数之一是一个名为 `task_id` 的输出变量。它返回与执行 ILM 策略关联的任务的自动创建名称。

您可以在 SQL*Plus 会话中使用命令 `print task_id` 来获取任务的实际值。通过调用 `DBMS_ILM.EXECUTE_ILM`，您在当前模式中执行 ILM 策略。策略评估的结果在另一个新的字典视图 `USER_ILMEVALUATIONDETAILS` 中：

```
SQL> select selected_for_execution, job_name, policy_name
  2  from user_ilmevaluationdetails
  3   where task_id = :task_id
  4  /
SELECTED_FOR_EXECUTION         JOB_NAME                       POLICY_NAME
------------------------------ ------------------------------ ---------------
SELECTED FOR EXECUTION         ILMJOB408                      P1
```

在这种情况下，该策略已被选中执行。要了解数据移动结果的更多信息，您可以检查 `USER_ILMRESULTS`：

```
SQL> select job_state,start_time,completion_time from USER_ILMRESULTS
  2  where task_id = :task_id and job_name = 'ILMJOB408';
JOB_STATE              START_TIME                   COMPLETION_TIME
---------------------- ------------------------------ ------------------------------
COMPLETED SUCCESSFULLY 08-OCT-14 10.00.42.603455 PM 08-OCT-14 10.00.55.250004 PM
```

作业的执行也记录在计划程序视图中，例如 `DBA_SCHEDULER_JOB_RUN_DETAILS`。在底层，Oracle 执行一个 PL/SQL 块，如果您有兴趣查看，该块记录在 `SYS.ILM_RESULTS$` 中。数据库执行的实际代码存储在 `PAYLOAD` 列中。字典视图已经表明作业执行成功，但您可以自己查看效果。回到 `USER_TAB_PARTITIONS`，您会看到该分区的表空间名称已更改：

```
SQL> select partition_name,tablespace_name from user_tab_partitions
  2  where table_name = 'ADODEMO' and partition_name = 'SYS_P797';
PARTITION_NAME                 TABLESPACE_NAME
------------------------------ ---------------
SYS_P797                       ILM_COMPRESS
1 row selected.
```

作为另一个预期的副作用，您在原始表空间中也获得了一些额外的可用空间。存储分层选项在 Exadata 中非常有用，特别是当您使用 ZFS 存储设备或 Pillar Axiom/FS1 阵列作为数据移动目标时。请记住，这些存储都不允许您在其上进行 *Smart Scan*。另一方面，移动到此类表空间的数据可能本来就是冷数据，访问频率不高。

## 启用 ILM 实现压缩

前文已经提及的另一选项，允许你在数据变冷时对其进行压缩。热图对实现此功能至关重要。没有它，Oracle 就无法跟踪数据的访问和操作情况。而且，因为 Oracle 内核开发者体恤 DBA，他们甚至添加了一些统计信息来向性能架构师提示热图的使用情况：

```sql
SQL> select name from v$statname where lower(name) like '%heat%';

NAME
----------------------------------------------------------------
Heatmap SegLevel - Write
Heatmap SegLevel - Full Table Scan
Heatmap SegLevel - IndexLookup
Heatmap SegLevel - TableLookup
Heatmap SegLevel - Flush
Heatmap SegLevel - Segments flushed
Heatmap BlkLevel Tracked
Heatmap BlkLevel Not Tracked - Memory
Heatmap BlkLevel Not Updated - Repeat
Heatmap BlkLevel Flushed
Heatmap BlkLevel Flushed to SYSAUX
Heatmap BlkLevel Flushed to BF
Heatmap BlkLevel Ranges Flushed
Heatmap BlkLevel Ranges Skipped
Heatmap BlkLevel Flush Task Create
Heatmap Blklevel Flush Task Count

16 rows selected.
```

当你使用合著者 Tanel Poder 的 snapper 工具来捕获一段时间内的性能计数器变化时，你实际上可能开始看到其中一些统计信息！为了模拟对数据表`ADODEMO`的访问，编写了一个小程序：

```sql
create or replace procedure ADODEMOPROC as
  v_id number;
  v_date date;
begin
  v_date :=  to_date('01-AUG-2014') + dbms_random.value(1,10);
  select count(id) into v_id from martin.adodemo where date_created = trunc(v_date);
end;
/
```

其意图是在多个调度器作业中使用该查询来模拟用户活动。这个活动是让下一个示例按预期工作所必需的。由于上述查询最初执行的是全表扫描，因此在`DATE_CREATED`列上添加了一个索引。调度器作业通过直接调用`DBMS_SCHEDULER`来创建：

```sql
SQL> begin
  2    for i in 1..5 loop
  3      dbms_scheduler.create_job(
  4        job_name => 'ADODEMOJOB_' || i,
  5        job_type => 'STORED_PROCEDURE',
  6        job_action => 'ADODEMOPROC',
  6        start_date => systimestamp,
  8        repeat_interval => 'freq=secondly;interval=10',
  9        enabled => true);
 10    end loop;
 11  end;
 12  /

PL/SQL procedure successfully completed.
```

ILM 策略是演示中缺失的部分。如本例所示，该策略可以分配给整个表，并将被子分区继承：

```sql
SQL> alter table adodemo
  2  ilm add policy column store compress for query low
  3  segment after 30 days of no access;

Table altered.

SQL> select policy_name, subobject_name, object_type, inherited_from, enabled
  2  from user_ilmobjects where object_name = 'ADODEMO';

POLICY_NAM SUBOBJECT_NAME                 OBJECT_TYPE        INHERITED_FROM       ENA
---------- ------------------------------ ------------------ -------------------- ---
P1         SYS_P797                       TABLE PARTITION    POLICY NOT INHERITED NO
P3         P_MANUAL                       TABLE PARTITION    TABLE                YES
P3         SYS_P796                       TABLE PARTITION    TABLE                NO
P3         SYS_P797                       TABLE PARTITION    TABLE                YES
P3         SYS_P798                       TABLE PARTITION    TABLE                YES
P3         SYS_P799                       TABLE PARTITION    TABLE                YES
P3         SYS_P800                       TABLE PARTITION    TABLE                YES
P3         SYS_P801                       TABLE PARTITION    TABLE                YES
P3         SYS_P802                       TABLE PARTITION    TABLE                YES
P3                                      TABLE              POLICY NOT INHERITED YES

10 rows selected.
```

策略`P1`是在上一节中实现并执行的数据移动策略。策略名称不是用户可定义的。换句话说，是由 Oracle 分配的。策略`P3`是刚刚通过上述命令添加的新策略。如前段所述，你可以看到分区继承了分配给表的策略。策略的作用域是数据段，对于堆表来说就是分区。与存储分层策略一样，新策略会在维护窗口打开时进行评估。

评估的结果再次被记录在`USER_ILMEVALUATIONDETAILS`中。你的段可能会被选中执行，如下所示：

```sql
SQL> select policy_name POLICY, object_name "TABLE", subobject_name "PARTITION",
  2   selected_for_execution, job_name
  3  from user_ilmevaluationdetails;

POLICY     TABLE      PARTITION       SELECTED_FOR_EXECUTION                     JOB_NAME
---------- ---------- --------------- ------------------------------------------ ----------
P3         ADODEMO    SYS_P796        SELECTED FOR EXECUTION                     ILMJOB428
P1         ADODEMO    SYS_P797        PRECONDITION NOT SATISFIED
P3         ADODEMO    SYS_P797        PRECONDITION NOT SATISFIED
P3         ADODEMO    SYS_P798        PRECONDITION NOT SATISFIED
...
```

你可以在`USER_ILMRESULTS`中核实作业的完成情况。`SYS.ILM_RESULTS$`中的有效负载显示了另一个压缩分区的 PL/SQL 块。作业成功完成后，你应该能在数据字典中看到结果：

```sql
SQL> select partition_name, table_name, compression, compress_for
  2    from user_tab_partitions
  3   where table_name = 'ADODEMO'
  4    and partition_name = 'SYS_P796';

PARTITION_NAME       TABLE_NAME                     COMPRESS COMPRESS_FOR
-------------------- ------------------------------ -------- ------------------------------
SYS_P796             ADODEMO                        ENABLED  QUERY LOW
```

这是关于 ADO 选项功能的一个简短介绍。还有许多其他选项可用，例如使用自定义函数进行分层。与许多出色的 Oracle 特性一样，使用自动数据优化需要你拥有额外的付费许可（高级压缩）。

## 总结

HCC（混合列压缩）在 Oracle 11g 第 2 版中引入，提供了卓越的压缩能力，远超之前版本的任何功能。这主要得益于采用了行业标准的压缩算法，以及将压缩单元的大小从单个数据库块（通常为 8K）增加到更大的 32K 或 64K 单元。尽管 12c 版本增强了行级锁机制，但由于锁定问题以及更新的行会被移动到压缩率低得多的格式（OLTP 压缩格式）中，该特性仅适用于不再被修改的数据。因此，HCC 应只用于不再修改（或仅偶尔修改）的数据。由于压缩可以在分区级别定义，常见的做法是看到表中包含压缩和未压缩分区的混合体。这种技术在许多情况下可以替代需要将数据移动到备用存储介质然后从数据库中清除的 ILM 方法。借助 Oracle 12c，数据生命周期管理过程可以实现自动化。

