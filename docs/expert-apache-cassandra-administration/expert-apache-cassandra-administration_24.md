# docker run -it --link cassandra1 --rm cassandra:3.0.4 \
> sh -c 'exec cqlsh 172.17.0.2'
Connected to Test Cluster at 172.17.0.2:9042.
[cqlsh 5.0.1 | Cassandra 3.0.4 | CQL spec 3.4.0 | Native protocol v4]
Use HELP for help.
cqlsh>
```

### 为 Docker 设置 Cassandra 环境变量

当您在 Docker 上启动 Cassandra 镜像时，可以传递几个可用的环境变量。更具体地说，您在发出 `docker run` 命令时在命令行上传递这些环境变量。您在命令行上传递的每个环境变量都会设置节点 `cassandra.yaml` 文件中相关变量的值。

您在上一节中学习了如何指定 `CASSANDRA_SEEDS` 环境变量。除了该变量之外，还有其他几个。以下是最有用的几个：

*   `CASSANDRA_LISTEN_ADDRESS`：确定侦听传入连接的 IP 地址。如果您选择默认值 `auto`，它会将 `cassandra.yaml` 文件中的 `listen_address` 选项设置为容器的 IP 地址。
*   `CASSANDRA_BROADCAST_ADDRESS`：此变量指定节点向集群中其余节点通告的 IP 地址。这将设置 `cassandra.yaml` 文件中 `broadcast_address` 和 `broadcast_rpc_address` 属性的值。
*   `CASSANDRA_ENDPOINT_SNITCH`：此属性允许您通过设置 `cassandra.yaml` 文件中的 `endpoint_snitch` 属性来设置 snitch 实现。
*   `CASSANDRA_NUM_TOKENS`：此变量通过设置 `num_tokens` 属性来设置此节点的令牌数量。
*   `CASSANDRA_DC`：通过设置 `dc` 属性的值来设置此节点的数据中心的名称。如果省略此变量，数据中心的名称默认为 `datacenter1`。
*   `CASSANDRA_RACK`：此变量通过设置 `rack` 属性的值来设置机架的名称。如果省略它（就像我这里所做的），该变量默认为 `rack1`。



# 在 Docker 上存储 Cassandra 数据并使用 BDD 构建集群

## 在 Docker 上存储 Cassandra 数据

有不止一种方法来存储在 Docker 容器中运行的 Cassandra 所使用的数据。其中两种选项如下：

*   **使用 Docker 卷**：默认情况下，Docker 通过使用其内部卷管理机制将 Cassandra 数据文件写入主机的文件系统来管理数据存储。虽然容器可以轻松查看和访问此数据，但对于运行在 Docker 主机服务器上（而非容器内）的工具和应用程序来说，情况则不尽相同。它们可能无法轻松定位 Cassandra 的数据文件。此外，当你关闭容器，或者容器停止运行，或者 Docker 主机宕机时，数据将会丢失。
*   **在主机服务器上创建数据目录**：你也可以将主机目录挂载为数据卷。你将主机上的数据目录挂载到容器内可见的一个目录。这样，工具和应用程序就可以轻松访问主机系统中的文件。你只需确保在主机系统上正确设置目录权限和任何 ACL（访问控制列表，用于控制对目录的访问）等，以便用户可以访问这些目录。

以下是一个示例，展示了如何挂载一个带有 Docker 主机文件系统的卷：

```
$ docker run –name my_container -v /host/dir://container/dir cassandra
```

当你这样做时，容器进程写入 `/container/dir` 目录的所有内容都会直接写入主机文件系统上的 `/host/dir` 目录。

> **注意**：CassandraCloud 是 Cloudurable 提供的一个工具，它简化了 Cassandra 在各种环境（包括 EC2、Docker 和 VirtualBox）中的配置。详情请访问 [`https://cloudurable.github.io/cassandra-cloud/`](https://cloudurable.github.io/cassandra-cloud/).

### 使用 Docker-Compose 和行为驱动开发创建 Cassandra 集群

你可以通过使用行为驱动开发来快速搭建 Cassandra 的开发和测试环境。`BDD` 在许多方面与测试驱动开发（`TDD`）相似，但后者侧重于单元测试和集成测试，而 `BDD` 则侧重于系统与其用户之间的交互。

当你使用 `BDD` 时，你描述在各种场景下应该发生什么。例如，对于一个网站，你以用户点击和在表单中输入数据的形式描述工作流程。对于系统管理员来说，工作流程则由系统管理员与系统及程序交互时输入的命令组成。

在本节中，我将展示如何使用 `BDD` 以及一个名为 `Cucumber` 的工具，配合 `Docker Compose` 来开发运行 Cassandra 的新 Docker 容器。以下是 `BDD` 和 `Docker Compose` 如何帮助你：

*   `BDD` 以测试驱动的方式帮助开发新的 Docker 容器。
*   `Docker Compose` 帮助你编排 Docker 容器。

`BDD`（加上 `Cucumber`）和 `Docker-Compose` 共同构成了一种启动新集群的令人兴奋的方式。虽然你可以使用 `Vagrant`（一个便于为开发环境启动虚拟机的工具）做类似的事情，但我在这里描述的方法要简单得多，也更有趣。

### 准备前提条件

使用 `BDD` 设置 Cassandra 集群需要满足三个条件：

*   **Docker**：如前所述，Docker 是一个容器平台，可帮助你创建和管理容器。
*   **Docker Compose**：这是一个帮助你定义和运行多容器 Docker 应用程序的工具。你使用一个 Compose 文件来配置应用程序所需的服务。之后，你可以用一条命令创建并启动所有服务，如下所示：
    ```
    $ docker-compose up -d
    ```
*   **Cucumber**：`Cucumber` 是一个测试工具，通过运行用 `BDD` 风格编写的自动化验收测试来帮助测试其他软件。

#### 安装 Docker

你需要 Docker，如果你之前按照我的讨论运行过基于 Docker 的 Cassandra 集群，那么我假设你已经安装了。如果没有，你可以下载并安装 Docker。

#### 安装 Docker-Compose

你可以通过以下步骤安装 `Docker-Compose`（在 Ubuntu 系统上）：

```
