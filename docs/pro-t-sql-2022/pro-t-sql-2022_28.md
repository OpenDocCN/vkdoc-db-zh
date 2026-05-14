# 支持遗留代码

应用程序是随着时间的推移而开发的，其开发方式将决定它们日后维护的难易程度。当你试图通过添加或删除列来修改表结构时，或许可以利用视图，使得在后端进行变更的同时，应用程序仍能按预期运行。类似地，你可能会发现数据库中存在许多非规范化表。本节也将讨论如何利用视图来帮助推进数据库模式向更加规范化的方向发展。

软件开发中的一个关键问题涉及处理技术债。对于许多组织而言，应用程序最初是在特定时间线下开发的，以满足当前业务需求。这可能导致应用程序在未及规划未来支持方式的情况下被快速开发出来。这常常成为开发人员与数据库管理员之间紧张关系的唯一根源。

修改表的难点在于如何在不破坏整个应用程序的前提下进行变更。存在一些策略，可以让你在向现有表添加新列时不破坏现有代码。你需要一种方法，既能允许现有代码按原样引用表，又能以符合现有标准的方式存储这些新信息。本节讨论的选项旨在短期内支持新功能。建议你更新应用程序代码和数据库对象，以保持符合最佳实践。

真正的挑战在于，当一个应用程序被开发出来时，它通常没有完善的文档记录。如果应用程序文档不全，并且 T-SQL 查询直接保存在应用程序代码中，那么支持遗留代码将变得更加困难。如果数据库代码嵌在应用程序中，那么不仅需要定位特定的数据库代码，还需要更新它，这将需要付出更多努力。因为这不仅仅像修改一个表或存储过程那样简单，以提高 T-SQL 代码的灵活性和可维护性。

随着业务的增长和变化，同一个应用程序可能需要进行修改以处理新的功能。这些新功能可能需要存储新的信息。为了存储这些信息，你可能希望向表中添加一个列，或者完全创建一个新表。挑战在于试图理解所有依赖于该项目的元素。虽然你可以验证数据库对象之间的依赖关系，但问题是你并不知道应用程序当前是如何访问这个表的。你的应用程序可能通过预处理语句和即席查询来访问数据表。如果你的应用程序使用了预处理语句或即席查询，那么在向表中添加新列时，可能需要更改你的应用程序代码，特别是当你的编码标准没有规定在 T-SQL 查询中必须始终指定列名时。

允许向表中添加额外列的最简单选项是，创建另一个具有与你要修改的表相同主键的表。在这种情况下，你还需要重命名现有表，以便可以创建一个与原表同名的视图。所有访问原始未修改表的查询现在都将访问此视图。当你准备好更新所有应用程序代码时，可以删除该视图并将表重命名回其原始名称。

这些新列可以被添加到第二个表中。你可能会发现，由于遗留应用程序代码的编写方式，你无法实现如代码清单 13-8 所示的变更。你可能希望向`dbo.Customer`表添加`IsActive`列，但你的应用程序可能在内部嵌入了 T-SQL 查询。因此，你可能不确定添加该新列会导致什么功能中断。代码清单 13-10 中创建名为`dbo.CustomerIsActive`的新表是一种选择。

```sql
CREATE TABLE dbo.CustomerIsActive(
CustomerID      INT          IDENTITY(1,1)    NOT NULL,
FirstName       VARCHAR(40)                   NOT NULL,
LastName        VARCHAR(100)                  NOT NULL,
Address         VARCHAR(100)                  NOT NULL,
City            VARCHAR(100)                  NOT NULL,
PostalCode      VARCHAR(20)                       NULL,
Country         VARCHAR(75)                   NOT NULL,
IsActive        BIT                           NOT NULL,
CONSTRAINT DF_CustomerIsActive_IsActive DEFAULT ((1)),
DateCreated     DATETIME2(2)                  NOT NULL
CONSTRAINT DF_CustomerIsActive_DateCreated DEFAULT (SYSDATETIME()),
DateModified    DATETIME2(2)                  NOT NULL
CONSTRAINT DF_CustomerIsActive_DateModified DEFAULT (SYSDATETIME ()),
DateDisabled    DATETIME2(2)                      NULL
CONSTRAINT DF_CustomerIsActive_DateDisabled DEFAULT (NULL),
CONSTRAINT PK_CustomerIsActive PRIMARY KEY CLUSTERED (CustomerID ASC)
);
```
代码清单 13-10：创建带有 IsActive 列的 dbo.Customer 表副本

你可以将所有数据从`dbo.Customer`表移动到`dbo.CustomerIsActive`表。然后，你可以让应用程序继续插入或更新数据记录。你需要删除`dbo.Customer`表，并创建一个同名的视图。然而，如果日期数据量相当大或数据量巨大，你可能需要尝试另一种方法。一种方法是将`dbo.Customer`表重命名为`dbo.CustomerXYZ`。然后你创建一个名为`dbo.Customer`的视图，模拟旧的`dbo.Customer`表的结构。完成后，你可以开始向`dbo.CustomerXYZ`表添加列。视图的一个示例如代码清单 13-11 所示。

```sql
CREATE VIEW dbo.Customer
AS
SELECT CustomerID,
FirstName,
LastName,
[Address],
City,
PostalCode,
Country,
DateCreated,
DateModified
FROM dbo.CustomerIsActive;
```
代码清单 13-11：创建视图以匹配 dbo.Customer 的原始结构

现有的应用程序代码可以引用该视图来选择或修改数据。这不是一个理想的长期解决方案，旨在让你能够逐步实现重构应用程序以使用新表的目标。如果你在应用程序中直接编写了数据库代码，那么删除视图并将表重命名回`dbo.Customer`所需的过渡期可能需要付出额外的努力来管理。在确定如何修改数据库对象同时让现有代码继续运行时，你需要专注于实施一个能够让你在将来轻松继续开发 T-SQL 代码的解决方案。

除了视图，你还可以选择根据所需规范创建一个新对象。这就像创建代码清单 13-10 中所示的相同表一样。然后你可以添加一个触发器，将原始表的任何数据修改传递到这个新的数据库对象中。你需要为插入、更新和删除操作各创建一个 DML 触发器。代码清单 13-12 展示了如何创建一个插入触发器；你仍然需要创建一个用于更新的触发器和一个用于删除的触发器。


```
CREATE TRIGGER CustomerOrder_InsertCustomerOrderIsActive
ON dbo.CustomerOrder
FOR INSERT
AS
INSERT INTO dbo.CustomerOrderIsActive
(
CustomerID,
FirstName,
LastName,
[Address],
City,
PostalCode,
Country,
IsActive,
DateCreated,
DateModified,
DateDisabled
)
SELECT
CustomerID,
FirstName,
LastName,
[Address],
City,
PostalCode,
Country,
IsActive,
DateCreated,
DateModified,
DateModified,
DateDisabled
FROM inserted;
```

**清单 13-12：当记录被插入到 `dbo.CustomerOrder` 时创建触发器**

这将允许你继续对所有应用程序代码使用原始数据库对象。你可以保留这个额外的表和触发器，直到准备好开始在数据库代码中使用新表为止。

除了添加新列，你可能还需要使用一些其他策略来重构表。这可以包括增加数据库的规范化程度。遇到包含远超预期列数的遗留表是很常见的，尤其是在规范化程度高的数据库中。通过创建一个可以修改表设计并创建抽象层的空间，你可以在不影响应用程序性能的情况下修改数据库对象。

有时从最终目标开始会更容易。在这个例子中，你正试图将一个遗留表重新设计成两个或多个规范化表。想法是在实现这一目标的同时，让应用程序照常运行。在这个例子中，你有一个类似清单 13-13 的表。

```
CREATE TABLE dbo.OrderDetail(
OrderDetailID       INT           IDENTITY(1,1)   NOT NULL,
CustomerOrderID     INT                           NOT NULL,
ProductName         VARCHAR(25)                   NOT NULL,
ProductPrice        DECIMAL(6,2)                  NOT NULL,
QuantitySold        SMALLINT                      NOT NULL,
IsActive            BIT                           NOT NULL,
DateCreated         DATETIME2(2)                  NOT NULL,
DateModified        DATETIME2(2)                  NOT NULL,
DateDisabled        DATETIME2(2)                      NULL,
CONSTRAINT PK_OrderDetail PRIMARY KEY CLUSTERED (CustomerOrderID ASC, OrderDetailID ASC)
);
```

**清单 13-13：原始未规范化表**

一旦确定了要规范化的表，就需要设计用于替换这个原始表的新表。清单 13-14 展示了可以创建的表的示例，以便可以过渡到更规范化的数据表。

```
CREATE TABLE dbo.Product(
ProductID       INT           IDENTITY(1,1)     NOT NULL,
ProductName     VARCHAR(25)                     NOT NULL,
ProductPrice    DECIMAL(6,2)                    NOT NULL,
IsActive        BIT                             NOT NULL,
DateCreated     DATETIME2(2)                    NOT NULL,
DateModified    DATETIME2(2)                    NOT NULL,
DateDisabled    DATETIME2(2)                        NULL,
CONSTRAINT PK_Product PRIMARY KEY CLUSTERED (ProductID ASC)
);
CREATE TABLE dbo.OrderDetail_Normal(
OrderDetailID      INT           IDENTITY(1,1)    NOT NULL,
CustomerOrderID    INT                            NOT NULL,
ProductID          INT                            NOT NULL,
ProductPrice       DECIMAL(6,2)                   NOT NULL,
QuantitySold       SMALLINT                       NOT NULL,
IsActive           BIT                            NOT NULL,
DateCreated        DATETIME2(2)                   NOT NULL,
DateModified       DATETIME2(2)                   NOT NULL,
DateDisabled       DATETIME2(2)                       NULL,
CONSTRAINT PK_OrderDetail PRIMARY KEY CLUSTERED (CustomerOrderID ASC, OrderDetailID ASC)
);
```

**清单 13-14：规范化表**

在前面的例子中，如果确定 `dbo.Product` 表不会有数据修改，你也可以使用视图与 `dbo.OrderDetail` 表进行交互。清单 13-15 展示了一个使用这些新表的视图。

```
CREATE VIEW dbo.OrderDetail
AS
SELECT dtl.OrderDetailID,
dtl.CustomerOrderID,
prd.ProductName,
dtl.ProductPrice,
dtl.QuantitySold,
dtl.IsActive,
dtl.DateCreated,
dtl.DateModified,
dtl.DateDisabled
FROM dbo.OrderDetail_Normal dtl
INNER JOIN dbo.Product prd
ON dtl.ProductID = prd.ProductID;
```

**清单 13-15：使用规范化表的视图**

与清单 13-11 中的逻辑类似，你可以创建一个名为 `dbo.OrderDetail` 的视图，以便应用程序可以继续与新表交互。根据原始表和新表的目的，你可能无法将所有代码更新为使用这些新表。如果是这样，你可能不得不依赖触发器来更新这些表。


