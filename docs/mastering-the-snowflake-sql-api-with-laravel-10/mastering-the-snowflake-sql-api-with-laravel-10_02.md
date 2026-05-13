# 1. Snowflake

## 为什么选择 Snowflake？

Snowflake 的三位创始人于 2012 年 7 月在加利福尼亚州圣马特奥创立了公司。在几年内，随着其客户群和投资资本以指数级速度持续增长，该公司成为云数据或“数据即服务”的领导者。到 2023 年，公司收入达到 20.67 亿美元。尽管选择 Snowflake 而非其他云数据提供商的原因有很多，但该公司专注于其价值主张，即客户投资回报率（ROI）提高 612%，每天能够运行超过 26 亿次查询，以及拥有超过 1300 个合作伙伴的丰富生态系统。再加上该平台不断增长的数据库驱动程序解决方案、强大的 API、无与伦比的计算和数据基础设施，以及包括对 AI 和 ML 的原生支持在内的新增功能路线图，很容易理解该平台的吸引力。

采用 Snowflake 的公司这样做是因为它们不想承担数据库基础设施和网络正常运行时间的开销和管理，而是选择严重依赖 Snowflake 的快速数据访问、按需仓库初始化及其经过验证的后端基础设施。利用 Snowflake，公司可以通过利用 Snowflake 的数据仓库即服务来摆脱传统数据库解决方案所需的大部分 DBA 开销，从而允许公司将其时间、资源和金钱集中在数据仓库、数据分析以及 AI 和 ML 计划上。

随着 Snowflake 的持续增长，他们采用了“一个平台，拥有近乎无限潜力”的口号。这包括在最佳性能下执行工作负载的能力；完全自动化安全、治理和可用性；以及使用他们的数据市场在全球范围内进行安全协作。Snowflake 的独特架构设计使企业能够跨云进行全球连接，以调动其数据并打破数据孤岛。

## 什么是 Snowflake API？

`Snowflake SQL API` 是一个 `RESTful API`，任何人都可以使用它来访问和更新 `Snowflake` 实例中的数据。该 API 支持执行查询操作、配置用户和角色、创建表以及管理 `Snowflake` 部署的许多其他方面。通过利用 API 开发最佳实践，Snowflake 使用户能够提交 `SQL` 语句以供执行，包括大多数 `DDL` 和 `DML` 语句、检查任何语句执行的状态、按需取消语句执行以及提供并发获取查询结果的能力。

`API` 代表应用程序编程接口（Application Programming Interface），它是一个应用程序提供给其他应用程序的软件中介，允许两个应用程序相互通信。`RESTful` 标准代表表述性状态转移（REpresentational State Transfer），是一种架构风格。`REST` 定义了一套原则和标准协议，API 可以围绕这些原则和协议构建。`REST` 是构建 API 的广泛接受的架构风格。

`Snowflake` 在其 `SQL API` 中采用数据分区架构，以帮助用户获取查询结果，同时在底层确定返回的分区数量和每个分区的大小。为了增强安全性，可以通过利用 `Snowflake` 网络策略来保护 `SQL API`，这些策略限制了对启用 API 的账户的访问。虽然其功能近乎无限，但存在一些功能限制，包括无法对 `Snowflake Stage` 使用 `GET` 和 `PUT` 命令。此外，`SQL API` 目前不支持利用 `Python` 编程语言的存储过程。



## Snowpark API

除了 Snowflake SQL API，Snowflake 还提供了一个直观的 API 用于与 Snowpark 库交互。使用此 API，你可以在本地开发环境中编写 Snowpark Python 代码，并在 Snowflake 中原生执行。这使得用户能够创建和利用 Python 存储过程、强大的 Python 库（如 `DataFrame` 和 `Pandas`）、创建 `UDF` 和 `UDTF`，以及执行机器学习任务。

本书的范围主要集中在 Snowflake SQL API 上，但部分章节也会涉及 Snowpark API 的相关方面，以便用户能更全面地理解 Snowflake API 的功能。

关于 Snowpark API 的更多信息，你可以参考官方的 Snowpark Library for Python API 参考文档：[`https://docs.snowflake.com/developer-guide/snowpark/reference/python/latest/index`](https://docs.snowflake.com/developer-guide/snowpark/reference/python/latest/index)。

## 为何选择 Laravel？

Laravel 是一个强大的 PHP 框架，它为开发者提供了轻量级的代码库、广泛的包与社区生态，以及在模型-视图-控制器 (MVC) 遵从度不同层级上开发的灵活性。当我最初开始使用 Snowflake SQL API 时，是为了构建我们的 `Black Diamond 360` ([`https://blackdiamond360.com/`](https://blackdiamond360.com/)) 解决方案，用于管理和监控 Snowflake 实例。由于该平台是作为一个多租户解决方案构建的，并且由于 PHP 驱动的有限可用性或缺乏开箱即用的 Laravel Snowflake 包，我们的团队选择转而利用 Snowflake API。这使我们能更轻松地设计和开发多租户解决方案，并提供了 Snowflake 随 SQL API 附带的安全性的额外好处。

本书的大部分内容旨在为开发者提供使用 Snowflake SQL API 所需的知识。因为 SQL API 是一个 RESTful API，所以任何支持 REST API 调用的编程语言或框架都可以使用。

## Snowflake SQL API 与驱动程序

如果你对 Snowflake 或大多数数据库解决方案有高级接触经验，你可能已经知道数据库驱动程序的存在。驱动程序是在你的应用程序中原生支持 Snowflake 的一种好方法。虽然它们比 SQL API 的限制更少，但它们也有一些缺点。首先，在本书撰写时，Snowflake 仅支持七种驱动程序（`GO`、`JDBC`、`.NET`、`Node.JS`、`ODBC`、`PHP PDO` 和 `Python`）。其次，驱动程序升级可能意味着在你的应用程序强制使用废弃版本之前，需要进行更密集的测试和代码更改。

虽然驱动程序在许多应用程序中扮演着明确的角色，但有时你可能会选择使用 API 的方式。例如，在我们的 `Black Diamond` 工具中，我们运行一个多租户应用程序，允许客户将一个或多个 Snowflake 实例连接到该工具。虽然在 Laravel 或其他语言中，动态配置加载 Snowflake 连接到支持的驱动程序是可行的，但这确实引入了一些复杂性甚至潜在的安全风险。

你可能使用 API 而不是驱动程序的其他用例则更直接一些。为 iOS 或 Android 构建原生移动应用程序，其中主要应用程序是 Web 应用，而移动应用程序是支持性接口，API 可以让你快速高效地连接到你的 Snowflake 数据以拉取、推送、更新甚至删除数据。如果你正在使用没有驱动程序支持的集成工具，或者你处于安装驱动程序受限的环境中，SQL API 也能帮助克服这一点，让你访问 Snowflake 中的数据。

总的来说，在任何使用驱动程序可能造成复杂性或限制的情况下，SQL API 的优势就开始显现。但也许我个人最喜欢 API 的一点是，Snowflake 维护了一个 Snowflake API 状态页面，该页面可以被程序化地查询。通过这种方式，我的应用程序可以快速判断 Snowflake 是否出现中断，然后适当地处理中断。

## 理解 Snowflake SQL API 架构

创建 Snowflake SQL API 是为了给开发者提供一个更轻量级的选择，无需安装沉重的包、库或驱动程序，也无需使用现有 Snowflake 驱动程序不支持的语言。该 API 提供了一组端点，允许你通过标准的 `HTTP` 传输层安全 (`TLS`) 协议发出 SQL 命令并获取这些命令的结果。所有结果都以 `JSON V2` 格式返回，为开发者提供了一种简单的方法，使用几乎所有现代编程语言中可用的标准库来构建客户端。

虽然 Snowflake 将其 API 标识为 RESTful API，但它并不完全符合传统 `REST` API 定义的标准。`REST` API，根据定义，其资源端点采用 URL 形式，通常通过 `HTTP` 命令 (`GET`、`DELETE`、`PATCH`、`PUT` 等) 进行操作。相反，Snowflake SQL API 只提供了几个端点 URL，旨在作为向 Snowflake 发送 SQL 语句并通过 `HTTP` 检索结果的媒介，类似于 `RPC` 服务。

当用户向端点提交请求时，API 首先判断其是否为有效请求。如果无效，则返回一个 `HTTP` 错误代码，并停止处理。如果请求有效，它会创建一个对 Snowflake 实例的 `HTTP` 调用，并将查询提交给处理器。然后处理器开始执行语句，并可能处于三种状态。第一种状态是“睡眠状态”，API 在此状态下等待一段设定的时间，然后再次检查结果。第二种状态是错误状态，在此状态下语句返回 Snowflake 错误，并作为 `HTTP` 错误传递回 API 响应体。最后一种状态是成功状态，在此状态下 API 响应体返回一个 `HTTP` 成功响应以及有效载荷的 `JSON` 主体。

在第三种成功状态下，API 响应体还可能包含对额外分区的引用。因为某些数据集可能太大而无法一次性传输，Snowflake 会将它们拆分成多个分区。随后的请求可以发送到 Snowflake SQL API，以获取额外的分区或数据页，其工作方式类似于大多数现代编程语言中的分页。

要执行一个新的语句，开发者必须构造要传递给端点的 `HTTP` 请求主体，包括要执行的 SQL 语句，并将其作为 `POST` 请求发送给 API。Snowflake 提供了包含自定义 `UUID` 的选项，而不是自动生成的 UUID，以帮助识别重复请求并使分区获取更容易。我们稍后将更详细地讨论其工作原理。你还可以指示是否要异步执行语句，允许自定义应用程序在指定如何处理查询响应时更具灵活性。

