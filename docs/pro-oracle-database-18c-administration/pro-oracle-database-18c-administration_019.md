# users.sql

此脚本显示有关用户创建时间及其账户是否被锁定的信息。当您排查连接问题时，此脚本非常有用。将脚本放在诸如`HOME/scripts`之类的目录中。以下是用于显示用户账户信息的典型`users.sql`脚本：

```
SELECT
username
,account_status
,lock_date
,created
FROM dba_users
ORDER BY username;
```

您可以从 SQL*Plus 执行此脚本，如下所示：

```
SQL> @users.sql
```

以下是一些示例输出：

```
USERNAME        ACCOUNT_ST LOCK_DATE    CREATED
--------------- ---------- ------------ ------------
SYS             OPEN                    09-NOV-12
SYSBACKUP       OPEN                    09-NOV-12
SYSDG           OPEN                    09-NOV-12
```

在 Oracle Database 18c 可插拔数据库环境中从根容器运行`users.sql`时，您需要将`DBA_USERS`更改为`CDB_USERS`，并添加`NAME`和`CON_ID`列以报告所有可插拔数据库中的所有用户；例如，

```
SELECT
c.name
,u.username
,u.account_status
,u.lock_date
,u.created
FROM cdb_users    u
,v$containers c
WHERE u.con_id = c.con_id
ORDER BY c.name, u.username;
```

