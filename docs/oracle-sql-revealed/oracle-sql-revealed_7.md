# **递归子查询因式分解**

递归子查询因式分解没有预定义的伪列，因此计算逻辑需要手动实现。例如，当指定 `search depth first` 时，如果一个节点后面跟着下一层级的节点，则该节点不是叶子节点——这就是计算 `is_leaf` 列所使用的逻辑。

路径计算的机制则更有趣一些。路径表达式引用了前一次迭代计算出的路径。这是与 `connect by` 最重要的区别之一，后者只允许引用现有列（而非计算列）。因此，如果目标是计算从当前节点到根节点的路径，而不是从根节点到当前节点的路径，那么可以通过将 `r.path || '->' || t.id` 替换为 `'->' || t.id || r.path` 来实现。另一方面，使用 `connect by` 及其内置功能是无法做到这一点的。

##### 生成递归序列

Listing 6-4 展示了如何使用递归子查询因式分解来生成上一章中的递归序列。在第一种情况下，当前值仅依赖于前一个值，而在第二种情况下，它依赖于前一个值和再前一个值。递归子查询因式分解只允许引用前一次迭代的值，因此为了能够使用 `i-2` 次迭代的值，我们引入了一个辅助列。

```
with t(id, result) as
(
select 0 id, 1 result from dual
union all
select t.id + 1, round(100 * sin(t.result + t.id))
from t
where t.id < 20
)
select * from t;
with t (lvl, result, tmp) as
(
select 1, 0, 1 from dual
union all
select lvl + 1, tmp, tmp + result
from t
where lvl < 15)
select lvl, result from t;
Listing 6-4
生成递归序列
```

如果我们需要使用前几次迭代的值，可以添加多个辅助列，或者使用一个集合列。

总之，这种方法比 `connect by` 高效得多，因为无需遍历预先生成的值来生成每个新行。

##### 使用前几次迭代的值

如前所述，递归子查询因式分解允许引用前一次迭代计算出的值。此技术可用于使用二分法进行求根。此方法纯粹是为了突出 SQL 语言的能力，实际上没有使用 SQL 完成此任务的必要。

考虑函数 `y`，它在点 `1` 和 `2` 处的符号不同。

```
create or replace function y(x in number) return number as
begin return x*x - 2; end;
```

Listing 6-5 展示了如何在范围 `[1; 2]` 内以 `0.01` 的精度查找根。

```
with t(id, x, x0, x1) as
(
select 0, 0, 1, 2
from dual
union all
select t.id + 1,
(t.x0 + t.x1) / 2,
case
when sign(y(x0)) = sign(y((t.x0 + t.x1) / 2))
then (t.x0 + t.x1) / 2
else x0
end,
case
when sign(y(x1)) = sign(y((t.x0 + t.x1) / 2))
then (t.x0 + t.x1) / 2
else x1
end
from t
where abs((t.x0 + t.x1) / 2 - t.x) > 1e-2
)
select t.*, (x0+x1)/2 result from t;
ID          X         X0         X1     RESULT
---------- ---------- ---------- ---------- ----------
0          0          1          2        1.5
1        1.5          1        1.5       1.25
2       1.25       1.25        1.5      1.375
3      1.375      1.375        1.5     1.4375
4     1.4375      1.375     1.4375    1.40625
5    1.40625    1.40625     1.4375   1.421875
6   1.421875    1.40625   1.421875  1.4140625
Listing 6-5
使用二分法和递归子查询因式分解子句查找根
```

该算法根据以下规则在每一步划分范围：如果函数在中点的符号与右边界的符号相同，则将右边界移动到中点，否则将左边界移动到中点。重复迭代，直到当前步骤的中点与上一步的中点之差小于 `0.01`。

所需精度在第 6 次迭代时得到满足，找到的根是下一次迭代的中点，即 `1.4140625`。

每次迭代的边界都是使用前一次迭代的值计算的。使用 `connect by` 无法实现此方法。使用术语 “迭代” 而不是 “层级” 是为了强调算法的迭代性质。



##### 遍历层级结构

文档说明 `«subquery_factoring_clause»` 支持递归子查询因子化（递归 `WITH`），并允许查询层级数据。此功能比 `CONNECT BY` 更强大，因为它提供了深度优先搜索和广度优先搜索，并支持多个递归分支。听起来 `CONNECT BY` 总是执行深度优先搜索，而递归子查询因子化的遍历算法可以通过在 `search_clause` 中指定 `depth-first` 或 `breadth-first` 来影响。让我们检查一下这种印象是否正确。

函数 `stop_at` 在访问特定节点时设置一个标志，如果标志被设置则返回一个非空值。

```
create or replace function stop_at(p_id in number, p_stop in number)
return number is
begin
if p_id = p_stop then
dbms_application_info.set_client_info('1');
return 1;
end if;
for i in (select client_info from v$session where sid = userenv('sid')) loop
return i.client_info;
end loop;
end;
```

```
exec dbms_application_info.set_client_info('');
PL/SQL 过程已成功完成。
with rec(lvl, id) as
(
select 1, id
from t_two_branches where id_parent is null
union all
select r.lvl + 1, t.id
from t_two_branches t
join rec r on t.id_parent = r.id
where stop_at(t.id, 101) is null
)
search breadth first by id set ord
--search depth first by id set ord
select *
from rec;
LVL         ID        ORD
---------- ---------- ----------
1          0          1
1        100          2
2          1          3
exec dbms_application_info.set_client_info('');
PL/SQL 过程已成功完成。
with rec(lvl, id) as
(
select 1, id
from t_two_branches where id_parent is null
union all
select r.lvl + 1, t.id
from t_two_branches t
join rec r on t.id_parent = r.id
where stop_at(t.id, 101) is null
)
--search breadth first by id set ord
search depth first by id set ord
select *
from rec;
LVL         ID        ORD
---------- ---------- ----------
1          0          1
2          1          2
1        100          3
代码清单 6-6
指定广度优先和深度优先搜索
```

Oracle 11.2 和 12.1 在两种情况下都只返回 3 行；然而，预期查询会为深度优先搜索返回第一个分支的所有节点，因为它们都不等于 101。因此看起来，无论我们在 `search_clause` 中指定何种方法，Oracle 总是执行广度优先搜索，然后相应地对结果排序。另一方面，Oracle 12.2 返回以下结果。

```
LVL         ID        ORD
---------- ---------- ----------
1          0          1
1        100          2
2          1          3
3          2          4
4          3          5
5          4          6
6          5          7
7          6          8
8          7          9
9          8         10
10          9         11
11         10         12
已选择 12 行。
LVL         ID        ORD
---------- ---------- ----------
1          0          1
2          1          2
3          2          3
4          3          4
5          4          5
6          5          6
7          6          7
8          7          8
9          8          9
10          9         10
11         10         11
1        100         12
已选择 12 行。
```

这意味着它执行的是深度优先搜索，无论 `search_clause` 中指定了什么——在两种情况下都返回了第一个分支的所有节点。

现在让我们检查 `CONNECT BY` 的结果。

```
select rownum rn, level lvl, id, id_parent
from t_two_branches
start with id_parent is null
connect by prior id = id_parent
and stop_at(id, 101) is null;
RN        LVL         ID  ID_PARENT
---------- ---------- ---------- ----------
1          1          0
2          2          1          0
3          3          2          1
4          4          3          2
5          5          4          3
6          6          5          4
7          7          6          5
8          8          7          6
9          9          8          7
10         10          9          8
11         11         10          9
12          1        100
已选择 12 行。
```

这与 12.2 版本上递归子查询因子化的深度优先搜索结果相同。

总而言之，`CONNECT BY` 总是使用深度优先搜索遍历层级结构，而递归子查询因子化的行为在 12.2 版本中从广度优先搜索变为了深度优先搜索。`search_clause` 只影响最终顺序，不影响 Oracle 用来遍历层级结构的算法。对于 `CONNECT BY`，通过按 `LEVEL` 对结果排序可以轻松模拟广度优先搜索。

##### 再谈循环

让我们研究如何使用递归子查询因子化和上一章的图（图 5-1）来处理循环。

```
with t(id, id_parent) as
(
select * from graph where id_parent = 1
union all
select g.id, g.id_parent
from t
join graph g on t.id = g.id_parent
)
search depth first by id set ord
cycle id set cycle to 1 default 0
select * from t;
ID  ID_PARENT        ORD CYCLE
---------- ---------- ---------- ----------
2          1          1 0
3          1          2 0
4          3          3 0
5          4          4 0
3          5          5 1
代码清单 6-7
通过 ID 检测循环
```

`"cycle id set cycle to 1 default 0"` 指示 Oracle，如果检测到按 `id` 的循环，则将 `"cycle"` 列设置为 1。Oracle 不会为有问题的行查找子行，但会继续查找其他非循环行。如果一行的某个祖先行在循环列中具有相同的值，则该行被视为形成循环。换句话说，如果一行被标记为循环，则结果集中已存在的一行在指定列中具有相同的值。

在上面的例子中，`ID = 3` 的行被标记为循环，因为 `ID = 3` 已经存在于结果中。在 `"CONNECT BY"` 子句的情况下，`ID = 5` 的行被标记为循环，因为它的子行（`ID = 3` 的行）也是它的祖先——参见上一章图 5-1。

与 `CONNECT BY` 不同，我们可以指定使用哪一列来检测循环。因此，如果我们在 `cycle_clause` 中指定 `id_parent`，结果会略有不同——当我们第二次遇到 `ID_PARENT = 3` 的节点时，执行停止。

```
with t(id, id_parent) as
(
select * from graph where id_parent = 1
union all
select g.id, g.id_parent
from t
join graph g on t.id = g.id_parent
)
search depth first by id set ord
cycle id_parent set cycle to 1 default 0
select * from t;
ID  ID_PARENT        ORD C
---------- ---------- ---------- -
2          1          1 0
3          1          2 0
4          3          3 0
5          4          4 0
3          5          5 0
4          3          6 1
代码清单 6-8
通过 ID_PARENT 检测循环
```

我们可以注意到，按 `ID` 的循环是在与 `CONNECT BY` 查询和条件 `"nocycle prior id = id_parent and prior id_parent is not null"`（参见代码清单 5-8）相同的行中识别的。然而，情况并非总是如此。如果根节点是循环的一部分，结果可能不同。让我们看看当我们从 `ID = 3` 的节点开始时的结果。



##### 限制条件

递归成员在查询中受到一系列限制；特别是，不能在其中使用 `DISTINCT`、`GROUP BY`、`HAVING`、`CONNECT BY`、聚合函数、`MODEL` 子句等。有人可能会问，这是当前实现的限制，还是使用如此复杂逻辑时递归执行本身就无意义。我倾向于认为其中一些限制在未来会被解除。另一方面，对于某些限制，即使在当前版本中，我们也可以使用变通方法，但这可能看起来有些笨拙。特别是，我们可以使用分析函数和额外的过滤器来获得聚合结果——尽管这不是在实际任务中应该使用的方式。

以下演示主要是出于学术目的。所以，假设我们需要构建一个父子层次结构，其中父节点是当前层级上所有 ID 的总和。

```
with t0(id, id_parent, letter) as
(select 1, 0, 'B' from dual
union all select 2, 1, 'D' from dual
union all select 3, 1, 'A' from dual
union all select 10, 5, 'C' from dual
union all select 66, 6, 'X' from dual),
t(id, id_parent, sum_id, lvl, str, rn) as
(select id, id_parent, id, 1, letter, 1 from t0 where id_parent = 0
union all
select
t0.id,
t0.id_parent,
sum(t0.id) over (),
t.lvl + 1,
listagg(letter, ', ') within group (order by letter) over (),
rownum
from t
join t0 on t.sum_id = t0.id_parent and t.rn = 1)
select * from t;
ID  ID_PARENT     SUM_ID      LVL STR              RN
------- ---------- ---------- -------- -------- --------
1          0          1        1 B                 1
3          1          5        2 A, D              2
2          1          5        2 A, D              1
10          5         10        3 C                 1
```

这里使用了分析函数来替代聚合函数，并在连接条件中添加了“`t.rn = 1`”以避免重复，因为分析函数的值对所有行都是相同的，它们没有被分组到每个组一行。

如果我们只对每一层级的聚合结果感兴趣，那么可以使用如下查询实现：

```
select sum_id, lvl, str from t where rn = 1;
```

在递归成员中使用分析函数在 Oracle 11.2.0.1 中会导致«`ORA-32486: unsupported operation in recursive branch of recursive WITH clause`»错误；然而，此问题已在 11.2.0.3 中修复。

##### 总结

“`RECURSIVE WITH`”在标准 SQL:1999 中定义，而“`CONNECT BY`”子句是 Oracle 特有的功能。尽管如此，在所有可能的情况下，使用 `CONNECT BY` 而非递归子查询因式分解是有意义的，因为它经过更好优化且运行更快。如前所述，在 `CONNECT BY` 查询中无法引用计算出的表达式，而 `RECURSIVE WITH` 提供了这一功能。因此，如果在遍历层次结构时需要进行任何类似累积的计算，那么递归子查询因式分解可能是最佳选择。

遍历层次结构并不是递归子查询因式分解的唯一应用。当需要对记录集多次应用相同逻辑时，它可以用于各种任务。需要强调的是，最终结果包含来自所有迭代的记录集，而在当前迭代中只能访问前一次迭代的行。返回所有迭代记录集的要求可能会导致工作区使用量激增和其他开销。因此，如果你不需要所有迭代的记录集而只需要最后一次迭代的，那么使用 PL/SQL 循环和集合或临时表是更合理的。

即使递归子查询因式分解和 `CONNECT BY` 具有处理循环的内置功能，在琐碎的情况下使用它们才有意义。对于更复杂的情况，过程化方法更好。无论如何，`CONNECT BY` 和递归子查询因式分解处理循环的方式略有不同，因此了解细节可能很重要。



