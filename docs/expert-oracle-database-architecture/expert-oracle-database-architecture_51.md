# 索引组织表的性能与选项

以下性能对比展示了使用索引组织表（IOT）与堆组织表在`闩锁活动`方面的显著差异，其中`Run1`使用了`HEAP_ADDRESSES`表，而`Run2`使用了`IOT_ADDRESSES`表。

```
STAT...no work - consistent re       338,579         72,476        -266,103
STAT...consistent gets from ca       675,052        408,920        -266,132
STAT...consistent gets               675,052        408,920        -266,132
STAT...session logical reads         675,191        409,035        -266,156
STAT...table fetch by rowid          336,435         67,283        -269,152
STAT...buffer is not pinned co       605,596        269,132        -336,464
LATCH.cache buffers chains         1,014,203        481,954        -532,249
STAT...temp space allocated (b     1,048,576              0      -1,048,576
STAT...logical read bytes from 5,531,164,672  3,350,814,720  -2,180,349,952
Run1 latches total versus runs -- difference and pct
Run1               Run2              Diff        Pct
1,064,843          497,082           -567,761    214.22%
```

正如您所见，`闩锁活动`显著且可重复地减少了，这主要归功于`高速缓存缓冲区链闩锁`（用于保护缓冲区缓存的那个闩锁）。在这种情况下，IOT 可以提供以下好处：

*   提高了缓冲区缓存的效率，因为任何给定的查询只需要在缓存中保留更少的数据块。
*   减少了对缓冲区缓存的访问，从而提高了可扩展性。
*   检索数据所需的总体工作量减少，因为速度更快。
*   每次查询的物理 I/O 可能更少，因为任何给定查询所需的独立数据块更少，并且一次对地址的物理 I/O 很可能就能检索到所有需要的数据（不像堆表实现那样，一次只能检索一个）。

## 使用 BETWEEN 查询的场景

如果您经常对主键或唯一键使用`BETWEEN`查询，同样的优势也适用。物理存储排序的数据也会提高这些查询的性能。例如，我在数据库中维护了一个股票报价表。每天，我为数百只股票收集股票代码、日期、收盘价、当日最高价、最低价、成交量和其他相关信息。表结构如下：

```sql
SQL> create table stocks
( ticker      varchar2(10),
day         date,
value       number,
change      number,
high        number,
low         number,
vol         number,
primary key(ticker,day) )
organization index;
Table created.
```

我经常一次查看一只股票在一段时间内的数据（例如，计算移动平均线）。如果我使用堆组织表，两只相同股票代码（如`ORCL`）的行恰好存在于同一个数据库块中的概率几乎为零。这是因为每天晚上，我都会插入所有股票当天的记录。这至少会填满一个数据库块（实际上，是许多块）。因此，我每天添加一条新的`ORCL`记录，但它所在的块与表中已有的所有其他`ORCL`记录都不同。如果我执行如下查询：

```sql
SQL> Select * from stocks
where ticker = 'ORCL'
and day between sysdate-100 and sysdate;
```

Oracle 会读取索引，然后通过`rowid`访问表以获取行的其余数据。由于我加载表的方式，我检索的 100 行中的每一行都将位于不同的数据库块上——每一次访问很可能都是一次物理 I/O。

现在考虑我将相同的数据存储在 IOT 中。同样的查询只需要读取相关的索引块，因为所有数据已经包含在索引中。不仅表访问被移除了，而且特定日期范围内的所有`ORCL`行在物理存储上也是彼此邻近的。因此，产生的逻辑 I/O 和物理 I/O 都更少。

## IOT 的选项与注意事项

现在您了解了何时以及如何使用 IOT。接下来您需要理解这些表有哪些选项以及需要注意什么（注意事项）。这些选项与堆组织表的选项非常相似。我们将再次使用`DBMS_METADATA`来查看细节。让我们从 IOT 的三个基本变体开始：

```sql
$ sqlplus eoda/foo@PDB1
SQL> create table t1
(  x int primary key,
y varchar2(25),
z date)
organization index;
Table created.
SQL> create table t2
(  x int primary key,
y varchar2(25),
z date)
organization index
OVERFLOW;
Table created.
SQL> create table t3
(  x int primary key,
y varchar2(25),
z date)
organization index
overflow INCLUDING y;
Table created.
```

我们将详细解释`溢出段`和`INCLUDING`子句的作用，但首先让我们查看第一张表的详细 SQL 定义：

```sql
SQL> set long 100000
SQL> select dbms_metadata.get_ddl( 'TABLE', 'T1' ) from dual;
DBMS_METADATA.GET_DDL('TABLE','T1')

CREATE TABLE "EODA"."T1"
(    "X" NUMBER(*,0),
"Y" VARCHAR2(25),
"Z" DATE,
PRIMARY KEY ("X") ENABLE
) SEGMENT CREATION DEFERRED
ORGANIZATION INDEX NOCOMPRESS PCTFREE 10 INITRANS 2 MAXTRANS 255 LOGGING
STORAGE(
BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
TABLESPACE "USERS"
PCTTHRESHOLD 50
```

这个表引入了一个新选项`PCTTHRESHOLD`，我们稍后会探讨。您可能注意到前面的`CREATE TABLE`语法中缺少了一些东西：没有`PCTUSED`子句，但有`PCTFREE`。这是因为索引是一种复杂的数据结构，不像堆那样是随机组织的，数据必须存放在它所属的位置。与堆不同（堆中有时有空闲块可供插入），在索引中，新条目总有可用的空间。如果数据根据其值属于某个给定的数据块，无论该块有多满或多空，数据都会存放到那里。此外，`PCTFREE`仅在创建对象并向索引结构中填充数据时使用。它的使用方式与在堆组织表中不同。`PCTFREE`会在新创建的索引上保留空间，但不会用于后续的操作。我们在堆组织表中考虑过的关于`FREELIST`的因素同样完全适用于 IOT。

### NOCOMPRESS 与 COMPRESS 选项

首先，我们来看`NOCOMPRESS`选项。这个选项的实现方式与前面讨论的表压缩不同。它对索引组织表的任何操作都有效（而表压缩对常规路径操作可能有效也可能无效）。使用`NOCOMPRESS`，它告诉 Oracle 存储索引条目中的每一个值（即，不压缩）。如果对象的主键由`A`、`B`和`C`列组成，那么`A`、`B`和`C`的每一次出现都会被物理存储。`NOCOMPRESS`的反面是`COMPRESS N`，其中`N`是一个整数，表示要压缩的列数。这会移除重复的值，并在块级别进行因子分解，因此重复出现的`A`和可能的`B`的值不再被物理存储。例如，考虑如下创建的表：

```sql
SQL> create table iot
( owner, object_type, object_name,
primary key(owner,object_type,object_name))
organization index
NOCOMPRESS
as
select distinct owner, object_type, object_name from all_objects;
Table created.
```

仔细思考，`OWNER`的值被重复了成百上千次。每个模式（`OWNER`）往往拥有许多对象。甚至`OWNER, OBJECT_TYPE`这个值对也会重复很多次，因为一个给定的模式会有几十张表、几十个包等等。只有三个列组合在一起才不会重复。我们可以让 Oracle 抑制这些重复值。我们不希望一个索引块包含表 10-1 所示的值，而是可以使用`COMPRESS 2`（提取前两列作为因子），得到一个包含表 10-2 所示值的块。

**表 10-2：索引叶子块，COMPRESS 2**

| Sys,table | t1 | t2 | t3 |
| --- | --- | --- | --- |
| t4 | t5 | … | … |
| … | t103 | t104 | … |
| t300 | t301 | t302 | t303 |

**表 10-1：索引叶子块，NOCOMPRESS**


