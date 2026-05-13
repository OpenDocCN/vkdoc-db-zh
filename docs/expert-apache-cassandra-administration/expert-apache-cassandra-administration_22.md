# sudo docker ps
CONTAINER ID  IMAGE      COMMAND    CREATED     STATUS    PORTS       NAMES
7528dc5679b0        cassandra:3.0.4     "/docker-entrypoin..."   5 seconds ago   Up 4 seconds     7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   cass3
30ed76a87b3c     cassandra:3.0.4     "/docker-entrypoin..."   5 minutes ago       Up 5 minutes        7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp       cass2
34842881b8d4     cassandra:3.0.4     "/docker-entrypoin..."   8 minutes ago       Up 8 minutes      7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp    cassandra1
#
```

最后的 `nodetool status` 命令显示 Cassandra 集群的所有三个节点都在运行。

```bash
