# 第八章 测试

结构测试的目标之一是验证所设计的数据库结构及其对象是否满足业务需求。因此，我不会预先创建大量表，而是逐一审查每个对象，并详细说明我对每个数据库对象的设计决策。请记住，这个业务场景（以及所有数据库对象）是虚构的，并非生产就绪；最终得到的数据库将是最基本的骨架。

让我们从创建数据库开始。公司名为 Bean About Town（纯属虚构，尽管快速搜索显示伦敦至少有三家精明的公司有同样的想法）。对于接下来的命令，我们将使用 root 用户。让我们打开一个 CockroachDB shell 并创建数据库 `bean_about_town`。我们将把所有数据库对象都安排在这个数据库下：

```
$ cockroach sql \
--certs-dir=certs

CREATE DATABASE bean_about_town;
USE bean_about_town;
```

数据库将有三个主要用户组：客户、咖啡烘焙部门成员和会计部门成员。让我们创建三个具有简单密码的用户，以确保权限和特权得到合理的隔离。

我们将有一个可以访问面向客户表的 `retail_user`，一个可以访问烘焙相关表的 `roasting_user`，以及一个可以访问账目相关表的 `finance_user`：

```
CREATE USER retail_user WITH LOGIN PASSWORD '7efd7222dfdfb107';
CREATE USER roasting_user WITH LOGIN PASSWORD '9389884141ecaf04';
CREATE USER finance_user WITH LOGIN PASSWORD 'a02308ce58c92131';
```

在整个业务中，我们将处理不同类型订单的创建和接收：

-   `Retail` – 接收客户订单。
-   `Roasting` – 为原材料下采购订单，并接收来自业务零售端（例如，一个履行在线订单的仓库需要烘焙咖啡豆）的订单。为简单起见，在此示例中，我们只为采购订单创建表。
-   `Finance` – 将客户订单与在线销售对账，并将烘焙订单与公司库存对账。我们暂时忽略财务部门为其所需物品下采购订单的需求。

了解到我们需要概念化多种订单类型，很明显，创建 schema 将简化此过程。现在让我们创建一些 schema 和数据库对象。为简化起见，我不会创建 `roasting` 或 `finance` schema，因为仅通过 `retail` schema 我们就可以涵盖各种类型的测试。

现在让我们创建 `retail` schema。`retail` schema 将保存与业务零售操作相关的所有数据库对象：

```
CREATE SCHEMA retail AUTHORIZATION retail_user;
```

接下来，我们将创建一个表来保存客户信息：

```
CREATE TABLE retail.customer (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    full_name STRING(255) NOT NULL,
    email STRING(320) NOT NULL,
    join_date TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

关键设计决策：

-   `客户姓名` – 并非每个人都有名和姓，因此与其要求分别提供名和姓，不如考虑直接要求提供客户的全名。
-   `join_date` – 处理时区时值得问的一个问题是，是否需要按用户时间还是系统时间捕获时间戳。如果是用户时间，使用 `TIMESTAMPTZ` 数据类型是有意义的，因为这允许我们提供特定用户时区的时间。

接下来，我们将创建一个表来保存产品信息：

```
CREATE TABLE retail.product (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING(100) NOT NULL,
    sku STRING(16) NOT NULL
);
```

关键设计决策：

-   `用户体验 (UX)` – 与订单表一样，考虑公司内部用户的体验很重要。如果使用库存单位 (SKU) 来标识产品，请确保它们易于使用和记忆。

我们不会将价格与产品存储在一起，而是创建一种允许产品价格随时间变化的方法。例如，客户可能在促销期间购买产品。如果他们稍后要求退款，企业不能因按非促销价退款而亏本。

以下表格是历史价格的基本实现：

```
CREATE TABLE retail.product_price (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES retail.product(id),
    amount DECIMAL NOT NULL,
    start_date TIMESTAMPTZ NOT NULL,
    end_date TIMESTAMPTZ
);
```

关键设计决策：

-   `简单性` – 为保持简单明了，企业决定只处理一种货币。因此，我目前不会明确存储货币。
-   `正确性` – 对于应使用小数还是整数来存储货币价值，存在不同的观点。如果使用浮点数进行计算，可能会出现舍入错误，因此有些人更喜欢使用整数，并且仅在代码中临时转换为小数进行计算。这样，他们可以控制小数的舍入方式。由于我不会执行任何计算，我决定保持简单，使用小数。
-   `灵活性` – 通过将产品价格存储在产品表之外，我们在定价方面有了灵活性。通过包含价格的 `start_date` 和 `end_date`，我们可以安排价格变更，而无需手动更新它们。

接下来，我们将创建一个表来保存客户订单信息：

```
CREATE TABLE retail.order (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reference STRING(16) NOT NULL,
    customer_id UUID NOT NULL REFERENCES retail.customer(id),
    delivery_instructions STRING(255),
    order_date TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

关键设计决策：

-   `用户体验 (UX)` – 考虑一下如果用户看到 UUID 订单号时的体验。如果他们在联系客服时需要输入或说出一个 UUID，这可能会损害他们的体验。因此，请为他们创建简短、易于阅读的标识符。

每个订单可以包含多个产品，每个产品可以出现在多个订单中。我们将通过第三张表来解决这个多对多关系：

```
CREATE TABLE retail.product_order (
    order_id UUID NOT NULL REFERENCES retail.order(id),
    product_id UUID NOT NULL REFERENCES retail.product(id),
    product_price_id UUID NOT NULL REFERENCES retail.product_price(id)
);
```

在与财务部门沟通后，我们了解到他们团队的成员需要能够访问 `retail` schema 中的对象，以便生成日终报告等。目前，如果他们尝试访问 `retail` schema 的对象，他们将看不到任何内容：

```
$ cockroach sql \
--certs-dir=certs \
--url "postgres://finance_user:a02308ce58c92131@localhost:26257/bean_about_town" \
--execute "SELECT COUNT(*) FROM retail.order"

ERROR: user finance_user does not have SELECT privilege on relation order
```

授予他们对 `retail` schema 对象的 `USAGE` 和 `SELECT` 权限将解决此问题：

```
GRANT SELECT, INSERT, UPDATE, DELETE ON retail.* TO retail_user;
GRANT SELECT ON retail.* TO retail_user;
GRANT USAGE ON SCHEMA retail TO finance_user;
GRANT SELECT ON retail.* TO finance_user;
```

### 功能测试

功能测试验证数据库是否满足业务需求和用例。我们可以从用户角度（黑盒）或凭借对数据库的深入理解和直接访问（白盒）运行功能测试。我们现在将逐一介绍每种测试类型。

为避免复杂性失控，我将测试范围限制在覆盖 `retail` schema。

#### 黑盒测试



黑盒测试从用户的角度测试数据库。我们不对其内部结构做任何假设，因此测试可以通过用户界面或 API 进行。由于我们没有实际的数据库或用户界面，我将提供一个简单的 API 来抽象数据库操作。为了简洁起见，我将使用来自[第 5 章](https://doi.org/10.1007/978-1-4842-82243_5)的 Crystal 驱动。

在开始之前，我们先看看这个测试需要实现的用例：

*   `创建客户` – 向数据库中插入一个新客户并返回其 ID。
*   `获取客户` – 根据 ID 从数据库中选择一位客户。
*   `创建带价格的产品` – 插入一个产品及其相关的一组价格。
*   `获取产品` – 选择数据库中所有产品的列表。
*   `下订单` – 为客户插入一份包含产品列表的订单。

#### 创建一个简单的应用程序

首先，使用以下命令为应用程序创建脚手架：

```
crystal init app api
cd api
```

接下来，通过将它们添加到 `shard.yml` 文件并安装，引入我们将需要的依赖项：

```
dependencies:
  kemal:
    github: kemalcr/kemal
  pg:
    github: will/crystal-pg
shards install
```

依赖项安装好后，我们就可以编写 API 了。与[第 5 章](https://doi.org/10.1007/978-1-4842-82243_5)中的示例代码一样，这段代码远未达到生产质量，仅作为黑盒测试的一种机制提供。

让我们从导入一些模块开始：

```
require "kemal"
require "db"
require "pg"
require "json"
require "uuid"
require "uuid/json"
```

接下来，我们将创建一个到数据库的连接。请注意，此连接将是明文的，这显然不适合生产环境使用：

```
db = PG.connect "postgres://retail_user:7efd7222dfdfb107@localhost:26257/bean_about_town?auth_methods=cleartext"
```

现在我们准备好连接我们的 API 处理器了。让我们从一个过滤器中间件处理器开始，它将为所有响应附加一个 JSON `Content-Type` 头。这将在我们所有处理器执行之前运行，并为所有响应添加正确的 `Content-Type` 头：

```
before_all do |env|
  env.response.content_type = "application/json"
end
```

接下来，我们将添加一个 `POST` 请求处理器来插入客户。这个处理器将

*   从 JSON 请求体中读取 `full_name` 和 `email` 字符串值
*   将它们插入到 `retail.customer` 表中
*   返回客户的 ID 和 `join_date`

```
post "/customers" do |env|
  full_name = env.params.json["full_name"].as(String)
  email = env.params.json["email"].as(String)
  id, join_date = db.query_one "INSERT INTO retail.customer (full_name, email) VALUES ($1, $2) RETURNING id, join_date", full_name, email, as: { UUID, Time }
  { "id": id, "join_date": join_date }.to_json
end
```

接下来，我们将添加一个 `GET` 请求处理器来按 ID 选择客户。这个处理器将

*   从请求 URL 路径中读取一个 ID
*   从数据库中选择一个具有匹配 ID 的客户
*   返回一个表示该用户的 JSON 对象

```
get "/customers/:id" do |env|
  id = env.params.url["id"]
  full_name, email, join_date = db.query_one "SELECT full_name, email, join_date FROM retail.customer WHERE id = $1 LIMIT 1", id, as: { String, String, Time }
  { "id": id, "full_name": full_name, "email": email, "join_date": join_date }.to_json
end
```

现在我们将处理产品。让我们添加一个 `POST` 请求端点来插入产品及其价格。这个处理器将

*   从请求体中解析一个产品对象
*   开启一个事务，该事务将
    *   将一个产品插入到 `product` 表
    *   将该产品的价格插入到 `produce_price` 表
*   提交事务
*   返回产品的 ID

由于我们是从请求中读取一个对象，我创建了两个类：一个 `Product` 类用于保存产品的顶层信息，以及一个 `ProductPrice` 类用于包含该产品的单个、可选择是否过期的价格。

```
class Product
  include JSON::Serializable
  def initialize(@name : String, @sku : String)
    @prices = Array(ProductPrice).new
  end
```



