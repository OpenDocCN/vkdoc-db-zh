# 创建密码文件

创建密码文件是可选的。要求使用密码文件有一些充分的理由：

*   你想为非 `sys` 用户分配 `sys*` 权限（如 `sysdba`、`sysoper`、`sysbackup` 等）。
*   你想通过 Oracle Net 使用 `sys*` 权限远程连接到你的数据库。
*   设置 Oracle Data Guard 并且需要在备用服务器上使用密码文件。
*   某个 Oracle 特性或工具要求使用密码文件。

请执行以下步骤来实现密码文件：

1.  使用 `orapwd` 工具创建密码文件。
2.  将初始化参数 `REMOTE_LOGIN_PASSWORDFILE` 设置为 `EXCLUSIVE`。

在 Linux/Unix 环境中，使用 `orapwd` 工具创建密码文件，如下所示：

```bash
$ cd $ORACLE_HOME/dbs
$ orapwd file=orapw<SID> password=<password>
```

在 Linux/Unix 环境中，密码文件通常存储在 `ORACLE_HOME/dbs` 中；在 Windows 中，它通常放置在 `ORACLE_HOME\database` 目录中。

你在前一条命令中指定的文件名格式可能因操作系统而异。例如，在 Windows 中，格式是 `PWD<ORACLE_SID>.ora`。以下示例展示了在 Windows 环境中的语法：

```bash
c:\> cd %ORACLE_HOME%\database
c:\> orapwd file=PWD<SID>.ora password=<password>
```

要启用密码文件的使用，请将初始化参数 `REMOTE_LOGIN_PASSWORDFILE` 设置为 `EXCLUSIVE`（这是默认值）。如果该参数未设置为 `EXCLUSIVE`，那么你需要修改你的参数文件：

```sql
SQL> alter system set remote_login_passwordfile='EXCLUSIVE' scope=spfile;
```

你需要停止并启动实例以使上述设置生效。

你可以通过 `GRANT <any SYS privilege>` 语句向密码文件添加用户。使用这些权限和密码文件进行安全配置时需要谨慎。只有需要这些权限的账户才应被授予访问密码文件的权限。以下示例授予 `heera` 用户 `SYSDBA` 权限（从而将 `heera` 添加到密码文件）：

```sql
SQL> grant sysdba to heera;
Grant succeeded.
```

启用密码文件还允许你通过 Oracle Net 连接，使用 `SYS*` 级别的权限远程连接到你的数据库。此示例展示了使用 `SYSDBA` 级别权限进行远程连接的语法：

```bash
$ sqlplus /@<db_name> as sysdba
```

这允许你执行远程维护，使用 `sys*` 权限（`sysdba`、`sysoper`、`sysbackup` 等），否则你需要物理登录到数据库服务器。你可以通过查询 `V$PWFILE_USERS` 视图来验证哪些用户拥有 `sys*` 权限：

```sql
SQL> select * from v$pwfile_users;
```

以下是一些示例输出：

```text
USERNAME                   SYSDB SYSOP SYSAS SYSBA SYSDG SYSKM     CON_ID
-------------------------- ----- ----- ----- ----- ----- ----- ----------
SYS                        TRUE  TRUE  FALSE FALSE FALSE FALSE          0
```

特权用户的概念对于 RMAN 备份和恢复也很重要。与 SQL*Plus 类似，RMAN 使用操作系统认证和密码文件来允许特权用户连接到数据库。只有特权账户才允许备份、恢复和恢复数据库。

