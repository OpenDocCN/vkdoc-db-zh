# 4. 使用 MATCH_RECOGNIZE 进行行模式匹配

#### SQL 查询示例

```sql
with laps
as (
select drv.driverid
, drv.forename||' '||drv.surname as driver
, ltm.lap
, ltm.milliseconds
, avg (ltm.milliseconds)
over (partition by drv.driverid) as avg_laptime
from   f1data.laptimes ltm
join   f1data.drivers drv
on   drv.driverid = ltm.driverid
where  ltm.raceid = 1093 -- 美洲赛道
)
select driver
, case lap_count
when 1
then '可能是在第 '||to_char (firstlap -1)||' 圈进站'
else
'** 从第 '
||to_char (firstlap - 1)
||' 圈开始的危险阶段，持续 '
||to_char (lap_count)
||' 圈'
||'，结束于第 '||to_char (lastlap)||' 圈'
end as what_happened
from   laps
match_recognize (
partition by driverid
order by lap
measures
first (slow_lap.lap) as firstlap
, last  (slow_lap.lap) as lastlap
, count (slow_lap.lap) as lap_count
, driver               as driver
one row per match
pattern (
faster_than_avg{1} slow_lap{1,} faster_than_avg{1}
)
define
faster_than_avg as milliseconds  avg_laptime
)
/
```

#### 查询结果

```
DRIVER           WHAT_HAPPENED
---------------- ----------------------------------------
Lewis Hamilton   可能是在第 12 圈进站
Lewis Hamilton   ** 从第 17 圈开始的危险阶段，持续 8 圈，结束于第 25 圈
Lewis Hamilton   可能是在第 34 圈进站
Fernando Alonso  ** 从第 17 圈开始的危险阶段，持续 8 圈，结束于第 25 圈
Sebastian Vettel ** 从第 17 圈开始的危险阶段，持续 8 圈，结束于第 25 圈
Sebastian Vettel 可能是在第 41 圈进站
Sergio Pérez     可能是在第 14 圈进站
Sergio Pérez     ** 从第 17 圈开始的危险阶段，持续 8 圈，结束于第 25 圈
Sergio Pérez     可能是在第 38 圈进站
>
Yuki Tsunoda     可能是在第 10 圈进站
Yuki Tsunoda     ** 从第 17 圈开始的危险阶段，持续 8 圈，结束于第 25 圈
Yuki Tsunoda     可能是在第 33 圈进站
Mick Schumacher  ** 从第 17 圈开始的危险阶段，持续 8 圈，结束于第 25 圈
Mick Schumacher  可能是在第 33 圈进站
Guanyu Zhou      可能是在第 9 圈进站
Guanyu Zhou      ** 从第 17 圈开始的危险阶段，持续 8 圈，结束于第 25 圈
已选择 43 行。
```

### 总结

在数据集中搜索特定模式对业务可能非常有用。在引入 `match_recognize` 子句之前，在数据集中寻找模式是一项艰巨的任务。将行模式匹配功能靠近数据执行，使其运行非常高效。

本章仅仅触及了 `match_recognize` 功能可能性的皮毛。行模式匹配还有许多其他用例，例如解决**装箱问题**。

## 5. 分页与集合操作

使用 Oracle 数据库时，处理大型数据集相当常见。但并非所有数据都应该展示给数据库应用程序的用户。通常，仅前 50 条记录就足以供用户交互。本章探讨了一个典型的 Top-N 查询，用于检索前 50 条记录。展示接下来的 50 条记录，或任何偏移量，是分页类查询的示例，这也是本章的重点。

此外，本章还将探讨不同的集合操作符，用于获取多个数据集之间的差异和共同元素。在这方面，Oracle Database 21c 增加了更多功能，以更好地符合 ANSI 标准。

### Top-N 与分页

在 Oracle 中，获取较大集合的前几条记录，例如前 5 条，一直是可以实现的，但方法总是有点“笨拙”，而且并非总是像它本应那样直接。

对于以下示例，使用如下定义创建了一个视图。

```sql
create view all_driverstandings
as
select rcs.year
, rcs.round
, cct.name
, drs.position
, drv.driverid
, drv.forename ||' '||drv.surname as driver
, drs.points
, drs.wins
from   f1data.driverstandings drs
join   f1data.drivers drv
on   drv.driverid = drs.driverid
join   f1data.races rcs
on   rcs.raceid = drs.raceid
join   f1data.circuits cct
on   cct.circuitid = rcs.circuitid
/
```

`ALL_DRIVERSTANDINGS` 视图展示了按年份和场次划分的车手排名。该视图显示了超过 33,000 条记录，因为数据可追溯至 20 世纪 50 年代。

以下是首次尝试，用于获取 2021 赛季且仅针对两位车手的该视图中的前五行。

```sql
select round
, position
, driver
, points
, wins
from   all_driverstandings
where  year = 2021
and    driverid in (830, 844) -- Leclerc, Verstappen
and    rownum <= 5
order  by round
/
```

```
ROU   POSITION DRIVER                 POINTS       WINS
--- ---------- ------------------ ---------- ----------
1          2 Max Verstappen             18          0
1          6 Charles Leclerc             8          0
2          4 Charles Leclerc            20          0
20          1 Max Verstappen          351.5          9
20          6 Charles Leclerc           152          0
```

如输出所示，结果集是不正确的。在前五行中，不应该包含第 20 圈的数据。这个查询的问题在于，`rownum` 伪列是在最终的排序操作*之前*确定的。当一行因符合过滤条件而被考虑输出时，`rownum` 会为其分配一个值，从 1 开始。下一个符合过滤条件的行被分配值 2，依此类推。当 `rownum` 达到 5（在前面的查询中）时，不再选择更多记录。在此步骤之后，结果集才按照 `order by` 子句指定的按场次排序。可以在执行计划中观察到此行为（仅显示相关部分）。

```sql
select *
from dbms_xplan.display_cursor(null, null, 'BASIC PREDICATE')
/
```

```
PLAN_TABLE_OUTPUT

| Id  | Operation                          | Name            |

|   0 | SELECT STATEMENT                   |                 |
|   1 |  SORT ORDER BY                     |                 |
|*  2 |   COUNT STOPKEY                    |                 |
>
Predicate Information (identified by operation id):

2 - filter(ROWNUM>
```

过滤器在操作 ID 2 中应用（`rownum <= 5`）。结果数据集在步骤 1 中进行排序。

要正确获取预期的输出，需要将排序后的数据集推入一个内联视图（或子查询因子化子句）中。

```sql
select *
from   (select round
, position
, driver
, points
, wins
from   all_driverstandings
where  year = 2021
and    driverid in (830, 844) -- Leclerc, Verstappen
order  by round
)
where  rownum <= 5
/
```

```
ROU   POSITION DRIVER                 POINTS       WINS
--- ---------- ------------------ ---------- ----------
1          2 Max Verstappen             18          0
1          6 Charles Leclerc             8          0
2          4 Charles Leclerc            20          0
2          2 Max Verstappen             43          1
3          2 Max Verstappen             61          1
```

现在优化器理解了意图，这反映在执行计划中（仅显示相关部分）。

```
|   0 | SELECT STATEMENT                    |
|*  1 |  COUNT STOPKEY                      |
|   2 |   VIEW                              |
|*  3 |    SORT ORDER BY STOPKEY            |
>
Predicate Information (identified by operation id):

1 - filter(ROWNUM>
```


#### 行限制子句

自 Oracle Database 12c 起，这种 Top-N 查询被极大简化，并提供了额外的功能，例如通过行限制子句实现分页。

以下查询将获取两位车手在本赛季的前五名成绩。

```
select round
, position
, driver
, points
, wins
from   all_driverstandings
where  year = 2022
and    driverid in (830, 844) -- Leclerc, Verstappen
order  by round
fetch  first 5 rows only
/
ROU   POSITION DRIVER                 POINTS       WINS
--- ---------- ------------------ ---------- ----------
1         19 Max Verstappen              0          0
1          1 Charles Leclerc            26          1
2          1 Charles Leclerc            45          1
2          3 Max Verstappen             25          1
3          1 Charles Leclerc            71          2
```

查询的意图立即变得清晰：获取前五条记录。而“前五条”由查询中的排序子句决定。

行限制子句可以容纳一个静态的、需要查询的固定行数，也可以是一个绑定变量。以下示例使用 SQLcl 声明了一个名为 `b` 的绑定变量并执行。

```
var b number
begin
:b := 5;
end;
/
select round
, position
, driver
, points
, wins
from   all_driverstandings
where  year = 2022
and    driverid in (830, 844) -- Leclerc, Verstappen
order  by round
fetch  first :b rows only
/
ROU   POSITION DRIVER                 POINTS       WINS
--- ---------- ------------------ ---------- ----------
1         19 Max Verstappen              0          0
1          1 Charles Leclerc            26          1
2          1 Charles Leclerc            45          1
2          3 Max Verstappen             25          1
3          1 Charles Leclerc            71          2
```

如请求所示，前面的查询返回了五行数据，但如果存在并列值怎么办？在这种情况下，你可能会认为查询应该返回六条记录，因为两位车手都参与了前三轮比赛。当排序依据的值不是确定的（本例中是 `round`），最后一条记录是任意的。行限制子句也可以处理并列值，使用 `rows with ties` 子句代替 `rows only`。

```
select round
, position
, driver
, points
, wins
from   all_driverstandings
where  year = 2022
and    driverid in (830, 844) -- Leclerc, Verstappen
order  by round
fetch  first 5 rows with ties
/
ROU   POSITION DRIVER                 POINTS       WINS
--- ---------- ------------------ ---------- ----------
1         19 Max Verstappen              0          0
1          1 Charles Leclerc            26          1
2          1 Charles Leclerc            45          1
2          3 Max Verstappen             25          1
3          1 Charles Leclerc            71          2
3          6 Max Verstappen             25          1
```

除了通过行限制子句返回一个固定行数（静态或绑定变量），还可以返回总行数的百分比。以下示例返回百分之六的记录。本例中，总结果集有 38 条记录，百分之六是 2.28 条记录。

```
select round
, position
, driver
, points
, wins
from   all_driverstandings
where  year = 2022
and    driverid in (830, 844)
order  by round
fetch  first 6 percent rows only
/
ROU   POSITION DRIVER                 POINTS       WINS
--- ---------- ------------------ ---------- ----------
1         19 Max Verstappen              0          0
1          1 Charles Leclerc            26          1
2          1 Charles Leclerc            45          1
```

百分比会向上取整（后台使用了 `ceil` 函数），导致前面的结果显示三条记录。当然，百分比也可以与 `rows with ties` 结合使用。

```
select round
, position
, driver
, points
, wins
from   all_driverstandings
where  year = 2022
and    driverid in (830, 844)
order  by round
fetch  first 6 percent rows with ties
/
ROU   POSITION DRIVER                 POINTS       WINS
--- ---------- ------------------ ---------- ----------
1         19 Max Verstappen              0          0
1          1 Charles Leclerc            26          1
2          1 Charles Leclerc            45          1
2          3 Max Verstappen             25          1
```


### 分页

行限制子句还提供了用于分页类型查询的功能。除了显示前 x 行外，还可以跳过前 x 行并显示下一组记录。以下示例显示了一个 `offset` 子句，它跳过 2022 年第一轮比赛中的前十行。

```sql
select round
, position
, driver
, points
, wins
from   all_driverstandings
where  year = 2022
and    round = 1
order  by points desc
offset 10 rows
/
ROU   POSITION DRIVER                 POINTS       WINS
--- ---------- ------------------ ---------- ----------
1         11 Mick Schumacher             0          0
1         12 Lance Stroll                0          0
1         13 Alexander Albon             0          0
1         14 Daniel Ricciardo            0          0
1         15 Lando Norris                0          0
1         16 Nicholas Latifi             0          0
1         17 Nico Hülkenberg             0          0
1         18 Sergio Pérez                0          0
1         20 Pierre Gasly                0          0
1         19 Max Verstappen              0          0
```

`offset` 子句仅限于 x 行；不能指定行的百分比。如果表达式计算结果为数值，则可以指定该表达式。

对数据集进行分页的最常见方法是首先显示一组行（假设是五条记录），当导航到下一页时，跳过已经看过的前五行，显示接下来的五行。这可以使用 `offset` 子句和 `fetch` 子句来实现。以下示例跳过前五行并返回接下来的五行。

```sql
select position
, driver
, points
, wins
from   all_driverstandings
where  year = 2022
and    round = 1
order  by position
offset 5 rows
fetch  next 5 rows only
/
POSITION DRIVER                 POINTS       WINS
---------- ------------------ ---------- ----------
6 Valtteri Bottas             8          0
7 Esteban Ocon                6          0
8 Yuki Tsunoda                4          0
9 Fernando Alonso             2          0
10 Guanyu Zhou                 1          0
```

对于分页，使用 `fetch next` 而不是 `fetch first`。在分页查询中将行数硬编码是没有意义的，很可能使用此查询的应用程序希望指定偏移量以及向用户显示多少行。以下查询使用 SQLcl 中的绑定变量显示了获取接下来五条记录的分页查询。

```sql
var offset number
var next  number
begin
:offset := 10;
:next := 5;
end;
/
select position
, driver
, points
, wins
from   all_driverstandings
where  year = 2022
and    round = 1
order  by position
offset :offset rows
fetch  next :next rows only
/
POSITION DRIVER                 POINTS       WINS
---------- ------------------ ---------- ----------
11 Mick Schumacher             0          0
12 Lance Stroll                0          0
13 Alexander Albon             0          0
14 Daniel Ricciardo            0          0
15 Lando Norris                0          0
```

使用此查询，当偏移量使用 0 时，也可以显示结果集中的前五行。

```sql
begin
:offset := 0;
end;
/
>
POSITION DRIVER                 POINTS       WINS
---------- ------------------ ---------- ----------
1 Charles Leclerc            26          1
2 Carlos Sainz               18          0
3 Lewis Hamilton             15          0
4 George Russell             12          0
5 Kevin Magnussen            10          0
```

根据所使用的查询，获取第十页可能比获取第一页更耗时。

#### 底层机制

在底层，行限制子句利用了分析函数，如 `row_number()` 或 `rank()`，当检查行限制部分中第一个查询的执行计划的谓词信息时可以观察到这一点（仅显示相关信息）。

```sql
select *
from dbms_xplan.display_cursor (null, null, 'BASIC PREDICATE')
/
PLAN_TABLE_OUTPUT

| Id  | Operation                          | Name            |

|   0 | SELECT STATEMENT                   |                 |
|*  1 |  VIEW                              |                 |
|*  2 |   WINDOW SORT PUSHED RANK          |                 |
Predicate Information (identified by operation id):

1 - filter("from$_subquery$_002"."rowlimit_$$_rownumber"<=5)
2 - filter(ROW_NUMBER() OVER ( ORDER BY "RCS"."ROUND")<=5)
```

正如在谓词信息中所看到的，分析函数 `row_number()` 与原始查询中的排序结合使用。

**注意**
从查询中省略 `order by` 子句不会引发错误或给出警告。在查询中省略 `order by` 可能导致分析函数中的“order by null”可能不会给出预期的结果。

### 集合操作符

让我们使用来自 `drivers` 表的两个数据子集来演示集合操作符的工作原理。首先，有一组名字是 Graham 的车手。

```sql
create view grahams
as
select drv.driverid
, drv.driverref
, drv.forename
, drv.surname
, drv.nationality
from   f1data.drivers drv
where  drv.forename = 'Graham'
/
```

该视图显示以下结果集。

```sql
DRIVERID DRIVERREF           FORENAME   SURNAME    NATIONALITY
-------- ------------------- ---------- ---------- --------------
336 mcrae               Graham     McRae      New Zealander
289 hill                Graham     Hill       British
745 graham_whitehead    Graham     Whitehead  British
```

其次，有一组姓氏是 Hill 的车手。

```sql
create view hills
as
select drv.driverid
, drv.driverref
, drv.forename
, drv.surname
, drv.nationality
from   f1data.drivers drv
where  drv.surname = 'Hill'
/
```

该视图显示以下结果集。

```sql
DRIVERID DRIVERREF           FORENAME   SURNAME    NATIONALITY
-------- ------------------- ---------- ---------- --------------
71 damon_hill          Damon      Hill       British
289 hill                Graham     Hill       British
403 phil_hill           Phil       Hill       American
```

这些视图基于同一个表，每个视图包含数据的一个子集。


## 联合 (Union)

将之前创建的两个视图结合起来，可以使用 `union` 操作符。对于此操作符，两个集合必须具有相同的形状和形式；也就是说，每列的数据类型必须匹配或至少属于同一族。整数和浮点数可以混合匹配，但不能将 `VARCHAR2` 与数字混合。

将两个结果集联合起来会得到以下结果。

```sql
select g.driverid
, g.driverref
, g.forename
, g.surname
, g.nationality
from   grahams g
union
select h.driverid
, h.driverref
, h.forename
, h.surname
, h.nationality
from   hills h
/
DRIVERID DRIVERREF        FORENA SURNAME   NATIONALITY
---------- ---------------- ------ --------- -------------
71 damon_hill       Damon  Hill      British
289 hill             Graham Hill      British
336 mcrae            Graham McRae     New Zealander
403 phil_hill        Phil   Hill      American
745 graham_whitehead Graham Whitehead British
```

`union` 操作符不仅仅是连接两个结果集。它还会对它们进行排序，并从最终结果中删除重复的行。这种行为可以在前面查询的执行计划中观察到。

```sql
select *
from dbms_xplan.display_cursor(null, null, 'BASIC')
/
PLAN_TABLE_OUTPUT

| Id  | Operation                   | Name    |

|   0 | SELECT STATEMENT            |         |
|   1 |  SORT UNIQUE                |         |
|   2 |   UNION-ALL                 |         |
|*  3 |    TABLE ACCESS STORAGE FULL| DRIVERS |
|*  4 |    TABLE ACCESS STORAGE FULL| DRIVERS |

```

执行计划的步骤 1 中可见 `sort unique` 步骤，导致输出中有 5 条记录。`Graham Hill` 只出现一次，尽管他在两个视图中都存在。

从 Oracle Database 21c 开始，这个联合操作符可以通过添加 `distinct` 来更明确地编写。

```sql
union distinct
```

工作原理保持不变；它增加了语法清晰度。

当需要所有数据，无论是否存在重复时，使用 `union all` 操作符。这消除了执行计划中对 `sort unique` 步骤的需要。

```sql
select g.driverid
, g.driverref
, g.forename
, g.surname
, g.nationality
from   grahams g
union all
select h.driverid
, h.driverref
, h.forename
, h.surname
, h.nationality
from   hills h
/
DRIVERID DRIVERREF        FORENA SURNAME   NATIONALITY
---------- ---------------- ------ --------- -------------
336 mcrae            Graham McRae     New Zealander
289 hill             Graham Hill      British
745 graham_whitehead Graham Whitehead British
71 damon_hill       Damon  Hill      British
289 hill             Graham Hill      British
403 phil_hill        Phil   Hill      American
```

正如你在最终结果中看到的，驾驶员 289 (`Graham Hill`) 出现了两次。在执行计划中，不再有 `sort unique`。

```sql
| Id  | Operation                  | Name    |

|   0 | SELECT STATEMENT           |         |
|   1 |  UNION-ALL                 |         |
|   2 |   TABLE ACCESS STORAGE FULL| DRIVERS |
|   3 |   TABLE ACCESS STORAGE FULL| DRIVERS |

```

## 减去 (Minus) 与排除 (Except)

添加以下视图以演示 `minus` 和 `minus all` 操作符的工作原理。

```sql
create or replace view grahams_and_hills
as
select g.driverid
, g.driverref
, g.forename
, g.surname
, g.nationality
from   grahams g
union all
select h.driverid
, h.driverref
, h.forename
, h.surname
, h.nationality
from   hills h
/
```

此视图暴露以下数据。

```sql
DRIVERID DRIVERREF        FORENA SURNAME   NATIONALITY
---------- ---------------- ------ --------- -------------
336 mcrae            Graham McRae     New Zealander
289 hill             Graham Hill      British
745 graham_whitehead Graham Whitehead British
71 damon_hill       Damon  Hill      British
289 hill             Graham Hill      British
403 phil_hill        Phil   Hill      American
```

请注意，`Graham Hill` 在结果集中出现了两次。

`minus` 操作符仅过滤出存在于第一个输入集合但不存在于第二个输入集合中的行。为了演示这个操作，让我们使用 `grahams_and_hills` 视图，并通过一个选择所有 `British` 驾驶员的查询来使用 `minus` 操作符。

```sql
select gh.driverid
, gh.driverref
, gh.forename
, gh.surname
, gh.nationality
from grahams_and_hills gh
minus
select drv.driverid
, drv.driverref
, drv.forename
, drv.surname
, drv.nationality
from   f1data.drivers drv
where  drv.nationality = 'British'
/
DRIVERID DRIVERREF           FORENAME   SURNAME    NATIONALITY
-------- ------------------- ---------- ---------- --------------
336 mcrae               Graham     McRae      New Zealander
403 phil_hill           Phil       Hill       American
```

如你所见，尽管 `Graham Hill`（一位 `British` 驾驶员）在第一个集合（`grahams_and_hills` 视图）中出现了两次，但他不再出现在最终结果集中。`Graham Hill` 的两次出现都从第一个结果集中移除了。

从 Oracle Database 21c 开始，这个 `minus` 操作符可以通过添加 `distinct` 来更明确地编写。

```sql
minus distinct
```

工作原理保持不变；它增加了语法清晰度。

自 Oracle Database 21c 起，`minus` 操作符也有一个 `all` 版本。如果将脚本改为使用 `minus all` 操作符，你会得到不同的结果。

```sql
select gh.driverid
, gh.driverref
, gh.forename
, gh.surname
, gh.nationality
from   grahams_and_hills gh
minus all
select drv.driverid
, drv.driverref
, drv.forename
, drv.surname
, drv.nationality
from   f1data.drivers drv
where  drv.nationality = 'British'
/
DRIVERID DRIVERREF           FORENAME   SURNAME    NATIONALITY
-------- ------------------- ---------- ---------- --------------
336 mcrae               Graham     McRae      New Zealander
289 hill                Graham     Hill       British
403 phil_hill           Phil       Hill       American
```

在这种情况下，`Graham Hill` 确实出现在最终结果集中。这是因为他出现在第一个集合（`grahams_and_hills` 视图）中两次，但在第二个集合中只出现一次。该记录只被移除了一次，而不是所有出现都被移除。

还实现了 `except` 操作符以符合 ANSI 标准。`except` 的工作原理类似于 `minus`，包括所有变体。

```sql
except
except distinct
except all
```



## 6. 条件编译

条件编译自 Oracle 10g 起就已可用。此功能在无数场景中都非常有用。例如，您可以使用条件编本来检查数据库环境（开发、测试、生产）、检查编译器优化级别、根据环境包含或排除代码，并为未来数据库版本的发布准备代码。

### 指令

条件编译语法由三条指令组成：选择指令、查询指令和错误指令。这些指令以美元符号（`$`）为前缀，以区别于普通的 PL/SQL 语法。

#### 选择指令

选择指令使用以下构建块。

```
$if $then [ $elsif ] [ $else ] $end
```

一个代码块可能如下所示。

```
$if some_condition = true $then
select 'Patrick' into l_name from dual;
$else
select 'Alex' into l_name from dual;
$end
```

这看起来很像普通的 PL/SQL 条件，但美元符号告诉编译器它正在进入一个条件编译块。如果条件（`查询指令`）求值为 true，则跟在 `$then` 之后的代码块会被包含在编译后的程序中。如果查询指令为 false，则编译器会评估 `$elsif` 条件（如果可用）。如果没有条件为 true，则会包含 `$else` 分支中的块（如果可用）。

#### 查询指令

在选择指令之后，您需要指定一个查询指令，以确定选择指令的哪个分支应被包含在编译后的程序中。查询指令可以是一个标志，使用 `plsql_ccflags` 设置来定义。您可以使用 `alter session` 语句来设置这些标志。

```
alter session set plsql_ccflags = 'myflag:true'
```

但您也可以在编译程序时指定标志。

```
alter package mypackage compile plsql_ccflags='myflag:true'
```

如果一个标志不存在，则它会被求值为 false。标志只能保存静态布尔值。要使用数值，您必须创建一个包来将这些值保存为常量。在 Oracle Database 19c 之前，您只能使用整数或 `pls_integer`；从该版本起，所有数值数据类型都已可用。查询指令必须是静态布尔表达式。

注意

您不能在查询指令中使用 `VARCHAR2` 常量。

### 总结

本章探讨了前 N 条和分页查询。内联视图、分析函数以及 `offset` 和 `fetch` 子句使查询更具可读性且更易于维护。

对集合操作的增强增加了语法清晰度，并提高了对 ANSI 标准的遵循度。

