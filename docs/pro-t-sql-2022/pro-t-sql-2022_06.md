# T-SQL 格式化标准

制定 T-SQL 格式化标准的另一个因素是创建一个易于员工记忆或参考的标准。你希望团队成员在实施新标准时能够成功，而不会在编写代码时被所有细节压得喘不过气。如果你的所有 T-SQL 代码都必须手动编写，并且你的公司没有可以自动格式化 T-SQL 代码的软件，这一点尤为重要。

对于 `INSERT`、`UPDATE` 和 `DELETE` 语句，也有一些格式上的考虑。清单 3-8 包含了一个 `INSERT` 语句示例。在这个例子中，我列出了 `INSERT` 的所有列名。

```
INSERT INTO dbo.Product (ProductName, ProductPrice) VALUES
('Telescope', 599.00);
清单 3-8
插入数据的查询
```

虽然列出列名可能看起来没有必要，但这种格式标准使得正在插入的数据变得易于识别，并且当列被添加或列顺序改变时，这种格式还能保护应用程序代码免受未来问题的影响。更新数据的格式很简单，如清单 3-9 所示。我仍然遵循与保留字和引用用户定义数据库对象相同的格式。

```
UPDATE dbo.Customer
SET Address = '123 Main Street'
WHERE CustomerID = 1;
清单 3-9
更新数据的简单查询
```

你还可以看到我始终在运算符前后留出空格。我在几个例子中都这样做了。就像我为格式化所做的其他决定一样，我相信在等号前后添加空格可以提高 T-SQL 代码的可读性。我还包含了清单 3-10 来展示如何在 T-SQL 中格式化删除数据。

```
DELETE FROM dbo.Product
WHERE ProductName = 'Electric surfboard';
清单 3-10
删除数据的查询
```

这个例子是一个简单的删除，当涉及到连接（`JOIN`）时，删除数据的格式可能会变得更加复杂。删除数据在 SQL Server 中通常比其他数据操作活动显得更为重大。有时你可能想编写一个查询来系统性地删除表中的数据。当我刚开始编写从表中删除数据的查询时，我会从编写 `SELECT` 语句开始。这将有助于解决几个因素。我可以清楚地看到哪些数据会受到影响。我还可以获得预期受影响记录的行数。一旦我写好了 `SELECT` 语句，我就可以轻松地修改代码以删除必要的记录。清单 3-11 中的查询显示了我用来准备删除数据记录的 `SELECT` 语句。

```
SELECT ord.CustomerOrderID, ord.OrderNumber
FROM dbo.CustomerOrder ord
INNER JOIN dbo.OrderDetail dtl
ON ord.CustomerOrderID = dtl.CustomerOrderID
WHERE dtl.ProductID = 2;
清单 3-11
选择 ProductID 为 2 的订单
```

在这种情况下，我正准备从 `dbo.CustomerOrder` 表中删除 `ProductID` 为 2 的记录。使用清单 3-11 的结果，我可以确认我正在删除什么数据以及预计会删除多少条记录。在审查了清单 3-11 的结果后，我可以更新我的 T-SQL 代码来删除这些记录。在清单 3-12 中，我用引用 `dbo.CustomerOrder` 表别名的 `DELETE FROM` 替换了 `SELECT` 语句。

```
BEGIN TRAN
DELETE FROM ord
FROM dbo.CustomerOrder ord
INNER JOIN dbo.OrderDetail dtl
ON ord.CustomerOrderID = dtl.CustomerOrderID
WHERE dtl.ProductID = 2;
COMMIT
清单 3-12
删除 ProductID 为 2 的订单
```

多年来，我发现最好将复杂的 `DELETE` 语句包装在显式事务中。默认情况下，当我们在 SQL Server 上执行 T-SQL 代码时，我们使用的是隐式事务。这意味着 SQL Server 知道在我们执行事务后自动提交事务。我们也可以选择指定显式事务。在这种情况下，SQL Server 在我们发送 `COMMIT` 命令之前不会完成 T-SQL 代码的执行。特别是在删除操作中，但通常对于任何复杂的代码，我发现最好始终小心谨慎。我有两个选择。我可以单独运行 `BEGIN TRAN` 和 `DELETE` 语句。如果结果看起来不错，我可以运行 `COMMIT`。如果结果不如人意，我可以回滚并重新开始。如果你使用这种方法，你必须确保正确地 `ROLLBACK` 或 `COMMIT` 显式事务。

另一个选择是运行包含 `BEGIN TRAN` 和 `DELETE` 语句的整个脚本，但加上 `ROLLBACK`。如果行数看起来正确，我可以将 `ROLLBACK` 改为 `COMMIT` 并重新执行。

这让我能够验证受影响的记录数量。这是我确认查询是否按预期工作的最后机会。如果我返回的记录数量与我预期的不同，我可以发出 `ROLLBACK`，SQL Server 将撤销我尝试运行的代码。这只在显式事务中才有可能。还值得一提的是，如果你使用了多级事务（称为嵌套事务），`ROLLBACK` 的功能可能与你预期的不同，因为它会回滚所有嵌套的开放事务。

这就引出了设计 T-SQL 格式时的另一个考虑因素。你需要了解你的 T-SQL 代码将如何被编写和存储。如果所有 T-SQL 代码都将手动编写且无法通过第三方工具格式化，那么你可能需要保持编码标准非常简单，并限制特殊情况的准则。然而，如果你有可用的第三方工具，你可以创建一个该工具能处理的最复杂的格式。

优秀开发者或工程师的特质之一是纪律性。编写 T-SQL 也是如此。在某种程度上，你使用哪种特定的格式风格并不重要。重要的是要与该格式保持一致。理想情况下，你应该努力让你的整个团队就一种标准的 T-SQL 格式化方法达成一致。

当你的整个团队都以相同的方式编写 T-SQL 代码时，查看别人的代码就会容易得多。你不再需要同时转换格式和编码风格，这可以加快分析和解决问题的速度。格式化你的 T-SQL 代码使其更具可读性，完全是关于代码的外观。

你应该朝着为公司编写的所有 T-SQL 代码拥有统一格式的方向努力。由于无论谁编写代码，格式都将相同，这也意味着每个阅读代码的人都会越来越熟悉快速解读他们的代码。在清单 3-13 中，我创建了一个用户定义的表类型。

```
CREATE TYPE ProductPricing AS TABLE
(
ProductName  VARCHAR(25),
ProductPrice DECIMAL(6,2)
);
清单 3-13
创建用户定义的表类型
```

我在清单 3-14 中创建临时表时也使用了统一的格式。

```
CREATE TABLE #TempCustomerFirstOrder
(
CustomerID    INT,
FirstName     VARCHAR(40),
LastName      VARCHAR(100),
FirstOrder    DATETIME2(2),
QuantityOrder SMALLINT
);
清单 3-14
创建临时表
```

你还可以看到我在清单 3-15 中创建表变量时使用了相同的格式。

```
DECLARE @TempCustomerFirstOrder TABLE
(
CustomerID    INT,
FirstName     VARCHAR(40),
LastName      VARCHAR(100),
FirstOrder    DATETIME2(2),
QuantityOrder SMALLINT
);
清单 3-15
创建表变量
```



对比代码清单 3-13、3-14 和 3-15，你可以很快发现我在创建表和声明变量时使用了统一的格式。这样，任何其他人看到这种格式的代码，都能立即明白这是在创建表。

关于大小写，至少有两件事需要考虑。一是与关键字相关的大写，二是所有其他术语的大小写。我倾向于将所有保留字大写。`T-SQL` 的保留字列表可能相当长。我选择将任何不属于数据库架构组成部分的词都视为保留字。对于数据库对象和列名，我按其创建时的形式大小写书写。创建表别名时，我使用小写。你可以在代码清单 3-16 中看到，保留字被大写，而数据库对象名的每个单词的首字母则大写。

```
CREATE TRIGGER dbo.ArchiveProduct
ON dbo.Product
AFTER INSERT, UPDATE
AS
IF (ROWCOUNT_BIG() = 0)
RETURN;
INSERT INTO dbo.ProductArchive (ProductID, ProductPrice, DateCreated)
SELECT inserted.ProductID, inserted.ProductPrice, GETDATE()
FROM inserted;
代码清单 3-16
创建一个 DML 触发器
```

在确定 `T-SQL` 的格式标准时，另一个需要考虑的因素是是否以及如何使用别名。别名是一种允许你创建一个简短名称来引用表的方法。在编写查询时，也可以在 `SELECT` 语句中为列名创建别名。如何使用别名值取决于别名是针对表还是列。然而，总体概念是相同的。如果为表名创建了别名，则必须在整个查询中使用该别名来代替表名。对于列，别名常用于重命名列或使输出更用户友好。通常，列别名不会被引用。

我在编写查询时唯一会引用列别名的情况是，必须按一个已被别名化的列进行排序，特别是当该列包含的逻辑不仅仅是重命名原始列名时。可以在 `ORDER BY` 语句中提供一个代表列顺序的数字，而不是提供列名。虽然这是一种快速排序数据的方法，但不建议在永久性的应用程序代码中使用，因为 `SELECT` 语句中的列顺序可能会更改，而 `ORDER BY` 子句却未得到更新。这将导致返回的数据可能按非预期的顺序排列。

关于 `T-SQL` 代码格式化的另一个有争议的话题是，在编写代码时如何格式化逗号。有些人喜欢将逗号放在每行的开头。这可以提高可读性，并帮助其他人快速识别这是多行中的一行。将逗号放在每行开头确实简化了调试，因为很容易注释掉以逗号开头的单行，而查询的其余部分仍能正确解析。我更喜欢将逗号放在结尾，这样我就可以忽略逗号，专注于查询中返回的列。

对于 `WHERE` 子句中的多个条件，我确实倾向于在每个条件前加上运算符。对于许多类型的条件，这个运算符要么是 `OR`，要么是 `AND`。这使我能够快速查看 `WHERE` 子句中的逻辑，而无需扫描整个 `WHERE` 子句来确定所有条件如何协同工作。你也可以选择让每行以运算符结尾，但你可能会发现这会使调试复杂化。

在使用 `T-SQL` 代码时，有时你需要编写复杂的代码。这些代码可能包括子查询，或者 `WHERE` 子句中涉及 `AND` 或 `OR` 的逻辑。如果在 `WHERE` 子句中存在像子查询或 `AND` 和 `OR` 混合这样的逻辑，我会将它们用括号括起来。我会缩进括号内的所有代码，以便于识别哪些逻辑被组合在一起。有时，两个表之间会有多个连接条件。如果两个表之间存在多个连接条件，我会缩进除第一个条件之外的所有其他连接条件，以便他人能轻松看出两个表之间存在多个连接条件。

在向代码添加额外的逻辑层级时，你还需要考虑如何格式化 `T-SQL` 代码。`T-SQL` 代码块有各种原因，包括 `TRY… CATCH` 块、`IF… ELSE` 语句、`BEGIN…END` 或其他需要分割代码的情况。对于这些场景，我会缩进代码块内部。如果代码块最终被嵌套，我会对每个后续的代码块进行缩进。我倾向于缩进我的代码块，以便审查我代码的人能看到父级活动，例如代码清单 3-17 中的 `WHILE` 循环。一旦你看到第一级缩进，你就知道所有缩进的逻辑都属于同一个代码块。

```
SET NOCOUNT ON;
DECLARE @ProductID INT,
@ProductName VARCHAR(25),
@message VARCHAR(50);
PRINT '-------- 产品列表 --------';
DECLARE product_cursor CURSOR DYNAMIC
FOR
SELECT ProductID, ProductName
FROM dbo.Product;
OPEN product_cursor;
FETCH NEXT FROM product_cursor
INTO @ProductID, @ProductName;
WHILE @@FETCH_STATUS = 0
BEGIN
PRINT ' ';
SELECT @message = '----- 产品客户：' + @ProductName + '-----';
PRINT @message;
SELECT LEFT(cus.FirstName, 25), LEFT(cus.LastName, 25), SUM(dtl.QuantitySold) AS QuantitySold
FROM dbo.CustomerOrder ord
INNER JOIN dbo.OrderDetail dtl
ON ord.CustomerOrderID = dtl.CustomerOrderID
INNER JOIN dbo.Product prd
ON dtl.ProductID = prd.ProductID
INNER JOIN dbo.Customer cus
ON ord.CustomerID = cus.CustomerID
WHERE prd.ProductID = @ProductID
GROUP BY cus.FirstName, cus.LastName, prd.ProductName;
FETCH NEXT FROM product_cursor INTO @ProductID, @ProductName;
END;
CLOSE product_cursor;
DEALLOCATE product_cursor;
代码清单 3-17
创建一个游标
```

一致地格式化 `T-SQL` 代码可以提高你自己以及将来任何需要审查你代码的人的可读性。格式良好的代码有助于在故障排除、性能调优和代码增强时提供清晰度。在你的组织中创建一个 `T-SQL` 格式化标准，也有助于新员工入职或培训初级数据库开发人员。一旦确定了 `T-SQL` 格式化标准，你将需要考虑采取哪些步骤来为你的 `T-SQL` 代码创建命名约定。



## T-SQL 命名规范

当你编写 T-SQL 时，你可以选择如何编写代码。正如第 2 章所述，你很可能会在 T-SQL 中创建持久化对象。无论你的目的如何，遵循良好的命名策略都能让其他人更容易理解你的 T-SQL 代码的用途。理想情况下，你的团队成员应该能够根据代码所在对象的名称来判断其目的。这对于新员工或经验较少的员工来说尤其有帮助。

命名规范的实践与 T-SQL 格式化的实践类似。命名规范所涉及的一个方面是外观和感觉。这可能涉及对象使用的大写方式。在为数据库对象提供大小写方案时，有多种选项可用。主要选择是 `camel case`（驼峰命名法）或 `pascal case`（帕斯卡命名法）。`驼峰命名法`是指第一个单词的首字母不大写，但所有其他单词的首字母大写。`帕斯卡命名法`是指每个单词的首字母都大写。这些大小写风格的主要区别在于数据库对象第一个单词的首字母。`代码清单 3-18` 展示了一个使用`驼峰命名法`的查询。

```sql
SELECT productID, productName, dateCreated, dateModified
FROM dbo.product;
```
`代码清单 3-18`
使用驼峰命名法的查询

相反，你可以在`代码清单 3-19`中看到编写相同查询时`帕斯卡命名法`的样子。

```sql
SELECT ProductID, ProductName, DateCreated, DateModified
FROM dbo.Product;
```
`代码清单 3-19`
使用帕斯卡命名法的查询

此外，还有一种选项是首字母不大写，并且在单词之间使用下划线。我个人通常不喜欢数据库名称中出现非字母字符，但我确实知道有些人更喜欢下划线。如果你想要另一种选择，`代码清单 3-20`展示了如何以所谓的`蛇形命名法`来命名表和列。

```sql
SELECT productID, product_name, date_created, date_modified
FROM dbo.product;
```
`代码清单 3-20`
使用蛇形命名法的查询

在确定为你的命名规范使用哪种大小写时，务必也要注意数据库的排序规则以及是否有任何表使用了特殊的排序规则。了解大小写敏感性将有助于确保你的命名规范和格式标准保持一致。

这还可能涉及对象在 `对象资源管理器` 中出现的位置。其中一些与正在命名的对象类型以及谁会查找这些对象有关。如果你想按用途查找对象，可能需要指定`架构`名称来将这些对象分组。这对于应用程序或服务可能特别有用。根据预期的故障排除类型，数据库对象，特别是`存储过程`，可以按其执行的操作来命名。这便于搜索进行`选择`数据操作的存储过程与进行`插入`数据操作的存储过程。然而，还有另一个选项，即让受影响的主要表作为存储过程名称的第一个词。这允许某人在`对象资源管理器`中按受影响的表进行搜索，以查看存在的所有存储过程。

另一个考虑因素是在为数据库对象命名时是否可以使用`保留字`。如果在数据库对象名称中使用了保留字，你将需要在对象名称中添加另一个词，以便在引用对象名称时不需要使用方括号。

在确定命名规范时，你可能还需要考虑表中是否有一些列会与其他表中的列具有相同的名称。这类列的一些例子包括：指定记录创建日期、记录最后更新日期的列，以及记录是否被软删除的列。对于指定状态或类型的表，这些表可以包含一个状态或类型名称列以及一个关联的描述列。你可能会决定，希望所有这些列在所有包含这些列的表中都使用完全相同的名称。

在编写查询时，如果可能，我倾向于不对列名使用别名。我也不显示多个创建日期或更新日期列。因此，对于创建日期、修改日期和软删除标志，我使用完全相同的名称。然而，在编写查询时，我也经常返回多个表中的名称或描述字段。因此，我更喜欢这些列在列名中包含表名作为一部分。例如，我不会在 `Customer` 和 `Product` 表中使用列名 `Description`，而是倾向于在 `Customer` 表中将列命名为 `CustomerDescription`，在 `Product` 表中将列命名为 `ProductDescription`。这使得其他人可以查看 select 语句并轻松识别正在引用的表。这也意味着我在编写查询时需要别名的列更少。

命名持久化数据库对象也可能很棘手。为表命名可能不同于为索引、视图、触发器或函数命名。再次强调，为这些对象命名不仅仅是给它们一个描述性的名称。这也可能是给它们一个能清楚表明其对象类型的名称。这是因为许多数据库工程师和开发人员使用 `对象资源管理器` 作为查找对象的主要工具。如前所述，如果表名以名词开头，存储过程以动词开头，那么我接下来需要弄清楚如何将其他数据库对象与表和存储过程区分开来。

其中一个选项是在对象名称前加上对象类型的缩写。对于索引，你可以对非聚集索引使用 `IX_`，对聚集索引使用 `CX_`。在命名索引时，一旦指定了 `IX_` 或 `CX_`，接下来的项目应该是索引所在的表名。表名之后应该是索引中包含的列的列表。这些列应按照创建索引时指定的顺序列出。在`代码清单 3-21`中，你可以看到一个聚集索引。聚集索引名称以 `CX` 开头，后跟表名，然后是列名。每个部分之间用下划线分隔。

```sql
CREATE CLUSTERED INDEX CX_Product_ProductName
ON dbo.Product (ProductName);
```
`代码清单 3-21`
创建聚集索引

创建非聚集索引遵循相同的模式。在`代码清单 3-22`中，你可以看到非聚集索引也包含多个列。

```sql
CREATE NONCLUSTERED INDEX IX_Customer_DateCreated_FirstName_LastName_Country
ON dbo.Customer
(
DateCreated ASC,
FirstName ASC,
LastName ASC,
Country ASC
);
```
`代码清单 3-22`
创建包含多列的非聚集索引


### 数据库对象命名规范

#### 主键与外键命名

在创建主键和外键时也遵循同样的原则。如果在`T-SQL`中创建主键或外键时未指定名称，`SQL Server`会随机分配一个名称。因此，**最佳实践**是显式命名主键或外键。

- 命名主键时，应使其以`PK_`开头，代表主键。
- 类似地，外键应以`FK_`开头。
- 名称的下一部分是分配该键的表名。
- 接着是用于定义该键的一个或多个列名。如果指定了多个列，这些列在键定义中应按顺序列出。

清单[3-23]展示了如何在表创建后添加并命名主键。

```
ALTER TABLE dbo.Product
ADD CONSTRAINT PK_Product_ProductID
PRIMARY KEY (ProductID);
Listing 3-23
Add a Primary Key
```

可以看到，主键名称以`PK`开头，后跟表名和用于创建主键的列名。

类似地，外键名称以`FK_`开头。名称的下一部分是外键所属的表名，接着是用于定义外键的列。如果指定了多个列，它们应按顺序列出。在表创建后添加外键可以参考清单[3-24]。

```
ALTER TABLE dbo.CustomerOrder
ADD CONSTRAINT FK_Order_CustomerID
FOREIGN KEY (CustomerID)
REFERENCES dbo.Customer(CustomerID);
Listing 3-24
Add a Foreign Key
```

与创建主键类似，外键遵循相似的命名结构：以`FK`开头，后跟表名和列名。

#### 视图与触发器命名

我也倾向于在视图名称前加上`vw_`，在触发器前加上`tr_`。还有其他不那么明显的命名选项。例如，你可以为视图或触发器定义保留特定的名词或动词列表。这提供了更多灵活性，也能防止`Object Explorer`中的列表项都以相同的前三个字符开头。这能让任何人都轻松识别出这些对象既不是表也不是存储过程。这对于视图和触发器尤其重要。

最大的挑战之一是，如果命名不能明显区分它们是视图还是触发器，人们可能会花费大量时间寻找这些对象而无法找到。视图像表一样用于连接查询，而触发器则像存储过程一样用于修改对象。触发器更棘手，因为它们可能导致你花费大量时间研究存储过程，试图找出值发生变化的原因。

#### 表命名

命名表时，应仅使用名词。这有助于表明这些对象用于存储，而不是执行任何特定活动。

在命名对象时，一个常被忽视直至为时已晚的命名约定是：对象名是使用单数还是复数。在首次出现复数数据库对象之前，这可能不是问题。但一旦架构中存在复数对象，其问题会很快变得明显。这是因为数据库中一旦存在复数对象，编写查询时就越来越难以记住哪些表是复数、哪些是单数。

选择表名时，还应确保名称具有描述性。应以一种能让其他数据库工程师和开发者轻松了解表中存储信息类型的方式来描述表。使用名词命名表也表明该对象用于存储，而非执行任何特定活动。

#### 存储过程命名

大多数用户很容易熟悉表和存储过程。选择一种命名约定来区分这些对象是很容易的，例如让存储过程以动词开头，或在存储过程名称中包含其他高级条件。清单[3-25]展示了一个以动词开头的存储过程。

```
EXECUTE dbo.GetCustomer;
Listing 3-25
Stored Procedure Beginning with a Verb
```

如果我看到这个存储过程名，我期望它能检索所有客户。清单[3-26]中的存储过程返回特定客户的所有信息。

```
EXECUTE dbo.GetCustomerByCustomerID 1;
Listing 3-26
Stored Procedure with Selection Criteria
```

### 总结

由此可见，在命名`T-SQL`数据库对象时有许多考虑因素。在许多情况下，一个好的名称能帮助你轻松确定数据库对象的用途。在特殊情况下，它还能帮助你识别对象的类型。虽然你通常在格式化和注释`T-SQL`之后才命名对象，但我将注释放在最后，因为它既涉及数据库对象的创建，也涉及数据库对象内部的`T-SQL`代码。


## T-SQL 注释

虽然编写 T-SQL 的主要目的是让应用程序能够对数据执行特定操作，但同样重要的是确保他人能够理解你未来的 T-SQL 代码。至少，这能确保你不是唯一一个为某段代码或业务逻辑负责的人。它还有助于你团队的其他成员增强对自己工作能力的信心，并加深对幕后代码的理解。

在许多情况下，快速浏览 T-SQL 代码或查看数据库对象的名称并不足以理解 T-SQL 代码的用途。清单 3-27 展示了在数据库对象顶部创建头部区域的示例 T-SQL 代码。

```sql
/*-------------------------------------------------------------*\
Name:             
Author:           
Created Date:     
Description:      
Sample Usage:

Change Log:
Update on  by : 
Update on  by : 
\*-------------------------------------------------------------*/
Listing 3-27
持久化数据库对象头部的注释
```

这个头部的目的是让其他用户能够一目了然地了解此数据库对象的概况。审查此 T-SQL 的目的不同，对用户重要的信息也会不同。对于那些不熟悉所有业务应用细节的人来说，描述部分会提供关于该 T-SQL 数据库对象用途的高层次概念。同样地，示例用法让那些调查性能问题的人能够了解应用程序如何使用这段 T-SQL 代码。数据库对象的作者可以帮助确定原始创建者是否仍在公司，以回答有关该数据库对象的更具体问题。如果你的数据库处于源代码控制之下，你可能会选择省略其中一些字段。我将在第 11 章进一步讨论源代码控制。在清单 3-28 中，你可以看到创建最初见于清单 3-4 的视图时头部信息的样子。

```sql
/*-------------------------------------------------------------*\
Name:             dbo.CustomerProduct
Author:           Elizabeth Noble
Created Date:     2022-10-31
Description:      Simple view to display all customer purchases
Sample Usage:
SELECT FirstName, LastName, ProductName, QuantitySold
FROM dbo.CustomerProduct;
Change Log:
Update on 2022-10-31 by enoble: Added header to view
\*-------------------------------------------------------------*/
CREATE OR ALTER VIEW dbo.CustomerProduct
AS
SELECT cus.FirstName, cus.LastName, prd.ProductName, SUM(dtl.QuantitySold) AS QuantitySold
FROM dbo.CustomerOrder ord
INNER JOIN dbo.OrderDetail dtl
ON ord.CustomerOrderID = dtl.CustomerOrderID
INNER JOIN dbo.Product prd
ON dtl.ProductID = prd.ProductID
INNER JOIN dbo.Customer cus
ON ord.CustomerID = cus.CustomerID
GROUP BY cus.FirstName, cus.LastName, prd.ProductName;
Listing 3-28
创建带有注释头部的视图
```

### 为复杂逻辑添加注释

有时，编写易于阅读的简单 T-SQL 可能就足够了。然而，也有些时候，数据库对象可能包含难以快速理解的复杂 T-SQL 代码。我发现在编写代码后添加注释的一个挑战是，我的注释往往过于技术化，难以轻松描述我试图实现的目标。你需要确保不熟悉你代码的人也能轻松理解你 T-SQL 代码的目的。

如果存在任何复杂逻辑，你需要清晰地解释该复杂逻辑是如何工作的。如果你使用了不是最佳实践的 T-SQL 编码方式，这一点尤其重要。任何关于为何选择非标准实践的解释都有助于节省他人的宝贵时间。如果你已经确定标准的最佳实践在特定情况下性能不够好，这也能节省你团队成员的时间。回顾清单 3-3，`SELECT`语句中有一个子查询。初次编写这段代码时，使用子查询的原因可能是显而易见的，但未来你可能会忘记。此外，阅读我代码的其他人可能也不会明显看出我为何选择为该列包含一个子查询。在清单 3-29 中，我展示了如何添加注释，以便更容易快速理解所编写代码的目的或逻辑。

```sql
-- 外部查询显示所有购买了包含“board”一词产品的客户
SELECT
-- 下面子查询用于显示也购买了类似“board”零件的客户
---- 所购买的所有零件
(
SELECT prd.ProductName
FROM dbo.CustomerOrder ord
INNER JOIN dbo.OrderDetail dtl
ON ord.CustomerOrderID = dtl.CustomerOrderID
INNER JOIN dbo.Product prd
ON dtl.ProductID = prd.ProductID
WHERE ord.CustomerID = bord.CustomerID
GROUP BY prd.ProductName
) AS ProductName
FROM dbo.CustomerOrder bord
INNER JOIN dbo.OrderDetail bdtl
ON bord.CustomerOrderID = bdtl.CustomerOrderID
INNER JOIN dbo.Product bprd
ON bdtl.ProductID = bprd.ProductID
WHERE bprd.ProductName LIKE '%board%';
Listing 3-29
包含子查询的查询
```

### 先编写注释

如果你在开始编写任何 T-SQL 代码之前就先为 T-SQL 代码添加注释，通常最容易在代码中包含注释。在清单 3-30 中，你可以看到头部信息和起始注释。这些注释指明了编写该存储过程背后的概念。

```sql
/*-------------------------------------------------------------*\
Name:             dbo.GetCustomerPurchase
Author:           Elizabeth Noble
Created Date:     2022-10-30
Description:      Lookup customer information for a given product
Sample Usage:
DECLARE @ProductID INT;
SET @ProductID = 1;
EXECUTE dbo.GetCustomerPurchase @ProductID;
\*-------------------------------------------------------------*/
-- 获取出售给客户的产品
-- 显示总销售数量以及最近的购买日期
-- 按购买时的产品价格显示购买数量详情
---- 的详细信息
Listing 3-30
创建新的存储过程
```

一旦你指定了查询的总体逻辑，就可以继续编写 T-SQL 以满足这些要求。如果你经常发现自己是在重新表述已经写好的代码，而不是解释代码的总体目的，这通常会很有帮助。

在编写任何 T-SQL 代码之前编写这些注释，可以确保注释解释的是存储过程的目的，而不是代码如何执行。

### 总结

你现在应该已经准备好开始为你自己和你的组织定义 SQL 格式化标准了。这将使你和你团队的成员能够快速审查组织的 T-SQL 代码。此外，我还讨论了在为你的 T-SQL 代码提供额外文档时可以使用的策略。为你的 T-SQL 代码添加注释，可以让其他人理解 T-SQL 代码在业务逻辑和高级技术逻辑方面应该做什么。我还介绍了在为你的组织定义命名约定时可用的选项。定义良好的命名约定应该使任何访问数据库架构的人都能更容易地知道在哪里找到数据库对象。

你现在应该已经熟悉了第 1 章中的 SQL Server 数据类型以及使用它们的最佳时机。你也应该对编写 T-SQL 时可用的各种数据库对象感到得心应手。既然你更熟悉如何设置代码样式以提高可读性和理解度，你就已经准备好学习更多关于使用参数、复杂逻辑和存储过程设计 T-SQL 代码的知识了，这些将在下一节中介绍。



