# 第 19 章 ■ SQL 监控与调优

##### 19-7. 使用 AUTOTRACE 显示执行计划

### 问题

在执行 SQL 语句时遇到了性能问题，你希望查看优化器计划如何检索数据以构建查询结果集。

### 解决方案

生成执行计划的一个简便方法是使用 `AUTOTRACE` 实用程序。在生成解释计划之前，必须先确保 `AUTOTRACE` 在你的数据库中可用。

### 设置 `AUTOTRACE`

按照以下步骤设置 `AUTOTRACE` 工具以显示解释计划。

1.  确保 `PLAN_TABLE` 表存在。要查看你的模式中是否有 `PLAN_TABLE`，尝试描述它：
    ```
    SQL> desc plan_table;
    ```
    如果 `PLAN_TABLE` 不存在，你需要创建一个。运行此脚本在你的模式中创建 `PLAN_TABLE`：
    ```
    SQL> @?/rdbms/admin/utlxplan
    ```

2.  你的模式还需要访问 `PLUSTRACE` 角色。你可以使用以下语句验证对 `PLUSTRACE` 角色的访问权限：
    ```
    select username, granted_role from user_role_privs
    where granted_role='PLUSTRACE';
    ```
    如果你没有访问 `PLUSTRACE` 角色的权限，请以 `SYS` 模式身份执行步骤 3 和 4。

3.  以 `SYS` 身份连接并运行 `plustrce.sql` 脚本：
    ```
    SQL> conn / as sysdba
    SQL> @?/sqlplus/admin/plustrce
    ```

4.  将 `PLUSTRACE` 角色授予希望使用 `AUTOTRACE` 工具的开发人员（或特定角色）：
    ```
    SQL> grant plustrace to star1;
    ```

### 生成执行计划

以下是使用 `AUTOTRACE` 工具的最简单方法：
```
SQL> set autotrace on;
```
现在，你运行的任何 SQL 都会生成一个解释计划。
```
select emp_name from emp;
```
这是输出的部分列表：
```
| Id | Operation         | Name | Rows | Bytes | Cost (%CPU)| Time     |
|----|-------------------|------|------|-------|------------|----------|
|  0 | SELECT STATEMENT  |      |    1 |    17 |      5 (0)| 00:00:01 |
|  1 | TABLE ACCESS FULL | EMP  |    1 |    17 |      5 (0)| 00:00:01 |
```
注意：
- 此语句使用了动态采样

统计信息：
```
28  recursive calls
0   db block gets
33  consistent gets
```
要关闭 `AUTOTRACE`，请使用 `OFF` 选项：
```
SQL> set autotrace off;
```

注意：执行计划通常也被称为解释计划。这是因为你可以使用 SQL 的 `EXPLAIN PLAN` 语句来生成执行计划。

### 工作原理

`AUTOTRACE` 工具可以以多种模式调用。例如，如果你只想生成统计信息（而不生成结果集），请使用此模式：
```
SQL> set autotrace trace explain
```
要查看所有 `AUTOTRACE` 选项，请输入此语句：
```
SQL> set autotrace help
```
实际上，前一条语句显示了帮助，因为 `HELP` 选项不存在，并且当输入了不正确的选项时，`AUTOTRACE` 会自动显示用法信息。表 19-1 解释了可以调用 `AUTOTRACE` 的不同模式。

**表 19-1. AUTOTRACE 选项**

| 选项 | 结果 |
|-----------|--------|
| `SET AUTOTRACE ON` | 查询输出、解释计划、统计信息 |
| `SET AUTOTRACE OFF` | 关闭 `AUTOTRACE` |
| `SET AUTOTRACE ON EXPLAIN` | 查询输出、解释计划、无统计信息 |
| `SET AUTOTRACE ON EXPLAIN STAT` | 查询输出、解释计划、统计信息 |
| `SET AUTOTRACE ON STAT` | 查询输出、统计信息、无解释计划 |
| `SET AUTOTRACE TRACE` | 解释计划、统计信息，生成但不显示结果 |


