# 引言

我很少有机会为一本我参与编写的书作介绍。通常，我很满足于自己身处幕后——这确实是图书编辑应有的位置。但这次我破例了，因为本书的内容唤起了我作为开发者在往昔岁月中的许多回忆。

`Expert PL/SQL Practices` 这本书讲述的是如何有效地运用 PL/SQL。它不是一本关于语法的书，而是一本关于如何将语法和特性与良好的开发实践相结合，从而创建出可靠、可扩展且能长期维护的应用程序的书。

对于任何工具，首先需要了解的一点就是何时使用它。Riyaj Shamsudeen 在他的开篇章节 `Do Not Use!` 中巧妙地探讨了何时该使用 PL/SQL 的问题。我把这一章放在本书开头，是基于我个人的经历：我最成功的一次性能优化案例发生在 1990 年代末，当时我用一条 SQL 语句取代了客户端 PC 上的一堆过程化代码，将一个耗时 36 小时以上的任务缩短到只需几分钟。PL/SQL 并非罪魁祸首，但我从中学到的教训是：当可能时，基于集合的方法——通常比编写过程化代码更可取。

Michael Rosenblum 紧接着撰写了一章关于动态 SQL 的精彩内容，展示了当你直到运行时才知道 SQL 语句时该如何编写代码。这让我想起了 1990 年代初在陶氏化学公司的一段时光，当时我使用 Rdb 的扩展动态游标功能集为一个医疗记录系统编写数据加载应用程序。我仍然记得，那是我开发过的最有趣的应用程序之一。

Dominic Delmolino 探讨了如何使用 PL/SQL 进行并行处理。他涵盖了你能获得的益处以及适合并行处理的工作负载类型。只是请务必小心，好吗？我作为 DBA 犯过的最大错误之一，就是有一次未经深思熟虑就为一个关键应用表设置了并行度，只为了让一个报表跑得更快。结果仿佛我的电话和回车键连在了一起，因为我改动后不到一分钟，电话就响了。电话那头的经理非常不满。不用说，我从那时起就意识到，实施并行处理值得比我之前投入的多那么一点点的思考。Dominic 的章节将帮助你避免类似的窘境。

书中有几章涉及代码规范和良好的编程实践。Stephan Petit 提供了一套有用的命名和编码约定。Torben Holm 讲解了 PL/SQL 警告和条件编译。Lewis Cunningham 呈现了一章发人深省的内容，关于代码分析，以及真正理解你所编写代码及其使用方式的重要性。Robyn Sands 在她关于演进式数据建模的章节中，帮助你思考灵活性与良好设计。Melanie Caffrey 则带你了解各种可用的游标类型，帮助你根据任何给定情况做出正确的游标选择。

其他章节涉及调试和故障排除。Sue Harper 介绍了 PL/SQL 单元测试，特别是现在已内置于 SQL Developer 中的相关功能集。（我记得当年我还在纸上编写单元测试脚本）。让你自己免受回归错误的尴尬吧。自动化单元测试让你能轻松方便地验证你在修复一个问题时，没有意外破坏两个新功能。

John Beresniewicz 接着撰写了一章关于契约式编程的内容。John 方法的一个关键部分是使用断言来验证在代码各个关键点处应为真的条件。我最早是在石器时代的 PowerBuilder 编程中了解到断言技术的。我一直很高兴看到 John 将这项技术推广到 PL/SQL 领域。

Arup Nanda 帮助你掌控依赖关系和失效问题。依赖性问题可能是应用程序出现看似随机、难以复现错误的一个根源。Arup 展示了如何控制那些不可避免发生的情况，这样你就不会被意外错误搞得措手不及。

我们几乎不可能将性能和可扩展性排除在讨论之外。Ron Crisco 讨论了如何分析代码以找到最大的优化机会。Adrian Billington 探讨了从 SQL 语句中调用 PL/SQL 时涉及的性能方面。Connor McDonald 介绍了通过批量 SQL 操作可获得的巨大性能优势。

一个不常被想到的、关于可扩展性的独特方面，是应用程序规模和开发人员数量的问题。PL/SQL 是否适合涉及数十、甚至数百名程序员的大规模开发？Martin Büchi 在他关于大规模 PL/SQL 编程的章节中，通过讲述他成功维护一个由超过 170 名开发人员维护的、代码量达 1100 万行的应用程序的经历，表明 PL/SQL 完全能够胜任这项任务。

你大概能感觉到我对这本书的兴奋之情。作者们都是顶尖高手。每人都撰写了自己充满热情且特别精通的 PL/SQL 方面。如果你已经过了学习语法的阶段，那么请坐下来，阅读这本书，提升你运用 PL/SQL 和 Oracle 数据库全部能力来交付应用程序的水平。

Jonathan Gennick
Apress 出版社助理编辑总监

## 第 1 章

### 请勿使用

**Riyaj Shamsudeen 著**

恭喜你购买了本书。PL/SQL 是你工具箱中的一件利器；然而，你应该明白，PL/SQL 的使用并不适合所有场景。本章将教你何时应该用 PL/SQL 编写应用程序，如何编写可扩展的代码，更重要的是，何时不应该用 PL/SQL 编写程序。滥用某些 PL/SQL 构造会导致代码无法扩展。在本章中，我将回顾各种误用 PL/SQL 导致不适用并造成应用程序不可扩展的案例。

**PL/SQL 与 SQL**

SQL 是一种集处理语言，如果语句是基于集思考方式编写的，则 SQL 语句的扩展性更好。PL/SQL 是一种过程化语言，SQL 语句可以嵌入到 PL/SQL 代码中。

SQL 语句在 SQL 执行器（更通常称为 SQL 引擎）中执行。PL/SQL 代码则由 PL/SQL 引擎执行。PL/SQL 的强大之处在于它能够将 PL/SQL 的过程化能力与 SQL 的集处理能力相结合。


### 逐行处理

在一个典型的逐行处理程序中，代码会打开一个游标，循环遍历从该游标中检出的行，并对这些行进行处理。这种基于循环的处理结构**极不推荐**，因为它会导致代码无法扩展。代码清单 1-1 展示了一个使用此结构的程序示例。

**`代码清单 1-1`.** 逐行处理

```sql
DECLARE
  CURSOR c1 IS
    SELECT prod_id, cust_id, time_id, amount_sold
    FROM sales
    WHERE amount_sold > 100;
  c1_rec c1%rowtype;
  l_cust_first_name customers.cust_first_name%TYPE;
  l_cust_lasT_name customers.cust_last_name%TYPE;
BEGIN
  FOR c1_rec IN c1
  LOOP
    -- 查询客户详情
    SELECT cust_first_name, cust_last_name
    INTO l_cust_first_name, l_cust_last_name
    FROM customers
    WHERE cust_id=c1_rec.cust_id;
    --
    -- 插入到目标表
    --
    INSERT INTO top_sales_customers (
      prod_id, cust_id, time_id, cust_first_name, cust_last_name,amount_sold
      )
      VALUES
      (
        c1_rec.prod_id,
        c1_rec.cust_id,
        c1_rec.time_id,
        l_cust_first_name,
        l_cust_last_name,
        c1_rec.amount_sold
      );
  END LOOP;
  COMMIT;
END;
/
PL/SQL 过程已成功完成。
已用时间: 00:00:10.93
```

在代码清单 1-1 中，程序声明了一个游标 `c1`，并使用游标 FOR 循环语法隐式地打开该游标。对于从游标 `c1` 检索到的每一行，程序都会查询 `customers` 表，将 `first_name` 和 `last_name` 填充到变量中。随后，一行数据被插入到 `top_sales_customers` 表中。

代码清单 1-1 所例示的编码实践存在问题。即使循环中调用的 SQL 语句经过高度优化，程序的执行也可能消耗大量时间。假设查询 `customers` 表的 SQL 语句耗时 0.1 秒，而 `INSERT` 语句耗时 0.1 秒，那么每次循环执行的总耗时就是 0.2 秒。如果游标 `c1` 检索了 100,000 行，那么该程序的总耗时将是 100,000 乘以 0.2 秒：20,000 秒，大约 5.5 小时。优化这种程序结构并不容易。Tom Kyte 将这种类型的处理称为 ***slow-by-slow processing***（慢速处理），原因显而易见。

![images](img/square.jpg) **注意** 本章示例使用的是 SH 示例模式，这是甲骨文公司提供的示例模式之一。要安装示例模式，可以使用甲骨文提供的软件。您可以从 [`http://download.oracle.com/otn/solaris/oracle11g/R2/solaris.sparc64_11gR2_examples.zip`](http://download.oracle.com/otn/solaris/oracle11g/R2/solaris.sparc64_11gR2_examples.zip) 下载适用于 11gR2 Solaris 平台的版本。安装说明请参阅解压后软件目录中的 Readme 文档。其他平台和版本的 Zip 文件也可从甲骨文网站获取。

代码清单 1-1 中的代码还有另一个固有问题。SQL 语句在循环中被从 PL/SQL 中调用，因此执行会在 PL/SQL 引擎和 SQL 引擎之间来回切换。这两个环境之间的这种切换被称为***context switch***（上下文切换）。上下文切换会增加程序的执行时间并引入不必要的 CPU 开销。您应该通过消除或减少这两个环境之间的切换来降低上下文切换的次数。

您通常应该避免逐行处理。更好的编码实践是将代码清单 1-1 中的程序改写成一条 SQL 语句。代码清单 1-2 重写了代码，完全避免了使用 PL/SQL。

**`代码清单 1-2`.** 重写的逐行处理

```sql
--
-- 插入到目标表
--
INSERT
INTO top_sales_customers
  (
    prod_id,
    cust_id,
    time_id,
    cust_first_name,
    cust_last_name,
    amount_sold
  )
SELECT s.prod_id,
  s.cust_id,
  s.time_id,
  c.cust_first_name,
  c.cust_last_name,
  s.amount_sold
FROM sales s,
     customers c
WHERE s.cust_id    = c.cust_id and
s.amount_sold> 100;
已创建 135669 行。
已用时间: 00:00:00.26
```

代码清单 1-2 中的代码，除了能解决逐行处理的缺点外，还有更多优势。可以使用并行执行来调整重写后的 SQL 语句。通过使用多个并行执行进程，您可以显著减少执行的耗时。此外，代码变得更加简洁和可读。

![images](img/square.jpg) **注意** 如果您将 PL/SQL 循环代码重写为连接语句，需要考虑重复记录的问题。如果 `customers` 表中存在相同 `cust_id` 列的重复记录，那么重写的 SQL 语句将检出比预期更多的行。然而，在这个特定的例子中，`customers` 表的 `cust_id` 列上存在主键，因此使用 `cust_id` 列上的等值谓词不会有重复数据的风险。


### 逐行嵌套处理

在 PL/SQL 语言中，您可以嵌套游标。一种常见的编码实践是：从一个游标中检索值，将这些值传递给另一个游标，再将第二级游标的值传递给第三级游标，以此类推。但是，如果游标嵌套层次很深，基于循环的代码的性能问题就会加剧。由于游标的嵌套，SQL 执行次数会急剧增加，导致程序运行时间延长。

在代码清单 1-3 中，游标`c1`、`c2`和`c3`是嵌套的。游标`c1`是顶级游标，从表`t1`中检索行；然后打开游标`c2`，传入来自游标`c1`的值；接着打开游标`c3`，传入来自游标`c2`的值。对于从游标`c3`检索到的每一行，都会执行一条`UPDATE`语句。即使`UPDATE`语句经过优化，执行只需 0.01 秒，深层嵌套的游标仍会导致程序性能下降。假设游标`c1`、`c2`和`c3`分别检索 20 行、50 行和 100 行数据。那么代码将循环处理 100,000 行，程序的总耗时将超过 1,000 秒。优化这类程序通常需要完全重写。

**代码清单 1-3.** 使用嵌套游标进行逐行处理

```sql
DECLARE
  CURSOR c1 AS
    SELECT n1 FROM t1;
  CURSOR c2 (p_n1) AS
    SELECT n1, n2 FROM t2 WHERE n1=p_n1;
  CURSOR c3 (p_n1, p_n2) AS
    SELECT text FROM t3 WHERE n1=p_n1 AND n2=p_n2;
BEGIN
  FOR c1_rec IN c1
  LOOP
    FOR c2_rec IN c2 (c1_rec.n1)
    LOOP
      FOR c3_rec IN c3(c2_rec.n1, c2_rec.n2)
      LOOP
        -- 在此处执行某些 SQL;
        UPDATE … SET ..where n1=c3_rec.n1 AND n2=c3_rec.n2;
      EXCEPTION
      WHEN no_data_found THEN
        INSERT into… END;
      END LOOP;
    END LOOP;
  END LOOP;
COMMIT;
END;
/
```

代码清单 1-3 中的代码还存在另一个问题，即执行了`UPDATE`语句。如果`UPDATE`语句引发了`no_data_found`异常，则会执行`INSERT`语句。使用`MERGE`语句，可以将这类处理从 PL/SQL 转移到 SQL 引擎中执行。

从概念上讲，代码清单 1-3 中的三个循环代表了表`t1`、`t2`和`t3`之间的等值连接。在代码清单 1-4 中，该逻辑被重写为一个带别名`t`的 SQL 语句。`UPDATE`和`INSERT`的组合逻辑被替换为一条`MERGE`语句。`MERGE`语法提供了在行存在时更新、不存在时插入的能力。

**代码清单 1-4.** 使用 MERGE 语句重写的逐行处理逻辑

```sql
MERGE INTO fact1 USING
(SELECT DISTINCT c3.n1,c3.n2
 FROM t1, t2, t3
 WHERE t1.n1      = t2.n1
 AND t2.n1        = t3.n1
 AND t2.n2        = t3.n2
) t
ON (fact1.n1=t.n1 AND fact1.n2=t.n2)
WHEN matched THEN
  UPDATE SET .. WHEN NOT matched THEN
  INSERT .. ;
  COMMIT;
```

请不要在 PL/SQL 语言中编写具有深层嵌套游标的代码。检查一下，看看是否可以改用 SQL 来编写此类代码。

### 查询（Lookup Queries）

查询通常用于填充变量或执行数据验证。在循环中执行查询会导致性能问题。

在代码清单 1-5 中，高亮显示的查询使用查询来获取`country_name`。对于游标`c1`返回的每一行，都会执行一次获取`country_name`的查询。随着从游标`c1`检索到的行数增加，查询的执行次数也会增加，导致代码性能低下。

**代码清单 1-5.** 查询示例，是代码清单 1-1 的修改版

```sql
DECLARE
  CURSOR c1 IS
    SELECT prod_id, cust_id, time_id, amount_sold
    FROM sales
    WHERE amount_sold > 100;
  l_cust_first_name customers.cust_first_name%TYPE;
  l_cust_last_name customers.cust_last_name%TYPE;
  l_Country_id countries.country_id%TYPE;
  l_country_name countries.country_name%TYPE;
BEGIN
FOR c1_rec IN c1
LOOP
  -- 查询客户详情
  SELECT cust_first_name, cust_last_name, country_id
  INTO l_cust_first_name, l_cust_last_name, l_country_id
  FROM customers
  WHERE cust_id=c1_rec.cust_id;

  -- 查询获取 country_name
  SELECT country_name
  INTO l_country_name
  FROM countries WHERE country_id=l_country_id;
  --
  -- 插入目标表
  --
  INSERT
  INTO top_sales_customers
    (
      prod_id, cust_id, time_id, cust_first_name,
      cust_last_name, amount_sold, country_name
    )
    VALUES
    (
      c1_rec.prod_id, c1_rec.cust_id, c1_rec.time_id, l_cust_first_name,
      l_cust_last_name, c1_rec.amount_sold, l_country_name
    );
END LOOP;
COMMIT;
END;
/
PL/SQL procedure successfully completed.
Elapsed: 00:00:16.18
```

代码清单 1-5 中的示例是简化的。获取`country_name`的查询可以重写为主游标`c1`本身的一个连接（join）。第一步，您应该将该查询修改为一个连接。不过，在现实世界的应用程序中，这种重写并不总是可行的。

如果您无法通过重写代码来减少查询的执行次数，那么还有另一个选择。您可以定义一个关联数组来缓存查询的结果，并在后续执行中重用该数组，从而有效减少查询的执行次数。

代码清单 1-6 说明了数组缓存技术。程序不再为游标`c1`的每一行都执行查询来获取`country_name`，而是将一个键值对（在本例中是(`country_id`, `country_name`)）存储在一个名为`l_country_names`的关联数组中。关联数组类似于索引，可以通过键值访问任何给定的值。

在执行查询之前，使用`EXISTS`运算符检查数组中是否存在与`country_id`键值匹配的元素。如果数组中存在该元素，则直接从该数组中获取`country_name`，而无需执行查询。如果不存在，则执行查询，并将新元素添加到数组中。

您还应该理解，此技术适用于键的唯一值较少的情况。在此示例中，由于`country_id`列的唯一值较少，查询的执行次数*可能*会大大降低。使用示例模式，查询的最大执行次数将为 23，因为`country_id`列只有 23 个不同的值。

**代码清单 1-6.** 使用关联数组的查询示例


