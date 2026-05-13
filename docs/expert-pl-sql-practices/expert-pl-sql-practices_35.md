# 检查函数：返回真值或断言失败

前面的例子引出了一个关于断言返回 `BOOLEAN` 值的函数真值的有趣考量。例如，`process_record` 过程是否应该将 `ready_to_process` 函数断言为一个前置条件？

```sql
PROCEDURE process_record (rec1 my_rectype)
IS
    l_module VARCHAR2(30) := 'PROCESS_RECORD';
BEGIN
  -- 前置条件：测试记录是否准备好处理
  assertpre(ready_to_process(rec1), 'REC1 ready to process', l_module);
```

这样做到底行不行，答案是“视情况而定”。Bertrand Meyer 对此有如下论述：

> *如果你通过坚持使用简单且不言自明的正确函数来施加适当的注意，那么在断言中使用函数例程可以为你提供一种强大的抽象手段。*
>
> Bertrand Meyer, *《面向对象的软件构造》*, 第二版

我非常喜欢“简单且不言自明的正确”这个短语，因为它精辟地抓住了无缺陷代码的本质。在我自己的编程实践中，我有时发现同一个包中的多个模块可能都引用并操作某些私有的全局数据结构，并且都需要对这些结构断言一组共同的前置条件以保证正确性。在这种情况下，实现一个将这些断言组合在一起的函数可以消除大量的代码重复。

```sql
FUNCTION check_all_ok(module_IN VARCHAR2) RETURN BOOLEAN
IS
BEGIN
  assert(condition1, 'condition1', module_IN);
  assert(condition2, 'condition2', module_IN);
  …
  assert(conditionN, 'conditionN', module_IN);

  RETURN TRUE;
END check_all_ok;
```

只要被测试的条件本身是局限于包变量或数据结构以及 `PL/SQL` 语言原语的简单表达式，那么这类“检查函数”就可以被认为是“简单且不言自明的正确”的，因为它们显然只能返回 `TRUE` 或抛出 `ASSERTFAIL` 异常。因此，`check_all_ok` 函数可以安全地在依赖它的各个模块中被断言。

```sql
assertpre(check_all_ok(l_module), 'check_all_ok', l_module);
```

请注意，调用者的本地模块名称被传递给 `check_all_ok` 函数，这样生成的任何 `ASSERTFAIL` 异常都将被正确地标记上检查失败所在的模块名称。

### 无缺陷代码的原则

本章实际上已经快速而简洁地涵盖了大量内容。本章开头附近引用的参考文献可以让你更全面地了解契约式设计的深度和丰富性。我提到自己对契约式编程的吸引力植根于对缺陷（尤其是我自己代码中的缺陷）的厌恶。软件契约的目的是通过明确假设和期望，并在代码本身中直接强制执行这些假设和期望，来消除软件接口的歧义。通过这些手段，一整类缺陷可以在程序中预先设计排除——或者至少被迅速暴露和处理。

这里介绍的机制和技术绝不意味着在 `PL/SQL` 中提供了对契约式设计的完整支持。事实上，Bertrand Meyer 创造了一种名为 Eiffel 的完整编程语言及其开发环境来支持它，正是因为现有的语言和工具不能完全支持这种范式。然而，我们必须与 `PL/SQL` 共存并尽力而为，我尝试提出了一种在 `PL/SQL` 环境中采用契约式设计哲学的实用方法。

在结束之际，我将简要讨论几个用于开发无缺陷代码的契约式原则。

### 严格断言前置条件

`PL/SQL` 程序通常在通过输入参数值传递给它们的或作为包全局变量可用的简单或复杂数据结构上执行其算法。在这方面，它们与其他任何语言编写的程序并无不同。程序算法通常对它们所操作的数据有非常具体的要求，当这些要求未得到满足时，算法很可能会失败或给出不可靠的结果。这些要求正是契约式设计所指的前置条件，严格强制执行前置条件要求是确保你的算法每次都能成功执行的首要且最佳步骤。这听起来可能显而易见，但是，有多少程序在编写时会详细关注识别其算法对输入的每一个假设，并明确地将它们记录下来？而在这些程序中，又有多少会在代码本身中实际强制执行这些要求，使得无效输入无法进入核心部分，可以这么说？

正如已经多次提到的，许多算法并非设计为在 `NULL` 输入上运行，或者当遇到 `NULL` 输入时，它们会采用特殊情形代码产生特殊情形输出。为什么不干脆完全排除 `NULL` 值，将这些程序限制在它们所设计的简单情形中呢？这并不是说 `NULL` 输入总是可以禁止的，但这也许正是契约谈判应该开始的地方。如果客户端确实无法遵守，那么可能需要放宽前置条件，并相应地调整程序，但至少那样你会知道原因所在。

另一个好的实践是梳理并明确强制执行每一个单独的前置条件，而不是将多个前置条件叠加在一个断言中。例如，如果两个输入参数 `x` 和 `y` 被要求具有非 `NULL` 值，并且 `x` 必须大于 `y`，那么所有这三个前置条件都可以用

```sql
assertpre(x > y,'x > y', l_module)
```

来强制执行。如果 `x` 或 `y`（或两者都是）为 `NULL`，此断言将失败；然而，这些要求在断言中并不明确，必须被推断出来。契约式设计的核心在于明确和精确，不容忍推断和影射。因此，在这个例子中应该有三个前置断言。

前面的例子也说明了断言的一个我尚未强调的主要好处，即它们不仅强制执行，而且记录了软件契约。编写代码使用你的模块的程序员如果能够阅读源代码，就可以直接从前置条件中读取正确使用的要求。

明确要求良好的输入，那么良好的输出就更容易保证。



#### 彻底模块化

在要求调用者提供良好输入并在代码中强制执行这些要求之后，你的模块就有义务产生正确的输出。拥有成百上千行代码并对数据结构进行多次转换的程序，很难甚至不可能验证其正确性。然而，针对理解透彻且经过验证的数据结构、使用非常特定算法的紧凑计算单元，实际上可能具有“不证自明地正确”的特性。这只能通过模块化来实现，即将庞大复杂的计算分解为越来越小的、自包含的计算子组件，然后将它们串联起来以实现更大的目标。每个子组件在算法上应尽可能简单，以便通过检查和最少的脑力劳动就能获得对其正确性的信心。

请注意，模块化代码并不总是意味着将其分解为独立的可执行模块；它也可以是将单个大型程序组织成一组子转换，这些子转换从提供的输入逐步生成预期的输出。在这些情况下，你会希望在子转换结束、另一个开始时，对中间计算状态进行断言。

通过将代码模块化为更多、更小的组件，你创建了更多的接口和更多表达契约的机会。如果你坚持严格的前置条件测试这一原则，这意味着你的代码中将内置越来越多的断言，保护所有这些接口的契约要求。每个断言都是在运行时执行的正确性测试：你的代码就像在断言的烈火中淬炼过的钢一样变得坚韧。漏洞怎么可能存活？

在可信数据上运行的简单且不证自明的正确算法，是通往无漏洞代码的正途。

#### 采用基于函数的接口

在采用了前两个原则之后，你的简单且不证自明的正确的模块应该是什么样子？当模块必须将计算结果传回给调用者时，函数在强制执行契约方面提供了明显的优势。这些优势之前已经提到过，但值得重申。

*   函数提供单一输出，因此与单一目的一致。
*   函数的编写方式可以使后置条件断言精确地定位在 `RETURN` 语句之前。
*   函数可以与变量或条件测试互换使用，从而实现高度紧凑和富有表现力的代码，尤其是当它们断言 `NON NULL` 后置条件时。

这并不是说过程没有其用武之地，也不是说你应该避免它们。然而，我见过的许多 PL/SQL 代码都是过程过多而函数过少。当需求是特定输入数据被计算转换并返回某些输出数据时，基于函数的接口为采用严格的契约导向提供了更好的机会。

#### 遇到 ASSERTFAIL 时崩溃

在按契约设计中，断言失败就意味着漏洞，句号。捕获 `ASSERTFAIL` 异常并以某种方式掩盖或隐藏其发生的程序，实际上是在歪曲事实：一个正确性漏洞已被发现却未被报告。更好的做法是让程序崩溃，并利用异常消息中提供的诊断信息来查找和修复漏洞。

用安迪·亨特和戴夫·托马斯的话说（见上文参考文献）：

> *死去的程序不会说谎*

如果你发现自己被迫捕获和处理 `ASSERTFAIL`，因为尽管有它，正确的计算仍能进行，那么可能有一些契约问题需要理清。契约导向并不意味着通过不可协商的前置条件对客户端模块实施独裁控制。契约是双方之间的协议，条款需要双方都能接受。因此，与其捕获和处理 `ASSERTFAIL`，不如尽可能重新协商契约。

#### 对你的后置条件进行回归测试

人们非常强调对前置条件进行严格断言，而对后置条件则关注得少得多。在大多数情况下，要在给定前置条件的前提下绝对保证输出正确性，充其量是困难的，在某些情况下几乎不可能。函数 `even` 提供的例子是不寻常的，它拥有一个用于互补函数 `odd` 的替代算法，这提供了一种断言输出正确性的方法。即使可能，断言完整的后置条件正确性的计算成本也可能使计算输出本身的成本有效地翻倍。

然而，有一种常见情况，即对于给定输入的正确输出是众所周知的，那就是单元测试。典型的单元测试包括在特定输入集下将程序输出与众所周知的预期结果进行匹配。这实际上相当于断言程序在给定这些输入下满足了后置条件。所以，希望如果你在进行生产级的 PL/SQL 编程，你已经有一个测试框架就位，并通过定期的回归测试来验证模块的正确性。

这里需要注意的一点是，只有外部声明的模块才能进行回归测试。一个可能的变通方法是准备两个版本的包规范：一个只暴露打算供外部使用的模块，另一个则同时暴露内部模块以用于回归测试。当然，两个版本都应该使用相同的包体，并且必须非常小心，除了声明的模块外，不要让规范发生分歧。这不是理想的，但它是可行的。



#### 避免正确性与性能的权衡陷阱

最后，提出一条警示性原则：避免陷入我所谓的“错误的正确性-性能权衡陷阱”。如果你采纳本文推荐的面向契约的方法，你可能会发现，严格断言契约元素不仅会导致大量额外的键入工作，而且每次`assert`调用都是实际执行的代码，虽然其开销**极其微小**，但并非完全免费。在那些断言了长串复杂前置条件（可能在一个游标循环的数千行中）的模块中，当代码进行性能分析时，这些断言可能会显得“昂贵”。

在这种情况下，你可能会基于其计算成本而倾向于移除这些断言；我的建议是尽可能避免这种诱惑。断言关乎确保代码的正确性；而性能关乎快速执行代码。不存在所谓的“正确性-性能权衡”：快速获得的错误结果仍然是错误的，因此是无用的。

然而，如果契约断言带来的性能影响似乎高到无法容忍，并且所涉及的模块高度受信，那么放宽这条准则并停止测试某些断言可能是必要的。与其从源代码中完全移除它们，不如简单地将它们注释掉。请记住，源代码中的断言也是模块所遵循契约的重要文档工件。

或者，在 Oracle 10g 及更高版本中，你可以使用 PL/SQL 条件编译功能，在某种程度上鱼与熊掌兼得。考虑以下过程，当编译时设置了条件编译标志 `DBC` 为 `TRUE`，则有条件地包含一个前置条件断言：

```sql
PROCEDURE proc02
  IS
    l_module VARCHAR2(30) := 'PROC02';
  BEGIN
    -- 条件前置条件断言
    $IF $$DBC $THEN
    assertpre(FALSE,'Assert FALSE precondition',l_module);
    $END

    -- 过程逻辑
    null;
  END proc02;
```

该过程可以设置标志为 `TRUE` 进行编译用于测试，并重新编译时将其设置为 `FALSE` 以发布到生产环境。在生产环境中，如果怀疑程序存在问题，可以将标志重新设为 `TRUE` 并重新编译代码，以重新启用断言测试来发现和诊断错误。再次强调，这并不推荐，因为生产环境中的正确性测试最为重要，但在性能影响高得无法接受的情况下，这是可能的。

#### Oracle 11g 优化编译

Oracle 11g 引入了 PL/SQL 的优化编译，包括子程序内联，这在小的局部程序被频繁调用时可以获得显著的性能优势。深入讨论该主题超出了本章范围，但基本上，内联可以用被调用模块代码的副本“内联”地替换对编译单元内局部模块的调用，从而消除调用栈开销。

内联优先针对较小、较简单的局部模块进行，这恰恰描述了我多年来一直使用和推荐的包局部断言机制。最终，我偏爱将断言局部化在每个包内的这种非传统做法得到了一些真正的验证！因此，采用包局部断言并启用完全优化，可以最大限度地降低在 PL/SQL 代码中强制实施软件契约的成本。在我看来，这听起来像是双赢。

通过在编译前将 `PLSQL_OPTIMIZE_LEVEL` 参数设置为值 `3` 来启用 PL/SQL 内联，如下所示：

```sql
ALTER SESSION SET PLSQL_OPTIMIZE_LEVEL=3;
```

你可以使用 `PLSQL_WARNINGS` 参数获知内联优化和其他编译器警告，如下所示：

```sql
ALTER SESSION SET PLSQL_WARNINGS='ENABLE:ALL';
```

对那些充斥着契约强制断言的包，通过实验来衡量 PL/SQL 内联带来的性能收益将会很有趣。

### 总结

契约式设计是一种强大的软件工程范式，用于从你的代码中消除与模块接口（APIs）相关的缺陷。软件契约由前置条件（表示对调用模块施加的要求）和后置条件（表示对被调用模块施加的义务）来指定。这些契约元素通过一个称为 `assert` 的软件机制来强制实施。断言失败总是指示调用模块或被调用模块中的代码存在错误，具体取决于违反的是前置条件还是后置条件。

在 PL/SQL 中采用面向契约的方法，可以通过一个简单、标准化的断言机制，并在整个代码库中采用规范化、标准化的用法来实现。本章向你展示了我为此开发的具体技术，如果你选择采纳，我真诚地希望你发现这些技术很有价值。

## 第 9 章

## 从 SQL 调用 PL/SQL

作者：Adrian Billington

函数是任何设计良好的 PL/SQL 应用程序不可或缺的部分。它们是编程最佳实践的体现，例如代码模块化、重用以及业务或应用逻辑的封装。当用作大型程序的简单构建块时，它们可以是一种优雅而简单的方式，以最低成本扩展功能，同时降低代码复杂度。

反之，当 PL/SQL 函数被大量使用时，特别是在 SQL 语句中，可能会带来一系列相关的成本，其中最显著的是性能成本。根据函数的性质，仅仅从 SQL 调用 PL/SQL 和/或过度的 I/O 就可能降低即使是最简单的查询的性能。

鉴于此，我将描述从 SQL 调用 PL/SQL 函数所涉及的各种成本，更重要的是，演示一系列可用于减轻其影响的技术。到本章结束时，你将更清楚地意识到在应用程序中使用 PL/SQL 函数的成本。因此，你将能更好地为新的或重构的应用程序做出良好的设计决策，并改进对系统关键部分的性能调优工作。

注意：本章中的示例均已在 Oracle Enterprise Linux 11.2.0.2 数据库上运行。完整的代码清单可从 Apress 网站下载。在许多情况下，我已将输出精简到传达信息和保持章节流畅所需的最低限度。所有示例均可在 SQL*Plus 中执行，下载包中包含了我使用的除 Oracle 数据库软件自带工具外的任何实用程序。



### 在 SQL 中使用 PL/SQL 函数的成本

SQL 和 PL/SQL 语言是无缝集成的。自 `Oracle Database 9i` 起，这两种语言甚至共享同一个解析器。在我们编写的大多数代码中，我们利用这种互操作性来获益，例如在 PL/SQL 程序中使用 SQL 游标，或从 SQL 调用 PL/SQL 函数。我们这样做时，通常很少关心 Oracle Database 在底层需要做些什么来运行我们的程序。

尽管存在这种集成，但将它们结合起来使用仍会产生运行时成本，特别是在从 SQL 调用 PL/SQL 时（当然，这也是本章的重点）。在接下来的几页中，我将探讨其中的一些成本。有趣的是，Oracle 一直在努力减少在多个版本中结合 SQL 和 PL/SQL 代码的影响，因此我所描述的问题可能并非其最终状态。

我对在 SQL 中使用 PL/SQL 函数成本的调查可以分为三个主要领域：

*   性能
*   可预测性（与性能相关）
*   副作用

![images](img/square.jpg) **注意** 我不涉及语法或操作限制（例如禁止在从 SQL 调用的 PL/SQL 函数中使用 DML）。这些本身并非运行时成本，并且在其他地方有详细记载。

### 上下文切换

上下文切换是一种几乎无法减少的计算成本，它发生在 CPU、操作系统和软件中。其机制根据发生位置而不同，但它通常是一种 CPU 密集型操作，设计人员会尽可能地对其进行优化。

当我们考虑与 Oracle Database 相关的“上下文切换”时，我们特指 SQL 引擎和 PL/SQL 引擎之间处理控制的交换（而不必理解在此过程中发生了什么）。这两个引擎是独立且不同的，但我们交替使用它们。这意味着，当从 PL/SQL 调用 SQL 或反之亦然时，调用上下文需要存储其处理状态，并将控制权和数据移交给对应的引擎（该引擎可能从之前的某个切换点继续，也可能不）。这种切换循环计算密集，并且通常可以重复多次，以至于其对响应时间的影响可能变得相当明显。

对许多人来说，第一次接触 Oracle Database 中的上下文切换可能是在 `Oracle Database 8i` 中引入 `BULK COLLECT` 和 `FORALL` PL/SQL 特性时，这些特性 *专门* 旨在减少上下文切换。这些性能特性使得可以在单个上下文切换中在 SQL 和 PL/SQL 引擎之间绑定数据数组，极大地减少了当时流行的逐行 PL/SQL 技术所产生的上下文切换次数。在几个主要版本中，这些语言特性带来的几乎所有性能收益都可归因于它们促进的上下文切换减少。（在最近的版本中，当使用 PL/SQL 数组处理时，也可以实现一些进一步的收益，例如批量 INSERT 操作的重做优化。）

![images](img/square.jpg) **注意** 减少上下文切换是一种非常有效的性能技术，以至于 Oracle 设计人员在 `Oracle Database 10g` 引入的 PL/SQL 优化编译器中，为游标 FOR 循环添加了隐式的 `BULK COLLECT`。

所有这些对 PL/SQL 程序员来说当然是好消息，但过度上下文切换的影响在其他地方仍然会遇到。特别是，从 SQL 调用 PL/SQL 函数需要支付性能代价。

#### 测量上下文切换

尽管 Oracle Database 的工具非常出色（确实非常出色），但上下文切换本身无法直接测量；它发生在内核的深处，没有简单的统计信息或计数器可供量化。但鉴于从 SQL 调用 PL/SQL（反之亦然，虽然我将不再提醒读者这一点）必然导致上下文切换，其影响可以通过一组简单的测量和提供的实用程序推断出来。

代码清单 9-1 展示了两个 SQL 语句的简单比较，其中一个语句为每一行调用一个 PL/SQL 函数。该函数除了返回输入参数外不执行任何操作，时间使用 `SQL*Plus` 计时器进行比较。我使用了 500 的数组大小和 `Autotrace` 来抑制输出。

***代码清单 9-1.** 从 SQL 调用 PL/SQL 函数的成本（基线）*

```sql
SQL> SELECT ROWNUM AS r
  2  FROM   dual
  3  CONNECT BY ROWNUM <= 1e6;

1000000 rows selected.

Elapsed: 00:00:01.74

Statistics
----------------------------------------------------------
          0  recursive calls
          0  db block gets
          0  consistent gets
          0  physical reads
          0  redo size
    6290159  bytes sent via SQL*Net to client
      40417  bytes received via SQL*Net from client
       2001  SQL*Net roundtrips to/from client
          1  sorts (memory)
          0  sorts (disk)
    1000000  rows processed
```

基线 SQL 语句在不到 2 秒的时间内生成了 100 万行数据。从 `Autotrace` 输出可以看出，这种形式的 SQL 语句除了 CPU（以及更大的行数下的网络传输和内存）外，对资源消耗很小。在 SQL 中添加 PL/SQL 函数调用对经过时间有很大影响，如代码清单 9-2 所示。

***代码清单 9-2.** 从 SQL 调用 PL/SQL 函数的成本（使用函数）*

```sql
SQL> CREATE FUNCTION plsql_function(
  2                   p_number IN NUMBER
  3                   ) RETURN NUMBER AS
  4  BEGIN
  5     RETURN p_number;
  6  END plsql_function;
  7  /

Function created.

SQL> SELECT plsql_function(ROWNUM) AS r
  2  FROM   dual
  3  CONNECT BY ROWNUM <= 1e6;

1000000 rows selected.

Elapsed: 00:00:05.61
```

我省略了 `Autotrace` 输出，因为它与基线 SQL 的输出几乎相同，但你可以看到，添加对这个最简单查询的函数调用几乎增加了 4 秒的运行时间。

如前所述，无法直接测量上下文切换本身，但你可以使用诸如 `SQL trace`、`PL/SQL Hierarchical Profiler`（`Oracle Database 11g` 特性）或 `V$SESS_TIME_MODEL` 视图等工具来测量在 PL/SQL 引擎中花费的总时间。代码清单 9-3 显示了针对先前 SQL 的 Hierarchical Profiler 会话的摘要结果。

***代码清单 9-3.** 从 SQL 调用 PL/SQL 函数的成本（Hierarchical Profiler 结果）*

```text
FUNCTION                     LINE#      CALLS SUB_ELA_US FUNC_ELA_US
-------------------------- ------ ---------- ---------- -----------
__plsql_vm                      0    1000003    3122985     2006576
PLSQL_FUNCTION                  1    1000000    1116012     1116012
__anonymous_block               0          3        397         311
DBMS_OUTPUT.GET_LINES         180          2         86          82
DBMS_OUTPUT.GET_LINE          129          2          4           4
DBMS_HPROF.STOP_PROFILING      59          1          0           0
```



此性能剖析文件清晰地显示，`PL/SQL` 函数被调用了 100 万次，这占用了略高于 1 秒的耗时。同样有趣的是，`PL/SQL` 虚拟机（即 `PL/SQL` 引擎）在函数执行之外花费了 2 秒的处理时间。这部分时间中的一部分可归因于上下文切换。总体而言，`PL/SQL` 引擎的耗时超过了 3 秒。（注意，`DBMS_OUTPUT` 调用是 `SQL*Plus` 中 `serveroutput` 设置的结果，在此实例中无异于背景噪音。）

![images](img/square.jpg) 注：关于 `PL/SQL` 层次化剖析器的详细讨论，请参见第 13 章。

这些数据通过查询的扩展 `SQL` 跟踪得到了验证，正如清单 9-4 中的 `TKProf` 输出所展示的那样。

**清单 9-4.** 从 SQL 调用 PL/SQL 函数的成本（tkprof 输出）
```
call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          0          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch     2001      4.92       5.08          0          0          0     1000000
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total     2003      4.92       5.08          0          0          0     1000000

<snip>

Rows (1st)   Row Source Operation
----------   ---------------------------------------------------
   1000000   COUNT  (cr=0 pr=0 pw=0 time=3595520 us)
   1000000    CONNECT BY WITHOUT FILTERING (cr=0 pr=0 pw=0 time=1693263 us)
         1     FAST DUAL  (cr=0 pr=0 pw=0 time=28 us cost=2 size=0 card=1)
```

我删除了 Rows (max) 和 Rows (avg) 统计信息以缩减输出，但你可以从行源统计中看出，为结果集投射这 100 万行大约花费了 3.6 秒（这包括了相关等待摘要中显示的一小部分网络传输时间），而生成这些行本身仅花费了 1.7 秒。

这些例子尽可能简化，以展示从 `SQL` 调用 `PL/SQL` 函数的成本。尽管它们的耗时微不足道，但你可以清楚地看到 `PL/SQL` 函数为响应时间增加了显著的开销（超过 300%）。当然，那些耗时更长的 `PL/SQL` 函数（尤其是包含 `SQL` 的函数）将会比上下文切换问题导致更严重的性能问题。然而，对于纯 `PL/SQL` 函数（即函数体中没有 `SQL`），上下文切换可能是 `PL/SQL` 虚拟机总耗时中的主要部分。

上下文切换对于那些本身就需要做大量其他工作（例如访问和连接数据集）的查询，也会产生性能影响。在清单 9-5 中，我有一个轻量级的 `PL/SQL` 函数，并用它来格式化销售报告的一部分。和之前一样，我使用了 `Autotrace` 和 `SQL*Plus` 计时来记录运行时统计信息。

**清单 9-5.** 在更密集查询中的上下文切换
```
SQL> CREATE FUNCTION format_customer_name (
  2                  p_first_name IN VARCHAR2,
  3                  p_last_name  IN VARCHAR2
  4                  ) RETURN VARCHAR2 AS
  5  BEGIN
  6     RETURN p_first_name || ' ' || p_last_name;
  7  END format_customer_name;
  8  /
Function created.

SQL> SELECT t.calendar_year
  2  ,      format_customer_name(
            c.cust_first_name, c.cust_last_name
            )                 AS cust_name
  3  ,      SUM(s.quantity_sold) AS qty_sold
  4  ,      SUM(s.amount_sold)   AS amt_sold
  5  FROM   sales     s
  6  ,      customers c
  7  ,      times     t
  8  WHERE  s.cust_id = c.cust_id
  9  AND    s.time_id = t.time_id
 10  GROUP  BY
         t.calendar_year
 11  ,      format_customer_name(
            c.cust_first_name, c.cust_last_name
            )
 12  ;

11604 rows selected.
```

**耗时: 00:00:06.94**
```
Statistics
----------------------------------------------------------
       3216  consistent gets
          0  physical reads
```

注意，我修剪了 `Autotrace` 输出，仅包含 I/O 统计信息。报告完成大约需要 7 秒。我在清单 9-6 中将其与纯 `SQL` 实现进行了比较。

**清单 9-6.** 在更密集查询中的上下文切换（SQL 实现）
```
SQL> SELECT t.calendar_year
  2  ,      c.cust_first_name || ' ' || c.cust_last_name AS cust_name
  3  ,      SUM(s.quantity_sold)                          AS qty_sold
  4  ,      SUM(s.amount_sold)                            AS amt_sold
  5  FROM   sales     s
  6  ,      customers c
  7  ,      times     t
  8  WHERE  s.cust_id = c.cust_id
  9  AND    s.time_id = t.time_id
 10  GROUP  BY
         t.calendar_year
 11  ,      c.cust_first_name || ' ' || c.cust_last_name
 12  ;

11604 rows selected.
```

**耗时: 00:00:01.40**
```
Statistics
----------------------------------------------------------
       3189  consistent gets
          0  physical reads
```

通过移除函数而使用 `SQL` 表达式，报告的运行时间减少到 1.5 秒以下。因此你可以看到，即使在访问和连接多个行源时，`PL/SQL` 函数的存在仍然会影响查询的整体响应时间。

##### 其他 PL/SQL 函数类型

上下文切换的惩罚适用于任何从 `SQL` 调用的 `PL/SQL` 源（包括用户定义的聚合函数、类型方法或表函数）。一个很好的例子是 Tom Kyte 多年前编写的 `STRAGG` 函数，用于演示用户定义的聚合函数（当时是 Oracle 9*i*的新特性）。直到 Oracle 数据库 11*g*，都没有与 `STRAGG` 等效的内置函数，因此这个实用函数作为 Oracle 论坛上解决字符串聚合问题的方案变得极其流行。

`LISTAGG` 内置函数的发布，通过简单地移除 `SQL` 引擎执行上下文切换的需要，提供了相对于 `STRAGG` 显著的性能提升。我在清单 9-7 中用一个简单的例子展示了这一点，该例子聚合了提供的 `SH.CUSTOMERS` 表中每个客户的 ID 范围。

**清单 9-7.** 内置聚合函数 vs. 用户定义聚合函数
```
SQL> SELECT cust_first_name
  2  ,      cust_last_name
  3  ,      LISTAGG(cust_id) WITHIN GROUP (ORDER BY NULL)
  4  FROM   customers
  5  GROUP  BY
         cust_first_name
  6  ,      cust_last_name;

5601 rows selected.
```

**耗时: 00:00:00.23**
```
SQL> SELECT cust_first_name
  2  ,      cust_last_name
  3  ,      STRAGG(cust_id)
  4  FROM   customers
  5  GROUP  BY
         cust_first_name
  6  ,      cust_last_name;

5601 rows selected.
```

**耗时: 00:00:01.40**

我抑制了 `Autotrace` 输出，但两种方法都没有产生物理 I/O，且逻辑 I/O 是相同的。然而，用户定义的聚合函数耗时是内置等效函数的六倍以上。

总而言之，我已经证明了仅仅从 `SQL` 调用 `PL/SQL` 函数就可能是昂贵的，这与函数本身的性质无关。稍后，我将展示各种你可以应用的方法，以减少或完全消除 `SQL` 查询中的这部分成本。



### 执行

常言道，最快完成某事的方式就是不去做它。同样普遍认为，即使某事快如纳米级，但只要执行次数足够多，其累积耗时就会开始攀升。这正是从 SQL 中调用 PL/SQL 函数时常见的情况。因此，PL/SQL 函数被 SQL 语句调用的次数越多，在函数本身以及相关的上下文切换上所耗费的时间和资源就越多。

当一个 PL/SQL 函数被包含在 SQL 语句中时，该函数应该执行多少次呢？回答这个问题需要考虑诸多因素；即使你对该函数在查询中的使用有所了解，也无法确定。

SQL 是一种非过程化语言，但试图预测 PL/SQL 函数被 SQL 查询执行的次数，却是一种典型的过程化思维方式。你无法控制 SQL 引擎的内部工作方式，因此也无法绝对控制 PL/SQL 函数被调用的次数。（我说“绝对”是因为有一些技术可以用来减少函数的执行次数，但你无法可靠地预测最终将其减少到的调用次数。）如果你不去尝试控制执行次数，那么随之而来的就是，你无法影响你的函数对 SQL 语句响应时间所做的贡献。

### 查询重写与 CBO

还需要考虑基于成本的优化器的影响。随着 Oracle 数据库的每个版本发布，CBO 变得越来越复杂；特别是，它发现了更多通过转换技术来优化查询的方法。这意味着，你调用 PL/SQL 函数所在的查询块，可能不会出现在 CBO 生成的最终查询版本中。

这可以通过 CBO 的视图合并转换的一个小例子轻松证明。清单 9-8 包含一个模拟报告，该报告按客户和日历年汇总产品销售情况。此示例中的 PL/SQL 函数维护一个计数器来追踪执行次数。它从 CUSTOMERS 表的一个内联视图中被调用，旨在预计算并限制函数调用的次数。

**清单 9-8.** CBO 查询转换与函数执行

```sql
SQL> SELECT t.calendar_year
  2  ,      c.cust_name
  3  ,      SUM(s.quantity_sold) AS qty_sold
  4  ,      SUM(s.amount_sold)   AS amt_sold
  5  FROM   sales     s
  6  ,     (
  7         SELECT cust_id
  8         ,      format_customer_name (
  9                   cust_first_name, cust_last_name
 10                   ) AS cust_name
 11         FROM   customers
 12        )          c
 13  ,      times     t
 14  WHERE  s.cust_id = c.cust_id
 15  AND    s.time_id = t.time_id
 16  GROUP  BY
 17         t.calendar_year
 18  ,      c.cust_name
 19  ;
```

```
11604 rows selected.
```

执行完该报告后，我可以检索最新的计数器来查看函数执行了多少次，如清单 9-9 所示。请注意，你可以参考 Apress 网站上的源代码，了解 `COUNTER` 包的实现细节及其在 `FORMAT_CUSTOMER_NAME` 函数中的使用。

**清单 9-9.** CBO 查询转换与函数执行（显示函数调用次数）

```sql
SQL> exec counter.show('Function calls');
```

```
Function calls: 918843

PL/SQL procedure successfully completed.
```

PL/SQL 函数已被执行超过 918,000 次，尽管我的本意是内联视图会将函数执行次数限制在仅 55,500 次（即 CUSTOMERS 表每行一次）。然而，这并未奏效，因为 CBO 直接将内联视图与外部查询块合并了，结果我的 PL/SQL 函数被应用到了比我预期大得多的行源上。清单 9-10 中的执行计划显示，内联视图已被分解（即被合并），并且投影信息清楚地表明函数调用被应用在计划的最后一步。

**清单 9-10.** CBO 查询转换与函数执行（PL/SQL 函数的投影）

```text
----------------------------------------------------
| Id  | Operation                     | Name       |
----------------------------------------------------
|   0 | SELECT STATEMENT              |            |
|   1 |   HASH GROUP BY               |            |
|   2 |    HASH JOIN                  |            |
|   3 |     PART JOIN FILTER CREATE   | :BF0000    |
|   4 |      TABLE ACCESS FULL        | TIMES      |
|   5 |     HASH JOIN                 |            |
|   6 |      TABLE ACCESS FULL        | CUSTOMERS  |
|   7 |      PARTITION RANGE JOIN-FILTER|            |
|   8 |       TABLE ACCESS FULL       | SALES      |
----------------------------------------------------

Column Projection Information (identified by operation id):
-----------------------------------------------------------

   1 - "T"."CALENDAR_YEAR"[NUMBER,22],
       "FORMAT_CUSTOMER_NAME"("CUST_FIRST_NAME","CUST_LAST_NAME")[4000],
       SUM("S"."AMOUNT_SOLD")[22], SUM("S"."QUANTITY_SOLD")[22]
   2 - (#keys=1) "T"."CALENDAR_YEAR"[NUMBER,22],
       "CUST_LAST_NAME"[VARCHAR2,40], "CUST_FIRST_NAME"[VARCHAR2,20],
       "S"."AMOUNT_SOLD"[NUMBER,22], "S"."QUANTITY_SOLD"[NUMBER,22]
   3 - "T"."TIME_ID"[DATE,7], "T"."TIME_ID"[DATE,7],
       "T"."CALENDAR_YEAR"[NUMBER,22]
   4 - "T"."TIME_ID"[DATE,7], "T"."CALENDAR_YEAR"[NUMBER,22]
   5 - (#keys=1) "CUST_LAST_NAME"[VARCHAR2,40],
       "CUST_FIRST_NAME"[VARCHAR2,20], "S"."AMOUNT_SOLD"[NUMBER,22],
       "S"."TIME_ID"[DATE,7], "S"."QUANTITY_SOLD"[NUMBER,22]
   6 - "CUST_ID"[NUMBER,22], "CUST_FIRST_NAME"[VARCHAR2,20],
       "CUST_LAST_NAME"[VARCHAR2,40]
   7 - "S"."CUST_ID"[NUMBER,22], "S"."TIME_ID"[DATE,7],
       "S"."QUANTITY_SOLD"[NUMBER,22], "S"."AMOUNT_SOLD"[NUMBER,22]
   8 - "S"."CUST_ID"[NUMBER,22], "S"."TIME_ID"[DATE,7],
       "S"."QUANTITY_SOLD"[NUMBER,22], "S"."AMOUNT_SOLD"[NUMBER,22]
```

视图合并、子查询非嵌套以及其他复杂的查询转换并不是唯一可能影响 PL/SQL 函数执行次数的 CBO 相关问题。CBO 也可以自由忽略你的表或谓词列出的顺序（尽管对于后者有一个显著的例外，专门针对 PL/SQL 函数，我稍后会描述）。连接和谓词的排序会对执行计划每一步的基数产生巨大影响；这反过来可能会导致你的 PL/SQL 函数执行次数比你预期的更多，或者更少。



##### 内部优化

除了众多的 CBO 优化之外，Oracle 数据库还具有一系列在运行时应用的内部优化，试图揣测这些优化的行为往往徒劳无功。它们不仅可能随着数据库版本而变化，还可能取决于多种因素，其中一些您可以推导出来，而另一些则对您隐藏。

Oracle 数据库对 `DETERMINISTIC` 函数的优化就是这样一个例子。因为根据定义，确定性函数对于特定的输入值总是返回相同的结果，所以如果遇到相同的输入，Oracle 数据库可以避免重新执行 PL/SQL 函数。然而，这种优化的效率取决于其他因素，例如数据类型、数据长度、基数（即不同值的数量）、数据库版本，甚至环境设置（如数组提取大小）。关于这些因素如何导致函数执行次数大幅变化的精彩讨论，请参阅 Dom Brooks 在 [`http://orastory.wordpress.com`](http://orastory.wordpress.com) 上的“[*确定性判定*]”系列文章。

当 PL/SQL 函数调用被封装在标量子查询中时（一种减少函数在 SQL 中影响的常用技术，我将在本章后面演示），也会看到类似的行为。虽然标量子查询缓存似乎比确定性函数的优化更有效，但它仍然依赖于一些相同的因素（如数据类型、长度和顺序），此外还取决于内部缓存的大小（因数据库版本而异）。更深入地描述支持标量子查询缓存的内部哈希表，请参阅 Jonathan Lewis 的 *Cost-Based Oracle Fundamentals* (Apress, 2005)。这个哈希表相当小的事实意味着，除了最简单的输入值集合外，冲突（即缓存“未命中”）几乎不可避免。由于您无法预测这些冲突何时发生，因此您应该接受这样一个事实：使用标量子查询缓存通常可以获得*一些*好处，但无法精确量化其程度。

#### 次优数据访问

当 PL/SQL 函数包含 SQL 语句时会发生什么？在大多数大型数据库应用程序中都可以找到一种极其常见的设计模式，即将 SQL 查询包装在 PL/SQL 函数中。通常，此类“getter”函数背后的原则是合理的（即封装常见查找、可重用的数据访问方法等），但此类设计决策的性能影响可能是毁灭性的，尤其是当 PL/SQL 函数从 SQL 语句中调用时。

当 SQL 语句被包装在 PL/SQL 函数中并从 SQL 查询调用时，上下文切换的成本会自动加倍——现在在函数内部和外部都会发生。然而，更关键的是，通过将 SQL 查询包装在函数中，如果查询位于调用 SQL 语句的主体中，CBO 本可以拥有的优化选择实际上被禁用了。这对您的查询经过时间可能产生严重影响（可以说，在这些场景下，上下文切换的成本相形见绌）。

考虑 清单 9-11 中的示例。我有一个 PL/SQL 函数，它返回给定日期两种货币之间的汇率。请注意，为了示例的目的，我故意保持了简单性。

***清单 9-11.** 将 SQL 查询封装在 PL/SQL 函数中（创建 PL/SQL 函数）*

```sql
SQL> CREATE FUNCTION get_rate(
  2                    p_rate_date IN rates.rate_date%TYPE,
  3                    p_from_ccy  IN rates.base_ccy%TYPE,
  4                    p_to_ccy    IN rates.target_ccy%TYPE
  5                    ) RETURN rates.exchange_rate%TYPE AS
  6     v_rate rates.exchange_rate%TYPE;
  7  BEGIN
  8     SELECT exchange_rate INTO v_rate
  9     FROM   rates
 10     WHERE  base_ccy   = p_from_ccy
 11     AND    target_ccy = p_to_ccy
 12     AND    rate_date  = p_rate_date;
 13     RETURN v_rate;
 14  END get_rate;
 15  /
Function created.
```

在清单 9-12 所示的按产品和日历年份的销售报告中，我使用了此函数来计算以 GBP 和 USD 计价的收入。请注意，我使用了 Autotrace 来抑制结果集，并启用了 SQL*Plus 计时器。

***清单 9-12.** 将 SQL 查询封装在 PL/SQL 函数中（使用 PL/SQL 的 SQL）*

```sql
SQL> SELECT t.calendar_year
  2  ,      p.prod_name
  3  ,      SUM(s.amount_sold)                                      AS amt_sold_usd
  4  ,      SUM(s.amount_sold * get_rate(s.time_id, 'USD', 'GBP')) AS amt_sold_gbp
  5  FROM   sales     s
  6  ,      products  p
  7  ,      times     t
  8  WHERE  s.prod_id = p.prod_id
  9  AND    s.time_id = t.time_id
 10  GROUP  BY
 11         t.calendar_year
 12  ,      p.prod_name
 13  ;

272 rows selected.

Elapsed: 00:01:05.71
```

这个查询的经过时间大约是 66 秒——对于一个少于 100 万行的聚合查询来说，这是一个糟糕的响应时间。此语句的 Autotrace 输出见清单 9-13。

***清单 9-13.** 将 SQL 查询封装在 PL/SQL 函数中（使用 PL/SQL 的 SQL 的 Autotrace 输出）*

```sql
Statistics
----------------------------------------------------------
     918848  recursive calls
         12  db block gets
    1839430  consistent gets
          0  physical reads
       3196  redo size
      16584  bytes sent via SQL*Net to client
        449  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
        272  rows processed
```



## **清单 9-14.** 在 PL/SQL 函数中封装 SQL 查询（纯 SQL 解决方案）

```sql
SQL> SELECT t.calendar_year
  2  ,      p.prod_name
  3  ,      SUM(s.amount_sold)                 AS amt_sold_usd
  4  ,      SUM(s.amount_sold*r.exchange_rate) AS amt_sold_gbp
  5  FROM   sales      s
  6  ,      times      t
  7  ,      products   p
  8  ,      rates      r
  9  WHERE  s.time_id        = t.time_id
 10  AND    s.prod_id        = p.prod_id
 11  AND    r.rate_date  (+) = s.time_id
 12  AND    r.base_ccy   (+) = 'USD'
 13  AND    r.target_ccy (+) = 'GBP'
 14  GROUP  BY
 15         t.calendar_year
 16  ,      p.prod_name
 17  ;
```
```text
272 行已选择。

已用时间: 00:00:00.97
```

性能差异惊人！通过移除 PL/SQL 函数并直接连接到 `RATES` 表，响应时间已从 66 秒降至仅 1 秒。

**注意：** PL/SQL 函数在 SQL 中调用时有一个不寻常的“特性”：未处理的 `NO_DATA_FOUND` 异常会静默返回 `NULL`，而不是传播错误。我在清单 9-11 的 `GET_RATE` 函数中没有显式处理 `NO_DATA_FOUND` 异常，因此即使我查找的汇率不存在，我在清单 9-12 中的查询也不会失败。相反，该函数将返回 `NULL`，我的查询在行数上将是无损的。为了在清单 9-14 的纯 SQL 查询版本中模拟这一点，我只是对 `RATES` 表使用了外连接。

纯 SQL 报告的 Autotrace 统计信息显示在清单 9-15 中。

## **清单 9-15.** 在 PL/SQL 函数中封装 SQL 查询（纯 SQL 报告的 Autotrace 输出）

```text
统计信息
----------------------------------------------------------
          0  递归调用
          0  数据库块读取
       1758  一致性读取
          0  物理读取
          0  重做大小
      16584  通过 SQL*Net 发送到客户端的字节数
        425  通过 SQL*Net 从客户端接收的字节数
          2  SQL*Net 往返次数
          0  排序（内存）
          0  排序（磁盘）
        272  处理的行数
```

Autotrace 输出突显了满足两条 SQL 语句所需逻辑 I/O 的显著差异。PL/SQL 函数产生了超过 180 万次 `一致性读取` 的开销。清单 9-16 中的 TKProf 报告清楚地显示了在函数中封装汇率查找的成本。

## **清单 9-16.** 在 PL/SQL 函数中封装 SQL 查询（tkprof 输出）

```sql
SELECT t.calendar_year
,      p.prod_name
,      SUM(s.amount_sold)                                     AS amt_sold_usd
,      SUM(s.amount_sold * get_rate(s.time_id, 'USD', 'GBP')) AS amt_sold_gbp
FROM   sales      s
,      products   p
,      times      t
WHERE  s.prod_id = p.prod_id
AND    s.time_id = t.time_id
GROUP  BY
       t.calendar_year
,      p.prod_name
```
```text
调用        次数       CPU      耗时       磁盘      查询    当前        行数
------- ------  -------- ---------- ---------- ---------- ----------  ----------
解析        1      0.00       0.00          0          0          0           0
执行        1      0.00       0.00          0          0          0           0
获取        2    105.53     106.38          0       1734          0         272
------- ------  -------- ---------- ---------- ---------- ----------  ----------
<输出已截断>

********************************************************************************
SQL ID: ajp04ks60kqx9 执行计划哈希值: 139684570

SELECT EXCHANGE_RATE
FROM
 RATES WHERE BASE_CCY = :B3 AND TARGET_CCY = :B2 AND RATE_DATE = :B1

调用        次数       CPU      耗时       磁盘      查询    当前        行数
------- ------  -------- ---------- ---------- ---------- ----------  ----------
解析        0      0.00       0.00          0          0          0           0
执行   918843     12.50      13.85          0          0          0           0
获取    918843      9.74      10.86          0    1837686          0      918843
------- ------  -------- ---------- ---------- ---------- ----------  ----------
```

TKProf 报告显示，PL/SQL 函数中的 SQL 被视为一个独立的查询；在本例中，它占用了销售查询产生的所有额外逻辑 I/O 以及几乎一半的额外运行时间（如果你好奇本例中剩余的运行时间花在了哪里，分层性能分析器报告显示，总共约有 59 秒花在了 PL/SQL 引擎整体上。其中约 47 秒的 PL/SQL 时间由汇率查找 SQL 本身承担——是上述 SQL 跟踪报告数字的两倍——这意味着大约 23 秒是 PL/SQL 开销）。基础查询本身的逻辑 I/O 开销很小，实际上与纯 SQL 实现相当。

这种分离突显了一个重要观点。如果嵌入的 SQL 被视为一个独立的查询，CBO 就无法在外层查询的上下文中对其进行优化。CBO 完全不知道函数实际做了什么，因此 `RATES` 表甚至没有进入 CBO 的决策考量。结果，如你所见，可能是灾难性的。

因此，你可以看到，从性能角度看，将 PL/SQL 查找函数转换为 SQL 是非常有意义的。如果性能至关重要，并且此类查找函数在你的应用程序中被大量使用，建议你尽可能移除它们。我将在本章后面演示一些替代的封装方法，可以帮助你解决这个问题。

然而，我认识到要“扫除”一个遗留应用程序以转换其 SQL 并不总是可行（它可能嵌入许多不同的应用层和技术中，或者回归测试开销可能过于巨大）。在这种情况下，有一些方法可以用来降低 PL/SQL 函数的成本，我也会在后面展示其中一些。

### 优化器的难题

到目前为止的许多示例展示了 PL/SQL 函数在 SQL 语句的 `SELECT` 块（投影）中被调用，但在谓词中使用用户编写的函数同样常见。这可能引发一系列问题。首先，正如我之前演示的，在谓词中使用的 PL/SQL 函数将执行不可预测的次数。其次，应用于索引列的 PL/SQL 函数（或任何函数）将阻止索引的使用。第三，CBO 对 PL/SQL 函数知之甚少，因为它通常没有可用的统计数据。这可能导致次优的执行计划。我将在这里集中讨论第三个问题，因为我在前面已经描述了第一个问题，而第二个问题在 Oracle DBA/开发者社区中已被充分理解。



