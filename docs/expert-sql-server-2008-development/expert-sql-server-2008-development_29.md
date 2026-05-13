# 第一章 数据库世界的软件开发方法论

## 性能 vs. 设计 vs. 现实

架构纯粹主义者可能会争辩说，性能不应影响应用程序设计；它只是一个实现细节，可以在代码层面解决。但我们这些曾在一线奋战、不得不面对设计糟糕的架构所带来的现实的人知道，事实并非如此。

事实上，性能在几乎所有应用中都与设计密不可分。试想那些发送过多数据或需要太多客户端请求才能将所需信息填满用户屏幕的“过于絮叨的”接口，或者那些每次用户请求都必须返回中央服务器以执行关键功能的应用。在许多情况下，这些性能缺陷可以在设计阶段就被识别并修复，从而避免它们在日后显现。然而，重要的是不要在这方面走得太远：设计不应为了规避可能永远不会发生的预期“性能问题”而变得过度扭曲。

### 应用逻辑

如果说数据逻辑*绝对*属于数据库，业务逻辑*可能*在数据库中占有一席之地，那么应用逻辑就是应该尽可能远离核心数据的一套规则。构成应用逻辑的规则包括诸如用户界面行为、字符串和数字格式化规则、本地化以及其他通常与用户界面相关的问题。考虑到前面讨论的应用层次结构（一个数据库可能被多个应用共享，而这些应用又可能被多个用户界面共享），很明显，将用户界面数据与应用程序或核心业务数据混在一起会引发严重的耦合问题，并最终降低数据共享的可能性。

请注意，我并不是说总是应该避免在数据库中持久化与用户界面相关的实体。对于许多应用来说，这样做当然有其道理。我所警告的是，未能清晰区分用户界面元素与应用程序其余数据所带来的风险。

在可能的情况下，务必创建不同的表（最好放在不同的模式中，甚至完全不同的数据库中），以存储纯粹与应用相关的数据。这将使你能够尽可能保持应用与数据的解耦。

### “对象-关系阻抗不匹配”

使得信息在面向对象系统和关系数据库之间难以迁移的主要障碍在于，从基本设计的角度来看，这两种系统类型是不兼容的。关系数据库采用规范化规则设计，通过将信息拆分成由键相互关联的表来帮助确保数据完整性。而面向对象系统在这方面则往往宽松得多。对象中包含的数据虽然相关，但可能不会在数据库的单个表中建模，这种情况相当常见。

例如，考虑下面这个零售系统中产品的类：

```
class Product
{
    string UPC;
    string Name;
    string Description;
    decimal Price;
    datetime UpdatedDate;
}
```

乍一看，这个类中定义的字段似乎彼此紧密相关，人们可能会期望它们总是属于数据库中的单个表。然而，这个产品类可能仅代表了某个产品在最后更新日期那一刻的视图。

在数据库中，数据可能建模如下：

```
CREATE TABLE Products
(
    UPC varchar(20) PRIMARY KEY,
    Name varchar(50)
);

CREATE TABLE ProductHistory
(
    UPC varchar(20) FOREIGN KEY REFERENCES Products (UPC),
    Description varchar(100),
    Price decimal,
    UpdatedDate datetime,
    PRIMARY KEY (UPC, UpdatedDate)
);
```



这里需要注意的关键点是，数据的**对象表示**可能与其在数据库中的**建模方式**毫无关联，反之亦然。面向对象的世界和关系型世界各有其目标与达成目标的手段，开发者不应试图强行将两者糅合在一起，否则可能会导致功能受损。

## 表真的只是伪装的类吗？

一些入门级的数据库教材有时会提到，可以将表比作类，将行比作类的实例（即对象）。初看之下，这很有道理；表和类一样，为实体定义了一组属性（称为列）。它们也可以（粗略地）为实体定义一组方法，其形式是触发器。

然而，相似之处到此为止。面向对象系统的关键基础是**继承**和**多态**，这两者在 SQL 数据库中即使不是完全无法表示，也是极其困难的。此外，数据库和面向对象系统中访问相关信息的路径也大相径庭。面向对象系统中的一个实体可以“拥有”一个子实体，通常使用“点”符号来访问。例如，一个书店对象可能拥有一个书籍集合：

```sql
Books = BookStore.Books;
```

在这个面向对象的例子中，书店“拥有”书籍。但在 SQL 数据库中，这种实体间的关系是通过**键**来维持的，其中子实体指向其父实体。关系并非书店拥有书籍，而是反过来：书籍通过一个外键指向书店：

```sql
CREATE TABLE BookStores
(
    BookStoreId int PRIMARY KEY
);

CREATE TABLE Books
(
    BookStoreId int REFERENCES BookStores (BookStoreId),
    BookName varchar(50),
    Quantity int,
    PRIMARY KEY (BookStoreId, BookName)
);
```

虽然面向对象和 SQL 表示法可以存储相同的信息，但其实现方式差异如此之大，以至于说一个表代表一个类（至少在当前的 SQL 数据库中）是不合理的。

## 对继承建模

在面向对象设计中，对象之间可以存在两种基本关系：“**有-一个**”关系，即一个对象“拥有”另一个对象的实例（例如，书店拥有书籍）；以及“**是-一个**”关系，即一个对象的类型是另一个对象的子类型（或*子类*）（例如，书店是一种商店）。在 SQL 数据库中，“有-一个”关系很常见，而“是-一个”关系则可能难以实现。

考虑一个名为“Products”的表，它可能代表某公司所有可供销售产品的实体类。该表可能包含通常属于产品的列（属性），如“价格”、“重量”和“UPC”。这些通用属性适用于该公司销售的所有产品。然而，公司可能销售许多产品子类，每个子类都有自己特定的额外属性集。例如，如果公司同时销售书籍和 DVD，书籍可能具有“页数”属性，而 DVD 则可能具有“时长”和“格式”属性。

在面向对象的世界中，子类化是通过继承模型实现的，这些模型在诸如 C#等语言中实现。在这些模型中，一个给定的实体可以是一个子类的成员，同时在处理该层级的代码中，它仍被普遍视为*超类*的成员。这使得在销售点应用程序的结账部分，可以无缝地处理书籍和 DVD，同时为每个子类保留独立的属性，以供应用程序的其他需要部分使用。

在 SQL 数据库中，对继承建模可能很棘手。下面的代码清单展示了一种可行的方法：

```sql
CREATE TABLE Products
(
    UPC int NOT NULL PRIMARY KEY,
    Weight decimal NOT NULL,
    Price decimal NOT NULL
);

CREATE TABLE Books
(
    UPC int NOT NULL PRIMARY KEY
        REFERENCES Products (UPC),
    PageCount int NOT NULL
);

CREATE TABLE DVDs
(
    UPC int NOT NULL PRIMARY KEY
        REFERENCES Products (UPC),
    LengthInMinutes decimal NOT NULL,
    Format varchar(4) NOT NULL
        CHECK (Format IN ('NTSC', 'PAL'))
);
```

使用此代码清单创建的数据库结构如图 1-2 所示。

*图 1-2\. 在 SQL 数据库中对继承建模*

尽管该模型成功地将书籍和 DVD 确立为产品的子类型，但它存在几个严重问题。首先，就目前情况而言，该模型无法强制子类型的唯一性。单个 UPC 可以同时属于 Books 和 DVDs 两个子类型。这在现实世界的大多数情况下都显得不太合理（尽管可能有一本书附带一张 DVD，那样的话这个模型或许说得通）。

另一个问题是属性的访问。在面向对象系统中，子类自动继承其超类的所有属性；一个书籍实体将包含书籍和通用产品的所有属性。然而，在此处提出的模型中情况并非如此。要获取书籍或 DVD 的通用产品属性，需要连接回`Products`表。这实际上破坏了处理子类型的完整意义。

解决这些问题并非不可能，但需要费些功夫。保证子类型唯一性的一种方法是在超类型中填充一个额外的属性来标识每个实例的子类型。下表展示了如何实现此解决方案：

```sql
CREATE TABLE Products
(
    UPC int NOT NULL PRIMARY KEY,
    Weight decimal NOT NULL,
    Price decimal NOT NULL,
    ProductType char(1) NOT NULL
        CHECK (ProductType IN ('B', 'D')),
    UNIQUE (UPC, ProductType)
);

CREATE TABLE Books
(
    UPC int NOT NULL PRIMARY KEY,
    ProductType char(1) NOT NULL
        CHECK (ProductType = 'B'),
    PageCount int NOT NULL,
    FOREIGN KEY (UPC, ProductType) REFERENCES Products (UPC, ProductType)
);

CREATE TABLE DVDs
(
    UPC int NOT NULL PRIMARY KEY,
    ProductType char(1) NOT NULL
        CHECK (ProductType = 'D'),
    LengthInMinutes decimal NOT NULL,
    Format varchar(4) NOT NULL
        CHECK (Format IN ('NTSC', 'PAL')),
    FOREIGN KEY (UPC, ProductType) REFERENCES Products (UPC, ProductType)
);
```

通过将子类型定义为超类型的一部分，可以创建一个`UNIQUE`约束，使 SQL Server 能够强制每个超类型实例只允许存在一个子类型。在每个子类型表中，通过`ProductType`列上的`CHECK`约束进一步强化了这种关系，确保只允许插入正确的产品类型。

通过使用索引视图和`INSTEAD OF`触发器，可以进一步扩展此方法。可以为每个子类型创建一个视图，该视图封装了检索超类型属性所需的连接。通过创建视图来隐藏连接，使用者无需感知子类型/超类型关系，从而解决了属性访问问题。索引有助于提高性能，而触发器则允许视图可更新。

在 SQL 数据库中，几乎可以表示任何能在面向对象系统中体现的关系，但数据库开发者理解其中的复杂性至关重要。将面向对象数据（正确地）映射到数据库通常并非易事，对于复杂的对象图来说，这可能是一个相当大的挑战。

## “大量空列”继承模型



