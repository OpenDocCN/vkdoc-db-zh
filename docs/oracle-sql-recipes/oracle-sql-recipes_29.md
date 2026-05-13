# 第四章 创建与派生数据

关于虚拟列存在一些限制，其中一部分显而易见，另一部分则不那么明显。如果你的虚拟列包含内置函数或用户自定义函数，那么被引用的函数必须声明为 `DETERMINISTIC`（确定性）；换句话说，对于同一组输入值（例如，行中的其他列），该函数必须返回相同的结果。因此，你不能在虚拟列中使用诸如 `SYSDATE` 或 `SYSTIMESTAMP` 之类的函数，因为只要检索该行，它们几乎总是会返回不同的结果（除非是在同一秒内针对 `SYSDATE`，或同一百万分之一秒内针对 `SYSTIMESTAMP` 进行检索！）。如果你将一个非确定性的用户函数声明为 `DETERMINISTIC` 并在虚拟列中引用它，那就是自找麻烦；包含该虚拟列的查询结果很可能不正确，或者会根据查询执行的时间、执行者等因素而产生不同结果。

虚拟列的其他限制则不那么明显。一个虚拟列不能引用同一表中的另一个虚拟列（尽管你当然可以将一个虚拟列的定义包含在第二个虚拟列的定义中）。此外，虚拟列只能引用同一表内的列。总的来说，虚拟列在一定程度上减少了你可能需要在数据库视图中维护的元数据量，从而增强了数据库的易用性。

##### 4-2. 返回不存在的行

### 问题

用于表主键的序列号存在间隔，你希望找出中间缺失的序列号。

### 解决方案

尽管主键通常是（并且最好是）对最终用户不可见的，但有时你可能希望重用序列号，前提是必须将序列号的位数保持在尽可能低的水平，以满足下游系统或流程的需求。例如，公司棒球队使用 `EMPLOYEE_ID`（员工 ID）列中的数字作为队服背后的号码。经理希望尽可能重复使用旧队服号码，因为队服背后的号码限制为三位数。

要找出被跳过或当前未使用的员工编号，你可以使用一个包含两个嵌套子查询的查询，如下所示：

```sql
with all_used_emp_ids as
(select level poss_emp_id from (select max(employee_id) max_emp_num
from employees)
connect by level <= max_emp_num)
select poss_emp_id
from all_used_emp_ids
where poss_emp_id not in (select employee_id from employees)
order by poss_emp_id
;
```

```
POSS_EMP_ID
. . .
101 rows selected
```

该查询检索 `EMPLOYEES` 表中当前最高员工编号之下所有未使用的员工编号。

### 工作原理

该解决方案使用子查询因子化和两个显式子查询来找出缺失的员工编号列表。`WITH` 子句中的查询创建了一个从 1 到 `EMPLOYEES` 表中最高员工编号值的连续数字序列（无间隔）。子查询的高亮部分负责检索最高编号：

```sql
(select level poss_emp_id from (select max(employee_id) max_emp_num
from employees)
connect by level <= max_emp_num)
```

`WITH` 子句中查询的其余部分生成了这个无间隔的数字序列。它利用了 Oracle 层次查询特性（`CONNECT BY` 子句和 `LEVEL` 伪列）的一个微妙特性来生成序列中的每个数字。有关需要遍历层次数据的其他有用示例，请参见第 13 章。

以下是 `SELECT` 查询的主要部分：

```sql
select poss_emp_id
from all_used_emp_ids
where poss_emp_id not in (select employee_id from employees)
```

它从无间隔序列中检索所有可能的员工编号，这些编号尚未存在于 `EMPLOYEES` 表中。最终的结果集按 `POSS_EMP_ID` 排序，以便轻松地将最低的可用号码分配给下一个员工。如果你想始终排除一位数或两位数的编号，可以向 `WHERE` 子句中添加另一个谓词，如下所示：

```sql
with all_used_emp_ids as
(select level poss_emp_id from (select max(employee_id) max_emp_num
from employees)
connect by level <= max_emp_num)
select poss_emp_id
from all_used_emp_ids
where poss_emp_id not in (select employee_id from employees)
and poss_emp_id > 99
order by poss_emp_id
;
```

```
POSS_EMP_ID
2 rows selected
```

在修改后的查询结果中，在当前使用的最高员工编号之下，唯一可以重复使用的三位数编号是 179 和 183。

##### 4-3. 将行转换为列

### 问题

你的事务数据存储在高度规范化的数据库结构中，你希望为业务分析师轻松创建交叉表风格的报告，在单一行中以多列形式返回总计值或其他聚合结果。

### 解决方案

在 `SELECT` 语句中使用 `PIVOT` 关键字，将多个行中的值（来自一列或多列）聚合并展开到查询输出中的多列。例如，在 Oracle 订单录入示例模式 `OE` 中，你希望从所有订单中挑选五种关键产品，并找出哪些客户购买了这些产品，以及购买了多少。

`ORDERS` 表和 `ORDER_ITEMS` 表定义如下：

```
describe orders;

Name                                      Null      Type
----------------------------------------- -------- --------------------------------
ORDER_ID                                  NOT NULL NUMBER(12)
ORDER_DATE                                NOT NULL TIMESTAMP(6) WITH LOCAL TIME ZONE
ORDER_MODE                                         VARCHAR2(8)
CUSTOMER_ID                              NOT NULL NUMBER(6)
ORDER_STATUS                                      NUMBER(2)
ORDER_TOTAL                                       NUMBER(8,2)
SALES_REP_ID                                      NUMBER(6)
PROMOTION_ID                                      NUMBER(6)

8 rows selected
```

```
describe order_items;

Name                                      Null      Type
----------------------------------------- -------- --------------------------------
ORDER_ID                                  NOT NULL NUMBER(12)
LINE_ITEM_ID                              NOT NULL NUMBER(3)
PRODUCT_ID                                NOT NULL NUMBER(6)
UNIT_PRICE                                         NUMBER(8,2)
QUANTITY                                           NUMBER(8)
LINE_TOTAL_PRICE                                   VARCHAR2(14)

6 rows selected
```

以下是按客户检索前五种产品数量总计的透视查询：

```sql
with order_item_query as
(select customer_id, product_id, quantity
from orders join order_items using(order_id))
select * from order_item_query
pivot (
sum(quantity) as sum_qty
for (product_id) in (3170 as P3170,
                     3176 as P3176,
                     3182 as P3182,
                     3163 as P3163,
                     3165 as P3165)
)
order by customer_id
;
```

```
CUSTOMER_ID P3170_SUM_QTY P3176_SUM_QTY P3182_SUM_QTY P3163_SUM_QTY P3165_SUM_QTY
----------- ------------- ------------- ------------- ------------- -------------
        101           208
        104            70            72            77            61            64
        106            62            55
        107            76
        108            45            31
        109           115           112
        116            48            24             5            10
        117            63            67
        118            42
        119            36            30
. . .
        170            92

47 rows selected
```

从这个查询的结果中，你可以很容易地看出客户编号 104 是 `PIVOT` 子句中指定的所有产品的最大买家。

### 工作原理

`PIVOT` 运算符，顾名思义，将行旋转为列。在解决方案中，你会注意到一些熟悉的结构，以及一些不那么熟悉的。首先，解决方案使用了 `WITH` 子句（子查询因子化）将基础查询与查询的其余部分更清晰地分离开；以下是解决方案中使用的 `WITH` 子句：

```sql
with order_item_query as
(select customer_id, product_id, quantity
from orders join order_items using(order_id))
```

使用 `WITH` 子句还是内联视图是风格问题。无论哪种情况，请确保只包含你将用于分组、聚合的列，或者在 `PIVOT` 子句中使用的列。



## 第四章 ■ 创建与推导数据

##### 4-4. 在多个列上进行透视

否则，如果你的查询中有多余的列，你的结果将会比预期多出许多行！这非常类似于在带有聚合的常规 `SELECT` 语句中的 `GROUP BY` 子句添加额外的列，通常会增加结果中的行数。

`PIVOT` 子句本身会在查询结果中生成额外的列。以下是解决方案中 `PIVOT` 子句的第一部分：

`sum(quantity) as sum_qty`

你指定一个或多个希望出现在新列中的聚合，并为 Oracle 将用于每个新列分配一个标签后缀。（我们将在本节后面展示如何在多个列上进行透视或聚合多个列。）在本例中，所有派生列都将以 `SUM_QTY` 结尾。

`FOR` 子句中的列会筛选用于聚合结果的行。`FOR` 子句中列的每个值，或每个指定的两列或多列组合，都将在结果中产生一个额外的派生列。解决方案中的 `FOR` 子句如下：`for (product_id) in (3170 as P3170, 3176 as P3176, 3182 as P3182, 3163 as P3163, 3165 as P3165)`

该查询将仅使用 `PRODUCT_ID` 在列表中的行，因此其作用类似于 `WHERE` 子句，在聚合之前筛选结果。每个透视值后指定的文本字符串用作新列名的前缀，如解决方案查询的结果所示。

最后，`ORDER BY` 子句的作用与在任何其他查询中的作用相同——按指定的列对最终结果集进行排序。

[www.it-ebooks.info](http://www.it-ebooks.info/)

#### 问题

你已经实现了前一个方案中展示的解决方案。现在你希望不仅根据产品的总数量进行透视，还根据购买的总美元价值进行透视。

#### 解决方案

以下查询扩展了前一个方案的解决方案，为每个产品编号添加了一个总金额列：

```sql
with order_item_query as
(select customer_id, product_id, quantity, unit_price
 from orders join order_items using(order_id))
select * from order_item_query
pivot (
 sum(quantity) as sum_qty,
 sum(quantity*unit_price) as sum_prc
 for (product_id) in (3170 as P3170,
                      3176 as P3176,
                      3182 as P3182,
                      3163 as P3163,
                      3165 as P3165)
 )
order by customer_id
;
```

```
CUSTOMER_ID P3170_SUM_QTY P3170_SUM_PRC P3176_SUM_QTY P3176_SUM_PRC . . .
----------- ------------- ------------- ------------- ------------- ---------
        104            70         10164            72         8157.6
        106            62         7024.6
        116            48         6969.6            24         2719.2
        118            42         6098.4
        119            36         5227.2
. . .
```

这份报告进一步强化了原始方案的发现：客户编号 `104` 在支付总金额方面也是一个大买家。

[www.it-ebooks.info](http://www.it-ebooks.info/)

#### 工作原理

一旦掌握了 `PIVOT` 子句的基础，更进一步操作就很容易了。只需根据需要向你的 `PIVOT` 子句添加任意多个额外的聚合。不过要记住，添加第二个聚合会使派生列的数量翻倍，添加第三个则使其变为三倍，依此类推。

让我们进一步扩展这个解决方案，再添加一个透视列。在 `ORDERS` 表中，如果销售是通过电话进行的，`ORDER_MODE` 列包含字符串 `direct`；如果订单是在公司互联网站点下达的，则包含 `ONLINE`。要在 `ORDER_MODE` 上进行透视，只需将该列添加到 `FOR` 子句中，并在 `FOR` 列表中指定成对的值而不是单个值，如下所示：

```sql
with order_item_query as
(select customer_id, product_id, quantity, unit_price, order_mode
 from orders join order_items using(order_id))
select * from order_item_query
pivot (
 sum(quantity) as sum_qty,
 sum(quantity*unit_price) as sum_prc
 for (product_id, order_mode) in ((3170, 'direct' ) as P3170_DIR,
                                  (3170, 'online' ) as P3170_ONL,
                                  (3176, 'direct' ) as P3176_DIR,
                                  (3176, 'online' ) as P3176_ONL,
                                  (3182, 'direct' ) as P3182_DIR,
                                  (3182, 'online' ) as P3182_ONL,
```



```
(3163, 'direct' ) as P3163_DIR,
(3163, 'online' ) as P3163_ONL,
(3165, 'direct' ) as P3165_DIR,
(3165, 'online' ) as P3165_ONL)
)
order by customer_id
;

CUSTOMER_ID P3170_DIR_SUM_QTY P3170_DIR_SUM_PRC P3170_ONL_SUM_QTY P3170_ONL_SUM_PRC
----------- ----------------- ----------------- ----------------- -----------------
104 70 10164
116 24 3484.8 24 3484.8
118 42 6098.4
. . .
```

输出在水平方向上已被截断以方便阅读。对于产品编号、订单模式及两个聚合值的每一种组合，都有一列。查看按 93

[www.it-ebooks.info](http://www.it-ebooks.info/)

