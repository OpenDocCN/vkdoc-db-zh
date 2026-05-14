# 在 CDB 中创建可插拔数据库

创建 CDB 后，就可以开始在其内部创建 PDB。当你指示 Oracle 创建 PDB 时，其底层过程实际上是复制现有数据库（种子数据库、PDB 或非 CDB）的数据文件，然后使用新 PDB 的元数据实例化 CDB。这里的关键是正确指定你希望 Oracle 用作模板来创建新 PDB 的源数据库。

有多种工具可用于创建（克隆）PDB：即`CREATE PLUGGABLE DATABASE` SQL 语句、`DBCA`实用程序和 Enterprise Manager Cloud Control。本章重点介绍使用 SQL 和`DBCA`实用程序。如果你理解了如何使用 SQL 和`DBCA`创建 PDB，那么你应该能轻松地使用 Enterprise Manager 界面来实现相同的目标。

使用`CREATE PLUGGABLE DATABASE`语句时，你可以使用以下任何来源来创建 PDB：

*   种子数据库
*   现有 PDB（本地或远程）
*   非 CDB 数据库
*   已拔出的 PDB

使用`DBCA`时，你可以从以下任何来源创建 PDB：

*   种子数据库
*   RMAN 备份
*   已拔出的 PDB

在以下章节中，将介绍所有`CREATE PLUGGABLE DATABASE`变体来创建 PDB。对于`DBCA`，我仅展示如何从种子数据库创建 PDB。你应该能够根据你的各种需求修改该示例。

## 克隆种子数据库

`CREATE PLUGGABLE DATABASE`语句可用于通过复制种子数据库的数据文件来创建 PDB。为此，首先以`SYS`用户（或具有创建 PDB 权限的公共用户）连接到根容器数据库：

```
$ sqlplus sysuser/pass@CDB1 as sysdba
```

以下 SQL 语句创建一个名为`SALESPDB`的可插拔数据库：

```
SQL> CREATE PLUGGABLE DATABASE salespdb
ADMIN USER salesadm IDENTIFIED BY foo
FILE_NAME_CONVERT = ('/u01/app/oracle/oradata/CDB/pdbseed',
'/u01/app/oracle/oradata/CDB/salespdb');
```

运行前面的代码后，你应该会看到类似以下的输出：

```
Pluggable database created.
```

> **注意**
> 如果你使用的是`OMF`，在创建可插拔数据库时不需要指定`FILE_NAME_CONVERT`子句，因为 Oracle 会自动确定可插拔数据库数据文件的名称和位置。

请注意，此示例中的`FILE_NAME_CONVERT`子句有两个字符串。一个指定种子数据库数据文件的位置：

```
/u01/app/oracle/oradata/CDB/pdbseed
```

第二个字符串是你希望创建新 PDB 数据文件的位置：

```
/u01/app/oracle/oradata/CDB/salespdb
```

你需要将这些字符串修改为适合你环境的适当值。

使用`CREATE PLUGGABLE DATABASE`语句创建 PDB 时，有几个可用选项。表 22-2 总结了各种子句的含义。

**表 22-2 可插拔数据库创建选项**

| 参数 | 描述 |
| :--- | :--- |
| `ADMIN USER` | 创建的用于管理任务的本地用户。该用户被分配`PDB_DBA`角色。 |
| `MAXSIZE` | 可插拔数据库可消耗的最大存储空间；如果未指定，则对 PDB 可使用的存储量没有限制。 |
| `MAX_SHARED_TEMP_SIZE` | 连接到 PDB 的会话可以使用的共享临时表空间的最大量。 |
| `DEFAULT TABLESPACE` | 指定在可插拔数据库内创建的新用户的默认永久表空间。 |
| `DATAFILE` | 与默认表空间关联的数据文件的路径和文件名。 |
| `PATH_PREFIX` | 指定添加到可插拔数据库的任何新数据文件必须存在于此目录或其子目录中。 |
| `FILE_NAME_CONVERT` | 指定种子数据库数据文件的位置以及它们应被复制到的目标位置。 |

## 克隆现有 PDB

你可以从当前连接的（本地）CDB 中的现有 PDB 创建 PDB，也可以将远程 CDB 中的 PDB 复制为副本来创建 PDB。这两种技术将在接下来的两个小节中详细介绍。

### 本地

在此示例中，使用现有 PDB（`SALESPDB`）来创建新 PDB（`SALESPDB2`）。首先，连接到根容器，并将现有的源 PDB 置于只读模式：

```
$ sqlplus sysuser/pass@CDB1 as sysdba
SQL> alter pluggable database salespdb close;
SQL> alter pluggable database salespdb open read only;
```

现在，运行以下 SQL 以创建新的 PDB：

```
SQL> CREATE PLUGGABLE DATABASE salespdb2
FROM salespdb
FILE_NAME_CONVERT = ('/u01/app/oracle/oradata/CDB/salespdb',
'/u01/dbfile/CDB/salespb2')
STORAGE (MAXSIZE 6G MAX_SHARED_TEMP_SIZE 100M);
```

在前面的示例中，位于`/u01/app/oracle/oradata/CDB/salespdb`目录中与`SALESPDB`关联的数据文件被用来在`/u01/dbfile/CDB/salespdb2`目录中创建数据文件。如果目标目录事先不存在，系统会为你创建。你还可以对克隆的 PDB 指定管理限制，例如将其最大大小限制为 6GB，以及限制其在共享临时表空间中可消耗的共享资源最大量为 100MB。

### 远程

你也可以将远程 PDB 克隆为副本以创建 PDB。首先，你需要创建一个从 CDB 到将用作克隆源的 PDB 的数据库链接。本地用户和在数据库链接中指定的用户都必须具有`CREATE PLUGGABLE DATABASE`权限。

此示例显示以`SYS`身份本地连接到根容器。这是将在其中创建新 PDB 的数据库：

```
$ sqlplus sysuser/pass@CDB1 as sysdba
```

在此数据库中，创建一个到远程 CDB 中 PDB 的数据库链接。远程 CDB 包含一个名为`SALESPDB`的 PDB，其中有一个用户已被授予`CREATE PLUGGABLE DATABASE`权限。该用户将用于数据库链接：

```
create database link salespdb
connect to mv_maint identified by foo
using 'speed2:1521/salespdb';
```

接下来，连接到包含将被克隆的 PDB 的远程数据库：

```
$ sqlplus sysuser/foo@speed2:1521/salespdb as sysdba
```

关闭 PDB 并以只读模式打开它：

```
SQL> alter pluggable database salespdb close;
SQL> alter pluggable database salespdb open read only;
```

现在，以`SYS`身份连接到目标 CDB，并如所示克隆远程 PDB 来创建新的 PDB：

```
$ sqlplus sysuser/pass@CDB1 as sysdba
SQL> CREATE PLUGGABLE DATABASE salespdb3
FROM salespdb@salespdb
FILE_NAME_CONVERT = ('/u01/app/oracle/oradata/CDB/salespdb',
'/u01/dbfile/CDB2/salespdb3');
```

## 从非 CDB 数据库克隆

有三种方式可以从现有的非 CDB 创建 PDB：

*   使用`DBMS_PDB`包生成元数据，然后使用`CREATE PLUGGABLE DATABASE` SQL 语句创建 PDB。
*   数据泵（使用可传输表空间功能）。
*   GoldenGate 复制。

以下示例使用`DBMS_PDB`包从非 CDB 创建 PDB。有关数据泵和 GoldenGate 的详细信息，请分别参阅 Oracle 数据库实用程序指南和 GoldenGate 特定文档，这些文档可从 Oracle 网站（ [`http://otn.oracle.com`](http://otn.oracle.com) ）的技术网络区域获取。

> **注意**
> 使用`DBMS_PDB`包将非 CDB 转换为 PDB 时，非 CDB 必须是 Oracle 12c 或更高版本。

首先，将非 CDB 置于只读模式：

```
SQL> startup mount;
SQL> alter database open read only;
```

然后，运行`DBMS_PDB`包以创建描述非 CDB 数据库结构的 XML 文件：

```
BEGIN
DBMS_PDB.DESCRIBE(pdb_descr_file => '/orahome/oracle/ncdb.xml');
END;
/
```

创建 XML 文件后，关闭非 CDB 数据库：

```
SQL> shutdown immediate;
```


## 将非 CDB 转换为 PDB

### 准备环境并检查兼容性

接下来，设置您的 Oracle OS 变量（如 `ORACLE_SID` 和 `ORACLE_HOME`），然后连接到将容纳非 CDB 作为 PDB 的 CDB 数据库：

```
$ sqlplus / as sysdba
```

现在，您可以选择检查非 CDB 是否与它将插入的 CDB 兼容。运行此代码时，请提供先前创建的 XML 文件的目录和名称：

```
SQL> SET SERVEROUTPUT ON
SQL> DECLARE
hold_var boolean;
begin
hold_var := DBMS_PDB.CHECK_PLUG_COMPATIBILITY(pdb_descr_file=>'/orahome/oracle/ncdb.xml');
if hold_var then
dbms_output.put_line('YES');
else
dbms_output.put_line('NO');
end if;
end;
/
```

如果没有兼容性问题，上述代码将显示 `YES`；如果 PDB 不兼容，则会显示 `NO`。您可以查询 `PDB_PLUG_IN_VIOLATIONS` 视图以获取 PDB 与 CDB 不兼容的详细原因。

### 创建 PDB

接下来，使用以下 SQL 从非 CDB 创建 PDB。您必须指定详细信息，例如先前创建的 XML 文件的名称和位置、非 CDB 数据文件的位置以及您希望创建新数据文件的位置：

```
SQL> CREATE PLUGGABLE DATABASE dkpdb
USING '/orahome/oracle/ncdb.xml'
COPY
FILE_NAME_CONVERT = ('/u01/dbfile/dk/',
'/u01/dbfile/CDB/dkpdb/');
```

如果成功，您应该会看到：

```
Pluggable database created.
```

现在，以 `SYS` 用户身份连接到新创建的 PDB：

```
$ sqlplus sys/foo@'speed2:1521/dkpdb' as sysdba
```

作为最后一步，运行以下脚本：

```
SQL> @?/rdbms/admin/noncdb_to_pdb.sql
```

您现在应该可以打开 PDB 并开始使用它。

## 从 CDB 中拔出 PDB

在将 PDB 插入另一个 CDB 之前，必须先将其拔出。拔出意味着将 PDB 与 CDB 解除关联，并生成一个描述被拔出 PDB 的 XML 文件。此 XML 文件将来可用于将 PDB 插入另一个 CDB。

以下是拔出 PDB 所需的步骤：

1.  关闭 PDB（这会将其打开模式更改为 `MOUNTED`）
2.  通过 `ALTER PLUGGABLE DATABASE ... UNPLUG` 命令拔出可插拔数据库

首先，以 `SYS` 用户身份连接到根容器，然后关闭 PDB：

```
$ sqlplus sysuser/pass@CDB1 as sysdba
SQL> alter pluggable database dkpdb close immediate;
```

接下来，拔出 PDB。请确保为您环境中 XML 文件的位置指定一个存在的目录：

```
SQL> alter pluggable database dkpdb unplug into
'/orahome/oracle/dba/dkpdb.xml';
```

XML 文件包含有关 PDB 的元数据，例如其数据文件。如果您想将 PDB 插入另一个 CDB，则需要此 XML 文件。

> **注意**
> 一旦 PDB 被拔出，在可以将其插回原始 CDB 之前，必须先将其删除。

## 将拔出的 PDB 插入 CDB

### 检查兼容性

在 PDB 可以插入 CDB 之前，它必须在数据文件字节顺序和已安装的兼容数据库选项方面与 CDB 兼容。从 18c 开始，PDB 中的字符集可以与源 PDB 不同。您可以通过 `DBMS_PDB` 包来验证兼容性。您必须向该包提供 PDB 被拔出时创建的 XML 文件的目录和名称作为输入。以下是一个示例：

```
SQL> SET SERVEROUTPUT ON
SQL> DECLARE
hold_var boolean;
begin
hold_var := DBMS_PDB.CHECK_PLUG_COMPATIBILITY(pdb_descr_file=>'/orahome/oracle/dba/dkpdb.xml');
if hold_var then
dbms_output.put_line('YES');
else
dbms_output.put_line('NO');
end if;
end;
/
```

如果没有兼容性问题，上述代码将显示 `YES`；如果 PDB 不兼容，则会显示 `NO`。您可以查询 `PDB_PLUG_IN_VIOLATIONS` 视图以获取 PDB 与 CDB 不兼容的详细原因。

### 插入 PDB

插入 PDB 使用 `CREATE PLUGGABLE DATABASE` 命令。当您将 PDB 插入 CDB 时，必须提供一些关键信息，使用以下两个子句：

*   `USING` 子句：指定 PDB 被拔出时创建的 XML 文件的位置。
*   `COPY FILE_NAME_CONVERT` 子句：指定 PDB 数据文件的源以及 PDB 数据文件将在目标 CDB 内创建的位置。

要插入 PDB，请以特权用户身份连接到 CDB，然后运行以下命令：

```
SQL> CREATE PLUGGABLE DATABASE dkpdb
USING '/orahome/oracle/dba/dkpdb.xml'
COPY
FILE_NAME_CONVERT = ('/u01/app/oracle/oradata/CDB1/dkpdb',
'/u01/dbfile/CDB2/dkpdb');
```

您现在可以打开 PDB 并开始使用它。

## 使用 DBCA 从种子库创建 PDB

您可以使用 DBCA 实用程序通过指定 `-createPDBFrom DEFAULT` 子句从种子数据库创建 PDB。以下示例在名为 `CDB` 的 CDB 中创建一个名为 `HRPDB` 的可插拔数据库：

```
$ dbca -silent -createPluggableDatabase -sourceDB CDB -pdbName hrpdb
-createPDBFrom DEFAULT
-pdbAdminUserName adminplug -pdbAdminPassword foo
-pdbDatafileDestination /u01/dbfile/CDB/hrpdb
```

上面的代码行必须在一行中输入。此外，如果 PDB 目标目录不存在，DBCA 将自动创建它。随着命令的进行，您应该会看到类似于以下内容的输出：

```
Creating Pluggable Database
4% complete
12% complete
...
Completing Pluggable Database Creation
100% complete
Look at the log file "/orahome/app/oracle/cfgtoollogs/dbca/CDB.log" for further details.
```

作为最后一步，您应该检查日志文件并确保 PDB 的创建没有问题。

## 检查可插拔数据库的状态

创建 PDB 后，您可能需要检查其状态。连接到根容器时，您可以查看 CDB 中所有 PDB 的状态。例如，具有 DBA 权限的用户可以通过此查询报告所有 PDB 的状态：

```
SQL> select pdb_id, pdb_name, status from cdb_pdbs;
```

以下是一些示例输出：

```
PDB_ID PDB_NAME             STATUS
---------- -------------------- -------------
2 PDB$SEED             NORMAL
3 SALESPDB             NORMAL
4 HRPDB                NORMAL
```

下一个查询报告可插拔数据库是否已打开：

```
SQL> select con_id, name, open_mode from v$pdbs;
```

以下是一些示例输出：

```
CON_ID NAME                           OPEN_MODE
---------- ------------------------------ ----------
2 PDB$SEED                       READ ONLY
3 SALESPDB                       READ WRITE
4 HRPDB                          READ WRITE
```

如果您直接连接到 PDB 时运行上述查询，则 `CDB_PDBS` 中不会显示任何信息。此外，`V$PDBS` 将仅显示当前连接的 PDB 的信息。

## 管理可插拔数据库

您可以直接连接到 PDB 执行许多数据库管理任务。您可以打开/关闭 PDB、检查其状态、显示当前连接的用户等等。您可以以特权连接到根容器来管理可插拔数据库，也可以以特权用户身份直接连接到 PDB 本身执行任务。

请记住，当您以 `SYS` 用户身份连接到 CDB 中的 PDB 时，您只能对所连接的 PDB 执行 `SYS` 特权操作。您无法启动/停止容器实例或查看与 CDB 中其他 PDB 相关的数据字典信息。职责分离可以让一个团队或 DBA 仅管理一个或多个 PDB，而另一个团队或 DBA 负责 CDB 管理。

### 连接到 PDB

您可以以 `SYS` 用户身份本地或通过网络连接到 PDB。
要进行本地连接，首先以具有 PDB 权限的普通用户身份连接到根容器，然后使用 `SET CONTAINER` 命令连接到所需的 PDB：

```
SQL> alter session set container=salespdb;
```


先前的连接不需要监听器或密码文件；而通过网络的连接则两者都需要。以下示例通过 SQL*Plus 建立网络连接，并在连接时指定 PDB 的主机、监听器端口和服务名：

```bash
$ sqlplus pdbsys/foo@speed2:1521/salespdb as sysdba
```

如果你不确定如何设置监听器和密码文件，请参阅第 2 章。如果你使用 DBCA 实用程序创建 PDB，其监听器将被自动设置。关于如何向监听器注册 PDB 服务名的说明，请参见下一节。

## 在 PDB 环境中管理监听器

回顾第 2 章可知，监听器是支持远程网络连接到数据库的进程。大多数数据库环境都需要一个监听器才能运行。当客户端尝试连接到远程数据库时，它会提供三条关键信息：监听器所在的主机、监听器正在监听的主机端口以及数据库服务名。

每个数据库都分配有一个或多个服务名。默认情况下，通常会有一个源自数据库唯一名称和域名的服务名。你可以手动为数据库创建一个或多个服务名。DBA 有时会创建多个服务，以便可以针对每个服务控制或监控资源使用情况。例如，可能为销售应用创建一个服务，为人力资源应用创建另一个服务。每个应用通过其服务名连接到数据库。该服务的连接信息会出现在每个会话的 `V$SESSION` 视图的 `SERVICE_NAME` 列中。

如果你在没有 `listener.ora` 文件的情况下启动默认监听器，PMON 后台进程会自动将任何数据库（包括任何可插拔数据库）作为服务进行注册：

```bash
$ lsnrctl start
```

最终，你应该能看到数据库（包括任何 PDB）已注册到默认监听器。

注意：启动监听器时，如果存在 `listener.ora` 文件，监听器将尝试静态注册该文件中出现的所有服务名。

默认情况下，PDB 使用与 PDB 名相同的服务名进行注册。这个默认服务通常是你用来建立连接的服务：

```bash
$ sqlplus pdbsys/foo@speed2:1521/salespdb as sysdba
```

你可以通过以 `SYS` 身份连接到根容器并查询来验证哪些服务正在运行：

```sql
SQL> select name, network_name, pdb from v$services order by pdb, name;
```

你也可以通过 `lsnrctl` 实用程序验证监听器正在监听哪些服务：

```bash
$ lsnrctl services
```

Oracle 建议你为任何需要访问 PDB 的应用程序配置一个额外的服务（除了默认服务）。你可以通过 `SRVCTL` 实用程序或 `DBMS_SERVICE` 包手动配置服务。此示例展示了如何通过 `DBMS_SERVICE` 包配置一个服务。首先，通过默认服务以 `SYS` 身份连接到你想要在其中创建服务的 PDB：

```bash
$ sqlplus pdbsys/foo@speed2:1521/salespdb as sysdba
```

确保 PDB 处于读写模式打开状态：

```sql
SQL> SELECT con_id, name, open_mode FROM v$pdbs;
```

接下来，创建一个服务。此代码创建并启动了一个名为 `SALESWEST` 的服务：

```sql
SQL> exec DBMS_SERVICE.CREATE_SERVICE(service_name => 'SALESWEST', network_name => 'SALESWEST');
SQL> exec DBMS_SERVICE.START_SERVICE(service_name => 'SALESWEST');
```

现在，应用用户可以通过该服务连接到 `SALESPDB` 可插拔数据库：

```bash
$ sqlplus appuser/pass@speed2:1521/saleswest
```

注意：如果你的服务器上有多个 CDB 数据库，请确保 PDB 服务名在该服务器上的所有 CDB 数据库中是唯一的。不建议将两个同名的 PDB 数据库注册到一个公共监听器。这将导致混淆你实际连接到的是哪个 PDB。

## 显示当前连接的 PDB

从 SQL*Plus 中，有几种简单的方法可以显示你当前连接的 PDB 的名称。此示例使用 `SHOW` 命令来显示容器 ID、名称和用户：

```sql
SQL> show con_id con_name user
```

这是一些示例输出：

```text
CON_ID
---------
3
CON_NAME
---------
SALESPDB
USER is "SYS"
```

你也可以通过 SQL 查询显示相同的信息：

```sql
SELECT SYS_CONTEXT('USERENV', 'CON_ID') AS con_id,
SYS_CONTEXT('USERENV', 'CON_NAME') AS cur_container,
SYS_CONTEXT('USERENV', 'SESSION_USER') AS cur_user
FROM DUAL;
```

这是一些示例输出：

```text
CON_ID     CUR_CONTAINER   CUR_USER
---------- --------------- ----------
3          SALESPDB        SYS
```

请记住，`SYS_CONTEXT` 函数可用于显示其他有用的信息，例如 `SERVICE_NAME`、`DB_UNIQUE_NAME`、`INSTANCE_NAME` 和 `SERVER_HOST`；例如：

```sql
SELECT
SYS_CONTEXT('USERENV', 'SERVICE_NAME') as service_name,
SYS_CONTEXT('USERENV', 'DB_UNIQUE_NAME') as db_unique_name,
SYS_CONTEXT('USERENV', 'INSTANCE_NAME') as instance_name,
SYS_CONTEXT('USERENV', 'SERVER_HOST') as server_host
from dual ;
```

## 启动/停止 PDB

当你启动/停止一个 PDB 时，你并不是在启动/停止一个实例。相反，你是在使 PDB 可用或不可用，打开或关闭。你可以从以 `SYS` 身份连接到根容器或直接以 `SYS` 身份连接到 PDB 来更改 PDB 的打开模式。

### 从根容器

要从根容器更改 PDB 的打开模式，请按如下方式操作：

```sql
SQL> alter pluggable database salespdb open;
```

你也可以以特定状态启动可插拔数据库，例如只读：

```sql
SQL> startup pluggable database salespdb open read only;
```

要关闭 PDB，你可以指定 PDB 的名称：

```sql
SQL> alter pluggable database salespdb close immediate;
```

你也可以在连接到根容器时打开或关闭所有可插拔数据库。在实际的生产系统中，不建议这样做，因为这些可能都是不同的应用程序，除非正在进行服务器维护，否则可能没有理由关闭所有 PDB。

```sql
SQL> alter pluggable database all open;
SQL> alter pluggable database all close immediate ;
```

### 从可插拔数据库

要打开/启动可插拔数据库，以 `SYS` 身份连接到该可插拔数据库：

```bash
$ sqlplus pdbsys/foo@salespdb as sysdba
SQL> startup;
```

要关闭数据库，请发出以下命令：

```sql
SQL> shutdown immediate;
```

## 修改特定于 PDB 的初始化参数

Oracle 允许在以特权用户身份连接到 PDB 时修改某些初始化参数。你可以通过以下查询查看这些参数：

```sql
SQL> SELECT name
FROM v$parameter
WHERE ispdb_modifiable='TRUE'
ORDER BY name;
```

这是输出的一小部分：

```text
NAME
--------------------------------
sort_area_size
sql_trace
sqltune_category
star_transformation_enabled
statistics_level
```

当你直接连接到 PDB 并进行初始化参数更改时，这些更改仅影响当前连接的 PDB。参数更改不会影响根容器或其他 PDB。例如，假设你想更改 `OPEN_CURSORS` 的值。首先，以特权用户身份直接连接到 PDB，并发出 `ALTER SYSTEM` 语句：

```bash
$ sqlplus pdbsys/foo@speed2:1521/salespdb as sysdba
SQL> alter system set open_cursors=100;
```

先前的修改仅针对 `SALESPDB` PDB 更改了 `OPEN_CURSORS` 的值。此外，`SALESPDB` 的 `OPEN_CURSORS` 设置将在数据库重启后持续有效。

## 重命名 PDB


