# 何时应使用位图索引？

位图索引最适用于*低不同基数*的数据（即，与整个集合的基数相比，具有相对较少离散值的数据）。很难为此设定一个具体的数值——换句话说，很难准确定义什么是真正的低不同基数。在一个包含几千条记录的集合中，2 可能算是低不同基数，但在只有两行的表中，2 就不算低不同基数。在一个拥有数千万条记录的表中，100,000 也可能被认为是低不同基数。因此，低不同基数是相对于结果集大小而言的。这类数据指的是在行集中不同项的数量除以行数后得到一个很小的数值（接近于零）。例如，一个`GENDER`（性别）列可能的取值是`M`、`F`和`NULL`。如果一个表包含 20,000 条员工记录，那么你会看到 3/20000 = 0.00015。同样地，10,000,000 条结果中有 100,000 个唯一值，其比率为 0.01——同样非常小。这些列将是位图索引的候选对象。它们可能*不会*是 B*树索引的候选对象，因为每个值往往会检索出极大比例的表数据。如前所述，B*树索引通常应具有选择性。而位图索引则不应具有选择性——恰恰相反，它们通常应该非常*不具选择性*。

## 用于即席查询的位图索引

位图索引在您有大量即席查询的环境中非常有用，特别是那些以即席方式引用多列或产生如`COUNT`等聚合结果的查询。例如，假设您有一个包含三列的大表：`GENDER`、`LOCATION`和`AGE_GROUP`。在此表中，`GENDER`取值为`M`或`F`，`LOCATION`可取值`1`至`50`，而`AGE_GROUP`是一个代码，代表`18 岁及以下`、`19-25`、`26-30`、`31-40`和`41 岁及以上`。您需要支持大量采用以下形式的即席查询：

```sql
SQL> select count(*)
from t
where gender = 'M'
and location in ( 1, 10, 30 )
and age_group = '41 and over';
SQL> select *
from t
where (   ( gender = 'M' and location = 20 )
or ( gender = 'F' and location = 22 ))
and age_group = '18 and under';
SQL> select count(*) from t where location in (11,20,30);
SQL> select count(*) from t where age_group = '41 and over' and gender = 'F';
```

您会发现传统的 B*树索引方案无法满足需求。如果您想使用索引来获取答案，您将需要至少三种、最多六种可能的 B*树索引组合来通过索引访问数据。由于这三列中的任何一列或任何子集都可能出现在查询条件中，您将需要在以下组合上创建大型连接 B*树索引：

*   `GENDER`、`LOCATION`、`AGE_GROUP`：用于使用所有三列，或`GENDER`与`LOCATION`，或仅`GENDER`的查询
*   `LOCATION`、`AGE_GROUP`：用于使用`LOCATION`和`AGE_GROUP`或仅`LOCATION`的查询
*   `AGE_GROUP`、`GENDER`：用于使用`AGE_GROUP`和`GENDER`或仅`AGE_GROUP`的查询

为了减少搜索的数据量，为了减小被扫描的索引结构的大小，其他排列组合也可能是合理的。这忽略了在如此低基数的数据上创建 B*树索引并非好主意的事实。

## 位图索引的工作原理

这正是位图索引发挥作用的地方。使用三个小型位图索引（每个列一个），您将能够高效地满足所有前述的谓词条件。Oracle 将简单地使用`AND`、`OR`和`NOT`函数，结合这三个索引的位图，来找出引用这三列中任何组合的任何谓词的解集。它会将得到的合并位图（如果需要）将 1 转换为 rowid，然后访问数据（如果您只是统计符合条件的行数，Oracle 将只统计 1 的位数）。

### 示例：创建和使用位图索引

让我们来看一个例子。首先，我们将生成符合我们指定的不同基数的测试数据，为其建立索引并收集统计信息。我们将利用`DBMS_RANDOM`包来生成符合我们分布的随机数据：

```sql
SQL> create table t
( gender not null,
location not null,
age_group not null,
data
)
as
select decode( round(dbms_random.value(1,2)),
1, 'M',
2, 'F' ) gender,
ceil(dbms_random.value(1,50)) location,
decode( round(dbms_random.value(1,5)),
1,'18 and under',
2,'19-25',
3,'26-30',
4,'31-40',
5,'41 and over'),
rpad( '*', 20, '*')
from dual connect by level < 100000
/

Table created.

SQL> create bitmap index gender_idx on t(gender);
Index created.
SQL> create bitmap index location_idx on t(location);
Index created.
SQL> create bitmap index age_group_idx on t(age_group);
Index created.
SQL> exec dbms_stats.gather_table_stats( user, 'T');
PL/SQL procedure successfully completed.
```

### 查询执行计划分析

现在，让我们看看之前各种即席查询的执行计划：

```sql
SQL> set autotrace traceonly explain
SQL> select count(*)  from t
where gender = 'M'
and location in ( 1, 10, 30 )
and age_group = '41 and over';
```

```
Execution Plan

Plan hash value: 320981916

| Id  | Operation                     | Name          | Rows  | Bytes | Cost (%CPU)| ...
|---|---|---|---|---|---|...
|   0 | SELECT STATEMENT              |               |    1  |    13 |    9    (0)| ...
|   1 |  SORT AGGREGATE               |               |    1  |    13 |            | ...
|   2 |   BITMAP CONVERSION COUNT     |               |  608  |  7904 |    9    (0)| ...
|   3 |    BITMAP AND                 |               |       |       |            | ...
|   4 |     BITMAP OR                 |               |       |       |            | ...
|*  5 |      BITMAP INDEX SINGLE VALUE| LOCATION_IDX  |       |       |            | ...
|*  6 |      BITMAP INDEX SINGLE VALUE| LOCATION_IDX  |       |       |            | ...
|*  7 |      BITMAP INDEX SINGLE VALUE| LOCATION_IDX  |       |       |            | ...
|*  8 |     BITMAP INDEX SINGLE VALUE | AGE_GROUP_IDX |       |       |            | ...
|*  9 |     BITMAP INDEX SINGLE VALUE | GENDER_IDX    |       |       |            | ...

Predicate Information (identified by operation id):
   5 - access("LOCATION"=1)
   6 - access("LOCATION"=10)
   7 - access("LOCATION"=30)
   8 - access("AGE_GROUP"='41 and over')
   9 - access("GENDER"='M')
```

这个例子展示了位图索引的强大之处。Oracle 能够识别`location in (1, 10, 30)`，并知道为这三个值读取 location 上的索引，然后在逻辑上对位图中的“位”进行或（OR）操作。接着，它将得到的结果位图与`AGE_GROUP='41 AND OVER'`和`GENDER='M'`的位图进行逻辑与（AND）操作。然后，只需简单地统计 1 的个数，答案就准备好了。

```sql
SQL> select * from t
where (   ( gender = 'M' and location = 20 )
or ( gender = 'F' and location = 22 ))
and age_group = '18 and under';
```

```
Execution Plan

Plan hash value: 705811684

| Id  | Operation                           | Name          |  Rows | Bytes | Cost (%CPU)...
|---|---|---|---|---|---|...
```



|   0 | `SELECT STATEMENT`                    |               |   408 | 13872 |   68    (0)...
|   1 |  `TABLE ACCESS BY INDEX ROWID BATCHED`| `T`             |   408 | 13872 |   68    (0)...
|   2 |   `BITMAP CONVERSION TO ROWIDS`       |               |       |       |            ...
|   3 |    `BITMAP AND`                       |               |       |       |            ...
|*  4 |     `BITMAP INDEX SINGLE VALUE`       | `AGE_GROUP_IDX` |       |       |            ...
|   5 |     `BITMAP OR`                       |               |       |       |            ...
|   6 |      `BITMAP AND`                     |               |       |       |            ...
|*  7 |       `BITMAP INDEX SINGLE VALUE`     | `LOCATION_IDX`  |       |       |            ...
|*  8 |       `BITMAP INDEX SINGLE VALUE`     | `GENDER_IDX`    |       |       |            ...
|   9 |      `BITMAP AND`                     |               |       |       |            ...
|* 10 |       `BITMAP INDEX SINGLE VALUE`     | `LOCATION_IDX`  |       |       |            ...
|* 11 |       `BITMAP INDEX SINGLE VALUE`     | `GENDER_IDX`    |       |       |            ...

`由操作 ID 标识的谓词信息：`

4 - `access("AGE_GROUP"='18 及以下')`
7 - `access("LOCATION"=22)`
8 - `access("GENDER"='F')`
10 - `access("LOCATION"=20)`
11 - `access("GENDER"='M')`
```

这显示了相似的逻辑：执行计划表明，`OR` 连接的条件是通过将适当的位图进行 `AND` 运算，然后再将这些结果进行 `OR` 运算来评估的。再加上一个 `AND` 来满足 `AGE_GROUP='18 AND UNDER'` 的条件，我们就得到了完整的结果。由于我们这次要求获取实际的行数据，Oracle 会将每个位图的 1 和 0 转换为 ROWID，以检索源数据。

在数据仓库或支持大量即席 SQL 查询的大型报表系统中，这种能够同时使用尽可能多有用索引的能力确实非常方便。在这种情况下使用传统的 B*Tree 索引将不会那么常用或可用，并且随着即席查询需要搜索的列数增加，您所需的 B*Tree 索引组合数量也会增加。

然而，有些时候位图索引是 `不` 适用的。它们在读取密集型环境中运行良好，但极不适合写入密集型环境。原因是单个位图索引键条目指向 `许多` 行。如果某个会话修改了索引数据，那么在大多数情况下，该索引条目指向的所有行实际上都会被锁定。Oracle 无法锁定位图索引条目中的单个位；它会锁定整个位图索引条目。任何其他需要更新 `同一个位图索引条目` 的修改操作都将被锁定。这将严重抑制并发性，因为每次更新看起来都会锁定可能数百行，从而阻止它们的位图列被并发更新。它不会像您可能想象的那样锁定 `每一` 行——只是锁定其中的许多行。位图是分块存储的，因此使用前面的 `EMP` 示例，我们可能会发现索引键 `ANALYST` 在索引中出现多次，每次指向数百行。修改 `JOB` 列的某行的更新操作需要独占访问这两个索引键条目：*旧* 值的索引键条目和*新* 值的索引键条目。在该 `UPDATE` 提交之前，这两个条目指向的数百行将无法被其他会话修改。

### 位图连接索引

通常，索引是基于单个表创建的，并且只使用该表的列。位图连接索引打破了这一规则，它允许您使用其他表的列来索引给定的表。实际上，这使您能够在一个索引结构中实现数据的反规范化，而不是在表本身中进行反规范化。

考虑简单的 `EMP` 和 `DEPT` 表。`EMP` 表有一个指向 `DEPT` 的外键（`DEPTNO` 列）。`DEPT` 表有 `DNAME` 属性（部门名称）。最终用户经常会问这样的问题：“销售部门有多少人？”、“谁在销售部门工作？”、“你能展示销售部门绩效排名前 N 的员工吗？”。请注意，他们不会问，“DEPTNO 30 部门有多少人？” 他们不使用这些键值；相反，他们使用人类可读的部门名称。因此，他们最终会运行如下查询：

```sql
$ sqlplus scott/tiger@PDB1
SQL> select count(*)
from emp, dept
where emp.deptno = dept.deptno
and dept.dname = 'SALES';
SQL> select emp.*
from emp, dept
where emp.deptno = dept.deptno
and dept.dname = 'SALES';
```

这些查询几乎必然需要使用传统索引来访问 `DEPT` 表和 `EMP` 表。我们可能会使用 `DEPT.DNAME` 上的索引来查找 `SALES` 行并检索 `SALES` 的 `DEPTNO` 值，然后使用 `EMP.DEPTNO` 上的 `INDEX` 来查找匹配的行；但是，通过使用位图连接索引，我们可以避免所有这些操作。位图连接索引允许我们索引 `DEPT.DNAME` 列，但让该索引指向的不是 `DEPT` 表，而是 `EMP` 表。这是一个相当激进的概念——能够索引来自其他表的属性——这可能会改变您在报表系统中实现数据模型的方式。实际上，您可以鱼与熊掌兼得。您可以保持规范化数据结构的完整性，同时获得反规范化的好处。

这是我们为这个示例创建的索引：

```sql
SQL> create bitmap index emp_bm_idx
on emp( d.dname )
from emp e, dept d
where e.deptno = d.deptno;
Index created.
```

注意 `CREATE INDEX` 的开头看起来“正常”，在表上创建索引 `INDEX_NAME`。但从那里开始，它就偏离了“正常”。我们看到对 `DEPT` 表中一列的引用：`D.DNAME`。我们看到一个 `FROM` 子句，使得这个 `CREATE INDEX` 语句类似于一个查询。我们有一个多表之间的连接条件。这个 `CREATE INDEX` 语句索引的是 `DEPT.DNAME` 列，但是是在 `EMP` 表的上下文中索引的。如果我们提出前面提到的那些问题，我们会发现数据库根本不会访问 `DEPT` 表，它也无需这样做，因为 `DNAME` 列现在存在于指向 `EMP` 表行的索引中。为了说明目的，我们将使 `EMP` 和 `DEPT` 表看起来很大（以避免 CBO 认为它们很小而对其进行全表扫描，而不是使用索引）：

```sql
SQL> begin
dbms_stats.set_table_stats( user, 'EMP',
numrows => 1000000, numblks => 300000 );
dbms_stats.set_table_stats( user, 'DEPT',
numrows => 100000, numblks => 30000 );
dbms_stats.delete_index_stats( user, 'EMP_BM_IDX' );
end;
/
PL/SQL procedure successfully completed.
```

注意

您可能想知道为什么我之前调用了 `DELETE_INDEX_STATS`，这是因为 `CREATE INDEX` 在创建索引时会自动执行 `COMPUTE STATISTICS`。因此，在这种情况下，Oracle 被“欺骗”了——它认为它看到了一个有 1,000,000 行的表，并且上面有一个很小的索引（实际上该表只有 14 行）。索引统计信息是准确的；表统计信息是“伪造的”。我也需要“伪造”索引统计信息——或者我本可以在索引之前向表中加载 1,000,000 条记录。

然后我们将执行我们的查询：

```sql
SQL> set autotrace traceonly explain
SQL> select count(*)
from emp, dept
where emp.deptno = dept.deptno
and dept.dname = 'SALES';
执行计划

计划哈希值: 2538954156
```



| 序号 | 操作                           | 名称       | 行数 | 字节数 | 代价（%CPU）| 时间     |
|---:|:-----------------------------|:-----------|-----:|-------:|------------:|:---------|
|   0 | SELECT STATEMENT            |            |     1 |      3 |    7    (0) | 00:00:01 |
|   1 |  SORT AGGREGATE             |            |     1 |      3 |             |          |
|   2 |   BITMAP CONVERSION COUNT   |            |   250K|   732K |    7    (0) | 00:00:01 |
|*  3 |    BITMAP INDEX SINGLE VALUE| `EMP_BM_IDX` |       |       |             |          |

谓词信息（通过操作 ID 标识）：

3 - access("`EMP`"."`SYS_NC00009$`"='`SALES`')

如您所见，要回答这个特定问题，我们实际上无需访问 `EMP` 或 `DEPT` 表——整个答案直接来自索引本身。回答该问题所需的所有信息都已存在于索引结构中。

此外，我们得以跳过访问 `DEPT` 表，并通过使用 `EMP` 上包含我们所需 `DEPT` 数据的索引，直接访问到所需的行：

```
SQL> select emp.*  from emp, dept
where emp.deptno = dept.deptno
and dept.dname = 'SALES';
Execution Plan

Plan hash value: 4261295901

| Id  | Operation                           | Name       |  Rows | Bytes | Cost (%CPU)...

|   0 | SELECT STATEMENT                    |            | 10000 |  849K | 6139    (1)...
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| `EMP`        | 10000 |  849K | 6139    (1)...
|   2 |   BITMAP CONVERSION TO ROWIDS       |            |       |       |            ...
|*  3 |    BITMAP INDEX SINGLE VALUE        | `EMP_BM_IDX` |       |       |            ...

Predicate Information (identified by operation id):

3 - access("`EMP`"."`SYS_NC00009$`"='`SALES`')
```

位图连接索引有一个前提条件。连接条件必须关联到另一张表的主键或唯一键。在前面的例子中，`DEPT.DEPTNO` 是 `DEPT` 表的主键，并且该主键必须存在；否则，将会发生错误：

```
SQL> create bitmap index emp_bm_idx
on emp( d.dname )
from emp e, dept d
where e.deptno = d.deptno;
from emp e, dept d
*
ERROR at line 3:
ORA-25954: missing primary key or unique constraint on dimension
```

### 位图索引总结

如果有疑问，就试试看（当然是在您的非 OLTP 系统中）。为表添加一个位图索引（或一批索引）并观察其效果是件很简单的事。此外，创建位图索引通常比创建 B*树索引快得多。实践是检验位图索引是否适合您环境的最好方法。我经常被问到，“如何定义低基数？”对此没有一刀切的答案。有时是 100,000 条记录中的 3 个值。有时是 1,000,000 条记录中的 10,000 个值。低基数并不意味着唯一值的个数是单位数。通过实验可以发现位图索引对您的应用是否是个好主意。总的来说，如果您的环境规模庞大、以只读为主，并且包含大量即席查询，那么一组位图索引可能正是您所需要的。

## 基于函数的索引

基于函数的索引使我们能够对计算列进行索引，并在查询中使用这些索引。简而言之，此功能允许您实现不区分大小写的搜索或排序、对复杂方程式进行搜索，并通过实现您自己的函数和运算符然后在其上进行搜索来高效地扩展 SQL 语言。

使用基于函数的索引的原因有很多，其中最主要的是：

*   它们易于实施并能立即带来价值。
*   它们可用于加速现有应用程序，而无需更改其任何逻辑或查询。

以下小节提供了实现基于函数的索引的相关案例。

### 一个简单的基于函数的索引示例

考虑下面的例子。我们想在 `EMP` 表的 `ENAME` 列上执行不区分大小写的搜索。在基于函数的索引出现之前，我们会采用一种截然不同的方式。我们会在 `EMP` 表中添加一个额外的列，例如称为 `UPPER_ENAME`。该列将由数据库触发器在 `INSERT` 和 `UPDATE` 时维护；该触发器只需设置 `NEW.UPPER_NAME := UPPER(:NEW.ENAME)`。这个额外的列会被索引。现在有了基于函数的索引，我们不再需要这个额外的列。

我们首先在 `SCOTT` 模式下创建演示表 `EMP` 的副本，并向其中添加一些数据：

```
$ sqlplus eoda/foo@PDB1
SQL> create table emp as
select *
from scott.emp
where 1=0;
Table created.
SQL> insert into emp
(empno,ename,job,mgr,hiredate,sal,comm,deptno)
select rownum empno,
initcap(substr(object_name,1,10)) ename,
substr(object_type,1,9) JOB,
rownum MGR,
created hiredate,
rownum SAL,
rownum COMM,
(mod(rownum,4)+1)*10 DEPTNO
from all_objects
where rownum < 10000;
9999 rows created.
```

接下来，我们将在 `ENAME` 列的 `UPPER` 值上创建一个索引，这实际上创建了一个不区分大小写的索引：

```
SQL> create index emp_upper_idx on emp(upper(ename));
Index created.
```

最后，我们将分析该表，因为如前所述，我们需要利用 CBO（基于成本的优化器）来使用基于函数的索引。从技术上讲，此步骤并非必需，因为默认会使用 CBO，并且动态采样会收集所需的信息，但收集统计信息是更正确的方法：

```
SQL> exec dbms_stats.gather_table_stats(user,'EMP',cascade=>true);
PL/SQL procedure successfully completed.
```

现在，我们在列的 `UPPER` 值上有了一个索引。任何已经发出如下不区分大小写查询的应用程序都将利用此索引，从而获得索引带来的性能提升：

```
SQL> set autotrace traceonly explain
SQL> select * from emp where upper(ename) = 'KING';
Execution Plan

Plan hash value: 3831183638

| Id  | Operation                           | Name          |  Rows | Bytes | Cost (%CPU)...

|   0 | SELECT STATEMENT                    |               |     2 |   110 |    2    (0)...
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| `EMP`           |     2 |   110 |    2    (0)...
|*  2 |   INDEX RANGE SCAN                  | `EMP_UPPER_IDX` |     2 |       |    1    (0)...

Predicate Information (identified by operation id):

2 - access(`UPPER`("`ENAME`")='`KING`')
```

在此功能可用之前，`EMP` 表中的每一行都会被扫描、转为大写并进行比较。相比之下，有了 `UPPER(ENAME)` 上的索引，查询将常量 `KING` 带到索引，范围扫描少量数据，并通过 rowid 访问表以获取数据。这非常快。

当在列上索引用户编写的函数时，这种性能提升最为明显。例如，假设我们看到类似这样的情况：

```
SQL> select my_function(ename)
from emp
where some_other_function(empno) > 10;
```

这很棒，因为现在我们可以有效地扩展 SQL 语言以包含特定于应用程序的函数。然而，不幸的是，上述查询的性能有时令人失望。假设 `EMP` 表中有 1000 行数据。函数 `SOME_OTHER_FUNCTION` 将在查询期间执行 1000 次，每行执行一次。此外，假设该函数执行需要百分之一秒，那么这个相对简单的查询现在至少需要十秒钟。

让我们看一个真实的例子，我们将在 PL/SQL 中实现一个修改版的 `SOUNDEX` 例程。此外，我们将在过程中使用一个包全局变量作为计数器，这将允许我们执行使用 `MY_SOUNDEX` 函数的查询，并准确查看它被调用了多少次：


```
SQL> create or replace package stats
as
cnt number default 0;
end;
/
程序包已创建。
SQL> create or replace
function my_soundex( p_string in varchar2 ) return varchar2
deterministic
as
l_return_string varchar2(6) default substr( p_string, 1, 1 );
l_char      varchar2(1);
l_last_digit    number default 0;
type vcArray is table of varchar2(10) index by binary_integer;
l_code_table    vcArray;
begin
stats.cnt := stats.cnt+1;
l_code_table(1) := 'BPFV';
l_code_table(2) := 'CSKGJQXZ';
l_code_table(3) := 'DT';
l_code_table(4) := 'L';
l_code_table(5) := 'MN';
l_code_table(6) := 'R';
for i in 1 .. length(p_string)
loop
exit when (length(l_return_string) = 6);
l_char := upper(substr( p_string, i, 1 ) );
for j in 1 .. l_code_table.count
loop
if (instr(l_code_table(j), l_char ) > 0 AND j  l_last_digit)
then
l_return_string := l_return_string || to_char(j,'fm9');
l_last_digit := j;
end if;
end loop;
end loop;
return rpad( l_return_string, 6, '0' );
end;
/
函数已创建。
```

请注意此函数中我们使用了关键字 `DETERMINISTIC`。该声明表示，对于相同的输入，前面的函数将始终返回完全相同的输出。这对于在用户自编写的函数上创建基于函数的索引是必需的。我们必须告知 Oracle 该函数是 `DETERMINISTIC` 的，并且在给定相同输入时会返回一致的结果。我们是在告诉 Oracle，应信任此函数在调用间针对相同输入返回相同的值。若非如此，通过索引访问数据与全表扫描访问时，我们会得到不同的答案。这种确定性设置意味着，例如，我们无法在函数 `DBMS_RANDOM.RANDOM`（随机数生成器）上创建索引。它的结果不是确定性的；给定相同的输入，我们会得到随机的输出。另一方面，第一个示例中使用的内置 SQL 函数 `UPPER` 是确定性的，因此我们可以在列的 `UPPER` 值上创建索引。

现在我们有了函数 `MY_SOUNDEX`，让我们看看在没有索引的情况下它的性能如何。这里使用了我们之前创建的 `EMP` 表，该表大约有 10,000 行：

```
SQL> set autotrace on explain
SQL> variable cpu number
SQL> exec :cpu := dbms_utility.get_cpu_time
PL/SQL 过程已成功完成。
SQL> select ename, hiredate from emp where my_soundex(ename) = my_soundex('Kings');
ENAME      HIREDATE
---------- ---------
Ku$_Chunk_ 1979-12-17
Ku$_Chunk_ 1979-12-17
Ku$_Chunk_ 1979-12-17
Ku$_Chunk_ 1979-12-17
执行计划
----------------------------------------------------------
Plan hash value: 3956160932

| Id  | Operation         | Name | Rows  | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT  |      |  100  |  1900 |   24    (9)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| EMP  |  100  |  1900 |   24    (9)| 00:00:01 |

谓词信息 (由操作 ID 识别):
----------------------------------------------------------
   1 - filter("MY_SOUNDEX"("ENAME")="MY_SOUNDEX"('Kings'))
SQL> set autotrace off
SQL> set serverout on
SQL> begin
dbms_output.put_line
( 'cpu time = ' || round((dbms_utility.get_cpu_time-:cpu)/100,2) );
dbms_output.put_line( 'function was called: ' || stats.cnt );
end;
/
cpu time = .2
function was called: 9916
PL/SQL 过程已成功完成。
```

我们可以看到此查询执行花费了 0.2 个 CPU 秒，并且必须对表进行全表扫描。函数 `MY_SOUNDEX` 被调用了近 10,000 次（根据我们的计数器）。然而，我们希望它的调用频率能低得多。

让我们看看对函数创建索引如何能提升性能。我们要做的第一件事是如下创建索引：

```
SQL> create index emp_soundex_idx on emp( substr(my_soundex(ename),1,6) );
索引已创建。
```

在这个 `CREATE INDEX` 命令中值得注意的是 `SUBSTR` 函数的使用。这是因为我们索引的是一个返回字符串的函数。如果我们索引的是返回数字或日期的函数，这个 `SUBSTR` 就不是必需的。我们必须对返回字符串的用户自编函数使用 `SUBSTR` 的原因是，这类函数返回 `VARCHAR2(4000)` 类型。这很可能太大而无法索引——索引条目必须能放入约数据块大小四分之三的空间内。如果我们尝试直接索引，将会收到（在块大小为 4KB 的表空间中）如下错误：

```
SQL> create index emp_soundex_idx on
emp( my_soundex(ename) ) tablespace ts4k;
emp( my_soundex(ename) ) tablespace ts4k
*
第 2 行出现错误:
ORA-01450: 超出最大键长度 (3118)
```

这并非因为索引实际包含那么大的键，而是从数据库角度看，*它可能*包含那么大。但数据库理解 `SUBSTR`。它看到 `SUBSTR` 的输入是 1 和 6，知道其返回的最大值是六个字符；因此，它允许创建索引。这个尺寸问题可能困扰你，尤其是在使用组合索引时。以下是一个块大小为 8KB 的表空间示例：

```
SQL> create index emp_soundex_idx on
emp( my_soundex(ename), my_soundex(job) );
emp( my_soundex(ename), my_soundex(job) )
*
第 2 行出现错误:
ORA-01450: 超出最大键长度 (6398)
```

数据库认为最大键大小是 8000 字节，再次导致 `CREATE` 语句失败。因此，要在返回字符串的用户自编函数上创建索引，我们应在 `CREATE INDEX` 语句中约束返回类型。在此示例中，我们知道 `MY_SOUNDEX` 最多返回六个字符，因此我们对前六个字符取子串。

现在我们准备测试带有索引的表的性能。我们想监控索引对 `INSERT` 操作的影响以及对 `SELECT` 操作的加速效果，以观察各自的影响。在未加索引的测试用例中，我们的查询耗时超过一秒，如果我们对插入操作运行 `SQL_TRACE` 和 `TKPROF`，可以观察到，在没有索引的情况下，插入 9999 条记录大约耗时 0.30 秒：

```
insert into emp NO_INDEX
(empno,ename,job,mgr,hiredate,sal,comm,deptno)
select rownum empno,
initcap(substr(object_name,1,10)) ename,
substr(object_type,1,9) JOB,
rownum MGR,
created hiredate,
rownum SAL,
rownum COMM,
(mod(rownum,4)+1)*10 DEPTNO
from all_objects
where rownum < 10000
调用     计数      CPU    耗时      磁盘      查询    当前        行
------- ------  ------ --------- -------- ---------- ----------  ----------
解析        1    0.08      0.08        0          0          0           0
执行        1    0.15      0.22        0       2745      13763        9999
获取        0    0.00      0.00        0          0          0           0
------- ------  ------ --------- -------- ---------- ----------  ----------
总计        2    0.23      0.30          0       2745      13763        9999
```

但是有了索引后，它大约耗时 0.57 秒：

```
调用     计数      CPU    耗时      磁盘      查询    当前        行
------- ------  ------ --------- -------- ---------- ----------  ----------
解析        1    0.07      0.07        0          0          0           0
执行        1    0.39      0.49      131       2853      23313        9999
获取        0    0.00      0.00        0          0          0           0
------- ------  ------ --------- -------- ---------- ----------  ----------
总计        2    0.46      0.57      131       2853      23313        9999
```

这是管理新索引于 `MY_SOUNDEX` 函数上引入的开销——既有仅因存在索引而带来的性能开销（任何类型的索引都会影响插入性能），也有此索引必须调用存储过程 9999 次这一事实。

现在，为了测试查询性能，我们将重新运行查询：
```


