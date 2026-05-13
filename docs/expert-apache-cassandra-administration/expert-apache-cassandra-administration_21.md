# docker run --name cassandra3 -m 2g -d -e CASSANDRA_SEEDS="172.17.0.3" cassandra:3.0.4
7528dc5679b02a59bb13f30ce0dc8c7617f0c268fd52256d8e2d9ea5faf0586a
#
```

`CASSANDRA_SEEDS` 环境变量是一个以逗号分隔的 IP 地址列表，gossip 协议使用该列表在新创建的 Cassandra 节点加入集群时引导它们。

检查 Docker 进程以确保所有三个 Cassandra 容器都在运行。

```bash
