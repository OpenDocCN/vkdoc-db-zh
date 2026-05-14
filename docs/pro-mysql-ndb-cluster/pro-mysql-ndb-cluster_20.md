# 使用 NDB$EPOCH_TRANS 方法设置冲突检测

##### 前提条件

本示例假设已经安装了两个集群，并且它们正在运行多主复制。每个集群中的一个 SQL 节点同时充当主节点和从节点。为复制分配的 SQL 节点 `server_id` 分别为 `cluster1` 的 1001 和 `cluster2` 的 2001。图 6-16 展示了此假设配置。

除了图 6-16 中描绘的 SQL 节点外，可能还存在其他 SQL 节点或 API 节点。为了简化，它们被省略了。在此设置下，`log_slave_updates` 应该已经被设置。

##### 配置集群

首先要做的是在两个集群的 `my.cnf` 中添加额外的配置。清单 6-15 展示了 `cluster1` 的示例配置。

```
[mysqld]
ndbcluster
ndb_connectstring = mgmhost
log_bin = mysql-bin
log_slave_updates
server_id = 1001
... snip ...
# Additional configurations for NDB$EPOCH_TRANS method
ndb_log_update_as_write = OFF
ndb_log_updated_only = OFF
ndb_log_apply_status = ON
log_bin_use_v1_row_events = OFF
ndb_log_transaction_id = ON
```

清单 6-15. NDB$EPOCH_TRANS 方法的示例配置

要使更改生效，必须重新启动 SQL 节点，因为某些选项不是动态的，无法在不重启服务器的情况下更改。重启两个 SQL 节点后，使用 `START SLAVE` 命令再次启动复制。回想一下，重启主 SQL 节点会导致其在离线期间错过接收事件。为防止此问题，你有两个选择——在应用程序离线时重启 SQL 节点，以确保重启期间没有更新；或者先在备用复制通道上配置 SQL 节点，然后将复制切换到备用通道。请注意，在前一种情况下，复制会因为 `LOST_EVENT` 事件而停止。如果重启期间确实没有进行任何修改，你可以通过 `SET GLOBAL sql_slave_skip_counter = 1` 安全地忽略该错误。

##### 添加冲突检测条目

下一步是向 `mysql.ndb_replication` 表添加条目，以配置冲突检测和解决。假设除了 `server_id` 为 1001 和 2001 的 SQL 节点外，还配置了一个 `server_id` 为 1002 和 2002 的备用复制通道。在这种情况下，冲突检测和解决只需在主节点端（`server_id` 为 1001 和 1002 的 SQL 节点）进行。这是因为 `NDB$EPOCH_TRANS` 方法是非对称的，主节点总是获胜。清单 6-16 展示了一个向 `mysql.ndb_replication` 表添加条目的示例。如果表尚未创建，请先创建它。

```
mysql> INSERT INTO mysql.ndb_replication VALUES ('test', 't_conflict', 1001, NULL, 'NDB$EPOCH_TRANS()');
Query OK, 1 row affected (0.00 sec)
mysql> INSERT INTO mysql.ndb_replication VALUES ('test', 't_conflict', 1002, NULL, 'NDB$EPOCH_TRANS()');
Query OK, 1 row affected (0.00 sec)
```

清单 6-16. 为 NDB$EPOCH_TRANS 方法向 `mysql.ndb_replication` 表添加所需条目

由于 `NDB$EPOCH` 和 `NDB$EPOCH_TRANS` 方法每个 `server_id` 需要一个条目，因此本例需要两个条目。请注意，清单 6-16 中省略了 `NDB$EPOCH_TRANS()` 的参数。本例中使用了默认值 6。

##### 创建异常表与目标表

在创建目标表之前，先创建异常表。本例中表名必须是 `t_conflict$EX`。异常表的定义因选择而异——是否包含每个非主键列，以及是否使用可选列。清单 6-17 展示了 `t_conflict` 目标表的示例异常表。

```
CREATE TABLE t_conflict$EX (
NDB$server_id INT UNSIGNED,
NDB$master_server_id INT UNSIGNED,
NDB$master_epoch BIGINT UNSIGNED,
NDB$count INT UNSIGNED,
NDB$OP_TYPE ENUM('WRITE_ROW','UPDATE_ROW', 'DELETE_ROW',
'REFRESH_ROW', 'READ_ROW') NOT NULL,
NDB$CFT_CAUSE ENUM('ROW_DOES_NOT_EXIST', 'ROW_ALREADY_EXISTS',
'DATA_IN_CONFLICT', 'TRANS_IN_CONFLICT') NOT NULL,
NDB$ORIG_TRANSID BIGINT UNSIGNED NOT NULL,
id BIGINT UNSIGNED not null,
col1$OLD VARCHAR(64) CHARACTER SET utf8,
col1$NEW VARCHAR(64) CHARACTER SET utf8,
PRIMARY KEY (NDB$server_id, NDB$master_server_id,
NDB$master_epoch, NDB$count)
) ENGINE = NDBCluster;
```

清单 6-17. 包含可选列和一个非主键列的异常表示例

最后，创建目标表，如清单 6-18 所示。

```
mysql> CREATE TABLE t_conflict (
id BIGINT UNSIGNED NOT NULL PRIMARY KEY,
col1 VARCHAR(64),
col2 DATETIME,
INDEX ix1 (col1, col2)
) ENGINE=NDBCluster CHARACTER SET utf8;
Query OK, 0 rows affected (0.47 sec)
```

清单 6-18. 为 NDB$EPOCH_TRANS 方法的冲突检测和解决创建目标表

##### 查看日志并测试

你将在 SQL 节点的错误日志中看到类似清单 6-19 的消息。

```
2017-05-06T03:28:22.329499Z 10 [Note] NDB Binlog: CREATE TABLE Event: REPLF$test/t_conflict$EX
2017-05-06T03:28:22.778946Z 10 [Note] NDB Slave: Table test.t_conflict logging exceptions to test.t_conflict$EX
2017-05-06T03:28:22.778965Z 10 [Note] NDB Slave: Table test.t_conflict using conflict_fn NDB$EPOCH_TRANS.
2017-05-06T03:28:22.779511Z 10 [Note] NDB Binlog: CREATE TABLE Event: REPLF$test/t_conflict
2017-05-06T03:28:22.793209Z 10 [Note] NDB Binlog: logging ./test/t_conflict (FULL,USE_UPDATE)
```

清单 6-19. 创建表时在错误日志中出现的示例消息

现在你可以测试冲突检测和解决了。清单 6-20 展示了使用 `NDB$EPOCH_TRANS` 方法进行冲突检测和解决的示例。假设 `mysqlP>` 代表主节点（`cluster1` 上 `server_id = 1001` 的 SQL 节点会话）的提示符，`mysqlS>` 代表辅助节点（`cluster2` 上 `server_id` = 2001 的 SQL 节点会话）的提示符。

```
##### PRIMARY #####
mysqlP> INSERT INTO t_conflict VALUES (1, 'Sun', NOW());
Query OK, 1 row affected (0.00 sec)
mysqlP> BEGIN;
Query OK, 0 rows affected (0.00 sec)
mysqlP> UPDATE t_conflict SET col1 = 'Moon';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
##### SECONDARY #####
mysqlS> BEGIN;
Query OK, 0 rows affected (0.00 sec)
mysqlS> UPDATE t_conflict SET col1 = 'Jupiter';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
##### PRIMARY #####
mysqlP> COMMIT;
Query OK, 0 rows affected (0.00 sec)
##### SECONDARY #####
mysqlS> COMMIT;
Query OK, 0 rows affected (0.00 sec)
##### PRIMARY #####
mysqlP> SELECT * FROM t_conflict;
+----+------+---------------------+
| id | col1 | col2                |
+----+------+---------------------+
|  1 | Moon | 2017-05-06 12:37:50 |
+----+------+---------------------+
1 row in set (0.00 sec)
mysqlP> SELECT * FROM t_conflict$EX\G
*************************** 1\. row ***************************
NDB$server_id: 1
NDB$master_server_id: 1001
NDB$master_epoch: 1835213755777034
NDB$count: 3
NDB$OP_TYPE: UPDATE_ROW
NDB$CFT_CAUSE: TRANS_IN_CONFLICT
NDB$ORIG_TRANSID: 1279907607296
id: 1
col1$OLD: Sun
col1$NEW: Jupiter
1 row in set (0.00 sec)
##### SECONDARY #####
mysqlS> SELECT * FROM t_conflict;
+----+------+---------------------+
| id | col1 | col2                |
+----+------+---------------------+
|  1 | Moon | 2017-05-06 12:37:50 |
+----+------+---------------------+
1 row in set (0.00 sec)
mysqlS> SELECT * FROM t_conflict$EX;
Empty set (0.01 sec)
```

清单 6-20. 使用 NDB$EPOCH_TRANS 方法测试冲突检测和解决

请注意从此测试示例中得出的以下事实：


## 使用 NDB$EPOCH2 方法设置冲突检测

为 MySQL NDB Cluster 7.4 系列新增的 `NDB$EPOCH2` 方法设置过程与 `NDB$EPOCH_TRANS` 方法类似。其配置方式与 `NDB$EPOCH` 和 `NDB$EPOCH_TRANS` 略有不同。

第一个区别在于服务器配置。代码清单 6-21 展示了在 `cluster1` 上充当主从角色的 SQL 节点的配置示例。

```
[mysqld]
ndbcluster
ndb_connectstring = mgmhost
log_bin = mysql-bin
log_slave_updates
server_id = 1001
... snip ...
# Additional configurations for NDB$EPOCH_TRANS method
ndb_log_update_as_write = OFF
ndb_log_updated_only = OFF
ndb_log_apply_status = ON
ndb_slave_conflict_role = PRIMARY
代码清单 6-21.
NDB$EPOCH2 方法的 SQL 节点配置示例
```

请注意，此 SQL 节点的角色是使用 `ndb_slave_conflict_role` 选项定义的，而不是通过 `mysql.ndb_replication` 表中的条目。另一方面，别忘了为辅助集群上的 SQL 节点指定 `ndb_slave_conflict_role = SECONDARY`。必须在两个集群上显式设置角色。

此配置不包括在 `NDB$EPOCH_TRANS` 方法配置示例中出现的 `log_bin_use_v1_row_events = OFF` 和 `ndb_log_transaction_id = ON`，因为该方法是 `NDB$EPOCH2`，它不是一个事务单元方法。

下一步是向 `mysql.ndb_replication` 表添加一个条目，如代码清单 6-22 所示。

```
mysql> INSERT INTO mysql.ndb_replication VALUES ('test', 't_conflict', 0, NULL, 'NDB$EPOCH2(7)');
Query OK, 1 row affected (0.00 sec)
代码清单 6-22.
为 NDB$EPOCH2 方法添加必需条目到 mysql.ndb_replication 表
```

请注意，`server_id` 列设置为 0。这表示冲突检测和解决将在所有配置为复制从节点的 SQL 节点上执行。`NDB$EPOCH2` 方法的不对称性由 `ndb_slave_confict_role` 选项保证，而不是 `mysql.ndb_replication` 表中的 `server_id` 列。

像 `NDB$EPOCH_TRANS` 示例一样创建异常表和目标表。然后，针对目标 `test.t_conflict` 表激活冲突检测和解决。

##### 冲突检测所需的应用程序修改

虽然循环 NDB 集群复制的冲突检测和解决功能在您需要从两个集群更新相同数据时非常方便，但它并非没有代价。您需要修改应用程序以适应冲突检测和解决。在本节中，我们将讨论如何调整您的应用程序以适应冲突检测和解决。

##### 选择正确的冲突检测方法

使您的应用程序适应冲突检测和解决的第一步是选择合适的方法。正如本章所讨论的，方法有两类——基于时间戳的方法和基于纪元的方法。表 6-8 列出了基于时间戳和基于纪元的方法之间的主要区别。

表 6-8.
基于时间戳与基于纪元方法的主要区别

| 特性 | 基于时间戳的方法 | 基于纪元的方法 |
| --- | --- | --- |
| 支持的集群数量 | 循环 NDB 集群复制中的任意数量集群。 | 两个 |
| 集群对称性 | 对称；冲突可能在两个集群上被检测到。 | 不对称；集群将具有主或辅助角色。主节点始终获胜。 |
| 表结构变更 | 需要；需要添加时间戳列。 | 不需要，但表内部存储额外的位。 |
| 冲突解决单位 | 行；仅解决冲突的行。可能破坏事务一致性。 | 行或事务；事务单元方法可以确保事务一致性。 |
| 冲突解决是否自动？ | 对于 `NDB$MAX` 和 `NDB$MAX_DELETE_WIN` 是。对于 `NDB$OLD` 否。 | 是。 |

当您决定使用基于纪元的方法之一时，我推荐较新的版本 `NDB$EPOCH2` 和 `NDB$EPOCH2_TRANS`（优于 `NDB$EPOCH` 和 `NDB$EPOCH_TRANS`），因为新方法易于配置并具有删除-删除冲突处理功能。

##### 更新时间戳列

使用基于时间戳的冲突检测和解决时，填充时间戳列是应用程序的责任。应用程序必须在更新表时随时更新时间戳列。否则，冲突无法被检测到。这需要额外的开发工作量。

更重要的是，`NDB$MAX` 和 `NDB$MAX_DELETE_WIN` 方法需要外部序列生成器。这需要额外的开发和运维成本。

触发器对于填充或更新时间戳值很有用。对于 `NDB$OLD` 方法，实现一个在 `UPDATE` 时递增时间戳列的触发器就足够了。`INSERT` 触发器不是必需的，因为在执行 `CREATE TABLE` 时可以指定列的默认值。对于 `NDB$MAX` 和 `NDB$MAX_DELETE_WIN` 方法，开发一个用户定义函数来从外部序列生成器检索新的时间戳值是个好主意，因为它可以从 `UPDATE` 触发器中调用。

##### 监控冲突检测

除非您的应用程序可以忽略所有已解决的冲突，否则您需要监控冲突的状态，如本节前面所述。由于检测到冲突时没有推送式通知，您的应用程序必须定期监控冲突。

##### 修复冲突

在某些情况下，自动冲突解决对您的应用程序来说还不够，您的应用程序需要采取进一步的措施，例如：

*   通知用户他们的更新可能被取消。
*   查询另一个集群以确保修改未被识别为冲突。
*   验证数据库是否一致且没有违反约束。必要时调整数据。

存储在异常表中的信息在对冲突采取进一步措施时很有用。处理完冲突后，您可以删除异常表中的行。如果您愿意，也可以保留异常表中的行以供后续审查。

#### 冲突检测的注意事项与限制

尽管冲突检测是一项方便的功能，但它存在一些缺点和限制。因此，即使冲突检测和解决可用，使用多主 NDB 集群复制时也需要格外小心和付出额外努力。

#### 了解更多

*   两个集群上的表即使更新为不同的值也具有相同的行值。这意味着冲突已解决，并且当行同时从两个集群更新时，主集群的修改获胜。
*   只有 `cluster1` 上的异常表有一个条目表明发生了冲突。这意味着冲突检测和解决仅在主集群上完成，如 `mysql.ndb_replication` 表中所定义。

如果您想进一步测试，请将以下过程作为一课来尝试。

*   使用带有 `-vv` 选项的 `mysqlbinlog` 命令检查二进制日志。查看两个集群上写入了哪些事件。请注意，每指定一次该选项，详细级别就会增加。如果详细级别为 1 或更大，则打印行值。如果详细级别为 2 或更大，则打印额外信息。
*   从异常表中删除行。
*   截断异常表并插入新行。看看会发生什么。
*   测试删除-删除冲突处理。



##### 二进制日志大小

当采用冲突检测与解决机制时，二进制日志将需要更多空间。因此，需要准备比无冲突检测的 NDB 集群复制所需的更多磁盘空间和网络带宽。

将 `ndb_log_update_as_write` 设置为 `OFF` 会使更新操作的二进制日志大小约增加一倍，这一点非常重要。

由于基于时间戳的方法需要在目标表中包含时间戳列，这既增加了表的大小，也增加了二进制日志的大小。

基于纪元的方法需要将 `ndb_log_apply_status` 选项设置为 `ON`，这将在从库的二进制日志中为每个事件占用额外的空间。当使用事务单元方法时，请确保 `log_bin_use_v1_row_events` 设置为 `OFF`，并且 `ndb_log_transaction_id` 设置为 `ON`，这会为每个事件要求额外的字节。

##### 性能开销

每次在从库 SQL 节点上应用二进制日志时，从库 SQL 线程和数据节点都会检查是否发生冲突。与无冲突检测的 NDB 集群复制相比，这需要额外的 CPU 资源。您可能不得不升级作为从库的 SQL 节点的 CPU。

##### 延迟至关重要

当使用冲突检测与解决时，复制延迟变得比无冲突检测的 NDB 集群复制更为关键。延迟时间越长，发生冲突的可能性就越高。因此，在使用冲突检测与解决时，尽快应用复制非常重要。

##### 事务处理

事务必须是原子的；所有修改要么全部应用，要么完全不做。然而，对于行级冲突解决方法，只有导致冲突的行会在事务提交后被修改。这破坏了事务的原子性（即 ACID 中的 "A"），数据库将进入不一致状态。

对于基于纪元的方法，事务在提交到辅助集群后将被取消。这意味着在辅助集群上无法确保持久性（即 ACID 中的 "D"）。

因此，在启用冲突检测与解决的 NDB 集群复制中，不可能充分发挥事务的能力。这将使应用开发比在通常的、完全事务化的 MySQL NDB 集群上开发困难得多。因此，即使可以检测和解决冲突，我也通常不推荐使用多主 NDB 集群复制。只有当您准备好应对因违反事务模型而带来的额外工作时，才使用它。

### 复制到 InnoDB

标准 MySQL 复制支持从一种存储引擎复制到另一种，但有一些限制。也可以配置从 MySQL NDB 集群复制到 `InnoDB`。尽管 MySQL NDB 集群是一个高度可扩展的系统，但需要这种类型的复制来提高可扩展性。存在一些无法通过独立的 MySQL NDB 集群设置解决的问题，例如：

*   对同一数据集的访问无法扩展，因为 MySQL NDB 集群采用无共享架构，数据按行进行水平分布。
*   有序索引扫描扩展性不佳，除非使用用户定义的分区，因为扫描必须涉及所有数据节点才能完成。
*   在 `InnoDB` 上进行分析查询通常比在 MySQL NDB 集群上更快。

复制到 `InnoDB` 是克服这些性能问题的一种便捷方法。

#### 要求与限制

由于 `InnoDB` 和 `NDBCluster` 是不同的存储引擎，从 `NDBCluster` 主库复制到 `InnoDB` 从库有一些要求。请注意，虽然它们是不同的存储引擎，但在复制时并非所有功能都受支持。

##### 使用 MySQL NDB Cluster 捆绑的 mysqld

强烈建议您对主库和从库使用相同的二进制文件。`InnoDB` 存储引擎也包含在 MySQL NDB 集群捆绑的 `mysqld` 中。因此，您不仅可以将其用作 SQL 节点，还可以将其用作从库 MySQL 服务器。与标准 MySQL 服务器软件包中包含的 `mysqld` 程序相比，MySQL NDB 集群捆绑的 `mysqld` 程序具有一些附加功能。由于从库必须重现与主库相同的修改，功能上的差异可能会产生问题。

##### 二进制日志格式要求

在 NDB 集群复制中，主库 SQL 节点以特殊格式存储二进制日志，这种格式只能由 MySQL NDB 集群从库处理。必须让主库 MySQL NDB 集群的二进制日志匹配 `InnoDB`。在 MySQL NDB 集群上，当 `ndb_log_update_as_write` 设置为 `ON`（这是 MySQL NDB 集群的默认设置）时，更新会作为写操作记录到二进制日志中。在从库端，MySQL NDB 集群可以处理这样的二进制日志，不会报告错误。然而，`InnoDB` 不能。要从 MySQL NDB 集群复制到 `InnoDB`，必须在主库 SQL 节点上设置 `ndb_log_update_as_write = OFF`。

在主库 SQL 节点上，由于架构原因，偶尔会将重复条目记录到二进制日志中，这会导致在插入时出现 `ER_DUP_ENTRY` 错误，或在删除时出现 `ER_KEY_NOT_FOUND` 错误。为防止从库 SQL 节点出现问题，必须设置 `slave_exec_mode = IDEMPOTENT`。此选项的默认值是 `STRICT`，这些错误将被当作实际错误处理。当此选项设置为 `IDEMPOTENT` 时，从库 SQL 线程模拟 NDB 集群复制从库的行为。它忽略这些错误，并且写行事件（基于行格式的 "插入"）被视为 `REPLACE` 命令处理。

##### MySQL NDB 集群系统表

在 `mysql` 数据库中有几个系统表。如本章所述，主库 SQL 节点生成事件来更新 `mysql.ndb_apply_status` 表。当启用索引统计功能时，`ANALYZE TABLE` 命令会更新两个系统表 `mysql.ndb_index_stat_head` 和 `mysql.ndb_index_stat_sample`。这些表在 SQL 节点连接到集群时创建。因此，它们不会在 `InnoDB` 从库上自动创建。您必须手动创建它们，或者设置复制过滤器来过滤它们。但是，我建议不要过滤 `mysql.ndb_apply_status` 表。当您切换到备用复制通道时，需要此表。

##### 环形复制与冲突检测

配置涉及 MySQL NDB 集群和 `InnoDB` 的环形复制在技术上可能是可行的。但是，这种配置不受支持。我不建议使用跨存储引擎的环形复制。

此外，`InnoDB` 不支持冲突检测。这是 MySQL NDB 集群独有的功能。因此，无法检测或避免由环形拓扑引起的潜在冲突。


##### 外键

外键是确保多个表之间引用约束的有用特性。然而，外键的实现方式取决于存储引擎，且在 `InnoDB` 和 `NDBCluster` 之间有所不同。例如，检查外键约束的时机不同：`NDBCluster` 存储引擎在提交时检查，而 `InnoDB` 在每条语句后检查。这可能导致复制失败。

要防止此问题，请从从库的 `InnoDB` 表中移除外键。当从库移除外键后，从库上就不会执行任何约束检查。不过，约束检查应该已在主库上执行过了。传播到从库的修改必须没有违反约束。

唯一剩下的问题是外键导致的级联更新和删除。级联更新和删除是在存储引擎内部完成的。因此，由级联操作引起的修改不会写入二进制日志。这假设了在从库上必须执行与主库相同的级联操作。当主库表上有外键而从库表上没有时，这就会产生问题。如果你是从 MySQL NDB Cluster 复制到 `InnoDB`，请不要使用外键的级联更新和删除。

##### 唯一键

NDB Cluster 复制通过延迟约束检查直到事务提交来避免潜在的唯一键约束冲突。在很旧的版本中，唯一键约束曾破坏过复制，但在最近的版本中已不是问题。然而，当复制到 `InnoDB` 时，这仍然是个问题，因为 `InnoDB` 在每条语句后都会检查唯一键约束。因此，请在从库端移除唯一键约束，因为可以假设所有来自主库的修改在每次事务提交时都不会违反唯一键约束。

##### 行大小限制

虽然 MySQL NDB Cluster 的行容量最高可达 14KB，但 `InnoDB` 默认是 8KB。当表的行大小超过 8KB 时，这可能会导致问题。在这种情况下，请使用 32KB 或 64KB 的 `innodb_page_size`。

#### 设置到 InnoDB 的复制

在本节中，我们将讨论如何设置从 MySQL NDB Cluster 到 `InnoDB` 的复制。

##### 配置主 SQL 节点并创建复制用户

你需要启用二进制日志并显式设置 `server_id`。别忘了设置 `ndb_log_update_as_write = OFF`，这是从 `NDBCluster` 主库复制到其他存储引擎从库所必需的。清单 6-23 展示了主 SQL 节点的配置示例。

```
[mysqld]
ndbcluster
... snip ...
server_id = 1001
log_bin = mysql-bin
binlog_format = ROW
ndb_log_update_as_write = OFF
```
清单 6-23. 复制到 InnoDB 时主 SQL 节点的示例配置

如果不存在合适的用户，请创建一个从库用户。有关示例，请参考清单 6-2。

##### 为复制配置从库

在从库端，必须显式设置 `server_id`，并且必须更改 `slave_exec_mode` 的默认值。清单 6-24 展示了一个 `InnoDB` 从库的配置示例。

```
[mysqld]
server_id = 2001
slave_exec_mode = IDEMPOTENT
replicate_wild_ignore_table = mysql.ndb_index%
```
清单 6-24. 从 MySQL NDB Cluster 主库复制时，从库 MySQL 服务器的示例配置

如果需要，你可以设置额外的复制过滤器。然后，启动从库 MySQL 服务器，或者如果它已在运行，则重启它。

##### 从主库获取备份

在主集群上执行 `START BACKUP` 命令以进行原生在线备份。有关原生备份的更多信息，请参阅第 8 章。除非确保在备份期间没有进行任何修改，否则原生备份是从在线 MySQL NDB Cluster 获取备份的唯一方式。清单 6-25 展示了一个原生备份的输出示例。

```
ndb_mgm> START BACKUP
Connected to Management Server at: mgmhost
Waiting for completed, this may take several minutes
Node 1: Backup 1 started from node 255
ndb_mgm> Node 1: Backup 1 started from node 255 completed
StartGCP: 527473 StopGCP: 527476
#Records: 18444 #LogRecords: 112
Data: 575936 bytes Log: 4536 bytes
```
清单 6-25. 原生备份的示例输出

请注意，备份完成时控制台打印的 `StopGCP` 值。这个值在后续步骤中是必需的。在此示例中，`StopGCP` 是 `527476`。

我建议使用 `mysqldump` 命令为模式再做一次备份，因为基于 SQL 的 DDL 在 `InnoDB` 从库上更容易恢复。使用 `mysqldump` 命令时指定 `--no-data` 选项以跳过数据备份。清单 6-26 展示了一个从主 SQL 节点获取模式备份的示例。

```
shell$ mysqldump -h masterhost -uroot -p db_name --no-data > dump.sql
```
清单 6-26. 从主 SQL 节点获取模式备份

##### 将模式恢复到从库

使用 `mysqldump` 命令生成的转储文件恢复模式。在恢复到从库 SQL 节点之前，请更改转储文件中的存储引擎。在类 UNIX 系统上，这可以使用 `sed` 命令完成，如下所示：

```
shell$ sed -i 's/ENGINE=ndbcluster/ENGINE=InnoDB/g' dump.sql
```

然后，像往常一样将转储文件恢复到 MySQL 服务器；例如，使用 `mysql` CLI 中的 `SOURCE` 命令：

```
mysql> SOURCE dump.sql
```

检查所有表是否已在从库上创建。


##### 将数据恢复到从库

这是此过程中最棘手的步骤。一个原生 NDB 备份由两部分组成：数据和日志。两者都必须被恢复，才能在某个特定时间点恢复一致的快照。有两种方法可以将备份恢复到 `InnoDB` 从库。

一种方法是使用一个中间临时集群。先将原生备份恢复到临时集群，确保没有进行任何修改，然后使用 `mysqldump` 命令从临时集群中获取一个备份。使用 `mysqldump` 生成的转储文件可以像处理标准 MySQL 服务器一样进行恢复。

另一种方法是将数据部分转换为制表符分隔的文件，然后使用 `LOAD DATA INFILE` 命令恢复。转换可以使用 `ndb_restore` 命令完成。所需的选项是 `--print-data`、`--tab` 和 `--append`。`--print-data` 选项指示 `ndb_restore` 命令打印数据，而不是将其恢复到正在运行的集群。`--tab` 选项指定创建制表符分隔文件的目标目录；在此目录下，每个表都会创建一个制表符分隔文件。`--append` 选项表示当制表符分隔文件已存在时，不会覆盖它们，而是追加新数据。

清单 6-27 展示了一个将备份转换为制表符分隔文件的示例。

```
shell$ ndb_restore -n 1 -b 1 --print-data --tab=/backup/tab-files --append \
/path/to/backup
清单 6-27.
使用 ndb_restore 命令将原生备份转换为制表符分隔文件
```

由于原生备份是针对每个数据节点创建的，因此需要对所有数据节点重复相同的命令。`ndb_restore` 命令中的 `-n` 选项指定生成备份的节点 ID，`-b` 选项指定分配给每个备份的备份 ID。有关 `ndb_restore` 命令的更多信息，请参阅第 8 章。然后，使用 `LOAD DATA INFILE` 命令将制表符分隔的文件恢复到 `InnoDB` 从库中。确保在恢复日志部分之前，数据部分已完全恢复。

下一步是恢复原生备份中包含的日志文件。从 MySQL NDB Cluster 7.5.4 开始，`ndb_restore` 命令可以生成可直接在 MySQL 服务器上执行的 SQL 语句。要生成可执行的 SQL 日志，请指定 `--print-sql-log` 选项。即使指定了此选项，也会打印不必要的标题和页脚行。要抑制这些行，请按清单 6-28 所示执行 `ndb_restore` 命令。

```
shell$ ndb_restore -n 1 -b 1 --print-sql-log /path/to/backup \
| egrep '^INSERT|^DELETE|^UPDATE' >> dump-log.sql
清单 6-28.
将原生备份日志转换为 SQL 格式
```

对所有数据节点重复相同的命令。然后，在从服务器上执行生成的 SQL 文件。由于 `ndb_restore` 命令可用于针对较旧版本创建的备份，因此你可以使用 7.5.4 或更新版本捆绑的 `ndb_restore` 命令将备份日志转换为 SQL 格式。

##### 创建系统表

创建 `mysql.ndb_apply_status` 表，其结构与主库相同，但存储引擎除外。使用 `InnoDB` 存储引擎而不是 `NDBCluster` 来创建它。

##### 设置复制

最后，使用 `CHANGE MASTER TO` 命令设置复制。第一步是从主库 SQL 节点的 `mysql.ndb_binlog_index` 表中检索二进制日志文件名和位置。所需的信息是备份时检索到的 `StopGCP` 值。清单 6-29 展示了一个检索二进制日志文件名和位置的示例查询。

```
mysql> SET @stopgcp = 527476;
mysql> SELECT
-> SUBSTRING_INDEX(File, '/', -1) AS binlog_file,
-> Position AS binlog_position
-> FROM mysql.ndb_binlog_index
-> WHERE gci > @stopgcp
-> ORDER BY epoch ASC LIMIT 1;
清单 6-29.
使用 StopGCP 从 mysql.ndb_binlog_index 表中检索二进制日志文件名和位置
```

在 `CHANGE MASTER TO` 命令中指定此查询检索到的文件名和位置。然后，执行 `START SLAVE` 命令启动复制，并使用 `SHOW SLAVE STATUS` 命令检查复制状态。

#### 使用 InnoDB 作为从库时的技巧

使用 `InnoDB` 作为 MySQL NDB Cluster 的从库时，有几个技巧。

##### 从备用通道恢复复制

即使主库 SQL 节点在从库是 `InnoDB` 的情况下也可能因各种原因（例如主库 SQL 节点离线）导致二进制日志中缺失数据。因此，主库 SQL 节点需要冗余。在发生不可恢复的复制故障（如 `LOST_EVENTS` 事件）时，就像在 NDB Cluster 到 NDB Cluster 复制中一样，使用存储在 `mysql.ndb_apply_status` 表中的纪元信息切换到备用的主库 SQL 节点。

##### 通过跳过日志同步来加速更新

通常，MySQL NDB Cluster 的写入速度比 `InnoDB` 快。这将导致 `InnoDB` 从库上出现不必要的复制延迟。为了防止 `InnoDB` 从库上的延迟，我建议在从库上将 `innodb_flush_log_at_trx_commt` 设置为 0 或 2。这将跳过在事务提交时将 `InnoDB` 日志同步到磁盘。此设置通常不推荐使用，因为未同步到磁盘的最后提交的事务可能在机器故障时丢失。

由于 `mysql.ndb_apply_status` 表是在与其他表相同的事务中更新的，因此你可以在发生崩溃时使用表中的纪元安全地重启复制。即使由于某种原因无法重启复制，你也可以使用主库数据再次设置 `InnoDB` 从库。甚至，如果你有多个从服务器，也可以使用带有 `--dump-slave` 选项的 `mysqldump` 命令以及其他标准方法（如 MySQL Enterprise Backup）来克隆从库。


### MySQL 服务器集群复制相关选项

供参考，与 NDB 集群复制相关的 MySQL 服务器选项列于表 6-9 中。

表 6-9. `mysqld` 中与 NDB 集群复制相关的选项列表

| 选项名 | 默认值 | 描述 |
| --- | --- | --- |
| `log_bin` | 无 | 指定后启用二进制日志。此选项的参数用作二进制日志文件的基础名称。 |
| `sync_binlog` | 0 (<= 7.4) 1 (>= 7.5) | 在处理此选项指定的事件数后将二进制日志同步到磁盘。在 MySQL NDB 集群上，请将此选项设置为更大的值，例如 1000。 |
| `log_slave_updates` | `OFF` | 在从服务器上启用时，所有从主服务器传播的修改都会记录在从服务器的二进制日志中。 |
| `ndb_log_bin` | `ON` | 启动二进制日志注入器线程。 |
| `server_id` | 无 | 分配给服务器的服务器标识符。 |
| `server_id_bits` | 32 | `server_id` 的有效位数。此选项的范围是 7 ∼ 32。 |
| `binlog_format` | `STATEMENT` (<= 7.4) `ROW` (>= 7.5) | 二进制日志的格式。NDB 集群复制仅支持行格式。 |
| `expire_logs_days` | 无 | 在此选项指定的天数后自动删除二进制日志。 |
| `ndb_log_updated_only` | `ON` | 启用后，仅记录行中修改的部分到二进制日志。使用冲突检测或 `InnoDB` 从服务器时必须设为 `OFF`。 |
| `ndb_log_update_as_write` | `ON` | 启用后，对 NDB 表的更新在二进制日志中记录为写入。使用冲突检测或 `InnoDB` 从服务器时必须设为 `OFF`。 |
| `ndb_log_apply_status` | `OFF` | 启用后，来自主直接主服务器的针对 `ndb_apply_status` 表的修改会写入二进制日志。使用基于纪元的冲突检测方法时必须设为 `OFF`。 |
| `ndb_log_binlog_index` | `ON` | 纪元与二进制日志位置之间的映射被写入 `ndb_binlog_index` 表。 |
| `slave_allow_batching` | `OFF` | 启用后，更新会在从服务器上以批处理方式应用。这将提高从服务器性能。 |
| `log_bin_use_v1_row_events` | `OFF` | 禁用时，使用二进制日志格式的版本 2。`ndb_log_transaction_id` 选项必需。 |
| `ndb_log_transaction_id` | `OFF` | 启用后，`NDBCluster` 存储引擎的事务 ID 会写入每个二进制日志条目。基于纪元的冲突检测方法需要。 |
| `ndb_slave_conflict_role` | 无 | 指定服务器的角色为 `PRIMARY` 或 `SECONDARY`。`NDB$EPOCH2` 和 `NDB$EPOCH2_TRANS` 冲突检测方法需要。 |
| `ndb_log_exclusive_reads` | `OFF` | 启用后，排他读操作会被写入二进制日志，以便进行冲突检测。 |

### NDB 集群复制的注意事项与限制

NDB 集群复制存在若干限制。以下功能在 NDB 集群复制上不可用。如果需要使用这些功能，请考虑使用 `InnoDB` 和标准 MySQL 复制代替 NDB 集群复制。

*   GTID（全局事务标识符）
*   多线程从服务器
*   多源复制
*   组复制

使用复制时，请确保所有表都有显式主键，因为如果某些表没有显式主键，在发生节点故障时可能会在复制中出错。当应用修改而扫描表时，如果表没有显式主键也会出现问题。总之，从开发和复制的角度来看，强烈建议所有表都拥有主键。

### 总结

NDB 集群复制是一项强大的功能，可补充 MySQL NDB 集群的多个方面。例如：

*   灾难恢复或备用集群；
*   相同数据集的读取扩展；
*   跨存储引擎复制；

这使得 MySQL NDB 集群在各种情况下更加有用。

从下一节开始，我们将讨论日常任务和维护相关主题。为了保持数据库集群的健康，应每天对其进行良好的维护。维护任务的第一个主题是 MySQL NDB 集群的客户端和实用程序。

## 第三部分 日常任务与维护

## 7. NDB 管理客户端与其他 NDB 实用程序

MySQL NDB 集群附带了一系列用于管理、获取集群信息以及进行故障排除的实用程序。最常用的实用程序是 NDB 管理客户端 `ndb_mgm`，它连接到管理节点，可用于获取状态信息、创建备份等。本章详细介绍管理客户端，并概述其他实用程序。

### NDB 管理客户端

NDB 管理客户端很特殊，因为它是唯一一个专门与管理节点通信的实用程序。所有数据节点、管理节点和 API/SQL 节点都需要节点 ID 才能连接，而 NDB 管理客户端不需要节点 ID。这意味着即使所有配置的 API 节点 ID 都已用尽，也总是可以使用管理客户端连接到集群。管理客户端可以执行的一些任务包括：

*   启动处于“未启动”状态的数据节点。示例请参见第 10 章。
*   停止单个管理或数据节点、所有数据节点或所有管理和数据节点。详情请参见第 10 章。
*   重启管理或数据节点。示例请参见第 10 章。
*   启动和中止在线 NDB 备份。详情请参见第 8 章。
*   显示集群状态信息，包括哪些节点在线以及接受来自哪些主机的连接。
*   管理集群日志。详情请参见第 16 章。
*   创建报告。
*   创建或删除节点组。创建新节点组的示例请参见第 10 章。
*   进入和退出单用户模式。
*   清除陈旧的会话。
*   设置用于提示符的文本。

本节的其余部分讨论 `ndb_mgm` 客户端最常见的用法，不包括其他章节讨论的情况，例如创建备份。

#### 调用 NDB 管理客户端

启动管理客户端时只有少数几个选项可用，其中最重要的如下（按选项名称的长格式字母顺序排列）：

*   `--defaults-extra-file=...`。除了默认配置文件外，还从该文件读取配置选项。另请参见 `--defaults-file` 选项。
*   `--defaults-file=...`。其工作方式与其他所有 MySQL 程序相同。`ndb_mgm` 将读取 `[mysql_cluster]` 和 `[ndb_mgm]` 组。读取配置文件对于设置 `ndb-connectstring` 选项可能很有用。
*   `--execute=.../-e`。要执行的命令。其工作方式与 `mysql` 命令行客户端相同。所有可以通过管理客户端交互执行的命令，也可以直接使用 `-e` 或 `--execute=...` 命令行选项执行。如果命令包含多个单词，则必须用引号将命令括起来，或用反斜杠 (`\`) 转义空格。例如：`ndb_mgm -e "START BACKUP"`。以这种方式执行命令的一个优点是，输出可以重定向到其他程序或文件。
*   `--help`。显示有关可用选项和默认值的信息。
*   `--ndb-connectstring=.../-c`。以分号分隔的管理节点主机名和端口列表。每个管理节点的格式为 `hostname:port`。默认值为 `localhost:1186`。详见第 4 章。
*   `--no-defaults`。不读取任何配置文件。另请参见 `--defaults-file`。
*   `--prompt=.../-p`。指定用于提示符的文本。默认为 `ndb_mgm>`。此功能在 MySQL NDB Cluster 7.5 中是新的。其用法类似于设置 `mysql` 命令行客户端的提示符，但不支持像 `\c` 这样用于添加计数器的特殊序列。

要获取有关命令行的更多信息，包括此处未提及的选项，请使用 `--help` 参数，如代码清单 7-1 所示。

```
shell$ ndb_mgm --help
Usage: ./mysql/bin/ndb_mgm [OPTIONS] [hostname [port]]
MySQL distrib mysql-5.7.18 ndb-7.5.6, for linux-glibc2.5 (x86_64)
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/local/mysql/etc/my.cnf ∼/.my.cnf
The following groups are read: mysql_cluster ndb_mgm
The following options may be given as the first argument:
--print-defaults        Print the program argument list and exit.
--no-defaults           Don't read default options from any option file,
except for login file.
--defaults-file=#       Only read default options from the given file #.
--defaults-extra-file=# Read this file after the global files are read.
--defaults-group-suffix=# Also read groups with concat(group, suffix)
--login-path=#          Read this path from the login file.
-?, --usage         Display this help and exit.
-?, --help          Display this help and exit.
-V, --version       Output version information and exit.
-c, --ndb-connectstring=name
Set connect string for connecting to ndb_mgmd. Syntax:
"[nodeid=;][host=][:]". Overrides
specifying entries in NDB_CONNECTSTRING and my.cnf
--ndb-mgmd-host=name same as --ndb-connectstring
--ndb-nodeid=#      Set node id for this node. Overrides node id specified in
--ndb-connectstring.
--ndb-optimized-node-selection
Select nodes for transactions in a more optimal way
(Defaults to on; use --skip-ndb-optimized-node-selection to disable.)
-c, --connect-string=name
same as --ndb-connectstring
--core-file         Write core on errors.
--character-sets-dir=name
Directory where character sets are.
--connect-retry-delay=#
Set connection time out. This is the number of seconds
after which the tool tries reconnecting to the cluster.
--connect-retries=# Set connection retries. This is the number of times the
tool tries connecting to the cluster.
-e, --execute=name  execute command and exit
-p, --prompt=name   Set prompt to string specified
-v, --verbose=#     Control the amount of printout
-t, --try-reconnect=#
Same as --connect-retries
Variables (--variable-name=value)
and boolean options {FALSE|TRUE}  Value (after reading options)
--------------------------------- ----------------------------------------
ndb-connectstring                 (No default value)
ndb-mgmd-host                     (No default value)
ndb-nodeid                        0
ndb-optimized-node-selection      TRUE
connect-string                    (No default value)
core-file                         FALSE
character-sets-dir                (No default value)
connect-retry-delay               5
connect-retries                   12
execute                           (No default value)
prompt                            (No default value)
verbose                           1
try-reconnect                     12
代码清单 7-1.
ndb_mgm --help 的输出
```


#### 从客户端内部获取帮助

管理客户端所支持命令的文档可在 MySQL 参考手册中找到，地址是 [`https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster-mgm-client-commands.html`](https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster-mgm-client-commands.html)。获取快速参考的一个更简便方法是使用管理客户端内部的 `HELP` 命令。帮助信息概览如清单 7-2 所示。这与上一小节使用 `ndb_mgm --help` 所提供的帮助不同，它提供的是关于可用命令的帮助。

```
ndb_mgm> HELP

NDB Cluster -- Management Client -- Help

HELP                                   打印帮助文本
HELP COMMAND                           打印 COMMAND（例如 SHOW）的详细帮助
SHOW                                   打印集群信息
CREATE NODEGROUP ,...          添加包含节点的节点组
DROP NODEGROUP                     删除 ID 为 NG 的节点组
START BACKUP [NOWAIT | WAIT STARTED | WAIT COMPLETED]
START BACKUP [] [NOWAIT | WAIT STARTED | WAIT COMPLETED]
START BACKUP [] [SNAPSHOTSTART | SNAPSHOTEND] [NOWAIT | WAIT STARTED | WAIT COMPLETED]
启动备份（默认为 WAIT COMPLETED，SNAPSHOTEND）
ABORT BACKUP                中止备份
SHUTDOWN                               关闭集群中的所有进程
PROMPT []               在指定的字符串和默认提示符之间切换提示符（如果未指定字符串）
CLUSTERLOG ON [] ...         启用集群日志记录
CLUSTERLOG OFF [] ...        禁用集群日志记录
CLUSTERLOG TOGGLE [] ...     切换严重性过滤器的开/关状态
CLUSTERLOG INFO                        打印集群日志信息
 START                             启动数据节点（使用 -n 启动）
 RESTART [-n] [-i] [-a] [-f]       重启数据节点或管理服务器节点
 STOP [-a] [-f]                    停止数据节点或管理服务器节点
ENTER SINGLE USER MODE             进入单用户模式
EXIT SINGLE USER MODE                  退出单用户模式
 STATUS                            打印状态
 CLUSTERLOG {=}+  为集群日志设置日志级别
PURGE STALE SESSIONS                   重置管理服务器中保留的 nodeid
CONNECT []              连接到管理服务器（如果已连接则重新连接）
 REPORT               显示 的报告
QUIT                                   退出管理客户端
 = ALERT | CRITICAL | ERROR | WARNING | INFO | DEBUG
 = STARTUP | SHUTDOWN | STATISTICS | CHECKPOINT | NODERESTART | CONNECTION | INFO | ERROR | CONGESTION | DEBUG | BACKUP | SCHEMA
 = BACKUPSTATUS | MEMORYUSAGE | EVENTLOG
    = 0 - 15
       = ALL | 任何数据库节点 ID
有关 COMMAND 的详细帮助，请使用 HELP COMMAND 。
清单 7-2.
ndb_mgm 中的 HELP 命令
```

提示

与 `mysql` 命令行客户端不同，此客户端不支持多行语句，因此不需要分隔符。这意味着你可以在管理客户端中执行命令，而无需在末尾指定分号 (`;`)。与 SQL 语句类似，命令不区分大小写。

正如 `HELP COMMAND` 命令的描述所述，你也可以获取给定命令的更详细帮助。例如，要了解更多关于 `START BACKUP` 命令的信息，请使用此命令：

```
ndb_mgm> HELP START BACKUP

NDB Cluster -- Management Client -- Help for START BACKUP command

START BACKUP  启动集群备份
START BACKUP [] [SNAPSHOTSTART | SNAPSHOTEND] [NOWAIT | WAIT STARTED | WAIT COMPLETED]
为集群启动备份。
...
```

使用内置帮助是验证命令语法和用法的非常有用的方法。

#### 设置提示符

在 MySQL NDB Cluster 7.5 及更高版本中，可以更改提示符。默认提示符是 `ndb_mgm>`。但是，如果你管理多个集群，将提示符设置为不同的值有助于降低在错误集群上执行命令的风险。

要更改提示符，请使用 `PROMPT` 命令，后跟你想要使用的字符串。该字符串不应加引号。例如，要将提示符设置为 `mgm - production>`，请使用以下命令：

```
ndb_mgm> PROMPT mgm - production>
Prompt set to mgm - production>
mgm - production>
```

末尾不需要空格，因为提示符文本末尾的所有空白字符都会被自动修剪，然后会自动添加一个空格。

要将提示符重置为默认值，请执行不带参数的 `PROMPT` 命令：

```
mgm - production> PROMPT
Returning to default prompt of ndb_mgm>
ndb_mgm>
```

在 MySQL NDB Cluster 7.5 中，提示符也可以在命令行上设置，因此也可以通过 MySQL 配置文件设置：

```
shell$ ndb_mgm --prompt="mgm - production>"
Connected to Management Server at: localhost:1186
Prompt set to mgm - production>
-- NDB Cluster -- Management Client --
mgm - production>
```

#### 显示集群状态

管理客户端中最常用的命令之一是 `SHOW` 命令。它提供集群状态的概览。此外，`STATUS` 命令提供更简单的输出，并且只包含数据节点。`SHOW` 命令不接受任何参数，因此其用法非常简单：

```
ndb_mgm> SHOW
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

对于在线节点，信息包括以下数据：

*   为该节点分配的节点 ID。
*   节点连接的主机。
*   使用的版本。
*   对于数据节点，该数据节点所属的节点组。
*   对于数据节点，当前主节点（president 节点）旁边会有一个星号 (`*`)。在此示例中，`id = 1` 是主节点。
*   如果数据节点正在重启，这将会反映出来，但不会显示上次完成的启动阶段的详细信息（参见第 10 章）。

对于当前未使用的插槽，信息包括：

*   可用的节点 ID。如果配置节点时没有明确的 NodeId，管理节点仍会为该插槽分配一个节点 ID。
*   允许连接的来源。如果未配置 HostName，文本将显示允许来自任何主机的连接。

`SHOW` 命令的示例可在第 10 章和第 11 章中找到。第 10 章涵盖重启操作，第 11 章涵盖升级和降级。

`STATUS` 命令对于快速获取一个或所有数据节点的状态概览非常有用。该命令需要一个参数——添加在 `STATUS` 关键字之前——该参数必须是 `ALL` 或一个节点 ID。由于状态信息包含关于最新重启阶段的信息，因此它对于监控数据节点的重启很有用。例如，获取节点 2 的状态：

```
ndb_mgm> 2 STATUS
Node 2: starting (Last completed phase 4) (mysql-5.7.16 ndb-7.5.4)
```

获取所有数据节点的状态：

```
ndb_mgm> ALL STATUS
Node 1: started (mysql-5.7.16 ndb-7.5.4)
Node 2: starting (Last completed phase 100) (mysql-5.7.16 ndb-7.5.4)
```


#### 单用户模式

单用户模式用于某些维护场景，此时必须确保应用程序未连接到数据节点，或者仅通过单个 API 节点连接。在单用户模式下，只允许一个 API/SQL 节点连接。这也意味着，如果一个 SQL 节点配置的 `ndb_cluster_connection_pool` 值大于一，它将无法连接，因为它需要所有请求的连接槽都可用。如果 SQL 节点配置为使用大于一的连接池，而单用户模式维护窗口又需要使用 SQL 节点，解决方案是：要么使用一个专为此类情况准备的备用 SQL 节点，要么以 `ndb_cluster_connection_pool = 1` 的配置重启一个现有的 SQL 节点。

进入单用户模式的命令是 `ENTER SINGLE USER MODE`，该命令需要指定允许连接到集群的节点 ID。例如，要允许节点 ID 为 54 的节点连接，而不允许其他任何 API/SQL 节点连接，请使用以下命令：

```
ndb_mgm> ENTER SINGLE USER MODE 54;
Single user mode entered
Access is granted for API node 54 only.
```

该命令可能需要一点时间才能完成，在此期间其他节点会被断开连接。在单用户模式下，`SHOW` 和 `STATUS` 命令会反映当前状态，例如：

```
ndb_mgm> ALL STATUS
Node 1: single user mode (mysql-5.7.16 ndb-7.5.4)
Node 2: single user mode (mysql-5.7.16 ndb-7.5.4)
```

一旦维护窗口结束，请使用 `EXIT SINGLE USER MODE` 命令恢复正常模式：

```
ndb_mgm> EXIT SINGLE USER MODE
Exiting single user mode in progress.
Use ALL STATUS or SHOW to see when single user mode has been exited.
```

#### 创建报告

管理客户端可用于创建一系列报告，从而为集群提供有用的信息。报告可以针对单个数据节点生成，也可以针对所有数据节点生成。目前有三种可用的报告类型：

*   `MemoryUsage`：数据节点的数据内存和索引内存使用情况。
*   `BackupStatus`：数据节点的备份状态。
*   `EventLog`：来自数据节点事件日志的消息。

`MemoryUsage` 报告是最常用的报告，它提供了关于数据和（唯一哈希）索引当前内存使用情况的信息：

```
ndb_mgm> ALL REPORT MemoryUsage
Node 1: Data usage is 55%(7070 32K pages of total 12800)
Node 1: Index usage is 43%(4459 8K pages of total 10304)
Node 2: Data usage is 55%(7072 32K pages of total 12800)
Node 2: Index usage is 43%(4460 8K pages of total 10304)
```

内存使用信息以百分比和页面数量表示。

`BackupStatus` 报告要么会报告备份未在进行，要么会提供正在进行的备份的进度信息。例如，当没有备份在进行而你请求所有数据节点的备份信息时：

```
ndb_mgm> ALL REPORT BackupStatus
Node 1: Backup not started
Node 2: Backup not started
```

数据节点 1 在备份期间的状态：

```
ndb_mgm> 1 REPORT BackupStatus
Node 1: Local backup status: backup 5 started from node 49
#Records: 556508 #LogRecords: 0
Data: 18873520 bytes Log: 0 bytes
```

有关备份的更多信息，请参见第 8 章。

最后一种报告是 `EventLog` 报告。这可用于获取关于数据节点活动的详细信息，其详细程度高于输出日志（参见第 16 章），且无需登录到数据节点。事件日志是一个环形缓冲区，因此可用的事件数量是有限的。代码清单 7-3 展示了该报告的一个示例。如果选择来自所有数据节点的事件，事件将会交错显示。

```
ndb_mgm> ALL REPORT EventLog
2016-12-18 16:24:13 Node 1: Node 50 Connected
2016-12-18 16:24:13 Node 1: Communication to Node 2 opened
2016-12-18 16:24:14 Node 2: Node 50 Connected
2016-12-18 16:24:14 Node 2: Communication to Node 1 opened
2016-12-18 16:24:14 Node 1: Node 2 Connected
2016-12-18 16:24:14 Node 2: Node 1 Connected
2016-12-18 16:24:17 Node 1: Node 2: API mysql-5.7.16 ndb-7.5.4
2016-12-18 16:24:17 Node 2: Node 1: API mysql-5.7.16 ndb-7.5.4
...
2016-12-18 16:38:39 Node 1: Backup 5 started from node 49
2016-12-18 16:39:28 Node 1: Backup 5 started from node 49 completed. StartGCP: 3596 StopGCP: 3600 #Records: 2423505 #LogRecords: 0 Data: 74515992 bytes Log: 0 bytes
2016-12-18 16:39:31 Node 1: LDM(1): Completed LCP, #frags = 34 #records = 1211469, #bytes = 60693392
2016-12-18 16:39:31 Node 1: LDM(2): Completed LCP, #frags = 34 #records = 1212045, #bytes = 60759696
2016-12-18 16:39:33 Node 2: LDM(1): Completed LCP, #frags = 34 #records = 1211469, #bytes = 60693392
2016-12-18 16:39:33 Node 2: LDM(2): Completed LCP, #frags = 34 #records = 1212045, #bytes = 60759696
...
2016-12-18 16:43:33 Node 2: Trans. Count = 0, Commit Count = 0, Read Count = 0, Simple Read Count = 0, Write Count = 0, AttrInfo Count = 0, Concurrent Operations = 0, Abort Count = 0 Scans = 0 Range scans = 0, Local Read Count = 0 Local Write Count = 0
2016-12-18 16:43:33 Node 1: Operations=0
2016-12-18 16:43:33 Node 1: Global checkpoint 3718 started
2016-12-18 16:43:33 Node 1: Global checkpoint 3718 completed
2016-12-18 16:43:34 Node 1: Trans. Count = 0, Commit Count = 0, Read Count = 0, Simple Read Count = 0, Write Count = 0, AttrInfo Count = 0, Concurrent Operations = 0, Abort Count = 0 Scans = 0 Range scans = 0, Local Read Count = 0 Local Write Count = 0
2016-12-18 16:43:35 Node 1: Global checkpoint 3719 started
2016-12-18 16:43:35 Node 1: Global checkpoint 3719 completed
2016-12-18 16:43:37 Node 1: Global checkpoint 3720 started
2016-12-18 16:43:37 Node 1: Global checkpoint 3720 completed
代码清单 7-3.
EventLog 报告
```

#### 清除陈旧会话

此命令极少需要使用。节点故障后，可能会发生节点 ID 未被释放的情况，这意味着该节点无法重新加入集群。在这种情况下，可以使用 `PURGE STALE SESSIONS` 命令来检查是否有任何节点 ID 可以被释放，如果可以，则释放它们。该命令没有参数。使用该命令的示例如下：

```
ndb_mgm> PURGE STALE SESSIONS
No sessions purged
```

### 其他 NDB 实用工具

MySQL NDB Cluster 下载版中包含多个实用工具。这些工具都安装在 `bin` 目录中。如果你使用 RPM 包安装 MySQL NDB Cluster，对于 7.5 版本，这些工具包含在客户端 RPM 中，例如 `mysql-cluster-community-client-7.5.4-1.el7.x86_64.rpm`，而对于更早的版本，这些工具则包含在服务器 RPM 中。其中一些实用工具的源代码也是使用 `NDB API` 的有用示例。

表 7-1 概述了 7.5.6 版本中包含的实用工具。本书通篇提供了使用其中几个实用工具的示例。

表 7-1.
MySQL NDB Cluster 7.5.6 包含的客户端实用工具概述


| 工具程序 | 描述 |
| --- | --- |
| `ndb_blob_tool` | 可用于检查和修复 BLOB 表。BLOB 表在第 2 章中讨论。 |
| `ndb_config` | 获取当前配置信息以及配置选项的描述。第 10 章包含一个使用 `ndb_config` 的示例。 |
| `ndb_delete_all` | 删除表中的所有行。相当于在 SQL 节点中执行不带 `WHERE` 子句的 `DELETE FROM <table name>`，但使用 NDB API 实现。警告：表中的所有数据都将被删除。 |
| `ndb_desc` | 提供表的详细信息，包括列、索引、分区信息以及 BLOB 表的信息。第 2 章包含几个示例。 |
| `ndb_drop_index` | 从表中删除索引。警告：切勿对 SQL 节点使用的表使用此工具——使用 `ndb_drop_index` 后，该表将无法从 SQL 节点访问！ |
| `ndb_drop_table` | 删除表。通常，使用 `ndb_drop_table` 工具比从 SQL 节点执行 `DROP TABLE` 语句更快。 |
| `ndb_error_reporter` | 从管理节点和数据节点收集日志和跟踪文件。这是一个 Perl 脚本，需要对节点的 ssh 访问权限。另请参见第 17 章。 |
| `ndb_index_stat` | 启用、禁用和更新索引统计信息。另请参见第 9 章。 |
| `ndbinfo_select_all` | 使用 NDB API 查询 `ndbinfo` 模式中的信息。另请参见第 16 章了解有关 `ndbinfo` 模式的信息。 |
| `ndb_mgm` | NDB 管理客户端。 |
| `ndb_move_data` | 使用 NDB API 在两个表之间移动数据。 |
| `ndb_print_backup_file` | 打印有关备份文件的信息。此实用程序不连接到管理节点。 |
| `ndb_print_file` | 输出有关磁盘数据文件的信息。此实用程序不连接到管理节点。 |
| `ndb_print_frag_file` | 输出有关片段列表文件（NDB 文件系统中 D1 和 D2 目录的 DBDIH 子目录下的 S*.FragList 文件）的信息。此实用程序不连接到管理节点。 |
| `ndb_print_schema_file` | 输出有关 NDB 模式文件（NDB 文件系统中 D1 和 D2 目录的 DBDICT 子目录下的 P0.SchemaLog 文件）的信息。此实用程序不连接到管理节点。 |
| `ndb_print_sys_file` | 输出有关 NDB 模式文件（NDB 文件系统中 D1 和 D2 目录的 DBDIH 子目录下的 P0.sysfile 文件）的信息。此实用程序不连接到管理节点。 |
| `ndb_redo_log_reader` | 以人类可读的格式输出重做日志文件（NDB 文件系统中 D8 到 D39 目录下的 S*.FragLog 文件）的内容。此实用程序不连接到管理节点。 |
| `ndb_restore` | 恢复备份。有关详细信息，请参见第 8 章。 |
| `ndb_select_all` | 使用 NDB API 选择表中的所有数据。 |
| `ndb_select_count` | 使用 NDB API 获取一个或多个表中的行数。 |
| `ndb_setup.py` | 启动 Web 守护进程的 Python 脚本，从而允许您使用 Web 浏览器设置集群。另请参见第 5 章。 |
| `ndb_show_tables` | 列出集群中的表。有关输出示例和描述，请参见第 2 章。 |
| `ndb_size.pl` | 根据现有的非 NDB 集群数据库生成所需配置选项估算的 Perl 脚本。第 18 章中有一个其用法的示例。 |
| `ndb_waiter` | 等待集群达到给定状态。它通常在脚本中使用。第 10 章包含一个示例。 |

**提示**

有关实用程序的更多信息，请参见 [`dev.mysql.com/doc/refman/5.7/en/mysql-cluster-programs.html`](https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster-programs.html)，其中包含有关所有 NDB Cluster 程序的信息。

### 总结

本章讨论了 NDB 管理客户端的用途。该客户端可用于各种管理任务，从获取当前内存使用情况报告到重新启动节点和管理节点组。MySQL NDB Cluster 还包括其他几个客户端实用程序，这里也进行了简要讨论。本书的其余部分将使用管理客户端和其他实用程序，并提供其使用和输出的示例。下一章关于备份和恢复也不例外。

