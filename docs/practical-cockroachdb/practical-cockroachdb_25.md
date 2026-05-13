# 第 8 章测试

*   开启一个事务，该事务将
*   向`order`表插入一条订单
*   将产品和支付价格插入`product_order`表
*   关闭事务
*   返回订单 ID 和客户参考号

###### `Order`

```crystal
class Order

include JSON::Serializable

property id : UUID?

property customer_id : UUID

property delivery_instructions : String?

property products : Hash(UUID, UUID)

end
```

###### POST `/orders`

```crystal
post "/orders" do |env|

request = Order.from_json env.request.body.not_nil!

id = UUID.empty

ref = ('A'..'Z').to_a.shuffle(Random::Secure)[0, 8].join

db.transaction do |tx|

id = tx.connection.query_one "INSERT INTO retail.order (reference,

customer_id, delivery_instructions)

VALUES ($1, $2, $3)

RETURNING id", ref, request.customer_

id, request.delivery_instructions, as:

{ UUID }

request.products.each do |product_id, price_id|

tx.connection.exec "INSERT INTO retail.product_order (order_id,

product_id, product_price_id)

VALUES ($1, $2, $3)", id, product_id, price_id

end

end

{ "id": id }.to_json

end
```

