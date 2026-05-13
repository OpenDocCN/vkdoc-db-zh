# 第 8 章测试

##### 类定义

###### `Product`

```crystal
property id : UUID?

property name : String

property sku : String

property prices : Array(ProductPrice)

end
```

###### `ProductPrice`

```crystal
class ProductPrice

include JSON::Serializable

def initialize(@id : UUID, @amount : Float64)

end

property id : UUID?

property amount : Float64

property start_date : Time?

property end_date : Time?

end
```

##### 处理器实现

###### POST `/products`

```crystal
post "/products" do |env|

request = Product.from_json env.request.body.not_nil!

id = UUID.empty

db.transaction do |tx|

id = tx.connection.query_one "INSERT INTO retail.product (name, sku)

VALUES ($1, $2)

RETURNING id", request.name, request.sku,

as: { UUID }

request.prices.each do |price|

tx.connection.exec "INSERT INTO retail.product_price (product_id,

amount, start_date, end_date)

VALUES ($1, $2, $3, $4)", id, price.amount,

price.start_date, price.end_date

end

end

{ "id": id }.to_json

end
```
与客户处理器类似，我们也将为产品添加一个 GET 请求处理器。该处理器将
*   为每个产品选择最低的生效价格。生效价格定义为：`start_date`不在未来，并且`end_date`在未来或为空的最低价格。
*   构建找到的产品集合。
*   将产品作为 JSON 数组返回。

