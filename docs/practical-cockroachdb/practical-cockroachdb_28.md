# 第 8 章 测试

'2022-01-24 19:14:54.499657+00:00') OR (`end_date` IS NULL))

```
│
└── • scan
    estimated row count: 5 (100% of the table; stats collected 33 minutes ago)
    table: product_price@primary
    spans: FULL SCAN
```

哎呀！从`FULL SCAN`来看，很明显我们缺少一些索引。CockroachDB 将`start_date <= filter`、`end_date >= filter`和`end_date IS NULL`这几个过滤条件标识为可能导致此问题的原因。如果不修复，随着表的增长，该查询的性能将继续下降。

让我们为这些列添加一些索引，以使我们的查询更高效。

```sql
CREATE INDEX start_date_asc_idx ON retail.product_price(start_date ASC);
CREATE INDEX end_date_desc_idx ON retail.product_price(end_date DESC);
```

由于`start_date`过滤器检查的是`start_date`列值是否小于或等于给定值，我指定了默认的`ASC`排序顺序（这可以省略，但我保留它以便更清晰）。由于`end_date`过滤器检查的是`end_date`列值是否大于或等于给定值，我指定了`DESC`排序顺序。

让我们在应用了这些索引后再次尝试该查询：

```sql
EXPLAIN WITH min_product_price AS (
  SELECT DISTINCT ON(product_id) product_id, id, min(amount) AS amount
  FROM retail.product_price
  WHERE start_date <= now()
    AND (end_date >= now() OR end_date IS NULL)
  GROUP BY id
  ORDER BY product_id, amount
)
SELECT p.id, p.name, p.sku, pp.id price_id, pp.amount
FROM min_product_price pp
JOIN retail.product p ON pp.product_id = p.id;
```

```
info
distribution: full
vectorized: true

• lookup join
│ estimated row count: 2
│ table: product@primary
│ equality: (product_id) = (id)
│ equality cols are key
│
└── • render
   │ estimated row count: 3
   │
   └── • distinct
      │ estimated row count: 3
      │ distinct on: any_not_null
      │
      └── • sort
         │ estimated row count: 3
         │ order: +min
         │
         └── • group
            │ estimated row count: 3
            │ group by: id
            │ ordered: +id
            │
            └── • filter
               │ estimated row count: 3
               │ filter: start_date <= '2022-01-24 19:47:57.059747+00:00'
               │
               └── • index join
                  │ estimated row count: 0
                  │ table: product_price@primary
                  │
                  └── • sort
                     │ estimated row count: 0
                     │ order: +id
                     │
                     └── • scan

estimated row count: 0 (<0.01% of the table; stats collected 16 minutes ago)
table: product_price@end_date_desc_idx
spans: [ - /'2022-01-24 19:47:57.059747+00:00'] [/NULL - /NULL]
```

好多了！我们继续。

```sql
EXPLAIN SELECT o.id, o.reference, o.delivery_instructions, order_date,
       p.name, pp.amount
FROM retail.order o
JOIN retail.product_order po ON o.id = po.order_id
JOIN retail.product_price pp on po.product_price_id = pp.id
JOIN retail.product p ON po.product_id = p.id
WHERE o.customer_id = '85f05ac3-931f-4417-a6ae-1403468a2c10';
```

```
info
distribution: full
vectorized: true

• lookup join
│ estimated row count: 2
│ table: product@primary
│ equality: (product_id) = (id)
│ equality cols are key
│
└── • lookup join
   │ estimated row count: 2
   │ table: product_price@primary
   │ equality: (product_price_id) = (id)
   │ equality cols are key
   │
   └── • lookup join
      │ estimated row count: 2
      │ table: order@primary
      │ equality: (order_id) = (id)
      │ equality cols are key
      │ pred: customer_id = '85f05ac3-931f-4417-a6ae-1403468a2c10'
      │
      └── • scan
         estimated row count: 2 (100% of the table; stats collected 1 hour ago)
         table: product_order@primary
         spans: FULL SCAN
```

又是一个全表扫描！然而，造成这次全表扫描的原因是多表连接时缺少索引，而不是单个表缺少索引。让我们为参与连接的列添加索引，并重新运行 explain：

```sql
CREATE INDEX order_customer_id_idx ON retail.order(customer_id);
CREATE INDEX product_order_order_id_idx ON retail.product_order(order_id);
CREATE INDEX product_price_product_id_idx ON retail.product_price(product_id);
```

```sql
EXPLAIN SELECT o.id, o.reference, o.delivery_instructions, order_date,
       p.name, pp.amount
FROM retail.order o
JOIN retail.product_order po ON o.id = po.order_id
JOIN retail.product_price pp on po.product_price_id = pp.id
JOIN retail.product p ON po.product_id = p.id;
```



WHERE `o`.`customer_id` = '85f05ac3-931f-4417-a6ae-1403468a2c10';

info

distribution: `local`
vectorized: `true`

```
• 查找连接
│ 估计行数: 1
│ 表: `product_price@primary`
│ 等值条件: (`product_price_id`) = (`id`)
│ 等值条件列是键
│
└── • 查找连接
    │ 估计行数: 0
    │ 表: `product@primary`
    │ 等值条件: (`product_id`) = (`id`)
    │ 等值条件列是键
    │
    └── • 查找连接
        │ 表: `product_order@primary`
        │ 等值条件: (`rowid`) = (`rowid`)
        │ 等值条件列是键
        │
        └── • 查找连接
            │ 估计行数: 0
            │ 表: `product_order@product_order_order_id_idx`
            │ 等值条件: (`id`) = (`order_id`)
            │
            └── • 索引连接
                │ 估计行数: 0
                │ 表: `order@primary`
                │
                └── • 扫描
                    估计行数: 0 (小于表的 0.01%；统计数据收集于 4 分钟前)
                    表: `order@order_customer_id_idx`
                    范围: [/'00088f1e-4f35-4629-a07c-5b9fc0c2bac7' - /'00088f1e-4f35-4629-a07c-5b9fc0c2bac7']
```

