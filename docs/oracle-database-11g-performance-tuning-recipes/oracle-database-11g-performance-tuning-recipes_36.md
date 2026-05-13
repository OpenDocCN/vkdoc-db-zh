# 将 CURSOR_SHARING 设置为非默认值的影响

`FORCE` 和 `SIMILAR` 这两种设置都可以帮助你规避应用程序中未使用绑定变量的问题，方法是让数据库生成绑定值（即系统生成的绑定值，与用户指定的绑定值相对）。然而，你需要意识到，当将 `cursor_sharing` 参数设置为 `FORCE` 与设置为 `SIMILAR` 时，优化器的行为存在差异。这里关键要理解的是，查询性能与共享池中多次执行查询所占用的空间之间存在权衡。

以下是 `cursor_sharing` 参数设置为 `EXACT`、`FORCE` 和 `SIMILAR` 时的性能影响总结。我们假设存在以下包含字面量的查询：

`select * from employees where job = 'Clerk'`

请注意，如果查询使用的是绑定变量而非字面量，其形式如下：

`select * from employees where job=:b`

> `EXACT`：数据库不会替换任何字面量，优化器看到的查询就是其呈现给优化器的样子。优化器会根据语句中的字面量值，为每次语句执行生成不同的执行计划。因此，该计划将是最优的，但每个语句都有自己的父游标，因此执行次数多的语句可能会在共享池中占用大量空间。这可能导致闩锁争用和性能下降。
>
> `FORCE`：无论是否存在直方图，优化器都会将字面量值替换为绑定值，并按以下形式优化此查询：
>
> `select * from employees where job=:b`
>
> 无论字面量值如何，优化器对每个 SQL 语句都使用单一执行计划。因此，执行计划可能不是最优的，因为该计划是通用的，而非基于字面量值。如果查询使用了字面量，优化器会利用这些值来寻找最有效的执行计划。如果 SQL 语句中没有字面量，优化器很难确定最佳执行计划。通过“窥视”绑定变量的值，优化器可以更好地了解 `where` 子句条件的**选择性**——这几乎就像在 SQL 语句中使用了字面量一样。优化器在硬解析阶段窥视绑定值。由于执行计划是基于优化器碰巧窥视到的绑定变量特定值生成的，因此该计划可能对绑定变量的所有可能值并非都是最优的。
>
> 在此示例中，优化器基于它看到的 `JOB` 列的特定值使用绑定窥视。这种情况下，优化器使用值 `Clerk` 来估算查询的基数。当执行相同语句时（`JOB` 列使用不同值，比如 `Manager`），优化器将使用第一次生成（`JOB`=`Clerk` 时）的相同计划。由于只有一个父游标和仅为不同语句生成的子游标，对共享池的压力较小。请注意，子游标在共享池中占用的空间远少于父游标。通常，将 `cursor_sharing` 参数设置为 `FORCE` 能立即解决数据库中严重的闩锁争用，这是少数几个能帮助你快速减少闩锁争用的“灵丹妙药”之一。
>
> `SIMILAR`（`JOB` 列上没有直方图）：数据库将使用字面量替换——它为 `JOB` 列的字面量值（`Clerk`）使用系统生成的绑定值。这是因为 `JOB` 列上缺少直方图告诉优化器该列的数据不是偏斜的，因此即使字面量值不同，优化器也会为语句的每次执行选择相同的计划。优化器认为不应为仅字面量值不同的语句更改执行计划，因为数据是均匀分布的。在查询列上没有直方图的情况下，`SIMILAR` 设置提供的查询性能以及对共享池的影响与指定 `FORCE` 设置时相同。
>
> `SIMILAR`（`JOB` 列上有直方图）：当优化器看到 `JOB` 列上的直方图时，它会意识到该列数据是偏斜的——这告诉优化器，查询结果可能因 `JOB` 列的字面量值不同而有很大差异。因此，优化器根据字面量值为每个语句生成不同的计划——因此计划非常高效，就像指定 `EXACT` 设置时一样。在有直方图的情况下使用 `SIMILAR` 选项确实会在共享池中使用更多空间，但不如使用 `EXACT` 设置时那么多。原因在于每个语句拥有自己的子游标，而不是父游标。

在 `cursor_sharing` 参数的各种设置之间做选择，实质上是评估什么对数据库性能更为关键：使用默认的 `EXACT` 设置或 `SIMILAR`（相关列上有直方图）确实能提供更好的查询性能，但会导致生成大量父游标（`EXACT` 设置）或子游标（`SIMILAR` 设置）。如果共享池压力巨大，并伴随闩锁争用，整个数据库性能将变得很差。在这种情况下，你最好实施全系统解决方案，将 `cursor_sharing` 参数设置为 `FORCE`，因为这能保证每个 SQL 语句只有一个子游标。如果你担心单个 SQL 语句的影响，只需删除该 SQL 语句所用相关列上的直方图，并将 `cursor_sharing` 参数设置为 `FORCE`——这将确保优化器为该列使用系统生成的绑定值，并确保该 SQL 语句在共享池中占用少得多的空间。正如你将在下一节看到的，如果你将 `cursor_sharing` 参数设置为 `FORCE` 并保留列上的直方图，Oracle Database 11g 的**自适应游标共享**提供了一个更好的解决方案。

## 13-15. 理解自适应游标共享

### 问题

你的数据库使用了用户定义的绑定变量。你想知道是否有任何方法可以优化数据库行为，使其不会对所有绑定变量值都“盲目地”使用相同的执行计划。

### 解决方案

在之前的版本中，无论绑定变量的值如何，Oracle 对每次 SQL 语句的执行都使用单一执行计划。在 Oracle Database 11g 中，名为 `自适应游标共享` 的数据库功能使带有绑定变量的 SQL 语句能够使用多个执行计划，每个执行计划基于绑定变量的值生成。自适应游标共享默认启用，且无法禁用它。


#### 自适应游标共享工作原理

自适应游标共享功能旨在改进包含绑定变量的 SQL 查询的执行计划。要理解自适应游标共享如何提供帮助，首先需要理解 Oracle 的**绑定窥探**功能的工作原理。绑定窥探（在 Oracle 9i 中引入）允许优化器在数据库首次调用游标时窥探绑定变量的值。优化器使用这个“窥探到的值”来确定 `WHERE` 子句的选择性。

使用用户定义绑定变量的问题在于，执行计划无法准确衡量 `WHERE` 子句的选择性。绑定窥探通过让优化器表现得像它实际在使用字面值而非绑定变量，从而帮助改善情况，进而为带有绑定变量的 SQL 语句生成更好的执行计划。当 `WHERE` 子句中的列值分布均匀时，绑定窥探效果良好。如果列值是倾斜的，优化器通过窥探用户定义的绑定变量值所选择的执行计划，可能并不一定对所有可能的绑定变量值都是最优的。因此，最终会导致一种情况：如果 SQL 语句使用的绑定变量值是优化器窥探到的那个值，执行计划会非常高效；而对于所有其他可能的绑定变量值，执行计划则效率低下。

让我们通过一个涉及具有倾斜数据的列的示例，来学习自适应游标共享是如何工作的。

我们的测试表 `DEMO` 有 78,681 行数据。数据有三列，且都是倾斜的。因此，在收集此表的统计信息时，我们在这三列上创建了直方图，如下所示。

```sql
SQL> select column_name,table_name,histogram from user_TAB_COLUMNS
  2      where table_name='DEMO';

COLUMN_NAME                     TABLE_NAME                     HISTOGRAM
------------------------------ ------------------------------ ---------------
RNUM                            DEMO                           HEIGHT BALANCED
RNAME                           DEMO                           HEIGHT BALANCED
STATUS                          DEMO                           FREQUENCY
```

请注意，当优化器注意到表上有直方图时，它会将该列中的数据标记为倾斜。`STATUS` 列有两个值：`Coarse` 和 `Fine`。只有 157 行的值是 `Coarse`，而 78,524 行的值是 `Fine`，这使得数据极度倾斜。

让我们执行一系列操作来说明自适应游标共享是如何工作的。发出一个查询，绑定变量设置为值 `Coarse`。由于 `DEMO` 表中只有很少的行具有此值，我们期望数据库使用索引范围扫描，这正是优化器所做的。这是我们的查询及其执行过程：

```sql
SQL> var b varchar2(6)
SQL> exec :b:='Coarse';

PL/SQL procedure successfully completed.

SQL> select /*+ ACS */ count(*) from demo where status = :b;
  COUNT(*)
----------
       157

SQL> select * from table(dbms_xplan.display_cursor);

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------
SQL_ID  cxau3vvabpzd0, child number 0
-------------------------------------
select /*+ ACS */ count(*) from demo where status = :b

Plan hash value: 3478245284
--------------------------------------------------------------------------------
| Id  | Operation         | Name       | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |            |       |       |     1 (100)|          |
|   1 |  SORT AGGREGATE   |            |     1 |     6 |            |          |

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------
|*  2 |   INDEX RANGE SCAN| IDX01_DEMO |   157 |   942 |     1   (0)| 00:00:52 |
--------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("STATUS"=:B)
19 rows selected.
```

接下来，执行以下语句以检查数据库是否已将 `STATUS` 列标记为**绑定敏感**或**绑定感知**，或两者兼有：

```sql
SQL> select child_number, executions, buffer_gets, is_bind_sensitive as
  2   "BIND_SENSI", is_bind_aware as "BIND_AWARE", is_shareable as "BIND_SHARE"
  3   from v$SQL
  4*  where sql_text like 'select /*+ ACS */%'
SQL> /

CHILD_NUMBER EXECUTIONS BUFFER_GETS BIND_SENSI   BIND_AWARE  BIND_SHARE
------------ ---------- ----------- -----------  ----------  -----------
           0          1          43          Y            N            Y

SQL>
```

请注意，数据库将 `STATUS` 列标记为绑定敏感，因为 `STATUS` 列上有直方图。每次使用不同的绑定变量值执行查询时，数据库都会将执行统计信息与先前执行的统计信息进行比较。如果执行统计信息差异显著，它会将该列标记为绑定感知。数据库在决定是否将语句标记为绑定感知时使用的输入之一是处理的行数。一旦游标被标记为绑定感知，优化器将根据绑定变量的值选择执行计划。这里，`IS_BIND_AWARE` 列标记为 `N`，因为没有先前的执行统计信息可供比较。`BIND_SHAREABLE` 列标记为 `Y`。

再次发出查询，绑定变量设置为值 `Fine`。由于几乎所有行的 `STATUS` 列值都是 `Fine`，我们期望优化器倾向于进行全表扫描。然而，优化器选择的执行计划与之前完全相同（`INDEX RANGE SCAN`）。原因是数据库正在使用与第一次执行相同的执行计划——例如：

```sql
SQL> exec :b := 'Fine';

PL/SQL procedure successfully completed.

SQL> select /*+ ACS */ count(*) from demo where status = :b;

  COUNT(*)
----------
     78524

SQL> select * from table(dbms_xplan.display_cursor);

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------
SQL_ID  cxau3vvabpzd0, child number 0
-------------------------------------
select /*+ ACS */ count(*) from demo where status = :b

Plan hash value: 3478245284
--------------------------------------------------------------------------------
| Id  | Operation         | Name       | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |            |       |       |     1 (100)|          |
|   1 |  SORT AGGREGATE   |            |     1 |     6 |            |          |

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------
|*  2 |   INDEX RANGE SCAN| IDX01_DEMO |   157 |   942 |     1   (0)| 00:00:52 |

--------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("STATUS"=:B)

19 rows selected.
```

由于 SQL 语句的游标被标记为绑定敏感，优化器使用了与之前相同的执行计划（`INDEX RANGE SCAN`）。在以下示例中请注意，`BIND_AWARE` 列仍然标记为 `N`。优化器正在使用与之前相同的游标（`child_number` 0）。

```sql
SQL> select child_number, executions, buffer_gets, is_bind_sensitive as
  2   "BIND_SENSI", is_bind_aware as "BIND_AWARE", is_shareable as "BIND_SHARE"
  3   from v$sql
  4   WHERE sql_text like 'select /*+ ACS */%';

CHILD_NUMBER EXECUTIONS BUFFER_GETS BIND_SENSI   BIND_AWARE  BIND_SHARE
------------ ---------- ----------- -----------  ----------  -----------
           0          2         220           Y           N            Y

SQL>
```

再次执行查询，使用的 `STATUS` 列值与上一次查询相同（'`Fine`'）。看！优化器现在使用了 `INDEX FAST FULL SCAN`，而不是 `INDEX RANGE SCAN`。执行计划的更改是自动的——就好像优化器在不断学习，并在确信新计划更高效时修改计划。以下是执行过程和新的计划：

```sql
SQL>  exec :b := 'Fine';
PL/SQL procedure successfully completed.

SQL> select /*+ ACS */ count(*) from demo where status = :b;
  COUNT(*)
----------
     78524

SQL> select * from table(dbms_xplan.display_cursor);
PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------
SQL_ID  cxau3vvabpzd0, child number 1
-------------------------------------
select /*+ ACS */ count(*) from demo where status = :b

Plan hash value: 2683512795
--------------------------------------------------------------------------------
| Id  | Operation             | Name       | Rows  | Bytes | Cost (%CPU)| Time   |

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |            |       |       |    45 (100)|        |
   |
|   1 |  SORT AGGREGATE       |            |     1 |     6 |            |        |
   |
|*  2 |   INDEX FAST FULL SCAN| IDX01_DEMO | 78524 |   460K|    45   (0)| 00:38:
PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------
37 |
--------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
   2 - filter("STATUS"=:B)
19 rows selected.
```

请注意，`BIND_AWARE` 列现在显示值 `Y`。当我们第二次使用相同的绑定变量值（`Fine`）执行查询时，由于查询被标记为绑定敏感，数据库会评估上一次执行的执行统计信息。由于统计信息不同，它将游标标记为绑定感知。优化器随后决定新计划更优，因此执行硬解析并生成了一个使用 `INDEX FAST FULL SCAN` 而不是 `INDEX RANGE SCAN` 的新执行计划。以下查询显示了子游标的详细信息以及查询是绑定敏感还是绑定感知。

```sql
SQL> select child_number, executions, buffer_gets, is_bind_sensitive as
  2   "BIND_SENSI", is_bind_aware as "BIND_AWARE", is_shareable as "BIND_SHARE"
  3   from v$sql
  4   WHERE sql_text like 'select /*+ ACS */%';

CHILD_NUMBER EXECUTIONS BUFFER_GETS BIND_SENSI   BIND_AWARE  BIND_SHARE
------------ ---------- ----------- -----------  ----------  -----------
           0          2         220           Y           N            Y
           1          1         184           Y           Y            Y

SQL>
```

请注意，`IS_BIND_AWARE` 列现在显示值 `Y`。还要注意，出现了一个新的子游标（`child_number` 1），它代表包含 `INDEX FAST FULL SCAN` 的新执行计划——这个新游标被标记为绑定感知。

我们再次执行查询，但这次使用原始的绑定变量值 `Coarse`。优化器将通过执行 `INDEX RANGE SCAN` 来选择正确的执行计划。以下是有关子游标的信息，以及查询是绑定敏感还是绑定感知。

```sql
SQL> select child_number, executions, buffer_gets, is_bind_sensitive as
  2   "BIND_SENSI", is_bind_aware as "BIND_AWARE", is_shareable as "BIND_SHARE"
  3   from v$sql
  4   where sql_text like 'select /*+ ACS */%';

CHILD_NUMBER EXECUTIONS BUFFER_GETS BIND_SENSI   BIND_AWARE  BIND_SHARE
------------ ---------- ----------- -----------  ----------  -----------
           0          2         220           Y           N            N
           1          1         184           Y           Y            Y
           2          1           2           Y           Y            Y

SQL>
```

数据库为此查询创建了一个新的子游标（`child_number`=2），并将原始游标（`child_cursor`=0）标记为非绑定感知。最终，数据库将从此游标从共享池中移除此游标。

在我们的示例中，我们在测试中仅使用了两个绑定变量值。如果有数十种不同的绑定变量值会怎样？Oracle 并非总是为每个不同的绑定变量值执行硬解析。最初它会对一些绑定变量值执行硬解析，在此过程中确定各种绑定变量与相关执行计划之间的关系。在建立了绑定变量值与相关执行计划的初始映射之后，Oracle 足够智能，可以简单地从缓存中选择最优的子游标，而无需为其他绑定值执行硬解析。

自适应游标共享是在 Oracle Database 11g 版本中引入的一项新功能。在更早的版本中，当 DBA 遇到数据库由于绑定窥探效应显然开始为某个 SQL 语句使用不合适的执行计划的情况时，常常会刷新共享池（更糟糕的是，有时会重启数据库）。在 11g 版本中，你无需做任何事情——优化器在遇到倾斜数据时会自动更改执行计划。借助自适应游标共享，数据库对使用绑定变量的语句使用多个执行计划，确保始终根据绑定变量的值使用最佳执行计划。自适应游标共享意味着当不同的绑定变量值表明查询需要处理的数据量不同时，Oracle 会调整其行为，为查询使用不同的执行计划，而不是对所有绑定值坚持使用相同的计划。由于自适应游标共享仅在字面值被绑定替换的地方有效，Oracle 鼓励你对 `cursor_sharing` 参数使用 `FORCE` 设置。如果将该参数设置为 `SIMILAR`，并且在列上有直方图，优化器不会用绑定变量执行字面替换，因此自适应游标共享不会发生。你必须将 `cursor_sharing` 参数设置为 `FORCE`，自适应游标共享才能工作，从而让优化器为不同的绑定变量值选择最优的执行计划。

## 13-16. 在表达式上创建统计信息

### 问题

你想在表达式（例如用户创建的函数）上创建统计信息。

### 解法

按以下方式执行 `DBMS_STATS` 包中的 `GATHER_TABLE_STATS` 过程，以收集表达式上的统计信息。在此示例中，我们正在为 `lower` 函数收集统计信息，该函数转换 `cust_state_province` 列。

```sql
SQL> execute dbms_stats.gather_table_stats('sh','customers',-
   > method_opt =>'for all columns size skewonly -
   > for columns(lower(cust_state_province)) size skewonly');

PL/SQL 过程成功完成。

SQL>
```

或者，你可以通过调用 `create_extended_stats` 函数来收集表达式统计信息——例如：

```sql
SQL> select
  2  dbms_stats.create_extended_stats(null,'customers','(lower(cust_state_province))')
  3  from dual;
```

注意，`lower(cust_state_province)` 被称为一个*扩展*，因为对函数收集统计信息是 Oracle 扩展统计信息的一种类型。你为表达式和列组（参见配方 13-17）收集的任何统计信息都称为“扩展统计信息”。

### 工作原理

优化器了解表列的选择性，并使用选择性估计值来创建最优执行计划。然而，在查询的 `WHERE` 子句中对列应用函数会使优化器难以处理，因为它无法估计底层列的选择性。下面是一个使优化器工作更困难的函数示例：

```sql
SQL> select count(*) from customers
     where lower(cust_state_province) = 'CA';
```

函数上的表达式统计信息使优化器能够为涉及表达式的谓词获得更准确的选择性值。

你可以发出以下查询以查找表列上表达式统计信息的详细信息：

```sql
SQL> select extension_name, extension
     from user_stat_extensions
     where table_name='CUSTOMERS';

EXTENSION_NAME                                      EXTENSION
--------------------------------------------------  ------------------------------
SYS_STUBPHJSBRKOIK9O2YV3W8HOUE                      (LOWER("CUST_STATE_PROVINCE"))
SQL>
```

你可以使用 `drop_extended_stats` 函数删除已在表上收集的表达式统计信息：

```sql
SQL> exec dbms_stats.drop_extended_stats(null,'customers','(lower(cust_state_province))');

PL/SQL 过程成功完成。

SQL>
```

注意，扩展统计信息既包括函数等表达式上的统计信息，也包括为由两个或多个相关列组成的列组收集的统计信息。配方 13-17 展示了如何收集列组的统计信息。

## 13-17. 为相关列创建统计信息

### 问题

你意识到作为连接条件一部分的表中的某些列是相关的。你希望让优化器知道这种关系。

### 解法

为了为两个或多个相关列生成统计信息，你必须首先创建一个*列组*，然后为表收集新的统计信息，以便优化器可以使用新生成的“扩展统计信息”。使用 `DBMS_STATS.CREATE_EXTENDED_STATS` 函数来定义一个由表中两个或多个列组成的列组。以下是执行此函数以在表 `SH.CUSTOMERS` 中创建包含 `COUNTRY_ID` 和 `CUST_STATE_PROVINCE` 列的列组的方法。

```sql
SQL> select dbms_stats.create_extended_stats(null,'CUSTOMERS', '(country_id,cust_state_province)') from dual;

DBMS_STATS.CREATE_EXTENDED_STATS(NULL,'CUSTOMERS','(COUNTRY_ID,CUST_STATE_PROVINCE)')
--------------------------------------------------------------------------------
SYS_STUJGVLRVH5USVDU$XNV4_IR#4
SQL>
```

创建列组后，收集 `CUSTOMERS` 表的新统计信息，以为新列组生成统计信息。

```sql
SQL> exec dbms_stats.gather_table_stats(null,'customers');

PL/SQL 过程成功完成。

SQL>
```

### 工作原理

通常，表中一列中的值会影响该表中另一列的值，这是由于存储在这两列中的数据之间存在的自然关系。例如，`SH.CUSTOMERS` 表中的 `CUST_STATE_PROVINCE` 列的值受到 `COUNTRY_ID` 列值的影响。你只会在美国找到 `CUST_STATE_PROVINCE` 值为 `Florida` 的记录。优化器不了解现实生活中的关系，因此当多个相关列出现在查询的 `WHERE` 子句或 `group_by` 键中时，往往会对至关重要的基数统计信息产生错误的估计。列组统计信息有助于优化器捕获表列之间的相关性。如果查询包含谓词 `CUST_STATE_PROVINCE ='Florida'` 和 `COUNTRY_ID=U.S.`，Oracle 可以通过查找列组的统计信息而不是使用两个列的单独统计信息，来推导出这两个谓词组合选择性的更好估计值。

数据库为你在创建的列组上收集的统计信息称为扩展统计信息。这些统计信息为优化器提供了更准确的基数估计，这有助于优化器生成更高效的执行计划。当你创建扩展统计信息时，Oracle 会维护你创建的列组的统计信息子集，包括不同值的数量、空值以及组的直方图。即使查询包含除了列组中列之外的列，优化器也会利用可用的扩展统计信息。例如，假设你已如本配方所示，使用 `CUST_STATE_PROVINCE` 和 `COUNTRY_ID` 列创建了一个列组。如果查询的 `WHERE` 子句除了这两个列之外还包含 `CUST_CITY` 列，Oracle 仍将利用 `CUST_STATE_PROVINCE` 和 `COUNTRY_ID` 列上的扩展统计信息。

## 13-18. 自动创建列组

### 问题

你知道通过在相关表列上生成统计信息来创建扩展统计信息有助于生成更好的执行计划。你想了解如何选择候选列组来创建扩展统计信息。


### 13-18. 自动列组创建

#### 解决方案

在 Oracle Database 11.2.0.2 及更高版本中，您可以使用 *自动列组创建功能*，让数据库告诉您必须创建哪些列组。此功能仅用于创建列组，不能用于为包含表达式的列收集扩展统计信息（参见配方 13-16）。

要使用自动列组创建功能并让数据库提供建议，您必须让数据库监控其工作负载。首先执行 `DBMS_STATS.SEED_COL_USAGE` 过程来确定需要创建的适当列组：

```
SQL> begin
      dbms_stats.seed_col_usage(null,null,900);
      end;
      /
```

通过执行此过程，您告知数据库监控工作负载 15 分钟（900 秒），以确定是否需要创建任何列组。该过程捕获列使用信息并存储在 `sys.col_group_usage$` 视图中。

接下来运行一些查询以创建工作负载。如果查询运行时间很长，您可以仅运行这些查询的执行计划，以便数据库捕获列组信息。监控期（15 分钟）结束后，使用以下查询查看捕获的列使用信息：

```
SQL> select dbms_stats.report_col_usage(user,'customers') from dual;
```

`REPORT_COL_USAGE` 过程显示 `CUSTOMERS` 表的列使用情况报告，基于您已执行的查询和运行的执行计划。列使用情况显示了数据库如何使用 `CUSTOMERS` 表的每一列，并以以下格式列出：

*   `Equality predicates (EQ)`：如果某列用于等式谓词，例如在子句 `where COUNTRY_ID='US'` 中，则该列是独立使用的。在这种情况下不需要扩展统计信息。
*   `FILTER`：如果一组列用于 `SELECT` 语句，并且这些列中的一个或多个出现在该语句的 `GROUP BY` 子句中，则 `SELECT` 语句中的所有列都被记录为一个列组过滤器。
*   `GROUP_BY`：在 `GROUP_BY` 子句中一起使用的所有列。

查看列使用情况报告后，您可以允许 Oracle 自动为过滤器谓词中使用的列和 `GROUP_BY` 子句中使用的列创建列组。通过执行以下过程完成：

```
SQL> select dbms_stats.create_extended_stats(user,'customers') from dual;
```

或者，您可以只为指定的列创建列组，通过发出以下命令：

```
SQL> select dbms_stats.create_extended_stats(null,'CUSTOMERS', '(cust_city,cust_state_province,country_id)') from dual
SQL> /
SYS_STUMZ$C3AIHLPBROI#SKA58H_N
SQL>
```

此时，您已经创建了列组，但列组上没有统计信息。重新收集 `CUSTOMERS` 表的统计信息以为新列组生成统计信息——例如：

```
SQL> exec dbms_stats.gather_table_stats(user,'customers')
PL/SQL procedure successfully completed.
SQL>
```

#### 工作原理

让 Oracle 根据工作负载期间的实际列使用情况指出潜在的列组，比您尝试为每个表确定适当的列组要高效得多。一旦运行了工作负载，您可以查看列使用情况报告，并通过执行 `dbms_stats.create_extended_stats` 函数并将 `table_name` 参数的值传递为 `NULL`，来要求数据库同时为整个模式创建所有提议的列组。

### 13-19. 维护分区表上的统计信息

#### 问题

您频繁地向一个或多个分区加载数据，维护全局统计信息成了问题。您希望在不经历耗时耗资源的过程的情况下收集新的全局统计信息。

#### 解决方案

您可以使用 Oracle 11g 版本中的 *增量统计信息维护* 功能，在每次加载新分区后维护全局统计信息。例如，如果您想维护 `SH.SALES` 表的全局统计信息，请遵循以下步骤：

1.  为 `SH.SALES` 表启用增量统计信息收集：
    ```
    SQL> exec dbms_stats.set_table_prefs('SH','SALES','INCREMENTAL','TRUE');
    PL/SQL procedure successfully completed.
    SQL>
    ```
2.  在向分区加载数据后，收集全局表级统计信息，如下所示：
    ```
    SQL> exec dbms_stats.gather_table_stats('SH','SALES');
    PL/SQL procedure successfully completed.
    SQL>
    ```

要为分区表设置增量统计信息收集功能，必须为 `ESTIMATE_PERCENT` 参数指定 `AUTO_SAMPLE_SIZE` 值，并为 `GRANULARITY` 参数指定 `AUTO` 值。



