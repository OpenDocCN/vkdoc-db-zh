# 第九章 故障排除 AlwaysOn

SQL Server 公开了大量与高可用性和灾难恢复对象相关的元数据，特别是围绕 AlwaysOn 功能集。这些元数据可用于快速识别配置、查找问题的根本原因或编写针对可能发生的事件的自动响应脚本。以下部分将讨论可用的元数据，并提供如何使用它的示例。

### AlwaysOn 故障转移群集实例元数据

从数据库引擎内部，可以查看关于群集实例及其承载的 Windows 群集的大量元数据。这些信息对 DBA 来说非常宝贵。以下部分将介绍一些最有用和最有趣的元数据对象。

#### 发现实例所在的节点

自然地，DBA 需要知道群集中哪个节点正在托管故障转移群集实例，尤其是在尝试诊断连接或性能问题时。如果您的组织有政策规定 DBA 不允许访问操作系统，则无法使用故障转移管理器。幸运的是，SQL Server 中有一个 DMV（动态管理视图）会公开此信息。`sys.dm_os_cluster_nodes` DMV 将返回表 9-1 中详述的列。

© Peter A. Carter 2016
P. A. Carter, *SQL Server AlwaysOn Revealed*, DOI 10.1007/978-1-4842-2397-0_9
第九章 ■ 故障排除 ALWAYSON

**表 9-1. `sys.dm_os_cluster_nodes` 列**

| 列名 | 描述 |
| :--- | :--- |
| NodeName | 群集节点的名称 |
| Status | 节点的当前状态。可能的值为：<br>• 0 - 表示节点已启动<br>• 1 - 表示节点已关闭<br>• 2 - 表示节点已暂停<br>• 3 - 表示节点正在加入群集<br>• 4 - 表示状态未知 |
| status_description | 状态的文本描述。可能的值为：<br>• Up<br>• Down<br>• Paused<br>• Joining<br>• Unknown |
| is_current_owner | 指示实例当前是否由此节点托管。可能的值为：<br>• 0 - 表示节点不拥有该实例<br>• 1 - 表示节点拥有该实例 |

清单 9-1 中的查询将返回当前托管实例的群集节点的名称。

**清单 9-1.** 发现实例所在的节点

```sql
SELECT NodeName
FROM sys.dm_os_cluster_nodes
WHERE is_current_owner = 1;
```

#### 查看运行状况检查配置

如果协助 Windows 管理团队处理群集实例的重复故障转移，DBA 可能希望公开哪些条件可能导致故障转移，以确保配置了适当的级别。这可以通过使用`sys.`


## 第 9 章 ■ AlwaysOn 故障排除

`dm_os_cluster_properties` 动态管理视图 (DMV) 返回表 9-2 中详述的列。

`表 9-2. sys.dm_os_cluster_properties 的列`

| `列` | `描述` |
| :--- | :--- |
| `VerboseLogging` | 指示群集使用的日志记录级别。可能的值有：<br>• 0 - 指示日志记录已关闭<br>• 1 - 指示仅记录错误<br>• 2 - 指示记录错误和警告 |
| `SQLDumperDumpFlags` | 指定 `SQLDumper` 将生成的转储文件类型。可能的值有：<br>• 0x0120 - 指示小型转储<br>• 0x0110 - 指示完整转储<br>• 0x8100 - 指示筛选转储 |
| `SQLDumperDumpPath` | 指定 `SQLDumper` 将输出转储文件的文件路径 |
| `SQLDumperDumpTimeOut` | `SQLDumper` 创建转储文件时的超时值。以毫秒为单位指定 |
| `FailureConditionLevel` | 将导致发生故障转移的故障级别。故障条件级别的完整描述见表 9-3 |
| `HealthCheckTimeout` | 数据库引擎等待返回运行状况信息的持续时间，之后它将判定实例无响应 |

`FailureConditionLevel` 列返回的可能故障条件级别详见表 9-3。

`表 9-3. 故障条件级别`

| `条件级别` | `描述` |
| :--- | :--- |
| 0 | 不会发生自动故障转移 |
| 1 | 当 SQL Server 服务关闭时发生自动故障转移 |
| 2 | 当满足以下条件时将发生自动故障转移：<br>• 满足级别 1 的条件<br>• 超过 `HealthCheckTimeout` 值 |
| 3 | 当满足以下条件时将发生自动故障转移：<br>• 满足级别 2 的条件<br>• 运行状况检查返回系统错误 |
| 4 | 当满足以下条件时将发生自动故障转移：<br>• 满足级别 3 的条件<br>• 运行状况检查返回资源错误 |
| 5 | 当满足以下条件时将发生自动故障转移：<br>• 满足级别 4 的条件<br>• 运行状况检查返回查询处理错误 |

清单 9-2 中的查询将返回当前的故障转移条件级别和当前的运行状况检查超时值。

`清单 9-2. 返回运行状况检查配置`

```
SELECT
    FailureConditionLevel
    , HealthCheckTimeout
FROM sys.dm_os_cluster_properties ;
```

可以通过使用 `sp_server_diagnostics` 系统存储过程手动确定实例的当前运行状况。该过程接受一个参数 `@repeat_interval`，它指定该过程应返回结果的频率（以秒为单位）。如果省略该参数，则结果只会返回一次。如果为该参数传递了值，则该值必须大于 5。该过程返回表 9-4 中详述的结果集。

`表 9-4. sp_server_diagnostics 返回的列`

| `列` | `描述` |
| :--- | :--- |
| `creation_time` | 指示创建行的时间 |
| `component_type` | 指示组件的类型。可能的值有：<br>• Instance<br>• AlwaysOn: Availability Group |
| `component_name` | 指示组件的名称。可能的值有：<br>• system<br>• resource<br>• query_processing<br>• io_subsystem<br>• events<br>• [可用性组名称] |
| `State` | 组件的运行状况状态。可能的值有：<br>• 0 - 指示状态未知<br>• 1 - 指示状态为干净（表示健康）<br>• 2 - 指示存在警告<br>• 3 - 指示存在错误 |
| `state_desc` | 组件状态的文本描述。可能的值有：<br>• Unknown<br>• Clean<br>• Warnings<br>• Errors |
| `Data` | 组件特定数据的 XML 表示形式。例如，资源组件包括指定可用物理内存和可用虚拟内存的元素。它还包括属性，例如内存不足异常的计数 |

清单 9-3 中的脚本将返回 `sp_server_diagnostics` 系统存储过程的完整结果集，以及已从 XML 中提取的值。

```
EXEC sp_server_diagnostics 5;
```


## 第 9 章：AlwaysOn 故障排除

**提示：** 关于分解 XML 的讨论超出了本书的范围。不过，我推荐 Apress 出版的 *SQL Server DBA 专业脚本与自动化指南* 一书，其中可以找到关于出于管理目的使用 XML 的讨论。该书可在 [www.apress.com/9781484219423](http://www.apress.com/9781484219423) 购买。

### 清单 9-3：检索诊断信息

```sql
CREATE TABLE ##Server_Diagnostics
(
    creation_time DATETIME,
    component_type NVARCHAR(8),
    component_name NVARCHAR(128),
    [state] TINYINT,
    state_desc NVARCHAR(8),
    [data] XML
) ;

INSERT INTO ##Server_Diagnostics
EXEC sp_server_diagnostics ;

SELECT *,
    data.value('(/system/@systemCpuUtilization)[1]','int') AS SystemCPUUtilization,
    data.value('(/system/@sqlCpuUtilization)[1]','int') AS SQLServerCPU,
    data.value('(/resource/@outOfMemoryExceptions)[1]','int') AS OutOfMemoryExceptions
FROM ##Server_Diagnostics ;

DROP TABLE ##Server_Diagnostics ;
```

### AlwaysOn 可用性组元数据

元数据也可用于排查可用性组的问题。以下部分将讨论一些与可用性组相关且最实用、最有趣的元数据对象。

#### 确定上次故障转移原因

如果可用性组发生了故障转移，您可能首先想要回答的问题是“何时？”和“为什么？”。这些可以使用 `sys.dm_hadr_availability_replica_states` DMV 来回答。该对象返回表 9-5 中详述的列。

#### 表 9-5：sys.dm_hadr_availability_replica_states

| 列 | 描述 |
| :--- | :--- |
| `replica_id` | 副本的 GUID |
| `group_id` | 可用性组的 GUID |
| `is_local` | 指示副本是本地还是远程。可能的值有：<br>• `0` - 指示远程辅助副本<br>• `1` - 指示本地副本 |
| `role` | 指示当前分配给副本的角色。可能的值有：<br>• `0` - 指示角色正在解析中<br>• `1` - 指示副本具有主角色<br>• `2` - 指示副本当前具有辅助角色 |
| `role_desc` | 副本当前角色的文本描述。可能的值有：<br>• `RESOLVING`<br>• `PRIMARY`<br>• `SECONDARY` |
| `operational_state` | 指示副本的当前操作状态。可能的值有：<br>• `0` - 指示故障转移正在等待中<br>• `1` - 指示状态为挂起<br>• `2` - 指示在线<br>• `3` - 指示离线<br>• `4` - 指示失败<br>• `5` - 指示失败，且无仲裁<br>• `NULL` - 指示副本不是本地的 |
| `operational_state_desc` | 操作状态的文本描述。可能的值有：<br>• `PENDING_FAILOVER`<br>• `PENDING`<br>• `ONLINE`<br>• `OFFLINE`<br>• `FAILED`<br>• `FAILED_NO_QUORUM`<br>• `NULL` |
| *(续表)* | |
| **列** | **描述** |
| `connected_state` | 指示辅助副本当前是否已连接到主副本。可能的值有：<br>• `0` - 指示副本已与主副本断开连接<br>• `1` - 指示副本已连接到主副本 |
| `connected_state_desc` | 连接状态的文本描述。可能的值有：<br>• `DISCONNECTED`<br>• `CONNECTED` |
| `recovery_health` | 指示可用性组内的数据库是否联机。可能的值有：<br>• `0` - 指示至少有一个数据库未联机<br>• `1` - 指示所有数据库均已联机<br>• `NULL` - 指示可用性组不是本地的 |
| `recovery_health_desc` | `recovery_health` 的文本描述。可能的值有：<br>• `ONLINE_IN_PROGRESS`<br>• `ONLINE`<br>• `NULL` |
| `synchronization_health` | 指示可用性组数据库的同步状态。可能的值有：<br>• `0` - 指示至少有一个数据库处于 `NOT SYNCHRONIZING` 状态。这称为不健康<br>• `1` - 指示至少有一个数据库未处于... |


## 第九章 ■ AlwaysOn 故障排除

##### 表 9-5. （续）

| 列名 | 描述 |
| :--- | :--- |
| synchronization_health_desc | 同步运行状况状态的文本描述。可能的值为 `NOT_HEALTHY`， `PARTIALLY_HEALTHY`， `HEALTHY` |
| last_connect_error_number | 上次连接错误的错误号。 |
| last_connect_error_description | 上次连接错误的描述。 |
| last_connect_error_timestamp | 上次连接错误的日期和时间。 |

清单 9-4 中的查询演示了如何返回上次连接错误的时间和原因。这将指明故障转移发生的时间和原因。每个副本与可用性组的组合将返回一行。你会注意到，我们将 `sys.dm_hadr_availability_replica_states` DMV 与 `sys.availability_replicas` 和 `sys.availability_groups` DMV 进行了联接，以检索承载副本的节点名称以及可用性组的名称。

#### 清单 9-4. 确定上次故障转移的时间和原因

```sql
SELECT
ar.replica_server_name
,ag.name
,ars.last_connect_error_description
,ars.last_connect_error_timestamp
FROM sys.dm_hadr_availability_replica_states ars
INNER JOIN sys.availability_replicas ar
ON ar.group_id = ars.group_id
AND ars.replica_id = ar.replica_id
INNER JOIN sys.availability_groups ag
ON ag.group_id = ar.group_id ;
```

### 评估可用性数据库的状态

你可能已经注意到，`sys.dm_hadr_availability_replica_states` DMV 会提供包含不健康数据库的可用性组的详细信息。然而，其结果不够精细，无法让你发现具体哪些数据库不健康。这些信息可以从 `sys.dm_hadr_database_replica_states` DMV 中检索，该 DMV 返回表 9-6 中详述的列。

#### 表 9-6. sys.dm_hadr_database_replica_states 的列

| 列名 | 描述 |
| :--- | :--- |
| database_id | 数据库的 ID。 |
| group_id | 可用性组的 GUID。 |
| replica_id | 可用性副本的 GUID。 |
| group_database_id | 数据库在可用性组内的 ID。 |
| is_local | 指示数据库是本地还是远程。可能的值为 `0`（非本地）和 `1`（本地）。 |
| is_primary_replica | 指示数据库副本当前是主角色还是辅助角色。可能的值为 `0`（辅助）和 `1`（主）。 |
| synchronization_state | 指示数据库同步状态。可能的值为 `0`（未同步），`1`（正在同步），`2`（已同步），`3`（正在还原）和 `4`（正在初始化）。 |
| synchronization_state_desc | 同步状态的文本描述。可能的值为 `NOT SYNCHRONIZING`， `SYNCHRONIZING`， `SYNCHRONIZED`， `REVERTING`， `INITIALIZING`。 |
| is_commit_participant | 指示事务提交是否已同步。异步副本上的数据库始终报告 `0`，该值仅对同步副本上的主数据库准确。可能的值为 `0`（未同步）和 `1`（已同步）。 |
| synchronization_health | 指示数据库的事务同步运行状况状态。可能的值为 `0`（NOT_HEALTHY），`1`（PARTIALLY_HEALTHY）和 `2`（HEALTHY）。 |


#### 数据库同步状态参数说明

`指示数据库的同步状态。`

可能的值为

• `0` - 表示不健康。这意味着数据库未在同步

• `1` - 表示部分健康。这意味着数据库正在同步

• `2` - 表示健康。这意味着数据库已同步

`synchronization_health_desc`

同步健康状态的文字描述。

可能的值为

• `NOT_HEALTHY`

• `PARTIALLY_HEALTHY`

• `HEALTHY`

`database_state`

指示数据库的当前状态。该值反映了 `sys.databases` 目录视图中的值。

可能的值为

• `0` - 表示数据库处于在线状态

• `1` - 表示数据库正在还原

• `2` - 表示数据库正在恢复

• `3` - 表示数据库处于等待恢复状态

• `4` - 表示数据库处于可疑状态

• `5` - 表示数据库处于紧急模式

• `6` - 表示数据库处于离线状态

`database_state_desc`

数据库状态的文字描述。可能的值为

• `ONLINE`

• `RESTORING`

• `RECOVERING`

• `RECOVERY_PENDING`

• `SUSPECT`

• `EMERGENCY`

• `OFFLINE`

`is_suspended`

指示数据库是否已挂起。

可能的值为

• `0` - 表示已恢复

• `1` - 表示已挂起

`suspend_reason`

如果数据库已挂起，`suspend_reason` 列将指示原因。可能的值为

• `0` - 表示用户手动挂起了数据移动

• `1` - 表示在强制故障转移后挂起

• `2` - 表示在重做阶段发生错误

• `3` - 表示在日志捕获过程中发生错误

• `4` - 表示在写入日志时发生错误

• `5` - 表示数据库在重启前已挂起

• `6` - 表示在撤销阶段发生错误

• `7` - 表示日志链不匹配错误

• `8` - 表示在计算辅助副本的同步点时发生错误

`suspend_reason_desc`

`suspend_reason` 列的文字描述。可能的值为

• `SUSPEND_FROM_USER`

• `SUSPEND_FROM_PARTNER`

• `SUSPEND_FROM_REDO`

• `SUSPEND_FROM_CAPTURE`

• `SUSPEND_FROM_APPLY`

• `SUSPEND_FROM_RESTART`

• `SUSPEND_FROM_UNDO`

• `SUSPEND_FROM_REVALIDATION`

• `SUSPEND_FROM_XRF_UPDATE`

`recovery_lsn`

在主副本上，`recovery_lsn` 表示事务日志的结尾（事务日志中用于时间点恢复的最后一点）。在辅助副本上，该列表示需要重新同步的点。但是，如果该值大于或等于 `last_hardened_lsn`，则表示不需要重新同步。

`truncation_lsn`

对于主副本，该列表示所有辅助副本上的最小日志截断 LSN。对于辅助副本，该列表示该特定数据库副本的日志截断点。

`last_sent_lsn`

指示已发送的最后一个日志块的结尾。

`last_sent_time`

发送最后一个日志块的日期和时间。

`last_recieved_lsn`

指示要接收的最后一个日志块的结尾。

`last_hardened_lsn`

指示要固化（hardened）的最后一个日志块的开始。对于异步提交副本，该值将为 `NULL`。

`last_hardened_time`

固化的 LSN 的日期和时间。

`last_redone_lsn`

在辅助副本上重做的最后一条日志记录的 LSN。

`last_redone_time`

在辅助副本上重做的最后一条 LSN 的时间戳。

`log_send_queue_size`

尚未发送到辅助副本的日志记录的大小，以千字节为单位。

`log_send_rate`

日志记录被发送到辅助副本的速度，以千字节/秒为单位。

`filestream_send_rate`

FILESTREAM 文件被发送到辅助副本的速度，以千字节/秒为单位。

`end_of_log_lsn`

日志缓存中最后一条日志记录的 LSN。

`last_commit_lsn`


## 第 9 章 ■ AlwaysOn 故障排除

### 事务日志中最后提交事务的 LSN

### `last_commit_time`
事务日志中最后提交的 LSN 的时间戳

### `low_water_mark_for_ghosts`
虚影清理任务（用于物理删除已逻辑删除的行）使用此列在数据库所有副本中的最小值，来确定从何处开始清理记录。

### `secondary_log_seconds`
辅助副本落后于主副本的秒数

清单 9-5 中的脚本演示了如何评估 App1 可用性组内的可用性数据库的健康状况。您会注意到，我们使用 `DB_NAME()` 函数返回数据库名称，并将 `sys.dm_hadr_database_replica_states` DMV 与 `sys.availability_groups` 和 `sys.availability_replicas` 目录视图连接，以返回可用性组和副本的名称。

`清单 9-5.` 评估可用性数据库的状态
```sql
SELECT
    DB_NAME(database_id)
    ,ag.name
    ,ar.replica_server_name
    ,is_primary_replica
    ,synchronization_state_desc
    ,synchronization_health_desc
    ,database_state_desc
FROM sys.dm_hadr_database_replica_states drs
INNER JOIN sys.availability_groups ag
    ON drs.group_id = ag.group_id
INNER JOIN sys.availability_replicas ar
    ON drs.replica_id = ar.replica_id
WHERE ag.name = 'App1';
```

### 摘要
SQL Server 提供了大量元数据，可帮助您排查问题并审核配置。虽然本章节讨论了一些最有用的元数据，但我强烈建议您进一步探索 `hadr` 系列 DMV。

对于故障转移群集实例，`sys.dm_os_cluster_nodes` DMV 会公开群集中节点的运行状况状态。可以通过调用系统存储过程 `sp_server_diagnostics` 找到更详细的故障排除信息。

**提示：** 系统存储过程 `sp_server_diagnostics` 也可用于排除 AlwaysOn 可用性组的故障，并为实例上承载的每个可用性组返回一行。

`sys.dm_hadr_availability_replica_states` DMV 公开了托管 AlwaysOn 可用性组的副本的详细健康状态信息。要深入查看参与可用性组的数据库的健康状态，可以查询 `sys.dm_hadr_database_replica_states` DMV。

上述两个 DMV 都可以与 `sys.availability_groups` 和 `sys.availability_replicas` DMV 连接，以获取有关配置的文本信息，而不是 GUID。`sys.dm_hadr_database_replica_states` DMV 也可以与 `sys.databases` 连接，以获取有关数据库配置的更多信息。

## 索引
**A**
- `数据库页面`，92–93
- `数据同步页面`，98–99
- `活动节点`，10
- `端点选项卡`，95
- `高级加密标准 (AES)`，95
- `FAILOVER_CONDITION_LEVEL 参数`，108
- `AlwaysOn 介绍页面`，90–91
- `侦听器对话框`，100
- `侦听器选项卡`，97–98
- `多子网群集`，97
- `网络流量`，97
- `副本页面`，93–94
- `脚本`，101
- `服务账户`，95
- `摘要页面`，100
- `同步提交选项`，93–94
- `验证页面`，99–100

**AlwaysOn 管理**
- `可用性组`
- `群集维护`，149
- `群集节点配置页面`，153
- `配置可能的所有者`，152
- `故障转移群集管理器`，150
- `移动群集角色对话框`，150
- `在节点间移动实例`，149，151
- `覆盖优先级`，149
- `PowerShell`，151，154
- `删除节点向导`，153–154
- `删除可能的所有者`，151–152
- `滚动修补升级`，151–152
- `软件保障`，149
- `指定名称页面`，91–92

**AlwaysOn 可用性组 (AOAG)**
- `活动/被动群集`，26–27
- `App1 和 App2`，83
- `App1Customers 和 App1Sales 数据库`，121
- `异步故障转移`，157
- `自动页面修复`，22
- `备份数据库`，90，27

**C**
- `数据库和日志文件`，164
- `数据库创建`，84
- `群集技术`，18

**D**
- `数据层应用程序`，19–20
- `灾难恢复`
- `HA/DR 拓扑`，25，83
- `高可用性选项卡`，89
- `Last_Failover_Reason 返回时间和原因`，199


## 索引

### AlwaysOn 可用性组

- `sys.dm_hadr_availability_replica_states`，197–199
- ASYNCDR，26–27, 121
- 监听器对话框
    - 创建
- App2Customers 数据库，109
- `Application Intent` 参数，94
- 备份与还原
    - 参数，106–108
    - 数据库，109
- `Backup Preferences` 选项卡，96
- `Backup Preferences` 选项卡，111
- 数据库镜像端点，95
- `常规` 选项卡，110

© Peter A. Carter 2016

P.A. Carter, `SQL Server AlwaysOn 揭秘`, DOI 10.1007/978-1-4842-2397-0

### AlwaysOn 仪表板

- IP 地址，112
- 添加/删除列，169
- `主角色` 属性，110
- App1 可用性组，167–168
- 副本属性，110
- 群集仲裁信息
    - `会话超时` 属性，110
    - 屏幕，168–169
- TCP 端点，109
- `分组依据` 按钮，169
- 事务日志备份，109
- 同步状态，168
- 负载均衡，19

### AlwaysOn 故障转移群集实例

- 活动/活动配置，11
- 监控工具
    - 活动节点，10
    - AlwaysOn 仪表板，167
    - AlwaysOn 运行状况跟踪，170
    - 多个监听器
        - 客户端访问点页面，161–162
        - 确认页面，162
        - 依赖项选项卡，163
        - 硬编码的连接字符串，161
- 多子网群集，21, 121
- 性能基准，117
- `PRIMARYREPLICA`，83
- 五节点 N+M 配置，12–13
- 高可用性，9
- 攻击方法学，73
- 安装故障转移群集规则，64–65
- 删除数据库，163–164
- 使应用程序进入安全状态并进行故障转移，157, 159
- 横向扩展需求，19
- 单一连接字符串，19
- 单用户模式，163
- SQL Server 配置，89
- 独立实例，89
- 可用性数据库状态
    - 评估可用性数据库运行状况，204
    - 返回，194
- `DB_NAME()` 函数，204
- `sys.dm_hadr_database_replica_states` 列，200–203
- 暂停数据移动，164
- SYNCHA，83
- 同步提交模式，112
- 同步故障转移
    - 介绍页面，155
    - Microsoft 更新页面，62–63
    - 主副本页面，155
    - 副本页面，155–156
    - 摘要页面，156
- 同步副本，18
- 任务，83, 121
- 非包含对象，同步，161

### 群集

- 创建
    - 管理访问点，44–45
    - 开始页面，37–38
    - 确认页面，41–42, 46
    - DHCP，44
    - PowerShell，48
    - 报告，47
    - 服务器页面，38–39
    - 摘要页面，42–43, 46–47
    - 测试选项页面，40–41
    - 验证报告，43–44, 48
    - 验证警告页面，39–40
- 数据中心，13
- 磁盘配置，29
- 安装，故障转移功能
    - 开始页面，31
    - 确认页面，36–37
    - 功能页面，34–35
    - 安装类型页面，31–32
    - 管理工具，35–36
    - 服务器角色页面，33–34
    - 服务器选择页面，32–33
- 仲裁
    - 定义，13
    - 高可用性，13
    - 模型，14
    - 多子网群集，14
    - 分区，13
    - 脑裂，13
- SAN 复制，9
- SDK 和管理工具，67
- 服务，PowerShell
    - 命令，37
- 站点感知群集功能，10
- SR 技术，9

### 数据库

- 元数据
    - DBA，191
    - DMV，191
    - 故障条件级别，194
    - 承载实例，192
    - 检索诊断信息，sp_server_diagnostics，194–195
    - `sys.dm_os_cluster_nodes` 列，192
    - `sys.dm_os_cluster_properties` 列，193
    - 查看运行状况检查配置
        - Windows 管理团队，192

### 安装

- 群集网络配置页面，69
- 群集资源组页面，68
- 排序规则选项卡，71–72
- 数据库引擎配置页面，72
- 数据目录选项卡，73–74
- 错误日志和默认扩展事件运行状况跟踪，66
- 功能选择页面，65–66
- FILESTREAM 选项卡，75–76
- 混合模式身份验证，73
- MSSQL13.[实例名称]，66
- 节点
    - 群集网络配置页面，78–79
    - 群集节点配置页面，78
    - 许可条款页面，77
    - 参数，81
    - PowerShell，80–81
    - 产品密钥页面，77
    - 准备添加节点页面，80
    - 服务账户页面，79–80
- 参数安装，77
- 执行卷维护任务，71
- PowerShell 安装，59, 76–77
- 产品密钥页面，60–61
- 产品更新页面，63–64
- 服务器配置页面，70, 72–73
- 服务账户选项卡，70–71
- SQL Server 安装中心—安装，59–60

### 监控

- AlwaysOn 仪表板，167
- AlwaysOn 运行状况跟踪，170
- 群集磁盘选择页面，69–70
- 群集网络配置页面，69
- 群集仲裁信息屏幕，168–169
- 日志流复制，26
- AlwaysOn 故障转移群集实例
    - 活动节点，10
    - 群集磁盘选择页面，69–70
    - 群集网络配置页面，69
    - 群集资源组页面，68
    - 排序规则选项卡，71–72
    - 确认页面，162
    - 数据库引擎配置页面，72
    - 数据目录选项卡，73–74
    - 依赖项选项卡，163
    - 错误日志和默认扩展事件运行状况跟踪，66
    - 功能选择页面，65–66
    - FILESTREAM 选项卡，75–76
    - 实例配置页面，67
    - 许可条款页面，61–62
    - Microsoft 更新页面，62–63
    - 混合模式身份验证，73
    - MSSQL13.[实例名称]，66
    - 主副本页面，155
    - 产品密钥页面，60–61
    - 产品更新页面，63–64
    - 副本页面，155–156
    - 服务器配置页面，70, 72–73
    - 服务账户选项卡，70–71
    - SQL Server 安装中心—安装，59–60
    - 摘要页面，156

### MSDTC 配置

- 客户端访问点页面，53
- 确认页面，54
- 创建，55

### 生产环境步骤，157

### 可读的辅助副本，144

### SQL Server

- Integration Services 服务，66

### 可用性数据库状态

- `DB_NAME()` 函数，204
- `sys.dm_hadr_database_replica_states` 列，200–203
- 评估可用性数据库运行状况，204

### 同步故障转移

- 介绍页面，155
- Microsoft 更新页面，62–63
- 主副本页面，155
- 副本页面，155–156
- 摘要页面，156

### 任务，83, 121


## 系统索引

### A
*   `AlwaysOn 运行状况跟踪`，165
*   `异步镜像`，15
*   `可用性组侦听器`，18
*   `可用性组故障转移`，20

### B
*   `关键业务应用程序`，1

### C
*   `磁盘检查命令 (CHKDSK)`，5
*   `群集`
    *   `ClustNode1` 和 `ClustNode2`，29
    *   故障排除问题，29
*   `群集验证向导`，58, 122
*   `配置群集仲裁向导`，124
*   `停机成本`
    *   App2 可用性组，143
    *   群集，142
    *   编码，144
    *   灾难恢复站点，142
    *   网络流量，142
    *   故障转移脚本，160
    *   步骤，159
    *   拓扑，142–143
*   `使用选项创建事件会话`，188–189

### D
*   `数据库镜像`
    *   AlwaysOn 可用性组，15
    *   数据层应用程序，15
    *   弃用技术，15
    *   灾难恢复解决方案，15
    *   高性能模式，15–16
    *   高安全性、自动故障转移模式，16–17
    *   模式，15
    *   网络延迟，16
    *   主服务器和辅助服务器，16
    *   同步和异步方法，16
    *   TCP 端点，15
    *   Windows 群集服务，15
    *   见证服务器，16
*   `数据损坏`，5
*   `灾难恢复 (DR)`，1
*   `分布式可用性组`，19, 149
*   `分布式资源调度器 (DRS)`，18
*   `分布式事务协调器 (DTC)`，52
*   `动态管理视图 (DMV)`，191
*   `动态仲裁`，14

### E
*   `扩展事件`
    *   操作，181
    *   AlwaysOn 事件，172–180
    *   通道，172
    *   CPU 利用率，171
    *   关键字/类别，171–172
    *   映射，182
    *   监视可用性组会话
    *   捕获全局字段页面，185
    *   捕获页面，184–185
    *   数据存储，186–187
    *   筛选器页面，186
    *   属性页面，183
    *   摘要页面，187
    *   模板页面，183–184
    *   包，171
    *   前身，171
    *   谓词，181–182
    *   分析器，171
    *   会话，182
    *   目标，180–181
    *   类型，182
    *   WMI，171

### F
*   `故障转移群集管理器`，48, 122
*   `完全限定域名 (FQDN)`，96

### G
*   `全局字段`，181

### H
*   `高可用性 (HA)`
    *   数据损坏/人为错误，1
    *   实现，1
*   `超文本标记语言 (HTML)`，46

### I, J, K
*   `IP 地址`
    *   可用性组侦听器属性，139–140
    *   确认页面，127–128
    *   核心群集资源窗口，128
    *   依赖关系选项卡，129–130
    *   依赖关系报告，140–141
    *   常规选项卡，129
    *   或依赖关系，130
    *   脚本，140

### L
*   `可用性级别`
    *   计算，2–3
    *   停机时间，2
    *   整体监控工具，2
    *   网络/应用服务器，1–2
    *   主动维护，4
    *   服务级别协议和服务级别目标，3–4
    *   正常运行时间，1
*   `日志序列号 (LSN)`，22
*   `日志传送`
    *   灾难恢复，23
    *   灾难恢复和报告服务器，23–24
    *   故障转移，25
    *   恢复模式，24
    *   远程监视服务器，25
    *   拓扑，23

### M
*   `Microsoft 群集服务 (MCS)`，58
*   `Microsoft 分布式事务协调器 (MSDTC)`，30, 52
*   `Microsoft 客户体验改善计划`，77
*   `混合模式身份验证`，73

### N, O, P
*   `节点`，10
*   `非功能性需求 (NFRs)`，25

### Q
*   `仲裁`
    *   配置选项页面，124–125
    *   配置文件共享见证页面，126–127
    *   确认页面，127–128
    *   投票配置页面，125
    *   见证页面，126

### R
*   `可读辅助副本`
    *   可用性组侦听器，145
    *   负载平衡拓扑，145–146
    *   日志流，144
    *   日志截断，144
    *   读取意向流量，147
    *   只读路由配置，145, 147
    *   轮询算法，145
    *   快照隔离，144
    *   临时统计信息，144
    *   报告，144
*   `恢复点目标 (RPO)`，93
    *   应用程序，4
    *   数据损坏，5
    *   数据仓库，4
    *   站点内可用性和站点间恢复，4
    *   OLTP（联机事务处理）数据库，4
*   `恢复时间目标 (RTO)`，97
    *   数据损坏，5
    *   站点内/站点间故障转移，5
    *   未提交事务，4
*   `冗余基础设施`，7

### S
*   `服务级别协议 (SLAs)`，3–4, 93
*   `服务级别目标 (SLOs)`，3–4
*   `SQLCMD 模式`，100
*   `SQL Server 集成服务 (SSIS)`，30, 52, 65
*   `SQL Server Management Studio (SSMS)`，131
*   `备用服务器分类`，6
*   `存储副本 (SR)`，9
*   `同步提交模式`
    *   可用性组拓扑，112
    *   网络延迟和磁盘性能，112
    *   性能测试结果
    *   SQL Server 2014，117
    *   SQL Server 2016，117
    *   SAN 复制，118
    *   脚本，112–116
    *   三节点群集，118
*   `系统运营中心 (SOC)`，167

### T, U
*   `TempDB 数据库`，74
*   `决胜器`，15
*   `总拥有成本 (TCO)`，6
*   `事务撤消文件 (TUF)`，24
*   `传输控制协议 (TCP)`，96

### V
*   `虚拟计算机对象 (VCO)`，98
*   `虚拟机 (VMs)`，18

### W, X, Y, Z
*   `Windows 群集服务 (WCS)`，9, 30
*   `Windows Server 更新服务 (WSUS)`，62

### 其他独立条目
*   `系统数据库`，66
*   `停机时间`，52
*   `任务`，59
*   `DTC 资源`，55
*   `TempDB 选项卡`，74–75
*   `高可用性向导`，52
*   `三节点群集`，10
*   `角色页面`，52
*   `三节点以上配置`，12
*   `SQL Server`，52
*   `双节点群集`，10
*   `存储页面`，53–54
*   `Windows 身份验证模式`，73
*   `Windows Server`，55
*   `Windows 防火墙`，65
*   `仲裁配置`
*   `云见证`，49
*   `扩展事件会话`，170
*   `磁盘`，48, 51
*   `目标数据`，170
*   `文件共享见证`，49
*   `选项页面`，49
*   `PowerShell 命令`，51
*   `存储见证页面`，50
*   `摘要页面`，51
*   `见证页面`，49, 50
*   `故障转移选项卡`，57
*   `常规选项卡`，55–56
*   `选项`，56
*   `任务`，30
*   `App2 可用性组`，143
*   `无形成本`，5
*   `可用性级别`，5–6
*   `预计生命周期`，5
*   `有形成本`，5
*   `分布式事务协调器 (DTC)`，52
*   `动态管理视图 (DMV)`，191
*   `备份首选项选项卡`，133
*   `代码实现`，135–139
*   `连接时间`，141
*   `数据同步页面`，134–135
*   `端点选项卡`，132–133
*   `侦听器选项卡`，134
*   `副本页面`，131
*   `SQLCMD 模式`，135
*   `SSMS`，131
*   `摘要页面`，135
*   `验证页面`，135



*   内容概览
*   目录
*   关于作者
*   关于技术审阅者
*   致谢
*   第 1 章：高可用性与灾难恢复概念
    *   可用性级别
        *   服务级别协议与服务级别目标
        *   主动维护
    *   恢复点目标和恢复时间目标
    *   停机成本
    *   备用服务器的分类
    *   小结
*   第 2 章：理解高可用性与灾难恢复技术
    *   AlwaysOn 故障转移群集
        *   活动/活动配置
        *   三节点以上配置
        *   仲裁
    *   数据库镜像
    *   AlwaysOn 可用性组
        *   自动页修复
    *   日志传送
        *   恢复模式
        *   远程监视服务器
        *   故障转移
    *   组合技术
    *   小结
*   第 3 章：实现群集
    *   构建群集
        *   安装故障转移群集功能
        *   创建群集
    *   配置群集
        *   更改仲裁
        *   配置 MSDTC
        *   配置角色
    *   小结
*   第 4 章：实现 AlwaysOn 故障转移群集实例
    *   构建实例
    *   使用 PowerShell 安装实例
    *   添加节点
    *   使用 PowerShell 添加节点
    *   小结
*   第 5 章：使用 AlwaysOn 可用性组实现高可用性
    *   准备可用性组
    *   配置 SQL Server
    *   创建可用性组
        *   使用新建可用性组向导
        *   编写可用性组脚本
        *   使用新建可用性组对话框
    *   同步提交模式的性能考量
    *   小结
*   第 6 章：使用 AlwaysOn 可用性组实现灾难恢复
    *   配置群集
        *   添加节点
        *   修改仲裁
        *   添加 IP 地址
    *   配置可用性组
        *   添加并配置副本
        *   添加 IP 地址
        *   改善连接时间
    *   分布式可用性组
    *   配置可读的辅助副本
    *   小结
*   第 7 章：管理 AlwaysOn
    *   管理群集
        *   在节点间移动实例
        *   滚动修补升级
        *   从群集中移除节点
    *   管理 AlwaysOn 可用性组
        *   故障转移
            *   同步故障转移
            *   异步故障转移
            *   故障转移分布式可用性组
        *   同步非包含对象
        *   添加多个侦听器
        *   其他管理考量
    *   小结
*   第 8 章：监控 AlwaysOn 可用性组
    *   AlwaysOn 仪表板
    *   AlwaysOn 运行状况跟踪
    *   使用扩展事件监控 AlwaysOn
        *   扩展事件概念
            *   包
            *   事件
            *   目标
            *   操作
            *   谓词
            *   类型
            *   映射
            *   会话
        *   创建事件会话以监控可用性组
    *   小结
*   第 9 章：故障排除 AlwaysOn
    *   AlwaysOn 故障转移群集实例元数据
        *   发现托管实例的节点
        *   查看运行状况检查配置
    *   AlwaysOn 可用性组元数据
        *   确定上次故障转移的原因
        *   评估可用性数据库的状态
    *   小结
*   索引
