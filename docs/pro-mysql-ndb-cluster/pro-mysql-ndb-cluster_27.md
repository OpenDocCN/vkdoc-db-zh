# 依赖性错误处理
```shell
shell$ rpm -ivh mysql-cluster-community-server-7.5.4-1.el7.x86_64.rpm
error: Failed dependencies:
mysql-cluster-community-client(x86-64) >= 5.7.9 is needed by mysql-cluster-community-server-7.5.4-1.el7.x86_64
mysql-cluster-community-common(x86-64) = 7.5.4-1.el7 is needed by mysql-cluster-community-server-7.5.4-1.el7.x86_64
```

在这种情况下，这是因为 MySQL NDB Cluster 服务器 RPM 已被拆分为多个 RPM，以便更大程度地选择要安装的二进制文件。在其他情况下，可能是需要新库或现有库的更新版本。在所有情况下，请阅读错误消息以查看哪个依赖项未满足。然后在 `yum` 或 `rpm` 命令中包含提供缺失依赖项的包。

#### 在线降级

本示例中执行的降级与第一个案例研究中执行的升级相反。也就是说，集群将从使用版本 7.5.4 开始，降级到版本 7.4.13：

```shell
shell$ ndb_mgm -e "SHOW"
Connected to Management Server at: 192.168.56.101:1186
Cluster Configuration

[ndbd(NDB)]     2 node(s)
id=1    @192.168.56.103  (mysql-5.7.16 ndb-7.5.4, Nodegroup: 0, *)
id=2    @192.168.56.104  (mysql-5.7.16 ndb-7.5.4, Nodegroup: 0)
[ndb_mgmd(MGM)] 2 node(s)
id=49   @192.168.56.101  (mysql-5.7.16 ndb-7.5.4)
id=50   @192.168.56.102  (mysql-5.7.16 ndb-7.5.4)
[mysqld(API)]   6 node(s)
id=51   @192.168.56.103  (mysql-5.7.16 ndb-7.5.4)
id=52   @192.168.56.104  (mysql-5.7.16 ndb-7.5.4)
id=53 (not connected, accepting connect from 192.168.56.101)
id=54 (not connected, accepting connect from 192.168.56.102)
id=55 (not connected, accepting connect from any host)
id=56 (not connected, accepting connect from any host)
```

假设 7.4.13 二进制文件已在所有主机上准备就绪。

照例，最佳实践是从创建备份开始。由于在降级过程中比升级更可能遇到复杂情况，因此确保备份可用尤为重要。由于 SQL 节点将作为降级的一部分重新初始化，因此必须拥有所有非 `NDBCluster` 数据的备份。还值得拥有包含 `NDBCluster` 表的模式的 `CREATE DATABASE` 语句列表。可以使用以下查询创建此列表：

```sql
mysql> SELECT DISTINCT
CONCAT('CREATE SCHEMA IF NOT EXISTS `', SCHEMA_NAME, '`;')
FROM information_schema.SCHEMATA
INNER JOIN information_schema.TABLES ON
TABLES.TABLE_SCHEMA = SCHEMATA.SCHEMA_NAME
WHERE TABLES.ENGINE = 'ndbcluster'
AND SCHEMATA.SCHEMA_NAME != 'mysql';
+------------------------------------------------------------+
| CONCAT('CREATE SCHEMA IF NOT EXISTS `', SCHEMA_NAME, '`;') |
+------------------------------------------------------------+
| CREATE SCHEMA IF NOT EXISTS `world`;                       |
+------------------------------------------------------------+
1 row in set (0.02 sec)
```

备份就绪后，管理节点和数据节点的降级遵循与升级相同的步骤。首先关闭两个管理节点，然后使用 7.4.13 二进制文件重新启动它们：

```shell
shell$ ndb_mgm -e "49 STOP"
Connected to Management Server at: 192.168.56.101:1186
Node 49 has shutdown.
Disconnecting to allow Management Server to shutdown
shell$ ndb_mgm -e "50 STOP"
Connected to Management Server at: 192.168.56.102:1186
Node 50 has shutdown.
Disconnecting to allow Management Server to shutdown
```

在 `NodeId = 49` 的管理节点所在主机上：

```shell
shell$ sudo -u mysql /opt/cluster/7.4.13/bin/ndb_mgmd \
--config-file=/etc/config.ini --config-dir=/cluster/config \
--ndb-nodeid=49 --reload
```

对于 `NodeId = 50` 也类似：

```shell
shell$ sudo -u mysql /opt/cluster/7.4.13/bin/ndb_mgmd \
--config-file=/etc/config.ini --config-dir=/cluster/config \
--ndb-nodeid=50 --reload
```

管理节点降级后，继续处理数据节点。首先降级 `NodeId = 1` 的数据节点：

```shell
shell$ ndb_mgm -e "1 STOP"
Connected to Management Server at: 192.168.56.101:1186
Node 1 has shutdown.
shell$ sudo -u mysql /opt/cluster/7.4.13/bin/ndbmtd --ndb-nodeid=1
2016-12-17 18:21:32 [ndbd] INFO     -- Angel connected to '192.168.56.101:1186'
2016-12-17 18:21:32 [ndbd] INFO     -- Angel allocated nodeid: 1
```

对于 `NodeId = 2`：

```shell
shell$ ndb_mgm -e "2 STOP"
Connected to Management Server at: 192.168.56.101:1186
Node 2 has shutdown.
shell$ sudo -u mysql /opt/cluster/7.4.13/bin/ndbmtd --ndb-nodeid=2
2016-12-17 18:55:42 [ndbd] INFO     -- Angel connected to '192.168.56.101:1186'
2016-12-17 18:55:42 [ndbd] INFO     -- Angel allocated nodeid: 2
```

降级的最后部分是 SQL 节点，这也是最困难的部分。MySQL NDB Cluster 7.5.4 将包含一些 `InnoDB` 表，因为 MySQL Server 5.7（用于 NDB Cluster 7.5）需要这些表。MySQL Server 5.7 和 MySQL Server 5.6（用于 NDB Cluster 7.4）之间的 `InnoDB` 不兼容。虽然 MySQL 知道如何在升级时处理此问题，但不支持 `InnoDB` 的降级。因此，SQL 节点必须重新初始化。

要降级第一个 SQL 节点，首先将其关闭：

```shell
shell$ /opt/cluster/7.5.4/bin/mysqladmin --host=127.0.0.1 shutdown
```

要重新初始化 SQL 节点，首先删除数据目录中的所有内容以及位于数据目录外的任何 `InnoDB`（或其他存储引擎的文件）。例如，如果 `datadir = /var/lib/mysql` 且所有文件都存储在此目录中，则可以按如下方式进行重新初始化：

```shell
shell$ rm -rf /var/lib/mysql
shell$ mkdir /var/lib/mysql
shell$ chown mysql:mysql /var/lib/mysql
shell$ /opt/cluster/7.4.13/scripts/mysql_install_db \
--basedir=/opt/cluster/7.4.13 --datadir=/var/lib/mysql \
--user=mysql
Installing MySQL system tables...
...
```

然后可以再次启动该节点：

```shell
shell$ /opt/cluster/7.4.13/bin/mysqld &
```

恢复 SQL 节点的非 `NDBCluster` 数据——包括重新设置权限。最后对其他 SQL 节点重复此操作。

#### 离线升级

第四个也是最后一个示例将是一次离线升级。与之前的升级示例一样，它将从版本 `7.4.13` 升级到 `7.5.4`。示例开始时，MySQL NDB 集群 `7.4.13` 已安装并处于在线状态：

```
shell$ ndb_mgm -e "SHOW"
Connected to Management Server at: 192.168.56.101:1186
Cluster Configuration

[ndbd(NDB)]     2 node(s)
id=1    @192.168.56.103  (mysql-5.6.34 ndb-7.4.13, Nodegroup: 0, *)
id=2    @192.168.56.104  (mysql-5.6.34 ndb-7.4.13, Nodegroup: 0)
[ndb_mgmd(MGM)] 2 node(s)
id=49   @192.168.56.101  (mysql-5.6.34 ndb-7.4.13)
id=50   @192.168.56.102  (mysql-5.6.34 ndb-7.4.13)
[mysqld(API)]   6 node(s)
id=51   @192.168.56.103  (mysql-5.6.34 ndb-7.4.13)
id=52   @192.168.56.104  (mysql-5.6.34 ndb-7.4.13)
id=53 (not connected, accepting connect from 192.168.56.101)
id=54 (not connected, accepting connect from 192.168.56.102)
```

像往常一样，首先创建备份。然后作为关闭过程的第一步，停止 SQL 节点。关闭 SQL 节点的确切方法取决于平台和所使用的二进制文件。SQL 节点离线后，使用管理客户端中的 `SHUTDOWN` 命令关闭管理节点和数据节点：

```
shell$ ndb_mgm -e "SHUTDOWN"
Connected to Management Server at: 192.168.56.101:1186
4 NDB Cluster node(s) have shutdown.
Disconnecting to allow management server to shutdown.
```

等待关机完成，然后替换所有二进制文件，并使用升级后的二进制文件重新启动集群。首先启动管理节点：

```
shell$ sudo -u mysql ndb_mgmd --config-file=/etc/config.ini \
--config-dir=/cluster/config --ndb-nodeid=49 –reload
MySQL Cluster Management Server mysql-5.7.16 ndb-7.5.4
shell$ sudo -u mysql ndb_mgmd --config-file=/etc/config.ini \
--config-dir=/cluster/config --ndb-nodeid=50 --reload
MySQL Cluster Management Server mysql-5.7.16 ndb-7.5.4
```

下一步是启动数据节点。并行启动两个数据节点以缩短重启所需时间：

```
shell$ sudo -u mysql ndbmtd --ndb-nodeid=1
2016-12-17 20:13:25 [ndbd] INFO     -- Angel connected to '192.168.56.101:1186'
2016-12-17 20:13:25 [ndbd] INFO     -- Angel allocated nodeid: 1
```

另一个数据节点：

```
shell$ sudo -u mysql ndbmtd --ndb-nodeid=2
2016-12-17 20:13:35 [ndbd] INFO     -- Angel connected to '192.168.56.101:1186'
2016-12-17 20:13:35 [ndbd] INFO     -- Angel allocated nodeid: 2
```

当数据节点完成重启后，根据平台要求逐一启动 SQL 节点。在每个 SQL 节点启动后，务必执行 `mysql_upgrade` 以检查和升级表：

```
shell$ mysql_upgrade --host=127.0.0.1
Checking if update is needed.
Checking server version.
Running queries to upgrade MySQL server.
...
Upgrade process completed successfully.
Checking if update is needed.
```

### 总结

升级和降级在最好的情况下也是不简单的任务，如果还涉及主要版本的变更，则可能需要大量的工作。在某些环境中，一次升级需要准备数月时间。MySQL NDB 集群提供了一些便利，因为升级——以及可能的降级——可以在线执行。

本章讨论了在线和离线流程的升级和降级步骤。此外，还提供了四个不同升级和降级场景的案例研究。

下一章将讨论 MySQL NDB 集群中的安全注意事项。

