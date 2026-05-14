# 直方图类型比较与查询优化

`SQL> EXPLAIN PLAN SET STATEMENT_ID '101' FOR SELECT * FROM t WHERE val2 = 101;`
`SQL> EXPLAIN PLAN SET STATEMENT_ID '102' FOR SELECT * FROM t WHERE val2 = 102;`
`SQL> EXPLAIN PLAN SET STATEMENT_ID '103' FOR SELECT * FROM t WHERE val2 = 103;`
`SQL> EXPLAIN PLAN SET STATEMENT_ID '104' FOR SELECT * FROM t WHERE val2 = 104;`
`SQL> EXPLAIN PLAN SET STATEMENT_ID '105' FOR SELECT * FROM t WHERE val2 = 105;`
`SQL> EXPLAIN PLAN SET STATEMENT_ID '106' FOR SELECT * FROM t WHERE val2 = 106;`

```
SQL> SELECT statement_id, cardinality
  2 FROM plan_table
  3 WHERE id = 0;

STATEMENT_ID CARDINALITY
------------ -----------
101                    50
102                    50
103                    50
104                    50
105                   400
106                   300
```

考虑到这两种直方图类型的基本特性，显然**频率直方图**比**高度平衡直方图**更准确。高度平衡直方图的主要问题是，有时一个值是否被识别为常见值可能是偶然的。例如，在图 4-6 所示的直方图中，桶 4 和桶 5 之间的分点恰好位于值从 105 变为 106 的点附近。

因此，即使数据分布的微小变化也可能导致不同的直方图和不同的估计值。以下示例说明了这种情况：

`SQL> UPDATE t SET val2 = 105 WHERE val2 = 106 AND rownum <= 13;`

```
SQL> SELECT endpoint_value, endpoint_number
  2 FROM user_tab_histograms
  3 WHERE table_name = 'T'
  4 AND column_name = 'VAL2'
  5 ORDER BY endpoint_number;

ENDPOINT_VALUE ENDPOINT_NUMBER
-------------- ---------------
           101               0
           104               1
           105               4
           106               5
```

`SQL> EXPLAIN PLAN SET STATEMENT_ID '101' FOR SELECT * FROM t WHERE val2 = 101;`
`SQL> EXPLAIN PLAN SET STATEMENT_ID '102' FOR SELECT * FROM t WHERE val2 = 102;`
`SQL> EXPLAIN PLAN SET STATEMENT_ID '103' FOR SELECT * FROM t WHERE val2 = 103;`
`SQL> EXPLAIN PLAN SET STATEMENT_ID '104' FOR SELECT * FROM t WHERE val2 = 104;`
`SQL> EXPLAIN PLAN SET STATEMENT_ID '105' FOR SELECT * FROM t WHERE val2 = 105;`
`SQL> EXPLAIN PLAN SET STATEMENT_ID '106' FOR SELECT * FROM t WHERE val2 = 106;`

```
SQL> SELECT statement_id, cardinality
  2 FROM plan_table
  3 WHERE id = 0;

STATEMENT_ID CARDINALITY
------------ -----------
101                    80
102                    80
103                    80
104                    80
105                   600
106                    80
```

因此，在实践中，高度平衡直方图不仅可能产生误导，还可能导致查询优化器估计的不稳定。

同样值得注意的是，视图`user_tab_histograms`会为每个没有直方图的列显示两行。这是因为最小值和最大值分别存储在端点号 0 和 1 中。例如，没有直方图的列`id`的内容如下：

```
SQL> SELECT endpoint_value, endpoint_number
  2 FROM user_tab_histograms
  3 WHERE table_name = 'T'
  4 AND column_name = 'ID'
  5 ORDER BY endpoint_number;

ENDPOINT_VALUE ENDPOINT_NUMBER
--------------- ---------------
              1               0
           1000               1
```

