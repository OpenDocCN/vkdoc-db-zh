# 在 Linux/Unix 上

在 Linux/Unix 上，您可以使用不带任何参数的 `id` 命令快速验证您的帐户所属的操作系统组：

```
$ id
uid=500(oracle) gid=500(oinstall) groups=500(oinstall),501(dba),502(oper),503(asmdba),
504(asmoper),505(asmadmin),506(backupdba)
```

前面的输出表明 `oracle` 用户包含在几个组中，其中一个是 `dba`。任何属于 `dba` 组的用户都可以使用 `SYSDBA` 权限连接到数据库。拥有 `SYSDBA` 权限的用户可以启动和停止数据库。此示例使用操作系统认证以 `SYS` 用户身份连接到您的数据库：

```
$ sqlplus / as sysdba
```

使用操作系统认证时不需要用户名或密码（因此只有一个斜杠而没有用户/密码），因为 Oracle 首先检查操作系统用户是否是特权操作系统组的成员，如果是，则连接而不检查用户名/密码。您可以通过发出以下命令来验证是否已作为 `SYS` 连接：

```
SQL> show user
USER is "SYS"
```

特权操作系统组是在安装 Oracle 软件时建立的。有几个与备份和恢复相关的操作系统组：

*   `dba`
*   `oper`
*   `backupdba`（从 Oracle 12c 开始提供）

每个操作系统组对应某些数据库权限。表 1-1 显示了操作系统组到数据库系统权限和操作的映射。

表 1-1. 操作系统组到与备份和恢复相关权限的映射

| 操作系统组 | 数据库系统权限 | 授权的操作 |
| --- | --- | --- |
| `dba` | `sysdba` | 启动、关闭、更改数据库、创建和删除数据库、切换归档日志模式、备份和恢复数据库。 |
| `oinstall` | `none` | 安装和升级 Oracle 二进制文件。 |
| `oper` | `sysoper` | 启动、关闭、更改数据库、切换归档日志模式、备份和恢复数据库。 |
| `backupdba` | `sysbackup` | 从 Oracle 12*c* 开始可用，此权限允许您启动、关闭并执行所有备份和恢复操作。 |

### 使用密码文件

如果您不使用操作系统认证，那么您可以使用密码文件以特权用户身份连接到数据库。密码文件允许您从 SQL*Plus 或 RMAN 执行以下操作：

*   以非 `SYS` 数据库用户身份使用 `sys*` 权限连接到您的数据库
*   通过网络使用 `sys*` 权限连接到远程数据库

密码文件必须使用 `orapwd` 实用程序手动创建，并通过 SQL `grant` 命令填充。要实现密码文件，请执行以下步骤：

1.  使用 `orapwd` 实用程序创建密码文件。
2.  将初始化参数 `remote_login_passwordfile` 设置为 `exclusive`。

在 Linux/Unix 环境中，使用 `orapwd` 实用程序创建密码文件，如下所示：

```
$ cd $ORACLE_HOME/dbs
$ orapwd file=orapw$ORACLE_SID password=<sys password>
```

在 Linux/Unix 环境中，密码文件通常存储在 `ORACLE_HOME/dbs` 目录中，在 Windows 中，它通常放置在 `ORACLE_HOME\database` 目录中。您在上一条命令中指定的文件名格式可能因操作系统而异。例如，在 Windows 上格式是 `PWD<ORACLE_SID>.ora`。以下显示了 Windows 环境中的语法：

```
c:\> cd %ORACLE_HOME%\database
c:\> orapwd file=PWD<ORACLE_SID>.ora password=<sys password>
```

要启用密码文件的使用，请将初始化参数 `remote_login_passwordfile` 设置为 `exclusive`（这是默认值）。您可以如下所示验证其值：

```
SQL> show parameter remote_login_password

NAME                                 TYPE        VALUE
--------------------------           ------      ---------
remote_login_passwordfile            string      EXCLUSIVE
```

如果需要，您可以手动设置 `remote_login_passwordfile` 参数，如下所示：

```
$ sqlplus / as sysdba
SQL> alter system set remote_login_passwordfile=exclusive scope=spfile;
```

然后您需要停止并启动数据库以使此参数生效（有关停止/启动数据库的更多详细信息将在本章后面介绍）。前面的示例假设您使用的是服务器参数文件（`spfile`）。如果您没有使用 `spfile`，则必须通过文本编辑器手动编辑 `init.ora` 文件并添加此条目：

```
remote_login_passwordfile=exclusive
```

然后停止并启动数据库以实例化该参数。启用密码文件后，您就可以创建数据库用户并根据需要为他们分配 `sys*` 权限。例如，假设您有一个名为 `DBA_MAINT` 的数据库用户，您想授予其 `SYSBACKUP` 权限：

```
$ sqlplus / as sysdba
SQL> grant sysbackup to dba_maint;
```

使用密码文件连接到数据库的语法如下：

```
$ sqlplus <username>/<password>[@<db conn string>] as sys[dba|oper|backup]
```

例如，使用 `DBA_MAINT` 数据库用户，您可以使用 `SYSBACKUP` 权限连接到数据库，如下所示：

```
$ sqlplus dba_maint/foo as sysbackup
```

因为您提供了用户名/密码并尝试以 `sys*` 级别权限（作为非 `SYS` 用户）连接，Oracle 将验证密码文件是否就位（对于本地数据库），并且提供的用户名/密码是否在密码文件中。您可以通过查询 `V$PWFILE_USERS` 视图来验证哪些用户具有 `sys*` 权限：

```
SQL> select * from v$pwfile_users;
```

以下是一些示例输出：

```
USERNAME                       SYSDB SYSOP SYSAS SYSBA SYSDG SYSKM     CON_ID
-----------------              ----- ----- ----- ----- ----- ----- ----------
SYS                            TRUE  TRUE  FALSE FALSE FALSE FALSE          0
DBA_MAINT                      FALSE FALSE FALSE TRUE  FALSE FALSE          0
```

## 操作系统认证与密码文件

对于本地连接（在物理登录到数据库服务器时进行），操作系统认证优先于密码文件认证。换句话说，如果您登录到一个属于已认证组（如 `dba`）的操作系统帐户，那么在使用 `sys*` 权限连接到本地数据库时，您输入什么用户名和密码并不重要。例如，您可以使用不存在的用户名/密码以 `sysdba` 身份连接：

```
$ sqlplus bogus/wrong as sysdba
SQL> show user;
USER is "SYS"
```

前面的连接有效，因为 Oracle 忽略了提供的用户名/密码，因为用户首先通过操作系统认证进行了验证。但是，当您不使用操作系统认证来建立特权本地连接，或者当您尝试通过网络连接到远程数据库时，则会使用密码文件。



