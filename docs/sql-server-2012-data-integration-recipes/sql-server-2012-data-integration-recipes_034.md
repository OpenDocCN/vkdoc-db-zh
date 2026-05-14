# 2-5. 使用 T-SQL 导入文本文件

## 问题

你希望使用 T-SQL 导入文本文件，因为你正在构建一个脚本化的 ETL 解决方案。

## 解决方案

使用 `OPENROWSET` 导入文本文件。

在以下示例中，我将使用 `C:\SQL2012DIRecipes\CH02\Invoices.Txt` 文件作为源数据。你可以使用以下代码片段仅使用 T-SQL 加载数据（完整代码位于 `C:\SQL2012DIRecipes\CH02\OpenrowsetLoad.sql`）：

```sql
SELECT
    CAST(BILLNO AS VARCHAR(5)) AS BILLNO
INTO
    MyTextImport
FROM
    OPENROWSET('MSDASQL',
        'Driver = {Microsoft Text Driver (*.txt; *.csv)}; DefaultDir = C:\SQL 2012 DI Recipes\CH02;',
        'SELECT INVOICENUMBER AS BILLNO FROM INVOICES.TXT WHERE CLIENTID = 1');
```

如果路径中有空格，则需要使用 `DefaultDir` 参数将路径与实际文件名分开，如下所示：

```sql
SELECT
    InvoiceNumber
INTO
    MyTextImport2
FROM
    OPENROWSET('MSDASQL',
        'Driver = {Microsoft Text Driver (*.txt; *.csv)}; DefaultDir = C:\SQL2012DIRecipes\CH02;',
        'select * from Invoices.txt');
```

而文件名中的空格要求将文件名用方括号括起来才能正常工作：

```sql
SELECT
    InvoiceNumber
INTO
    MyTextImport3
FROM
    OPENROWSET('MSDASQL',
        'Driver = {Microsoft Text Driver (*.txt; *.csv)}; DefaultDir = C:\SQL2012DIRecipes\CH02;',
        'SELECT * FROM [In voices.txt]');
```

如果你愿意（这是另一种说法：“如果你遇到 Microsoft 文本驱动程序的问题，请尝试将其作为第一个备选方案”），你可以使用 MSDASQL 和 Jet 文本驱动程序，像这样：

```sql
SELECT
    InvoiceNumber
INTO
    MyTextImport4
FROM
    OPENROWSET('MSDASQL',
        'Driver = {Microsoft Access Text Driver (*.txt, *.csv)};',
        'SELECT * FROM C:\SQL2012DIRecipes\CH02\Invoices.txt');
```

请注意，在这种情况下 `(*.txt`, `*.csv)` 是用逗号分隔的。同样值得注意的是，你可以直接使用 ACE OLEDB 驱动程序（在“配方 1-1”中描述），而不需要像 MSDASQL 驱动程序那样运行 ODBC over OLEDB，如下所示：

```sql
SELECT
    InvoiceNumber
INTO
    MyTextImport5
FROM
    OPENROWSET('Microsoft.ACE.OLEDB.12.0',
        'Text;Database = C:\SQL2012DIRecipes\CH02;HDR = Yes',
        'SELECT * FROM Invoices.txt');
```

## 工作原理

有时，当你想要导入文本文件时，SSIS 要么大材小用，要么根本就不是你首选的解决方案。这就是 `OPENROWSET` 的用武之地。它既允许你查询源数据文件，又返回一个行集，然后可以将其用作子查询的一部分、在 `JOIN` 语句中或在公用表表达式（CTE）中插入数据。


