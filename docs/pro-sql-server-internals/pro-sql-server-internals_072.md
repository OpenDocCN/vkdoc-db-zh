# 第 25 章 ■ 查询优化与执行



## 合并连接

合并连接是另一个很好的范例。虽然它在已排序的输入上比嵌套循环更高效，但人们很容易忽略为合并连接准备输入的 `Sort` 操作的开销。

通常，你应该在分析中牢记连接行为以及每种连接类型的优缺点。

## 聚合

聚合对一组值执行计算并返回单个值。SQL 中聚合的一个典型例子是 `MIN()` 函数，它返回其处理的一组值中的最小值。

SQL Server 支持两种类型的聚合运算符：`stream`（流）聚合和 `hash`（哈希）聚合。

### 流聚合

`stream aggregate` 基于已排序的输入执行聚合；例如，当数据按 `group by` 子句中指定的列排序时。清单 25-10 展示了流聚合算法。

***清单 25-10.*** 流聚合算法

```
/* Pre-requirement: input is sorted */

for each row R1 from input

begin

    if R1 does not match current group by criteria

    begin

        return current aggregate results (if any)

        clear current aggregate results

        set current group criteria to match R1

    end

    update aggregate results with R1 data

end

return current aggregate results (if any)
```

由于需要已排序的输入，SQL Server 经常将流聚合与 `Sort` 运算符一起使用。让我们看一个例子，创建一个包含公司销售信息的表格。之后，运行一个查询来计算每个客户的总销售额。执行此操作的代码如清单 25-11 所示。

***清单 25-11.*** 使用流聚合的查询

```
create table dbo.Orders

(

    OrderID int not null,

    CustomerId int not null,

    Total money not null,

    constraint PK_Orders primary key clustered(OrderID)

);

;with N1(C) as (select 0 union all select 0) -- 2 rows

,N2(C) as (select 0 from N1 as T1 cross join N1 as T2) -- 4 rows

,N3(C) as (select 0 from N2 as T1 cross join N2 as T2) -- 16 rows

,N4(C) as (select 0 from N3 as T1 cross join N3 as T2) -- 256 rows

,Nums(Num) as (select row_number() over (order by (select null)) from N4)

insert into dbo.Orders(OrderId, CustomerId, Total)

select Num, Num % 10 + 1, Num from Nums;

select Customerid, sum(Total) as [Total Sales]

from dbo.Orders

group by CustomerId;
```

你可以在图 25-10 中看到查询的执行计划。由于`CustomerId` 列上没有索引，SQL Server 需要添加一个 `Sort` 运算符来为 `Stream Aggregate` 运算符保证已排序的输入。

***图 25-10.** 使用流聚合的查询执行计划*

### 哈希聚合

`hash aggregate` 与哈希连接非常相似。它针对大型输入，并需要内存来存储哈希表。哈希聚合算法如清单 25-12 所示。

***清单 25-12.*** 哈希聚合算法

```
for each row R1 from input

begin

    calculate hash value of R1 group columns

    check for a matching row in hash table

    if matching row exists

        update aggregate results of matching row

    else

        insert new row into hash table

end

return all rows from hash table with aggregate results
```

类似于哈希连接，哈希表可能溢出到 `tempdb`，这对聚合的性能有负面影响。

### 聚合比较

与连接一样，流聚合和哈希聚合针对不同的用例。流聚合在输入已排序时效果最好——无论是由于现有索引，还是数据量小易于排序。另一方面，哈希聚合针对大型、未排序的输入。

表 25-3 比较了哈希聚合和流聚合。

***表 25-3.** 聚合比较*

| | **流聚合** | **哈希聚合** |
| :--- | :--- | :--- |
| **最佳用例** | 输入规模小，数据可排序 | 输入规模中到大，且未排序 |



