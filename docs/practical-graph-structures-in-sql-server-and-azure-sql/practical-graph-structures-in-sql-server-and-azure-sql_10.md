# SQL 图数据操作：插入、删除与层级处理示例

## 数据准备与层级展示

首先，我们插入一些测试数据来演示删除操作。这些行不包括活动行，因为那会以与本示例无关的方式限制删除操作。

```sql
EXEC SqlGraph.Company$Insert @Name = 'Georgia HQ',
@ParentCompanyName = 'Company HQ';
EXEC SqlGraph.Company$Insert @Name = 'Atlanta Branch',
@ParentCompanyName = 'Georgia HQ';
EXEC SqlGraph.Company$Insert @Name = 'Dalton Branch',
@ParentCompanyName = 'Georgia HQ';
EXEC SqlGraph.Company$Insert @Name = 'Texas HQ',
@ParentCompanyName = 'Company HQ';
EXEC SqlGraph.Company$Insert @Name = 'Dallas Branch',
@ParentCompanyName = 'Texas HQ';
EXEC SqlGraph.Company$Insert @Name = 'Houston Branch',
@ParentCompanyName = 'Texas HQ';
```

现在，查看树中的节点：

```sql
SELECT HierarchyDisplay
FROM SqlGraph.CompanyHierarchyDisplay
ORDER BY Hierarchy;
```

层级结构现在如下所示：

```
HierarchyDisplay

Company HQ
--> Georgia HQ
--> --> Atlanta Branch
--> --> Dalton Branch
--> Maine HQ
--> --> Camden Branch
--> --> Portland Branch
--> Tennessee HQ
--> --> Knoxville Branch
--> --> Memphis Branch
--> --> Nashville Branch
--> Texas HQ
--> --> Dallas Branch
--> --> Houston Branch
```

尝试仅使用默认参数（保留子行）删除 `Georgia HQ`，使用以下调用：

```sql
EXEC SqlGraph.Company$Delete @Name = 'Georgia HQ';
```

因为您将边约束创建为 `NO ACTION`，当您尝试删除父节点时，此操作会失败并显示以下错误消息：

```
Msg 547, Level 16, State 0, Procedure SqlGraph.Company$Delete, Line 60
The DELETE statement conflicted with the EDGE REFERENCE constraint "EC_ReportsTo$DefinesParentOf". The conflict occurred in database "GraphDBTests", table "SqlGraph.ReportsTo".
```

如果您尝试删除叶节点，因为它图中没有依赖项，所以会成功。

```sql
--Delete Atlanta
EXEC SqlGraph.Company$Delete @Name = 'Atlanta Branch';
```

检查结构，您会看到 `Atlanta Branch` 行已消失。接下来，尝试在删除指定节点的同时删除其子行：

```sql
EXEC SqlGraph.Company$Delete @Name = 'Georgia HQ',
@DeleteChildNodesFlag = 1;
```

现在您会看到 `Georgia HQ` 和 `Dalton Branch` 行都已消失。

```
HierarchyDisplay

Company HQ
--> Maine HQ
--> --> Camden Branch
--> --> Portland Branch
--> Tennessee HQ
--> --> Knoxville Branch
--> --> Memphis Branch
--> --> Nashville Branch
--> Texas HQ
--> --> Dallas Branch
--> --> Houston Branch
```

接下来，让我们看看 `@ReparentChildNodesToParentFlag` 的作用。执行以下代码会删除 `Texas HQ` 行，但现在 `Dallas Branch` 和 `Houston Branch` 节点将作为 `Company HQ` 节点的子节点上移。

```sql
EXEC SqlGraph.Company$Delete @Name = 'Texas HQ',
@ReparentChildNodesToParentFlag = 1;
```

查看数据，您可以看到 `Dallas Branch` 和 `Houston Branch` 节点现在是 `Company HQ` 的子节点：

```
HierarchyDisplay

Company HQ
--> Dallas Branch
--> Houston Branch
--> Maine HQ
--> --> Camden Branch
--> --> Portland Branch
--> Tennessee HQ
--> --> Knoxville Branch
--> --> Memphis Branch
--> --> Nashville Branch
```

最后，清理此示例中剩余的节点：

```sql
EXEC SqlGraph.Company$Delete @Name = 'Dallas Branch';
EXEC Graph.Company$Delete @Name = 'Houston Branch';
```

## 树形输出代码

在本节中，您将学习一些在处理树结构时很方便的额外技术。每种技术都将被编码到一个对象中，并且（并非巧合地）将成为下一章性能测试代码的基础，届时您将使用比此处所用大得多的数据集。

这些过程将执行以下操作：

*   **返回部分树**：这将扩展本章前面创建的 `CompanyHierarchyDisplay` 视图的功能，让您选择起始节点，然后返回位于其下方的节点。
*   **确定子节点是否存在**：这是在构建安全应用程序时通常需要的一段代码。您需要知道某个节点是否存在于树的任何层级。
*   **聚合每一级别的子活动**：从根到叶，汇总树中每个节点的所有子节点的销售活动总和。

### 返回部分树

这个返回表的用户定义函数是本章前面创建的视图的扩展。不同之处在于，它允许您从特定节点开始，而不是总是返回整棵树。

```sql
CREATE OR ALTER FUNCTION SqlGraph.Company$ReturnHierarchy
(
@CompanyName varchar(20)
)
RETURNS @Output TABLE (CompanyId INT, Name VARCHAR(20),
Level INT, Hierarchy NVARCHAR(4000),
IdHierarchy NVARCHAR(4000),
HierarchyDisplay NVARCHAR(4000))
AS
BEGIN
--get the identifier for the node you want to start with
DECLARE @CompanyId int, @NodeName nvarchar(max)
SELECT  @CompanyId = CompanyId,
@Nodename = Name
FROM   SqlGraph.Company
WHERE  Name = @CompanyName;
;WITH baseRows as
(
--include node that you are looking for
SELECT @companyId as CompanyId, @NodeName as Name,
1 as Level,
'\' + Cast(@companyId as nvarchar(10)) + '\' as
IdHierarchy, --\ delimited but with id num
'\' + @NodeName AS Hieararchy
UNION ALL
SELECT
LAST_VALUE(ToCompany.CompanyId)
WITHIN GROUP (GRAPH PATH) AS CompanyId,
LAST_VALUE(ToCompany.Name)
WITHIN GROUP (GRAPH PATH) AS NodeName,
1+COUNT(ToCompany.Name)
WITHIN GROUP (GRAPH PATH) AS levels,
'\' + CAST(FromCompany.CompanyId as NVARCHAR(10))+
'\' + STRING_AGG(cast(ToCompany.CompanyId as
nvarchar(10)), '\')
WITHIN GROUP (GRAPH PATH) + '\' AS Idhierarchy,
'\' +  FromCompany.NAME + '\' +
STRING_AGG(ToCompany.Name, '\')
WITHIN GROUP (GRAPH PATH) + '\' AS hierarchy
FROM
SqlGraph.Company AS FromCompany,
SqlGraph.ReportsTo FOR PATH AS ReportsTo,
SqlGraph.Company FOR PATH AS ToCompany
WHERE
MATCH(SHORTEST_PATH(FromCompany(-(ReportsTo)
->ToCompany)+))
--start the processing from the parameter's companyId
AND FromCompany.CompanyId = @companyId
)
INSERT INTO @Output
SELECT *, REPLICATE('--> ',LEVEL - 1) + Name
AS HierarchyDisplay
FROM  BaseRows
RETURN;
END;
```

如果您只想查看 `Tennessee HQ` 的数据树，可以使用

```sql
SELECT *
FROM   SqlGraph.Company$ReturnHierarchy('Tennessee HQ');
```

这将返回三行，其输出与仅显示子树的视图相同。


### 确定是否存在子节点

以下过程将允许您查询一个公司是否是另一个公司行的子节点。这在许多情况下都很有用，例如在安全场景中，当您开始拥有继承的组权限时，您就可以查看某个用户是否在继承链的任何级别上属于某个组。

```sql
CREATE OR ALTER FUNCTION SqlGraph.Company$CheckForChild
(
@CompanyName varchar(20),
@CheckForChildOfCompanyName varchar(20)
)
RETURNS bit
AS
BEGIN
-- 获取要检查的公司的 ID 值
DECLARE @CompanyId INT, @ChildFlag BIT = 0;
SELECT @CompanyId = CompanyId
FROM   SqlGraph.Company
WHERE  Name = @CompanyName;
-- 获取要检查其是否为父节点的公司的 ID
DECLARE @CheckForChildOfCompanyId int;
SELECT  @CheckForChildOfCompanyId = CompanyId
FROM   SqlGraph.Company
WHERE  Name = @CheckForChildOfCompanyName;
-- 查询结构
;WITH baseRows AS
(
-- 获取与 @checkForChildOfCompanyId 的关系
SELECT LAST_VALUE(ToCompany.CompanyId) WITHIN GROUP
(GRAPH PATH) AS ChildCompanyId
FROM
SqlGraph.Company AS FromCompany,
SqlGraph.ReportsTo FOR PATH AS ReportsTo,
SqlGraph.Company FOR PATH AS ToCompany
WHERE  MATCH(SHORTEST_PATH(FromCompany(-(ReportsTo)
->ToCompany)+))
AND FromCompany.CompanyId = @CheckForChildOfCompanyId
)
-- 然后筛选查看传入的行是否匹配
SELECT @ChildFlag = 1
FROM  BaseRows
WHERE childCompanyId = @CompanyId;
RETURN @ChildFlag;
END;
```

了解数据后，您现在可以使用上一节中的新函数

```sql
SELECT HierarchyDisplay
FROM   SqlGraph.Company$ReturnHierarchy('Company HQ' )
ORDER BY Hierarchy;
```

该函数返回

```
HierarchyDisplay

Company HQ
--> Maine HQ
--> --> Portland Branch
--> --> Camden Branch
--> Tennessee HQ
--> --> Nashville Branch
--> --> Knoxville Branch
--> --> Memphis Branch
```

您可以这样测试该函数。以 `Camden Branch` 为基础检查几种场景：

```sql
SELECT (CASE SqlGraph.Company$CheckForChild
('Camden Branch','Company HQ')
WHEN 1 THEN 'Yes' ELSE 'No' END)
AS Camden_to_Company,
(CASE SqlGraph.Company$CheckForChild
('Camden Branch','Maine HQ')
WHEN 1 THEN 'Yes' ELSE 'No' END) AS Camden_to_Maine,
(CASE SqlGraph.Company$CheckForChild
('Camden Branch','Tennessee HQ')
WHEN 1 THEN 'Yes' ELSE 'No' END)
AS Camden_to_Tennessee;
```

这会返回

```
Camden_to_Company Camden_to_Maine Camden_to_Tennessee
----------------- --------------- -------------------
Yes               Yes             No
```

如您所见，`Camden Branch` 是 `Company HQ` 和 `Maine HQ` 的子节点，但不是 `Tennessee Branch` 的子节点。在下一章中，您将在几种不同的树实现上使用此函数，以比较在更大的树中每个操作所需的时间。

### 聚合每个级别的子节点活动

接下来要实现的代码是能够对树进行聚合。例如，假设您有以下销售数据：

```
Company HQ
--> Maine HQ
--> --> Portland Branch  $1
--> --> Camden Branch    $1
--> Tennessee HQ
--> --> Nashville Branch $1
--> --> Knoxville Branch $1
--> --> Memphis Branch   $1
```

您需要能够不仅说明叶节点各有 1 美元的销售额；您还需要说明 `Tennessee HQ` 的销售额为 3 美元，`Maine` 为 2 美元，整个组织为 5 美元。查看上面文本中的层级结构，您可能认为这是一个相当简单的操作（相对而言，它确实是，但这是一段需要解释的、相当可观的代码）。

只需获取层级结构的较低级别，然后获取父级并求和，接着再获取下一个父级并求和。问题是，在现实世界中，层级结构很少如此平衡，而且很可能不那么浅（最可能的情况是，您的实际场景还会加入销售人员及其销售额，这将使实现更具挑战性）。

处理此问题的方法是采用递归操作，然后将其转换为关系型过程。

此操作的基础是我所谓的 `expanded hierarchy`（我在 Ralph Kimball 的数据仓库课程中学到了基础技术。关于 Kimball 辅助表方法的更多内容将在下一章介绍）。这基本上是获取层级结构中每个作为父节点的节点，并获取它们的整个层级结构。它使用您之前使用的代码来获取一行的所有子节点，但不是筛选到单个节点，而是为 *每个* 节点返回其层级结构。然后，每个节点都会将其所有子行添加到输出/表中。

例如，考虑树的以下子图：

```
--> Tennessee HQ
--> --> Nashville Branch $1
--> --> Knoxville Branch $1
--> --> Memphis Branch   $1
```

代码将输出这个扩展的层级结构，如下所示：

```
CompanyName           IncludesCompanyName
--------------------- --------------------
Tennessee HQ          Tennessee HQ
Tennessee HQ          Nashville Branch
Tennessee HQ          Knoxville Branch
Tennessee HQ          Memphis Branch
Nashville Branch      Nashville Branch
Knoxville Branch      Knoxville Branch
Memphis Branch        Memphis Branch
```

有了这组数据，您就可以使用 `IncludesCompanyName` 连接到活动数据以获取活动总和，然后您可以按 `CompanyName` 列进行分组以获取不同节点的销售额。由此输出的行数可能相当可观，但正如您将在下一章中看到的，即使是超过 200,000 个节点，对于我的台式计算机来说也不算太难处理。


