# 数据中心二
110.56.12.120=DC2:RAC1
110.50.13.201=DC2:RAC1
110.54.35.184=DC2:RAC1
50.33.23.120=DC2:RAC2
50.45.14.220=DC2:RAC2
50.17.10.203=DC2:RAC2
```

此示例显示了一个包含两个物理数据中心、每个数据中心有两个机架的文件。`PropertyFileSnitch` 使用 `cassandra-topologies.properties` 文件。如果你没有在 `cassandra-topologies.properties` 文件中标识集群的任何节点，数据库会假定它们位于默认数据中心 (`datacenter`) 和默认机架 (`rack1`) 中。

当你在集群中添加和删除节点时，必须更新此文件，以使 Cassandra 了解节点所属的机架和数据中心。虽然 Cassandra 可以自己弄清楚这些，但从性能角度来看，你主动提供这些信息给 Cassandra 会更好。

以下是一个典型的 `cassandra-rackdc.properties` 文件的内容：

```
