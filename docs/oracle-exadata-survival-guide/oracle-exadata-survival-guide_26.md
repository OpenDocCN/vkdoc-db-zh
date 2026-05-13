# SCAN 监听器与数据库创建指南

## Oracle Home 的注册与补丁应用

前述命令将确保 `Oracle Home` 被注册到中央 `inventory` 中。正确执行克隆操作非常重要，因为它会对后续应用数据库补丁（特别是季度捆绑补丁）产生重大影响。如果 `Oracle Home` 未注册，你将无法为其应用补丁。

很可能需要在此基础版本之上安装额外的数据库补丁。一旦有额外的补丁应用到这个 `Oracle Home`，它就可以作为后续克隆操作的源。

## SCAN 监听器

虽然 `ERP 11i` 和 `R12` 都可以利用单客户端访问名称 (`SCAN`) 监听器，但其使用是可选的。在撰写本文时，所有的 `EBS` 客户端版本都低于 11.2；因此，`EBS` 无法充分利用 `SCAN` 监听器。如果你运行的是 `11i`，那么你可能会被建议不要为你的 `ERP` 数据库使用 `SCAN` 监听器，因为 `Autoconfig` 不支持它——尽管有手动设置步骤。此外，如果你正在搭建一个包含多个不同数据库的开发/测试环境，你将需要设置多个监听器——每个环境一个。也许你正在将一个 `11i` 数据库从其他平台迁移过来，而 1521 端口（用 `Oracle ERP` 的术语说，即端口池零）已经使用了多年，因此可能难以更改。无论如何，你可能会发现自己处于一种不希望 `SCAN` 监听器运行在默认 1521 端口上的情况。在这种情况下，你将不得不重新配置 `SCAN` 监听器以使其在另一个端口上运行。具体选择哪个端口其实并不太重要，但为了保持相当简单，我们将使用 `1523`。

“My Oracle Support”上有一份说明也概述了相关步骤 (`How to Modify SCAN Setting or SCAN Listener Port after Installation [ID 972500.1]`)。以该说明为指导，我们执行以下操作：

首先，修改任何现有的非 `ERP` 数据库的 `remote_listener` 和 `local_listener` 参数。在每个这样的数据库上，以 `SYS` 身份执行以下命令：

```sql
SQL> alter system set remote_listener='exacluster-scan:1523' scope=spfile sid='*' ;
SQL> alter system set local_listener=
   '(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=exa01-vip)(PORT=1523))))'
   scope=spfile sid='dbm1' ;
SQL> alter system set local_listener=
   '(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=exa02-vip)(PORT=1523))))'
   scope=spfile sid='dbm2' ;
```

设置 `Grid Oracle Home` 的环境，并为 `ASM` 执行相同的操作。注意，`ASM` 没有 `remote_listener`：

```sql
alter system set local_listener=  '(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=exa01-vip)(PORT=1523))))'
   scope=spfile sid='+ASM1' ;
alter system set local_listener=  '(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=exa01-vip)(PORT=1523))))'
   scope=spfile sid='+ASM2' ;
```

关闭所有数据库。

```bash
$ srvctl stop database -d dbm
```

现在，我们将设置 `Grid Oracle Home` 的环境并更改监听器端口，如下所示：

```bash
$ srvctl modify listener -l LISTENER -p "TCP:1523"
$ srvctl modify scan_listener -p 1523
$ srvctl stop scan_listener
$ srvctl start scan_listener
```

重启数据库，应该就全部完成了。

```bash
$ srvctl start database –d dbm
```

## 创建数据库

无论是执行全新安装还是从其他服务器迁移，你都必须为你未来的完整 `ERP` 环境创建一个空数据库。实际上，这与在任何使用 `Oracle Managed Files` 的 `RAC` 环境中创建数据库并无区别。有趣的部分将在下一节讨论设置监听器时到来。然而，为了文档的完整性，我们必须提及，是的，需要创建一个数据库。

最简单的方法是使用 `dbca`。设置 `ORACLE_HOME` 并相应地修改 `PATH`，如下所示：

```bash
$ export ORACLE_HOME=/u01/app/oracle/product/11.2.0.3/erpdevdb
$ export PATH=$PATH:$ORACLE_HOME/bin
$ dbca
```



运行 `dbca` 的步骤已有详尽文档，因此这里不再赘述。请按常规选择用于 Oracle 托管文件的磁盘组、使用的字符集以及各项内存配置选项。完成后，你可能仍需调整重做日志大小以及表空间（`SYSTEM`、`UNDOTBS1` 和 `UNDOTBS2`、`TEMP`、`USERS` 等）的存储参数。

数据库创建完成后，请将该实例（数据库本身应已被添加）添加到集群中每个节点的 `oratab` 文件中，并使用 Oracle 的 `oraenv` 实用程序来设置环境。节点 1 上的 `oratab` 文件应类似于以下内容：
```
+ASM1:/u01/app/11.2.0.3/grid:N
agent11g:/u01/app/oracle/GCAgents/oracle/MW/agent11g:N
dbm1:/u01/app/oracle/product/11.2.0.3/dbhome_1:N
erpdev1:/u01/app/oracle/product/11.2.0.3/erpdevdb:N
dbm:/u01/app/oracle/product/11.2.0.3/dbhome_1:N           # line added by Agent
erpdev:/u01/app/oracle/product/11.2.0.3/erpdevdb:N        # line added by Agent
```
节点 2 的 `oratab` 文件将类似，区别在于会列出本地属于该节点的实例（`dbm2`、`erpdev2` 等）。

## 设置监听器
之前，我们将 SCAN 监听器移到了 `1523` 端口。因为 `1521` 端口现在空闲，如果我们愿意，可以创建一个使用该端口的新监听器。此监听器将专用于我们的 `ERPDEV` 数据库。无论你使用哪个端口，都必须遵循这些步骤来为你的新数据库创建专用监听器。关键点是不能使用 SCAN 监听器正在使用的端口。

关闭新创建的 `ERPDEV` 数据库：
```
$ srvctl stop database -d erpdev
```
在 `$ORACLE_HOME/network/admin` 目录下，创建一个新的 `tnsnames.ora` 文件，其中包含以下条目：
```
erpdev=
        (DESCRIPTION=
                (ADDRESS=(PROTOCOL=tcp)(HOST=exa01-vip)(PORT=1521))
                (ADDRESS=(PROTOCOL=tcp)(HOST=exa02-vip)(PORT=1521))
            (LOAD_BALANCE = yes)
            (CONNECT_DATA=
                (SERVER=DEDICATED)
                (SERVICE_NAME=erpdev)
            )
        )

erpdev1_LOCAL=
        (DESCRIPTION=
                (ADDRESS=(PROTOCOL=tcp)(HOST=exa01-vip)(PORT=1521))
        )

erpdev2_LOCAL=
        (DESCRIPTION=
                (ADDRESS=(PROTOCOL=tcp)(HOST=exa02-vip)(PORT=1521))
        )

erpdev_REMOTE=
        (DESCRIPTION=
            (ADDRESS_LIST=
                (ADDRESS=(PROTOCOL=tcp)(HOST=exa01-vip)(PORT=1521))
                (ADDRESS=(PROTOCOL=tcp)(HOST=exa02-vip)(PORT=1521))
                (ADDRESS=(PROTOCOL=tcp)(HOST=exa01-vip)(PORT=1523))
                (ADDRESS=(PROTOCOL=tcp)(HOST=exa02-vip)(PORT=1523))
            )
        )
```
注意 `erpdev_REMOTE` 条目中的 `1523` 端口引用。这些地址条目也会导致数据库服务向默认监听器注册。如果将来任何时候需要为此数据库运行 `dbca`，则数据库必须向默认监听器注册。你可能需要运行 `dbca` 的原因多种多样，例如配置 Database Vault 或 Oracle Internet Directory。

接下来，创建监听器并将其关联到此 Oracle Home。
```
$ srvctl add listener -l erpdev -o $ORACLE_HOME -p 1521
$ srvctl setenv listener -l erpdev -T TNS_ADMIN=$ORACLE_HOME/network/admin
$ srvctl setenv database -d erpdev -T TNS_ADMIN=$ORACLE_HOME/network/admin
```
启动数据库并更改其监听器参数。
```
$ srvctl start database -d erpdev

SQL> alter system set local_listener=
   '(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=exa01-vip)(PORT=1521)))'
   sid='erpdev1' ;

SQL> alter system set local_listener=
   '(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=exa02-vip)(PORT=1521)))'
   sid='erpdev2' ;

SQL> alter system set remote_listener='erpdev_REMOTE' ;
```
如果你还希望将数据库注册到 SCAN 监听器，可以按如下方式设置 `remote_listener` 参数：
```
SQL> alter system set remote_listener='erpdev_REMOTE','exacluster-scan:1523' ;
```

## 数据迁移
至此，我们有了一个空数据库和该数据库的监听器。除非这是 Oracle ERP 的全新安装，否则下一个（也是最耗时的）步骤是将现有数据库迁移到我们崭新、闪亮的 Exadata 数据库。迁移步骤因源平台、ERP 版本和许多其他因素而异。简而言之，你需要熟悉大量的“My Oracle Support”说明，才能将你的数据库从 A 点转移到 Exadata。我们无法在此尝试涵盖该领域。因此，我们将在导入完成且已运行 Autoconfig 的时刻继续。请参阅本章末尾的 MOS 说明列表，这应为你顺利完成任务提供一个良好的基础。

## 环境配置
在传统的单节点 ERP 数据库环境中，SQLNet 配置文件存储在 `$ORACLE_HOME/network/admin/<context>` 下，其中 `<context>` 是 `instance_server`。是 Autoconfig 创建了这些文件。同时，它创建的 `env` 文件会将 `TNS_ADMIN` 设置为相同的 `$ORACLE_HOME/network/admin/<context>` 位置。然而，在 RAC 环境中，我们希望使用 `srvctl` 来管理环境（即启动和停止数据库及监听器）。既然如此，我们就不能以这种方式设置 `TNS_ADMIN`，因为显然它在每个节点上都必须不同，因为实例名称会变化。

传统的单节点设置：
```
TNS_ADMIN=$ORACLE_HOME/network/admin/erpdev_dbsrvr
$ORACLE_HOME/network/admin/erpdev_dbsrvr/tnsnames.ora
$ORACLE_HOME/network/admin/erpdev_dbsrvr/listener.ora
```
此外，还会有一个 `ifile` 参数指向另一个可用于自定义设置的文件。在 Exadata (RAC) 上，我们将反转这种做法。尤其要注意，我们根本不会使用 Autoconfig 生成的 `listener.ora`，因为它的条目不适用于 RAC。

请注意，Autoconfig 创建的 `tnsnames.ora` 文件通常就 RAC 所需而言充满了错误。应审查该文件并进行调整。我们之前创建的 `tnsnames.ora` 文件可以用作此文件的模型。然后，我们要避免这些更改因后续运行 Autoconfig 而被覆盖。

管理此问题有几种选择。
1.  将文件复制到 `$ORACLE_HOME/network/admin/tnsnames.ora` 上，即：
```
    $ cd $ORACLE_HOME/network/admin/erpdev1_exadb01
    $ cp tnsnames.ora ../.
```
2.  重命名原始文件并向 `$ORACLE_HOME/network/admin/tnsnames.ora` 添加一个 `ifile` 条目，该条目将指向它。
```
    $ cd $ORACLE_HOME/network/admin/erpdev1_exadb01
    $ cp tnsnames.ora tnsnames.ora.erpdev
    $ vi $ORACLE_HOME/network/admin/tnsnames.ora
    添加 ➔ ifile=<ORACLE_HOME>/network/admin/erpdev1_exadb01/tnsnames.ora.erpdev
```
因为 Autoconfig 自然地将其活动限制在 `$ORACLE_HOME/network/admin/<context>` 目录内，所以此选项效果很好。`$ORACLE_HOME/network/admin` 下的文件不会受到影响。无论你选择做什么，都必须完全告知将来可能在这些服务器上运行 Autoconfig 或向远程数据库添加条目到 `tnsnames.ora` 的任何人。

![image](img/sq.jpg) **注意** `ifile` 条目必须包含完整路径。`$ORACLE_HOME` 在此处无效。确保在两个节点上都使用完整路径。

### 设置环境



