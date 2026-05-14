# 第 29 章 ■ 查询存储

#### 管理与监控查询存储

尽管查询存储不应给系统带来明显的性能开销，但监控其健康状况和对性能的影响仍然很重要。这将使您能够以最小化性能开销并提供足够细粒度的分析数据的方式，微调查询存储的参数。

我们在[第 26 章](http://dx.doi.org/10.1007/978-1-4842-1964-5_26)中讨论过的数据库级别的参数化或使用计划指南。

#### 清单 29-5. 检测具有相同哈希值的查询

```sql
select top 100
 q.query_hash
 ,count(*) as [Query Count]
 ,avg(rs.count_executions) as [Avg Exec Count]
from
 sys.query_store_query q join sys.query_store_plan qp on
  q.query_id = qp.query_id
 join sys.query_store_runtime_stats rs on
  qp.plan_id = rs.plan_id
group by
 q.query_hash
having
 count(*) > 1
order by
 [Avg Exec Count] asc, [Query Query] desc
```

如您所见，查询存储提供的信息为系统分析和性能调优提供了无限的可能性。

查询存储的大小取决于数据保留策略（由 `STALE_QUERY_THRESHOLD_DAYS` 和 `SIZE_BASED_CLEANUP_POLICY` 设置控制）和收集模式（由 `QUERY_CAPTURE_MODE` 和 `MAX_PLAN_PER_QUERY` 设置指定）。此外，运行时统计信息存储的大小在很大程度上取决于聚合间隔，该间隔由 `INTERVAL_LENGTH_MINUTES` 值定义。聚合间隔越短，存储中保存的数据就越多。

以符合您需求的方式定义聚合间隔非常重要。将 `INTERVAL_LENGTH_MINUTES` 值设置得过小会产生过量的数据，从而使分析更加复杂。例如，如果您想创建系统的通用基线，一天的聚合间隔就足够了。但是，如果您需要详细分析工作负载在一天中的变化情况，则应使用一小时甚至更短的间隔。一如既往，关键在于避免在系统中收集不必要的信息。

您可以使用 `sys.database_query_store_options` 视图来分析查询存储的大小及其状态，如清单 29-6 所示。您应通过分析 `current_storage_size_mb` 和 `max_storage_size_mb` 值来监控查询存储的可用空间。请记住：当查询存储已满时，它将切换为只读模式。

#### 清单 29-6. 分析查询存储状态

```sql
select actual_state_desc, desired_state_desc, current_storage_size_mb
 ,max_storage_size_mb, readonly_reason, interval_length_minutes
 ,stale_query_threshold_days, size_based_cleanup_mode_desc
 ,query_capture_mode_desc
from sys.database_query_store_options
```

您可以使用 `ALTER DATABASE SET QUERY_STORE CLEAR` 语句或在 Management Studio 中清除查询存储中的数据。或者，您可以使用 `sys.sp_query_store_remove_query` 存储过程按查询清除查询存储，如清单 29-7 所示。此代码清除所有超过三天且仅执行过一次的查询。顺便提一下，`sys.sp_query_store_remove_plan` 存储过程允许您从查询存储中移除单个计划。

#### 清单 29-7. 从查询存储中移除查询

```sql
declare
 @RecId int = -1
 ,@QueryId int;

declare
 @Queries table
 (
  RecId int not null identity(1,1) primary key,
  QueryId int not null
 );

insert into @Queries(QueryId)
select p.query_id
from sys.query_store_plan p join sys.query_store_runtime_stats rs on
 p.plan_id = rs.plan_id
group by
 p.query_id
having
 sum(rs.count_executions) < 2 and
 max(rs.last_execution_time) < dateadd(day,-72,getdate());

while 1 = 1
begin
 select top 1 @RecId = RecId, @QueryID = QueryId
 from @Queries
 where RecId > @RecId
 order by RecID;

 if @@rowcount = 0
  break;

 exec sys.sp_query_store_remove_query @QueryID;
end;
```

有几种方法可以监控查询存储的性能。


