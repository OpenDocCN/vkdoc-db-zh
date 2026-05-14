# 以数据集的方式思考

既然我已经讨论了什么是数据集，我希望你能以接纳数据集的方式去思考数据。在开始编写 T-SQL 代码之前，先理解存储了什么信息、数据如何存储，以及如何选择或使用具有相似特征的数据组，这可能会为你节省时间和精力。当底层表设计得最适合数据集操作时，处理数据集也会更容易完成。

你的目标是学会以数据集的方式进行思考。这意味着要思考如何一次处理多条记录，而不是单条记录。例如，你可能有一个像生成报告这样的任务。挑战自己，尝试用单个查询生成一份报告。虽然这并非在所有情况下都是性能最优的，但会让你习惯于思考如何描述将要进入报告的那一组行。这实际上需要一种更少算术性、更多代数性的方法。通常，执行 `SELECT` 语句是适应基于集的事务最简单的方式。考虑表 5-7 中显示的数据。

**表 5-7 客户数据**

| FirstName | LastName | Address | City | PostalCode | Country |
| --- | --- | --- | --- | --- | --- |
| Myra | Acharya | 30 Magrath Road | Bengaluru | 560025 | India |
| Josè | Gomez | 790 Calle Cinco de Mayo | Cancun | 53778 | Mexico |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | Egypt |
| Marty` | Bethel | 750 Cherry Rd | Memphis | 38117 | United States |
| Stacy | Alexander | 123 Rua de Santa Catarina | Porto | 1234-567 | Portugal |

这是一份部分客户的地址列表。你可以编写清单 5-1 中的 T-SQL 查询来获取上面的数据集。

```sql
SELECT FirstName,
       LastName,
       [Address],
       City,
       PostalCode,
       Country
FROM dbo.Customer
WHERE CustomerID BETWEEN 401405 AND 401409
ORDER BY City,
       Country;
-- 清单 5-1
-- 部分客户列表
```

通过查看前面的数据，你可以发现某些列数据之间的相似性。

你可以创建一个数据集来获取孟菲斯市客户的所有订单。这些记录如表 5-8 所示。

**表 5-8 孟菲斯市的客户**

| FirstName | LastName | Address | City | PostalCode | Country |
| --- | --- | --- | --- | --- | --- |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | Egypt |
| Marty` | Bethel | 750 Cherry Rd | Memphis | 38117 | United States |

如果你想使用 T-SQL 来查找相同的信息，可以编写清单 5-2 中的查询。

```sql
SELECT FirstName, LastName
FROM dbo.Customer
WHERE City = 'Memphis';
-- 清单 5-2
-- 孟菲斯市的客户
```

你也可以获取所有产品单价大于或等于 $500 的订单数据集，如表 5-9 所示。

**表 5-9 产品单价等于 $500 的客户订单**

| FirstName | LastName | CustomerOrderID | QuantitySold |
| --- | --- | --- | --- |
| Karim | Khalil | 802818 | 2 |
| Marty` | Bethel | 802817 | 1 |
| Myra | Acharya | 802816 | 1 |

清单 5-3 是检索表 5-9 中数据所需的查询。

```sql
SELECT cus.FirstName,
       cus.LastName,
       ord.CustomerOrderID,
       SUM(dtl.QuantitySold) AS QuantitySold
FROM dbo.Customer cus
INNER JOIN dbo.CustomerOrder ord
       ON cus.CustomerID = ord.CustomerID
INNER JOIN dbo.OrderDetail dtl
       ON ord.CustomerOrderID = dtl.CustomerOrderID
WHERE dtl.ProductPrice = 249.99
GROUP BY cus.FirstName,
       cus.LastName,
       ord.CustomerOrderID;
-- 清单 5-3
-- 用于识别产品单价等于 $500 的客户订单的查询
```

现在你有了一个数据集，可以考虑如何与该数据集进行交互。知道可以将相似的数据分组并对整个数据集执行操作，是有效使用 T-SQL 代码的关键。如果你想找出产品的平均购买数量，可以将订单总额除以单位数量。要计算每条记录的平均购买量，可以手动为每条记录计算成本。在这种情况下，`CustomerOrderID` 802817 的订单总额是 $2,295。这张发票上售出的总单位数是 5：三个望远镜和两个充气桨板。结果是每个售出单位 $459。或者，你可以使用数据集中的列进行相同的计算。要将单位成本作为数据集计算，你可以指定 `Average Unit Price`。以这种方式处理数据的结果如表 5-10 所示。

**表 5-10 计算得出的平均单位价格**

| CustomerOrderID | Average Unit Price |
| --- | --- |
| 802817 | $459.00 |
| 802924 | $249.00 |

使用 SQL Server 最大的挑战之一就是学习如何以数据集的方式思考。思考数据集最简单的方法是，将检索数据的过程视为对一列或列的子集执行操作。如果你能找到某种模式，说明如何对数据的子集执行该操作，那么你就是在以数据集的方式思考。

### 识别数据集合

查看 `Customer` 表，你可以创建一个数据集合，其中所有城市的名称都包含“Memphis”。`Customer` 表中所有可用值如表 5-11 所示。

表 5-11
客户通用列表

| `First Name` | `Last Name` | `Address` | `City` | `Postal Code` | `Country` |
| --- | --- | --- | --- | --- | --- |
| Myra | Acharya | 30 Magrath Road | Bengaluru | 560025 | India |
| Josè | Gomez | 790 Calle Cinco de Mayo | Cancun | 53778 | Mexico |
| Stacy | Parks | 123 Rua de Santa Catarina | Porto | 1234-567 | Portugal |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | Egypt |
| Marty` | Bethel | 750 Cherry Rd | Memphis | 38117 | United States |

有几种不同的方法可以实现这一点。你可以编写一个查询来查找 `CustomerID` 为 401407 或 401405 的记录。你可以看到此逻辑的结果如表 5-12 所示。

表 5-12
孟菲斯市的客户

| `First Name` | `Last Name` | `Address` | `City` | `Postal Code` | `Country` |
| --- | --- | --- | --- | --- | --- |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | Egypt |
| Marty` | Bethel | 750 Cherry Rd | Memphis | 38117 | United States |

回到前面的表格，为孟菲斯添加一个新客户。表 5-13 显示了添加的新记录。

表 5-13
添加新的孟菲斯客户

| `First Name` | `Last Name` | `Address` | `City` | `Postal Code` | `Country` |
| --- | --- | --- | --- | --- | --- |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | Egypt |
| Marty` | Bethel | 750 Cherry Rd | Memphis | 38117 | United States |
| Ruth | Johnson | 4711 Ponce de Leon | Memphis | 38110 | United States |

如果你想仍然显示所有城市为孟菲斯的记录，并且使用与之前相同的逻辑（即使用 `CustomerID` 来显示所需的值），你将不会得到预期的结果。如果你不更改数据访问方式，并使用与获取表 5-13 结果相同的逻辑，你将得到表 5-14 中显示的结果集。

表 5-14
使用 `CustomerID` 查询孟菲斯的客户

| `CustomerID` | `First Name` | `Last Name` | `City` | `Postal Code` | `Country` |
| --- | --- | --- | --- | --- | --- |
| 401407 | Karim | Khalil | Memphis | 3364932 | Egypt |
| 401405 | Marty` | Bethel | Memphis | 38117 | United States |

表 5-15 通过更改逻辑来查看城市名称以查找所有等于“Memphis”的城市，从而显示了预期的结果。

表 5-15
城市名称为“孟菲斯”的客户

| `CustomerID` | `First Name` | `Last Name` | `City` | `Postal Code` | `Country` |
| --- | --- | --- | --- | --- | --- |
| 401407 | Karim | Khalil | Memphis | 3364932 | Egypt |
| 401405 | Marty` | Bethel | Memphis | 38117 | United States |
| 401408 | Ruth | Johnson | Memphis | 38110 | United States |

在这种情况下，我会想要运行一个查询，其中城市名称等于“Memphis”。如你在此示例中所见，这不仅在数据集合方面存在限制（影响性能），还可能影响功能。

使用数据集合时最大的挑战之一与插入数据记录有关。在大多数情况下，当你插入数据记录时，一次只插入一条记录。这可能导致你养成一次只处理插入一条单独记录的习惯。当你设计新表或移动数据时，你可能会遇到可以从数据集合角度考虑插入操作的情况。这通常在你创建查询以将数据插入另一个表时发生。

虽然这些情况看起来不会经常出现，但我在填充临时表、表变量和公用表表达式时经常使用数据集合。

以集合形式插入数据建立在以数据集合形式选择数据的基础之上。一旦你以集合形式选择了数据，就可以以集合形式插入数据。我使用将数据插入对象（如临时表）的方式有几个不同的原因。插入数据集合的一个更常见的情况是当我连接几个不同的表，并实现涉及复杂计算或函数的准则时。查看表 5-16 和表 5-17，你可以看到一些插入到 `dbo.Customer` 中的数据示例。

表 5-17
作为记录添加的 Ruth

| `CustomerID` | `First Name` | `Last Name` | `City` | `Postal Code` | `Country` |
| --- | --- | --- | --- | --- | --- |
| 401408 | Ruth | Johnson | Memphis | 38110 | United States |

表 5-16
作为记录添加的 Marty`

| `CustomerID` | `First Name` | `Last Name` | `City` | `Postal Code` | `Country` |
| --- | --- | --- | --- | --- | --- |
| 401405 | Marty` | Bethel | Memphis | 38117 | United States |

虽然所有这些条目都可以单独插入，但也可以一次插入多条记录。在这种情况下，它可能看起来像表 5-18 所示。

表 5-18
作为数据集合添加客户

| `CustomerID` | `First Name` | `Last Name` | `City` | `Postal Code` | `Country` |
| --- | --- | --- | --- | --- | --- |
| 401405 | Marty` | Bethel | Memphis | 38117 | United States |
| 401408 | Ruth | Johnson | Memphis | 38110 | United States |

我使用批量插入到临时对象中的另一个场景是，当这些记录可能需要通过连接到包含多表连接的其他查询来进行额外修改时。


