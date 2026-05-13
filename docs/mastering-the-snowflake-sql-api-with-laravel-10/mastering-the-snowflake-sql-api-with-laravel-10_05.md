# 4. 使用 Snowflake SQL API 的 SQL 基础

在本章中，我们将回顾 SQL 在 Snowflake 及其 Snowflake SQL API 背景下的一些基本功能。本章很大一部分内容将介绍常见的数据库和 SQL 概念，同时向您介绍 Snowflake 的特定细微差别以及 SQL API 的功能和限制。在数据管理和分析领域中，掌握基本的 SQL 函数就如同掌握了通往一个强大王国的钥匙。通过在 Snowflake 动态生态系统中提供 SQL 的构建模块，无论您是初涉数据领域的新手，还是寻求精进技能的经验丰富的实践者，本章都将成为您驾驭 Snowflake SQL API 全部潜力的门户。

我们将从探索关键的 `SELECT` 语句开始，了解其用于精确数据检索和分析的细微差别。随后对 `INSERT`、`UPDATE` 和 `DELETE` 操作的关注，为您提供了一个在应用程序中塑造和维护数据集的战略视角，这对于战略性的数据治理至关重要。然后，我们将探讨 `JOIN` 操作，这对于在数据集之间编织复杂关系至关重要。我们将理解 `JOIN` 条件和 `WHERE` 子句如何采用细致入微的条件逻辑方法，以赋能专业人士提取目标数据集，随后介绍数据聚合和分组。

为确保我们执行的是性能优化的查询，我们将探索子查询和公共表表达式（CTE）如何在单个 API 调用中用于快速排序和筛选较大数据集，然后再将其与更精细的数据连接。这种基本的 SQL 技术是我们在 Black Diamond 采用的关键策略，用于定位可优化的查询，降低运行仓库的计算成本，并减少远程和本地溢出。

最后，我们将探讨 Snowflake 的绑定变量功能。绑定使处理动态筛选条件更容易，也使处理数据类型转换更便捷。除了使代码更简洁、更易读外，它还减少了代码中冗长的 SQL 查询，选择将这项有时繁琐的任务交给 Snowflake SQL API 引擎处理，而不是由我们自己处理。

本章不仅仅是一个教程；对于寻求利用 Snowflake SQL API 掌控数据的专业开发人员和管理者来说，它是一份战略指南。当我们深入研究这些基本 SQL 函数的复杂性时，最终目标是让您掌握所需技能，以编排复杂的数据操作，并为您的应用程序成功获取可操作的见解。

## 注意事项

我们并未涵盖此代码的所有安全影响，因此让我们花点时间来讨论一下。首先，不建议您将公钥和私钥直接存储在代码中。您可以选择将其存储在本地数据库中并为代码加盐，或者安装到第三方工具中，例如 AWS Secrets Manager。保护私钥的安全对于应用程序的安全至关重要。如果您遵循了本章前面关于网络策略的讨论，您可以通过将 API 用户的登录限制在您的应用程序服务器的 IP 地址，来进一步限制其通过 UI 或 API 被使用。如果您的私钥被泄露，恶意用户还需要访问您的服务器才能用它进行任何有意义的操作。由于此密钥可能有权访问大量安全数据，还建议您不要重复使用来自任何其他系统的私钥（例如您的 SSH 密钥或 GitHub 密钥）。

另一个应该指出的风险是，这可能会允许 SQL 注入到您的 Snowflake 实例中。毕竟，我们正在使用 Laravel 通过 API 将 SQL 代码注入到 Snowflake，这正是我们想要的，但我们希望防止其被恶意使用。您可以做的一件事是，不允许任何 SQL 直接从应用程序的 URL 或表单传递给 `execute` 函数。有许多方法可以降低风险，但我认为最重要的是，不要允许您的 `execute` 函数直接接收任何由终端用户通过 URL、表单或其他方法直接生成的 SQL 查询。

最后，您可能考虑通过 `lifetime` 变量将 JWT 令牌的过期时间降低到小于 59 分钟。这确保了如果有人对您的服务器进行“中间人”攻击，他们无法抓取 JWT 令牌并在其他地方使用它。同样，如果您既保护了私钥安全，又为 API 用户实施了网络策略，就能进一步缓解此风险。

## 结论

您现在应该对 Snowflake SQL API 的基础和入门必备步骤有了扎实的理解。您已经学会了如何连接到 Snowflake、导航环境以及执行基本的数据操作。凭借这些基础知识，您已具备深入探索该平台更高级特性和功能的基础。随着您继续前行，本章所获得的技能将成为通向掌握 Snowflake 强大数据管理和分析能力的关键垫脚石。

让我们来剖析并解释我们在做什么。我们首先定义 `execute` 函数，它带有一些我们可以提供的动态参数，然后生成 JWT 令牌和 API 的基础 URL。真正的魔法发生在 `$query` 变量中，我们在那里定义了使用 Laravel HTTP 包发送的有效负载。在头部信息中，我们告知 API 我们正在发送 JSON 主体，并告诉 Snowflake 期望使用 JWT 密钥对进行身份验证。因为我们使用了 JWT，所以需要提供 HTTP 包的 `withToken()` 方法，它会自动将 JWT 令牌正确地传递给 API，并告知 Snowflake 我们期望返回 JSON 响应（错误、成功消息或查询结果）。在 `post()` 方法内部，我们将主体定义为我们想要执行的 SQL 语句、它应该针对哪个数据库和模式执行、它应使用哪个仓库以及要使用的用户角色。参数中的 `query tag` 是可选的，但当我们在 Snowsight 中使用查询历史选项卡来筛选由我们的 API 特别发送的查询以便更快地进行调试或分析时，这很有帮助。

假设我们收到了单页查询结果，并且存在多页或分区，我们可以使用 JSON 响应获取语句句柄和组中的下一个分区，并将其传递给特定于分区获取的辅助函数。

```php
public static function getPartition($statementHandle, $partition)
{
    $jwt = static::generateJwt();
    $base_url = static::createBaseUrl($account, $region);
    $query = Http::withHeaders([
        'Content-Type' => 'application/json',
        'User-Agent' => 'myApplication/1.0',
        'X-Snowflake-Authorization-Token-Type' => 'KEYPAIR_JWT',
    ])->acceptJson()->withToken($jwt)->get($base_url . '/api/v2/statements/' . $statementHandle .
    '?partition=' . $partition);
    return $query;
}
```
代码清单 3-5: Laravel Snowflake 分页

上面的代码与我们的 `execute` 辅助方法非常相似，但需要的数据要少得多。我们实际上没有传递任何 JSON 主体，甚至没有进行 `POST` 调用，而是进行了一次 `GET` 调用。这将指向我们在先前调用中已生成的结果，由语句句柄引导，并返回指定分区的结果。

现在您拥有了一个功能性的 Laravel 服务，即 `SnowflakeService` 类，您可以将此您的其他代码（例如控制器）中，并使用 `execute()` 方法通过 API 快速传递 SQL 查询。



### SQL 基础：`SELECT`、`INSERT`、`UPDATE` 与 `DELETE`

在本节中，我们将深入探讨 SQL 的核心基础知识，聚焦于四个基础操作：`SELECT`、`INSERT`、`UPDATE` 和 `DELETE`。作为数据探索和操作的支柱，这些操作在从数据集中塑造和提取洞察方面起着至关重要的作用。

#### `SELECT` 语句

`SELECT` 语句用于数据检索和分析。在此，我们将剖析其细节，探索如何构建精确的查询以揭示有价值的洞察。我们将涵盖 `SELECT` 子句的方方面面，使您能够有效地导航和解读海量数据集。`SELECT` 语句是任何以数据为核心的操作的心脏，也是每位数据专业人员常用工具库中的利器。`SELECT` 语句的多功能性，明确了在您应用场景下的目标：赋予您所需的技能，以有效地导航和解读海量数据集——无论您面对的是数百万条还是仅仅几条记录——并将该数据移交给您的应用程序进行进一步处理。

为什么我要从 `SELECT` 语句开始？因为 `SELECT` 语句是最常执行的 SQL 语句。您通过 Snowflake SQL API 执行的查询中，至少有 75%使用了 `SELECT` 子句。除此之外，理解 `SELECT` 子句的工作原理，特别是在定义要查询的列集方面，将使您更轻松地处理返回的数据结果集。首先，让我们看看 Snowflake 中 `SELECT` 语句的技术定义。

```
SELECT [ { ALL | DISTINCT } ]
[ TOP <n> ]
[ { <object_name> | <alias> } .* ]*
[ ILIKE '<pattern>' ]
[ EXCLUDE
{
   <col_name> | ( <col_name> , <col_name> , . . . )
}
]
[ REPLACE
{
   ( <expression> AS <col_name> [ , <expression> AS <col_name> , . . . ] )
}
]
[ RENAME
{
   <col_name> AS <new_col_name>
| ( <col_name> AS <new_col_name> , <col_name> AS <new_col_name> , . . . )
}
]
```
**列表 4-1** Snowflake `SELECT` 语句语法定义

起初这可能看起来很复杂，这也是可以理解的，特别是如果您是 SQL 新手，很少或没有接触过大多数数据库提供商用来解释特定 SQL 功能如何工作的语法定义格式。第一行告诉查询处理引擎我们正在执行一个 `SELECT` 子句，并指定我们想要返回哪些记录。默认情况下，引擎假定使用 `ALL`，这会返回数据集中的所有记录，如果这是您的目标，则无需指定。`DISTINCT` 关键字将从您的结果集中消除重复值。这两者都使用标准的操作顺序来执行。我的意思是，任何 `WHERE` 子句和 `JOIN` 语句将首先被处理，然后如果您指定了 `DISTINCT` 关键字，结果集将被进一步过滤。在我们的查询窗口中，我们会这样写：`SELECT DISTINCT **` 或 `SELECT **`。

结构中的下一个关键字是 `TOP <n>` 关键字。在较旧的数据库系统中，在查询中使用 `TOP <n>` 关键字有一个明确的用例。Snowflake 包含了它，就像许多其他数据库解决方案一样，以便为您的查询提供跨平台支持。然而，此关键字支持的功能与 `LIMIT` 子句相同，并且 Snowflake 认为它们是等效的，没有区别。与 `LIMIT` 子句一样，这会限制结果集中返回的行数。根据操作顺序，无论您是否使用了 `DISTINCT` 关键字，您的结果集都将首先基于您的 `WHERE` 和 `JOIN` 语句（以及我们稍后将探讨的其他子句）进行过滤，然后在特定限制内返回最终结果集。例如，在查询窗口中执行 `SELECT TOP 100 **` 将仅在结果集中返回 100 条记录。在没有使用 `ORDER BY` 子句或其他查询语法的情况下，具体使用哪 100 条记录完全由查询处理引擎决定，并且每次执行可能都不同。除了这个基本功能之外，必须知道的是，如果您在 SQL 语句中利用 `OFFSET` 子句，`OFFSET` 子句要求您使用 `LIMIT`，并且不支持 `TOP`。

接下来，`<object_name>` 或 `<alias>` 是一种标识在 `FROM` 子句中定义的对象或别名的方法。如果您要将多个数据表连接在一起，您将需要一种方法来引用每个数据集中的列。这可能会限制返回的列数，控制返回列的顺序，或提供一种方法来区分跨多个数据集中同名的多个列。

最后，`ILIKE`、`EXCLUDE`、`REPLACE` 和 `RENAME` 关键字提供了在处理特定列时进一步操作返回列（而非数据本身）的方法。`ILIKE` 将查找与模式匹配的列名，并仅返回这些列，使您能够快速从一个宽表中提取满足模式条件的列子集。类似地，如果您正在处理一个非常宽的表，并且需要返回除几列之外的所有列，使用 `EXCLUDE` 关键字来排除那几列会更容易，而不是指定您确实想要返回的大部分列。最后，`REPLACE` 和 `RENAME` 在更改列名方面提供了非常相似的功能。`RENAME` 只是将列从表中的名称更改为指定的名称（例如，将 `AVG_COST_ACCR` 更改为 `AVERAGE_COST_ACCURED`），而 `REPLACE` 关键字接受一个表达式以进行动态重命名（例如，将列名中的下划线替换为破折号）。

那么，为什么所有这些都很重要呢？嗯，除了为您提供所需的工具，以便以您能快速理解的格式引用列名，或者在跨多个表或数据集连接时选择特定列之外，这些关键字让您能够做两件事：确保您执行的是优化过的查询，并且不需要您确切知道表的结构或返回数据集的结构（尤其是在跨多个数据集连接时）。

对于前者，使用星号 (`*`) 执行 `SELECT` 查询是一种糟糕的做法。这是因为星号将按列在表结构中出现的顺序返回所有列，而我们并不总是需要所有这些列的数据。这会大大增加计算时间、通过 API 返回数据集所需的时间，并可能产生其他性能影响。

在后一个例子中，如果您必须知道表结构的确切格式，那么预测在 API 结果集中检索特定数据的难度会大大增加。当您将多个数据集连接在一起时，这种复杂性会成倍增加，您必须预测所有这些列在最终的 JSON 响应中是如何被赋予序数位置的。让我们看一些真实的例子来说明我的意思。请看以下的 API 调用。

```
$query = Http::withHeaders([
    'Content-Type' => 'application/json',
    'User-Agent' => 'myApplication/1.0',
    'X-Snowflake-Authorization-Token-Type' => 'KEYPAIR_JWT',
])->acceptJson()->withToken($jwt)->post($snowflake_api_base_url.'/api/v2/statements', [
    'statement' => "SELECT * FROM customers;",
    'database' => $database,
    'warehouse' => $connection->warehouse,
    'schema' => $schema,
    'role' => $connection->role,
    'parameters' => [
        'query_tag' => 'black-diamond',
    ],
]);
```
**列表 4-2** Laravel REST API 处理程序 — `SELECT` 所有列

具体来说，我们关注带下划线的 JSON 键。您可以看到我们正在对 `customers` 表执行 `SELECT` 查询，并告诉查询处理引擎返回该表的所有列。现在，如果您不知道该表的确切结构，您将无法轻松地从返回的结果集中提取每一行所需的数据。以下是 API 返回的结果集示例。


## Laravel API 响应处理与 SQL 操作

### API 响应数据处理

```
{
"code": "090001",
"statementHandle": "536fad38-b564-4dc5-9892-a4543504df6c",
"sqlState": "00000",
"message": "successfully executed",
"createdOn": 1597090533987,
"statementStatusUrl": "/api/v2/statements/536fad38-b564-4dc5-9892-a4543504df6c",
"resultSetMetaData" : {
"rowType": [
{
"name":"ROWNUM",
"type":"FIXED",
"length":0,
"precision":38,
"scale":0,
"nullable":false
}, {
"name":"ACCOUNT_NAME",
"type":"TEXT",
"length":1024,
"precision":0,
"scale":0,
"nullable":false
}, [. . .]
"numRows" : 50000,
"format" : "jsonv2",
"partitionInfo" : [ {
"rowCount" : 12288,
"uncompressedSize" : 124067,
"compressedSize" : 29591
}, {
"rowCount" : 37712,
"uncompressedSize" : 414841,
"compressedSize" : 84469
}],
},
"data": [
["customer1", "1234 A Avenue", "98765", "2021-01-20 12:34:56.03459878"],
["customer2", "987 B Street", "98765", "2020-05-31 01:15:43.765432134"],
["customer3", "8777 C Blvd", "98765", "2019-07-01 23:12:55.123467865"],
["customer4", "64646 D Circle", "98765", "2021-08-03 13:43:23.0"]
]
}
```
**清单 4-3** Laravel API 结果响应

请注意，我们最关心的响应键是 `data` 键。通过这个键，我们可以遍历每一行并提取数据，例如：

```
$row0 = $query_result['data'][0];
$row1 = $query_result['data'][1];
```

但如果我们想要每行数据末尾的特定列，比如 `CREATED_ON` 列，该怎么办呢？如果我们不知道表的确切结构，就必须先遍历 `[resultSetMetaData][rowType][name]` 键来获取序号键和对应的值。在这个例子中，那将是第五个键（或者说序号键为四，因为它的行为类似于数组）。你的应用程序代码仅仅为了找出需要做下面这件事，就会变得非常混乱和复杂：

```
$row0_created = $query_result['data'][0][4];
$row1_created = $query_result['data'][1][4];
```

相反，我们应该在 `SELECT` 语句中明确定义我们的列。这样做的好处是可以通过不拉取我们不计划使用的数据来优化查询，同时也让我们能够控制每一列的序号键位置。观察下面这段重构后的代码：

```
$query = Http::withHeaders([
    'Content-Type' => 'application/json',
    'User-Agent' => 'myApplication/1.0',
    'X-Snowflake-Authorization-Token-Type' => 'KEYPAIR_JWT',
])->acceptJson()->withToken($jwt)->post($snowflake_api_base_url.'/api/v2/statements', [
    'statement' => "SELECT CREATED_ON, ADDRESS, ZIP FROM customers;",
    'database' => $database,
    'warehouse' => $connection->warehouse,
    'schema' => $schema,
    'role' => $connection->role,
    'parameters' => [
        'query_tag' => 'black-diamond',
    ],
]);
```
**清单 4-4** 带列指定的 Laravel REST API 处理器

因为我们已经在 `SELECT` 语句中定义了列，并且是按照我们想要的顺序定义的，现在我们可以高度准确地预测访问特定列所需的序号键。现在我们的代码看起来像这样：

```
$row0_address = $query_result['data'][0][1];
$row1_address= $query_result['data'][1][1];
```

我们之所以能做到这一点，是因为我们知道列会按照在 `SELECT` 语句中指定的顺序返回（0 => `CREATED_ON`, 1=> `ADDRESS`, 2 => `ZIP`）。在这个例子中，我们消除了让你事先知道表结构或依赖你的应用程序来推导正确键的需要。如果我们必须依赖我们对表的了解，那么如果开发者在表中间添加了一个新列，就会将错误数据引入我们的应用程序。假设我们依赖应用程序来推导键，那么你就是在编写大量不必要的代码，每次都要遍历结果元数据来查找序号键，这既增加了开发时间开销，也增加了应用程序执行开销。

### INSERT 语句

有时，你可能需要将新数据插入到你的 Snowflake 实例中。这些数据可能是你的应用程序生成的、用户输入接收的，或是你的应用程序从第三方处理后必须持久化到数据库的数据。由于我和我的团队围绕 Snowflake SQL API 构建的应用程序性质——即一个用于 Snowflake 可观测性和性能的多租户解决方案——我们很少使用 API 向数据库插入数据。这是因为驱动我们工具的数据存在于我们内部的 Snowflake 实例中，我们使用一个 PHP 驱动程序与之交互，但有时我们不得不在应用程序数据库中为每次安装创建的客户实例中创建数据。向 Snowflake 插入数据使用 `INSERT` 子句，其语法如下：

```
INSERT [ OVERWRITE ] INTO <table_name> [ ( <col_name> [ , ... ] ) ]
{
    VALUES ( { <value> | DEFAULT | NULL } [ , ... ] ) [ , ( ... ) ] |
    <query>
}
```
**清单 4-5** Snowflake INSERT 语句语法

这里的语法比我们看到的 `SELECT` 子句要直接得多。你可以使用 `INSERT` 子句并指定列名以及每条记录的值或数据来插入新数据，或者选择性地覆盖现有数据。API 确实允许你插入多行数据，但这会给你的代码增加一些复杂性，我们将在下文讨论。需要注意的一点是：`OVERWRITE` 关键字的行为不像数据的“upsert”。相反，这是一种快速截断表中所有现有数据并用新数据替换的方法，而无需指定多个查询。

当涉及到插入数据时，插入单条数据记录相当直接。在你的 SQL 语句中，你需要指定列名，然后提供数据的变量作为值，这样你就可以构建动态的 `INSERT` 语句。可能会变得复杂的情况是：你需要填充很多列，或者你正在执行多插入操作。Snowflake（尽管是无意的）确实为此提供了一个简洁的解决方案。

简要触及一个更高级的主题，像大多数数据库解决方案一样，Snowflake 提供了创建存储过程的能力。你可以直接在 Snowflake 中创建这些过程，使用 SnowSQL 命令行工具，或者让你的应用程序通过 API 运行一次性配置脚本。例如，我的应用程序期望在客户的 Snowflake 实例中存在各种存储过程才能正常运行。当客户首次设置新连接时，系统会提示他们运行一个“部署配置”脚本。

此脚本使用 API 在他们的 Snowflake 实例中创建各种对象、任务和存储过程。我们对这个配置进行版本控制，并将其存储在连接表的一个名为 `config_version_deployed` 的列中。如果该值为 `NULL` 或设置为低于当前版本的版本，将向用户显示一个横幅，提示他们运行新的或最新的配置。因为我们必须通过 API 来完成此操作，以减少客户的手动工作，所以配置脚本确实会变得相当长和复杂。然而，权衡之下，应用程序其余部分的代码更简洁，并被认为是一个整体可靠的解决方案。

## SQL 语句的存储过程与动态查询

为什么了解这一点很重要？在需要向表中插入多条记录的场景下，我们为该表构建了一个存储过程来处理数据插入。该函数体就是我们的 SQL 语句，设置在一个循环中。该函数接受三个参数：表名、列名数组和值数组。由于我们在应用程序中将其编码为每次都以相同方式处理，我们可以确保值的一级数组键与列的顺序键相匹配，从而能够在存储过程中动态构建`INSERT`查询。

这里需要注意的一点是，我们创建了一个允许我们动态处理任何表插入的存储过程。这减少了我们需要创建的存储过程数量——一个动态过程代替了为每个要插入数据的表创建一个过程。这使我们能够遵循 DRY（不要重复自己）方法，减少了必须运行的可能消耗客户更多额度的配置升级次数，并且总体上是我们选择遵循的设计模式。你可能有一个用例，希望或需要每个表由自己的函数处理，这完全是可行的。处理这个问题没有绝对正确或错误的方式。

#### UPDATE 语句

`UPDATE`子句的工作方式与`INSERT`子句类似，使你能够更新数据库中的一条或多条现有记录。在许多情况下，你都需要更新数据，范围从非常简单到高度复杂，而该语句足够灵活，可以处理这些情况。假设你需要跟踪用户上次登录系统的时间。一个非常简单的`UPDATE`查询可以将`"last_login_timestamp"`设置为当前日期/时间，每次用户在应用程序上处理登录请求时执行。一个更复杂的例子可能是，如果你的应用程序执行各种数据计算，无论是按用户要求还是在后台进行，并且需要将这些数据持久化到数据库中。我们可以在下面看到`UPDATE`子句的语法，它相当直接。

```sql
UPDATE <table_name>
SET <column1> = <value1> [ , <column2> = <value2> , ... ]
[ FROM <from_clause> ]
[ WHERE <condition> ]
Listing 4-6
Snowflake UPDATE statement dictionary
```

由于`UPDATE`语句的简单性，你可能会误以为你的应用程序代码也会同样简单。虽然这对你的许多需求来说可能是真的，但我们必须再次审视查询如何迅速增长并对你的应用程序代码产生巨大需求。类似于`INSERT`语句，拥有一个包含大量列集合的`UPDATE`语句，甚至包含一个支持非结构化数据的列，`UPDATE`查询会迅速增长并产生大量技术债务。

如果你的查询涉及少于五个列，并且不需要处理非结构化数据，你可以将其视为一个经验法则，即可以直接在 API 调用中创建查询。然而，如果你的查询不符合此模式，或者你只是想要更简洁的代码，你可以选择像我们为插入数据那样创建一个动态存储过程。需要记住的一点是，提供至少一个`WHERE`条件非常重要，正如在 SQL 中进行`UPDATE`操作时应该始终做的那样，以防止你错误地覆盖好数据。由于需要提供这些条件，如果你选择创建一个存储过程供你的代码通过 API 调用，你需要有一个参数来支持`where`条件数组，这些条件可以动态加载到查询中。

### DELETE 语句

四个基础 SQL 子句中的最后一个是`DELETE`子句。与清空表中所有数据的`TRUNCATE`子句不同，`DELETE`子句允许你指定`WHERE`条件来有选择地删除数据，甚至允许你指定其他表或查询来支持你的`WHERE`条件。该子句的语法是四个中最简单的，但在这简单之中蕴含着成为非常强大查询的灵活性。

```sql
DELETE FROM <table_name>
[ USING <using_clause> [, <using_clause> ] ]
[ WHERE <condition> ]
Listing 4-7
Snowflake DELETE statement dictionary
```

使用`DELETE`子句时，提供至少一个`WHERE`条件至关重要，以防止你删除所有数据。如果你的意图是真正清空表，建议你使用`TRUNCATE`子句，因为它更快。`DELETE`子句必须在删除前评估每条记录（即使没有提供`WHERE`条件），而`TRUNCATE`子句则简单地清空表，而不在事先评估任何记录。

在我们的应用程序中，并且作为我们遵循的最佳实践，我们不会从数据库中删除数据。相反，我们采用一个`"is_deleted"`标志，指示记录不应再被使用，但允许我们保留数据。由于各种原因，这可能不是你应用程序的可行方法，但如果其中一个原因是存储容量，值得注意的是，存储多年来已经有了很大发展，并且在大规模下相当便宜。

你可能选择直接从表中删除数据而不是提供你版本的`"is_deleted"`标志的情况包括：遵守政府合规性（如 GDRP）、删除坏数据或清除不应长期保存或在客户流失后保留的敏感客户数据。

如果你预计需要通过 SQL API 执行数据删除，通常可以在应用程序代码中实时构建这些查询。然而，如果你正在处理复杂的`WHERE`条件，特别是涉及使用其他表或查询来细化要删除的目标数据，你可能会选择使用存储过程采用更动态的方法。同样，需要记住的两件事是：防止你的代码变得过于庞大而难以管理，以及在应用程序开发中遵循 DRY 方法。

## JOIN 操作与 WHERE 子句

既然我们已经涵盖了四个基础查询子句，我们将深入研究连接和过滤子句。这两个主题被归为一类，因为表连接为查询结果提供了附加数据，并增加了过滤数据的功能。像大多数数据库系统一样，Snowflake 支持四种常见的连接类型：内连接、外连接、交叉连接和自然连接。

正如我所说，连接可以具有过滤数据的附加功能，无论是通过连接本身的性质，还是通过提供作为过滤器作用于被连接数据的连接条件。其中之一是内连接。在内连接中，一个表中的每一行与另一个表中的一行匹配。当一个表中的一行与另一个表中的行不匹配时，这些行将从查询结果中排除。假设你有一个大学的学生列表，任何注册至少一门课程的学生被认为是活跃学生。通过在`student_id`等键上将学生与课程注册表连接，你只会得到有学生的课程，同样，只会得到有相关课程的学生。如果你将此结果进一步使用`SELECT DISTINCT`子句在`student_id`、`first_name`和`last_name`上进行过滤，你将得到一份大学活跃学生的唯一列表。我们这样做是为了考虑注册多门课程的学生，这会因内连接的性质而在结果集中产生重复的学生记录。

让我们探索 Snowflake 中的各种`JOIN`。

#### 自然连接（NATURAL JOIN）或 JOIN

使用`NATURAL JOIN`子句，或简写为`JOIN`，您可以连接两个具有相同名称和对应数据的列的表。继续我们的大学示例，我们可能有一个名为`enrolled_classes`的表和一个`class_grades`表。这两个表都会有一个名为`student_id`的列，以便将学生与其注册的课程关联起来，并将学生与其课程成绩关联起来。

由于两个表都有一个名为`student_id`的列，连接将自动构建`ON`子句并在这些列上进行连接。这意味着您不必编写`ON`条件：`[NATURAL] JOIN on enrolled_classes.student_id = class_grades.student_id`。连接的性质已经假定您想要连接这些列并为您处理。如果两个表都有多个匹配列，自然连接将在所有这些列上自动连接。

自然连接将仅包含共享列的一个记录，并省略其他记录。自然连接可以与外连接结合使用，您可以在`WHERE`子句中指定筛选条件，但不能指定自定义的`ON`子句条件。

#### 左外连接（LEFT OUTER JOIN）

左外连接包括主表中的所有行，即使有些行不匹配，以及连接表中的任何匹配行。主表，或称“左表”，是在`FROM`子句中指定的表。连接表，或称“右表”，是在连接子句中指定的表。例如，使用我们的大学示例，我们可能有以下查询：

```sql
SELECT s.full_name, s.student_id, c.class_name FROM students s
LEFT OUTER JOIN enrolled_classes ec ON ec.student_id = s.student_id
```

在此示例中，我们将获得所有学生及其学生 ID 的列表，以及`ON`子句中匹配条件所指示的他们注册的任何课程的课程名称。我们可以通过提供一个`WHERE`子句，如`WHERE c.class_name IS NOT NULL`，进一步筛选此结果，仅显示注册了课程的学生。如果我们想显示未注册课程的学生，我们会这样做：`WHERE c.class_name IS NULL`。

#### 右外连接（RIGHT OUTER JOIN）

右外连接包括连接表或“右表”中的所有行，以及主表（或“左表”）中的任何匹配行。

```sql
SELECT s.full_name, s.student_id, c.class_name FROM students s
RIGHT OUTER JOIN enrolled_classes ec ON ec.student_id = s.student_id
```

在此示例中，我们将获得`enrolled_classes`表中所有`class_name`值的行，以及`ON`子句中指定的任何匹配记录的`full_name`和`student_id`值。

#### 全外连接（FULL OUTER JOIN）

全外连接将包括两个表中的所有行，无论`ON`条件如何，为您提供一个完整的数据集，就好像所有行都在一个表中一样。匹配的行将作为单个行返回，不匹配的行将各自作为自己的行返回。任何不匹配的学生将没有`class_name`值，任何不匹配的`enrolled_classes`将没有`full_name`或`student_id`值。

```sql
SELECT s.full_name, s.student_id, c.class_name FROM students s
FULL OUTER JOIN enrolled_classes ec ON ec.student_id = s.student_id
```

#### 内连接（INNER JOIN）

在我们左外连接的示例中，我们描述了如何使用`WHERE`子句来过滤掉不匹配的行。一种更简短的写法，也是更优化的方法，是使用内连接。当您使用内连接时，它仅返回那些基于您的`ON`条件存在匹配的行。

```sql
SELECT s.full_name, s.student_id, c.class_name FROM students s
INNER JOIN enrolled_classes ec ON ec.student_id = s.student_id
```

在此示例中，仅返回`student_id`匹配的行。因此，您将始终拥有`full_name`、`student_id`和`class_name`的值。由于一个学生可能注册多门课程，您可以预期`full_name`和`student_id`记录将为学生注册的每门课程重复。

#### 交叉连接（CROSS JOIN）

最后，我们有交叉连接。这种连接获取两个表并返回所有可能的行组合（有时称为“笛卡尔积”或“笛卡尔连接”）。这意味着大多数行将包含实际上不相关的行的部分，这可能使交叉连接在许多情况下毫无用处。

使用交叉连接可能导致返回的数据集变得非常大，从而产生更昂贵的查询执行计划。当第一个表有 N 行，第二个表有 Z 行时，结果将是两者的乘积（N x Z）。这意味着第一个表有 100 行，第二个表有 100 行，将导致一个包含 100,000 行的数据集。

最后要注意的是，与自然连接一样，交叉连接不支持`ON`子句。这是因为所有可能的连接都被接受，使得`ON`子句的使用本质上无效。

## 聚合、分组与排序数据

既然我们对如何跨多个表连接数据有了很好的理解，让我们看看如何对这些数据进行提炼并赋予混乱以秩序。我们将了解一些聚合函数、分组函数和数据排序。由于这些函数非常广泛且用途多样，尤其是聚合函数，我们不会在本材料中涵盖所有内容。相反，这旨在介绍如何进一步提炼和组织数据以使结果更有意义。


### SQL 数据聚合与排序指南

#### GROUP BY

我们首先来看`GROUP BY`子句，因为它对支持聚合函数很重要。`GROUP BY`子句将具有相同`group-by-item`表达式的行进行分组，并为结果组计算聚合函数。你可以在分组中指定多个项来进一步细化结果。让我们看一个例子：

```sql
SELECT  cv.student_id FROM campus_violations cv
GROUP BY cv.student_id
```

这个查询将对两个值进行分组，并为每个唯一的分组返回一行。如果一个学生在校园内有多次违规，它只会返回一次`student_id`。这种方法可用于确定哪些学生自入学以来曾经有过校园违规。

```sql
SELECT  cv.student_id, cv.type FROM campus_violations cv
GROUP BY cv.student_id, cv.type
```

进一步细化，我们可以按类型分组。如果一个学生有多个“停车”类型的违规，你将得到一条包含其`student_id`和停车类型的记录。如果该学生有其他违规类型，每种类型都将返回一条该`student_id`和其违规类型的记录，导致你看到该`student_id`因不同类型而出现多次。

```sql
SELECT  cv.student_id, cv.type FROM campus_violations cv
GROUP BY cv.student_id, cv.type
HAVING COUNT(cv.student_id) > 1
```

到这里，也许我们只想看到屡犯者。在这里，我们通过告诉查询仅当`student_id`在校园内有多次违规时，才返回其`student_id`和关联的违规类型，来进一步优化查询。如果某个`student_id`在表中只出现一次，它将不会被返回。`HAVING`子句允许你定义通用表达式来评估`GROUP BY`如何返回数据。

```sql
SELECT  cv.student_id, cv.type, COUNT(cv.full_name) AS violation_count FROM campus_violations cv
GROUP BY cv.student_id, cv.type
HAVING COUNT(cv.student_id) > 1
```

最后，通过在`SELECT`子句中提供聚合函数`COUNT`，我们可以了解一个学生每种类型的违规次数。如果一个学生有六次停车违规，你将看到一条包含`student_id`、停车类型和数值 6 的记录结果。

需要注意的是，当你使用`GROUP BY`子句时，你必须要么匹配你的`SELECT`子句，使`SELECT`子句中出现的任何列也出现在`GROUP BY`子句中；要么你必须将不匹配的列包装在聚合函数中。在前面的例子中，你会注意到`full_name`被包装在`COUNT`聚合函数中，但`full_name`没有出现在`GROUP BY`子句中。这是因为聚合函数本质上已经将数据分组在一起。

#### 聚合函数

聚合函数是转换数据以便进行更有意义探索的手段。你可以使用`AVG`函数获取所有值的平均值，使用`SUM`函数获取所有值的总和，使用`MIN`函数获取最小值，等等。让我们回到大学的例子，看一个查询示例：

```sql
SELECT s.full_name, s.student_id, g.grade FROM students s
INNER JOIN grades g ON g.student_id = s.student_id
```

此示例将返回所有学生及其成绩列表。由于一个学生可能有多个成绩（取决于他们是否选修了多门课程），你会看到`full_name`和`student_id`重复出现多次。我们可以进一步优化此查询，为学生定义一个平均绩点：

```sql
SELECT s.full_name, s.student_id, AVG(g.grade) AS grade_average FROM students s
INNER JOIN grades g ON g.student_id = s.student_id
GROUP BY s.full_name, s.student_id
```

这个例子引入了我们的第一个聚合函数，平均值函数（`AVG`），并定义了列名（`AS grade_average`）。如果我们没有定义列名，结果集将返回列名为`AVG(g.grade)`。返回的列表将为你提供唯一的`full_name`和`student_id`值列表，以及匹配记录的所有成绩值的平均值。如果我们想知道每个学生收到的最低成绩是什么，我们可以在查询中添加`MIN(g.grade)`，它将返回该学生成绩列表中的最低分。

#### 数据排序

最后，让我们看看如何对数据进行排序。当你返回数据集时，你可能需要它以特定的信息顺序输出。也许你希望所有的`student_id`出现在一起，或者你希望每条记录的最新版本排在前面。当你使用`ORDER BY`子句时，你可以指定数据如何返回。

```sql
SELECT s.full_name, s.student_id, s.enrolled_date FROM students s
ORDER BY s.enrolled_date DESC
```

在这个例子中，我们返回所有曾经在大学注册过的学生列表。`ORDER BY`子句表示按`enrolled_date`降序排序。结果集将返回最近入学的学生，然后时间向后推移。如果你将`DESC`标志改为`ASC`，它将返回最老的学生，即最早入学的学生，然后时间向前推移。你也可以为排序指定多个`ORDER BY`条件。

```sql
SELECT s.full_name, s.student_id, s.enrolled_date, s.last_seen_date FROM students s
ORDER BY s.last_seen_date DESC, s.enrolled_date DESC
```

在这个例子中，我们将获得所有曾经在大学注册过的学生的列表。你会首先看到他们在大学系统中最后出现的时间顺序。如果多个学生有相同的`last_seen`日期（比如新学期开始），则会进一步按最近入学的学生排序。

重要的是要知道，如果没有`ORDER BY`子句，查询返回的结果是一个无序集合。这意味着如果你多次运行相同的查询，每次运行的输出排序可能不同。如果顺序很重要，或者你想有更大的机会重用先前查询运行的缓存数据，你应该利用`ORDER BY`子句。



