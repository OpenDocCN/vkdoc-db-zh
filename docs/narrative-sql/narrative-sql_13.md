# 6. 使用 ORDER BY 和 LIMIT 排序数据

数据分析要求分析师对数据进行排序，以提供清晰度、洞察力和有效沟通。SQL 中的 `ORDER BY` 子句使模式和趋势更加可见，帮助分析师优先处理发现并检测异常值。当你适当地排序数据时，数据中的模式和关系会变得更清晰，从而让你能够做出更好的决策并采取行动。例如，考虑一个没有任何顺序的产品销售列表。识别顶级产品、跟踪趋势甚至发现异常情况的任务将非常具有挑战性。数据排序提供了清晰度，并使你能够快速识别模式并做出明智的决策。`LIMIT` 子句通过专注于最相关的数据点（例如顶级表现者或最近的条目）来补充这一点，因此更容易专注于最重要的内容。

## ORDER BY 简介

SQL 中的 `ORDER BY` 子句用于根据一列或多列对结果进行排序。通过控制数据显示的顺序，你可以更容易地识别趋势、顶级表现者或特定模式。`ORDER BY` 子句的基本语法如下：

```sql
SELECT column1, column2, ...
FROM table_name
ORDER BY column1 [ASC | DESC], column2 [ASC | DESC];
```

这里，`column1` 和 `column2` 是你想要排序的列。你可以选择 `ASC` 表示升序（默认）或 `DESC` 表示降序。也可以按多列排序。需要注意的是，如果两行在 `column1` 中的值相同，SQL 将使用 `column2` 作为次要排序条件。这种灵活的排序有助于更有效地呈现数据。使用 `ORDER BY`，你可以快速创建结构良好的输出，无论你想要一个畅销商品列表还是按雇佣日期排列的员工列表。

## 在现实场景中排序数据

`ORDER BY` 子句对于在各种现实场景中有效地组织 SQL 查询数据至关重要。表 `6-1` 说明了如何使用 `ORDER BY` 子句来识别畅销产品、优先处理订单、突出显示高分学生或组织客户反馈。这些场景展示了 `ORDER BY` 通过按多列排序以实现特定目标（例如项目排名、识别趋势或优先处理紧急事项）的灵活性。对于每个场景，都包含一个 SQL 查询示例，向你展示如何构建 `ORDER BY` 子句。表 `6-1` 还解释了每个查询的结构以及它产生的输出类型。

表 6-1：`ORDER BY` 子句

| 现实场景 | ORDER BY 子句的作用 | SQL 查询示例 | 描述 |
| --- | --- | --- | --- |
| 电子商务商店中的畅销产品 | 识别最畅销的产品。 | `SELECT product_name, total_sales FROM products ORDER BY total_sales DESC;` | 按 `total_sales` 降序检索最畅销的产品。 |
| 公司中的员工工资 | 显示降序排列的工资。 | `SELECT employee_name, salary FROM employees ORDER BY salary DESC;` | 列出所有员工，按工资从高到低排序。 |
| 按日期排序的客户反馈 | 最新的反馈优先显示；有助于处理当前的客户关注点或问题。 | `SELECT customer_id, feedback, feedback_date FROM feedbacks ORDER BY feedback_date DESC;` | 显示所有客户反馈，最新的条目优先显示。 |
| 订单履行优先级 | 根据紧急程度（订单日期或交付截止日期）优先处理订单。 | `SELECT order_id, customer_name, order_date FROM orders ORDER BY delivery_date ASC;` | 按 `delivery_date` 排序列出订单，以优先处理最早的截止日期。 |
| 考试中得分最高的学生 | 突出显示分数最高的学生。 | `SELECT student_name, score FROM exam_results ORDER BY score DESC;` | 按考试成绩降序检索排名靠前的学生。 |
| 博客上最新的文章 | 根据发布日期显示文章，确保向用户展示最新的内容。 | `SELECT title, publication_date FROM articles ORDER BY publication_date DESC;` | 列出所有博客文章，从最近发布的开始。 |
| 按金额排序的金融交易 | 识别大额交易；有助于欺诈检测。 | `SELECT transaction_id, transaction_amount FROM transactions ORDER BY transaction_amount DESC;` | 显示所有交易，按金额从高到低排序。 |
| 终生价值最高的客户 | 根据产生的总收入识别有价值的客户。 | `SELECT customer_id, lifetime_value FROM customers ORDER BY lifetime_value DESC LIMIT 10;` | 检索终生价值排名前十的客户，按降序排列。 |
| 按评分排序的电影 | 评分最高的电影优先显示；可用于推荐系统或评论。 | `SELECT movie_name, rating FROM movies ORDER BY rating DESC;` | 列出所有电影，从评分最高到最低开始。 |
| 社交媒体帖子的最新更新 | 最新的帖子、评论或反应优先，以保持信息流的相关性和时效性。 | `SELECT post_id, content, post_date FROM posts ORDER BY post_date DESC;` | 显示所有社交媒体帖子，从最新的开始。 |
| 最热门的话题标签 | 实时识别当前趋势和使用最多的话题标签。 | `SELECT hashtag, usage_count FROM hashtags ORDER BY usage_count DESC;` | 检索按使用次数排序的热门话题标签。 |

表 `6-1` 说明了数据排序如何为各种现实场景提供洞察力。当记录按日期排序时，时间序列数据可以揭示随时间变化的趋势，例如季节性高峰。销售经理可以通过分析按时间顺序排序的月度销售数据来识别增长趋势和下降期，从而采取积极措施。根据交易金额对金融交易进行排序有助于识别异常情况，例如异常高的交易。通过根据项目完成时间等指标对员工绩效进行排序，可以识别出顶级表现者。这样，分析师可以专注于关键领域，获得洞察力，并根据可见的模式和趋势做出明智的决策。

## LIMIT 简介

`LIMIT` 子句指定了 SQL 查询应返回多少行。当你只需要数据的一个子集（例如顶级结果或条目样本）而无需检索整个数据集时，它尤其有用。`LIMIT` 允许你控制显示多少数据，以便你可以专注于最相关的记录。需要注意的是，`LIMIT` 子句在 PostgreSQL、MySQL 和 SQLite 中是标准的，但其他数据库可能需要替代语法。以下是 `LIMIT` 子句结构的基本说明：

```sql
SELECT column1, column2, ...
FROM table_name
ORDER BY column_name [ASC | DESC]
LIMIT number_of_rows;
```

`SELECT column1, column2, ...` 指定你想要从表中检索的列，`FROM table_name` 指示你从中查询数据的表，`ORDER BY column_name [ASC | DESC]` 指定在应用 `LIMIT` 之前用于对行进行排序的列。分析师通常将 `ORDER BY` 与 `LIMIT` 结合使用，以确保选择最相关的数据，而 `LIMIT number_of_rows` 将结果集限制为指定的行数。例如，如果你使用 `LIMIT 5`，则只会返回前五行。


## 使用 OFFSET 和 LIMIT 进行分页

对于像 Web 应用程序这样的用户界面，分页对于高效显示数据至关重要，它将大型数据集划分为较小的块。想象浏览一个包含数千种商品的目录——每页查看 10 或 20 个项目比一次性加载整个列表要容易得多。SQL 中的 `OFFSET` 和 `LIMIT` 子句就用于分页。`OFFSET` 指定了检索的起始点，而 `LIMIT` 指定了要检索多少行。这意味着在 `LIMIT` 子句生效之前，用户可以跳过指定数量的行。通常，PostgreSQL、MySQL 和 SQLite 都支持 `OFFSET`。其他数据库如 SQL Server 和 Oracle 则使用不同的方法，包括 `TOP`、`ROWNUM` 或 `OFFSET-FETCH`。

**注意**

使用 `OFFSET` 子句时，结果集会在返回任何内容之前跳过指定数量的行。除了分页，这还允许您跳转到数据集内的任何位置，从给定的索引开始。

以下是关于 `OFFSET` 和 `LIMIT` 如何工作的简要说明。

```sql
SELECT column1, column2, ...
FROM table_name
ORDER BY column_name [ASC | DESC]
LIMIT number_of_rows OFFSET start_position;
```

如前所述，`LIMIT` 子句决定了要显示的行数。`OFFSET` 指定了在应用 `LIMIT` 之前应跳过多少行。在大多数情况下，`start_position` 是基于当前页码和每页行数计算的。表 6-2 说明了如何在 PostgreSQL 中使用 `OFFSET` 和 `LIMIT` 进行分页。

### 表 6-2：用于分页的 OFFSET 和 LIMIT

| 场景 | 常见用途 | OFFSET | LIMIT | 查询语句 | 描述 |
| --- | --- | --- | --- | --- | --- |
| 电子商务产品列表 | 为用户浏览产品进行分页 | 根据页码变化 | 通常为 10-50 | `SELECT * FROM products ORDER BY product_id LIMIT 20 OFFSET (page_number - 1) * 20;` | 这里，表达式 `(page_number - 1) * 20` 计算分页中每页的起始点 (`OFFSET`)。通过从 `page_number` 减去 1，该公式确保第一页的偏移量为 0，而每后续一页起始位置向后移动 20 条记录。 |
| 社交媒体信息流 | 在用户信息流中加载帖子 | 基于滚动次数 | 10-20 | `SELECT * FROM posts WHERE user_id = user_id ORDER BY post_date DESC LIMIT 10 OFFSET (scroll_count - 1) * 10;` | 为用户信息流加载特定数量的最新帖子。该查询通过基于滚动次数设置偏移量来实现分页。 |
| 新闻网站 | 显示分页的新闻文章 | 按页计算 | 5-15 | `SELECT * FROM articles ORDER BY publish_date DESC LIMIT 10 OFFSET (page_number - 1) * 10;` | 获取按最新发布日期排序的一页文章，并根据指定的 `page_number` 调整。 |
| 管理员仪表盘日志 | 查看日志或报告 | 按请求的页码 | 可配置，20-100 | `SELECT * FROM logs ORDER BY timestamp DESC LIMIT 50 OFFSET (page_number - 1) * 50;` | 以时间戳降序加载日志条目供管理员查看，每页包含 50 条记录，按页码分页。 |
| 文件或数据导出 | 带分页限制地导出数据 | 持续更新 | 用户指定或 1,000 | `SELECT * FROM data ORDER BY record_id LIMIT 1000 OFFSET (batch_number - 1) * 1000;` | 以每批 1000 条记录导出数据；对于大型数据集很有用。根据批次调整偏移量以维持内存效率。 |

如表 6-2 所示，每个查询都旨在通过使用 `LIMIT` 和 `OFFSET` 来控制数据检索，从而高效处理大型数据集。这样做是为了确保各种应用程序都能拥有流畅的用户体验。

**注意**

像 `(page_number - 1) * page_size` 这样的表达式通常用于计算分页数据的起始点或偏移量。一般来说，`page_number - 1` 调整了页面通常从 1 开始（例如，第 1 页、第 2 页等）但数据偏移量从 0 开始的事实。通过减去 1，您使页码与数据索引对齐。此外，`page_size` 将调整后的页码乘以每页所需的记录数 (`page_size`)。这将每页准确地向前移动一页的记录量。例如，如果 `page_size` 为 `10`：

*   第 1 页的偏移量是 `(1 - 1) * 10 = 0`（从第一条记录开始）。
*   第 2 页的偏移量是 `(2 - 1) * 10 = 10`（从第 11 条记录开始）。

在数据库查询中，这种方法被广泛用于以可管理的部分导航数据。

`OFFSET` 通常会影响大型数据集的性能，因为它在内部会掠过行。对于较大的偏移量，最好考虑使用索引分页，这将在第 9 章中介绍。



## 第一个故事：高速公路建设与交通状况

当一条新高速公路在一座繁华城市建成时，一位名叫佩德罗的有才华的数据分析师被请来调查其对交通的影响。市议会议员们迫切想要确定，这条高速公路是否按计划减少了交通问题。佩德罗使用了一个包含主要路线上车辆数量、旅行时间和拥堵等级的数据集。利用这些数据，佩德罗计划识别趋势、比较变化，并确定高速公路是否带来了城市亟需的缓解效果。他的旅程始于定义查询，这些查询将揭示隐藏的模式和洞见，为议会未来的决策提供信息。以下是佩德罗寻求答案的问题：

1.  在高速公路建设前后，最拥堵的五条路线分别是哪些？
2.  高速公路修建前后，各路线的平均旅行时间有何变化？
3.  根据数据集提供的数据时间，哪些路线显示出显著的交通流改善？
4.  高速公路对先前高度拥堵的路线产生了何种影响？它们在建成后是否仍属于最拥堵路线之列？

佩德罗使用两张表格来比较高速公路建设前后的交通状况。这些表格包含的字段有助于回答前面提供的分析问题。

表 6-3 包含高速公路建设前的交通数据。它包括特定路线、车辆数量、平均旅行时间和拥堵等级的信息，使佩德罗能够评估基线状况。

**表 6-3**
`TrafficData_Before` 表

| 路线 _ID | 路线名称 | 车辆数量 | 平均旅行时间 | 拥堵等级 | 数据日期 |
| --- | --- | --- | --- | --- | --- |
| 1 | 谷路 | 2500 | 45 min | 高 | 2022-06-01 |
| 2 | 河畔大道 | 1800 | 38 min | 中 | 2022-06-01 |
| 3 | 主街 | 3000 | 50 min | 高 | 2022-06-01 |
| 4 | 第五大道 | 2200 | 40 min | 高 | 2022-06-01 |
| 5 | 橡木大道 | 1600 | 35 min | 中 | 2022-06-01 |
| 6 | 公园巷 | 1300 | 30 min | 低 | 2022-06-01 |
| 7 | 枫树道 | 1700 | 32 min | 低 | 2022-06-01 |
| 8 | 白桦街 | 1900 | 42 min | 中 | 2022-06-01 |
| 9 | 日落大道 | 2800 | 48 min | 高 | 2022-06-01 |
| 10 | 雪松路 | 2100 | 37 min | 中 | 2022-06-01 |

表 6-4 记录了高速公路建成后同一路线的交通数据。它包括更新的车辆数量、平均旅行时间和拥堵等级。佩德罗将使用此数据来确定交通流的改善程度。

**表 6-4**
`TrafficData_After` 表

| 路线 _ID | 路线名称 | 车辆数量 | 平均旅行时间 | 拥堵等级 | 数据日期 |
| --- | --- | --- | --- | --- | --- |
| 1 | 谷路 | 2000 | 30 min | 中 | 2023-06-01 |
| 2 | 河畔大道 | 1500 | 28 min | 低 | 2023-06-01 |
| 3 | 主街 | 2500 | 40 min | 中 | 2023-06-01 |
| 4 | 第五大道 | 1800 | 35 min | 中 | 2023-06-01 |
| 5 | 橡木大道 | 1400 | 25 min | 低 | 2023-06-01 |
| 6 | 公园巷 | 1200 | 27 min | 低 | 2023-06-01 |
| 7 | 枫树道 | 1600 | 29 min | 低 | 2023-06-01 |
| 8 | 白桦街 | 1700 | 31 min | 中 | 2023-06-01 |
| 9 | 日落大道 | 2300 | 37 min | 中 | 2023-06-01 |
| 10 | 雪松路 | 1900 | 32 min | 中 | 2023-06-01 |

为了深入了解高速公路建设前后的交通状况，佩德罗分析了这些数据。

### 查找最拥堵的路线

以下查询旨在找出高速公路建成前后最拥堵的五条路线。佩德罗编写了两个 SQL 查询：一个用于 `TrafficData_Before` 表，另一个用于 `TrafficData_After` 表。这些表格保存了不同时间段的数据，但佩德罗确保两个查询的结构完全相同，这使得他可以直接比较结果：

```sql
-- 高速公路建设前最拥堵的 5 条路线
SELECT Route_Name, Vehicle_Count, Avg_Travel_Time, Congestion_Level
FROM TrafficData_Before
ORDER BY Congestion_Level DESC, Vehicle_Count DESC
LIMIT 5;
-- 高速公路建设后最拥堵的 5 条路线
SELECT Route_Name, Vehicle_Count, Avg_Travel_Time, Congestion_Level
FROM TrafficData_After
ORDER BY Congestion_Level DESC, Vehicle_Count DESC
LIMIT 5;
```

两个查询都按 `Congestion_Level` 对数据排序，其中 `High` 比 `Moderate` 或 `Low` 更拥堵。然后，他按 `Vehicle_Count` 降序对数据排序，以找出交通最繁忙的路线。通过指定 `LIMIT 5`，可以将结果限制为仅前五条最拥堵的路线。

需要注意的是，在 SQL 中使用 `ORDER BY` 子句时，某些值如果能被识别，则具有特定的顺序。这里，由于 H、M、L 的字母顺序，`High` 将排在 `Moderate` 之前，`Moderate` 将排在 `Low` 之前。因此，上面的查询可以正常工作。但在需要定义特定顺序的地方，查询的写法必须不同。可以使用 `CASE` 语句来显式指定顺序。要将拥堵等级从 `High` 定义到 `Moderate` 再到 `Low`，请考虑以下查询：

```sql
SELECT Route_Name, Vehicle_Count, Avg_Travel_Time, Congestion_Level
FROM TrafficData_After
ORDER BY
    CASE
        WHEN Congestion_Level = 'High' THEN 1
        WHEN Congestion_Level = 'Moderate' THEN 2
        WHEN Congestion_Level = 'Low' THEN 3
    END,
    Vehicle_Count DESC
LIMIT 5;
```

这样，每个拥堵等级都有一个数值。`High` 具有最高优先级（1），其次是 `Moderate`（2），然后是 `Low`（3）。这种方法确保无论排序方法如何，都能实现所需的顺序。

**注意**
在大多数比较分析中，在不同数据集上使用相同的查询结构是一种常见且有效的方法，因为它允许在不同时间段或条件下直接比较结果。通过保持相似的查询结构，分析师确保数据的变化是由数据本身引起的，而不是由查询的逻辑差异引起的。这种方法的结果更加可靠和有效，因为它减少了无意中产生偏见的可能性。

表 6-5 显示了高速公路建成前最拥堵的五条路线，其中主街的车流量和拥堵程度最高。

**表 6-5**
建设前拥堵路线

| 路线名称 | 车辆数量 | 平均旅行时间 | 拥堵等级 |
| --- | --- | --- | --- |
| 主街 | 3000 | 50 min | 高 |
| 谷路 | 2500 | 45 min | 高 |
| 日落大道 | 2800 | 48 min | 高 |
| 第五大道 | 2200 | 40 min | 高 |
| 河畔大道 | 1800 | 38 min | 中 |

表 6-6 显示了高速公路建成后最拥堵的五条路线，表明拥堵和旅行时间有所减少。

**表 6-6**
建设后拥堵路线

| 路线名称 | 车辆数量 | 平均旅行时间 | 拥堵等级 |
| --- | --- | --- | --- |
| 主街 | 2500 | 40 min | 中 |
| 日落大道 | 2300 | 37 min | 中 |
| 谷路 | 2000 | 30 min | 中 |
| 第五大道 | 1800 | 35 min | 中 |
| 雪松路 | 1900 | 32 min | 中 |

### 分析平均旅行时间变化

以下查询用于查找高速公路建成前后的平均旅行时间变化。通过执行这些查询，佩德罗可以计算出所有路径在高速公路建成前后的平均旅行时间，这将有助于他确定整个城市的旅行时间是否有明显改善。

```sql
-- 建设前的平均旅行时间
SELECT AVG(CAST(SUBSTRING(Avg_Travel_Time FROM '^[0-9]+') AS INT)) AS Avg_Travel_Time_Before FROM TrafficData_Before;
-- 建设后的平均旅行时间
SELECT AVG(CAST(SUBSTRING(Avg_Travel_Time FROM '^[0-9]+') AS INT)) AS Avg_Travel_Time_After FROM TrafficData_After;
```

这些查询使用 `类型转换` 将 `Avg_Travel_Time` 值从像 `'30 min'` 这样的字符串样式转换为整数。这种转换是必要的，因为 `AVG` 函数需要数值输入来计算平均值。通过提取旅行时间的数字部分并将其转换为整数，佩德罗可以有效地分析数据，并就高速公路对旅行时间的影响得出有意义的结论。

表 6-7 和表 6-8 显示了高速公路建设前后所有路线的平均旅行时间，表明建设后的旅行时间显著减少。

**表 6-7**
所有路线的平均旅行时间（建设前）

| Avg_Travel_Time_Before |
| --- |
| 39.7 |

**表 6-8**
所有路线的平均旅行时间（建设后）

| Avg_Travel_Time_After |
| --- |
| 31.4 |

### 识别有显著改善的路线

提供此查询旨在根据数据集中提供的数据时间找出交通流的改善情况：

```sql
SELECT Route_Name, Vehicle_Count, Avg_Travel_Time, Congestion_Level
FROM TrafficData_After
WHERE Data_Date BETWEEN '2023-05-01' AND '2023-06-01'
ORDER BY Avg_Travel_Time ASC
LIMIT 5;
```

此查询筛选出特定日期范围 `'2023-05-01'` 到 `'2023-06-01'` 的数据，并按 `Avg_Travel_Time` 升序排序，以找出旅行时间较短的路线，表明交通流有所改善。`LIMIT 5` 将输出限制为改善效果最好的前五条路线。

表 6-9 显示了在上述 30 天内旅行时间改善最大的路线，其中橡木大道和公园巷等路线的旅行时间最短，拥堵等级最低。

**表 6-9**
旅行时间改善最大的路线

| 路线名称 | 车辆数量 | 平均旅行时间 | 拥堵等级 |
| --- | --- | --- | --- |
| 橡木大道 | 1400 | 25 min | 低 |
| 公园巷 | 1200 | 27 min | 低 |
| 河畔大道 | 1500 | 28 min | 低 |
| 枫树道 | 1600 | 29 min | 低 |
| 谷路 | 2000 | 30 min | 中 |

### 检查先前拥堵的路线

为了找出先前高度拥堵的受影响路线，并查看它们在建成后是否仍属于最拥堵路线之列，佩德罗使用了以下查询：

```sql
SELECT Route_Name, Congestion_Level
FROM TrafficData_Before
WHERE Congestion_Level = 'High'
UNION
SELECT Route_Name, Congestion_Level
FROM TrafficData_After
WHERE Congestion_Level = 'High';
```

此查询合并了高速公路建设前后具有 `High` 拥堵等级的路线。`UNION` 操作符显示了先前非常繁忙的路线，并指明它们是否仍然繁忙。

**注意**
`UNION` 操作符用于将两个或多个 `SELECT` 查询的结果组合成一个结果集。`UNION` 中的每个 `SELECT` 语句必须具有相同数量、相同顺序且数据类型兼容的列。使用 `UNION` 时，只返回唯一的行，因此查询之间的任何重复行都将自动删除。要保留重复项，可以使用 `UNION ALL`。PostgreSQL 中的其他集合操作符，包括 `UNION`、`UNION ALL`、`INTERSECT`、`INTERSECT ALL`、`EXCEPT` 和 `EXCEPT ALL`，将在后续章节中讨论。

表 6-10 显示了高速公路建成前拥堵等级为高的路线，并指明是否有任何路线在建成后仍然高度拥堵；没有路线在建设后仍保持 `High`，表明情况有所改善。

**表 6-10**
建设后仍高度拥堵的路线

| 路线名称 | 拥堵等级 |
| --- | --- |
| 主街 | 高 |
| 谷路 | 高 |
| 日落大道 | 高 |

佩德罗使用 SQL 分析了高速公路建设前后的交通数据。通过对大型数据集进行仔细的排序、筛选和分页，佩德罗得出了表明整体交通状况有所改善的洞见。他的分析为市政官员提供了清晰的证据，证明高速公路已经实现了缓解拥堵的目标。这验证了城市的投入，并为未来的基础设施项目设定了数据驱动的标准。




## 自定义排序：ORDER BY 的高级用例

### 大小写敏感性与字符串排序

在 SQL 中，大小写敏感性会影响结果的排序顺序，尤其是在比较大小写字符时。如果大写字母没有得到正确处理，SQL 数据库可能会将大写字母排在小写字母之后，这可能导致非预期的结果。通过使用 `COLLATE` 子句，分析师可以执行不区分大小写或区分大小写的排序。大多数数据库允许定义文本应如何进行比较和排序。

### 什么是 COLLATE？

在 SQL 中，你可以使用 `COLLATE` 子句来应用字符串比较和排序的规则，例如大小写敏感性（例如，区分 A 和 a）、口音敏感性（用于多语言数据集）以及语言或区域的排序约定。

注意

SQL 中的 `排序规则` 是一组规则，用于确定文本字符串如何被比较、排序和整理。在处理不同语言时，排序规则尤为重要，因为不同语言有独特的字母排序和字符比较规则。例如，排序规则控制：

1.  **大小写敏感性：** 决定大写和小写字母是否被视为相等（不区分大小写）或不同（区分大小写）。
2.  **口音敏感性：** 指定字符上的重音或变音符号（如 é 与 e）是否被视为唯一。
3.  **特定于区域的排序规则：** 基于语言或区域规则定义排序顺序，这些规则可能有所不同。例如，在某些语言中，像 ø 这样的特殊字符可能会以不同方式排序。

### PostgreSQL 中的排序规则

在 PostgreSQL 中，排序规则可以应用于多个级别，包括：

*   **数据库级别：** 定义数据库内所有字符串列的默认排序规则。
*   **列级别：** 为表中的特定列设置特定的排序规则，覆盖数据库的默认值。
*   **查询级别：** 允许在查询中使用 `COLLATE` 来临时更改该特定操作的排序行为。

PostgreSQL 提供了许多基于区域设置的预定义排序规则，使开发人员能够选择适合特定语言和大小写敏感性需求的规则。

### 使用 COLLATE

在 PostgreSQL 中，`COLLATE` 是一个子句，允许你为查询或操作指定排序顺序。这决定了文本字符串如何被排序和比较。排序规则影响字符顺序和大小写敏感性。`COLLATE` 用于排序和比较字符串，特别适用于特定于区域的排序。例如，在德语中，ä 和 a 的排序方式可能与英语不同。`COLLATE` 可以与 `CREATE TABLE`、`SELECT` 或任何文本操作一起使用。PostgreSQL 自带多个内置排序规则。你可以使用以下选项列出所有可用的排序规则。

#### 与 CREATE TABLE 一起使用 COLLATE

```
CREATE TABLE employees (
name TEXT COLLATE "en_US",
department TEXT COLLATE "de_DE"
);
```

在这里，`name` 列将使用美式英语规则排序，`department` 列将使用德语规则排序。

#### 在 SELECT 查询中使用 COLLATE

```
SELECT * FROM employees ORDER BY name COLLATE "fr_FR";
```

此查询使用法语排序规则按 `name` 对结果排序，忽略默认的排序规则。

#### 示例：使用不同排序规则的大小写敏感性

第一步，你将创建一个 `users` 表，其中包含用户名的列。你将使用不同的大小写类型来比较和排序用户名，以观察它们如何工作。以下数据存储在 `users` 表的 `username` 列中：

```
CREATE TABLE users (
username TEXT
);
INSERT INTO users (username) VALUES
('Alice'),
('alice'),
('Bob'),
('bob');
```

这个表现在有混合大小写的条目：Alice, alice, Bob, bob, Charlie, 和 charlie。使用 `COLLATE`，你可以在使用区分大小写和不区分大小写的排序规则查询表时比较结果。

#### 区分大小写的排序

为了了解区分大小写的排序如何工作，你将使用 `C` 排序规则，它通常是区分大小写的，并将大写字母排在小写字母之前。

```
SELECT * FROM users
ORDER BY username COLLATE "C";
```

表 6-11 显示了此查询的结果。

表 6-11

区分大小写的排序输出

| 用户名 |
| --- |
| Alice |
| Bob |
| Charlie |
| alice |
| bob |
| charlie |

在这种情况下，大写名称（Alice, Bob, Charlie）列在小写名称（alice, bob, charlie）之前。这种排序是因为 `C` 排序规则将大写字母视为不同于小写字母，并将它们排在前面。

#### 不区分大小写的排序

不区分大小写的排序规则 `en_US.utf8` 在排序时将大写和小写字符视为相等。

```
SELECT * FROM users
ORDER BY username
COLLATE "en_US.utf8";
```

在这种情况下，如表 6-12 所示，Alice 和 alice 被分在一组，Bob 和 bob 一组，Charlie 和 charlie 一组。这表明 `en_US.utf8` 排序规则在排序时忽略大小写差异，从而产生更“人性化”的排序。

表 6-12

不区分大小写的排序输出

| 用户名 |
| --- |
| Alice |
| alice |
| Bob |
| bob |
| Charlie |
| charlie |

### 对 NULL 值进行排序

在 PostgreSQL 中，排序时处理 `NULL` 值至关重要，尤其是在组织可能包含缺失条目的数据时。默认情况下，在升序排序中，`NULL` 值被认为低于任何其他值；在降序排序中，则高于任何其他值。然而，PostgreSQL 提供了显式选项来将 `NULL` 值排序在最前或最后，使你能够控制它们在有序结果中的位置。

#### 对 NULL 值排序的策略：NULLS FIRST 和 NULLS LAST

对 `NULL` 值进行排序有两种策略。第一种是 `NULLS FIRST`，第二种是 `NULLS LAST`。

`NULLS FIRST`：将所有 `NULL` 值放在结果集的开头。这在你希望缺失值突出显示时非常有用，例如，优先处理需要关注的记录。

```
ORDER BY column_name NULLS FIRST;
```

`NULLS LAST`：将所有 `NULL` 值放在结果集的末尾。这在 `NULL` 值不太相关，并且你希望实际数据值首先出现时非常有用。

```
ORDER BY column_name NULLS LAST;
```

#### 示例：购买历史中包含 NULL 值的客户数据

以表 `customer` 为例，该表包含客户信息，如表 6-13 所示，其中包含一个 `last_purchase_date` 列。在某些情况下，`last_purchase_date` 可能为 `NULL`，表示该客户尚未购买任何商品。

表 6-13

客户表

| 客户 ID | 客户姓名 | 最后购买日期 |
| --- | --- | --- |
| 1 | Alice | 2023-06-15 |
| 2 | Bob |   |
| 3 | Charlie | 2023-07-20 |
| 4 | Diana |   |
| 5 | Eve | 2023-05-10 |

##### 使用 NULLS FIRST 排序

要列出最近购买的客户排在前面，而尚未进行购买的客户出现在顶部，请使用以下查询：

```
SELECT * FROM customers
ORDER BY last_purchase_date
DESC NULLS FIRST;
```

表 6-14 显示在排序过程中 `NULL` 值被排在最前面。

表 6-14

使用 NULLS FIRST 的客户排序

| 客户 ID | 客户姓名 | 最后购买日期 |
| --- | --- | --- |
| 2 | Bob | NULL |
| 4 | Diana | NULL |
| 3 | Charlie | 2023-07-20 |
| 1 | Alice | 2023-06-15 |
| 5 | Eve | 2023-05-10 |

在这种情况下，由于使用了 `NULLS FIRST`，`NULL` 值出现在顶部，随后是按降序排列的最近购买日期。


##### 示例 2：使用 `NULLS LAST` 排序

你可以使用 `NULLS LAST` 来显示尚未购买任何商品的顾客：

```sql
SELECT * FROM customers
ORDER BY last_purchase_date
DESC NULLS LAST;
```

表 6-15 显示了在排序过程中，`NULL` 值被排在了最后。

表 6-15

使用 `NULLS LAST` 对顾客进行排序

| customer_id | customer_name | last_purchase_date |
| --- | --- | --- |
| 3 | Charlie | 2023-07-20 |
| 1 | Alice | 2023-06-15 |
| 5 | Eve | 2023-05-10 |
| 2 | Bob | NULL |
| 4 | Diana | NULL |

首先，非 `NULL` 日期按降序排列，由于使用了 `NULLS LAST`，`NULL` 值位于底部。

`NULLS FIRST` 的目的是将 `NULL` 值优先放在顶部，而 `NULLS LAST` 则将 `NULL` 值移到有序结果的底部。你可以控制 `NULL` 值在结果中的显示方式，这使得数据呈现能根据上下文更具意义。

## 常见陷阱与最佳实践

### 避免歧义排序：始终明确列名

在排序数据时，列名的歧义可能导致意外结果甚至查询错误。这在涉及多表（如连接）或复杂子查询的查询中很常见，因为这些列可能具有相似的名称。

在 PostgreSQL 中，当你查询具有重叠列名的多个表时，例如 `customers` 和 `orders` 表中都有 `customer_id`，如果列名在使用时没有别名或表名前缀，数据库引擎就无法立即确定你指的是哪个表的列。这在排序（`ORDER BY`）和过滤（`WHERE`）子句中尤其成问题。歧义发生的常见原因有三个。

1.  `列名重叠`：当连接多个表时，它们可能有同名的列，例如 `customer_id`。如果不指定表名或别名，PostgreSQL 就无法确定你指的是哪一列。
2.  `ORDER BY` 子句要求：PostgreSQL 期望排序引用明确无误，因为它依赖于表列值。如果遇到一个在多个表中存在但没有显式表引用的列名，它就不知道该按哪一列排序，从而导致歧义。
3.  `聚合和排序缺乏上下文`：在查询中，如果列没有明确关联到某个表，PostgreSQL 就缺乏上下文来识别应从哪个数据源提取数据，从而导致错误或意外行为。

### 避免歧义

当你查询或排序的列属于多个表时，请使用表名或别名来标识每个列所属的表。这样，PostgreSQL 就会使用正确的数据并消除歧义。

#### 排序数据时列名歧义的一个示例

`Customers` 表存储每个客户的信息，包括唯一的 `customer_id` 和 `name`。`Orders` 表记录购买交易，包括 `order_id`、用于将订单链接到特定客户的 `customer_id`，以及每笔交易的 `order_date`。表 6-16 和 6-17 能够跟踪客户随时间推移的购买情况，将每个订单与特定客户关联起来。

表 6-17

订单表

| order_id | customer_id | order_date |
| --- | --- | --- |
| 1 | 1 | 2023-06-15 |
| 2 | 1 | 2023-07-20 |
| 3 | 2 | 2023-05-10 |
| 4 | 3 | 2023-08-01 |
| 5 | 3 | 2023-07-22 |
| 6 | 4 | 2023-04-18 |
| 7 | 5 | 2023-05-25 |

表 6-16

客户表

| customer_id | name |
| --- | --- |
| 1 | Alice |
| 2 | Bob |
| 3 | Charlie |
| 4 | Diana |
| 5 | Eve |

此查询的目的是检索 `customer_id` 和 `order_date`，并按 `order_date` 排序。如果在查询中没有显式使用表名或别名，PostgreSQL 可能会因列名不明确而感到困惑，特别是当同一个列名在两个表中都存在时：

```sql
SELECT customer_id, order_date
FROM customers
JOIN orders ON customers.customer_id = orders.customer_id
ORDER BY order_date DESC;
```

`order_date` 列只存在于 `Orders` 表中，但 `customer_id` 列在 `Customers` 和 `Orders` 中都有。如果没有明确的澄清，PostgreSQL 可能无法可靠地确定 `order_date` 的预期来源，甚至可能混淆 `customer_id`。因此，执行此查询时会抛出以下错误：

```text
psql:commands.sql:36: ERROR:  column reference "customer_id" is ambiguous
LINE 1: SELECT customer_id, order_date
```

通过指定表名或别名，你可以消除任何潜在的歧义。以下是修正后的查询：

```sql
SELECT customers.customer_id, orders.order_date
FROM customers
JOIN orders ON customers.customer_id = orders.customer_id
ORDER BY orders.order_date DESC;
```

这个版本在 `ORDER BY` 子句中明确引用了 `orders.order_date` 列，确保了 PostgreSQL 知道使用哪个表进行排序。

结果显示了来自 `Orders` 表的 `order_date` 列，按降序排列，如表 6-18 所示。

表 6-18

基于 order_date 的降序排列结果

| customer_id | order_date |
| --- | --- |
| 3 | 2023-08-01 |
| 3 | 2023-07-22 |
| 1 | 2023-07-20 |
| 1 | 2023-06-15 |
| 5 | 2023-05-25 |
| 2 | 2023-05-10 |
| 4 | 2023-04-18 |

## 使用 `ORDER BY` 和 `LIMIT` 提升绘图效率

`ORDER BY` 和 `LIMIT` 在绘图时特别有用，因为它们允许你：

*   **聚焦相关数据**：在分析大型数据集时，绘图应专注于关键洞察而非所有数据点。
*   **提高绘图清晰度**：通过将数据限制为可管理的子集（例如最相关或最具影响力的行），图表变得更容易理解。
*   **增强性能**：包含数千行的大型数据集会减慢绘图过程并使可视化工具不堪重负，导致加载时间变长和响应性降低。使用 `LIMIT` 和 `ORDER BY` 可以避免因仅获取部分行而导致绘图过载。这使得可视化过程更快、更高效。
*   **突出趋势和异常值**：在绘制特定数据点时，有序和有限的数据能更容易地识别趋势、异常值并进行比较。

在本章中，`plot` 通常指以结构化、有意义的方式排列或呈现数据，通常是为可视化做准备。使用 `ORDER BY` 和 `LIMIT` 子句通过排序和筛选数据到最相关的条目来实现这一点。尽管数据存储和查询功能强大，但在你准备好并排序数据后，可以使用第三方可视化工具来创建可视化。`ORDER BY` 根据指定的列组织结果集，可以是升序（`ASC`）或降序（`DESC`），使你可以优先处理数据。同时，`LIMIT` 限制返回的行数，使得更容易专注于一个子集，例如前几条结果。这些查询共同提炼数据，以便进行更清晰、更有见地的分析和呈现。

## 总结

本章解释了在 SQL 查询中使用 `ORDER BY` 和 `LIMIT` 子句对于有效组织和管理数据集的重要性。它强调了这些子句如何通过按特定顺序排序数据并限制返回的行数来缩小查询结果范围。通过有选择地限制和选取数据，分析师可以增强数据分析和可视化任务的清晰度和性能。本章通过过滤数据以获取针对性见解、提高可视化绘图效果以及优化查询性能，展示了这些 SQL 功能。

### 核心要点

*   `ORDER BY` 和 `LIMIT` 有助于在大型数据集中聚焦最相关的条目。这使得分析师可以优先处理那些直接影响洞察的数据，例如特定列中的最高或最低值。

*   通过限制返回的行数，`ORDER BY` 和 `LIMIT` 使图表更具可读性，仅呈现最相关的数据，并降低了图形或表格中的视觉复杂性。

*   `ORDER BY` 和 `LIMIT` 通过仅返回最关键的数据点，帮助利益相关者做出明智决策，无需筛选不必要的细节。

*   使用 `OFFSET` 和 `LIMIT` 有助于在查看大型数据集时保持有序结构，因为每页显示一致数量的条目，提高了数据的可访问性和清晰度。通过减少数据库的计算负载，`LIMIT` 还能加快查询速度并降低资源消耗。

### 关键收获

*   `ORDER BY`：使用 `ORDER BY` 按特定列对数据进行排序，优先显示关键洞察，如最高或最低值。这使得在大型数据集中识别趋势和模式变得更容易。

*   `LIMIT`：通过应用 `LIMIT`，分析师可以限制返回的行数，从而只返回最相关的数据点。这在可视化中特别有用，当只需要顶部或底部的记录时，可以减少杂乱并增强清晰度。

*   `OFFSET`：将 `OFFSET` 与 `LIMIT` 结合用于分页，允许你以可管理的块状检索数据，并逐页导航大型数据集，而不是一次性加载全部。通过结合使用 `OFFSET` 和 `LIMIT`，分页成为可能。这种技术支持更用户友好的数据呈现，并通过增量加载数据而非同时处理所有数据来确保高效性能。

### 未来展望

下一章“与子查询的动态对话”将探索编写子查询的艺术，为数据分析增添深度和维度。通过使用嵌套查询，你可以获得有意义的洞察并应对复杂的分析挑战。

## 测试你的技能

Jack 是银石城图书馆的数据分析师，该图书馆拥有庞大的藏书量和读者群。图书馆管理层希望通过了解借阅模式、热门书目和会员活动来改善用户体验。请帮助他使用 `ORDER BY`、`LIMIT` 和 `OFFSET` 等 SQL 技术基于图书馆数据做出决策。图书馆数据库包含一个名为 `BookLoans` 的表，其列如表 6-19 所示。

表 6-19

`BookLoans` 表

| 列名 | 描述 |
| --- | --- |
| `loan_id` | 每次图书借阅的唯一标识符 |
| `member_id` | 每个会员的唯一标识符 |
| `book_title` | 所借图书的标题 |
| `borrow_date` | 图书借出的日期 |
| `return_date` | 图书归还的日期 |
| `loan_duration` | 借阅时长（天），计算为 `return_date` - `borrow_date` |
| `borrow_count` | 该书被借阅的次数 |

1.  识别图书馆馆藏中借阅次数最多的前五本书。按 `borrow_count` 降序排序结果，以便借阅次数最多的书出现在顶部。

2.  为帮助员工监控近期活动，检索过去一个月内借出的图书列表，按 `borrow_date` 从最近到最早排序。每页仅显示五条结果，并使用 `OFFSET` 在页面间导航。

3.  确定会员中前五个最长的借阅时长。按 `loan_duration` 降序排序结果，以便最长的时长显示在前面。

4.  找到借阅次数最少的书，并在第二页（第 6-10 行）显示。按 `borrow_count` 升序排序，并使用 `OFFSET` 跳过前五条记录。

5.  识别过去一年中借阅次数最多的十本书。按 `borrow_count` 降序排序，并通过 `borrow_date` 筛选过去一年内的结果。

