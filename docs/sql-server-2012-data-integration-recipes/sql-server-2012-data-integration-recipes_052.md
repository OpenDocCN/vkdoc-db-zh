# 使用 SQLXML Bulk Load 加载具有关系数据的 XML 文件

您可以使用 SQLXML Bulk Load 可执行文件来加载存储在 XML 文件中的关系型表。

1.  删除前一个配方中使用的两个表，并使用外键约束重新创建它们。此操作需要以下 DDL（存储为`C:\SQL2012DIRecipes\CH03\LoadRelationalXML.Sql`）：
    ```sql
    DROP TABLE CarSales_Staging.dbo.Invoice_XML_Multi;
    DROP TABLE CarSales_Staging.dbo.Invoice_Lines_XML_Multi;
    CREATE TABLE CarSales_Staging.dbo.Invoice_XML_Multi
    (
     ID INT NOT NULL PRIMARY KEY,
     InvoiceNumber VARCHAR (50) NULL,
     DeliveryCharge SMALLMONEY NULL
    ) ;
    GO
    CREATE TABLE CarSales_Staging.dbo.Invoice_Lines_XML_Multi
    (
     ID INT NOT NULL PRIMARY KEY,
     InvoiceID INT NOT NULL FOREIGN KEY REFERENCES Invoice_XML_Multi (ID),
     SalePrice MONEY NULL,
    ) ;
    GO
    ```

2.  创建一个合适的 XML 模式（以下内容存储为`C:\SQL2012DIRecipes\CH03\SQLXMLBulkLoadImportReferential.xsd`）：
    ```xsd
    <xsd:schema xmlns:xsd = " http://www.w3.org/2001/XMLSchema"
                xmlns:sql = "urn:schemas-microsoft-com:mapping-schema">
        <xsd:annotation>
            <xsd:appinfo>
                <sql:relationship name = "InvoiceToLine"
                    parent=     "Invoice_XML_Multi"
                    parent-key = "ID"
                    child=      "Invoice_Lines_XML_Multi"
                    child-key=  "InvoiceID" />
            </xsd:appinfo>
        </xsd:annotation>
        <xsd:element name = "Invoice" sql:relation = "Invoice_XML_Multi" >
            <xsd:complexType>
                <xsd:sequence>
                    <xsd:element name = "ID"             type = "xsd:integer" sql:field = "ID" />
                    <xsd:element name = "InvoiceNumber"  type = "xsd:string"  sql:field = "InvoiceNumber" />
                    <xsd:element name = "DeliveryCharge" type = "xsd:decimal" sql:field = "DeliveryCharge" />
                    <xsd:element name = "Invoice_Lines"
                                 sql:relation = "Invoice_Lines_XML_Multi"
                                 sql:relationship = "InvoiceToLine" >
                        <xsd:complexType>
                            <xsd:sequence>
                                <xsd:element name = "ID"        type = "xsd:integer" sql:field = "ID" />
                                <xsd:element name = "InvoiceID" type = "xsd:integer" sql:field = "InvoiceID" />
                                <xsd:element name = "SalePrice" type = "xsd:string"  sql:field = "SalePrice" />
                            </xsd:sequence>
                        </xsd:complexType>
                    </xsd:element>
                </xsd:sequence>
            </xsd:complexType>
        </xsd:element>
    </xsd:schema>
    ```

3.  定位您的数据文件（此处存储为`C:\SQL2012DIRecipes\CH03\SQLXMLSourceDataRelatedTables.xml`）：
    ```xml
    <?xml version = "1.0" encoding = "UTF-8" ?>
    <ROOT>
        <Invoice>
            <ID > 3</ID>
            <InvoiceNumber > 3A9271EA-FC76-4281-A1ED-714060ADBA30</InvoiceNumber>
            <DeliveryCharge > 250</DeliveryCharge>
            <Invoice_Lines>
                <ID > 1</ID>
                <InvoiceID > 3</InvoiceID>
                <SalePrice > 5000</SalePrice>
            </Invoice_Lines>
            <Invoice_Lines>
                <ID > 2</ID>
                <InvoiceID > 5</InvoiceID>
                <SalePrice > 12500</SalePrice>
            </Invoice_Lines>
        </Invoice>
    </ROOT>
    ```

4.  修改 `.vbs` 脚本，通过修改 `objBL.Execute` 语句来引用新的 `.xml` 和 `.xsd` 文件，然后运行此示例：
    ```vbscript
    objBL.Execute "C:\SQL2012DIRecipes\CH03\SQLXMLBulkLoadImportReferential.xsd", _
    "C:\SQL2012DIRecipes\CH03\SQLXMLSourceDataRelatedTables.xml"
    ```

## 工作原理

本配方的示例建立在前一个配方的方法之上。但是，它需要满足以下先决条件：
*   具有从主键到外键关系的目标表。这就是我们必须重新创建目标表的原因。
*   源数据中“父-子”数据关系在 XML 结构中直接表达。在源数据中就是这种情况，其中 `<Invoice>` 元素包含一个或多个 `<Invoice_Lines>` 元素。

更一般地说，如果源 XML 是层次结构化的，以映射到关系源，那么 `SQLXML Bulk Load` 可以将数据分解到 SQL Server 表中，同时维护外键关系。在这里，与前一个配方一样，任何潜在的困难都在于 XSD 文件的定义和创建。实际上，所有重要的内容都在模式文件中。我将解释其要点。

首先，注意模式文件开头的 `<xsd:annotation>` 和 `<xsd:appinfo>` 元素。外键映射在这些元素内指定。它必须存在并正确定义，加载才能工作。

然后，在这些元素内部，有 `sql:relationship name = "InvoiceToLine"` 元素。它为每个表之间的关系命名或标识（如果您愿意），作为 `appinfo` 的一部分，随后它将被用作模式文件中“表”的 `sql:element` 定义的一部分。关系定义如下：
*   `parent = "Invoice_XML_Multi"`：“父”表。
*   `parent-key = "ID"`：“父”表的主键列。
*   `child = "Invoice_Lines_XML_Multi"`：“子”表。
*   `child-key = "InvoiceID"`：“子”表的外键列。
*   `sql:relationship = "InvoiceToLine"`：在包含子元素的 XML 元素内部，此属性指定要使用的关系。

从这些信息中可以看出，`ID` 字段现在被视为 `Invoice_XML_Multi` 表中的主键，而 `InvoiceID` 被视为 `Invoice_Lines_XML_Multi` 表中的外键。执行 `.vbs` 脚本会将数据加载到两个表中，同时将 `ID` 保持为 `Invoice_Lines_XML_Multi` 的外键约束。

如果您有一个定义了两个表之间主键/外键关系的映射模式（例如 `Invoice` 和 `Invoice_Lines` 之间），则**必须**先在模式中描述具有主键的表。具有外键列的表必须出现在模式的后面部分。

您还需要知道，`SQLXML Bulk Load` 在节点进入作用域时生成记录，并在节点退出作用域时将这些记录发送到 SQL Server。这意味着记录的数据必须存在于节点的作用域内。因此，“子”表的数据不能出现在“父”表的数据之后，就像加载多个（独立的）表时的情况一样。

到目前为止，在所有使用的 `SQLXML Bulk Load` 示例中，我们都在源数据中使用了以元素为中心的 XML。因为我不想给人留下这是唯一可用选项的印象，我想明确指出，您当然也可以使用以属性为中心的 XML——或者实际上是两者的混合。

所以，总结一下，到目前为止您看到的是：
```xsd
<xsd:element name = "InvoiceID" type = "xsd:integer" sql:field = "InvoiceID" />
```
它同样可以是：
```xsd
<xsd:attribute name = "InvoiceID" type = "xsd:integer" sql:field = "InvoiceID" />
```

我没有给出一个仅使用 XML 属性的完整示例，但这里提供的模式文件应该足以让您创建一个源数据完全是基于属性的模式文件。无论如何，只要映射是准确的，您的数据就会被加载。


