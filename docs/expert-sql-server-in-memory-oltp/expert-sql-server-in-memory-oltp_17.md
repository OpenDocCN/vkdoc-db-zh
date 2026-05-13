# 代码清单 4-4. 桶计数与性能：索引扫描查询

```sql
select count(*)
from dbo.HashIndex_HighBucketCount
with (index= PK_HashIndex_HighBucketCount)
option (maxdop 1);
select count(*)
from dbo.HashIndex_LowBucketCount
with (index= PK_HashIndex_LowBucketCount)
option (maxdop 1);
```

