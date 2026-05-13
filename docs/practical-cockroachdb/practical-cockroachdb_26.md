# 第 8 章测试

我们将创建的最后一个处理器是一个 GET 请求处理器，用于根据给定的客户 ID 选择其订单。该处理器将
*   从请求 URL 路径读取客户 ID
*   使用客户 ID 选择其所有订单
*   遍历结果，构建包含产品的订单集合
*   返回订单数组

###### `Order` (扩展定义)

```crystal
class Order

include JSON::Serializable

def initialize(@id, @reference, @customer_id, @date, @delivery_

instructions)

@products = Hash(String, Float64).new

end

property id : UUID

property reference : String

property customer_id : UUID

property date : Time

property delivery_instructions : String

property products : Hash(String, Float64)

end
```

###### GET `/customers/:id/orders`

```crystal
get "/customers/:id/orders" do |env|

customer_id = UUID.new(env.params.url["id"])

orders = Hash(UUID, Order).new
```


## 第 8 章：测试

##### 启动 API 服务器

剩下的所有工作就是启动 API 服务器进行监听。下面的代码将：
- 在端口 `3000` 上启动 Kemal Web 服务器

```
Kemal.run
```

代码就绪后，我们可以使用以下命令启动服务器：

```
$ crystal run src/api.cr
[development] Kemal is ready to lead at http://0.0.0.0:3000
```

##### 测试应用程序

让我们编写一些 `curl` 命令来测试 API（并在此过程中测试数据库）。在这一测试阶段，我们不仅需要断言数据库在接收到预期数据时行为符合预期，还需要断言它在接收到*非预期*数据时行为也符合预期。这些通常被称为边界测试。

在本节中，我们将遍历每个用例，以断言数据库能够支持它们。让我们从创建一个客户开始：

```
$ curl -X POST 'localhost:3000/customers' \
    -H 'Content-Type: application/json' \
    -d '{
  "full_name": "Malala Yousafzai",
  "email": "malala.yousafzai@example.com"
}'
{"id":"7d640140-5581-41cf-a9db-0bccaf90f2de","join_date":"2022-01-20T18:26:18Z"}
```

这涵盖了快乐路径：一个合理长度的姓名和电子邮件地址。我们的数据库支持 Unicode 字符吗？让我们一探究竟：

```
$ curl -X POST 'localhost:3000/customers' \
    -H 'Content-Type: application/json' \
    -d '{
  "full_name": "Наде́жда Андре́евна Толоко́нникова",
  "email": "nadya.tolokno@example.com"
}'
{"id":"85f05ac3-931f-4417-a6ae-1403468a2c10","join_date":"2022-01-20T18:26:34Z"}
```

看起来支持！那长名字呢？数据库会拒绝超过我们定义的 `255` 个字符长度的 `full_name` 吗？

```
$ curl -X POST 'localhost:3000/customers' \
    -H 'Content-Type: application/json' \
    -d '{
  "full_name": "aaaaaaaaaabbbbbbbbbbccccccccccddddddddddeeeeeeeeeeaaaa aaaaaabbbbbbbbbbccccccccccddddddddddeeeeeeeeeeaaaaaaaaaabbbbbbbbbbcc ccccccccddddddddddeeeeeeeeeeaaaaaaaaaabbbbbbbbbbccccccccccdddddddddd eeeeeeeeaaaaaaaaaabbbbbbbbbbccccccccccddddddddddeeeeeeeeeefffffg",
  "email": "full_name@example.com"
}'
...ERROR OMITTED
```

服务器没有任何错误处理，但日志确认我们尝试存储一个 `full_name` 长度为 `256` 个字符的客户失败了：

```
Exception: value too long for type STRING(255) (PQ::PQError)
```

那么边界情况，一个 `255` 个字符长的 `full_name` 呢？

```
$ curl -X POST 'localhost:3000/customers' \
    -H 'Content-Type: application/json' \
    -d '{
  "full_name": "aaaaaaaaaabbbbbbbbbbccccccccccddddddddddeeeeeeeeeeaaaa aaaaaabbbbbbbbbbccccccccccddddddddddeeeeeeeeeeaaaaaaaaaabbbbbbbbbbcc ccccccccddddddddddeeeeeeeeeeaaaaaaaaaabbbbbbbbbbccccccccccdddddddddd eeeeeeeeaaaaaaaaaabbbbbbbbbbccccccccccddddddddddeeeeeeeeeefffff",
  "email": "full_name@example.com"
}'
{"id":"d93bedb4-3e59-4bd5-b018-b6d89d688904","join_date":"2022-01-20T18:26:53Z"}
```

暂时忽略电子邮件地址，我们的创建客户用例看起来不错，似乎我们的数据库正在满足这个用例的基本要求。现在让我们继续处理获取客户的用例。

首先，我们将发出一个请求来获取一个现有客户：

```
$ curl -X GET 'localhost:3000/customers/85f05ac3-931f-4417-a6ae-1403468a2c10'
{"id":"85f05ac3-931f-4417-a6ae-1403468a2c10","full_name":"Наде́жда Андре́евна Толоко́нникова","email":"nadya.tolokno@example.com","join_date":"2022-01-20T18:26:34Z"}
```

这返回了我们最初为该用户插入的数据。显然，关于谁被允许请求特定用户没有任何安全检查，但从数据库的角度来看，这是我们的快乐路径测试满足了。不快乐的路径呢？让我们尝试在 URL 路径中省略客户 ID、通过不存在的 ID 搜索客户，以及使用格式错误的 UUID 搜索客户。对于这些测试，我将响应限制为仅头部，因为没有适当的错误处理，我们将收到 HTML 页面响应。

```
curl -I -X GET localhost:3000/customers/
HTTP/1.1 404 Not Found
Connection: keep-alive
X-Powered-By: Kemal
Content-Type: text/html
Transfer-Encoding: chunked
```

在对路径中没有 ID 的请求的响应中，我们收到一个 `404`，因为路径期望的是 `/customers/{CUSTOMER_ID}`。此测试尚未触及数据库，这是预期的。

```
curl -I -X GET localhost:3000/customers/df24592d-62dd-4876-b54a-405be8614aaf
HTTP/1.1 500 Internal Server Error
Connection: keep-alive
X-Powered-By: Kemal
Content-Type: text/html
Transfer-Encoding: chunked
```

在对不存在的 ID 的请求的响应中，我们收到一个 `500`，原因是数据库调用周围的异常处理不足。检查服务器日志，我们可以看到该请求没有产生结果。再次，一个预期的结果：

```
Exception: no results (DB::NoResultsError)
```

```
curl -I -X GET localhost:3000/customers/a
HTTP/1.1 500 Internal Server Error
Connection: keep-alive
X-Powered-By: Kemal
Content-Type: text/html
Transfer-Encoding: chunked
```

在对格式错误的客户 ID 的请求的响应中，我们收到一个 `500`。再次，这是由于数据库调用周围的错误处理不足。再次检查服务器日志揭示了我们收到响应的根本原因。此请求也到达了数据库，但在查询准备期间返回了错误。再次，一个预期的结果：

```
Exception: error in argument for $1: could not parse string "a" as uuid (PQ::PQError)
```

接下来是产品的创建。这个场景提供了对多语句事务的第一次测试。在这个事务中，我们插入一个产品，然后使用先前插入的产品 ID 插入一些价格。让我们发出一个请求：

```
$ curl -X POST 'localhost:3000/products' \
    -d '{
  "name": "Where Have You Bean All My Life?",
  "sku": "L2JM9XAO",
  "prices": [
    { "amount": 6.00, "start_date": "2022-01-01T09:00:00Z", "end_date": "2022-01-01T10:00:00Z" },
    { "amount": 12.00, "start_date": "2022-01-01T09:00:00Z" }
  ]
}'
{"id":"d77bfea2-f0df-4da9-aeef-7c8d9c362e38"}
```

我们为我们刚刚插入的产品提供了两个价格：全价 `12.00` 和半价 `6.00`。半价优惠设置在一小时后过期，因此在测试时，我预计每个请求的此产品价格为 `12.00`。

我们还想断言，如果有多个活动价格，我们总是返回最便宜的那个：

```
$ curl -X POST 'localhost:3000/products' \
    -d '{
  "name": "Has Bean",
  "sku": "1OLEMX05",
  "prices": [
    { "amount": 6.00, "start_date": "2022-01-02T09:00:00Z", "end_date": "2022-01-02T10:00:00Z" },
    { "amount": 10.00, "start_date": "2022-01-01T09:00:00Z" },
    { "amount": 12.00, "start_date": "2022-01-01T09:00:00Z" }
  ]
}'
{"id":"386e841e-4da8-4688-82a4-71d6fbb369fa"}
```

与其针对此端点运行一套探索性测试，让我们直接跳到获取产品及其价格：

```
$ curl -X GET 'localhost:3000/products'
[
  {"id":"386e841e-4da8-4688-82a4-71d6fbb369fa","name":"Has Bean","sku":"1OLEMX05","prices":[{"id":"cc71e211-abc6-42d5-8f59-d357042334b0","amount":10.0}]},
  {"id":"d77bfea2-f0df-4da9-aeef-7c8d9c362e38","name":"Where Have You Bean All My Life?","sku":"L2JM9XAO","prices":[{"id":"cbdc9d9c-a987-4655-b3f8-6b9b2ec9c50e","amount":12.0}]}
]
```


