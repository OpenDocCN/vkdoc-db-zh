# 3. 分析函数

基础 SQL 提供行级可见性，而聚合函数允许我们按组分析数据，使得每一行根据 `group by` 表达式对应到一个特定的组（关于聚合函数的更多细节将在下一章“聚合函数”中提供）。

分析函数引入了窗口级可见性。窗口定义了用于对每个输入行应用函数的行子集，其定义对所有行都是相同的，并在函数的 `analytic clause`（分析子句）中指定。分析函数在诸如 `join`、`where`、`group by`、`having` 等所有操作之后，但在 `order by` 之前进行求值。因此，例如，它们只能出现在 `select` 列表或 `order by` 子句中，而不能出现在 `where` 子句中。与应用聚合函数后结果集中每组由一行表示不同，应用分析函数后，记录集中的行数保持不变。

通过一个例子来解释其工作原理会更容易，如清单 3-1 所示。

```sql
with t as
(select rownum id, trunc(rownum / 4) part from dual connect by rownum <= 6)
select t.*,
sum(id) over(partition by part order by id) sum1,
sum(id) over(partition by part) sum2
from t
order by id;
ID       PART       SUM1       SUM2
---------- ---------- ---------- ----------
1          0          1          6
2          0          3          6
3          0          6          6
4          1          4         15
5          1          9         15
6          1         15         15
6 rows selected.
清单 3-1
分析函数
```

`partition by part` 意味着我们为每个 `part` 独立地应用分析函数。如果省略，则整个记录集被视为一个分区。如果没有 `order by` 子句，则每行的窗口覆盖当前分区的所有行，因此所有行的结果都相同。使用 `order by` 子句时，每行的窗口覆盖从分区开始到当前行的所有行。这可以通过在 `order by` 后指定一个 `windowing clause`（开窗子句）来调整。当指定了 `order by` 时，默认行为是 `range between unbounded preceding and current row`（或简写为 `range unbounded preceding`）；否则是 `range between unbounded preceding and unbounded following`。

`partition by` 子句不是强制性的，同样，在 `order by` 后也可能不指定 `windowing clause`；然而，对于某些函数，必须始终提供 `order by`——例如，在使用 `row_number` 或 `rank` 的情况下。

使用分析函数和单表访问可以实现的逻辑，否则将需要额外的连接或子查询。清单 3-2 展示了如何在不使用分析函数的情况下实现清单 3-1 中的逻辑。

```sql
with t as
(select rownum id, trunc(rownum / 4) part from dual connect by rownum <= 6)
select t.*,
(select sum(id) from t t0 where t0.part = t.part and t0.id <= t.id) sum1,
(select sum(id) from t t0 where t0.part = t.part) sum2
from t
order by id;
ID       PART       SUM1       SUM2
---------- ---------- ---------- ----------
1          0          1          6
2          0          3          6
3          0          6          6
4          1          4         15
5          1          9         15
6          1         15         15
6 rows selected.
清单 3-2
重写查询而不使用分析函数
```

回到查询转换的话题，Oracle 无法将此查询重写为使用分析函数以避免不必要的连接和表扫描。这种智能在不久的将来不太可能被加入。

即使连接条件中使用了不同的列，分析函数也有助于避免连接。在清单 3-3 中，使用关联标量子查询和分析函数计算了相同的值。

```sql
exec dbms_random.seed(99);
create table ta as
select rownum id,
trunc(dbms_random.value(1, 5 + 1)) x1,
trunc(dbms_random.value(1, 5 + 1)) x2,
trunc(dbms_random.value(1, 5 + 1)) x3
from dual
connect by level <= 10;
select (select sum(x3) from ta t0 where t0.x2 = ta.x2) s,
case when x1 > x2 then
sum(x3) over(order by x2 range between greatest(x1 - x2, 0)
following and greatest(x1 - x2, 0) following)
else
sum(x3) over(order by x2 range between greatest(x2 - x1, 0)
preceding and greatest(x2 - x1, 0) preceding)
end sa,
ta.*
from ta
order by id;
S         SA         ID         X1         X2         X3
---------- ---------- ---------- ---------- ---------- ----------
4          4          1          3          1          2
10         10          2          1          5          4
1          1          3          2          5          1
9          9          4          5          3          4
4          4          5          3          1          1
6          6          6          5          1          1
7          7          7          5          3          1
4          4          8          3          1          5
9          9          9          5          2          1
9          9         10          5          1          2
10 rows selected .
清单 3-3
使用分析函数避免连接
```

为了计算 `x2` 等于 `x1` 的行的 `x3` 之和，我们使用了一个具有等于 `x1` 与 `x2` 之差的范围偏移量的窗口。根据 `x1` 大于还是小于 `x2`，我们考虑 `following`（之后）或 `preceding`（之前）的行。对于每一行，我们只关心一个总和，但 Oracle 需要为所有行计算两个总和。因此，为了避免当 `x1-x2` 或 `x2-x1` 为负数时出现异常，我们应用了 `greatest` 函数。

除了按范围进行的逻辑偏移，窗口还可以通过按行进行物理偏移来指定。为了突显差异，让我们考虑以下任务。有一个包含 ATM 取款信息的表，我们需要为每次取款计算以下内容：

*   考虑到当前交易和之前 5 笔交易（总共 6 笔取款），金额不低于 50 的交易数量（`cnt1`）；
*   考虑到当前交易和之前 5 分钟范围内（`cnt2`）的交易，金额不低于 50 的交易数量。

```sql
exec dbms_random.seed(11);
create table atm as
select trunc(sysdate) + (2 * rownum - 1) / (24 * 60) ts,
trunc(dbms_random.value(1, 20 + 1)) * 5 amount
from dual
connect by level <= 15;
select to_char(ts, 'mi') minute,
amount,
count(nullif(sign(amount - 50), -1))
over(order by ts rows 5 preceding) cnt1,
count(nullif(sign(amount - 50), -1))
over(order by ts range interval '5' minute preceding) cnt2
from atm;
MI     AMOUNT       CNT1       CNT2
-- ---------- ---------- ----------
01         85          1          1
03         15          1          1
05        100          2          2
07         40          2          1
09         30          2          1
11         50          3          1
13         85          3          2
15         60          4          3
17          5          3          2
19        100          4          2
21         25          4          1
23         30          3          1
25         80          3          1
27          5          2          1
29         35          2          1
15 rows selected .
清单 3-4
使用开窗子句实现逻辑
```

清单 3-5 展示了一个稍简单的例子，用以突出范围偏移和行偏移之间的差异。



L1 和 L2 在 `id = 5` 时出现差异，是因为在第一种情况中 `last_value` 的上限是 `3.6` (`4.6 – 1`)，而在第二种情况下它只是前一行的值 `4.5`。

对于某些分析函数，窗口子句没有意义，因此不能为其指定，例如 `lag/lead`。

尽管灵活性很高，分析函数仍有一些限制：

1.  当按多列排序时，只允许 `unbounded preceding`、`current row`、`unbounded following` 边界。例如，如果我们有一个包含点信息（坐标 x 和 y）的表，那么就不可能计算每一行中，有多少点存在于从当前点出发的给定 x 和 y 偏移范围内。
2.  不能在函数中引用当前行的属性。例如，如果我们想计算从当前点到所有其他点的距离之和，那么使用分析函数是无法实现的。然而，如果目标是计算到某个特定点的距离之和，则可以轻松地为不同的行范围完成。

具体细节及简短内联注释如下。

```sql
with points as
(select rownum id, rownum * rownum x, mod(rownum, 3) y
from dual
connect by rownum <= 6)
, t as
(select p.*,
-- the number of points within the distance of 5 by x coordinate
-- cannot be solved with analytic functions for more than one coordinate
count(*) over(order by x range between 5 preceding and 5 following) cnt,
-- sum of the distances to the point (3, 3) for all rows
-- between unbounded preceding and current row ordered by id
-- cannot be solved using analytic functions if required to calculate
-- distance between other rows and current row rather than a constant point
round(sum(sqrt((x - 3) * (x - 3) + (y - 3) * (y - 3)))
over(order by id),
2) dist
from points p)
select t.*,
(select count(*)
from t t0 where t0.x between t.x-5 and t.x + 5) cnt1,
(select count(*)
from t t0 where t0.x between t.x-5 and t.x + 5 and t0.y between t.y-1 and t.y + 1) cnt2,
(select round(sum(sqrt((x - 3) * (x - 3) + (y - 3) * (y - 3))), 2)
from t t0 where t0.id <= t.id) dist1,
(select round(sum(sqrt((x - t.x) * (x - t.x) + (y - t.y) * (y - t.y))), 2)
from t t0 where t0.id <= t.id) dist2
from t
order by id;
ID    X    Y  CNT    DIST     CNT1     CNT2    DIST1    DIST2
---- ---- ---- ---- ------- ------- -------- --------- --------
1    1    1    2    2.83        2        2     2.83        0
2    4    2    3    4.24        3        2     4.24     3.16
3    9    0    2   10.95        2        1    10.95    13.45
4   16    1    1    24.1        1        1     24.1    34.11
5   25    2    1   46.13        1        1    46.13     70.2
6   36    0    1   79.26        1        1    79.26   125.28
6 rows selected.
清单 3-6
分析函数的限制
```

因此，值 `cnt2` 和 `dist2` 无法使用分析函数计算。

还值得一提的是，如果排序键的类型不支持算术运算，则无法使用逻辑偏移（`range`）。显然，物理偏移（`rows`）没有此限制。

大多数分析函数也可以充当聚合函数（如果未指定 `over`）；然而，其中一些是纯粹的分析函数，例如 `row_number` 或 `rank`。如前所述，`order by` 对于此类函数是必需的。

`listagg` 是分析函数的一个特例。首先，它不是可交换的，这意味着第一个值和第二个值的连接与第二个值和第一个值的连接不同，这与 `sum` 或 `average` 不同。其次，不能在分析子句中指定 `order by`。第三，无法在函数中使用 `distinct` 关键字。`listagg` 和 UDF `stragg`（源代码可在 AskTom 上找到）之间的一些差异如清单 3-7 所示。

```sql
with t as
(select rownum id, column_value value
from table(sys.odcinumberlist(2, 1, 1, 3, 1))),
t0 as
(select t.*, row_number() over(partition by value order by id) rn from t)
select t1.*,
(select listagg(value, ',') within group(order by value)
from t t_in
where t_in.id <= t1.id) cumul_ord
from (select t0.*,
listagg(value, ',') within group(order by value) over() list_ord,
listagg(decode(rn, 1, value), ',') within group(order by value) over() dist_ord,
stragg(value) over(order by id) cumul,
stragg(distinct value) over() dist,
stragg(decode(rn, 1, value)) over(order by id) cumul_dist
from t0) t1
order by id;
ID VALUE  RN  LIST_ORD   DIST_ORD  CUMUL     DIST   CUMUL_DIST CUMUL_ORD
---- -----  --  ---------  --------  -------   -----  ---------- ---------
1     2   1  1,1,1,2,3  1,2,3     2         1,2,3  2          2
2     1   1  1,1,1,2,3  1,2,3     2,1       1,2,3  2,1        1,2
3     1   2  1,1,1,2,3  1,2,3     2,1,1     1,2,3  2,1        1,1,2
4     3   1  1,1,1,2,3  1,2,3     2,1,1,3   1,2,3  2,1,3      1,1,2,3
5     1   3  1,1,1,2,3  1,2,3     2,1,1,3,1 1,2,3  2,1,3      1,1,1,2,3
清单 3-7
listagg 与 stragg 的差异
```

简而言之，`listagg` 无法使用窗口排序获得累积连接。另一方面，可以为 `stragg` 指定窗口排序，但在这种情况下无法为结果指定连接顺序。

因此，如果目标是使用窗口排序连接值并指定结果本身的排序，那么使用分析函数和单次表扫描是无法实现的。在上面的例子中，它是使用标量子查询计算的。

重要的一点是，分析函数并非万能药。有时使用连接可能更高效。让我们考虑以下情况。由 `batch_id` 标识的数据批次被写入一个带有 `batch_id` 索引的流表中。我们的目标是计算最后一个 `batch_id` 的 `sum(value)`。参见清单 3-8。

```sql
create table stream as
select batch_id, value
from (select rownum value from dual connect by rownum <= 10000) x1,
(select rownum batch_id from dual connect by level <= 1000)
order by 1, 2;
create index stream_batch_id_idx on stream(batch_id);
exec dbms_stats.gather_table_stats(user, 'stream');
alter session set statistics_level = all;
select sum(s.value)
from stream s
where batch_id = (select max(s0.batch_id) from stream s0);
select * from table(dbms_xplan.display_cursor(null,null,'IOSTATS LAST'));
select sum(value)
from (select s.*, dense_rank() over(order by batch_id) drnk from stream s)
where drnk = 1;
select * from table(dbms_xplan.display_cursor(null,null,'IOSTATS LAST'));
清单 3-8
不同方法：分析函数 vs 连接
```

执行计划如清单 3-9 所示（出于格式化目的，Name 和 Starts 列已被省略）。因此，在 `where` 子句中使用标量子查询（需要额外连接）的版本比使用分析函数的版本快 35 倍 `–` 0.09 秒 vs 3.48 秒。分析查询的大部分时间花费在排序上 `–` 3.47 `–` 1.40 = 2.07 秒，更不用说逻辑读取次数增加了 400 多倍。

```
| Id  | Operation                     | E-Rows | A-Rows |   A-Time   | Buffers | Reads |
```



##### 分析函数与连接查询

###### 执行计划对比

```
|  0  | SELECT STATEMENT              |        |      1 |00:00:00.09 |      43 |    40 |
|  1  |  SORT AGGREGATE               |      1 |      1 |00:00:00.09 |      43 |    40 |
|  2  |   TABLE ACCESS BY INDEX ROWID |  10000 |  10000 |00:00:00.09 |      43 |    40 |
|* 3  |    INDEX RANGE SCAN           |  10000 |  10000 |00:00:00.06 |      25 |    22 |
|  4  |     SORT AGGREGATE            |      1 |      1 |00:00:00.05 |       3 |     3 |
|  5  |      INDEX FULL SCAN (MIN/MAX)|      1 |      1 |00:00:00.05 |       3 |     3 |
```

```
| Id  | Operation                   | E-Rows  | A-Rows  |   A-Time   | Buffers | Reads |
|  0  | SELECT STATEMENT            |         |      1  |00:00:03.48 |   17823 | 17820 |
|  1  |  SORT AGGREGATE             |      1  |      1  |00:00:03.48 |   17823 | 17820 |
|* 2  |   VIEW                      |     10M |  10000  |00:00:03.47 |   17823 | 17820 |
|* 3  |    WINDOW SORT PUSHED RANK  |     10M |  10001  |00:00:03.47 |   17823 | 17820 |
|  4  |     TABLE ACCESS FULL       |     10M |     10M |00:00:01.40 |   17823 | 17820 |
```

**清单 3-9**
分析函数与连接查询：执行计划

###### 性能考虑

在某些情况下，由分析查询引起的排序操作可能非常低效，以至于即使没有任何索引，使用额外连接的方法也可能具有更好的性能。尽管这种情况相当罕见，但考虑不同的方法来获取所需的结果集总是有意义的。

###### 函数的差异与互换性

本节并非旨在描述`row_number`和`rank`之间，或`rank`和`dense_rank`之间的差异。相反，我们将考虑如何利用不同函数的特点和窗口子句，来实现相同的逻辑。

有时你可能会遇到如**清单 3-10**所示的代码。

```sql
max(version) over (partition by dt order by version
rows between unbounded preceding and unbounded following) latest_version
```
**清单 3-10**
使用无界范围的`order by`

在这种情况下，指定`order by`没有任何意义，因为每一行的窗口都是整个分区。因此，**清单 3-11**展示了逻辑上相同的表达式。

```sql
max(version) over (partition by dt) latest_version
```
**清单 3-11**
按分区求最大值

然而，有时即使窗口是整个分区，指定`order by`也是有意义的。

考虑以下任务：对于每一行，我们需要得出对应最大日期的最大值。这在下面的`m2`和`m3`表达式中得以实现。

```sql
with t(id, value, dt, part) as
(
select 1, 10, date '2015-07-01', 1 from dual
union all select 2, 3, date '2015-08-01', 1 from dual
union all select 3, 2, date '2015-09-01', 1 from dual
union all select 4, 0, date '2016-11-01', 1 from dual
union all select 5, 5, date '2016-11-01', 1 from dual
union all select 6, 9, date '2017-01-01', 1 from dual
union all select 7, 4, date '2017-01-01', 1 from dual
)
select
t.*,
max(value) over (partition by part) m1,
max(value) keep (dense_rank last order by dt) over (partition by part) m2,
last_value(value)
over (partition by part order by dt, value
rows between unbounded preceding and unbounded following) m3,
max(value)
over (partition by part order by dt, value
rows between unbounded preceding and unbounded following) m4
from t
order by id ;
ID           VALUE DT        PART   M1   M2   M3   M4
---------- ---------- -------- ---------- ---- ---- ---- ----
1         10 01.07.15          1   10    9    9   10
2          3 01.08.15          1   10    9    9   10
3          2 01.09.15          1   10    9    9   10
4          0 01.11.16          1   10    9    9   10
5          5 01.11.16          1   10    9    9   10
6          9 01.01.17          1   10    9    9   10
7          4 01.01.17          1   10    9    9   10
7 rows selected.
```
**清单 3-12**
最大日期对应的最大值

因此，最大日期是`01.01.2017`，它有两个对应的值——`4`和`9`。结果可以通过在关键字`keep`之后指定`"last"`函数并按`dt`排序，或者通过使用`"last_value"`函数并按`dt`和`value`排序来计算。

如果我们需要获取最小值，则可以使用`min`函数代替`max`，或者简单地在`last_value`函数中为`value`指定降序排序。

```sql
min(value) keep (dense_rank last order by dt) over (partition by part) m2,
last_value(value)
over (partition by part order by dt, value desc
rows between unbounded preceding and unbounded following) m3
from t
```
**清单 3-13**
最大日期对应的最小值

所以，在第一种情况下我们使用了另一个函数，而在第二种情况下只改变了按`value`的排序方向。

最后一个例子突出了`"last_value"`函数和`"ignore nulls"`构造的具体特点。在 10g 之前无法指定`"ignore nulls"`，但解决方法非常直接。

```sql
with t(id, value, part) as
(
select 1, null, 1 from dual
union all select 2, 'one', 1 from dual
union all select 3, null, 1 from dual
union all select 1, 'two', 2 from dual
union all select 2, null, 2 from dual
union all select 3, null, 2 from dual
union all select 4, 'three', 2 from dual
)
select t.*, max(value) over(partition by part, cnt) lv0
from (select t.*,
last_value(value ignore nulls) over(partition by part order by id) lv,
count(value) over(partition by part order by id) cnt
from t
order by part, id) t;
ID VALUE       PART LV           CNT LV0
---------- ----- ---------- ----- ---------- ---
1                1                0
2 one            1 one            1 one
3                1 one            1 one
1 two            2 two            1 two
2                2 two            1 two
3                2 two            1 two
4 three          2 three          2 three
7 rows selected.
```
**清单 3-14**
`last_value` + `ignore nulls`及针对旧版本的变通方法

我们在内联视图中使用`count`来构建分区，该分区包含当前值及后续所有空白值的行，然后在其上使用`max`函数来模拟`last_value` + `ignore nulls`的行为。因此，显然，具有完全不同目的的函数可用于实现相同的逻辑。

##### 总结

分析函数是一个非常强大的工具，可用于获取原本需要自连接或子查询的结果。它们自 Oracle 8i 引入以来已经显著发展；然而，包括 Oracle 12c 在内的许多版本中，它们的能力仍在持续发展。Oracle 提供了灵活的窗口子句定义来调整分析窗口的默认定义，这个特性有其自身的局限性，但对于大多数实际任务来说，内置的灵活性已经绰绰有余。


