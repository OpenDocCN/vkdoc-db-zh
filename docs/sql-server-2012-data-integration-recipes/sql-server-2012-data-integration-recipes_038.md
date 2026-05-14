# 2-7. 使用 T-SQL 导入数据以备使用链接服务器

## 问题

您最终希望使用链接服务器来导入文本文件。在设置完成之前，您希望为（最终的）链接服务器编写 T-SQL 代码，以便在链接服务器设置好后无需重写太多代码。

## 解决方案

使用 `OPENDATASOURCE`。以下代码展示了如何操作（`C:\SQL2012DIRecipes\CH02\OpendatasourceAndDestinationTable.sql`）。

1.  创建一个结构合适的目标表：
    ```sql
    CREATE TABLE Text_OpenrowsetInsert
    (
     ID
     ,InvoiceNumber INT
     ,ClientID INT
     ,TotalDiscount NUMERIC (18,2)
     ,DeliveryCharge NUMERIC (18,2)
    );
    GO
    ```
2.  运行以下代码片段，加载 `C:\SQL2012DIRecipes\CH02\Invoices.Txt` 源文件：
    ```sql
    INSERT INTO    Text_OpenrowsetInsert (ID, InvoiceNumber, ClientID, TotalDiscount, DeliveryCharge)
     SELECT         F1,F2,F3,F4,F5
     FROM           OpenDataSource('Microsoft.ACE.OLEDB.12.0',
                                  'Data Source = C:\SQL2012DIRecipes\CH02;
                                   Extended Properties = "Text;HDR = NO;"'
                                  )... Invoices#txt;
    ```

## 工作原理

如果您正在考虑使用链接服务器连接到文本文件，但想先“试试水”，那么您可能希望使用 `OPENDATASOURCE` 来读取您的平面文件。

在以下情况下，您可能希望使用 `OPENDATASOURCE`：
- 当您有一个结构一致的、带分隔符的 CSV 或文本文件时。
- 当您希望指定诸如标题行等参数，或者即使没有列名也要指定要选择的列时。
- 当源数据文件结构可靠且一致（或至少被 Microsoft 文本驱动程序认为是无错误的）时。

我尽可能使用 ACE 驱动程序配合 `OPENDATASOURCE`。这有几个很好的理由：
- 它能优雅地处理第一行没有列标题的情况，将列命名为 `F1`、`F2`…`F’n’` 等形式。
- 通常很少有 32 位/64 位问题。

对于 Jet 驱动程序（因此在 32 位环境中定义），代码是：
```sql
INSERT INTO      Text_OpenrowsetInsert (ID, InvoiceNumber, ClientID, TotalDiscount, DeliveryCharge)
SELECT           F1,F2,F3,F4,F5
FROM             OPENDATASOURCE(
                   'Microsoft.Jet.OLEDB.4.0',
                   'Data Source = C:\SQL2012DIRecipes\CH02;
                    Extended Properties = "Text;HDR = YES;"'
                  )...Invoices#txt;
```

通过将扩展属性标志设置为 `"Text;HDR = YES;"` 或 `"Text;HDR = NO;"`，您可以轻松指定是否存在标题行。

> **注意**
> 您必须在文件名中使用井号 (`#`) 而不是点号 (`.`)。

如果第一行中没有列名，ACE 驱动程序会将列命名为 `F1`、`F2` 等。然后可以使用这些名称来查询数据，如下所示。在此情况下，第一行接受 `NULLS`。
```sql
SELECT           F1
FROM             OPENDATASOURCE(
                   'Microsoft.Jet.OLEDB.4.0',
                   'Data Source = C:\SQL2012DIRecipes\CH02;
                    Extended Properties = "Text;HDR = NO;"'
                  )...Invoices#txt
```

如果您创建了系统 DSN（在配方 4-8 中描述），那么您可以将其与 `OPENDATASOURCE` 一起使用，如下所示：
```sql
SELECT          ID, InvoiceNumber, ClientID, TotalDiscount, DeliveryCharge
FROM            OPENDATASOURCE('SQLOLEDB', 'DSN = MyDSN;')...Invoices#txt
```

#### 提示、技巧与陷阱

- 在 32 位环境中，您可以使用 MS Jet 驱动程序代替 ACE 驱动程序。
- 如果文本文件不包含列标题，并且您通过 MSDASQL 使用文本驱动程序，则需要一个 `Schema.ini` 文件（在配方 2-6 中描述）。这是因为 `OPENDATASOURCE` 期望具有标题的表格结构，而 MSDASQL 驱动程序不会（不像 Jet 和 ACE）提供“虚拟”列标题。
- 值得注意的是，如果您使用 MSDASQL 访问古老的文本文件驱动程序，现在可以为旧版本的 64 位 Windows 计算机下载 64 位版本。该驱动程序包含在 Windows Server 2008 到 Vista SP1 中。您可以在 `www.microsoft.com/en-gb/download/details.aspx?id=20065` 找到它。
- 如果您确实创建了架构信息文件，那么您在那里指定的列名将被 Jet 驱动程序使用，而不是 `F1`、`F2` 等形式——甚至可以覆盖文件包含的列标题。
- 与 `OPENDATASOURCE` 和 ACE 驱动程序一起使用的 T-SQL 可以使用 `WHERE`、`ORDER BY`、`CAST`、`CONVERT` 进行扩展。您也可以为列名指定别名。

