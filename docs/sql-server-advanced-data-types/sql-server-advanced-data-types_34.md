# 第三章 使用 T-SQL 构建 XML

```xml
<LineItem Quantity="35" UnitPrice="1.89">
    <Product StockItemName="Packing knife with metal insert blade (Yellow) 9mm" />
</LineItem>

<LineItem Quantity="150" UnitPrice="2.74">
    <Product StockItemName="Shipping carton (Brown) 480x270x320mm" />
</LineItem>
</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">
    <LineItem Quantity="175" UnitPrice="1.14">
        <Product StockItemName="Shipping carton (Brown) 356x229x229mm" />
    </LineItem>
</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72787">
    <LineItem Quantity="4" UnitPrice="345.00">
        <Product StockItemName="Ride on big wheel monster truck (Black) 1/12 scale" />
    </LineItem>
</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">
    <LineItem Quantity="70" UnitPrice="26.00">
        <Product StockItemName="10 mm Anti static bubble wrap (Blue) 10m" />
    </LineItem>
</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73340">
    <LineItem Quantity="20" UnitPrice="112.00">
        <Product StockItemName="32 mm Double sided bubble wrap 50m" />
    </LineItem>
</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">
    <LineItem Quantity="20" UnitPrice="18.50">
        <Product StockItemName="Office cube periscope (Black)" />
    </LineItem>
</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73356">
    <LineItem Quantity="10" UnitPrice="285.00">
        <Product StockItemName="Ride on vintage American toy coupe (Black) 1/12 scale" />
    </LineItem>
</SalesOrder>
</Customers>
```

基于表连接的嵌套并不总是足够的。在此示例中，某些 `<LineItem>` 元素可能包含多个 `<Product>` 元素，这显然是不正确的。销售订单上的每个行项目不可能对应多个产品。这是因为查询中没有包含 `Sales.OrderLines` 表的主键，意味着从 `Sales.OrderLines` 表返回的元组集合不一定都是唯一的。`UnitPrice` 和 `OrderQty` 的值恰好是重复的。如果发生这种情况，`FOR XML AUTO` 会对它们进行分组。如果包含了主键，这个问题就不会发生。

## `使用 FOR XML PATH`

有时，你需要比 `RAW` 模式或 `AUTO` 模式提供的对结果 XML 文档形状的**更多控制**。当你需要定义自定义输出时，可以使用 `PATH` 模式。

`PATH` 模式提供了极大的灵活性，因为它允许你定义结果 XML 中每个节点的位置。这是通过指定查询中的每个列如何映射到 XML 来实现的，使用列名或别名。

如果列别名以 `@` 符号开头，将创建一个属性。如果不使用 `@` 符号，该列将映射为一个元素。将成为属性的列必须在定义为元素的同级节点的列之前指定。

如果你想定义节点在层次结构中的位置，可以使用 `/` 符号。例如，如果你希望订单日期嵌套在一个名为 `<OrderHeader>` 的元素下，你可以为 `OrderDate` 列指定其列别名为 `'/OrderHeader/OrderDate'`。

`PATH` 模式允许你创建高度定制和复杂的结构。例如，假设你需要创建一个 XML 文档，其格式如 `Listing 3-14` 所示（参见 `#p105`）。这里，你会注意到有一个名为 `<SalesOrders>` 的根节点。层次结构中的下一个节点是 `<Orders>`。这是一个可重复元素，客户每下一个订单就会出现一次，这与我们探索 `AUTO` 模式时的情况相同。不同之处在于，每个销售订单都有其自身的层次结构。首先，有一个通用订单信息部分，存储在一个名为 `<OrderHeader>` 的节点中。该元素是 `<CustomerName>`、`<OrderDate>` 和 `<OrderID>` 的父元素。这些值将存储为简单元素。



还有一个用于存储订单行项目详情的部分，这些信息保存在一个名为`<OrderDetails>`的节点中。此元素包含一个名为`<Product>`的重复子元素。这个重复元素是每个订单内`<ProductName>`(`StockItemName`)和`<ProductID>`(`StockItemID`)的父节点，同时也包含每件商品的`<Price>`(`unitprice`)和`<Qty>`。
这些值都作为`<Product>`元素的属性存储。

### 清单 3-14. XML 输出的所需格式
```xml
<SalesOrders>
  <Order>
    <OrderHeader>
      <CustomerName>Agrita Abele</CustomerName>
      <OrderDate>2016-05-18</OrderDate>
      <OrderID>72637</OrderID>
    </OrderHeader>
    <OrderDetails>
      <Product ProductID="96" ProductName="&quot;The Gu&quot; red shirt XML tag t-shirt (Black) XXL" Price="18.00" Qty="24" />
      <Product ProductID="107" ProductName="Superhero action jacket (Blue) 3XS" Price="25.00" Qty="1" />
      <Product ProductID="138" ProductName="Furry animal socks (Pink) S" Price="5.00" Qty="96" />
    </OrderDetails>
  </Order>
  <Order>
    <OrderHeader>
      <CustomerName>Agrita Abele</CustomerName>
      <OrderDate>2016-05-18</OrderDate>
      <OrderID>72669</OrderID>
    </OrderHeader>
    <OrderDetails>
      <Product ProductID="147" ProductName="Halloween skull mask (Gray) M" Price="18.00" Qty="60" />
      <Product ProductID="206" ProductName="Permanent marker black 5mm nib (Black) 5mm" Price="2.70" Qty="84" />
      <Product ProductID="165" ProductName="10 mm Anti static bubble wrap (Blue) 10m" Price="26.00" Qty="70" />
      <Product ProductID="3" ProductName="Office cube periscope (Black)" Price="18.50" Qty="20" />
    </OrderDetails>
  </Order>
  <Order>
    <OrderHeader>
      <CustomerName>Agrita Abele</CustomerName>
      <OrderDate>2016-05-18</OrderDate>
      <OrderID>72671</OrderID>
    </OrderHeader>
    <OrderDetails>
      <Product ProductID="105" ProductName="Alien officer hoodie (Black) 4XL" Price="35.00" Qty="6" />
      <Product ProductID="209" ProductName="Packing knife with metal insert blade (Yellow) 9mm" Price="1.89" Qty="35" />
      <Product ProductID="183" ProductName="Shipping carton (Brown) 480x270x320mm" Price="2.74" Qty="150" />
    </OrderDetails>
  </Order>
  <Order>
    <OrderHeader>
      <CustomerName>Agrita Abele</CustomerName>
      <OrderDate>2016-05-18</OrderDate>
      <OrderID>72713</OrderID>
    </OrderHeader>
    <OrderDetails>
      <Product ProductID="204" ProductName="Tape dispenser (Red)" Price="32.00" Qty="20" />
    </OrderDetails>
  </Order>
  <Order>
    <OrderHeader>
      <CustomerName>Agrita Abele</CustomerName>
      <OrderDate>2016-05-19</OrderDate>
      <OrderID>72770</OrderID>
    </OrderHeader>
    <OrderDetails>
      <Product ProductID="99" ProductName="&quot;The Gu&quot; red shirt XML tag t-shirt (Black) 5XL" Price="18.00" Qty="120" />
      <Product ProductID="110" ProductName="Superhero action jacket (Blue) S" Price="25.00" Qty="8" />
      <Product ProductID="196" ProductName="Black and orange handle with care despatch tape 48mmx100m" Price="4.10" Qty="96" />
      <Product ProductID="181" ProductName="Shipping carton (Brown) 356x229x229mm" Price="1.14" Qty="175" />
    </OrderDetails>
  </Order>
  <Order>
    <OrderHeader>
      <CustomerName>Agrita Abele</CustomerName>
      <OrderDate>2016-05-19</OrderDate>
      <OrderID>72787</OrderID>
    </OrderHeader>
    <OrderDetails>
      <Product ProductID="37" ProductName="Developer joke mug - when your hammer is C++ (Black)" Price="13.00" Qty="2" />
      <Product ProductID="36" ProductName="Developer joke mug - when your hammer is C++ (White)" Price="13.00" Qty="9" />
      <Product ProductID="133" ProductName="Furry gorilla with big eyes slippers (Black) XL" Price="32.00" Qty="9" />
      <Product ProductID="220" ProductName="Novelty chilli chocolates 250g" Price="8.55" Qty="240" />
      <Product ProductID="75" ProductName="Ride on big wheel monster truck (Black) 1/12 scale" Price="345.00" Qty="4" />
    </OrderDetails>
  </Order>
  <Order>
    <OrderHeader>
      <CustomerName>Agrita Abele</CustomerName>
      <OrderDate>2016-05-27</OrderDate>
      <OrderID>73340</OrderID>
    </OrderHeader>
    <OrderDetails>
      <Product ProductID="117" ProductName="Superhero action jacket (Blue) 5XL" Price="34.00" Qty="2" />
    </OrderDetails>
  </Order>
</SalesOrders>
```

## 第 3 章 使用 T-SQL 构建 XML



