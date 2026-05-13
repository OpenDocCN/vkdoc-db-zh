# 第十七章 ■ 数据库管理

## 创建数据库

### 初始化文件

在 Linux/Unix 系统中，初始化文件（文本文件 `init.ora` 或二进制文件 `spfile`）默认位于 `ORACLE_HOME/dbs` 目录。在 Windows 系统中，默认目录是 `ORACLE_HOME\database`。

启动实例时，Oracle 会首先在默认位置查找名为 `spfile<ORACLE_SID>.ora` 的二进制初始化文件。如果没有二进制 `spfile`，则会查找名为 `init<ORACLE_SID>.ora` 的文本文件。如果在默认位置找不到初始化文件，Oracle 将报错。你可以通过 `STARTUP` 命令的 `PFILE` 子句明确指定使用哪个目录和文本初始化文件，以覆盖默认设置。

以下是一个典型的 Oracle Database 11g `init.ora` 文本文件内容：

```text
db_name=INVREP
db_block_size=8192
compatible=11.1.0
memory_target=800M
memory_max_target=800M
processes=200
control_files=(/ora01/oradata/INVREP/control01.ctl,
/ora02/oradata/INVREP/control02.ctl)
diagnostic_dest=/orahome/oracle
job_queue_processes=10
open_cursors=300
fast_start_mttr_target=500
undo_management=AUTO
undo_tablespace=UNDOTBS1
remote_login_passwordfile=EXCLUSIVE
```

以下是一个典型的 Oracle Database 10g `init.ora` 文本文件内容：

```text
db_name=INVREP
db_block_size=8192
pga_aggregate_target=400M
workarea_size_policy=AUTO
sga_max_size=500M
sga_target=500M
processes=400
control_files=(/ora01/oradata/INVREP/control01.ctl,
/ora01/oradata/INVREP/control02.ctl)
job_queue_processes=10
open_cursors=300
fast_start_mttr_target=500
background_dump_dest=/orahome/oracle/admin/INVREP/bdump
user_dump_dest=/orahome/oracle/admin/INVREP/udump
core_dump_dest=/orahome/oracle/admin/INVREP/cdump
undo_management=AUTO
undo_tablespace=UNDOTBS1
remote_login_passwordfile=EXCLUSIVE
```

### 数据字典脚本

数据库成功创建后，你可以运行 Oracle 提供的两个脚本来实例化数据字典。必须以 `SYS` 模式运行这些脚本：

```sql
SQL> show user
USER is "SYS"
SQL> @?/rdbms/admin/catalog
SQL> @?/rdbms/admin/catproc
```

接下来，以 `SYSTEM` 模式创建产品用户配置表：

```sql
SQL> connect system/manager
SQL> @?/sqlplus/admin/pupbld
```

这些表允许 SQL*Plus 基于用户禁用特定命令。如果未运行 `pupbld.sql` 脚本，所有非 `SYS` 用户在登录 SQL*Plus 时将看到以下警告：
`Error accessing PRODUCT_USER_PROFILE`
`Warning: Product user profile information not loaded!`
`You may need to run PUPBLD.SQL as SYSTEM`

##### 17-2. 删除数据库

### 问题

你想删除一个数据库，并移除与其关联的所有数据文件、控制文件和联机重做日志。

### 解决方案

确保你在正确的服务器上，并连接到正确的数据库。在 Linux/Unix 系统上，从操作系统提示符执行以下命令：

```bash
$ uname -a
```

然后连接到 SQL*Plus，并确认你连接的是确实要删除的数据库：

```sql
select name from v$database;
```

确认无误后，从具有 `SYSDBA` 权限的账户执行以下 SQL 命令：

```sql
shutdown immediate;
startup mount exclusive restrict;
drop database;
```



■ **注意** 显然，在删除数据库时您必须格外小心。执行删除数据库操作时系统*不会*给予提示，而且截至本文撰写时，尚不存在 `UNDROP ACCIDENTALLY DROPPED DATABASE`（撤消意外删除的数据库）命令。

### 工作原理

删除数据库时请*极其*谨慎，因为此操作将移除数据文件、控制文件和在线重做日志文件。

`DROP DATABASE` 命令在您需要移除某个数据库时非常有用。这可以是一个测试数据库或一个不再使用的旧数据库。

`DROP DATABASE` 命令不会移除旧的归档重做日志文件。您需要使用操作系统命令（例如 Linux/Unix 中的 `rm` 或 Windows 命令提示符下的 `DEL`）手动删除这些文件。您也可以指示 `RMAN` 来删除归档重做日志文件。

##### 17-3. 验证连接信息

### 问题

您正在运行多个 `DDL` 脚本以进行应用程序升级。在执行命令之前，您希望确保已连接到正确的数据库环境。

### 解决方案

验证环境的最简单方法是连接到数据库并查询 `V$DATABASE` 视图：

```
select name from v$database;
```

您也可以运行 `SHOW USER` 命令来快速验证您连接到数据库时所使用的模式：

```
SQL> show user;
```

如果您希望 `SQL` 提示符显示用户和实例信息，可以通过以下命令进行设置：

```
SQL> set sqlprompt '&_user.@&_connect_identifier.> '
```

以下是您的 `SQL` 命令提示符可能的外观示例：

```
SYS@RMDB11>
```

如果您希望在以不同用户连接到数据库时提示符发生变化，请将上述代码行添加到您的 `glogin.sql` 文件或 `login.sql` 文件中。`glogin.sql` 文件通常位于 `ORACLE_HOME/sqlplus/admin` 目录下，并且会在任何用户连接到 `SQL*Plus` 时运行。

如果您只希望在连接到数据库时显示更改，请在您启动 `SQL*Plus` 的目录中创建一个 `login.sql` 文件，或者确保 `login.sql` 文件位于您的 `SQLPATH` 操作系统环境变量所包含的目录中。

### 工作原理

确保正确连接至少涉及两个方面。首先，您必须确定已连接到正确的数据库。其次，您必须确保在该数据库内连接到了适当的用户。解决方案展示了如何验证数据库和用户。验证这些信息后，您就可以放心地运行您的 `DDL` 脚本了。

有时在将代码部署到不同的开发、测试和生产环境时，能收到关于是否处于正确环境的提示会非常方便。实现此功能的技术需要两个文件：`answer_yes.sql` 和 `answer_no.sql`。以下是 `answer_yes.sql` 的内容：

```
-- answer_yes.sql

PROMPT

PROMPT Continuing...
```

而这是 `answer_no.sql`：

```
-- answer_no.sql

PROMPT

PROMPT Quitting and Discarding changes...

ROLLBACK;

EXIT;
```

现在，您可以将以下代码放入部署脚本的开头部分，它会提示您是否处于正确的环境以及是否要继续：

```
WHENEVER SQLERROR EXIT FAILURE ROLLBACK;

WHENEVER OSERROR EXIT FAILURE ROLLBACK;

SELECT name FROM v$database;

SHOW user;

SET ECHO OFF;

PROMPT

ACCEPT answer PROMPT 'Correct environment? Enter yes to continue: '

@@answer_&answer..sql
```

如果您输入 `yes`，则 `answer_yes.sql` 脚本将运行，您将继续运行您调用的任何其他脚本。如果您输入 `no`，则 `answer_no.sql` 脚本将运行，您将退出 `SQL*Plus` 并最终返回到 `OS` 提示符。如果您直接按 `Enter` 键而不输入任何内容，您也会退出并返回到 `OS` 提示符。

##### 17-4. 创建表空间

### 问题

您想要创建一个表空间。

### 解决方案

使用 `CREATE TABLESPACE` 语句来创建表空间。此示例展示了创建表空间时常用的选项：


