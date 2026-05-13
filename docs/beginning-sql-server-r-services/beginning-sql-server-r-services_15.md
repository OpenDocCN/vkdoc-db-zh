# 软件需求文档

### 项目

SQL Server R Services 入门

### 作者

Bradley Beard，首席开发工程师，We R Pros, LLC

### 接受方

R. Customer，首席信息官，PleaseGiveUs-R-AnalysisData.com

### 问题

PleaseGiveUs-R-AnalysisData.com（“客户”）向 We R Pros, LLC（“供应商”）提出了一个独特的请求。

客户拥有有限数量的天气相关数据，但客户没有分析这些数据的方法。客户希望供应商能够建议哪些产品和服务可以从这些数据集的分析中提炼出来。

本质上，客户并不确切知道自己想要什么，但知道自己想要点什么，并且依赖供应商的专业知识和经验为客户提供一个或一套解决方案。这些数据相当宝贵，其中无疑隐藏着不少“璞玉”。

### 解决方案

在初步检查客户的数据后，收集到以下信息：

*   数据文件是 1 个 Microsoft Excel 文件。
*   文件大小约为 5.5MB。
*   文件长度正好是 113,333 行（减去一行标题行）。
*   文件包含以下标题行：
    *   `Year`
    *   `AdjustedMonth`
    *   `AdjustedDay`
    *   `AirportID`
    *   `AdjustedHour`
    *   `Timezone`
    *   `Visibility`
    *   `DryBulbFarenheit`
    *   `DryBulbCelsius`
    *   `DewPointFarenheit`
    *   `DewPointCelsius`
    *   `RelativeHumidity`
    *   `WindSpeed`
    *   `Altimeter`

根据这些列，确定将创建以下报告：

*   每个 `AirportID` 的平均风速
*   每个 `AirportID` 的平均温度（°F）

需要注意的是，可以生成更多报告，但客户特别要求创建这些报告。

### 语言/平台

本解决方案将使用 SQL Server 2016 收集和存储数据，使用 R 分析数据，并使用 SQL Server Reporting Services 提供基于数据的报告。

### 交付媒介

报告的交付媒介将按客户要求确定；报告可以图像格式、Excel 格式或 PDF 格式交付。具体格式可以在日后确定，因为通过 SQL Server Reporting Services 的交付方式允许以多种格式导出报告。


