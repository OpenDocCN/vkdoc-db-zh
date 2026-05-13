# 部署拓扑

```sql
SELECT lease_holder_locality, replicas, replica_localities
FROM [SHOW RANGES FROM TABLE example_regional_rows]
WHERE "start_key" NOT LIKE '%Prefix%';
```

`region=us-central1,zone=us-central1b`
`{1,3,4,6,8}`
```
{
"region=us-east1,zone=us-east1a",
"region=us-east1,zone=us-east1c",
"region=us-central1,zone=us-central1c",
"region=us-central1,zone=us-central1b",
"region=us-west1,zone=us-west1b"
}
```

`region=us-east1,zone=us-east1a`
`{1,3,5,7,8}`
```
{
"region=us-east1,zone=us-east1a",
"region=us-east1,zone=us-east1c",
"region=us-central1,zone=us-central1a",
"region=us-west1,zone=us-west1a",
"region=us-west1,zone=us-west1b"
}
```

`region=us-west1,zone=us-west1b`
`{1,4,6,7,8}`
```
{
"region=us-east1,zone=us-east1a",
"region=us-central1,zone=us-central1c",
"region=us-central1,zone=us-central1b",
"region=us-west1,zone=us-west1a",
"region=us-west1,zone=us-west1b"
}
```

默认的复制因子是五，因此可以看到使用了所有区域中的节点来存储数据。这是因为 CockroachDB 不知道您为什么要对数据进行地理分区，因此在默认的生存目标 `REGION` 下，它会尝试以尽可能弹性的方式复制数据。当数据可以在区域间自由流动时，这是可以接受的，但出于监管原因需要固定数据时则不可接受。

请注意，`example_regional_table` 表在 `us-central1` 区域内有两个节点，其 leaseholder 也位于 `us-central1`。

