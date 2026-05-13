# 管理员

## 查询历史

查询历史允许您查看在您的 `Snowflake 实例` 中执行的所有查询。系统提供了多种过滤器来缩小查询范围，其中最有用的是执行查询的`用户`，这有助于更轻松地调试 API 调用中出现的意外结果。`SQL 文本`会向您展示在 Snowflake 中执行的确切 SQL 语句，这有助于了解您通过 API 传递的 SQL 是否被 Snowflake 正确接收。您还可以使用`查询 ID`（如果您从 API 结果中获取了 ID）进行额外过滤，以及查看`通过/失败`状态和执行查询的`用户`。Snowflake 提供了大约十列，您可以根据需要查看的信息配置显示在屏幕上。

如果您深入查看任何一个查询活动，您将看到有关该查询的更多详细信息，包括完整的 `SQL 文本`以及 Snowflake 针对该查询返回的结果或响应。`查询概况`选项卡可能是查询历史中最有用的部分，它为您提供了查询的完整执行计划。如果某个查询加载结果时间过长，或者根据您的超时设置即将超时，您可以使用此视图来更好地理解查询的执行情况以及查询的每个功能耗时多久。利用这些详细信息，您可以开始缩小查询中可能受益于优化的元素范围，从而获得更好的整体查询性能。

## 复制历史

复制历史为您提供了 Snowflake 中 `COPY` 命令的详细视图。使用这些信息，您可以查看正在复制到数据库或从数据库复制出的任何数据文件的进度，查看过去的加载记录，并深入了解单个加载请求，以了解更多关于进入您 Snowflake 实例的数据及其来源。

## 任务历史

如果您正在利用 Snowflake 的 `Tasks` 来运行计划或手动的查询、存储过程或函数，`任务历史`页面会为您分解说明了哪些任务已运行、何时运行、运行状态以及其他重要的元数据。深入查看任何任务将带您进入其`运行历史`页面，在那里您可以查看过去的运行记录、运行结果以及下一次任务计划运行的时间（如果您使用的是计划）。

## 动态表

`动态表`允许您物化指定查询的结果，从而无需创建单独的表来处理转换和数据更新功能。相反，您可以将目标表定义为动态表，并指定执行任何转换的查询语句。动态表会执行刷新过程，以定期更新数据，从而取代完整的 `insert`、`update` 和 `delete` 查询的需要。

`动态表`选项卡允许您查看已创建的动态表、其上次刷新时间以及其他有价值的元数据，这些信息可以帮助您调试预期数据未出现在表中的原因。

## 使用情况

`使用情况`屏幕允许您详细了解 Snowflake 的支出情况。您可以使用各种过滤器，按仓库、标签、用户、服务等可视化您的支出情况。市面上有许多付费和免费的支出工具可以利用这些数据，为您提供更深入的洞察和支出优化，例如 Black Diamond ([`https://blackdiamond360.com`](https://blackdiamond360.com))。

建议您至少每月使用此页面或各种可用工具之一检查您的支出情况，以了解是否存在异常查询或仓库正在消耗大量积分，从而不必要地增加您组织的开销。随着您对 Snowflake SQL API 越来越熟悉并开始更深入地使用它，此页面尤其有助于显示您是否正确优化了您的 API 代码，并帮助识别恶意用户可能正在访问您的 API 并无谓地增加积分消耗的领域。

## 仓库

当您首次设置 Snowflake 实例时，您会在 `仓库`部分花费相当多的时间。Snowflake 仓库是 Snowflake 数据库的一组计算资源。这些资源包括 CPU、内存分配和临时存储。Snowflake 为仓库使用 `T 恤尺码` 大小调整（`x-small`、`small`、`medium`、`large` 等），每个大小使用不同的计算方式来确定其运行时消耗的积分数量。除了仓库大小调整，您还可以配置仓库集群。仓库的这一方面会对您的积分消耗产生额外影响。由于其定价模型，根据当前需求配置每个仓库非常重要。仓库可以随时升级或降级，且不影响下游代码，因此请谨慎行事，并明白“少即是多”。

配置仓库时，您需要考虑查询的复杂性、仓库将返回的平均数据量以及向仓库提交查询的频率。对于需要很长时间返回的复杂查询或大型数据集，您需要增大仓库大小。更大的仓库可以处理更复杂的 SQL 查询，并比小仓库更快地返回数据。另一方面，对于接收大量并发 SQL 请求（无论大小）的仓库，更大的仓库并无帮助，因为每个仓库的并发限制由每个查询的大小和复杂性决定。在这种情况下，您需要启用仓库集群。这允许您的仓库扩展出额外的节点来处理可能原本会排队等待的查询。

在为 Black Diamond 使用 Snowflake SQL API 时，我们采用了跨 API 使用单一共享`仓库`的方法。这是因为我们不会将客户数据存储在自己的 Snowflake 实例中，并且除了少数例外情况，我们收到的并发请求的复杂性和数量都是低风险，可以通过一个中等大小的仓库和最多两个节点的集群来缓解。相反，在您创建账户时，会在您的 Snowflake 实例中创建一个专用的仓库，Black Diamond 请求发送到您的实例时，应由 Snowflake SQL API 专门使用此仓库。

我们创建了一些额外的统计信息来提取有关您仓库的详细信息。利用这些详细信息，我们的团队可以手动审查那些遭受应用速度缓慢困扰的账户，并采取手动操作修改 API 使用的仓库以进一步优化它。此外，工具内的自动化功能可以采取一些有限的自导向操作来缓解性能问题，同时努力保持积分消耗成本较低。

## 资源监控器

如果您选择不为 Snowflake 实例使用第三方工具，`资源监控器`是监控积分消耗的绝佳方式。您可以将资源监控器分配给一个或所有仓库，并为每个仓库指定积分配额。此配额可以设置为按日、周或月触发刷新。此外，您可以提供暂停设置，以防止仓库超过其分配的使用量。当您刚开始使用 Snowflake SQL API 进行开发时，提供一个带有截止限的低配额很有帮助，可以防止异常代码在无人监管时消耗大量积分，并随着您的应用程序发展和您对 API 越来越熟悉而逐渐提高该配额。



## 用户与角色

我们在本章前面已经介绍过 `用户与角色`，但本节将让你更深入地了解系统中分配的用户、他们上次登录的时间以及他们被授予的角色。你可以在 `用户` 选项卡中管理用户的所有方面，包括创建和删除用户。`角色` 选项卡则允许你对组织内的角色执行许多相同的操作。此外，它还提供了一个有用的角色思维导图视图，让你可以快速可视化角色的布局以及为每个角色设置的继承关系。

## 安全性

`安全性` 部分允许你为组织定义 `网络策略`，并查看当前连接到 Snowflake 的开放会话（用户）。`网络策略` 是进一步加固 Snowflake 访问安全的好方法，它提供了通过特定 IP 地址登录或阻止特别有问题的 IP 地址的能力。我强烈建议你至少要求所有用户使用多因素认证（`MFA`），特别是如果他们不在内部网或专用的 `IP 地址/IP 范围` 内，而你无法将访问权限限制在这些地址之内。

由于无法通过 API 支持多因素认证，强烈建议你为服务器的 `IP 地址` 创建一个 `策略` 并将其分配给你的 API 用户。除了你创建的 `公钥/私钥对` 之外，万一你的密钥泄露，这将提供额外的一层安全保障。

## 账单与条款

`账单与条款` 部分提供了你 Snowflake 合同的完整详情。它概述了你已接受的 `服务条款` 及其接受时间、账单联系人、你的支付方式，以及你所有使用情况声明的可下载版本。请注意，对于与 Snowflake 签订预付容量合同、向其账户分配大量积分的用户，可能不会看到列出的 `支付方式`。

## 联系人

`联系人` 页面允许你设置将接收来自 Snowflake 通知的电子邮件地址。最重要的是 `安全通知`，我建议将其分配给能够定期审查这些通知并采取相应行动的人。`隐私通知` 让你了解 Snowflake 发布的最新隐私政策变更或隐私问题。相比之下，`产品通知` 则会让你了解 Snowflake 的行为变更、性能下降、API 变更以及其他主动通知。

## 账户

通常，除非你所在的大型组织有特定的数据需求（包括数据本地化和数据治理策略），否则你不会花太多时间在 `账户` 页面上。当你首次注册 Snowflake 时，会创建一个默认的 `账户`，该账户同时也作为你 Snowflake 实例的组织管理账户。如果你发现自己需要额外的 `账户`，可以在此处创建和管理它们。

你可能使用 `账户` 的一个例子是，如果你为多个客户提供服务，并希望按 `账户` 而不是数据库和角色来隔离客户的账户。这样可以让客户更好地控制他们的数据和数据访问权限，而不会冒数据暴露给你可能服务的其他客户的风险。

另一个例子是，假设你有数据因治理原因必须保留在某些区域。一个很好的用例是澳大利亚的教育数据，政府规定此类数据不能离开澳大利亚国境。你可以轻松设置一个新的 `账户`，该账户位于澳大利亚的服务器上，并将所有特定于澳大利亚的数据存储在那里，从而满足政府法规的要求。你还可以设置一个 `账户`，该账户从你的主 `账户` 复制一份数据，并将其存储在地理位置靠近你一大群客户的服务器上，这样他们就不必提交需要跨越半个全球才能检索数据的网络请求，从而能够更快地访问他们的数据。

## 合作伙伴连接

`合作伙伴连接`，类似于 `市场`，允许你直接连接到为 Snowflake 构建工具的合作伙伴。一个流行的工具，数据 DevOps，是 `DataOps.Live`。这个工具允许你使用 `DataOps` 和底层的 `DBT` 功能来管理 Snowflake 实例的 `CI/CD` 管道，为你提供真正的敏捷数据摄取、建模和转换过程。

因为这些合作伙伴是专门为 `Partner Connect` 构建的，所以你可以快速在 `Partner Connect` 门户内授予对 Snowflake 实例的必要访问权限。这确保你不必将敏感信息（如登录详情）传递给第三方网站，如果该系统未采用适当的安全标准，这些信息可能会被泄露。

当你利用 `Snowflake SQL API` 来构建你的项目时，你可能希望记住 `Snowflake Partner Connect`。它不仅为你提供了一种更安全的方式，让你的客户将其应用程序连接到他们的 Snowflake 实例，而且还通过允许你在 `Partner Connect` 门户上列出，提供了另一个推广渠道。

### 访问 Snowflake SQL API

在本节中，我将帮助你开始使用 `Snowflake SQL API`。如前所述，`Snowflake SQL API` 是一个 `REST API`，可用于访问和更新 Snowflake 数据库中的数据。使用该 API，你可以执行标准的 `SQL` 查询以及大多数 `DDL` 和 `DML` 语句，还可以使用该 API 从 Snowflake 数据库访问有关你的 Snowflake 实例的 `元数据`。在本节中，我们将使用 `Laravel 10` 来向你展示如何连接到 `Snowflake SQL API`。这里假设你已经在 `服务器` 或 `本地主机` 上安装并运行了基础版 `Laravel 10`，并且熟悉 `Laravel` 和 `PHP`。




### 创建雪花（Snowflake）服务

为了确保我们的代码遵循 **DRY（不要自我重复）** 原则，请导航到 `app ➤ Services` 并创建一个名为 `SnowflakeService.php` 的新文件。在文件开头，你需要包含一些随着深入开发会变得必要的包。

```php
<?php
namespace App\Services;
use Carbon\Carbon;
use Firebase\JWT\JWT;
use Firebase\JWT\Key;
use Illuminate\Http\Request;
use App\Http\Controllers\Controller
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Http;
use violuke\RsaSshKeyFingerprint\FingerprintGenerator;
class SnowflakeService
{
// 此处存放未来代码
}
```

**代码清单 3-1** Laravel 雪花服务

以上代码定义了我们的命名空间和一些重要包，包括 Firebase JWT 包和由 `` `violuke` `` 提供的 `RsaSsheKeyFingerprint` 包。现在包已经就位，我们需要创建获取 JWT 令牌的代码，该令牌将用于你的 API 调用头中。你需要为你的账户生成一对公钥和私钥。为此，请下载 SnowSQL 命令行工具包，然后执行以下命令：

首先，生成一个未加密版本或一个加密版本。

```
$ openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt
```

或者

```
$ openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 des3 -inform PEM -out rsa_key.p8
```

此命令会生成一个 PEM 格式的私钥。请确保其外观与下面类似：

```
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIIE6TAbBgkqhkiG9w0BBQMwDgQILYPyCppzOwECAggABIIEyLiGSpeeGSe3xHP1
wHLjfCYycUPennlX2bd8yX8xOxGSGfvB+99+PmSlex0FmY9ov1J8H1H9Y3lMWXbL
...
-----END ENCRYPTED PRIVATE KEY-----
```

拿到私钥后，你需要生成一个公钥。使用以下命令：

```
$ openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```

此命令将创建一个文件，其内容应包含类似格式：

```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAy+Fw2qv4Roud3l6tjPH4
zxybHjmZ5rhtCz9jppCV8UTWvEXxa88IGRIHbJ/PwKW/mR8LXdfI7l/9vCMXX4mk
...
-----END PUBLIC KEY-----
```

将这两个文件存储在本地计算机上一个安全的私有目录中。完成后，在 Snowflake 中打开一个新的 SQL 工作表，并将公钥分配给你的用户：

```
ALTER USER johndoe SET RSA_PUBLIC_KEY='MIIBIjANBgkqh...';
```

既然你已经将公钥分配给了 API 用户，我们就可以开始构建 JWT 令牌生成所需的代码了。回到你的 Laravel 代码，打开我们之前创建的 `SnowflakeService.php` 文件。按照下面的代码创建一个名为 `generateJWT` 的新函数。

```php
public static function generateJwt()
{
$account = '账户标识符'
$username = 'snowflake_user_name'
$qualified_username = $account . '.' . $username;
$now = Carbon::now('UTC')->timestamp;
$lifetime =  Carbon::now('UTC')->addMinutes(59)->timestamp;
$public_key = '粘贴公钥'
$private_key = '粘贴私钥'
$fingerprint = FingerprintGenerator::getFingerprint($public_key, 'sha256');
$payload = [
'iss' => $account . '.' . $username . '.SHA256: ' . $fingerprint,
'sub' => $qualified_username,
'iat' = $now
'exp' = $lifetime
];
$jwt_token = JWT::encode($payload, $private_key, 'RS256');
return $jwt_token;
}
```

**代码清单 3-2** Laravel JWT 生成

让我们花点时间解析一下上面的代码。我们首先定义了用于生成 JWT 令牌的静态函数，并指定了令牌本身以及指纹所需的几个变量。当我刚开始使用雪花 SQL API 时，我低估了指纹的重要性；我使用了由 SnowSQL 生成的指纹，并以为那就是我所需要的全部，因为它“正常工作”了大约两周。当它停止工作时，我通过逆向工程应用程序与 SnowSQL 生成的 JWT 令牌进行了大量调试，才弄清楚指纹有多么重要。此外，当时也没有很多关于如何使用 PHP 甚至 Laravel 获取此指纹的优质文档，所以我希望这能为未来的用户提供他们所需的基础，让他们第一次就能正确完成。

一旦你从公钥生成了指纹，并定义了其他一些唯一变量，我们就将它们传递到一个负载（payload）数组中。`JWT::encode()` 函数接受该数组作为第一个参数，私钥作为第二个参数，然后我们告诉它使用 `RS256` 进行加密和验证。如果你正确填写了所有内容，你将获得一个 JWT 令牌，该令牌稍后可以在 `SnowflakeService` 类中使用，或者你可以将其打印到屏幕上并通过 `snowsql` 命令行进行测试。

接下来的代码段很小，但它是我创建的一个辅助方法，因为我开始处理多个雪花账户。API URL 实际上随每个雪花**账户**（而非用户）而变化，它是指定你要进行身份验证和执行操作的目标实例的方式。无论你是访问单个账户还是多个账户，我仍然推荐使用这个辅助函数，因为随着应用程序的增长，它可以在代码的其他领域发挥作用。

```php
public static function createBaseURL($account, $region)
{
$snowflake_api_base_url = 'https://' . $account . '.' . $region . '.snowflakecomputing.com';
return $snowflake_api_base_url;
}
```

**代码清单 3-3** Laravel 雪花基础 URL 配置

这个函数非常直观，它生成了指定代码应向何处发起 API 调用所需的唯一 URL。我们不提供任何 REST 函数路径，例如 `/api/v2/statements`，因为我们实际上可以调用三个函数，我们希望保持灵活性以便动态附加。接下来我们将看到的两个函数是“魔法”发生的地方。第一个是一个基本的执行函数，它处理了大多数 SQL 语句的 API 头的许多前期配置，第二个函数是我们处理分区数据的方式，因为 API 返回的是“页”数据，你必须以分页的方式请求它们。

```php
public static function execute($account, $region, $database, $schema, $statement)
{
$jwt = static::generateJwt();
$base_url = static::createBaseURL($account, $region);
$query = Http::withHeaders ([
'Content-Type' => 'application/json',
'User-Agent' => 'myApplication/1.0',
'X-Snowflake-Authorization-Token-Type' => 'KEYPAIR_JWT',
])->acceptJson()-withToken($jwt)->post($base_url . '/api/v2/statements', [
'statement' = $statement,
'database' = $database,
'warehouse' => 'warehouse_name',
'schema' => $schema,
'role' => 'role_name' // 可选择作为函数参数提供
'parameters' => [
'query_tag' => 'my-application-name',
],
]);
return $query;
}
```

**代码清单 3-4** Laravel 查询执行



