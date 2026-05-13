# 第 18 章 ■ 对象管理

##### 18-8. 创建基于函数的索引

### 问题

你正在运行这条 SQL 查询：

```
select emp_name from emp where UPPER(emp_name) = 'DAVE';
```

你已经在 `EMP_NAME` 列上创建了一个普通的 B-tree 索引，但你注意到，当你对 `EMP_NAME` 列应用 `UPPER` 函数时，Oracle 并未使用该索引。你想知道是否可以创建一个 Oracle 会使用的索引。

### 解决方案

创建一个基于函数的索引，以提高在 `WHERE` 子句中使用了 SQL 函数的查询的性能。此示例在 `UPPER(EMP_NAME)` 上创建了一个基于函数的索引：

```
create index user_upper_idx on emp(upper(emp_name));
```

为什么要使用基于函数的索引？因为这是 Oracle 在此场景下唯一会使用的索引类型。如果你在 `WHERE` 子句中对列应用 SQL 函数，Oracle 将忽略该列上的常规索引。

### 工作原理

基于函数的索引允许在 SQL 查询的 `WHERE` 子句中对函数引用的列进行索引查找。基于函数的索引可以像前面的示例那样简单，也可以基于存储在 PL/SQL 函数中的复杂逻辑。

任何用户创建的 SQL 函数在用于基于函数的索引之前，必须声明为 `deterministic`（确定性）。确定性意味着对于一组给定的输入，函数总是返回相同的结果。当你创建一个希望用于基于函数的索引的用户定义函数时，必须使用关键字 `DETERMINISTIC`。

从 `DBA/ALL/USER_IND_EXPRESSIONS` 视图中选择，以显示与基于函数的索引相关的 SQL：

```
select index_name, column_expression from user_ind_expressions;
```

如果你正在从 SQL*Plus 运行此 SQL，请不要忘记使用 `SET LONG` 命令，以便显示 `COLUMN_EXPRESSION` 列的完整内容。例如：

```
SQL> set long 500
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

