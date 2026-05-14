# 第 15 章 ■ 物化视图

## 源 Oracle OS 变量

`. /var/opt/oracle/oraset $ORACLE_SID`

`date`

`sqlplus -s <<EOF`

`rep_mv2/foobar`

`WHENEVER SQLERROR EXIT FAILURE`

`exec dbms_mview.refresh('CWP_COUNTRY_INFO','C');`

`EOF`

```bash
if [ $? -ne 0 ]; then
    echo "not okay"
    $MAILX -s "Problem with MV refresh on $HOSTNAME $jobname" $MAIL_LIST <<EOF
    $HOSTNAME $jobname MVs not okay.
    EOF
else
    echo "okay"
    $MAILX -s "MV refresh OK on $HOSTNAME $jobname" $MAIL_LIST <<EOF
    $HOSTNAME $jobname MVs okay.
    EOF
fi
```

`date`

`exit 0`

对于这个特定的物化视图刷新作业，以下是调用它的对应 cron 条目：

```
25 16 * * * /orahome/oracle/bin/mvref_cwp.bsh DWREP 1>/orahome/oracle/bin/log/mvref_cwp.log 2>&1
```

此作业每天下午 4:25 运行。有关使用 cron 调度作业的详细信息，请参见 **第 21 章**。

### 创建具有刷新间隔的物化视图

在最初创建物化视图时，你可以选择指定 `START WITH` 和 `NEXT` 子句，以指示 Oracle 设置一个内部数据库作业（通过 `DBMS_JOB` 包），来周期性地启动物化视图的刷新。如果省略 `START WITH` 和 `NEXT`，则不会设置作业，你必须使用其他技术（例如像 cron 这样的调度工具）。

我几乎从不将 `START WITH` 和 `NEXT` 指定为刷新机制。我强烈倾向于使用另一个调度工具，例如 cron。使用 cron 时，可以轻松创建一个详细记录作业运行情况以及是否存在任何问题的日志文件。此外，使用 cron 时，可以轻松地将日志文件通过电子邮件发送到分发列表，以便支持 DBA 了解任何问题。

无论如何，理解 `START WITH` 和 `NEXT` 的工作原理很重要，因为迟早你会发现自己身处一个 DBA 或开发人员倾向于使用 `DBMS_JOB` 包进行刷新的环境。当你在对刷新问题进行故障排除时，必须理解这种刷新机制的工作原理。

`START WITH` 参数指定你希望物化视图首次刷新发生的日期。`NEXT` 参数指定一个日期表达式，Oracle 用它来计算刷新之间的时间间隔。

例如，这个物化视图在将来一分钟（`sysdate+1/1440`）进行初始刷新，随后每天刷新一次（`sysdate+1`）：

```sql
create materialized view inv_mv
refresh
    start with sysdate+1/1440
    next sysdate+1
as
select
    inv_id
    ,inv_desc
from inv;
```

你可以通过查询 `USER_JOBS` 来查看计划作业的详细信息：

```sql
select
    job
    ,schema_user
    ,to_char(last_date,'dd-mon-yyyy hh24:mi:ss') last_date
    ,to_char(next_date,'dd-mon-yyyy hh24:mi:ss') next_date
    ,interval
    ,broken
from user_jobs;
```

以下是一些示例输出：

```
JOB SCHEMA_USE LAST_DATE             NEXT_DATE            INTERVAL       B
------ ---------- -------------------- -------------------- --------------- -
50    INV        21-oct-2010 15:30:28                     sysdate+1       N
```

你也可以在 `USER_REFRESH` 视图中查看作业信息：

```sql
select
    rowner
    ,rname
    ,job
    ,to_char(next_date,'dd-mon-yyyy hh24:mi:ss')
    ,interval
    ,broken
from user_refresh;
```

以下是一些示例输出：

```
ROWNER      RNAME      JOB TO_CHAR(NEXT_DATE,'DD-MON- INTERVAL       B
----------- ---------- --- -------------------------- --------------- -
INV         INV_MV     50  22-oct-2010 15:30:29       sysdate+1       N
```

当你删除一个物化视图时，关联的作业也会被移除。如果你想手动移除一个作业，请使用 `DBMS_JOB` 的 `REMOVE` 过程。此示例移除作业编号 32（该编号是从之前的查询中识别出来的）：

```sql
SQL> exec dbms_job.remove(32);
```

`注意` 你不能在 `ON COMMIT` 刷新的物化视图中结合使用 `START WITH` 或 `NEXT`。

### 高效执行完全刷新

当物化视图执行完全刷新时，默认行为是使用 `DELETE` 语句从物化视图表中删除所有记录。删除完成后，从主表中选择记录并插入到物化视图表中。删除和插入作为单个事务完成；这意味着在完全刷新过程中从物化视图进行选择的任何用户看到的数据，都是 `DELETE` 语句执行前的状态。任何在 `INSERT` 提交后立即访问物化视图的用户，看到的都是数据的最新视图。

在某些场景下，你可能希望修改此行为。如果要刷新大量数据，`DELETE` 语句可能需要很长时间。你可以选择通过 `ATOMIC_REFRESH` 参数指示 Oracle 尽可能高效地执行数据移除。当此参数设置为 `FALSE` 时，它允许 Oracle 在执行完全刷新时使用 `TRUNCATE` 语句代替 `DELETE`：

```sql
SQL> exec dbms_mview.refresh('INV_MV',method=>'C',atomic_refresh=>false);
```

对于大型数据集，`TRUNCATE` 比 `DELETE` 运行更快，因为 `TRUNCATE` 没有生成重做日志的开销。使用 `TRUNCATE` 语句的缺点是，在刷新进行期间，从物化视图查询的用户可能会看到零行数据。

### 处理 ORA-12034 错误

当你尝试对物化视图执行快速刷新时，有时可能会收到 `ORA-12034` 错误。例如：

```sql
SQL> exec dbms_mview.refresh('PRODUCTLINEITEM','F');
```

该语句随后会抛出此错误消息：

```
BEGIN dbms_mview.refresh('PRODUCTLINEITEM','F'); END;
*
ERROR at line 1:
ORA-12034: materialized view log on "CDS_PROD_ES2_LIVE"."PRODUCTLINEITEM"
younger than last refresh
```

要解决此错误，请尝试完全刷新物化视图：

```sql
SQL> exec dbms_mview.refresh('PRODUCTLINEITEM','C');
```

完全刷新完成后，你应该能够执行快速刷新而不会收到错误：

```sql
SQL> exec dbms_mview.refresh('PRODUCTLINEITEM','F');
```

当 Oracle 确定物化视图日志是在关联物化视图的上次刷新之后创建的时，就会抛出 `ORA-12034` 错误。换句话说，物化视图日志比物化视图的上次刷新更“年轻”。有几种可能的原因：
*   物化视图日志被删除并重新创建。
*   物化视图日志被清除。
*   主表被重组。
*   主表被截断。
*   上次刷新失败。

在这种情况下，Oracle 知道在物化视图的上次刷新时间和物化视图日志创建时间之间可能产生了事务。在此场景中，你必须首先执行一次完全刷新，然后才能开始使用快速刷新机制。

## 监控物化视图刷新

本节包含一些非常实用的示例，说明如何监控物化视图刷新作业。示例包括如何查看上次刷新时间、确定作业是否正在执行、确定刷新作业的进度，以及检查物化视图在过去一天内是否未刷新。像这样的脚本对于故障排除和诊断刷新问题非常宝贵。

### 查看物化视图的上次刷新时间

在对物化视图进行故障排除时，通常首先要检查 `DBA/ALL/USER_MVIEWS` 中的 `LAST_REFRESH_DATE`。查看此信息可以确定物化视图是否按计划刷新。以物化视图所有者的身份运行此查询，以显示上次刷新日期：

```sql
select
    mview_name
    ,to_char(last_refresh_date,'dd-mon-yy hh24:mi:ss')
    ,refresh_mode
    ,refresh_method
from user_mviews
order by 2;
```

以下是一些示例输出：

```
MVIEW_NAME                  TO_CHAR(LAST_REFRESH_DAT REFRES REFRESH_
---------------------------- ------------------------ ------ --------
GEM_COMPANY_MV              29-jul-10 06:18:58       DEMAND COMPLETE
TOP_REG_DAILY               29-jul-10 06:57:33       DEMAND FORCE
```

`DBA/ALL/USER_MVIEWS` 的 `LAST_REFRESH_DATE` 列显示物化视图上次成功完成刷新的日期和时间。如果物化视图从未成功刷新过，则 `LAST_REFRESH_DATE` 为 `NULL`。

### 确定刷新是否正在进行中

如果你需要知道哪些物化视图正在运行，请使用此查询：

```sql
select
    sid
    ,serial#
    ,currmvowner
    ,currmvname
from v$mvrefresh;
```

以下是一些示例输出：

```
SID    SERIAL#  CURRMVOWNER          CURRMVNAME
------ -------- -------------------- ------------------------------
1034   47872    REP_MV               USERS_MV
```

### 监控实时刷新进度

如果你处理大型物化视图，下一个查询将向你显示刷新操作的实时进度。在进行故障排除时，此查询非常有用。以内部 `SYS` 表的权限用户身份运行以下命令：

```sql
column "MVIEW BEING REFRESHED" format a25
column inserts format 9999999
column updates format 9999999
column deletes format 9999999
--
select
    currmvowner_knstmvr || '.' || currmvname_knstmvr "MVIEW BEING REFRESHED",
    decode(reftype_knstmvr, 1, 'FAST', 2, 'COMPLETE', 'UNKNOWN') reftype,
    decode(groupstate_knstmvr, 1, 'SETUP', 2, 'INSTANTIATE',
           3, 'WRAPUP', 'UNKNOWN' ) STATE,
    total_inserts_knstmvr inserts,
    total_updates_knstmvr updates,
    total_deletes_knstmvr deletes
from x$knstmvr x
where type_knst = 6
and exists (select 1
            from v$session s
            where s.sid=x.sid_knst
            and s.serial#=x.serial_knst);
```

当物化视图首次开始刷新时，你会看到此输出：

```
MVIEW BEING REFRESHED   REFTYPE  STATE      INSERTS   UPDATES   DELETES
------------------------- -------- ----------- --------- --------- ---------
REP_MV.USERS_MV         UNKNOWN  SETUP             0         0         0
```

几秒钟后，物化视图进入 `INSTANTIATE` 状态：

```
REP_MV.USERS_MV         FAST     INSTANTIATE       0         0         0
```

随着物化视图的刷新，`INSERTS`、`UPDATES` 和 `DELETES` 列会相应更新：

```
REP_MV.USERS_MV         FAST     INSTANTIATE     860       274         0
```

当物化视图几乎完成刷新时，它会进入 `WRAPUP` 状态：

```
REP_MV.USERS_MV         FAST     WRAPUP         5284      1518         0
```

物化视图完成刷新后，查询将返回无行：

```
no rows selected
```

正如你可以想象的那样，此查询对于故障排除和诊断物化视图刷新问题非常有用。

### 检查物化视图是否在时间周期内刷新

在处理物化视图时，如果能有某种自动化方式来确定是否发生刷新，那会很方便。使用以下 shell 脚本来检测哪些物化视图在过去一天内未刷新，然后如果检测到任何此类情况，就发送电子邮件：

```bash
#!/bin/bash
```

详见第 2 章，了解如何使用类似`oraset`的工具来加载操作系统变量。


