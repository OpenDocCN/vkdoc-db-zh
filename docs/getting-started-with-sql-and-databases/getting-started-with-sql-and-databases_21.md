# Microsoft SQL Server 中的日期格式化

Microsoft SQL 中的 `format()` 函数也可用于日期。例如：

```sql
WITH vars AS (
SELECT cast('1969-07-20 20:17:40' AS datetime)
AS moonshot
)
SELECT
format(moonshot,'dddd, d MMMM yyy') AS fulldate,
format(moonshot,'ddd d MMM yyy') AS shortdate
FROM vars;
```

你可以通过以下链接了解更多关于各种日期格式代码的信息：

[`learn.microsoft.com/en-us/dotnet/standard/base-types/custom-date-and-time-format-strings`](https://learn.microsoft.com/en-us/dotnet/standard/base-types/custom-date-and-time-format-strings)

