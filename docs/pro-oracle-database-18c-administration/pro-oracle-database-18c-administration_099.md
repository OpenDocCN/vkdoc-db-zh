# 创建 CDB

要使用可插棒数据库功能，你必须专门创建一个支持可插拔的 CDB。有几种不同的技术可以创建 CDB：

*   使用 `dbca` 实用程序
*   使用 DBCA 生成所需的脚本，然后手动运行这些脚本来创建 CDB
*   手动发出 SQL `CREATE DATABASE` 命令，使用 RMAN 复制现有的 CDB

接下来的两节重点介绍如何使用 `CREATE DATABASE` 命令和 DBCA 创建数据库。有关使用 RMAN 复制 CDB 或可插棒数据库的详细信息，请参阅 Darl Kuhn、Sam Alapati 和 Arup Nanda 所著的 *RMAN Recipes for Oracle Database 12c*，第二版（Apress，2013）。

## 使用数据库配置助手 (DBCA)

你可以使用 DBCA 实用程序通过图形界面或命令行模式创建 CDB。使用图形界面时，系统会提示你是否要创建 CDB。

本示例将引导你使用命令行模式。首先，确保你的 `ORACLE_SID`、`ORACLE_HOME` 和 `PATH` 变量已为 CDB 环境设置（有关设置 OS 变量的详细信息，请参见第 2 章）；例如，

```
$ export ORACLE_SID=CDB
$ export ORACLE_HOME=/u01/app/oracle/product/18.1.0.1/db_1
$ export PATH=$ORACLE_HOME/bin:$PATH
```

现在，使用 DBCA 创建 CDB。使用 DBCA 的命令行模式时，必须指定 `createAsContainerDatabase` 子句。以下代码创建一个名为 `CDB` 的数据库：

```
dbca -silent -createDatabase -templateName General_Purpose.dbc -gdbname CDB
-sid CDB -responseFile NO_VALUE -characterSet AL32UTF8 -memoryPercentage 30
-emConfiguration LOCAL -createAsContainerDatabase true
-sysPassword foo -systemPassword foo
```

前面的代码必须在一行中输入（这里为了适应页面才显示在多行）。你提供的 SID（本例中为 `CDB`）不能存在于你的 `oratab` 文件中。在 Linux 系统上，`oratab` 文件通常位于 `/etc` 目录。在其他 Unix 系统（如 Solaris）上，`oratab` 文件通常位于 `/var/opt/oracle` 目录。

按下 Enter 键后，你应该会看到类似以下的输出：

```
Copying database files
...
9% complete
41% complete
Creating and starting Oracle instance
43% complete
48% complete
...
```

DBCA 助手需要几分钟时间来创建新的 CDB。创建此数据库时，请确保有足够的磁盘空间可用（大约 5GB 应该足够）。DBCA 运行完成后，你应该拥有一个功能齐全的 CDB 数据库，并可以开始在其中创建 PDB。

## 通过 DBCA 生成 CDB 创建脚本

你可以使用 DBCA 生成创建 CDB 所需的脚本，然后可以选择手动运行这些脚本来执行该过程。接下来的代码从命令行调用 DBCA 实用程序，并生成创建名为 `CDB1` 的 CDB 所需的脚本：

```
dbca -silent -generateScripts -templateName General_Purpose.dbc -gdbname CDB1
-sid CDB1 -responseFile NO_VALUE -characterSet AL32UTF8 -memoryPercentage 30
-emConfiguration LOCAL -createAsContainerDatabase true
-sysPassword foo -systemPassword foo
```

前面的代码在执行时必须在一行中输入。以下是典型的输出片段：

```
Database creation script generation
1% complete
.....
100% complete
Look at the log file "/orahome/app/oracle/admin/CDB1/scripts/CDB1.log"...
```

DBCA 运行完成后，将目录更改为新生成脚本的位置：

```
$ cd $ORACLE_BASE/admin/CDB1/scripts
```

现在，显示目录中的脚本：

```
$ ls
```

以下是一些示例输出：



## 创建 Oracle CDB 数据库

以下文件列表包含了创建数据库所需的脚本和配置文件：
```
CDB1.log                  cloneDBCreation.sql       lockAccount.sql
CDB1.sh                   init.ora                  postDBCreation.sql
CDB1.sql                  initCDB1Temp.ora          postScripts.sql
CloneRmanRestore.sql      initCDB1TempOMF.ora       rmanRestoreDatafiles.sql
tempControl.ctl
```

要创建数据库，请运行 `CDB1.sh` 脚本：
```
$ CDB1.sh
```

前面的 shell 脚本调用了 `CDB1.sql`，该脚本随后会调用 RMAN 工具。RMAN 从备份中创建所需的数据文件，并创建一个名为 `CDB1` 的数据库。将 `ORACLE_SID` 操作系统变量设置为新创建的 CDB 名称后，您应该就能够启动数据库了。

## 手动使用 SQL 创建

首先，确保为您的 CDB 环境设置了 `ORACLE_SID`、`ORACLE_HOME` 和 `PATH` 变量（有关设置操作系统变量的详细信息，请参见第 2 章）；例如：
```
$ export ORACLE_SID=CDB
$ export ORACLE_HOME=/u01/app/oracle/product/18.1.0.1/db_1
$ export PATH=$ORACLE_HOME/bin:$PATH
```

接下来，在 `ORACLE_HOME/dbs` 目录中创建一个参数初始化文件。请确保将 `ENABLE_PLUGGABLE_DATABASE` 参数设置为 `TRUE`。这里，使用操作系统文本编辑器创建一个名为 `initCDB.ora` 的文件，并在其中放入以下参数规范：
```
db_name='CDB'
enable_pluggable_database=true
audit_trail='db'
control_files='/u01/dbfile/CDB/control01.ctl','/u01/dbfile/CDB/control02.ctl'
db_block_size=8192
db_domain="
memory_target=629145600
memory_max_target=629145600
open_cursors=300
processes=300
remote_login_passwordfile='EXCLUSIVE'
undo_tablespace='UNDOTBS1'
```

现在，使用操作系统文本编辑器创建一个名为 `credb.sql` 的文件，并在其中放入合适的 `CREATE DATABASE` 语句：
```
SQL> CREATE DATABASE CDB
MAXLOGFILES 16
MAXLOGMEMBERS 4
MAXDATAFILES 1024
MAXINSTANCES 1
MAXLOGHISTORY 680
CHARACTER SET US7ASCII
NATIONAL CHARACTER SET AL16UTF16
DATAFILE
'/u01/dbfile/CDB/system01.dbf' SIZE 500M
EXTENT MANAGEMENT LOCAL
UNDO TABLESPACE undotbs1 DATAFILE
'/u01/dbfile/CDB/undotbs01.dbf' SIZE 800M
SYSAUX DATAFILE
'/u01/dbfile/CDB/sysaux01.dbf' SIZE 500M
DEFAULT TEMPORARY TABLESPACE TEMP TEMPFILE
'/u01/dbfile/CDB/temp01.dbf' SIZE 500M
DEFAULT TABLESPACE USERS DATAFILE
'/u01/dbfile/CDB/users01.dbf' SIZE 50M
LOGFILE GROUP 1
('/u01/oraredo/CDB/redo01a.rdo') SIZE 50M,
GROUP 2
('/u01/oraredo/CDB/redo02a.rdo') SIZE 50M
USER sys    IDENTIFIED BY foo
USER system IDENTIFIED BY foo
USER_DATA TABLESPACE userstbs DATAFILE
'/u01/dbfile/CDB/userstbsp01.dbf' SIZE 500M
ENABLE PLUGGABLE DATABASE
SEED FILE_NAME_CONVERT = ('/u01/dbfile/CDB','/u01/dbfile/CDB/pdbseed');
```

`CREATE DATABASE` 语句中有几个子句仅与 PDB 相关。例如，`ENABLE PLUGGABLE DATABASE` 子句是必需的，因为您希望在 CDB 中创建 PDB。`USER_DATA TABLESPACE` 子句指定了在种子数据库中创建一个额外的表空间；这个表空间也将被复制到从种子数据库克隆的任何 PDB 中。此外，`SEED FILE_NAME_CONVERT` 指定了种子数据库文件的命名方式以及它们将位于的目录。

接下来，确保已创建参数文件和 `CREATE DATABASE` 语句中引用的任何目录：
```
$ mkdir -p /u01/dbfile/CDB/pdbseed
$ mkdir -p /u01/dbfile/CDB
$ mkdir -p /u01/oraredo/CDB
```

现在，以 `nomount` 模式启动数据库，并运行 `credb.sql` 脚本：
```
$ sqlplus / as sysdba
SQL> startup nomount;
SQL> @credb.sql
```

如果成功，您应该会看到：
```
Database created.
```

Oracle 建议您使用 `catcon.pl` Perl 脚本为 CDB 运行任何 Oracle 提供的 SQL 脚本。因此，要为 CDB 创建数据字典，请使用 `catcon.pl` Perl 脚本。首先切换到 `ORACLE_HOME/rdbms/admin` 目录：
```
$ cd $ORACLE_HOME/rdbms/admin
```

现在使用 `catcon.pl` 以 `SYS` 身份运行 `catalog.sql` 脚本：
```
$ catcon.pl -n 1 -U SYS -c 'catalog.sql'
```


## 验证 CDB 是否已创建

要验证数据库是否创建为 CDB，首先以 `SYS` 身份连接到根容器：

```bash
$ sqlplus / as sysdba
```

现在，你可以通过此查询确认 CDB 是否已成功创建。如果数据库创建为 CDB，`V$DATABASE` 的 `CDB` 列将包含 `YES` 值：

```sql
SQL> select name, cdb from v$database;
```

以下是一些示例输出：

```sql
NAME      CDB
--------- ---
CDB       YES
```

此时，你的 CDB 数据库中应该有两个容器：根容器和种子可插拔数据库。你可以使用此查询进行检查：

```sql
SQL> select con_id, name from v$containers;
```

以下是一些示例输出：

```sql
CON_ID NAME
------ --------------------
1 CDB$ROOT
2 PDB$SEED
```

你还应该有与根数据库和种子数据库关联的数据文件。你可以通过此查询查看与每个容器关联的数据文件：

```sql
SQL> select con_id, file_name from cdb_data_files order by 1;
```

以下是一些输出：

```sql
CON_ID FILE_NAME
------ -------------------------------------------------------
1 /u01/app/oracle/oradata/CDB/system01.dbf
1 /u01/app/oracle/oradata/CDB/sysaux01.dbf
1 /u01/app/oracle/oradata/CDB/undotbs01.dbf
1 /u01/app/oracle/oradata/CDB/users01.dbf
2 /u01/app/oracle/oradata/CDB/pdbseed/system01.dbf
2 /u01/app/oracle/oradata/CDB/pdbseed/sysaux01.dbf
```

请注意，如果你查询的是 `DBA_DATA_FILES` 而不是 `CDB_DATA_FILES`，你将只会看到与根容器（你当前连接的容器）关联的四个数据文件；例如，

```sql
SQL> select file_name from dba_data_files;
FILE_NAME
-------------------------------------------------------
/u01/app/oracle/oradata/CDB/system01.dbf
/u01/app/oracle/oradata/CDB/sysaux01.dbf
/u01/app/oracle/oradata/CDB/users01.dbf
/u01/app/oracle/oradata/CDB/undotbs01.dbf
```

## 管理根容器

在管理 CDB 时，大多数情况下，你是以 `SYS` 身份连接到根容器，并像操作非 CDB 数据库一样执行任务。然而，有几点需要特别注意，这些是维护 CDB 所特有的。以下任务只能在以 `SYSDBA` 权限连接到根容器时执行：

*   启动/停止实例
*   启用/禁用归档日志模式
*   管理影响 CDB 内所有数据库的实例设置，例如整体内存大小
*   备份和恢复数据库中的所有数据文件
*   管理控制文件（添加、恢复、删除等）
*   管理在线重做日志
*   管理根 `UNDO` 表空间
*   管理根 `TEMP` 表空间
*   创建公共用户和角色

以下各节将讨论这些主题。

### 连接到根容器

以 `SYS` 身份连接到根容器允许你执行通常与数据库管理相关的所有任务。你可以通过操作系统认证或网络连接从数据库服务器本地以 `SYS` 身份连接（网络连接需要监听器和密码文件）。建议为 CDB 管理员设置角色，以便能够使用个人账户，而不是通过 `SYS` 登录。只有在执行服务器任务时才需要在服务器上登录，否则出于安全和合规性考虑，应允许通过网络连接到 CDB。

#### 通过网络

如果你通过网络发起远程连接，那么你需要先在目标数据库服务器上设置一个监听器并创建一个密码文件（详见第 2 章）。一旦监听器和密码文件建立，你就可以通过网络远程连接，如下所示：

```bash
$ sqlplus user/pass@connection_string as sysdba
```

```sql
SQL> show user
USER is "user"
SQL> show con_id, conname
CON_ID
----------
1
CON_NAME
----------
CDB$ROOT
SQL> show user
USER is "user"
```

有关如何在插件式环境中实现监听器的详细信息，请参阅本章后面的“在 PDB 环境中管理监听器”部分。

### 显示当前连接的容器信息

从 `SQL*Plus` 出发，有几种简单的方法可以显示你当前连接的 CDB 的名称。此示例使用 `SHOW` 命令来显示容器 ID、名称和用户：

```sql
SQL> show con_id con_name user
```

你也可以通过 SQL 查询显示相同的信息：

```sql
SQL> SELECT SYS_CONTEXT('USERENV', 'CON_ID') AS con_id,
SYS_CONTEXT('USERENV', 'CON_NAME') AS cur_container,
SYS_CONTEXT('USERENV', 'SESSION_USER') AS cur_user
FROM DUAL;
```

以下是一些示例输出：

```sql
CON_ID               CUR_CONTAINER        CUR_USER
-------------------- -------------------- --------------------
1                    CDB$ROOT             USER
```

### 启动/停止根容器

你只能以特权用户身份连接到根容器时启动/停止 CDB。启动和停止根容器的过程与非 CDB 数据库相同。要启动 CDB，首先以 `SYS` 身份连接，并发出 `startup` 命令：

```bash
$ sqlplus / as sysdba
```

```sql
SQL> startup;
```

启动 CDB 数据库不会打开任何关联的 PDB。你可以使用此命令打开所有 PDB：

```sql
SQL> alter pluggable database all open;
```

要关闭 CDB 数据库，请发出以下命令：

```sql
SQL> shutdown immediate;
```

与非 CDB 数据库一样，上面的命令会关闭 CDB 实例并断开任何连接到数据库的用户。如果有任何可插拔数据库处于打开状态，它们将被关闭，用户将被断开连接。

### 创建公共用户

在插件式环境中有两种类型的用户：本地用户和公共用户。本地用户只不过是在 PDB 中创建的常规用户。PDB 中的本地用户类型的行为与非 CDB 环境中的用户相同。管理本地用户没什么特别之处。你可以像管理非 CDB 环境中的用户一样管理它们。

公共用户是 Oracle Database 12c 中的一个新概念，仅适用于插件式数据库环境。公共用户是存在于根容器和每个 PDB 中的用户。这种类型的用户必须最初在根容器中创建，并且会自动在所有现有 PDB 以及未来创建的任何 PDB 中创建。

提示
`SYS` 和 `SYSTEM` 账户是 Oracle 在插件式环境中自动创建的公共用户。

创建公共用户时，用户名必须以字符串 `C##` 或 `c##` 开头。例如，以下命令在所有 PDB 中创建一个公共用户：

```sql
SQL> create user c##dba identified by foo;
```


