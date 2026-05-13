# 第 9 章 生产环境

这里有几点需要注意：

• `"aws:///"` 这个 KMS 密钥 URL 方案并非拼写错误。

• 可在 URL 中使用的密钥 ID 可以通过运行以下 AWS CLI 命令并提取你想要使用的密钥的 `KeyId` 来获取：

```
$ aws kms list-keys
{
  "Keys": [
    {
      "KeyId": "********-****-****-****-************",
      "KeyArn": "arn:aws:kms:eu-west-2:***:key/********-****-****-****-************"
    },
    {
      "KeyId": "********-****-****-****-************",
      "KeyArn": "arn:aws:kms:eu-west-2:***:key/********-****-****-****-************"
    }
  ]
}
```

• 通过使用配置 `AUTH=implicit`，我是在告诉 CockroachDB 使用与访问存储桶相同的凭证来访问 KMS 密钥。如果你的 KMS 密钥使用不同的凭证，请为该密钥提供关联的 `AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY` 和 `REGION` 查询参数。

如果你尝试在没有 KMS 密钥的情况下恢复备份，将会收到错误，因为 CockroachDB 知道此备份是使用 KMS 密钥手动加密的：

```
RESTORE TABLE sensors.sensor_reading FROM '2022/03/08-183653.00' IN 's3://practical-cockroachdb-backups?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&AWS_REGION=eu-west-2';
```

ERROR: file appears encrypted -- try specifying one of `"encryption_passphrase"` or `"kms"`: proto: wrong wireType = 5 for field EntryCounts

## 第 9 章 生产环境

*使用* KMS 密钥进行恢复会按预期执行恢复操作：

```
RESTORE TABLE sensors.sensor_reading FROM '2022/03/08-183653.00' IN 's3://practical-cockroachdb-backups?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&AWS_REGION=eu-west-2'
  WITH kms = 'aws:///****?AUTH=implicit&REGION=eu-west-2';
```

```
  job_id           |  status   | fraction_completed | rows | index_entries | bytes
-------------------+-----------+--------------------+------+---------------+-------
742834697875423233 | succeeded |         1          | 1000 |        0      | 52140
```

要加密一个多区域集群，只需为想要加密的每个区域提供一个存储在该区域的 KMS 密钥。我加密的集群当前位于单个区域，但通过向密钥数组添加更多 KMS 密钥，你可以加密更多区域：

```
BACKUP INTO 's3://practical-cockroachdb-backups?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&AWS_REGION=eu-west-2'
  WITH kms = (
    'aws:///****AUTH=implicit&REGION=eu-west-2'
  );
```

```
  job_id           |  status   | fraction_completed | rows | index_entries | bytes
-------------------+-----------+--------------------+------+---------------+-------
742839873407320065 | succeeded |         1          | 1055 |       65      | 76388
```

```
RESTORE TABLE sensors.sensor_reading FROM '2022/03/08-191858.46' IN 's3://practical-cockroachdb-backups?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&AWS_REGION=eu-west-2'
  WITH kms = (
    'aws:///****AUTH=implicit&REGION=eu-west-2'
  );
```

```
  job_id           |  status   | fraction_completed | rows | index_entries | bytes
-------------------+-----------+--------------------+------+---------------+-------
742840069222793217 | succeeded |         1          | 1000 |        0      | 52150
```

请注意，恢复的行数少于备份的行数。这是因为备份是针对整个集群进行的，而恢复操作仅针对 `sensor_reading` 表。

如果你不想使用云提供商存储的密钥来加密备份，可以提供自己的密码用于加密。以这种方式加密备份的过程与使用云密钥加密非常相似：

```
BACKUP INTO 's3://practical-cockroachdb-backups?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&AWS_REGION=eu-west-2'
  WITH encryption_passphrase = 'correct horse battery staple';
```

```
  job_id           |  status   | fraction_completed | rows | index_entries | bytes
-------------------+-----------+--------------------+------+---------------+-------
742837673818750977 | succeeded |         1          | 1046 |       48      | 69464
```

恢复过程也与 KMS 方式非常相似：

```
RESTORE TABLE sensors.sensor_reading FROM '2022/03/08-190747.27' IN 's3://practical-cockroachdb-backups?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&AWS_REGION=eu-west-2'
```



## 第 9 章 生产环境

KEY=`****&AWS_REGION=eu-west-2'`

WITH `encryption_passphrase` = 'correct horse battery staple';

`job_id` | `status` | `fraction_completed` | `rows` | `index_entries` | `bytes`
--- | --- | --- | --- | --- | ---
742838121558212609 | succeeded | 1 | 1000 | 0 | 52150

#### 区域感知备份

要使备份具备区域感知能力，我们只需向`BACKUP`命令传递一些额外的配置，以确保备份保留在各自的区域内。

让我们使用演示命令创建一个新集群来模拟一个多区域集群。

在下面的例子中，我们指定默认情况下，CockroachDB 应备份到美西存储桶，但具有`us-east-1`区域性的节点除外，这些节点将备份到美东存储桶：

```
BACKUP INTO (
    's3://practical-cockroachdb-backups-us-west?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&COCKROACH_LOCALITY=default',
    's3://practical-cockroachdb-backups-us-east?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&COCKROACH_LOCALITY=region%3Dus-east-1'
);

RESTORE TABLE `sensor_reading` FROM '2022/06/29-183902.85' IN (
    's3://practical-cockroachdb-backups-us-west?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&AWS_REGION=us-west-1&COCKROACH_LOCALITY=default',
    's3://practical-cockroachdb-backups-us-east?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&AWS_REGION=us-east-1&COCKROACH_LOCALITY=region%3Dus-east-1'
);
```

KMS 密钥和加密口令短语可以与区域备份结合使用，从而实现灵活的 CockroachDB 备份和恢复。

#### 计划备份

可以安全地假设，你不会想在每晚午夜手动备份你的集群。这就是 CockroachDB 备份计划的用武之地。以下语句创建了两个备份计划：

- **每周完整备份** – 创建每周完整备份，以确保每周都捕获一个干净的基线备份。
- **每日增量备份** – 根据我们的指令，使用`crontab`语法创建每日备份。

```
CREATE SCHEDULE cluster_backup
    FOR BACKUP INTO 's3://practical-cockroachdb-backups?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&AWS_REGION=eu-west-2'
    WITH revision_history
    RECURRING '@daily'
    WITH SCHEDULE OPTIONS first_run = 'now';
```

此语句的输出将包含两个新创建计划的信息：

- **每周完整备份:**
    - ID: `742844058827390977`
    - 状态: `ACTIVE`
- **每日增量备份:**
    - ID: `742844058837024769`
    - 状态: `PAUSED`: 等待首次备份完成

一旦每日初始备份完成，其状态将从`PAUSED`变为`ACTIVE`。这可以通过调用`SHOW SCHEDULES`来查看：

```
$ cockroach sql --url "postgresql://localhost:26260" --insecure \
--execute "SHOW SCHEDULES" \
--format records
-[ RECORD 1 ]
id             | 742844058837024769
label          | cluster_backup
schedule_status| ACTIVE
next_run       | 2022-03-13 00:00:00+00
recurrence     | @weekly
-[ RECORD 3 ]
id             | 742844058827390977
label          | cluster_backup
schedule_status| ACTIVE
next_run       | 2022-03-09 00:00:00+00
recurrence     | @daily
```

我们的每日备份作业将于明天午夜执行，而每周备份作业将于周日执行。

我们要求 CockroachDB 立即开始第一次每日备份。如果在白天创建计划，这不太可能是你期望的行为。要为首次运行设置不同的开始时间，只需将`first_run`值替换为时间戳。例如：

```
CREATE SCHEDULE cluster_backup
    FOR BACKUP INTO 's3://practical-cockroachdb-backups?AWS_ACCESS_KEY_ID=****&AWS_SECRET_ACCESS_KEY=****&AWS_REGION=eu-west-2'
    WITH revision_history
    RECURRING '@daily'
    WITH SCHEDULE OPTIONS first_run = '2022-03-08 00:00:00+00';
```

### 集群设计

在设计集群时，有许多配置需要决定，以获得最佳性能和弹性。在本节中，我们将重点讨论这些决策及其权衡。

#### 集群规模规划


##### CockroachDB 集群架构与生产环境规划

如果你将 CockroachDB 集群视为分布在各个节点上的一组 vCPU（虚拟 CPU），那么设计集群架构的过程就会变得更简单。

拥有少量大型节点的集群与拥有大量小型节点的集群之间存在权衡。

在由少量大型节点组成的小型集群中，每台机器额外的计算能力以及满足大型查询所需的较少网络跳数，使得集群更加**稳定**。

在由大量小型节点组成的大型集群中，用于分发数据和并行化备份与恢复等分布式操作的额外节点，使得集群更加**有弹性**。

在稳定性和弹性之间做决定是你的选择，但作为经验法则，Cockroach Labs 建议你采取折中方案，在尽可能少的节点上分布 vCPU，同时仍能实现弹性。他们的 [生产检查清单](https://www.cockroachlabs.com/docs/stable/recommended-production-settings.html) 提供了每种情况的详细分解。

#### 节点规模

Cockroach Labs 提供了一套简单的建议，用于根据 vCPU 数量来确定集群节点的规模。让我们针对最低和推荐的 vCPU 数量逐一了解这些建议。请注意，这些只是一般性建议。根据你独特的性能需求，你可能需要不同的比例：

*   **内存** – 每 vCPU 4GB：
    *   4 vCPUs -> 每节点 16GB RAM
    *   8 vCPUs -> 每节点 32GB RAM
*   **存储** – 每 vCPU 150GB：
    *   4 vCPUs -> 每节点 600GB 磁盘
    *   8 vCPUs -> 每节点 1.2TB 磁盘
*   **IOPS** – 每 vCPU 500 IOPS（每秒输入/输出操作数）：
    *   4 vCPUs -> 2,000 IOPS
    *   8 vCPUs -> 4,000 IOPS
*   **MB/s** – 每 vCPU 30MB/s（每秒传输到磁盘或从磁盘传出的数据量）：
    *   4 vCPUs -> 120MB/s
    *   8 vCPUs -> 240MB/s

常言道：“过早优化是万恶之源。” 在为你的 CockroachDB 集群构建架构时也是如此。早在 2018 年，当我创建一个 CockroachDB 集群时，我问 Cockroach Labs 团队是否建议立即在集群前放置一个外部缓存来处理读取密集型工作负载。他们的回答是先利用 CockroachDB 自身的缓存。这样做可以减少网络跳数并提供查询原子性，而数据库与缓存的组合则无法做到这一点。

使用 CockroachDB，你不仅可以扩展集群中的节点数量，还可以扩展节点本身。因此，设计时应支持一个合理的初始容量，并由此开始增长。

### 监控

安全地在生产环境中运行任何软件的一个重要方面是监控。在本节中，我们将为 CockroachDB 集群配置监控，并针对一个简单指标设置警报。

在此示例中，我们将使用 Prometheus，这是一个流行的开源监控系统，CockroachDB 对其有很好的支持。

首先，我们将创建一个要监控的集群：

```shell
$ cockroach start \
--insecure \
--store=node1 \
--listen-addr=localhost:26257 \
--http-addr=localhost:8080 \
--join=localhost:26257,localhost:26258,localhost:26259

$ cockroach start \
--insecure \
--store=node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257,localhost:26258,localhost:26259

$ cockroach start \
--insecure \
--store=node3 \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--join=localhost:26257,localhost:26258,localhost:26259

$ cockroach init --insecure --host=localhost:26257
```

接下来，我们将创建一个 Prometheus 实例来监控我们的三个节点。我们将创建 `prometheus.yml` 和 `alert_rules.yml` 配置文件，并将 Prometheus 实例指向这些文件。

