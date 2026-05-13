# Java 大页面设置

默认情况下，在大多数新的 Linux 发行版中，透明大页面功能是启用的，这意味着在处理透明大页面时，内核以 2MB 大小的块分配内存，而不是 4K。有时，对于使用 4K 大小页面分配内存的应用程序，当内核需要对被许多微小 4K 页面碎片化的大型 2MB 页面进行碎片整理时，服务器性能会受到影响。

您可以通过禁用大页面的碎片整理来避免因碎片整理导致的性能影响，如下所示：
```
$ echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

### 安装 PDSH

由于您将管理多节点集群，获得一个能同时运行命令或向多个节点发送文件的工具是个好主意。通过使用像 `pdsh` 这样的工具在您的整个集群上同时运行命令，可以使集群管理变得简单。我将向您展示如何下载、安装和使用此工具。

`pdsh` 实用程序是 `rsh` 命令的一个变体，是一个高性能的并行 shell 实用程序。`rsh` 允许您在单个远程主机上运行命令，而 `pdsh` 允许您同时在多个远程服务器上运行命令。

当您需要在 Cassandra 节点的所有节点上发出相同的命令时，只需使用 `pdsh` 从单个服务器发出该命令，它就会在集群上执行该命令。

您可以使用 `pdsh` 发布几种类型的 Linux 命令，包括查看文件内容的命令。

您可以通过在命令行发出命令来使用 `pdsh`，或者以交互方式运行该工具。交互运行时，`pdsh` 会提示您输入命令，并在出现回车时执行它们。您也可以在文件中指定命令。

`pdsh` 发行版还包括一个名为 `pdcp` 的并行远程复制实用程序，它将文件从本地主机并行复制到一组远程主机。

您可以通过以下方式安装 `pdsh`：
```
# rpm –Uvh http://download.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm
# yum install pdsh
```

使用 `pdsh` 执行远程操作非常简单。以下示例展示了如何通过从集群中的任何节点运行单个命令来检查集群中所有节点的日期：
```
# pdsh –w "all_nodes" date
```

参数 `all_nodes` 指向一个列出集群中所有节点的文件。您也可以在发出 `pdsh` 命令时通过指定适当的选项来排除某些服务器。


### 初始化 Cassandra 多节点集群（单数据中心与多数据中心）

在第 2 章中，你已经学习了如何配置单节点的 Cassandra 集群。在本章中，你将学习如何配置多节点的 Cassandra 集群，首先是单数据中心，然后是多数据中心。

在多节点集群中，Cassandra 会自动发现节点，因此要设置一个 Cassandra 集群，你只需在所有节点上安装 Cassandra，如下一小节所示。安装 Cassandra 后，你启动 Cassandra 实例，它们会自动组成一个集群。你只需要让每个节点知道其余节点的 IP 地址；这就够了！

正如你从第 1 章中可能回忆起的那样，**数据中心**只不过是一组节点的集合，代表了一组具有相同复制属性的节点。数据中心可以是逻辑的，也可以是物理的。

在接下来的讨论中，我将从头开始创建一个 Cassandra 集群。如果你是想将一个单节点集群转变为多节点集群，则必须首先停止 Cassandra 服务器并清除数据，如第 2 章所述。

我在这里展示的步骤将使你能够使用单数据中心安装、配置和运行一个多节点集群。要设置具有多个数据中心的多节点集群，你需要遵循相同的步骤，但要在 `cassandra-rackdc.properties` 文件中配置多个数据中心，正如我在本节后面解释的那样。

### 先决条件

在启动你的多节点集群之前，你必须处理一些先决条件，如下文所述。

#### 配置防火墙端口访问

当你有一个单节点集群时，防火墙访问端口不是什么大问题。在多节点集群中，你必须确保，如果承载 Cassandra 集群的节点上运行着防火墙，你必须打开多个端口，包括一些 Cassandra 端口，以使节点之间能够相互通信。

如果你在其中某个节点上启动 Cassandra 后忘记打开端口，该节点将不会加入集群，并且会作为一个独立的 Cassandra 实例运行。

你必须打开三组端口：公共端口、Cassandra 节点间端口和 Cassandra 客户端端口。

##### 公共端口

端口号 `22`，用作 SSH 端口

##### Cassandra 节点间端口

端口 `7000`：用于 Cassandra 节点间集群通信

端口 `7001`：用于 Cassandra SSL 节点间集群通信

端口 `7199`：用于 Cassandra JMX 监控

##### Cassandra 客户端端口

端口 `9042`：Cassandra 原生客户端端口

端口 `9160`：Cassandra 客户端端口 (Thrift)

#### 为数据中心选择名称

在创建多节点集群之前，为集群中的每个数据中心和机架选择一个命名约定。数据中心名称是一个必需参数，用于确保不属于此数据中心的节点不会尝试加入它。

提示

一旦你为数据中心分配了一个名称，以后就无法更改。

在这里，我选择 `datacenter1` 作为数据中心的名称。

#### 收集所有节点的 IP 地址

你需要获取集群中所有节点的 IP 地址。在这个例子中，我有六个节点，它们的 IP 地址是：

```
node0 192.168.177.132
node1 192.168.177.133
node2 192.168.177.134
node3 192.168.177.135
node4 192.168.177.136
node5 192.168.177.137
```

这个六节点集群将跨越两个机架，并且有一个数据中心。

#### 选择作为种子节点的节点

**种子提供者**是集群中的一个节点，它帮助 Cassandra 节点相互发现并了解环的拓扑结构。这是多节点集群的一个必需参数。

你通过配置 `cassandra.yaml` 文件中的 `seed_provider` 参数来指定集群的种子节点。

你通过设置此参数的 `seeds` 属性的值来配置 `seed_provider` 参数，如下所示：

```
seed_provider:
- class_name: org.apache.cassandra.locator.SimpleSeedProvider
parameters:
- seeds: "192.168.177.132,192.168.177.135"
```

你以逗号分隔的 IP 地址列表（“<ip1>, <ip2>, <ip3>”）形式提供种子节点列表。

每个数据中心可以只用一个种子节点，但最佳实践是拥有多个种子节点。在这个例子中，我选择有两个种子，所以我的 `seeds` 属性值为：

```
"192.168.177.132,192.168.177.135"
```

### 配置集群

你在至关重要的 `cassandra.yaml` 文件中配置集群属性，该文件可以在 `$CASSANDRA_HOME/conf` 目录中找到。

正如我在第 2 章中提到的，这个文件中有数百个可以配置的参数，但在这里列出一大堆参数对你并没有太大帮助。因此，我列出了启动新集群所需的最少属性集，这意味着其余参数将使用它们的默认值。当你学习后续章节时，你会找到关于可以在 `cassandra.yaml` 文件中指定的所有配置参数的解释。

为集群的每个节点在 `cassandra.yaml` 文件中设置以下属性。或者，你可以在一个节点上设置属性，然后使用前面描述的 `pdsh` 工具将文件复制到其他节点。

#### num_tokens 属性

`num_tokens` 属性定义了 Cassandra 分配给特定节点的令牌数量。相对于其他节点，令牌数量越高，该节点存储的数据量就越大。由于理想情况下所有节点规模相同，你希望所有节点具有相同数量的令牌。

注意

`initial_token` 属性是一个遗留参数，你必须忽略它。如果你指定了 `initial_token` 属性，它会覆盖 `num_tokens` 属性。

`num_tokens` 属性有助于创建**虚拟节点**或 **vnodes**，这有助于将令牌范围划分为许多小范围。然后，每个 Cassandra 节点被分配一组 vnodes。Cassandra 根据该节点的 `num_tokens` 值计算每个集群节点的令牌范围。

Vnodes 在处理具有不同配置和容量的机器的集群时特别有用。你可以为拥有更多计算资源的机器分配更多的 vnodes，通过与其余节点相比，为这些节点的 `num_tokens` 属性指定更高的值来实现。

由于 vnodes 有助于将令牌环拆分为多个较小的范围，它们有助于将集群操作更均匀地负载到各个节点上，从而加快一些操作，例如引导新节点。

`num_tokens` 属性的默认值以及推荐值是 `256`。

#### –seeds 属性

我已经解释了 `–seeds` 属性，它要求你指定你选作种子节点的节点的内部 IP 地址。

#### listen_address 参数

`listen_address` 参数指的是节点的 IP 地址。这是要绑定的网络地址，并告诉其他 Cassandra 节点连接到此地址。你可以设置它，也可以保持原样。如果你不设置此属性，Cassandra 会从主机获取本地地址，但有时它无法获取正确的地址，在这种情况下，你必须在 `cassandra.yaml` 文件中提供 `listen_address`。

你不能将 `listen_address` 属性的值设置为 `0.0.0.0`。


#### `rpc_address` 和 `broadcast_rpc_address` 属性

`rpc_address` 属性指定了用于绑定 Thrift RPC 服务和原生传输服务器的地址。你可以将 `rpc_address` 属性留空；这种情况下，Cassandra 会根据你为节点配置的主机名来获取它。

与 `listen_address` 属性的情况不同，你可以为 `rpc_address` 属性指定值 `0.0.0.0`。这种情况下，你必须同时为 `broadcast_rpc_address` 属性设置一个值：

```
broadcast_rpc_address: 192.168.177.135
```

#### `endpoint_snitch` 选项

在 Cassandra 集群中，snitch（嗅探器）有两个作用：

1.  它告知 Cassandra 网络拓扑信息，以便高效地路由请求。
2.  它使 Cassandra 能够将数据副本分散在集群中，从而避免关联性故障。Cassandra 使用数据中心和机架来逻辑分组集群的节点。它尽力避免将同一数据的多个副本存储在单个机架上。Cassandra 提供了半打（六种）snitch，但对于生产环境，首选选项是 `GossipingPropertyFileSnitch`。

当你选择 `GossipingPropertyFileSnitch` 选项时，你需要在该节点的 `cassandra-rackdc.properties` 文件中指定数据中心和机架。然后，Cassandra 会通过 gossip 协议将此信息传播到其他节点。

#### `auto_bootstrap` 属性

`auto_bootstrap` 属性的默认值是 `true`，并且该参数不会出现在 `cassandra.yaml` 文件中。这个参数使得新的非种子节点将数据迁移到自身。当你正在初始化一个没有数据的新集群时，请添加以下属性：

```
auto_bootstrap=false
```

使用我在这里列出的最小配置属性集，我的 `cassandra.yaml` 文件内容如下：

```
cluster_name: "MyCluster'
num_tokens: 256
seed_provider:
- class_name: org.apache.cassandra.locator.SimpleSeedProvider
parameters:
- seeds: "192.168.177.132,192.168.177.135"
listen_address:
endpoint_snitch: GossipingPropertyFileSnitch
```

#### 配置数据中心和机架名称

在我的集群中，我有一个数据中心和一个机架。我必须在 `cassandra-rackdc.properties` 文件中指定数据中心和机架名称。

```
# indicate the rack and dc for this node
dc=DC1
rack=RACK1
```

我需要在集群的所有六个节点上都这样做。

至此，Cassandra 的安装和配置就完成了。我将在“启动和停止多节点集群”一节中展示如何启动这个集群。

### 使用多个数据中心初始化集群

前面的讨论展示了如何设置使用单个数据中心和单个机架的 Cassandra 集群。配置使用多个数据中心和机架的集群同样简单！你遵循相同的步骤，只是你需要在 `cassandra-rackdc.properties` 文件中配置多个数据中心和机架。

在单数据中心的情况下，我在 `cassandra-rackdc.properties` 文件中指定了以下属性：

```
dc=DC1
rack=RACK1
```

我想设置两个数据中心，每个数据中心包含我六个 Cassandra 节点中的三个。为了做到这一点，我以如下方式配置 `cassandra-rackdc.properties` 文件。

在前三个节点（192.168.177.32, 192.168.177.33, 和 192.168.177.34）的 `cassandra-rackdc.properties` 文件中，我指定以下数据中心和机架值：

```
dc=dc1
rack=rack1
```

对于我六节点集群的其他三个节点，我指定以下值：

```
dc=dc2
rack=rack2
```

这就是创建一个多数据中心集群所需做的全部工作（我把机架改成了 `rack2`，但这其实不是必须的）。

### 启动和停止多节点集群

你的集群已经配置完毕，准备就绪，只是需要将其启动。要启动集群，首先依次启动两个种子节点。完成后，再依次启动其他四个节点。由于这是一个 tarball 安装，我使用以下命令启动节点：

```
$ $CASSANDRA_HOME/bin/cassandra
```

你现在可以通过执行以下命令来检查环是否正在运行：

```
# CASSANDRA_HOME/bin/nodetool status
```

为准备投入使用，所有六个节点的状态都应显示为 `UN`（Up Normal，正常运行）。

你可以使用以下脚本集来启动和停止你的多节点集群。

#### 启动集群的脚本

没有脚本就无法管理集群。你可以编写比我这里提供的更复杂的脚本，但下面这个脚本在帮助启动集群方面可以胜任工作。

在这个例子中，我的 Cassandra 集群中有三个节点。

```
#!/bin/bash
SERVERS="
192.168.177.131
192.168.177.132
192.168.177.133"
for SERVERNAME in $SERVERS; do
sleep 30
echo Starting node $SERVERNAME...
sudo –u cassandra ssh $SERVERNAME "usr/share/cassandra/bin/cassandra"
done
```

`sleep` 命令用于在启动每个服务器前提供一个短暂的间隔。在生产集群中，不带 `sleep` 命令运行此脚本可能会导致问题。

#### 停止集群的脚本

你知道可以使用 `cassandra` 命令来启动集群，但不能用它来停止集群。你必须杀死正在运行的 Cassandra 实例的 PID 才能停止它。为了自动化停止一组节点的 Cassandra 实例，你可以使用以下策略：

*   使用第一个脚本来遍历服务器列表并调用你的停止脚本。
*   使用第二个脚本，通过杀死 Cassandra 实例的 PID 来停止服务。

以下是这两个脚本。

**脚本 1**

使用此脚本调用 `cassandra-kill.sh` 脚本。

```
#!/bin/bash
SERVERS="
192.168.177.131
192.168.177.132
192.168.177.133"
for SERVERNAME in $SERVERS; do
echo Starting node $SERVERNAME...
sudo –u cassandra ssj $SERVERNAME "/usr/share/cassandra/bin/cassandra"
done
```

**脚本 2 (`cassandra-kill.sh`)**

你可以创建一个简单的 shell 脚本，如下所示，来关闭一个 Cassandra 节点：

```
#!/bin/bash
CASS_PID='ps –ef |grep CasandraDaemon |grep –v grep |awk '{ print $2 }"
if [[ "$CASSPID" == '']]
then
echo Cassandra is NOT running
else
kill $CASS_PID
fi
```

### 集群中节点的启动过程

在第 2 章中，我展示了单节点的启动过程。在本章中，我展示了如何创建一个包含多个节点的集群。请注意，当你启动第一个节点时，它显示已准备好工作，并显示其他节点加入集群。在这种情况下，我正在查看节点 `192.168.177.132` 的启动消息。

```
INFO  14:12:49 Node /192.168.177.132 state jump to NORMAL
INFO  14:12:49 Waiting for gossip to settle before accepting client requests...
INFO  14:12:57 No gossip backlog; proceeding
INFO  14:12:58 Starting listening for CQL clients on /0.0.0.0:9042 (unencrypted)...
INFO  14:12:58 Binding thrift service to /0.0.0.0:9160
INFO  14:12:58 Listening for thrift clients...
INFO  14:12:59 Handshaking version with /192.168.177.135
INFO  14:12:59 Scheduling approximate time-check task with a precision of 10 milliseconds
INFO  14:12:59 Handshaking version with /192.168.177.135
INFO  14:13:01 Node /192.168.177.135 has restarted, now UP
INFO  14:13:01 Updating topology for /192.168.177.135
INFO  14:13:01 Updating topology for /192.168.177.135
INFO  14:13:01 InetAddress /192.168.177.135 is now UP
INFO  14:13:01 Handshaking version with /192.168.177.135
INFO  14:13:01 Node /192.168.177.135 state jump to NORMAL
```

类似地，当你关闭第二个节点（或它崩溃）时，第一个节点的日志消息会显示该信息：

```
INFO  14:13:42 Handshaking version with /192.168.177.135
INFO  14:19:14 InetAddress /192.168.177.135 is now DOWN
```

### 启动时的常见错误

在运行像 Cassandra 这样的分布式数据库时，你可能会遇到无数错误。我想指出两个常见问题以及如何克服它们。

### 节点 IP 地址变更

在虚拟机上运行 Cassandra 时，节点的 IP 地址有时可能会发生变化。当这种情况发生时，集群中的其他节点自然无法连接到该节点，它会显示为宕机节点。

修复 IP 地址变更的方法很简单。你只需编辑该节点的 `cassandra.yaml` 文件，并将所有设置了节点 IP 地址的地方（例如 `listen_address` 配置属性）从旧的 IP 地址更改为新的 IP 地址即可。

### 模式版本不匹配

有时在创建键空间或表时会遇到错误，例如下面这个，Cassandra 因版本不匹配而报错：

```
cqlsh> create keyspace mykeyspace2
... with replication = {'class': 'NetworkTopologyStrategy', 'datacenter1' :2}
... and durable_writes = false;
Warning: schema version mismatch detected, which might be caused by DOWN nodes; if this is not the case, check the schema versions of your nodes in system.local and system.peers.
OperationTimedOut: errors={'192.168.177.135': 'Request timed out while waiting for schema agreement. See Session.execute_async and Cluster.max_schema_agreement_wait.'}, last_host=192.168.177.135
cqlsh>
```

尽管出现此消息，Cassandra 仍会创建键空间或表。当发生模式不一致时，请遵循以下步骤：

1.  运行 `nodetool describecluster` 命令。

    ```
    $ sudo nodetool describecluster
    Cluster Information:
    Name: Test Cluster
    Snitch: org.apache.cassandra.locator.DynamicEndpointSnitch
    Partitioner: org.apache.cassandra.dht.Murmur3Partitioner
    Schema versions:
    UNREACHABLE:27a8739d-28ac-34b7-b738-1b84859866cf: [192.168.177.135]
    282bdefc-9643-3fa5-b03a-4b9894cabb29: [192.168.177.132]
    $
    ```

2.  重启无法访问的节点。
3.  再次运行 `nodetool describecluster` 命令，并确保所有节点具有相同的版本号。命令输出必须显示集群中所有节点共用一个模式版本。

注意

如果你有多个不匹配的模式（三个或更多），你需要停止某个特定模式的节点，让其他节点稳定下来，然后逐一重启这些节点。这种情况在多数据中心集群中偶尔会发生。

### 具有不同设置的键空间

如果节点的键空间设置不同，你会注意到以下情况：

```
$ sudo nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load    Tokens       Owns    Host ID      Rack
DN  192.168.177.128  114.06 MiB  256          ?       99c43633-c691-4dee-b7af-35bc6e74dd67  rack1
UN  192.168.177.132  123.84 MiB  256          ?       b0ade950-937a-457c-95eb-d3032897eeb1  rack1
Note: Non-system keyspaces don't have the same replication settings, effective ownership information is meaningless
$
```

### 节点宕机

如果其中一个节点宕机，你将看到如下输出：

```
$ sudo nodetool describecluster
Cluster Information:
Name: Test Cluster
Snitch: org.apache.cassandra.locator.DynamicEndpointSnitch
Partitioner: org.apache.cassandra.dht.Murmur3Partitioner
Schema versions:
44ed2562-e330-3030-af87-89cde4aa8992: [192.168.177.135]
UNREACHABLE: [192.168.177.132]
#
```

运行 `nodetool status` 命令来检查集群中两个节点的状态：

```
$ sudo nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID      Rack
UN  192.168.177.132  276 KiB    256          52.2%             b0ade950-937a-457c-95eb-d3032897eeb1  rack1
UN  192.168.177.135  296.85 KiB  256         47.8%             99c43633-c691-4dee-b7af-35bc6e74dd67  rack1
#
```

### 在 Amazon EC2 上运行 Cassandra

许多组织和个人在公有云上运行 Cassandra 集群。你可以在 Microsoft Azure、Google Cloud 和 Amazon Web Services (AWS) 上运行 Cassandra。在本节中，我将指导你在 AWS 上创建一个 Cassandra 集群。

在 Amazon EC2 上安装 Cassandra 时，你需要使用受支持平台（如 Ubuntu 16.04 LTS）的 API 来创建实例，并确保从可信来源获取 AMI（Amazon 系统映像）。一旦你下载了 AMI 并使服务器运行起来，Cassandra 的安装过程就与第 2 章类似。

### 使用可信的 AMI

AMI 是一个虚拟设备，用于在 Amazon Elastic Compute Cloud（“EC2”）中创建虚拟机。AMI 是一个机器模板，你可以使用它在 AWS 云中创建新服务器。

你必须仅使用来自可信来源的 AMI，例如以下这些：

*   `Ubuntu Amazon EC2 AMI 定位器`
*   `Debian Amazon EC2 映像`
*   `Amazon EC2 云上的 CentOS-6 映像`

使用不可信来源的 AMI 会带来安全风险；由于它们配置 EC2 安装的方式，其性能也会较慢。

### 为 Cassandra 设置 AWS 实例

在 AWS 上进行任何操作之前，你必须拥有一个账户。因此，如果你还没有账户，请现在创建一个。

一旦你准备好了 AWS 账户，请按照后面几个部分中显示的步骤，在 AWS EC2 虚拟机（称为 EC2 实例）上创建运行 Cassandra 的集群。

#### 启动 EC2 实例创建

在 AWS 控制面板中，点击计算部分下的 EC2 徽标。

#### 选择 AWS 区域

为启动实例选择一个合适的区域。例如，如果你在美国，你可能希望选择北弗吉尼亚区域。

#### 创建 EC2 实例

在“创建实例”部分，你会看到一个“启动实例”按钮。点击它以启动“启动实例”向导。

这是创建 EC2 虚拟机的关键步骤。你可以通过不同方法为机器选择操作系统，例如下载 Amazon Machine Image (AMI)。如果你有自己的“黄金镜像”，甚至可以提供它。为了简化，这里我们将在 Ubuntu 服务器上运行集群，因此请选择“快速启动”菜单。

快速启动菜单包含创建 EC2 实例需要完成的六个步骤。

1.  选择 Amazon Machine Image (AMI)：选择 Ubuntu 服务器 (Ubuntu 14.04 LTS)。
2.  选择实例类型：使用本地 SSD，而不是 EBS 存储。这是一个测试集群，因此选择 `m3` 大小。`m3` 大小不属于免费套餐，所以会产生少量费用。免费套餐的实例对于深入学习 Cassandra 来说配置过低。你的实例类型选择是 `m3.large`，这是一台拥有 2 个核心、7.5GB RAM 和 32GB SSD 存储的机器。
3.  配置实例详细信息：此步骤用于指定 EC2 实例的数量。配置一个三节点集群，因此在“实例数量”属性的值中填入 3，其余属性（关机行为等）保持默认设置。
4.  添加存储：此时你不需要任何额外存储，因此直接进入下一步。
5.  标记实例：用键值对标记实例有助于轻松排序和查找实例，但你可以跳过此步骤，因为你只有三个 EC2 节点。
6.  配置安全组：此步骤允许你配置一个安全组，即一组用于控制进出实例流量的防火墙规则。在此步骤中，执行以下操作：
    *   选择“创建新的安全组”选项。
    *   将安全组命名为 `MySecurityGroup`，并添加四条入站规则，如下所示：

        ```
        类型              协议     端口范围        来源
        SSH              TCP      22             0.0.0.0/0 (允许来自任何地方)
        自定义 TCP 规则   TCP      7000-7001      0.0.0.0/0
        自定义 TCP 规则   TCP      7199           0.0.0.0/0
        自定义 TCP 规则   TCP      9042           0.0.0.0/0
        自定义 TCP 规则   TCP      9160           0.0.0.0/0
        ```

        这些端口与本章前面“配置端口”部分描述的端口相同。
7.  在实例创建的最后一步“审核实例启动”中，检查你的实例选择，然后按“启动”按钮，以便 AWS 为你创建实例。
8.  一旦 AWS 启动实例，它会要求你选择在登录新实例时要输入的密钥对：

    ```
    选择现有密钥对或创建新密钥对
    ```

    密钥对是 AWS 存储的公钥和你存储的私钥文件的组合。你使用密钥对安全地登录到你的实例。在新的 Ubuntu 服务器上，你的私钥文件使你能够通过 SSH 安全地登录到实例。

**提示**

安全地存储你的私钥文件（`.pem` 文件），因为丢失它意味着你需要终止所有实例并从头开始。

通过提供密钥对名称 `mykeypair` 创建一个新的密钥对，然后单击“下载密钥对”以下载私钥文件。此时，所有 EC2 实例都在运行，并且所有使用量的计费立即开始。在你完成与 Cassandra 的工作后关闭实例是个好主意，这样在你没有使用测试集群时就不会产生额外费用！

你现在可以通过点击 EC2 控制台中的“实例”选项卡来查看实例。所有三个实例的“实例状态”列都将显示“正在运行”。通过选择三个实例中的任何一个，你可以获取该实例的描述，包括其公有 IP 地址。然后，你可以使用该实例的 IP 地址启动一个 Putty 会话。

### 安装 Cassandra

现在你的 AWS EC2 实例已经运行，是时候安装 Cassandra 了。请按照以下步骤在 AWS EC2 实例上安装、配置和启动 Cassandra。

你在第 2 章中学到，你可以将 Cassandra 安装为服务，也可以从二进制 tar 包安装。既然你已经学习了如何从 tar 包安装，那么学习如何将 Cassandra 安装为服务是个好主意。以下是安装 Cassandra 为服务必须遵循的步骤。

1.  你可以从 Apache 或 DataStax 获取 Cassandra Debian 软件包。这里使用 DataStax 软件仓库，操作如下：

    ```
    $ echo "deb http://debian.datastax.com/community stable main"| sudo tee –a /etc/apt/sources.list.d/cassandra.sources.list
    ```

2.  运行 `apt-get update` 命令。

    ```
    $ sudo apt-get update
    ```

3.  如果你收到任何关于没有 DataStax 软件仓库公钥的错误，你需要添加 DataStax 公共仓库密钥，如下所示，然后重新运行 `apt-get update` 命令。

    ```
    $ curl –L http://debian.datastax.com/debian/repo_key | sudo apt-key add-
    $ sudo apt-get update
    ```

4.  安装 Cassandra 二进制文件。

    ```
    $ sudo apt-get install cassandra
    ```

    与第 2 章中展示的二进制 tar 包安装方法（针对单实例）不同，将 Cassandra 安装为服务会自动启动 Cassandra 实例。如果你现在执行命令 `sudo service cassandra status`，它将显示 cassandra 正在此节点上运行。
5.  下一步是在其余的 EC2 实例上重复之前的三个步骤。一旦完成，你就可以运行 `nodetool status` 命令来检查 Cassandra 实例的状态。

    ```
    $ sudo nodetool status
    Datacenter: datacenter1
    =======================
    Status=Up/Down
    |/ State=Normal/Leaving/Joining/Moving
    --  Address   Load   Tokens    Owns    Host ID     Rack
    UN  192.168.177.132  123.84 MiB  256          ?b0ade950-937a-457c-95eb-d3032897eeb1  rack1
    Note: Non-system keyspaces don't have the same replication settings, effective ownership information is meaningless
    $
    ```

三个 Cassandra 节点之间尚未进行通信。接下来让我们启用节点间通信。


### 配置 Cassandra 集群

在全部三台 EC2 实例上，编辑 `cassandra.yaml` 文件并添加以下属性：

```
cluster_name: 'My AWS Cluster'
seeds: "192.168.177.132"
broadcast_address: 192.168.177.132
listen_address:
```

这些属性与我之前创建六节点 Cassandra 集群时解释的相同。与那个集群的情况一样，我决定不为 `listen_address` 属性指定值，因此我必须为 `broadcast_address` 属性指定一个值。

**注意**

用户常常看到 EC2 专用的 snitch 并使用它，因为它是为 AWS 集群设计的。然而，推荐使用的 snitch 是 `GossipingPropertyFileSnitch`。

一旦你编辑了 `cassandra.yaml` 文件，就在所有三个节点上重启 Cassandra 服务，并确保清除所有系统数据：

```
$ sudo service cassandra stop
$ sudo rm –rf /var/lib/Casandra/data/system/*
$ sudo service cassandra start
```

运行 `nodetool status` 命令显示所有三个节点现在都在运行，新的三节点 AWS EC2 Cassandra 集群已准备就绪。

```
INFO  22:20:27 Handshaking version with /192.168.177.135
INFO  22:20:27 Node /192.168.177.135 has restarted, now UP
INFO  22:20:27 InetAddress /192.168.177.135 is now UP
INFO  22:20:27 Node /192.168.177.135 state jump to NORMAL
INFO  22:20:27 Node /192.168.177.132 state jump to NORMAL
INFO  22:20:27 Updating topology for /192.168.177.135
INFO  22:20:27 Updating topology for /192.168.177.135
INFO  22:20:27 Waiting for gossip to settle before accepting client requests...
WARN  22:20:27 Not marking nodes down due to local pause of 11051514701 > 5000000000
INFO  22:20:35 No gossip backlog; proceeding
```

一旦集群启动并运行，所有功能都与非 AWS 的 Cassandra 集群相同。

## 总结

成功的集群安装取决于先决条件的满足。无论你创建的是本地集群还是云端集群，你都可以使用一组最基础的配置属性启动集群。与上一章的单节点安装一样，一旦你学会了如何启动和停止集群，你就可以在后续章节中学习时配置额外的属性。Cassandra 带有大量配置项，接下来的每一章都会介绍几个新的配置属性。

