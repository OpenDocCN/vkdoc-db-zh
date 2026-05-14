# 3-10. 在批量加载中混合使用 XML 元素和属性

确实，您不仅限于仅使用 XML 元素或属性，而是可以在 XML 批量加载中混合使用这两者，如下例所示：

XML 模式如下（存储为 `C:\SQL2012DIRecipes\CH03\SQLXMLBulkLoadImportElementAndAttribute.xsd`）。目标表是之前使用过的 `Client_XMLBulkLoad`：

```xsd
<xsd:schema xmlns:xsd = "http://www.w3.org/2001/XMLSchema"
            xmlns:sql = "urn:schemas-microsoft-com:mapping-schema">
    <xsd:element name = "Invoice" sql:relation = "Client_XMLBulkLoad" >
        <xsd:complexType>
            <xsd:sequence>
                <xsd:element name = "ClientName" type = "xsd:integer" sql:field = "ClientName"/>
                <xsd:element name = "County" type = "xsd:integer " sql:field = "County"/>
            </xsd:sequence>
            <xsd:attribute name = "ID" type = "xsd:integer" />
        </xsd:complexType>
    </xsd:element>
</xsd:schema>
```

以下是数据文件（存储为 `C:\SQL2012DIRecipes\CH03\SQLXMLSourceDataElementAndAttribute.xml`）：

```xml
<?xml version = "1.0" encoding = "UTF-8" ?>
<Invoice ID = "3">
    <ClientName > John Smith</ClientName>
    <County > Staffs</County>
</Invoice>
```

要运行此示例，您需要修改 `.vbs` 脚本以引用新的 `.xml` 和 `.xsd` 文件（`objBL.Execute "C:\SQL2012DIRecipes\CH03\SQLXMLBulkLoadImportElementAndAttribute.xsd", "C:\SQL2012DIRecipes\CH03\SQLXMLSourceDataElementAndAttribute.xml"`）。运行该脚本将把数据加载到目标表中，同时保持关系完整性。

![image](img/sq.jpg)

**注意** 在保持引用完整性的同时加载表，可能比在没有引用完整性要求的多表加载慢得多。

## 3-11. 克服 XML 文件批量加载的挑战

### 问题

您希望利用 SQLXML 批量加载器的所有功能，以确保快速、无差错地批量加载 XML 数据。

### 解决方案

应用相关的 SQLXML 批量加载选项来加速加载和/或解决潜在的加载问题。

可用选项见 表 3-2。

**表 3-2. SQLXML 批量加载选项**

| 参数 | 说明 | 示例 |
| --- | --- | --- |
| `BulkLoad` | 不会加载任何数据，但将生成表结构（假设指定了 `SchemaGen`）。 | =False |
| `CheckConstraints` | 确保插入表中的数据遵守为表指定的所有约束（例如主键和外键约束）。如果此参数设置为 True 且违反了约束，过程将失败。 | =False |
| `ForceTableLock` | 此属性应用表锁，可减少加载时间。但是，如果无法获取表锁，过程将失败。 | =True |
| `IgnoreDuplicateKeys` | 为每一行插入发出一个 `COMMIT`（需要将 `Transaction` 属性设置为 False）。请注意，这非常慢！ | =False |
| `KeepIdentity` | 指定使用源 XML 中的 `IDENTITY` 值。SQL Server 在加载过程中不会生成 `IDENTITY` 值。 | =True |
| `KeepNulls` | 源数据中的任何 `NULLS` 将被插入到目标表中。 | =True |
| `SchemaGen` | 将此属性设置为 True 将创建目标表。XML 模式文件中定义的映射将作为表结构的基础。表必须不存在，否则整个过程将失败。 | =True |
| `SGDropTables` | 将此属性设置为 True 将删除任何现有的目标表——前提是不存在外键约束等情况。 | =False |
| `TempFilePath` | 指定如果过程在事务中时使用的临时文件路径。这允许您指定不同的磁盘阵列以提高速度。 | ="\\TheServer\TheShare" |
| `Transaction` | 保证操作是一个事务——即要么全部加载，要么完全失败。 | =True |
| `XMLFragment` | 此属性指定 XML 数据文件是 XML 片段（没有单一的顶级节点）。对于 XML 片段必须设置此项，否则整个过程将失败。 | =True |

### 工作原理

SQLXML 批量加载器的功能远不止前面示例中展示的那些。不利用其强大功能似乎有些可惜，这些功能包括：

*   在源数据中混合使用 XML 属性和元素。
*   使用可用的 XML 数据类型。
*   自动创建和删除目标表（以及启用或禁用约束和事务）。
*   使用溢出列来捕获未映射的数据。
*   在映射数据时应用各种模式调整。

这些选项可以添加到调用 SQLXML 批量加载器的 `.vbs` 文件中。如果使用得当，它们可以显著增强此实用程序的有效性。

重要的是要记住，SQLXML 批量加载器是一种批量加载机制，针对速度进行了优化，而速度意味着简单性——因此其参数与您可能已经从 `BCP` 或 `BULK INSERT` 中了解的参数非常相似（或者，如果您还不了解它们，请参阅 第 2 章 中的说明）。

所有参数的使用方式都与我们到目前为止在本章中看到的 `.vbs` 文件中的方式相同；即：

```
Object.Parameter = attribute
```

因此，要强制表锁，命令将是 `objBL.ForceTableLock = True`。

到目前为止使用的所有示例都假设您使用 Windows 集成安全性。考虑到情况可能并非总是如此，您需要知道替代方案是在 `ConnectionString` 参数中使用：

```
Uid = UserName; Pwd = ThePasswordTouse
```

或者

```
integrated security = SSPI
```

您可以使用 SQLXML 创建任何所需的表，然后再选择是否加载数据。例如，在我们一直使用的 VBScript 文件中，添加以下几行来创建表但不加载数据：

```
objBL.SchemaGen = True
objBL.BulkLoad = False
```

使用 SQLXML 批量加载器创建表有点粗糙，但在刚开始进行 XML 批量加载过程时可以节省大量时间。您以后总可以微调表。事实上，这正是许多专家建议的做法。

尽管速度会更慢，但 SQLXML 在加载时可以使用事务以确保在失败时回滚。这需要在 VBScript 文件中添加以下几行：

```
objBL.Transaction = True
objBL.TempFilePath = \\Server\Share
```

在事务中批量加载数据要慢得多——对于大型导入，您必须确保指定临时文件的驱动器上有足够的磁盘空间。这需要 `TempFilePath` 参数。关于 `TempFilePath` 参数，请记住，临时文件路径必须是一个共享位置，目标 SQL Server 实例的服务帐户以及运行批量加载应用程序的帐户都可以访问。这意味着，除非您在本地服务器上进行批量加载，否则临时文件路径必须是 UNC 路径（例如 `\\servername\sharename`）。如果速度至关重要，那么将文件加载到空表或分区中（如果出错可以截断）可能是更快的解决方案。

另一个需要考虑的情况是，如果源 XML 中的数据比预期的多，并且这些数据未被映射，会发生什么？除非 SQLXML 批量加载器被告知如何处理这些“溢出”数据，否则它会简单地忽略它们。由于这在实践中可能很危险，这里有一个简单的技术来捕获溢出数据，然后您可以单独处理它们——或者将其作为指示器，表明需要对模式文件进行更多工作。

1.  对于模式文件中的每个表映射，指定哪个列将处理任何（及所有）溢出。



