# 10-2. 配置域和值分布

## 问题
您希望从 SQL Server 源表中获取域和值分布数据。

## 解决方案
使用 T-SQL 函数返回域和值分布信息。本质上，这意味着发现包含每个值的记录数以及该值在数据集中的百分比。

表 10-3 中的代码片段为您提供了数据表中域分布的快照。这两个代码示例都对 `CarSales.dbo.Stock` 表进行了配置。

表 10-3. SQL Server 数据源中的域和值分布

| 分布 | 代码 | 注释 |
| --- | --- | --- |
| 域分布 | `SELECT TOP (100) PERCENT Marque, COUNT(Marque) AS NumberOfMarques,` | 此代码片段结合分析了： |
|  | `COUNT(Marque)/` | 域值； |
|  | `(SELECT CAST(COUNT(*) AS NUMERIC(18,8))` | 包含每个值的记录数； |
|  | `FROM CarSales.dbo.Stock)` |  |
|  | `* 100 AS DomainPercent` | 每个值在数据集中的百分比。 |
|  | `FROM          CarSales.dbo.Stock` |  |
|  | `GROUP BY      Marque` |  |
|  | `ORDER BY      Marque DESC;` |  |
| 数值分布 | `SELECT      ID, COUNT(ID) AS NumberOfValues` | 此代码片段结合分析了： |
|  | `FROM        CarSales.dbo.Stock` | 数值； |
|  | `GROUP BY    ID` |  |
|  | `ORDER BY    NumberOfValues DESC;` | 包含每个值的记录数。 |

## 工作原理
值分布简单来说就是获取以下指标：
- 包含指定字段中每个值（文本或数值）的记录数。
- 包含指定字段中每个值（文本或数值）的记录百分比。

对包含大量不同值的集合应用值分布分析本身可能不是特别有用。我建议这种方法最适合用于分析包含少量不同值的字段——最常见的是查找或参考值——其中数据分布是数据可靠性的指标。

## 提示、技巧和陷阱
*   此解决方案中的方法也可以应用于非 SQL Server 数据源或链接服务器。
*   配置链接服务器源可能会非常慢，虽然这对于初步数据分析有用，但作为常规 ETL 流程的一部分可能不实用。
*   您可以在配方 10-5 中了解如何存储配置请求的输出。

