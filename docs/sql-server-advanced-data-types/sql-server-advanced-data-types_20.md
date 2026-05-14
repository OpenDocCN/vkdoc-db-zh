# 第 2 章 理解 XML

XML 的初学者常常对**格式良好的 XML 文档**和**有效的 XML 文档**之间的区别感到困惑。即使一个 XML 文档是格式良好的，除非其组成部分符合关联模式中提供的详细信息，否则它并不被视为有效的。XML 模式的原始形式被称为`DTD`（文档类型定义），但现在被称为`XSD`（XML 模式定义）。`SQL Server`不支持`DTD`模式，因此本章的重点将放在理解`XSD`模式上。

## 理解 XSD 模式

XML 文档的格式可以由`XSD`模式定义。`XSD`模式将定义文档的结构，包括数据类型、是否允许复杂类型（复合元素），以及元素在文档中必须出现（或被限制出现）的次数。它还定义了元素的顺序以及元素是否是必需的。`XSD`模式的主要组成部分如下：

- 元素声明，定义元素的属性：
  - 元素名称
  - 元素默认值
  - 元素类型
  - 元素完整性约束
- 属性声明，定义属性的属性：
  - 属性名称
  - 属性默认值
  - 属性类型
  - 属性约束
- 简单类型和复杂类型
- 模型组和属性组定义
- 元素粒子和属性使用

如果一个元素绑定到原始数据类型并且不包含子元素或属性，它就是一个简单类型。如果一个元素具有子元素、属性或其他特殊属性（例如绑定到有序序列），则必须将其定义为复杂类型。

模型组和属性组是可以在多个类型定义中重用的命名节点组。元素粒子和属性使用定义了节点的复杂属性。对于属性，这可能包括节点的可选性。对于元素，这还可能包括节点的最小和最大出现次数。

列表 2-8 展示了列表 2-7 中格式良好 XML 文档的模式声明。

***列表 2-8.*** XSD 模式

```xml
<xs:schema attributeFormDefault="unqualified"
elementFormDefault="qualified" xmlns:xs="http://www.
w3.org/2001/XMLSchema">

<xs:element name="SalesOrders">

<xs:complexType>

<xs:sequence>

<xs:element name="SalesOrder" maxOccurs="unbounded"
minOccurs="0">

<xs:complexType>

<xs:sequence>

<xs:element name="LineItem" maxOccurs="unbounded"
minOccurs="0">

<xs:complexType>

<xs:simpleContent>

<xs:extension base="xs:string">

<xs:attribute type="xs:short"
name="StockItemID" use="optional"/>

<xs:attribute type="xs:byte"
name="Quantity" use="optional"/>

<xs:attribute type="xs:float"
name="UnitPrice" use="optional"/>

</xs:extension>

Chapter 2 Understanding XML

</xs:simpleContent>

</xs:complexType>

</xs:element>

</xs:sequence>

<xs:attribute type="xs:date" name="OrderDate"
use="optional"/>

<xs:attribute type="xs:byte" name="CustomerID"
use="optional"/>

<xs:attribute type="xs:short" name="OrderID"
use="optional"/>

</xs:complexType>

</xs:element>

</xs:sequence>

</xs:complexType>

</xs:element>

</xs:schema>
```

#### 提示

有一些免费的在线工具可以根据 XML 文档创建`Xsd`模式。例如，[`FreeFormatter.com`](http://freeformatter.com)提供了一个`Xsd`生成工具，可以在[`www.freeformatter.com/xsd-generator.html.`](http://www.freeformatter.com/xsd-generator.html)找到。或者，Visual Studio 也可以自动生成模式。

XML 文档必须是格式良好的才能绑定到模式。如果一个 XML 文档绑定到了模式，则被称为有效的或类型化的 XML。`SQL Server`通过 XML 模式集合支持使用`XSD`模式，这将在第 4 章中讨论。

## SQL Server 中的 XML 使用场景

在`SQL Server`中处理 XML 的能力在许多不同情况下都非常有用。例如，想象一下你有一个中间件



SQL Server 数据库提供了强大的 XML 数据处理能力。从网站提取销售订单的应用程序可将其推送到 SQL Server 中。数据库的原生工具允许开发人员使用 `XQuery Nodes` 方法或 `OPENXML()` 函数将 XML 销售订单解析为关系表结构。反之，若要将销售订单传递给需要 XML 文档的中间件系统，你可以运行查询以检索关系数据，并附加 `FOR XML` 子句将数据转换为 XML。

SQL Server 对架构验证的支持也意味着，应用程序可以在通过网络发送数据之前，根据 SQL Server 开发人员提供的架构验证其数据。SQL Server 也可以在将文档传递给应用程序之前，根据 XML 架构验证文档。本质上，你可以在应用程序的不同层级之间创建一个数据契约，节省时间并实现更好的错误处理。

在另一个将数据存储为 XML 的用例中，假设你有一个包含详细产品定义和描述的 XML 文档。虽然你可以将其作为文件维护在文件系统中，但将其存储在 SQL Server 中允许你查询 XML 文档并将结果与关系信息连接起来。在 SQL Server 中可以使用 `XQuery` 查询 XML 文档。

