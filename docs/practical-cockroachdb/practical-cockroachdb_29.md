# 第 8 章 测试

好多了。需要注意的是，当对表进行写入时，会发生锁定，从而导致竞争。要查看数据库竞争发生的位置，您可以访问管理界面的 `/sql-activity?tab=Statements` 页面，或在 SQL shell 中运行以下命令。第一条命令高亮显示`表`的竞争情况，第二条命令高亮显示`索引`的竞争情况：

```sql
SELECT schema_name, table_name, num_contention_events FROM crdb_internal.cluster_contended_tables;
```

```
schema_name | table_name | num_contention_events
*--------------+---------------+------------------------*
retail       | order         | 22
public       | jobs          | 7
retail       | product_order | 6
```

```sql
SELECT schema_name, table_name, index_name, num_contention_events FROM crdb_internal.cluster_contended_indexes;
```

```
schema_name | table_name | index_name               | num_contention_events
*--------------+---------------+-----------------------+-----------------*
retail       | order         | order_customer_id_idx | 15
retail       | product_order | primary               | 6
```

根据这些信息，我们可以确定数据库中的竞争热点并进行处理。减少竞争的一种方法是将操作拆分为独立的语句。我们的 API 在同一个事务中执行每条创建产品和订单的语句，这会产生竞争，尤其是在许多语句执行时。在更具生产适用性的环境中，我倾向于使用多行 DML（数据操作语言）而非执行多条语句。考虑以下 `INSERT` 语句。第一条使用单行 DML 插入三条记录，而第二条利用了多行 DML。使用 DML 的好处是网络往返次数更少，SQL 语句解析更少，并且锁定行的时间更短，从而减少了表或索引上的竞争。

单行 DML:

```sql
INSERT INTO a_table (a_column) VALUES ('1');
INSERT INTO a_table (a_column) VALUES ('2');
INSERT INTO a_table (a_column) VALUES ('3');
```

多行 DML:

```sql
INSERT INTO a_table (a_column) VALUES ('1'), ('2'), ('3');
```

根据您使用的语言和驱动程序，您可能能够利用 `reWriteBatchedInserts` 功能，它会自动将单行 DML 查询转换为多行 DML 语句。

1 [www.cockroachlabs.com/blog/multi-row-dml](http://www.cockroachlabs.com/blog/multi-row-dml)
2 [www.cockroachlabs.com/docs/stable/build-a-java-app-with-cockroachdb.html](http://www.cockroachlabs.com/docs/stable/build-a-java-app-with-cockroachdb.html)

## 第 8 章 测试

在现实世界中，我会保留事务并利用多行 DML，以确保由产品和价格插入所产生的更改一起提交，如果发生任何错误，它们也会一起回滚，不会产生副作用或孤立行。

我们创建的 API 很小，只包含少量查询，使得这种白盒测试快速而轻松。如果您执行了成百上千条查询，却没有时间逐一检查，该怎么办？

CockroachDB 的 `SHOW FULL TABLE SCANS` 语句会返回所有



### 非功能性测试

非功能性测试旨在验证数据库是否满足各种*“性”*能要求：可靠性、可扩展性等。本节将介绍在集成 CockroachDB 时你可能遇到的一些最常见的非功能性测试类型。

#### 性能测试

要从性能测试中获得价值，我们首先必须了解应用程序将如何使用我们的数据库。哪些表会被最频繁使用？调用它们会产生什么影响？是否存在导致大量数据库调用的客户端交互？

让我们从了解客户端交互及其如何导致数据库调用开始本节内容。我们将通过按预期调用频率降序列出这些交互来完成：

• **获取所有产品** – 这将每次调用触发一个相当简单的 `SELECT` 语句。该语句可能会返回大量数据。

• **获取一个客户** – 这将每次调用触发一个简单的 `SELECT` 语句。此请求使用主键索引，因此本质上是优化的。

• **获取客户订单** – 这将触发一个使用了三个连接（join）的简单 `SELECT` 语句。

• **创建客户** – 这将为每个新客户触发一次简单的 `INSERT` 语句。

• **创建订单** – 与创建产品的请求类似，这将触发至少一个 `INSERT` 语句，以及针对所订购的每个产品的后续 `INSERT` 语句。所有这些都在一个简单的事务中完成。

• **创建产品** – 这将触发至少一个 `INSERT` 语句，以及针对每个产品价格的后续 `INSERT` 语句。所有这些都在一个简单的事务中完成。

我将使用 `k6`（一个用 Go 和 JavaScript 编写、可编写脚本的开源负载测试工具）来模拟这些事务。

为了简化，我只实现两个 `k6` 脚本。为了简洁起见，我不会花时间模拟准确的用户行为：

• **products.js** – 一个通过 API 将多个随机产品插入数据库的脚本

• **customers.js** – 一个模拟用户浏览产品、下订单然后检查其订单状态的脚本

首先，让我们创建 `products.js` 脚本来模拟用户创建产品。该脚本将

• 向 `/products` 端点发送一个 `POST` 请求，其 JSON 主体代表一个产品，包含随机名称、`SKU` 和一个有效的、随机价格

• 如果响应状态不是 `200`（成功），则判定失败

• 如果在运行完所有测试迭代后，错误率大于 1%，或 95% 百分位的请求持续时间大于 100 毫秒，则判定失败

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { randomString, randomIntBetween } from 'https://jslib.k6.io/k6-utils/1.1.0/index.js';

const BASE_URL = 'http://localhost:3000';

export const options = {
thresholds: {
http_req_failed: ['rate<0.01'],
http_req_duration: ['p(95)<100'],
}
};

export default() => {
const createResponse = http.post(
`${BASE_URL}/products`,
JSON.stringify({
name: 'Name ' + randomString(16),
sku: 'SKU ' + randomString(10),
prices: [
{
amount: randomIntBetween(1, 100),
start_date: addDays(new Date(), -10),
end_date: addDays(new Date(), 10),
}
]
}),
{ headers: {'Content-Type': 'application/json'} }
);

check(createResponse, { 'create product response': (r) => r.status === 200 });
};

function addDays(date, days) {
date.setDate(date.getDate() + days);
return date;
}
```

现在，让我们创建 `customers.js` 脚本来模拟一个客户通过 API 使用数据库。该脚本将

• 向 `/customers` 端点发送一个 `POST` 请求，其 JSON 主体代表一个客户，包含随机的 `full_name` 和 `email`

• 如果创建客户的响应状态不是 `200`（成功），则判定失败

• 从响应体 `JSON` 中捕获客户 ID 并传递它



