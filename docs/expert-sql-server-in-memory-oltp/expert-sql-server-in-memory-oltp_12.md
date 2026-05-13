# 代码清单 4-2. 获取哈希索引信息

代码清单 4-2 展示了使用 `sys.dm_db_xtp_hash_index_stats` 视图返回桶计数和行链信息的查询。请记住，此视图会扫描整个表，当表很大时，这非常耗时。

```sql
select
s.name + '.' + t.name as [Table]
,i.name as [Index]
,stat.total_bucket_count as [Total Buckets]
,stat.empty_bucket_count as [Empty Buckets]
,floor(100\. * empty_bucket_count / total_bucket_count)
as [Empty Bucket %]
,stat.avg_chain_length as [Avg Chain]
,stat.max_chain_length as [Max Chain]
from
sys.dm_db_xtp_hash_index_stats stat
join sys.tables t on
stat.object_id = t.object_id
join sys.indexes i on
stat.object_id = i.object_id and
stat.index_id = i.index_id
join sys.schemas s on
t.schema_id = s.schema_id
```

