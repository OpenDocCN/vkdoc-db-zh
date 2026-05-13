# 过度访问 DUAL

代码过度访问 `DUAL` 表的情况并不少见。您应避免过度使用 `DUAL` 表访问。从 PL/SQL 访问 `DUAL` 会导致上下文切换，从而影响性能。本节将回顾一些过度访问 `DUAL` 的常见原因，并讨论相应的缓解方案。

## 日期算术运算

没有理由为了执行算术运算或 `DATE` 操作而访问 `DUAL` 表，因为大多数操作都可以使用 PL/SQL 语言结构完成。甚至 `SYSDATE` 也可以在 PL/SQL 中直接访问，无需访问 SQL 引擎。在代码清单 1-7 中，高亮显示的 SQL 语句使用 `SELECT.. from DUAL` 语法计算 UNIX 纪元时间（纪元时间定义为自 1970 年 1 月 1 日午夜以来经过的秒数）。虽然访问 `DUAL` 表很快，但执行该语句仍然会导致 SQL 和 PL/SQL 引擎之间的上下文切换。

***代码清单 1-7.** 过度访问 DUAL — 算术运算*

```
DECLARE
  l_epoch INTEGER;
BEGIN
  SELECT ((SYSDATE-TO_DATE('01-JAN-1970 00:00:00', 'DD-MON-YYYY HH24:MI:SS'))
          * 24 *60 *60 )
  INTO l_epoch
  FROM DUAL;
  dbms_output.put_line(l_epoch);
END;
```

您可以使用 PL/SQL 结构执行算术计算来避免不必要的 `DUAL` 表访问，就像这样：

```
l_epoch := (SYSDATE- TO_DATE('01-JAN-1970 00:00:00', 'DD-MON-YYYY HH24:MI:SS'))
          * 24 *60 *60 ;
```

根本没有必要调用 SQL 来执行数值运算。直接在 PL/SQL 中完成工作即可。您的代码将更易于阅读，性能也会更好。

过度访问以查询当前日期或时间戳是导致 `DUAL` 表访问增加的另一个原因。考虑直接在 SQL 语句中编码调用 `SYSDATE`，而不是将 `SYSDATE` 选择到变量中，然后将该值传回 SQL 引擎。如果需要在插入行后访问列值，请使用 `returning` 子句获取列值。如果需要在 PL/SQL 本身中访问 `SYSDATE`，请使用 PL/SQL 结构将当前日期获取到变量中。

### 访问序列

不必要访问 `DUAL` 表的另一个常见原因是从序列中检索下一个值。代码清单 1-8 展示了一个代码片段，它从 `cust_id_seq` 中选择下一个值到一个变量中，然后使用该变量插入到 `customers` 表中。

***代码清单 1-8.** 过度访问 DUAL — 序列*

```
DECLARE
  l_cust_id NUMBER;
BEGIN
  FOR c1 in (SELECT cust_first_name, cust_last_name FROM customers
             WHERE cust_marital_status!='married')
  LOOP
    SELECT cust_hist_id_seq.nextval INTO l_cust_id FROM dual;
    INSERT INTO customers_hist
      (cust_hist_id, first_name, last_name )
    VALUES
      (l_cust_id, c1.cust_first_name, c1.cust_last_name)
      ;
  END LOOP;
END;
/
PL/SQL 过程已成功完成。

已用时间: 00:00:01.89
```

更好的方法是避免将值检索到变量中，而是直接在 `INSERT` 语句本身中从序列获取值。以下代码片段演示了一个 `INSERT` 语句，它使用序列生成的值将行插入到 `customers` 表中。通过这种编码实践，您可以避免访问 `DUAL` 表，从而避免引擎之间的上下文切换。

```
Insert into customers (cust_id, ...)
Values (cust_id_seq.nextval,...);
```

更好的做法是将 PL/SQL 块重写为 SQL 语句。例如，下面的重写语句在 0.2 秒内完成，而基于 PL/SQL 循环的处理运行时间为 1.89 秒：

```
INSERT INTO customers_hist
SELECT
   cust_hist_id_seq.nextval, cust_first_name, cust_last_name
FROM customers  
WHERE cust_marital_status!='married';

已创建 23819 行。
已用时间: 00:00:00.20
```

## 填充主从关系行

过度访问 `DUAL` 表的另一个常见原因是向涉及主从关系的表中插入数据。通常，在这种编码实践中，主表的主键值从序列获取到局部变量中。然后，在向主表*和*从表插入数据时使用该局部变量。之所以形成这种方法，是因为在向从表插入数据时需要用到主表的主键值。

Oracle 数据库版本 9i 引入的一项新 SQL 功能通过允许您返回插入行的值提供了更好的解决方案。您可以使用 `DML RETURNING` 子句从新插入的主行中检索键值。然后，您可以在向从表插入时使用该键值。例如：

```
INSERT INTO customers (cust_id, ...)
VALUES (cust_id_seq.nextval,...)
RETURNING cust_id into l_cust_id;
...
INSERT INTO customer_transactions (cust_id, ...)
VALUES (l_cust_id,...)
...
```



### 不必要的函数执行

执行函数调用通常意味着必须将指令集的不同部分加载到 CPU 中。执行会从指令的一部分跳转到另一部分。这种执行跳转会引发性能问题，因为它需要清空并重新填充指令流水线。其结果是额外的 CPU 使用率。

通过避免不必要的函数执行，你可以避免指令流水线无谓的清空与重填，从而最小化对 CPU 的需求。重申一下，我并非反对模块化的编码实践。我只反对过度和不必要的函数调用执行。我最好通过例子来解释。

在清单 1-9 中，`log_entry` 是一个调试函数，每次验证时都会被调用。但该函数本身会检查 `v_debug`，并且仅在调试标志设置为 true 时才插入消息。想象一个程序，在循环中执行数百个这样复杂的业务验证。本质上，即使 `v_debug` 标志设置为 false，`log_entry` 函数也会被不必要地调用数百万次。

**清单 1-9. 不必要的函数调用**

```sql
create table log_table ( message_seq number, message varchar2(512));
create sequence message_id_seq;

DECLARE
 l_debug BOOLEAN := FALSE;
 r1 integer;
 FUNCTION log_entry( v_message IN VARCHAR2, v_debug in boolean)
  RETURN number
  IS
  BEGIN
    IF(v_debug) THEN
      INSERT INTO log_table
        (message_seq, MESSAGE)
        VALUES
        (message_id_seq.nextval, v_message
        );
    END IF;
    return 0;
  END;
BEGIN
 FOR c1 IN (
        SELECT s.prod_id, s.cust_id,s.time_id,
               c.cust_first_name, c.cust_last_name,
               s.amount_sold
        FROM sales s,
             customers c
        WHERE s.cust_id    = c.cust_id and
        s.amount_sold> 100)
  LOOP
    IF c1.cust_first_name IS NOT NULL THEN
      r1 := log_entry ('first_name is not null ', l_debug );
    END IF;

    IF c1.cust_last_name IS NOT NULL THEN
      r1 := log_entry ('Last_name is not null ', l_debug);
    END IF;
  END LOOP;
END;
/

PL/SQL procedure successfully completed.
Elapsed: 00:00:00.54
```

清单 1-9 中的代码可以重写，以仅在变量 `l_debug` 标志设置为 true 时才调用 `log_entry` 函数。此重写减少了 `log_entry` 函数不必要的执行。重写后的程序在 0.43 秒内完成。执行次数越多，性能提升越明显。

```sql
...
IF first_name IS NULL AND l_debug=TRUE THEN
  log_entry('first_name is null ');
END IF;
...
/
PL/SQL procedure successfully completed.
Elapsed: 00:00:00.43
```

对于一个更好的方法，可以考虑使用条件编译结构来完全避免执行此代码片段。在清单 1-10 中，高亮显示的代码使用了 `$IF-$THEN` 结构和一个条件变量 `$$debug_on`。如果条件变量 `$$debug_on` 为 true，则执行代码块。在生产环境中，`$$debug_on` 变量将为 FALSE，从而消除函数执行。注意，程序的耗时进一步减少到 0.34 秒。

**清单 1-10. 使用条件编译避免不必要的函数调用**

```sql
DECLARE
 l_debug BOOLEAN := FALSE;
 r1 integer;
 FUNCTION log_entry( v_message IN VARCHAR2, v_debug in boolean)
  RETURN number
  IS
  BEGIN
    IF(v_debug) THEN
      INSERT INTO log_table
        (message_seq, MESSAGE)
        VALUES
        (message_id_seq.nextval, v_message
        );
    END IF;
    return 0;
  END;

BEGIN
FOR c1 IN (
        SELECT s.prod_id, s.cust_id,s.time_id,
               c.cust_first_name, c.cust_last_name,
               s.amount_sold
        FROM sales s,
             customers c
        WHERE s.cust_id    = c.cust_id and
        s.amount_sold> 100)
  LOOP
   $IF $$debug_on $THEN
    IF c1.cust_first_name IS NOT NULL  THEN
      r1 := log_entry ('first_name is not null ', l_debug );
    END IF;
    IF c1.cust_last_name IS NOT NULL   THEN
      r1 := log_entry ('Last_name is not null ', l_debug);
    END IF;
   $END
    null;
  END LOOP;
END;
/
PL/SQL procedure successfully completed.
Elapsed: 00:00:00.34
```

不必要地调用函数的问题往往在从另一个模板程序复制并修改的程序中频繁出现。请注意这个问题。如果一个函数不需要被调用，就避免调用它。

### 解释型与原生编译

默认情况下，PL/SQL 代码作为解释型代码执行。在 PL/SQL 编译期间，代码被转换为中间格式并存储在数据字典中。在执行时，该中间代码由引擎执行。

Oracle Database 9i 版本引入了一个名为原生编译的新特性。PL/SQL 代码被编译成机器指令，并作为共享库存储。由于现代编译器可以内联子程序并避免指令跳转，使用原生编译时，过度的函数执行可能影响较小。



#### 昂贵的函数调用

如果执行一个函数会消耗几秒钟的实际时间，那么在循环中调用该函数将导致代码性能低下。你应该对频繁执行的函数进行优化，使其尽可能高效地运行。

在清单 1-11 中，如果函数 `calculate_epoch` 在循环中被调用数百万次。即使该函数的执行仅消耗 0.01 秒，一百万次执行也会导致 2.7 小时的耗时。解决此性能问题的一个选项是优化函数使其能在几毫秒内执行，但如此大幅度的优化并非总能实现。

***清单 1-11.** 昂贵的函数调用*

```sql
CREATE OR REPLACE FUNCTION calculate_epoch (d in date)
 RETURN NUMBER DETERMINISTIC IS
 l_epoch number;
BEGIN
l_epoch := (d - TO_DATE('01-JAN-1970 00:00:00', 'DD-MON-YYYY HH24:MI:SS'))
          * 24 *60 *60 ;
 RETURN l_epoch;
END calculate_epoch;
/

SELECT  /*+ cardinality (10) */ max( calculate_epoch (s.time_id))  epoch
FROM sales s
 WHERE s.amount_sold> 100 and
       calculate_epoch (s.time_id) between 1000000000 and 1100000000;

     EPOCH
----------
1009756800

Elapsed: 00:00:01.39
```

另一个选项是预先存储函数执行的结果，从而避免在查询中执行函数。你可以通过使用基于函数的索引来实现这一点。在清单 1-12 中，创建了一个基于函数 `calculate_epoch` 的函数索引。SQL 语句的性能从 1.39 秒提升到了 0.06 秒。

***清单 1-12.** 使用函数索引的昂贵函数调用*

```sql
CREATE INDEX compute_epoch_fbi ON sales
(calculate_epoch(time_id))
Parallel (degree 4);

SELECT  /*+ cardinality (10) */ max( calculate_epoch (s.time_id))  epoch
FROM sales s
 WHERE s.amount_sold> 100 and
       calculate_epoch (s.time_id) between 1000000000 and 1100000000;
     EPOCH
----------
1009756800

Elapsed: 00:00:00.06
```

你也应该理解函数索引是有成本的。`INSERT`语句和`UPDATE`语句（更新`time_id`列时）将产生调用函数和维护索引的成本。需要仔细权衡 DML 操作中函数执行的成本与`SELECT`语句中函数执行的成本，以选择成本更低的方案。

![images](img/square.jpg) **注意** 从 Oracle Database 11g 版本开始，你可以创建一个虚拟列，然后在该虚拟列上创建索引。索引虚拟列的效果与函数索引相同。虚拟列相对于函数索引的一个优势是，你可以使用虚拟列对表进行分区，而仅使用函数索引是无法做到这一点的。

从 Oracle Database 11g 版本开始可用的`result_cache`函数，是调优昂贵 PL/SQL 函数执行的另一个选项。函数执行的结果会被记录在实例的共享全局区（SGA）中分配的结果缓存中。使用相同参数重复执行函数时，将直接从函数缓存中获取结果，而无需重复执行函数。清单 1-13 展示了利用`result_cache`来提升性能的函数示例：SQL 语句在 0.81 秒内完成。

***清单 1-13.** 使用 Result_cache 的函数*

```sql
DROP INDEX compute_epoch_fbi;
CREATE OR REPLACE FUNCTION calculate_epoch (d in date)
 RETURN NUMBER  DETERMINISTIC RESULT_CACHE IS
 l_epoch number;
BEGIN
 l_epoch := (d - TO_DATE('01-JAN-1970 00:00:00', 'DD-MON-YYYY HH24:MI:SS'))
          * 24 *60 *60 ;
 RETURN l_epoch;
END calculate_epoch;
/

SELECT  /*+ cardinality (10) */ max( calculate_epoch (s.time_id))  epoch
FROM sales s
 WHERE s.amount_sold> 100 and
       calculate_epoch (s.time_id) between 1000000000 and 1100000000;

     EPOCH
----------
1009756800

Elapsed: 00:00:00.81
```

总之，过度的函数执行会导致性能问题。如果你无法减少或消除函数执行，可以考虑采用函数索引或`result_cache`作为短期解决方案，以最小化函数调用带来的影响。

### 数据库链接调用

过多的数据库链接调用会影响应用程序性能。在循环中通过数据库链接访问远程表或修改远程表并不是一种可扩展的方法。对于每次访问远程表，数据库链接所涉及的数据库之间会交换多个 SQL*Net 数据包。如果数据库位于地理位置上分离的数据中心，或者更糟，遍布全球，那么对 SQL*Net 流量的等待将导致程序性能问题。

在清单 1-14 中，对于游标返回的每一行，都访问了远程数据库中的`customer`表。假设网络往返调用需要 100 毫秒，那么 100 万次往返调用大约需要 27 小时才能完成。位于国内不同地区的数据库之间 100 毫秒的响应时间并不少见。

***清单 1-14.** 过多的数据库链接调用*

```sql
DECLARE
  V_customer_name VARCHAR2(32);
BEGIN
  ...
  FOR c1 IN (SELECT …)
  LOOP
    ...
    SELECT customer_name
    INTO v_customer_name
    FROM customers@remotedb
    WHERE account_id = c1.account_id;
    ...
  END LOOP;
END;
```

明智地使用物化视图可以减少程序执行期间的网络往返调用。对于清单 1-14 的情况，可以将`customer`表创建为物化视图。在程序执行前刷新物化视图，并在程序中访问该物化视图。将表物化在本地减少了 SQL*Net 往返调用的次数。当然，作为应用程序设计者，你需要比较将整张表物化的成本与在循环中访问远程表的成本，并选择最优解决方案。

将程序重写为与远程表连接的 SQL 语句是另一个选项。Oracle Database 中的查询优化器可以优化此类语句，以减少 SQL*Net 往返的开销。要使用此技术，你应该重写程序，使得 SQL 语句只执行一次，很可能不在循环中执行。

将数据物化在本地或重写为带有远程连接的 SQL 语句，是调优清单 1-14 程序的初步步骤。然而，如果你连这些都无法做到，还有一个变通方法。作为临时措施，你可以将程序转换为使用多进程架构。例如，进程#1 处理 1 到 100,000 号客户，进程#2 处理 100,001 到 200,000 号客户，以此类推。通过创建 10 个进程将此逻辑应用到示例程序中，你可以将程序的总运行时间减少到大约 2.7 小时。使用`DBMS_PARALLEL_EXECUTE`是考虑将代码拆分为并行处理的另一个选项。



### 过度使用触发器

触发器通常用 PL/SQL 编写，但你也可以用 Java 编写触发器代码。从性能角度看，过度使用触发器并不理想。行变更在 SQL 引擎中执行，而触发器在 PL/SQL 引擎中执行。你又一次遇到了可怕的**上下文切换**问题。

在某些情况下，触发器是不可避免的。例如，无法避免触发器中的复杂业务验证。在这些场景中，你应该将那种复杂的验证逻辑用 PL/SQL 代码编写。应避免对简单的验证过度使用触发器。例如，使用检查约束（check constraints）而不是触发器来检查列的有效值列表。

此外，避免对同一触发操作使用多个触发器。与其为同一操作编写两个不同的触发器，不如将它们合并为一个，以最小化上下文切换的次数。

### 过度提交

在 PL/SQL 循环中，每插入、修改（或删除）一行后都进行提交，这种情况并不少见。在每行后都提交的编码习惯会导致程序执行变慢。频繁提交会产生更多重做日志（redo），要求日志写入器（Log Writer）频繁地将日志缓冲区内容刷新到日志文件，可能导致数据完整性问题，并消耗更多资源。PL/SQL 引擎经过优化以减少频繁提交的影响，但在减少提交方面，编写良好的代码是无可替代的。

你应该只在业务事务完成时提交。如果你在业务事务边界之前提交，可能会遇到数据完整性问题。如果你必须提交以提高可重启性，请考虑批量提交。例如，与其每行都提交，不如每 1000 或 5000 行进行一次批量提交（具体批处理大小的选择取决于你的应用）。更少的提交将减少程序的运行时间。此外，应用提交次数减少也将提升数据库的性能。

### 过度解析

不要在 PL/SQL 循环中使用动态 SQL 语句，因为这样做会导致过度解析问题。相反，应通过使用绑定变量来减少硬解析的数量。

在清单 1-15 中，访问 `customers` 表以检索客户详情，通过游标 `c1` 传递 `cust_id`。构造了一条带有字面值的 SQL 语句，然后使用原生动态 SQL 的 `EXECUTE IMMEDIATE` 结构来执行。问题在于，对于从游标 `c1` 检索到的每一行唯一值，都会构造一条新的 SQL 语句并发送到 SQL 引擎执行。

执行时不存在于共享池中的语句会引发硬解析。过度的硬解析会给库缓存（library cache）带来压力，从而降低应用的可扩展性和并发性。随着从游标 `c1` 返回的行数增加，硬解析的数量会线性增长。这个程序在开发数据库中处理少量行时可能有效，但在生产环境中，这种方法很可能成为问题。

`清单 1-15. 过度解析`

```sql
DECLARE
  ...
BEGIN
 FOR c1_rec IN c1
 LOOP
  -- 查询客户详情
  EXECUTE IMMEDIATE
   'SELECT cust_first_name, cust_last_name, country_id
    FROM customers
    WHERE cust_id= ' || c1_rec.cust_id  INTO l_cust_first_name, l_cust_last_name,
l_country_id;
  ...
 END LOOP;
COMMIT;
END;
/
```

你应该尽可能减少硬解析。循环中的动态 SQL 语句往往会增加硬解析的影响，如果执行的并发度增加，这种影响会被放大。

### 小结

本章回顾了使用某些 PL/SQL 结构不当的各种情况。请记住，SQL 是一种集合语言，而 PL/SQL 是一种过程化语言，在设计程序时，应考虑以下建议作为指导原则：

*   使用 SQL 解决查询问题。**以集合的方式思考！** 调优用 SQL 编写的查询比调优（比如）具有嵌套循环的 PL/SQL 程序更容易，后者本质上是以逐行处理的方式执行查询。
*   如果你必须用 PL/SQL 编写程序，尽量将工作**卸载到 SQL 引擎**。随着 Exadata 等新技术的出现，这一点变得越来越重要。Exadata 数据库一体机提供的智能扫描（Smart Scan）功能可以将工作卸载到存储节点，从而提高用 SQL 编写的程序的性能。PL/SQL 结构无法从 Exadata 数据库一体机中获得此类益处（至少截至 11gR2 版本）。
*   如果你必须使用基于循环的处理，请使用 PL/SQL 中提供的**批量处理**机制。通过本章讨论的技术，减少 PL/SQL 中不必要的工作，例如不必要的函数执行或对 `DUAL` 表的过度访问。
*   仅在最后手段时才使用基于单行、循环的处理。

确实，用 PL/SQL 处理你所有的数据和业务逻辑。用 Java 或其他语言处理表示逻辑和用户验证。通过本章概述的技术，你可以编写出高度可扩展的 PL/SQL 语言程序。

## 第 2 章



