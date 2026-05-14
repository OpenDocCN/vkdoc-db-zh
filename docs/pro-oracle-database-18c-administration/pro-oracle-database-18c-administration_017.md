# top.sql

以下脚本列出消耗 CPU 最多的 SQL 进程。它对于识别有问题的 SQL 语句非常有用。将此脚本放在诸如`HOME/scripts`之类的目录中：

```
select * from(
select
sql_text
,buffer_gets
,disk_reads
,sorts
,cpu_time/1000000 cpu_sec
,executions
,rows_processed
from v$sqlstats
order by cpu_time DESC)
where rownum < 11;
```

执行此脚本的方法如下：

```
SQL> @top
```

以下是输出的一个片段，显示了一个正在消耗大量数据库资源的 SQL 语句：

```
INSERT INTO "REP_MV"."GEM_COMPANY_MV"
SELECT   CASE GROUPING_ID(trim(upper(nvl(ad.organization_name,u.company))))
WHEN 0 THEN
trim(upper(nvl(ad.organization_name,u.company)))
11004839   20937562        136   21823.59         17       12926019
```

