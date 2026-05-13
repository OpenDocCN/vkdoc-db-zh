# 部署拓扑

请注意，在 `example_regional_rows` 表（其*行*是区域特定的表）中，为该表使用的每个地理分区区域都有两个节点。

与 `example_regional_table` 示例类似，range leaseholder 也存在于该区域中。

如果您要求数据存在于单个区域中，则需要将数据库的生存目标从 `REGION` 更改为 `ZONE`，并更新复制因子以反映该区域中的节点数。这个集群有三个区域，每个区域有三个节点，因此反映这一点的命令是：

```sql
ALTER DATABASE "topologies_demo" SURVIVE ZONE FAILURE;
SET override_multi_region_zone_config = true;
ALTER TABLE example_regional_rows CONFIGURE ZONE USING num_voters = 3;
ALTER TABLE example_regional_rows CONFIGURE ZONE USING num_replicas = 3;
SET override_multi_region_zone_config = true;
ALTER TABLE example_regional_table CONFIGURE ZONE USING num_voters = 3;
ALTER TABLE example_regional_table CONFIGURE ZONE USING num_replicas = 3;
```

大约一分钟后，再次检查表 ranges。这次，您会注意到因为我们显式移除了对区域故障的生存要求，所有数据现在都存在于我们为表配置的区域中：

```sql
SELECT lease_holder_locality, replicas, replica_localities
FROM [SHOW RANGES FROM TABLE example_regional_table];
```

`region=us-central1,zone=us-central1a`
`{4,6,9}`
```
{
"region=us-central1,zone=us-central1c",
"region=us-central1,zone=us-central1a",
"region=us-central1,zone=us-central1b"
}
```

```sql
SELECT lease_holder_locality, replicas, replica_localities
FROM [SHOW RANGES FROM TABLE example_regional_rows]
WHERE "start_key" NOT LIKE '%Prefix%';
```

