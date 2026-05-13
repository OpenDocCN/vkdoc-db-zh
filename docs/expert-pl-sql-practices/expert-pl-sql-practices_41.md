# REF 游标简述

尽管本章还有另外两个更全面介绍 REF 游标的章节（“静态 REF 游标”和“动态 REF 游标”），但由于它们的结构——必须显式地打开、提取和关闭——值得在本节简要概述一下。如果你要向客户端返回一个结果集，使用 REF 游标是避免大量重复代码的好方法。本质上，你打开哪个游标通常取决于从请求客户端接收到的输入。清单 10-4 提供了一个 PL/SQL 中 REF 游标场景的简要示例：

**清单 10-4.** REF 游标动态工作，但需声明

```sql
DECLARE
   prod_cursor sys_refcursor;
BEGIN
IF ( input_param = 'C' )
THEN
      OPEN prod_cursor FOR
         SELECT * FROM prod_concepts
          WHERE concept_type = 'COLLATERAL'
            AND concept_dt   < TO_DATE( '01-JAN-2003', 'DD-MON-YYYY');
ELSE
      OPEN prod_cursor FOR
         SELECT * FROM prod_concepts
          WHERE concept_category = 'ADVERTISING';
END IF;
LOOP
       FETCH prod_cursor BULK COLLECT INTO .... LIMIT 500;
       ...在这里处理结果的程序化代码...
       EXIT WHEN prod_cursor%NOTFOUND;
END LOOP;
   CLOSE prod_cursor;
END;
```

清单 10-4 说明了 REF 游标如何根据从客户端接收到的输入动态打开。在清单 10-4 所示的示例中，如果接收到的输入是 ‘C’，则打开一个游标。但是，如果输入不是 ‘C’，则会打开一个完全不同的游标。只有在打开正确的游标后，才会从中提取数据并关闭它。这之所以可行，是因为游标变量的名称不会改变——改变的只是其内容。

需要注意的是，此 REF 游标是一个*静态* REF 游标的例子。它被称为静态 REF 游标，因为它与一个在编译时已知的查询相关联，这与*动态* REF 游标相反，后者的查询直到运行时才知晓。与静态 REF 游标关联的查询是*静态*的（永不改变）。使用 REF 游标时，如果可能，应使用静态 REF 游标。任何类型的动态代码都会破坏依赖关系，因为你需要等到运行时才能发现

*   你将要处理的列的数量或类型，或者
*   你将要处理的绑定变量的数量或类型。

如果你在创建 REF 游标之前*确实*知道这两件事，那么你应该选择创建静态 REF 游标。静态代码几乎总是优于动态代码，因为它在编译时对 RDBMS 是已知的，因此编译器可以为你捕获许多错误。动态代码在运行时才被 RDBMS 所知，因此可能不可预测且容易出错。

REF 游标将在后续章节中详细阐述。目前，知道 REF 游标是使用显式游标的又一个理由就足够了。实际上，只有少数几种情况下显式游标会比隐式游标成为更有用的编程选择。当执行无法使用 SQL 完成的批量处理或向客户端返回数据集时，你需要使用显式游标。在几乎所有其他情况下，在不得不使用显式游标之前，你应该尝试使用 SQL 或隐式 PL/SQL 游标。



### 隐式游标

与显式游标不同，隐式游标在使用前无需声明（当然，如果你愿意，也可以声明，稍后你会看到声明隐式游标的一个示例）。此外，隐式游标也无需显式地打开、提取和关闭。它们不使用 `open, fetch, close` 语法，而是使用 `for … in` 语法或简单的 `select ... into` 语法。一个典型的隐式游标示例如清单 10-5 所示。

**清单 10-5.** 用于仅获取一个值的隐式游标

```sql
CREATE FUNCTION f_get_name (ip_emp_id IN NUMBER) RETURN VARCHAR2
AS
lv_ename emp.ename%TYPE;
BEGIN
   SELECT ename INTO lv_ename FROM emp WHERE emp_id = f_get_name.ip_emp_id;
   RETURN lv_ename;
END;
```

你可能认出了这里询问的业务问题。它与清单 10-2 中询问的业务问题相同。两者的主要区别在于，显然，清单 10-5 中的解决方案采用了隐式游标。当你的目标仅仅是获取一个值时，应始终优先考虑使用隐式游标。将这段代码的简洁性与清单 10-2 中显式游标示例的冗长和易错性进行比较。在这个隐式游标示例中，没有对 `SQL%FOUND` 或 `SQL%NOTFOUND` 进行检查。代码非常清晰。这是一个简单的 `select ... into` 游标。如果只选择了一个名字，那么就返回一个名字。可以编写一个异常处理部分来处理两种情况：一是输入的员工 ID 没有对应的姓名，二是由于数据错误存在多个姓名。

![images](img/square.jpg) **注意** 请注意，到目前为止的示例都省略了异常处理部分。这是有意为之。如果你的代码中存在错误，并且你没有异常处理部分，Oracle 将会显示你的 PL/SQL 代码中发生错误的确切行号。如果有了异常处理部分，显示的错误行将是你的异常处理部分开始的地方。如果你正在向用户提供传递到堆栈的指令，以便用户在与你的程序交互时采取不同的操作，这可能是有用的。然而，如果错误信息是给你和其他程序员的，那么指出发生错误的确切行号对于你能够快速有效地进行调试至关重要。

当你提出的问题最多只有一个答案时，根本不需要 `for … in` 游标。在这种情况下，使用简单的 SQL `SELECT` 语句来检索一个可能包含零条或一条记录需要处理的答案。清单 10-5 假设所提出的业务问题永远不会返回多条记录。如果是这种情况，简单的 SQL `select … into` 语句是实现该解决方案的最佳编程选择。如果存在多条记录，你的程序将返回一个错误。这是一件好事。你需要知道数据是否有误。同样，如果不存在任何记录，你的程序也会返回一个错误。你的程序期望有且仅有一个答案——任何其他结果都是错误，你的程序会立即失败。使用显式游标通常不会有第二次提取（如清单 10-2 中描述的）来测试是否存在更多行，那将是一个错误。而 `select … into` 版本是“无错误的”。

#### 隐式游标的剖析

如前所述，隐式游标可以采用两种形式之一。它可以使用 FOR LOOP 实现，或者更具体地说，使用 `for ... in` 形式实现；也可以使用简单的 SQL `select … into` 语法实现。隐式游标不需要你显式打开它。当使用 `for … in` 形式实现隐式游标时，请牢记以下几点：

*   一旦你的隐式游标进入其 for 循环结构，游标即被初始化，结果集也被确定。
*   `IN` 步骤与“显式游标的剖析”一节中概述的 `FETCH` 步骤非常相似，但有一个关键区别。使用显式游标时，如果你想确保没有更多行需要处理，你需要添加类似下面的语法（示例见清单 10-3 和 10-4）：
    ```sql
    EXIT WHEN cursor%NOTFOUND;
    ```
    这种额外的语法对于隐式游标不是必需的。它们不需要检查游标是否有更多数据要处理来知道何时退出。
*   同样，`CLOSE` 语法对于隐式游标也不是必需的，因为它会在 `FOR` 循环终止后自动退出。

`select … into` 游标的行为方式与 `for … in` 游标不同。清单 10-6 说明了这两种实现之间的区别。

将每个 `BEGIN … END` 匿名块与紧随其后的 `DECLARE … BEGIN … END` 匿名块进行比较。

**清单 10-6.** 一个 Select … Into 游标与一个 For … In 游标

```sql
BEGIN
   FOR x IN (SELECT * FROM dual) LOOP ... END LOOP;
END;
```

```sql
DECLARE
   CURSOR c IS SELECT * FROM dual;
BEGIN
   OPEN c;
   LOOP
      FETCH c INTO …
      EXIT WHEN c%NOTFOUND;
       …
   END LOOP;
   CLOSE c;
END;
```

```sql
BEGIN
   SELECT *INTO ...FROM dual;
END;
```

```sql
DECLARE
   CURSOR c IS SELECT * FROM dual;
   l_rec dual%ROWTYPE;
BEGIN
   OPEN c;
   FETCH c INTO l_rec;
      IF (SQL%NOTFOUND)
      THEN
         RAISE NO_DATA_FOUND;
      END IF;
   FETCH c INTO l_rec;
      IF (SQL%FOUND)
      THEN
         RAISE TOO_MANY_ROWS;
      END IF;
   CLOSE c;
END;
```

从概念上讲，`select … into` 就像清单中所示的两个过程块。然而，隐式游标的 `for … in` 处理执行的是数组提取（从 Oracle10g 开始），因此实际上更像是批量提取和收集，使其比对应的显式游标高效得多。

`select … into` 游标使用不同的关键字，应在你期望最多处理零条到一条记录时实现。而 `for … in` 游标应在你期望处理多条记录时实现。一旦你 `SELECT … INTO` 某个 PL/SQL 变量，游标就会随着 `SELECT` 关键字被初始化，它*隐式地* `INTO` PL/SQL 变量进行*提取*，然后由于没有更多数据需要检索而*自动关闭*。




### 隐式游标与额外获取理论

一个常被讨论的、关于为何应避免使用隐式游标（尽管多年来已被证伪，但我仍会在奇怪的公司政策数据库编程指南中看到它被提及）的理论是：隐式游标通过执行一次额外的行获取来测试行是否存在，从而进行了无用功。该理论已在多本（为保持匿名）关于 Oracle RDBMS 的独立教育著作中提出。该理论的问题在于，它很少有实际证据支持。从 Oracle 7.1 版本开始，测试这个理论在任何版本上都相当容易。事实上，挑战任何被写入公司数据库编程政策的命令，应该是你的道德责任。如果它被陈述出来，那么它就应该是可证明的。如果它无法被证明，那么你就应该质疑它是否应该继续作为一条有效的命令。

使用你选择的任何隐式游标来测试这个理论都相当简单。出于说明目的，你可能想尝试一个更普遍可用的案例，比如从 Oracle 数据字典表中进行一个简单的 `SELECT` 查询来测试这个理论。代码清单 10-7 概述了一个简单的测试案例：

**代码清单 10-7.** 一个简单的隐式游标获取测试案例

```sql
SQL> DECLARE
  2      v_test NUMBER := 0;
  3  BEGIN
  4      SELECT user_id INTO v_test FROM all_users WHERE username = 'SYS';
  5  END;
  6   /
PL/SQL 过程已成功完成。
```

这是一个易于执行的测试，因为它不包含任何循环结构。预期至少且最多返回一个值。如果 `SELECT` 语句需要执行一次额外的获取，那么在简单的 SQL 跟踪会话的 TKPROF 输出中应该可以看到这次额外获取。代码清单 10-8 展示了代码清单 10-7 中测试案例的 TKPROF 输出。

**代码清单 10-8.** 隐式游标额外获取测试的 TKPROF 输出（11.2.0.1）

```
********************************************************************************

SQL ID: bs5v1gp6hakdp
Plan Hash: 3123615307
SELECT USER_ID
FROM
 ALL_USERS WHERE USERNAME = 'SYS'

call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          0          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch        1      0.00       0.00          0          6          0           1
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        3      0.00       0.00          0          6          0           1

Misses in library cache during parse: 1
Optimizer mode: ALL_ROWS
Parsing user id: 84     (recursive depth: 1)

Rows     Row Source Operation
-------  ---------------------------------------------------
      1  NESTED LOOPS  (cr=6 pr=0 pw=0 time=0 us cost=3 size=32 card=1)
      1   NESTED LOOPS  (cr=4 pr=0 pw=0 time=0 us cost=2 size=29 card=1)
      1    TABLE ACCESS BY INDEX ROWID USER$ (cr=2 pr=0 pw=0 time=0 us cost=1 size=26 card=1)
      1     INDEX UNIQUE SCAN I_USER1 (cr=1 pr=0 pw=0 time=0 us cost=0 size=0 card=1)(object id 46)
      1    TABLE ACCESS CLUSTER TS$ (cr=2 pr=0 pw=0 time=0 us cost=1 size=3 card=1)
      1     INDEX UNIQUE SCAN I_TS# (cr=1 pr=0 pw=0 time=0 us cost=0 size=0 card=1)(object id 7)
      1   TABLE ACCESS CLUSTER TS$ (cr=2 pr=0 pw=0 time=0 us cost=1 size=3 card=1)
      1    INDEX UNIQUE SCAN I_TS# (cr=1 pr=0 pw=0 time=0 us cost=0 size=0 card=1)(object id 7)

********************************************************************************
```

正如你从 TKPROF 输出中所看到的，并没有执行额外的获取。因此，我只能想到少数几个理由，说明你为什么不想尽一切努力确保使用隐式游标而非显式游标。如“显式游标和批量处理”小节中所述，当需要对从数据库获取的数据执行 `FORALL` 批量处理时，你将需要使用显式游标。换句话说，如果你只是选择数据并使用 `htp.p` 在 APEX 中输出数据，或者使用 `utl_file.put_line` 将所选数据发送到文件等，则无需使用显式游标进行批量处理。在这些情况下，Oracle RDBMS 在后台*已经*在进行批量处理了。

只有当你将数据取出，在过程中进行处理，然后使用 `FORALL` 更新、插入和/或删除将其放回数据库时，你才需要显式游标处理。在同一小节中，REF 游标被简要说明为另一个使用显式游标的原因——如果你的需求是将数据传递给位于数据库之外但连接至数据库的客户端应用程序。接下来的两节将向你介绍可用的两种 REF 游标类型。



### 静态 REF 游标

正如“REF 游标简介”一节所述，如果你需要将结果集返回给客户端，REF 游标（也称为 *reference* 游标或*游标变量*）是实现这一目标的最佳方式。REF 游标通常依赖于客户端的输入来决定具体打开哪个游标，从而决定返回哪个结果集。鉴于向客户端返回数据时可能存在的未知因素，能够在运行时选择（此处无意双关）要执行的查询，为你的程序提供了极大的灵活性。这种能力也有助于你避免编写重复代码。你已在“REF 游标简介”一节中看到了一个静态 REF 游标的例子。清单 10-9 在此静态 REF 游标示例的基础上进行了扩展，加入了包参数规范：

**清单 10-9.** 在 PL/SQL 包中声明的静态 REF 游标 (11.2.0.1)

```sql
CREATE OR REPLACE PACKAGE product_pkg
AS
   -- 在包规范中，声明游标变量类型
   TYPE my_cv IS REF CURSOR;

   PROCEDURE get_concepts ( ip_input    IN  NUMBER,
                            op_rc      OUT  my_cv);

END product_pkg;

CREATE OR REPLACE PACKAGE BODY product_pkg
AS
   -- 在包体的存储过程中，使用包规范中声明的类型来声明游标变量

   PROCEDURE get_concepts (ip_input    IN  NUMBER,
                           op_rc      OUT  my_cv)
   IS
   BEGIN
      IF ( ip_input = 1 )
      THEN
         OPEN op_rc FOR
             SELECT * FROM prod_concepts
              WHERE concept_type  = 'COLLATERAL'
                AND concept_dt    < TO_DATE( '01-JAN-2003', 'DD-MON-YYYY');
      ELSE
         OPEN op_rc FOR
             SELECT * FROM prod_concepts
              WHERE concept_category = 'ADVERTISING';
      END IF;
   END get_concepts;

END product_pkg;
```

清单 10-9 中的包和包体展示了使用游标变量的代码。包规范声明了一个 REF 游标类型，如这行代码所示：

```sql
TYPE my_cv IS REF CURSOR;
```

请注意，这种游标变量是 `弱类型` 的。`弱类型` 游标变量在返回结果集方面为你提供了更大的灵活性。然而，因为它不像下面这样被创建为特定类型的记录

```sql
TYPE my_cv IS REF CURSOR RETURN prod_concepts%ROWTYPE;
```

或

```sql
TYPE ProdConTyp IS RECORD (
     prod_id                  NUMBER,
     con_id                   NUMBER,
     req_id                   NUMBER,
     concept_type       VARCHAR2(30),
     concept_category   VARCHAR2(30),
     concept_dt                DATE);
     TYPE my_cv IS REF CURSOR RETURN ProdConTyp;
```

所以它更容易出错。如果你发现自己不得不使用游标变量，应尽可能创建 `强类型` 的（如上所示），而不是 `弱类型` 的。包规范中声明的类型随后在包体中被引用，以声明实际的游标变量，如下列代码所示：

```sql
op_rc      OUT  my_cv)
```

从这一点开始，这个游标变量可以用于并重复用于任何你想要的结果集，如下面的代码所示，该代码打开 REF 游标并将其作为存储过程 `get_concepts` 中的输出参数返回：

```sql
IF ( ip_input = 1 )
THEN
   OPEN op_rc FOR
             SELECT * FROM prod_concepts
              WHERE concept_type = 'COLLATERAL'
                AND concept_dt   < TO_DATE( '01-JAN-2003', 'DD-MON-YYYY');
ELSE
          OPEN op_rc FOR
             SELECT * FROM prod_concepts
              WHERE concept_category = 'ADVERTISING';
END IF;
```

请注意，在此代码中，从 `PROD_CONCEPTS` 表返回的所有列取决于最终执行的 `WHERE` 子句。在这种情况下，最好对用于返回这两个结果集的游标变量进行 `强类型化`，因为它们包含相同的列列表。然而，为了说明目的，如果可能存在第三种结果，如下扩展 `IF` 语句

```sql
ELSIF ( ip_input = 2 )
THEN
   OPEN op_rc FOR
             SELECT prod_name FROM products p, prod_concepts pc
              WHERE p.prod_id    = pc.prod_id
                AND concept_type = 'COLLATERAL'
                AND concept_dt   < TO_DATE( '01-JAN-2003', 'DD-MON-YYYY');
```

那么这个结果集将与其他两个完全不同。在这个特定情况下，在所有条件相同的情况下，如示例所示的 `弱类型` 游标变量将是唯一适用于此场景的游标变量类型。换句话说，因为示例中选择的游标变量类型几乎可以接受任何类型的结果集，所以它很可能会成功。

但是，因为其结构在编译时对编译器是未知的，所以如果与此 REF 游标相关的故障发生，那将是在运行时。当你规避可能的编译器错误并允许它们（如果它们会发生的话）在运行时发生时，你总是要承担风险，因为这样的错误会影响你的用户。请记住，仅仅因为你*可以*使用某种特定类型的游标，并不意味着它是你最佳的选择。你的目标是尽可能高效地编写 PL/SQL。如果你正在使用 REF 游标，则假定你符合以下条件之一：

*   你正在将结果集传递给客户端，并且/或者
*   你直到运行时才知道查询会是什么，并且/或者
*   没有其他方法可用于实现你的目标，只能使用隐式游标。

使用 REF 游标时，有几点需要牢记。REF 游标属于在当前时间点处理/获取其记录的任何一方（无论是客户端程序还是 PL/SQL 程序）。因为 REF 游标是指向游标的指针，并且实际上可以指向许多不同的游标，所以它不会被缓存在 PL/SQL 中。语句缓存对于减少总的解析调用次数并因此提高应用程序可扩展性至关重要。重复解析相同的 SQL 会降低你的可扩展性并增加你的工作量。通过在 PL/SQL 中使用非 REF 游标，你可以大大减少解析。关于 PL/SQL 游标缓存的更多讨论见“关于解析的几句话”一节。

这里有一个注意事项：你可以通过适当设置初始化文件中 `session_cached_cursors` 参数的值来限制必然会发生的软解析工作量（或者更准确地说，是限制软解析执行的工作量，因为你无法限制软解析）。对于 REF 游标，PL/SQL 在发出请求并运行相关代码之前，不会知道游标将是什么。因此，由于使用 REF 游标的开销稍高，最好先看看是否可以通过使用常规游标来实现你的目标。

为什么这些要点很重要？因为使用游标变量有大量限制。如果你认为通过使用 REF 游标而不是非 REF 游标来达成目标是选择了简单的方法，那么实际上你选择了一个长期管理起来会困难得多的方法。

