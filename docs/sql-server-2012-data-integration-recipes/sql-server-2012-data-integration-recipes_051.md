# SQLXML 批量加载使用技巧与多表加载

## 注意事项

在编写 XSD 映射文件时，需要特别注意以下几点：

-   确保不会意外关闭诸如 `xsd:sequence` 和 `xsd:complexType` 之类的元素，甚至是一些 `xsd:element` 定义。
-   严格区分大小写。因此，在前面的例子中，如果有一个名为 “Country” 的元素，而模式映射元素名为 “country”，则会导致过程失败。

这个简单的脚本假设目标表已经存在（不过，你也可以在下一部分展示的加载过程中创建表）。它也假定了使用集成安全性。有趣的是，有点反直觉的是，XML 模式文件（`.xsd`）作为第一个参数传递给 XML 批量加载器，而 XML 文件本身则作为第二个参数传递。

#### 提示、技巧与陷阱

-   此示例使用 XML 元素，但以属性为中心的 XML 源数据也可以通过 `SQLXML 批量加载器` 加载。
-   复杂的 XML 源文件可以使用 XSLT 进行“扁平化”，生成适合 `SQLXML` 处理的文件。在 SSIS 中，执行 XSLT 的 XML 任务可以放在批量加载步骤之前。
-   不支持内联模式。实际上，如果你的源 XML 文档中有内联模式，`XML 批量加载器` 将忽略它。
-   XML 文档会检查格式是否良好，但不会进行验证。如果 XML 文档格式不正确，处理将被取消。

## 3-9. 从单个 XML 源文件同时加载多个表

### 问题
你想从一个包含多个不相关表数据的 XML 源文件中加载数据。

### 解决方案
使用 `SQLXML 批量加载器` 可执行文件和一个精心制作的 `XSD` 文件。

你可以按照以下方式从单个 XML 文件加载多个表。

1.  定位你的 XML 文件，该文件包含来自多个表的数据。本例使用 `C:\SQL2012DIRecipes\CH03\SQLXMLSourceDataMultipleTables.xml`，其中包含 `Invoice` 和 `Invoice_Lines` 表的数据。内容如下：

    ```xml
    <?xml version = "1.0" encoding = "UTF-8" ?>
    <ROOT>
    <Invoice>
        <ID> 3</ID>
        <InvoiceNumber> AA/1/2014-07-25</InvoiceNumber>
        <DeliveryCharge> 250</DeliveryCharge->
    </Invoice>
    <Invoice_Lines>
        <InvoiceID> 3</InvoiceID>
        <SalePrice> 5000</SalePrice>
    </Invoice_Lines>
    <Invoice_Lines>
        <InvoiceID>3</InvoiceID>
        <SalePrice> 12500</SalePrice>
    </Invoice_Lines>
    </ROOT>
    ```

2.  在 SQL Server 中创建用于保存数据所需的表 `C:\SQL2012DIRecipes\CH03\tblInvoiceMulti.Sql`：

    ```sql
    CREATE TABLE CarSales_Staging.dbo.Invoice_XML_Multi
    (
     ID INT NULL,
     InvoiceNumber VARCHAR (50) NULL,
     DeliveryCharge SMALLMONEY NULL
    ) ;
    GO
    CREATE TABLE CarSales_Staging. dbo.Invoice_Lines_XML_Multi
    (
     InvoiceID INT NULL,
     SalePrice MONEYNULL,
    ) ;
    GO
    ```

3.  创建一个 XSD 文件（存储为 `C:\SQL2012DIRecipes\CH03\SQLXMLBulkLoadImportMultipleTables.xsd`）：

    ```xml
    <xsd:schema xmlns:xsd = " http://www.w3.org/2001/XMLSchema "
                xmlns:sql = "urn:schemas-microsoft-com:mapping-schema">
    
      <xsd:element name = "ROOT" sql:is-constant = "1" >
        <xsd:complexType>
         <xsd:sequence>
            <!-- Invoice -->
            <xsd:element name = "Invoice" sql:relation = "Invoice_XML_Multi" >
             <xsd:complexType>
              <xsd:sequence>
               <xsd:element name = "ID"              type = "xsd:integer"
                                                      sql:field = "ID" />
               <xsd:element name = "InvoiceNumber"   type = "xsd:string"
                                                      sql:field = "InvoiceNumber" />
               <xsd:element name = "DeliveryCharge"  type = "xsd:decimal"
                                                      sql:field = "DeliveryCharge" />
              </xsd:sequence>
             </xsd:complexType>
            </xsd:element>
    
            <!-- Invoice Lines -->
            <xsd:element name = "Invoice_Lines" sql:relation = "Invoice_Lines_XML_Multi" >
             <xsd:complexType>
              <xsd:sequence>
               <xsd:element name = "InvoiceID"  type = "xsd:integer" sql:field = "InvoiceID" />
               <xsd:element name = "SalePrice"  type = "xsd:string"  sql:field = "SalePrice" />
              </xsd:sequence>
             </xsd:complexType>
            </xsd:element>
         </xsd:sequence>
        </xsd:complexType>
      </xsd:element>
    </xsd:schema>
    ```

4.  创建一个 `.vbs` 脚本，引用相应的 `.XML` 和 `.XSD` 文件，如下所示：

    ```vbscript
    Set objBL = CreateObject("SQLXMLBulkLoad.SQLXMLBulkload.4.0")
    objBL.ConnectionString = "provider = SQLOLEDB;data source = ADAM02; ←database = CarSales_Staging;
    integrated security = SSPI"
    objBL.ErrorLogFile = "C:\SQL2012DIRecipes\CH03\SQLXMLBulkLoadImporterror.log"
    objBL.Execute "C:\SQL2012DIRecipes\CH03\SQLXMLBulkLoadImportMultipleTables.xsd", ←
    "C:\SQL2012DIRecipes\CH03\SQLXMLSourceDataMultipleTables.xml"
    Set objBL = Nothing
    ```

5.  双击 `.vbs` 文件以运行 `SQLXML 批量加载器` 并导入源数据。

### 工作原理
XML 文件相对于 CSV 或其他平面文件的一个优势是，能够在单个源文件中存储多个不相关的表。这实际上允许你将整个数据子集传输到一个文件中，准备同时加载到目标数据库的多个表中。如果你使用 `SQLXML 批量加载器` 来执行此过程，数据加载速度会非常快。

此技术基于源 XML 布局非常简单的前提。它假定你本质上是从 XML 文件中的“表转储”加载数据，其中每个表按顺序存储在文件中。为了模拟这种场景，这里使用的数据文件包含两个 XML 格式的表：`Invoice` 和 `Invoice_Lines`。这个示例文件只有一个 `Invoice` 记录和两个 `Invoice_Lines` 记录，但显然现实中可能有数十个表，每个表包含数百万条记录。请注意，XML 中并未假定 `Invoice` 和 `Invoice_Lines` 之间存在任何关系链接。

这种方法唯一可能的问题是，你必须定义的 XSD 文件可能会变得极其复杂。然而，创建它不一定困难，只是繁琐。所以请准备好耐心，并为一些重复的测试做好准备。

如你所见，每个映射到表的 XML 元素都是一个独立的 XML 节点，元素的顺序并不重要。然而，重要的是要有一个单一的根节点——除非 `XMLFragment` SQLXML 参数设置为 `true`（默认为 `False`），如配方 3-11 中所述。

#### 提示、技巧与陷阱

-   在加载完成后，通过添加主键和外键约束来强制执行引用完整性，没有任何东西可以阻止你这样做。实际上，这比从“层次化”XML 源加载数据（如下一个配方所述）要快得多。
-   此示例假定目标表没有 `IDENTITY` 字段。与大多数批量加载形式一样，这种情况也可以处理（稍后将解释）。
-   值得注意的是，XML 文件（数据文件）可以包含比模式（`.xsd`）文件中定义的更多的“表”和“记录”。但是，只有在 XSD 文件中指定的那些才会被加载。

## 3-10. 从 XML 源文件加载和解析关系表

### 问题
你想加载一个源 XML 文件，该文件包含描述为关系表的 XML 数据。

### 解决方案
使用 `SQLXML 批量加载器` 可执行文件和一个精心制作的 `XSD` 文件来维护源文件中假定或定义的外键关系。

