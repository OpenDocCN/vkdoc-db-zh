# 转换为字符串类型

对于所有数据库管理系统，您都可以使用 `cast()` 函数强制将值转换为字符串：

```sql
--  标准 SQL
SELECT
cast(id AS varchar(5))||': '||givenname||' '
||familyname||' - '||cast(dob as varchar(12))
AS info
FROM customers;
--  MSSQL：
SELECT
cast(id AS varchar(5))+': '+givenname+' '
+familyname+' - '+cast(dob as varchar(12))
AS info
FROM customers;
```

`cast()` 函数返回一个等效值，在本例中是长度最多为 5 或 12 个字符的字符串。

您应该看到与之前相同的结果，但现在转换操作是明确指定的。对于 `VARCHAR` 类型，您需要稍加注意。如您所见，您指定了一个最大长度。如果这个长度不够，您很可能会看到字符串被截断。如果您不确定应该分配多少长度，保守估计总是安全的。

这里，`NULL` 值又回来困扰我们了。虽然 `id`、`givenname` 和 `familyname` 都有值，但 `dob` 可能没有。Oracle 是唯一一个会礼貌地返回空字符串的数据库，而其他所有系统都会因为遇到 `NULL` 而“崩溃”。

由于缺少 `dob` 值应该不是什么大问题，您可以将其（以及它前面的连字符）合并为一个空字符串：

```sql
--  标准 SQL
SELECT
cast(id AS varchar(5))||': '||givenname||' '||familyname
||coalesce(' - '||cast(dob as varchar(12)),'') AS info
FROM customers;
--  MSSQL：
SELECT
cast(id AS varchar(5))+': '+givenname+' '+familyname
+coalesce(' - '+cast(dob as varchar(12)),'') AS info
FROM customers;
```

这类似于您之前对艺术家姓名的处理：

| info |
| --- |
| 474: Judy Free - 1978-04-01 |
| 186: Ray Gunn |
| 144: Ray King |
| 179: Ivan Inkling |
| 475: Drew Blood - 1989-12-06 |
| 523: Seymour Sights - 1965-01-06 |
| ~ 304 rows ~ |

虽然转换为字符串类型通常可以自动处理，但*从*字符串类型转换则可能是个挑战。下面的语句应该能成功执行：

```sql
SELECT cast('20 Jul 1969' as date) AS moon_landing;
```

但是，如果您尝试使用不同的日期格式，Oracle 会让事情变得困难；其他数据库应该没问题。

另一方面，别指望这个能工作：

```sql
SELECT cast('tomorrow' as date) AS birthday;
```

有些编程语言（如 PHP）实际上会解析这种字符串，但对于 SQL，只有 PostgreSQL 支持这种用法。甚至不要尝试使用其他类型的字符串。您很可能会得到一个错误。

## 转换日期字面量

您可能需要使用 `cast()` 的一个地方是尝试指定日期字面量时。因为日期字面量使用单引号，数据库管理系统可能会混淆它到底应该被视为字符串。

例如，当上下文很明显时，例如与已知日期类型进行比较时使用日期字面量，您无需担心：

```sql
SELECT * FROM customers WHERE dob<'1980-01-01';
```

因为 `dob` 已知是日期类型，所以字面量也必须是一个日期。

然而，在之前的许多例子中，我们在没有已知日期类型的情况下使用了日期字面量。在那里，我们必须使用 `cast()` 来强制处理：

```sql
SELECT cast('20 Jul 1969' as date) AS moon_landing;
```

如果您不确定一个字面量将如何被解释，无论如何都对其进行转换总是安全的。

