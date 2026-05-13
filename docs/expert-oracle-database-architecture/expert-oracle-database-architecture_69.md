# 日期、时间戳和间隔类型

Oracle 原生数据类型 `DATE`、`TIMESTAMP` 和 `INTERVAL` 密切相关。`DATE` 和 `TIMESTAMP` 类型以不同的精度存储固定的日期/时间。`INTERVAL` 类型用于轻松存储一段时间量，例如“8 小时”或“30 天”。两个时间戳相减的结果可能是一个间隔；将八小时间隔添加到 `TIMESTAMP` 的结果是一个新的 `TIMESTAMP`，时间晚了八小时。

`DATE` 数据类型在 Oracle 的许多版本中都存在——从我对 Oracle 的经验来看，至少可以追溯到第 5 版甚至更早。相比之下，`TIMESTAMP` 和 `INTERVAL` 类型是相对较新的成员。因此，您会发现 `DATE` 数据类型是存储日期/时间信息最普遍的类型。但许多新应用程序正在使用 `TIMESTAMP` 类型，原因有二：它支持小数秒（`DATE` 类型不支持），并且它支持时区（`DATE` 类型也不支持）。

在讨论 `DATE`/`TIMESTAMP` 格式及其用途之后，我们将逐一查看每种类型。



### 格式

我在此不打算涵盖所有的 `DATE`、`TIMESTAMP` 和 `INTERVAL` 格式。这一点在 *Oracle Database SQL Language Reference* 手册中有详尽的阐述，该手册对所有人免费开放。有丰富的格式可供您使用，透彻理解它们至关重要。我强烈建议您去研究一下。

我想在这里讨论一下这些格式的作用，因为围绕这个话题存在很多误解。这些格式用于两件事：

*   以您喜欢的方式，格式化从数据库中输出的数据。
*   告诉数据库如何将输入字符串转换为 `DATE`、`TIMESTAMP` 或 `INTERVAL`。

仅此而已。多年来我观察到的一个普遍误解是，使用的格式会以某种方式影响磁盘上存储的内容以及数据实际保存的方式。*格式对数据的存储方式完全没有影响。格式仅用于将存储 `DATE` 所用的单一二进制格式转换为字符串，或将字符串转换为存储 `DATE` 所用的单一二进制格式。* 这对于 `TIMESTAMP` 和 `INTERVAL` 同样适用。

我关于格式的建议很简单：使用它们。当您向数据库发送表示 `DATE`、`TIMESTAMP` 或 `INTERVAL` 的字符串时，请使用格式。*不要* 依赖默认的日期格式——默认值可以，也可能在未来某个时候被某人更改。

注意

请回顾第 1 章，了解一个真正重要的安全原因，即永远不要在不指定显式格式的情况下使用 `TO_CHAR`/`TO_DATE`。在那一章中，我描述了一个终端用户可以实施的 SQL 注入攻击，仅仅是因为开发者忘记了使用显式格式。此外，执行日期操作而不使用显式日期格式可能而且将会导致错误的答案。为了理解这一点，请告诉我这个字符串代表什么日期：‘01-02-03’。无论您说它代表什么，我都会告诉您您是错的。永远不要依赖默认值！

如果您依赖默认日期格式而它发生了更改，您的应用程序可能会受到负面影响。如果日期无法转换，它可能会向终端用户返回错误，存在严重的安全漏洞，或者同样糟糕的是，它可能默默地插入*错误的数据*。考虑以下依赖默认日期掩码的 `INSERT` 语句：

```
Insert into t ( date_column ) values ( '01/02/03' );
```

假设应用程序依赖的默认日期掩码是 `DD/MM/YY`。那将是 2003 年 2 月 1 日（假设代码是在 2000 年之后执行的，但我们稍后会讨论其影响）。现在，假设有人认为正确的日期格式应该是 `MM/DD/YY`。突然间，之前的日期变成了 2003 年 1 月 2 日。或者有人认为 `YY/MM/DD` 是正确的，现在你得到了 2001 年 2 月 3 日。简而言之，如果没有日期格式伴随该日期字符串，有很多方式可以解释它。那个 `INSERT` 语句应该是：

```
Insert into t ( date_column ) values ( to_date( '01/02/03', 'DD/MM/YY' ) );
```

而如果要我发表意见，它必须是：

```
Insert into t ( date_column ) values ( to_date( '01/02/2003', 'DD/MM/YYYY' ) );
```

也就是说，必须使用四位数的年份。几年前，我们行业已经通过惨痛教训认识到，为了“节省” 2 个字节，我们花费了多少时间和精力来修复软件。随着时间的推移，我们似乎忘记了这个教训。如今没有借口*不*使用四位数的年份！仅仅因为千禧年（2000 年）已经过去，并不意味着你现在可以使用两位数的年份。想想出生日期，例如。如果您使用以下方式输入出生日期：

```
Insert into t ( DOB ) values ( to_date( '01/02/10', 'DD/MM/YY' ) );
```

那是 2010 年 2 月 1 日，还是 1910 年 2 月 1 日？任何一个都是有效的值；您不能随便挑一个作为正确的。

同样的讨论也适用于离开数据库的数据。如果您执行 `SELECT DATE_COLUMN FROM T` 并将该列作为字符串提取到您的应用程序中，那么您应该对其应用显式日期格式。无论您的应用程序期望什么格式，都应该在那里显式指定。否则，在未来某个时候当有人更改默认日期格式时，您的应用程序可能会崩溃或行为异常。

接下来，让我们更详细地看看数据类型本身。


## DATE 类型

`DATE` 类型是一种固定宽度的 7 字节日期/时间数据类型。它始终包含世纪、世纪内年份、月份、月中日、小时、分钟和秒这七个属性。Oracle 使用内部格式来表示这些信息，因此它存储的并不是 2005 年 6 月 25 日 12:01:00 对应的 20, 05, 06, 25, 12, 01, 00。使用内置的 `DUMP` 函数，我们可以查看 Oracle 实际存储的内容：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t ( x date );
Table created.
SQL> insert into t (x) values
( to_date( '25-jun-2005 12:01:00',
'dd-mon-yyyy hh24:mi:ss' ) );
1 row created.
SQL> select x, dump(x,10) d from t;
X         D
--------- -----------------------------------
25-JUN-05 Typ=12 Len=7: 120,105,6,25,13,2,1
```

世纪和年份字节（`DUMP` 输出中的 120,105）以加 100 表示法存储。你需要从中减去 100 才能确定正确的世纪和年份。采用加 100 表示法是为了支持公元前（BC）和公元（AD）日期。如果你从世纪字节中减去 100 得到一个负数，那么它就是一个公元前日期。例如：

```
SQL> insert into t (x) values( to_date( '01-jan-4712bc', 'dd-mon-yyyybc hh24:mi:ss' ) );
1 row created.
SQL> select x, dump(x,10) d from t;
X         D
--------- -----------------------------------
25-JUN-05 Typ=12 Len=7: 120,105,6,25,13,2,1
01-JAN-12 Typ=12 Len=7: 53,88,1,1,1,1,1
```

因此，当我们插入 01-JAN-4712BC 时，世纪字节是 53，而 53 – 100 = –47，这就是我们插入的世纪。因为它是负数，我们知道这是一个公元前日期。这种存储格式也使得日期在二进制意义上可以自然排序。由于公元前 4712 年*小于*公元前 4710 年，我们希望二进制表示也能支持这一点。通过转储这两个日期，我们可以看到公元前 4710 年 1 月 1 日比公元前 4712 年同一天*更大*，因此它们可以很好地排序和比较：

```
SQL> insert into t (x) values ( to_date( '01-jan-4710bc', 'dd-mon-yyyybc hh24:mi:ss' ) );
1 row created.
SQL> select x, dump(x,10) d from t;
X         D
--------- -----------------------------------
25-JUN-05 Typ=12 Len=7: 120,105,6,25,13,2,1
01-JAN-12 Typ=12 Len=7: 53,88,1,1,1,1,1
01-JAN-10 Typ=12 Len=7: 53,90,1,1,1,1,1
```

接下来的两个字段，即月份和日字节，是自然存储的，没有任何修改。因此，6 月 25 日使用月份字节 6 和日字节 25。小时、分钟和秒字段以加 1 表示法存储，这意味着我们必须从每个组件中减去 1 才能看到实际时间。因此，午夜在日期字段中表示为 1,1,1。

正如你所看到的，这种 7 字节格式可以自然排序——它是一个可以按二进制方式从小到大（或反之）高效排序的 7 字节字段。此外，其结构允许轻松截断，而无需将日期转换为其他格式。例如，将我们刚刚存储的日期（25-JUN-2005 12:01:00）截断到日（移除小时、分钟、秒）非常简单。只需将最后三个字节设置为 1,1,1，时间组件就相当于被擦除了。考虑一个新表 `T`，包含以下插入操作：

```
SQL> create table t ( what varchar2(10), x date );
Table created.
SQL> insert into t (what, x) values ( 'orig', to_date( '25-jun-2005 12:01:00', 'dd-mon-yyyy hh24:mi:ss' ) );
1 row created.
SQL> insert into t (what, x)
select 'minute', trunc(x,'mi') from t
union all
select 'day', trunc(x,'dd') from t
union all
select 'month', trunc(x,'mm') from t
union all
select 'year', trunc(x,'y') from t;
4 rows created.
SQL> select what, x, dump(x,10) d from t;
WHAT     X         D
-------- --------- -----------------------------------
orig     25-JUN-05 Typ=12 Len=7: 120,105,6,25,13,2,1
minute   25-JUN-05 Typ=12 Len=7: 120,105,6,25,13,2,1
day      25-JUN-05 Typ=12 Len=7: 120,105,6,25,1,1,1
month    01-JUN-05 Typ=12 Len=7: 120,105,6,1,1,1,1
year     01-JAN-05 Typ=12 Len=7: 120,105,1,1,1,1,1
```

为了将该日期截断到年份，数据库所需做的就是在最后 5 个字节中放入 1——这是一个非常快的操作。我们现在得到了一个可排序、可比较的、被截断到年份级别的 `DATE` 字段，而且是以最高效的方式获得的。

### 对 DATE 进行加减时间

我经常被问到的一个问题是：“如何对 `DATE` 类型进行加减时间？”例如，如何给一个 `DATE` 加一天，或八小时，或一年，或一个月等等。你通常会使用三种技术：

*   直接给 `DATE` 加一个 `NUMBER`。给 `DATE` 加 1 是增加一天的方法。因此，给 `DATE` 加 1/24 就是增加一小时，依此类推。

*   你可以使用 `INTERVAL` 类型（稍后描述）来增加时间单位。`INTERVAL` 类型支持两种粒度：年和月，或者天/小时/分钟/秒。也就是说，你可以有一个包含若干年和月的间隔，*或者*一个包含若干天、小时、分钟和秒的间隔。

*   使用内置的 `ADD_MONTHS` 函数添加月份。由于添加一个月通常不像加上 28 到 31 天那么简单，因此实现了一个特殊用途的函数来方便此操作。

表 12-3 展示了你将用来向日期添加（当然也可以减去）N 个时间单位的技术。

表 12-3
向日期添加时间

| 时间单位 | 操作 | 描述 |
| --- | --- | --- |
| N 秒 | `DATE + n/24/60/60` <br> `DATE + n/86400` <br> `DATE + NUMTODSINTERVAL(n,'second')` | 一天有 86,400 秒。因为加 1 表示加一天，所以加 1/86400 就是加一秒。我更喜欢 n/24/60/60 技术，而不是 1/86400 技术。它们是等价的。一个更具可读性的方法是使用 `NUMTODSINTERVAL`（数字转天/秒间隔）函数来加 N 秒。 |
| N 分钟 | `DATE + n/24/60` <br> `DATE + n/1440` <br> `DATE + NUMTODSINTERVAL(n,'minute')` | 一天有 1440 分钟。因此加 1/1440 就是给 `DATE` 加一分钟。一个更具可读性的方法是使用 `NUMTODSINTERVAL` 函数。 |
| N 小时 | `DATE + n/24` <br> `DATE + NUMTODSINTERVAL(n,'hour')` | 一天有 24 小时。因此加 1/24 就是给 `DATE` 加一小时。一个更具可读性的方法是使用 `NUMTODSINTERVAL` 函数。 |
| N 天 | `DATE + n` | 直接给 `DATE` 加 N 来加或减 N 天。 |
| N 周 | `DATE + 7*n` | 一周是七天，所以只需将 7 乘以要加或减的周数。 |
| N 月 | `ADD_MONTHS(DATE,n)` <br> `DATE + NUMTOYMINTERVAL(n,'month')` | 你可以使用 `ADD_MONTHS` 内置函数，或者给 `DATE` 加一个 N 月的间隔。请参阅后面关于对 `DATE` 使用月份间隔的重要注意事项。 |
| N 年 | `ADD_MONTHS(DATE,12*n)` <br> `DATE + NUMTOYMINTERVAL(n,'year')` | 你可以使用 `ADD_MONTHS` 内置函数配合 `12*n` 来加或减 N 年。使用年份间隔也可以达到类似目的，但请参阅后面关于对日期使用年份间隔的重要注意事项。 |

一般来说，在使用 Oracle `DATE` 类型时，我建议：

*   使用 `NUMTODSINTERVAL` 内置函数来加小时、分钟和秒。
*   加一个简单的数字来加天。
*   使用 `ADD_MONTHS` 内置函数来加月和年。

我不推荐使用 `NUMTOYMINTERVAL` 函数（来加月和年）。原因与这些函数在月末的行为有关。

`ADD_MONTHS` 函数会特殊处理月末的日子。它实际上会为我们四舍五入日期——如果我们给一个有 31 天的月份加一个月，而下个月不足 31 天，`ADD_MONTHS` 将返回下个月的最后一天。此外，给一个月的最后一天加一个月会得到下个月的最后一天。当我们给一个有 30 天或更少天数的月份加一个月时，可以看到这一点：


## `ADD_MONTHS` 与 日期间隔

```
$ sqlplus eoda/foo@PDB1
SQL> alter session set nls_date_format = 'dd-mon-yyyy hh24:mi:ss';
会话已更改。
SQL> select dt, add_months(dt,1) from (select to_date('29-feb-2000','dd-mon-yyyy') dt from dual );
DT                   ADD_MONTHS(DT,1)
-------------------- --------------------
29-feb-2000 00:00:00 31-mar-2000 00:00:00
SQL> select dt, add_months(dt,1) from (select to_date('28-feb-2001','dd-mon-yyyy') dt from dual );
DT                   ADD_MONTHS(DT,1)
-------------------- --------------------
28-feb-2001 00:00:00 31-mar-2001 00:00:00
SQL> select dt, add_months(dt,1) from (select to_date('30-jan-2001','dd-mon-yyyy') dt from dual );
DT                   ADD_MONTHS(DT,1)
-------------------- --------------------
30-jan-2001 00:00:00 28-feb-2001 00:00:00
SQL> select dt, add_months(dt,1) from (select to_date('30-jan-2000','dd-mon-yyyy') dt from dual );
DT                   ADD_MONTHS(DT,1)
-------------------- --------------------
30-jan-2000 00:00:00 29-feb-2000 00:00:00
```

可以看到，将 2000 年 2 月 29 日加上一个月，结果是 2000 年 3 月 31 日？2 月 29 日是该月的最后一天，所以`ADD_MONTHS`返回了下个月的最后一天。此外，请注意将 2000 年 1 月 30 日和 2001 年 1 月 30 日加上一个月，分别得到的是 2000 年 2 月和 2001 年 2 月的最后一天。

如果我们比较一下加上一个间隔（interval）会如何工作，会看到非常不同的结果：

```
SQL> select dt, dt+numtoyminterval(1,'month') from (select to_date('29-feb-2000','dd-mon-yyyy') dt from dual);
DT                   DT+NUMTOYMINTERVAL(1
-------------------- --------------------
29-feb-2000 00:00:00 29-mar-2000 00:00:00
SQL> select dt, dt+numtoyminterval(1,'month') from (select to_date('28-feb-2001','dd-mon-yyyy') dt from dual);
DT                   DT+NUMTOYMINTERVAL(1
-------------------- --------------------
28-feb-2001 00:00:00 28-mar-2001 00:00:00
```

注意，结果日期不是下个月的最后一天，而是下个月的`相同`日期。可以说这种行为是可接受的，但考虑一下当结果月份没有那么多天时会发生什么：

```
SQL> select dt, dt+numtoyminterval(1,'month') from (select to_date('30-jan-2001','dd-mon-yyyy') dt from dual);
select dt, dt+numtoyminterval(1,'month')
*
第 1 行出现错误:
ORA-01839: 指定的月份日期无效

SQL> select dt, dt+numtoyminterval(1,'month') from (select to_date('30-jan-2000','dd-mon-yyyy') dt from dual);
select dt, dt+numtoyminterval(1,'month')
*
第 1 行出现错误:
ORA-01839: 指定的月份日期无效
```

根据我的经验，这使得在日期算术中通用性地使用月间隔变得不可能。年间隔也会出现类似问题：将 2000 年 2 月 29 日加上一年，会导致运行时错误，因为 2001 年 2 月 29 日不存在。

## 计算两个 DATE 之间的差值

另一个常被问到的问题是，“如何检索两个日期之间的差值？”答案看似简单：直接相减即可。这将返回一个表示两个日期之间天数的数字。此外，你还有内置函数`MONTHS_BETWEEN`，它将返回一个表示两个日期之间月数（包括小数月份）的数字。最后，借助`INTERVAL`数据类型，你还有另一种方法来查看两个日期之间经过的时间。以下 SQL 查询演示了将两个日期相减（显示它们之间的天数）、使用`MONTHS_BETWEEN`函数以及使用`INTERVAL`类型相关的两个函数的结果：

```
SQL> select dt2-dt1 ,
months_between(dt2,dt1) months_btwn,
numtodsinterval(dt2-dt1,'day') days,
numtoyminterval(trunc(months_between(dt2,dt1)),'month') months
from (select to_date('29-feb-2000 01:02:03','dd-mon-yyyy hh24:mi:ss') dt1, to_date('15-mar-2001 11:22:33','dd-mon-yyyy hh24:mi:ss') dt2
from dual );
DT2-DT1 MONTHS_BTWN DAYS                           MONTHS
---------- ----------- ------------------------------ -------------
380.430903  12.5622872 +000000380 10:20:30.000000000  +000000001-00
```

这些都是正确的值，但对我们来说还不够有用。大多数应用程序希望显示两个日期之间的年、月、日、小时、分钟和秒。通过组合使用前面的函数，我们可以实现这个目标。我们将选取两个间隔：一个用于年和月，另一个仅用于日、小时等。我们将使用内置函数`MONTHS_BETWEEN`来确定两个日期之间的小数月份值，然后使用内置函数`NUMTOYMINTERVAL`将该数字转换为年和月。此外，我们将使用`MONTHS_BETWEEN`从较大的日期中减去两个日期之间的整数月份值，以得到它们之间的天数和小时数：

```
SQL> select numtoyminterval
(trunc(months_between(dt2,dt1)),'month')
years_months,
numtodsinterval
(dt2-add_months( dt1, trunc(months_between(dt2,dt1)) ),
'day' )
days_hours
from (select to_date('29-feb-2000 01:02:03','dd-mon-yyyy hh24:mi:ss') dt1,
to_date('15-mar-2001 11:22:33','dd-mon-yyyy hh24:mi:ss') dt2
from dual );
YEARS_MONTHS    DAYS_HOURS
--------------- ------------------------------
+000000001-00   +000000015 10:20:30.000000000
```

现在很清楚，这两个`DATE`之间相差 1 年、15 天、10 小时、20 分钟和 30 秒。

### TIMESTAMP 类型

`TIMESTAMP`类型与`DATE`非常相似，增加了对小数秒和时区的支持。我们将在以下三个部分中探讨`TIMESTAMP`类型：一个部分仅关于小数秒支持但不支持时区，另外两个部分关于存储带时区支持的`TIMESTAMP`的两种方法。

#### TIMESTAMP

基本的`TIMESTAMP`数据类型语法很简单：

```
TIMESTAMP(n)
```

其中`N`是可选的；它用于指定时间戳中小数秒的精度，可取值在 0 到 9 之间。如果指定 0，则`TIMESTAMP`在功能上等同于`DATE`，实际上，它以相同方式存储相同的值：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t
( dt   date,
ts   timestamp(0));
表已创建。
SQL> insert into t values ( sysdate, systimestamp );
已创建 1 行。
SQL> select dump(dt,10) dump, dump(ts,10) dump from t;
DUMP
--------------------------------------------------------------------------------
DUMP
--------------------------------------------------------------------------------
Typ=12 Len=7: 120,121,7,17,22,31,15
Typ=180 Len=7: 120,121,7,17,22,31,16
```

数据类型不同（由`Typ=字段`指示），但它们存储数据的方式是相同的。当您指定要保留的一些小数秒位数时，`TIMESTAMP`数据类型的长度将与`DATE`类型不同，例如：

```
SQL> create table t
( dt   date,
ts   timestamp(9));
表已创建。
SQL> insert into t values ( sysdate, systimestamp );
已创建 1 行。
SQL> select dump(dt,10) dump, dump(ts,10) dump from t;
DUMP
--------------------------------------------------------------------------------
DUMP
--------------------------------------------------------------------------------
Typ=12 Len=7: 120,121,7,17,24,47,9            Typ=180 Len=11: 120,121,7,17,24,47,9,11,126,246,112
```

现在`TIMESTAMP`消耗 11 字节的存储空间，末尾的额外 4 字节包含小数秒，我们可以通过查看存储的时间来看到：

```
SQL> alter session set nls_date_format = 'dd-mon-yyyy hh24:mi:ss';
会话已更改。
SQL> select * from t;
DT
--------------------------------------------------------------------------------
TS
--------------------------------------------------------------------------------
17-jul-2021 23:46:08
17-JUL-21 11.46.08.192870000 PM
SQL> select dump(ts,16) dump from t;
DUMP
--------------------------------------------------------------------------------
Typ=180 Len=11: 78,79,7,11,18,2f,9,b,7e,f6,70
SQL> select to_number('0b7ef670','xxxxxxxx') from dual;
TO_NUMBER('0B7EF670','XXXXXXXX')
-------------------------------
```

我们可以看到存储的小数秒就在最后 4 字节中。这次我们使用了`DUMP`函数以`HEX`（16 进制）检查数据，这样我们可以轻松地将这 4 字节转换为十进制表示。


## 向时间戳（TIMESTAMP）添加或减去时间

我们应用于`DATE`类型进行日期算术的相同技术也适用于`TIMESTAMP`，但使用前述技术时，`TIMESTAMP`在许多情况下会被转换为`DATE`。例如：

```
SQL> alter session set nls_date_format = 'dd-mon-yyyy hh24:mi:ss';
Session altered.
SQL> select systimestamp ts, systimestamp+1 dt
2  from dual;
TS                                       DT

18-JUL-21 12.00.56.623614 AM +00:00      19-jul-2021 00:00:56
```

请注意，加 1 确实使`SYSTIMESTAMP`前进了一天，但小数秒部分丢失了，时区信息也是如此。这时使用`INTERVAL`（时间间隔）会更为重要：

```
SQL> select systimestamp ts, systimestamp +numtodsinterval(1,'day') dt  from dual;
TS                                       DT

18-JUL-21 12.00.56.625828 AM +00:00     19-JUL-21 12.00.56.625828000 AM +00:00
```

使用返回`INTERVAL`类型的函数保留了`TIMESTAMP`的精确度。使用`TIMESTAMP`时需要格外小心以避免隐式转换。但请记住这个警告：如果向`TIMESTAMP`添加月份或年份间隔，而结果日期不是有效日期——操作将会失败（例如，如果通过`INTERVAL`添加月份，向一月的最后一天添加一个月总会失败）。

## 获取两个时间戳（TIMESTAMP）之间的差值

这是`DATE`和`TIMESTAMP`类型显著不同的地方。两个`DATE`相减的结果是一个`NUMBER`，而两个`TIMESTAMP`相减的结果是一个`INTERVAL`：

```
$ sqlplus eoda/foo@PBD1
SQL> select dt2-dt1
from (select to_timestamp('29-feb-2000 01:02:03.122000',
'dd-mon-yyyy hh24:mi:ss.ff') dt1,
to_timestamp('15-mar-2001 11:22:33.000000',
'dd-mon-yyyy hh24:mi:ss.ff') dt2
from dual );
DT2-DT1

+000000380 10:20:29.878000000
```

两个`TIMESTAMP`值之间的差值是一个`INTERVAL`，它向我们展示了两者之间的天数以及小时/分钟/秒数。如果我们希望得到年、月等信息，就需要回到与处理日期类似的查询：

```
SQL> select numtoyminterval
(trunc(months_between(dt2,dt1)),'month')
years_months,
dt2-add_months(dt1,trunc(months_between(dt2,dt1)))
days_hours
from (select to_timestamp('29-feb-2000 01:02:03.122000',
'dd-mon-yyyy hh24:mi:ss.ff') dt1,
to_timestamp('15-mar-2001 11:22:33.000000',
'dd-mon-yyyy hh24:mi:ss.ff') dt2
from dual );
YEARS_MONTHS  DAYS_HOURS
------------- -----------------------------
+000000001-00 +000000015 10:20:30.000000000
```

请注意，在这种情况下，由于我们使用了`ADD_MONTHS`，`DT1`被隐式转换为`DATE`类型，我们丢失了小数秒。我们需要添加更多的代码来保留它们。我们可以使用`NUMTOYMINTERVAL`来添加月份并保留`TIMESTAMP`；然而，这样可能会遇到运行时错误：

```
SQL> select numtoyminterval
(trunc(months_between(dt2,dt1)),'month')
years_months,
dt2-(dt1 + numtoyminterval( trunc(months_between(dt2,dt1)),'month' ))
days_hours
from (select to_timestamp('29-feb-2000 01:02:03.122000',
'dd-mon-yyyy hh24:mi:ss.ff') dt1,
to_timestamp('15-mar-2001 11:22:33.000000',
'dd-mon-yyyy hh24:mi:ss.ff') dt2
from dual );
dt2-(dt1 + numtoyminterval( trunc(months_between(dt2,dt1)),'month' ))
*
ERROR at line 4:
ORA-01839: date not valid for month specified
```

我个人认为这是不可接受的。但事实是，当你需要显示包含年和月的信息时，`TIMESTAMP`的精确度已经被破坏了。一年的长度不是固定的（可能是 365 或 366 天），一个月也不是。如果你要显示包含年和月的信息，微秒的丢失就无关紧要了；此时显示到秒的信息已经绰绰有余。

## 带时区的时间戳（TIMESTAMP WITH TIME ZONE）类型

`TIMESTAMP WITH TIME ZONE`类型继承了`TIMESTAMP`类型的所有特性，并增加了时区支持。`TIMESTAMP WITH TIME ZONE`类型占用 13 字节的存储空间，多出的 2 字节用于保存时区信息。它与`TIMESTAMP`在结构上的区别仅在于增加了这 2 字节：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t
( ts    timestamp,
ts_tz timestamp with time zone );
Table created.
SQL> insert into t ( ts, ts_tz )  values ( systimestamp, systimestamp );
1 row created.
SQL> select * from t;
TS                                  TS_TZ
----------------------------------- -----------------------------------
18-JUL-21 12.05.15.818195 AM        18-JUL-21 12.05.15.818195 AM +00:00
SQL> select dump(ts) dump, dump(ts_tz) dump from t;
UMP                                                         DUMP

Typ=180 Len=11: 120,121,7,18,1,6,16,48,196,170,56
Typ=181 Len=13: 120,121,7,18,1,6,16,48,196,170,56,20,60
```

在检索时，默认的`TIMESTAMP WITH TIME ZONE`格式包含了时区信息（执行此操作时我使用的是美国山区夏令时）。

`TIMESTAMP WITH TIME ZONE`以数据存储时指定的任何时区来存储数据。时区成为数据本身的一部分。请注意带`TIME ZONE`的`TIMESTAMP`比`TIMESTAMP`列多存储了两个字节。检索时会使用末尾的这 2 个字节来正确调整`TIMESTAMP`值。

本书无意涵盖时区的所有细微差别；那是一个在其他地方已有充分论述的话题。为此，我只想指出，这种数据类型提供了对时区的支持。这种支持在当今的应用程序中比以往任何时候都更有意义。在遥远的过去，应用程序远不如现在这样全球化。在互联网广泛使用之前的时代，应用程序通常是分布式和去中心化的，时区隐含地基于服务器所在的位置。今天，随着大型集中式系统被全球用户使用，跟踪和使用时区的需求变得非常现实。

在时区支持被内置到数据类型之前，应用程序需要负责在一个列中存储`DATE`，在另一个列中存储时区信息，然后提供函数将`DATE`从一个时区转换到另一个时区。现在这是数据库的工作了，它可以将数据存储在多个时区中：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t
( ts1  timestamp with time zone,
ts2  timestamp with time zone);
Table created.
SQL> insert into t (ts1, ts2)
values ( timestamp'2014-02-27 16:02:32.212 US/Eastern',
timestamp'2014-02-27 16:02:32.212 US/Pacific' );
1 row created.
```

并且可以对它们执行正确的`TIMESTAMP`算术运算：

```
SQL> select ts1-ts2 from t;
TS1-TS2

-000000000 03:00:00.000000
```

由于这两个时区之间有三小时的时差，即使它们显示的时间都是 16:02:32.212，报告的间隔也是三小时的差异。在对`TIMESTAMP WITH TIME ZONE`类型执行`TIMESTAMP`算术运算时，Oracle 会自动将两种类型都转换为 UTC（协调世界时）时间，然后执行操作。


## TIMESTAMP WITH LOCAL TIME ZONE 类型

该类型的工作方式与 `TIMESTAMP` 列非常相似。它是一个 7 字节或 11 字节的字段（取决于 `TIMESTAMP` 的精度），但存储时会标准化为本地数据库的时区。为了演示这一点，我们将再次使用 `DUMP` 命令。首先，我们创建一个包含三列的表——一列 `DATE`，一列 `TIMESTAMP WITH TIME ZONE`，和一列 `TIMESTAMP WITH LOCAL TIME ZONE`——然后我们将相同的值插入到这三列中：

```
$ sqlplus eoda/foo@PDB1
create table t
( dt   date,
ts1  timestamp with time zone,
ts2  timestamp with local time zone);
Table created.
SQL> insert into t (dt, ts1, ts2)
values ( timestamp'2014-02-27 16:02:32.212 US/Pacific',
timestamp'2014-02-27 16:02:32.212 US/Pacific',
timestamp'2014-02-27 16:02:32.212 US/Pacific' );
1 row created.
SQL> select dbtimezone from dual;
DBT

MST
```

现在，当我们如下转储这些值时：

```
SQL> select dump(dt), dump(ts1), dump(ts2) from t;
DUMP(DT)

DUMP(TS1)

DUMP(TS2)

Typ=12 Len=7: 120,114,2,27,17,3,33
Typ=181 Len=13: 120,114,2,28,1,3,33,12,162,221,0,137,156
Typ=231 Len=11: 120,114,2,27,18,3,33,12,162,221,0
```

我们可以看到，在此情况下，存储了三种完全不同的日期/时间表示形式：

*   `DT`：此列存储了日期/时间 27-FEB-2014 16:02:32。由于我们使用了 `DATE` 类型，时区和小数秒丢失了。根本没有进行任何时区转换。我们存储了插入时的确切日期/时间，但丢失了时区信息。
*   `TS1`：此列保留了 `TIME ZONE` 信息，并被标准化为相对于该 `TIME ZONE` 的 UTC 时间。插入的 `TIMESTAMP` 值位于美国太平洋时区，在撰写本文时，该时区比 UTC 晚八小时。因此，存储的日期/时间是 28-FEB-2014 00:02:32。它将我们的输入时间提前了八小时以使其成为 UTC 时间，并将时区 US/Pacific 作为最后 2 个字节保存，以便之后可以正确解释此数据。
*   `TS2`：此列 `被假定为数据库的时区，即 US/Mountain`。现在，16:02:32 US/Pacific 等于 17:02:32 US/Mountain，因此这就是存储在字节 `...18,3,33...` 中的值（超 1 表示法；记得减 1）。

由于 `TS1` 列在最后 2 个字节中保留了原始时区，我们在检索时将看到以下内容：

```
SQL> select ts1, ts2 from t;
TS1

TS2

27-FEB-14 04.02.32.212000 PM US/PACIFIC
27-FEB-14 05.02.32.212000 PM
```

数据库将能够显示该信息，但带有 `LOCAL TIME ZONE` 的 `TS2` 列（数据库的时区）以数据库的时区显示时间，该时区是此列（实际上也是此数据库中所有带有 `LOCAL TIME ZONE` 的列）的假定时区。我的数据库处于美国山地时区，因此输入时的 16:02:32 US/Pacific 在输出时显示为下午 5:00 山地时间。

注意
如果日期存储时是标准时间生效期间，而检索时是夏令时生效期间，您可能会得到略有不同的结果。前面示例中的输出将显示两个小时的差异，而不是您直观认为的一小时差异。我指出这一点只是为了强调，时区计算远比它看起来复杂得多！

`TIMESTAMP WITH LOCAL TIME ZONE` 为大多数应用程序提供了足够的支持，前提是您不需要记住源时区，而只需要一种能提供全球一致的日期/时间类型处理的数据类型。此外，`TIMESTAMP(0) WITH LOCAL TIME ZONE` 为您提供了等同于支持时区的 `DATE` 类型——它消耗 7 字节存储，并能够以标准化的 UTC 形式存储日期。

关于 `TIMESTAMP WITH LOCAL TIME ZONE` 类型的一个警告是，一旦您创建了包含此列的表，您将发现数据库的时区被冻结了——并且您将无法更改它：

```
SQL> alter database set time_zone = 'PST';
alter database set time_zone = 'PST'
*
ERROR at line 1:
ORA-30079: cannot alter database timezone when database has
TIMESTAMP WITH LOCAL TIME ZONE columns
SQL> !oerr ora 30079
30079, 00000, "cannot alter database timezone when database has
TIMESTAMP WITH LOCAL TIME ZONE columns"
// *Cause:  An attempt was made to alter database timezone with
//          TIMESTAMP WITH LOCAL TIME ZONE column in the database.
// *Action: Either do not alter database timezone or first drop all the
//          TIMESTAMP WITH LOCAL TIME ZONE columns.
```

原因应该很明显：如果您要更改数据库的时区，您将必须重写每一个带有 `TIMESTAMP WITH LOCAL TIME ZONE` 列的表，因为考虑到新的时区，它们当前的值将是错误的！

