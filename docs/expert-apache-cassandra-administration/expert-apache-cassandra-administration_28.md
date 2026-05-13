# cucumber features/cassandra.feature
Feature: Cassandra Cluster
As a Big Data Administrator
I want to deploy a cassandra cluster
So I can store and query lots of data
cassandra uses an image, skipping
cassandra-dev uses an image, skipping
Pulling cassandra (cassandra:latest)...
latest: Pulling from library/cassandra
ockerbdd_cassandra_1 is up-to-date
dockerbdd_cassandra-dev_1 is up-to-date
Scenario: Launch a CQL Shell                                       # features/cassandra.feature:6
Given the services are running                                   # features/step_definitions/docker_compose_steps.rb:3
Running: docker-compose build && (docker-compose up -d || true)
Process exited successfully
[cqlsh 5.0.1 | Cassandra 3.11.0 | CQL spec 3.4.4 | Native protocol v4]
And I run "cqlsh cassandra -e 'show version'" on "cassandra-dev" # features/step_definitions/docker_compose_steps.rb:14
Running: docker-compose run cassandra-dev bash -i -c "sleep 1; cqlsh cassandra -e 'show version'"
Process exited successfully
Then I should see "CQL spec"                                     # features/cassandra.feature:9
And I should see "Cassandra 3.7"                                 # features/cassandra.feature:10
1 scenario (1 undefined)
4 steps (2 undefined, 2 passed)
0m8.169s
#
```

Docker-Compose 为你创建了单节点的 Cassandra 集群，而 BDD 则在此集群上运行你指定的测试。

当你首次运行此处展示的 `cucumber` 命令时，基于你指定的场景（`"Given the services are running"`），Docker-Compose 会构建你在 `docker-compose.yml` 文件中指定的所有 Docker 镜像。在后续的运行中，`docker-compose` 会识别出 `cassandra` 和 `cassandra-dev` 实例都使用了可用的镜像，因此不再构建镜像，而是直接启动这两个 Cassandra 实例。

你可以使用 `docker ps` 命令来验证这两个 Docker 容器正在运行。

```
$ sudo docker ps
CONTAINER ID    IMAGE    COMMAND      CREATED  STATUS     PORTS      NAMES
b005b61706ca    cassandra   "/docker-entrypoint.s"   44 minutes ago   Up 40 minutes   7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcpdockerbdd_cassandra-dev_1
ed526bd724c7        cassandra           "/docker-entrypoint.s"   44 minutes ago      Up 40 minutes       7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcpdockerbdd_cassandra_1
$
```

然后，你可以像这样通过 bash 进入一个容器（`dockerbdd_cassandra_dev1`）：

```
$ sudo docker exec -it b005b61706ca bash
/# hostname
b005b61706ca
/#
```

最后，你可以在新的 Docker 容器中使用 CQL shell，如下所示：

```
$ cqlsh
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.0 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh>
```

你可以从 Coshxlab 的 Ben Taitelbaum 撰写的一篇文章中获取更多关于 BDD 驱动的 Cassandra 集群的详细信息，包括创建多节点 Cassandra 集群的细节，文章位于以下位置：

[`www.coshx.com/blog/2016/08/01/cassandra-cluster-in-docker-using-bdd/`](http://www.coshx.com/blog/2016/08/01/cassandra-cluster-in-docker-using-bdd/)

## 使用 Cassandra 集群管理器快速启动集群

Cassandra 集群管理器 (CCM) 是一个脚本库，可帮助你在本地服务器上创建和运行 Cassandra 集群。CCM 不适用于生产环境或对集群进行压力测试；它是一种测试和学习如何运行 Cassandra 集群的方法。CCM 是 Docker 和 Vagrant 之外另一种快速启动 Cassandra 集群的方式。

你可以在以下情况下使用 CCM：

*   当你想运行 Cassandra 而不想费心安装它时
*   用于测试故障场景
*   当开发涉及多个版本而无需担心硬件时，因为你可以在本地节点上运行所有内容
*   用于集成测试
*   用于测试升级

CCM 的惊人之处在于它能让你多快地启动并运行一个新集群。你可以在不到一分钟的时间内启动一个多节点（本地）集群。



### 安装 CCM

安装 CCM 很简单：只需确保你在开始安装 CCM 之前已安装 Python（至少是 2.7 版本）和 `ant`。

1.  要安装 CCM，你需要通过运行以下命令从 GitHub 获取所需文件：

    ```
    root@ubuntu2:/home/samalapati# git clone https://github.com/pcmanus/ccm.git
    Cloning into 'ccm'...
    remote: Counting objects: 4372, done.
    remote: Compressing objects: 100% (15/15), done.
    remote: Total 4372 (delta 8), reused 14 (delta 6), pack-reused 4351
    Receiving objects: 100% (4372/4372), 1.80 MiB | 0 bytes/s, done.
    Resolving deltas: 100% (3040/3040), done.
    Checking connectivity... done.
    root@ubuntu2:/home/samalapati# ls
    ```

2.  一旦你从 GitHub 下载了脚本和其他文件，运行 CCM 安装脚本。

    ```
    $ sudo ./setup.py install
    ```

安装文件位于 `ccm` 目录下。

```
$ ls
ccm      Documents  examples.desktop  Pictures  Templates
Desktop  Downloads  Music             Public    Videos
$ cd ccm
$ ls
ccm  ccmlib  license.txt  MANIFEST.in  misc  README.md  setup.py  ssl  tests
$ sudo ./setup.py install
running install
running build
running build_py
creating build
...
creating build/scripts-2.7
...
running install_egg_info
$
```

你可以通过输入 `ccm`、`ccm -help` 或 `ccm <command> -h.` 来查看 CCM 提供的各种功能。当然，你能做的第一件也是最重要的事情是创建一个集群，接下来我将介绍这个任务。

### 使用 CCM 创建 Cassandra 集群

按照上一节所示安装好 CCM 后，就该创建一个 Cassandra 集群了。让我们创建一个包含三个节点、运行 Cassandra 3.1 版本的集群。默认情况下，CCM 创建的是单令牌节点，因此需要告诉它使用虚拟节点。

CCM 会下载其所需的所有源文件并为你编译它们。

```
$ sudo ccm create -n 3 -v 3.1 testcluster --vnodes
18:55:53,875 ccm INFO Downloading http://archive.apache.org/dist/cassandra/3.1/apache-cassandra-3.1-bin.tar.gz to /tmp/ccm-tlSojg.tar.gz (29.376MB)
306318:55:59,731 ccm INFO Extracting /tmp/ccm-tlSojg.tar.gz as version 3.1 ...
30802666  [100.00%]Current cluster is now: testclusster
$
```

命令 `ccm list` 会显示此节点上由 CCM 管理的所有 Cassandra 集群。

```
$ sudo ccm list
*testcluster
$
```

#### 检查集群状态

你可以通过以下方式检查新三节点集群的状态（注意在 CCM 环境中不能使用 `nodetool` 命令！）：

```
$ sudo ccm status
Cluster: 'testcluster'

node1: DOWN (Not initialized)
node3: DOWN (Not initialized)
node2: DOWN (Not initialized)
$
```

新集群中的所有三个节点都处于 `DOWN` 状态，因为你刚刚创建了集群，但尚未启动它。

#### 启动集群

执行 `ccm start` 命令来启动 CCM 集群。在此示例中，我启动集群并使用 `ccm status` 命令检查其状态。

```
$ sudo ccm start
$ sudo ccm status
Cluster: 'testcluster'

node1: UP
node3: UP
node2: UP
$
```

命令 `ccm node1 status` 提供了关于集群状态的更详细信息。

```
$ sudo ccm node1 status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens       Owns    Host ID              Rack
UN  127.0.0.1  86.51 KB   256          ?       659b06e0-a4d9-49f5-b923-0bab9833daa3  rack1
UN  127.0.0.2  86.47 KB   256          ?       4035e764-a152-40e6-ac9e-f9ddf1e603b4  rack1
UN  127.0.0.3  92.34 KB   256          ?       40182f69-a81f-4a23-90ae-e9e1f0929c5e  rack1
Note: Non-system keyspaces don't have the same replication settings, effective ownership information is meaningless
$
```

你可以通过运行命令 `ccm stop` 来停止集群节点。

### 使用 CCM

在本节中，我将通过展示如何运行一些重要命令来简要介绍 CCM，其中第一条命令允许你访问 CQL shell。

#### 在 CCM 中使用 cqlsh

要使用 CQL，运行带有 `cqlsh` 选项的 `ccm` 命令，并确保指定一个节点。

```
$ sudo  ccm node1 cqlsh
Connected to testcluster2 at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.0 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh>
```

#### 获取 SSTable 信息

`ccm node1 getsstables` 命令显示此节点上的所有 SSTable。

```
$ sudo ccm node1 getsstables
/samalapati/.ccm/testcluster/node1/data0/system_schema/triggers-4df70b666b05325195a132b54005fd48/ma-1-big-Data.db
/samalapati/.ccm/testcluster/node1/data0/system_schema/types-5a8b1ca866023f77a0459273d308917a/ma-1-big-Data.db
...
$
```

### 运行 Apache Spark 与 Cassandra

Apache Spark 是一个非常流行、快速的通用大规模数据处理引擎。Spark 可以运行在 Apache Hadoop、Apache Mesos、云端或独立模式下。Spark 可以访问多种数据源，例如 Hadoop 的 HDFS、Hbase、AWS 的 S3，当然还有 Apache Cassandra。

在本节中，我将展示如何安装 Apache Spark，并通过 DataStax 的 Spark-Cassandra 连接器从 Spark 集群操作 Cassandra 数据库。

### 安装先决条件

要安装 Apache Spark，有几个先决条件，我将在本节中进行回顾。确保在你将要运行 Spark 集群的服务器上已经安装了以下软件，如果缺失，请安装它们。

1.  确保至少安装了 Java 1.8.x。

    ```
    # java -version
    java version "1.8.0_131"
    Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
    Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, mixed mode)
    #
    ```

2.  你还需要 Scala。

    ```
    # scala -version
    Scala code runner version 2.11.7 -- Copyright 2002-2013, LAMP/EPFL
    #
    ```

    你可以通过以下方式安装 Scala：

    ```
    # wget www.scala-lang.org/files/archive/scala-2.11.8.deb
    # sudo dpkg -i scala-2.11.8.deb
    ```

3.  最后，你需要 SBT（Simple Build Tool）来编译 Scala 程序。

    ```
    # echo "deb https://dl.bintray.com/sbt/debian /" | sudo tee -a /etc/apt/sources.list.d/sbt.list
    # sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2EE0EA64E40A89B84B2DF73499E82A75642AC823
    # sudo apt-get update
    # sudo apt-get install sbt
    ```

    检查 SBT 版本：

    ```
    # sbt sbt-version
    ...
    [info] 0.13.15
    #
    ```

拥有正确的 SBT 版本非常重要；不兼容的版本是使用 Spark 和 Cassandra 时许多问题的根源。

现在你已准备好安装 Apache Spark。

### 安装 Apache Spark

为你当前的目的（学习如何从 Spark 集群连接到 Cassandra 集群）安装并开始使用 Apache Spark 非常直接。请按照以下步骤安装 Spark。

1.  `$ wget` [`http://d3kbcqa49mib13.cloudfront.net/spark-2.0.2-bin-hadoop2.7.tgz`](http://d3kbcqa49mib13.cloudfront.net/spark-2.0.2-bin-hadoop2.7.tgz)
2.  `$ tar zxf spark-2.0.2-bin-hadoop2.7.tgz`
3.  `$ sudo mv spark-2.0.2-bin-hadoop2.7 /usr/local/spark/`
4.  安装 Spark 二进制文件后，为 Spark 设置一个主目录。

    ```
    export SPARK_HOME=/usr/local/spark
    ```

5.  导出 PATH 变量。

    ```
    export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:$SPARK_HOME/bin"
    ```


### 配置 Spark 集群

你可以通过 YARN 集群管理器 (Hadoop)、Apache Mesos 或 Spark 独立集群来运行 Spark 集群。让我们尝试最后一种方案，这样你可以同时运行 Spark 和 Cassandra 集群。

Spark 由一个主节点和多个工作节点组成。为了简单起见，让我们将主服务和工作服务运行在同一个节点上，但也可以轻松配置将工作节点运行在不同的节点上。

主要的配置文件是 `spark-env` 文件，位于 `/etc/spark/conf` 目录中。在此文件中，你可以指定 Spark 独立部署模式下使用的守护进程选项。也就是说，你为主节点和工作节点指定选项。

你可以将 `SPARK_WORKER_CORES` 和 `SPARK_WORKER_MEMORY` 等所有配置属性保留为其默认值。只需确保为你指定 Spark 主实例将运行的节点的详细信息。

```bash
export SPARK_MASTER_HOST=192.168.159.129
```

这样，主节点将运行在 IP 为 192.168.159.129 的服务器上。你必须将此属性添加到集群中每个节点的 `/etc/spark/conf/spark-env` 文件中。由于你将主实例和工作实例运行在同一节点上，所以这没问题。

在生产集群中，你可以将 Spark 配置为服务运行，但这里我只关注演示如何从 Spark 连接到 Cassandra，因此我们不会费心创建 Spark 服务。相反，让我们手动启动和停止 Spark 集群。

### 启动 Spark 集群

由于 Spark 集群只有一个工作实例与主实例运行在同一节点上，你需要执行以下步骤来启动 Spark 集群。

1.  移动到 Spark 的 `sbin` 目录，该目录包含集群的启动/停止脚本。

    ```bash
    # cd /usr/local/spark/spark-2.0.2-bin-hadoop2.7/sbin
    ```

2.  启动 Spark 主实例。

    ```bash
    # ./start-master.sh
    starting org.apache.spark.deploy.master.Master, logging to /usr/local/spark/spark-2.0.2-bin-hadoop2.7/logs/spark-root-org.apache.spark.deploy.master.Master-1-ubuntu.out
    #
    ```

3.  启动单个工作实例。

    ```bash
    # ./start-slave.sh spark://ubuntu:7077
    starting org.apache.spark.deploy.worker.Worker, logging to /usr/local/spark/spark-2.0.2-bin-hadoop2.7/logs/spark-root-org.apache.spark.deploy.worker.Worker-1-ubuntu.out
    #
    ```

4.  确保主实例和工作实例正在运行。

    ```bash
    # ps -ef|grep spark
    root      14941   2008  0 10:37 pts/1    00:00:04 /usr/lib/jvm/java-8-oracle/jre/bin/java -cp /usr/local/spark/spark-2.0.2-bin-hadoop2.7/conf/:/usr/local/spark/spark-2.0.2-bin-hadoop2.7/jars/* -Xmx1g org.apache.spark.deploy.master.Master --host ubuntu --port 7077 --webui-port 8080
    root      15073   2008  1 10:39 pts/1    00:00:04 /usr/lib/jvm/java-8-oracle/jre/bin/java -cp /usr/local/spark/spark-2.0.2-bin-hadoop2.7/conf/:/usr/local/spark/spark-2.0.2-bin-hadoop2.7/jars/* -Xmx1g org.apache.spark.deploy.worker.Worker --webui-port 8081 spark://ubuntu:7077
    #
    ```

到目前为止一切正常，Spark 集群已启动并运行。在下一节中，我将展示如何从 Spark 集群连接到 Cassandra 数据库。

### 从 Spark 集群连接到 Cassandra

在 Spark 中访问 Cassandra 表的一种方法是将它们转换为 Spark RDD（弹性分布式数据集）。你可以将 Cassandra 的表数据视为 Spark RDD，也可以将 Spark RDD 写入 Cassandra 表。

你可以通过多种方式从 Spark 访问 Cassandra。以下项目提供了库，使你能够从 Spark 程序读取和写入 Cassandra 表：

*   [`https://github.com/datastax/spark-cassandra-connector`](https://github.com/datastax/spark-cassandra-connector)
*   [`http://tuplejump.github.io/calliope/pyspark.html`](http://tuplejump.github.io/calliope/pyspark.html)
*   [`https://github.com/TargetHolding/pyspark-cassandra`](https://github.com/TargetHolding/pyspark-cassandra)

让我们使用这些项目中的第一个，即 spark-cassandra-connector。

spark-cassandra-connector（DataStax Cassandra 连接器）是由 DataStax 提供的一个 Scala 库，它使得可以同时使用 Spark 和 Cassandra。你可以从 [`https://github.com/datastax/spark-cassandra-connector`](https://github.com/datastax/spark-cassandra-connector) 获取此连接器的最新版本。

当你编写代码从 Spark 访问 Cassandra 时，你的代码，连同 spark-cassandra-connector 及其所有必要的依赖项，会被打包并发送到 Spark 集群的节点。

你通过 `spark-submit` 接口向 Spark 集群提交作业。当你这样做时，你可以在命令行包含一个肥 jar 以包含 spark-cassandra 连接器。

这里的示例将展示如何从 Spark 集群执行临时工作。因此，请使用 `spark-shell` 而不是 `spark-submit` 来与 Spark（和 Cassandra）交互。

在以下示例中，你启动 `spark-shell` 并调用 spark-cassandra-connector，以便可以从 Spark 集群与 Cassandra 通信。`–packages` 选项使你能够指定 `spark-cassandra-connector`。此选项从 Spark 包下载连接器及其依赖项。

请注意，你需要从 `$SPARK_HOME/bin` 目录运行 `spark-shell` 命令，而不是从启动 Spark 实例的 `$SPARK_HOME/sbin` 目录运行。

```bash
bin# ./spark-shell --conf spark.cassandra.connection.host=192.168.159.129 --packages datastax:spark-cassandra-connector:2.0.0-M2-s_2.11
...
org.scala-lang#scala-reflect;2.11.8 from central in [default]

|                  |            modules            ||   artifacts   |
|       conf       | number| search|dwnlded|evicted|| number|dwnlded|

|      default     |   7   |   1   |   1   |   0   ||   7   |   0   |

Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
Setting default log level to "WARN".Spark context Web UI available at http://192.168.159.129:4040
Spark context available as 'sc' (master = local[*], app id = local-1499634822206).
Spark session available as 'spark'.
Welcome to
____              __
/ __/__  ___ _____/ /__
_\ \/ _ \/ _ '/ __/  '_/
/___/ .__/\_,_/_/ /_/\_\   version 2.0.2
/_/
Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_131)
Type in expressions to have them evaluated.
Type :help for more information.
scala>
```

你可以使用 Scala、Python（或 Java）编写 Spark 代码。在这种情况下，你通过执行 `spark-shell` 命令选择了 Scala。要用 Python 编写代码，请运行 `pyspark` 命令，如下所示：

```bash
root@ubuntu:/usr/local/spark/spark-2.0.2-bin-hadoop2.7/bin# ./pyspark \
> --master spark://ubuntu:7077 \
> --packages datastax:spark-cassandra-connector:2.0.0-M2-s_2.11 \
> --conf spark.cassandra.connection.host=ubuntu
Python 2.7.12 (default, Nov 19 2016, 06:48:10)
...
Welcome to
/ __/__  ___ _____/ /__
_\ \/ _ \/ _ '/ __/  '_/
/__ / .__/\_,_/_/ /_/\_\   version 2.0.2
/_/
Using Python version 2.7.12 (default, Nov 19 2016 06:48:10)
SparkSession available as 'spark'.
>>> posts = spark.read.format("org.apache.spark.sql.cassandra").options(table="posts", keyspace="posts_db").load()
```


# 8. 备份、恢复与数据迁移

### 从 Spark 操作 Cassandra

在运行命令访问 Cassandra 数据库之前，你必须首先导入 Spark 连接器命名空间，如下所示：

```scala
scala> import com.datastax.spark.connector._
import com.datastax.spark.connector._
scala>
```

让我们操作来自 `cyclist` 键空间的 `cyclist_name` 表。首先要做的是为该表数据创建一个 Spark RDD。

```scala
scala> val rdd1 = sc.cassandraTable("cycling", "cyclist_name")
rdd: com.datastax.spark.connector.rdd.CassandraTableScanRDD[com.datastax.spark.connector.CassandraRow] = CassandraTableScanRDD[2] at RDD at CassandraRDD.scala:18
scala>
```

在命令中，`sc` 代表 Spark Context。当你调用 `spark-shell` 时，Spark 会自动创建一个名为 `sc` 的 Spark Context。Spark Context 是所有 Spark 功能的主要入口点，你必须按照我此处展示的方式来使用它。

Spark Context 使你能够连接到 Spark 集群，你可以用它在 Spark 集群中创建 Spark RDD、数据框、数据集等。在某种程度上，Spark Context 是 Spark 集群的协调者（驱动程序）。`SparkContext` 对象协调着构成 Spark 应用程序的集群中独立的进程集。Spark Context 连接到集群管理器，该管理器可以是 Mesos、YARN 或独立集群管理器（如前所述，我使用的是 Spark 独立集群）。

接下来，让我们通过运行以下代码来测试 Spark 集群到 Cassandra 数据库的连接，该代码使用 `first` 函数获取表 `CYCLIST_NAME` 中的第一行：

```scala
scala> println(rdd.first)
CassandraRow{id: e7ae5cf3-d358-4d99-b900-85902fda9bb0, firstname: Alex, lastname: FRAME}
scala>
```

以下命令使用 `foreach` 结构打印表 `CYCLIST_NAME` 中的所有行：

```scala
scala> rdd.foreach(println)
CassandraRow{id: e7ae5cf3-d358-4d99-b900-85902fda9bb0, firstname: Alex, lastname: FRAME}
CassandraRow{id: 5b6962dd-3f90-4c93-8f61-eabfa4a803e2, firstname: Marianne, lastname: VOS}
CassandraRow{id: e7cd5752-bc0d-4157-a80f-7523add8dbcd, firstname: Anna, lastname: VAN DER BREGGEN}
scala>
```

既然你已经成功测试了从 Cassandra 表检索数据，那么让我们从 Spark 向 Cassandra 表插入一些数据。你首先创建一个名为 `collection` 的 RDD。

```scala
scala> val collection = sc.parallelize(Seq(("cat", 30), ("fox", 40)))
collection: org.apache.spark.rdd.RDD[(String, Int)] = ParallelCollectionRDD[4] at parallelize at :27
scala>
```

接着，你执行 Spark 的 `saveToCassandra` 函数，将两行新数据插入到键空间 `test` 的表 `kv` 中。

```scala
scala> collection.saveToCassandra("test", "kv", SomeColumns("key", "value"))
scala>
```

你通过查询该表来确认这两行数据已插入表 `test.kv`。

```cql
cqlsh> select * from test.kv;
key    | value
--------+-------
cat |    30
fox |    40
...
cqlsh>
```

好了！多亏了 spark-cassandra-connector，你现在可以在 Spark 集群内操作 Cassandra 表数据了。

利用 Cassandra Spark 集成，你可以做很多事情。你可以将 Spark 的计算能力与 Cassandra 的存储处理能力相结合，做出很酷的事情。然而，所有这些都属于另一本书的领域，因此我们继续前进。

## Cassandra 与 SMACK 技术栈：一个日益增长的趋势

虽然大数据项目最初主要面向批处理，但当前的趋势是转向敏捷系统，这些系统处理流数据以支持分析系统。组织越来越关注实时或近实时的用例，如推荐引擎和欺诈检测。

SMACK（一个使用开源技术 Apache Spark、Apache Mesos、Akka、Apache Cassandra 和 Apache Kafka 的系统）或其此类开源技术的变体，在流数据分析和挖掘中正日益成为关键参与者。像 SMACK 这样的技术正在成为构建利用“实时”流数据的应用程序的平台，使组织能够有效地从流数据集中提取有意义的信息。

本书致力于阐述面向 Cassandra 管理员的概念和技术，因此，我无法深入探讨 SMACK 技术栈所有组件的细节。但是，我想向你介绍这组关键技术，因为如果你正在使用 Cassandra，你最终很可能会接触到它。

以下是 SMACK 技术栈五个组件的作用：

*   Apache Spark（处理组件）：Spark 是一个用于分布式、大规模数据集的处理引擎，特别适用于机器学习算法。你可以使用相同的接口进行 SQL 查询、机器学习和流数据处理。
*   Apache Mesos（容器组件）：Mesos 是一个集群资源管理系统，为分布式应用程序提供资源隔离和共享。Mesos 使用两级调度机制，它向框架（即运行在 Mesos 之上的应用程序）提供资源。Mesos 决定每个框架获得的资源，而框架决定它们接受哪些资源以及在分配给它们的资源上执行哪些应用程序。
*   Akka（提供模型）：Akka 是一个用于构建高并发和弹性消息驱动应用程序的工具包和运行时。“Actor”是并发计算的通用原语，它响应收到的消息。作为对消息的响应，Actor 可以做出本地决策、创建其他 Actor、发送更多消息等等。
*   Apache Cassandra：到现在，你应该知道 Cassandra 是什么以及它擅长什么了！
*   Apache Kafka（代理）：Kafka 是一个流行的高吞吐量、低延迟消息系统，用于处理实时数据流。Kafka 对数据流进行分区，并将其分布在集群中，这允许多个协调的“消费者”消费由“生产者”生成的数据。

## 总结

本章旨在展示一些有趣的方法，让你可以使用不同的技术设置测试或生产环境。Docker 容器很流行，你可以在 Docker 上运行生产 Cassandra 集群。在 Docker 上运行生产集群时，你需要处理存储和网络策略，我这里没有讨论这些。Docker 的文档相当好，你可能想利用它来了解更多关于这些以及其他 Docker 容器主题的知识。

Apache Spark 作为一种快速处理框架正变得越来越流行，我展示了如何从 Spark 连接到 Cassandra 数据库并执行 DML 操作。当然，这只是一个开始，如果你希望更深入地研究 Apache Spark，其源文档非常不错。

CCM 是一个很酷的工具，因为它能帮助你在短短几分钟内启动集群，而无需了解任何 Cassandra 安装程序，也无需花费时间设置任何先决条件来运行一个用于探索目的的简单测试集群。

# 8. 备份、恢复与数据迁移

备份和恢复数据是 Cassandra 管理员的关键职能。本章向你展示如何通过定期拍摄数据快照来备份数据。Cassandra 还允许你进行增量备份和备份提交日志段。所有这些备份在你需要执行数据库完整恢复时都派得上用场。

批量加载和复制数据也是一项常见任务，我解释了如何使用 `COPY` 命令以及 sstableloader 实用程序来加载数据。



### 备份数据

备份 Cassandra 数据库涉及备份数据库 `data` 目录下的所有 SSTable 文件。你可以在数据库运行时进行备份。你可以备份整个数据库，或仅备份一个键空间，甚至只备份一张表。

你创建的备份（快照）可能不一致，但这完全不是问题。进行备份的目的是，如果你因任何原因丢失了部分或全部数据，可以恢复数据库。当你恢复快照时，Cassandra 会利用你在前面章节中学到的一致性机制，使集群中所有节点的数据保持一致。

`nodetool snapshot` 命令使你能够备份 Cassandra 的数据。该工具会同时备份模式信息和 SSTables 中的数据。当你需要从快照恢复表时，你将需要数据以及模式。这样，当你需要重新创建节点的部分或全部 SSTables 时，你将同时拥有创建表所需的数据和元数据。

创建快照意味着磁盘使用量会增加，因为你将相同的数据存储在节点上的两个位置。`nodetool snapshot` 命令仅在单节点级别工作。你可以使用像 `pdsh` 这样的并行 Shell 工具来运行集群范围的快照，通过执行 `nodetool snapshot` 命令。

你可以使用以下命令备份节点上的所有键空间：

```
$ nodetool snapshot
Requested creating snapshot(s) for [all keyspaces] with snapshot name [1499019048455] and options {skipFlush=false}
Snapshot directory: 1499019048455
$
```

Cassandra 会创建快照，其中包含此节点上所有表的数据和模式，并将每张表的数据存储在该表对应的目录中。

```
$ CASSANDRA_HOME/data/data/keyspace_name/table_UID/snapshot_name
```

在这个例子中，是以下目录：

```
data/data/cycling/cyclist_name-351e8df05f4011e7b2672fabda59ff45/snapshots/1499019048455
```

你将在此目录中找到与表 `cyclist_name` 相关的所有文件：

```
