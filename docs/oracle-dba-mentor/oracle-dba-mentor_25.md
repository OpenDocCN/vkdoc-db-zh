# Oracle 数据字典与动态性能视图指南

## 常用 DBA 视图

本节将介绍一些您可能会用到的常见数据字典视图及其简要说明。毫无疑问，您使用的视图将远超此列表范围。

*   `DBA_BLOCKERS`：显示哪个会话正在阻塞谁。
*   `DBA_CONSTRAINTS`：显示应用于数据库中所有表的约束。
*   `DBA_DATA_FILES`：显示构成数据库的文件。
*   `DBA_DEPENDENCIES`：显示数据库中依赖于其他对象的对象。
*   `DBA_FREE_SPACE`：显示每个表空间中的可用空间。
*   `DBA_IND_COLUMNS`：显示数据库中每个索引的列。
*   `DBA_INDEXES`：显示数据库中的所有索引。
*   `DBA_OBJECTS`：显示数据库中的所有对象。
*   `DBA_ROLES`：显示数据库中的所有角色。
*   `DBA_ROLE_PRIVS`：显示数据库中授予他人的角色。
*   `DBA_SCHEDULER_JOBS`：显示数据库中的所有计划作业。
*   `DBA_SEGMENTS`：显示所有作为段的对象，即表和索引。
*   `DBA_SEQUENCES`：显示数据库中的所有序列对象。
*   `DBA_SOURCE`：显示构成视图、过程、包、触发器等的源代码。
*   `DBA_SYS_PRIVS`：显示授予他人的所有系统权限。
*   `DBA_SYNONYMS`：显示数据库中的所有同义词。
*   `DBA_TAB_COLUMNS`：显示数据库中所有表的所有列。
*   `DBA_TAB_COMMENTS`：显示所有表的注释。
*   `DBA_TAB_PRIVS`：显示授予他人的所有表的权限。
*   `DBA_TABLES`：显示数据库中的所有表。
*   `DBA_TABLESPACES`：显示数据库中的所有表空间。
*   `DBA_TEMP_FILES`：显示数据库中临时表空间的所有临时文件。
*   `DBA_TRIGGERS`：显示数据库中所有表上的所有触发器。
*   `DBA_USERS`：显示数据库中的所有用户。
*   `DBA_VIEWS`：显示数据库中的所有视图。

在您的职业生涯中，可能已经查询过此列表中的许多视图。如果对某些视图不熟悉，请花些时间了解它们包含的信息。

## 有用的视图

有许多有用的数据字典静态视图不以 `DBA_`、`ALL_`、`USER_` 或 `CDB_` 开头，您需要熟悉它们。

*   `DATABASE_PROPERTIES`：显示永久数据库属性，例如时区和语言。
*   `DICT`：显示所有数据字典视图。
*   `RECYCLEBIN`：显示回收站中的表，这些表可能可以从意外删除中恢复。
*   `SESSION_PRIVS`：显示当前会话可用的权限。
*   `SESSION_ROLES`：显示为当前会话启用的角色。

## 动态性能视图

对于静态数据字典视图，需要某种事务来更改其输出。而对于动态性能视图，则不需要此类事务。考虑清单 14-9 中的示例，它显示了有多少会话连接到 Oracle 实例。

```sql
SQL> select count(*) from v$session;
COUNT(*)
---
        36

SQL> select count(*) from v$session;
COUNT(*)
---
        37
```

清单 14-9 会话计数

第一次执行查询时，有 36 个会话连接到 Oracle 实例。第二次执行时，有 37 个会话连接。由于这是一个受控环境，我知道用户会话都没有生成任何事务，但数值仍然发生了变化。这就是为什么这类视图被称为 *动态* 的。即使数据库中没有任何用户事务，这些值也可能并将会随时间变化。

静态视图和动态性能视图之间还有另一个重要区别，这与实例关闭有关。如果数据库中有 1,498 个表，然后将其关闭，那么在启动时，必须仍然有 1,498 个表。而对于动态性能视图，当实例关闭时，所有信息都会丢失。这些视图返回的任何计数器在实例重新启动时都会从头开始。

大多数动态性能视图以 `V$` 开头。在清单 14-9 的示例中，我们看到对 `V$SESSION` 的查询。`V$` 视图通常供数据库管理员用来了解其数据库的运行状况。如果您正在寻找特定的 `V$` 视图，可以查询 `DICT` 或查阅《*Database Reference*》。

### 常用 V$ 视图

以下列表显示了许多常用的 `V$` 视图。与 `DBA_` 视图一样，您还可以使用许多其他视图。

*   `V$ACCESS`：显示哪个会话在库缓存中锁定了对象。
*   `V$CACHE`：显示缓冲区缓存的内容。
*   `V$CONTROLFILE`：显示控制文件的状态。
*   `V$DATABASE`：显示记录在控制文件中的数据库信息。
*   `V$FILESTAT`：显示数据库文件的 I/O 统计信息。
*   `V$INSTANCE`：显示有关当前运行实例的信息。
*   `V$LOG`：显示有关联机重做日志的信息。
*   `V$LOGFILE`：显示联机重做日志成员。
*   `V$MYSTAT`：显示有关当前会话的统计信息。
*   `V$PARAMETER`：显示当前会话的参数会话。
*   `V$SESSION`：显示当前连接到实例的会话。
*   `V$SESSION_EVENT`：显示每个会话自启动以来一直在等待的事件。
*   `V$SESSION_WAIT`：显示每个会话当前正在等待的详细信息。
*   `V$SGAINFO`：显示 SGA 每个组件的大小。
*   `V$SPPARAMETER`：显示存储在 SPFILE 中的参数值。
*   `V$SQL`：显示有关存储在共享池中的 SQL 语句的详细信息。
*   `V$VERSION`：显示 Oracle 版本。

当然，在您的职业生涯中，您还可以使用并将使用更多 `V$` 视图。您可能已经使用过上面列表中的一些。如果对某些视图不熟悉，请花些时间查询每个视图并进行进一步研究。

### GV$ 视图

现在介绍对上一节讨论的 `V$` 视图的一个扩展。在我们的测试环境中，启动 Oracle 时，数据库只有一个实例。Oracle 数据库的一个额外付费选项是真正应用集群（RAC），它允许您在多个服务器上为同一数据库启动多个实例。Oracle RAC 用于高可用性和高可扩展性。

`V$` 视图仅显示当前实例。如果您使用 Oracle RAC，则配置中会有其他实例。`GV$` 视图在 Oracle RAC 中非常有用，因为它们允许您查询数据库所有实例的信息。我一直认为它们是全局 `V$` 视图。如果您在 Oracle RAC 中查询 `V$SESSION`，它会显示连接到您所连接实例的会话。如果您查询 `GV$SESSION`，它会显示所有实例上的所有会话。

如果您不使用 Oracle RAC，`GV$` 视图仍然有效，但它们只包含一个实例（唯一运行的实例）的信息。`GV$` 视图还包含一个额外的列 `INST_ID`，用于表示该行信息来自哪个实例。在非 RAC 的单实例环境中，`INST_ID` 列始终为 1。

## 实用示例

本章讨论了 Oracle 数据字典，包括静态视图和动态性能视图。数据字典是你可以深入了解你的数据库的地方。在本节中，我将提供一些实用的示例。你可以使用本节中的许多脚本，甚至可以根据你的特定需求进行定制。请跟随本节学习如何利用数据字典的示例。

每位 Oracle DBA 在其职业生涯中的某个时刻，都曾接手过别人建立的数据库。处于这种情况下的 DBA 通常需要找出关于该数据库的一些基本信息以便了解更多。当我第一次遇到一个已存在的数据库时，我经常会对数据字典运行一系列查询，以便更熟悉这个系统。我首先要做的事情之一就是确保我知道实例名称、主机名和数据库的版本，如清单 14-10 所示。

```
SQL> select instance_name,host_name,version,startup_time
2  from v$instance;
INSTANCE_NAME    HOST_NAME                      VERSION           STARTUP_T
---------------- ------------------------------ ----------------- ---------
orcl             dbamentor.localdomain          12.2.0.1.0        20-AUG-18
清单 14-10
V$INSTANCE 查询
```

接下来，我使用类似于清单 14-11 的查询来确定构成数据库的表空间。

```
SQL> select tablespace_name from dba_tablespaces
2  order by tablespace_name;
TABLESPACE_NAME
--------------------------------
APPS_TS
SYSAUX
SYSTEM
TEMP
UNDOTBS1
USERS
清单 14-11
数据库的表空间
```

我自然会对这些表空间的数据文件位于磁盘存储的哪个位置感到好奇。除了数据文件，我还想知道临时文件、控制文件和在线重做日志的位置。所有这些信息都存储在不同的数据字典视图中，但我可以执行一个 `UNION` 操作将它们整合在一起，如清单 14-12 所示。

```
SQL> select 'DATA' as type,file_name,bytes from dba_data_files
2  union all
3  select 'TEMP',file_name,bytes from dba_temp_files
4  union all
5  select 'REDO',lf.member,l.bytes
6  from v$logfile lf join v$log l on lf.group#=l.group#
7  union all
8  select 'CTL',value,NULL from v$parameter where name='control_files';
TYPE FILE_NAME                                               BYTES
---- -------------------------------------------------- ----------
DATA /u01/app/oracle/oradata/orcl/system01.dbf           734003200
DATA /u01/app/oracle/oradata/orcl/sysaux01.dbf          1184890880
DATA /u01/app/oracle/oradata/orcl/undotbs01.dbf          492830720
DATA /u01/app/oracle/oradata/orcl/users01.dbf              5242880
DATA /u01/app/oracle/oradata/orcl/apps_ts01.dbf         5368709120
TEMP /u01/app/oracle/oradata/orcl/temp01.dbf              20971520
REDO /u01/app/oracle/oradata/orcl/redo01.log             209715200
REDO /u01/app/oracle/oradata/orcl/redo02.log             209715200
REDO /u01/app/oracle/oradata/orcl/redo03.log             209715200
CTL  /u01/app/oracle/oradata/orcl/control01.ctl
CTL  /u01/app/oracle/oradata/orcl/control02.ctl
清单 14-12
数据库文件
```

我现在对这个数据库的存储配置有了更好的了解。我想确保表空间的空间没有不足，因此我可以运行一个类似于清单 14-13 的查询。

```
SQL> select f.tablespace_name, to_char(f.bytes,'99,999,999,999,999') as bytes_alloc,
2  nvl(to_char(se.bytes,'99,999,999,999,999'),LPAD('Empty',19)) as bytes_used,
3  to_char(nvl(trunc((se.bytes/f.bytes)*100,2),0),'990.00') as pct_used
4  from
5  ( select df.tablespace_name, sum(bytes) as bytes
6    from dba_data_files df group by df.tablespace_name) f,
7  ( select s.tablespace_name, sum(bytes) as bytes
8    from dba_segments s group by s.tablespace_name ) se
9  where f.tablespace_name=se.tablespace_name (+)
10  order by f.tablespace_name;
TABLESPACE_NAME        BYTES_ALLOC         BYTES_USED          PCT_USE
---------------------- ------------------- ------------------- -------
APPS_TS              5,368,709,120              Empty             0.00
SYSAUX               1,184,890,880      1,122,566,144            94.74
SYSTEM                 734,003,200        730,726,400            99.55
UNDOTBS1               492,830,720         13,959,168             2.83
USERS                    5,242,880              Empty             0.00
清单 14-13
表空间空闲空间
```

我可以看到有两个表空间是空的，而 `SYSTEM` 和 `SYSAUX` 表空间的使用率超过了 90%。

接下来，我通常想了解数据库中的用户。Oracle 包含许多为支持内部数据库操作而创建的用户。我通常只关心非 Oracle 维护的用户，如清单 14-14 所示。

```
SQL> select username,account_status from dba_users
2  where oracle_maintained='N' order by username;
USERNAME                       ACCOUNT_STATUS
------------------------------ --------------------------------
PEASLAND                       OPEN
TEST_USER                      OPEN
HR                             OPEN
清单 14-14
非 Oracle 用户
```

在讨论用户账户时，安全应该是每个人最关心的问题。接下来，我将查询数据字典以确定正在使用的安全配置文件，如清单 14-15 所示。

```
SQL> select username,profile from dba_users
2  where oracle_maintained='N' order by username;
USERNAME                       PROFILE
------------------------------ ------------------------------
PEASLAND                       DEFAULT
TEST_USER                      DEFAULT
清单 14-15
用户的配置文件
```

我看到用户正在使用名为 `DEFAULT` 的配置文件。要了解有关此配置文件的更多信息，我可以再次查询数据字典，如清单 14-16 所示。

```
SQL> select resource_name,limit from dba_profiles
2  where profile='DEFAULT';
RESOURCE_NAME                    LIMIT
-------------------------------- ------------------------------
COMPOSITE_LIMIT                  UNLIMITED
SESSIONS_PER_USER                UNLIMITED
CPU_PER_SESSION                  UNLIMITED
CPU_PER_CALL                     UNLIMITED
LOGICAL_READS_PER_SESSION        UNLIMITED
LOGICAL_READS_PER_CALL           UNLIMITED
IDLE_TIME                        UNLIMITED
CONNECT_TIME                     UNLIMITED
PRIVATE_SGA                      UNLIMITED
FAILED_LOGIN_ATTEMPTS            5
PASSWORD_LIFE_TIME               90
PASSWORD_REUSE_TIME              730
PASSWORD_REUSE_MAX               8
PASSWORD_VERIFY_FUNCTION         ORA12C_VERIFY_FUNCTION
PASSWORD_LOCK_TIME               .0069
PASSWORD_GRACE_TIME              7
INACTIVE_ACCOUNT_TIME            365
清单 14-16
默认配置文件设置
```

如果上述任何配置文件设置不符合我的安全预期，我可以着手进行改进。

了解数据库中哪些用户被授予了强大的 `DBA` 角色是很有用的。清单 14-17 显示了一个简单的查询，用于确定拥有 `DBA` 角色的用户。

```
SQL> select grantee from dba_role_privs
2  where granted_role='DBA';
GRANTEE
------------------------------
PEASLAND
SYSTEM
SYS
清单 14-17
DBA 用户
```


我总是期望看到`SYSTEM`和`SYS`用户拥有此角色。任何其他用户都应该是`DBA`团队的一部分。

接下来，我通常会尝试了解数据库中正在使用的初始化参数。我只关心那些被设置为非默认值的参数，因此我在清单 14-18 中的查询将给我非默认值。

```sql
SQL> select name,value from v$parameter
2  where isdefault='FALSE'
3  order by name;
NAME                           VALUE
------------------------------ ----------------------------------------
audit_file_dest                /u01/app/oracle/admin/orcl/adump
audit_trail                    DB
compatible                     12.2.0
control_files                  /u01/app/oracle/oradata/orcl/control01.c
tl, /u01/app/oracle/oradata/orcl/control
02.ctl
db_block_size                  8192
db_name                        orcl
diagnostic_dest                /u01/app/oracle
dispatchers                    (PROTOCOL=TCP) (SERVICE=orclXDB)
nls_language                   AMERICAN
nls_territory                  AMERICA
open_cursors                   300
pga_aggregate_target           810549248
processes                      300
remote_login_passwordfile      EXCLUSIVE
sga_target                     4294967296
undo_tablespace                UNDOTBS1
```
**清单 14-18** 非默认参数

接下来，我确定数据库中的计划任务。我不关心 Oracle 内部操作的任务，因此我忽略了那些由 `SYS` 和 `ORACLE_OCM` 拥有的任务，如清单 14-19 所示。

```sql
SQL> select owner,job_name,repeat_interval
2  from DBA_scheduler_jobs
3  where owner'SYS' and owner'ORACLE_OCM'
4  order by 1,2;
OWNER     JOB_NAME          REPEAT_INTERVAL
--------- ----------------- --------------------------------------------
HR        EMP_ROLLUP_JOB    freq=daily;byhour=01;byminute=01;bysecond=01
```
**清单 14-19** 计划任务

仅仅通过几次查询，并快速浏览数据字典，我就对我继承的这个数据库了解了不少。正如生活中任何事情一样，当我们得知一个问题的答案时，很可能会引发新的问题。我们需要做的就是找到合适的数据字典视图，进一步了解这个特定的数据库。

没有作者能写一本包含你数据库具体信息的书。我最多只能向你展示如何利用数据字典。你现在有能力深入了解你正在管理的特定数据库。

## 继续前进

数据字典是数据库的内部文档。作为一名 Oracle DBA，你会发现自己需要不断地查询数据字典来解答许多问题。Oracle 文档告诉你 Oracle 数据库通常是**如何工作**的。而数据字典则具体地告诉你**关于你的数据库**的信息。

在下一章中，我们将讨论 My Oracle Support。这个宝贵的资源能让你了解到比文档所写的更多的 Oracle 数据库知识。它可以帮助你判断一个问题是否与缺陷（bug）相关，以及是否有已知的解决方法或补丁。如果 Oracle 文档和数据字典无法回答你的问题，My Oracle Support 通常是下一个去处。

