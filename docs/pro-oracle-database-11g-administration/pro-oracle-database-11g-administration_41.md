# 第 6 章 ■ 用户和基本安全性

## 管理默认用户

创建数据库时，Oracle 会创建几个默认的数据库用户。创建的特定用户因数据库版本而异。如果您刚刚创建了数据库，可以按如下方式查看默认用户帐户：

```sql
SQL> select username from dba_users order by 1;
```

以下是一些默认数据库用户帐户的示例列表：

```
USERNAME
-------------------------
APPQOSSYS
DBSNMP
DIP
ORACLE_OCM
OUTLN
SYS
SYSTEM
```

为了开始保护数据库，您至少应更改每个默认帐户的密码，然后锁定您不使用的任何帐户。创建数据库后，我通常会锁定每个默认帐户并将其密码设置为过期；我只在需要时才解锁默认用户。

以下脚本生成锁定所有用户并将其密码设置为过期的 SQL 语句：

```sql
select
'alter user ' || username || ' password expire account lock;'
from dba_users;
```

锁定的用户只能通过将用户更改为解锁状态来访问。例如：

```sql
SQL> alter user outln account unlock;
```

密码过期的用户在首次以该用户身份连接到数据库时会被提示输入新密码。连接到用户时，Oracle 会检查当前密码是否已过期，如果已过期，则提示您：

```
ORA-28001: the password has expired

Changing password for <user>
New password:
```

输入新密码后，系统会提示您再次输入：

```
Retype new password:
Password changed
Connected.
```

**注意：** 您可以锁定 `SYS` 帐户，但这对您使用操作系统身份验证或密码文件以 `SYS` 用户身份连接的能力没有影响。

如果您从其他 DBA 那里继承了一个数据库，那么有时确定是其他 DBA 创建了用户还是用户是 Oracle 创建的默认帐户是很有用的。如前所述，通常在创建数据库时会为您创建几个用户帐户。帐户数量因版本和安装的选项而异。运行此查询以显示由其他 DBA 创建的用户与由 Oracle 创建的用户（例如在创建数据库时默认创建的用户）：

```sql
select
  distinct u.username
  ,case when d.user_name is null then 'DBA created account'
   else 'Oracle created account'
  end
from   dba_users u
       ,default_pwd$ d
where  u.username=d.user_name(+);
```

**注意：** `DEFAULT_PWD$` 视图从 Oracle Database 11g 开始可用。有关检查默认密码的指南的更多详细信息，请参阅 My Oracle Support 说明 227010.1。

在确定密码是否安全时，检查用户密码是否曾经更改过是很有用的。如果某个用户的密码从未更改过，这可能被视为安全风险。此示例执行此类检查：

```sql
select
  name
  ,to_char(ctime,'dd-mon-yy hh24:mi:ss')
  ,to_char(ptime,'dd-mon-yy hh24:mi:ss')
  ,length(password)
from   user$
where  password is not null
and    password not in ('GLOBAL','EXTERNAL')
and    ctime=ptime;
```

在此脚本中，`CTIME` 列包含用户创建时的时间戳。`PTIME` 列包含密码更改时的时间戳。如果 `CTIME` 和 `PTIME` 相同，则表明密码从未更改。

您还应该检查您的数据库以确定是否有任何帐户使用默认密码。如果您使用的是 Oracle Database 11*g* 或更高版本，您可以检查 `DBA_USERS_WITH_DEFPWD` 视图以确定是否任何 Oracle 创建的用户帐户仍设置为默认密码：

```sql
SQL> select * from dba_users_with_defpwd;
```

如果您未使用 Oracle Database 11*g*，则必须手动检查密码或使用脚本。下面是一个简单的 shell 脚本，它尝试使用默认密码连接到数据库：

```bash
#!/bin/bash
if [ $# -ne 1 ]; then
echo "Usage: $0 SID"
exit 1
fi
```

