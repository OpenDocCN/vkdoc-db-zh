# 创建键空间

我们需要创建一个键空间来存储表。在 `CreateCassandraDatabase` 应用程序中添加一个静态方法 `createKeyspace()` 来创建键空间。

CQL 3（Cassandra 查询语言 3）增加了有条件运行 `CREATE` 语句的支持，这意味着只有当要构建的对象尚不存在时才会创建该对象。`IF NOT EXISTS` 子句用于条件创建。

创建一个名为 `datastax` 的键空间，使用复制策略类为 `SimpleStrategy`，复制因子为 1。

```
session.execute("CREATE KEYSPACE IF NOT EXISTS datastax WITH replication "
    + "= {'class':'SimpleStrategy', 'replication_factor':1};");
```

在 `main` 方法中调用 `createKeyspace()` 方法。当应用程序运行时，键空间就会被创建。

Cassandra 支持以下策略类，如表 6-3 所示，这些类指的是副本放置策略类。

**表 6-3. 策略类**

| 类 | 描述 |
| --- | --- |
| `org.apache.cassandra.locator.SimpleStrategy` | 仅用于单个数据中心。第一个副本根据分区器的决定放置在一个节点上。随后的副本以顺时针方式放置在节点环中的下一个节点上，不考虑拓扑。仅当使用 `SimpleStrategy` 类时才需要复制因子。 |
| `org.apache.cassandra.locator.NetworkTopologyStrategy` | 用于多个数据中心。指定在每个数据中心存储多少个副本。尝试将副本存储在同一数据中心内的不同机架上，因为同一机架内的节点更有可能一起发生故障。 |

