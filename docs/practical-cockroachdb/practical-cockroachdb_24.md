# 第 8 章测试

###### GET `/products`

```crystal
get "/products" do |env|

products = Array(Product).new

db.query "WITH min_product_price AS (

SELECT DISTINCT ON(product_id) product_id, id, min(amount)

AS amount

FROM retail.product_price

WHERE start_date <= now()

AND (end_date >= now() OR end_date IS NULL)

GROUP BY id

ORDER BY product_id, amount

)

SELECT p.id, p.name, p.sku, pp.id, pp.amount FROM min_product_

price pp

JOIN retail.product p ON pp.product_id = p.id" do |rs|

rs.each do

product_id, name, sku, price_id, amount = rs.read(UUID, String,

String, UUID, PG::Numeric)

product = Product.new(name, sku)

product.id = product_id

product.prices << ProductPrice.new(price_id, amount.to_f)

products << product

end

end

products.to_json

end
```
接下来我们将创建的处理器是一个用于插入订单的 POST 请求处理器。与产品 POST 处理器类似，我们将从调用方捕获 JSON 请求，因此我们将引入一个新类来处理序列化。该处理器将
*   从请求体解析订单对象
*   生成一个简短的客户参考号

