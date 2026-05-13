# 第 18 章 ■ 对象管理

##### 18-10. 创建索引组织表

### 问题

你有一个主要通过基于主键或主键值范围的查询来访问表数据的应用程序。你希望高效地查询此表。

### 解决方案

当表数据通常通过主键查询访问时，索引组织表（IOT）是高效的对象。使用 `ORGANIZATION INDEX` 子句来创建索引组织表：

```
create table prod_sku
(prod_sku_id number,
 sku varchar2(256),
 create_dtt timestamp(5),
 constraint prod_sku_pk primary key(prod_sku_id)
 )
organization index
including sku
pctthreshold 30
tablespace inv_mgmt_data
overflow
tablespace mts;
```

### 工作原理

索引组织表将表行的全部内容存储在 B-tree 索引结构中。索引组织表为那些在主键上进行精确匹配和/或范围搜索的查询提供了快速访问。

`INCLUDING` 子句指定了哪些列开始存储在溢出数据段中。在前面的示例中，`SKU` 和 `CREATE_DTT` 列存储在溢出段中。

`PCTTHRESHOLD` 指定了为索引组织表行在索引块中保留的空间百分比。此值可以是 1 到 50，如果未指定值，则默认为 50。索引块中必须有足够的空间来存储主键。

`OVERFLOW` 子句详细说明了应使用哪个表空间来存储溢出数据段。



你会注意到，在创建索引组织表（IOT）时，所使用的表名会在 `DBA/ALL/USER_TABLES` 中有一条记录。此外，在 `DBA/ALL/USER_INDEXES` 中也会有一条记录，其名称是指定的主键约束名。对于索引组织表，`INDEX_TYPE` 列的值将为 `IOT - TOP`：

```sql
select index_name,table_name,index_type from user_indexes;
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第 18 章 ■ 对象管理

以下是输出示例：

```text
INDEX_NAME TABLE_NAME INDEX_TYPE

-------------------- -------------------- ---------------------------

PROD_SKU_PK PROD_SKU IOT - TOP
```

##### 18-11. 创建视图

#### 问题
你想要存储一个经常运行的 SQL 查询。

#### 解决方案
使用 `CREATE VIEW` 命令来创建视图。以下代码创建一个视图（如果视图已存在，则替换它），该视图从 `SALES` 表中选择一部分列和行：

```sql
create or replace view sales_rockies as

select sales_id, amnt, state

from sales

where state in ('CO','UT','WY','ID','AZ');
```

现在，你可以像对待表一样对待 `SALES_ROCKIES` 视图。只从一个表中选择的视图称为简单视图。拥有该视图访问权限的模式可以执行其拥有对象权限的任何 `SELECT`、`INSERT`、`UPDATE` 或 `DELETE` 操作，并且 DML 操作会导致底层表数据被更改。

如果你想只允许更改视图范围内的底层表数据，则指定 `WITH CHECK OPTION`：

```sql
create or replace view sales_rockies as

select sales_id, amnt, state

from sales

where state in ('CO','UT','WY','ID','AZ')

with check option;
```

使用 `WITH CHECK OPTION` 意味着你只能插入或更新那些会被视图查询返回的行。例如，以下更新语句会成功，因为我们没有以会导致其不被视图返回的方式更改底层数据：

```sql
update sales_rockies set state='CO' where sales_id=160;
```

然而，接下来的这个更新语句将失败，因为它试图将 `STATE` 列更新为一个无法通过视图选取的值：

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第 18 章 ■ 对象管理

```sql
update sales_rockies set state='CA' where sales_id=160;
```

如果你不希望用户能够对视图执行 `INSERT`、`UPDATE` 或 `DELETE` 操作，那么就不要授予该用户对底层表的那些对象权限。如果你不希望拥有底层表（或拥有修改表数据权限）的账户能够通过视图更改底层表数据，请使用 `WITH READ ONLY` 子句创建视图：

```sql
create or replace view sales_rockies as

select sales_id, amnt, state

from sales

where state in ('CO','UT','WY','ID','AZ')

with read only;
```

如果你需要修改视图所基于的 SQL 查询，那么可以先删除再重新创建它，或者像前面的例子那样使用 `CREATE OR REPLACE` 语法。`CREATE OR REPLACE` 方法的优点是，你无需为先前已授予权限的用户重新建立对视图的访问权限。如果你删除并重新创建视图，则需要向先前被授予对该已删除和重建对象访问权限的任何用户或角色重新授予权限。

`注意` 有一个 `ALTER VIEW` 命令用于修改视图的约束属性。你也可以使用 `ALTER VIEW` 命令来重新编译视图。

#### 工作原理
你可以将视图视为存储在数据库中的 SQL 语句。当你从视图中查询时，Oracle 会在数据字典中查找视图定义，执行视图所基于的查询，并返回结果。换句话说，视图是建立在其他表和/或视图之上的逻辑表。视图用于：
-   创建一种高效的存储 SQL 查询以供重用的方法。
-   在应用程序和物理表之间提供一个接口层。
-   向应用程序隐藏 SQL 查询的复杂性。
-   仅向用户报告列和/或行的子集。



