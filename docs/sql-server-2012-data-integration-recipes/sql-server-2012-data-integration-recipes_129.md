# 10-4. 从文本文件进行源数据剖析（续）

```sql
txt')  SELECT  @Registration_Year_NULL = SUM(Registration_Year)  ,@Cost_Price_NULL = SUM(Cost_Price) FROM    #NullSourceRecords  PRINT    @ROWCOUNT PRINT    @Mileage_MAX PRINT    @Mileage_MIN PRINT    @Registration_Year_NULL PRINT    @Cost_Price_NULL PRINT    CAST(@Registration_Year_NULL AS NUMERIC (12,6))               / CAST(@ROWCOUNT AS NUMERIC (12,6))
```

## 工作原理

读完 10-4 节后（或者如果你正在使用自己的大型文本文件作为数据源，你很快就会发现），你可能已经猜到，每次希望剖析某一列时都重新读取整个文件，是一种非常冗长繁琐的数据剖析方法。因此，如果你要剖析多个列，我建议尽可能通过分组剖析数据来最小化数据读取的次数，这样每次读取源文件时就能获取尽可能多的剖析元素。

当前小节中的代码片段只读取了源文本文件两次——且仅两次。第一次遍历文件是为了查找两列（`Registration_Year` 和 `Cost_Price`）中的 `NULL` 值。它创建一个临时表，其中包含一个狭窄的数据集，仅包含 1 或 0，用以指示每列是否存在 `NULL` 值。然后对这些列进行求和，以得出相关列的 `NULL` 值总数。第二次解析源文件时不应用 `WHERE` 子句——并返回记录数以及所需列的最大值和最小值。之后就可以进行任何百分比计算。

#### 提示、技巧与陷阱

*   关于与文本文件一起使用 `OPENROWSET` 的更详细讨论，请参阅 2-5 节。具体来说，请记住 `OPENROWSET` 是为临时、偶尔的连接而构建的，对于更常规的连接，链接服务器是推荐的解决方案。
*   对于链接服务器，你可以使用类似以下的 SQL 来简化操作（其中 `MyOracleDatabase` 是链接服务器名称）。请记住使用四部分名称来正确引用表：
    ```sql
    SELECT COUNT(*)
    FROM MyOracleDatabase..HR.EMPLOYEES
    WHERE LAST_NAME IS NULL
    ```
*   如果你的源数据文件不包含列名，剖析外部数据可能会更费力。在这种情况下，你需要一个 `Schema.Ini` 文件，如 2-6 节所述。以下是一个数据源文件 (`C:\SQL2012DIRecipes\CH10\StockNoHeaders.Txt`) 的 `Schema.Ini` 文件 (`C:\SQL2012DIRecipes\CH10\Schema.Ini`)，该文件不包含标题行：
    ```ini
    [StockNoHeaders.txt]
    Format=CSVDelimited
    ColNameHeader=False
    MaxScanRows=0
    Col1=MAKE Long
    Col2=MARQUE long Width 20
    Col3=MODEL Text Width 50
    Col4=PRODUCT_TYPE Text Width 15
    Col5=REGISTRATION_YEAR Text Width 4
    Col6=MILEAGE Long
    Col7=COST_PRICE Long
    CharacterSet=ANSI
    ```
    运行本小节中的代码——并使用传递查询 `SELECT * FROM StockNoHeaders.txt`——将剖析一个不包含标题行的文本文件中的源数据。

