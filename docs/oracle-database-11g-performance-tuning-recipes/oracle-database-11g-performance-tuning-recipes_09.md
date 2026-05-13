# 2-8. 创建函数索引

## 问题

你需要创建一个函数索引。

## 解决方案

使用 `CREATE INDEX` 语句并在其定义中包含一个函数或表达式来创建函数索引：

```sql
SQL> create index cust_upper_idx1
  2  on cust(upper(first_name));
```

这将在 `cust` 表上创建一个函数索引，该索引基于 `UPPER(first_name)` 表达式。

## 工作原理

**函数索引** 在其定义中使用函数或表达式创建。函数索引允许在查询的 `WHERE` 子句中引用 SQL 函数的列上进行索引查找。该索引可以像本解决方案示例中那样简单，也可以基于存储在 PL/SQL 函数中的复杂逻辑。

![images](img/square.jpg) `Note` 任何用户创建的 SQL 函数在用于函数索引之前，都必须声明为确定性的。`DETERMINISTIC` 意味着对于一组给定的输入，函数总是返回相同的结果。在创建要在函数索引中使用的用户定义函数时，必须使用关键字 `DETERMINISTIC`。

如果你想查看函数索引的定义，可以从 `DBA/ALL/USER_IND_EXPRESSIONS` 视图中选择，以显示与索引关联的 SQL。如果你使用的是 SQL*Plus，请务必先发出 `SET LONG` 命令，例如：

```sql
SQL> SET LONG 500
SQL> select index_name, column_expression from user_ind_expressions;
```

此示例中的 `SET LONG` 命令告诉 SQL*Plus 显示 `COLUMN_EXPRESSION` 列中最多 500 个字符，该列的类型为 `LONG`。

