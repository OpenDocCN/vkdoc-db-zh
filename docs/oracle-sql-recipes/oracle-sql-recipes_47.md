# 第 17 章 ■ 数据库管理

##### 17-8. 关联一组权限

### 问题

你想将权限分组，以便通过一次操作将它们分配给一个模式。

### 解决方案

首先，以一个具有 `CREATE ROLE` 系统权限的用户身份连接到数据库。接下来创建一个角色，并将你希望分组在一起的系统或对象权限分配给它。此示例使用 `CREATE ROLE` 命令创建 `JR_DBA` 角色：

```sql
create role jr_dba;
```

接下来的几行 SQL 将系统权限授予新创建的角色：

```sql
grant select any table to jr_dba;
grant create any table to jr_dba;
grant create any view to jr_dba;
grant create synonym to jr_dba;
grant create database link to jr_dba;
```

然后，将该角色授予你希望拥有这些权限的任何模式：

```sql
grant jr_dba to lellison;
grant jr_dba to cphillips;
```

现在，模式 `LELLISON` 和 `CPHILLIPS` 就可以执行诸如创建同义词、视图等任务。作为 DBA 模式，要查看分配了角色的模式，可以查询 `DBA_ROLES_PRIVS` 视图：

```sql
select grantee, granted_role from dba_role_privs order by 1;
```

要查看授予当前连接用户的角色，可以从 `USER_ROLE_PRIVS` 视图查询：

```sql
select * from user_role_privs;
```

要从角色撤销权限，使用 `REVOKE` 命令：

```sql
revoke create database link from jr_dba;
```

同样，使用 `REVOKE` 命令从模式中移除角色：

```sql
revoke jr_dba from lellison;
```

> **注意**
>
> 与其他数据库对象不同，角色没有所有者。角色由分配给它的权限定义。

### 工作原理

数据库*角色*是一组系统或对象权限的逻辑分组，可以分配给模式或其他角色。角色帮助你管理数据库安全性的各个方面。它们还提供了一个中心对象，权限可以分配给它；该角色随后可以被分配给多个用户。

角色的典型用法是当你需要授予多个用户对另一个用户的表的只读访问权限时。要实现这一点，首先创建一个名为 `SELECT_APP` 的角色：

```sql
create role select_app;
```

接下来，为需要只读访问的每个表授予 `SELECT` TABLE 访问权限。以下代码动态创建一个脚本，为模式中的每个表向指定角色授予 `SELECT` 访问权限：

```sql
CONN &user_name/&password
DEFINE sel_user=SELECT_APP
SET LINES 132 PAGES 0 ECHO OFF FEEDBACK OFF VERIFY OFF HEAD OFF TERM OFF TRIMS ON
SPO gen_grants_dyn.sql
--
SELECT 'grant select on ' || table_name || ' to &&sel_user ;'
FROM user_tables;
--
SPO OFF;
SET ECHO ON FEEDBACK ON VERIFY ON HEAD ON TERM ON;
--
@@gen_grants_dyn
```

现在，你可以将 `SELECT_APP` 角色分配给任何需要对拥有表的模式内的表进行只读访问的用户。

### PL/SQL 与角色

如果你使用 PL/SQL，有时在尝试编译过程或函数时会遇到此错误：

`PL/SQL: ORA-00942: table or view does not exist`

令你困惑的是，你可以描述该表：

`SQL> desc app_table;`

那么为什么 PL/SQL 似乎无法识别该表呢？这是因为 PL/SQL 要求包、过程或函数的所有者必须被显式授予对代码中引用的任何对象的权限。PL/SQL 代码的所有者不能通过角色获得这些授权。

尝试以 PL/SQL 代码所有者的身份执行：

`SQL> set role none;`

现在尝试运行访问该问题表的 SQL 语句：

`SQL> select count(*) from app_table;`

如果你不再能访问该表，那么你就是通过角色获得了访问权限。要解决此问题，请显式向 PL/SQL 代码的所有者授予对任何表的访问权限（以表所有者的身份）：

`SQL> connect owner/pass`
`SQL> grant select on app_table to proc_owner;`

现在，你应该能够以 PL/SQL 代码所有者的身份连接并成功编译你的代码。

##### 17-9. 创建用户

### 问题

你需要创建一个新用户。

### 解决方案

使用 `CREATE USER` 语句创建用户。此示例创建一个名为 `HEERA` 的用户，密码为 `CHAYA`，并分配默认临时表空间为 `TEMP`，默认表空间为 `USERS`：

```sql
create user heera identified by chaya
default tablespace users
temporary tablespace temp;
```

这会创建一个基本的模式，该模式在数据库中没有任何执行操作的权限。要使该模式有用，必须至少授予其 `CREATE SESSION` 系统权限：

```sql
grant create session to heera;
```

如果新模式需要能够创建表，则需要授予额外的权限，如 `CREATE TABLE`。

```sql
grant create table to heera;
```

新模式还需要在其需要创建对象的任何表空间上授予配额权限：

```sql
alter user heera quota unlimited on users;
```

> **注意**
>
> DBA 使用的一个常见技术是将预定义的 `CONNECT` 和 `RESOURCE` 角色授予新创建的模式。这些角色包含诸如 `CREATE SESSION` 和 `CREATE TABLE`（以及其他一些，因版本而异）之类的系统权限。我们不建议这样做，因为 Oracle 已声明这些角色在未来版本中可能不再可用。

### 工作原理

用户是一个数据库账户，它是允许你登录数据库的机制。你必须具有 `CREATE USER` 系统权限才能创建用户。创建用户时，你建立了初始的安全性和存储属性。这些用户属性可以使用 `ALTER USER` 语句进行修改。

你可以通过从 `DBA_USERS` 视图中选择来查看用户信息：

```sql
select username, default_tablespace, temporary_tablespace from dba_users;
```

以下是一些示例输出：

```
USERNAME               DEFAULT_TABLESPACE       TEMPORARY_TABLESPACE
---------------------- ------------------------ ------------------------------
C2R                    USERS                    TEMP
REP_MV_SEL             USERS                    TEMP
STAR_SEL               USERS                    TEMP
SYS                    SYSTEM                   TEMP
```

你所有的用户都应被分配一个已创建为*临时*类型的临时表空间。如果用户的临时表空间是 `SYSTEM`，那么他们所需的任何临时磁盘存储的排序区都将在 `SYSTEM` 表空间中获取区间。这可能导致 `SYSTEM` 表空间暂时被填满。除了 `SYS` 用户外，你的任何用户都不应将默认表空间设置为 `SYSTEM`。你不希望除 `SYS` 外的任何用户在 `SYSTEM` 表空间中创建对象。`SYSTEM` 表空间应保留给 `SYS` 用户的对象。

##### 17-10. 删除用户

### 问题

你想删除一个用户。

### 解决方案

使用 `DROP USER` 语句删除数据库账户。下一个示例删除用户 `HEERA`：

```sql
drop user heera;
```

如果该用户拥有任何数据库对象，此命令将不起作用。使用 `CASCADE` 子句删除用户并删除其对象：

```sql
drop user heera cascade;
```

> **注意**
>
> 如果被删除的用户拥有大量数据库对象，`DROP USER` 语句可能需要异常长的时间才能执行。在这种情况下，你可能需要考虑在删除用户之前先删除该用户的对象。

### 工作原理



当你删除一个用户时，该用户拥有的所有表也将被删除。此外，所有索引、触发器和参照完整性约束都将被移除。如果其他模式中存在依赖于被删除的主键和唯一键约束的参照完整性约束，这些参照约束也将在其他模式中被删除。Oracle 将使依赖于被删除用户对象的视图、同义词、过程、函数或包失效，但不会删除它们。

有时你可能会遇到想要删除一个用户，但不确定是否有人在使用它的情况。在这些场景下，更安全的方法是先锁定该用户，看看谁会有反应：
```sql
alter user heera account lock;
```
任何尝试连接到该用户的用户或应用程序现在都将收到此错误：`ORA-28000: the account is locked`

要查看数据库中的用户和锁定日期，请执行此查询：
```sql
select username, lock_date from dba_users;
```
要解锁账户，请执行此命令：
```sql
alter user heera account unlock;
```
锁定用户是保护数据库安全和发现哪些用户正在被使用的非常便捷的技术。

##### 17-11. 修改密码

### 问题
你想修改一个用户的密码。

### 解决方案
使用 `ALTER USER` 命令来修改现有用户的密码。此示例将 `HEERA` 用户的密码更改为 `foobar`：
```sql
alter user heera identified by foobar;
```
只有当你的用户被授予了 `ALTER USER` 权限时，你才能更改其他账户的密码。此权限被授予 `DBA` 角色。

### 工作原理
更改用户密码后，该用户后续连接数据库时必须使用 `ALTER USER` 语句更改后的密码。

在 Oracle Database 11g 或更高版本中，修改密码时默认区分大小写。有关如何禁用密码大小写敏感性的详细信息，请参阅配方 17-12。如果你使用的是 Oracle Database 10g 或更低版本，密码不区分大小写。

### SQL*Plus PASSWORD 命令
你可以使用 `SQL*Plus` 的 `PASSWORD` 命令更改用户密码。（与所有 `SQL*Plus` 命令一样，它可以缩写。）系统将提示你输入新密码：
```
SQL> passw heera
Changing password for heera
New password:
Retype new password:
Password changed
```
此方法的优点是可以在屏幕上不显示新密码的情况下更改用户密码。

##### 17-12. 强制密码复杂性

### 问题
你想确保你的数据库用户创建的密码不会被黑客轻易猜到。

### 解决方案
以 `SYS` 模式身份运行以下脚本：
```
SQL> @?/rdbms/admin/utlpwdmg
Function created.
Profile altered.
Function created.
```
对于 Oracle Database 11g，将 `DEFAULT` 配置文件的 `PASSWORD_VERIFY_FUNCTION` 设置为 `verify_function_11G`：
```sql
alter profile default limit PASSWORD_VERIFY_FUNCTION verify_function_11G;
```
对于 Oracle Database 10g，将 `DEFAULT` 配置文件的 `PASSWORD_VERIFY_FUNCTION` 设置为 `VERIFY_Function`：
```sql
alter profile default limit PASSWORD_VERIFY_FUNCTION verify_function;
```
如果出于任何原因需要退出新的安全设置，运行此语句以禁用密码函数：
```sql
alter profile default limit PASSWORD_VERIFY_FUNCTION null;
```

### 工作原理
启用后，密码验证函数可确保用户正确创建或修改其密码。`utlpwdmg.sql` 脚本创建一个函数来检查密码，确保其满足基本安全标准，如最短密码长度、密码不能与用户名相同，等等。在 Oracle Database 11g 中，密码安全 PL/SQL 函数名为 `VERIFY_FUNCTION_11G`。

一旦这个 PL/SQL 函数在 `SYS` 模式中创建，就可以将其分配给配置文件的 `PASSWORD_VERIFY_FUNCTION` 参数。



