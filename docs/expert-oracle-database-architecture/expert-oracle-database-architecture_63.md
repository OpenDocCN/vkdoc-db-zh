# 数据库函数索引性能优化：使用 SUBSTR、视图与虚拟列

## SQL 执行示例

```sql
SQL> exec stats.cnt := 0
PL/SQL 过程已成功完成。
SQL> exec :cpu := dbms_utility.get_cpu_time
PL/SQL 过程已成功完成。
SQL> set autotrace on explain
SQL> select ename, hiredate from emp where substr(my_soundex(ename),1,6) = my_soundex('Kings');
ENAME      HIREDATE
---------- ---------
Ku$_Chunk_ 17-DEC-19
Ku$_Chunk_ 17-DEC-19
Ku$_Chunk_ 17-DEC-19
Ku$_Chunk_ 17-DEC-19
计划哈希值: 1897478402

| ID  | 操作                           | 名称            |  行数 | 字节数 |  代价 ...

|   0 | SELECT 语句                    |                 |   100 |  3300 |    12 ...
|   1 |  通过索引 ROWID 批量访问表     | EMP             |   100 |  3300 |    12 ...
|*  2 |   索引范围扫描                  | EMP_SOUNDEX_IDX |    40 |       |     1 ...

谓词信息 (由操作 ID 识别):

2 - 访问(SUBSTR("EODA"."MY_SOUNDEX"("ENAME"),1,6)="MY_SOUNDEX"('King s'))
SQL> set autotrace off
SQL> set serverout on
SQL> begin
dbms_output.put_line
( 'cpu time = ' || round((dbms_utility.get_cpu_time-:cpu)/100,2) );
dbms_output.put_line( 'function was called: ' || stats.cnt );
end;
/
CPU 时间 = .01
函数被调用次数: 1
PL/SQL 过程已成功完成。
```

## 性能对比分析

如果比较两个例子（未索引与已索引），我们发现插入到索引表的时间大约是原来的两倍多一点。然而，查询时间从十分之二秒变成了几乎瞬时完成。这里需要注意的要点如下：

*   插入 9999 条记录大约需要两倍的时间。为用户编写的函数创建索引必然会影响插入和某些更新的性能。当然，你应该意识到任何索引都会影响性能。例如，我做了一个简单的测试，不使用`MY_SOUNDEX`函数，只对`ENAME`列本身创建索引。这导致`INSERT`操作大约需要一秒钟才能完成——PL/SQL 函数并不是造成全部开销的原因。由于大多数应用程序只插入和更新单个条目，而每行插入时间少于万分之一秒，在典型的应用程序中你可能根本注意不到这一点。因为我们只插入一行数据，所以我们只为在列上执行函数支付一次代价，而不是查询数据的数千次。

*   虽然插入慢了两倍，但查询却快了很多倍。它只对`MY_SOUNDEX`函数进行了几次求值，而不是近一万次。我们的查询在性能上的差异是可测量的，并且相当大。此外，随着表的大小增长，全表扫描查询的执行时间会越来越长。而基于索引的查询在表变大时，将始终以几乎相同的性能特征执行。

*   我们必须在查询中使用`SUBSTR`。这不如直接编码`WHERE MY_SOUNDEX(ename)=MY_SOUNDEX( 'King' )`那么好，但我们可以轻松解决这个问题，稍后就会看到。

因此，插入操作受到了影响，但查询运行得非常快。以小幅降低插入/更新性能为代价，收益是巨大的。此外，如果你从不更新`MY_SOUNDEX`函数调用中涉及的列，则更新完全不会受到惩罚（`MY_SOUNDEX`仅在`ENAME`列被*修改*且其值发生变化时才会被调用）。

## 使用视图隐藏 SUBSTR 调用

让我们看看如何使查询不必使用`SUBSTR`函数调用。使用`SUBSTR`调用可能容易出错——我们的最终用户必须知道从第 1 个字符开始取 6 个字符。如果他们使用不同的大小，索引将不会被使用。而且，我们希望在服务器端控制要索引的字节数。这将允许我们以后用 7 个字节而不是 6 个字节重新实现`MY_SOUNDEX`函数。我们可以很容易地使用虚拟列来隐藏`SUBSTR`——或者在任何版本中使用视图，如下所示：

```sql
SQL> create or replace view emp_v as
select ename, substr(my_soundex(ename),1,6) ename_soundex, hiredate
from emp;
视图已创建。
SQL> exec stats.cnt := 0;
PL/SQL 过程已成功完成。
SQL> exec :cpu := dbms_utility.get_cpu_time
PL/SQL 过程已成功完成。
SQL> select ename, hiredate from emp_v where ename_soundex = my_soundex('Kings');
ENAME      HIREDATE
---------- ---------
Ku$_Chunk_ 17-DEC-19
Ku$_Chunk_ 17-DEC-19
Ku$_Chunk_ 17-DEC-19
Ku$_Chunk_ 17-DEC-19
SQL> set serverout on
SQL> begin
dbms_output.put_line
( 'cpu time = ' || round((dbms_utility.get_cpu_time-:cpu)/100,2) );
dbms_output.put_line( 'function was called: ' || stats.cnt );
end;
/
CPU 时间 = .01
函数被调用次数: 1
PL/SQL 过程已成功完成。
```

我们会看到与基础表相同的查询计划。我们在这里所做的只是在视图中隐藏了`SUBSTR( F(X), 1, 6 )`函数调用。优化器仍然能识别出这个虚拟列实际上就是索引列，因此操作正确。我们看到了相同的性能提升和相同的查询计划。使用这个视图与使用基础表一样好——甚至更好，因为它隐藏了复杂性，并允许我们以后更改`SUBSTR`的大小。

## 使用真实虚拟列实现

我应该指出，我们还有另一种实现选择。我们可以使用真实的虚拟列，而不是使用带有“虚拟列”的视图。使用此功能需要删除我们现有的基于函数的索引：

```sql
SQL> drop index emp_soundex_idx;
索引已删除。
```

然后向表添加虚拟列并为该列创建索引：

```sql
SQL> alter table emp add
ename_soundex as
(substr(my_soundex(ename),1,6));
表已更改。
SQL> create index emp_soundex_idx on emp(ename_soundex);
索引已创建。
```

现在我们可以直接查询基础表——完全不需要额外的视图层：

```sql
SQL> exec stats.cnt := 0;
PL/SQL 过程已成功完成。
SQL> exec :cpu := dbms_utility.get_cpu_time
PL/SQL 过程已成功完成。
SQL> select ename, hiredate from emp where ename_soundex = my_soundex('Kings');
ENAME      HIREDATE
---------- ---------
Ku$_Chunk_ 17-DEC-19
Ku$_Chunk_ 17-DEC-19
Ku$_Chunk_ 17-DEC-19
Ku$_Chunk_ 17-DEC-19
SQL> set serverout on
SQL> begin
dbms_output.put_line
( 'cpu time = ' || round((dbms_utility.get_cpu_time-:cpu)/100,2) );
dbms_output.put_line( 'function was called: ' || stats.cnt );
end;
/
CPU 时间 = 0
函数被调用次数: 1
PL/SQL 过程已成功完成。
```



### 仅索引部分行

除了透明地帮助使用内置函数（如 `UPPER`、`LOWER` 等）的查询之外，基于函数的索引还可用于有选择地仅索引表中的某些行。正如我们稍后将讨论的，`B*Tree` 索引不包含完全为 `NULL` 键的条目。也就是说，如果你在表 `T` 上有一个索引 `I`（如下所示），并且有一行中 `A` 和 `B` 都为 `NULL`，则索引结构中将不会有该条目：

```
Create index I on t(a,b);
```

当你只想索引表中的某些行时，这一点就派上用场了。

考虑一个大型表，其中有一个名为 `PROCESSED_FLAG` 的 `NOT NULL` 列，该列可能取两个值之一，`Y` 或 `N`，默认值为 `N`。新添加的行值为 `N`，表示尚未处理，处理完成后更新为 `Y`，表示已处理。我们希望索引此列以便快速检索值为 `N` 的记录，但表中有数百万行，且几乎所有行的值都是 `Y`。由此产生的 `B*Tree` 索引将会很大，并且在我们从 `N` 更新到 `Y` 时维护它的成本会很高。这个表听起来像是位图索引的候选（毕竟这是低基数列），但这是一个事务型系统，许多人会同时插入已处理列设置为 `N` 的记录，并且正如我们之前讨论的，位图索引不利于并发修改。如果我们再考虑到在这个表中不断地将 `N` 更新为 `Y`，那么位图索引就完全不可行了，因为这个过程会完全串行化。

所以，我们真正想要的是只索引感兴趣的记录（即 `N` 记录）。我们将看到如何使用基于函数的索引来实现这一点，但在那之前，让我们先看看如果只使用常规的 `B*Tree` 索引会发生什么。使用本书开头设置部分中描述的 `BIG_TABLE` 标准脚本，我们将更新 `TEMPORARY` 列，将 `Y` 翻转为 `N`，将 `N` 翻转为 `Y`：

```
$ sqlplus eoda/foo@PDB1
SQL> update big_table set temporary = decode(temporary,'N','Y','N');
1000000 rows updated.
```

然后我们检查 `Y` 与 `N` 的比例：

```
SQL> select temporary, cnt, round( (ratio_to_report(cnt) over ()) * 100, 2 ) rtr
from (select temporary, count(*) cnt
from big_table
group by temporary);
T        CNT        RTR
- ---------- ----------
Y     998728      99.87
N       1272        .13
```

如我们所见，在表中的 1,000,000 条记录中，只有大约千分之一点五的数据应该被索引。如果我们在 `TEMPORARY` 列（在此示例中扮演 `PROCESSED_FLAG` 列的角色）上使用常规索引，我们会发现该索引有 1,000,000 个条目，占用近 14MB 空间，高度为 3：

```
SQL> create index processed_flag_idx on big_table(temporary);
Index created.
SQL> analyze index processed_flag_idx validate structure;
Index analyzed.
SQL> select name, btree_space, lf_rows, height from index_stats;
NAME                 BTREE_SPACE    LF_ROWS     HEIGHT
-------------------- ----------- ---------- ----------
PROCESSED_FLAG_IDX      14528892    1000000          3
```

任何通过此索引的检索都将导致三次 I/O 操作才能到达叶块。这个索引不仅宽，而且高。为了获取第一个未处理的记录，我们将不得不执行至少四次 I/O（三次针对索引，一次针对表）。

我们如何改变这一切？我们需要让索引变得更小、更易于维护（在更新时运行时开销更小）。这时基于函数的索引就派上用场了，它允许我们简单地编写一个函数，当不想索引某行时返回 `NULL`，当想要索引时则返回非 `NULL` 值。例如，既然我们只对 `N` 记录感兴趣，那么我们就只索引这些：

```
SQL> drop index processed_flag_idx;
Index dropped.
SQL> create index processed_flag_idx on big_table( case temporary when 'N' then 'N' end );
Index created.
SQL> analyze index processed_flag_idx validate structure;
Index analyzed.
SQL> select name, btree_space, lf_rows, height from index_stats;
NAME                 BTREE_SPACE    LF_ROWS     HEIGHT
-------------------- ----------- ---------- ----------
PROCESSED_FLAG_IDX         32016       1272          2
```

差异相当大——索引大约是 32KB，而不是 14MB。高度也降低了。如果我们使用这个索引，将比使用之前那个更高的索引少执行一次 I/O。

### 实现选择性唯一性

基于函数的索引的另一个有用技术是将其用于强制执行某些类型的复杂约束。例如，假设你有一个包含版本化信息的表，比如一个 `projects` 表。项目有两种状态：`ACTIVE` 或 `INACTIVE`。你需要强制执行一条规则，例如“活动项目必须具有唯一的名称；非活动项目则不需要”。也就是说，只能有一个活动的“project X”，但你可以拥有任意多个非活动的 project X。

开发人员听到这个需求时的第一反应通常是，“我们就运行一个查询看看是否有活动的 project X，如果没有，我们就创建我们的。”如果你阅读了第 7 章，你就会明白这种简单的实现在多用户环境中是行不通的。如果两个人试图同时创建一个新的活动 project X，他们都会成功。我们需要将 project X 的创建过程串行化，但唯一的方法是锁定整个 `projects` 表（并发性不高），或者使用基于函数的索引，让数据库为我们完成。

基于我们可以在函数上创建索引、`B*Tree` 索引中不为完全 `NULL` 的条目创建索引、以及我们可以创建 `UNIQUE` 索引这几个事实，我们可以轻松地执行以下操作：

```
Create unique index active_projects_must_be_unique
On projects ( case when status = 'ACTIVE' then name end );
```

这样就能实现。当 `status` 列为 `ACTIVE` 时，`NAME` 列将被唯一索引。任何试图创建同名活动项目的操作都将被检测到，并且对此表的并发访问完全不受影响。



### 关于 ORA-01743 的注意事项

我注意到基于函数的索引存在一个怪癖：如果对内置函数 `TO_DATE` 创建索引，在某些情况下会失败：

```sql
$ sqlplus eoda/foo@PDB1
SQL> create table t ( year varchar2(4) );
表已创建。
SQL> create index t_idx on t( to_date(year,'YYYY') );
create index t_idx on t( to_date(year,'YYYY') )
*
第 1 行出现错误:
ORA-01743: 只能对纯函数建立索引
```

这看起来很奇怪，因为我们`有时`可以使用`TO_DATE`创建函数，像这样：

```sql
SQL> create index t_idx on t( to_date('01'||year,'MMYYYY') );
索引已创建。
```

随附的错误信息也不太具有启发性：

```sql
SQL> !oerr ora 1743
01743, 00000, "only pure functions can be indexed"
// *原因: 被索引的函数使用了 SYSDATE 或用户环境。
// *操作: PL/SQL 函数必须是纯的 (RNDS, RNPS, WNDS, WNPS)。SQL
//          表达式不得使用 SYSDATE、USER、USERENV() 或任何
//          其他依赖于会话状态的内容。依赖于 NLS 的函数
//          是可以的。
```

我们没有使用 `SYSDATE`。我们没有使用用户环境（真的吗？）。没有使用任何 `PL/SQL` 函数，也没有涉及任何会话状态相关的操作。诀窍在于我们使用的格式：`YYYY`。这种格式，在输入完全相同的情况下，会根据你调用它的月份返回不同的答案。例如，在五月的任何时间，`YYYY` 格式都会返回 5 月 1 日，在六月则返回 6 月 1 日，依此类推：

```sql
SQL> select to_char( to_date('2015','YYYY')， 'DD-Mon-YYYY HH24:MI:SS' ) from dual;
TO_CHAR(TO_DATE('200

01-May-2015 00:00:00
```

事实证明，当与 `YYYY` 一起使用时，`TO_DATE` 不是确定性的！这就是为什么无法创建索引：它只能在创建索引的那个月（或插入/更新某行的那个月）正常工作。所以，这是由于用户环境造成的，而当前日期本身也属于用户环境。

要在基于函数的索引中使用 `TO_DATE`，`必须`使用明确且确定性的日期格式——无论当前是哪一天。

### 基于函数的索引总结

基于函数的索引易于使用和实现，并能提供即时价值。它们可用于加速现有应用程序，而无需更改其任何逻辑或查询。可能会观察到数量级的性能提升。你可以使用它们来预计算复杂值，而无需使用触发器。此外，如果表达式在基于函数的索引中被物化，优化器可以更准确地估计选择性。如之前的 `PROCESSED_FLAG` 示例所示，你可以使用基于函数的索引来有选择地仅索引感兴趣的行。实际上，你可以使用该技术来索引 `WHERE` 子句。最后，你可以使用基于函数的索引来实现某种完整性约束：选择性唯一性（例如，“当某些条件为真时，字段 X、Y 和 Z 必须唯一”）。

基于函数的索引会影响插入和更新的性能。这个警告是否与你相关，必须由你自己判断。如果你插入数据非常频繁而查询很少，那么这可能不是一个适合你的功能。另一方面，请记住，你通常插入一行一次，而查询它数千次。插入时的性能影响（你的最终用户可能永远不会注意到）可能会被加速查询数千倍所抵消。总的来说，在这种情况下，利远大于弊。

### 应用域索引

应用域索引是 Oracle 所说的`可扩展索引`。它们允许你创建自己的索引结构，这些结构的工作方式与 Oracle 提供的索引类似。当有人使用你的索引类型发出 `CREATE INDEX` 语句时，Oracle 将运行你的代码来生成索引。如果有人分析索引以计算其统计信息，Oracle 将执行你的代码以你关心的存储格式生成统计信息。当 Oracle 解析查询并制定可能使用你的索引的查询计划时，Oracle 会在评估不同计划时询问你该函数的执行成本如何。简而言之，应用域索引使你能够实现数据库中尚不存在的新的索引类型。例如，如果你开发用于分析数据库中存储图像的软件，并生成有关图像的信息（例如其中发现的颜色），你可以创建自己的 `image` 索引。当图像被添加到数据库时，你的代码会被调用以从图像中提取颜色并将其存储在某个地方（你想要存储的任何地方）。在查询时，当用户要求所有蓝色图像时，Oracle 会在适当的时候要求你从索引中提供答案。

应用域索引的最佳示例是 Oracle 自己的`文本索引`。该索引用于对大型文本项提供关键字搜索。你可以像这样创建一个简单的文本索引：

```sql
$ sqlplus eoda/foo@PDB1
SQL> create index myindex on mytable(docs)
indextype is ctxsys.context;
索引已创建。
```

然后使用该索引类型引入到 `SQL` 语言中的文本运算符：

```sql
SQL> select * from mytable where contains( docs， 'some words' ) > 0;
```

它甚至会响应如下命令：

```sql
SQL> begin
dbms_stats.gather_index_stats( user， 'MYINDEX' );
end;
/
PL/SQL 过程已成功完成。
```

它将在运行时与优化器协作，以确定使用文本索引相对于其他索引或全表扫描的相对成本。所有这些有趣之处在于，你或我本可以开发这个索引。文本索引的实现没有使用`内核内部知识`。它是使用专用的、有文档记录的、公开的 API 完成的。`Oracle` 数据库内核并不知道文本索引是如何存储的（API 为每个创建的索引将其存储在许多物理数据库表中）。Oracle 也不知道插入新行时发生的处理过程。Oracle 文本实际上是一个构建在数据库之上的应用程序，但以完全集成的方式实现。对你我而言，它看起来就像任何其他 Oracle 数据库内核功能，但实际上并非如此。

我个人尚未发现有必要去构建一种新的奇特索引结构。我认为这个特性主要对拥有创新索引技术的第三方解决方案提供商有用。

我认为应用域索引最有趣的地方在于，它们允许他人提供我可以用于应用程序的新索引技术。大多数人永远不会使用这个特定的 API 来构建新的索引类型，但我们大多数人会使用最终成果。我参与的几乎每个应用程序似乎都有一些相关的`文本`、需要处理的`XML`，或需要存储和分类的`图像`。使用应用域索引功能实现的 Oracle Multimedia 功能集提供了这些能力。随着时间的推移，可用的索引类型集合会不断增长。我们将在后续章节中更深入地探讨文本索引。


## 不可见索引

你可以选择让索引对优化器不可见。索引的“不可见”仅指优化器在生成执行计划时不会使用该索引。你可以创建不可见索引，也可以将现有索引更改为不可见。下面我们将创建一个表，加载测试数据，生成统计信息，然后创建一个不可见索引：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t(x int);
Table created.
SQL>  insert into t select round(dbms_random.value(1,10000)) from dual
connect by level   exec dbms_stats.gather_table_stats(user,'T');
PL/SQL procedure successfully completed.
SQL> create index ti on t(x) invisible;
Index created.
```

现在我们开启自动跟踪，并运行一个查询。通常期望优化器在生成执行计划时会使用索引：

```
SQL> set autotrace traceonly explain
SQL> select * from t where x=5;

| Id  | Operation         | Name | Rows  | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT  |      |    2  |     8 |    7    (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| T    |    2  |     8 |    7    (0)| 00:00:01 |

```

上面的输出显示优化器没有使用该索引。你可以通过将会话参数 `OPTIMIZER_USE_INVISIBLE_INDEXES` 设置为 `TRUE`（默认值为 `FALSE`），在会话期间切换索引对优化器的可见性。例如，对于当前连接的会话，以下命令指示优化器在生成执行计划时考虑不可见索引：

```
SQL> alter session set optimizer_use_invisible_indexes=true;
```

重新运行之前的查询，显示优化器现在开始利用该索引：

```
SQL> select * from t where x=5;

| Id  | Operation        | Name |  Rows | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT |      |     2 |     8 |    1    (0)| 00:00:01 |
|*  1 |  INDEX RANGE SCAN| TI   |     2 |     8 |    1    (0)| 00:00:01 |

```

如果你希望所有会话都考虑使用不可见索引，则需要通过 `ALTER SYSTEM` 语句来修改 `OPTIMIZER_USE_INVISIBLE_INDEXES` 参数。这会使所有不可见索引在生成执行计划时对优化器可见。

你可以通过将索引更改为可见，使其对优化器永久可见：

```
SQL> alter index ti visible;
Index altered.
```

请记住，尽管不可见索引对优化器不可见，但它仍可能在以下方面影响性能：

*   当底层表有记录插入、更新或删除时，不可见索引仍会消耗空间和资源。这可能会影响性能（减慢 DML 语句的速度）。

*   当 B*Tree 索引被放置在外键列上时，Oracle 仍可使用不可见索引来防止某些锁定情况。

*   如果你创建了一个唯一的不可见索引，无论可见性设置如何，列的唯一性都将被强制执行。

因此，即使你创建了一个不可见索引，它仍然可能影响 SQL 语句的行为。错误地认为不可见索引对使用包含这些索引的表的应用程序没有任何影响。不可见索引的“不可见”仅指优化器在生成执行计划时不会考虑使用它们，除非被明确指示。

那么，不可见索引有什么用呢？这些索引仍然需要维护（因此会降低性能），但查询无法“看到”它们，因此永远不会提升性能。一个例子是当你想要从生产系统中删除一个索引时。思路是你可以将索引设为不可见，并观察性能是否下降。在这种情况下，在删除索引之前，你还必须注意该索引是否放置在外键列上，或者是否被用于强制唯一性。另一个例子是你想向生产系统添加一个索引并进行测试以确定是否提高了性能。你可以将索引添加为不可见索引，并在会话中选择性地使其可见以确定其有用性。这里同样需要注意，尽管索引是不可见的，它仍会消耗空间并需要资源来维护。

## 同一列组合上的多个索引

在 Oracle Database 12c 之前，你不能在同一个表上定义具有完全相同列组合的多个索引。例如：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t(x int);
Table created.
SQL> create index ti on t(x);
Index created.
SQL> create bitmap index tb on t(x) invisible;
ERROR at line 1:
ORA-01408: such column list already indexed
```

从 12c 开始，你可以在同一组列上定义多个索引。但是，仅当索引在物理上不同时才可以这样做，例如，一个索引创建为 B*Tree 索引，第二个索引创建为位图索引。此外，对于表上相同的列组合，只能有一个可见索引。因此，在 Oracle 12c 数据库中运行之前的 `CREATE INDEX` 语句是可行的：

```
SQL> create table t(x int);
Table created.
SQL> create index ti on t(x);
Index created
SQL> create bitmap index tb on t(x) invisible;
Index created
```

为什么你希望在相同的列组合上定义两个索引？假设你最初为一个数据仓库星型模式构建了所有 B*Tree 索引在事实表的外键列上，后来通过测试发现位图索引对于应用于该星型模式的查询类型性能更好。因此，你希望尽可能无缝地转换到位图索引。所以你首先将位图索引创建为不可见。然后当你准备好时，你可以删除 B*Tree 索引，再将位图索引更改为可见。


## 索引扩展列

随着 Oracle 12c 的推出，`VARCHAR2`、`NVARCHAR2` 和 `RAW` 数据类型现在可配置为存储最多 32,767 字节的信息（此前，`VARCHAR2` 和 `NVARCHAR2` 的限制是 4000 字节，`RAW` 是 2000 字节）。由于第 12 章已详述如何为数据库启用扩展数据类型，此处不再重复。本节重点探讨如何索引扩展列。

让我们从创建一个带扩展列的表开始，然后尝试在该列上创建常规的 B*树索引：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t(x varchar2(32767));
Table created.
```

注意：如果您尝试在尚未配置扩展数据类型的数据库中创建 `VARCHAR2` 列长度超过 4000 字节的表，Oracle 将抛出 `ORA-00910: specified length too long for its datatype` 错误信息。

接下来，我们尝试在扩展列上创建索引：

```
SQL> create index ti on t(x);
create index ti on t(x)
*
ERROR at line 1:
ORA-01450: maximum key length (6398) exceeded
```

我们之前在本章中见过此错误。之所以报错，是因为 Oracle 对索引键长度施加了最大限制，该限制大约是块大小的四分之三（本例中数据库的块大小为 8K）。即使该索引中目前没有任何条目，Oracle 也知道对于一个最多可容纳 32,767 字节的列，其索引键大小有可能超过 6398 字节，因此不允许您在此场景下创建索引。

这并不意味着您无法索引扩展列；而是必须使用技术将索引键长度限制在 6398 字节以下。考虑到这一点，有几个选项变得显而易见：

*   基于 `SUBSTR` 或 `STANDARD_HASH` 函数创建虚拟列，然后在该虚拟列上创建索引。
*   使用 `SUBSTR` 或 `STANDARD_HASH` 函数创建基于函数的索引。
*   创建基于更大块大小的表空间；例如，16K 的块大小将允许索引键大小约为 12,000 字节。话虽如此，如果您需要一个 12,000 字节的索引键，那您可能做错了什么，需要重新思考您的操作。此方法将不予探讨。

让我们从查看虚拟列解决方案开始。

### 虚拟列解决方案

这里的想法是，首先在扩展列上应用一个 SQL 函数来创建一个虚拟列，该函数返回的值小于 6398 字节。然后，可以对该虚拟列创建索引，这为针对扩展列的查询提供了提升性能的机制。一个示例将演示这一点。首先，创建一个带扩展列的表：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t(x varchar2(32767));
Table created.
```

现在向表中插入一些测试数据：

```
SQL> insert into t select to_char(level)|| rpad('abc',10000,'xyz')
from dual connect by level < 1001
union
select to_char(level)
from dual connect by level < 1001;
2000 rows created.
```

现在假设您知道扩展列的前十個字符具有足够的选择性，足以返回表中的一小部分行。因此，您基于扩展列的子字符串创建一个虚拟列：

```
SQL> alter table t add (xv as (substr(x,1,10)));
Table altered.
```

现在在虚拟列上创建索引并收集统计信息：

```
SQL> create index te on t(xv);
Index created.
SQL> exec dbms_stats.gather_table_stats(user,'T');
PL/SQL procedure successfully completed.
```

现在，当查询虚拟列时，优化器可以在 `WHERE` 子句中的等值和范围谓词中利用该索引，例如：

```
SQL> set autotrace traceonly explain
SQL> select count(*) from t where x = '800';

| Id  | Operation                            | Name |  Rows | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |      |     1 |  5011 |    2    (0)| 00:00:01 |
|   1 |  SORT AGGREGATE                      |      |     1 |  5011 |            |          |
|*  2 |   TABLE ACCESS BY INDEX ROWID BATCHED| T    |     1 |  5011 |    2    (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | TE   |     1 |       |    1    (0)| 00:00:01 |
```

请注意，即使索引建立在虚拟列上，当直接针对扩展列 `X`（而非虚拟列 `XV`）进行查询时，优化器仍可以使用它。优化器也可以在范围类搜索中使用这种类型的索引：

```
SQL> select count(*) from t where x >'800' and x<'900';

| Id  | Operation                            | Name |  Rows | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |      |     1 |  5011 |    4    (0)| 00:00:01 |
|   1 |  SORT AGGREGATE                      |      |     1 |  5011 |            |          |
|*  2 |   TABLE ACCESS BY INDEX ROWID BATCHED| T    |   239 |  1169K|    4    (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | TE   |   241 |       |    2    (0)| 00:00:01 |
```

与 `SUBSTR` 函数类似，您也可以基于 `STANDARD_HASH` 函数创建虚拟列。`STANDARD_HASH` 函数可应用于长字符串，并返回一个相当唯一且远小于 6398 字节的 `RAW` 值。让我们看几个基于 `STANDARD_HASH` 的虚拟列示例。

假设使用的表和种子数据与之前的 `SUBSTR` 示例相同，这里我们向表中添加一个使用 `STANDARD_HASH` 的虚拟列，创建索引并生成统计信息：

```
SQL> alter table t add (xv as (standard_hash(x)));
Table altered.
SQL> create index te on t(xv);
Index created.
SQL> exec dbms_stats.gather_table_stats(user,'T');
PL/SQL procedure successfully completed.
```

`STANDARD_HASH` 在 `WHERE` 子句中使用等值谓词时效果很好。例如：

```
SQL> set autotrace traceonly explain
SQL> select count(*) from t where x='300';

| Id  | Operation                            | Name |  Rows | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |      |     1 |  5025 |    2    (0)| 00:00:01 |
|   1 |  SORT AGGREGATE                      |      |     1 |  5025 |            |          |
|*  2 |   TABLE ACCESS BY INDEX ROWID BATCHED| T    |     1 |  5025 |    2    (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | TE   |     1 |       |    1    (0)| 00:00:01 |
```

基于 `STANDARD_HASH` 的虚拟列上的索引允许进行高效的等值搜索，但对于范围搜索无效，因为数据在索引中是基于随机化的哈希值存储的，例如：

```
SQL> select count(*) from t where x >'800' and x<'900';

| Id  | Operation          | Name |  Rows | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |      |     1 |  5004 |    6    (0)| 00:00:01 |
|   1 |  SORT AGGREGATE    |      |     1 |  5004 |            |          |
|*  2 |   TABLE ACCESS FULL| T    |   239 |  1167K|    6    (0)| 00:00:01 |
```



### 基于函数的索引解决方案

此处的概念是，你正在构建一个索引，并对其应用一个函数，这种方式既能限制索引键的长度，又能产生可用的索引。在这里，我使用与上一节相同的代码来创建一个带有扩展列的表，并用测试数据填充它：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t(x varchar2(32767));
Table created.
SQL> insert into t
select to_char(level)|| rpad('abc',10000,'xyz')
from dual connect by level < 1001
union
select to_char(level)
from dual connect by level < 1001;
2000 rows created.
```

现在假设你熟悉这些数据，并且知道扩展列的前十个字符通常足以标识一行；因此，你在前十个字符的子字符串上创建了一个索引，并为表生成了统计信息：

```
SQL> create index te on t(substr(x,1,10));
Index created.
SQL> exec dbms_stats.gather_table_stats(user,'T');
PL/SQL procedure successfully completed.
```

当 `WHERE` 子句中存在相等谓词和范围谓词时，优化器可以使用这样的索引。以下是一些说明此情况的示例：

```
SQL> set autotrace traceonly explain
SQL> select count(*) from t where x = '800';

| Id  | Operation                            | Name |  Rows | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT                     |      |     1 | 16407 |    2    (0)| 00:00:01 |
|   1 |  SORT AGGREGATE                      |      |     1 | 16407 |            |          |
|*  2 |   TABLE ACCESS BY INDEX ROWID BATCHED| T    |     1 | 16407 |    2    (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | TE   |     8 |       |    1    (0)| 00:00:01 |

```

此示例使用了一个范围谓词：

```
SQL> select count(*) from t where x>'200' and x<'400';

| Id  | Operation                            | Name |  Rows | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT                     |      |     1 |  5011 |    6    (0)| 00:00:01 |
|   1 |  SORT AGGREGATE                      |      |     1 |  5011 |            |          |
|*  2 |   TABLE ACCESS BY INDEX ROWID BATCHED| T    |   477 |  2334K|    6    (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | TE   |   479 |       |    3    (0)| 00:00:01 |

```

假设表与之前 `SUBSTR` 示例中使用的相同且种子数据一致，这里我们添加一个使用 `STANDARD_HASH` 的基于函数的索引：

```
SQL> create index te on t(standard_hash(x));
```

现在验证基于相等的搜索是否使用了索引：

```
SQL> set autotrace traceonly explain
SQL> select count(*) from t where x = '800';

| Id  | Operation                            | Name |  Rows | Bytes | Cost (%CPU)| Time    |

|   0 | SELECT STATEMENT                     |      |     1 |  5004 |    4    (0)| 00:00:01|
|   1 |  SORT AGGREGATE                      |      |     1 |  5004 |            |         |
|*  2 |   TABLE ACCESS BY INDEX ROWID BATCHED| T    |     1 |  5004 |    4    (0)| 00:00:01|
|*  3 |    INDEX RANGE SCAN                  | TE   |     8 |       |    1    (0)| 00:00:01|

```

这允许进行高效的基于相等的搜索，但不适用于基于范围的搜索，因为数据存储在基于随机化哈希值的索引中。

## 关于索引的常见问题解答与误解

正如我在本书引言中所说，我收到了许多关于 Oracle 的问题。我是《Oracle Magazine》中"Ask Tom"专栏以及网址为 `http://asktom.oracle.com` 网站的 Tom，我在那里回答人们关于 Oracle 数据库和工具的问题。根据我的经验，索引这个话题吸引了最多的问题。在本节中，我将回答一些最常被问到的问题。有些答案可能看起来是常识，而另一些答案可能会让你感到惊讶。可以说，围绕着索引存在许多误解和误解。

### 索引在视图上有效吗？

一个相关的问题是："我如何为视图创建索引？"嗯，事实是视图只不过是一个存储的查询。Oracle 会将访问视图的查询文本替换为视图定义本身。视图是为了最终用户或程序员的方便——优化器处理的是针对基表的查询。当你使用视图时，任何以及所有如果查询是针对基表编写本可以使用的索引都将被考虑。要为视图创建索引，你只需为基表创建索引。


### 空值与索引是否协同工作？

B*树索引（集群 B*树索引这一特殊情况除外）不存储完全为 NULL 的条目，但位图索引和集群索引会存储。这种副作用可能令人困惑，但当你理解“不存储完全为 NULL 的键”意味着什么后，它实际上可以为你所用。

要观察“空值*不*被存储”这一事实的影响，请考虑以下示例：

```sql
$ sqlplus eoda/foo@PDB1
SQL> create table t ( x int, y int );
表已创建。
SQL> create unique index t_idx on t(x,y);
索引已创建。
SQL> insert into t values ( 1, 1 );
已创建 1 行。
SQL> insert into t values ( 1, NULL );
已创建 1 行。
SQL> insert into t values ( NULL, 1 );
已创建 1 行。
SQL> insert into t values ( NULL, NULL );
已创建 1 行。
SQL> analyze index t_idx validate structure;
索引已分析。
SQL> select name, lf_rows from index_stats;
NAME                    LF_ROWS
-------------------- ----------
T_IDX                         3
```

表中有四行数据，但索引中只有三行。前四行中，至少有*一个*索引键元素*不*为 NULL 的行存在于索引中。最后一行`(NULL, NULL)`不在索引中。其中一个令人困惑的点在于，当索引是唯一索引时（如此处所示）。请考虑以下三个`INSERT`语句的影响：

```sql
SQL> insert into t values ( NULL, NULL );
已创建 1 行。
SQL> insert into t values ( NULL, 1 );
insert into t values ( NULL, 1 )
*
第 1 行出现错误:
ORA-00001: 违反唯一约束条件 (EODA.T_IDX)
SQL> insert into t values ( 1, NULL );
insert into t values ( 1, NULL )
*
第 1 行出现错误:
ORA-00001: 违反唯一约束条件 (EODA.T_IDX)
```

新的`(NULL, NULL)`行并不被认为与旧的`(NULL, NULL)`行相同：

```sql
SQL> select x, y, count(*)
from t
group by x,y
having count(*) > 1;
X          Y   COUNT(*)
---------- ---------- ----------

```

这看起来不可能；如果我们考虑所有 NULL 条目，我们的唯一键就不唯一了。事实是，在 Oracle 中，当考虑唯一性时，`(NULL, NULL)` 不等于 `(NULL, NULL)`——这是 SQL 标准所规定的。然而，就聚合而言，`(NULL,NULL)` 和 `(NULL,NULL)` 被视为相同。两者在比较时是唯一的，但在`GROUP BY`子句中则被视为相同。这是需要考虑的一点：每个*唯一*约束应该至少包含一个`NOT NULL`列，才能真正实现唯一性。

关于索引和空值，常见的问题是：“为什么我的查询没有使用索引？”该查询类似于以下形式：

```sql
SQL> select * from T where x is null;
```

这个查询无法使用我们刚创建的索引——`(NULL, NULL)`这一行根本不在索引中；因此，使用索引会返回错误的结果。只有至少*一个*列被定义为`NOT NULL`时，查询才能使用索引。例如，以下示例展示了，如果存在一个以`X`为前导列的索引，且索引中至少另一列在基础表中被定义为`NOT NULL`，那么 Oracle 将对`X IS NULL`谓词使用索引：

```sql
SQL> create table t ( x int, y int NOT NULL );
表已创建。
SQL> create unique index t_idx on t(x,y);
索引已创建。
SQL> insert into t values ( 1, 1 );
已创建 1 行。
SQL> insert into t values ( NULL, 1 );
已创建 1 行。
SQL> begin
  2    dbms_stats.gather_table_stats(user,'T');
  3  end;
  4  /
PL/SQL 过程已成功完成。
```

当我们这次查询该表时，会发现如下情况：

```sql
SQL> set autotrace on
SQL> select * from t where x is null;
X          Y
---------- ----------

执行计划
...
| Id  | Operation        | Name  | Rows  | Bytes | Cost (%CPU)| Time     |
|---  |---               |---    |---    |---    |---         |---       |
|   0 | SELECT STATEMENT |       |    1  |     5 |    1    (0)| 00:00:01 |
|*  1 |  INDEX RANGE SCAN| T_IDX |    1  |     5 |    1    (0)| 00:00:01 |
```

之前我说过，你可以利用“B*树索引不存储完全为 NULL 的条目”这一事实为你所用——方法如下。假设你有一个表，其中一列正好取两个值。这些值分布非常不均衡；比如，90%或更多的行取一个值，10%或更少的行取另一个值。你可以高效地索引这一列，以便快速访问占少数的行。当你希望使用索引来获取占少数的行，但又希望使用全表扫描来获取占多数的行，并且希望节省空间时，这种方法就非常方便。解决方案是对占多数的行使用 NULL，对占少数的行使用你想要的任何值；或者如前面所演示的，使用基于函数的索引来仅索引函数返回的非 NULL 值。

现在你已经了解了 B*树如何处理空值，你可以利用这一点，并对那些所有列都允许为空的列集上的唯一约束采取预防措施（在这种情况下，要做好存在多行全为 NULL 的可能性的准备）。

