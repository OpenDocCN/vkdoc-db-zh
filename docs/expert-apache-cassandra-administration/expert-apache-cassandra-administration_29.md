# ls -altr
total 52
-rw-r--r-- 4 root root   92 Jul  2 11:03 mc-3-big-Summary.db
-rw-r--r-- 4 root root   60 Jul  2 11:03 mc-3-big-Index.db
-rw-r--r-- 4 root root   16 Jul  2 11:03 mc-3-big-Filter.db
-rw-r--r-- 4 root root    9 Jul  2 11:03 mc-3-big-Digest.crc32
-rw-r--r-- 4 root root  144 Jul  2 11:03 mc-3-big-Data.db
-rw-r--r-- 4 root root   43 Jul  2 11:03 mc-3-big-CompressionInfo.db
-rw-r--r-- 4 root root   92 Jul  2 11:03 mc-3-big-TOC.txt
-rw-r--r-- 4 root root 4668 Jul  2 11:03 mc-3-big-Statistics.db
-rw-r--r-- 1 root root  856 Jul  2 11:03 schema.cql
-rw-r--r-- 1 root root   31 Jul  2 11:03 manifest.json
drwxr-xr-x 2 root root 4096 Jul  2 11:03 .
drwxr-xr-x 7 root root 4096 Jul  2 11:10 ..
$
```

此目录中的所有文件都有 `.db` 扩展名，表示它们都是名为 `CYCLIST_NAME` 的 SSTable 的一部分。`schema.cql` 文件包含创建此表的完整 CQL 代码。

你也可以通过指定键空间名称，使用 `nodetool snapshot` 命令备份单个键空间。

```
$ nodetool snapshot cycling
Requested creating snapshot(s) for [cycling] with snapshot name [1499020818525] and options {skipFlush=false}
Snapshot directory: 1499020818525
$
```

### 管理快照

你可以通过执行 `nodetool listsnapshots` 命令列出节点上的所有快照，包括它们的名称和大小详情。

```
$ nodetool listsnapshots
Snapshot Details:
Snapshot name Keyspace name      Column family name             True size Size on disk
1499021840269 system_distributed parent_repair_history          23.69 KiB 23.72 KiB
1499021840269 system_distributed repair_history                 5.69 KiB  5.72 KiB
1499021840269 system_distributed view_build_status              0 bytes   13 bytes
1499021840269 cycling            cyclist_name                   5 KiB     5.87 KiB
1499021840269 test5              kv                             4.78 KiB  5.62 KiB
1499021840269 test               my_table                       0 bytes   842 bytes
1499021840269 cycling2           cyclist_name                   0 bytes   870 bytes
1499021840269 system_auth        roles                          4.95 KiB  4.98 KiB
1499021840269 system_auth        role_members                   0 bytes   13 bytes
1499021840269 system_auth        resource_role_permissons_index 0 bytes   13 bytes
1499021840269 system_auth        role_permissions               0 bytes   13 bytes
1499021840269 system_traces      sessions                       0 bytes   13 bytes
1499021840269 system_traces      events                         0 bytes   13 bytes
Total TrueDiskSpaceUsed: 44.11 KiB
$
```

快照会占用空间。它们也有日期标记。这意味着你必须经常清理不再需要的旧快照文件。你可以通过删除整个快照目录来完成此操作。你可以编写备份脚本，以便在创建新快照之前删除旧快照。

`nodetool clearsnapshot` 命令允许你为一个或多个键空间删除已创建的快照。

你可以通过运行 `nodetool clearsnapshot` 命令并指定快照目录名称来删除单个快照。

```
$ ./nodetool clearsnapshot 1499020818525
Requested clearing snapshot(s) for [1499020818525]
$
```

你可以通过执行不带快照名称的 `nodetool clearsnapshot` 命令来删除节点上的所有快照目录。

```
$ ./nodetool clearsnapshot
Requested clearing snapshot(s) for [all keyspaces]
$
```

以下是你可以为 `nodetool clearsnapshot` 命令指定的一些重要选项：

*   `-t:` 使你能够指定一个快照名称。
*   `keyspace`: 删除你指定的键空间中的快照。你可以指定多个键空间，每个键空间用空格分隔。
*   `snapshot`: 你想要删除的快照的名称。

在生产环境中，要进行数据库备份，你需要创建一个快照，将文件打包（`tar`）并将压缩文件存储在网络备份位置。

#### 在压缩数据前创建自动快照

你可以启用数据库，在执行 SSTables 压缩之前自动创建快照（我在第 11 章详细解释了压缩功能）。通过在 `cassandra.yaml` 文件中将 `snapshot_before_compaction` 属性设置为 `true` 来完成此操作。默认值为 `false`。

请记住，由于数据库不会自动删除旧的快照，你必须意识到自动快照的空间后果。

### 执行增量备份

一旦你创建了系统范围的快照，就可以利用 Cassandra 的增量备份功能来备份自创建完整快照以来已更新的数据。

增量备份默认是禁用的，你可以使用以下命令启用它们：

```
$ nodetool enable backup
```

或者，你可以通过将 `cassandra.yaml` 文件中的 `incremental-backups` 属性的值更改为 `true` 来长期启用增量备份。你可以使用以下命令禁用增量备份：

```
$ nodetool disable backup
```

增量备份是自动的。Cassandra 会在所属键空间的 `data` 文件夹的 `backups` 子目录中，为每个刷新到磁盘的 SSTable 创建一个硬链接。但是，Cassandra 不会删除这些硬链接，因此管理员必须处理这些链接。

注意：运行 `nodetool statusbackup` 命令以获取备份状态。

### 使用各种恢复方法恢复数据

要完全恢复数据，你当然需要一个完整的备份，它由以下部分组成：

*   在某个时间点的快照
*   自创建该快照以来的所有增量备份
*   自创建最后一个增量备份以来的所有提交日志段


### 从快照恢复数据

要从快照恢复表，请确保拥有该表的所有快照文件，包括你在创建初始快照后进行的任何增量备份。有两种方法可以使用快照进行恢复，我将在此说明。

在两种恢复方法中，表模式都必须存在于数据库中；恢复过程不会自动重建模式。不过，如果需要，你可以运行快照目录中找到的 `createschema.sql` 脚本来创建模式。

#### 从快照目录复制数据

1.  **截断**你要恢复的表。大多数情况下，你需要移除现有数据，因为意外丢失的数据可能拥有比快照数据更旧的墓碑标记。在恰好丢失磁盘并在恢复前启动数据库的情况下，节点将拥有比快照更新的数据，因此你**不需要**截断表。在此示例中，键空间是 `cycling`，表名是 `cyclist_name`。在截断表之前，转到 `cycling` 键空间的数据目录，你会看到表 `cyclist_name` 的 `data` 文件夹包含所有正常的 `.db` 文件。

```
$CASSANDRA_HOME/data/data/cycling# cd c*
samalapati@ubuntu:/cassandra/apache-cassandra-3.10/data/data/cycling/cyclist_name-39cd6de060de11e7805be14006afbdda# ls
backups                      mc-1-big-Data.db       mc-1-big-Filter.db  mc-1-big-Statistics.db  mc-1-big-TOC.txt
mc-1-big-CompressionInfo.db  mc-1-big-Digest.crc32  mc-1-big-Index.db   mc-1-big-Summary.db     snapshots
$
```

**注意：** SSTable 的 `snapshots` 目录默认是空的，直到你创建了快照。

2.  接下来，你**截断**表 `cyclist_name`。

```
cqlsh:cycling> truncate cyclist_name;
cqlsh:cycling>
```

**注意：** 在执行 `truncate` 命令之前，请确保所有节点都已启动。否则，你可能会看到如下错误：`TruncateError: Error during truncate: Cannot achieve consistency level ONE`。

3.  你可以验证 `cyclist_name` 表中不再有任何数据。

```
cqlsh:cycling> select * from cyclist_name;
id | firstname | lastname
----+-----------+----------
(0 rows)
cqlsh:cycling>
```

4.  现在检查 `cycling.cyclist_name` 表的 `data` 目录；当你截断表时，所有 `.db` 文件都从该目录消失了。只剩下表的空壳，但数据库已永久从磁盘上清除了其所有内容。你在截断 `cyclist_name` 表之前拍摄了 `cycling` 键空间的快照。因此，你期望在相应的目录中找到 `cyclist_name` 表的快照，该目录通常是：

```
data_directory/keyspace_name/table_name-UUID/snapshost/snapshot_name
```

在我的情况下，是以下目录，并且在表 `cyclist_name` 的 `snapshot` 目录下有我需要的所有文件：

```
data/data/cycling/cyclist_name-39cd6de060de11e7805be14006afbdda/snapshots/1499189424022# ls -altr
total 52
-rw-r--r-- 2 root root   16 Jul  4 10:30 mc-1-big-Filter.db
-rw-r--r-- 2 root root   92 Jul  4 10:30 mc-1-big-TOC.txt
-rw-r--r-- 2 root root   92 Jul  4 10:30 mc-1-big-Summary.db
-rw-r--r-- 2 root root 4668 Jul  4 10:30 mc-1-big-Statistics.db
-rw-r--r-- 2 root root   60 Jul  4 10:30 mc-1-big-Index.db
-rw-r--r-- 2 root root   10 Jul  4 10:30 mc-1-big-Digest.crc32
-rw-r--r-- 2 root root   155 Jul  4 10:30 mc-1-big-Data.db
-rw-r--r-- 2 root root   43 Jul  4 10:30 mc-1-big-CompressionInfo.db
-rw-r--r-- 1 root root   856 Jul  4 10:30 schema.cql
-rw-r--r-- 1 root root   31 Jul  4 10:30 manifest.json
drwxr-xr-x 3 root root 4096 Jul  4 10:30 ..
drwxr-xr-x 2 root root 4096 Jul  4 10:30 .
/data/data/cycling/cyc
```

5.  将 `snapshot` 目录下的所有文件复制到表 `cyclist_name` 的 `data` 目录中。

```
root@ubuntu:/cassandra/apache-cassandra-3.10/data/data/cycling/cyclist_name-39cd6de060de11e7805be14006afbdda# cp /cassandra/apache-cassandra-3.10/data/data/cycling/cyclist_name-39cd6de060de11e7805be14006afbdda/snapshots/1499189424022/* .
root@ubuntu:/cassandra/apache-cassandra-3.10/data/data/cycling/cyclist_name-39cd6de060de11e7805be14006afbdda# ls
1499189424022  manifest.json                mc-1-big-Data.db       mc-1-big-Filter.db  mc-1-big-Statistics.db  mc-1-big-TOC.txt  snapshots
backups        mc-1-big-CompressionInfo.db  mc-1-big-Digest.crc32  mc-1-big-Index.db   mc-1-big-Summary.db     schema.cql
root@ubuntu:/cassandra/apache-cassandra-3.10/data/data/cycling/cyclist_name-39cd6de060de11e7805be14006afbdda#
```

运行 `nodetool refresh` 命令，以便 Cassandra 知道数据文件现已恢复。

```
$ nodetool refresh cycling cyclist_name
$
```

Cassandra 的日志文件将显示刷新了多少个 SSTable，从而提供了一种检查刷新过程的安全方法。

6.  查询表 `cyclist_name` 以验证恢复是否成功。

```
cqlsh:cycling> select * from cyclist_name;
id                                   | firstname | lastname
--------------------------------------+-----------+-----------
fb372533-eb95-4bb4-8685-6ef61e994caa |   Michael |   MATTHEWS
220844bf-4860-49d6-9a4b-6b5d3a79cbfb |     Paolo |  TIRALONGO
6ab09bec-e68e-48d9-a5f8-97e6fb4c9b47 |    Steven | KRUIKSWIJK
(3 rows)
cqlsh:cycling>
```

**注意：** 你可以设置 `auto_snapshot` 属性，以便在每次截断或删除表时自动备份（创建快照）。默认情况下，`auto_snapshot` 属性是启用的。

嘿，发生了什么？这个表最初有六行，但恢复后，只剩下三行！嗯，集群中有两个节点，每个节点大致拥有一半的数据。`nodetool snapshot` 命令确实在两个节点上都创建了单独的快照目录。任何单个节点上的快照都不会包含所有数据。例如，如果你的复制因子是 3，并且集群中有四个节点，那么每个节点将拥有大约每个副本的 0.75。

在此示例中，测试集群有两个节点。但是，你只在一个节点上执行了恢复。因此，你必须像之前一样，在第二个节点上恢复 `.db` 文件。

```
