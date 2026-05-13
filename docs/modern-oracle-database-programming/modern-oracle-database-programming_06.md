# SQL 窗口函数：lag、lead 与排名分析

#### 引言

默认情况下，`lag`函数访问前一行的值。偏移量为一行。`lead`函数同样如此，偏移量为一行，但它检索当前行之后一行的值。这两个函数都可以接受第二个参数来更改行偏移量。当需要回看两行时，可以使用以下语句。

```sql
lag (ltm.milliseconds, 2)
over (partition by ltm.driverid
order by ltm.lap)
```

前面结果集中的第一行在`diff`列没有值，这是可以解释的，因为没有前一行。该函数超出了窗口的范围。默认情况下，当出现这种情况时，返回值是`NULL`。第三个参数可以覆盖此行为；这样，就可以区分`NULL`值（指向前一行值为`NULL`的行）和分析函数指向窗口外行的情况。可以通过以下表达式修改前面的示例，通过将默认值设置为第一圈的圈速，在 diff 列中显示适当的值。

```sql
lag (ltm.milliseconds, 1, ltm.milliseconds)
over (partition by ltm.driverid
order by ltm.lap)
```

这将产生以下结果。

```
LAP MILLISECONDS       DIFF
---------- ------------ ----------
1       100236          0
2        97880      -2356
3        98357        477
4        98566        209
```

#### 最快圈速与最慢圈速

与聚合函数不同，分析函数不会减少行数。同时显示所有圈速并标记最快和最慢的圈速，使用分析函数很容易做到，无需自连接。最快的圈速是圈速最低的那一圈，这需要`min`函数，而最慢的圈速需要`max`函数。

以下示例使用`case`表达式将当前行的圈速与整体最快圈速进行比较。当圈速匹配时，显示单词 *F A S T*。对最慢圈速也做了类似处理。

由于结果集对比赛（巴林大奖赛）和车手（夏尔·勒克莱尔）进行了筛选，因此在分析函数中无需使用分区子句。

```sql
select ltm.lap
, ltm.milliseconds
, case
when min (ltm.milliseconds) over ()
= ltm.milliseconds
then 'F A S T'
end as fastest
, case
when max (ltm.milliseconds) over ()
= ltm.milliseconds
then 'S L O W'
end as slowest
from   f1data.laptimes ltm
join   f1data.races rcs
on   rcs.raceid = ltm.raceid
where  rcs.race_date between trunc (sysdate, 'yy') and sysdate
and    ltm.driverid = 844 -- Charles Leclerc
and    rcs.raceid = 1074 -- Bahrain Grand Prix
/
LAP MILLISECONDS FASTEST SLOWEST
---------- ------------ ------- -------
1        99070
2        97853
3        98272
4        98414
5        98471
6        98712
7        98835

46       125151
47       163846         S L O W
48       151364
49       146221
50       144429
51        94570 F A S T
52        95027
53        95127

```

#### 排名：Top-N

谈论比赛数据离不开排名，因为每场比赛都以前三名登上领奖台结束。三种不同的分析函数处理排名，其差异很细微。对相等值的处理方式不同。

有三种排名函数：

*   `rank`表示相等的值被分配相同的排名。下一个排名会跳过相等值的数量，这会导致非连续的排名。这种排名被称为 *奥林匹克排名法*。

*   `dense_rank`表示相等的值被分配相同的排名。下一个排名不会跳过数值；它创建连续的排名。

*   `row_number`忽略重复值，其工作方式类似于`rownum`，创建连续的排名。

在该示例中，对每位车手的总积分（不考虑赛季或比赛）进行排名。`DRIVERSTANDINGS`表包含车手累积的所有积分。行按`DRIVERID`分区，并按积分降序排序。结果仅显示了积分为 190 分的塞尔吉奥·佩雷斯。

`rank()`分析函数（下面结果集中的 rk 列）将前三行排名为 1，而第四行的值为 4。`dense_rank()`函数（dr 列）也将前三行的值分配为 1，而第四行的值为 2。最后，`row_number()`函数（rn 列）为每一行分配任意值。

```sql
select drv.forename ||' '|| drv.surname as driver
, drs.points
, rank () over win rk
, dense_rank () over win dr
, row_number() over win rn
from   f1data.driverstandings drs
join   f1data.drivers drv
on   drv.driverid = drs.driverid
where  drv.driverid  = 815
window win as (partition by drv.driverid
order by drs.points desc)
/
DRIVER              POINTS         RK         DR         RN
--------------- ---------- ---------- ---------- ----------
Sergio Pérez           190          1          1          1
Sergio Pérez           190          1          1          2
Sergio Pérez           190          1          1          3
Sergio Pérez           178          4          2          4
Sergio Pérez           165          5          3          5
```

#### 数据去重

当需要对数据进行去重时，`row_number()`排名函数非常有用。在以下示例中，基于`DRIVERS`表创建了一个新表`DUP_DRIVERS`，并且其中的数据被复制了。

```sql
create table dup_drivers
as
select *
from f1data.drivers
/
insert into dup_drivers
select *
from dup_drivers
/
commit
/
```

目前，此表中有超过 1700 名车手。

```sql
select count(*)
from dup_drivers
/
COUNT(*)

```

使用`row_number()`排名函数进行去重的思路是：基于重复值（如`driverid`）创建分区，为该分区中的每一行分配一个任意的排名，并删除所有排名大于 1 的记录。

第一步，数据集按`driverid`分区，并使用`row_number()`为每一行分配排名（仅显示八行）。在这种情况下，分区内行的顺序并不重要。分析函数中的排序反映了这一点（`order by null`）。

```sql
select rowid as rid
, forename ||' '|| surname as driver
, row_number() over
(partition by driverid
order by null
) as rn
from   dup_drivers
fetch  first 8 rows only
/
RID                DRIVER                  RN
------------------ --------------- ----------
AAAYwqAAAAABAB7AAA Lewis Hamilton           1
AAAYwqAAAAABAEHAAA Lewis Hamilton           2
AAAYwqAAAAABAB7AAB Nick Heidfeld            1
AAAYwqAAAAABAEHAAB Nick Heidfeld            2
AAAYwqAAAAABAB7AAC Nico Rosberg             1
AAAYwqAAAAABAEHAAC Nico Rosberg             2
AAAYwqAAAAABAB7AAD Fernando Alonso          1
AAAYwqAAAAABAEHAAD Fernando Alonso          2
8 rows selected.
```

下一步是过滤掉所有排名大于 1 的记录。因为分析函数不能在最终谓词（where 子句）中使用，所以需要将前面的查询推入到一个内联视图中。

```sql
select rid
, driver
from   (select rowid as rid
, forename ||' '|| surname as driver
, row_number() over
(partition by driverid
order by null
) as rn
from   dup_drivers
fetch  first 8 rows only)
where  rn > 1
/
RID                DRIVER
------------------ ---------------
AAAYwqAAAAABAEHAAA Lewis Hamilton
AAAYwqAAAAABAEHAAB Nick Heidfeld
AAAYwqAAAAABAEHAAC Nico Rosberg
AAAYwqAAAAABAEHAAD Fernando Alonso
```

最后一步是从表中删除这些行。查询中移除了对前八行的限制。

```sql
delete
from   dup_drivers
where  rowid in (
select rid
from   (select rowid as rid
, row_number() over
(partition by driverid
order by null
) as rn
from   dup_drivers)
where  rn > 1)
/
854 rows deleted.
```


#### 可重用窗口子句

自 Oracle Database 21c 起，可以定义可重用的命名窗口子句。命名窗口子句允许您在查询中为多个分析函数使用相同的定义，而无需复制该定义。命名窗口子句还可以包含其他命名窗口。

在示例 2-2 中，定义了五个命名窗口。第一个窗口 (`w_part`) 按 `driverid` 定义分区。第二个窗口使用 `w_part` 并按积分降序对数据排序。第三、第四和第五个窗口使用排序后的分区，每个窗口分别使用 `rows`、`range` 和 `groups` 偏移量指定了不同的窗口子句。

```sql
select points
, whole_partition
, rows_
, groups_
, range_
from (
select drv.driverid
, drs.points
, max (drs.points) over w_part  as whole_partition
, max (drs.points) over w_rows  as rows_
, max (drs.points) over w_group as groups_
, max (drs.points) over w_range as range_
from   f1data.driverstandings drs
join   f1data.drivers drv
on   drv.driverid = drs.driverid
window w_part  as (partition by drv.driverid)
,w_sort  as (w_part order by drs.points desc)
,w_rows  as (w_sort rows   between 10 preceding
and current row)
,w_group as (w_sort groups between 10 preceding
and current row)
,w_range as (w_sort range  between 10 preceding
and current row)
)
where  driverid = 815
and    rownum <= 20
order  by driverid, points desc
/
```

```
POINTS WHOLE_PARTITION      ROWS_    GROUPS_     RANGE_
---------- --------------- ---------- ---------- ----------
190             190        190        190        190
190             190        190        190        190
190             190        190        190        190
178             190        190        190        178
165             190        190        190        165
150             190        190        190        150
147             190        190        190        150
135             190        190        190        135
129             190        190        190        135
129             190        190        190        135
125             190        190        190        135
125             190        190        190        135
120             190        190        190        129
118             190        178        190        125
110             190        165        190        120
108             190        150        178        118
104             190        147        165        110
104             190        135        165        110
104             190        129        165        110
104             190        129        165        110
20 rows selected.
```

示例 2-2
可重用窗口子句

`rows` 窗口子句使用物理偏移量。前面的示例显示，在窗口内最多包含当前行之前的十行。`range` 窗口子句使用逻辑偏移量。值小于当前行值 10 以内的数据都在范围内。`groups` 窗口子句将值分组，其中具有相同值的行位于同一组中。在前面的示例中，偏移量是指当前行之前的组数。

注意

在前面的示例中，使用了语法 `over <window_name>`。也可以使用括号括住窗口名：`over (<window_name>)`。两者并不等效。后者意味着复制并修改窗口规范，如果引用的窗口规范包含窗口子句，则会被拒绝。

### 透视和逆透视

最终用户并不总是希望看到数据在页面上纵向排列；他们更喜欢数据横向展示。`pivot` 子句是一个聚合运算符，可将行转换为列。类似地，有时相反的操作更有意义，`unpivot` 子句可以协助完成此操作。

### 透视

对于示例，使用 `results` 表。该表是反规范化的，包含有关车手、车队、所得积分和每场比赛完成圈数的信息。以下是该表的一部分摘录。

```sql
select rcs.year
, drv.driverref
, rsl.points
, rsl.laps
from   f1data.results rsl
join   f1data.races rcs
on   rcs.raceid = rsl.raceid
join   f1data.drivers drv
on   drv.driverid = rsl.driverid
where  drv.driverref in ('norris', 'bottas')
/
```

```
YEAR DRIVERREF      POINTS       LAPS
---------- ---------- ---------- ----------
2013 bottas              0         57
2013 bottas              0         56
2013 bottas              0         56
2013 bottas              0         57
2013 bottas              0         52
2022 norris              2         51
2022 bottas              0         50
2022 bottas              6         70
2022 norris              0         70
2022 norris              8         52
2022 bottas              0         20
```

目标是汇总 Norris 和 Bottas 每年的所有积分，并将两位车手的结果并排显示。

对于这个初始需求，只需要 `year`、`driverref` 和 `points`。前面数据集中的 `laps` 现在可以忽略。前面的查询是 `pivot` 子句的输入。聚合函数是 `sum`（用于汇总所有积分）。聚合函数和透视列需要在 `pivot` 子句中指定。透视列不是动态的。

在下面的查询中，输入查询已被提取到一个命名的子查询（称为 `results`）中，并对车手 Norris 和 Bottas 的积分进行了汇总。属于输入查询的一部分但未在 `pivot` 子句中使用的列（如 `year`）可以被视为“分组依据”列。所有属于输入查询的一部分但未在 `pivot` 子句中使用的列都隐式用作“分组依据”列。

提示

即使只使用单个表，在透视结果集时也建议使用内联视图或子查询因子化（`with` 子句）。这样，如果基础表或视图新增了列，也能保障透视操作的安全性。

```sql
with results as (
select rcs.year
, drv.driverref
, rsl.points
from   f1data.results rsl
join   f1data.races rcs
on   rcs.raceid = rsl.raceid
join   f1data.drivers drv
on   drv.driverid = rsl.driverid
where  drv.driverref in ('norris', 'bottas')
)
select year
, norris
, bottas
from   results
pivot  (
sum (points)
for driverref in ( 'norris' as norris
, 'bottas' as bottas
)
)
/
```

```
YEAR     NORRIS     BOTTAS
---------- ---------- ----------
2013                     4
2014                   186
2015                   136
2016                    85
2017                   305
2018                   247
2019         49        326
2020         97        223
2021        160        219
2022         54         44
```

透视列被赋予了别名，以防止列名变为区分大小写的形式，即列名会是包含引号的 `‘norris’`。

`pivot` 子句中可以有多个聚合函数。当还需要汇总完成圈数时，`laps` 必须包含在输入查询中，并且完成圈数的聚合函数必须包含在 `pivot` 子句中。



## 3. 连接

## 透视（PIVOT）

```
with results as (
select rcs.year
, drv.driverref
, rsl.points
, rsl.laps --<----- 输入中添加了圈数
from   f1data.results rsl
join   f1data.races rcs
on   rcs.raceid = rsl.raceid
join   f1data.drivers drv
on   drv.driverid = rsl.driverid
where  drv.driverref in ('norris', 'bottas')
)
select year
, norris_points
, norris_laps
, bottas_points
, bottas_laps
from   results
pivot  ( sum (points) as points
, sum (laps)   as laps
for driverref in ( 'norris' as norris
, 'bottas' as bottas
)
)
order  by year
/
YEAR NORRIS_POINTS NORRIS_LAPS BOTTAS_POINTS BOTTAS_LAPS
---------- ------------- ----------- ------------- -----------
2013                                       4        1070
2014                                     186        1110
2015                                     136        1037
2016                                      85        1186
2017                                     305        1168
2018                                     247        1202
2019            49        1102           326        1233
2020            97        1015           223         994
2021           160        1223           219        1134
2022            54         570            44         541
10 行 已选择。
```

聚合函数也被赋予别名，以区分它们所代表的值。列名来源于透视列，而别名则来源于聚合。

## 逆透视（UNPIVOT）

如果说透视（pivot）是将行变为列的操作，那么逆透视（unpivot）则做相反的事情。`排位赛`表有三列，分别对应每次排位。排位赛在比赛前一天进行，用于确定发车顺位以及谁获得杆位。每一轮排位都是淘汰赛，只有最快的车手进入下一轮。以下是罗伯特·多恩博斯（Robert Doornbos）的排位赛表摘录；他只进入过两次第二轮，一次进入最终轮。

```
select drv.forename ||' '||drv.surname as driver
, rcs.name
, qlf.q1
, qlf.q2
, qlf.q3
from   f1data.qualifying qlf
join   f1data.drivers drv
on   drv.driverid = qlf.driverid
join   f1data.races rcs
on   rcs.raceid = qlf.raceid
where  drv.driverref = 'doornbos'
and    qlf.q1 is not null
/
DRIVER          NAME                 Q1       Q2       Q3
--------------- -------------------- -------- -------- --------
Robert Doornbos Chinese Grand Prix   1:46.387 1:45.747 1:48.021
Robert Doornbos Japanese Grand Prix  1:32.402
Robert Doornbos Brazilian Grand Prix 1:12.530 1:12.591
Robert Doornbos German Grand Prix    1:18.313
Robert Doornbos Hungarian Grand Prix 1:25.484
Robert Doornbos Italian Grand Prix   1:24.904
Robert Doornbos Belgian Grand Prix   1:49.779
Robert Doornbos Japanese Grand Prix  1:52.894
Robert Doornbos Chinese Grand Prix   1:39.460
9 行 已选择。
```

目标是将三个排位时间列（`Q1`、`Q2` 和 `Q3`）转换为行。

输入查询与前面的查询相同，只是现在作为一个命名的查询块。需要逆透视的列必须包含在输入查询中，本例中就是三个排位列。在逆透视子句中，新列被标记为 `qtime`，它包含三个排位列逆透视后的值，而 `quali_round` 标签表示它所代表的轮次。

```
with qualies as (
select drv.forename ||' '||drv.surname as driver
, rcs.name
, qlf.q1
, qlf.q2
, qlf.q3
from   f1data.qualifying qlf
join   f1data.drivers drv
on   drv.driverid = qlf.driverid
join   f1data.races rcs
on   rcs.raceid = qlf.raceid
where  drv.driverref = 'doornbos'
and    qlf.q1 is not null
)
select driver
, name
, quali_round
, qtime
from   qualies
unpivot (
qtime for quali_round in (q1, q2, q3)
)
/
DRIVER          NAME                 QU QTIME
--------------- -------------------- -- --------
Robert Doornbos Chinese Grand Prix   Q1 1:46.387
Robert Doornbos Chinese Grand Prix   Q2 1:45.747
Robert Doornbos Chinese Grand Prix   Q3 1:48.021
Robert Doornbos Japanese Grand Prix  Q1 1:32.402
Robert Doornbos Brazilian Grand Prix Q1 1:12.530
Robert Doornbos Brazilian Grand Prix Q2 1:12.591
Robert Doornbos German Grand Prix    Q1 1:18.313
Robert Doornbos Hungarian Grand Prix Q1 1:25.484
Robert Doornbos Italian Grand Prix   Q1 1:24.904
Robert Doornbos Belgian Grand Prix   Q1 1:49.779
Robert Doornbos Japanese Grand Prix  Q1 1:52.894
Robert Doornbos Chinese Grand Prix   Q1 1:39.460
12 行 已选择。
```

默认情况下，`NULL` 列会从结果集中排除。输出中没有 `Q2` 或 `Q3` 为 `NULL` 的行。可以通过在逆透视子句后声明 `include nulls` 来覆盖此行为。

##### 处理 NULL 值

```
with qualies as (
select drv.forename ||' '||drv.surname as driver
, rcs.name
, qlf.q1
, qlf.q2
, qlf.q3
from   f1data.qualifying qlf
join   f1data.drivers drv
on   drv.driverid = qlf.driverid
join   f1data.races rcs
on   rcs.raceid = qlf.raceid
where  drv.driverref = 'doornbos'
and    qlf.q1 is not null
)
select driver
, name
, quali_round
, qtime
from   qualies
unpivot include nulls (
qtime for quali_round in (q1, q2, q3)
)
/
DRIVER          NAME                 QU QTIME
--------------- -------------------- -- --------
Robert Doornbos Chinese Grand Prix   Q1 1:46.387
Robert Doornbos Chinese Grand Prix   Q2 1:45.747
Robert Doornbos Chinese Grand Prix   Q3 1:48.021
Robert Doornbos Japanese Grand Prix  Q1 1:32.402
Robert Doornbos Japanese Grand Prix  Q2
Robert Doornbos Japanese Grand Prix  Q3
Robert Doornbos Brazilian Grand Prix Q1 1:12.530
Robert Doornbos Brazilian Grand Prix Q2 1:12.591
Robert Doornbos Brazilian Grand Prix Q3

```

### 总结

本章讨论了分析函数及其最常见的应用场景。一旦开始使用分析函数，你会发现更多的可能性。这里也存在一个警示：分析函数是在结果集确定之后（`连接`、`where`、`group by`、`having`）和最终排序之前（`order by`）应用的。当你在分析函数中使用 `distinct` 操作符时，可能你需要的是聚合函数而不是分析函数。

本章最后讨论了结果集的透视（pivoting）和逆透视（unpivoting）。要透视结果，需要在 `pivot` 子句中聚合度量值。使用 `pivot`，行变为列；使用 `unpivot`，列变为行。

## 3. 连接

与其他数据库开发人员讨论连接语法几乎总是会引向关于首选数据连接方式的讨论。有些人是 `ANSI` 风格连接的狂热支持者，而另一些人则倾向于坚持传统方式。我们偏爱的方法是前者——`ANSI` 风格。

本章讨论了使用 `ANSI` 风格连接的优势，并描述了最常见的连接：`外连接`、`内连接`和`全连接`。它还涵盖了一些不太为人所知的连接，如 `分区外连接` 和 `横向连接`。当提到表时，指的是可以在查询中使用的所有类型的数据源，例如视图或物化视图。


### 为什么选择 ANSI 连接？

使用 ANSI 风格连接语法的主要优势在于将连接条件与过滤条件分离。传统语法会将连接条件与过滤条件混在一起，而使用 ANSI 风格则完全避免了这种情况。连接条件指定了表如何关联，而过滤条件则位于 `where` 子句中。

传统的表连接方式是在 `from` 子句中列出表，并在 `where` 子句中指定连接条件。当数据需要额外过滤时，这些过滤条件也会进入 `where` 子句。这可能导致混淆，无法确定 `where` 子句中的条件究竟是连接条件还是过滤条件。

ANSI 风格的语法强制要求将连接条件指定在 `on` 子句或 `using` 子句中。而过滤条件则始终放在 `where` 子句中。

在传统风格中，如果忘记指定连接条件，会导致产生笛卡尔积。

使用 ANSI 风格语法则不会意外产生笛卡尔积。当传统风格的代码确实意图创建笛卡尔积时，需要添加适当的注释，以告知后续开发者这个笛卡尔积是故意为之。而 ANSI 风格的笛卡尔积则一目了然：`cross join`。这直接向后续开发者表明该笛卡尔积是故意的，无需额外注释。

使用外连接表时，传统风格不那么直观。外连接符号 `(+)` 应该加在外连接的表上，还是加在另一张表上？必须对外连接的所有列都应用 `(+)` 符号，连接才能保持为外连接。其他数据库厂商的数据库开发者不会熟悉这种 Oracle 风格的外连接语法。

ANSI 风格的外连接更容易理解。要编写一个查询，对表 A 和 B 执行外连接，并返回 A 表（位于连接关键字左侧的表）的所有行，只需在 `from` 子句中指定 `left outer join`。而使用传统风格编写外连接时，必须对外连接操作符 `(+)` 应用于表 B 的所有列，这可能有违直觉。

实际上，底层使用哪种连接风格并不重要。优化器会重写它们，并以相同的方式执行。

无论你偏好哪种连接风格，一个共识是：不应在同一个查询中混合使用这两种风格。虽然这是可能的，但被认为是不良实践。最常见的是，所有数据库开发者在项目开始时就商定好查询风格。

### 自然连接

自然连接是指连接条件由两表中列的 `名称` 推导而来。这种设计堪称 ANSI 标准中的一个败笔，不应被使用。

以下用 `DRIVERS`、`RESULTS` 和 `CONSTRUCTORS` 表来演示自然连接的荒谬之处。使用自然连接组合这些表，并执行以下查询：

```
select ctr.name
,drv.forename
,drv.surname
from f1data.drivers drv
natural
join f1data.results rst
natural
join f1data.constructors ctr
/
no rows selected
```

该查询没有返回任何结果，检查执行计划后，原因一目了然。

```
explain plan for
select ctr.name
, drv.forename
, drv.surname
from   f1data.drivers drv
natural
join   f1data.results rst
natural
join   f1data.constructors ctr
/
select *
from   table (dbms_xplan.display (format => 'BASIC +PREDICATE' ))
/

| Id  | Operation                    | Name         |

|   0 | SELECT STATEMENT             |              |
|*  1 |  HASH JOIN                   |              |
|   2 |   JOIN FILTER CREATE         | :BF0000      |
|*  3 |    HASH JOIN                 |              |
|   4 |     TABLE ACCESS STORAGE FULL| CONSTRUCTORS |
|   5 |     TABLE ACCESS STORAGE FULL| DRIVERS      |
|   6 |   JOIN FILTER USE            | :BF0000      |
|*  7 |    TABLE ACCESS STORAGE FULL | RESULTS      |

Predicate Information (identified by operation id):

1 - access("RST"."CONSTRUCTORID"="CTR"."CONSTRUCTORID" AND
"DRV"."DRIVERID"="RST"."DRIVERID")
3 - access("DRV"."URL"="CTR"."URL" AND
"DRV"."NATIONALITY"="CTR"."NATIONALITY")
7 - storage(SYS_OP_BLOOM_FILTER(:BF0000,"RST"."DRIVERID"))
filter(SYS_OP_BLOOM_FILTER(:BF0000,"RST"."DRIVERID"))
```

从访问谓词信息中可以看到，自然连接执行的连接条件是基于列名的相似性。有些是正确的，比如 constructorid 和 driverid，但其他的则毫无意义，比如 nationality 或 url。重申一次，这个 ANSI 标准中的败笔不应被使用。

### 内连接

将两个表关联起来的最常用方法是内连接。表根据连接条件中指定的某些列进行连接。以下示例展示了 `drivers` 表和 `results` 表之间的内连接。

```
select crt.raceid
, crt.points
, csr.name
, csr.nationality
from   f1data.constructors csr
inner
join   f1data.constructorresults crt
on   csr.constructorid = crt.constructorid
/
RACEID     POINTS NAME            NATIONALITY
---------- ---------- --------------- ---------------
18         14 McLaren         British
18          8 BMW Sauber      German
18          9 Williams        British
18          5 Renault         French
18          2 Toro Rosso      Italian
18          1 Ferrari         Italian
18          0 Toyota          Japanese
18          0 Super Aguri     Japanese
18          0 Red Bull        Austrian
18          0 Force India     Indian
```

在这个查询中，通过使用可选关键字 `inner`，明确指定了这是一个内连接。连接条件在 `on` 子句中指定。也可以使用 `using` 子句，它表示两表之间存在一个同名公共列。`using` 子句会在两个列之间进行相等性比较。而 `on` 子句允许在连接列之间进行其他类型的比较。以下示例与前一个示例等价，但在连接条件中使用了 `using` 子句。

```
select crt.raceid
, crt.points
, csr.name
, csr.nationality
from   f1data.constructors csr
join   f1data.constructorresults crt
using  (constructorid)
/
RACEID     POINTS NAME            NATIONALITY
---------- ---------- --------------- ---------------
18         14 McLaren         British
18          8 BMW Sauber      German
18          9 Williams        British
18          5 Renault         French
18          2 Toro Rosso      Italian
18          1 Ferrari         Italian
18          0 Toyota          Japanese
18          0 Super Aguri     Japanese
18          0 Red Bull        Austrian
18          0 Force India     Indian

```

`using` 子句中指定的、用于连接条件的列，在查询的任何地方都不能使用表别名作为前缀。需要明确的是，这些列是可以使用的；只是不能在前面加上表别名。毕竟，用于连接表的列是表之间的公共列。

### 外连接

外连接会显示一张表中的所有行，无论其在另一张表中是否存在匹配记录。可能有些车队目前还没有任何比赛结果。

使用 ANSI 风格的外连接时，查询中表的顺序很重要。一个`左`外连接会指定显示连接关键字左侧表中的*所有行*，即使它与连接关键字右侧的表没有匹配项。`右`外连接也是同样的道理，但针对的是连接关键字右侧的行。

> 注意
>
> 在网络论坛和各种示例中，书写外连接最常见的方式是使用`左外`连接。这可能是因为主表通常被首先书写。尽管常见，但这并非强制要求，而是一种最佳实践。本章及全书都将使用`左外连接`。

在以下示例中，将显示所有车队，即使它们在`constructorresults`表中没有任何积分。

```sql
select crt.raceid
,      crt.points
,      csr.name
,      csr.nationality
from   f1data.constructors csr
left outer
join   f1data.constructorresults crt
on     csr.constructorid = crt.constructorid
/
RACEID     POINTS NAME            NATIONALITY
---------- ---------- --------------- -----------
18         14 McLaren         British
19         10 McLaren         British
20          4 McLaren         British

Pawl            American
Rae             American
Adams           American
```

> 注意
>
> `outer`和`inner`关键字是可选的，但强烈建议使用它们以使语法更清晰。

## 几乎外连接

“几乎外连接”是指在查询中书写了外连接，但被`where`子句中的额外条件所否定。

在以下示例中，目标是查看所有车队及其车队成绩；但是，如果车队是`红牛`，你还想看到他们的积分。对于其他车队，此信息无关紧要，无需显示。该查询的首次尝试如下。

```sql
select crt.raceid
,      crt.points
,      csr.name
,      csr.nationality
from   f1data.constructors csr
left outer
join   f1data.constructorresults crt
on     csr.constructorid = crt.constructorid
where  csr.constructorref = 'red_bull'
/
RACEID     POINTS NAME            NATIONALITY
---------- ---------- --------------- ---------------
1062          2 Red Bull        Austrian
1063       12.5 Red Bull        Austrian

1072         18 Red Bull        Austrian
1073         26 Red Bull        Austrian
1074          0 Red Bull        Austrian
1075         37 Red Bull        Austrian
1076         18 Red Bull        Austrian
1077         58 Red Bull        Austrian
336 rows selected.
```

如示例所示，查询中使用了外连接，但结果只显示了`红牛`的数据。所有其他车队的成绩都缺失了。这个查询并没有显示先前陈述的目标结果。要获得所需的查询结果集，过滤条件必须是一个连接条件。

```sql
select crt.raceid
,      crt.points
,      csr.name
,      csr.nationality
from   f1data.constructors csr
left outer
join   f1data.constructorresults crt
on     csr.constructorid  = crt.constructorid
and    csr.constructorref = 'red_bull'
/
RACEID     POINTS NAME            NATIONALITY
---------- ---------- --------------- ---------------
1062          2 Red Bull        Austrian
1063       12.5 Red Bull        Austrian

Alpine F1 Team  French
McLaren         British
Martini         French
Pawl            American
Cooper-BRM      British
546 rows selected.
```

现在查询按指定方式显示了结果，仅显示`红牛`车队的积分。

#### 全外连接

数据模型中没有一个可以应用全外连接的合适例子，但为了演示目的，将`RACES`和`CIRCUITS`之间的外键从必需改为可选。其思路是，未来应该可以安排一场比赛，而赛道是未知的。也可能存在一个赛道，但目前还没有为它安排任何比赛。

在每个表中，都添加了一条带有虚构数据的额外记录。

```sql
Insert into f1data.races
( raceid
, year
, round
, circuitid
, name
, race_date
)
values
( 0
, 2023
, 42
, null
, 'Hitchhiker Race'
, date '2023-10-07'
)
/
```

```sql
insert into f1data.circuits
( circuitid
, circuitref
, name
, location
, country
)
values
( 0
, 'brabantbaan'
, 'Brabant Baan'
, 'Oosterhout'
, 'Netherlands')
/
```

在以下示例中，执行了一个全外连接，并对结果进行了过滤，仅显示在前面代码中刚刚添加的行。

```sql
select rce.name      as race
,      rce.race_date
,      cct.name      as circuit
,      cct.location
from   f1data.races rce
full outer
join   f1data.circuits cct
on     rce.circuitid = cct.circuitid
where  rce.rowid is null
or     cct.rowid is null
/
RACE             RACE_DATE  CIRCUIT          LOCATION
---------------  ---------  ---------------  -----------
Hitchhiker Race  07-OCT-23  Brabant Baan     Oosterhout
```

一条来自`RACES`表的记录被显示出来，即使没有赛道来举办该赛事；同时一个没有安排比赛的赛道也被显示出来。

### 交叉连接

在笛卡尔积中，第一张表的所有记录都与第二张表的记录相连接。使用 ANSI 连接语法，可以清楚地看出这是有意为之，而不是意外缺少了连接条件。在 ANSI 连接语法中，这被称为`交叉连接`。以下示例创建了一个车手的笛卡尔积；`drivers`表与自身相连接。为了展示交叉连接的效果，使用了一个计数。

```sql
select count(*)
from   f1data.drivers drv
/
COUNT(*)

```

表中有 854 名车手。当这张表与自身进行笛卡尔积连接时，会生成所有可能的车手组合，结果是 854 × 854 条记录：729316 条记录。

```sql
select count(*)
from   f1data.drivers drv
cross
join   f1data.drivers drv2
/
COUNT(*)

```

由于使用了明确的`交叉连接`语法，下一个阅读此代码的开发者会清楚地知道这个笛卡尔积是特意为之。


### 分区外连接

常规外连接通过向结果集添加空记录来弥补连接条件中缺少匹配行的情况。然而，有时仅添加单行记录是不够的。

本章前面部分，日历中添加了一场额外的比赛。这场比赛没有比赛结果。我们的目标是显示每位车手在这场比赛中的结果，即使没有结果（实际上也没有）。

涉及的表有 `RACES`（比赛）、`RESULTS`（结果）和 `DRIVERS`（车手，用于显示每位车手的姓名）。第一次尝试是将结果表与比赛表和车手表进行外连接。

```sql
select rce.name
, drv.forename||' '||drv.surname as driver
, rst.result_number
from   f1data.races rce
left outer
join   f1data.results rst
on   rce.raceid = rst.raceid
left outer
join   f1data.drivers drv
on   drv.driverid = rst.driverid
where  rce.raceid = 0
/
```
```
NAME            DRIVER          RESULT_NUMBER
--------------- --------------- -------------
Hitchhiker Race
```

这个结果并非我们所求。比赛只有一行记录。没有列出任何车手，而这正是我们要求的。与 `RESULTS` 表的外连接只添加了一行，因为没有匹配项，但它应该为每位车手添加一行。`RESULTS` 表应按 `driverid` 细分并与 `RACES` 表进行外连接；这样，每位车手就有一条记录。通过对表添加 `partition by` 子句，外连接变为*按分区*执行，而非针对整张表。由于分区是按 `driverid` 进行的，每位车手都有一条 `RESULTS` 记录。因为每位车手都有了结果记录，就不再需要对 `drivers` 表进行外连接了。以下查询仅对当前赛季（2022 年）的车手显示分区外连接。

```sql
select rce.name
, drv.forename||' '||drv.surname as driver
, rst.result_number
from   f1data.races rce
left outer
join   f1data.results rst partition by (rst.driverid)
on   rce.raceid = rst.raceid
join   f1data.drivers drv
on   drv.driverid = rst.driverid
where  rce.raceid = 0
and    drv.driverid in ( 1, 4, 830, 815, 847
, 844, 832, 846, 839
, 20, 822, 817, 842
, 855, 854, 825, 840
, 852, 848, 849)
/
```
```
NAME            DRIVER             RESULT_NUMBER
--------------- ------------------ -------------
Hitchhiker Race Lewis Hamilton
Hitchhiker Race Fernando Alonso
Hitchhiker Race Sebastian Vettel
Hitchhiker Race Sergio Pérez
Hitchhiker Race Daniel Ricciardo
Hitchhiker Race Valtteri Bottas
Hitchhiker Race Kevin Magnussen
Hitchhiker Race Max Verstappen
Hitchhiker Race Carlos Sainz
Hitchhiker Race Esteban Ocon
Hitchhiker Race Lance Stroll
Hitchhiker Race Pierre Gasly
Hitchhiker Race Charles Leclerc
Hitchhiker Race Lando Norris
Hitchhiker Race George Russell
Hitchhiker Race Alexander Albon
Hitchhiker Race Nicholas Latifi
Hitchhiker Race Yuki Tsunoda
Hitchhiker Race Mick Schumacher
Hitchhiker Race Guanyu Zhou
```

当记录没有被过滤以仅显示特定车手的数据时，`RESULTS` 表中列出的每位车手都会有一条记录。由于结果表包含了自 1950 年代以来所有比赛的记录，数据量会更大。

### 横向连接

除非使用横向连接，否则无法将表与关联子查询连接。使用横向内联视图，你可以在查询的 `from` 子句中指定出现在横向内联视图左侧的表。

横向连接最常见的用例是使用 `JSON_TABLE` 运算符将 JSON 文档转换为关系格式，或使用 `XML_TABLE` 运算符将 XML 文档进行类似转换。

以下查询从 `driverstandings` 表中选取了一组特定的车手及其积分，但只取积分排名前两位（按积分从高到低排序）且积分大于零的记录。横向内联视图与 `drivers` 表在 `driverid` 列上存在关联。

```sql
select drv.driverid
, drv.forename ||' '||drv.surname as driver
, dsg.points
, dsg.position
from   f1data.drivers drv
cross
join lateral
(select dsg.points
, dsg.position
from   f1data.driverstandings dsg
where  dsg.driverid = drv.driverid
and    dsg.points > 0
order  by dsg.points desc
fetch  first 2 rows only) dsg
where  drv.driverid in ( 855, 854, 840, 852, 848, 849)
order  by driverid desc
/
```
```
DRIVERID DRIVER              POINTS   POSITION
---------- --------------- ---------- ----------
855 Guanyu Zhou              6         17
855 Guanyu Zhou              6         18
854 Mick Schumacher         12         15
854 Mick Schumacher         12         15
852 Yuki Tsunoda            32         14
852 Yuki Tsunoda            20         14
849 Nicholas Latifi          7         17
849 Nicholas Latifi          7         16
848 Alexander Albon        105          7
848 Alexander Albon         93          8
840 Lance Stroll            75         11
840 Lance Stroll            74         10
```

在结果中，从关联子查询中选择了两列。使用标量子查询是无法做到这一点的。标量子查询只能返回一个值。

注意：除了 `cross join lateral`，你也可以使用 `inner join lateral`；然而，内连接还需要一个连接条件（`on` 子句或 `using` 子句）。由于表和内联视图之间已经存在关联，再有连接条件就是多余的，并且无助于提高可读性或可维护性。不推荐使用 `inner join lateral`。

`cross apply` 连接是 `cross join` 的一种变体，与后者不同，它可以使用关联子查询。前面例子的相同结果可以使用 `cross apply` 而非 `cross join lateral` 来实现。在下面的例子中，应用了 `cross apply` 连接。

```sql
select drv.driverid
, drv.forename ||' '||drv.surname as driver
, dsg.points
from   f1data.drivers drv
cross
apply  (select dsg.points
, dsg.driverid
from   f1data.driverstandings dsg
where  dsg.driverid = drv.driverid
and    dsg.points > 0
order  by dsg.points desc
fetch  first 2 rows only
) dsg
where  drv.driverid in ( 855, 854, 840, 852, 848, 849)
order  by driverid desc
/
```
```
DRIVERID DRIVER                 POINTS
---------- ------------------ ----------
855 Guanyu Zhou                 5
855 Guanyu Zhou                 5
854 Mick Schumacher             4
852 Yuki Tsunoda               32
852 Yuki Tsunoda               20
849 Nicholas Latifi             7
849 Nicholas Latifi             7
848 Alexander Albon           105
848 Alexander Albon            93
840 Lance Stroll               75
840 Lance Stroll               74
```

为了演示 `outer apply` 连接，我们向 drivers 表添加了一个额外的车手。

```sql
insert into f1data.drivers
( driverid
, driverref
, driver_number
, code
, forename
, surname
)
values
( 0
, 'alxntn'
, 53
, 'ALX'
, 'Alex'
, 'Nuijten'
)
/
```

`outer apply` 连接是左外连接的一种变体，可以使用关联的内联视图。以下示例显示了与之前相同的结果，但包含了刚刚创建的、没有任何车手积分记录的额外车手（因为他水平没那么好）。




### 概要

本章介绍了 ANSI 连接。将连接条件与过滤条件分开的写法，使得这类连接比传统的表连接方式更加优雅。开发者之间对于哪种方式更好的讨论会一直存在，这通常也取决于个人偏好。请根据你最熟悉的方式选择，但要了解 ANSI 连接的可能性，并知道在需要时如何解读和应用它们。一旦你习惯了 ANSI 连接，就再也不想回去了。

有一点希望持不同意见的双方都能认同：**不要**在同一个查询中混合使用传统语法和 ANSI 连接。

## 4. 发现模式

Oracle Database 12c 引入了行模式匹配功能，可以对数据集中的模式进行分析。`match_recognize`子句实现了这一点。在某种程度上，分析函数也可用于进行一些基本的模式匹配，但这有其局限性。过去在行序列中寻找模式的其他变通方法编写困难，往往难以理解且执行效率低下。`match_recognize`子句使用原生 SQL 解决了这些问题，允许在数据集中进行模式检测。

### 寻找合同

数据模型中没有关于车队与车手之间合同的信息。或许可以通过观察比赛来推断合同。当一名车手为某支车队比赛时，你可以假设他们之间存在合同。

现有的信息包括比赛、车手和车队（制造商）。这里做出的假设是：在一场比赛期间，车手受雇于制造商，合同从赛季第一场比赛开始，到赛季最后一场比赛结束。以下查询检索了历年关于基米·莱科宁的信息。

```
select rcs.year
, ctr.name            as constructor
, min (rcs.race_date) as startrace
, max (rcs.race_date) as endrace
from f1data.results rsl
join f1data.races rcs
on rsl.raceid = rcs.raceid
join f1data.constructors ctr
on ctr.constructorid = rsl.constructorid
where rsl.driverid = 8 -- Kimi Räikkönen
group by rcs.year
,ctr.name
order by rcs.year
/
YEAR CONSTRUCTO STARTRACE ENDRACE
---------- ---------- --------- ---------
2001 Sauber     04-MAR-01 14-OCT-01
2002 McLaren    03-MAR-02 13-OCT-02
2003 McLaren    09-MAR-03 12-OCT-03
2004 McLaren    07-MAR-04 24-OCT-04
2005 McLaren    06-MAR-05 16-OCT-05
2006 McLaren    12-MAR-06 22-OCT-06
2007 Ferrari    18-MAR-07 21-OCT-07
2008 Ferrari    16-MAR-08 02-NOV-08
2009 Ferrari    29-MAR-09 01-NOV-09
2012 Lotus F1   18-MAR-12 25-NOV-12
2013 Lotus F1   17-MAR-13 03-NOV-13
2014 Ferrari    16-MAR-14 23-NOV-14
2015 Ferrari    15-MAR-15 29-NOV-15
2016 Ferrari    20-MAR-16 27-NOV-16
2017 Ferrari    26-MAR-17 26-NOV-17
2018 Ferrari    25-MAR-18 25-NOV-18
2019 Alfa Romeo 17-MAR-19 01-DEC-19
2020 Alfa Romeo 05-JUL-20 13-DEC-20
2021 Alfa Romeo 28-MAR-21 12-DEC-21
```

这个结果集基于基米·莱科宁历年为不同制造商参赛的成绩。他的职业生涯始于索伯车队，随后是迈凯伦、法拉利、路特斯、再次法拉利，最后在阿尔法·罗密欧结束。

目标是对连续的值进行分组，即当制造商在连续多个赛季中保持不变时，将它们折叠成一个单一的组。基米在迈凯伦比赛的那些年（2002 年至 2006 年）将被折叠成一个组。

前面的查询是模式匹配`match_recognize`子句的*输入*。在下面的例子中，它被命名为`races`，并用于子查询因子化子句（`with`子句）中。

我们想要的模式是基于*连续*年份中相同的制造商对记录进行分组。这意味着基米曾为法拉利效力的两个独立时期将被识别出来。

在数据集中寻找连续模式时，排序顺序很重要。在这个例子中，按年份排序是最合理的。这在`match_recognize`子句中通过`order by`来实现：

```
order by year
```




#### 对记录进行分类

在寻找模式之前，必须对每一行数据进行分类。行的分类方式取决于具体需求。以下定义通过比较当前行的构造函数与组中 `first` （第一个）构造函数来进行分类。

```
define eq as constructor = first (constructor)
```

对于结果集中的第一行，组的开始就是当前行（索伯车队）。结果集中的第二行是下一个组的开始（迈凯伦车队）。第三行仍然属于迈凯伦车队组，依此类推。有了这个定义，就可以确定模式了。模式定义如下。

```
pattern (eq+)
```

这意味着一个或多个被分类为“eq”的记录属于同一个组。在模式子句中，可以使用正则表达式来指定需要在数据集中匹配的模式。表 4-1 概述了最常见的正则表达式量词。

表 4-1
正则表达式量词

| 类型 | 描述 |
| --- | --- |
| `*` | 零个或多个 |
| `+` | 一个或多个 |
| `?` | 零个或一个 |
| `{n}` | 恰好 n 个 |
| `{n,}` | 至少 n 个 |
| `{,m}` | 最多 m 个 |
| `{n, m}` | 介于 n 和 m 个之间 |
| `&#124;` | 或；尝试此模式，否则尝试彼模式 |

下面的查询使用内置的 `match_number()` 函数来展示此数据集中的不同分组。

```
with race_years as (
select rcs.year
, ctr.name as constructor
, min (rcs.race_date) as startrace
, max (rcs.race_date) as endrace
from   f1data.results rsl
join   f1data.races rcs
on   rsl.raceid = rcs.raceid
join   f1data.constructors ctr
on   ctr.constructorid = rsl.constructorid
where  rsl.driverid = 8 -- 基米·莱科宁
group  by rcs.year
, ctr.name
)
select rce.mnr
, rce.constructor
, rce.startrace
, rce.endrace
from   race_years
match_recognize (
order by year
measures
match_number() as mnr
all rows per match
pattern (eq+)
define eq as constructor = first (constructor)
) rce
order  by year
/
MNR CONSTRUCTOR               STARTRACE ENDRACE
---------- ------------------------- --------- ---------
1 Sauber                    04-MAR-01 14-OCT-01
2 McLaren                   03-MAR-02 13-OCT-02
2 McLaren                   09-MAR-03 12-OCT-03
2 McLaren                   07-MAR-04 24-OCT-04
2 McLaren                   06-MAR-05 16-OCT-05
2 McLaren                   12-MAR-06 22-OCT-06
3 Ferrari                   18-MAR-07 21-OCT-07
3 Ferrari                   16-MAR-08 02-NOV-08
3 Ferrari                   29-MAR-09 01-NOV-09
4 Lotus F1                  18-MAR-12 25-NOV-12
4 Lotus F1                  17-MAR-13 03-NOV-13
5 Ferrari                   16-MAR-14 23-NOV-14
5 Ferrari                   15-MAR-15 29-NOV-15
5 Ferrari                   20-MAR-16 27-NOV-16
5 Ferrari                   26-MAR-17 26-NOV-17
5 Ferrari                   25-MAR-18 25-NOV-18
6 Alfa Romeo                17-MAR-19 01-DEC-19
6 Alfa Romeo                05-JUL-20 13-DEC-20
6 Alfa Romeo                28-MAR-21 12-DEC-21
```

如结果所示，`MNR` 列显示了 `match_number()` 以识别不同的分组。这是对找到的匹配项进行的连续编号。在前面的查询中，显示了所有行。`all rows per match` 子句返回所有行，因此很容易识别哪些记录被分在了一组。`all rows per match` 子句还确保返回输入数据集的所有列，以及 `measures` 子句中列出的所有内容。

##### 表别名

表别名放在 `match_recognize` 子句的最终括号之后。

现在已经清楚哪些记录应该归为一组，可以修改查询以仅显示 `one row per match` （每个匹配一行）。度量值是在查询其他部分可用的表达式。以下是一些值得关注的度量值。

*   组的构造函数：为此，使用了 `first (constructor)` 表达式。在这种情况下，也可以使用 `last (constructor)`，因为组内的构造函数保持不变。
*   与该构造函数合同的第一年：此度量值的表达式是 `first (startrace)`。
*   与该构造函数合同的最后一年：此度量值的表达式是 `last (endrace)`。

```
    with race_years as (
    select rcs.year
    , ctr.name as constructor
    , min (rcs.race_date) as startrace
    , max (rcs.race_date) as endrace
    from   f1data.results rsl
    join   f1data.races rcs
    on   rsl.raceid = rcs.raceid
    join   f1data.constructors ctr
    on   ctr.constructorid = rsl.constructorid
    where  rsl.driverid = 8
    group  by rcs.year
    , ctr.name
    )
    select rce.constructor
    , rce.strt
    , rce.stp
    from   race_years
    match_recognize (
    order by year
    measures
    first (constructor) as constructor
    , extract (year from first (startrace)) as strt
    , extract (year from last (endrace)) as stp
    one row per match
    pattern (eq+)
    define eq as constructor = first (constructor)
    ) rce
    order by rce.strt
    /
    CONSTRUCTOR                     STRT        STP
    ------------------------- ---------- ----------
    Sauber                          2001       2001
    McLaren                         2002       2006
    Ferrari                         2007       2009
    Lotus F1                        2012       2013
    Ferrari                         2014       2018
    Alfa Romeo                      2019       2021
    6 rows selected.
    ```

使用了 `extract` 函数来获取年份。它只显示年份而非具体日期。

##### 度量值子句的别名设置

度量值子句中的表达式必须始终设置别名，即使该表达式是输入表的列也是如此。


### 逻辑组

与分析函数类似，也可以在 `match_recognize` 子句中对输入数据集进行分区。这样，模式就会应用于每个逻辑数据集组。下一个示例以与之前相同的方式确定 2021 年之后的合同，但这次涵盖所有车手。子查询因子化子句被修改为包含车手的姓名。接着，在 `match_recognize` 子句中添加了下面的分区子句。

```
partition by driver
```

这将为每位车手创建一个逻辑组，并应用相同的模式检测。以下是确定自 2021 年以来每位车手合同的完整查询。

```
with race_years as (
select drv.forename || ' ' || drv.surname as driver
, rcs.year
, ctr.name as constructor
,min (rcs.race_date) as startrace
,max (rcs.race_date) as endrace
from   f1data.results rsl
join   f1data.races rcs
on   rsl.raceid = rcs.raceid
join   f1data.constructors ctr
on   ctr.constructorid = rsl.constructorid
join   f1data.drivers drv
on   drv.driverid = rsl.driverid
where  rcs.year >= 2021
group  by rcs.year
,ctr.name
,drv.forename
,drv.surname
)
select rce.driver
, rce.constructor
, rce.strt
, rce.stp
from   race_years
match_recognize (
partition by driver
order by year
measures
first (constructor) as constructor
, extract (year from first (startrace)) as strt
, extract (year from last (endrace)) as stp
, match_number() as mnr
one row per match
pattern (eq+)
define eq as constructor = first (constructor)
) rce
order  by rce.constructor
, rce.strt
/
DRIVER               CONSTRUCTOR             STRT        STP
-------------------- ----------------- ---------- ----------
Robert Kubica        Alfa Romeo              2021       2021
Antonio Giovinazzi   Alfa Romeo              2021       2021
Kimi Räikkönen       Alfa Romeo              2021       2021
Valtteri Bottas      Alfa Romeo              2022       2022
Guanyu Zhou          Alfa Romeo              2022       2022
Yuki Tsunoda         AlphaTauri              2021       2022
Pierre Gasly         AlphaTauri              2021       2022
Fernando Alonso      Alpine F1 Team          2021       2022
Esteban Ocon         Alpine F1 Team          2021       2022
Sebastian Vettel     Aston Martin            2021       2022
Lance Stroll         Aston Martin            2021       2022
Nico Hülkenberg      Aston Martin            2022       2022
Charles Leclerc      Ferrari                 2021       2022
Carlos Sainz         Ferrari                 2021       2022
Mick Schumacher      Haas F1 Team            2021       2022
Nikita Mazepin       Haas F1 Team            2021       2021
Kevin Magnussen      Haas F1 Team            2022       2022
Lando Norris         McLaren                 2021       2022
Daniel Ricciardo     McLaren                 2021       2022
Valtteri Bottas      Mercedes                2021       2021
Lewis Hamilton       Mercedes                2021       2022
George Russell       Mercedes                2022       2022
Sergio Pérez         Red Bull                2021       2022
Max Verstappen       Red Bull                2021       2022
George Russell       Williams                2021       2021
Nicholas Latifi      Williams                2021       2022
Alexander Albon      Williams                2022       2022
Nyck de Vries        Williams                2022       2022
```

### 末圈更快

每场比赛的耗时记录在 `LAPTIMES` 表中。你常听说比赛末圈的单圈时间更短，因为赛车油量更少，车重更轻。然而，轮胎衰变会导致单圈耗时更长。让我们来验证一下这个理论，看看实际情况如何。

首先，你需要判断一圈是比前一圈慢还是快。以下是“更快”和“更慢”的定义。

```
faster as laptime  prev (laptime)
```

该定义使用 `prev()` 函数将当前单圈时间与前一圈时间进行比较。

在本例中，所有记录都需要被标记为 `FASTER` 或 `SLOWER`，因此使用了 `all rows per match`。使用的模式如下。

```
pattern ( faster* slower* )
```

该模式解读为：零个或多个被标记为 `FASTER` 的记录，后跟零个或多个被标记为 `SLOWER` 的记录。将所有部分组合在一起，得到以下查询。

```
with cota as (
select ltm.lap
, numtodsinterval( ltm.milliseconds / 1000
, 'second') as laptime
from   f1data.laptimes ltm
where  ltm.raceid = 1093  -- Circuit of the Americas
and    ltm.driverid = 830 -- Max Verstappen
)
select *
from   cota
match_recognize (
order by lap
measures
match_number() as mnr
,classifier() as cls
all rows per match
pattern (faster* slower* )
define
faster as laptime  prev (laptime)
)
order  by lap
/
LAP        MNR CLS     LAPTIME
---------- ---------- ------- -------------------
1          1         +00 00:01:41.343000
2          2 SLOWER  +00 00:01:41.696000
3          2 SLOWER  +00 00:01:41.893000
4          2 SLOWER  +00 00:01:42.581000
5          2 SLOWER  +00 00:01:42.870000
6          3 FASTER  +00 00:01:42.826000
7          3 FASTER  +00 00:01:42.314000
8          3 SLOWER  +00 00:01:42.830000
9          4 FASTER  +00 00:01:42.412000
10          4 SLOWER  +00 00:01:42.492000
>
37         13 FASTER  +00 00:01:39.541000
38         13 SLOWER  +00 00:01:40.211000
39         13 SLOWER  +00 00:01:40.337000
40         14 FASTER  +00 00:01:40.108000
41         14 FASTER  +00 00:01:40.077000
42         14 FASTER  +00 00:01:40.010000
43         14 FASTER  +00 00:01:39.776000
44         14 SLOWER  +00 00:01:40.410000
45         15 FASTER  +00 00:01:39.986000
46         15 FASTER  +00 00:01:39.945000
47         15 SLOWER  +00 00:01:40.058000
48         15 SLOWER  +00 00:01:40.271000
49         16 FASTER  +00 00:01:40.230000
50         16 SLOWER  +00 00:01:40.830000
51         17 FASTER  +00 00:01:40.152000
52         17 SLOWER  +00 00:01:40.431000
53         18 FASTER  +00 00:01:40.297000
54         18 FASTER  +00 00:01:40.168000
55         18 SLOWER  +00 00:01:40.674000
56         19 FASTER  +00 00:01:40.654000
```

第一圈没有分类，因为没有前一圈可供比较。第二圈到第五圈被归类为更慢，这可以从 `laptime` 列的数据中推断出来。最后，第六圈是第一次单圈时间比前一圈快。

**注意**

在上面的结果集中，还有一个 `MNR` 列，显示匹配编号。具有相同 `match_number` 的记录属于同一个匹配模式。第五圈是匹配该模式的最后一圈。第六圈被分类为更快的记录，这使模式重新开始。

现在，也可以分析是否存在连续多圈都比前一圈快的情况。当连续超过两圈更快时，这些圈应该出现在输出中。模式如下。

```
faster{2,}
```

该模式解读为：至少两条被分类为更快的记录。


### 分析单圈用时模式

```sql
with cota as (
select ltm.lap
, numtodsinterval( ltm.milliseconds / 1000
, 'second') as laptime
from   f1data.laptimes ltm
where  ltm.raceid = 1093  -- 美洲赛道
and    ltm.driverid = 830 -- 马克斯·维斯塔潘
)
select first_lap
, last_lap
, lap_count
from   cota
match_recognize (
order by lap
measures
first (lap) as first_lap
, last (lap)  as last_lap
, count(*)    as lap_count
one row per match
pattern ( faster{2,} )
define
faster as laptime < prev (laptime)
)
/
FIRST_LAP   LAST_LAP  LAP_COUNT
---------- ---------- ----------
6          7          2
15         16          2
25         26          2
40         43         4
45         46         2
53         54         2
已选择 6 行。
```

从第 40 圈开始，每一圈都比前一圈更快，并且持续了四圈。输出中也包含了更快圈数的计数。

稍微放宽模式条件，比如允许在一系列快圈中出现一个慢圈。模式如下：

```
faster{2,} slower {1} faster{1,}
```

该模式的含义是：两圈（或更多）更快的圈，接着一圈（且仅一圈）更慢的圈，再接着一圈（或更多）更快的圈。

使用此模式并定义了什么是慢圈之后，结果如下：

```
FIRST_LAP   LAST_LAP  LAP_COUNT
---------- ---------- ----------
6          9          4
40         46         7
53         56         4
```

基于这些结果，尽管存在轮胎磨损，但在比赛末段圈速变快是非常可能的。

### 胜场分析

`CONSTRUCTORSTANDINGS` 表包含了每场比赛、车队和记录的累计胜场数。以下是 2022 赛季前五场比赛中法拉利和红牛车队的输出结果。

```sql
select ctr.name
, cts.wins
, rcs.name as race_name
, rcs.round
, rcs.year
from   f1data.constructorstandings cts
join   f1data.constructors ctr
on   ctr.constructorid = cts.constructorid
join   f1data.races rcs
on   rcs.raceid = cts.raceid
where  rcs.year = 2022
and    ctr.constructorid in (6,9)
and    rcs.round <= 5
order  by ctr.name
, rcs.round
/
NAME     WINS RACE_NAME                 ROUND   YEAR
-------- ---- ------------------------- ----- ------
法拉利       1 巴林大奖赛                    1   2022
法拉利       1 沙特阿拉伯大奖赛              2   2022
法拉利       2 澳大利亚大奖赛                3   2022
法拉利       2 艾米利亚-罗马涅大奖赛          4   2022
法拉利       2 迈阿密大奖赛                  5   2022
红牛         0 巴林大奖赛                    1   2022
红牛         1 沙特阿拉伯大奖赛              2   2022
红牛         1 澳大利亚大奖赛                3   2022
红牛         2 艾米利亚-罗马涅大奖赛          4   2022
红牛         3 迈阿密大奖赛                  5   2022
```

`wins` 列是累计胜场数。法拉利赢得了第一轮，但输掉了第二轮，所以胜场数保持为 1。他们赢得了第三轮，所以胜场数增加到二，以此类推。

基于这些数据，可以做出如下 *获胜* 的定义：

```
winning as (wins > prev (wins))
```

当当前胜场数大于前一次胜场数时，即认为该轮获胜。类似地，*未获胜* 的定义如下：

```
not_winning as (wins = prev (wins))
```

当当前记录的胜场数与前一次相比保持不变时，意味着其他车队赢得了该轮（即当前记录对应的车队未获胜）。

以下是用于对数据集中的每条记录进行分类的模式：

```
( winning | not_winning )
```

这意味着一条记录要么被分类为 winning，要么为 not winning。让我们测试一下。

```sql
with ctrs as (
select ctr.name
, cts.wins
, rcs.name as race_name
, rcs.round
, rcs.year
from   f1data.constructorstandings cts
join   f1data.constructors ctr
on   ctr.constructorid = cts.constructorid
join   f1data.races rcs
on   rcs.raceid = cts.raceid
where  rcs.year = 2022
and    ctr.constructorid in (6,9)
and    rcs.round <= 5
)
select round
, race_name
, name
, cls
, wins
from   ctrs
match_recognize (
partition by name, year
order by round
measures
classifier () as cls
all rows per match
pattern ( winning | not_winning )
define
winning as (wins > prev (wins))
, not_winning as (wins = prev (wins))
)
order  by year, round, name
/
ROUND RACE_NAME                 NAME     CLS          WINS
----- ------------------------- -------- ------------ ----
2 沙特阿拉伯大奖赛              法拉利   NOT_WINNING     1
2 沙特阿拉伯大奖赛              红牛     WINNING         1
3 澳大利亚大奖赛                法拉利   WINNING         2
3 澳大利亚大奖赛                红牛     NOT_WINNING     1
4 艾米利亚-罗马涅大奖赛          法拉利   NOT_WINNING     2
4 艾米利亚-罗马涅大奖赛          红牛     WINNING         2
5 迈阿密大奖赛                  法拉利   NOT_WINNING     2
5 迈阿密大奖赛                  红牛     WINNING         3
```

现在结果集中只有八条记录。第一轮的数据缺失了。这条记录无法与前一条进行比较，因此还没有分类。

#### 未定义的分类

解决第一轮缺失问题的一个方法是向模式中添加一个额外的分类。

```
pattern ( winning | not_winning | unknown )
```

这需要在 `define` 子句中添加一个额外的定义，但实际上并不需要。当模式遇到未定义的定义时，它会被假定为真。由于第一轮不匹配任何其他已定义的分类，它会与“始终为真”的定义匹配，该定义被命名为 unknown。结果集中最前面的记录将变为：

```
ROUND RACE_NAME                 NAME     CLS          WINS
----- ------------------------- -------- ------------ ----
1 巴林大奖赛                    法拉利   UNKNOWN         1
1 巴林大奖赛                    红牛     UNKNOWN         0
2 沙特阿拉伯大奖赛              法拉利   NOT_WINNING     1
2 沙特阿拉伯大奖赛              红牛     WINNING         1
```

这解决了记录缺失的问题，但却是不正确的。法拉利赢得了赛季第一场比赛，但它被标记为 UNKNOWN。红牛也是如此；已知他们没有赢得第一场比赛。也许会想把 winning 的定义改为：

```
winning as (wins > prev (wins) or wins = 1)
```

但这也不正确；它会正确标记法拉利的第一场比赛，但法拉利的第二场比赛也会被标记为 WINNING（因为胜场数保持为 1）。应用这个修改后的定义将导致以下结果：

```
ROUND RACE_NAME                 NAME     CLS          WINS
----- ------------------------- -------- ------------ ----
1 巴林大奖赛                    法拉利   WINNING         1
1 巴林大奖赛                    红牛     UNKNOWN         0
2 沙特阿拉伯大奖赛              法拉利   WINNING         1
2 沙特阿拉伯大奖赛              红牛     WINNING         1
```

解决方案是扩展 `winning` 的定义，如下所示：

```
winning as (wins > prev (wins)
or (wins = 1 and round = 1))
```

这专门检查是否赢得了当年的第一轮比赛。`match_recognize` 子句变为：

```sql
match_recognize (
partition by name
,year
order by round
measures
classifier () as cls
all rows per match
pattern ( winning | not_winning )
define
winning as (wins > prev (wins)
or (wins = 1 and round = 1))
, not_winning as (wins = prev (wins)
or  wins = 0)
)
```

结果如下：

```
ROUND RACE_NAME                 NAME     CLS          WINS
----- ------------------------- -------- ------------ ----
1 巴林大奖赛                    法拉利   WINNING         1
1 巴林大奖赛                    红牛     NOT_WINNING     0
2 沙特阿拉伯大奖赛              法拉利   NOT_WINNING     1
2 沙特阿拉伯大奖赛              红牛     WINNING         1
3 澳大利亚大奖赛                法拉利   WINNING         2
3 澳大利亚大奖赛                红牛     NOT_WINNING     1
4 艾米利亚-罗马涅大奖赛          法拉利   NOT_WINNING     2
4 艾米利亚-罗马涅大奖赛          红牛     WINNING         2
5 迈阿密大奖赛                  法拉利   NOT_WINNING     2
5 迈阿密大奖赛                  红牛     WINNING         3
```

现在每条记录都有了正确的分类：WINNING 或 NOT_WINNING。


### 最长连胜纪录

本书撰写时，2022 赛季尚未结束，仅法拉利和红牛车队赢得过分站赛。但哪支车队拥有最多的连续胜利？让我们一探究竟。

上一节通过以下`define`子句将比赛标识为胜利。

```
winning as (wins > prev (wins)
or (wins = 1 and round = 1))
```

基于此定义，可应用以下模式查找至少一条符合此分类的记录。

```
( winning+ )
```

以下是查找车队最长连胜纪录的完整查询。

```
with ctr as (
select ctr.name
, cts.wins
, rcs.name as race_name
, rcs.round
, rcs.year
from   f1data.constructorstandings cts
join   f1data.constructors ctr
on   ctr.constructorid = cts.constructorid
join   f1data.races rcs
on   rcs.raceid = cts.raceid
where  rcs.year = 2022
)
select *
from   ctr
match_recognize (
partition by name
order by round
measures
classifier () as cls
, first (round) as first_win
, last (round)  as last_win
, count(*)      as race_count
one row per match
pattern ( winning+ )
define
winning as (wins > prev (wins)
or (wins = 1 and round = 1))
)
order  by race_count desc
/
NAME     CLS           FIRST_WIN   LAST_WIN RACE_COUNT
-------- ------------ ---------- ---------- ----------
Red Bull WINNING              12         19          8
Red Bull WINNING               4          9          6
Ferrari  WINNING              10         11          2
Red Bull WINNING               2          2          1
Ferrari  WINNING               1          1          1
Ferrari  WINNING               3          3          1
6 rows selected.
```

红牛车队取得了八连胜：从第 12 轮到第 19 轮。

### 进站维修与危险状况

分析单圈用时也能洞察比赛中何时发生进站维修。当车手进站维修时，下一圈（出场圈）的用时会比赛事平均用时更长。类似地，可以检测到危险状况。当出现黄旗情况时，赛车需要减速，导致单圈用时增加。若车手在危险状况下进站（这通常是一项好策略），这些情况则无法被检测到。

在使用模式匹配之前，必须对输入数据进行预处理。除了车手、圈数和单圈用时外，还需要（每位车手的）平均单圈用时。为此，使用了`avg`分析函数，如下例所示。

```
select drv.driverid
, drv.forename||' '||drv.surname as driver
, ltm.lap
, ltm.milliseconds
, avg (ltm.milliseconds)
over (partition by drv.driverid
) as avg_laptime
from   f1data.laptimes ltm
join   f1data.drivers drv
on   drv.driverid = ltm.driverid
where  ltm.raceid = 1093 -- 美洲赛道
order  by drv.driverid
, ltm.lap
/
DRIVERID DRIVER                  LAP MILLISECONDS AVG_LAPTIME
---------- ---------------- ---------- ------------ -----------
1 Lewis Hamilton            1       102630  109584.107
1 Lewis Hamilton            2       102325  109584.107
1 Lewis Hamilton            3       102175  109584.107
1 Lewis Hamilton            4       102798  109584.107
1 Lewis Hamilton            5       103196  109584.107
1 Lewis Hamilton            6       102576  109584.107
1 Lewis Hamilton            7       102702  109584.107
1 Lewis Hamilton            8       103212  109584.107
>
855 Guanyu Zhou              52       103373  110854.482
855 Guanyu Zhou              53       105200  110854.482
855 Guanyu Zhou              54       104102  110854.482
855 Guanyu Zhou              55       103812  110854.482
855 Guanyu Zhou              56       103881  110854.482
```

根据以下定义，可以定义`faster_than_avg`和`slow_lap`。

```
faster_than_avg as milliseconds  avg_laptime
```

定义完成后，最简单的方法是从显示所有记录开始，并使用以下模式进行分类。

```
faster_than_avg* slow_lap*
```

这导出了以下的`match_recognize`子句。

```
match_recognize (
partition by driverid
order by lap
measures
classifier() as what
all rows per match
pattern (
faster_than_avg* slow_lap*
)
define
faster_than_avg as milliseconds  avg_laptime
)
```

仔细观察周冠宇的前 26 圈并据此分类每一圈，得到以下结果。

```
DRIVER                  LAP MILLISECONDS WHAT
---------------- ---------- ------------ ----------------
Guanyu Zhou               1       112824 SLOW_LAP
Guanyu Zhou               2       105535 FASTER_THAN_AVG
Guanyu Zhou               3       104538 FASTER_THAN_AVG
Guanyu Zhou               4       104511 FASTER_THAN_AVG
Guanyu Zhou               5       104904 FASTER_THAN_AVG
Guanyu Zhou               6       104476 FASTER_THAN_AVG
Guanyu Zhou               7       104454 FASTER_THAN_AVG
Guanyu Zhou               8       105360 FASTER_THAN_AVG
Guanyu Zhou               9       105956 FASTER_THAN_AVG
Guanyu Zhou              10       127583 SLOW_LAP
Guanyu Zhou              11       103895 FASTER_THAN_AVG
Guanyu Zhou              12       104152 FASTER_THAN_AVG
Guanyu Zhou              13       104739 FASTER_THAN_AVG
Guanyu Zhou              14       103824 FASTER_THAN_AVG
Guanyu Zhou              15       103741 FASTER_THAN_AVG
Guanyu Zhou              16       104030 FASTER_THAN_AVG
Guanyu Zhou              17       104266 FASTER_THAN_AVG
Guanyu Zhou              18       134425 SLOW_LAP
Guanyu Zhou              19       138587 SLOW_LAP
Guanyu Zhou              20       144384 SLOW_LAP
Guanyu Zhou              21       142312 SLOW_LAP
Guanyu Zhou              22       133593 SLOW_LAP
Guanyu Zhou              23       169446 SLOW_LAP
Guanyu Zhou              24       184224 SLOW_LAP
Guanyu Zhou              25       139116 SLOW_LAP
Guanyu Zhou              26       105555 FASTER_THAN_AVG
```

从输出可见，第一圈比他平均单圈用时慢。这不太可能由进站维修或危险状况导致。第 10 圈是慢圈，可能是进站维修所致。同样清晰可见的是，在第 18 圈到第 25 圈期间发生了某些状况，因为这些圈的用时明显慢于其他圈。

如前所述，进站维修后的下一圈用时更长。要确定进入维修通道的圈数，必须减去一圈以获得正确的圈号。这可以在最终查询中修正。

基于这些假设，进站维修后单圈用时会增加一圈，而发生危险状况时则会增加多圈。可以推导出以下模式。

```
faster_than_avg{1} slow_lap{1,} faster_than_avg{1}
```

该模式表示：仅一圈（单圈用时）快于平均值，后跟一圈或多圈慢圈，再后跟一圈（且仅一圈）快于平均值的圈。这排除了当第一圈是慢圈的情况，例如周冠宇的情况。

以下是检测（可能的）进站维修和危险状况的完整查询。

