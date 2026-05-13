# 数字类型

Oracle 支持三种适用于存储数字的原生数据类型：

*   `NUMBER`：Oracle `NUMBER` 类型能够以极高的精度存储数字——实际上是 38 位精度。其底层数据格式类似于*压缩十进制*（`packed decimal`）表示法。Oracle `NUMBER` 类型是变长格式，长度从 0 到 22 字节不等。它适用于存储小至 10e-130 和大到但不包括 10e126 的任何数字。这是迄今为止当今使用最广泛的 `NUMBER` 类型。

*   `BINARY_FLOAT`：这是一种 IEEE 原生单精度浮点数。在磁盘上，它将消耗 5 字节的存储：浮点数占 4 个固定字节，长度字节占 1 个字节。它能够以大约 ± 10^(38.53) 的范围和六位精度存储数字。

*   `BINARY_DOUBLE`：这是一种 IEEE 原生双精度浮点数。在磁盘上，它将消耗 9 字节的存储：浮点数占 8 个固定字节，长度字节占 1 个字节。它能够以大约 ± 10^(308.25) 的范围和 13 位精度存储数字。

从这个简要概述可以看出，Oracle `NUMBER` 类型的精度远高于 `BINARY_FLOAT` 和 `BINARY_DOUBLE` 类型，但范围比 `BINARY_DOUBLE` 小得多。也就是说，你可以在 `NUMBER` 类型中非常精确地存储具有许多有效数字的数字，但你可以在 `BINARY_FLOAT` 和 `BINARY_DOUBLE` 类型中存储更小和更大的数字。举一个简单的例子，我们可以创建一个包含各种数据类型的表，看看在相同输入下存储的内容是什么：

```sql
$ sqlplus eoda/foo@PDB1
SQL> create table t
  2  ( num_col   number,
  3    float_col binary_float,
  4    dbl_col   binary_double);
Table created.
SQL> insert into t ( num_col, float_col, dbl_col )
  2  values ( 1234567890.0987654321, 1234567890.0987654321, 1234567890.0987654321 );
1 row created.
SQL> set numformat 99999999999.99999999999
SQL> select * from t;
NUM_COL                FLOAT_COL                  DBL_COL
------------------------ ------------------------ ------------------------
1234567890.09876543210   1234567940.00000000000   1234567890.09876540000
```

请注意，`NUM_COL` 返回了我们作为输入提供的确切数字。输入数字的有效数字少于 38 位（我提供了一个有 20 位有效数字的数字），因此精确的数字被保留了。然而，使用 `BINARY_FLOAT` 类型的 `FLOAT_COL` 无法准确表示这个数字。事实上，它只准确保留了七位数字。`DBL_COL` 的表现要好得多，在这种情况下准确地表示了 17 位数字。总的来说，这应该可以很好地说明 `BINARY_FLOAT` 和 `BINARY_DOUBLE` 类型不适合金融应用！如果你尝试不同的值，你会看到不同的结果：

```sql
SQL> delete from t;
1 row deleted.
SQL> insert into t ( num_col, float_col, dbl_col )
  2  values ( 9999999999.9999999999, 9999999999.9999999999, 9999999999.9999999999);
1 row created.
SQL> select * from t;
NUM_COL                FLOAT_COL                  DBL_COL
------------------------ ------------------------ ------------------------
9999999999.99999999990  10000000000.00000000000  10000000000.00000000000
```

再次说明，`NUM_COL` 准确地表示了数字，但 `FLOAT_COL` 和 `DBL_COL` 没有。这并不意味着 `NUMBER` 类型能够以无限的准确性/精度存储东西——只是它关联的精度要大得多。很容易从 `NUMBER` 类型观察到类似的结果：

```sql
SQL> delete from t;
1 row deleted.
SQL> insert into t ( num_col ) values ( 123 * 1e20 + 123*1e-20 ) ;
1 row created.
SQL> set numformat 999999999999999999999999.999999999999999999999999
SQL> select num_col, 123*1e20, 123*1e-20 from t;
NUM_COL                     123*1E20                   123*1E-20
------------------------ ------------------------ ------------------------
12300000000000000000000.000000000000000000000000
12300000000000000000000.000000000000000000000000
                            .000000000000000001230000
```

如你所见，当我们把一个非常大的数字（`123*1e20`）和一个非常小的数字（`123*1e-20`）组合在一起时，我们丢失了精度，因为这次算术运算需要超过 38 位的精度。大数字本身可以被忠实表示，小数字也可以，但大数加小数的结果却不行。我们可以如下验证这不仅仅是一个显示/格式问题：

```sql
SQL> select num_col from t where num_col = 123*1e20;
NUM_COL
------------------------
12300000000000000000000.000000000000000000000000
```

`NUM_COL` 中的值等于 `123*1e20`，而不是我们试图插入的值。


## NUMBER 类型语法与用法

`NUMBER` 类型的语法很简单：

```
NUMBER( p,s )
```

`P` 和 `S` 是可选参数，用于指定以下内容：

*   Precision（精度），即总位数：默认精度是 38，有效值范围是 1 到 38。字符 `*` 也可以用来表示 38。
*   Scale（小数位数），即小数点右边的位数：小数位数的有效值是 –84 到 127，其默认值取决于是否指定了精度。如果未指定精度，则小数位数默认为最大范围。如果指定了精度，则小数位数默认为 0（小数点右边没有数字）。因此，例如，定义为 `NUMBER` 的列存储浮点数（带小数位），而 `NUMBER(38)` 只存储整数数据（无小数），因为后一种情况中小数位数默认为 0。

你应该将精度和小数位数视为数据的*编辑器*——某种数据完整性工具。精度和小数位数*完全不影响*数据在磁盘上的存储方式，只影响允许哪些值以及数字如何舍入。例如，如果某个值超过了允许的精度，Oracle 会返回错误：

```
SQL> create table t ( num_col number(5,0) );
Table created.
SQL> insert into t (num_col) values ( 12345 );
1 row created.
SQL> insert into t (num_col) values ( 123456 );
insert into t (num_col) values ( 123456 )
*
ERROR at line 1:
ORA-01438: value larger than specified precision allowed for this column
```

因此，你可以使用精度来强制执行某些数据完整性约束。在这个例子中，`NUM_COL` 是一个不允许超过五位数的列。

另一方面，小数位数用于控制数字的舍入。例如：

```
SQL> create table t ( msg varchar2(10), num_col number(5,2) );
Table created.
SQL> insert into t (msg,num_col) values ( '123.45',  123.45 );
1 row created.
SQL> insert into t (msg,num_col) values ( '123.456', 123.456 );
1 row created.
SQL> select * from t;
MSG           NUM_COL
---------- ----------
123.45         123.45
123.456        123.46
```

注意，这次数字 123.456 超过五位数却成功插入了。这是因为在本例中使用的小数位数将 123.456 舍入为两位数，得到 123.46，*然后* 123.46 再根据精度进行验证，发现符合要求，于是被插入。然而，如果我们尝试执行以下插入操作，它会失败，因为数字 1234.00 总共超过了五位数：

```
SQL> insert into t (msg,num_col) values ( '1234', 1234 );
insert into t (msg,num_col) values ( '1234', 1234 )
*
ERROR at line 1:
ORA-01438: value larger than specified precision allowed for this column
```

当你指定小数位数为 2 时，小数点左边最多允许三位数字，右边两位。因此，该数字不符合要求。`NUMBER(5,2)` 列可以容纳介于 999.99 和 –999.99 之间的所有值。

允许小数位数从 –84 到 127 可能看起来很奇怪。负的小数位数有什么用？它允许你舍入小数点左边的值。正如 `NUMBER(5,2)` 将值舍入到最接近的 0.01 一样，`NUMBER(5,-2)` 会舍入到最接近的 100，例如：

```
SQL> create table t ( msg varchar2(10), num_col number(5,-2) );
Table created.
SQL> insert into t (msg,num_col) values ( '123.45',  123.45 );
1 row created.
SQL> insert into t (msg,num_col) values ( '123.456', 123.456 );
1 row created.
SQL> select * from t;
MSG           NUM_COL
---------- ----------
123.45            100
123.456           100
```

这些数字被舍入到了最接近的 100。我们仍然有五位数的精度，但现在小数点左边允许有七位数（包括末尾的两个 0）：

```
SQL> insert into t (msg,num_col) values ( '1234567', 1234567 );
1 row created.
SQL> select * from t;
MSG           NUM_COL
---------- ----------
123.45            100
123.456           100
1234567       1234600
SQL> insert into t (msg,num_col) values ( '12345678', 12345678 );
insert into t (msg,num_col) values ( '12345678', 12345678 )
*
ERROR at line 1:
ORA-01438: value larger than specified precision allowed for this column
```

因此，精度决定了舍入后数字中允许的位数，使用小数位数来确定如何舍入。精度是一种完整性约束，而小数位数是一种编辑。

有趣且有用的是，`NUMBER` 类型实际上在磁盘上是一种变长数据类型，将消耗 0 到 22 字节的存储空间。很多时候，程序员认为数值数据类型是定长类型——这是他们在使用 2 字节或 4 字节整数以及 4 字节或 8 字节浮点数编程时通常看到的情况。Oracle `NUMBER` 类型类似于变长字符串。我们可以观察包含不同数量有效数字的数字会发生什么。我们将创建一个包含两个 `NUMBER` 列的表，并在第一列中填充许多具有 2、4、6、...、28 位有效数字的数字。然后，我们将简单地给每个数字加 1：

```
SQL> create table t ( x number, y number );
Table created.
SQL> insert into t ( x )
select to_number(rpad('9',rownum*2,'9'))
from all_objects
where rownum  update t set y = x+1;
14 rows updated.
```

现在，如果我们使用内置的 `VSIZE` 函数（该函数显示列占用的存储量），我们可以查看每行中两个数字的大小差异：

```
SQL> set numformat 99999999999999999999999999999
SQL> column v1 format 99
SQL> column v2 format 99
SQL> select x, y, vsize(x) v1, vsize(y) v2
2    from t order by x;
X                              Y  V1  V2
------------------------------ ------------------------------ --- ---
99                            100   2   2
9999                          10000   3   2
999999                        1000000   4   2
99999999                      100000000   5   2
9999999999                    10000000000   6   2
999999999999                  1000000000000   7   2
99999999999999                100000000000000   8   2
9999999999999999              10000000000000000   9   2
999999999999999999            1000000000000000000  10   2
99999999999999999999          100000000000000000000  11   2
9999999999999999999999        10000000000000000000000  12   2
999999999999999999999999      1000000000000000000000000  13   2
99999999999999999999999999    100000000000000000000000000  14   2
9999999999999999999999999999  10000000000000000000000000000  15   2
14 rows selected.
```

我们可以看到，随着我们给 `X` 增加有效数字，所需的存储空间越来越大。每增加两位有效数字，就多消耗一个字节。但一个仅比它大一的数字始终只占用 2 字节。当 Oracle 存储一个数字时，它会尽可能少地存储来表示该数字。它通过存储有效数字、用于放置小数点的指数以及数字的符号信息（正或负）来实现这一点。因此，一个数字包含的*有效*数字越多，它消耗的存储空间就越大。

最后一个事实解释了为什么了解数字存储在变长字段中很有用。在尝试估算表的大小时（例如，弄清楚表中 1,000,000 行需要多少存储空间），你必须仔细考虑 `NUMBER` 字段。你的数字会占用 2 字节还是 20 字节？平均大小是多少？这使得在没有代表性测试数据的情况下准确估算表的大小非常困难。你可以得到最坏情况大小和最佳情况大小，但实际大小很可能介于两者之间。

### BINARY_FLOAT/BINARY_DOUBLE 类型语法与用法

`BINARY_FLOAT` 和 `BINARY_DOUBLE` 是许多程序员习惯使用的 IEEE 标准浮点数。关于这些数字类型的完整描述及其底层实现，我建议阅读 [`http://en.wikipedia.org/wiki/Floating-point`](http://en.wikipedia.org/wiki/Floating-point)。在该参考资料中，关于浮点数基本定义的以下内容值得注意（强调为我所加）：

> *浮点数是有理数一个特定子集上的数字的表示方法，常用于在计算机上**近似**表示任意实数。具体来说，它表示一个尾数（significand，非正式地也称为 mantissa）乘以一个基数（在计算机中通常为 2）的整数次幂（指数）。当基数为 2 时，它是科学记数法（基数为 10）的二进制类比。*

它们用于*近似*数字；其精度远不及前面描述的内置 Oracle `NUMBER` 类型。浮点数常用于科学应用，并且由于其运算可以在硬件（CPU，芯片）中完成，而非在 Oracle 子程序中完成，因此在许多类型的应用中非常有用。因此，如果您在科学应用中进行真正的数值计算，算术运算会快得多，但您不会想使用浮点数来存储财务信息。例如，假设您想将 0.3 和 0.1 作为浮点数相加。您可能会认为答案当然是 0.4。但您错了（在浮点算术中）。答案比 0.4 稍大一点：

```
$ sqlplus eoda/foo@PDB1
SQL> select to_char( 0.3f + 0.1f, '0.99999999999999' ) from dual;
TO_CHAR(0.3F+0.1F
------------------
0.40000000600000
```

这不是一个错误，这就是 IEEE 浮点数的运作方式。因此，它们适用于特定领域的问题，但绝对不适用于计算美元和美分的问题！

在表中声明此类型的列非常简单：

```
BINARY_FLOAT
BINARY_DOUBLE
```

就是这样。这些类型没有任何选项。

### 非原生数字类型

除了 `NUMBER`、`BINARY_FLOAT` 和 `BINARY_DOUBLE` 类型外，Oracle 在语法上还支持以下数值数据类型：

*   `NUMERIC(p,s)`：完全映射到 `NUMBER(p,s)`。如果未指定 `p`，则默认为 38。
*   `DECIMAL(p,s)` *或* `DEC(p,s)`：完全映射到 `NUMBER(p,s)`。如果未指定 `p`，则默认为 38。
*   `INTEGER` *或* `INT`：完全映射到 `NUMBER(38)` 类型。
*   `SMALLINT`：完全映射到 `NUMBER(38)` 类型。
*   `FLOAT(p)`：映射到 `NUMBER` 类型。
*   `DOUBLE PRECISION`：映射到 `NUMBER` 类型。
*   `REAL`：映射到 `NUMBER` 类型。

注意

当我说“语法上支持”时，我的意思是 `CREATE` 语句可以使用这些数据类型，但在底层，它们实际上都是 `NUMBER` 类型。Oracle 原生只有三种数值格式。任何其他数值数据类型的使用最终都映射到原生的 Oracle `NUMBER` 类型。

### 性能考虑

一般来说，Oracle `NUMBER` 类型是大多数应用的最佳选择。然而，该类型也存在性能影响。Oracle `NUMBER` 类型是一种*软件数据类型*——它在 Oracle 软件本身中实现。我们无法使用原生硬件操作来将两个 `NUMBER` 类型相加，因为这是在软件中模拟的。而浮点类型则没有这种实现。当我们把两个浮点数相加时，Oracle 会使用硬件来执行操作。

这一点很容易看出来。如果我们创建一个包含大约 70,000 行的表，并使用 `NUMBER` 和 `BINARY_FLOAT`/`BINARY_DOUBLE` 类型将相同的数据放入其中，如下所示：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t
  2  ( num_type     number,
  3    float_type   binary_float,
  4    double_type  binary_double);
Table created.
SQL> insert /*+ APPEND */  into t select rownum, rownum, rownum from all_objects;
72089 rows created.
SQL> commit;
Commit complete.
```

然后我们针对每种类型的列执行相同的查询，使用像 `LN`（自然对数）这样的复杂数学函数。我们在 `TKPROF` 报告中观察到截然不同的 CPU 使用率：

```
select sum(ln(num_type)) from t

call     count       cpu    elapsed
------- ------  -------- ----------
total        4      4.45       4.66

select sum(ln(float_type)) from t

call     count       cpu    elapsed
------- ------  -------- ----------
total        4      0.07       0.08

select sum(ln(double_type)) from t

call     count       cpu    elapsed
------- ------  -------- ----------
total        4      0.06       0.06
```

在此示例中，Oracle `NUMBER` 类型使用的 CPU 是浮点类型的约 63 倍。但是，您必须记住，三个查询返回的答案并不完全相同！

```
SQL> set numformat 999999.9999999999999999
SQL> select sum(ln(num_type)) from t;
SUM(LN(NUM_TYPE))
-----------------
734280.3209126472927309

SQL> select sum(ln(double_type)) from t;
SUM(LN(DOUBLE_TYPE))
--------------------
734280.3209126447300000
```

浮点数是对数字的近似，具有 6 到 13 位的精度。`NUMBER` 类型得到的答案比浮点数精确得多。然而，当您进行数据挖掘或科学数据的复杂数值分析时，这种精度损失通常是可接受的，并且可以获得显著的性能提升。

注意

如果您对浮点算术及其后续精度损失的细节感兴趣，请参阅 [`https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html`](https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html)。

应当注意，在这种情况下，我们或许可以鱼与熊掌兼得。使用内置的 `CAST` 函数，我们可以在执行复杂数学运算之前，将 Oracle `NUMBER` 类型实时转换为浮点类型。这导致的 CPU 使用率非常接近原生浮点类型：

```
select sum(ln(cast( num_type as binary_double ) )) from t

call     count       cpu    elapsed
------- ------  -------- ----------
total        4      0.08       0.08
```

这意味着我们可以非常精确地存储数据，当需要原始速度，并且浮点类型显著优于 Oracle `NUMBER` 类型时，我们可以使用 `CAST` 函数来实现该目标。

## Long 类型

Oracle 中的 `LONG` 类型有两种：

*   一种 `LONG` 文本类型，能够存储 2GB 的文本。存储在 `LONG` 类型中的文本受字符集转换影响，类似于 `VARCHAR2` 或 `CHAR` 类型。
*   一种 `LONG RAW` 类型，能够存储 2GB 的原始二进制数据（不受字符集转换影响的数据）。

`LONG` 类型可以追溯到 Oracle 版本 6，当时它们被限制为 64KB 数据。在版本 7 中，它们被增强到支持最多 2GB 的存储，但到版本 8 发布时，它们已被 LOB 类型取代，我们稍后将讨论。

与其解释如何使用 `LONG` 类型，不如解释为什么您不想在应用程序中使用 `LONG`（或 `LONG RAW`）类型。首先，Oracle 文档在处理 `LONG` 类型时非常明确。*Oracle Database SQL Language Reference* 手册中声明：

> *不要创建带有 LONG 列的表。而是使用 LOB 列（CLOB，NCLOB，BLOB）。LONG 列仅出于向后兼容性而支持。*

### LONG 和 LONG RAW 类型的限制

`LONG`和`LONG RAW`类型受表 12-2 中概述的限制约束。尽管可能显得有些超前，但我还是添加了一列来说明作为`LONG`/`LONG RAW`类型替代品的相应`LOB`类型是否受到相同的限制。

表 12-2
Long 类型与 LOBs 类型对比

| LONG/LONG RAW 类型 | CLOB/BLOB 类型 |
| --- | --- |
| 每个表只能有一个`LONG`或`LONG RAW`列。 | 每个表最多可以有 1000 个`CLOB`或`BLOB`类型的列。 |
| 用户定义类型不能定义具有`LONG`/`LONG RAW`类型的属性。 | 用户定义类型可以完全使用`CLOB`和`LOB`类型。 |
| `LONG`类型不能在`WHERE`子句中被引用。 | `LOB`s 可以在`WHERE`子句中被引用，并且`DBMS_LOB`包中提供了大量函数来操作它们。 |
| `LONG`类型不支持分布式事务。 | `LOB`s 支持分布式事务。 |
| `LONG`类型不能使用基本或高级复制功能进行复制。 | `LOB`s 完全支持复制。 |
| `LONG`列不能出现在`GROUP BY`、`ORDER BY`或`CONNECT BY`子句中，也不能出现在使用`DISTINCT`、`UNIQUE`、`INTERSECT`、`MINUS`或`UNION`的查询中。 | `LOB`s 可以出现在这些子句中，条件是对`LOB`应用一个函数，将其转换为标量 SQL 类型（包含原子值），例如`VARCHAR2`、`NUMBER`或`DATE`。 |
| PL/SQL 函数/过程不能接受`LONG`类型的输入。 | PL/SQL 可以完全与`LOB`类型配合使用。 |
| SQL 内置函数不能用于`LONG`列（例如`SUBSTR`）。 | SQL 函数可以用于`LOB`类型。 |
| 不能在`CREATE TABLE AS SELECT`语句中使用`LONG`类型。 | `LOB`s 支持`CREATE TABLE AS SELECT`。 |
| 不能对包含`LONG`类型的表使用`ALTER TABLE MOVE`。 | 可以移动包含`LOB`s 的表。 |

如你所见，表 12-2 列出了相当长的限制清单；当表中存在`LONG`列时，很多事情你根本无法做。对于所有新的应用程序，甚至不要考虑使用`LONG`类型。相反，应使用适当的`LOB`类型。对于现有的应用程序，如果你遇到了表 12-2 中的任何限制，你应该认真考虑将`LONG`类型转换为相应的`LOB`类型。已经采取了措施来提供向后兼容性，因此为`LONG`类型编写的应用程序可以透明地适用于`LOB`类型。

> **注意**
> 几乎不言而喻，在将生产系统从`LONG`类型修改为`LOB`类型之前，你应该对你的应用程序进行完整的功能测试。

### 处理遗留的 LONG 类型

一个经常出现的问题是：“Oracle 中的数据字典怎么办？”其中充斥着`LONG`列，这使得使用字典列变得有问题。例如，无法使用 SQL 在`ALL_VIEWS`字典视图中搜索包含文本`HELLO`的所有视图：

```
$ sqlplus eoda/foo@PDB1
SQL> select * from all_views where text like '%HELLO%';
where text like '%HELLO%'
*
ERROR at line 3:
ORA-00932: inconsistent datatypes: expected CHAR got LONG
```

这个问题不仅限于`ALL_VIEWS`视图；许多视图都受到影响：

```
SQL> select table_name, column_name
from dba_tab_columns
where data_type in ( 'LONG', 'LONG RAW' )
and owner = 'SYS'
and table_name like 'DBA%'
order by table_name;
TABLE_NAME                     COLUMN_NAME
------------------------------ ------------------------------
DBA_ADVISOR_SQLPLANS           OTHER
DBA_ARGUMENTS                  DEFAULT_VALUE
DBA_CLUSTER_HASH_EXPRESSIONS   HASH_EXPRESSION
DBA_CONSTRAINTS                SEARCH_CONDITION
DBA_TRIGGERS_AE                TRIGGER_BODY
DBA_VIEWS                      TEXT
DBA_VIEWS_AE                   TEXT
DBA_ZONEMAPS                   QUERY
DBA_ZONEMAP_MEASURES           MEASURE
32 rows selected.
```

那么，解决方案是什么？如果你想在 SQL 中利用这些列，那么你需要将它们转换为 SQL 友好的类型。你可以使用一个用户定义的函数来实现这一点。以下示例演示了如何使用一个`LONG SUBSTR`函数来完成此操作，该函数将允许你将`LONG`类型的任何 4000 字节有效转换为`VARCHAR2`以供 SQL 使用。完成后，你将能够查询：

```
SQL> select * from (
select owner, view_name,
long_help.substr_of( 'select text
from dba_views
where owner = :owner
and view_name = :view_name',
1, 4000,
'owner', owner,
'view_name', view_name ) substr_of_view_text
from dba_views
where owner = user
)
where upper(substr_of_view_text) like '%INNER%';
```

你已经将`TEXT`列的前 4000 字节从`LONG`转换为`VARCHAR2`，现在可以对它使用谓词了。使用相同的技术，你还可以为`LONG`类型实现自己的`INSTR`、`LIKE`等函数。在本书中，我将只演示如何获取`LONG`类型的子字符串。

我们将实现的包具有以下规范：

```
SQL> create or replace package long_help
authid current_user
as
function substr_of
( p_query in varchar2,
p_from  in number,
p_for   in number,
p_name1 in varchar2 default NULL,
p_bind1 in varchar2 default NULL,
p_name2 in varchar2 default NULL,
p_bind2 in varchar2 default NULL,
p_name3 in varchar2 default NULL,
p_bind3 in varchar2 default NULL,
p_name4 in varchar2 default NULL,
p_bind4 in varchar2 default NULL )
return varchar2;
end;
/
Package created.
```

注意第 2 行，我们指定了`AUTHID CURRENT_USER`。这使得包以调用者的身份运行，并拥有所有角色和权限。这一点很重要，原因有二。首先，我们希望数据库安全性不被破坏——这个包将只返回我们（调用者）被允许看到的列的子字符串。具体来说，这意味着这个包不易受到 SQL 注入攻击——它不是以包所有者的身份运行，而是以调用者的身份运行。其次，我们希望在数据库中安装一次这个包，并让其功能可供所有人使用；使用调用者权限可以让我们做到这一点。如果我们使用 PL/SQL 的默认安全模型——定义者权限——包将以包所有者的权限运行，这意味着它只能看到包所有者能够看到的数据，而这可能不包括调用者被允许看到的数据集合。


函数 `SUBSTR_OF` 背后的概念是处理一个最多只选择一行一列的查询：即我们感兴趣的 `LONG` 值。`SUBSTR_OF` 将在需要时解析该查询，为其绑定任何输入，并以编程方式获取结果，返回 `LONG` 值的必要部分。

## 包主体实现

包主体（即实现）以两个全局变量开始。`G_CURSOR` 变量持有在整个会话期间保持打开的持久游标。这是为了避免反复打开和关闭游标，以及避免过度解析 SQL。第二个全局变量 `G_QUERY` 用于记录我们在此包中最后解析的 SQL 查询文本。只要查询保持不变，我们只会解析它一次。因此，即使我们在一个查询中查询 5000 行，只要传递给此函数的 SQL 查询没有改变，我们也只会进行一次解析调用：

```
SQL> create or replace package body long_help
as
g_cursor number := dbms_sql.open_cursor;
g_query  varchar2(32765);
```

接下来，此包中是一个私有过程 `BIND_VARIABLE`，我们将用它来绑定调用者传递给我们的输入。我们将其作为一个单独的私有过程来实现只是为了方便；我们只想在输入名称 `NOT NULL` 时才进行绑定。与其在代码中为每个输入参数检查四次，不如在这个过程中检查一次：

```
procedure bind_variable( p_name in varchar2, p_value in varchar2 )
is
begin
if ( p_name is not null )
then
dbms_sql.bind_variable( g_cursor, p_name, p_value );
end if;
end;
```

接下来是包主体中 `SUBSTR_OF` 的实际实现。此例程以包规范中的函数声明和一些局部变量的声明开始。`L_BUFFER` 将用于返回值，`L_BUFFER_LEN` 将用于保存 Oracle 提供函数返回的长度：

```
function substr_of
( p_query in varchar2,
p_from  in number,
p_for   in number,
p_name1 in varchar2 default NULL,
p_bind1 in varchar2 default NULL,
p_name2 in varchar2 default NULL,
p_bind2 in varchar2 default NULL,
p_name3 in varchar2 default NULL,
p_bind3 in varchar2 default NULL,
p_name4 in varchar2 default NULL,
p_bind4 in varchar2 default NULL )
return varchar2
as
l_buffer       varchar2(4000);
l_buffer_len   number;
begin
```

现在，我们的代码首先对 `P_FROM` 和 `P_FOR` 输入进行合理性检查。`P_FROM` 必须是一个大于或等于 1 的数字，`P_FOR` 必须在 1 到 4000 之间——就像内置函数 `SUBSTR` 一样：

```
if ( nvl(p_from,0) not between 1 and 4000 )
then
raise_application_error
(-20002, 'From must be a positive number' );
end if;
if ( nvl(p_for,0) not between 1 and 4000 )
then
raise_application_error
(-20003, 'For must be between 1 and 4000' );
end if;
```

接下来，我们将检查是否收到需要解析的新查询。如果我们最后解析的查询与当前查询相同，我们可以跳过此步骤。需要注意的是，在第 47 行，我们正在验证传递给我们的 `P_QUERY` 只是一个 `SELECT`——我们将*仅*使用此包来执行 SQL `SELECT` 语句。此检查为我们验证了这一点：

```
if ( p_query <> g_query or g_query is NULL )
then
if ( upper(trim(nvl(p_query,'x'))) not like 'SELECT%')
then
raise_application_error
(-20001, 'This must be a select only' );
end if;
dbms_sql.parse( g_cursor, p_query, dbms_sql.native );
g_query := p_query;
end if;
```

我们准备好将输入绑定到此查询。任何传递给我们的非 `NULL` 名称都将被绑定到查询，这样当我们执行它时，它能找到正确的行：

```
bind_variable( p_name1, p_bind1 );
bind_variable( p_name2, p_bind2 );
bind_variable( p_name3, p_bind3 );
bind_variable( p_name4, p_bind4 );
```

现在我们可以执行查询并获取行了。使用 `DBMS_SQL.COLUMN_VALUE_LONG`，我们提取 `LONG` 的必要子串并将其返回：

```
dbms_sql.define_column_long(g_cursor, 1);
if (dbms_sql.execute_and_fetch(g_cursor)>0)
then
dbms_sql.column_value_long
(g_cursor, 1, p_for, p_from-1,
l_buffer, l_buffer_len );
end if;
return l_buffer;
end substr_of;
end;
/
Package body created.
```

## 使用示例

就是这样——您应该能够将此包用于数据库中*任何*遗留的 `LONG` 列，从而执行许多以前无法进行的 `WHERE` 子句操作。例如，您现在可以查找您模式中所有分区，其 `HIGH_VALUE` 包含 2014 年（请记住，如果没有分区高值包含 2014 的表，您不会期望看到任何返回结果）：

```
SQL> select * from (
select table_owner, table_name, partition_name,
long_help.substr_of
( 'select high_value
from all_tab_partitions
where table_owner = :o
and table_name = :n
and partition_name = :p',
1, 4000,
'o', table_owner,
'n', table_name,
'p', partition_name ) high_value
from all_tab_partitions
where table_owner = user
)
where high_value like '%2014%';
TABLE_OWNER TABLE_NAME  PARTITION_NAME HIGH_VALUE
----------- ----------- -------------- --------------------
EODA        F_CONFIGS   CONFIG_P_7     20140101
```

使用相同的技术——即在函数中处理返回单个 `LONG` 列的单行查询结果——您可以根据需要实现自己的 `INSTR`、`LIKE` 等功能。

## 局限性与替代方案

此实现适用于 `LONG` 类型，但不适用于 `LONG RAW` 类型。`LONG RAW` 不支持分段访问（`DBMS_SQL` 中没有 `COLUMN_VALUE_LONG_RAW` 函数）。幸运的是，这不是一个非常严重的限制，因为数据字典中不使用 `LONG RAW`，并且需要对其进行“子串”操作以便搜索的情况也很少见。不过，如果您确实需要这样做，除非 `LONG RAW` 在 32KB 或以下，否则您不能使用 PL/SQL，因为 PL/SQL 本身没有处理超过 32KB 的 `LONG RAW` 的方法。必须使用 Java、C、C++、Visual Basic 或其他语言。

另一种方法是使用 `TO_LOB` 内置函数和全局临时表，将 `LONG` 或 `LONG RAW` 临时转换为 `CLOB` 或 `BLOB`。您的 PL/SQL 过程可以如下所示：

```
Insert into global_temp_table ( blob_column )
select to_lob(long_raw_column) from t where...
```

这在偶尔需要处理单个 `LONG RAW` 值的应用程序中效果很好。但是，由于涉及的工作量，您不希望持续这样做。如果您发现自己需要频繁使用此技术，您绝对应该一次性将 `LONG RAW` 转换为 `BLOB`。

