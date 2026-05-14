# 使用 `FOR XML PATH` 生成 XML

```sql
SELECT ClientName, C.Country
FROM dbo.Client C
FOR XML PATH('Client'), ROOT('CarSales');
```
结果将会是：
```xml
<CarSales>
<Client>
<ID> 3</ID>
<ClientName> John Smith</ClientName>
<Country> 1</Country>
</Client>
```
（为节省空间，省略了部分记录）
```xml
<Client>
<ID> 7</ID>
<ClientName> Slow Sid</ClientName>
<Country> 3</Country>
</Client>
</CarSales>
```

## 定义属性
如果你更倾向于以属性为中心的 XML，那么你可以通过这种方式使用别名来塑造它 (`C:\SQL2012DIRecipes\CH07\AttributeCentricXML.sql`)：
```sql
SELECT C.ID '@ID',
       C.ClientName '@ClientName',
       C.Country '@Country'
FROM dbo.Client C
FOR XML PATH('Client'), ROOT('CarSales');
```
你应该会得到：
```xml
<CarSales>
<Client ID="3" ClientName="John Smith" Country="1"/>
<Client ID="4" ClientName="Bauhaus Motors" Country="2"/>
<Client ID="5" ClientName="Honest Fred" Country="3"/>
<Client ID="6" ClientName="Fast Eddie" Country="2"/>
<Client ID="7" ClientName="Slow Sid" Country="3"/>
</CarSales>
```

## 输出更复杂的嵌套 XML
最后，为了让你走上生成更复杂输出的道路，你可以使用如下代码通过别名字段来创建嵌套 XML (`C:\SQL2012DIRecipes\CH07\ComplexNestedXML.sql`)：
```sql
SELECT C.ID,
       C.ClientName,
       C.Country,
       C.Town,
       I.TotalDiscount AS 'Invoice/@TotalDiscount',
       I.InvoiceNumber AS 'Invoice/InvoiceNumber'
FROM dbo.Client C
INNER JOIN dbo.Invoice I ON C.ID = I.ClientID
FOR XML PATH('Client'), ROOT('CarSales');
```
这个查询的输出将是
```xml
<CarSales>
<Client>
<ID> 3</ID>
<ClientName> John Smith</ClientName>
<Country> 1</Country>
<Town> Uttoxeter</Town>
<Invoice TotalDiscount="500.00">
<InvoiceNumber> 3A9271EA-FC76-4281-A1ED-714060ADBA30</InvoiceNumber>
</Invoice>
</Client>
<Client>
<ID> 4</ID>
<ClientName> Bauhaus Motors</ClientName>
<Country> 2</Country>
<Town> Oxford</Town>
<Invoice TotalDiscount="100.00">
<InvoiceNumber> C9018CC1-AE67-483B-B1B7-CF404C296F0B</InvoiceNumber>
</Invoice>
</Client>
</CarSales>
```
显然，所示的输出只是最终结果的一小部分。

## 工作原理
在前面的一些方法中，我使用了 `FOR XML PATH()` 来“塑造”输出的 XML。现在，不求面面俱到，这里是一个关于使用 `FOR XML PATH()` 来交付你想要导出的 XML 的速成课程。
需要注意的主要关键字是 `AUTO`，它用于使用源表来定义 XML 层次结构。你需要记住 `SELECT` 子句中列的顺序很重要，因为这会影响 XML 元素的嵌套。
希望这能说明：
*   你通过使用从顶层元素开始的 XML 节点树的完整路径为它们设置别名来定义嵌套元素。
*   你通过在属性名前加上 at 符号 (`@`)，并使用从顶层元素开始的 XML 节点树的完整路径，来创建属性而非元素。
*   在定义任何低级别元素之前，你必须先为节点定义属性。
在定义复杂的多级 XML 模式时，你需要注意拼写错误。很容易就会创建出并行的层级。同时记住模式是区分大小写的。在讨论这一点时，值得注意的是：
*   在 `SELECT` 子句中，必须先从 `Client` 表选择字段，然后再选择 `Invoice` 表的字段，以获得 XML 中生成的层次结构，其中 invoice 元素嵌套在 Client 数据内部。
*   表别名提供元素名称。
*   列别名提供元素名称。
在输出 XML 时，可以使用 表 7-8 中显示的选项。

### 表 7-8. `FOR XML PATH()` 选项
| 选项 | 描述 |
| --- | --- |
| `TYPE` | 将结果作为 SQL Server 的 XML 数据类型输出，而不是 `NVARCHAR(MAX)`。 |
| `ELEMENTS` | 确保使用 XML 元素，而不是（默认的）属性。 |
| `XSINIL` | 强制输出空元素，而不是完全跳过。 |
| `ROOT` | 添加一个你选择的根元素。 |

还有一个有点令人畏惧的 `FOR XML EXPLICIT` 命令，当你需要输出的 XML 无法使用 `PATH` 或 `AUTO` 创建时，你可以使用它。`XML Explicit` 超出了这些方法的范围。我留给你自己通过其他来源（如联机丛书和互联网）去研究这个选项。

