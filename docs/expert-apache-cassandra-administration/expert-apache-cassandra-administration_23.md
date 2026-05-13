# sudo docker exec -i -t cassandra1 sh -c 'nodetool status'
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID      Rack
UN  172.17.0.3  101.89 KB  256          67.3%             968690b8-8dc4-4242-943f-b62d37e44084  rack1
UN  172.17.0.2  107.45 KB  256          68.7%             f217c613-3eb9-445a-884a-183e8d177927  rack1
UN  172.17.0.4  15.42 KB   256          64.0%             820f24f7-d931-4a9f-87e3-8abd72d8a9bb  rack1
#
```

请注意，您无法在主 Linux 服务器中运行 Cassandra 实用程序 `nodetool`、`cqlsh` 或任何其他此类命令。由于 Cassandra 运行在 Docker 容器内部，您只能在 Docker 中运行这些命令。您必须使用 `docker exec` 命令在 Docker 容器内运行命令。

#### 在基于 Docker 的集群中运行 cqlsh

要从基于 Docker 的集群中的三个节点之一运行 `cqlsh`，您需要“bash 进入”其中一个节点，例如节点 `cassandra1`，如下所示：

```bash
$ sudo docker exec -it cassandra1 /bin/bash
samalapati@34842881b8d4:/# cqlsh
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.0.4 | CQL spec 3.4.0 | Native protocol v4]
Use HELP for help.
cqlsh>
```

`docker exec` 命令使您能够在 Docker 容器内运行命令。前面的命令在容器 `cassandra1` 内为您提供了一个 bash shell。

一旦您通过 bash shell 登录到节点，就可以照常运行 `cqlsh` 命令以进入 `cqlsh` 提示符。

或者，您可以指定 Docker 的 `--link` 选项来连接到 Docker 实例，并在 `exec cqlsh <IPAddress>` 选项的帮助下登录 `cqlsh`。

```bash
