# INTERVAL 类型

## INTERVAL YEAR TO MONTH

`INTERVAL` `YEAR TO MONTH` 的语法很简单：

```
INTERVAL YEAR(n) TO MONTH
```

其中 `N` 是可选的数字位数，用于指定年的位数，范围从 0 到 9，默认为 2（存储 0 到 99 的年数）。它允许你存储任意年数（最多九位数）和月数。我更倾向于使用 `NUMTOYMINTERVAL` 函数来创建此类型的 `INTERVAL` 实例。例如，要创建一个五年零两个月的间隔，我们可以使用以下方法：

```
SQL> select numtoyminterval(5,'year')+numtoyminterval(2,'month') from dual;
NUMTOYMINTERVAL(5,'YEAR')+NUMTOYMINTERVAL(2,'MONTH')

+000000005-02
```

或者，利用一年有 12 个月这个事实，通过单个函数调用，我们可以采用以下方法：

```
SQL> select numtoyminterval(5*12+2,'month') from dual;
NUMTOYMINTERVAL(5*12+2,'MONTH')

+000000005-02
```

两种方法都很好用。另一个函数 `TO_YMINTERVAL` 可以将字符串转换为年/月 `INTERVAL` 类型：

```
SQL> select to_yminterval( '5-2' ) from dual;
TO_YMINTERVAL('5-2')

+000000005-02
```

但由于大多数情况下，我的应用程序中年和月是两个 `NUMBER` 字段，我发现 `NUMTOYMINTERVAL` 函数比从数字构建格式化字符串更有用。最后，你也可以直接在 SQL 中使用 `INTERVAL` 字面量，完全绕过函数：

```
SQL> select interval '5-2' year to month from dual;
INTERVAL'5-2'YEARTOMONTH

+05-02
```

## INTERVAL DAY TO SECOND

`INTERVAL` `DAY TO SECOND` 类型的语法很简单：

```
INTERVAL DAY(n) TO SECOND(m)
```

其中 `N` 是可选的数字位数，用于指定日部分的位数，范围从零到九，默认为二。`M` 是秒字段小数部分中要保留的位数，范围从零到九，默认为六。同样，我更喜欢使用 `NUMTODSINTERVAL` 函数来创建此 `INTERVAL` 类型的实例：

```
SQL> select numtodsinterval( 10, 'day' )+
numtodsinterval( 2, 'hour' )+
numtodsinterval( 3, 'minute' )+
numtodsinterval( 2.3312, 'second' )
from dual;
NUMTODSINTERVAL(10,'DAY')+NUMTODSINTERVAL(2,'HOUR')+NUMTODSINTERVAL(3,'MINU

+000000010 02:03:02.331200000
```

或者简单地

```
SQL> select numtodsinterval( 10*86400+2*3600+3*60+2.3312, 'second' ) from dual;
NUMTODSINTERVAL(10*86400+2*3600+3*60+2.3312,'SECOND')

+000000010 02:03:02.331200000
```

利用一天有 86,400 秒，一小时有 3600 秒等事实。或者，像之前一样，我们可以使用 `TO_DSINTERVAL` 函数将字符串转换为 `DAY TO SECOND` 间隔：

```
SQL> select to_dsinterval( '10 02:03:02.3312' ) from dual;
TO_DSINTERVAL('1002:03:02.3312')

+000000010 02:03:02.331200000
```

或者直接在 SQL 中使用 `INTERVAL` 字面量：

```
SQL> select interval '10 02:03:02.3312' day to second from dual;
INTERVAL'1002:03:02.3312'DAYTOSECOND

+10 02:03:02.331200
```

