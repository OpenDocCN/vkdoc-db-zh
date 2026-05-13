# 参考分区

参考分区解决了父/子表等分区的问题；即，当你需要子表以这样的方式进行分区，使得每个子表分区与一个父表分区具有一一对应的关系。这在诸如数据仓库等情况下非常重要，在那里你希望保持特定数量的数据在线（比如最近五年的 `ORDER` 信息），并且需要确保相关的子数据（`ORDER_LINE_ITEMS` 数据）也在线。在这个经典的例子中，`ORDERS` 表通常会有一个列 `ORDER_DATE`，从而可以轻松地按月分区，进而便于保持最近五年的数据在线。随着时间推移，你只需准备好下个月的分区用于加载，并丢弃最旧的分区。然而，当你考虑 `ORDER_LINE_ITEMS` 表时，你会发现你会遇到一个问题。它没有 `ORDER_DATE` 列，并且在 `ORDER_LINE_ITEMS` 表中没有任何内容可以用来对其进行分区；因此，它不利于清除旧信息或加载新信息。

在过去，在参考分区出现之前，开发人员不得不对数据进行反规范化，实际上就是将 `ORDER_DATE` 属性从父表 `ORDERS` 复制到子表 `ORDER_LINE_ITEMS` 表中。这带来了典型的数
据冗余问题，即增加了存储开销、增加了数据加载资源、级联更新问题（如果你修改了父表，你必须确保更新父数据的所有副本），等等。此外，如果你在数据库中启用了外键约束（你应该这样做），你会发现你失去了截断或丢弃父表中旧分区的能力。例如，让我们从 `ORDERS` 表开始，建立传统的 `ORDERS` 和 `ORDER_LINE_ITEMS` 表：

```
$ sqlplus eoda/foo@PDB1
SQL> create table orders
(
order#      number primary key,
order_date  date,
data       varchar2(30)
)
enable row movement
PARTITION BY RANGE (order_date)
(
PARTITION part_2020 VALUES LESS THAN (to_date('01-01-2021','dd-mm-yyyy')) ,
PARTITION part_2021 VALUES LESS THAN (to_date('01-01-2022','dd-mm-yyyy'))
);
Table created.
SQL> insert into orders values (1, to_date( '01-jun-2020', 'dd-mon-yyyy' ), 'xxx' );
1 row created.
SQL> insert into orders values (2, to_date( '01-jun-2021', 'dd-mon-yyyy' ), 'xxx' );
1 row created.
```

现在，我们将创建 `ORDER_LINE_ITEMS` 表——包含一些指向 `ORDERS` 表的数据：

```
SQL> create table order_line_items
(
order#      number,
line#       number,
order_date  date, -- 手动从 ORDERS 复制而来！
data       varchar2(30),
constraint c1_pk primary key(order#,line#),
constraint c1_fk_p foreign key(order#) references orders
)
enable row movement
PARTITION BY RANGE (order_date)
(
PARTITION part_2020 VALUES LESS THAN (to_date('01-01-2021','dd-mm-yyyy')) ,
PARTITION part_2021 VALUES LESS THAN (to_date('01-01-2022','dd-mm-yyyy'))
);
Table created.
SQL> insert into order_line_items values ( 1, 1, to_date( '01-jun-2020', 'dd-mon-yyyy' ), 'yyy' );
1 row created.
SQL> insert into order_line_items values ( 2, 1, to_date( '01-jun-2021', 'dd-mon-yyyy' ), 'yyy' );
1 row created.
```

现在，如果我们要删除包含 2020 年数据的 `ORDER_LINE_ITEMS` 分区，你我都知道，相应的 2020 年 `ORDERS` 分区也可以被删除，而不会违反引用完整性约束。你我知道这一点，但数据库并不知道这个事实：

```
SQL> alter table order_line_items drop partition part_2020;
Table altered.
SQL> alter table orders drop partition part_2020;
alter table orders drop partition part_2020
*
ERROR at line 1:
ORA-02266: unique/primary keys in table referenced by enabled foreign keys
```

因此，这种方法不仅笨重、资源密集，并且可能损害我们的数据完整性，它还阻止了我们做管理分区表时经常需要做的事情：清除旧信息。

引入参考分区。使用参考分区，子表将继承其父表的分区方案，而无需对分区键进行反规范化，*并且*它允许数据库理解子表与父表是等分区的。也就是说，当我们截断或删除相应的子表分区时，我们将能够截断或删除父表分区。

重新实现我们前面例子的简单语法可以如下所示。我们将重用现有的父表 `ORDERS`，并且只截断该表：

```
SQL> drop table order_line_items cascade constraints;
Table dropped.
SQL> truncate table orders;
Table truncated.
SQL> insert into orders values ( 1, to_date( '01-jun-2020', 'dd-mon-yyyy' ), 'xxx' );
1 row created.
SQL> insert into orders values ( 2, to_date( '01-jun-2021', 'dd-mon-yyyy' ), 'xxx' );
1 row created.
```

并创建一个新的子表：

```
SQL> create table order_line_items
(
order#      number,
line#       number,
data       varchar2(30),
constraint c1_pk primary key(order#,line#),
constraint c1_fk_p foreign key(order#) references orders
)
enable row movement
partition by reference(c1_fk_p);
Table created.
SQL> insert into order_line_items values ( 1, 1, 'yyy' );
1 row created.
SQL> insert into order_line_items values ( 2, 1, 'yyy' );
1 row created.
```

神奇之处在于 `CREATE TABLE` 语句的第 10 行。在这里，我们用 `PARTITION BY REFERENCE` 替换了范围分区语句。这允许我们指定要使用的外键约束名称，以确定我们的分区方案将是什么。在这里，我们看到外键指向 `ORDERS` 表——数据库读取了 `ORDERS` 表的结构并确定它有两个分区——因此，我们的子表也将有两个分区。事实上，如果我们现在查询数据字典，我们可以看到这两个表具有完全相同的分区结构：

```
SQL> select table_name, partition_name
from user_tab_partitions
where table_name in ( 'ORDERS', 'ORDER_LINE_ITEMS' )
order by table_name, partition_name;
TABLE_NAME           PARTITION_NAME
-------------------- --------------------
ORDERS               PART_2020
ORDERS               PART_2021
ORDER_LINE_ITEMS     PART_2020
ORDER_LINE_ITEMS     PART_2021
```

此外，由于数据库理解这两个表是相关的，我们可以删除父表分区，并让它自动清理相关的子表分区（因为子表从父表继承，任何对父表分区结构的更改都会级联下去）：

```
SQL> alter table orders drop partition part_2020 update global indexes;
Table altered.
SQL> select table_name, partition_name
from user_tab_partitions
where table_name in ( 'ORDERS', 'ORDER_LINE_ITEMS' )
order by table_name, partition_name;
TABLE_NAME           PARTITION_NAME
-------------------- --------------------
ORDERS               PART_2021
ORDER_LINE_ITEMS     PART_2021
```

因此，之前我们无法执行的 `DROP` 操作现在被允许了，并且它会自动级联到子表。此外，如果我们 `ADD` 一个分区，如下所示，我们可以看到该操作也会级联；父表和子表之间将存在一对一的对应关系：

```
SQL> alter table orders add partition
part_2022 values less than
(to_date( '01-01-2023', 'dd-mm-yyyy' ));
Table altered.
SQL> select table_name, partition_name
from user_tab_partitions
where table_name in ( 'ORDERS', 'ORDER_LINE_ITEMS' )
order by table_name, partition_name;
TABLE_NAME           PARTITION_NAME
-------------------- ---------------
ORDERS               PART_2021
ORDERS               PART_2022
ORDER_LINE_ITEMS     PART_2021
ORDER_LINE_ITEMS     PART_2022
```



