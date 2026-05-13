# 未执行的代码

如果有一种简单的方法可以选择所有未执行的、可执行的代码，那就太好了。`PL/Scope` 提供了部分你需要的信息，但 `DBMS_TRACE` 执行查询的方式（它捕获包含 `select` 语句的行，但不捕获其他行）带来了一些问题。此外，`PL/Scope` 并不会捕获所有潜在的可执行行，它只捕获包含标识符的行。如果你查看下面的演示过程 `TEST_PLSCOPE`，你会看到这个 `PROCEDURE` 有两行可执行代码（忽略过程的声明和定义）：即 `IF 1=1` 和 `execute immediate`。

```sql
CREATE OR REPLACE PROCEDURE TEST_PLSCOPE AS
BEGIN
  IF 1=1
  THEN
    execute immediate 'declare v1 number; begin select 1 into v1 from dual; end;';
  END IF;
END TEST_PLSCOPE;
```

正如下一个查询所示，如果你查询 `USER_IDENTIFIERS`，你只会看到声明和定义：

```sql
  SELECT object_name, line, usage
  FROM user_identifiers
  WHERE object_name = 'TEST_PLSCOPE'
    AND object_type = 'PROCEDURE'
ORDER BY usage_id;
```

```
OBJECT_NAME                    LINE                   USAGE
------------------------------ ---------------------- -----------
TEST_PLSCOPE                   1                      DECLARATION
TEST_PLSCOPE                   1                      DEFINITION
```

目前，你可以自动收集已执行代码的覆盖率，但无法自动收集未执行代码的信息。你能达到的最好程度是一个可以由人工仔细检查的自动化报告。这样的报告聊胜于无，但并不像我期望的那样完善。

下面的查询试图通过使用 `USER_IDENTIFIERS` 来检索未被 `DBMS_TRACE` 运行所包含的所有可执行行，从而使前面查询的代码覆盖率更加稳健：

```sql
SELECT id_data.object_name, id_data.line, us.text
  FROM (
    SELECT DISTINCT object_type, object_name, line
      FROM user_identifiers ui
      WHERE (object_type, object_name) IN
            (SELECT distinct event_unit_kind, event_unit
               FROM plsql_trace_events pse
               WHERE pse.event_unit = ui.object_name
                 AND pse.event_unit_kind = ui.object_type
                 AND pse.runid = :TRACE_RUNID
                 AND pse.event_kind NOT IN (50, 54))
                 AND usage NOT IN ('DEFINITION', 'DECLARATION')
    MINUS
    SELECT event_unit_kind, event_unit, event_line
      FROM plsql_trace_events pse
      WHERE pse.runid = :TRACE_RUNID
        AND pse.event_kind NOT IN (50, 54)
        AND event_line IS NOT NULL) id_data
  JOIN user_source us
    ON (us.name = id_data.object_name
    AND us.type = id_data.object_type
    AND us.line = id_data.line)
  ORDER BY id_data.object_name, id_data.line;
```

在这一部分结果中（以下输出来自结果集的中间部分），你可以看到我希望 `PL/Scope` 数据能输出的内容：

```
OBJECT_NAME                      LINE TEXT
------------------------------- ----- ------------------------------
DEPARTMENTS_API                    42       WHEN NO_DATA_FOUND
DEPARTMENTS_API                    44         RETURN v_manager_id;
EMPLOYEE_API                       27       v_exception := TRUE;
EMPLOYEE_API                       28       v_exception_message := v
                                          _exception_message || chr(10)
                                          ||

EMPLOYEE_API                       29                  'Department '
                                           || p_department_name || ' not
                                           found.';

EMPLOYEE_API                       35       v_exception := TRUE;
EMPLOYEE_API                       36       v_exception_message := v
                                          _exception_message || chr(10)
                                          ||
```

不幸的是，结果集开头包含一个我想过滤掉的例子。

```
OBJECT_NAME                      LINE TEXT
------------------------------- ----- ------------------------------
DEPARTMENTS_API                    10       INTO v_department_id
DEPARTMENTS_API                    12       WHERE upper(department_n
                                          ame) = upper(p_department_name
                                          );
```

这些行是作为从第 9 行开始的 `select` 语句的一部分执行的。`PL/Scope` 包含第 10 和 12 行是因为它们包含标识符：`v_department_id` 和 `p_department_name`。第 11 行是 `FROM` 子句，不包含任何标识符。

前面示例中显示的所有其他行确实没有执行。它们嵌入在异常处理代码中，而这正是你希望代码覆盖率报告能显示的内容类型。理想情况下，你会看到这些行未被执行，并据此创建测试来执行这些代码。

