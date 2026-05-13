# 第 2 章 安装 CockroachDB

**图 2-3.** 多区域、多节点集群

CockroachDB 允许你决定在发生区域级或地区级故障时集群应如何表现。这些被配置为“容灾目标”选项，你的选择应基于对**弹性**和性能的需求。

#### 区域故障目标

区域故障目标（CockroachDB 的默认设置）提供了对云供应商在一个区域内某个*区域*发生故障的弹性。如果同一区域内有两个区域发生故障，你的集群无法保证正常运行。如果你更看重**性能**而非弹性，这可能是一个不错的选择。

要配置区域故障弹性，请对数据库运行以下命令以配置主区域并设置容灾目标：

```sql
ALTER DATABASE "YOUR_DATABASE" PRIMARY REGION "us-east1";
ALTER DATABASE "YOUR_DATABASE" SURVIVE ZONE FAILURE;
```

通过运行以下命令，我们将看到副本数被设置为 3，这是 CockroachDB 无论是否显式配置容灾目标时的默认值。

```sql
SHOW ZONE CONFIGURATION FROM RANGE default;
```

#### 地区故障目标

地区故障目标提供了对云供应商某个*地区*发生故障的弹性。

这是通过将数据库的复制因子从 3 增加到 5 来实现的，这意味着在发生地区中断时，写入会变慢，因为在地区恢复之前，跟随者节点将离租约持有者节点更远。如果你更看重**弹性**而非性能，这可能是一个不错的选择。

要配置地区故障弹性，请对数据库运行以下命令以添加地区、配置主区域并设置容灾目标：

```sql
ALTER DATABASE "YOUR_DATABASE" PRIMARY REGION "us-east1";
ALTER DATABASE "YOUR_DATABASE" ADD REGION "us-west1";
ALTER DATABASE "YOUR_DATABASE" ADD REGION "us-central1";
ALTER DATABASE "YOUR_DATABASE" SURVIVE REGION FAILURE;
```

现在，该数据库已配置为承受地区故障，而非区域故障。

现在运行以下命令将副本数设置为 5。

```sql
SHOW ZONE CONFIGURATION FROM DATABASE defaultdb;
```

### `demo` 命令

`cockroach` CLI 的 `demo` 命令会在你的机器上启动一个内存中的 CockroachDB 企业版集群。此集群的企业版许可证有效期为一小时，这足以让你试用 CockroachDB 的企业功能。一旦过期，你只需重新运行 `demo` 命令即可启动新的一小时。

本书中的大多数示例，我们将通过 `demo` 命令运行本地 CockroachDB 企业版集群。演示的任何 CockroachDB 企业功能都将被清晰标注。

### CockroachDB 无服务器版/专用版

如果你更喜欢使用软件即服务（SaaS）工具，CockroachDB 无服务器版和 CockroachDB 专用版（统称为“CockroachDB 无服务器版”）是开始使用 CockroachDB 的绝佳方式。这两项服务都允许你在几秒钟内启动一个安全的 CockroachDB 集群。根据你的需求，它提供从免费的 CockroachDB 单核/5GB/多租户实例到托管在你选择的云供应商上的多地区企业集群。

#### 创建集群

现在，让我们创建并连接到一个免费层级的 CockroachDB 无服务器版实例。

首先，访问 [`cockroachlabs.cloud`](https://cockroachlabs.cloud)，如果你还没有账户，请注册一个。Cockroach Labs 不会向你索要任何账单信息。

接下来，选择免费计划，选择你首选的云供应商，选择一个部署区域，提供一个名称，然后创建你的集群。在撰写本文时，你可以选择将免费层级实例托管在 GCP 的 `europe-west1`、`us-central1` 和 `asia-southeast1` 区域，或 AWS 的 `eu-west-1`、`us-west-2` 和 `ap-southeast-1` 区域。

大约 20 秒后，你的集群就可以连接了。

#### 连接到你的集群

要从命令行连接到你的集群，请下载集群的根证书。为此，请运行以下命令，将加粗文本替换为你的集群 ID。这将是一个 UUID，当你通过 CockroachDB 无服务器版站点访问集群时，它会构成 URL 路径的一部分：

```bash
$ curl -o root.crt https://cockroachlabs.clusters/**YOUR_CLUSTER_ID**/cert
```

接下来，使用以下命令通过 SQL shell 连接到你的数据库。当你点击“连接”时，UI 会提供加粗的文本：

```bash
$ cockroach sql --url 'postgresql://**YOUR_USERNAME**:**YOUR_PASSWORD**@free-tier5.gcp-europe-west1.cockroachlabs.cloud:26257/defaultdb?sslmode=verify-full&sslrootcert=root.crt&options=--cluster=**YOUR_CLUSTER_NAME**'
```

### 小结

在本章中，我们涵盖了很多内容。让我们回顾一下所学到的知识：

*   **许可选项** – CockroachDB 提供了多种许可选项。CockroachDB 企业版提供地理分区等功能，这些功能在 CockroachDB 核心版中不可用，因此你必须根据需求选择合适的许可。
*   **本地安装** – 我们已经使用 `cockroach` 二进制文件、Docker 镜像和 Kubernetes 在本地安装了 CockroachDB。根据开发阶段的不同，每种方式都有其优势。我们还研究了多节点和多区域集群，这些场景在你的 CockroachDB 基础设施成熟后将变得很重要。
*   **CockroachDB 即服务** – 我们使用 CockroachDB 无服务器版创建了一个简单的免费层级 CockroachDB 集群，它为我们提供了一个 5GB 的数据库来试验 CockroachDB。

![](img/index-45_1.png)

