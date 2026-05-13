# Oracle 条件编译与对象失效

```sql
NAME          TYPE          SOURCE_SIZE PARSED_SIZE  CODE_SIZE ERROR_SIZE
------------- ------------- ----------- ----------- ---------- ----------
LEVEL2        PROCEDURE             530         917        478          0
```

与将 `TIMING_ON` 设置为 `FALSE` 进行编译的代码相比，`SOURCE_SIZE` 没有差异，但 `PARSED_SIZE` 和 `CODE_SIZE` 的值略有增加。这是由于调用了过程 `LOGIT`。

现在来看看原始 `LEVEL2` 过程的大小，这是将 `TIMING_ON` 设置为 `TRUE` 的情况。

```sql
SQL@V102> select * from USER_OBJECT_SIZE WHERE NAME = 'LEVEL2';
```

```sql
NAME          TYPE          SOURCE_SIZE PARSED_SIZE  CODE_SIZE ERROR_SIZE
------------- ------------- ----------- ----------- ---------- ----------
LEVEL2        PROCEDURE             730         917        497          0
```

`SOURCE_SIZE` 比未使用条件编译的 `LEVEL2` 过程要大，但 `PARSED_SIZE` 和 `CODE_SIZE` 与使用了条件编译的有限版本 `LEVEL2` 相同或几乎相同。这是因为在编译期间，条件编译代码被剥离了。因此，你可以看到，使用条件编译会导致在源代码中和编译期间（以及由依赖项触发的自动重新编译期间）占用更大的空间。如果 Oracle 执行自动重新编译，它会使用 `REUSE SETTINGS` 编译选项。

所以，不使用条件编译，你会得到：

```sql
SQL@V102> exec level2

Elapsed: 00:00:20.60
```

而使用条件编译（`TIMING_ON` 设置为 `TRUE`），你会得到：

```sql
SQL@V102> exec level2

Elapsed: 00:00:20.49
```

这里的差异只有零点几秒。这样短的时间是为换取条件编译的好处而付出的微不足道的代价。

## 失效

依赖于其他对象的对象，可能会在它们所依赖的对象发生变化或因某些原因失效时变得无效。让我们通过创建一个调用 `LEVEL2` 过程的 `LEVEL1` 过程，来稍微扩展一下这个想法。第一个例子运行在 Oracle 10.2 上。

```sql
SQL@V102> CREATE OR REPLACE PROCEDURE LEVEL1 AS
2  BEGIN
3    LEVEL2;
4  END;
5  /
Procedure created.
```

当你必须编译一个较低层级的过程时，如果存在依赖关系，Oracle 将使较高层级的过程失效，从而在下次调用较高层级过程时强制进行自动重新编译——当然，前提是它之前没有被手动编译过。

创建 `LEVEL1` 过程后，重新编译 `LEVEL2` 过程。

```sql
SQL@V102> ALTER PROCEDURE LEVEL2 COMPILE REUSE SETTINGS;

Procedure altered.
```

然后检查这些过程的 `STATUS`。

```sql
SQL@V102> SELECT OBJECT_NAME, STATUS FROM USER_OBJECTS WHERE OBJECT_NAME IN
 ('LEVEL1','LEVEL2');

OBJECT_NAME                    STATUS
------------------------------ -------
LEVEL1                         INVALID
LEVEL2                         VALID
```

执行 `LEVEL1` 过程将强制重新编译并验证该过程。

```sql
SQL@V102> exec LEVEL1

PL/SQL procedure successfully completed.

SQL@V102> SELECT OBJECT_NAME, STATUS FROM USER_OBJECTS WHERE OBJECT_NAME IN
 ('LEVEL1','LEVEL2');

OBJECT_NAME                    STATUS
------------------------------ -------
LEVEL1                         VALID
LEVEL2                         VALID
```

`LEVEL1` 过程已被动态重新编译，现在再次有效。这与条件编译无关——这只是它的工作方式。

> **注意**
> 在一个系统中有数百个存储过程相互调用，然后有意或间接地重新编译一个过程，可能是一场灾难。让我用一个真实的故事来强调这一点。在一家拥有数百并发用户的公司中，一次手动更新表的操作出了错误——一个列被设置成了错误的值。于是数据库管理员使用 `SELECT AS OF TIMESTAMP` 创建了该表的一个副本。到目前为止，一切正常。然后他重命名了原始表，并快速将副本的表重命名为原始名称。嗯，Oracle 会注意到这种名称交换，如果存在任何依赖关系的话。在这个案例中，该表被一个低层级过程引用，该过程变为无效，导致所有依赖于此过程的所有过程也变为无效。这使得几乎所有的用户会话都排队等待重新编译代码。由于没有人能在系统上工作，数据库不得不重启。重启后，系统照常工作。

在 Oracle 11g 中，Oracle 放宽了对依赖对象失效的规则。如果你不更改过程参数的足迹，Oracle 不会强制使依赖对象失效，无论它遇到的是包中的过程还是独立过程。如果你在 Oracle 11g 而不是 Oracle 10g 中重新运行前面涉及 `LEVEL1` 和 `LEVEL2` 的例子，结果将如下所示：

```sql
SQL@V112> SELECT OBJECT_NAME, STATUS FROM USER_OBJECTS WHERE OBJECT_NAME IN
 ('LEVEL1','LEVEL2');

OBJECT_NAME                    STATUS
------------------------------ -------
LEVEL1                         VALID
LEVEL2                         VALID

SQL@V112> ALTER PROCEDURE LEVEL2 COMPILE REUSE SETTINGS;

Procedure altered.

SQL@V112> SELECT OBJECT_NAME, STATUS FROM USER_OBJECTS WHERE OBJECT_NAME IN
 ('LEVEL1','LEVEL2');

OBJECT_NAME                    STATUS
------------------------------ -------
LEVEL1                         VALID
LEVEL2                         VALID

SQL@V112>
```

注意，两个过程最终都是有效的。因为你在 Oracle 11g 中，并且因为你的过程足迹没有改变，Oracle 不会强制重新编译。


#### 控制编译

为每个过程设置不同的 `plsql_ccflags`，从而产生大量编译器标志，可能会令人困惑。即使你可以执行 `ALTER PROCEDURE <过程名> COMPILE REUSE SETTINGS` 命令，但在某个时候，你可能会在未设置适当编译器标志的情况下意外编译了一个过程。并且，如果你没有在条件编译代码中检查是否设置了某个 `ccflag`，你的代码可能会以“错误”的设置进行编译。避免这种情况的一个方法是构建一个小型框架来控制重新编译，如下例所示：

```
SQL@V112> CREATE TABLE CC_FLAGS (
2  OWNER        VARCHAR2(30),
3  OBJECT_TYPE  VARCHAR2(30),
4  OBJECT_NAME  VARCHAR2(30),
5  CCFLAGS      VARCHAR2(4000));

表已创建。
```

`添加包含每个过程信息的行。`

```
SQL@V112> INSERT INTO CC_FLAGS (OWNER, OBJECT_TYPE, OBJECT_NAME, CCFLAGS) VALUES('ACME' ,
![images](img/U003.jpg)
 'PROCEDURE', 'LEVEL2', 'TIMING_ON:FALSE');

已创建 1 行。
```

```
SQL@V112> INSERT INTO CC_FLAGS (OWNER, OBJECT_TYPE, OBJECT_NAME, CCFLAGS) VALUES('ACME' ,
![images](img/U003.jpg)
'PROCEDURE', 'LEVEL1', '');

已创建 1 行。
```

```
SQL@V112> COMMIT;

提交完成。
```

```
SQL@V112> BEGIN
2  FOR I IN (SELECT * FROM CC_FLAGS) LOOP
3   EXECUTE IMMEDIATE 'ALTER ' ||I.OBJECT_TYPE||' '||I.OWNER||'.'||I.OBJECT_NAME||' COMPILE PLSQL_CCFLAGS='''||I.CCFLAGS||'''';
4 END LOOP;
5 END;
6 /

PL/SQL 过程已成功完成。
```

因此，运行完这个匿名 PL/SQL 块后，过程将按照它们各自的设置进行编译。这或许可以（或应该）是一个存储过程，甚至可以带参数来控制编译一个还是全部；如果是这样，不要把这个过程放在 `CC_FLAGS` 表中，否则当正在运行的过程尝试编译自身时，你会陷入死锁局面。

手动编译带有特殊 `plsql_ccflags` 的存储过程如下例所示：

```
SQL@V112> ALTER PROCEDURE SPECIAL1 COMPILE PLSQL_CCFLAGS='SPECIAL_ONE:42';

过程已变更。
```

在你的开发环境中，你可以使用这个框架，并从中生成部署脚本。如果某些存储过程在部署后需要使用特殊标志进行编译，比如说你需要调试某个过程，可以按所示的方法完成。

在创建部署脚本时，我会确保在添加存储过程之前设置好 `PLSQL_CCFLAGS`、`PLSQL_WARNINGS`；如果存在不同的标志和警告，我会在每个存储过程之前进行设置，如下列代码所示：

```
…
ALTER SESSION SET PLSQL_CCFLAGS='TIMING_ON:FLASE';
ALTER SESSION SET PLSQL_WARNINGS='ENABLE:SEVERE,ENABLE_PERFORMANCE,ERROR:7204';
@@ GET_T1_KEY
ALTER SESSION SET PLSQL_WARNINGS='ENABLE:SEVERE,ENABLE:PERFORMANCE';
@@ SET_T1_KEY
…
```

#### 查询变量

到目前为止，我一直在讨论 `plsql_ccflags` 参数和条件编译。你可以在不设置 `plsql_ccflags` 参数的情况下使用条件编译。方法是通过查询你自己的包常量或 Oracle 定义的常量。让我们通过一个例子来演示。

首先，你创建一个 `PACKAGE` 规范，其中包含一个名为 `TIMING_ON` 的常量，其值设置为 `FALSE`。

```
SQL@V112> CREATE OR REPLACE PACKAGE TIMING AS
2     TIMING_ON CONSTANT BOOLEAN := FALSE;
3  END;
4  /
```

然后你重新创建引用该 `PACKAGE` 常量的存储过程。

`TIMING.TIMING_ON`

```
SQL@V112> CREATE OR REPLACE PROCEDURE LEVEL2 AS
2  $IF TIMING.TIMING_ON $THEN
3     L_START_TIME TIMESTAMP;
4  $END
5  BEGIN
6  DBMS_APPLICATION_INFO.SET_MODULE($$PLSQL_UNIT ,NULL);
7  $IF TIMING.TIMING_ON $THEN
8     L_START_TIME:=CURRENT_TIMESTAMP;
9  $END
10  -- -----------------------------------------------
11  -- 应用逻辑已省略
12  -- -----------------------------------------------
13  $IF TIMING.TIMING_ON $THEN
14    LOGIT (P_PG=>$$PLSQL_UNIT, P_START_TIME=>L_START_TIME, P_STOP_TIME=>CURRENT_TIMESTAMP);
15  $END
16  DBMS_APPLICATION_INFO.SET_MODULE(NULL,NULL);
17  END LEVEL2;
18  /

过程已创建。
```

调用 `DBMS_PREPROCESSOR.PRINT_POST_PROCESSED_SOURCE` 显示，代码已经过预处理，且 `TIMING_ON = FALSE`，符合预期。

```
SQL@V112> CALL DBMS_PREPROCESSOR.PRINT_POST_PROCESSED_SOURCE('PROCEDURE', 'SCOTT','LEVEL2');
PROCEDURE LEVEL2 AS
BEGIN
DBMS_APPLICATION_INFO.SET_MODULE( 'LEVEL2'    ,NULL);
-- -----------------------------------------------
-- 应用逻辑已省略
-- -----------------------------------------------
DBMS_APPLICATION_INFO.SET_MODULE(NULL,NULL);
END LEVEL2;

调用完成。
```

将 `PACKAGE` 常量改为 `TRUE` 会使所有引用此包的存储过程变为无效状态。因此，它们都需要重新编译。

```
SQL@V112> CREATE OR REPLACE PACKAGE TIMING AS
2     TIMING_ON CONSTANT BOOLEAN := TRUE;
3  END;
4  /
```

```
SQL@V112> SELECT OBJECT_NAME, STATUS FROM USER_OBJECTS WHERE OBJECT_NAME IN
![images](img/U003.jpg)
 ('TIMING','LEVEL1','LEVEL2');

OBJECT_NAME                    STATUS
------------------------------ -------
TIMING                         VALID
LEVEL1                         INVALID
LEVEL2                         INVALID
```

代码重新编译后，它将与 `TIMING_ON` 的值匹配。在这种情况下，你已在所有相关的存储过程中启用了计时测量功能。请记住我之前关于在生产环境中重新编译所有代码的警告。



#### 关于条件编译的最后说明

Oracle 提供了许多查询变量。除了 `$$PLSQL_UNIT` 查询变量外，`$$PLSQL_LINE` 正逐渐成为我最喜欢的变量之一，因为它可以用于调试。

*   `$$PLSQL_CCFLAGS` 保存当前的 `PLSQL_CCFLAGS` 设置。
*   `$$PLSQL_WARNINGS` 保存当前的 `plsql_warnings` 设置。
*   `$$PLSQL_OPTIMIZE_LEVEL` 保存 `PLSQL_OPTIMIZE_LEVEL` 参数值。
*   `$$PLSQL_LINE` 保存 `$$PLSQL_LINE` 变量所在的行号。
*   `$$PLSCOPE_SETTING` 保存会话 `PLSCOPE_SETTINGS` 参数的当前值。
*   `$$PLSQL_CODE_TYPE` 保存参数 `plsql_code_type` 的当前值，值为 `NATIVE` 或 `INTERPRETED`。
*   `$$NLS_LENGTH_SEMANTICS` 保存 `NLS` 长度语义，值为 `BYTE` 或 `CHAR`。

作为一名“普通”用户，你可能无法执行 `show parameter <parameter>` 命令来显示诸如 `plsql_ccflags` 或 `plsql_warnings` 等参数的值。因此，这些查询变量可能会派上用场。创建一个包含匿名 PL/SQL 块的脚本并执行它，即可显示当前会话中查询变量的值，如下所示：

```sql
set serveroutput on
BEGIN
  dbms_output.put_line('PLSQL Unit      : ' ||$$PLSQL_UNIT);
  dbms_output.put_line('PLSQL CCFLAGS   : ' ||$$PLSQL_CCFLAGS);
  dbms_output.put_line('PLSQL Warnings  : ' ||$$PLSQL_Warnings);
  dbms_output.put_line('Optimize Level  : ' ||$$PLSQL_OPTIMIZE_LEVEL);
  dbms_output.put_line('Line: ' ||$$PLSQL_LINE);
  dbms_output.put_line('PLSCOPE setting: ' ||$$PLSCOPE_SETTING);
  dbms_output.put_line('PLSQL code type: ' ||$$PLSQL_CODE_TYPE);
  dbms_output.put_line('NLS length semantics: ' ||$$NLS_LENGTH_SEMANTICS);
 END;
/
```

执行该脚本以查看查询变量的值。

```sql
SQL@V112> @plsql_inqvar.sql
PLSQL Unit     :
PLSQL CCFLAGS  : TIMING_ON:TRUE
PLSQL Warnings : DISABLE:INFORMATIONAL,ENABLE:PERFORMANCE,ENABLE:SEVERE,ERROR:  7204
Optimize Level: 2
Line           : 6
PLSCOPE setting:
PLSQL code type: INTERPRETED
NLS length semantics: BYTE

PL/SQL procedure successfully completed.
```

通过这个小程序，你就可以看到查询变量的当前值。`$$PLSQL_UNIT` 查询变量为空，因为它的值是从一个匿名块中查询的。

![images](img/square.jpg) `注意` PL/SQL 查询变量在编译时被评估和转换，因此，如果任何标志发生了更改，将其放在存储过程中将需要重新编译该过程才能显示正确的值。

你可以执行以下命令来显示 `$$PLSQL_CCFLAGS` 的当前值：

```sql
SQL@V112> exec dbms_output.put_line($$PLSQL_CCFLAGS);

TIMING_ON:TRUE
```

条件编译的一个常见用途是根据 Oracle 版本来启用特定代码；换句话说，将变量定义为 `NUMBER` 或 `BINARY_FLOAT`，如下所示：

```sql
...
$IF DBMS_DB_VERSION.VERSION >=11 $THEN
   VAR BINARY_FLOAT;
$ELSE
  VAR NUMBER;
$END;
...
```

在 `DBMS_DB_VERSION` 的包规范中，有许多布尔型“版本”常量，可用于包含或排除代码。

```sql
…
$IF DBMS_DB_VERSION.VER_LE_11_2 $THEN
  …
$ELSE
  …
$END
…
```

稍微极端一点，你可以让存储过程的参数成为条件化的，如下面的示例所示。

```sql
SQL@V112> CREATE PROCEDURE EXTREME (P_PARAM VARCHAR2 $IF $$INQ_FLAG $THEN, P_PARAM2 VARCHAR2 $END) IS
   2   BEGIN
   3     NULL;
   4   END;
   5   /
```

因此，根据 `$INQ_FLAG` 的值，过程将有一个或两个参数——这可以实现，但并不推荐。

### 总结

在本章中，你了解了启用警告可能带来的好处。你研究了如何在会话、系统或过程级别启用警告。你探索了如何将警告升级为错误，或者如何通过禁用它们来忽略它们。你看到了 PL/SQL 警告确实能帮助你提升代码性能并使其更加健壮。不过，它也会给你带来额外的管理任务，特别是如果你有太多需要启用、禁用或升级为错误的警告例外情况。

![images](img/square.jpg) `注意` 如果你的代码已经在开发环境中检查过，你不一定非要在生产系统上启用 PL/SQL 警告。

在研究条件编译时，你看到使用条件编译可以通过仅在需要时启用日志或调试信息来限制 PL/SQL 使用的内存大小。你还看到，通过使用条件编译、设置标志并执行重新编译，你可以在不编辑代码的情况下启用或禁用代码。条件编译在诸如 C++ 等语言中被广泛使用。现在，你也可以将它纳入你的 PL/SQL 工具箱中。

## PL/SQL 单元测试

**作者：Sue Harper**

任何访问 Oracle 数据库的开发人员都会使用 SQL，无论是运行自己开发的 SQL 报表和查询，还是运行其他开发人员提供的。虽然未必可以说每个开发人员在使用 Oracle 数据库时都会运行和使用 PL/SQL，但这个比例无疑非常高，因为任何对 Oracle 数据库的过程化访问最好都使用 PL/SQL 来完成。作为一名开发人员，如果你创建并使用 PL/SQL，那么你应该调试和测试 PL/SQL，以确保代码的准确性和效率。

在本章中，你将学习 PL/SQL 单元测试：单元测试的含义、它在开发生命周期中的重要性，以及如何构建测试。有几种工具和实用程序可以帮助你构建测试。使用哪一个并不重要，重要的是你要养成针对代码构建和运行测试的习惯。我使用 Oracle SQL Developer 进行任何测试，因此我将使用该工具来说明构建和运行 PL/SQL 单元测试的方法。

### 为什么要测试你的代码？

软件和应用程序开发人员都会编写代码；有些人编写的代码比其他人更好，但每个人都试图编写出能实现预期功能的代码，并希望它能很好地做到这一点。这对于编写好代码来说，几乎算不上是充满信心的表态！不仅你的代码必须准确且性能良好，知道它稳健且能经得起时间的考验也很重要。这意味着代码应该以相同的方式执行，随着时间的推移产生相同的结果，而不受其周围环境变化的影响。

你如何保证这一点？也许你无法保证，因为你无法 100% 确定未来可能发生的所有变化，但如果你编写了可以在你想要或需要时随时运行的回归测试，那么你就可以确定一个程序单元是否按预期运行并持续如此。你编写的测试越多，变化越多，你就越能确保程序单元是稳健的，并且仍然按预期工作。

至关重要的是，你不仅要知道你的代码今天能正常工作，还要能够*证明*它有效果——并且在未来任何时间你都能证明同样的效果。根据程序单元的需求，你的代码可能必须支持一系列场景或用例，你应该能够测试广泛的用例。如果你能通过数据表明你已经测试了广泛的用例，并重新运行这些测试用例，你就更接近于经验证明代码的稳健性。如果你能够随时重新运行测试并再次验证结果，你就可以快速确定故障点并加以解决。

没有回归测试，你无法知道你的代码是否在工作，或者它何时停止了工作。即使是最好的代码也可能不可靠，但你无法证明它是好是坏。



### 什么是单元测试？

单元测试（*unit testing*）是 Java 和敏捷开发社区常用的术语，指编写小型、可重复的测试，用于测试一个“工作单元”或某个职责领域。编写单元测试在 Java 开发社区很常见，开发者通常在编写代码的同时编写测试。许多 Java 开发者使用 `JUnit`——一个用于编写单元测试的开发框架。大多数认真的 Java 开发者都会在编写开发代码的同时编写单元测试。有些人甚至在开始编写代码之前就先写好测试。他们知道代码预期的输入类型，也知道预期的结果，因此可以提前构建测试。Java 界的普遍理念是“写一点，测一点；再写一点，再测一点……”，这是明智的建议。

尽管 `PL/SQL` 开发者多年来一直在开发 `PL/SQL` 程序单元，并采用基于纸面的测试或使用 `SQL*Plus`，但支持构建测试套件来支撑和测试代码的单元测试框架却非常少。一些开发者确实会构建回归测试，但如果没有框架，构建单元测试会耗费时间，而且很难向管理层证明额外投入的合理性，因此大多数人并未涉足这一领域。在会议和培训活动中就此主题进行演讲时，通常只有一两位听众在为其 `PL/SQL` 回归测试编写单元测试。

作为 `PL/SQL` 开发者，你应该将单元测试视为开发周期的一部分，视为可交付成果的一部分。在编写 `PL/SQL` 代码的同时编写单元测试，并将其与程序单元一起存储。你应该能够随时运行和重新运行测试，以验证代码是否仍然按预期和期望的方式工作。

#### 调试还是测试？

调试一段代码和为其构建单元测试有什么区别？这不是一个非此即彼的情况：你确实需要调试代码以确保其完成所需功能，或者找出它为什么没有按你认为应该的方式工作。然而，如果考虑差异，其中一个区别在于可重复性。如果你编写了一个 `PL/SQL` 程序单元，并希望验证其是否执行了你期望的功能，一旦它成功编译，你就可以在代码的不同点插入检测代码，以验证状态、变量值、哪些过程调用了哪些过程等等。你可以传入不同的参数集并验证结果是否正确。但是，如果你想在一个月后验证同样的内容，就必须重复相同的过程。马丁·福勒（Martin Fowler），这位撰写了多本关于敏捷开发、重构和极限编程书籍的作者，对此总结得非常完美：

> 每当你想在打印语句或调试器表达式中键入内容时，不如将其写成一个测试。

#### 何时应该构建测试？

考虑你的 `PL/SQL` 代码的生命周期。你需要解决一个问题，并且知道代码应该做什么。你可以使用纸笔、文本编辑器或更复杂的图形化 `SQL` 开发工具编辑器来编写代码。一旦编写好 `PL/SQL` 代码，你就编译它以验证语法是否正确。如果代码编译成功，你就可以在 `Oracle Database` 中针对模式执行它。也许代码能立即按要求工作并成功得出结果。如果不能，你需要调试代码以确定问题出现的位置。这种编辑/编译/运行/编辑/编译/调试/编译/运行的循环可能会持续到你对程序单元满意，并准备交付代码为止。

许多单元测试的倡导者认为，你应该在开始编写代码之前就写好测试，并将其视为完整流程的一部分。其理念是，你已经知道代码的预期结果，并且知道结果是什么，因此你可以根据结果构建测试，然后编写代码以确保测试通过！我认为这是编写、规划和构建测试的结合，所以我敦促你将编写测试视为过程的一部分，无论是先编写代码然后添加一些测试，还是同时微调代码和测试。

### 构建单元测试的工具

市场上用于编写 `PL/SQL` 单元测试的工具数量有限；因此，许多用户编写了自己的测试机制。我在这里列出了三个较受欢迎的选项；它们都有很多值得推荐的地方。如果你刚接触测试领域，值得花些时间研究一下现有工具。

#### utPLSQL：使用命令行代码

`utPLSQL` 可能是已知最早的可用于 `PL/SQL` 单元测试的单元测试框架。它由 Steven Feuerstein 编写，已面世多年。它是一个用于构建单元测试的开源 `PL/SQL` 测试框架。你通过运行一个脚本来安装 `utPLSQL`，该脚本会安装测试所需的所有表、包、过程和其他对象。你可以在 `SQL*Plus` 或命令行中创建和构建所有测试，因此不需要特定工具或额外的客户端。`utPLSQL` 托管在 Sourceforge 上，网址是 [`http://utplsql.sourceforge.net`](http://utplsql.sourceforge.net)，并提供了丰富的资源、文档和示例，介绍如何安装软件、构建测试以及充分利用该框架。

#### Quest Code Tester for Oracle

Quest Code Tester for Oracle 是一款用于定义和运行测试的商业产品。你可以为单个程序或包构建测试，也可以构建单独的测试或套件。使用此类产品的一大优势是，你可以构建测试并在需要时重新运行它们，从而支持了为项目构建完整回归测试套件的论点。

Code Tester 与其他产品的不同之处在于，它允许你描述 `PL/SQL` 程序单元的预期行为，并将其存储在存储库中。然后，该工具基于这些定义生成测试代码。它根据你提供的代码提供一组可能的预期结果，你可以接受并包含或排除这些结果。这种代码生成是该工具中一个实用且强大的功能。当你开始构建测试时，可以使用“快速构建”选项向测试中添加测试用例。

Code Tester 是一个独立的产品（未集成到任何编辑环境中），允许你独立于开发环境构建和运行测试。缺点是，如果需要调试，你需要将代码导入到一个单独的 `PL/SQL` 环境中。不过，你可以从 Quest 的 Toad for Oracle 运行测试，并从该产品切换到 Code Tester 进行进一步工作。



