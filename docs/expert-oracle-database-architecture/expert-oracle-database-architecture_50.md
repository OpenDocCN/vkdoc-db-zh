# 索引组织表与堆表的性能对比

首先，我们收集表的统计信息：
```
SQL> exec dbms_stats.gather_table_stats( user, 'HEAP_ADDRESSES' );
PL/SQL procedure successfully completed.
SQL> exec dbms_stats.gather_table_stats( user, 'IOT_ADDRESSES' );
PL/SQL procedure successfully completed.
```

现在，我们可以看看预期的可衡量差异。使用 `AUTOTRACE`，我们将感受到这种变化：
```
SQL> set autotrace traceonly
SQL> select *  from emp, heap_addresses
where emp.empno = heap_addresses.empno
and emp.empno = 42;
Execution Plan

Plan hash value: 775524973

| Id  | Operation                          | Name           | Rows | Bytes | Cost (%CPU)|     Time  |

|   0 | SELECT STATEMENT                   |                |    4 |   292 |    8    (0)| 00:00:01  |
|   1 | NESTED LOOPS                       |                |    4 |   292 |    8    (0)| 00:00:01  |
|   2 | TABLE ACCESS BY INDEX ROWID        | EMP            |    1 |    27 |    2    (0)| 00:00:01  |
|*  3 | INDEX UNIQUE SCAN                  | EMP_PK         |    1 |       |    1    (0)| 00:00:01  |
|   4 | TABLE ACCESS BY INDEX ROWID BATCHED| HEAP_ADDRESSES |    4 |   184 | ...
|*  5 | INDEX RANGE SCAN                   | SYS_C0032863   |    4 |       |    2    (0)| 00:00:01  |

Predicate Information (identified by operation id):

3 - access("EMP"."EMPNO"=42)
5 - access("HEAP_ADDRESSES"."EMPNO"=42)
Statistics

1  recursive calls
0  db block gets
11  consistent gets
0  physical reads
0  redo size
1361  bytes sent via SQL*Net to client
543  bytes received via SQL*Net from client
2  SQL*Net roundtrips to/from client
0  sorts (memory)
0  sorts (disk)
4  rows processed
```

这是一个相当常见的执行计划：通过主键访问 `EMP` 表；获取行数据；然后使用 `EMPNO` 去地址表；并通过索引获取子记录。我们进行了 11 次 I/O 来检索这些数据。现在运行相同的查询，但对地址表使用 IOT：
```
SQL> select * from emp, iot_addresses
where emp.empno = iot_addresses.empno
and emp.empno = 42;
Execution Plan

Plan hash value: 252066017

| Id  | Operation                  | Name               |  Rows | Bytes | Cost (%CPU)|     Time |

|   0 | SELECT STATEMENT           |                    |     4 |   292 |    4    (0)| 00:00:01 |
|   1 | NESTED LOOPS               |                    |     4 |   292 |    4    (0)| 00:00:01 |
|   2 | TABLE ACCESS BY INDEX ROWID| EMP                |     1 |    27 |    2    (0)| 00:00:01 |
|*  3 | INDEX UNIQUE SCAN          | EMP_PK             |     1 |       |    1    (0)| 00:00:01 |
|*  4 | INDEX RANGE SCAN           | SYS_IOT_TOP_182459 |     4 |   184 |    2    (0)| 00:00:01 |

Predicate Information (identified by operation id):

3 - access("EMP"."EMPNO"=42)
4 - access("IOT_ADDRESSES"."EMPNO"=42)
Statistics

1  recursive calls
0  db block gets
7  consistent gets
0  physical reads
0  redo size
1361  bytes sent via SQL*Net to client
543  bytes received via SQL*Net from client
2  SQL*Net roundtrips to/from client
0  sorts (memory)
0  sorts (disk)
4  rows processed
```

我们减少了四次 I/O（这四次应该是可以推测出来的）；我们跳过了四次 `TABLE ACCESS` `(BY INDEX ROWID BATCHED)` 步骤。我们拥有的子记录越多，预计跳过的 I/O 也就越多。

那么，四次 I/O 意味着什么呢？在这个案例中，它占了查询执行的 I/O 的三分之一以上，如果我们重复执行这个查询，这些开销会累积起来。每次 I/O 和每次一致性获取都需要访问缓冲区缓存，虽然从缓冲区缓存中读取数据确实比磁盘快，但缓冲区缓存获取也**并非免费，也并不完全廉价**。每次获取都需要多次获取缓冲区缓存的闩锁，而闩锁是序列化设备，会限制我们的扩展能力。我们可以通过运行如下所示的 PL/SQL 块来衡量 I/O 减少和闩锁减少：
```
SQL> begin
for x in ( select empno from emp )
loop
for y in ( select emp.ename, a.street, a.city, a.state, a.zip
from emp, heap_addresses a
where emp.empno = a.empno
and emp.empno = x.empno )
loop
null;
end loop;
end loop;
end;
/
PL/SQL procedure successfully completed.
```

在这里，我们只是在模拟一个繁忙时段，为每个 `EMPNO` 运行一次查询，总计约 72,000 次。如果我们分别对 `HEAP_ADDRESSES` 和 `IOT_ADDRESSES` 表运行该查询，`TKPROF` 会显示以下结果：
```
SELECT EMP.ENAME, A.STREET, A.CITY, A.STATE, A.ZIP
FROM EMP, HEAP_ADDRESSES A WHERE EMP.EMPNO = A.EMPNO AND EMP.EMPNO = :B1
call     count       cpu  elapsed  disk      query    current        rows
------- ------  --------   ------ ----- ---------- ----------  ----------
Parse        1      0.00     0.00     0          0          0           0
Execute  72110      1.02     1.01     0          0          0           0
Fetch    72110      2.16     2.11     0     722532          0      288440
------- ------  -------- -------- ----- ---------- ----------  ----------
total   144221      3.18     3.12     0     722532          0      288440
...
Rows (1st) Rows (avg) Rows (max) Row Source Operation
---------- ---------- ---------- ------------------------------------------------
4  4  4 NESTED LOOPS  (cr=10 pr=0 pw=0 time=40 us cost=8 size=228 card=4)
1  1  1 TABLE ACCESS BY INDEX ROWID EMP (cr=3 pr=0 pw=0 time=11 us cost=2...
1  1  1 INDEX UNIQUE SCAN EMP_PK (cr=2 pr=0 pw=0 time=7 us cost=1 size=0...
4  4  4 TABLE ACCESS BY INDEX ROWID BATCHED HEAP_ADDRESSES (cr=7...
4  4  4 INDEX RANGE SCAN SYS_C0032863 (cr=3 pr=0 pw=0 time=10 us cost=2...
*********************************************************************************
SELECT EMP.ENAME, A.STREET, A.CITY, A.STATE, A.ZIP
FROM EMP, IOT_ADDRESSES A WHERE EMP.EMPNO = A.EMPNO AND EMP.EMPNO = :B1
call     count       cpu  elapsed  disk      query    current        rows
------- ------  -------- -------- ----- ---------- ----------  ----------
Parse        1      0.00     0.00     0          0          0           0
Execute  72110      1.04     1.01     0          0          0           0
Fetch    72110      1.64     1.63     0     437360          0      288440
------- ------  -------- -------- ----- ---------- ----------  ----------
total   144221      2.69     2.64     0     437360          0      288440
...
Rows (1st) Rows (avg) Rows (max) Row Source Operation
---------- ---------- ---------- ----------------------------------------------
4  4  4 NESTED LOOPS  (cr=7 pr=0 pw=0 time=28 us cost=4 size=228 card=4)
1  1  1 TABLE ACCESS BY INDEX ROWID EMP (cr=3 pr=0 pw=0 time=11 us cost=2...
1  1  1 INDEX UNIQUE SCAN EMP_PK (cr=2 pr=0 pw=0 time=7 us cost=1 size=0...
4  4  4 INDEX RANGE SCAN SYS_IOT_TOP_182459 (cr=4 pr=0 pw=0 time=15 us...
Rows  Row Source Operation
----  ----------------------------------------------------------------------
4  NESTED LOOPS  (cr=7 pr=3 pw=0 time=9 us cost=4 size=280 card=4)
1  TABLE ACCESS BY INDEX ROWID EMP (cr=3 pr=0 pw=0 time=0 us cost=2 size=30...)
1  INDEX UNIQUE SCAN EMP_PK (cr=2 pr=0 pw=0 time=0 us cost=1 size=0 ...)
4  INDEX RANGE SCAN SYS_IOT_TOP_93124 (cr=4 pr=3 pw=0 time=3 us cost=2 ...)
```

两个查询获取了完全相同的行数，但 `HEAP` 表执行了多得多的逻辑 I/O。随着系统并发程度的增加，我们同样预期 `HEAP` 表使用的 CPU 也会更快地增长，同时查询可能会等待进入缓冲区缓存的闩锁。使用 runstats（我自己设计的一个工具；详情请参考本书导言部分的“设置您的环境”），我们可以衡量闩锁的差异。在我的系统上，我观察到以下情况：



