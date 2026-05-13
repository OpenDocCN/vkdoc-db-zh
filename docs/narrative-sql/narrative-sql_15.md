# 8. 数据绘图中的条件逻辑

在本章中，你将学习 SQL 的条件逻辑如何改变数据分析和可视化工作流程。你将学习 SQL 中的条件逻辑，对数据进行分类，应用动态过滤以提高图表相关性从而增强可视化效果，为可视化创建颜色编码数据，使用条件表达式聚合数据，以及处理可视化中的缺失数据。

本章的重点不在于如何可视化或绘制数据，而在于使用 SQL 在可视化之前准备数据的关键过程。本章的主要贡献是解释如何使用 SQL 在数据到达可视化阶段之前对其进行操作、清理和结构化。SQL 本身不生成图表或图形表示；相反，它是一个强大的工具，用于将原始数据转换为适合分析的结构化格式。虽然其他编程语言和工具负责数据可视化，但 SQL 确保底层数据得到适当处理，使其准备好进行有效且有意义的表示。然而，一些数据库管理系统，例如用于 PostgreSQL 的 `pgAdmin`，提供了有限的可视化工具来图形化显示查询结果。但这些并不是标准 SQL 本身的一部分——它们是数据库用户界面的功能。为了绘制数据，SQL 可以与外部工具或编程语言结合使用，例如 Python（Matplotlib, Seaborn, Plotly, Pandas）、R（ggplot2, tidyverse）或 BI 工具（Tableau, Power BI, Looker, Metabase）。

## 引言

SQL 条件逻辑可以通过 `CASE`、`IF` 等表达式以及 `<`、`>`、`=` 等条件运算符来实现。使用这些结构，开发者可以根据特定标准动态转换或过滤数据。这使得 SQL 查询更加灵活，并能适应复杂分析任务的需求，允许它们操作数据集以满足特定要求。例如，条件逻辑可以将数据分组到不同的组中，根据条件创建动态列，并在查询本身中执行上下文相关的计算。

通过使用 SQL 条件表达式进行动态绘图，你可以增强数据分析能力，因为它们允许对数据集进行动态和上下文感知的绘图。作为数据准备的一部分，可以使用 `CASE` 语句为绘图中的分类轴创建条件标签，例如对年龄范围或收入水平进行分组。要执行上下文感知的过滤，一种选择是使用条件过滤，根据分析特定的阈值或趋势来包含或排除数据点，确保图表反映出有意义的洞察。此外，对于派生指标，可以直接在查询中计算新指标以突出显示特定趋势，例如动态定义高或低绩效阈值。

表 8-1 总结了常见的可视化图表类型以及用于准备数据的 PostgreSQL 查询。该表包括图表类型、其用例以及一个在 PostgreSQL 中准备数据的示例查询。本表中的每个示例查询都使用 `sales_data` 表作为基础，并设有一个解释列来说明其目的。`sales_data` 表包含以下列：`date` (`DATE`)，表示交易日期；`region` (`VARCHAR`)，指示销售区域；`product` (`VARCHAR`)，指定产品名称；`category` (`VARCHAR`)，定义产品类别；`sales` (`NUMERIC`)，表示销售额；以及 `profit` (`NUMERIC`)，表示利润额。

表 8-1

常见的可视化图表类型及用于准备数据的 PostgreSQL 查询


## 图表类型示例

| 图表类型 | 使用场景 | 示例 PostgreSQL 查询 | 说明 |
| --- | --- | --- | --- |
| 条形图 | 比较分类数据，例如各地区销售额 | `SELECT region, SUM(sales) AS total_sales FROM sales_data GROUP BY region;` | 按地区对销售额进行分组，显示各地区的总销售额。 |
| 折线图 | 显示随时间变化的趋势，例如股价 | `SELECT date, SUM(sales) AS daily_sales FROM sales_data GROUP BY date ORDER BY date;` | 汇总每日销售额以显示时间趋势。 |
| 散点图 | 可视化两个数值变量之间的关系，例如身高与体重 | `SELECT sales, profit FROM sales_data;` | 提取销售额和利润值以可视化它们的关系。 |
| 饼图 | 显示类别的比例或百分比，例如市场份额 | `SELECT category, SUM(sales) AS total_sales FROM sales_data GROUP BY category;` | 按类别对销售额进行分组，显示总销售额的比例。 |
| 直方图 | 显示频率分布，例如年龄分布 | `SELECT FLOOR(sales / 100) * 100 AS sales_range, COUNT(*) AS frequency FROM sales_data GROUP BY sales_range ORDER BY sales_range;` | 将销售额分段，例如 0–100、100–200，并统计每个范围内的交易次数以形成分布。 |
| 箱线图 | 显示数据分布和异常值，例如按班级的测试分数 | `SELECT region, sales FROM sales_data;` | 按地区检索销售额以分析分布并识别异常值。 |
| 热力图 | 显示两个变量的模式，例如按日期和时间的销售额 | `SELECT date, region, SUM(sales) AS total_sales FROM sales_data GROUP BY date, region;` | 按日期和地区对销售额进行分组，以在热力图中可视化模式。 |
| 堆叠条形图 | 比较跨类别的比例，例如按产品和地区的销售额 | `SELECT region, category, SUM(sales) AS total_sales FROM sales_data GROUP BY region, category;` | 按地区和类别对销售额进行分组，以显示每个地区中类别的贡献。 |
| 面积图 | 显示随时间变化的累积趋势，例如累计销售额 | `SELECT date, SUM(sales) OVER (ORDER BY date) AS cumulative_sales FROM sales_data;` | 计算随时间的累计销售额，以显示总销售额增长趋势。 |
| 气泡图 | 表示三个维度，例如利润、销售额和地区 | `SELECT region, SUM(sales) AS total_sales, SUM(profit) AS total_profit FROM sales_data GROUP BY region;` | 按地区汇总销售额和利润以表示三个维度，例如气泡大小 = 销售额。 |
| 雷达图 | 比较类别的多个变量，例如性能指标 | `SELECT category, AVG(sales) AS avg_sales, AVG(profit) AS avg_profit FROM sales_data GROUP BY category;` | 按类别汇总平均销售额和利润以进行多维比较。 |
| 瀑布图 | 可视化随时间或类别间的变化，例如利润率 | `SELECT product, SUM(sales) AS total_sales FROM sales_data GROUP BY product ORDER BY total_sales DESC;` | 按销售额对产品进行排名，以可视化畅销产品之间的增量变化。 |
| 甘特图 | 跟踪项目进度，例如任务的开始和结束日期 | `SELECT product, MIN(date) AS start_date, MAX(date) AS end_date FROM sales_data GROUP BY product;` | 计算每个产品销售的开始和结束日期，用于基于时间线的可视化。 |
| 矩形树图 | 显示层次数据，例如按产品类别和子类别的销售额 | `SELECT category, product, SUM(sales) AS total_sales FROM sales_data GROUP BY category, product;` | 按类别和产品对销售额进行分组，以表示层次数据。 |
| 甜甜圈图 | 类似于饼图但中心有挖空，例如利润份额 | `SELECT category, SUM(profit) AS total_profit FROM sales_data GROUP BY category;` | 按类别对利润进行分组以显示比例，类似于饼图但中心有挖空。 |
| 帕累托图 | 突出数据中最重要的因素，例如按产品的销售额 | `SELECT product, SUM(sales) AS total_sales FROM sales_data GROUP BY product ORDER BY total_sales DESC;` | 按销售额对产品进行排名，以识别最重要的贡献者，例如二八法则。 |
| 小提琴图 | 显示分布和密度，例如工资范围 | `SELECT category, sales FROM sales_data;` | 提取每个类别的销售额数据，以显示分布密度和变异性。 |
| 弦图 | 可视化实体之间的关系，例如贸易流动 | `SELECT region AS source, category AS target, SUM(sales) AS value FROM sales_data GROUP BY region, category;` | 基于销售额映射地区（源）和类别（目标）之间的关系。 |
| 网络图 | 显示节点之间的连接，例如社交网络 | `SELECT region AS source, product AS target FROM sales_data GROUP BY region, product;` | 突出地区和产品之间的连接，用于关系分析。 |
| 漏斗图 | 表示流程中的阶段，例如销售管线 | `SELECT category, COUNT(*) AS total_transactions FROM sales_data GROUP BY category ORDER BY total_transactions DESC;` | 跟踪每个类别的交易数量，以表示流程中的阶段，例如销售漏斗。 |

## 理解 SQL 中的条件逻辑

SQL 条件逻辑允许您根据特定条件控制查询输出。因此，它对于数据分类、处理缺失值、避免除以零等错误以及根据条件返回不同结果至关重要。`CASE`、`NULLIF` 和 `COALESCE` 是 PostgreSQL 中最常用的三个条件表达式。以 `if-else` 语句形式出现的 `CASE` 表达式是最强大的条件表达式。`NULLIF` 函数在两个值相等时返回 `NULL`，否则返回第一个值。`COALESCE` 函数从列表中返回第一个非空值。



## CASE 语句

`CASE` 语句允许以类似于 `if-else` 语句的方式处理查询。考虑一个对销售金额进行分类的例子。在表 8-2 中，列出了 `order_id` 和 `amount`。

表 8-2

订单表

| order_id | Amount |
| --- | --- |
| 101 | 1200 |
| 102 | 700 |
| 103 | 300 |

以下查询从 `Orders` 表中检索 `order_id` 和 `amount`，并使用 `CASE` 语句添加一个名为 `category` 的新列，结果如表 8-3 所示。`category` 列根据订单金额的货币价值，将其分类为 `高` (≥ $1000)、`中` (≥ $500) 和 `低` (< $500)。这允许通过创建简单的分类明细来快速可视化和分析订单价值，而无需修改原始表结构。

表 8-3

对订单表中的 order_id 和 Amount 进行分类

| order_id | Amount | Category |
| --- | --- | --- |
| 101 | 1200 | High |
| 102 | 700 | Medium |
| 103 | 300 | Low |

```sql
SELECT order_id, amount,
CASE
WHEN amount >= 1000 THEN 'High'
WHEN amount >= 500 THEN 'Medium'
ELSE 'Low'
END AS category
FROM orders;
```

此查询的结果显示在表 8-3 中。

下一个查询示例为一个更复杂的商业智能问题提供了解决方案。该查询旨在展示如何根据销售业绩对产品类别进行分类和理解。

此查询必须回答以下子问题：

*   有多少个产品类别？

*   每个产品类别的销售额是多少？

*   如何将产品类别分类为有意义的绩效层级？

*   每个类别的平均销售额和总销售额是多少？

在这里，`CASE` 语句的作用至关重要，因为它创建了销售量的有意义分类，将原始数值数据转化为可操作的见解，并允许对产品类别绩效进行快速的视觉和分析理解。

给定表 8-4 中的 `sales_data` 表的数据，可以识别出表现最佳的产品类别，了解不同产品线的销售分布，并就库存、营销和战略规划做出明智的决策。

表 8-4

sales_data 表

| product_category | sale_amount | sale_date |
| --- | --- | --- |
| Electronics | 1500 | 2024-01-15 |
| Clothing | 450 | 2024-01-16 |
| Electronics | 2300 | 2024-01-17 |
| Books | 750 | 2024-01-18 |
| Clothing | 1100 | 2024-01-19 |
| Electronics | 3200 | 2024-01-20 |
| Books | 250 | 2024-01-21 |
| Clothing | 600 | 2024-01-22 |
| Electronics | 4500 | 2024-01-23 |
| Books | 890 | 2024-01-24 |
| Clothing | 1750 | 2024-01-25 |
| Electronics | 5600 | 2024-01-26 |
| Books | 330 | 2024-01-27 |
| Clothing | 880 | 2024-01-28 |
| Electronics | 6700 | 2024-01-29 |

以下查询展示了准备销售数据的结构化方法。它使用子查询计算每个产品类别的总销售额，提供销售分布的概览。一个 `CASE` 语句根据预定义的阈值将这些总计分类为销售量层级，实现灵活的分类。聚合函数计算关键指标，如销售次数、每个类别的平均销售额和总销售额。这使得结果数据集适合用于可视化和进一步分析。通过结合动态分类并按总销售额排序，该查询以清晰且有意义的格式呈现洞察：

```sql
SELECT
product_category,
CASE
WHEN SUM(sale_amount) >= 10000 THEN 'High Volume'
WHEN SUM(sale_amount) >= 5000 THEN 'Medium Volume'
ELSE 'Low Volume'
END AS sales_volume_category,
COUNT(*) AS number_of_sales,
AVG(sale_amount) AS average_sale,
SUM(sale_amount) AS total_sales
FROM sales_data
GROUP BY product_category
ORDER BY total_sales DESC;
```

此查询的结果显示在表 8-5 中。

表 8-5

结构化数据表（查询执行后）

| product_category | sales_volume_category | number_of_sales | average_sale | total_sales |
| --- | --- | --- | --- | --- |
| Electronics | High Volume | 6 | 3966.66 | 23800.00 |
| Clothing | Low Volume | 5 | 956.00 | 4780.00 |
| Books | Low Volume | 4 | 555.00 | 2220.00 |

此查询高效地准备了可用于产生有意义洞察的结构化数据。计算字段——例如 `sales_volume_category`、`number_of_sales`、`average_sale` 和 `total_sales`——是数据可视化的关键要素。在下一阶段，这些数据可以使用各种图表类型进行展示：一个*柱状图*可以比较类别之间的绝对值；一个*饼图*显示每个类别占总销售额的比例；一个*散点图*可以扩展数据以探索关系，例如销售量与销售频率之间的关系。


### NULLIF

`NULLIF` 函数通过在两个值相等时返回 `NULL` 来防止错误。`NULLIF` 是一个强大的 SQL 函数，可防止被零除错误，并在数据分析过程中处理空值。本质上，它允许你将特定值替换为 `NULL`，这在执行百分比或比率等可能因零值导致计算问题的计算时至关重要。使用 `NULLIF`，数据分析师和数据库专业人员可以通过将潜在的有问题的零值转换为 `NULL`，来创建可靠、防错的查询，用于报告、绘图和统计分析。在数据转换中，这提供了一种处理边缘情况的干净、健壮的方法。

#### 注意

在 SQL 中，某些常见情况如果不小心处理，可能会导致错误或意外行为。这些情况包括被零除、区分空字符串与 `NULL` 值、日期范围边界、字符串比较中的大小写敏感性、数值精度损失以及在空数据集上的聚合。这些并非罕见的边缘情况；它们是编写健壮 SQL 查询的关键考虑因素，应在数据处理过程中明确处理。

例如，在目标为零或数据缺失的情况下，`NULLIF` 可用于分析以下员工绩效指标数据。表 8-6 显示了员工绩效指标，包括销售额、目标、培训小时数和完成的项目。这提供了每位员工的销售成就与其目标的结构化视图。

**表 8-6**

**performance_metrics 数据表（员工绩效指标）**

| emp_id | Name | Sales | Target | training_hours | projects_completed |
| --- | --- | --- | --- | --- | --- |
| 101 | Alice | 50000 | 45000 | 20 | 5 |
| 102 | Bob | 30000 | 0 | 15 | 3 |
| 103 | Charlie | 75000 | 60000 | 0 | 8 |
| 104 | Diana | 45000 | 40000 | 25 | 0 |
| 105 | Eve | 0 | 35000 | 30 | 4 |
| 106 | Frank | 85000 | 80000 | 10 | 7 |
| 107 | Grace | 25000 | 30000 | 0 | 2 |
| 108 | Henry | 55000 | 0 | 40 | 6 |
| 109 | Ivy | 65000 | 50000 | 35 | 0 |
| 110 | Jack | 40000 | 45000 | 0 | 5 |

以下查询通过计算关键指标（如目标达成率、生产率比率和绩效方差）来评估员工绩效。它使用 CTE 首先计算目标达成率百分比，并使用 `NULLIF(target, 0)` 处理被零除的情况。生产率比率是根据完成的项目相对于培训小时数得出的，而绩效方差衡量与销售目标的偏差。主查询根据预定义阈值将员工分类为绩效状态和效率等级。`NULL` 值被有效处理，确保了稳健的数据分析。在此查询中，结果按目标达成率排序，将高绩效员工排在前面。

```sql
WITH performance_metrics AS (
    SELECT
        emp_id,
        name,
        ROUND(
            sales::numeric / NULLIF(target, 0) * 100,
            2
        ) AS target_achievement,
        ROUND(
            projects_completed::numeric / NULLIF(training_hours, 0) * 100,
            2
        ) AS productivity_ratio,
        CASE
            WHEN sales = 0 THEN NULL
            WHEN target = 0 THEN NULL
            ELSE ROUND((sales - target)::numeric / target * 100, 2)
        END AS performance_variance
    FROM employee_metrics
)
SELECT
    pm.*,
    CASE
        WHEN target_achievement IS NULL THEN 'Invalid Target'
        WHEN target_achievement >= 100 THEN 'Exceeded'
        WHEN target_achievement >= 80 THEN 'On Track'
        ELSE 'Below Target'
    END AS performance_status,
    CASE
        WHEN productivity_ratio IS NULL THEN 'No Training Data'
        WHEN productivity_ratio > 20 THEN 'High Efficiency'
        WHEN productivity_ratio > 10 THEN 'Moderate Efficiency'
        ELSE 'Needs Improvement'
    END AS efficiency_rating
FROM performance_metrics pm
ORDER BY target_achievement DESC NULLS LAST;
```

此查询使用 `NULLIF` 函数来处理潜在的除法错误并确保计算准确，这很好地示例了 `NULLIF` 扮演的重要角色。通过应用 `NULLIF(target, 0)`，查询在计算目标达成率时防止了被零除。`NULLIF(training_hours, 0)` 确保即使没有培训小时数，生产率计算也保持有效。此外，`NULLIF` 与 `CASE` 语句结合使用，以创建有意义的分类。这允许查询在单一分析中管理各种零和 `NULL` 场景。

在此查询中，使用了 `::numeric` 转换来确保小数除法和精确舍入，特别是在原始数据类型可能是整数的情况下。例如，不进行转换，两个整数相除可能导致整数除法，例如 1/2 = 0 而不是 0.5。如果涉及的列已经是 `NUMERIC` 或 `DECIMAL` 类型，则可以安全地移除这些转换。但是，如果模式使用 `INTEGER` 或 `TEXT`，显式转换可保证在除法和舍入操作期间行为一致。

在此查询中，使用 `WITH` 子句创建了一个名为 `performance_metrics` 的 CTE。此 CTE 根据 `employee_metrics` 表计算员工的多个绩效相关指标。定义 CTE 后，主查询从中选择并根据计算出的指标添加列（`performance_status` 和 `efficiency_rating`）。

#### 注意

如前面章节所述，`WITH` 子句，也称为 CTE（公共表表达式），允许你定义一个临时结果集，该结果集可以在 `SELECT`、`INSERT`、`UPDATE` 或 `DELETE` 语句中引用。它通常用于通过将复杂查询分解为更小、更易管理的部分来简化复杂查询。`WITH` 子句的基本语法如下：

```sql
WITH cte_name AS (
    -- 定义 CTE 的子查询
    SELECT ...
    FROM ...
    WHERE ...
)
-- 引用 CTE 的主查询
SELECT ...
FROM cte_name
WHERE ...
```

原始指标到可操作见解的转换是通过安全计算目标达成率百分比、无错误地计算生产率比率、确定绩效方差以及对绩效和效率水平进行分类来实现的。此外，该查询考虑了目标或培训小时数为零的边缘情况，确保分析保持稳健。此示例突出了 `NULLIF` 在数据分析中起着至关重要的作用，尤其是在处理包含缺失值或零条目的真实世界数据集时。参见表 8-7。

**表 8-7**

**安全计算目标达成百分比的结果**

| emp_id | name | target_achievement | productivity_ratio | performance_variance | performance_status | efficiency_rating |
| --- | --- | --- | --- | --- | --- | --- |
| 106 | Frank | 106.25 | 70 | 6.25 | Exceeded | High Efficiency |
| 103 | Charlie | 125 | `NULL` | 25 | Exceeded | No Training Data |
| 109 | Ivy | 130 | 0 | 30 | Exceeded | Needs Improvement |
| 101 | Alice | 111.11 | 25 | 11.11 | Exceeded | High Efficiency |
| 104 | Diana | 112.5 | 0 | 12.5 | Exceeded | Needs Improvement |
| 110 | Jack | 88.89 | `NULL` | -11.11 | On Track | No Training Data |
| 107 | Grace | 83.33 | `NULL` | -16.67 | On Track | No Training Data |
| 105 | Eve | 0 | 13.33 | -100 | Below Target | Moderate Efficiency |
| 102 | Bob | `NULL` | 20 | `NULL` | Invalid Target | Moderate Efficiency |
| 108 | Henry | `NULL` | 15 | `NULL` | Invalid Target | Moderate Efficiency |


现在，由于包含了`NULL`值处理，该数据集对于深入的绩效评估和决策制定具有价值。表 8-7 展示了如何通过添加计算指标（如目标达成率、生产率比率、绩效方差、绩效状态和效率评级）来扩展绩效评估。目标达成率以百分比衡量，确保超额完成目标的员工得到认可。生产率比率根据培训时长评估效率，而绩效方差则捕捉与预期绩效的偏差。员工被归类到不同的绩效状态，如“超额完成”或“按计划进行”，效率评级则表明其工作成效。

### COALESCE

PostgreSQL 中的`COALESCE`是一个函数，它返回表达式列表中第一个非`NULL`值。因此，它是处理`NULL`值和提供默认值的关键组件。对于一个可能在列中包含缺失数据（`NULL`值）的表，`COALESCE`允许你用有意义的默认值替换这些`NULL`值。PostgreSQL 中`COALESCE`的基本语法如下：

```
COALESCE(value1, value2, ..., value_n)
```

`COALESCE`从提供的参数列表中返回第一个非`NULL`值。如果所有值都是`NULL`，则返回`NULL`。它通常用于通过提供默认或备用值来处理查询中的`NULL`值。

表 8-8 展示了一个会员数据表，该表需要通过假设一个默认有效期来填充缺失的到期日期。

表 8-8
会员数据表

| member_id | member_name | membership_type | expiration_date |
| --- | --- | --- | --- |
| 1 | John Doe | Gold | 2025-06-15 |
| 2 | Jane Smith | Silver | NULL |
| 3 | Mike Johnson | Gold | 2025-08-10 |
| 4 | Sarah Williams | Bronze | NULL |
| 5 | David Brown | Gold | 2025-07-01 |
| 6 | Emily Davis | Silver | NULL |
| 7 | James Wilson | Bronze | 2025-05-20 |
| 8 | Laura Martinez | Gold | NULL |
| 9 | Robert Taylor | Silver | 2025-09-25 |
| 10 | Sophia Anderson | Bronze | NULL |

以下查询旨在使用`COALESCE`来填充缺失的到期日期，假设缺失值的默认有效期为：Gold: 2025-12-31，Silver: 2025-11-30，Bronze: 2025-10-31。

```
SELECT
    member_id,
    member_name,
    membership_type,
    COALESCE(expiration_date,
             CASE
                 WHEN membership_type = 'Gold' THEN DATE '2025-12-31'
                 WHEN membership_type = 'Silver' THEN DATE '2025-11-30'
                 WHEN membership_type = 'Bronze' THEN DATE '2025-10-31'
             END
    ) AS adjusted_expiration_date
FROM memberships;
```

在此案例中，这些值是假设的，但在真实场景中，处理缺失值的最佳策略大多遵循此流程：
1.  分析缺失数据的模式。
2.  考虑业务背景，记录假设和方法。
3.  利用领域专业知识，通过已知模式和统计方法验证插补值，以确保插补值符合业务逻辑并保持数据完整性。

通过结合使用`COALESCE`和`CASE`语句，你可以确保缺失到期日期的会员根据其会员类型获得一个默认日期。这有助于保持数据一致性，并避免报告中出现`NULL`值。此查询确保缺失到期日期的会员将获得一个默认日期。这是基于他们的会员类型。如表 8-9 所示，这有助于保持数据一致性并避免报告中出现`NULL`值。

表 8-9
缺失的到期日期已被填充

| member_id | member_name | membership_type | adjusted_expiration_date |
| --- | --- | --- | --- |
| 1 | John Doe | Gold | 2025-06-15 |
| 2 | Jane Smith | Silver | 2025-11-30 |
| 3 | Mike Johnson | Gold | 2025-08-10 |
| 4 | Sarah Williams | Bronze | 2025-10-31 |
| 5 | David Brown | Gold | 2025-07-01 |
| 6 | Emily Davis | Silver | 2025-11-30 |
| 7 | James Wilson | Bronze | 2025-05-20 |
| 8 | Laura Martinez | Gold | 2025-12-31 |
| 9 | Robert Taylor | Silver | 2025-09-25 |
| 10 | Sophia Anderson | Bronze | 2025-10-31 |

## 第一个故事：医院的分析故事

索菲亚是一家大型都市医院的分析师，她负责为一个关键会议准备可视化图表。原始的医院数据混乱不堪，包含缺失值、异常值以及变量之间的复杂关系。如表 8-10 所示，一个`hospital_admissions`表有 20 行和多个`NULL`值。尽管如此，索菲亚继续运用 SQL 和她的分析思维，绘制出清晰、富有洞察力的图表。

表 8-10
hospital_admissions 表

| patient_id | Age | diagnosis_code | admission_date | discharge_date |
| --- | --- | --- | --- | --- |
| 1 | 12 | 150 | 2024-03-01 | 2024-03-03 |
| 2 | 45 | 120 | 2024-03-02 | 2024-03-05 |
| 3 | 67 | 180 | 2024-03-03 | 2024-03-10 |
| 4 | 8 | 90 | 2024-03-04 | 2024-03-06 |
| 5 | 32 | 110 | 2024-03-05 | 2024-03-08 |
| 6 | 55 | 130 | 2024-03-06 | 2024-03-12 |
| 7 | 70 | 140 | 2024-03-07 | `NULL` |
| 8 | 25 | 160 | 2024-03-08 | 2024-03-11 |
| 9 | 40 | 170 | 2024-03-09 | 2024-03-16 |
| 10 | 60 | 190 | 2024-03-10 | 2024-03-20 |
| 11 | 15 | `NULL` | 2024-03-11 | 2024-03-13 |
| 12 | 50 | 200 | 2024-03-12 | `NULL` |
| 13 | 28 | 105 | 2024-03-13 | 2024-03-15 |
| 14 | 72 | `NULL` | 2024-03-14 | 2024-03-21 |
| 15 | 10 | 115 | 2024-03-15 | `NULL` |
| 16 | 35 | 125 | 2024-03-16 | 2024-03-18 |
| 17 | 65 | 135 | 2024-03-17 | 2024-03-24 |
| 18 | 22 | 145 | 2024-03-18 | 2024-03-20 |
| 19 | 48 | 155 | 2024-03-19 | `NULL` |
| 20 | 58 | 165 | 2024-03-20 | 2024-03-27 |

在数据准备过程中，她遇到了六个主要挑战：
*   如何将医院患者按年龄组（如儿童、成人、老年）分类，以便更轻松地解读按年龄组划分的入院情况可视化图表？
*   如何动态过滤医院数据，仅显示因心脏相关问题入院的患者——例如，在 2024 年期间，诊断代码在 100 到 199 之间的患者？
*   按诊断代码范围（100–125, 126–150, 151–175, 和 176–199）分组，心脏相关问题患者的平均住院时长是多少？
*   如何预处理医院数据以创建一个`stay_category`列，使用条件逻辑根据住院天数将患者住院时长分类为短期住院、中期住院或长期住院？
*   如何计算每个年龄组（儿童、成人、老年）的患者总数和平均住院时长，以识别人口统计学趋势和资源需求？
*   如何通过处理缺失的出院日期（用`admission_date + 2`天填充）并排除住院时长超过 365 天的异常值，来清洗医院数据以用于可视化？

作为第一个挑战的一部分，目标是按年龄对医院患者进行分类，包括其年龄数据值的`hospital_admissions`数据表。索菲亚需要将患者分为年龄组（儿童、成人、老年），以便更好地可视化按年龄类别划分的入院情况。

```
SELECT
    patient_id,
    age,
    CASE
        WHEN age < 18 THEN 'Child'
        WHEN age >= 18 AND age < 65 THEN 'Adult'
        WHEN age >= 65 THEN 'Senior'
        ELSE 'Unknown'
    END AS age_group
FROM hospital_admissions
ORDER BY age_group;
```

此查询使用条件逻辑创建了一个`age_group`列。像这样对数据进行分类，通过将相似的数据点分组，提高了可视化图表的清晰度。在此查询中，`CASE`语句评估`age`列并根据其值分配类别。它将年龄小于 18 岁的个体归类为`Child`，年龄在 18 到 64 岁之间归类为`Adult`，年龄为 65 岁或以上归类为`Senior`。对于任何意外值（如`NULL`），则分配`Unknown`类别。



## SQL 数据查询与动态分类分析

在本次查询中，`CASE` 语句根据患者的年龄将其分类到不同的年龄组。`BETWEEN` 操作符用于定义 `Adult` 类别。需要注意的是，`BETWEEN` 是*包含性*的，这意味着边界值 18 和 64 也被包含在内。因此，年龄恰好为 18 岁或 64 岁的患者将归入 `Adult` 类别。这确保了所有年龄范围的分类清晰且不重叠。

如表 8-11 所示，结果通过 `ORDER BY` 子句按 `age_group` 列排序，以提高可读性。

**表 8-11：按 age_group 的数据分类**

| patient_id | Age | age_group |
| --- | --- | --- |
| 1 | 12 | Child |
| 4 | 8 | Child |
| 11 | 15 | Child |
| 15 | 10 | Child |
| 2 | 45 | Adult |
| 5 | 32 | Adult |
| 6 | 55 | Adult |
| 8 | 25 | Adult |
| 9 | 40 | Adult |
| 10 | 60 | Adult |
| 13 | 28 | Adult |
| 16 | 35 | Adult |
| 18 | 22 | Adult |
| 19 | 48 | Adult |
| 20 | 58 | Adult |
| 3 | 67 | Senior |
| 7 | 70 | Senior |
| 14 | 72 | Senior |
| 17 | 65 | Senior |

### 动态筛选心脏相关病例

在下一个挑战中，决策者希望根据诊断代码动态查看入院患者。Sofia 需要筛选数据，仅显示 `diagnosis_codes` 在 100 到 199 之间的心脏相关患者。

```sql
SELECT
patient_id,
diagnosis_code,
admission_date,
discharge_date
FROM hospital_admissions
WHERE diagnosis_code BETWEEN 100 AND 199
AND admission_date >= '2024-01-01'
AND admission_date <= '2024-12-31';
```

在此查询中，`WHERE` 子句允许进行动态筛选。此处，数据被缩小到仅显示 2024 年内的心脏相关入院记录，减少了杂乱信息，并将重点放在请求的子集数据上。`WHERE` 子句通过仅选择 `diagnosis_code` 在 100 到 199 之间（代表心脏相关问题）且 `admission_date` 在 2024 年内的行来进行过滤。如表 8-12 所示，动态筛选确保只有相关数据被纳入分析。

**表 8-12：动态筛选（心脏相关问题）**

| patient_id | diagnosis_code | admission_date | discharge_date |
| --- | --- | --- | --- |
| 1 | 150 | 2024-03-01 | 2024-03-03 |
| 2 | 120 | 2024-03-02 | 2024-03-05 |
| 3 | 180 | 2024-03-03 | 2024-03-10 |
| 5 | 110 | 2024-03-05 | 2024-03-08 |
| 6 | 130 | 2024-03-06 | 2024-03-12 |
| 7 | 140 | 2024-03-07 | NULL |
| 8 | 160 | 2024-03-08 | 2024-03-11 |
| 9 | 170 | 2024-03-09 | 2024-03-16 |
| 10 | 190 | 2024-03-10 | 2024-03-20 |
| 13 | 105 | 2024-03-13 | 2024-03-15 |
| 15 | 115 | 2024-03-15 | NULL |
| 16 | 125 | 2024-03-16 | 2024-03-18 |
| 17 | 135 | 2024-03-17 | 2024-03-24 |
| 18 | 145 | 2024-03-18 | 2024-03-20 |
| 19 | 155 | 2024-03-19 | NULL |
| 20 | 165 | 2024-03-20 | 2024-03-27 |

在医疗保健和医院记录中，诊断代码是用于表示特定医疗状况、诊断或患者入院原因的标准化代码。这些代码通常是《国际疾病分类》等编码系统的一部分。该系统广泛应用于医疗保健的统计、计费和研究目的。例如，100 到 199 范围内的诊断代码可能对应于心脏病、心律失常或高血压等心脏相关疾病。

### 按诊断代码范围分析平均住院时长

以下查询计算了心脏相关问题的平均住院时长，并按 `diagnosis_code` 范围（100–125、126–150、151–175 和 176–199）进行分组。

```sql
SELECT
CASE
WHEN diagnosis_code BETWEEN 100 AND 125 THEN '100-125'
WHEN diagnosis_code BETWEEN 126 AND 150 THEN '126-150'
WHEN diagnosis_code BETWEEN 151 AND 175 THEN '151-175'
WHEN diagnosis_code BETWEEN 176 AND 199 THEN '176-199'
ELSE 'Unknown'
END AS diagnosis_code_range,
COUNT(patient_id) AS patient_count,
AVG(COALESCE(discharge_date, admission_date + INTERVAL '2 days') - admission_date) AS avg_length_of_stay
FROM hospital_admissions
WHERE diagnosis_code BETWEEN 100 AND 199
AND admission_date >= '2024-01-01'
AND admission_date <= '2024-12-31'
GROUP BY diagnosis_code_range
ORDER BY diagnosis_code_range;
```

该查询使用 `CASE` 语句将 `diagnosis_code` 分组到定义的范围中。`COALESCE` 函数将 `NULL discharge_date` 值替换为 `admission_date` 加上两天，以便为所有患者计算住院时长。`AVG` 函数计算每个 `diagnosis_code` 范围的平均住院时长。`COUNT` 函数确定每个范围内的患者数量，从而更好地了解患者分布情况。`WHERE` 子句将数据筛选为仅包含 2024 年的心脏相关入院记录。该查询提供了有关各 `diagnosis_code` 范围平均住院时长的宝贵见解，帮助决策者优化资源并理解趋势。

### 住院时长分类与可视化

接下来，为了创建患者年龄与住院时长的散点图，Sofia 希望根据住院时长是短（< 3 天）、中（3-7 天）还是长（> 7 天）来分配颜色。

```sql
SELECT
patient_id,
age,
(discharge_date - admission_date) AS length_of_stay,
CASE
WHEN (discharge_date - admission_date) < 3 THEN 'Short Stay'
WHEN (discharge_date - admission_date) BETWEEN 3 AND 7 THEN 'Moderate Stay'
WHEN (discharge_date - admission_date) > 7 THEN 'Long Stay'
ELSE 'Unknown'
END AS stay_category
FROM hospital_admissions;
```

`CASE` 中的条件逻辑创建了一个 `stay_category` 列，该列可用于在可视化软件中为绘图点分配颜色（例如，短住院 = 蓝色，中住院 = 绿色，长住院 = 红色）。

在此查询中，`length_of_stay` 计算确定了 `discharge_date` 和 `admission_date` 之间的差值。然后，`CASE` 语句将住院时长短于三天的分类为 `Short Stay`，介于三天到七天之间的分类为 `Moderate Stay`，超过七天的分类为 `Long Stay`。如表 8-13 所示，对于缺失或无效的值，则分配 `Unknown` 类别。

**表 8-13：住院时长计算结果**

| patient_id | Age | length_of_stay | stay_category |
| --- | --- | --- | --- |
| 1 | 12 | 2 | Short Stay |
| 2 | 45 | 3 | Moderate Stay |
| 3 | 67 | 7 | Moderate Stay |
| 4 | 8 | 2 | Short Stay |
| 5 | 32 | 3 | Moderate Stay |
| 6 | 55 | 6 | Moderate Stay |
| 7 | 70 | `NULL` | Unknown |
| 8 | 25 | 3 | Moderate Stay |
| 9 | 40 | 7 | Moderate Stay |
| 10 | 60 | 10 | Long Stay |
| 11 | 15 | 2 | Short Stay |
| 12 | 50 | `NULL` | Unknown |
| 13 | 28 | 2 | Short Stay |
| 14 | 72 | 7 | Moderate Stay |
| 15 | 10 | `NULL` | Unknown |
| 16 | 35 | 2 | Short Stay |
| 17 | 65 | 7 | Moderate Stay |
| 18 | 22 | 2 | Short Stay |
| 19 | 48 | `NULL` | Unknown |
| 20 | 58 | 7 | Moderate Stay |

### 按年龄组分析平均住院时长

接下来，医院管理者希望了解各年龄组（儿童、成人、老年）的平均住院时长，以便更有效地规划资源。

```sql
SELECT
CASE
WHEN age < 18 THEN 'Child'
WHEN age BETWEEN 18 AND 64 THEN 'Adult'
WHEN age >= 65 THEN 'Senior'
ELSE 'Unknown'
END AS age_group,
COUNT(patient_id) AS total_patients,
AVG(discharge_date - admission_date) AS avg_length_of_stay
FROM hospital_admissions
GROUP BY age_group
ORDER BY age_group;
```



在此次查询中，在聚合中使用条件逻辑能提供有意义的洞察，例如按年龄组计算的平均住院时长。这种类型的汇总对于创建显示趋势的条形图或折线图至关重要。此处，`CASE`语句将患者分组为`儿童`、`成人`和`老年`类别。随后应用聚合函数，使用`COUNT(patient_id)`来计算每组患者数量，并使用`AVG(discharge_date - admission_date)`计算每组的平均住院时长。如表 8-14 所示，结果通过`GROUP BY`子句按`age_group`进行分组。

表 8-14

按年龄组分类的平均住院时长

| age_group | total_patients | avg_length_of_stay |
| --- | --- | --- |
| 儿童 | 4 | 2 |
| 成人 | 12 | 4.5 |
| 老年 | 4 | 7 |

下一个挑战是，一些患者的出院日期缺失，而另一些患者的住院时间极长（例如，由数据录入错误引起的异常值）。Sofia 需要为此绘图清理数据。缺失值应用`admission_date`加上两天的默认值填充，且任何超过 365 天的住院时长都应被排除。

```sql
SELECT
patient_id,
age,
admission_date,
COALESCE(discharge_date, admission_date + INTERVAL '2 days') AS clean_discharge_date,
GREATEST(0, EXTRACT(DAY FROM COALESCE(discharge_date, admission_date + INTERVAL '2 days') - admission_date)) AS clean_length_of_stay
FROM hospital_admissions
WHERE EXTRACT(DAY FROM COALESCE(discharge_date, admission_date + INTERVAL '2 days') - admission_date) <= 365;
```

此查询使用`COALESCE`处理缺失值，并确保只有干净、有效的数据被用于可视化，如表 8-15 所示。通过对`length_of_stay`应用过滤器来排除异常值。`COALESCE`函数将`NULL`的出院日期替换为`admission_date`加两天。`GREATEST`函数确保住院时长至少为 0 天。此外，`WHERE`子句排除了住院时长超过 365 天的行，过滤掉了异常值。

表 8-15

为绘图清理数据并处理用默认值填充的缺失值

| patient_id | 年龄 | admission_date | clean_discharge_date | clean_length_of_stay |
| --- | --- | --- | --- | --- |
| 1 | 12 | 2024-03-01 | 2024-03-03 | 2 |
| 2 | 45 | 2024-03-02 | 2024-03-05 | 3 |
| 3 | 67 | 2024-03-03 | 2024-03-10 | 7 |
| 4 | 8 | 2024-03-04 | 2024-03-06 | 2 |
| 5 | 32 | 2024-03-05 | 2024-03-08 | 3 |
| 6 | 55 | 2024-03-06 | 2024-03-12 | 6 |
| 7 | 70 | 2024-03-07 | 2024-03-09 | 2 |
| 8 | 25 | 2024-03-08 | 2024-03-11 | 3 |
| 9 | 40 | 2024-03-09 | 2024-03-16 | 7 |
| 10 | 60 | 2024-03-10 | 2024-03-20 | 10 |
| 11 | 15 | 2024-03-11 | 2024-03-13 | 2 |
| 12 | 50 | 2024-03-12 | 2024-03-14 | 2 |
| 13 | 28 | 2024-03-13 | 2024-03-15 | 2 |
| 14 | 72 | 2024-03-14 | 2024-03-21 | 7 |
| 15 | 10 | 2024-03-15 | 2024-03-17 | 2 |
| 16 | 35 | 2024-03-16 | 2024-03-18 | 2 |
| 17 | 65 | 2024-03-17 | 2024-03-24 | 7 |
| 18 | 22 | 2024-03-18 | 2024-03-20 | 2 |
| 19 | 48 | 2024-03-19 | 2024-03-21 | 2 |
| 20 | 58 | 2024-03-20 | 2024-03-27 | 7 |

## 总结

本章重点介绍了 SQL 条件逻辑在增强数据分析和可视化工作流程方面的强大功能。您探索了对数据进行分类、应用动态过滤以及处理缺失值或异常值以获得更具相关性的可视化的技术。条件逻辑通过根据特定阈值或趋势动态包含或排除数据点，实现了上下文感知的绘图。`CASE`语句允许为绘图设置条件标签，例如分组年龄范围或收入水平。SQL 还简化了派生指标的计算，从而更容易识别高绩效或低绩效趋势。这些技术共同提高了数据可视化的准确性、清晰度和洞察力。

### 关键点

*   SQL 中的条件逻辑通过允许动态和上下文感知的绘图来增强数据分析，从而实现更有意义的可视化。

*   使用`CASE`语句，可以对数据进行分类或条件标签。例如，分组年龄范围或收入水平以获得更清晰的表示。

*   动态过滤确保绘图通过根据阈值或趋势包含或排除数据点来关注相关的数据点。

*   SQL 的灵活性可以处理缺失和异常数据，提高可视化准确性和洞察力。

*   通过使用 SQL 条件逻辑，工作流程变得更具适应性，确保可视化反映有价值且可操作的洞察。

### 关键要点

*   **动态数据分类**：使用`CASE`语句动态分组或标记数据，以获得更清晰、更有意义的可视化。

*   **上下文感知过滤**：应用条件过滤根据特定阈值或趋势包含或排除数据点，确保绘图专注于相关的洞察。

*   **用于洞察的派生指标**：使用条件表达式直接在查询中计算相关指标，从而能够识别高绩效或低绩效阈值等模式。

*   **提高的可视化准确性**：通过 SQL 条件逻辑有效处理缺失或异常数据，确保更干净、更可靠的可视化。

*   **动态绘图准备**：通过上下文感知的 SQL 技术增强绘图工作流程，实现适应性强且富有洞察力的数据可视化。

### 展望

下一章“使用索引和视图优化您的脚本”将探讨提高 SQL 查询性能和效率的技术。这包括理解索引如何通过优化数据库访问和组织数据的方式来加速数据检索，以及视图如何通过创建可重用的虚拟表来简化复杂查询并精简工作流程的作用。通过结合索引和视图，您可以在保持清晰度和可维护性的同时提升查询执行时间。



## 测试你的技能

约翰是一所国际大学的数据分析师，负责分析学生在不同课程中的整体表现。大学管理层希望得到以下问题的答案，例如：

1.  学生在不同学习项目中的成绩如何比较，并按表现水平（如：高：85 分以上，中：70-84 分，低：低于 70 分）分类？
2.  每个学习项目中，表现优异（成绩 ≥ 90）或困难（成绩 ≤ 60）的学生占比是多少？
3.  哪些年龄段的学生在不同课程上倾向于表现更好？
4.  学生在核心课程（例如计算机科学的数据结构、商业的经济学）中的表现如何随学期变化？
5.  在同一门课程中，学生在不同学期成绩有所提升的比例是多少？

约翰意识到静态报告是不够的。他需要动态的、对上下文敏感的数据可视化，这些可视化可以根据特定的分析标准进行调整。约翰决定在可视化过程之前使用 SQL 进行数据准备。使用表 8-16、8-17 和 8-18 中的数据来回答这些问题。

表 8-18
成绩表
| `grade_id` | `student_id` | `course_id` | `grade` | `semester` |
| --- | --- | --- | --- | --- |
| 1 | 14 | 1 | 77 | 2024 年春季学期 |
| 2 | 4 | 6 | 83 | 2023 年秋季学期 |
| 3 | 12 | 4 | 93 | 2024 年春季学期 |
| 4 | 12 | 6 | 55 | 2023 年秋季学期 |
| 5 | 2 | 2 | 96 | 2023 年秋季学期 |
| 6 | 4 | 6 | 88 | 2024 年春季学期 |
| 7 | 7 | 3 | 66 | 2024 年春季学期 |
| 8 | 6 | 1 | 89 | 2024 年春季学期 |
| 9 | 8 | 2 | 81 | 2023 年秋季学期 |
| 10 | 6 | 2 | 62 | 2024 年春季学期 |
| 11 | 1 | 6 | 92 | 2024 年春季学期 |
| 12 | 11 | 7 | 52 | 2024 年春季学期 |
| 13 | 15 | 4 | 74 | 2024 年春季学期 |
| 14 | 7 | 4 | 89 | 2023 年秋季学期 |
| 15 | 7 | 7 | 92 | 2024 年春季学期 |

表 8-17
课程数据表
| `course_id` | `course_name` | `study_program` |
| --- | --- | --- |
| 1 | 数据结构 | 计算机科学 |
| 2 | 市场营销基础 | 工商管理 |
| 3 | 物理学 | 工程学 |
| 4 | 数据库 | 计算机科学 |
| 5 | 经济学 | 工商管理 |
| 6 | 热力学 | 工程学 |
| 7 | 机器学习 | 计算机科学 |

表 8-16
学生数据表
| `student_id` | `student_name` | `Age` | `study_program` |
| --- | --- | --- | --- |
| 1 | 爱丽丝·约翰逊 | 18 | 计算机科学 |
| 2 | 鲍勃·史密斯 | 22 | 工商管理 |
| 3 | 查理·戴维斯 | 19 | 工程学 |
| 4 | 戴安娜·金 | 21 | 计算机科学 |
| 5 | 伊桑·怀特 | 20 | 工程学 |
| 6 | 菲奥娜·布朗 | 23 | 工商管理 |
| 7 | 乔治·霍尔 | 18 | 工程学 |
| 8 | 汉娜·斯科特 | 22 | 计算机科学 |
| 9 | 伊恩·米切尔 | 19 | 工商管理 |
| 10 | 朱莉娅·洛佩兹 | 21 | 工程学 |
| 11 | 凯文·特纳 | 20 | 计算机科学 |
| 12 | 劳拉·休斯 | 23 | 工商管理 |
| 13 | 梅森·考克斯 | 19 | 工程学 |
| 14 | 妮娜·帕特尔 | 22 | 工商管理 |
| 15 | 奥利弗·斯通 | 18 | 计算机科学 |

