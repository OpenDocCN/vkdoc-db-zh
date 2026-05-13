# 表 4-1. INSERT 语句执行时间

表 4-1 显示了在我的测试环境中 `INSERT` 语句的执行时间。如你所见，向 `dbo.HashIndex_HighBucketCount` 表中插入数据比向对应的 `dbo.HashIndex_LowBucketCount` 表插入数据快大约 35 倍。

| dbo.HashIndex_HighBucketCount (1,048,576 个桶) | dbo.HashIndex_LowBucketCount (1,024 个桶) |
| --- | --- |
| 1,122 ms | 39,955 ms |

