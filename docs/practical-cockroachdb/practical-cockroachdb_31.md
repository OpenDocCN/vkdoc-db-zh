# 第八章 测试

✓ http_req_failed................: 0.00% ✓ 0 ✗ 37

iterations.....................: 37 4.99938/s

vus............................: 1 min=1 max=1

vus_max........................: 1 min=1 max=1

##### 扩展你的测试环境

如果你在运行前面的测试时遇到问题，可能是因为你三节点的 CockroachDB 集群是通过单个“热点”节点访问的。为了扩展你的测试环境并引入负载均衡，我现在将创建一个位于 HAProxy 后面的本地 Docker 化集群，这个容器由 Cockroach Labs 的杰出人物 Tim Veil 提供。

创建一个名为 `docker-compose.yml` 的 Docker Compose 配置文件，内容如下。这个文件将：

- 创建三个 CockroachDB 节点，它们在启动时将尝试共同组成一个集群
- 创建一个 HAProxy 负载均衡器，它将把来自主机的请求转发到每个 CockroachDB 节点

```yaml
services:
  node1:
    image: cockroachdb/cockroach:v21.2.4
    hostname: node1
    container_name: node1
    command: start --insecure --join=node1,node2,node3
    networks:
      - app-network
  node2:
    image: cockroachdb/cockroach:v21.2.4
    hostname: node2
    container_name: node2
    command: start --insecure --join=node1,node2,node3
    networks:
      - app-network
  node3:
    image: cockroachdb/cockroach:v21.2.4
    hostname: node3
    container_name: node3
    command: start --insecure --join=node1,node2,node3
    networks:
      - app-network
  haproxy:
    hostname: haproxy
    image: timveil/dynamic-haproxy:latest
    ports:
      - "26257:26257"
      - "8080:8080"
      - "8081:8081"
    environment:
      - NODES=node1 node2 node3
    links:
      - node1
      - node2
      - node3
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

要启动容器、初始化集群并进入一个 CockroachDB 容器的 shell，请运行以下命令：

```
$ docker compose up -d
$ docker exec -it node1 cockroach init --insecure
$ docker exec -it node1 /bin/bash
```

现在你已经连接到容器了，运行以下命令连接到 CockroachDB 的 SQL shell：

```
$ cockroach sql --insecure
```

你现在可以像之前一样创建数据库及其对象，并通过 `localhost:26257` 访问集群中的所有节点，这个改动对于正在运行的 API 将是透明的。

#### 弹性测试

弹性测试旨在确保我们的数据库在节点故障时保持可用。我们将使用本节之前的 Docker Compose 示例，因为它具有负载均衡，这意味着我们可以逐个从集群中移除节点而无需连接到其他节点。在我们的案例中，我们有一个三节点集群，因此最多可以容忍一个不可用的节点。

首先，让我们检查我们的节点是否正在运行。我们将通过运行 `docker ps` 命令来实现，该命令只显示正在运行的容器，并输出它们的 ID 和名称：

```
$ docker ps --format '{{.ID}} {{.Names}}'
207279bf61f8 lb-haproxy-1
82b6ddd19251 node2
4552971fd61b node3
45759f6a0830 node1
```

在开始从集群中移除节点之前，我们将检查集群节点的范围状态。在下面的输出中，我只保留了相关的范围信息：

```
$ cockroach node status --ranges --insecure
  id | is_available | leaders | leaseholders | ranges | unavailable | underreplicated
-----+--------------+---------+--------------+--------+-------------+----------------
   1 |       true   |      19 |           19 |     53 |           0 |                0
   2 |       true   |      14 |           14 |     53 |           0 |                0
   3 |       true   |      20 |           20 |     53 |           0 |                0
```

让我们从集群中移除一个节点。我们将从移除 `node1` 开始，观察集群复制如何受到影响，然后在继续处理下一个节点之前将其重新加入集群：

```
$ docker stop 45759f6a0830
45759f6a0830
```

以这种方式停止 Docker 容器等同于启动节点的正常关机；节点在终止前会被清空。这对我们的目的来说没问题，因为我们只是断言在一个节点离开后集群能继续运行。

短暂暂停后，我们将重新运行节点状态命令，看看这对集群产生了什么影响。



##### CockroachDB 节点状态测试

我们可以通过以下命令检查节点状态：

```bash
$ cockroach node status --ranges --insecure
```

输出如下：

```
id | is_available | leaders | leaseholders | ranges | unavailable | underreplicated
-----+--------------+---------+--------------+--------+-------------+----------------
1 | false | 0 | 0 | 53 | 0 | 0
2 | true | 22 | 22 | 53 | 0 | 22
3 | true | 31 | 31 | 53 | 0 | 31
```

可以看到`node1`现在不可用并且没有`leaseholders`。剩余节点中的`ranges`现在显示为`underreplicated`，因为复制不再发生在所有三个节点上。让我们重新启动`node1`，看看这如何影响复制：

```bash
$ docker start 45759f6a0830
45759f6a0830

$ cockroach node status --ranges --insecure
```

输出如下：

```
id | is_available | leaders | leaseholders | ranges | unavailable | underreplicated
-----+--------------+---------+--------------+--------+-------------+-----------------
1 | true | 13 | 13 | 53 | 0 | 0
2 | true | 18 | 18 | 53 | 0 | 0
3 | true | 24 | 24 | 53 | 0 | 0
```

我们恢复到了三个节点，`underreplication`问题得到解决。现在我将对`node2`重复此过程。请注意，我在停止和启动节点后，会等待几分钟再运行节点状态请求：

```bash
$ docker stop 82b6ddd19251
82b6ddd19251

$ cockroach node status --ranges --insecure
```

输出如下：

```
id | is_available | leaders | leaseholders | ranges | unavailable | underreplicated
-----+--------------+---------+--------------+--------+-------------+---------------------
1 | true | 22 | 22 | 53 | 0 | 22
2 | true | 0 | 0 | 53 | 0 | 0
3 | true | 31 | 31 | 53 | 0 | 31
```

```bash
$ docker start 82b6ddd19251
82b6ddd19251

$ cockroach node status --ranges --insecure
```

Chapter 8 testing

输出如下：

```
id | is_available | leaders | leaseholders | ranges | unavailable | underreplicated
-----+--------------+---------+--------------+--------+-------------+----------------
1 | true | 17 | 17 | 53 | 0 | 0
2 | true | 13 | 13 | 53 | 0 | 0
3 | true | 23 | 23 | 53 | 0 | 0
```

最后，我们将对`node3`执行相同的操作：

```bash
$ docker stop 4552971fd61b
4552971fd61b

$ cockroach node status --ranges --insecure
```

输出如下：

```
id | is_available | leaders | leaseholders | ranges | unavailable | underreplicated
-----+--------------+---------+--------------+--------+-------------+----------------
1 | true | 28 | 28 | 53 | 0 | 28
2 | true | 25 | 25 | 53 | 0 | 25
3 | true | 0 | 0 | 53 | 0 | 0
```

```bash
$ docker start 4552971fd61b
4552971fd61b

$ cockroach node status --ranges --insecure
```

输出如下：

```
id | is_available | leaders | leaseholders | ranges | unavailable | underreplicated
-----+--------------+---------+--------------+--------+-------------+----------------
1 | true | 22 | 22 | 53 | 0 | 0
2 | true | 18 | 18 | 53 | 0 | 0
3 | true | 13 | 13 | 53 | 0 | 0
```

完成单节点关闭测试后，我们的服务器已恢复到全部容量，并且没有发生任何中断。现在让我们尝试从集群中移除两个节点：

```bash
$ docker stop 45759f6a0830 82b6ddd19251
45759f6a0830
82b6ddd19251

$ cockroach node status --ranges --insecure
```

输出如下：

```
ERROR: server closed the connection.
Is this a CockroachDB node?
Chapter 8 testing
EOF
Failed running "node status"
```

```bash
$ cockroach node status --ranges --insecure
```

输出如下：

```
id | is_available | leaders | leaseholders | ranges | unavailable | underreplicated
-----+--------------+---------+--------------+--------+-------------+----------------
1 | true | 0 | 0 | 53 | 0 | 0
2 | true | 0 | 0 | 53 | 0 | 0
3 | true | 52 | 0 | 53 | 0 | 0
```

```bash
$ cockroach node status --ranges --insecure
```

输出如下：

```
id | is_available | leaders | leaseholders | ranges | unavailable | underreplicated
-----+--------------+---------+--------------+--------+-------------+----------------
1 | true | 14 | 12 | 53 | 0 | 0
2 | true | 16 | 16 | 53 | 0 | 0
3 | true | 23 | 22 | 53 | 0 | 0
```

观察到在我们停止两个节点后，集群不再可用。这是符合预期的，因为在一个三节点集群中，我们最多只能容忍一个节点不可用。

还观察到，在重启节点后立即运行节点状态请求时，没有任何节点拥有`leaseholders`。直到重启的节点（`node1`）...


