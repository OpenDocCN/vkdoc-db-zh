# 在 Kubernetes 上部署 SQL Server

### 部署步骤是什么？

我们为您构建了一个简单的教程，用于在 AKS 上试用 SQL Server，详见 [`https://aka.ms/sqlk8s`](https://aka.ms/sqlk8s)。

我还构建了一系列脚本，以更详细地解释部署步骤，详见 [`https://github.com/microsoft/bobsql/tree/master/demos/sqlserver2022/sqlk8s/deploy`](https://github.com/microsoft/bobsql/tree/master/demos/sqlserver2022/sqlk8s/deploy)。

其核心概念是，您向 `k8s` *声明*一个包含单个 SQL Server 容器镜像的 `Pod` 定义。命令行工具 `kubectl` 常用于向 `k8s` 提交您的声明（`kubectl` 使用 `k8s` API）。您还经常使用一个名为 `YAML` 文件的文本来提供声明的详细信息。

在示例脚本中，您需要执行以下步骤：

1.  为 `k8s` 对象创建一个新的命名空间。
2.  为管理员密码创建一个 Secret。
3.  创建一个新的映射到端口 `1433` 的负载均衡器服务。
4.  从 `managed-premium` 存储类创建一个 PVC。
5.  使用 `YAML` 文件部署一个使用 SQL Server 容器镜像的 `Pod`，该镜像引用了 Secret、负载均衡器和 PVC。在 `YAML` 文件中，声明一个副本数为 `1` 的 `Replicaset`，以确保始终有一个 SQL Server `Pod` 在运行。

部署 `Pod` 是一个异步操作。容器镜像如果在节点中不存在则会被拉取并执行。`Pod` 的整体编排远比这复杂。`K8s` 拥有复杂的算法来决定在哪些节点上调度和运行 `Pod`。它还必须通过事件通知来检测 `Pod` 故障或资源限制，并采取相应措施。

### 连接和使用 Kubernetes 上的 SQL Server

在像 `AKS` 这样的系统上，负载均衡器 `IP` 地址和端口是对外可用的，因此可以使用像 `SSMS` 这样的工具通过负载均衡器连接到在 `Pod` 中运行的 `SQL` 实例。在我位于 [`https://github.com/microsoft/bobsql/blob/master/demos/sqlserver2022/sqlk8s/deploy/step12_querysql.ps1`](https://github.com/microsoft/bobsql/blob/master/demos/sqlserver2022/sqlk8s/deploy/step12_querysql.ps1) 的示例脚本中，您可以看到一种动态确定负载均衡器 `IP` 地址以连接到 `SQL Server` 的方法。

`K8s` 包含内置服务来监控资源使用情况，以确保系统保持健康。我们看到的一个与 `k8s` 和 `SQL Server` 相关的问题是，`SQL Server` 可能会在节点内使用大量内存。这可能会触发 `k8s` 强制终止 `Pod` 并将其迁移到另一个节点。这是因为 `k8s` 包含检测节点内应有多少可用内存以保持其健康的算法。然而，`SQL Server` 并不了解这些算法。著名的 `SQL` 和 `Linux` 社区专家 Anthony Nocentino 对此场景进行了很好的阐述，详见 [`www.centinosystems.com/posts/2019-09-28-memory-settings-for-running-sql-server-in-kubernetes`](http://www.centinosystems.com/posts/2019-09-28-memory-settings-for-running-sql-server-in-kubernetes)，其中也包括了如何解决此问题。

### 使用 Kubernetes 实现高可用性

高可用性对于生产环境的 `SQL Server` 至关重要，因此 `k8s` 在系统中提供内置的*基础*高可用性是非常好的。

#### 使用 SQL Server 和 Kubernetes 实现基本高可用性

当您在 `k8s` 上部署一个 `SQL Server Pod` 并使用负载均衡器、`PVC` 和 `Replicaset` 时，您已经向 `k8s` 声明您希望使用基本的高可用性。

考虑以下两种场景：

*   **Pod 故障**
    *   如果 `SQL Server` 因任何原因崩溃，这将被视为 `Pod` 故障，`k8s` 将自动启动一个新的 `Pod`，该 `Pod` 通常会在同一节点上（如果资源可用）启动一个新的 `SQL Server` 容器。由于所有数据库都在 `PVC` 上，这类似于 `SQL Server` 容器的持久化存储。`SQL Server` 只需对所有数据库运行恢复。
    *   `Pod` 故障也可能因其他原因在 `k8s` 内部发生，`k8s` 可能决定在另一个节点上启动新的 `Pod`。节点具有私有 `IP` 地址，但由于您使用了负载均衡器，您的连接会自动重定向到新节点。并且由于您使用了 `PVC`，所有数据库即使在新节点上也是可用的。这就像一个具有共享存储的故障转移集群，而您无需设置或配置任何故障转移集群软件。
*   **节点故障**
    *   如果您在构建 `k8s` 集群时部署了多个节点，那么当某个节点发生故障时，也可能出现相同的情况。事实上，我建议您在任何用于 `SQL Server` 的 `k8s` 集群中*至少*拥有三个节点。一个节点将用于编排其他工作节点，您至少会有两个工作节点以实现冗余。
    *   以下是我的脚本，用于查看 `AKS` 上 `SQL Server` 的 `HA` 实战：[`https://github.com/microsoft/bobsql/tree/master/demos/sqlserver2022/sqlk8s/ha`](https://github.com/microsoft/bobsql/tree/master/demos/sqlserver2022/sqlk8s/ha)。

#### Always On 可用性组与 Kubernetes

在 `SQL Server 2019` 的预览版中，我们引入了一个与 `k8s` 集成的 `Always On` 可用性组部署概念。不幸的是，该功能未能出现在 `SQL Server 2019` 的最终版本中。我们在那里的工作确实演变为 `Azure Arc` 启用的 `SQL` 托管实例所附带的内置 `AG` 功能（通过商业服务层）。

还有另一个来自我们合作伙伴 `DH2i` ([`https://dh2i.com`](https://dh2i.com)) 的解决方案。您可以在 [`https://docs.microsoft.com/sql/linux/tutorial-sql-server-containers-kubernetes-dh2i`](https://docs.microsoft.com/sql/linux/tutorial-sql-server-containers-kubernetes-dh2i) 查看如何使用 `DH2i` 的解决方案在 `k8s` 上部署 `AG`。

### Helm 图表

有些人觉得 `k8s` 的部署体验有点复杂。于是出现了 `Helm` 图表的概念 ([`https://helm.sh`](https://helm.sh))。可以将 `Helm` 视为 `k8s` 的包管理器。您可以在 [`https://docs.microsoft.com/sql/linux/sql-server-linux-containers-deploy-helm-charts-kubernetes`](https://docs.microsoft.com/sql/linux/sql-server-linux-containers-deploy-helm-charts-kubernetes) 查看如何使用 `Helm` 部署 `SQL Server`。

## Linux、容器和 Kubernetes 的世界

对许多人来说，`SQL Server on Linux` 开启了新的大门和可能性。我曾与一些客户交谈，他们组织的不同部门使用 `Windows`，而另一些部门使用 `Linux`，但需要像 `SQL Server` 这样跨平台的一致数据库平台。

但 `Linux` 支持还开启了更多可能性。`SQL Server` 容器可能会改变您组织构建应用程序的方式。想象一下，您公司中的开发人员使用他们选择的操作系统，针对相同的、一致的 `SQL Server` 引擎和数据库构建应用程序。事实上，新的 `Azure SQL Database` 本地开发体验 ([`https://docs.microsoft.com/azure/azure-sql/database/local-dev-experience-overview`](https://docs.microsoft.com/azure/azure-sql/database/local-dev-experience-overview)) 就使用了 `SQL Server` 容器。或者考虑一下，您可以在单个虚拟机中使用容器整合多个 `SQL Server` 实例。

有了容器，就有了 `k8s`，而我们的 `Azure Arc` 启用的 `SQL` 托管实例证明了您可以使用 `k8s` 将 `Azure` 托管服务的强大功能引入您的基础设施。

所有这些都讲述了 `SQL Server` 的选择故事。具有兼容性的选择。您的应用程序无需更改，并且无论 `SQL Server` 部署在哪里，数据库都是可互换的。



