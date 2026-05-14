# 第 4 章 XML 的查询与分解

### 模式定义

```xml
<xs:sequence>
  <xs:element name="OrderHeader">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="CustomerName" type="xs:string" />
        <xs:element name="OrderDate" type="xs:date" />
        <xs:element name="OrderID" type="xs:unsignedInt" />
      </xs:sequence>
    </xs:complexType>
  </xs:element>
  <xs:element name="OrderDetails">
    <xs:complexType>
      <xs:sequence>
        <xs:element maxOccurs="unbounded" name="Product">
          <xs:complexType>
            <xs:attribute name="ProductID" type="xs:unsignedByte" use="required" />
            <xs:attribute name="ProductName" type="xs:string" use="required" />
            <xs:attribute name="Price" type="xs:decimal" use="required" />
            <xs:attribute name="Qty" type="xs:unsignedShort" use="required" />
          </xs:complexType>
        </xs:element>
      </xs:sequence>
    </xs:complexType>
  </xs:element>
</xs:sequence>
</xs:complexType>
</xs:element>
</xs:sequence>
</xs:complexType>
</xs:element>
</xs:schema>
```

### 命名空间提示

**提示** 因为该模式未指定命名空间，它将与默认的空字符串命名空间关联。可以使用 `<xs:schema>` 属性 `xmlns:ns=http://your-namespace` 向模式添加命名空间。

### 将模式绑定到列

我们可以通过使用清单 4-33 中的查询，将我们的模式绑定到 `OrderSummary` 列。

***清单 4-33.*** 将模式绑定到列

```sql
ALTER TABLE Sales.CustomerOrderSummary
ALTER COLUMN OrderSummary XML(OrderSummary);
```

### 在查询中引用模式

我们在构造或查询 XML 数据时也可以引用模式。`FOR XML` 子句包含一个 `WITH XMLNAMESPACES` 选项，可用于指定结果 XML 文档的目标命名空间，并且 XQuery 方法（如 `query`）可以以 `declare namespace` 语句开头。

**提示** 关于命名空间使用的完整讨论超出了本书的范围。但是，可以在 [`docs.microsoft.com/en-us/sql/t-sql/xml/with-xmlnamespaces?view=sql-server-2017`](https://docs.microsoft.com/en-us/sql/t-sql/xml/with-xmlnamespaces?view=sql-server-2017) 找到使用 `WITH XMLNAMESPACES` 的示例，在 [`docs.microsoft.com/en-us/sql/t-sql/xml/value-method-xml-data-type?view=sql-server-2017`](https://docs.microsoft.com/en-us/sql/t-sql/xml/value-method-xml-data-type?view=sql-server-2017) 可以找到使用命名空间与 `query` 方法的示例。

## 总结

XQuery 是一种可用于查询存储在 SQL Server 列和变量中的 XML 数据的语言。`exist()` 方法可用于检查节点或包含特定值的节点是否存在。`value()` 方法可用于从 XML 文档中提取标量值，而 `query()` 方法可用于返回仍为 XML 格式的 XML 文档子集。

FLWOR 语句可用于帮助导航和迭代 XML 文档。`for` 语句将变量绑定到输入序列。`let` 语句用于将 XQuery 表达式赋值给变量。`where` 语句可用于筛选返回的结果。`order by` 语句可选择性地用于对 FLWOR 语句的结果进行排序。`return` 语句指定将返回什么数据。

T-SQL 变量和列可以传递到 XQuery 语句中。当采用这种技术时，它被称为**跨域查询**。它允许轻松地将值绑定到 XQuery 语句，有助于简化逻辑并减少重复代码。

XML 数据可以使用 `modify()` 方法进行修改。此方法允许开发人员使用以下三个选项之一：`insert`、`replace value of` 或 `delete`。插入数据时，开发人员还可以利用更多选项，以便对插入发生的位置进行精细控制。例如，你可以选择在第一个、最后一个、某个节点之后或某个节点之前插入。

XML 数据可以转换为关系数据（即，分解）。


可以通过 `OPENXML()` 函数或 `nodes()` XQuery 方法来实现。

当使用 `OPENXML()` 函数时，必须先解析要分解的 XML 文档。这可以通过 `sys.sp_xml:preparedocument` 系统存储过程实现。一旦文档被分解，应使用 `sys.sp_xml:removedocument` 系统存储过程将解析树从内存中移除。

