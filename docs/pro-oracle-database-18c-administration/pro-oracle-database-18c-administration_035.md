# 管理物化视图日志和组

## 显示已注册到日志的物化视图

下一个查询显示所有已创建并关联到物化视图日志的物化视图的信息。请在主站点运行此查询：

```sql
SQL> select a.log_table, a.log_owner
,b.master mast_tab
,c.owner  mv_owner
,c.name   mview_name
,c.mview_site, c.mview_id
from dba_mview_logs a
,dba_base_table_mviews b
,dba_registered_mviews c
where b.mview_id = c.mview_id
and   b.owner    = a.log_owner
and   b.master   = a.master
order by a.log_table;
```

以下是一些示例输出：

```
LOG_TABLE             LOG_OWNE   MAST_TAB        MV_OWN   MVIEW_NAME
-------------------   --------   -------------   ------   ---------------- MVIEW_S   MVIEW_ID-------   -------
MLOG$_CMP_GRP_ASSOC   INV_MGMT   CMP_GRP_ASSOC   REP_MV   CMP_GRP_ASSOC_MV DWREP     651
MLOG$_CMP_GRP_ASSOC   INV_MGMT   CMP_GRP_ASSOC   TSTDEV   CMP_GRP_ASSOC_MV ENGDEV    541
```

## 从日志中清除未注册的物化视图

当你删除一个远程物化视图时，它应该从主数据库中注销。然而，这并不总是发生。远程数据库可能会被清除（例如，一个短期的开发数据库），而物化视图没有机会通过 `DROP MATERIALIZED VIEW` 语句自行注销。在这种情况下，物化视图日志并不知道一个依赖的物化视图已不再可用，因此会无限期地保留记录。

要从包含物化视图日志的数据库中清除不需要的物化视图信息，请执行 `DBMS_MVIEW` 的 `PURGE_MVIEW_FROM_LOG` 过程。此示例传入了要清除的物化视图的 ID：

```sql
SQL> exec dbms_mview.purge_mview_from_log(541);
```

此语句应更新数据字典，并从内部表 `SLOG$` 和 `DBA_REGISTERED_MVIEWS` 中删除信息。如果要清除的物化视图是关联到该物化视图日志表的最旧物化视图，那么相关的旧记录也会从物化视图日志中删除。

## 手动注销物化视图

如果某个远程物化视图已不可用但仍注册在物化视图日志表中，你可以在主站点手动将其注销。使用 `DBMS_MVIEW` 包的 `UNREGISTER_MVIEW` 过程来注销远程物化视图。为此，你需要知道远程物化视图的所有者、名称和站点（可从上一节查询的输出中获得）：

```sql
SQL> exec dbms_mview.unregister_mview('TSTDEV','CMP_GRP_ASSOC_MV','ENGDEV');
```

如果成功，上述操作将从 `DBA_REGISTERED_MVIEWS` 中移除一条记录。

## 在组中管理物化视图

物化视图组是一个有用的功能，它使你能够在一致的事务时间点刷新一组物化视图。如果你刷新的物化视图所基于的主表之间存在父子关系，那么你很可能应该使用一个刷新组。这种方法可以保证你刷新的物化视图集合中不会有任何孤立的子记录。以下部分描述了如何创建和维护物化视图刷新组。

> **注意**
> 你使用 `DBMS_REFRESH` 包来完成管理物化视图刷新组的大部分任务。该包在《Oracle 高级复制管理 API 参考指南》中有完整记录，可从 Oracle 网站的技术网络区域下载（ [`http://otn.oracle.com`](http://otn.oracle.com) ）。

### 创建物化视图组

你使用 `DBMS_REFRESH` 包的 `MAKE` 过程来创建物化视图组。创建物化视图组时，必须指定一个名称、组中物化视图的逗号分隔列表、下次刷新的日期以及用于计算下次刷新时间的时间间隔。以下是一个包含两个物化视图的组的示例：

```sql
SQL> begin
dbms_refresh.make(
name      => 'SALES_GROUP',
list      => 'SALES_MV, SALES_DAILY_MV',
next_date => sysdate-100,
interval  => 'sysdate+1'
);
end;
/
```

当你创建物化视图组时，Oracle 会自动创建一个数据库作业来管理该组的刷新。你可以通过查询 `DBA/ALL/USER_REFRESH` 来查看物化视图组的详细信息：

```sql
SQL> select rname, job, next_date, interval from user_refresh;
```

以下是一些示例输出：

```
RNAME            JOB   NEXT_DATE            INTERVAL
---------------  ----  -------------------  --------------
SALES_GROUP        3   20-OCT-12            sysdate+1
```

### 更改物化视图刷新组

你可以更改刷新组的特性，例如刷新日期或时间间隔。如果你依赖数据库作业作为刷新机制，那么你偶尔可能需要调整刷新特性。使用 `DBMS_REFRESH` 包的 `CHANGE` 函数来实现这一点。以下示例更改了 `INTERVAL` 计算：

```sql
SQL> exec dbms_refresh.change(name=>'SALES_GROUP',interval=>'SYSDATE+2');
```

再次强调，仅当你使用内部数据库作业来启动物化组刷新时，才需要更改刷新时间间隔。你可以使用以下查询验证刷新组的时间间隔和作业信息的详细信息：

```sql
SQL> select a.job, a.broken, b.rowner, b.rname, b.interval
from dba_jobs    a
,dba_refresh b
where a.job = b.job
order by a.job;
```

此示例的输出如下：

```
JOB  B ROWNER     RNAME           INTERVAL
---- - ---------- --------------- ---------------
3 N MV_MAINT   SALES_GROUP     SYSDATE+2
```

### 刷新物化视图组

创建组后，你可以使用 `DBMS_REFRESH` 包的 `REFRESH` 函数手动刷新它。此示例刷新你之前创建的组：

```sql
SQL> exec dbms_refresh.refresh('SALES_GROUP');
```

如果你检查 `USER_MVIEWS` 的 `LAST_REFRESH_DATE` 列，你会发现组中的所有物化视图都有相同的刷新时间。这是预期的行为，因为组中的物化视图都是在一致的事务时间点刷新的。

### DBMS_MVIEW 与 DBMS_REFRESH

你可能已经注意到，可以使用 `DBMS_MVIEW` 包来刷新一组物化视图。例如，你可以使用 `DBMS_MVIEW` 按如下方式刷新列表中的物化视图集合：

```sql
SQL> exec dbms_mview.refresh(list=>'SALES_MV,SALES_DAILY_MV');
```

此方法在一个事务中刷新列表中的每个物化视图。它等同于使用物化视图组。然而，当你使用 `DBMS_MVIEW` 时，你可以选择将 `ATOMIC_REFRESH` 参数设置为 `TRUE`（默认值）或 `FALSE`。例如，此处将 `ATOMIC_REFRESH` 参数设置为 `FALSE`：

```sql
SQL> exec dbms_mview.refresh(list=>'SALES_MV,SALES_DAILY_MV',atomic_refresh=>false);
```

此设置指示 `DBMS_MVIEW` 将列表中的每个物化视图作为一个单独的事务刷新。上面一行代码等同于以下两行：

```sql
SQL> exec dbms_mview.refresh(list=>'SALES_MV', atomic_refresh=>false);
SQL> exec dbms_mview.refresh(list=>'SALES_DAILY_MV', atomic_refresh=>false);
```

将此与 `DBMS_REFRESH` 的行为进行比较，后者是你用于设置和维护物化视图组的包。`DBMS_REFRESH` 包总是将一组物化视图作为一个一致的事务进行刷新。如果你总是需要将一组物化视图作为一个事务一致的组来刷新，请使用 `DBMS_REFRESH`。如果你在是否将物化视图列表作为一个一致事务刷新方面需要一些灵活性，请使用 `DBMS_MVIEW`。

### 确定组中的物化视图

当你调查物化视图刷新组的问题时，一个很好的起点是显示该组包含哪些物化视图。查询数据字典视图 `DBA_RGROUP` 和 `DBA_RCHILD`，查看刷新组中的物化视图：

```sql
SQL> select a.owner
,a.name mv_group
,b.name mv_name
from dba_rgroup a
,dba_rchild b
where a.refgroup = b.refgroup
and   a.owner    = b.owner
order by a.owner, a.name, b.name;
```

输出片段如下：

```
OWNER      MV_GROUP             MV_NAME
---------- -------------------- --------------------
MV_MAINT   SALES_GROUP          SALES_DAILY_MV
MV_MAINT   SALES_GROUP          SALES_MV
```

在 `DBA_RGROUP` 视图中，`NAME` 列代表刷新组的名称。`DBA_RCHILD` 视图包含刷新组中每个物化视图的名称。

### 将物化视图添加到刷新组


