# 清单 26-29. 使用 sys.dm_exec_requests

```sql
select
    er.session_id, er.user_id, er.status, er.database_id, er.start_time
    ,er.total_elapsed_time, er.logical_reads, er.writes
    ,substring(qt.text, (er.statement_start_offset/2)+1,
        ( ( case er.statement_end_offset
            when -1 then datalength(qt.text)
            else er.statement_end_offset
            end - er.statement_start_offset ) /2 ) +1 ) as [SQL]
    ,qp.query_plan, er.*
from
    sys.dm_exec_requests er with (nolock)
    cross apply sys.dm_exec_sql_text(er.sql_handle) qt
    cross apply sys.dm_exec_query_plan(er.plan_handle) qp
where
    er.session_id > 50 and /* Excluding system processes */
    er.session_id <> @@SPID
order by
    er.total_elapsed_time desc
```

