# 第 4 章 创建与衍生数据

订单模式的细分，似乎公司可以通过将客户 104 的部分或全部订单转移到互联网订单来节省成本。

最后，您可能无法预先知道透视标准，因为新产品可能每天都会出现，这使得查询的维护成为一个问题。一种可能的解决方案是使用 Java 或 PL/SQL 动态构建您的查询，尽管此解决方案可能存在潜在的安全性和性能问题。但是，如果您查询输出的消费者可以接受 XML 输出，您可以使用`PIVOT XML`子句来代替简单的`PIVOT`，并在`IN`子句中包含过滤条件，如下面这个修改了原始解决方案的示例：

```
with order_item_query as
(select customer_id, product_id, quantity
from orders join order_items using(order_id))
select * from order_item_query
pivot xml (
sum(quantity) as sum_qty
for (product_id) in (any)
)
order by customer_id
;

CUSTOMER_ID PRODUCT_ID_XML
---------------------- ------------------------------------------------------------
101 <PivotSet><item><column name =
"PRODUCT_ID">2264</column><column name = "SUM_QTY">29</column></item> . . .
102 <PivotSet><item><column name =
"PRODUCT_ID">2976</column><column name = "SUM_QTY">5</column></item> . . .
103 <PivotSet><item><column name =
"PRODUCT_ID">1910</column><column name = "SUM_QTY">6</column></item> . . .
. . .
170 <PivotSet><item><column name =
"PRODUCT_ID">3106</column><column name = "SUM_QTY">170</column></item> 47 rows selected
```

`ANY`关键字是`SELECT DISTINCT PRODUCT_ID FROM ORDER_ITEM_QUERY`的简写。除了`ANY`，您几乎可以放入任何返回您正在透视的域中值的 SQL 语句，如下例所示，您只想显示当前可订购产品的结果：

```
with order_item_query as
(select customer_id, product_id, quantity
from orders join order_items using(order_id))
select * from order_item_query
pivot xml (
sum(quantity) as sum_qty
for (product_id) in (select distinct product_id
from product_information
where product_status = 'orderable' )
)
order by customer_id
;
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

