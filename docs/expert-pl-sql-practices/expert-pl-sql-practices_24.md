# 使用批量绑定时的错误处理

使用批量绑定能带来巨大的好处。但在一个需要格外注意的方面是错误处理。在逐行修改行的开发模型中，当某条 SQL 语句执行失败时，有问题的行是隐含的——它就是你正在处理的那一行。

## 逐行处理 vs. 批量错误的模糊性

例如，在下面这个包含两条 `insert` 语句的简单 PL/SQL 块 (`bulk_error_1.sql`) 中，很容易看出是哪条 `insert` 语句出了问题：

```sql
SQL> alter table hardware
  2    add constraint
  3    hardware_chk check ( item > 0 );

Table altered.

SQL> begin
  2    insert into hardware ( item )  values (1);
  3    insert into hardware ( item ) values (-1);
  4    insert into hardware ( item ) values (2);
  5    insert into hardware ( item ) values (3);
  6    insert into hardware ( item ) values (4);
  7    insert into hardware ( item ) values (-2);
  8  end;
  9  /
begin
*
ERROR at line 1:
ORA-02290: check constraint (MCDONAC.HARDWARE_CHK) violated
ORA-06512: at line 3
```

当你转向以批量方式修改行时，情况会变得稍微复杂一些，因此需要更加谨慎。单个 `FORALL` 语句可能涉及处理数百行数据。当将前面的例子用批量模式重复时 (`bulk_error_2.sql`)，就无法立即看出是哪一行导致了错误。

```sql
SQL> declare
  2    type t_list is table of hardware.item%type;
  3    l_rows t_list := t_list(1,-1,2,3,4,-2);
  4  begin
  5    forall i in 1 .. l_rows.count
  6      insert into hardware ( item ) values (l_rows(i));
  7  end;
  8  /
declare
*
ERROR at line 1:
ORA-02290: check constraint (MCDONAC.HARDWARE_CHK) violated
ORA-06512: at line 5
```

与正常的语句级操作一样，默认行为是回滚整个批量绑定操作。因此，即使数组中的一行数据是有效的，目标表中也不会有任何行存在。

```sql
SQL> select count(*) from HARDWARE;

no rows selected.
```

## PL/SQL 中的隐式回滚行为

请注意，更改的回滚本身并不是批量绑定的属性；它是 PL/SQL 事务管理的一个标准部分。关于 PL/SQL 的一个常见误解是，它只是一系列独立 SQL 语句的包装器。因此，在第一个例子中，当第二条 `insert` 语句失败时，

```sql
begin
  insert into DEMO values (1);
  insert into DEMO values (-1);
end;
```

开发者会感到有必要采取补救措施来撤销第一条 `insert`。在 PL/SQL 模块中，经常可以看到类似下面这样的异常处理代码：

```sql
begin
  insert into DEMO values (1);
  insert into DEMO values (-1);
exception
  when others then
    rollback;
    raise;
end;
```

**没有要求执行这样的回滚**。此外，这种回滚通常会导致应用程序中的数据损坏。PL/SQL 块会隐式创建一个保存点。因此，无论错误发生在 PL/SQL 块中的哪个位置，块中的所有更改都会自动回滚到一个点，就好像这个 PL/SQL 例程从未被调用过一样。这种行为是 PL/SQL 真正强大的特性之一。很少有其他语言能让事务管理变得如此简单。

## 使用 `SAVE EXCEPTIONS` 子句

现在回到批量绑定的例子，通过简单的代码检查可以看出是值为 -1 的第二个数组条目导致了问题。然而，这是因为这个例子非常简单。错误信息本身并没有揭示是哪个数组条目引起的——只说明有一个或多个条目违反了约束。

在 Oracle 9 中，通过扩展批量绑定语法来添加 `SAVE EXCEPTIONS` 子句解决了这个问题。错误仍然会发生，但会提供额外的信息，以便诊断出哪些数组条目出错。让我们如下修改示例 (`bulk_error_3.sql`) 来演示如何使用 `SAVE EXCEPTIONS`：

```sql
SQL> declare
  2    type t_list is table of hardware.item%type;
  3    l_rows t_list := t_list(1,-1,2,3,4,-2);
  4  begin
  5    forall i in 1 .. l_rows.count save exceptions
  6      insert into hardware ( item ) values (l_rows(i));
  7  end;
  8  /
ERROR:
ORA-24381: error(s) in array DML
ORA-06512: at line 5
```

乍一看，似乎没取得什么进展，但这个例子表明，现在抛出的是一个新的异常 (`ORA-24381`)，而不是之前的约束违反错误 (`ORA-02290`)。通过处理这个特定的批量绑定异常，我可以访问一些特殊的属性，从而深入探究错误原因。

## 访问错误集合

例如，下面的代码 (`bulk_error_4.sql`) 在异常处理器中引入了更多代码，以揭示错误的真正原因：

```sql
SQL> set serverout on
SQL> declare
  2    type t_list is table of hardware.item%type;
  3    l_rows t_list := t_list(1,-1,2,3,4,-2);
  4
  5    bulk_bind_error exception;
  6    pragma exception_init(bulk_bind_error,-24381);
  7
  8  begin
  9    forall i in 1 .. l_rows.count save exceptions
 10      insert into hardware ( item ) values (l_rows(i));
 11
 12  exception
 13    when bulk_bind_error then
 14      dbms_output.put_line(
 15          'There were '||sql%bulk_exceptions.count||' errors in total');
 16      for i in 1 .. sql%bulk_exceptions.count loop
 17         dbms_output.put_line(
 18            'Error '||i||' occurred at array index:'||
 19             sql%bulk_exceptions(i).error_index);
 20         dbms_output.put_line('- error code:'||
 21             sql%bulk_exceptions(i).error_code);
 22         dbms_output.put_line('- error text:'||
 23             sqlerrm(-sql%bulk_exceptions(i).error_code));
 24      end loop;
 25
 26  end;
 27  /
There were 2 errors in total
Error 1 occurred at array index:2
- error code:2290
- error text:ORA-02290: check constraint (.) violated
Error 2 occurred at array index:6
- error code:2290
- error text:ORA-02290: check constraint (.) violated

PL/SQL procedure successfully completed.
```

当在 `FORALL` 中使用 `SAVE EXCEPTIONS` 语法时，任何错误都会在一个名为 `SQL%BULK_EXCEPTIONS` 的新集合中可用，该集合为每个错误包含一行。该新集合中的每一行包含以下内容：

*   `error_index:` 在 `FORALL` 中使用的集合的索引
*   `error_code:` Oracle 错误代码；请注意错误代码是正数（与 PL/SQL 中的 `SQLCODE` 内置函数不同）。

> **提示** 注意 Oracle 文档的旧版本。一些代码示例将 `SQL%BULK_EXCEPTIONS` 的引用直接内联在代码中，紧接在 `FORALL` 语句下面，这暗示着不会引发异常。如前一个演示所示，你必须在异常处理代码部分编写对 `SQL%BULK_EXCEPTIONS` 集合的引用。

此外，由于 `SQL%BULK_EXCEPTIONS` 属性是一个集合，可以捕获和处理多个错误。在前面的例子中，因为异常处理器没有将错误重新抛回调用环境，所以成功插入的行仍然保留在表中。

```sql
SQL> select item from HARDWARE;

         X
----------
         1
         2
         3
         4
```



#### 使用批处理保存异常

如前所述，当批量绑定大量行时，你将以较小的块来处理这些行，以避免消耗过多的会话 PGA 内存。但是，如果你是分批进行批量绑定，那么每个 `FORALL` 调用都会重新初始化 `SQL%BULK_EXCEPTIONS` 结构。因此，在这种情况下，该结构无法在一系列批量绑定调用中容纳整个被拒绝的行集。一种可能的解决方法是捕获每次批量绑定调用中的任何错误，将它们保存到表中，然后处理下一个 1000 行的批次。下面展示了这种方法的一个示例 (`bulk_error_5.sql`)：

```sql
SQL> create table ERRS
  2    ( error_index   number(6),
  3      error_code    number(6),
  4      item          number );

Table created.

SQL> set serverout on
SQL> declare
  2    type t_list is table of hardware.item%type;
  3    l_rows t_list := t_list(1,-1,2,3,4,-2);
  4
  5    bulk_bind_error exception;
  6    pragma exception_init(bulk_bind_error,-24381);
  7
  8    type t_err_list is table of ERRS%rowtype index by pls_integer;
  9    l_err_list t_err_list;
 10
 11  begin
 12    forall i in 1 .. l_rows.count save exceptions
 13      insert into hardware ( item ) values (l_rows(i));
 14
 15  exception
 16    when bulk_bind_error then
```

如果有行出错，那么一个新的结构 (`l_err_list`) 将被填充，其中包含来自 `sql%bulk_exceptions` 集合的所有信息。

```sql
 17      for i in 1 .. sql%bulk_exceptions.count loop
 18         l_err_list(i).error_index := sql%bulk_exceptions(i).error_index;
 19         l_err_list(i).error_code := sql%bulk_exceptions(i).error_code;
 20         l_err_list(i).item := l_rows(sql%bulk_exceptions(i).error_index);
 21      end loop;
```

然后，这些信息当然会使用批量绑定保存到 `ERRS` 表中！

```sql
 22      forall i in 1 .. l_err_list.count
 23         insert into ERRS values l_err_list(i);
 24
 25  end;
 26  /

PL/SQL procedure successfully completed.

SQL> select * from errs;

ERROR_INDEX ERROR_CODE       ITEM
----------- ---------- ----------
          2       2290          -1
          6       2290          -2
```

#### LOG ERRORS 子句

或者，你可以通过将代码转换为纯 SQL 方法并利用 `LOG ERRORS` 子句来实现类似的效果。从 10g 版本开始，如果你打算使用批量绑定的 DML 操作可以在 SQL 中原生表达，你可以以类似于 `SAVE EXCEPTIONS` 子句的方式捕获错误。下面的 DDL 和代码包含在 `bulk_log_errors.sql` 文件中。

首先，使用提供的 `DBMS_ERRLOG` 包创建一个表来捕获错误。

```sql
SQL> exec DBMS_ERRLOG.CREATE_ERROR_LOG('HARDWARE');

PL/SQL procedure successfully completed.
```

此执行创建了一个表，该表包含来自 `HARDWARE` 表的列，以及用于指示发生了何种错误的额外列。生成的表结构如下：

```sql
SQL> desc err$_HARDWARE
 Name                       Null?    Type
 -------------------------- -------- ----------------
 ORA_ERR_NUMBER$                     NUMBER
 ORA_ERR_MESG$                       VARCHAR2(2000)
 ORA_ERR_ROWID$                      ROWID
 ORA_ERR_OPTYP$                      VARCHAR2(2)
 ORA_ERR_TAG$                        VARCHAR2(2000)
 AISLE                               VARCHAR2(4000)
 ITEM                                VARCHAR2(4000)
 DESCR                               VARCHAR2(4000)
 STOCKED                             VARCHAR2(4000)
```

然后，使用附加的日志错误子句执行 SQL 语句以捕获错误，如下所示：

```sql
SQL> insert
  2  into HARDWARE ( item )
  3  with SRC_ROWS as
  4    ( select -3 + rownum     x from dual
  5      connect by level <= 6 )
  6  select x
  7  from SRC_ROWS
  8  log errors reject limit unlimited;

3 rows created.
```

你会发现，与批量绑定示例 (`bulk_bind_error_5.sql`) 类似，错误行及其错误原因已被捕获。

```sql
SQL> select
  2   item
  3  ,ora_err_number$
  4  ,ora_err_mesg$
  5  from err$_HARDWARE;

ITEM  ORA_ERR_NUMBER$ ORA_ERR_MESG$
----- --------------- ------------------------------------------------------------
-2               2290 ORA-02290: check constraint (MCDONAC.HARDWARE_CHK) violated
-1               2290 ORA-02290: check constraint (MCDONAC.HARDWARE_CHK) violated
0                2290 ORA-02290: check constraint (MCDONAC.HARDWARE_CHK) violated
```

全面介绍 `LOG ERRORS` 扩展超出了本书的范围，但前面的示例演示了如何以与 PL/SQL 中 `SAVE EXCEPTIONS` 功能非常相似的方式使用它，并且还有一个额外的好处：错误数据可直接在一个表中获得，以便进一步处理。

#### 健壮的批量绑定

当你使用 `bulk collect` 收集数据到集合中时，数组是从索引 1 开始填充的。标准文档中的几乎所有示例都基于此假设，代码通常类似于：

```sql
begin
  select ...
  bulk collect into <array>
  from ...

  for i in 1 .. <array>.count loop
    <processing>
  end loop;
end;
```

Oracle 不太可能改变这种行为，因此可以合理地假设任何使用 `bulk collect` 初始化的集合将始终从数组索引 1 开始。然而，集合并不完全属于 PL/SQL 中 `bulk collect`/`bulk bind` 功能的范畴。事实上，集合的出现早于批量操作，可以一直追溯到 Oracle 7。一旦数据被提取到集合中，开发人员可以自由地对集合内容进行任何操作。那么，如果集合不再包含一组从 `index=1` 开始的元素，批量绑定操作会发生什么？让我们探讨一些场景。

##### 场景 1：元素不从 1 开始

只要数组的索引是连续的，你就可以使用数组本身的属性来继续使用批量绑定。考虑以下示例 (`bulk_bind_scenario1.sql`)，其中数组从 10 开始。可以使用数组属性 `FIRST` 和 `LAST` 来定义批量绑定的数组范围。

```sql
SQL> declare
  2    type t_num_list is table of hardware.item%type index by pls_integer;
  3
  4    val t_num_list;
  5  begin
  6
  7    val(10) := 10;
  8    val(11) := 20;
  9    val(12) := 20;
 10
 11    FORALL i IN val.first .. val.last
 12       insert into hardware ( item ) values (val(i));
 13
 14  end;
 15  /

PL/SQL procedure successfully completed.
```

事实上，采用一个标准，即优先使用 `.FIRST` 和 `.LAST` 属性而不是 `1` 和 `.COUNT`，可能是合理的。然而，接下来的场景表明，这样做并不能提供完全的保障。


##### 场景二：元素不连续

让我们对前面的例子稍作修改：数组中缺少一个条目。以下是该示例以及因缺失条目导致的错误 (`bulk_bind_scenario2a.sql`)：

```sql
SQL> declare
  2    type t_num_list is table of hardware.item%type index by pls_integer;
  3
  4    val t_num_list;
  5  begin
  6
  7    val(10) := 10;
  8  --  val(11) := 20;
  9    val(12) := 20;
 10
 11    FORALL i IN val.first .. val.last
 12       insert into hardware ( item ) values (val(i));
 13
 14  end;
 15  /
declare
*
ERROR at line 1:
ORA-22160: element at index [11] does not exist
ORA-06512: at line 11
```

一旦集合变得稀疏，批量绑定将无法自动使用高低边界索引值进行工作。然而，从版本 10.2 开始，`FORALL` 语法被扩展，包含了 `INDICES OF` 和 `VALUES OF` 规范。使用 `INDICES OF` 可以解决上面遇到的 ORA-22160 问题，如下所示 (`bulk_bind_scenario2b.sql`)：

```sql
SQL> declare
  2    type t_num_list is table of hardware.item%type index by pls_integer;
  3
  4    val t_num_list;
  5  begin
  6
  7    val(10) := 10;
  8  --  val(11) := 20;
  9    val(12) := 20;
 10
 11    FORALL i IN INDICES OF val
 12       insert into hardware ( item ) values (val(i));
 13
 14  end;
 15  /
```

```
PL/SQL procedure successfully completed.
```

我非常喜欢这个语法。它不依赖于数组属性，并且代码独立于数据在数组中的分布方式。我建议采用一个标准：每当您想要处理整个集合时，就使用 `INDICES OF` 子句，并且在您的 PL/SQL 代码中应弃用 `.FIRST`、`.LAST` 和 `.COUNT` 的使用。遗憾的是，`INDICES OF` 扩展只能在 `FORALL` 语句中使用，而不能在标准的 `FOR` 循环中使用。

然而，如果您需要执行的批量绑定更像是对现有集合进行切割和切片，那么 `VALUES OF` 语法就派上用场了。`VALUES OF` 子句允许一定程度的间接寻址，有点类似于指针，使您能够对较大集合的选定子集进行批量绑定。

在下一个示例中 (`bulk_bind_values_of.sql`)，将填充一个包含 `STATUS` 属性的集合：状态标记为 `NEW` 的行将被插入到 `HARDWARE` 表中，而状态标记为 `UPD` 的行将更新其在 `HARDWARE` 表中的匹配行。因此，该示例模拟了一个 `MERGE` 语句。将检查该集合以确定哪些索引应用于更新，哪些应用于插入。然后，`VALUES OF` 子句将用于处理这些行。

首先，我将创建一些表示输入数据的结构。变量 `src` 将是输入数据，包含 100 行，每行状态为 `NEW` 或 `UPD`。

```sql
SQL> set serverout on
SQL> declare
  2    type t_input_row is record (
  3       item   hardware.item%type,
  4       descr  hardware.descr%type,
  5       status varchar2(3)
  6       );
  7
  8    type t_input_list is
  9       table of t_input_row
 10       index by pls_integer;
 11
 12    src t_input_list;
```

现在定义了两个变量 (`ind_new` 和 `ind_upd`)，它们将保存 `src` 中相应行的索引值。通过这种方式，不是将源数据复制到单独的集合中（一个用于 status = `NEW`，另一个用于 status = `UPD`），而是仅保留索引条目的记录。如果源数据量很大，避免复制尤为重要。

```sql
 13
 14    type t_target_indices is
 15       table of pls_integer
 16       index by pls_integer;
 17
 18    ind_new  t_target_indices;
 19    ind_upd  t_target_indices;
 20
 21  begin
```

现在，源数据被植入一些虚构的值（80 行状态为 `UPD`，20 行状态为 `NEW`）。在真实世界的例子中，数据可能在其他地方初始化并传递给应用程序进行处理。

```sql
 22    for i in 1 .. 100 loop
 23       src(i).item   := i;
 24       src(i).descr  := 'Item '||i;
 25       src(i).status :=  case when mod(i,5) = 0 then 'NEW' else 'UPD' end;
 26    end loop;
 27
```

现在扫描数据，并用与两个不同状态值对应的索引来填充索引数组。

```sql
 28
 29    for i in 1 .. 100 loop
 30       if src(i).status = 'NEW' then
 31          ind_new(ind_new.count) := i;
 32       else
 33          ind_upd(ind_upd.count) := i;
 34       end if;
 35    end loop;
```

最后，使用 `VALUES OF` 语法将更改传输到 `HARDWARE` 表中。即使与每个表相关的行在 `src` 集合中稀疏分布，`VALUES OF` 语法也提供了对它们进行批量绑定的直接访问。

```sql
 36
 37    forall i in values of ind_new
 38       insert into hardware ( aisle, item)
 39       values (1, src(i).item);
 40    dbms_output.put_line(sql%rowcount||' rows inserted');
 41
 42    forall i in values of ind_upd
 43       update hardware
 44       set descr = src(i).descr
 45       where aisle = 1
 46       and item = src(i).item;
 47    dbms_output.put_line(sql%rowcount||' rows updated');
 48
 49  end;
 50  /
```

```
20 rows inserted
80 rows updated
```

`VALUES OF` 和 `INDICES OF` 语法完善了批量绑定的实现。数组内的任何条目排列都可以被操作并绑定到数据库表中。

##### Oracle 的早期版本

如果您使用的是 Oracle 10.1 或更低版本，那么您的 Oracle 版本尚不支持 `VALUES OF` 或 `INDICES OF` 扩展，但并非没有解决办法。如果一个集合可能是稀疏的，那么将其转换为密集集合即可解决问题。这并非可以轻率为之的事情，因为如果集合很大，那么在对数据进行密集化处理时，您将在内存中持有该集合的两个副本。我将回到本节的第一个示例，其中输入数组不包含连续条目，并在不使用 `VALUES OF` 的情况下解决该问题 (`bulk_bind_scenario_oldver.sql`)。

首先，这里是声明第一个集合 `val` 的代码，它将是一个稀疏集合：

```sql
SQL> declare
  2    type t_num_list is table of hardware.item%type
  3      index by pls_integer;
  4
  5    val t_num_list;
```

现在定义第二个数组，它将包含来自稀疏集合 `val` 的条目：

```sql
  6
  7    dense_val t_num_list;
  8    idx       pls_integer;
  9
 10  begin
 11
 12    val(10) := 10;
 13    val(12) := 20;
 14
```

通过遍历 `val` 集合的条目（使用集合属性 `.FIRST` 和 `.NEXT`）来执行 `dense_val` 数组的填充。集合 `dense_val` 将从 0 开始（因为 `dense_val.count` 初始为零），并增长到 `val` 中的元素数量。

```sql
 15    idx := val.first;
 16    while idx is not null loop
 17       dense_val(dense_val.count) := val(idx);
 18       idx := val.next(idx);
 19    end loop;
```

然后使用 `dense_val` 集合而不是稀疏的 `val` 来执行批量绑定：

```sql
 20
 21    FORALL i IN dense_val.first .. dense_val.last
 22       insert into hardware ( item )
 23       values (dense_val(i));
 24
 25  end;
 26  /
```

```
PL/SQL procedure successfully completed.
```

### 对海量集合使用的合理性论证

正如本章通篇所见，一旦开始使用集合，你就需要注意其对内存消耗的影响。然而，有时你的性能要求源于数据库基础设施的其他部分。

你看到的许多示例都涉及向表中插入大量行。虽然批量绑定比单行插入快得多，但由于插入仍然是通过常规路径发出的 DML，它仍然会消耗大量的重做日志，因此性能可能仍会受到轻微影响。例如，性能可能因为空闲列表或段空间位图管理而受到影响，或者因为推进了表的高水位线而受到影响。

Oracle Database 11.2 引入了 `APPEND_VALUES` 提示，它会将常规路径插入语句转换为直接路径加载。以下是使用该提示的一个示例 (`bulk_bind_append_values_1.sql`)：

```
SQL> insert /*+ APPEND_VALUES */
  2  into HARDWARE ( item ) values (1);

1 row created.

SQL> select item from HARDWARE;
select item from HARDWARE
                 *
ERROR at line 1:
ORA-12838: cannot read/modify an object after modifying it in parallel

SQL> commit;

Commit complete.

SQL> select item from HARDWARE;

      ITEM
----------
         1
```

当 `APPEND_VALUES` 提示发布时，其实用性受到了质疑。毕竟，谁会为了仅仅插入一个`单行`数据，就愿意锁定一个表、推进其高水位线并必须立即结束事务呢？然而，当与批量绑定结合使用时，该功能的实用性就变得更加明显。在下一个示例 (`bulk_bind_append_values_2.sql`) 中，测量了会话级别的统计数据，以比较常规插入与通过批量绑定进行的直接加载插入在重做日志消耗上的差异：

```
SQL> declare
  2    type t_list is table of hardware.descr%type;
  3    l_rows t_list := t_list();
  4    l_now timestamp;
  5    l_redo1 int;
  6    l_redo2 int;
  7    l_redo3 int;
  8  begin
  9
 10    select value
 11    into   l_redo1
 12    from   v$mystat m, v$statname s
 13    where  s.statistic# = m.statistic#
 14    and    s.name = 'redo size';
 15

    -- 准备一个包含 1,000,000 行的数组，通过标准批量绑定插入，
    -- 并捕获此会话在此操作前后的重做消耗。
 16    for i in 1 .. 1000000 loop
 17       l_rows.extend;
 18       l_rows(i) := i;
 19    end loop;
 20
 21    l_now := systimestamp;
 22    forall i in 1 .. l_rows.count
 23      insert into hardware ( descr )  values (l_rows(i));
 24    dbms_output.put_line('Elapsed = '||(systimestamp-l_now));
 25
 26    select value
 27    into   l_redo2
 28    from   v$mystat m, v$statname s
 29    where  s.statistic# = m.statistic#
 30    and    s.name = 'redo size';
 31

    -- 然后截断该表，并使用 APPEND_VALUES 进行直接路径加载来重新执行加载。
 32    execute immediate 'truncate table hardware';
 33
 34    dbms_output.put_line('Redo conventional = '||(l_redo2-l_redo1));
 35
 36    l_now := systimestamp;
 37    forall i in 1 .. l_rows.count
 38      insert /*+ APPEND_VALUES */ into hardware ( descr ) values (l_rows(i));
 39    dbms_output.put_line('Elapsed = '||(systimestamp-l_now));
 40
 41    select value
 42    into   l_redo3
 43    from   v$mystat m, v$statname s
 44    where  s.statistic# = m.statistic#
 45    and    s.name = 'redo size';
 46
 47    dbms_output.put_line('Redo direct load = '||(l_redo3-l_redo2));
 48
 49  end;
 50  /
Elapsed = +000000000 00:00:03.057000000
Redo conventional = 67599300
Elapsed = +000000000 00:00:02.668000000
Redo direct load = 146912

PL/SQL procedure successfully completed.
```

正如预期的那样，直接路径加载操作更快，并且使用了少得多的重做日志。因此，虽然始终注意集合的内存消耗很重要，但当涉及到使用批量绑定进行直接加载插入时，如果数据库服务器上的内存允许，你可能会发现自己使用了比正常情况更大的集合大小。

### 真正的益处：客户端批量处理

正如我在本章开头提到的，我在五金店里的反复徘徊正在损害我的效率。然而，有时情况要糟糕得多：我付了一件东西的钱，开车回家，*然后*才意识到我需要回到车上，开车返回五金店，再买点别的东西。（考虑到敏感的读者，我不会包含我妻子在这种情况下使用的“爱称”！）

本章描述了在 PL/SQL 中使用批量操作访问数据的效率，但这等同于*已经身处*五金店。从外部客户端应用程序到数据库（网络往返）以及在逐行处理数据的成本，等同于开车往返于商店。这种成本可以通过一些简单的演示来量化。首先，我将（当然是用 PL/SQL！）构建一个副本，模拟许多客户端应用程序在 3GL 代码中实现的内容，即一个打开表上游标的过程和一个获取单行的过程。以下是代码，你可以在 `bulk_network_1.sql` 中找到：

```
SQL> create or replace
  2  package PKG1 is
  3
  4   procedure open_cur(rc in out sys_refcursor);
  5
  6   procedure fetch_cur(rc in out sys_refcursor, p_row out hardware.item%type);
  7
  8  end;
  9  /

Package created.

SQL> create or replace
  2  package body PKG1 is
  3
  4   procedure open_cur(rc in out sys_refcursor) is
  5   begin
  6     open rc for select item from hardware;
  7   end;
  8
  9   procedure fetch_cur(rc in out sys_refcursor, p_row out hardware.item%type) is
 10   begin
 11     fetch rc into p_row;
 12   end;
 13
 14  end;
 15  /

Package body created.
```

SQL*Plus 就足以作为调用此包的客户端应用程序。SQL*Plus 本身就是一个真正的 SQL 客户端，它每次获取调用所获取的行数是一个你可以显式控制的参数，所以上面的演示可以写成

```
set arraysize n  (n=1 代表单行获取，n=1000 代表多行获取)
select item from HARDWARE;
```

但我想模拟 3GL 应用程序将要做的事情，即包含显式的打开游标、重复获取然后关闭游标的调用。为了打开游标，然后重复获取行直到结果集耗尽，我可以使用脚本 `bulk_single_fetch_100000.sql`，它执行以下操作：

```
variable rc refcursor
exec pkg1.open_cur(:rc)
variable n number
exec     pkg1.fetch_cur(:rc,:n);
exec     pkg1.fetch_cur(:rc,:n);
[重复 100000 次]
```

为了查看演示的已用时间，而无需滚动浏览 100,000 行输出，我关闭了终端输出并记录了前后时间戳。

```
SQL> variable rc refcursor
SQL> exec pkg1.open_cur(:rc)
SQL> select to_char(systimestamp,'HH24:MI:SS.FF') started from dual;

STARTED
------------------
12:11:18.779000

SQL> set termout off
SQL> @bulk_single_fetch_1000.sql   -- 包含 1000 次获取调用
[重复 100 次]

SQL> set termout on
SQL> select to_char(systimestamp,'HH24:MI:SS.FF') ended from dual;

ENDED
------------------
12:12:47.270000
```

你可以看到，100,000 次到数据库的往返大约花费了 90 秒。我在启用会话跟踪的情况下重复了该演示，并检查了生成的 tkprof 格式化文件。结果如下：

```
SELECT ITEM
FROM   HARDWARE
```


下面是排版后的内容：

```
调用     次数       CPU 耗时    总耗时       磁盘 I/O      查询次数    当前状态        行数
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.05          0          1          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch   100000      1.17       0.94          0     100003          0      100000
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total   100002      1.17       1.00          0     100004          0      100000
```

你可以看到，在这 90 秒中，只有一秒是真正在数据库中执行操作。另外 89 秒都花在了网络往返上。数据库什么也没做，但从客户端应用的角度看，它却是在等待数据库。用汽车术语来说，这就是*空转但毫无进展*。在我目前的工作中，每当我们在中间层服务器上遇到表现出这种行为的 3GL 程序时，我们称之为“`中间件失败`”。用这种方式给劣质代码贴上标签，竟能如此有效地让开发团队*聚焦*问题，这真令人惊叹！

现在，让我们应用刚刚学到的批量处理知识，从数据库中批量获取数据并批量传回客户端。下面的代码 (`bulk_network_2.sql`) 是新的实现方式，其性能将会好得多：

```sql
SQL> create or replace
  2  package PKG2 is
  3
  4   type t_num_list is table of hardware.item%type index by pls_integer;
  5
  6   procedure open_cur(rc in out sys_refcursor);
  7
  8   procedure fetch_cur(rc in out sys_refcursor, p_rows out t_num_list);
  9
 10  end;
 11  /
```

`Package created.`

```sql
SQL> create or replace
  2  package body PKG2 is
  3
  4   procedure open_cur(rc in out sys_refcursor) is
  5   begin
  6     open rc for select item from hardware;
  7   end;
  8
  9   procedure fetch_cur(rc in out sys_refcursor, p_rows out t_num_list) is
 10   begin
 11     fetch rc bulk collect into p_rows limit 1000;
 12   end;
 13
 14  end;
 15  /
```

`Package body created.`

重新运行演示。每次，将从数据库批量收集 1000 行数据并传回客户端。因为 SQL Plus 本身不理解返回的数组，数据将简单地追加到一个大的`VARCHAR2`变量 (`the_data`) 中，以模拟客户端接收数组数据的过程。这是一个示例 (`bulk_multi_fetch_in_bulk.sql`)：

```sql
SQL> variable rc refcursor
SQL> exec pkg2.open_cur(:rc)

PL/SQL procedure successfully completed.

SQL> select to_char(systimestamp,'HH24:MI:SS.FF') started from dual;

STARTED
------------------
12:39:09.704000

SQL> variable the_data varchar2(4000);
SQL> set termout off

SQL> declare
  2    n pkg2.t_num_list;
  3  begin
  4    :the_data := null;
  5    pkg2.fetch_cur(:rc,n);
  6    for i in 1 .. n.count loop
  7      :the_data := :the_data||n(i);
  8    end loop;
  9  end;
[重复执行 100 次]

SQL> select to_char(systimestamp,'HH24:MI:SS.FF') ended from dual;

ENDED
------------------
12:39:09.837000
```

差异是惊人的。直接周转时间从 90 秒下降到了 0.13 秒。启用跟踪重新运行演示，揭示了数据库访问次数的减少。

```sql
SELECT ITEM
FROM HARDWARE

调用     次数       CPU 耗时    总耗时       磁盘 I/O      查询次数    当前状态        行数
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          0          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch      100      0.04       0.04          0        256          0      100000
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total      102      0.04       0.04          0        256          0      100000
```

仍然获取了 100,000 行数据（最右列），但只进行了 100 次获取调用。我无法过分强调这里所实现的好处，即使只是一个简单的演示。Oracle 的客户花费大量资金进行优化工作，试图通过缓存、创建索引、SQL 调优等方法从系统中额外榨出 10-15`%`的性能，这种情况并不少见。有时，客户甚至会购买更多硬件，并伴随相应的许可证成本增加。但请考虑一下这个演示中所获得的性能提升：

> 从 90 秒降到 0.13 秒，提升了近 700 倍。

作为一名应用程序开发者，如果你对代码的修改能让其速度提升数百倍，那将使你的应用程序更加成功，并让你广受欢迎！在客户端应用与数据库的交互中减少网络往返，很大程度上取决于开发者在其客户端应用语言中可用的工具、应用的设计（以智能地利用向数据库传递数据的方式）以及开发者的勤奋程度。但是，作为一名`数据库`开发者，如果你能确保你的 PL/SQL 数据库接口允许客户端应用程序批量发送和接收数据，那么你就离成功的应用程序性能更近了一步。



### 概要

我喜欢撰写、演示或谈论 PL/SQL 中的批量操作，原因之一在于使用它们几乎就是成功的保证。许多 Oracle 特性，无论新旧，只适用于特定的客户群或解决特定的技术问题，此外，还需要大量细致的测试，以确保使用这些特性不会对数据库或其应用程序的其他部分产生负面影响。另一方面，在 PL/SQL 中采用批量集合和批量绑定，在绝大多数情况下都能使应用程序受益。从以行为中心的数据提取转向以集合为中心的批量数据提取，所需付出的努力很小，甚至可能为零。你的选择相当简单。

*   如果你已经在使用 `FOR cursor_variable in (QUERY or CURSOR)` 语法，那么要实现批量集合，只需确保你使用的是较新版本的 Oracle，并且 PL/SQL 编译设置保持默认即可。
*   如果你没那么幸运，也只需进行一些简单的重新编码，将 `FETCH CURSOR INTO` 风格改为 `FETCH CURSOR BULK COLLECT INTO`。只需几个关键词和一些 PL/SQL 类型定义，就完成了！

从以行为中心的数据修改转向以集合为中心的批量绑定数据修改，所需付出的努力同样很小。

*   如果你已经在循环中执行 DML（`insert`, `update`, `delete`），只需添加一些适当的类型定义，并将 FOR 循环重构为 `FORALL`，工作就完成了。

就是这么简单。应用程序的性能提升通常被描述为这里或那里几个百分点的改进，但在本章中你已经看到，转向批量操作可以带来`数量级`的性能提升。在过去的 Oracle 版本中，PL/SQL 的某些元素不支持批量操作（例如，本地动态 SQL 和动态 ref 游标）。然而，所有这些限制在近期的数据库版本中都已被解除，因此永远没有理由不考虑使用批量操作。

所以，努力很小，风险很低，回报却巨大。一旦你在 PL/SQL 中的方法变得以集合为中心，你会惊讶于这种新获得的集合中心思维会迅速传播到你 Oracle 技能的其他领域。你会发现，你更多地通过 SQL 而非过程逻辑来实现目标，同时你也会开始利用那些能批量处理数据的机制，如管道函数。通过批量操作学到的相同技能，将成为你整个数据库开发中秉持集合中心思维的动力。

我希望你能分享我对批量操作的热情，并且能在你自己的工作中看到回报。至于我，我要去五金店了——这次是推着购物车去的。

## 第 7 章：了解你的代码

**作者：Lewis Cunningham**

本章的名称是“了解你的代码”。你可能会想，如果你写了一段代码，你应该很了解它。你很可能确实了解。至少你比其他人了解得更多。然而，即使代码是你写的，你仍然会基于记忆做出假设。这些假设很可能随着时间的推移而与现实渐行渐远。你有一个最佳猜测——一个有根据的假设——但这是主观的，并依赖于人的不可靠性。而如果你没有编写这段代码，那就完全没谱了。

既然能知道，为何要猜测？我第一次听到这个问题是在性能优化的语境下。我记得是 Cary Milsap（作家兼性能大师）说的（至少是我第一次听到他说的）。我稍后会从性能角度更多地谈谈了解你的代码，但这个概念适用的范围远不止性能。

如果你要进行一笔大额采购，需要开支票，你会只是希望（并假设）你的账户里有足够的钱吗？如果采购金额与你认为账户里有的钱只差几百美元呢？你会先用 ATM 或网页来核实一下吗？我的银行甚至允许我发送短信来查询余额。既然能知道，我为何要猜测？

Oracle 就像是为你提供了访问代码的 ATM。关于你的 PL/SQL 代码，你可能需要知道的一切，都可以在 Oracle 数据字典中和/或通过使用 Oracle 提供的工具找到。你甚至不需要 PIN 码（尽管你需要登录数据库，并且可能还需要获取一些权限）。

本章将向你展示在哪里可以找到获取“账户更新”的最佳位置。你将发送一些 SQL 来代替短信。本章将从使用 Oracle 数据字典（与你的 PL/SQL 代码相关）和 `PL/Scope` 对代码进行静态分析开始。然后，它会转向基于时间和基于事件的性能剖析（11g 版本之前），最后介绍上下文敏感、基于时间的性能剖析（11g 及以上版本）。

到本章结束时，你将能够拿一段你从未见过的代码，了解其结构、使用的数据类型、哪些变量使用了它们以及它们在何处被使用。你将知道应用程序中的运行时消耗在何处（无论高效与否）。到本章结束时，你将拥有了解你的代码所需的工具。无需猜测；你将了解代码。


