# lock.sql

此脚本显示对表持有锁从而阻止其他会话完成工作的会话。该脚本显示有关阻塞会话和等待会话的详细信息。您应将此脚本放在诸如`HOME/scripts`之类的目录中。以下是`lock.sql`的内容：

```
SET LINES 83 PAGES 30
COL blkg_user    FORM a10
COL blkg_machine FORM a10
COL blkg_sid     FORM 99999999
COL wait_user    FORM a10
COL wait_machine FORM a10
COL wait_sid     FORM 9999999
COL obj_own      FORM a10
COL obj_name     FORM a10
--
SELECT
s1.username    blkg_user
,s1.machine     blkg_machine
,s1.sid         blkg_sid
,s1.serial#     blkg_serialnum
,s1.sid || ',' || s1.serial# kill_string
,s2.username    wait_user
,s2.machine     wait_machine
,s2.sid         wait_sid
,s2.serial#     wait_serialnum
,lo.object_id   blkd_obj_id
,do.owner       obj_own
,do.object_name obj_name
FROM v$lock l1
,v$session s1
,v$lock l2
,v$session s2
,v$locked_object lo
,dba_objects do
WHERE s1.sid = l1.sid
AND   s2.sid = l2.sid
AND   l1.id1 = l2.id1
AND   s1.sid = lo.session_id
AND   lo.object_id = do.object_id
AND   l1.block = 1
AND   l2.request > 0;
```

`lock.sql`脚本对于确定哪个会话持有对象上的锁以及显示被阻塞的会话非常有用。您可以从 SQL*Plus 运行此脚本，如下所示：

```
SQL> @lock.sql
```

以下是输出的部分列表（已截断以便在一页内显示）：

```
BLKG_USER  BLKG_MACHI  BLKG_SID BLKG_SERIALNUM
---------- ---------- --------- --------------
KILL_STRING

WAIT_USER  WAIT_MACHI WAIT_SID WAIT_SERIALNUM BLKD_OBJ_ID OBJ_OWN    OBJ_NAME
---------- ---------- -------- -------------- ----------- ---------- ----------
MV_MAINT   speed            24             11
24,11
MV_MAINT   speed            87              7       19095 MV_MAINT   INV
```

在 Oracle Database 18c 可插拔数据库环境中从根容器运行`lock.sql`时，您需要将`DBA_OBJECTS`更改为`CDB_OBJECTS`，脚本才能在整个数据库中正确报告锁。您还应考虑在查询中添加`NAME`和`CON_ID`，以便查看锁发生的容器。以下是修改后的查询片段（您需要用您想要报告的列替换“...”）：

```
SELECT
u.name
,s1.username    blkg_user
...
,do.object_name obj_name
FROM v$lock l1
,v$session s1
,v$lock l2
,v$session s2
,v$locked_object lo
,cdb_objects do
,v$containers u
WHERE s1.sid = l1.sid
AND   s2.sid = l2.sid
AND   l1.id1 = l2.id1
AND   s1.sid = lo.session_id
AND   lo.object_id = do.object_id
AND   l1.block = 1
AND   l2.request > 0
AND   do.con_id = u.con_id;
```

