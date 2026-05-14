# 从外部表进行内联 SQL 查询

从 Oracle 18c 开始，利用 `EXTERNAL` 关键字，可以直接从文件中进行查询，而无需在数据字典中真正创建一个外部表。这使得外部数据可以成为子查询、虚拟视图或其他转换类型流程的一部分。以下是如何实现这一点的示例。

```sql
SELECT columns FROM EXTERNAL ((column definitions) TYPE [access_driver_type] external_table_properties [REJECT LIMIT clause])
```

```
SQL> SELECT first_name, last_name, hiredate, department_name from EXTERNAL(
(first_name varchare2(50),
last_name   varchar2(50),
hiredate           date,
department_name    varchar2(50))
TYPE ORACLE_LOADER
DEFAULT DIRECTORY EXT_DATA
ACCESS PARAMETERS (
RECORDS DELIMITED BY NEWLINE nobadfile, nologfile
fields date_format date mask "mm/dd/yy")
LOCATION ('empbydep.csv') REJECT LIMIT UNLIMITED) empbydep_external
where department='HR';
```

`empbydep_external` 表并非作为外部表被实际创建，但这些数据可供查询，并可在 `WHERE` 子句中指定任何列或使用不同的条件进行筛选。这对于 JSON 格式同样可行，并且在访问以 JSON 格式提供的数据 API 时非常有用。这种方法不会将数据加载到表中，但可以进行查询，并可通过多种不同方法用于视图、通过 API 获取的参考数据，以及在数据集成中补全数据集。以下是一个 JSON 文件的示例：

```
SQL> select * from external ((json_document CLOB)
TYPE ORACLE_LOADER
DEFAULT DIRECTORY EXT_DATA
ACCESS PARAMETERS (
RECORDS DELIMITED BY 0x'0A' FIELDS (json_document CHAR(5000)) )
location ('empbydep.json') REJECT LIMIT UNLIMITED) json_tab;
```

## 总结

`SQL*Loader` 是一个适用于各类数据加载任务的实用工具，而外部表在数据加载或查询过程中的数据转换方面非常有用。几乎所有你可以用 `SQL*Loader` 完成的事情，也可以用外部表来完成。外部表方法的优势在于其组件更少，并且接口是 `SQL*Plus`。大多数 DBA 和开发者都认为 `SQL*Plus` 比 `SQL*Loader` 的控制文件更易于使用。

你可以轻松地使用外部表来让 `SQL*Plus` 访问操作系统的平面文件。你只需在 `CREATE TABLE...ORGANIZATION EXTERNAL` 语句中定义平面文件的结构。外部表创建后，你就可以直接从该平面文件进行查询，就像查询数据库表一样。你可以从外部表进行选择，但不能对它执行插入、更新或删除操作。

创建外部表后，如果需要，你可以通过从该外部表执行 `CREATE TABLE AS SELECT` 来创建常规的数据库表，或者使用基于该外部表的视图用于其他查询。这提供了一种快速有效的方法来加载存储在外部操作系统文件中的数据。

外部表功能还允许你从表中选择数据并将它们写入二进制转储文件。`CREATE TABLE...ORGANIZATION EXTERNAL` 语句定义了用于卸载数据的表和列。以此方式创建的转储文件是平台无关的，这意味着你可以将其复制到使用不同操作系统的服务器上并无缝加载数据。此外，为了安全和高效地传输，转储文件还可以进行加密和压缩。你还可以使用并行功能来缩短创建转储文件所需的时间。

外部表设计还允许你直接从文件查询数据。这对于 JSON 格式（许多数据 API 可能提供的格式）也非常有用。筛选数据或以简化的方式将数据加载到其他表中，对于数据集成和 ETL 流程极其有用。

### 15. 物化视图
下一章将介绍物化视图。这些数据库对象为你提供了一个灵活、可维护且可扩展的机制，用于聚合和复制数据。


