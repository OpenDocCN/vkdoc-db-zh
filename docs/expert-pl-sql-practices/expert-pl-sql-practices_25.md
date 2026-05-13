# 本章将（以及不会）涵盖的内容

从架构师的视角（我指的是从设计正确性、代码正确性和标准符合性的角度）来看，当一个应用程序或程序单元投入生产时，绝对不应该有任何意外。所有设计都应该经过同行的早期评审——以确保符合业务需求和设计标准。所有代码都应该经过审查，以确保符合编码标准、性能标准、逻辑错误和基本文档。代码越符合标准，应用程序测试越充分，出现“哎呀”类错误的可能性就越小。无论应用程序多么符合标准，你都可能发现逻辑错误。测试应该能捕获大部分此类错误，但通常仍有一些会漏网。然而，漏网的错误不应该是常见错误。这些常见错误正是标准和回归测试旨在捕获的。

### 总体目标

本章将为你提供工具，通过利用 Oracle 以数据字典、性能剖析和 `PL/SQL compiler` 形式提供的宝贵数据，帮助你做出最佳决策。许多验证工作可以自动化，本章将说明在何处可以实现这一点。任何可以卸载给计算机完成的工作都应该这样做。让人类专注于那些需要人类智能的任务。即使完整的分析/发现无法自动化，至少数据的收集可以自动化或简化。

## 自动化与手动验证

有些事情不那么适合自动化。即使在今天，许多审查工作仍然需要人工验证。如果我能开发出一种自动化工具来捕获任何可能应用程序中的每一个逻辑错误，那将是非常有价值的东西，可以用来赚大钱。逻辑审查仍然属于人工执行手动检查的范畴，无论是通过代码审查还是通过测试。

文档也很棘手。很容易看到代码中有注释，但很难验证某个注释是否有用（甚至是否相关）。逻辑和注释都高度特定于手头的任务。本章不会试图涵盖逻辑或文档问题，因为 Oracle 在这方面能提供的帮助很少。

## 性能与分析

了解应用程序的性能剖析对应用程序的成功至关重要。然而，其缺点是，性能检查需要人类智能来验证。虽然你可以自动化一个系统，在某个函数执行时间超过 x 秒时发出警报，但需要人工来验证这是否可以接受。

Oracle 提供了工具来帮助收集分析系统所需的信息，使开发人员能够做出明智的性能决策。调整应用程序远远超出了本章的范围。然而，第 13 章 提供了解释性能剖析结果的方法。

本章不会提供调优建议，而是从了解代码与性能关系的角度，提供以下内容：有哪些工具可用于收集这些性能指标（主要是剖析器）、数据存储在何处，以及如何将这些数据用于回归。回归测试是验证变更前正常工作的功能在变更后是否仍然有效的最简单方法。通过保存 Oracle 提供的数据，你可以长期关注性能表现，并比较不同版本之间的差异。你可以在测试中设定阈值，使得剖析器的自动运行仅在出现异常时才发出警报。人类必须在第一次运行时确保性能结果正确，但一旦收集到良好的度量标准，只有当某个度量标准超出阈值时才需要人工干预。

## 测试与覆盖

除了回归测试，另一个测试好处是代码覆盖率。虽然并非绝对可靠，但代码覆盖率测试确实提供了一定保证，即代码至少会针对某些数据集执行。结合使用 Oracle 提供的工具和一些努力，可以验证程序中的每一行可执行代码都已被执行。这很有价值，但更有价值的是能够看到哪些行未被执行。这正是你希望尽可能自动化的地方。如果每一行都执行了，测试通过。如果并非每一行都执行了，测试失败。人工介入并创建一个或多个测试来覆盖每一行。

## 标准与自定义

本章大部分内容将使用数据字典和 `PL/SQL compiler` 提供的数据来确保符合公司标准。标准几乎从来不是“标准”的，因此“一刀切”的解决方案是不可能的。相反，本章将为你提供工具和指导，让你能够根据需要创建自己的解决方案，无论详细程度如何。使用简单的 `SQL` 和数据字典，你可以验证命名标准、查找未使用的变量、查找定义在多个作用域的变量、执行影响分析、查找不可接受的数据类型用法等等。

## 章节范围

我不会解释这些工具提供的每一个选项、列或报告。如果 Oracle 的文档很好（通常确实很好），我不会去复述它。本章不是关于这些工具的通用教程。它是关于你如何将这些工具整合到你的开发工作流程中，以实现更好应用程序的解释。


### 自动化代码分析

代码分析，也称为程序分析，是对源代码进行的分析。手动代码分析是由人进行的代码审查。此类分析还可能包括审查来自运行程序的日志数据和系统输出。自动化分析可以让分析师（根据我的经验，通常是开发人员）跳过分析中枯燥的部分，直接进入更有趣的工作。你不再需要逐行人工审查，而是可以运行某种程序来为你完成这些繁琐的工作。

让我们举个例子。假设你是一名开发人员，正在工作中。你接到一些新的需求，这意味着你必须修改一个现有的过程。可能是某个计算逻辑发生了变化，这个过程需要增加一个变量。业务部门希望你评估一下完成这项改动所需的时间。更有趣的是，编写并维护这个过程多年的那位同事最近中了彩票辞职了。你甚至从未见过这段代码（在代码审查那天你请病假了）。

你如何提供这个估算？凭空猜测一个数字？根据我的经验，我见过不少这种方法，但从未真正成功过。

我见过的第二种方法是手忙脚乱地找出源代码，进行紧急审查。快速浏览后，你认定这是个简单的改动，只需两个小时来编码和单元测试。

很好。快速分析之后，你准备开始编码了。一位业务人员对你能在 673 处调用此计算的地方（这是一个*非常*流行的计算）全部修改调用感到惊讶。现在你有点担心了。你在文件系统中快速执行了 `grep` 命令，查看所有调用该过程的地方。

意识到你的过程被如此多的地方调用后，你将最初的估算时间乘以二。这通常比不乘更准确一些，但还有更好的方法。如果我说你可以通过运行几个简单的查询就能找到所有调用它的地方，你会相信吗？这些查询不仅能找到函数被调用的位置，还能涵盖影响分析、命名标准合规性检查，并展示代码和变量使用情况、作用域冲突以及测试的代码覆盖率。

你可以通过手动审查源代码、程序日志数据等获取所有这些信息，但通过自动化这个过程，你将获得一个可重复、可预测、不易出错且大大节省时间的流程。在解读自动化或手动分析提供的数据方面，优秀的开发人员是无法被替代的，但自动化能为你节省大量时间。

### 静态分析

静态分析是指在不实际运行程序的情况下分析其源代码。你关注的是编译器在编译时已知的信息。静态分析根据程序中实际使用的功能提供有关程序的信息。

静态分析可以提供有关正在使用的数据类型、命名标准、安全性、访问的对象等方面的信息。静态分析可以直接利用源代码，也可以使用工具。如果你手动审查代码以确保符合命名标准，你就是在对该源代码进行手动静态分析。同行评审就是一种静态分析。如果你曾在审查同事的程序时发现一个 `SQL` 注入漏洞，你就理解了静态分析的重要性。

在 `Oracle PL/SQL` 中，有多种途径可以获取分析所需的信息。你通常会从数据字典开始。随着每个版本的发布，Oracle 都会用有价值的数据增强数据字典。稍后，在“执行静态分析”章节，我将带你了解一些重要的数据字典视图（以及一些分析查询），然后向你介绍一个将代码分析带入 21 世纪的新数据字典视图。这个新视图由 `PL/Scope` 提供，它是 `11g` 版本中增加的编译器功能。

### 动态分析

动态分析告诉你程序运行时会发现什么。动态分析也称为程序性能剖析。通过性能剖析，你运行一个程序，并以侵入式或非侵入式的方式收集程序执行路径和行为的统计数据。

`DBMS_PROFILER` 和 `DBMS_TRACE` 是 `11g` 版本之前的性能剖析工具。`DBMS_PROFILER` 是一个平面性能剖析器。平面性能剖析器提供简单的时间信息，例如不同的过程和函数调用耗时，但不包含上下文信息，比如哪个函数被调用以及调用顺序是什么。Oracle 通过 `DBMS_TRACE` 添加了调用上下文信息，但最终创建的工具具有侵入性（需要 `DEBUG` 权限）且输出繁杂（可能产生大量数据）。使用 `DBMS_PROFILER` 要好得多。

`DBMS_HPROF` 是 `11g` 版本及之后提供的一个层次化性能剖析工具，它非侵入且易于使用。`DBMS_HPROF` 像平面剖析器 `DBMS_PROFILER` 一样提供时间信息，但也包含了像 `DBMS_TRACE` 那样的上下文信息。它可以生成一些对性能调优非常有价值的报告，同时也会填充一些表，你可以随后对这些表编写查询。这对于随时间比较指标以及作为理解代码的额外工具非常有益。


### 何时进行分析？

分析代码的整个目的，是为了理解你的代码。绝大多数使用这些分析工具的人（至少那些向我询问如何使用这些工具的人），都是在紧要关头才使用：一次升级失败、用户抱怨数据库缓慢、一个似乎隐藏起来的棘手逻辑错误。

如同性能调优一样，代码分析可以在出问题时被召唤。但您难道不认为，一个为性能而设计的系统，更少可能在半夜把您吵醒吗？对于一个经过良好分析、符合标准的系统也是如此：无论是静态（人工或自动化）还是动态分析。这两种分析都应该建立基线。

分析应该在编写应用程序时（或在您接手它时）进行。我毫不怀疑每个人都熟悉代码评审。如前所述，代码评审（或同行评审）是一种人力驱动的静态分析。而 Oracle 提供的自动化分析（尤其是 `PL/Scope`）应被视为代码评审的延伸。

这种生产前分析/评审应当被记录、进行版本控制并保留。自动化分析作为数据字典视图中的数据，非常容易存档，并且可以在需要时随时召回（尽管您需要自己构建该功能）。随着每一次变更，都需要收集并保存分析的新版本。除了保存分析结果，您还需要查看它。您真的应该研究它。

通过静态分析（正如在讨论数据字典和 `PL/SCOPE` 时所提到的），您将理解代码的依赖关系、代码使用的标识符，以及这些标识符在何处/如何被使用。通过动态分析，您可以看到代码中的瓶颈（可修复或不可修复）；您可以了解不同的输入数据如何改变代码的执行轮廓；您可以验证代码覆盖率；等等。

使用各种工具提供的报告（或您自己用 SQL 编写的），您将为调试任何可能在半夜出现的问题打下良好基础。您还将拥有可以与新开发人员共享的文档（或用来说服管理层您确实在做工作）。

一旦您的代码投入生产，您将能够利用旧信息来验证一切是否仍按预期运行。在开发过程中，您将使用性能分析器持续验证您的代码是否工作正确，无论是从代码路径角度还是性能角度。无需将每次分析器运行都保存到版本控制系统，但您应该保存在每次实施或升级前运行的版本。

### 代码分析与插桩

有时会在分析与插桩之间产生混淆。术语 `插桩` 指的是为支持分析（特别是动态分析）而添加到系统中的代码。插桩使分析成为可能，但它本身并不是分析。

要决定应用程序需要哪种插桩（以及它是否需要插桩）或进行任何性能调优，您都必须（在某种程度上）分析代码。性能分析器是一种插桩，对于调优很有价值。同样地，理解应用程序或代码库的性能轮廓对于真正理解您的代码至关重要。应用程序将如何处理各种负载、数据输入等等？这些问题的答案是您在分析过程中想要捕捉的一些东西。

我讲这些是为了说明，没有哪种分析是在真空中完成的。执行代码分析是为了确保符合您想要遵循或被迫遵循的标准。一个无法运行的程序毫无价值。一个无法维护的程序同样毫无价值。在程序的生命周期中，绝大多数时间（和金钱）都将花在维护上。可维护性是在设计时决定的。

插桩是一种工具，在应用程序的整个生命周期中使用，用于记录有趣的信息。代码分析是一种工具，在程序投入生产运行之前，提供关于程序的有趣信息。两者对于成功、高性能、可维护的应用程序都至关重要。

### 执行静态分析

您可以使用两种资源来执行静态分析。首先，您可以使用数据字典。每个数据库，无论版本如何，都实现了一个数据字典，您可以从中获得的信息对于分析您的代码非常有帮助。字典是您的第一道防线，并且它始终可用。

如果您运行的是 Oracle Database 11g 或更高版本，您还可以使用一个名为 `PL/Scope` 的功能。它使您能够看到原本无法从数据字典获得的编译时数据。我将在接下来的两个小节中讨论这两种资源——数据字典和 `PL/Scope`。

#### 数据字典

既然您在阅读本书，我假设您至少对 Oracle 的数据字典有些了解。如果不是，您可能在职业生涯的这个阶段阅读本章还为时过早。我不打算解释数据字典是什么；有很多其他书籍涵盖了这个主题。

但是，我将讨论与理解您的代码相关的视图。我不讨论 `DBA_*`、`ALL_*` 和 `USER_*` 的区别。在本次讨论中我将使用 `USER_*`，但您可以假定对应的 `ALL` 和 `DBA` 版本也存在。同样，我不会对这些视图做大量的 describe 操作。我将仅涵盖与本次讨论相关的列。

![images](img/square.jpg) `注意` 我提供的示例使用了部分字符串和字符串格式。在真实场景中，实现这些示例所示概念时，诸如 select 条件、where 子句中的内容、格式字符串等，应该存在于表中，而不是硬编码。为简洁起见，我将这些值硬编码到示例中。此外，在真实实现中，您可能会使用正则表达式来缩小查询结果范围。然而，我坚持使用常规的 PL/SQL 函数，因为大多数读者会更熟悉它们。

如果您编译了本章提供的示例代码（可从本书在 `apress.com` 的目录页面获取），您将拥有我将作为示例结果展示的基本程序集。如果您拥有该代码，将更容易跟随正文。不过，如果您在自己的模式上运行它，看看会得到什么结果，可能会更有趣。

## USER_SOURCE

这个视图自 PL/SQL 还处于**石刻时代**就已存在，`USER_SOURCE`视图非常简单，就是你代码的逐行列表。如果你想浏览或导出代码（有点像手动查看），`USER_SOURCE`就是适合你的视图。我曾使用这些数据对一段代码进行逆向工程（现在`DBMS_METADATA`是更好的方法），并且我经常使用它来快速分析某个对象在何处被使用。它非常简单，就是源代码的逐行副本。

以下是一个使用`USER_SOURCE`提取包含特定内容（本例中为“`jobs_api.validate_job`”）的代码行的示例：

```sql
SELECT name, line
  FROM user_source
  WHERE lower(text) like '%jobs_api.validate_job%';
```

```
NAME                           LINE
------------------------------ ----------------------
EMPLOYEE_API                   40
TEST_CALLS_FNC                  5
TEST_CALLS_PRC                  4
TEST_CALLS_PRC                  8
```

这个查询曾经是数据库中代码分析的基本方法。它仍然适用于对数据库进行快速查找，但如果你看看这种查询的用例，你会发现通过使用其他视图可以获得更好的信息。

示例中的查询试图查找所有调用`JOBS_API.VAIDATE_JOB`程序单元的程序。不幸的是，该查询也会返回所有在注释中包含“`jobs_api.validate_job`”的调用。`USER_SOURCE`的主要问题在于它只是一个列表。我需要解析代码以丢弃注释（当你逐行查看代码时，注释可能隐藏在`/* */`中），而且我需要解析和分离各种类型的变量、可执行语句等，以便从结果中获得任何真正的信息。我个人不想维护自己的 PL/SQL 解析器。

## USER_DEPENDENCIES

当你想查看谁调用了谁时，`USER_DEPENDENCIES`是更好的方法。不幸的是，该视图粒度不够细，并且不允许动态依赖关系，例如`DBMS_SQL`或本地动态 SQL。分析人员必须具备特定的代码知识或手动审查代码。以下查询和结果显示了当前模式中的代码在何处引用了`JOBS_API`包：

```sql
SELECT name, type
FROM user_dependencies
WHERE referenced_name = 'JOBS_API';
```

```
NAME                           TYPE
------------------------------ ------------------
TEST_CALLS_PRC                 PROCEDURE
TEST_CALLS_FNC                 FUNCTION
JOBS_API                       PACKAGE BODY
EMPLOYEE_API                   PACKAGE BODY
```

通过这些结果，你可以确切地看到哪些程序单元真正在调用`JOBS_API`，但该视图只在包级别提供。你不知道包内的哪个过程发出了调用，也不知道程序单元被调用了多少次。

![images](img/square.jpg) **注意** Oracle 正在添加细粒度依赖关系，未来可能会提供一个视图来给我们缺失的细粒度详细信息。

我讨论的其余视图都是使用存储在`USER_SOURCE`视图中的源代码（由编译器在编译时）创建的。要查看实际代码（在分析代码时经常需要），你可以回连到这个视图。

## USER_PROCEDURES

对于许多目的，`USER_PROCEDURES`视图是比`USER_SOURCE`更好的了解一段代码的视图。该视图将提供有关代码的编译详细信息，例如它是何种类型的程序（函数、过程、包等），以及它是否是聚合的、确定性的等。可以说，与代码分析相关的最重要特性是，你可以将`USER_PROCEDURES`与下一节讨论的字典视图`USER_ARGUMENTS`结合起来，以获取程序所用参数的详细信息。

`USER_PROCEDURES`视图是分层的，因此你不仅可以识别所有程序，还可以识别包中存在哪些过程和函数。你还可以在此视图中看到实现方法的对象类型（关于对象和方法稍后详述）。

与`USER_ARGUMENTS`一样，`USER_PROCEDURES`包含与数据库中任何代码对象相关的信息。这意味着你可以找到与包、过程、函数、触发器和类型相关的信息。我将这些代码对象统称为程序和程序单元。我后面介绍的其他数据字典视图也是如此。

`USER_PROCEDURES`一开始可能有点令人困惑。`OBJECT_NAME`列标识对象。除非你查看的是存在于另一个对象（`PACKAGE`或`TYPE`）命名空间内的对象，否则`PROCEDURE_NAME`为空。例如，当你从`USER_PROCEDURES`中选择数据时，对于独立过程或函数，`PROCEDURE_NAME`将为空，而`OBJECT_NAME`将是该过程或函数的名称。对于包或类型，`PROCEDURE_NAME`将为空（对于包或类型对象），而`OBJECT_NAME`将是包或类型的名称。但是，每个包或类型在视图中还会有另一个条目，对应于每个函数、过程或方法。在这种情况下，`OBJECT_NAME`将是包的名称，`PROCEDURE_NAME`将是过程或函数（对于对象类型是方法）的名称。

以下查询显示，即使对象是一个过程，`PROCEDURE_NAME`列也为空。这是因为过程名称仅对包含多个程序单元（如包和类型）的对象进行填充。

```sql
select object_name, procedure_name, object_type
  from user_procedures
  where object_name = 'SECURE_DML';
```

```
OBJECT_NAME     PROCEDURE_NAME    OBJECT_TYPE
--------------- ----------------- -------------------
SECURE_DML                        PROCEDURE
```

另一方面，下一个查询显示，如果一个对象是包或类型，`OBJECT_NAME`是包的名称，`PROCEDURE_NAME`将包含包内过程或函数的名称：

```sql
select object_name, procedure_name, object_type
  from user_procedures
  where object_name = 'DEPARTMENTS_API';
```

```
OBJECT_NAME      PROCEDURE_NAME          OBJECT_TYPE
---------------- ----------------------- -------------------
DEPARTMENTS_API  DEPARTMENT_EXISTS       PACKAGE
DEPARTMENTS_API  DEPARTMENT_MANAGER_ID   PACKAGE
DEPARTMENTS_API  DEPARTMENT_MANAGER_ID   PACKAGE
DEPARTMENTS_API                          PACKAGE
```

如果一个对象是用户定义聚合或管道函数的实现，`IMPLTYPEOWNER`和`IMPLTYPENAME`将是其底层基类型。下一个查询显示，内置聚合函数`SYS_NT_COLLECT`是实现一个类型的函数示例，而内置的`DM_CL_BUILD`是实现一个类型的管道函数。

```sql
select object_type, aggregate, pipelined, impltypeowner, impltypename
  from all_procedures
  where owner = 'SYS'
    and (object_name = 'SYS_NT_COLLECT'
        or
        object_name = 'DM_CL_BUILD');
```


`对象类型 聚合 管道化 实现类型所有者       实现类型名称`
`------------ --------- --------- --------------- ------------------------------`
`函数         是        否        SYS             SYS_NT_COLLECT_IMP`
`函数         否        是        SYS             DMCLBIMP`

此视图中的每个对象都有一个 `OBJECT_ID` 和一个 `SUBPROGRAM_ID`。`OBJECT_ID` 可被视为此视图的主键（实际上是字典中对象的主键）。当与其他字典对象连接时（稍后我将用 `USER_ARGUMENTS` 做示例），`OBJECT_ID` 将是我用于连接的唯一 ID。如果代码是独立的，例如一个存储过程或函数，`SUBPROGRAM_ID` 始终为 1。如果代码属于一个包，则 `subprogram_id` 将为 0，包中的每个存储过程和函数将使 `subprogram_id` 递增 1。

`OVERLOAD` 列对于理解一段代码非常有用。如果在一个包中重载了一个存储过程或函数，`OVERLOAD` 将从 1 开始，并且对于该重载子程序的每个迭代递增 1。`SUBPROGRAM_ID` 始终递增，因此一个重载的存储过程将与该存储过程的重载版本具有不同的 `SUBPROGRAM_ID`。

结合 `USER_ARGUMENTS` 视图，你可以准确地查询重载存储过程的区别。我将在下一节提供一个示例。单独来看，`USER_PROCEDURES` 视图可用于命名标准检查，以及检查你可能感兴趣的编译信息，例如 `PIPELINED`、`AUTHID` 或 `DETERMINISTIC`。

### USER_ARGUMENTS

当你将 `USER_ARGUMENTS` 视图纳入考虑时，你可以开始查找有关代码及其参数的精细细节。刚开始使用时，查询这些数据可能会令人困惑。其对象命名方式与 `USER_PROCEDURES` 视图截然不同。

在此视图中，`OBJECT_NAME` 是拥有参数的程序的名称。无论该程序是独立的，还是位于包或类型中，存储过程或函数的名称都将出现在 `OBJECT_NAME` 字段中。如果 `PROCEDURE` 或 `FUNCTION` 位于包中，`PACKAGE_NAME` 将有值——否则将为 `NULL`。`ARGUMENT_NAME` 是参数的实际名称。

请注意，在 `USER_PROCEDURES` 中，`object_name` 是顶级对象。在包的情况下，它将是包或类型名称。对于独立代码，它将是存储过程名称、触发器名称或函数名称。在 `USER_ARGUMENTS` 中，`object_name` 是子程序名称。它始终是存储过程、函数或触发器的名称。在包的情况下，`object name` 将是子程序，而 `package name` 将是包的名称。这对于连接这两个表非常重要（请参见下一个查询示例）。

如果参数是一个锚定类型（`EMPLOYEES.EMPLOYEE_ID%TYPE`），你将看不到锚定类型。你会看到基础数据类型（例如 `NUMBER`）作为参数类型。但是，如果代码是使用锚定类型创建的，除了数据类型外，你还会看到该锚定值的精度和范围。通常你无法在参数中指定精度或范围，但由于 Oracle 知道基础对象的精度和范围，因此可以跟踪该信息。如果你想知道是*哪个*锚定类型，这并没有真正帮助。尽管如此，可用的信息仍然有用。

作为数据库架构师，我关注的一个重要问题是标准合规性。命名是否正确？是否使用了正确的数据类型？以下示例是我在 PL/Scope 出现之前使用的一个查询。它是确保程序单元参数命名标准合规性的一种简单方法。

```sql
select package_name, object_name, argument_name,
          position, in_out, data_type
  from user_arguments
  where argument_name not like 'P_%';
```

```
PACKAGE_NAME  OBJECT_NAME  ARGUMENT_NAME  POSITION  IN_OUT DATA_TYPE
------------- ------------ -------------- --------- ----------- -----------------------
JOBS_API      JOB_EXISTS   JOB_TITLE      1         IN         VARCHAR2
```

下一个示例是一个查询，它显示了一个重载存储过程参数的差异。我本可以只基于 `USER_ARGUMENTS` 视图来编写它，但我想展示一个连接两个视图的例子。

```sql
select ua.package_name, up.procedure_name, ua.argument_name, up.overload, position, IN_OUT
  from user_procedures up
  join user_arguments ua
  on (ua.object_name = up.procedure_name
  and ua.subprogram_id = up.subprogram_id)
  where up.overload is not null
  order by overload, position;
```

```
PACKAGE_NAME     PROCEDURE_NAME          ARGUMENT_NAME       OVERLOAD   POSITION IN_OUT
---------------- ----------------------- ------------------- ---------- --------
DEPARTMENTS_API  DEPARTMENT_MANAGER_ID                       1          0        OUT
DEPARTMENTS_API  DEPARTMENT_MANAGER_ID   P_DEPARTMENT_NAME   1          1        IN
DEPARTMENTS_API  DEPARTMENT_MANAGER_ID                       2          0        OUT
DEPARTMENTS_API  DEPARTMENT_MANAGER_ID   P_DEPARTMENT_ID     2          1        IN
```

注意 `POSITION` 和 `IN_OUT` 列。存储过程 `DEPARTMENT_MANAGER_ID` 实际上是一个函数。`IN_OUT` 列是“OUT”，位置是 0。对于函数，位置 0 是返回值（函数必须始终有返回值）。通过查看位置，你可以推断出一个程序单元是函数。函数返回值也没有参数名。如果你在 `USER_ARGUMENTS` 表中有一条没有名称的记录，那就是 `RETURN`。如果一个程序单元没有参数，它在 `USER_ARGUMENTS` 视图中将没有记录。

以下查询显示存储过程 `SECURE_DML` 存在于 `USER_PROCEDURES` 视图中：

```sql
SELECT count(*) FROM user_procedures
WHERE object_name = 'SECURE_DML';
```

```
COUNT(*)
----------------------
1
```

然而，下一个查询显示，即使存储过程 `SECURE_DML` 作为一个过程存在，它也没有参数，因此在 `USER_ARGUMENTS` 视图中不会有任何记录：

```sql
SELECT count(*) FROM user_arguments
WHERE object_name = 'SECURE_DML';
```

```
COUNT(*)
----------------------
0
```

另一个值得检查的标准，特别是因为它很容易自动化，就是验证你的标准禁止使用的数据类型（`CHAR`、`LONG`、`VARCHAR` 等）可以被轻松查询和质疑。尽管 PL/SQL 中的 `LONG` 现在只是 `VARCHAR2` 的子类型（与表中的 `LONG` 相对），但我见过许多标准要求 PL/SQL 代码使用 `VARCHAR2(32767)` 代替。

你可以使用以下查询相当容易地验证参数的数据类型信息：

```sql
SELECT object_name, argument_name, data_type
  FROM user_arguments
  WHERE data_type IN ('CHAR', 'LONG', 'VARCHAR');
```

```
OBJECT_NAME                    ARGUMENT_NAME                  DATA_TYPE
------------------------------ ------------------------------ ------------------------------
BAD_DATA_TYPES_PRC             P_DATA_IN                      CHAR
BAD_DATA_TYPES_PRC             P_DATA_OUT                     LONG
```

不幸的是，你*只能*从此视图获取参数的此类信息。当你验证这类信息时，你通常还希望验证变量以及参数。关于 PL/Scope 的部分将向你展示如何执行此操作。



##### USER_TYPES

`USER_TYPES` 视图提供关于你所拥有的类型的信息。虽然你可能不认为类型是 PL/SQL 对象，但许多（尽管不是全部）类型都有类型主体。类型主体就像其他任何程序单元一样是 PL/SQL。纯粹从 PL/SQL 的角度来看，此视图中重要的列是方法数量 (`METHODS`)、`LOCAL_METHODS`，以及超类型列 `SUPERTYPE_NAME` 和 `OWNER`。在接下来的几段中，我将假定读者对 Oracle 的对象类型系统有所了解。

快速查询此视图可以确定该类型是否包含任何 PL/SQL。如果方法数量大于 0，则规格说明（spec）背后就会有代码（即，一个主体）。此外，如果超类型列不为空，这意味着这是一个子类型。它从其超类型继承了一个规格说明（以及可能继承了一个主体）。同样重要的是 `LOCAL_METHODS` 列；这是在子类型中定义的方法数量（如果不是子类型则为 0，或者如果没有覆盖或扩展任何内容，也可能为 0）。

以下查询针对本章示例代码中提供的三个对象类型运行。这三个类型包括：一个没有主体（没有 PL/SQL）的类型、一个集合类型和一个带有主体的类型。

```sql
SELECT type_name, typecode, attributes, methods, supertype_name, local_methods
FROM user_types;
```

```
TYPE_NAME          TYPECODE     ATTRIBUTES     METHODS    SUPERTYPE_NAME       LOCAL_METHODS
------------------ ------------ -------------- ---------- -------------------- -------------
CODE_INFO          OBJECT       7              0
CODE_INFOS         COLLECTION   0              0
DEMO_TYPE          OBJECT       1              3
```

从输出中你可以看到，`CODE_INFO` 有七个属性，没有代码（没有方法），不从超类型继承，这意味着没有本地方法。`CODE_INFOS` 是一个集合。注意，它自身没有属性。`DEMO_TYPE` 有一个属性和三个方法。后者有 PL/SQL 供你查看。

##### USER_TYPE_METHODS

在调试类型体中的 PL/SQL 代码时，有一些重要的信息你需要知道——不仅在于代码如何编写，还在于方法如何定义。虽然所有列都有其价值，但对于了解你的代码来说，重要的列包括 `METHOD_NO`、`METHOD_TYPE`、`PARAMETERS`、`RESULTS`、`OVERRIDING` 和 `INHERITED`。

当你将 `USER_TYPE_METHODS` 与 `USER_PROCEDURES` 连接时，需将 `USER_TYPE_METHODS` 中的类型名连接到 `USER_PROCEDURES` 中的 `OBJECT_NAME`，并将 `METHOD_NO` 连接到 `SUBPROGRAM_ID`。`SUBPROGRAM_ID` 也会流入 `USER_ARGUMENTS` 视图。

如果你有重载的方法，`OVERLOAD` 列会有一个递增的序列号。如果 `OVERLOAD` 为空，则该方法未被重载。在使用 Oracle 类型系统时，重载方法非常常见，尤其是在构造函数中。

以下查询显示 `DEMO_TYPE` 类型提供了三个方法。成员过程 `OVERLOAD_PROC` 是重载的，而成员函数 `FUNC` 不是。

```sql
SELECT utm.type_name, utm.method_type, utm.method_name, up.overload,
       utm.parameters, utm.results
  FROM user_type_methods utm
  JOIN user_procedures up
   ON (utm.type_name = up.object_name
       AND
       utm.method_no = up.subprogram_id)
  WHERE utm.type_name = 'DEMO_TYPE';
```

```
TYPE_NAME      METHOD_TYPE METHOD_NAME    OVERLOAD   PARAMETERS    RESULTS
-------------- ----------- -------------- ---------- ------------- ----------------------
DEMO_TYPE      PUBLIC      FUNC                      2             1
DEMO_TYPE      PUBLIC      OVERLOAD_PROC  1          2             0
DEMO_TYPE      PUBLIC      OVERLOAD_PROC  2          2             0
```

`METHOD_TYPE` 告诉你方法的基本信息。它是 `PUBLIC` 方法（常规用法），还是 `ORDER` 或 `MAP` 类型（用于等价性比较）？

`PARAMETERS` 和 `RESULTS` 提供了方法的输入和输出。这两列实际上分别提供了输入和输出值的数量。在接下来的示例中，我将展示如何使用此视图以及你对 `PARAMETERS` 的了解，通过 `USER_ARGUMENTS` 来查找参数。你可能想知道为什么所有方法都说有两个参数，而它们定义时只有一个参数。类型方法总是包含 `SELF` 参数，即使它没有包含在规格说明中，或者在方法体中被引用。它总是被传入的。

`INHERITED` 和 `OVERRIDING` 分别告诉你该方法是从超类型继承的，还是覆盖了超类型中的方法。在调试和分析时，知道代码是否来自超类型非常重要。确保你正在查看正确的代码是至关重要的。

`USER_TYPE_METHODS` 不仅可以与 `USER_PROCEDURES` 连接，还可以与 `USER_ARGUMENTS` 连接，以获取关于你的对象类型的更多详细信息。以下查询显示了连接后的几行结果。这是一个用于验证命名规范和/或捕捉无效数据类型使用情况的绝佳查询。

```sql
  SELECT utm.type_name, utm.method_name, ua.argument_name,
       position, sequence, data_type
  FROM user_type_methods utm
  JOIN user_procedures up
   ON (utm.type_name = up.object_name
       AND
       utm.method_name = up.procedure_name)
  JOIN user_arguments ua
  ON (up.procedure_name = ua.object_name
      AND
      up.object_name = ua.package_name)
 WHERE utm.type_name = 'DEMO_TYPE'
   AND rownum < 5;  -- usign rownum just to keep the example short
```

```
TYPE_NAME   METHOD_NAME      ARGUMENT_NAME  POSITION   SEQUENCE  DATA_TYPE
----------- ---------------- -------------- ---------- --------- ------------
DEMO_TYPE   FUNC                          0          1         NUMBER
DEMO_TYPE   FUNC              SELF         1          2         OBJECT
DEMO_TYPE   FUNC              P_VALUE      2          3         VARCHAR2
DEMO_TYPE   OVERLOAD_PROC     SELF         1          1         OBJECT
```

##### USER_TRIGGERS

`USER_TRIGGERS` 包含有关在你的数据库中定义的触发器的信息。虽然它是查找代码的有用位置，并且确实包含有关触发器及其定义方式的有用信息，但它不包含太多关于代码本身的信息。请将此视图与遍历 `USER_SOURCE` 结合使用，以便逐行快速查看代码。



