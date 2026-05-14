# 第 29 章 ■ 查询存储

除了平均 I/O，你还可以选择不同的筛选标准。你也可以向查询的 `WHERE` 和/或 `HAVING` 子句添加谓词来缩小结果范围。例如，如果你想检测在 OLTP 环境中使用了并行的查询，然后对其进行优化或微调 `并行度成本阈值` 的值，可以添加通过 `DOP` 进行筛选。

**重要提示** SQL Server 2016 RTM 存在一个 bug，当 `sys.query_store_plan` 视图在同一语句中与其他视图联接时，有时会损坏其返回的执行计划的文本表示。你可以通过先获取 `plan_id`，然后在不涉及任何联接的情况下查询 `sys.query_store_plan` 视图来实现解决方法。此 bug 在某个 CU 版本中已修复。

## 清单 29-3. 获取有关性能回退的信息

清单 29-3 返回过去 72 小时内发生的查询性能回退信息。它使用平均查询时长增加一倍作为回退标准，并为每个查询返回一行，包含具有最低和最高平均时长的计划。你可以使用输出中的 `query_id` 来对回退进行进一步分析。

```sql
;with Regressions(query_id, query_text_id, plan1_id, plan2_id, plan1
,plan2, dur1, dur2, row_num)
as
(
    select
        q.query_id, q.query_text_id, qp1.plan_id, q2.plan_id
        ,qp1.query_plan, q2.query_plan, rs1.avg_duration, q2.avg_duration
        ,row_number() over (partition by qp1.plan_id order by rs1.avg_duration)
    from
        sys.query_store_query q join sys.query_store_plan qp1 on
            q.query_id = qp1.query_id
        join sys.query_store_runtime_stats rs1 on
            qp1.plan_id = rs1.plan_id
        join sys.query_store_runtime_stats_interval rsi1 on
            rs1.runtime_stats_interval_id = rsi1.runtime_stats_interval_id
        cross apply
        (
            select top 1
                qp2.query_plan, qp2.plan_id, rs2.avg_duration
            from
                sys.query_store_plan qp2
                join sys.query_store_runtime_stats rs2 on
                    qp2.plan_id = rs2.plan_id
                join sys.query_store_runtime_stats_interval rsi2 on
                    rs2.runtime_stats_interval_id =
                        rsi2.runtime_stats_interval_id
            where
                q.query_id = qp2.query_id and
                qp1.plan_id <> qp2.plan_id and
                rsi1.start_time < rsi2.start_time and
                rs1.avg_duration * 2 <= rs2.avg_duration
            order by
                rs2.avg_duration desc
        ) q2
    where
        rsi1.start_time >= dateadd(day,-3,getdate())
)
select
    r.query_id, qt.query_sql_text, r.plan1_id, r.plan1, r.plan2_id, r.plan2
    ,r.dur1, r.dur2
from
    Regressions r join sys.query_store_query_text qt on
        r.query_text_id = qt.query_text_id
where
    r.row_num = 1
order by
    r.dur2 / r.dur1 desc;
```

## 清单 29-4. 具有多个上下文设置的查询

你还可以使用查询存储来检测污染计划缓存的查询。清单 29-4 说明了如何获取有关因不同上下文设置而生成多个执行计划的查询的信息。发生这种情况的两个最常见条件是：会话使用了不同的 `SET` 选项，以及查询引用对象时没有使用架构名称。

```sql
select
    q.query_id, qt.query_sql_text
    ,count(distinct q.context_settings_id) as [Context Setting Cnt]
    ,count(distinct qp.plan_id) as [Plan Count]
from
    sys.query_store_query q join sys.query_store_query_text qt on
        q.query_text_id = qt.query_text_id
    join sys.query_store_plan qp on
        q.query_id = qp.query_id
group by
    q.query_id, qt.query_sql_text
having
    count(distinct q.context_settings_id) > 1
order by
    count(distinct q.context_settings_id);
```

清单 29-5 展示了如何查找具有重复 `query_hash` 值且执行次数较少的相似查询。通常，这些查询属于系统中的非参数化即席工作负载。你应该在代码中将这些查询参数化，或者，如果不可能，可以考虑强制它们。



