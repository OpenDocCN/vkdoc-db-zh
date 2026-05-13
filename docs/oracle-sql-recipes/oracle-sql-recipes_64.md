# Oracle 数据库性能监控与 SQL 调优

## 使用 AWR 报告
AWR 报告适用于查看整个系统性能并识别消耗资源最多的 SQL 查询。运行以下脚本生成 AWR 报告：
```
SQL> @?/rdbms/admin/awrrpt
```

从 AWR 输出中，在报告的“SQL ordered by Elapsed Time”或“SQL ordered by CPU Time”部分识别消耗资源最多的语句。以下是部分样本输出：

### SQL ordered by CPU Time 样本输出
```
DB/Inst: DWREP/DWREP  Snaps: 11384-11407
-> Resources reported for PL/SQL code includes the resources used by all SQL
   statements called by the code.
-> % Total DB Time is the Elapsed Time of the SQL statement divided into the
   Total Database Time multiplied by 100
                                                                          
    CPU    Elapsed      CPU per    % Total                                    
Time (s) Time (s)   Executions  Exec (s)  DB Time SQL Id                   
---------- ---------- ------------ ---------- ------- -------------          
     4,809     13,731           10    480.86     6.2 8wx77jyhdr31c          
```

**[IT 电子书库](http://www.it-ebooks.info/)**

### 示例 SQL 查询
```sql
Module: JDBC Thin Client
SELECT D.DERIVED_COMPANY
      ,CB.CLUSTER_BUCKET_ID
      ,CB.CB_NAME
      ,CB.SOA_ID
      ,COUNT(*) TOTAL
      ,NVL(SUM(CASE WHEN F.D_DATE_ID > TO_NUMBER(TO_CHAR(SYSDATE-30,'YYYYMMDD')) THEN 1 END), 0) RECENT
      ,NVL(D.BLACKLIST_FLG,0) BLACKLIST_FLG
  FROM F_DOWNLOADS F
      ,D_DOMAINS D
      ,D_PRODUCTS P
      ,PID_DF_ASSOC PDA
      ,( SELECT * FROM ( SELECT CLUSTER_
```

从 Oracle 数据库 10*g*开始，Oracle 会每小时自动对数据库进行一次快照，并填充存储统计信息的底层 AWR 表。默认情况下，统计信息会保留七天。

您还可以通过运行`awrsqrpt.sql`报告来为特定的 SQL 语句生成 AWR 报告。运行以下脚本时，系统将提示您输入感兴趣的查询的`SQL_ID`：
```
SQL> @?/rdbms/admin/awrsqrpt.sql
```

## 使用 ADDM
ADDM 报告提供了关于哪些 SQL 语句是调优候选者的有用建议。使用以下 SQL 脚本生成 ADDM 报告：
```
SQL> @?/rdbms/admin/addmrpt
```

查找报告中标有“SQL statements consuming significant database time.”的部分。

### ADDM 报告样本输出
```
FINDING 2: 29% impact (65043 seconds)
------------------------------
  SQL statements consuming significant database time were found.

  RECOMMENDATION 1: SQL Tuning, 6.7% benefit (14843 seconds)
  ----------------------------------------------------------
     ACTION: Investigate the SQL statement with SQL_ID "46cc3t7ym5sx0"
        for possible performance improvements.
     RELEVANT OBJECT: SQL statement with SQL_ID 46cc3t7ym5sx0 and
        PLAN_HASH 1234997150
     MERGE INTO d_files a
     USING
      ( SELECT
```

ADDM 报告分析 AWR 表中的数据，以识别潜在的瓶颈和高资源消耗的 SQL 查询。

## 使用 ASH
ASH 报告使您能够关注最近运行过且可能只执行了短暂时间的短生命周期 SQL 语句。使用以下脚本生成 ASH 报告：
```
SQL> @?/rdbms/admin/ashrpt
```

在输出中搜索标有“Top SQL”的部分。以下是部分样本输出：

### ASH 报告样本输出
```
SQL ID        Planhash            Sampled # of Executions    % Activity
----------------------- -------------------- -------------------- --------------
Event                          % Event Top Row Source              % RwSrc
------------------------------ ------- --------------------------------- -------
4k8runghhh31d                 3219321046               12      51.61
CPU + Wait for CPU             51.61 HASH JOIN                    12.26
select countryimp0_.COUNTRY_ID as COUNTRY_ID, countryimp0_.COUNTRY_NAME
```

前面的输出表明该查询正在等待 CPU 资源。在这种情况下，实际问题可能是另一个消耗 CPU 资源的查询。

ASH 报告何时比 AWR 或 ADDM 报告更有用？AWR 和 ADDM 输出显示的是总数据库时间方面的顶级消耗 SQL。如果 SQL 性能问题是短暂且生命周期短的，它可能不会出现在 AWR 和 ADDM 报告中。在这些情况下，ASH 报告更有用。

> **注意：** 您也可以从 Enterprise Manager 运行 AWR、ADDM 和 ASH 报告，您可能会觉得这比从 SQL*Plus 手动运行脚本更直观。

## 使用 Statspack



