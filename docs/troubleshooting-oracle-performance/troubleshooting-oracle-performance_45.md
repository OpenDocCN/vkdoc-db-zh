# SQL 调优集

简而言之，SQL 调优集是存储一组 SQL 语句及其关联的执行环境、执行统计信息，以及可选的执行计划的对象。它们从 Oracle Database 10g 开始可用。SQL 调优集使用包 `dbms_sqltune` 进行管理。你可以在*性能调优指南*手册中找到有关它们的更多信息。

为了简化函数 `create_tuning_task` 的执行，使其能够接受单个 SQL 语句作为参数，我编写了脚本 `tune_last_statement.sql`。思路是你在 SQL*Plus 中执行你想要分析的 SQL 语句，然后无参数调用该脚本。该脚本从视图 `v$session` 中获取当前会话最后执行的 SQL 语句的引用 (`sql_id`)，然后创建并执行一个引用它的调优任务。脚本的核心部分是以下匿名 PL/SQL 块：

```sql
DECLARE
  l_sql_id v$session.prev_sql_id%TYPE;
  l_tuning_task VARCHAR2(30);
BEGIN
  SELECT prev_sql_id INTO l_sql_id
  FROM v$session
  WHERE audsid = sys_context('userenv','sessionid');
  l_tuning_task := dbms_sqltune.create_tuning_task(sql_id => l_sql_id);
  dbms_sqltune.execute_tuning_task(:tuning_task);
  dbms_output.put_line(l_tuning_task);
END;
```

调优任务将其分析结果输出到多个数据字典视图中。与其直接查询这些视图（这有点麻烦），你可以使用包 `dbms_sqltune` 中的函数 `report_tuning_task` 来生成关于分析的详细报告。以下查询展示了其使用示例。注意，为了引用调优任务，使用了前面 PL/SQL 块返回的调优任务名称。

```sql
SELECT dbms_sqltune.report_tuning_task('TASK_16467')
FROM dual
```

由前面查询生成的、建议使用 SQL 配置文件的报告如下所示。请注意，这是脚本 `first_rows.sql` 生成的输出的一个摘录。

## 常规信息部分

```sql
调优任务名称                : TASK_16467
调优任务所有者               : OPS$CHA
作用域                           : 综合
时间限制(秒)             : 42
完成状态               : 已完成
开始于                      : 07/19/2007 12:45:55
完成于                    : 07/19/2007 12:45:55
SQL 配置文件发现数  : 1
```

## 方案名称与 SQL 文本

```sql
方案名称: OPS$CHA
SQL ID     : f3cq1hxz2d041
SQL 文本   : SELECT * FROM T ORDER BY ID
```

## 发现部分 (1 项发现)

### 1- SQL 配置文件发现 (参见下面的执行计划部分)

为此语句找到了一个可能更好的执行计划。

#### 建议 (估计收益：97.21%)

- 考虑接受推荐的 SQL 配置文件。
    ```sql
    execute dbms_sqltune.accept_sql_profile(task_name => 'TASK_16467',
            replace => TRUE);
    ```

## 执行计划部分

### 1- 调整成本后的原始计划

```sql
Plan hash value: 961378228
-------------------------------------------------------------------------------
| Id  | Operation        | Name| Rows  | Bytes|TempSpc| Cost (%CPU)| Time     |
-------------------------------------------------------------------------------
|   0 |SELECT STATEMENT  |     | 10000 | 1015K|       |   287   (1)| 00:00:04 |
|   1 | SORT ORDER BY    |     | 10000 | 1015K|  2232K|   287   (1)| 00:00:04 |
|   2 | TABLE ACCESS FULL| T   | 10000 | 1015K|       |    47   (0)| 00:00:01 |
-------------------------------------------------------------------------------
```

### 2- 使用 SQL 配置文件

```sql
Plan hash value: 1399892806
-------------------------------------------------------------------------------------
| Id  | Operation                   | Name  | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------
| 0   | SELECT STATEMENT            |       |    6  |   624 |    8    (0)| 00:00:01 |
| 1   |  TABLE ACCESS BY INDEX ROWID|  T    | 10000 |  1015K|    8    (0)| 00:00:01 |
| 2   |   INDEX FULL SCAN           | T_PK  |     6 |       |    2    (0)| 00:00:01 |
-------------------------------------------------------------------------------------
```

要使用 SQL 调优顾问建议的 SQL 配置文件，你必须接受它。下一节描述了如何操作。无论是否接受 SQL 配置文件，一旦你不再需要调优任务，都可以通过调用包 `dbms_sqltune` 中的过程 `drop_tuning_task` 来删除它：

```sql
dbms_sqltune.drop_tuning_task('TASK_16467');
```

