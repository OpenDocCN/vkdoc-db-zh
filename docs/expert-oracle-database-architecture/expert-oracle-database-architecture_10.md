# 3. 文件

在本章中，我们将研究构成数据库和实例的八种主要文件类型。与实例相关的文件很简单：

*   **参数文件**：这些文件告诉 Oracle 实例在哪里可以找到控制文件，它们还指定了某些初始化参数，用于定义某些内存结构的大小等等。我们将探讨存储数据库参数文件的两种可用选项。

*   **跟踪文件**：这些是由服务器进程创建的诊断文件，通常是在响应某些异常错误条件时生成的。

*   **告警文件**：这些文件与跟踪文件类似，但它们包含关于“预期”事件的信息，并且还以单一、集中的文件形式向 DBA 告警许多数据库事件。

构成数据库的文件是：

*   **数据文件**：这些是数据库使用的文件；它们存储你的表、索引以及所有其他数据段类型。

*   **临时文件**：这些文件用于基于磁盘的排序和临时存储。

*   **控制文件**：这些文件告诉你数据文件、临时文件和重做日志文件的位置，以及有关它们状态的其他相关元数据。它们还包含由 `RMAN`（Recovery Manager，备份与恢复工具）维护的备份信息。

*   **重做日志文件**：这些是你的事务日志。

*   **口令文件**：这些文件用于验证通过网络执行管理活动的用户。我们将不会详细讨论这些文件。如果你想通过网络远程连接到实例并以 `SYS` 用户身份登录，则需要这些文件。这些文件在 `Data Guard` 配置（主数据库和备用数据库）中也是必需的。

Oracle 使用几种可选的文件类型来加快备份和恢复操作。这两种文件是：

*   **更改跟踪文件**：此文件有助于实现 Oracle 数据的真正增量备份。它不必位于快速恢复区中，但由于它纯粹与数据库备份和恢复相关，我们将在该区域的背景下进行讨论。

*   **闪回日志文件**：这些文件存储数据库块的“前映像”，以便于执行 `FLASHBACK DATABASE` 命令。

我们还将看看通常与数据库关联的其他类型的文件，例如：

*   **数据泵文件**：这些文件由 Oracle Data Pump Export 进程生成，并由 Data Pump Import 进程使用。这种文件格式也可以由外部表创建和使用。

*   **平面文件**：这些是你可以在文本编辑器中查看的普通旧文件。你通常使用它们将数据加载到数据库中。

这些列表中最重要的文件是数据文件和重做日志文件，因为它们包含了你辛勤积累的数据。我可以丢失任何和所有其余文件，仍然能够访问我的数据。如果我丢失了重做日志文件，我可能开始丢失一些数据。如果我丢失了数据文件及其所有备份，我无疑将永远失去那些数据。

我们现在将看看这些文件的类型、它们通常的位置、命名方式以及我们可能期望在其中找到的内容。

## 参数文件

与 Oracle 数据库关联的参数文件有很多种，从客户端工作站上的 `tnsnames.ora` 文件（用于“找到”网络上的服务器），到服务器上的 `listener.ora` 文件（用于网络监听器启动），再到 `sqlnet.ora`、`cman.ora` 和 `ldap.ora` 文件等等。然而，最重要的参数文件是数据库的参数文件——没有它，我们甚至无法启动实例。其余的文件很重要；它们都与网络和连接到数据库相关。但是，它们超出了我们讨论的范围。关于它们的配置和设置信息，我建议你参考 *Database Net Services Administrator’s Guide*。这些文件通常由 DBA（而非开发人员）设置。

数据库的参数文件通常称为 init 文件，或 `init.ora` 文件。这是由于其历史上的默认名称是 `init<ORACLE_SID>.ora`。我称其为“历史上的”是因为 Oracle 此后实现了一种极大改进的存储数据库参数设置的方法：服务器参数文件，或简称 `SPFILE`。该文件的默认名称是 `spfile<ORACLE_SID>.ora`。我们将分别看看这两种参数文件。

注意

对于那些不熟悉 `SID` 或 `ORACLE_SID` 术语的人来说，需要给出一个完整的定义。`SID` 是一个*站点标识符*。在 UNIX/Linux 中，它和 `ORACLE_HOME`（Oracle 软件安装的位置）一起被哈希处理，以创建一个用于创建或附加共享全局区域（`SGA`）内存区域的唯一键名。如果你的 `ORACLE_SID` 或 `ORACLE_HOME` 设置不正确，并且你使用的是本地（而非基于网络的）连接，你将得到 `ORACLE NOT AVAILABLE` 错误，因为你无法附加到由此唯一键标识的共享内存段。在 Windows 上，共享内存的使用方式与 UNIX/Linux 不同，但 `SID` 仍然很重要。你可以在同一个 `ORACLE_HOME` 下拥有多个数据库，因此你需要一种方法来唯一标识与每个数据库关联的实例及其配置文件。

没有参数文件，你就无法启动 Oracle 数据库。这使得参数文件相当重要，Oracle 的备份和恢复工具 Recovery Manager (`RMAN`) 认识到这个文件的重要性，并允许你将服务器参数文件（但不是旧式的 `init.ora` 参数文件类型）包含在你的备份集中。然而，由于 `init.ora` 文件只是一个纯文本文件，你可以用任何文本编辑器创建它，因此它并不是一个你必须用生命守护的文件。你可以重新创建它，只要你知道里面曾经有什么（例如，如果你有权访问该数据库的告警日志，你可以从中检索该信息，并重建你的整个 `init.ora` 参数文件）。

我们现在将依次研究两种类型的数据库启动参数文件（`init.ora` 和 `SPFILE`），但在此之前，让我们先看看数据库参数文件是什么样子。


### 什么是参数？

简单来说，数据库参数可以被视为一个键/值对。要查看实例参数的当前值，可以查询 `V$` 视图 `V$PARAMETER`。或者，在 `SQL*Plus` 中，你可以使用 `SHOW PARAMETER` 命令，例如：

```bash
$ sqlplus / as sysdba
SQL> select value from v$parameter where name = 'db_block_size';
VALUE

SQL> show parameter db_block_s
NAME                                 TYPE        VALUE
------------------------------------ ----------- -------
db_block_size                        integer     8192
```

两种输出显示的基本信息相同，尽管从 `V$PARAMETER` 可以获取更多信息（比此示例中显示的列要多得多）。但 `SHOW PARAMETER` 在易用性上更胜一筹，并且它能自动进行“通配符匹配”。注意，我只输入了 `db_block_s`；`SHOW PARAMETER` 会自动在前后添加 `%`。

**注意**

所有 `V$` 视图和所有数据字典视图在 *Oracle Database Reference* 手册中都有完整文档记录。请将该手册视为特定视图中可用内容的权威来源。

如果你以权限较低的用户执行前面的示例（本书中 `EODA` 已被授予 `DBA` 角色），你将看到如下内容：

```sql
SQL> connect scott/tiger@PDB1
Connected.
SQL> select value from v$parameter where name = 'db_block_size';
select value from v$parameter where name = 'db_block_size'
*
ERROR at line 1:
ORA-00942: table or view does not exist
SQL> show parameter db_block_s
ORA-00942: table or view does not exist
```

默认情况下，“普通”账户未被授予访问 `V$` 性能视图的权限。你必须将 `V$PARAMETER` 上的 `select` 权限授予 `SCOTT`。以 `DBA` 账户执行以下操作：

```sql
$ sqlplus / as sysdba
SQL> alter session set container=PDB1;
SQL> grant select on sys.v_$parameter to scott;
```

如果你重新连接回 `SCOTT` 架构，现在应该能够查看参数了。值得一提的是，还有第三种方式可以通过 `DBMS_UTILITY` 包查看数据库参数（你仍然需要内部视图上的 `select` 权限才能通过此方式检索参数值）。示例如下：

```sql
SQL> conn scott/tiger@PDB1
SQL> create or replace
function get_param( p_name in varchar2 )
return varchar2
as
l_param_type  number;
l_intval      binary_integer;
l_strval      varchar2(256);
invalid_parameter exception;
pragma exception_init( invalid_parameter, -20000 );
begin
begin
l_param_type :=
dbms_utility.get_parameter_value
( parnam => p_name,
intval => l_intval,
strval => l_strval );
exception
when invalid_parameter
then
return '*access denied*';
end;
if ( l_param_type = 0 )
then
l_strval := to_char(l_intval);
end if;
return l_strval;
end get_param;
/
Function created.
```

如果你在 `SQL*Plus` 中执行此函数，将会看到：

```sql
SQL> set serverout on
SQL> exec dbms_output.put_line( get_param( 'db_block_size' ) );

PL/SQL procedure successfully completed.
```

并非每个参数都能通过 `dbms_utility.get_parameter_value` API 调用获取。具体来说，内存相关参数（如 `sga_max_size`、`db_cache_size`、`pga_aggregate_target` 等）是不可见的。我们在代码的第 17 到 21 行处理了这种情况——当遇到不允许查看的参数时，我们返回 `'*access denied*'`。如果你对整个受限参数列表感到好奇，你可以（任何被授予此函数 `EXECUTE` 权限的账户都可以）执行以下查询：

```sql
$ sqlplus scott/tiger@PDB1
SQL> select name, scott.get_param( name ) val
from v$parameter
where scott.get_param( name ) = '*access denied*';
NAME                           VAL
------------------------------ --------------------
sga_max_size                   *access denied*
shared_pool_size               *access denied*
large_pool_size                *access denied*
java_pool_size                 *access denied*
streams_pool_size              *access denied*
...
client_result_cache_lag        *access denied*
olap_page_pool_size            *access denied*
33 rows selected.
```

**注意**

在不同的版本上，此查询的结果会不同。随着参数数量的变化，你应该预期无法访问的参数的数量和值会随时间推移而增减。

参数的数量（及其名称）因版本而异。大多数参数（如 `db_block_size`）的生命周期非常长（它们不会因版本更迭而消失），但随着时间的推移，许多其他参数随着实现方式的变化而变得过时。例如，曾经有一个 `distributed_transactions` 参数，可以设置为某个正整数，用于控制数据库可以执行的并发分布式事务数。它在之前的版本中可用，但在任何近期的 Oracle 版本中都找不到。事实上，在后续版本中尝试使用该参数会引发错误。例如：

```sql
$ sqlplus / as sysdba
SQL> alter system set distributed_transactions = 10;
alter system set distributed_transactions = 10
*
ERROR at line 1:
ORA-25138: DISTRIBUTED_TRANSACTIONS initialization parameter has been made
obsolete
```

如果你想查看参数并了解哪些参数可用以及每个参数的作用，请参阅 *Oracle Database Reference* 手册。该手册的第一章详细检查了每个已记录的参数。总体而言，分配给每个参数的默认值（或者对于从其他参数获取默认设置的参数而言，其派生值）对于大多数系统来说已经足够。通常，某些参数的值（例如 `control_files` 参数（指定系统上控制文件的位置）、`db_block_size`、各种内存相关参数等）需要为每个数据库唯一设置。

请注意我在上一段中使用了“已记录的”这个术语。同样也存在未记录的参数。你可以通过它们的名称以下划线 (`_`) 开头来识别这些参数。关于这些参数有许多猜测。由于它们未被记录，有些人认为它们必定是“神奇的”，并且许多人假设它们为 Oracle 内部人员所熟知和使用。事实上，我发现情况恰恰相反。它们并不广为人知，也很少被使用。实际上，这些未记录参数中的大多数相当乏味，因为它们代表已弃用的功能和向后兼容性标志。另一些则有助于数据恢复，而不是数据库本身的恢复；例如，其中一些参数使数据库能在某些极端情况下启动，但仅够提取数据。之后你必须重建数据库。

除非 Oracle 技术支持如此指导你，否则没有理由在配置中使用未记录参数。许多参数具有可能造成灾难性后果的副作用。在我的生产数据库中，我不希望使用任何未记录的设置。

**警告**

仅在 Oracle 技术支持要求时使用未记录参数。它们的使用可能损害数据库，并且它们的实现方式可能会——也将会——随版本更迭而改变。

你可以通过以下两种方式之一设置各种参数值：仅用于当前实例，或是持久设置。确保参数文件包含你想要的值是你的责任。当使用传统的 `init.ora` 参数文件时，这是一个手动过程。要持久地更改参数值，使新设置在服务器重启后依然有效，你必须手动编辑和修改 `init.ora` 参数文件。而对于服务器参数文件，你会发现通过一条命令，这一切或多或少已经为你实现了自动化。

## 旧版 init.ora 参数文件

旧版的 `init.ora` 文件在结构上非常简单。它是一系列的变量键/值对。一个示例的 `init.ora` 文件可能如下所示：

```
control_files='/opt/oracle/oradata/CDB/control01.ctl'
db_block_size=8192
db_name='CDB'
```

事实上，这已经非常接近现实中你能使用的最基本的 `init.ora` 文件了，不过，如果我使用的块大小是我的平台上的默认值（默认块大小因平台而异），我就可以移除那个参数。参数文件至少用于获取数据库名称和控制文件的位置。控制文件告知 Oracle 其他每个文件的位置，因此对于启动实例的“引导”过程至关重要。

现在你了解了这些旧版数据库参数文件是什么，以及去哪里获取有关可设置有效参数的更多细节，你还需要知道在磁盘上的什么位置可以找到它们。该文件的默认命名约定是

```
init$ORACLE_SID.ora    (UNIX/Linux 环境变量)
init%ORACLE_SID%.ora   (Windows 环境变量)
```

并且默认会在以下位置找到：

```
$ORACLE_HOME/dbs       (UNIX/Linux)
%ORACLE_HOME%\DATABASE (Windows)
```

有趣的是，在许多情况下，你会发现这个参数文件的完整内容是这样的：

```
IFILE= /some/path/to/somewhere/init.ora'
```

`IFILE` 指令的工作方式类似于 C 语言中的 `#include file`。它将指定文件的内容包含到当前文件中。在这里，这个指令包含了一个来自非默认位置的 `init.ora` 文件。

应该指出的是，参数文件不必位于任何特定位置。启动实例时，你可以在启动命令中使用 `pfile=filename` 选项。当你想在你的数据库上尝试不同的 `init.ora` 参数以查看不同设置的效果时，这非常有用。

旧版参数文件可以通过任何文本编辑器进行维护。例如，在 UNIX/Linux 上，我会使用 `vi`；在众多 Windows 操作系统版本上，我会使用 `Notepad`；而在大型机上，我可能会使用 `Xedit`。重要的是要注意，你完全负责编辑和维护此文件。Oracle 数据库本身没有任何命令可用于维护 `init.ora` 文件中的值。例如，当你使用 `init.ora` 参数文件时，发出 `ALTER SYSTEM` 命令来更改 SGA 组件的大小，这不会在该文件中体现为永久更改。如果你想使该更改永久生效——换句话说，如果你希望它成为后续数据库重启的默认值——那么你需要确保手动更新所有可能用于启动此数据库的 `init.ora` 参数文件。

最后一个值得注意的有趣点是，旧版参数文件不一定位于数据库服务器上。引入服务器参数文件（我们稍后讨论）的原因之一就是为了弥补这种情况。旧版参数文件必须存在于尝试启动数据库的客户端机器上，这意味着如果你运行的是 UNIX/Linux 服务器，但通过在你的 Windows 桌面机器上安装的 `SQL*Plus` 通过网络进行管理，那么你需要在桌面上有该数据库的参数文件。

我仍然记得我是如何痛苦地发现参数文件并不存储在服务器上的。那是很多年前，当时引入了一个全新的（现已退役）名为 `SQL*DBA` 的工具。这个工具允许我们执行远程操作，特别是远程管理操作。我从我的服务器（当时运行 SunOS）能够远程连接到一台大型机数据库服务器。我也能够发出关闭命令。然而，就在那时我意识到我有点陷入困境——当我试图启动实例时，`SQL*DBA` 会抱怨找不到参数文件。我了解到这些参数文件——即 `init.ora` 纯文本文件——位于带有客户端的机器上；它们必须存在于*客户端*机器上——而*不是*在服务器上。`SQL*DBA` 在我的本地系统上寻找参数文件来启动大型机数据库。我不仅没有任何这样的文件，而且我也不知道要在文件里放些什么才能让系统重新启动！我不知道 `db_name` 或控制文件位置（即使只是弄清楚大型机文件的正确命名约定都有点困难），而且我也无法登录到大型机系统本身。从那以后我再也没有犯过同样的错误；这是一个痛苦的教训。

当 DBA 意识到 `init.ora` 参数文件必须驻留在启动数据库的客户端机器上时，导致了这些文件的激增。每个 DBA 都想从他们的桌面运行管理工具，因此每个 DBA 都需要在他们的桌面机器上有一份参数文件副本。像 Oracle Enterprise Manager (`OEM`) 这样的工具会增加又一个参数文件。这些工具试图在一台单一的机器上集中管理企业中的所有数据库，这台机器有时被称为*管理服务器*。这台单一机器将运行所有 DBA 用来启动、关闭、备份和以其他方式管理数据库的软件。这听起来是一个完美的解决方案：将所有参数文件集中在一个位置，并使用 GUI 工具执行所有操作。但现实情况是，有时在执行某些管理任务期间，直接从数据库服务器机器上的 `SQL*Plus` 发出管理启动命令要方便得多，因此我们最终又有了多个参数文件：一个在管理服务器上，一个在数据库服务器上。这些参数文件随后会彼此不同步，人们会纳闷为什么他们上个月做的参数更改可能会“消失”，然后以看似随机的方式重新出现。

于是引入了服务器参数文件（`SPFILE`），它现在可以成为数据库的唯一真实来源。

## 服务器参数文件 (SPFILE)

`SPFILE` 代表了 Oracle 访问和维护实例参数设置方式的根本性改变。`SPFILE` 消除了与旧版参数文件相关的两个严重问题：

*   *它阻止了参数文件的激增：* `SPFILE` 总是存储在数据库服务器上；`SPFILE` 必须存在于服务器机器本身，不能位于客户端机器上。这使得在参数设置方面拥有一个单一的“事实来源”变得切实可行。
*   *它消除了（实际上，它移除了能力）在数据库外部使用文本编辑器手动维护参数文件的需要：* `ALTER SYSTEM` 命令允许你将值直接写入 `SPFILE`。管理员不再需要手动查找和维护所有参数文件。

该文件的默认命名约定是

```
$ORACLE_HOME/dbs/spfile$ORACLE_SID.ora    (UNIX/Linux 环境变量)
%ORACLE_HOME/database/spfile%ORACLE_SID%.ora   (Windows 环境变量)
```

我强烈建议使用默认位置；否则就违背了 `SPFILE` 所代表的简洁性。当 `SPFILE` 位于其默认位置时，一切或多或少都是为你自动完成的。将 `SPFILE` 移动到非默认位置意味着你必须告诉 Oracle 在哪里找到 `SPFILE`，这又会导致旧版参数文件的所有原始问题重现！

**注意**

在 Oracle RAC 环境中，spfile 通常位于共享 ASM 磁盘上的目录中，例如 `+DATA/<dbname>/PARAMETERFILE`。你可以通过 `SRVCTL` 实用程序查看位置。


### 转换为 SP 文件

假设你有一个正在使用传统参数文件的数据库。迁移到 `SP 文件` 相当简单——你只需使用 `CREATE SPFILE` 命令。

注意：你也可以使用“反向”命令从 `SP 文件` 创建一个参数文件 (`PFILE`)。稍后我会解释为什么你可能需要这样做。

因此，假设你有一个 `init.ora` 参数文件，并且该文件位于服务器的默认位置，你只需执行 `CREATE SPFILE` 命令并重启服务器实例。你需要以特权账户连接到根容器才能执行此任务：

```sql
$ sqlplus / as sysdba
SQL> show parameter spfile;
NAME                                 TYPE        VALUE
------------------------------------ ----------- -------
spfile                               string
SQL> create spfile from pfile;
File created.
SQL> startup force;
ORACLE instance started.
Database mounted.
Database opened.
SQL> show parameter spfile;
NAME                                 TYPE        VALUE
------------------------------------ ----------- --------------------------
spfile                               string      /opt/oracle/product/21c/dbhome_1/dbs/spfileCDB.ora
```

回顾一下，我们在这里使用 `SHOW PARAMETER` 命令是为了表明最初我们没有使用 `SP 文件`，但在我们创建了一个并重启实例后，我们就在使用它了，并且它具有默认名称。

注意：在集群环境（使用 Oracle RAC）中，所有实例共享同一个 `SP 文件`，因此将 `P 文件` 转换为 `SP 文件` 的过程应该以受控的方式进行。单个 `SP 文件` 可以包含所有参数设置，甚至是实例特定的设置，但你必须使用如下格式将所有必要的参数文件合并为一个 `P 文件`。

在 RAC 环境中，为了从单个 `P 文件` 转换为所有实例共享的 `SP 文件`，你需要将你的单个 `P 文件` 合并为一个如下所示的单一文件：

```text
*.cluster_database=true
*.control_files='+DATA/CDB/CONTROLFILE/current.261.1064287219','+RECO/CDB/CONTROLFILE/current.256.1064287219'
*.db_block_size=8192
*.db_create_file_dest='+DATA'
*.db_name='cdb'
*.db_recovery_file_dest='+RECO'
*.db_recovery_file_dest_size=12207m
*.diagnostic_dest='/u01/app/oracle'
*.dispatchers='(PROTOCOL=TCP) (SERVICE=cdbXDB)'
*.enable_pluggable_database=true
cdb1.instance_number=1
cdb2.instance_number=2
*.local_listener='-oraagent-dummy-'
*.nls_language='AMERICAN'
*.nls_territory='AMERICA'
*.pga_aggregate_target=256m
*.remote_login_passwordfile='exclusive'
*.sga_target=2000m
cdb2.thread=2
cdb1.thread=1
cdb1.undo_tablespace='UNDOTBS1'
cdb2.undo_tablespace='UNDOTBS2'
```

也就是说，集群中所有实例共有的参数设置将以 "`*.`" 字符串开头。特定于单个实例的参数设置，例如 `INSTANCE_NUMBER` 和要使用的重做日志的 `THREAD`，会以实例名（Oracle SID）作为前缀。在上面的例子中：

*   这个 `P 文件` 是为一个双节点集群准备的，其实例名为 `CDB1` 和 `CDB2`。
*   `*.db_name = 'CDB'` 这个赋值表示所有使用此 `SP 文件` 的实例都将挂载一个名为 `CDB` 的数据库。
*   `cdb1.undo_tablespace='UNDOTBS1'` 表示名为 `CDB1` 的实例将使用该特定的撤消表空间，依此类推。

### 在 SP 文件中设置值

一旦我们的数据库在 `SP 文件` 上启动并运行，下一个问题就涉及如何设置和更改其中的值。请记住，`SP 文件` 是二进制文件，我们不能直接用文本编辑器编辑它们。答案是使用 `ALTER SYSTEM` 命令，其语法如下（<>内的部分是可选的，管道符号表示“列表中的一个”）：

```sql
Alter system set parameter=value
```

默认情况下，`ALTER SYSTEM SET` 命令将更新当前运行的实例，并为你更改 `SP 文件`——或者在可插拔数据库的情况下，更改该可插拔数据库的数据字典（有关可插拔数据库的更多信息，请参见下一节）。这极大地简化了管理，并消除了当使用 `ALTER SYSTEM` 添加或修改参数设置，却忘记更新或漏掉 `init.ora` 参数文件时出现的问题。

让我们看一下该命令的每个元素：

*   `parameter=value` 赋值提供了参数名和该参数的新值。例如，`pga_aggregate_target = 1024m` 会将 `pga_aggregate_target` 参数设置为 1024MB（1GB）。
*   `comment='text'` 是一个可选的注释，你可以将其与此参数的此次设置关联起来。该注释将出现在 `V$PARAMETER` 视图的 `UPDATE_COMMENT` 字段中。如果你使用了将更改保存到 `SP 文件` 的选项，该注释也会被写入 `SP 文件`，并在服务器重启后得以保留，因此数据库未来的重启时也能看到该注释。
*   `deferred` 指定系统更改是仅对后续会话生效（不包括当前已建立的会话，包括进行更改的会话）。默认情况下，`ALTER SYSTEM` 命令会立即生效，但有些参数不能立即更改——它们只能为新建立的会话更改。我们可以使用以下查询来查看哪些参数强制要求使用 `deferred`：

```sql
SQL> select name from v$parameter where issys_modifiable='DEFERRED';
NAME
--------------------------------
backup_tape_io_slaves
recyclebin
session_cached_cursors
private_temp_table_prefix
audit_file_dest
object_cache_optimal_size
object_cache_max_size_percent
sort_area_size
sort_area_retained_size
client_statistics_level
olap_page_pool_size
```

注意：你的结果可能不同；从版本到版本，哪些参数可以在线设置——但必须延迟设置——列表可能会也确实会改变。

代码显示 `SORT_AREA_SIZE` 可以在系统级别修改，但只能以延迟的方式进行。以下代码展示了如果我们尝试使用和不使用 `deferred` 选项来修改其值会发生什么：

*   `SCOPE=MEMORY|SPFILE|BOTH` 指示了此参数设置的“范围”。以下是我们设置参数值时的选择：
    *   `SCOPE=MEMORY` 仅在实例中更改设置；它不会在数据库重启后保留。下次启动数据库时，设置将是已记录在 `SP 文件` 中的值。
    *   `SCOPE=SPFILE` 仅在 `SP 文件` 中更改值。直到数据库重启并再次处理 `SP 文件` 后，更改才会生效。有些参数只能通过此选项更改。例如，`processes` 参数必须使用 `SCOPE=SPFILE`，因为你无法更改活动实例的值。
    *   `SCOPE=BOTH` 表示参数更改同时在内存和 `SP 文件` 中生效。更改将反映在当前实例中，并且下次启动时，此更改仍然有效。这是使用 `SP 文件` 时 `SCOPE` 的默认值。对于 `init.ora` 参数文件，默认且唯一有效的值是 `SCOPE=MEMORY`。如果实例是使用 `SP 文件` 启动的，这就是默认值。


