# Oracle 物化视图管理：作业、刷新与监控

您可以通过查询`USER_JOBS`视图来查看作业的详细信息。

```
JOB  SCHEMA_USE  LAST_DATE            NEXT_DATE            INTERVAL      B
---- ----------  -------------------- -------------------- ------------ -
1  MV_MAINT    28-1 月-2013 14:55:33 29-1 月-2013 14:55:33  sysdate+1    N
```

您也可以在`USER_REFRESH`视图中查看作业信息：

```
SQL> select rowner, rname, job
,to_char(next_date,'dd-mon-yyyy hh24:mi:ss')
,interval, broken
from user_refresh;
```

示例输出如下：

```
ROWNER     RNAME       JOB TO_CHAR(NEXT_DATE,'DD-MON-YYY INTERVAL     B
---------- ---------- ---- ----------------------------- ------------ -
MV_MAINT   SALES_MV      1 29-1 月-2013 14:55:33          sysdate+1    N
```

当您删除一个物化视图时，与之关联的作业也会被移除。如果您想手动移除一个作业，请使用`DBMS_JOB`包中的`REMOVE`过程。以下示例移除了作业号 1，该作业号可从前面的查询中获得：

```
SQL> exec dbms_job.remove(1);
```

> 注意：对于设置为`ON COMMIT`刷新的物化视图，您不能同时使用`START WITH`或`NEXT`子句。

## 高效执行完全刷新

当物化视图执行完全刷新时，默认行为是使用`DELETE`语句从物化视图表中移除所有记录。删除完成后，从主表中选择记录并插入到物化视图表中。删除和插入作为一个事务完成；这意味着在完全刷新过程中，任何从物化视图查询的用户看到的都是执行`DELETE`语句之前的数据。而在`INSERT`提交后立即访问物化视图的用户则能看到数据的最新视图。

在某些场景下，您可能希望修改此行为。如果刷新的数据量很大，`DELETE`语句可能耗时很长。您可以选择通过`ATOMIC_REFRESH`参数指示 Oracle 尽可能高效地执行数据移除操作。当此参数设置为`FALSE`时，它允许 Oracle 在执行完全刷新时使用`TRUNCATE`语句代替`DELETE`：

```
SQL> exec dbms_mview.refresh('SALES_MV',method=>'C',atomic_refresh=>false);
```

对于大数据集，`TRUNCATE`比`DELETE`工作得更快，因为`TRUNCATE`没有生成重做日志的开销。使用`TRUNCATE`语句的缺点是，在刷新过程中，查询物化视图的用户可能看到零行数据。

## 处理 ORA-12034 错误

当您尝试对物化视图执行快速刷新时，有时可能会遇到`ORA-12034`错误；例如：

```
SQL> exec dbms_mview.refresh('SALES_MV','F');
```

该语句随后会抛出以下错误消息：

```
ORA-12034: materialized view log on "MV_MAINT"."SALES" younger than last refresh
```

要解决此错误，请尝试完全刷新物化视图：

```
SQL> exec dbms_mview.refresh('SALES_MV','C');
```

完全刷新完成后，您应该能够执行快速刷新而不会收到错误：

```
SQL> exec dbms_mview.refresh('SALES_MV','F');
```

当 Oracle 确定物化视图日志是在关联物化视图中上次刷新之后创建时，就会抛出`ORA-12034`错误。换句话说，物化视图日志比物化视图的上次刷新时间更“年轻”。有几种可能的原因：

*   物化视图日志被删除并重新创建。
*   物化视图日志被清除。
*   主表被重组。
*   主表被截断。
*   上一次刷新失败。

在这种情况下，事务可能是在物化视图的上次刷新时间与物化视图日志创建时间之间创建的。在此场景中，您必须先执行一次完全刷新，然后才能开始使用快速刷新机制。

## 监控物化视图刷新

以下部分包含一些非常方便的示例，演示如何监控物化视图刷新作业。示例包括如何查看上次刷新时间、确定作业是否正在执行、确定刷新作业的进度以及检查物化视图在过去一天内是否未刷新。这样的脚本对于排查和诊断刷新问题非常宝贵。

### 查看物化视图的上次刷新时间

当您排查物化视图的问题时，通常首先检查的是`DBA/ALL/USER_MVIEWS`视图中的`LAST_REFRESH_DATE`列。查看此信息可以让您了解物化视图是否按计划刷新。以物化视图所有者身份运行此查询，以显示上次刷新日期：

```
SQL> select mview_name
,to_char(last_refresh_date,'dd-mon-yy hh24:mi:ss')
,refresh_mode, refresh_method
from user_mviews;
```

`DBA/ALL/USER_MVIEWS`视图的`LAST_REFRESH_DATE`列显示物化视图上次成功完成刷新的日期和时间。如果物化视图从未成功刷新过，则`LAST_REFRESH_DATE`为`NULL`。

### 确定刷新是否正在进行

如果您需要知道哪些物化视图正在运行，请使用此查询：

```
SQL> select sid, serial#, currmvowner, currmvname from v$mvrefresh;
```

示例输出如下：

```
SID   SERIAL#  CURRMVOWNER              CURRMVNAME
---------- ---------  -----------------------  -------------------
108      3037  MV_MAINT                 SALES_MV
```

### 监控实时刷新进度

如果您处理大型物化视图，下一个查询可以显示刷新操作的实时进度。在排查问题时，此查询可能非常有用。作为拥有内部 SYS 表权限的用户运行以下脚本：

```
SQL> column "MVIEW BEING REFRESHED" format a25
SQL> column inserts format 9999999
SQL> column updates format 9999999
SQL> column deletes format 9999999
--
SQL> select
currmvowner_knstmvr || '.' || currmvname_knstmvr "MVIEW BEING REFRESHED",
decode(reftype_knstmvr, 1, 'FAST', 2, 'COMPLETE', 'UNKNOWN') reftype,
decode(groupstate_knstmvr, 1, 'SETUP', 2, 'INSTANTIATE',
3, 'WRAPUP', 'UNKNOWN') STATE,
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

当物化视图刚开始刷新时，您会看到此输出：

```
MVIEW BEING REFRESHED     REFTYPE  STATE        INSERTS  UPDATES  DELETES
------------------------- -------- ----------- -------- -------- --------
MV_MAINT.SALES_MV         UNKNOWN  SETUP              0        0        0
```

几秒钟后，物化视图达到`INSTANTIATE`状态：

```
MV_MAINT.SALES_MV         FAST     INSTANTIATE        0        0        0
```

随着物化视图刷新，`INSERTS`、`UPDATES`和`DELETES`列会相应更新：

```
MV_MAINT.SALES_MV         FAST     INSTANTIATE      860      274        0
```

当物化视图几乎完成刷新时，它达到`WRAPUP`状态：

```
MV_MAINT.SALES_MV         FAST     WRAPUP          5284     1518        0
```

物化视图完成刷新后，查询将返回零行：

```
no rows selected
```

正如您所想，此查询对于排查和诊断物化视图刷新问题非常有用。

### 检查物化视图是否在指定时间段内刷新

处理物化视图时，拥有一个自动化的方法来确定刷新是否正在发生是很好的。使用以下 Shell 脚本检测哪些物化视图在过去一天内未刷新，然后如果检测到任何未刷新的物化视图，则发送电子邮件：

```
#!/bin/bash
# Source oracle OS variables, see Chapter 2 for details
. /etc/oraset $1
#
crit_var=$(sqlplus -s  1;
EOF)
#
if [ $crit_var -ne 0 ]; then
echo $crit_var
echo "mv_ref refresh problem with $1" | mailx -s "mv_ref problem" \
dkuhn@gmail.com
else
echo $crit_var
echo "MVs ok"
fi
#
exit 0
```

