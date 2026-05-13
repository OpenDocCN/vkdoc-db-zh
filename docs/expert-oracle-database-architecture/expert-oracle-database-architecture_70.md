# INTERVAL 类型

我们在上一节简要看到了 `INTERVAL` 类型的使用。它是一种表示时间跨度或时间间隔的方式。我们将在此节讨论两种间隔类型：`YEAR TO MONTH` 类型，能够存储以年和月指定的时间跨度；以及 `DAY TO SECOND` 类型，能够存储以天、小时、分钟和秒（包括小数秒）指定的时间跨度。

在深入两种 `INTERVAL` 类型的具体细节之前，我想先看看 `EXTRACT` 内置函数，它在处理这种类型时非常有用。`EXTRACT` 内置函数适用于 `TIMESTAMP` 和 `INTERVAL`，并从中返回各种信息，例如从 `TIMESTAMP` 中提取时区，或从 `INTERVAL` 中提取小时/天/分钟。让我们使用前面的例子，其中我们得到了 380 天 10 小时 20 分 29.878 秒的 `INTERVAL`：

```
SQL> select dt2-dt1
from (select to_timestamp('29-feb-2000 01:02:03.122000',
'dd-mon-yyyy hh24:mi:ss.ff') dt1,
to_timestamp('15-mar-2001 11:22:33.000000',
'dd-mon-yyyy hh24:mi:ss.ff') dt2
from dual );
DT2-DT1

+000000380 10:20:29.878000000
```

我们可以使用 `EXTRACT` 来看看提取每一段信息是多么容易：

```
SQL> select extract( day    from dt2-dt1 ) day,
extract( hour   from dt2-dt1 ) hour,
extract( minute from dt2-dt1 ) minute,
extract( second from dt2-dt1 ) second
from (select to_timestamp('29-feb-2000 01:02:03.122000',
'dd-mon-yyyy hh24:mi:ss.ff') dt1,
to_timestamp('15-mar-2001 11:22:33.000000',
'dd-mon-yyyy hh24:mi:ss.ff') dt2
from dual );
DAY       HOUR     MINUTE     SECOND
---------- ---------- ---------- ----------
380         10         20     29.878
```

此外，我们已经见过用于创建 `YEAR TO MONTH` 和 `DAY TO SECOND` 间隔的 `NUMTOYMINTERVAL` 和 `NUMTODSINTERVAL`。我发现这些函数是创建 `INTERVAL` 类型实例的最简单方法——甚至超过了字符串转换函数。与其将代表某个间隔的天数、小时数、分钟数和秒数的一堆数字连接起来，我更愿意累加四个 `NUMTODSINTERVAL` 调用来完成同样的事情。

`INTERVAL` 类型不仅可以存储时长，在某种程度上也可以存储时间。例如，如果您想存储特定的日期和时间，您有 `DATE` 或 `TIMESTAMP` 类型。但如果您只想存储上午 8:00 这个时间点呢？`INTERVAL` 类型（特别是 `INTERVAL DAY TO SECOND` 类型）对此会很方便。



