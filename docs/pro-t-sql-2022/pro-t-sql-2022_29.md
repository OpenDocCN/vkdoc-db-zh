# 处理事务型数据的报告

## 概述

我们经常发现，对数据库的主要需求之一就是提供报告功能。本节将介绍如何利用参数，在基于不同条件获取相同信息时，提高代码的可重用性。还将提供一个示例，说明如何使用视图来创建一个标准化的数据集，该数据集可以在多个应用程序中重复使用。这个视图也可以在其他代码中被复用。一个包含另一个视图的视图被称为 `嵌套视图`。但需要注意的是，使用视图，尤其是嵌套视图时，应谨慎行事，本节稍后将对此进行讨论。

## 处理 OLTP 数据库的报告

大多数应用程序使用的数据库都旨在处理大量事务。这些数据库的整体设计被称为联机事务处理或 `OLTP`。这些数据库通常设计用于快速存储和检索数据。这种行为涉及大量的数据库写入操作。

然而，您的业务最终可能决定需要从这个运行着某个应用程序的同一数据库中获取汇总或报告信息。理想情况下，您应该只让 `OLTP` 应用程序（而不是报告应用程序）访问事务服务器上的数据库。您需要充分了解您的业务，以及任何报告查询可能对应用程序性能产生的影响。您或许可以运行非常简单的 `SELECT` 语句，这些语句只访问一小部分数据，而不会对您的应用程序产生负面影响。您应尽一切努力，限制为处理应用程序之外的数据请求而给 SQL Server 带来的额外负载。

当为了报告目的访问大量数据时，可能会导致主要应用程序或 `OLTP` 应用程序的性能下降。这包括生成执行计划所需的额外 `CPU` 资源，这些计划可能已被清除出缓存，以便为运行报告工作负载的查询腾出空间。在大量数据被移入内存用于报告后，应用程序在内存中的数据也可能从缓冲池中被清除。这些问题会级联发生，最终导致您的应用程序性能受到报告工作负载的影响。如果您发现自己需要直接从事务型数据库进行报告，请与您的管理团队沟通报告可能对应用程序产生的影响。

## 收集报告数据的挑战

当您需要为报告访问数据时，需要对同一数据库进行大量的读取操作。事务型数据并非为处理报告可能需要的庞大数据量而设计。它通常针对读取少量数据的查询进行了优化，例如为应用程序屏幕读取一个订单或一个客户。此外还有其他挑战。

在某些情况下，收集这些报告数据可能很简单。可能涉及少量的连接操作，或者底层逻辑并不复杂。您最终可能写好了一份报告，并且由于报告质量不错，可能会被要求创建更多报告。这些额外的报告可能涉及许多表，或者查询的整体逻辑可能更加复杂。这通常是因为这些表在设计之初并未考虑到报告需求。在某些情况下，您可能会发现您被要求对不存在的数据进行报告。

## 硬编码报表的问题

为了快速开发报表，最简单的方法通常是为每个单独的报表编写特定的逻辑。久而久之，这可能导致您最终拥有大量本应返回相同或类似结果的报表。最好的情况是结果相似，但底层代码不同。随着业务可能会更改某些应用程序的功能，情况会变得更加复杂。可能很容易识别出一些需要更新的报表，而其他报表则被遗漏并开始返回不准确的结果。

在某个时间点，您可能会收到一个请求，要求根据您的事务型数据生成报告。通常，这些请求最初可能只是一个简单的一次性数据请求。假设一个用户想要访问名字为 Stacy 的客户的订单信息。虽然这可能最初只是一个运行查询并获取一些数据的请求，但您可能会发现，最终您的用户希望这个报告可以随时运行，或者由特定的一群人运行。此时，您可能会编写一个类似代码清单 13-16 中的查询。

```sql
SELECT cus.FirstName,
       cus.LastName,
       ord.CustomerOrderID,
       ord.OrderNumber,
       prd.ProductName,
       dtl.QuantitySold,
       dtl.ProductPrice
FROM dbo.Customer cus
INNER JOIN dbo.CustomerOrder ord
    ON cus.CustomerId = ord.CustomerID
INNER JOIN dbo.OrderDetail dtl
    ON ord.CustomerOrderID = dtl.CustomerOrderID
INNER JOIN dbo.Product prd
    ON dtl.ProductID = prd.ProductID
WHERE cus.FirstName = 'Stacy';
```
*代码清单 13-16: Stacy 客户的订单*

虽然报表开发的具体细节超出了本书的范围，但我很少在工作中遇到不需要定期从事务型数据库中请求某种报告数据的情况。前面的查询没有显示存储过程，但您同样可以轻松地使用存储过程来选择这些数据。代码清单 13-16 中查询的问题在于，它是硬编码的，只返回名字为 Stacy 的客户的结果。

随着时间的推移，企业在分析数据时通常需要不同或额外的信息。这可能表现为需要一份新报表来显示另一个客户的信息。现在，您的用户想要一份关于 Myra 订单的报告。这也可能是一个用 Myra 的报告替换 Stacy 报告的请求。无论哪种情况，您都需要创建一个类似于代码清单 13-17 的查询。

```sql
SELECT cus.FirstName,
       cus.LastName,
       ord.CustomerOrderID,
       ord.OrderNumber,
       prd.ProductName,
       dtl.QuantitySold,
       dtl.ProductPrice
FROM dbo.Customer cus
INNER JOIN dbo.CustomerOrder ord
    ON cus.CustomerId = ord.CustomerID
INNER JOIN dbo.OrderDetail dtl
    ON ord.CustomerOrderID = dtl.CustomerOrderID
INNER JOIN dbo.Product prd
    ON dtl.ProductID = prd.ProductID
WHERE cus.FirstName = 'Myra';
```
*代码清单 13-17: Myra 客户的订单*

虽然您已经满足了为 Stacy 创建一份报表和为 Myra 创建一份新报表的要求，但您也增加了维护报表的开销。代码清单 13-16 和 13-17 中的查询是分开维护的。如果其中一个报告的数据提取方式发生变化，您需要记住另一个报告可能也需要修改。

## 使用参数提高可重用性

如果您发现自己处于这种情况，下一步就是想办法开始合并报告中的数据集。您首先需要识别哪些报告返回相似的结果。这些可以是处理相同业务应用程序或相同功能的报告。您需要创建一个查询，返回该应用程序或功能的所有相关信息。这将作为基础数据源，用于更新您现有的报告。代码清单 13-18 中的查询展示了编写一个可用于两份报告的查询的一种方法。

```sql
DECLARE @FirstName VARCHAR(40);
SELECT cus.FirstName,
       cus.LastName,
       ord.CustomerOrderID,
       ord.OrderNumber,
       prd.ProductName,
       dtl.QuantitySold,
       dtl.ProductPrice
FROM dbo.Customer cus
INNER JOIN dbo.CustomerOrder ord
    ON cus.CustomerId = ord.CustomerID
INNER JOIN dbo.OrderDetail dtl
    ON ord.CustomerOrderID = dtl.CustomerOrderID
INNER JOIN dbo.Product prd
    ON dtl.ProductID = prd.ProductID
WHERE cus.FirstName = @FirstName;
```
*代码清单 13-18: 通过多个连接获取订单信息*



### 参数化查询与代码复用

在本次查询中，您使用了一个名为 `@FirstName` 的参数。此参数允许您为 `@FirstName` 变量设置一个值。该值可以根据运行的报告而改变。在使用报表服务器时，可以为每个报告设置 `@FirstName` 的值。此类查询允许您为多种类型的报告使用同一套代码。这意味着您可以将同一个查询用于 Stacy 和 Myra，或是针对特定客户的报告请求。

在清单 [13-18] 中创建查询，可以使您的整体代码可复用。这可以解决从事务型数据生成报告的相关问题之一。从事务型数据进行报告时可能遇到的另一个问题是表之间逻辑的复杂性。在清单 [13-19] 中，关于客户和订单的信息已被扁平化，以更紧密地符合数据仓库的设计方式。

```sql
CREATE VIEW dbo.CustomerOrderInformation
AS
SELECT
    cus.FirstName,
    cus.LastName,
    cus.FirstName + ' ' + cus.LastName AS FullName,
    cus.[Address],
    cus.City,
    cus.Country,
    cus.PostalCode,
    cus.IsActive AS CustomerIsActive,
    ord.CustomerOrderID,
    ord.OrderNumber,
    prd.ProductName,
    dtl.QuantitySold,
    dtl.ProductPrice,
    prd.IsActive AS ProductIsActive
FROM dbo.Customer cus
INNER JOIN dbo.CustomerOrder ord
    ON cus.CustomerID = ord.CustomerID
INNER JOIN dbo.OrderDetail dtl
    ON ord.CustomerOrderID = dtl.CustomerOrderID
INNER JOIN dbo.Product prd
    ON dtl.ProductID = prd.ProductID;
```
*清单 13-19 客户订单信息*

## 视图设计与简化逻辑

与清单 [13-18] 中的查询类似，清单 [13-19] 中的查询旨在让您的 T-SQL 代码能适应各种场景复用。这种设计降低了查询或报告所需逻辑的复杂性。清单 [13-20] 中的查询与清单 [13-18] 中的查询功能相同。

```sql
CREATE PROCEDURE dbo.CustomerOrderByFirstName
    @FirstName VARCHAR(40)
AS
SELECT
    co.FirstName,
    co.LastName,
    co.CustomerOrderID,
    co.OrderNumber,
    co.ProductName,
    co.QuantitySold,
    co.ProductPrice
FROM dbo.CustomerOrderInformation co
WHERE co.FirstName = @FirstName;
```
*清单 13-20 从视图中获取的配方成分*

这两个查询的主要区别在于，清单 [13-20] 所需的总体逻辑要简单直接得多。这可以让技术能力较弱的用户也能够基于相同的数据集创建报告。虽然这有助于保持数据的一致性和易于访问，但这种方法可能并非性能最佳。在使用此方法访问数据进行报告时，请务必监控性能并确认报告是否能在超时前返回数据。

## 使用 Azure Synapse Link 进行近实时报告

SQL Server 2022 还可以通过使用 Azure Synapse Link for SQL Server，帮助您基于事务型数据库的近实时副本来创建报告。Azure Synapse Link 会以极小的当前生产系统开销将数据发送到 Azure Synapse Analytics。

为了为 SQL Server 设置 Azure Synapse Link，您需要在 Azure 门户的租户中创建一个 Azure Data Lake Storage Gen2 帐户，并确保您的数据库有一个主密钥。然后，您可以通过 Synapse Studio (`https://ms.web.azuresynapse.net/en/`) 将 SQL Server 2022 实例与 Azure Synapse Analytics 通过 Azure Synapse Link 连接起来。您需要在 Synapse Studio 中创建一个 SQL 池，然后为该池创建一个主密钥。完成后，您可以创建一个到 SQL Server 的链接服务。为了连接到您的 SQL Server 2022 实例，您需要一个自承载集成运行时。完成后，您应添加一个到 Azure Data Lake Storage Gen2 帐户的链接服务。现在，Synapse Studio 已链接到 SQL Server 2022 和 Azure Data Lake Storage Gen2 的服务，您可以在两个链接服务之间集成链接连接。设置此连接时，您可以选择要同步的表。成功发布这些更改后，您就可以启动 Azure Synapse Link。您需要等待几分钟，然后确认数据已复制到 Azure Synapse Analytics。

## 数据集管理与嵌套视图的性能影响

此基础数据集随后可用于您所有的报告中。一旦您将报告更新为使用此基础数据集，请确认该数据集能够满足您未来的报告需求。如果您希望转向数据仓库，这能让您了解所需数据的结构。

正如本节开头所述，使用一个视图从另一个视图获取数据的概念称为嵌套视图。嵌套视图的想法非常诱人，因为视图可以使 T-SQL 更易于阅读和理解。然而，这种可读性通常是以性能为代价的。当访问视图时，SQL Server 会在运行时执行 `SELECT` 语句。这本质上使视图成为执行您经常运行的查询的快捷方式。视图仍然需要确定是否存在执行计划并查找要返回的数据。即使视图的结果将用于另一个视图，情况也是如此。根据视图的设计，这甚至可能导致 SQL Server 多次访问同一个表。在嵌套到另一个视图中且两个视图都访问 `dbo.Customer` 表的视图情况下，SQL Server 将需要访问 `dbo.Customer` 表两次，一次用于嵌套视图，一次用于另一个视图。



