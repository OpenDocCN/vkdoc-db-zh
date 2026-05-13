# ./nodetool refresh cycling cyclist_name
```

完成此操作后，再次查询 `cyclist_name` 表：

```
cqlsh:cycling> select * from cyclist_name;
id                                   | firstname | lastname
--------------------------------------+-----------+-----------------
e7ae5cf3-d358-4d99-b900-85902fda9bb0 |      Alex |           FRAME
fb372533-eb95-4bb4-8685-6ef61e994caa |   Michael |        MATTHEWS
5b6962dd-3f90-4c93-8f61-eabfa4a803e2 |  Marianne |             VOS
220844bf-4860-49d6-9a4b-6b5d3a79cbfb |     Paolo |       TIRALONGO
6ab09bec-e68e-48d9-a5f8-97e6fb4c9b47 |    Steven |      KRUIKSWIJK
e7cd5752-bc0d-4157-a80f-7523add8dbcd |      Anna | VAN DER BREGGEN
(6 rows)
cqlsh:cycling>
```

**提示：** 如果在执行恢复之前表不存在，请借助表的快照目录中的 `creatschema.cql` 文件创建该表。


# Cassandra 快照恢复与维护

Cassandra 的修复机制确保集群中的所有节点都拥有最新的数据。如果你没有在同一时间为不同的节点创建快照，你可能会读取到陈旧的数据（如果使用`ONE`一致性级别）。

这就是为什么在从快照恢复后，你应该运行`nodetool repair`命令。修复操作确保集群中的所有副本都拥有最新的数据。修复过程可能持续数小时，因为它是一个资源消耗大的操作，需要 RAM 和 CPU 资源来完成工作。作为其工作的一部分，修复任务会为数据生成默克尔树，并将其与其他节点上的默克尔树进行比较。它还会传输缺失和/或过时的数据。在修复运行期间，如果并非所有快照都包含相同的数据，你有时可能会读到陈旧的数据。

## 使用 sstableloader 恢复快照

假设你有一系列表需要从快照中恢复。你可以使用`sstableloader`工具来恢复快照。

### 节点重启方法步骤

通过节点重启从快照恢复数据的方法涉及关闭所有节点，然后在恢复快照数据后将它们重新启动。你可以一次性对集群中的所有节点使用此方法，或仅对单个节点使用。

以下是通过重启节点从快照恢复数据的步骤：

1.  关闭集群中的所有节点。
2.  运行`nodetool drain`命令，以确保没有数据丢失的风险。
3.  删除`commitlog`目录中的所有文件。

    ```bash
    samalapati@ubuntu:/cassandra/apache-cassandra-3.10/data/commitlog# ls
    CommitLog-6-1499187293466.log  CommitLog-6-1499187293467.log
    samalapati@ubuntu:/cassandra/apache-cassandra-3.10/data/commitlog# rm *
    ```

    删除每个表目录下的所有`.db`文件。你必须对正在恢复的所有键空间执行此操作。确保不要动`snapshots`和`backups`目录！完成后，此目录中不应有任何`.db`文件。

    ```bash
    $ ls
    backups  snapshots
    $
    ```

4.  将最新`snapshot`目录中的所有文件复制到此目录：

    ```bash
    $ cp /cassandra/apache-cassandra-3.10/data/data/cycling/cyclist_name-4d1743b060ef11e7805be14006afbdda/snapshots/1499196752095/* .
    $
    ```

5.  此时，Cassandra 已跨集群恢复了表数据。为确保数据一致，请运行`nodetool repair`命令。

    ```bash
    $ sudo nodetool repair
    [2017-07-04 13:19:20,492] Replication factor is 1\. No repair is needed for keyspace 'cycling'
    [2017-07-04 13:19:26,018] Repair completed successfully
    [2017-07-04 13:19:26,064] Repair command #1 finished in 5 seconds
    ```

### 提交日志归档与时间点恢复

Cassandra 允许你配置提交日志归档和时间点恢复。Cassandra 在以下条件下归档提交日志：

*   节点启动时
*   数据库将提交日志写入磁盘时
*   任何指定的时间点

你可以通过`commitlog_archiving.properties`配置文件来配置提交日志的归档。你可以在此文件中设置与提交日志段归档和恢复相关的多个属性，如下文所述。

### 手动归档提交日志

你可以使用`archive_command`命令归档一个提交日志段。`archive_command`命令的语法如下：

```
archive_command=/bin/ln %path /backup/%name
```

在此命令中，两个关键参数是：

*   `path`：你要归档的提交日志段的完整限定路径
*   `name`：提交日志的名称

### 恢复提交日志

你对提交日志段创建的归档使你能够将数据库恢复到某个时间点。你可以使用`restore_command`命令恢复已归档的提交日志。该命令的语法如下：

```
restore_command=cp -f %from %to
```

在此命令中，关键参数是：

*   `From`：已归档的提交日志段的路径
*   `To`：提交日志目录的名称

### 设置恢复目录位置

你可以配置`restore_directories`参数来设置你希望存储已归档提交日志的目录位置。当你运行`restore_command`命令时，数据库会在此处查找它需要恢复的已归档提交日志段。

### 设置恢复时间点

你可以通过`restore_point_in_time`参数指定一个时间戳，来告诉数据库应恢复提交日志段到何时。示例如下：

```
restore_point_in_time=2013:12:11 17:00:00
```

Cassandra 将恢复在你指定的时间戳之前及至该时间点创建的提交日志段。

## 向 Cassandra 加载批量数据

有两个基本工具可用于在 Cassandra 集群之间移动批量数据：CQL `COPY`命令和`sstableloader`工具。

*   `COPY`命令可以将 CSV 数据读取到 Cassandra 表中，并将 Cassandra 中的 CSV 数据写入文件系统。`COPY`命令的工作方式与关系数据库用于导出和导入数据的工具非常相似。
*   `sstableloader`工具可帮助你将外部数据批量加载到 Cassandra 集群中。

### 使用 COPY 命令导入导出数据

`COPY`命令使你能够向 Cassandra 数据库复制数据或从中复制数据。源文件和目标文件是 CSV（逗号分隔值）或分隔文本文件。在以下部分中，我将介绍如何使用`COPY`命令来导入和导出数据。

### 从 Cassandra 表复制数据

使用`COPY TO`命令将数据从 Cassandra 表复制到 CSV 文件。`COPY TO`命令的语法如下：

```sql
COPY table_name [( column_list )]
TO 'file_name'[, 'file2_name', ...] | STDOUT
[WITH option = 'value' [AND ...]]
```

`COPY TO`命令使用你可以指定的分隔符来分隔 CSV 文件中的字段。默认情况下，此命令导出所有字段，你可以指定列列表以仅导出表的部分行。

以下示例展示了如何从表`CYCLNG.CYCLIST_NAME`导出数据：

```bash
cqlsh> COPY cycling.cyclist_name TO 'cyclist.csv' WITH HEADER = TRUE;
Using 1 child processes
Starting copy of cycling.cyclist_name with columns [id, firstname, lastname].
Processed: 6 rows; Rate:       8 rows/s; Avg. rate:       7 rows/s
6 rows exported to 1 files in 0.849 seconds.
cqlsh>
```

Cassandra 将 CSV 文件放在你执行`COPY TO`命令所在目录的上一级目录中。如果你打开 CSV 文件，可以看到其中包含源表中所有行的逗号分隔列表。

```bash
cat cyclist_lastname.csv
id,lastname
fb372533-eb95-4bb4-8685-6ef61e994caa,MATTHEWS
220844bf-4860-49d6-9a4b-6b5d3a79cbfb,TIRALONGO
e7cd5752-bc0d-4157-a80f-7523add8dbcd,VAN DER BREGGEN
6ab09bec-e68e-48d9-a5f8-97e6fb4c9b47,KRUIKSWIJK
e7ae5cf3-d358-4d99-b900-85902fda9bb0,FRAME
5b6962dd-3f90-4c93-8f61-eabfa4a803e2,VOS
```



### 将数据复制到 Cassandra 表中

`COPY` 命令有助于将小型数据集（包含少于 200 万行的数据集）移入 Cassandra 表。要移动更大型的数据集，必须使用 Cassandra 批量加载器。

指定 `COPY FROM` 以将数据从 CSV 文件复制到 Cassandra 表中。以下是 `COPY FROM` 命令的通用语法：

```
COPY table_name [( column_list )]
FROM 'file_name'[, 'file2_name', ...] | STDIN
[WITH option = 'value' [AND ...]]
```

`COPY FROM` 命令会更新现有记录并验证主键。运行 `COPY FROM` 命令时，需要记住以下关键点：

*   CSV 文件中的每一行必须包含相同的字段。
*   CSV 文件的字段（列）数可以少于 Cassandra 表。
*   CSV 文件的字段数不能多于 Cassandra 表。
*   每行都必须有主键值。
*   数据库会将所有缺失或空字段设置为 null。

有关 `COPY FROM` 命令的示例，请使用与 `COPY TO` 命令示例相同的表 `CYCLIST_NAME`。由于 `COPY FROM` 命令要求目标表已存在，请先清空表中的数据，以便从之前生成的 CSV 文件导入数据。

```
cqlsh> COPY cycling.cyclist_name FROM 'cyclist.csv' WITH HEADER = TRUE;
Using 1 child processes
Starting copy of cycling.cyclist_name with columns [id, firstname, lastname].
Processed: 6 rows; Rate:      11 rows/s; Avg. rate:      15 rows/s
6 rows imported from 1 files in 0.389 seconds (0 skipped).
cqlsh>
```

### 运行 sstableloader 执行批量加载

Cassandra 的批量加载工具名为 `sstableloader`，可让你将大量数据加载到 Cassandra 表中。你还可以使用此工具将数据从 SSTable 加载到不同的集群。最后，你可以使用此工具恢复快照，我将在本章前面解释这一点。

与 `COPY` 命令的情况不同，目标表可以包含数据。

#### 使用 sstableloader 加载外部数据

`sstableloader` 实用程序使你能够将外部数据加载到集群中。在使用 `sstableloader` 实用程序加载外部数据之前，必须首先生成用于存放外部数据的 SSTable。

要生成 SSTable，必须使用 `SSTableWriter` API。`SSTableWriter` 会创建原始的 Cassandra 数据文件，这些文件可以批量加载到集群中。

注意

你可以运行多个 `sstableloader` 实用程序实例，以并行方式将外部数据加载到集群中。

由于 `sstableloader` 实用程序资源消耗较大，建议在未用作 Cassandra 节点的服务器上运行它。

以下是使用 `sstableloader` 实用程序加载数据必须遵循的步骤摘要。

1.  创建将用于将数据批量加载到集群中的原始 Cassandra 数据文件。这涉及编写一些使用 `SSTableWriter` API 的 Java 代码。
2.  `SSTableWriter` 会在你指定的目录中生成 SSTable。转到代码存储 SSTable 的位置，如下所示：

    ```
    $ cd /var/lib/cassanddra/data/Keyspace1/Standrad2
    ```

3.  你可以通过执行 `ls` 命令查看 Keyspace 的内容。

    ```
    $ ls
    Keyspace1-Standard1-jb-60-CRC.db
    Keyspace1-Standard1-jb-60-Data.db
    ...
    Keyspace1-Standard1-jb-60-TOC.txt
    ```

4.  通过指定目标集群中 `Keyspace1/Standard1/` 的路径来批量加载文件。

    ```
    $ sstableloader -d 192.168.159.129 /var/lib/Cassandra/data/Keyspace1/Standard1/
    ```

#### 将 SSTable 数据加载到不同的集群

你可以运行 `sstableloader` 实用程序，从不同的集群导入现有的 SSTable。在执行导入之前，请在源集群的每个节点上运行 `nodetool flush` 命令，以确保数据库将内存表刷新到即将导入到目标集群的 SSTable 中。

要运行 `sstableloader` 命令将数据导入目标集群，请遵循上一节描述的步骤。所有步骤的工作方式相同，只是你无需生成 SSTable，因为源表已包含它们。你是从一个集群的 SSTable 导入到另一个集群的 SSTable，因此此导入不涉及原始数据。

注意

如果你使用的是 DataStax Enterprise，可以使用 Apache Sqoop 将外部数据迁移到 Cassandra，因为它已包含运行 Sqoop 所需的一切。

## 总结

由于执行恢复时需要快照，因此仔细管理快照非常重要。Cassandra 不会为你管理快照，因此你必须制定有效的方法来存储快照并移除旧的快照，以免它们占用大量存储空间。

增量备份默认是开启的，你必须确保其保持有效状态，因为它们可以减少对 SSTable 进行完整快照的需求。

在恢复期间，你将需要所有快照、增量备份和提交日志段的备份。因此，重要的是要知道你可以随时访问所有这些，以便最小化恢复时间。为了确保能够正确执行恢复，对恢复过程进行演练非常重要。你肯定不想在真正执行恢复时才学习数据库恢复！

## 第三部分

维护、监控、调优和保护 Apache Cassandra

## 9. 维护 Cassandra

本章介绍常见的 Cassandra 管理任务和节点管理。

常见的集群管理任务包括修复和清理数据、重建索引、处理数据损坏以及刷新和排出数据。

节点管理包括添加和移除节点、替换和移动节点、移除死亡节点以及淘汰节点。由于 `nodetool` 命令是执行所有节点管理任务的关键，因此我首先回顾与节点管理相关的关键 `nodetool` 命令。

本章展示了如何执行管理任务，例如添加和停用数据中心。

本章还展示了如何切换 Snitch 以及如何监控和管理 Gossip。

### 常见的集群维护任务

作为管理员，你将执行各种任务以帮助 Cassandra 良好运行。在以下部分中，我将介绍一些最常见的数据库维护任务，例如修复数据、重建索引和清理数据。

#### 修复数据 (nodetool repair)

我在第 5 章详细解释了 Cassandra 的修复机制。随着时间的推移，由于 Cassandra 的可调一致性理念，节点可能与其他节点不同步。如果一个节点长时间宕机，它可能会错过在其他节点上执行的更改，特别是当它宕机的时间超过其他节点存储提示信息的时间时。

此外，指定低于 `ALL` 级别的写一致性级别意味着，即使一个或多个节点未确认写入，数据库也会将该写入声明为成功。可调一致性的原则可能导致同一数据在不同节点上存在不同版本。虽然数据库会自动执行某些类型的修复（如读修复），但管理员执行的关键操作之一是反熵修复，即使用 `nodetool repair` 命令手动修复数据。

提示

在执行修复时运行 `nodetool stats` 命令以监控修复进度。

##### 何时执行修复

修复的频率与配置的读写一致性级别有很大关系。如果你选择的是一致性级别提供较慢的级别，那么你应该更频繁地安排修复。

频繁的修复会产生开销成本，因此尽可能使用子范围修复，并将不同 Keyspace（或表）的修复安排在不同时间。



# Cassandra 数据库维护：nodetool 命令详解

## 当修复不可避免时

尽管使用`nodetool repair`命令执行的修复是可选的，但有几种操作在完成后需要您执行一次完整的修复。这些操作包括更改集群的`snitch`和更改键空间的复制级别。

## 运行 nodetool repair 命令

您可以为整个数据库或特定的键空间或表运行`nodetool repair`命令，如下列示例所示：

```
$ nodetool repair
$ nodetool repair cycling
$ nodetool repair cycling cycle
```

您也可以使用`-local`（或`–in-local-dc`）选项仅为本地数据中心运行`nodetool repair`命令。您还可以使用`-dc`（或`–in-dc`）选项指定一个数据中心。

当您在运行`nodetool repair`命令时指定`-full`选项时，Cassandra 会执行完整修复。由于完整修复可能耗时且占用资源，您可以要求它执行增量修复，即数据库仅修复之前未修复的数据。在 Cassandra 3.x 中，默认是增量修复。以下是日志文件的摘录，展示了`repair`命令的工作方式：

```
INFO  [Thread-8] 2017-07-16 09:22:23,564 RepairRunnable.java:136 - Starting repair command #1 (f30498c0-6a42-11e7-9731-6f6038c9cb1c), repairing keyspace system_traces with repair options (parallelism: parallel, primary range: false, incremental: true, job threads: 1, ColumnFamilies: [], dataCenters: [], hosts: [], # of ranges: 512, pull repair: false)
...
INFO  [Repair#1:1] 2017-07-16 09:22:27,833 RepairJob.java:172 - [repair #f35db540-6a42-11e7-9731-6f6038c9cb1c] Requesting merkle trees for sessions (to [/192.168.159.129, ubuntu/192.168.159.130])
...
INFO  [CompactionExecutor:9] 2017-07-16 09:22:28,547 CompactionManager.java:694 - [repair #f30498c0-6a42-11e7-9731-6f6038c9cb1c] Completed anticompaction successfully
INFO  [CompactionExecutor:10] 2017-07-16 09:22:28,759 RepairRunnable.java:340 - Repair command #1 finished in 5 seconds
$
```

## 通过从其他节点获取数据来重建数据（nodetool rebuild）

运行`nodetool rebuild`命令以重建令牌范围。该命令从单个源副本重建流，但同时操作多个节点。

当您向集群添加数据中心时，`nodetool rebuild`命令非常有用。您可以一次重建单个键空间，也可以指定多个键空间，每个键空间用逗号分隔。示例如下：

```
$ nodetool rebuild –keyspace cyclists, motorists
```

请记住：

*   默认情况下，此命令会选择集群中的任意数据中心。您可以通过`source-dc-name`选项指定数据库应从中选择其需要流式传输的数据源的数据中心。如果您不提供任何数据中心的名称，命令可能看似运行，但实际上不会执行任何操作。
*   您可以指定参数`tokens`以提供单个令牌的列表或令牌范围（`start_token, end_token`）。

## 清理不必要的键空间和分区键（nodetool cleanup）

当您向集群添加新节点时，一些节点可能会将其部分分区范围丢失给新节点。但是，Cassandra 此时不会从当前节点中删除不必要的键空间和分区键。

同样，如果您减少数据中心的复制因子，一些节点最终将存储数据库不再用作副本的数据。

在这两种情况下，当您添加节点或减少复制因子时，Cassandra 的合并进程最终将清除被丢弃和不必要的数据。您无需在每次此类集群更改后手动清理。但是，如果您希望更快地回收磁盘空间，可以运行`nodetool cleanup`命令。

> 注意
>
> 如果节点在某些表中使用了计数器列，一旦您运行`nodetool cleanup`命令，数据库会为该节点分配新的计数器 ID。

以下是展示如何运行`nodetool cleanup`命令的示例：

```
$ nodetool cleanup
```

由于您未指定键空间，此命令将清理所有不再属于此节点的键空间。您也可以选择指定一个键空间以及要清理的表列表。运行如下`nodetool cleanup`命令以清理特定键空间：

```
$ nodetool cleanup cycling
```

### 重建索引

您可以通过为表运行`nodetool rebuild_index`命令来重建二级索引。在该命令中，您必须指定键空间、表的名称，以及一个或多个用空格分隔的索引名称。

`nodetool rebuild_index`命令为表的索引执行完全重建。此命令需要三个参数：键空间、`cf`（列族，是表的同义词）和索引名称。示例如下：

```
$ nodetool rebuild_index cycling cycle test_idx;
```

## 刷新表的大小估算

当您向表中插入大量数据或截断表的数据时，表的大小估算会变得陈旧。运行`nodetool refreshsizeestimates`命令以刷新表大小估算。该命令会刷新`system.size_estimates`表。

## 关键 Nodetool 维护命令

在本章中，除其他内容外，我将向您展示如何在集群中添加、移动和删除节点。在执行与在集群中添加、移动和删除节点及数据中心相关的大部分任务时，您将运行一组关键的`nodetool`命令。例如，在停用节点后，您需要运行`nodetool`命令。我将在本节中解释与节点维护相关的常见`nodetool`命令。



### 节点退役 (`nodetool decommission`)

你可以运行 `nodetool decommission` 命令，让一个节点将其所有数据发送给环中的下一个节点。退役一个节点是引导节点的逆操作。以下是一个展示如何退役节点的示例。

1.  首先，检查节点状态，确保所有（本例中为两个）节点都处于启动状态。

```
$ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address          Load       Tokens       Owns (effective)  Host ID                                 Rack
UN  192.168.159.129  338.19 KiB  256          49.0%             0dbb9e0e-867e-4179-b6b6-631d38dd68f9  rack1
UN  192.168.159.130  339.49 KiB  256          51.0%             a1085901-738c-4bbd-b050-007f62da893d  rack1
$
```

2.  然后在 IP 为 192.168.159.129 的节点上运行 `nodetool decommission` 命令。

```
$ nodetool decommission
```

3.  检查集群状态，注意被退役的节点现在显示状态为 UL (UP LEAVING)，表明它正在从集群中过渡退出。

```
UN  192.168.159.129  338.19 KiB  256          49.0%             0dbb9e0e-867e-4179-b6b6-631d38dd68f9  rack1
UL  192.168.159.130  339.49 KiB  256          51.0%             a1085901-738c-4bbd-b050-007f62da893d  rack1
$
```

在运行 `nodetool decommission` 命令后，`system.log 文件` 显示以下输出：

```
CP Connection(4)-127.0.0.1] 2017-07-13 10:48:38,266 StorageService.java:1435 - LEAVING: sleeping 30000 ms for batch processing and pending range setup
INFO  [RMI TCP Connection(4)-127.0.0.1] 2017-07-13 10:49:08,776 StorageService.java:1435 - LEAVING: replaying batch log and streaming data to other nodes
INFO  [RMI TCP Connection(4)-127.0.0.1] 2017-07-13 10:49:08,902 StreamResultFuture.java:90 - [Stream #9254bbf0-67f3-11e7-9d74-6bc8e6fb7ba6] Executing streaming plan for Unbootstrap
INFO  [RMI TCP Connection(4)-127.0.0.1] 2017-07-13 10:49:08,904 StorageService.java:1435 - LEAVING: streaming hints to other nodes
INFO  [HintsDispatcher:2] 2017-07-13 10:49:08,963 HintsDispatchExecutor.java:152 - Transferring all hints to 0dbb9e0e-867e-4179-b6b6-631d38dd68f9
...
INFO  [RMI TCP Connection(4)-127.0.0.1] 2017-07-13 10:49:09,413 StorageService.java:3938 - Announcing that I have left the ring for 30000ms
INFO  [RMI TCP Connection(4)-127.0.0.1] 2017-07-13 10:49:39,428 Server.java:176 - Stop listening for CQL clients
WARN  [RMI TCP Connection(4)-127.0.0.1] 2017-07-13 10:49:39,429 Gossiper.java:1514 - No local state, state is in silent shutdown, or node hasn't joined, not announcing shutdown
...
$
```

4.  检查集群状态。果然，被退役的节点已经离开了集群。

```
$ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address          Load       Tokens       Owns (effective)  Host ID                                 Rack
UN  192.168.159.129  328.53 KiB  256          100.0%            0dbb9e0e-867e-4179-b6b6-631d38dd68f9  rack1
$
```

5.  接下来，如果你尝试通过重新启动该节点将其添加回集群，此尝试会失败，如下方 `cassandra` 命令的输出所示：

```
$ cassandra ...
, 9079045138376597795, 9105248878993821997, 9166471019946737685, 9167277031713212107, 9167870256226113023]
...
This node was decommissioned and will not rejoin the ring unless cassandra.override_decommission=true has been set, or all existing data is removed and the node is bootstrapped again
Fatal configuration error; unable to start server.  See log for stacktrace.
ERROR [main] 2017-07-13 10:58:29,816 CassandraDaemon.java:752 - Fatal configuration error
...
$
```

当你重启节点时，你尝试重新入群的节点无法加入环。如你所见，Cassandra 很贴心地提供了两种方式让该节点重新加入环：
*   移除被退役节点上的所有数据并重新引导它。
*   设置 `cassandra.override_decommission=true` 选项。

在接下来的章节中，我将展示如何使用这两种方法将退役的节点重新加入集群。

#### 移除所有数据并重启节点

将退役节点重新添加回集群的直接方法是简单地移除该节点上的所有旧数据，这些数据存储在 `/data` 下的以下目录中：
*   `commitlog`
*   `data`
*   `saved_caches`

当你退役节点时，Cassandra 会重新分配令牌。一旦你从被退役节点上移除了所有数据，Cassandra 就允许该节点重新加入，并再次为其分配其份额的令牌。

移除所有数据后，重启被退役的节点。

```
$ cassandra
...
INFO  [main] 2017-07-13 11:13:47,122 ColumnFamilyStore.java:406 - Initializing system.hints
INFO  [main] 2017-07-13 11:13:47,156 ColumnFamilyStore.java:406 - Initializing system.batchlog
INFO  [GossipStage:1] 2017-07-13 11:13:50,736 Gossiper.java:1056 - Node /192.168.159.129 is now part of the cluster
INFO  [main] 2017-07-13 11:13:51,939 StorageService.java:1435 - JOINING: waiting for ring information
INFO  [InternalResponseStage:1] 2017-07-13 11:13:52,829 ColumnFamilyStore.java:406 - Initializing cycling.cycle
INFO  [main] 2017-07-13 11:14:24,036 StorageService.java:1435 - JOINING: Starting to bootstrap...
INFO  [main] 2017-07-13 11:14:25,128 StorageService.java:1435 - JOINING: Finish joining ring
...
INFO  [main] 2017-07-13 11:14:25,193 SecondaryIndexManager.java:508 - Executing pre-join post-bootstrap tasks for: CFS(Keyspace='cycling', ColumnFamily="cycle")
INFO  [main] 2017-07-13 11:14:33,359 CassandraDaemon.java:725 - No gossip backlog; proceeding
...
```

被退役的节点现在重新加入了环。Cassandra 已经重新分配了集群中的数据。你可以通过运行 `nodetool status` 命令来确认节点已回到集群中。

```
$ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address          Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.159.129  350.33 KiB  256          48.5%             0dbb9e0e-867e-4179-b6b6-631d38dd68f9  rack1
UN  192.168.159.130  158.45 KiB  256          51.5%             9ee09b72-a6db-4de4-9aed-de32c5e7c344  rack1
$
```

#### 设置 `cassandra.override_decommission=true` 选项

或者，你可以在你已退役节点的 `cassandra-env.sh` 文件中设置以下选项，然后重启该节点。节点重启后，它将加入集群。

```
JVM_OPTS="$JVM_OPTS -D cassandra.override_decommission=true"
```



### 强制移除节点 (`nodetool assassinate`)

假设你的集群中有三个节点，现在需要移除其中一个节点。你已经对该节点执行了下线操作，但当你运行 `nodetool describecluster` 命令时，发现该节点处于 `UNREACHABLE`（不可达）状态，如下所示：

```
$ nodetool describecluster
Cluster Information:
Name: Test Cluster
Snitch: org.apache.cassandra.locator.DynamicEndpointSnitch
Partitioner: org.apache.cassandra.dht.Murmur3Partitioner
Schema versions:
57af9326-0783-3ea2-93ec-7706e8cad5e7: [192.168.159.129]
UNREACHABLE: [192.168.159.130]
$
```

`nodetool status` 命令显示该不可达节点的状态为 `DN`，表示节点已宕机。

```
$ nodetool status
Datacenter: DC1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address          Load       Tokens       Owns (effective)  Host ID                               Rack
DN  192.168.159.130  ?          256          49.7%             7d9e3ac3-ffdc-4cee-be82-4b0c6e614aa1  r1
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address          Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.159.129  383.92 KiB  256          50.3%             0dbb9e0e-867e-4179-b6b6-631d38dd68f9  rack1
$
```

遇到这种无法通过 `nodetool decommission` 命令将节点移出集群的情况时，可以运行 `nodetool assassinate` 命令来强制移除该节点。

```
$ nodetool assassinate
```

运行 `nodetool status` 命令检查集群状态，确认被强制移除的节点 (`192.168.159.130`) 已从集群中消失。

```
$ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address          Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.159.129  359.04 KiB  256          100.0%            0dbb9e0e-867e-4179-b6b6-631d38dd68f9  rack1
$
```

每当使用 `nodetool removenode` 命令移除节点失败时，就可以运行 `nodetool assassinate` 命令。该命令会移除一个已失效的节点，且不会复制其数据。

`nodetool assassinate` 命令移除的是你运行该命令所在的节点。你可以使用 `-h` 选项指定一个远程节点。

```
$ nodetool assassinate -h 192.168.159.129
```

### 节点管理

节点管理是 Cassandra 管理员任务清单中不可或缺的一部分。管理节点涉及添加、移动和替换节点，也包括管理缓慢或失效的节点，以及在范围移动后清理数据。

在以下章节中，我将解释如何执行关键的节点管理任务。

### 阻止节点加入集群

有时，你可能不希望某个节点加入集群。也就是说，你希望该节点保持启动和运行状态，但不作为环的一部分。你可以通过利用 Cassandra 命令行工具的 `-D` 选项来实现这一点。`-D` 选项允许你在 `cassandra-env.sh` 文件和命令行中指定启动参数。在此情况下，请在 `cassandra-env.sh` 文件中添加 `-Dcassandra.join.ring=false` 选项。

以下是阻止节点加入集群必须遵循的步骤。

1.  停止你希望从环中移除的节点。
2.  编辑 `cassandra-env.sh` 文件并添加以下行：

    ```
    JVM_OPTS="$JVM_OPTS -Dcassandra.join_ring=false"
    ```

3.  启动该节点。

    ```
    $ cassandra -R
    ...
    INFO  [main] 2017-07-15 12:31:36,878 CassandraDaemon.java:489 - JVM Arguments: [-...
    Djava.library.path=./../lib/sigar-bin, -Dcassandra.join_ring=false, -
    ...
    INFO  [GossipStage:1] 2017-07-15 12:31:44,587 Gossiper.java:1056 - Node /192.168.159.129 is now part of the cluster
    INFO  [GossipStage:1] 2017-07-15 12:31:44,589 Gossiper.java:1056 - Node /192.168.159.130 is now part of the cluster
    INFO  [main] 2017-07-15 12:31:45,757 StorageService.java:695 - Not joining ring as requested. Use JMX (StorageService->joinRing()) to initiate ring joining
    INFO  [main] 2017-07-15 12:31:45,758 CassandraDaemon.java:694 - Waiting for gossip to settle before accepting client requests...
    INFO  [GossipStage:1] 2017-07-15 12:31:46,429 StorageService.java:2248 - Node /192.168.159.129 state jump to NORMAL
    INFO  [main] 2017-07-15 12:31:53,763 CassandraDaemon.java:725 - No gossip backlog; proceeding
    $
    ```

从 Cassandra 工具的输出可以看出，节点 (`192.168.159.130`) 已正常启动，并且是集群的一部分。然而，由于你要求 Cassandra 不要将此节点加入集群，它保持启动状态但不在集群内。你可以通过运行 `nodetool status` 命令来确认这一点。

```
$ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address          Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.159.129  296.03 KiB  256          50.3%             0dbb9e0e-867e-4179-b6b6-631d38dd68f9  rack1
DN  192.168.159.130  316.55 KiB  256          49.7%             7d9e3ac3-ffdc-4cee-be82-4b0c6e614aa1  rack1
$
```

节点 `192.168.159.130` 显示状态为 `DN` (`DOWN NORMAL`)。该节点已启动，但不再是此集群的一部分。

随后，你可以运行 `nodetool join` 命令将此节点重新添加回环中，如下所示：

```
$ nodetool join
```

### 向数据中心添加节点

添加一个新节点涉及安装并启动新节点，以及清理现有节点以移除不再属于这些节点的密钥。请按照以下步骤向一个正在运行的集群中添加节点。

1.  在新节点上安装 Cassandra 二进制文件。
2.  在 `cassandra.yaml` 文件中配置以下属性：
    *   `cluster_name`: 集群名称。
    *   `endpoint_snitch`: `GossipingPropertyFileSnitch`（或任何你希望使用的 snitch）。
    *   `num_tokens`: 256（此设置 vnodes 的数量，你可以将其设置为与其他节点配置的令牌相同的数字）。
    *   `allocate_tokens_for_local_replication_factor`: 允许你指定键空间的复制因子。
    *   `seed_provider`: 指定集群中现有的一个种子节点。
3.  确保你在 `cassandra.yaml`、`cassandra-rackdc.properties` 和 `cassandra-topology.properties` 文件中配置的任何非默认设置都已添加。新节点上所有这些配置文件应使用与其他节点相同的非默认设置。
4.  启动新节点。
5.  使用 `nodetool status` 命令验证节点状态，并确保所有节点都显示 `UN` (`Up Normal`) 状态。
6.  在所有现有节点上运行 `nodetool cleanup` 命令。你必须等待一个节点上的命令完成后再继续处理其他节点。

此场景适用于向集群添加新节点。如果你要替换一个因故死亡的现有节点，你需要让 Cassandra 知道你添加的新节点将替换该失效节点。通过指定 `replace_address` 选项来实现这一点。

在执行以下步骤之前，请停止你希望替换的节点。

*   对于包安装，在启动新节点之前（参见步骤 4），将 `replace_address` 选项添加到 `cassandra-env.sh` 文件中。

    ```
    JVM_OPTS = "$JVM_OPTS -Dcassandra.replace_address=192.168.159.130"
    ```

*   对于 tarball 安装（如我的情况），使用 `cassandra.replace_address` 选项启动数据库。

你可以将失效节点的 `listen_address` 或 `broadcast_address` 作为 `cassandra.replace_address` 选项的值。你必须确保用来替换另一个节点的节点是全新的，或者如果它是一个现有节点，你已经移除了 `/data` 目录下的所有数据。


# 替换运行中的节点

偶尔，你可能需要出于维护目的而替换一个状态良好的节点。有两种方法可以替换节点：

*   添加新节点，然后停用你想替换的节点。
*   直接替换节点。

我将在后续章节中解释这两种方法。

## 先添加新节点，然后停用旧节点

通过先添加新节点来替换节点的方法很直接。以下是执行此操作的高级步骤。

1.  按照“添加新节点”部分的说明添加一个节点。
2.  添加新节点后，按照“停用节点”部分的说明，通过运行 `nodetool decommission` 命令来停用旧节点。
3.  在所有属于同一数据中心的节点上运行 `nodetool cleanup` 命令。

## 直接替换节点

上一节解释的方法涉及两次数据流传输或运行 `nodetool cleanup` 命令。你可以简单地通过 `-Dcassandra.replace_address` 选项启动新节点来完成相同的任务。

要使用此技术，你必须确保没有对任何键空间使用 `ONE` 一致性级别。这意味着该节点可能拥有某一行的唯一副本，因此你有丢失该数据的风险。

以下是直接替换节点的步骤。

*   首先，在你打算替换的节点上停止 Cassandra。
*   按照“替换故障节点”部分的步骤，用另一个节点替换该节点。

如果你要替换的节点恰好是一个种子节点，你必须从 `cassandra.yaml` 文件的 `-seeds` 列表中删除故障节点的 IP 地址。当然，你必须为集群中的所有节点执行此操作。如果你需要新节点（或另一个现有节点）来替代故障的种子节点，请将该节点的 IP 地址添加到 `–seeds` 列表中。

### 从集群中移除节点

如果你要移除的节点状态良好且正在运行，只需运行 `nodetool decommission` 命令即可将其从集群中移除。但是，如果节点已宕机，你当然无法停用它，这时你需要运行 `nodetool removenode` 命令来移除该节点。

你可以通过指定要移除节点的主机 ID 来使用 `nodetool removenode` 命令移除节点。

```
$ nodetool removenode 192.168.159.130
```

你可以通过运行 `nodetool removenode status` 命令来确认节点已被移除。

```
$ nodetool removenode status
```

### 替换故障节点

节点可能因多种原因而故障，例如硬件故障。你可以通过运行 `nodetool status` 命令检查节点状态来判断节点是否故障。如果你看到状态为 `DN`，并且在日志中没有看到任何与配置相关的错误，那么是硬件故障或其他问题导致节点崩溃或无法启动。在这些情况下，你必须替换故障节点。

请按照以下步骤替换故障节点。

1.  向你的网络中添加一个替换节点。
2.  如果故障节点恰好是一个种子节点，请在 `seed_provider` 属性下的 `-seeds` 列表中，用新节点的 IP 地址替换故障节点的 IP 地址。
3.  重新启动新节点。
4.  在新节点上安装 Cassandra。
5.  使用 `cluster_name` 和 `-seeds` 列表属性配置 `cassandra.yaml` 文件。
6.  在新节点上，添加机架和数据中心配置。假设你使用的是 `GossipPropertyFileSnitch` 或 `Ec2MultiRegionSnitch`，请将机架和数据中心信息添加到 `cassandra-rackdc.properties` 文件中。此外，删除 `cassandra-topology.properties` 文件。
7.  使用 `replace_address` 选项启动替换节点，如“向数据中心添加新节点”部分所述，可以通过将 `replace_address` 选项添加到 `cassandra-env.sh` 文件，或者通过使用 `replace_address` 选项启动 Cassandra 来实现。

### 将节点移动到不同的机架

如果你发现将节点放错了机架，可以通过将节点移动到正确的机架和数据中心来修复错误。在这种情况下，Cassandra 需要先将数据移出该节点，在更改节点所属的机架后，再将新数据加载到该节点中。

DataStax 推荐的移动节点方法是停用你想移动到不同机架的节点。一旦你按“停用节点”部分所述将其停用，就可以将该节点添加到你希望的目标机架和数据中心。

# 停用整个数据中心

当你停用一个数据中心时，你首先要通过阻止客户端向该数据中心节点写入数据来确保不会丢失任何数据。接下来，执行以下步骤以停用该数据中心。

1.  为确保所有数据都已传播到其他数据中心，请运行 `nodetool repair` 命令。

    ```
    $ nodetool repair -full
    [2017-07-15 11:50:07,969] Replication factor is 1\. No repair is needed for keyspace 'cycling'
    ...
    [2017-07-15 11:50:08,104] Starting repair command #1 (6c43e1b0-698e-11e7-849e-0920617bbdc2), repairing keyspace system_traces with repair options (parallelism: parallel, primary range: false, incremental: false, job threads: 1, ColumnFamilies: [], dataCenters: [], hosts: [], # of ranges: 512, pull repair: false)
    [2017-07-15 11:50:12,699] Repair completed successfully
    [2017-07-15 11:50:12,710] Repair command #1 finished in 4 seconds
    $
    ```

2.  下一步是删除所有对要移除数据中心的引用。你可以通过使用 `ALTER KEYSPACE` 命令修改集群中的所有键空间来实现这一点，使它们不再指向被移除的数据中心。`ALTER KEYSPACE` 命令允许你修改键空间的复制策略，以及数据库必须在每个数据中心创建的副本数量。如第 4 章所述，`SimpleStrategy` 为整个集群分配相同的复制因子，因此仅适用于开发和测试环境。生产环境必须使用 `NetworkTopologyStrategy` 选项。此策略允许你为每个数据中心指定复制因子。例如，如果你移除了 `datacenter1`，你必须确保为集群中的所有键空间配置一个不同的数据中心，如下所示：

    ```
    ALTER KEYSPACE cycling
    WITH REPLICATION = {
    'class'  :  'NetworkTopologyStrategy',
    'datacenter2'  :  3 }
    AND DURABLE_WRITES = true ;
    ```

3.  最后，在你希望从集群中移除的数据中心的每个节点上运行 `nodetool decommission` 命令。

# 切换嗅探器

关于切换嗅探器的一个关键点是，在更改嗅探器之后，你的集群拓扑是否会发生变化。复制策略基于嗅探器提供的信息放置副本，因为嗅探器的作用是告诉 Cassandra 如何分布副本。当你配置的新嗅探器将副本放置在不同位置时，就会发生拓扑变化（即，Cassandra 放置节点的机架或数据中心，或两者都发生变化）。

有时网络拓扑会发生变化，有时则不会。我将在以下章节中讨论这两种情况。


### 无拓扑变更

如果集群中尚无数据，则无需更改网络拓扑。你只需设置新的`snitch`即可。

例如，你可以从使用`SimpleSnitch`的单数据中心五个节点，转变为在同一单数据中心使用网络`snitch`（例如`GossipingPropertyFileSnitch`）的相同数量节点。

以下是无拓扑变更时切换`snitch`的步骤。

1.  根据你选择的`snitch`类型，创建用于标识数据中心和机架信息的相应属性文件，并将该文件放置在集群所有节点的 Cassandra 配置目录中。请注意，由于数据库仍在使用`cassandra.yaml`文件中先前配置的`snitch`运行，新的`snitch`尚未启用。属性文件可能是以下之一：
    *   用于`GossipingPropertyFileSnitch`、`Ec2Snitch`和`Ec2MultiRegionSnitch`类型的`cassandra-rackdc.properties`文件。
    *   用于所有其他类型网络`snitch`的`cassandra-topology.properties`文件。
2.  更新集群中每个节点上`cassandra.yaml`文件中的`endpoint_snitch`属性值。在此示例中，它是：
    ```yaml
    endpoint_snitch: GossipingPropertyFileSnitch
    ```
3.  逐个启动节点，以便你在`cassandra.yaml`文件中关于新网络`snitch`类型的更改能够生效。

### 存在拓扑变更

如果集群中已有数据，则会发生拓扑变更。当拓扑发生变化时，可能会向集群添加新的数据中心。

让我们考虑一种情况，即没有将数据中心拆分为多个数据中心（通过创建新数据中心）。你最初在一个集群中有五个节点，都使用`SimpleSnitch`。你现在通过更改为使用`RackInferringSnitch`的同一单数据中心中的五个节点但有两个机架的新配置，来进行一次存在拓扑变更的更改。

在这种情况下，拓扑发生了变化（从一个机架变为两个机架），但你没有添加任何新的数据中心。要切换`snitch`，首先执行上一节中的三个步骤。随后，在集群的每个节点上运行`nodetool repair`（顺序执行）和`nodetool cleanup`命令。

如果拓扑有变化并且需要添加新的数据中心，请遵循以下步骤：

1.  使用“创建新数据中心”部分中的步骤，用新节点和机架创建一个新的数据中心。
2.  将数据复制到新数据中心。
3.  从旧数据中心和机架中移除节点。
4.  在集群的每个节点上运行`nodetool repair`（顺序执行）和`nodetool cleanup`命令。

**注意**
如果你仅仅更改`snitch`类型和复制策略，并将一些节点移动到新的数据中心，你将会错误地复制数据。

### 管理 Gossip

你可以使用各种`nodetool gossip-`相关的命令来监控和管理 gossip。

### 获取关于 Gossip 的信息

你可以通过运行`nodetool gossipinfo`命令从集群获取 gossip 信息，如下所示：
```bash
$ nodetool gossipinfo
ubuntu/192.168.159.129
generation:1500142702
heartbeat:7498
STATUS:16:NORMAL,-1270071350005996462
LOAD:7455:342204.0
SCHEMA:12:0c912807-68bb-3cf6-91c3-ee14aba78ca6
DC:8:datacenter1
RACK:10:rack1
RELEASE_VERSION:4:3.10
INTERNAL_IP:6:192.168.159.129
RPC_ADDRESS:3:192.168.159.129
NET_VERSION:1:10
HOST_ID:2:0dbb9e0e-867e-4179-b6b6-631d38dd68f9
RPC_READY:28:true
TOKENS:15:
/192.168.159.130
...
$
```

#### 禁用和启用 Gossip

你可以通过在正在运行的节点上禁用 gossip 协议，有效地将其移出集群，而无需停止实例。

执行`nodetool disablegossip`命令以在节点上禁用 gossip 协议，如下所示：
```bash
$ nodetool disablegossip
```

你可以查看`system.log`文件，看到在 Cassandra 响应了你停止该节点 gossip 的请求后，已将该节点标记为宕机：
```log
WARN  [RMI TCP Connection(2)-127.0.0.1] 2017-07-15 13:31:47,640 StorageService.java:318 - Stopping gossip by operator request
INFO  [RMI TCP Connection(2)-127.0.0.1] 2017-07-15 13:31:47,648 Gossiper.java:1506 - Announcing shutdown
INFO  [RMI TCP Connection(2)-127.0.0.1] 2017-07-15 13:31:47,658 StorageService.java:2248 - Node ubuntu/192.168.159.130 state jump to shutdown
```

如果你运行`nodetool status`命令，它会显示节点模式为`UN`（UP NORMAL）。但是，由于你关闭了 gossip，该节点与集群的其他部分没有联系。

你可以通过运行`nodetool enablegossip`命令重新启用 gossip：
```bash
$ nodetool enablegossip
```

日志显示 Cassandra 已启动 gossip 进程。
```log
WARN  [RMI TCP Connection(6)-127.0.0.1] 2017-07-15 13:42:28,187 StorageService.java:331 - Starting gossip by operator request
INFO  [RMI TCP Connection(6)-127.0.0.1] 2017-07-15 13:42:28,197 StorageService.java:2248 - Node ubuntu/192.168.159.130 state jump to NORMAL
WARN  [GossipTasks:1] 2017-07-15 13:42:29,272 FailureDetector.java:288 - Not marking nodes down due to local pause of 640151266291 > 5000000000
```

#### 检查 Gossip 状态

你可以通过执行`nodetool statusgossip`命令来检查 gossip 是否正在运行，如下所示：
```bash
$ nodetool statusgossip
Running
$
$ nodetool disablegossip
$
$ nodetool statusgossip
not running
$
```

## 刷新与排干数据的区别

`nodetool flush`和`nodetool drain`命令都使你能够将数据从内存表（memtables）移动到磁盘上的 SSTables 中。然而，这两个工具之间存在区别。

### 排干节点

在准备升级 Cassandra 等情况下，你需要确保将所有内存表刷新到磁盘上的 SSTables 中。你可以运行`nodetool drain`命令将内存表刷新到磁盘上的 SSTables。

你执行`nodetool drain`命令如下所示：
```bash
$ nodetool drain
```

日志文件显示如下：
```log
INFO  [RMI TCP Connection(30)-127.0.0.1] 2017-07-15 13:54:58,272 StorageService.java:1435 - DRAINING: starting drain process
INFO  [RMI TCP Connection(30)-127.0.0.1] 2017-07-15 13:54:58,274 HintsService.java:221 - Paused hints dispatch
INFO  [RMI TCP Connection(30)-127.0.0.1] 2017-07-15 13:54:58,279 Server.java:176 - Stop listening for CQL clients
INFO  [RMI TCP Connection(30)-127.0.0.1] 2017-07-15 13:54:58,280 Gossiper.java:1506 - Announcing shutdown
INFO  [RMI TCP Connection(30)-127.0.0.1] 2017-07-15 13:54:58,311 StorageService.java:2248 - Node ubuntu/192.168.159.130 state jump to shutdown
INFO  [RMI TCP Connection(30)-127.0.0.1] 2017-07-15 13:55:00,714 StorageService.java:1435 - DRAINED
```

`nodetool drain`命令很有趣。当你运行此命令时，Cassandra 除了将内存表刷新到磁盘之外，还会执行其他操作。它还会停止监听来自客户端的任何请求或来自其他节点的连接请求。你可以从 Cassandra 在你排干节点后尝试启用 gossip 时的输出中看到这一点。
```bash
$ nodetool enablegossip
nodetool: Unable to start gossip because the node was drained.
See 'nodetool help' or 'nodetool help <command>'.
$
```


## 将内存表数据刷新到磁盘

您也可以通过运行 `nodetool flush` 命令，将内存表刷新到磁盘上的 SSTable 中。您可以刷新整个节点、一个键空间或特定的表。在对数据库或一个或多个键空间进行快照之前，先将内存表刷新到磁盘是一个很好的策略。

以下是 `nodetool flush` 命令的语法：

```
$ nodetool  flush --   (  ... )
```

您可以使用以下命令将所有内存表刷新到磁盘：

```
$ nodetool flush
```

您也可以只刷新一个或多个键空间或表。以下是如何刷新属于 `cycling` 键空间的内存表：

```
$ nodetool flush cycling
```

与 `nodetool drain` 命令不同，`nodetool flush` 命令仅将内存表刷新到磁盘，不做其他任何事情。因此，如果您只需要将数据刷新到 SSTable，使用此命令而不是 `nodetool drain` 命令是更好的选择。

那么，您应该刷新还是排空？正如我所解释的，`drain` 命令会停止 Cassandra 响应客户端请求和其他节点的请求。当您想在维护期间关闭节点并希望节点更快启动时，您会排空内存表。由于您在节点重启前刷新了内存表，因此节点在重启后无需重放提交日志。

### 维护数据中心

常见的与数据中心相关的维护任务包括：向集群添加数据中心、在不中断服务的情况下迁移和重命名集群，以及退役数据中心。

### 向集群添加数据中心

有时您可能需要将一个数据中心添加到现有集群中。本节将向您展示执行此操作的步骤。完成该过程后，新旧数据中心将相互复制数据。

最后，您需要运行 `nodetool rebuild` 命令，该命令在多个节点上操作，并从单个源副本流式传输数据以重建令牌范围。此命令有助于将新数据中心添加到现有集群。

以下是向集群添加数据中心的步骤：

1.  确保所有当前数据中心都使用 `NetworkTopologyStrategy` 作为其复制策略。如果未使用，请运行 `ALTER KEYSPACE` 命令进行修正。提示：向集群添加数据中心时，请不要忘记更新所有键空间的键空间复制策略，以包含新数据中心。
2.  在新数据中心的所有节点上安装 Cassandra 软件，但暂时不要启动 Cassandra 服务。
3.  接下来，您必须在属于您正在添加的新数据中心的所有节点上配置 `cassandra.yaml` 文件中的某些属性。确保重要的配置属性（如 `-seeds` 和 `endpoint_snitch`）在新旧节点上相同。此外，您还必须为新节点配置 `vnode` 令牌的分配。配置设置取决于 `vnode` 选择算法。
    *   随机选择算法：设置 `num_tokens` 属性（推荐值为 256）。
    *   分配算法：在所有节点上设置 `num_tokens` 属性（推荐值为 8）以及 `allocate_tokens_for_local_replication_factor` 属性。后一个属性的推荐值是以下之一：
        *   此数据中心中键空间的最高复制因子
        *   操作最繁忙的键空间的复制因子
4.  在属于新数据中心的每个节点上进行以下更改。
5.  在相应的属性文件中指定探测器类型。您不能使用 `SimpleSnitch`，因为它仅适用于单个数据中心，无法识别数据中心（或机架）信息。根据您选择的探测器，您必须在相应的属性文件中进行更改：
    *   `PropertyFileSnitch`：`cassandra-topology.properties` 文件
    *   `GossipingPropertyFileSnitch`：`cassandra-rackdc.properties` 文件
6.  在旧数据中心中，进行以下配置更改：
    *   在部分现有节点上，添加来自新数据中心的种子节点。由于您修改了 `cassandra.yaml` 文件，因此必须重启您进行此更改的节点。
    *   在相应的属性文件中（取决于探测器类型），指定新数据中心定义。
7.  在每个机架上启动 Cassandra，并继续操作直到启动所有节点。
8.  所有节点启动后，按如下方式更改键空间：

    ```
    ALTER KEYSPACE "my_ks" WITH REPLICATION =
    {'class': 'NetworkTopologyStrategy', 'ExistingDC':3, 'NewDC':3};
    ```

9.  最后，在您刚刚添加的新数据中心中的每个节点上运行 `nodetool rebuild` 命令。重建节点时，您可以指定几个可选参数，例如 `keyspace_name`、`token_spec` 和 `source_dc_name`。`nodetool rebuild` 命令通过从其他节点流式传输数据，一次重建一个或一组键空间。您必须通过 `keyspace_name` 属性指定要重建的键空间的名称，以及 `token_spec` 属性，该属性使您能够指定单个令牌、单个令牌列表或令牌范围（`start_token`，`end_token`）。您可以使用 `nodetool rebuild` 命令指定 `source-dc-name`。此属性指 Cassandra 用作流式传输源的数据中心名称。Cassandra 可以从任何数据中心构建，如果您省略源数据中心的名称，它会随机选择一个数据中心。如果重建因任何原因失败，您可以重新启动它，此时该过程将从停止的地方继续。您还可以通过指定 `-ts` 或 `–token` 选项来指定令牌列表或令牌范围（或多个范围）进行选择性重建。`nodetool rebuild` 命令最简单的调用方式是：

    ```
    $ nodetool rebuild
    ```

### 退役数据中心

退役数据中心等同于移除数据中心。您可以通过以下步骤退役一个数据中心而不会丢失任何数据。

1.  在开始退役数据中心之前，请确保客户端没有向该数据中心的任何节点写入数据。您可以通过运行以下命令确认任何节点中都没有待处理的写入请求：

    ```
    $ nodetool tpstats
    ```

2.  通过运行完全修复来传播您计划退役的数据中心中的数据：

    ```
    $ nodetool repair -full
    ```

3.  更改数据库中的所有键空间，确保没有任何键空间引用您正在退役的数据中心。假设您有三个数据中心：`DC1`、`DC2` 和 `DC3`。您要从集群中移除数据中心 `DC3`。因此，您必须从所有键空间配置中移除数据中心 `DC3`。如果您为任何键空间设置了 `DC3` 的复制因子，请运行 `ALTER KEYSPACE` 命令，如下所示，以从该键空间的配置中移除数据中心 `DC3`：

    ```
    cqlsh> alter keyspace cycling WITH replication = {'class':
    'NetworkTopologyStrategy', 'DC1':1,'DC2':2};
    ```

4.  在您要退役的数据中心的所有节点上运行 `nodetool decommission` 命令。

    ```
    $ nodetool decommission
    ```

5.  退役完成后，关闭节点。您可以通过运行 `nodetool status` 命令来确认数据中心中的所有节点都已被移除。

### 处理数据损坏

Cassandra 提供了工具来检查损坏的数据，并通过使用损坏的数据重建表来修复数据损坏。

以下部分将向您展示如何检测数据损坏以及如何重建损坏的表。


### 检查数据损坏

运行 `sstableverify` 命令来检查特定 SSTable 的错误或损坏数据。以下是一个示例，其中 `cycling` 是键空间的名称，`cycle` 是你想要检查的表名：

```
$ sstableverify --verbose cycling cycle
Verifying BigTableReader(path='/cassandra/apache-cassandra-3.10/data/data/cycling/cycle-2276fb7064e911e7b186d794a4e00229/mc-2-big-Data.db') (0.029KiB)
...
Checking computed hash of BigTableReader(path='/cassandra/apache-cassandra-3.10/data/data/cycling/cycle-2276fb7064e911e7b186d794a4e00229/mc-3-big-Data.db')
$
```

### 修复损坏的数据

你可以通过重建 SSTable 来修复损坏的数据。重建 SSTable 将移除损坏的数据，同时保留完好的数据。有两种工具可以帮助你重建表：`nodetool scrub` 命令和 `sstablescrub` 工具。这两个工具非常相似。你的首选是 `nodetool scrub` 命令。

#### 通过重建表来移除损坏的数据

你可以借助 `nodetool scrub` 命令移除损坏的数据。此命令为一个或多个表重建 SSTable。

`nodetool scrub` 命令的语法如下：

```
$ nodetool  scrub 
--  -ns | --no-snapshot
-s | --skip-corrupted    ...
```

以下是一个示例，展示了如何清理 `cycling` 键空间中的表：

```
INFO  [CompactionExecutor:50] 2017-07-16 11:47:14,560 OutputHandler.java:42 - Scrubbing BigTableReader(path='/cassandra/apache-cassandra-3.10/data/data/cycling/cyclist_name-4d1743b060ef11e7805be14006afbdda/mc-1-big-Data.db') (0.206KiB)
INFO  [CompactionExecutor:50] 2017-07-16 11:47:14,604 OutputHandler.java:42 - Scrub of BigTableReader(path='/cassandra/apache-cassandra-3.10/data/data/cycling/cyclist_name-4d1743b060ef11e7805be14006afbdda/mc-1-big-Data.db') complete; looks like all 0 rows were tombstoned
...
```

`nodetool scrub` 命令会重建 SSTable，并在此过程中丢弃可能已损坏的数据。为了保障数据安全，该命令在重建表之前还会对 SSTable 的数据文件创建快照。

以下是 `nodetool scrub` 命令的主要选项：

*   `--no-snapshot`：此选项将禁用 Cassandra 为要重建的 SSTable 创建默认快照。指定此选项可以节省存储空间，并且意味着你无需稍后手动移除这些快照。
*   `--skip-corrupted`：此选项允许重建操作跳过那些无法根据列的数据类型验证其值的损坏分区。

#### 离线重建表

`sstablescrub` 工具帮助你在节点离线时重建表。其工作方式与 `nodetool scrub` 命令相同。如果 `nodetool scrub` 命令因数据损坏而无法重建表，那么你的下一个选择应该是运行 `sstablescrub` 工具。

由于 `sstablescrub` 是一个离线工具，你必须首先关闭节点。之后，运行以下命令来清理表：

```
$ sstablescrub cycling cycle
```

### 管理移交与提示

你可以使用 `nodetool` 工具来管理移交过程和提示机制的多个方面。通过运行 `nodetool` 命令，你可以启用和禁用移交、暂停和恢复移交，以及为特定数据中心禁用和启用提示。以下是与移交和提示相关的 `nodetool` 命令列表：

*   `nodetool disablehandoff`：禁用在节点上存储提示
*   `nodetool enablehandoff`：启用在节点上存储提示
*   `nodetool pausehandoff`：暂停提示的分发
*   `nodetool resumehandoff`：恢复提示的分发

`nodetool enablehintsfordc` 命令使你能够为特定数据中心开启提示。当你通过 `cassandra.yaml` 文件中的 `hinted_handoff_disabled_datacenters` 属性将某个数据中心列入黑名单时，或者当你使用 `nodetool disablehintsfordc` 命令为某个数据中心禁用了提示时，可以使用此命令。

以下是如何为一个数据中心切换提示的开启与关闭状态：

```
$ nodetool enablehintsfordc DC2
$ nodetool disablehintsfordc DC2
```

如果某个数据中心宕机，或者你正在将其故障切换出去，你应该为其禁用提示。数据库将继续向其他数据中心发送提示。

### 清除节点上的 Gossip 状态

Cassandra 在每个节点上存储 Gossip 信息，以便节点在重启时使用。如果节点需要从集群中的其他地方检索 Gossip 数据，节点启动速度将会变慢。有时，当节点没有正确的 Gossip 状态时，你可能需要手动修复 Gossip 相关问题。

请按照以下步骤修复不正确的 Gossip 状态问题。

1.  关闭遇到 Gossip 问题的节点。
    ```
    $ nodetool assassinate
    ```
2.  停止客户端应用向 Cassandra 集群写入数据。
3.  将所有节点离线，首先排空每个节点。
    ```
    $ nodetool drain
    $ sudo service cassandra stop
    ```
4.  删除 `peers-UUID` 目录中的所有目录。
    ```
    $ sudo rm -r /var/lib/cassandra/data/system/peers-UUID/8
    ```
5.  从 `peers` 目录中移除数据后，在每个节点上运行以下 SQL 语句，以确保节点可以互相发现：
    ```
    cqlsh> select * from system.peers;
    ```
6.  在每个节点上设置 `cassandra.load_ring_state=false` 属性，以便在下一步重启节点时清除 Gossip 状态。你可以从命令行设置此属性，也可以通过将该属性添加到 `cassandra-env.sh` 文件中来实现（如本章“节点下线”小节中针对另一个参数的说明）。
7.  重启集群中的所有节点。确保撤销你在第 6 步中可能对 `cassandra-env.sh` 文件所做的任何更改。

## 总结

在本章中，我解释了几项重要的集群管理任务。`nodetool` 工具在移除节点、替换死亡节点、移动节点、销毁节点以及从环中移除节点时提供了巨大帮助。你还学习了如何退出和添加数据中心。

在测试集群中练习这些任务是在进行集群管理任务时建立信心的最佳方式。在执行某些变更时，你很可能会遇到小问题，而练习诸如退出节点和数据中心等任务会教会你如何快速修复这些问题，从而在生产环境中的变更过程中节省时间。

# 10. 监控、日志与指标

监控 Cassandra 在很多方面类似于管理传统关系型数据库。其复杂性在于 Cassandra 是一个分布式数据库，因此你还需要关注数据的分布以及集群中节点之间工作负载的平衡。

`nodetool` 工具提供了许多命令来帮助你监控集群，你在前面的章节中已经学习了其中的许多命令。

JConsole 是一个强大的监控工具，你既可以从命令行使用，也可以作为 GUI 工具使用。

Cassandra 提供了大量指标，帮助你评估和监控集群的性能和健康状况。这些指标包括表和键空间指标，以及缓存和客户端请求指标等。本章回顾了重要的 Cassandra 指标。

在本章的最后一节，我将展示如何设置和配置 Nagios，这是一个流行的监控 Cassandra 集群的工具。



### `nodetool proxyhistograms` 命令

`nodetool proxyhistograms` 命令显示集群中的网络统计信息。

```
$ nodetool proxyhistograms
proxy histograms
Percentile       Read Latency      Write Latency      Range Latency   CAS Read Latency  CAS Write Latency View Write Latency
(micros)           (micros)           (micros)           (micros)           (micros)           (micros)
50%                    943.13               0.00            4866.32               0.00               0.00               0.00
75%                   4055.27               0.00           14530.76               0.00               0.00               0.00
95%                   4055.27               0.00           20924.30               0.00               0.00               0.00
98%                   4055.27               0.00           20924.30               0.00               0.00               0.00
99%                   4055.27               0.00           20924.30               0.00               0.00               0.00
Min                    785.94               0.00            1358.10               0.00               0.00               0.00
Max                   4055.27               0.00           20924.30               0.00               0.00               0.00
$
```

### 获取表级统计信息

运行 `nodetool tablestats`（原 `nodetool cfstats`）命令可以获取一个或多个表的统计信息。务必指定键空间和表名，如以下示例所示。默认情况下，Cassandra 会输出所有表的统计信息！

Cassandra 在你刷新数据或数据库通过压缩更改 SSTable 时更新表统计信息。

你可以通过使用 `-I` 标志指定表名来让 Cassandra 忽略某些表。以下示例展示了如何获取某个表的统计信息：

```
$ nodetool tablestats test.kv2
Total number of tables: 40

Keyspace : test
Read Count: 0
Read Latency: NaN ms.
Write Count: 0
Write Latency: NaN ms.
Pending Flushes: 0
Table: kv2
SSTable count: 1
Space used (live): 5012
Space used (total): 5012
Space used by snapshots (total): 0
Off heap memory used (total): 194
SSTable Compression Ratio: 1.2
Number of keys (estimate): 1
Memtable cell count: 0
Memtable data size: 0
Memtable off heap memory used: 0
Memtable switch count: 0
Local read count: 0
Local read latency: NaN ms
Local write count: 0
Local write latency: NaN ms
Pending flushes: 0
Percent repaired: 0.0
Bloom filter false positives: 0
Bloom filter false ratio: 0.00000
Bloom filter space used: 176
Bloom filter off heap memory used: 168
Index summary off heap memory used: 18
Compression metadata off heap memory used: 8
Compacted partition minimum bytes: 30
Compacted partition maximum bytes: 35
Compacted partition mean bytes: 35
Average live cells per slice (last five minutes): NaN
Maximum live cells per slice (last five minutes): 0
Average tombstones per slice (last five minutes): NaN
Maximum tombstones per slice (last five minutes): 0
Dropped Mutations: 0
$
```

如你所见，`nodetool tablestats` 命令生成了关于表的详尽统计信息。除了空间使用统计外，该命令还揭示了读写延迟统计和布隆过滤器相关的统计信息。

### 从主机获取网络信息

你可以通过运行 `nodetool netstats` 命令来获取节点的网络信息。该命令的输出显示诸如节点的操作模式（NORMAL、DECOMMISSIONED、LEAVING 等）、读修复统计以及非活动、待处理和已完成的命令及其响应数量等信息。

以下是一个示例：

```
$ nodetool netstats -H
Mode: NORMAL
Not sending any streams.
Read Repair Statistics:
Attempted: 0
Mismatch (Blocking): 0
Mismatch (Background): 0
Pool Name                    Active      Pending      Completed   Dropped
Large messages                  n/a           1              0         0
Small messages                  n/a           0           1058         0
Gossip messages                 n/a           0          18589         0
$
```

默认情况下，Cassandra 会显示你发出此命令的节点的结果，但它允许你使用 `-h` 选项指定一个远程节点。

### `nodetool tablehistograms` 命令

`nodetool tablehistograms`（原 `nodetool cfhistograms`）命令提供关于表的统计信息，可用于绘制频率函数。

此命令不是累积性的；它仅监视自当前会话中你上次运行该命令以来的操作。

以下是如何运行 `nodetool tablehistograms` 命令的示例。在此命令中，`cycling` 指键空间，`cycle` 指表。

```
$ nodetool tablehistograms cycling cycle
cycling/cycle histograms
Percentile  SSTables     Write Latency      Read Latency    Partition Size        Cell Count
(micros)          (micros)           (bytes)
50%             0.00              0.00          0.00         35         1
75%             0.00              0.00          0.00         35         1
95%             0.00              0.00          0.00         35         1
98%             0.00              0.00          0.00         35         1
99%             0.00              0.00          0.00         35         1
Min             0.00              0.00          0.00         30         0
Max             0.00              0.00          0.00         35         1
$
```

以下是命令输出中关键列的描述：

*   `Percentile`：百分位数排名
*   `SSTables`：最近一次读取中每次读取访问的 SSTable 数量
*   `Write Latency`：最近写入的写入延迟（微秒）
*   `Read Latency`：最近读取的读取延迟（微秒）
*   `Partition Size`：分区大小（字节）

# 检查集群健康状况

在监控集群和执行例行维护任务时，`nodetool` 实用程序是你最好的帮手。

你经常用于检查集群健康状况的关键 `nodetool` 命令如下：

*   `nodetool status`
*   `nodetool info`
*   `nodetool tpstats`

在接下来的章节中，我将回顾这些关键的 nodetool 集群健康监控工具。

### `nodetool status` 命令

在本书中，你已经多次看到 `nodetool status` 命令的实际应用。此命令使你能够检查集群节点的健康状况。此外，它还能让你了解节点间的数据分布情况。使用此命令监控集群，如果它显示由于机架中节点过多而导致集群不平衡，请使用前一章中解释的技术移动部分节点。



### `nodetool info` 命令

运行 `nodetool info` 命令以获取节点信息，例如磁盘负载、运行时间和堆内存使用情况。该命令还提供有关数据库如何利用其三个缓存（键缓存、行缓存和计数器缓存）的宝贵信息。

`nodetool info` 命令提供有关节点的宝贵信息，例如以下内容：

*   磁盘存储（负载）信息
*   启动次数（世代）
*   运行时间
*   堆内存使用情况
*   键、行、计数器和块缓存信息
*   Gossip 状态（是否活动）
*   修复百分比
*   令牌信息（可选）

以下是一个示例：

```
$ nodetool info
ID                     : 7d9e3ac3-ffdc-4cee-be82-4b0c6e614aa1
Gossip active          : true
Thrift active          : false
Native Transport active: true
Load                   : 394.92 KiB
Generation No          : 1500219460
Uptime (seconds)       : 8441
Heap Memory (MB)       : 95.36 / 1014.00
Off Heap Memory (MB)   : 0.00
Data Center            : datacenter1
Rack                   : rack1
Exceptions             : 0
Key Cache              : entries 32, size 2.7 KiB, capacity 50 MiB, 159 hits, 200 requests, 0.795 recent hit rate, 14400 save period in seconds
Row Cache              : entries 0, size 0 bytes, capacity 0 bytes, 0 hits, 0 requests, NaN recent hit rate, 0 save period in seconds
Counter Cache          : entries 0, size 0 bytes, capacity 25 MiB, 0 hits, 0 requests, NaN recent hit rate, 7200 save period in seconds
Chunk Cache            : entries 17, size 1.06 MiB, capacity 221 MiB, 58 misses, 291 requests, 0.801 recent hit rate, 497.940 microseconds miss latency
Percent Repaired       : 100.0%
Token                  : (invoke with -T/--tokens to see all 256 tokens)
$
```

## 使用线程池统计信息 (nodetool tpstats)

`nodetool tpstats` 命令显示线程池的使用统计信息。Cassandra 将任务分为不同的阶段，每个阶段使用单独的队列和线程池。消息服务连接各个阶段。

`nodetool tpstats` 命令通过线程池提供有关操作每个阶段的信息。它显示以下内容：

*   活动线程数
*   等待线程池执行的请求数
*   线程池已完成的任务数
*   由于下一步的线程池已满而被阻塞的请求数
*   截至当前时间，此线程池中被阻塞的请求总数

当您刷新内存表或数据库压缩任何 SSTable 时，数据库会刷新 `nodetool tpstats` 命令提供的信息。

以下示例展示了如何运行 `nodetool tpstats` 命令：

```
$ nodetool tpstats
Pool Name              Active   Pending      Completed   Blocked  All time blocked
ReadStage                   0         0              0         0       0
MiscStage                   0         0              0         0       0
CompactionExecutor          0         0           5532         0       0
MutationStage               0         0           1028         0       0
MemtableReclaimMemory       0         0             29         0       0
PendingRangeCalculator      0         0              2         0       0
GossipStage                 0         0          26038         0       0
SecondaryIndexManagement    0         0              0         0       0
HintsDispatcher             0         0              0         0       0
RequestResponseStage        0         0           1031         0       0
Native-Transport-Requests   0         0              0         0       0
ReadRepairStage             0         0              0         0       0
CounterMutationStage        0         0              0         0       0
MigrationStage              0         0              0         0       0
MemtablePostFlush           0         0             73         0       0
PerDiskMemtableFlushWriter_ 0         0              0        29       0
ValidationExecutor          0         0              2         0       0
Sampler                     0         0              0         0       0
MemtableFlushWriter         0         0             29         0       0
InternalResponseStage       0         0              2         0       0
ViewMutationStage           0         0              0         0       0
AntiEntropyStage            0         0              8         0       0
CacheCleanupExecutor        0         0              0         0       0
Message type Dropped
READ                        0
RANGE_SLICE                 0
_TRACE                       0
HINT                         0
MUTATION                     0
COUNTER_MUTATION             0
BATCH_STORE                  0
BATCH_REMOVE                 0
REQUEST_RESPONSE             0
PAGED_RANGE                  0
READ_REPAIR                  0
$
```

`nodetool tpstats` 命令的输出显示了与数据库任务相关的特定线程池的统计信息。所有这些详细信息在您排查问题或调整数据库时非常有用。您可以根据池中的活动确定解决性能问题的策略。池中的高数值指向数据库中问题的症状。

以下是线程池、与这些池相关的任务以及您可以采取的改进措施的简要列表。

*   `AntiEntropyStage`: 此池处理修复消息。您可以使用 `nodetool repair` 命令执行修复。
*   `GossipStage`: 分发节点信息。由于某些模式彼此不同步，您可能会在此池中看到问题。使用 `nodetool resetlocalschema` 命令同步模式。
*   `HintedHandoff`: 将错过的更改（如更新和删除）发送到其他节点。有时由于各种原因，您会看到移交过程出现问题。您可以使用 `nodetool disablehandoff` 和 `nodetool repair` 命令来修复与移交相关的问题。
*   `MutationStage`: 此阶段在本地节点上执行插入和更新，并重放提交日志。进行中的提示也是此阶段的一部分。如果您看到待处理写入请求数量很高，可能表明节点过载。您可以添加节点或重写代码以减少变更。
*   `ReadRepairStage` : 此阶段显示执行读修复的等待。如果 `Pending` 数字很高，您可以为经常读取的表降低 `read_repair_chance` 值，例如设置为 0.11。表的 `read_repair_chance` 属性是成功的读操作将触发读修复的概率（默认值为 0.0）。

您可以密切监视线程池统计信息，如果您看到 `Pending` 任务列的值持续增加，那么就该为集群增加额外容量了。

`nodetool tpstats` 命令输出的底部是关于此节点丢弃消息的部分，意思是该节点收到的消息超出了其处理能力。如果您看到大量的被阻塞任务和/或丢弃的消息，则意味着数据库难以跟上当前的工作负载。



### 使用 JMX 客户端

Cassandra 通过 Java 管理扩展（JMX）公开了大量的度量指标和管理操作命令。JMX 提供了用于监控和管理 Java 应用程序及服务的工具。它可以监控任何统计信息，并管理 Java 应用程序以 MBean（管理 Bean）形式暴露的任何操作。

JMX 还使你能够远程连接到集群的实例，从而监控 Cassandra 性能（内存、线程、CPU 使用率等）并通过帮助你修改某些运行时属性来管理 Cassandra。MBean 是特殊的 JavaBean，使你能够从外部访问 JVM 内的资源。通过 JMX，你可以以编程方式检查各种条目的设置，例如内存、线程、CPU、Gossip 以及其他在 JMX 中被检测的与 Cassandra 相关的组件。

在底层，你执行的 `nodetool` 命令会访问 JMX 度量指标来完成其工作。Nodetool 支持关键的 JMX 度量指标和操作，以及额外的与 Cassandra 管理相关的命令（例如 `proxyhistograms` 命令）。然而，nodetool 无法访问某些度量指标，在这些情况下，你可以使用通用的 JMX 客户端，例如 JConsole 或 jmxsh，我将在以下章节中进行解释。

### JConsole

JConsole 是一个 JMX 客户端，它捕获 Cassandra 公开的 JMX 度量指标和操作，并以图形化方式显示它们。JConsole 是管理 MBean 的标准工具。JConsole 有时在连接远程服务器时难以使用，因为你必须在防火墙中开放其连接所需的端口。因此，此工具并非生产环境的理想选择，尽管它对开发和测试服务器非常棒。

提示

由于 JConsole 消耗大量资源，DataStax 建议你在未托管 Cassandra 实例的节点上运行该工具。

JConsole 提供有关内存和线程使用情况、Java 类加载以及 Java 虚拟机（JVM）和 MBean 的其他信息。你可以使用“内存”选项卡执行 Java 垃圾回收。

JConsole 提供的一个关键度量指标是压缩度量指标。通过监控压缩度量指标，你可以判断何时需要为集群增加容量。

你可以通过 JConsole 连接到 Cassandra 实例。为此，首先启动 JConsole。

```
