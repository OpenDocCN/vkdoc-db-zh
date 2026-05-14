# 10-3. 配置外部数据

## 问题
您希望配置尚未存储在 SQL Server 中的数据。

## 解决方案
使用 T-SQL 和 `OPENROWSET` 通过 OLEDB 访问源数据。

如果您要查看文本文件或 OLEDB 数据源中的数据，则可以使用 `OPENROWSET` 返回 `NULL` 计数（在此示例中，使用示例文件 `C:\SQL2012DIRecipes\CH10\Stock.Txt`）：
```sql
SELECT COUNT(*)
FROM OPENROWSET('MSDASQL',
    'Driver={Microsoft Access Text Driver (*.txt, *.csv)};
     DefaultDir= C:\SQL2012DIRecipes\CH10;',
    'select * from Stock.Txt')
WHERE Model IS NULL;
```

同样，可以使用以下 T-SQL 计算 `NULL` 的百分比：
```sql
SELECT
    (SELECT CAST(COUNT(*) AS NUMERIC (10,3))
     FROM OPENROWSET('MSDASQL',
         'Driver={Microsoft Access Text Driver (*.txt, *.csv)};
          DefaultDir= C:\SQL2012DIRecipes\CH10;',
         'SELECT * FROM Stock.txt WHERE Model IS NULL'))
    /
    (SELECT COUNT(*)
     FROM OPENROWSET('MSDASQL',
         'Driver={Microsoft Access Text Driver (*.txt, *.csv)};
          DefaultDir= C:\SQL2012DIRecipes\CH10;',
         'SELECT * FROM Stock.Txt'));
```

代码片段位于文件 `C:\SQL2012DIRecipes\CH10\ProfileExternalData.Sql` 中。

## 工作原理
配方 10-1 和 10-2 中描述的所有用于执行域和数据分析的技术都可以与 `OPENROWSET` 一起使用。这里的技巧是连接到“外部”（我指的是非 SQL Server）数据源。一旦使用必需的驱动程序完成此操作，您就可以在直通查询中使用适当的 SQL 代码片段。用于分析数据的函数与前两个配方中使用的函数相同。

`OPENROWSET` 可以访问的不仅仅是文本文件。然而，由于使用此命令从 Microsoft Access、Excel 和各种 RDBMS 读取数据的技术分别在 第 1 章 和 第 4 章 中介绍，有关实际外部连接的更多信息，请参阅这些章节。分析代码仍将与此处所示的相同。

> **注意** 在本配方的示例中，我使用 ACE 驱动程序读取文本文件，因为这允许代码在 32 位和 64 位环境中运行。为此，您必须按照配方 1-1 中的说明安装 ACE 驱动程序。如果您在 32 位环境中，则可以使用 Microsoft Text Driver，并将代码中的“Microsoft Access Text Driver (`*.txt`, `*.csv`)”替换为“Microsoft Text Driver (`*.txt`; `*.csv`)”。

