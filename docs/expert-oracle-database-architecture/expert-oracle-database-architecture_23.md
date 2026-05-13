# 删除父表记录导致的阻塞与索引外键解决方案

但是，如果我们进入另一个会话并尝试删除第一个父记录，我们会发现该会话立即被阻塞：

```
SQL> delete from p where x = 1;
```

它在执行删除操作之前，试图获取表 `C` 上的完全表锁。此时，任何其他会话都无法对 `C` 表发起新的 `DELETE`、`INSERT` 或 `UPDATE` 操作（已经启动的会话可以继续，但不能开始修改 `C` 的新会话）。

这种阻塞问题在更新主键值时也会发生。由于在关系数据库中更新主键是一个大忌，这通常不是更新操作的问题。然而，我曾见过这种主键更新成为一个严重问题，特别是当开发人员使用自动生成 SQL 的工具时，这些工具会更新每一列，无论最终用户是否实际修改了该列。例如，假设我们使用 `Oracle Forms` 并在任何表上创建默认布局。默认情况下，`Oracle Forms` 会生成一个修改我们选择显示的表中的每一列的更新语句。如果我们在 `DEPT` 表上构建一个默认布局并包含所有三个字段，那么无论我们修改 `DEPT` 表的哪一列，`Oracle Forms` 都会执行以下命令：

```
update dept set deptno=:1,dname=:2,loc=:3 where rowid=:4
```

在这种情况下，如果 `EMP` 表有一个指向 `DEPT` 的外键，并且 `EMP` 表中的 `DEPTNO` 列上没有索引，那么对 `DEPT` 表的一次更新将导致整个 `EMP` 表被锁定。如果您使用任何自动生成 SQL 的工具，这一点需要特别注意。即使主键的值没有改变，子表 `EMP` 也会在上述 SQL 语句执行后被锁定。对于 `Oracle Forms`，解决方案是将该表的 `UPDATE CHANGED COLUMNS ONLY` 属性设置为 `YES`。`Oracle Forms` 将生成一个仅包含已更改列（不包括主键）的 `UPDATE` 语句。

由删除父表行引起的问题要常见得多。正如我所演示的，如果我在表 `P` 中删除一行，那么在 DML 操作期间，子表 `C` 将被锁定，从而在事务持续期间阻止其他针对 `C` 的更新操作（当然，假设没有其他人正在修改 `C`，否则删除操作会等待）。这就是阻塞和死锁问题的根源。通过锁定整个表 `C`，我严重降低了数据库的并发性，以至于没有人能够修改 `C` 中的任何内容。此外，我增加了死锁的可能性，因为我现在持有大量数据直到提交。其他会话在 `C` 上被阻塞的可能性现在要高得多；任何试图修改 `C` 的会话都会被阻塞。因此，我将开始看到许多持有其他资源上已有锁的会话在数据库中被阻塞。如果其中任何被阻塞的会话实际上正在锁定我的会话也需要的资源，我们就会发生死锁。这种情况下，死锁是由我的会话阻止访问远多于其实际需要的资源（在这里，是单个表中的所有行）造成的。当有人抱怨数据库中出现死锁时，我会让他们运行一个查找未索引外键的脚本；99%的时间我们都能定位到问题表。通过简单地为该外键创建索引，死锁以及许多其他争用问题就会消失。以下示例演示了使用此脚本来定位表 `C` 中的未索引外键：

```
SQL> column columns format a30 word_wrapped
SQL> column table_name format a15 word_wrapped
SQL> column constraint_name format a15 word_wrapped
SQL> select table_name, constraint_name,
cname1 || nvl2(cname2,','||cname2,null) ||
nvl2(cname3,','||cname3,null) || nvl2(cname4,','||cname4,null) ||
nvl2(cname5,','||cname5,null) || nvl2(cname6,','||cname6,null) ||
nvl2(cname7,','||cname7,null) || nvl2(cname8,','||cname8,null)
columns
from ( select b.table_name,
b.constraint_name,
max(decode( position, 1, column_name, null )) cname1,
max(decode( position, 2, column_name, null )) cname2,
max(decode( position, 3, column_name, null )) cname3,
max(decode( position, 4, column_name, null )) cname4,
max(decode( position, 5, column_name, null )) cname5,
max(decode( position, 6, column_name, null )) cname6,
max(decode( position, 7, column_name, null )) cname7,
max(decode( position, 8, column_name, null )) cname8,
count(*) col_cnt
from (select substr(table_name,1,30) table_name,
substr(constraint_name,1,30) constraint_name,
substr(column_name,1,30) column_name,
position
from user_cons_columns ) a,
user_constraints b
where a.constraint_name = b.constraint_name
and b.constraint_type = 'R'
group by b.table_name, b.constraint_name
) cons
where col_cnt > ALL
( select count(*)
from user_ind_columns i,
user_indexes     ui
where i.table_name = cons.table_name
and i.column_name in (cname1, cname2, cname3, cname4,
cname5, cname6, cname7, cname8 )
and i.column_position <= cons.col_cnt
and ui.table_name = i.table_name
and ui.index_name = i.index_name
and ui.index_type IN ('NORMAL','NORMAL/REV')
group by i.index_name
);
TABLE_NAME      CONSTRAINT_NAME COLUMNS
--------------- --------------- ------------------------------
C               SYS_C0061427    X
```

这个脚本适用于最多包含八列的外键约束（如果您有更多列，您可能需要重新考虑您的设计）。它首先在之前的查询中构建了一个名为 `CONS` 的内联视图。这个内联视图将约束中适当的列名从行转置为列，结果是每个约束一行，最多有八列包含约束中的列名。此外，还有一个列 `COL_CNT`，它包含外键约束本身的列数。对于从内联视图返回的每一行，我们执行一个关联子查询，该子查询检查当前正在处理的表上的所有索引。它计算该索引中与外键约束列匹配的列数，然后按索引名称进行分组。因此，它生成一组数字，每个数字是该表上某个索引中匹配列的计数。如果原始的 `COL_CNT` 大于所有这些数字，则表示该表上没有索引支持该约束。如果 `COL_CNT` 小于所有这些数字，则表示至少有一个索引支持该约束。注意 `NVL2` 函数的使用，我们用它来将列名列表“粘合”成一个逗号分隔的列表。该函数接受三个参数：`A`，`B`，`C`。如果参数 `A` 不为空，则返回参数 `B`；否则返回参数 `C`。这个查询假设约束的所有者也是表和索引的所有者。如果另一个用户索引了该表或者表位于另一个模式中（这两种情况都很少见），它将无法正确工作。

先前的脚本还会检查索引类型是否为 B*Tree 索引（`NORMAL` 或 `NORMAL/REV`）。我们检查它是否是 B*Tree 索引，因为外键列上的位图索引不能防止锁定问题。

Note



在数据仓库环境中，通常在事实表的外键列上创建位图索引。然而，数据仓库环境中的数据加载通常通过计划的 ETL 过程有序进行，因此不会遇到像 OLTP 应用中可能遇到的那样：一个进程向子表插入数据的同时，另一个进程从父表删除数据。

因此，之前的脚本显示表`C`在列`X`上有一个外键但没有索引。通过在`X`上创建一个 B*Tree 索引，我们可以完全消除这个锁定问题。除了这个表锁之外，未索引的外键在以下情况下也可能出现问题：

*   当你有`ON DELETE CASCADE`约束但没有在子表上创建索引时。例如，`EMP`是`DEPT`的子表。`DELETE DEPTNO = 10`应级联到`EMP`。如果`EMP`中的`DEPTNO`未被索引，你将对`EMP`进行全表扫描，以对应从`DEPT`表中删除的每一行。这种全表扫描通常是不可取的，并且如果你从父表中删除多行，子表将为每个被删除的父行扫描一次。

*   当你从父表查询到子表时。再次考虑`EMP/DEPT`示例。在`DEPTNO`的上下文中查询`EMP`表非常常见。如果你经常运行以下查询（例如，生成报告），你会发现缺少索引会减慢查询速度：

```sql
select * from dept, emp
where emp.deptno = dept.deptno and dept.deptno = :X;
```

何时不需要为外键创建索引？一般来说，当满足以下条件时：

*   你不从父表删除数据。
*   你不更新父表的唯一/主键值（注意工具对主键的意外更新）。
*   你不从父表连接到子表（如从`DEPT`到`EMP`）。

如果你满足所有三个条件，可以放心地跳过索引；它不是必需的。如果你满足上述任何一个条件，请注意其后果。这是 Oracle 倾向于过度锁定数据的罕见情况之一。

### 锁升级

当发生锁升级时，系统会降低锁的粒度。一个例子是数据库系统将针对一个表的 100 个行级锁转换为一个表级锁。你现在使用一个锁来锁定所有内容，而且通常，你锁定的数据量也比以前多得多。锁升级在认为锁是稀缺资源且应避免开销的数据库中经常使用。

注意

Oracle 永远不会升级锁。永远不会。

Oracle 永远不会升级锁，但它确实执行锁转换或锁提升，这些术语经常与锁升级混淆。

注意

术语*锁转换*和*锁提升*是同义词。Oracle 通常将该过程称为*锁转换*。

Oracle 会在尽可能低的级别（即限制性最小的锁）上获取锁，并在必要时将该锁转换为限制性更强的级别。例如，如果你使用`FOR UPDATE`子句从表中选择一行，将创建两个锁。一个锁放置在你选择的行上（这将是一个排他锁；其他人不能以排他模式锁定该特定行）。另一个锁，一个`ROW SHARE TABLE`锁，放置在表本身上。这将阻止其他会话在表上放置排他锁，从而防止他们更改表的结构。另一个会话可以修改此表中的任何其他行而不会冲突。在表中有一行被锁定的情况下，尽可能多地允许可以成功执行的命令。

锁升级不是数据库的“功能”。它不是一个理想的属性。数据库支持锁升级这一事实意味着其锁定机制存在一些固有的开销，并且需要执行大量工作来管理数百个锁。在 Oracle 中，持有一个锁或一百万个锁的开销是相同的：没有。

## 锁类型

Oracle 中的三大类锁如下：

*   **DML 锁**：DML 代表*数据操作语言*。通常，这指的是`SELECT`、`INSERT`、`UPDATE`、`MERGE`和`DELETE`语句。DML 锁是允许多个数据修改并发进行的机制。例如，DML 锁可以是特定数据行上的锁，也可以是表级别的锁，锁定表中的每一行。
*   **DDL 锁**：DDL 代表*数据定义语言*（`CREATE`和`ALTER`语句等）。DDL 锁保护对象结构的定义。
*   **内部锁和门**：Oracle 使用这些锁来保护其内部数据结构。例如，当 Oracle 解析查询并生成优化的查询计划时，它会门住库缓存以将该计划放入其中供其他会话使用。门是 Oracle 使用的一种轻量级、低级别的序列化设备，功能类似于锁。不要被*轻量级*这个术语所混淆或误导；正如你将看到的，门是数据库争用的常见原因。它们在实现上是轻量级的，但在效果上并非如此。

我们现在将更详细地了解这些通用类别中的具体锁类型及其使用的影响。锁的种类比我能在这里涵盖的要多。我在后续章节中涵盖的是最常见的并且持有时间较长的锁。其他类型的锁通常持有时间非常短。

### DML 锁

DML 锁用于确保一次只有一个人修改一行，并且没有人可以删除你正在使用的表。Oracle 会或多或少地为你透明地放置这些锁。



