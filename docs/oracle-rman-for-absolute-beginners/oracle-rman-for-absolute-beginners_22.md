# 使用 Oracle Data Pump 进行高级数据过滤与导出

## 使用查询限制数据
请注意，此技术无法感知可能存在的任何外键约束，因此在考虑父-子关系之前，不能盲目地限制数据集。`QUERY`参数用于包含查询的通用语法如下：

```
QUERY = [schema.][table_name:] query_clause
```

查询子句可以是任何有效的 SQL 子句。查询必须用双引号或单引号括起来。我建议使用双引号，因为查询中可能需要嵌入单引号来处理`VARCHAR2`数据。此外，您应该使用参数文件，以避免操作系统对引号的解释产生混淆。

此示例使用参数文件并限制两个表导出的行数。导出时使用的参数文件如下：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
tables=inv,reg
query=inv:"WHERE inv_desc='Book'"
query=reg:"WHERE reg_id <=20"
```

假设您将上述代码行放在一个名为`inv.par`的文件中。导出作业引用该参数文件，如下所示：

```
$ expdp parfile=inv.par
```

生成的转储文件仅包含由`QUERY`参数过滤的行。再次提醒，请注意任何父-子关系，并确保导出的内容不会违反导入时的任何约束。

## 在导入时使用查询
您也可以在导入数据时指定查询。以下是一个参数文件，它根据`INV_ID`列限制导入`INV`表的行数：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
tables=inv,reg
query=inv:"WHERE inv_id > 10"
```

此文件内容被放入一个名为`inv2.par`的文件中，并在导入时引用，如下所示：

```
$ impdp parfile=inv2.par
```

`REG`表的所有行都被导入。只有`INV`表中`INV_ID`大于 10 的行被导入。

## 按百分比导出数据
在导出时，`SAMPLE`参数指示 Data Pump 根据您提供的数字检索一定百分比的行。Data Pump 在导出时不会跟踪父-子关系。因此，当您的表通过外键约束链接，并且您尝试随机选择一定百分比的行时，这种方法效果不佳。

此参数的通用语法如下：

```
SAMPLE=[[schema_name.]table_name:]sample_percent
```

例如，如果您想导出表中 10%的数据，可按如下方式操作：

```
$ expdp mv_maint/foo directory=dp_dir tables=inv sample=10 dumpfile=inv.dmp
```

下一个示例导出两个表，但仅导出`REG`表 30%的数据：

```
$ expdp mv_maint/foo directory=dp_dir tables=inv,reg sample=reg:30 dumpfile=inv.dmp
```

![Image](img/sq.jpg) **注意**  `SAMPLE`参数仅对导出有效。

## 从导出文件中排除对象
对于导出，`EXCLUDE`参数指示 Data Pump 不导出指定的对象（而`INCLUDE`参数则指示 Data Pump 在导出文件中仅包含特定对象）。`EXCLUDE`参数的通用语法如下：

```
EXCLUDE=object_type[:name_clause] [, ...]
```

`OBJECT_TYPE`是数据库对象，例如`TABLE`或`INDEX`。要查看可以过滤哪些对象类型，请查看`DATABASE_EXPORT_OBJECTS`、`SCHEMA_EXPORT_OBJECTS`或`TABLE_EXPORT_OBJECTS`的`OBJECT_PATH`列。例如，如果您想查看可以过滤哪些模式级对象，请运行此查询：

```
SELECT  object_path
FROM    schema_export_objects
WHERE   object_path NOT LIKE '%/%';
```

以下是输出的片段：

```
OBJECT_PATH
------------------
STATISTICS
SYNONYM
SYSTEM_GRANT
TABLE
TABLESPACE_QUOTA
TRIGGER
```

`EXCLUDE`参数指示 Data Pump 导出从导出中过滤掉特定对象。例如，假设您正在导出一个表但希望排除索引和授权：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp tables=inv exclude=index,grant
```

您可以通过使用`NAME_CLAUSE`进行更细粒度的过滤。`EXCLUDE`的`NAME_CLAUSE`选项允许您指定一个 SQL 过滤器。要排除名称以字符串“INV”开头的索引，请使用以下命令：

```
exclude=index:"LIKE 'INV%'"
```

前面的命令要求您使用引号；在这些情况下，我建议您使用参数文件。以下是一个包含`EXCLUDE`子句的参数文件：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
tables=inv
exclude=index:"LIKE 'INV%'"
```

`EXCLUDE`子句的某些方面可能看起来违反直觉。例如，考虑以下导出参数文件：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=sch.dmp
exclude=schema:"='HEERA'"
```

如果您尝试以这种方式排除用户，则会抛出错误。这是因为默认的导出模式是`SCHEMA`级别，并且 Data Pump 不能同时排除和包含一个模式。如果您想从导出文件中排除一个用户，请指定`FULL`模式，然后排除该用户：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=sch.dmp
exclude=schema:"='HEERA'"
full=y
```

## 排除统计信息
默认情况下，当您导出表对象时，任何统计信息也会被导出。您可以通过`EXCLUDE`参数防止统计信息被导入。这是一个例子：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp \
tables=inv exclude=statistics
```

在导入时，如果您尝试从原本不包含统计信息的转储文件中排除统计信息，则会收到此错误：

```
ORA-39168: Object path STATISTICS was not found.
```

如果导出转储文件中的对象从未生成过统计信息，您也会收到此错误。

## 在导出文件中仅包含特定对象
使用`INCLUDE`参数在导出文件中仅包含某些数据库对象。以下示例仅导出用户拥有的过程和函数：

```
$ expdp mv_maint/foo dumpfile=proc.dmp directory=dp_dir include=procedure,function
```

创建的`proc.dmp`文件仅包含重新创建用户拥有的任何过程和函数所需的 DDL。

使用`INCLUDE`时，您也可以指定仅导出特定的 PL/SQL 对象：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=ss.dmp \
include=function:\"=\'IS_DATE\'\"
```

当仅导出特定的 PL/SQL 对象时，由于需要在操作系统命令行上转义引号的问题，我建议使用参数文件。使用参数文件时，这不是问题。以下示例显示了导出特定对象的参数文件内容：

```
directory=dp_dir
dumpfile=ss.dmp
include=function:"='ISDATE'",procedure:"='DEPTREE_FILL'"
```

如果您指定一个不存在的对象，Data Pump 会抛出错误但会继续导出操作：

```
ORA-39168: Object path FUNCTION was not found.
```

## 导出表、索引、约束和触发器的 DDL
假设您想导出数据库中与表、索引、约束和触发器关联的 DDL。为此，请使用`FULL`导出模式，指定`CONTENT=METADATA_ONLY`，并且仅包含表：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=ddl.dmp \
content=metadata_only full=y include=table
```

当您导出一个对象时，Data Pump 也会导出任何依赖对象。因此，当您导出一个表时，您也会获得与该表关联的索引、约束和触发器。

## 从导入中排除对象
通常，您可以使用与在导出中过滤对象相同的技术来排除对象被导入。使用`EXCLUDE`参数来排除对象被导入。例如，要排除触发器和过程被导入，请使用以下命令：

```
$ impdp mv_maint/foo dumpfile=inv.dmp exclude=trigger,procedure
```



