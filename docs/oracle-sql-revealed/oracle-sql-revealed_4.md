# 4. 聚合函数

聚合函数对 `group by` 子句中定义的每个组返回一行。列名和表达式都可用于定义组，一个组是指 `group by` 中指定的所有表达式值都相同的行集合。每行属于且仅属于一个组。如果未指定 `group by`，则整个记录集被视为一个单一组，这种情况下，即使要分组的记录集为空，查询也始终返回一行。

代码清单 `4-1` 展示了如何基于第 `1` 章代码清单 `1-58` 中引入的表，计算所有作者的报告总次数和工作日数。

```sql
select p.name,
count(*) cnt_all,
count(distinct p.day) cnt_day,
listagg(p.day || ' ' || p.time || ':00', '; ') within group(order by w.id) details
from presentation p, week w
where p.day = w.day
group by p.name;
NAME    CNT_ALL    CNT_DAY DETAILS
---- ---------- ---------- ------------------------------------
John          3          2 monday 14:00; monday 9:00; friday 9:00
Rex            2             2 wednesday 11:00; friday 11:00
Listing 4-1
聚合函数。简单示例
```

当指定 `distinct` 关键字时，只有不同的值才会被传递给聚合函数。`listagg` 函数用于显示所有报告的详细信息，正如前一章指出的——`listagg` 不是可交换的，并且必须在 `within group` 关键字后指定顺序。还有一些其他聚合函数，其结果依赖于组内的顺序，例如 `percentile_cont`（更复杂的例子请参见第二部分中的测验“带偏移的百分位数”）。

大多数聚合函数返回原子类型的结果，其类型与参数类型相同——例如 `number`、`date`、`varchar2` 等。然而，有些函数只是将值组合在一起，而不是基于输入计算结果值——例如 `collect` 和 `xmlagg`。

用户定义函数（UDF）可以应用于 `collect` 函数之上，以处理每组的元素。下面的 `collagg` 函数可用于连接集合元素。

```sql
create or replace function collagg(p in strings) return varchar is
result varchar2(4000);
begin
for i in 1 .. p.count loop
result := result || ', ' || p(i);
end loop;
return(substr(result, 3));
end collagg;
/
Listing 4-2
连接集合元素
```

其中 `strings` 是定义为以下内容的集合：

```sql
create or replace type strings as table of varchar2(4000)
/
```

代码清单 `4-3` 展示了如何按报告者获取所有报告日的列表以及不同日期的列表。

```sql
select name,
collagg(cast(collect(p.day order by w.id desc) as strings)) days,
collagg(set(cast(collect(p.day order by w.id desc) as strings))) days_unique
from presentation p, week w
where p.day = w.day
group by p.name;
NAME DAYS                           DAYS_UNIQUE
---- ------------------------------ ---------------------------
John friday, monday, monday         friday, monday
Rex  friday, wednesday              friday, wednesday
Listing 4-3
使用 collect 函数
```

与 `listagg` 不同，`order by` 是在函数本身中指定的，`distinct` 关键字不被允许在 `collect` 中使用，但可以使用 `set` 函数来消除重复项。尽管 `set` 函数似乎能保留元素的顺序——但并不能保证在所有情况下都是如此。

类似的逻辑，包括排序，可以使用 `xmlagg` 来实现，如下列表达式所示，但在这种情况下，如果没有额外的内联视图，则无法消除重复项。

```sql
substr(xmlagg(xmlelement("x", ', ' || p.day) order by w.id desc)
.extract('//x/text()')
.getstringval(), 3) x
```

除了内置的聚合函数，Oracle（自版本 9i Release 1 起）还提供了用户定义聚合（UDAG）的接口，可用于实现任何复杂的分组逻辑。例如，有一个单行函数 `bitand`，它没有对应的聚合函数。如果需要，可以使用 UDF + `collect` 或 UDAG 来实现此逻辑。

在前一章中，展示了分析函数如何帮助避免连接，聚合函数也可用于此目的。

例如，一个 `requirements` 表包含职位和相应技能的信息。目标是找出需要 `Oracle` 知识但不需要 `Linux` 的职位。

```sql
create table requirements(position, skill) as
(
select 'Data Scientist', 'R' from dual
union all select 'Data Scientist', 'Python' from dual
union all select 'Data Scientist', 'Spark' from dual
union all select 'DB Developer', 'Oracle' from dual
union all select 'DB Developer', 'Linux' from dual
union all select 'BI Developer', 'Oracle' from dual
union all select 'BI Developer', 'MSSQL' from dual
union all select 'BI Developer', 'Analysis Services' from dual
union all select 'System Administrator', 'Linux' from dual
union all select 'System Administrator', 'Network Protocols' from dual
union all select 'System Administrator', 'Python' from dual
union all select 'System Administrator', 'Perl' from dual
);
```

一个直接的解决方案如下所示。

```sql
select position
from requirements r
where skill = 'Oracle'
and not exists (select null
from requirements r0
where r0.position = r.position
and r0.skill = 'Linux');
POSITION

BI Developer
```

此解决方案的主要缺点是 `where` 子句中的关联子查询，这会导致对 `requirements` 表的额外扫描。

另一种方法是计算 `Oracle` 和 `Linux` 技能的计数，并过滤掉不满足条件的职位。

```sql
select position
from requirements
group by position
having count(decode(skill, 'Oracle', 1)) = 1
and count(decode(skill, 'Linux', 1)) = 0;
POSITION

BI Developer
```

从性能角度来看，这个解决方案更可取，你可能会注意到聚合函数仅用于过滤目的，而不在 `select` 列表中。

让我们考虑一个更通用的例子。表 `entity` 和 `property` 用于实现实体-属性-值（EAV）模型——这种方法在数据库设计中用于在单个表中存储具有可变属性数量的实体。

```sql
with entity(id, name) as
(select 1, 'E1' from dual
union all select 2, 'E2' from dual
union all select 3, 'E3' from dual),
property(id, entity_id, name, value) as
(select 1, 1, 'P1', 1 from dual
union all select 2, 1, 'P2', 10 from dual
union all select 3, 1, 'P3', 20 from dual
union all select 4, 1, 'P4', 50 from dual
union all select 5, 2, 'P1', 1 from dual
union all select 6, 2, 'P3', 100 from dual
union all select 7, 2, 'P4', 50 from dual
union all select 8, 3, 'P1', 1 from dual
union all select 19, 3, 'P2', 10 from dual
union all select 10, 3, 'P3', 100 from dual)
Listing 4-4
EAV 模型
```

我们的目标是选择属性 `P1`、`P2`、`P3` 的值分别等于 1、10、100 的实体，并且假设每个实体的属性是唯一的。有时开发人员使用多个连接来实现这一点，这是非常低效的——实际上，连接的数量等于我们感兴趣的属性数量。

```sql
select e.name
from entity e
join property p1 on p1.entity_id = e.id and p1.name = 'P1'
join property p2 on p2.entity_id = e.id and p2.name = 'P2'
join property p3 on p3.entity_id = e.id and p3.name = 'P3'
where p1.value = 1 and p2.value = 10 and p3.value = 100;
NAME

E3
```

如果我们需要获取 20 个属性的值，这将导致 20 个连接，这根本不是一个可行的解决方案。

考虑到每个属性可能有一个值，也可能根本没有值，我们可以将每个实体的所有属性扁平化为一行，并在之上应用筛选条件。

##### 透视和逆透视操作符

`代码清单 4-5` 展示了使用 `group by` 实现的扁平化逻辑；但从 Oracle 11g 开始，可以使用 `pivot` 操作符实现相同的效果。

```sql
create table entity_flattened as
select *
from (select e.name name, p.name p_name, value
from entity e
join property p
on p.entity_id = e.id)
pivot(max(value) for p_name in('P1' p1_value, 'P2' p2_value, 'P3' p3_value));
```

`代码清单 4-6`
使用 `pivot` 操作符扁平化 EAV 模型

表 `entity_flattened` 包含一个与使用 `group by` 的内联视图中相同的记录集。关于 `pivot` 操作符最重要的一点是，查询中必须列出所有列，因为 Oracle 在解析阶段定义了结果集的所有列。也就是说，无法基于表中的数据或其他条件在记录集中动态创建列，因此如果你有此类需求，可以使用 `ODCItable` 接口（或从 Oracle 18c 开始的多态表函数）。`Pivot XML` 允许你为动态数量的列生成 XML，但如果希望以关系形式获得结果，你需要列出所有列以进行 XML 解析。此技术在 `代码清单 4-7` 中演示。

```sql
select name, x.*
from (select *
from (select e.name name, p.name p_name, value
from entity e
join property p
on p.entity_id = e.id)
pivot xml(max(value) value for p_name in(any))),
xmltable('/PivotSet' passing p_name_xml
columns
name1 varchar2(30)
path '/PivotSet/item[1]/column[@name="P_NAME"]/text()',
value1 varchar2(30)
path '/PivotSet/item[1]/column[@name="VALUE"]/text()',
name2 varchar2(30)
path '/PivotSet/item[2]/column[@name="P_NAME"]/text()',
value2 varchar2(30)
path '/PivotSet/item[2]/column[@name="VALUE"]/text()',
name3 varchar2(30)
path '/PivotSet/item[3]/column[@name="P_NAME"]/text()',
value3 varchar2(30)
path '/PivotSet/item[3]/column[@name="VALUE"]/text()') x;
NAME  NAME1 VALUE1 NAME2 VALUE2 NAME3 VALUE3
----- ----- ------ ----- ------ ----- ------
E1    P1    1      P2     10     P3     20
E2    P1    1      P3    100     P4     50
E3    P1    1      P2     10     P3    100
```

`代码清单 4-7`
解析 `pivot` XML

鉴于此逻辑与呈现结果相关，有时在客户端实现它是有意义的。

可以使用 `unpivot` 操作符执行反向操作，如 `代码清单 4-8` 所示。

```sql
select *
from entity_flattened
unpivot (value for p_name in
(p1_value as 'P1', p2_value as 'P2', p3_value as 'P3'));
NAME  P_NAME      VALUE
----- ------ ----------
E1    P1              1
E1    P2             10
E1    P3             20
E3    P1              1
E3    P2             10
E3    P3            100
E2    P1              1
E2    P3            100
8 rows selected.
```

`代码清单 4-8`
`Unpivot` 操作符

它为 `unpivot` 子句中列出的每个列创建新行，并复制所有剩余列的值。在上面的例子中，剩余列是 “`p1_value, p2_value, p3_value`” 和对应的 `name`。`unpivot` 不需要 “`any`” 关键字，因为要进行逆透视的记录集总是包含固定且预定义数量的列。Oracle 本可以引入像 “`unpivot (value for p_name in (any except name))`” 这样的语法糖，但这并非强烈必需。

`Unpivot` 可以使用笛卡尔连接实现。

```sql
select name,
p_name,
decode(p_name, 'P1', p1_value, 'P2', p2_value, 'P3', p3_value) value
from entity_flattened,
(select 'P1' p_name from dual
union all select 'P2' from dual
union all select 'P3' from dual)
where decode(p_name, 'P1', p1_value, 'P2', p2_value, 'P3', p3_value) is not null
order by 1, 2;
```

## Cube, Rollup, 分组集

Oracle 提供了计算总计和小计的附加功能。让我们考虑一个包含订单信息的表。

```sql
create table orders(order_id, client_id, product_id, quantity) as
(
select 1, 1, 1, 1 from dual
union all select 1, 1, 2, 2 from dual
union all select 1, 1, 3, 1 from dual
union all select 2, 2, 1, 1 from dual
union all select 2, 2, 5, 1 from dual
union all select 3, 1, 1, 1 from dual
union all select 3, 1, 4, 1 from dual
union all select 3, 1, 4, 1 from dual
union all select 4, 2, 4, 1 from dual
union all select 4, 2, 5, 1 from dual
);
```

“`Rollup`” 允许我们从右向左计算小计，“`cube`” 允许我们计算所列列的所有可能的小计。

```sql
select client_id, product_id, sum(quantity) cnt
from orders
group by rollup(client_id, product_id)
order by client_id, product_id;
CLIENT_ID PRODUCT_ID        CNT
---------- ---------- ----------
1          1          2
1          2          2
1          3          1
1          4          2
1                     7
2          1          1
2          4          1
2          5          2
2                     4

10 rows selected.
```

```sql
select client_id, product_id, sum(quantity) cnt
from orders
group by cube(client_id, product_id)
order by client_id, product_id;
CLIENT_ID PRODUCT_ID        CNT
---------- ---------- ----------
1          1          2
1          2          2
1          3          1
1          4          2
1                     7
2          1          1
2          4          1
2          5          2
2                     4
1          3
2          2
3          1
4          3
5          2

15 rows selected.
```

同样的效果可以分别使用以下语句完成。

```sql
grouping sets ((), (client_id), (client_id, product_id))
和
grouping sets ((), (client_id), (product_id), (client_id, product_id))
```

函数 `grouping` 和 `grouping_id` 可用于识别小计。`Grouping` 只接受单个表达式作为参数，而 `grouping_id` 可以接受多个表达式。

```sql
select decode(grouping(client_id), 1, 'all clients', client_id) as client_id,
decode(grouping(product_id), 1, 'all products', product_id) as product_id,
sum(quantity) cnt,
decode(grouping_id(client_id, product_id),
bitand(grouping_id(client_id, product_id), bin_to_num(0, 0)),
'client, product',
bitand(grouping_id(client_id, product_id), bin_to_num(0, 1)),
'client',
bitand(grouping_id(client_id, product_id), bin_to_num(1, 1)),
'grand total') slice
from orders
group by rollup(client_id, product_id)
order by client_id, product_id;
CLIENT_ID       PRODUCT_ID             CNT SLICE
--------------- --------------- ---------- --------------------
1               1                        2 client, product
1               2                        2 client, product
1               3                        1 client, product
1               4                        2 client, product
1               all products             7 client
2               1                        1 client, product
2               4                        1 client, product
2               5                        2 client, product
2               all products             4 client
all clients     all products            11 grand total
10 rows selected.
```

还有一个函数 `group_id` 可用于区分相同的切片。

