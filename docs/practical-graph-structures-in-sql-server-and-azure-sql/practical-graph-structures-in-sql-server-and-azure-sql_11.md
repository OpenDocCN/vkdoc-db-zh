# 实现组织树形结构：创建、查询与管理

## 存储过程定义

以下是一个名为 `SqlGraph.Company$ReportSales` 的存储过程，用于计算并按公司层级展示销售额：

```sql
CREATE OR ALTER PROCEDURE SqlGraph.Company$ReportSales
@DisplayFromNodeName VARCHAR(20)
AS
BEGIN
--output the Expanded Hierarchy...
WITH ExpandedHierarchy (ParentCompanyId, ChildCompanyId)
AS (
--gets all of the nodes of the hierarchy
SELECT Company.CompanyId AS ParentCompanyId,
Company.CompanyId AS ChildCompanyId
FROM SqlGraph.Company
UNION ALL
--joins back to the CTE to recursively retrieve the rows
--note that TreeLevel is incremented on each iteration
SELECT ExpandedHierarchy.ParentCompanyId,
ToCompany.CompanyId
FROM ExpandedHierarchy,
SqlGraph.Company AS FromCompany,
SqlGraph.ReportsTo,
SqlGraph.Company AS ToCompany
WHERE ExpandedHierarchy.ChildCompanyId = FromCompany.CompanyId
AND MATCH(FromCompany-(ReportsTo)->ToCompany)
),
--using the hierarchy returning function, get only the nodes
--that you desire
FilterAndSweeten AS
(
SELECT ExpandedHierarchy.*,
CompanyHierarchyDisplay.Hierarchy,
CompanyHierarchyDisplay.HierarchyDisplay,
CompanyHierarchyDisplay.Name
FROM   ExpandedHierarchy
JOIN [SqlGraph].[Company$ReturnHierarchy]
(@DisplayFromNodeName) AS CompanyHierarchyDisplay
ON CompanyHierarchyDisplay.CompanyId =
ExpandedHierarchy.ParentCompanyId
)
,--get totals for each Company for the aggregate
CompanyTotals
AS (SELECT CompanyId,
SUM(cast(Amount as decimal(20,2))) AS TotalAmount
FROM SqlGraph.Sale
GROUP BY CompanyId),
--aggregate each Company for the Company
Aggregations AS
--perform the math that gives every node their sums
(SELECT FilterAndSweeten.ParentCompanyId,
SUM(CompanyTotals.TotalAmount) AS TotalSalesAmount,
--these are unique and could be in the GROUP BY,
--but this works great and makes it clear we are
--grouping on the ParentCompanyId only
MAX(hierarchy) AS Hierarchy,
MAX(hierarchyDisplay) AS HierarchyDisplay,
MAX(Name) as Name
FROM FilterAndSweeten
JOIN CompanyTotals
ON CompanyTotals.CompanyId =
FilterAndSweeten.ChildCompanyId
GROUP BY FilterAndSweeten.ParentCompanyId)
--display the data...
SELECT Aggregations.ParentCompanyId,
Aggregations.Name,
Aggregations.TotalSalesAmount,
Aggregations.hierarchyDisplay
FROM Aggregations
ORDER BY Aggregations.hierarchy;
END;
```

## 执行与结果

现在，您可以执行此存储过程，按公司、区域及总公司查看销售额，例如执行：

```sql
EXECUTE SqlGraph.Company$ReportSales 'Company HQ';
```

返回结果如下：

```
ParentCompanyId hierarchyDisplay              TotalSalesAmount
--------------- ----------------------------- ------------------
1               Company HQ                    81.25
2               --> Maine HQ                  51.25
7               --> --> Portland Branch       22.50
8               --> --> Camden Branch         28.75
3               --> Tennessee HQ              30.00
4               --> --> Nashville Branch      3.75
5               --> --> Knoxville Branch      10.00
6               --> --> Memphis Branch        16.25
```

## 代码解析

让我们分解这段代码，看看它是如何工作的，因为这种技术在处理树形结构时非常有用。首先是扩展层级（expanded hierarchy）。其第一部分是将树中的每个节点与自身关联：

```sql
--output the Expanded Hierarchy...
WITH ExpandedHierarchy (ParentCompanyId, ChildCompanyId)
AS (
--gets all of the nodes of the hierarchy
SELECT Company.CompanyId AS ParentCompanyId,
Company.CompanyId AS ChildCompanyId
FROM SqlGraph.Company
)
SELECT *
FROM  ExpandedHierarchy
ORDER BY ExpandedHierarchy.ParentCompanyId;
```

执行此查询返回：

```
ParentCompanyId ChildCompanyId
--------------- --------------
1               1
2               2
3               3
4               4
5               5
6               6
7               7
8               8
```

这是必需的，因为 `MATCH` 子句要返回这些行，需要与同一节点存在循环关系。

接下来，扩展层级查询的第二半部分使用了递归关系，尽管它实际上只进行了一次迭代。`ExpandedHierarchy` 中的每个节点在查询的第二半部分被展开：

```sql
WITH ExpandedHierarchy (ParentCompanyId, ChildCompanyId)
AS (
--gets all of the nodes of the hierarchy
SELECT Company.CompanyId AS ParentCompanyId,
Company.CompanyId AS ChildCompanyId
FROM SqlGraph.Company
UNION ALL
SELECT ExpandedHierarchy.ParentCompanyId,
ToCompany.CompanyId
FROM ExpandedHierarchy,
SqlGraph.Company AS FromCompany,
SqlGraph.ReportsTo,
SqlGraph.Company AS ToCompany
WHERE ExpandedHierarchy.ChildCompanyId = FromCompany.CompanyId
AND MATCH(FromCompany-(ReportsTo)->ToCompany)
)
SELECT *
FROM   ExpandedHierarchy
--don't return the rows from the first query JUST for this
--example explanation only
WHERE ExpandedHierarchy.ParentCompanyId <> ExpandedHierarchy.ChildCompanyId
ORDER BY ParentCompanyId;
```

这返回：

```
ParentCompanyId ChildCompanyId
--------------- --------------
1               2
1               3
1               4
1               5
1               6
1               7
1               8
2               7
2               8
3               4
3               5
3               6
```

您可以看到，1 与其他所有节点关联，然后 2 和 3（`Tennessee` 和 `Maine`）关联到它们的子行。因此，现在合并两组输出后，每一行都与自身及其子行关联。

查询的其余部分相当直接。`FilterAndSweeten` CTE 只是连接 `ExpandedHierarchy`，并仅过滤那些作为您传递给 `Company$ReturnHierarchy` 的公司名称的输出的父行。它处理过滤并添加将要输出的数据。

```sql
FilterAndSweeten AS
(
SELECT ExpandedHierarchy.*,
CompanyHierarchyDisplay.Hierarchy,
CompanyHierarchyDisplay.HierarchyDisplay
FROM  ExpandedHierarchy
JOIN  [SqlGraph].[Company$ReturnHierarchy]
(@DisplayFromNodeName) AS CompanyHierarchyDisplay
ON CompanyHierarchyDisplay.CompanyId =
ExpandedHierarchy.ParentCompanyId
)
```

`CompanyTotals` CTE 只是获取每个独立公司的销售数据并进行求和：

```sql
,--get totals for each Company for the aggregate
CompanyTotals
AS (SELECT CompanyId,
SUM(cast(Amount as decimal(20,2))) AS TotalAmount
FROM SqlGraph.Sale
GROUP BY CompanyId),
--aggregate each Company for the Company
```

然后，您将数据聚合到 `ParentCompany` 值，包括最终显示所需的内容：

```sql
Aggregations AS
(SELECT FilterAndSweeten.ParentCompanyId,
SUM(CompanyTotals.TotalAmount) AS TotalSalesAmount,
MAX(hierarchy) AS hierarchy,
MAX(hierarchyDisplay) AS hierarchyDisplay
FROM FilterAndSweeten
JOIN CompanyTotals
ON CompanyTotals.CompanyId =
FilterAndSweeten.ChildCompanyId
GROUP BY FilterAndSweeten.ParentCompanyId)
```

最后，数据在此额外步骤中显示，只是为了使排序更容易：

```sql
--display the data...
SELECT Aggregations.ParentCompanyId,
Aggregations.hierarchyDisplay,
Aggregations.TotalSalesAmount
FROM Aggregations
ORDER BY Aggregations.hierarchy
END;
```

显然，代码中有很多内容，但像大多数 SQL 代码一样，一旦深入理解它在做什么，就相当直接了，并且它以非常强大的方式使用了关系引擎和图形引擎。

## 总结

在本章中，我展示了您需要实现自己组织树形结构的大部分代码：创建、修改、查询甚至销毁层次结构的方法。

到目前为止，您使用的是一个非常小的数据集，但在下一章中，您将把数据量增加五个数量级，并探索构建更大数据集的方法（假设您的生产树中节点超过八个）。


