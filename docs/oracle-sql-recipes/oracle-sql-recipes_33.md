# 第 15 章 ■ 分区

## 15-7. 基于引用约束进行分区

### 问题描述

你有一个父表 `ORDERS` 和一个子表 `ORDER_ITEMS`，它们通过 `ORDER_ID` 列上的主键和外键约束相关联。父表 `ORDERS` 在 `ORDER_DATE` 列上进行了分区。尽管子表 `ORDER_ITEMS` 不包含 `ORDER_DATE` 列，但你是否可以将其分区，使其记录以与父表 `ORDERS` 相同的方式分布？

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 解决方案

如果你使用的是 Oracle Database 11 *g* 或更高版本，可以使用 `PARTITION BY REFERENCE` 子句来指定子表应与其父表以相同的方式进行分区。以下示例创建了一个在 `ORDER_ID` 上有主键约束、在 `ORDER_DATE` 上有范围分区的父表：

```sql
create table orders(
    order_id number
    ,order_date date
    ,constraint order_pk primary key(order_id)
)
partition by range(order_date)
(
    partition p10 values less than (to_date('01-jan-2010','dd-mon-yyyy'))
    ,partition p11 values less than (to_date('01-jan-2011','dd-mon-yyyy'))
    ,partition pmax values less than (maxvalue)
);
```

接着创建子表 `ORDER_ITEMS`。它通过将外键约束命名为引用对象来进行分区：

```sql
create table order_items(
    line_id number
    ,order_id number not null
    ,sku number
    ,quantity number
    ,constraint order_items_pk primary key(line_id, order_id)
    ,constraint order_items_fk1 foreign key (order_id) references orders
)
partition by reference (order_items_fk1);
```

请注意，外键列 `ORDER_ID` 必须定义为 `NOT NULL`。外键列必须被启用并强制执行。

### 工作原理

从 Oracle Database 11 *g* 开始，你可以通过引用进行分区。这允许子表继承其父表的分区策略。任何父表的分区维护操作也会应用于子记录表。

在引入引用分区功能之前，你必须在子表中物理复制并维护父表列，这不仅需要更多的磁盘空间，而且在维护分区时也容易出错。

在创建引用分区的子表时，如果你没有显式命名子表分区，默认情况下，Oracle 将使用与父表相同的分区名称为子表创建分区。以下示例显式命名了子表的引用分区：

```sql
create table order_items(
    line_id number
    ,order_id number not null
    ,sku number
    ,quantity number
    ,constraint order_items_pk primary key(line_id, order_id)
    ,constraint order_items_fk1 foreign key (order_id) references orders
)
partition by reference (order_items_fk1)
(
    partition c10
    ,partition c11
    ,partition cmax
);
```

你不能指定引用表的分区范围。除非你为子分区指定表空间，否则引用表的分区将在与父分区相同的表空间中创建。

[www.it-ebooks.info](http://www.it-ebooks.info/)

---

**注意**  
在间隔分区中，你只能从表中指定单个键列，并且该列必须是 `DATE` 或 `NUMBER` 数据类型。

---

### 间隔分区（Interval Partitioning）

分区示例数据：
```text
F_SALES P1 1 TO_DATE(' 2010-01-01 00:00:00',...
F_SALES SYS_P93 2 TO_DATE(' 2013-01-01 00:00:00',...
```
自动创建了一个名为 `SYS_P93` 的分区，其上限值为 `2013-01-01`。

### 工作原理（间隔分区）

从 Oracle Database 11 *g* 开始，你可以指示 Oracle 自动向范围分区表添加分区。间隔分区指示 Oracle 在插入的数据超出范围分区表的最大界限时动态创建一个新分区。新添加的分区基于你指定的间隔。

## 15-8. 在虚拟列上进行分区

### 问题描述

你已在表中定义了一个虚拟列。你想基于该虚拟列对表进行分区。

### 解决方案



