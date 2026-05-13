# MySQL 服务器
[mysqld]
port            = 3306
socket          = /tmp/mysql.sock
skip-external-locking
key_buffer_size = 16K
max_allowed_packet = 1M
table_open_cache = 4
sort_buffer_size = 64K
read_buffer_size = 256K
read_rnd_buffer_size = 256K
net_buffer_length = 2K
thread_stack = 1024K
...
innodb_log_file_size = 5M
innodb_log_buffer_size = 8M
innodb_flush_log_at_trx_commit = 1
innodb_lock_wait_timeout = 50
innodb_log_files_in_group = 2
slow-query-log
general-log
...
```

清单 3-5

MySQL 配置文件摘录

请注意，我们使用方括号 `[]` 定义的节来分组设置。例如，我们看到一个名为 `[client]` 的节，用于定义读取配置文件的任何 MySQL 客户端的选项。同样，我们看到一个名为 `[mysqld]` 的节，它适用于服务器进程（因为可执行文件名为 `mysqld`）。还要注意，我们看到了基本选项的设置，如端口、套接字等。但是，我们也可以使用配置文件为 InnoDB、复制等设置选项。

我建议找到并浏览您安装的配置文件，以便查看选项及其值。如果您遇到需要更改选项的情况——比如为了测试效果或进行实验——您可以使用 `SET` 命令来更改值，可以作为全局设置（影响所有连接）或会话设置（仅适用于当前连接）。

但是，如果您更改了一个同时在配置文件中的全局设置，该值（状态）将仅保留到服务器重新启动。因此，如果您希望保留全局更改，应考虑将它们放在配置文件中。

相反，在会话级别设置值可能对有限时间有益，或者您可能只想为特定任务执行此操作。例如，以下代码关闭了二进制日志，执行一个或多个 SQL 命令，然后再打开二进制日志。这是一个简单但深刻的示例，说明如何在参与复制的服务器上执行操作而不影响其他服务器：

```sql
SET sql_log_bin=0;

SET sql_log_bin=1;
```

有关配置文件以及如何使用它来配置 MySQL 8 的更多信息，包括使用多个选项文件以及这些文件在每个平台上的位置，请参阅在线参考手册 ([`http://dev.mysql.com/doc/refman/8.0/en/`](http://dev.mysql.com/doc/refman/8.0/en/)) 中题为“使用选项文件”的部分。


#### 创建用户与授予访问权限

在使用 MySQL 之前，你还需要了解两个额外的管理操作：创建用户账户以及授予数据库访问权限。你必须首先发出 `CREATE USER` 命令，然后执行一个或多个 `GRANT` 命令。例如，下面展示了创建一个名为 `hvac_user1` 的用户并授予其访问 `room_temp` 数据库的权限：

```
CREATE USER 'hvac_user1'@'%' IDENTIFIED BY 'secret';
GRANT SELECT, INSERT, UPDATE ON room_temp.* TO 'hvac_user1'@'%';
```

第一条命令创建了名为 `hvac_user1` 的用户，但用户名中还包含一个 `@` 符号及其后的字符串。这第二个字符串是用户关联的主机名。也就是说，MySQL 中的每个用户都有一个用户名和一个主机名，以 `user@host` 的形式来唯一标识他们。这意味着用户和主机 `hvac_user1@10.0.1.16` 与用户和主机 `hvac_user1@10.0.1.17` 并不是同一个。然而，`%` 符号可以作为通配符，用于将用户关联到任何主机。`IDENTIFIED BY` 子句用于设置该用户的密码。

## 关于安全性的说明

为你的应用程序创建一个不具有 MySQL 系统完全访问权限的用户总是一个好主意。这样可以最大限度地减少意外更改，并防止被利用。例如，建议你创建一个用户，仅授予其访问你存储（或检索）数据的那些数据库的权限。同时，在使用主机通配符 `%` 时要小心。虽然它使得创建单个用户并让该用户可以从任何主机访问数据库服务器变得更容易，但它也使得那些心怀恶意的人（一旦他们发现了密码）能够更容易地访问你的服务器。

第二条命令授予了数据库访问权限。你可以授予用户的权限有很多。该示例展示了你可能最想授予传感器网络数据库用户的一组权限：读取（`SELECT`）、添加数据（`INSERT`）和更改数据（`UPDATE`）。有关安全性和账户访问权限的更多信息，请参阅在线参考手册。该命令还指定了授予权限的数据库和对象。因此，可以授予用户对某些表的读取（`SELECT`）权限，以及对其他表的写入（`INSERT`、`UPDATE`）权限。此示例授予用户访问 `room_temp` 数据库中所有对象（表、视图等）的权限。

## 基本 SQL 命令

如果你从未使用过数据库系统，学习和掌握它需要培训、经验和大量的毅力。要达到精通所需的知识中，最重要的是如何使用常见的 SQL 命令和概念。本节通过介绍最常见的 MySQL 命令和概念来完成 MySQL 入门，以此作为学习如何使用文档存储的基础。

> **注意**
> 本节并非复制参考手册，而是在较高层次上介绍这些命令和概念。如果你决定使用任何命令或概念，请参阅在线参考手册以获取更多细节、完整的命令语法以及其他示例。

本节回顾了最常见的 SQL 和 MySQL 特有的命令，你需要了解这些命令才能充分利用你的 MySQL 服务器数据库。虽然你已经看到过其中一些命令的实际应用，但本节提供了额外的信息来帮助你使用它们。

需要理解的一个重要规则是，用户提供的变量名是区分大小写的，并且遵循主机平台的大小写敏感性。请查看你平台的在线参考手册，了解大小写敏感性如何影响用户提供的变量。

> **注意**
> 本节中的大多数示例查询都来自物联网（IoT）应用程序，其中一个或多个传感器和设备记录数据。尽管如此，这些示例代表了我们通过 SQL 语句与 MySQL 交互的典型方式。

### 创建数据库和表

你需要学习和掌握的最基本的命令是 `CREATE DATABASE` 和 `CREATE TABLE` 命令。回想一下，像 MySQL 这样的数据库服务器允许你创建任意数量的数据库，你可以在其中添加表并以逻辑方式存储数据。

要创建数据库，请使用 `CREATE DATABASE` 后跟数据库的名称。如果你正在使用 MySQL 客户端，你必须使用 `USE` 命令切换到特定的数据库。客户端的焦点是最近指定的数据库，无论是通过启动时（在命令行上）还是通过 `USE` 命令指定的。

你可以通过首先引用数据库名称来覆盖此设置。例如，`SELECT * FROM db1.table1` 将无论设置的默认数据库如何都会执行。然而，省略数据库名称将导致 `mysql` 客户端使用默认数据库。下面显示了两条命令，用于创建数据库和更改数据库焦点：

```
mysql> CREATE DATABASE greenhouse;
mysql> USE greenhouse;
```

> **提示**
> 如果你想查看服务器上的所有数据库，请使用 `SHOW DATABASES` 命令。

创建一个表需要，没错，就是 `CREATE TABLE` 命令。这个命令有很多选项，不仅允许你指定列及其数据类型，还可以指定其他选项，如索引、外键等。也可以使用 `CREATE INDEX` 命令创建索引（见下面的代码）。下面展示了如何创建一个简单的表来存储植物传感器数据，比如可能用于监测个人温室的数据：

```
CREATE TABLE `greenhouse`.`plants` (
`plant_name` char(30) NOT NULL,
`sensor_value` float DEFAULT NULL,
`sensor_event` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
`sensor_level` char(5) DEFAULT NULL,
PRIMARY KEY `plant_name` (`plant_name`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

注意这里我指定了表名（`plants`）和四列（`plant_name`, `sensor_value`, `sensor_event` 和 `sensor_level`）。我使用了几种数据类型。对于 `plant_name`，我使用了一个最多 30 个字符的字符字段，为 `sensor_value` 使用了浮点数据类型，为 `sensor_event` 使用了时间戳值，为 `sensor_level` 使用了另一个 5 个字符的字符字段。

`TIMESTAMP` 数据类型在你想记录事件或动作的日期和时间的任何时候都特别有用。例如，了解传感器值何时被读取通常很有帮助。通过向表中添加 `TIMESTAMP` 列，你不需要在将值插入数据库表时计算、读取或以其他方式格式化日期和时间。

还要注意，我指定将 `plant_name` 列定义为键，这会创建一个索引。在这种情况下，它也是主键。`PRIMARY KEY` 短语告诉服务器确保表中存在且仅存在一行与该列的值匹配。你可以通过重复关键字来指定多个列用于主键。请注意，所有主键列都必须不允许为空（`NOT NULL`）。

如果你无法确定一组能唯一标识一行的列（而你希望有这样的行为——有些表没有此限制，但好的 DBA 不会这样），你可以使用一个适用于整数字段的人工数据类型选项，称为 `AUTO INCREMENT`。当用于一个列（必须是第一列）时，服务器会自动为插入的每一行递增此值。这样，它就创建了一个默认的主键。有关自动递增列的更多信息，请参阅在线参考手册。

最佳实践建议，在某些情况下，例如在每列值很大或有许多唯一值的表上，在字符字段上使用主键可能不是最优的。这可能会使搜索和索引变慢。在这种情况下，你可以使用一个自动递增字段来人为地添加一个更小（但有些晦涩）的主键。



### 数据搜索

除了前一个示例中显示的类型外，还有更多数据类型可用。你应该查阅在线参考手册以获取完整的数据类型列表。参见“`数据类型`”章节。如果你想了解表的布局或“模式”，请使用 `SHOW CREATE TABLE` 命令。

**提示**  
与数据库类似，你也可以使用 `SHOW TABLES` 命令获取数据库中所有表的列表。

你需要掌握的最常用的基础命令是从表中返回数据（也称为结果集或行）的命令。为此，你需要使用 `SELECT` 语句。这个 SQL 语句是数据库系统的工作主力。所有针对数据的查询都将通过此命令执行。因此，我们将花更多时间来研究可以使用的各种子句（部分），从列列表开始。

**注意**  
虽然我们首先研究 `SELECT` 语句，但如果你想在系统上尝试这些操作，请务必先运行下一节中的 `INSERT` 语句。

`SELECT` 语句允许你指定要从数据中选择哪些列。该列表作为语句的第一部分出现。第二部分是 `FROM` 子句，它指定了你想要从中检索行的表。

**注意**  
`FROM` 子句可以与 `JOIN` 操作符一起使用来连接表。你将在后面的章节中看到一个连接的简单示例。

你指定列的顺序决定了结果集中显示的顺序。如果你想要所有列，可以使用星号 (`*`) 代替。代码清单 3-6 演示了生成相同结果集的三个语句。也就是说，每个输出中都将显示相同的行。实际上，为简单起见，我使用了一个只有四行的表。

```
mysql> SELECT plant_name, sensor_value, sensor_event, sensor_level FROM greenhouse.plants;
+--------------------+--------------+---------------------+--------------+
| plant_name         | sensor_value | sensor_event        | sensor_level |
+--------------------+--------------+---------------------+--------------+
| fern in den        |       0.2319 | 2015-09-23 21:04:35 | NULL         |
| fern on deck       |         0.43 | 2015-09-23 21:11:45 | NULL         |
| flowers in bedroom1    |    0.301 | 2015-09-23 21:11:45 | NULL         |
| weird plant in kitchen |    0.677 | 2015-09-23 21:11:45 | NULL         |
+--------------------+--------------+---------------------+--------------+
4 rows in set (0.00 sec)
mysql> SELECT * FROM greenhouse.plants;
+--------------------+--------------+---------------------+--------------+
| plant_name         | sensor_value | sensor_event        | sensor_level |
+--------------------+--------------+---------------------+--------------+
| fern in den        |       0.2319 | 2015-09-23 21:04:35 | NULL         |
| fern on deck       |         0.43 | 2015-09-23 21:11:45 | NULL         |
| flowers in bedroom1    |    0.301 | 2015-09-23 21:11:45 | NULL         |
| weird plant in kitchen |    0.677 | 2015-09-23 21:11:45 | NULL         |
+--------------------+--------------+---------------------+--------------+
4 rows in set (0.00 sec)
mysql> SELECT sensor_value, plant_name, sensor_level, sensor_event FROM greenhouse.plants;
+--------------+-------------------+--------------+---------------------+
| sensor_value | plant_name        | sensor_level | sensor_event        |
+--------------+-------------------+--------------+---------------------+
|       0.2319 | fern in den       | NULL         | 2015-09-23 21:04:35 |
|         0.43 | fern on deck      | NULL         | 2015-09-23 21:11:45 |
|        0.301 | flowers in bedroom1    | NULL    |         23 21:11:45 |
|        0.677 | weird plant in kitchen | NULL    | 2015-09-23 21:11:45 |
+--------------+------------------------+---------+---------------------+
4 rows in set (0.00 sec)
```

**代码清单 3-6** `SELECT` 语句示例

请注意，前两个语句不仅结果行相同，列也相同且顺序一致，但是第三个语句，虽然生成了相同的结果行，但显示的列顺序不同。

你还可以在列列表中使用函数来执行计算和类似操作。一个特殊的例子是使用 `COUNT()` 函数来确定结果集中的行数，如下所示。有关 MySQL 提供的函数的更多示例，请参阅在线参考手册：

```
SELECT COUNT(*) FROM greenhouse.plants;
```

`SELECT` 语句中的下一个子句是 `WHERE` 子句。在这里，你指定要用于限制结果集中行数的条件。也就是说，只有那些符合条件的行。这些条件基于列，并且可以相当复杂。也就是说，你可以指定基于计算、连接结果等的条件。但大多数条件将是简单的一列或多列的相等或不等式，以便回答问题。例如，假设你想查看传感器读数小于 0.40 的植物。在这种情况下，我们执行以下查询并接收结果。注意，我只指定了两列：植物名称和传感器读数：

```
mysql> SELECT plant_name, sensor_value FROM greenhouse.plants WHERE sensor_value < 0.40;
+---------------------+--------------+
| plant_name          | sensor_value |
+---------------------+--------------+
| fern in den         |       0.2319 |
| flowers in bedroom1 |        0.301 |
+---------------------+--------------+
2 rows in set (0.01 sec)
```

你还可以使用其他子句，包括用于对行进行分组以进行聚合或计数的 `GROUP BY` 子句，以及用于对结果集进行排序的 `ORDER BY` 子句。让我们快速了解一下每个子句，从聚合开始。

假设你想计算表中每个传感器的读数平均值。在这种情况下，我们有一个包含各种传感器随时间推移的读数的表。虽然该示例仅包含四行（因此可能不具有统计意义），但它非常清楚地演示了聚合的概念，如代码清单 3-7 所示。注意，我们得到的结果只是四个传感器读数的平均值。

```
mysql> SELECT plant_name, sensor_value FROM greenhouse.plants WHERE plant_name = 'fern on deck';
+--------------+--------------+
| plant_name   | sensor_value |
+--------------+--------------+
| fern on deck |         0.43 |
| fern on deck |         0.51 |
| fern on deck |        0.477 |
| fern on deck |         0.73 |
+--------------+--------------+
4 rows in set (0.00 sec)
mysql> SELECT plant_name, AVG(sensor_value) AS avg_value FROM greenhouse.plants WHERE plant_name = 'fern on deck' GROUP BY plant_name;
+--------------+-------------------+
| plant_name   | avg_value         |
+--------------+-------------------+
| fern on deck | 0.536750003695488 |
+--------------+-------------------+
1 row in set (0.00 sec)
```

**代码清单 3-7** `GROUP BY` 示例

注意，我在列列表中指定了平均值函数 `AVG()`，并传入了我想要计算平均值的列名。MySQL 中有许多这样的函数可用于执行强大的计算。显然，这是数据库服务器中存在巨大能力的又一个例证，如果在网络中典型的轻量级传感器或聚合节点上实现，将需要更多的资源。

还要注意，我用 `AS` 关键字重命名了包含平均值的列。你可以使用它来重命名任何指定的列，从而更改结果集中的列名，正如你在代码清单中看到的那样。

`GROUP BY` 子句的另一个用途是计数。在这里，我们将 `AVG()` 替换为 `COUNT()`，并收到了匹配 `WHERE` 子句的行数。更具体地说，我们想知道每个植物存储了多少个传感器值。



## 查询数据

```sql
mysql> SELECT plant_name, COUNT(sensor_value) as num_values FROM greenhouse.plants GROUP BY plant_name;
+------------------------+------------+
| plant_name             | num_values |
+------------------------+------------+
| fern in den            |          1 |
| fern on deck           |          4 |
| flowers in bedroom1    |          1 |
| weird plant in kitchen |          1 |
+------------------------+------------+
4 rows in set (0.00 sec)
```

现在，假设我们希望看到结果集按传感器值排序。我们将使用与选择甲板上蕨类植物行相同的查询，但使用 `ORDER BY` 子句按传感器值对行进行升序和降序排序。列表 3-8 显示了每种选项的结果。

```sql
mysql> SELECT plant_name, sensor_value FROM greenhouse.plants WHERE plant_name = 'fern on deck' ORDER BY sensor_value ASC;
+--------------+--------------+
| plant_name   | sensor_value |
+--------------+--------------+
| fern on deck |         0.43 |
| fern on deck |        0.477 |
| fern on deck |         0.51 |
| fern on deck |         0.73 |
+--------------+--------------+
4 rows in set (0.00 sec)
mysql> SELECT plant_name, sensor_value FROM greenhouse.plants WHERE plant_name = 'fern on deck' ORDER BY sensor_value DESC;
+--------------+--------------+
| plant_name   | sensor_value |
+--------------+--------------+
| fern on deck |         0.73 |
| fern on deck |         0.51 |
| fern on deck |        0.477 |
| fern on deck |         0.43 |
+--------------+--------------+
4 rows in set (0.00 sec)
```
列表 3-8
`ORDER BY` 示例

正如我所提到的，`SELECT` 语句远不止这里展示的内容，但我们所看到的将让你在大多数中小型数据库解决方案的典型数据工作中走得很远。

### 创建数据

现在你已经创建了数据库和表，你将需要向表中加载或插入数据。你可以使用 `INSERT INTO` 语句来完成。在这里，我们指定表和行的数据。下面显示了一个简单的示例：

```sql
INSERT INTO greenhouse.plants (plant_name, sensor_value) VALUES ('fern in den', 0.2319);
```

在这个例子中，我通过指定名称和值来为我的一个植物插入数据。你可能会想，其他列怎么办？在这种情况下，其他列包括一个时间戳列，它将由数据库服务器填充。所有其他列（只有一个）将被设置为 `NULL`，这意味着没有可用的值、该值缺失、该值不是零或者该值为空。

请注意，我在行数据之前指定了列。当你想要插入的数据列数少于表包含的列数时，这是必需的。更具体地说，省略列列表意味着你必须为表中的所有列提供数据（或 `NULL`）。此外，列出的列的顺序可以不同于它们在表中定义的顺序。省略列列表将导致列数据的顺序基于它们在表中的出现顺序。

你也可以使用逗号分隔的行值列表，通过同一命令插入多行，如下所示：

```sql
INSERT INTO greenhouse.plants (plant_name, sensor_value) VALUES ('flowers in bedroom1', 0.301), ('weird plant in kitchen', 0.677), ('fern on deck', 0.430);
```

这里我用一个命令插入了多行。请注意，这只是一个简写机制，除了自动提交外，与发出单独的命令没有区别。

### 更新数据

有时你想要更改或更新数据。你可能遇到需要更改一个或多个列的值、替换多行的值或更正数字数据的格式甚至范围的情况。要更新数据，我们使用 `UPDATE` 命令。你可以更新特定的列、更新一组列、对一个或多个列执行计算等等。

更可能的情况是，你或你的用户会想要重命名数据库中的对象。例如，假设我们确定甲板上的植物实际上不是蕨类，而是一种奇异的开花植物。在这种情况下，我们希望将所有植物名为“fern on deck”的行更改为“flowers on deck”。以下命令执行此更改：

```sql
UPDATE greenhouse.plants SET plant_name = 'flowers on deck' WHERE plant_name = 'fern on deck';
```

请注意，这里的关键操作符是 `SET` 操作符。它告诉数据库为指定的列分配一个新值。你可以在命令中列出多个 `SET` 操作。

请注意，我在这里使用了 `WHERE` 子句来将 `UPDATE` 限制到特定的一组行。这与你在 `SELECT` 语句中看到的 `WHERE` 子句相同，作用也一样；它允许你指定限制受影响行的条件。如果你不使用 `WHERE` 子句，更新将应用于所有行。

注意

不要忘记 `WHERE` 子句！发出没有 `WHERE` 子句的 `UPDATE` 命令将影响表中的所有行！

### 删除数据

有时你最终会在表中得到需要删除的数据。也许你使用了测试数据并想要摆脱这些假行，或者你可能想要压缩或清除你的表，或者想要删除不再适用的行。要删除行，请使用 `DELETE FROM` 命令。

让我们看一个例子。假设你有一个正在开发的植物监测解决方案，并且你发现你的某个传感器或传感器节点读数太低，这是由于编码、接线或校准错误造成的。在这种情况下，我们希望删除所有传感器值小于 0.20 的行。以下命令执行此操作：

```sql
DELETE FROM plants WHERE sensor_value < 0.20;
```

注意

不要忘记 `WHERE` 子句！发出没有 `WHERE` 子句的 `DELETE FROM` 命令将永久删除表中的所有行！

请注意，我在这里使用了 `WHERE` 子句。也就是说，一个限制所操作行的条件语句。你可以使用任何列或条件；只要确保它们是正确的！我喜欢先在 `SELECT` 语句中使用相同的 `WHERE` 子句。例如，我会先发出以下命令来检查我是否将要删除我想要的行，并且仅删除这些行。请注意，这是相同的 `WHERE` 子句：

```sql
SELECT * FROM plants WHERE sensor_value < 0.20;
```



### 使用索引

表的创建不使用任何排序方式，也就是说，表本身是无序的。虽然 MySQL 每次返回的数据顺序可能相同，但除非创建索引，否则这种顺序并非隐含（或可靠的）。这里所说的排序，并非你通常所想的排序（那可以通过`SELECT`语句中的`ORDER BY`子句实现）。

实际上，索引是服务器在执行查询时用来读取数据的映射。例如，如果表上没有索引，而你想选择某列值大于特定值的所有行，服务器就必须读取所有行来找出匹配项。然而，如果我们在该列上添加了索引，服务器就只需读取符合条件的那些行。

需要说明的是，索引有多种形式。这里我指的是聚簇索引，其索引中包含了该列的值，这使得服务器仅需读取索引本身，而不必读取数据行，即可进行条件测试。

创建索引时，你既可以在`CREATE TABLE`语句中指定索引，也可以单独执行`CREATE INDEX`命令。以下是一个简单示例：

```sql
CREATE INDEX plant_name ON plants (plant_name);
```

这条命令在`plant_name`列上添加了一个索引。请观察它如何影响表的定义：

```sql
CREATE TABLE `plants` (
`plant_name` char(30) NOT NULL,
`sensor_value` float DEFAULT NULL,
`sensor_event` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
`sensor_level` char(5) DEFAULT NULL,
PRIMARY KEY (`plant_name`),
KEY `plant_name` (`plant_name`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
```

像这样创建的索引不影响表中行的唯一性，即确保特定列（或列组合）的某个值只能访问到唯一的一行。我指的是主键（或主索引）的概念，这是在创建表时使用的一个特殊选项，如前所述。

### 视图

视图是一个或多个表中查询结果的逻辑映射。它们可以像表一样在查询中被引用，这使其成为创建数据子集进行操作的强大工具。你可以使用`CREATE VIEW`来创建视图，并赋予它一个类似表的名称。以下是一个简单示例，我们创建一个测试视图来从表中读取值。在这个例子中，我们限制了视图的大小（行数），但你可以为视图使用各种条件，包括合并来自不同表的数据：

```sql
CREATE VIEW test_plants AS SELECT * FROM plants LIMIT 5;
```

在中小型数据库解决方案中，通常不会遇到视图，但我在这里介绍它们，是为了让你了解其存在，以便在你决定进行额外分析并希望将数据组织成更小的组以便于阅读时使用。

### 触发器

另一个高级概念（以及相关的 SQL 命令）是使用一种事件驱动机制，该机制在数据被更改时“被触发”。也就是说，你可以创建一小段 SQL 命令（一个过程），它将在数据被插入或更改时执行。

触发器可以在几种事件或条件下执行。你可以在更新、插入或删除操作之前或之后设置一个触发器。触发器与单个表关联，其主体是一个特殊结构，允许你对受影响的行进行操作。以下是一个简单示例：

```sql
DELIMITER //
CREATE TRIGGER set_level BEFORE INSERT ON plants FOR EACH ROW
BEGIN
IF NEW.sensor_value < 0.40 THEN
SET NEW.sensor_level = 'LOW';
ELSEIF NEW.sensor_value < 0.70 THEN
SET NEW.sensor_level = 'OK';
ELSE
SET NEW.sensor_level = 'HIGH';
END IF;
END //
DELIMITER ;
```

此触发器将在每次向表中插入数据之前执行。如你在复合语句（`BEGIN`…`END`）中所见，我们根据`sensor_value`的值将名为`sensor_level`的列设置为`LOW`、`OK`或`HIGH`。为了看到实际效果，请考虑以下命令。`FOR EACH ROW`语法允许触发器作用于事务中的所有行：

```sql
INSERT INTO plants (plant_name, sensor_value) VALUES ('plant1', 0.5544);
```

由于我们提供的值（0.5544）小于中间值（0.70），我们期望触发器会为我们填充`sensor_level`列。以下显示了当触发器触发时，确实发生了这种情况：

```sql
+-------------+--------------+---------------------+--------------+
| plant_name  | sensor_value | sensor_event        | sensor_level |
+-------------+--------------+---------------------+--------------+
| plant1      |       0.5544 | 2015-09-23 20:00:15 | OK           |
+-------------+--------------+---------------------+--------------+
1 row in set (0.00 sec)
```

这展示了一种有趣而强大的方式，你可以利用数据库服务器的能力创建派生列，从而节省应用程序中的处理能力和代码。我鼓励你考虑利用这个以及类似强大的概念，来充分发挥数据库服务器的能力。


### 简单连接

数据库系统最强大的概念之一是能够建立数据之间的关系（因此得名“关系型”）。也就是说，一张表中的数据可以引用另一张（或几张）表中的数据。最简单的形式被称为主从关系，即一张表中的一行引用或关联到另一张表中的一个或多个行。

主从关系一个常见（且经典）的例子来自订单跟踪系统，其中一张表包含订单数据，另一张表包含该订单的明细项。这样，我们可以将客户编号和配送信息等订单信息只存储一次，并在检索订单本身时对表进行“连接”。

让我们看一个来自名为 `world` 的示例数据库的例子。你可以在 MySQL 网站上找到这个数据库（`http://dev.mysql.com/doc/index-other.xhtml`）。可以自由下载它以及任何其他示例数据库。它们都展示了数据库系统的各种设计。在练习查询数据时你也会发现它很方便，因为它包含了不只几行的简单数据。

### 注意

如果你想运行下面的例子，需要按照示例文档中的描述安装 `world` 数据库（`http://dev.mysql.com/doc/world-setup/en/world-setup-installation.xhtml`）。

清单 3-9 展示了一个简单连接的例子。这里有很多内容，所以请花点时间检查 `SELECT` 语句的各个部分，特别是我是如何指定 `JOIN` 子句的。你可以忽略 `LIMIT` 选项，因为它只是限制结果集中的行数。

```sql
mysql> USE world;
mysql> SELECT Name, Continent, Language FROM Country JOIN CountryLanguage ON Country.Code = CountryLanguage.CountryCode LIMIT 10;
+-------------+---------------+------------+
| Name        | Continent     | Language   |
+-------------+---------------+------------+
| Aruba       | North America | Dutch      |
| Aruba       | North America | English    |
| Aruba       | North America | Papiamento |
| Aruba       | North America | Spanish    |
| Afghanistan | Asia          | Balochi    |
| Afghanistan | Asia          | Dari       |
| Afghanistan | Asia          | Pashto     |
| Afghanistan | Asia          | Turkmenian |
| Afghanistan | Asia          | Uzbek      |
| Angola      | Africa        | Ambo       |
+-------------+---------------+------------+
10 rows in set (0.00 sec)
```

**清单 3-9**
**简单连接示例**

这里我使用了一个 `JOIN` 子句，它指定了两张表，使得第一张表通过一个特定的列及其值与第二张表进行连接（`ON` 指定了匹配条件）。数据库服务器所做的是读取表中的每一行，并仅返回那些在指定列上匹配的行。任一表中不在另一表中的行都不会被返回。

#### 提示

你可以通过不同的连接来检索那些行。请参阅在线参考手册 `https://dev.mysql.com/doc/refman/8.0/en/join.xhtml`，以获取关于可能的连接类型（包括内连接和外连接）的更多详细信息。

还要注意，我只包含了少数几个列。在这个例子中，我从 `Country` 表中指定了国家名称和所在大陆，从 `CountryLanguage` 表中指定了语言列。如果列名不是唯一的（相同的列出现在每个表中），我必须通过表名来指定它们，例如 `Country.Name`。事实上，始终以这种方式限定列被认为是良好的实践。

这个例子中有一个我感觉需要指出的有趣反常现象。事实上，有些人会认为这是一个设计缺陷。注意在 `JOIN` 子句中，我为每个表指定了表名和列名。这是正常且正确的，但注意列名在两张表中并不匹配。虽然这真的无关紧要，只会造成一点点额外的输入，但一些数据库管理员会认为这是错误的，并希望将公共列名在两个表中设为相同。

连接的另一个用途是检索公共的、存档的或查找数据。例如，假设你有一个表存储那些不变（或很少变化）的事物的详细信息，例如与邮政编码关联的城市或与身份证号码（例如 SSN）关联的姓名。你可以将这些信息存储在另一个单独的表中，并在需要时通过一个公共列（及其值）连接数据。在这种情况下，那个公共列可以被用作外键，这是另一个高级概念。

外键用于维护数据完整性（即，如果你在一张表中有关联到另一张表的数据，但这种关系需要保持一致）。例如，如果你想确保在删除主表行时，所有从表行也被删除，你可以在主表中声明一个外键，指向从表的一列（或多列）。请参阅在线参考手册以获取关于外键的更多信息。

这次关于连接的讨论只触及了最基础的部分。确实，连接可以说是数据库系统中最困难且常常被混淆的领域之一。如果你发现你想使用连接来组合多张表或扩展数据，以便数据从多个表中提供（外连接），你应该花些时间深入研究数据库概念，例如 Clare Churcher 的书 *Beginning Database Design*（Apress，2012）。

## 存储例程

MySQL 中还有许多其他概念和命令可用，但其中两个可能令人感兴趣的是 `PROCEDURE`（存储过程）和 `FUNCTION`（存储函数），有时它们被统称为**存储例程**。我在此介绍这些概念，以便当你想要探索它们时，能在高层次上理解其用法。

假设你需要运行多个命令来更改数据。也就是说，你需要基于计算执行一些复杂的更改。对于这类操作，MySQL 提供了存储过程的概念。存储过程允许你在每次调用该过程时，执行一个复合语句（一系列 SQL 命令）。存储过程有时被认为是一种主要用于周期性维护的高级技术，但即使在更简单的情况下，它们也可能非常方便。

例如，假设你想开发一个使用 SQL 的自定义数据库应用，但因为在开发阶段，你需要周期性地重新开始，并希望先清除所有数据。如果只有一张表，存储过程帮助不大，但假设有若干张表分布在多个数据库中（对于较大的数据库来说这并不罕见）。这种情况下，存储过程可能会有帮助。

在 MySQL 客户端中输入包含复合语句的命令时，你需要临时更改**定界符**（分号 `;`），以便行尾的分号不会终止命令的输入。例如，在编写包含复合语句的命令前使用 `DELIMITER //`，使用 `//` 来结束命令，然后再用 `DELIMITER ;` 将定界符改回来。这只是在使用客户端时的操作。

由于存储过程可能相当复杂，如果你决定使用它们，请在尝试开发自己的存储过程之前，先阅读在线参考手册中关于 **`CREATE PROCEDURE` 和 `CREATE FUNCTION` 语法** 的部分。创建存储过程远比本节描述的要复杂。

现在，假设你想执行一个复合语句并返回一个结果——你想将其作为函数使用。你可以通过执行计算、数据转换或简单翻译来使用函数填充数据。因此，函数可用于提供值以填充列值、提供聚合、提供日期操作等等。

你已经见过几个函数（`COUNT`，`AVG`）。这些被认为是**内置函数**，在线参考手册中有专门的一节介绍它们。然而，你也可以创建自己的函数。例如，你可能希望创建一个函数来对数据执行某种规范化操作。更具体地说，假设有一个传感器在特定范围内产生一个值，但根据该值以及来自另一个传感器或查找表的另一个值，你希望对该值进行加、减、求平均等操作以校正它。你可以编写一个函数来完成此操作，并在触发器中调用它来为计算列填充值。

### 如何修改对象？

你可能想知道当需要修改表、过程、触发器等时该怎么办。放心，你不必从头开始！MySQL 为每个对象提供了 `ALTER` 命令。即，有 `ALTER TABLE`、`ALTER PROCEDURE` 等等。有关每个 `ALTER` 命令的更多信息，请参阅在线参考手册中名为“**数据定义语句**”的部分。

现在我们已经了解了基本的 SQL 命令及其使用方法，接下来让我们看看如何将我们的应用程序连接到 MySQL。

## 连接应用程序

你已经看到了如何使用 MySQL 客户端和 MySQL Shell 连接到 MySQL 服务器。这些是交互式工具，我们可以在其中执行查询，但对于从我们的应用程序或其他用户保存数据来说，这并无帮助。我们需要的是所谓的**连接器**。连接器是一个编程模块，旨在允许我们的脚本或程序将数据发送到数据库服务器。连接器还允许我们查询数据库服务器以从服务器获取数据。

我将介绍你在开发自己的应用程序时可能遇到的两种主要连接器。我将每个作为教程来呈现，以便你跟随操作。我从一个用于 Python 脚本的连接器（Connector/Python）开始，然后介绍一个用于编写简化 Java 的连接器（Connector/J）。但首先，让我们看看我们的应用程序可以使用哪些连接器。

### MySQL 数据库连接器

MySQL 有许多数据库连接器。Oracle 为多种语言提供了多个数据库连接器。表 3-1 显示了当前可从 [`http://dev.mysql.com/downloads/`](http://dev.mysql.com/downloads/) 下载的数据库连接器。

表 3-1
MySQL 连接器

| 连接器 | 描述 | 下载 URL |
| --- | --- | --- |
| C API (libmysqlclient) | 用于 C 开发的客户端库 | [`https://dev.mysql.com/downloads/c-api/`](https://dev.mysql.com/downloads/c-api/) |
| Connector/C++ | 标准化的 C++ 应用程序 | [`https://dev.mysql.com/downloads/connector/cpp/`](https://dev.mysql.com/downloads/connector/cpp/) |
| Connector/J | Java 应用程序 | [`https://dev.mysql.com/downloads/connector/j/`](https://dev.mysql.com/downloads/connector/j/) |
| Connector/Net | Windows .Net 平台 | [`https://dev.mysql.com/downloads/connector/net/`](https://dev.mysql.com/downloads/connector/net/) |
| Connector/Node.js | Node.js 应用程序 | [`https://dev.mysql.com/downloads/connector/nodejs/`](https://dev.mysql.com/downloads/connector/nodejs/) |
| Connector/ODBC | 通用的 ODBC 应用程序 | [`https://dev.mysql.com/downloads/connector/odbc/`](https://dev.mysql.com/downloads/connector/odbc/) |
| Connector/Python | Python 应用程序 | [`https://dev.mysql.com/downloads/connector/python/`](https://dev.mysql.com/downloads/connector/python/) |
| MySQL native driver for PHP (mysqlnd) | PHP 5.3 或更新版本的连接器 | [`https://dev.mysql.com/downloads/connector/php-mysqlnd/`](https://dev.mysql.com/downloads/connector/php-mysqlnd/) |

如你所见，几乎任何你可能遇到的编程语言都有对应的连接器。你可以在 [`https://dev.mysql.com/doc/connectors/en/`](https://dev.mysql.com/doc/connectors/en/) 找到上述每个连接器的文档。

### 示例数据库

如果你想跟随示例操作，需要设置示例数据库。如果已安装 MySQL，你可以使用 MySQL Shell 或客户端执行以下查询：

```sql
-- 一个用于存储植物土壤湿度和环境温度的数据库
CREATE DATABASE plant_monitoring;
USE plant_monitoring;
-- 此表存储关于植物的信息。
CREATE TABLE plant_monitoring.plants (
id int NOT NULL AUTO_INCREMENT,
name char(50) DEFAULT NULL,
location char(30) DEFAULT NULL,
climate enum ('inside','outside') DEFAULT 'inside',
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

我们将作为示例的一部分插入数据。如果你想同时运行两个示例，并且多次运行插入示例，请务必使用以下查询清空表：

```sql
DELETE FROM plant_monitoring.plants;
```

现在，让我们从 Connector/Python 开始，看两个例子。

### 示例连接器：Connector/Python

Oracle 提供的 Python 连接器是一款功能齐全的工具，可为 Python 应用程序和脚本提供与 MySQL 数据库服务器的连接。`Connector/Python` 支持所有当前的 MySQL 服务器版本。它旨在实现 Python 与 MySQL 之间的自动数据类型转换，使构建查询和解析结果变得轻松。它还支持压缩、允许通过 SSL 连接，并且支持所有 MySQL SQL 命令。

`Connector/Python` 必须安装在您运行代码的计算机上，方式与安装您可能使用的任何其他 Python 库相同。在 Python 脚本中使用 `Connector/Python` 包括导入基础模块、建立连接以及使用游标执行查询。

在我们开始探讨如何使用 `Connector/Python` 编写一些支持 MySQL 数据库的应用程序之前，让我们先谈谈如何获取和安装 `Connector/Python`。

**Python？那不是一种蛇吗？**

Python 编程语言是一种高级语言，其设计初衷是尽可能像阅读英语一样自然，同时保持简洁、易学且功能强大。Python 爱好者会告诉您，设计者确实实现了这些目标。

如果您从未使用过 Python，或者想了解更多关于它的信息，以下是一些介绍该语言的优秀书籍。互联网上也有大量资源可供利用，包括位于 `www.python.org/doc/` 的 Python 文档页面：

*   `Programming the Raspberry Pi`，作者 Simon Monk（麦格劳-希尔出版社，2013 年）
*   `Beginning Python from Novice to Professional`，第二版，作者 Magnus Lie Hetland（Apress 出版社，2008 年）
*   `Python Cookbook`，作者 David Beazley 和 Brian K. Jones（O'Reilly Media 出版社，2013 年）

有趣的是，Python 是以英国喜剧团体 Monty Python 的名字命名的，而不是那种爬行动物。在学习 Python 时，您可能会遇到一些对 Monty Python 剧集的戏谑引用。由于我对 Monty Python 情有独钟，我觉得这些引用很有趣。当然，您的感受可能有所不同。

#### 安装 Connector/Python

下载过程与您为服务器所做的相同。您可以从 Oracle 的 MySQL 网站 (`http://dev.mysql.com/downloads/connector/python/`) 下载 `Connector/Python`。该页面会自动检测您的平台并显示适用于该平台的可用下载项。您可能会看到多个选项。请务必选择与您的配置相匹配的那一个。

由于大多数平台都预装了 Python，您可能不需要对系统做任何准备；只需下载安装程序并安装即可。然而，首选的安装方法是使用 Python 包管理器（来自 PyPi）通过 `pip install` 命令来获取并安装连接器。以下演示了如何使用 `pip` 安装 `Connector/Python`：

```
% pip3 install mysql-connector-python
Collecting mysql-connector-python
Downloading mysql_connector_python-8.0.28-py2.py3-none-any.whl (342 kB)
|████████████████████████████████| 342 kB 198 kB/s
Collecting protobuf>=3.0.0
Downloading protobuf-3.19.3-cp310-cp310-macosx_10_9_universal2.whl (1.0 MB)
|████████████████████████████████| 1.0 MB 544 kB/s
Installing collected packages: protobuf, mysql-connector-python
Successfully installed mysql-connector-python-8.0.28 protobuf-3.19.3
```

注意 `pip` 安装程序命令。本例中使用了 `pip3` 命令，因为该计算机同时安装了 Python 2.X 和 3.X，我们希望确保将其安装到 Python 3.X 环境中。

**注意**

您应该安装 Python 3.7 或更高版本。

另请注意，被安装软件包所需的任何先决条件也会被自动下载和安装。正如您所见，使用 `pip` 比单独下载和安装软件包要容易得多。

**提示**

有关在特定平台上安装的具体说明，请参阅在线参考手册 (`http://dev.mysql.com/doc/connector-python/en/connector-python-installation.xhtml`)。

#### 检查安装

一旦安装了 `Connector/Python`，您可以通过下面的简短示例来验证它是否正常工作。首先输入命令 `python`（或者 `python3`，如果您想确保使用的是 Python 3.X 安装）。这将打开一个交互式提示符，允许您一次输入一行 Python 代码并执行；它是一个 Python 命令行解释器，对于测试小段代码非常有用。只需按照示例所示输入以下几行：

```
% python3
Python 3.10.2 (v3.10.2:a58ebcc701, Jan 13 2022, 14:50:16) [Clang 13.0.0 (clang-1300.0.29.30)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import mysql.connector
>>> print(mysql.connector.__version__)
8.0.28
>>>
>>> quit()
```

您应该会看到打印出的 `Connector/Python` 版本号。如果您看到任何关于找不到连接器的错误，请务必检查您的安装以确保其工作正常。一旦您可以成功访问 `Connector/Python`，就可以继续学习一些示例脚本了。

Python 脚本（应用程序）使用类似 `<something>.py` 的文件名和扩展名保存，并通过以下命令行方式执行。我们将使用此方法来执行以下示例。因此，对于每个示例，您应该在文本编辑器中打开一个文件，输入所示的代码，保存文件，然后从命令行运行该脚本：

```
$ python3 my_script.py
```

如果您拥有并熟悉某个 Python 集成开发环境（IDE），您可以使用它来代替通过文本编辑器创建文件并通过命令行执行。优秀的 Python IDE 示例包括 Thonny (`https://thonny.org/`) 和 PyCharm (`www.jetbrains.com/pycharm/download`)。两者都适用于 Windows、macOS 和 Linux。

Thonny 是一款免费、基础的 IDE，因此非常易于使用但功能有限；而 PyCharm 提供社区版和企业版，支持企业级 Python 开发的大量功能。

## 连接到 MySQL 服务器

让我们从一个简单的例子开始，我们将连接到 MySQL 服务器并获取数据库列表。我们将此示例命名为 `mysql_connector.py`。

在这个例子中，我们首先导入 Connector/Python 连接器类，然后为了保持整洁，我们使用一个字典来存储连接信息。有了连接字典后，我们调用 `connect()` 方法来连接服务器。

接下来，我们打开一个新的游标，并通过连接类的 `cursor()` 方法获取一个游标类的实例。使用游标类，我们可以调用 `execute()` 方法并传入一条 SQL 语句。执行后，获取返回的行并打印每行的第一列。最后，我们关闭游标和连接以整理资源。清单 3-10 展示了此示例的完整代码。正如你将看到的，它非常易于理解。

> **注意**
> 请务必将用户和密码更改为与你的安装相匹配。

```python
"""mysql_connect.py"""
#
