# 变更影响分析

变更影响分析允许开发人员或分析师评估代码中计划变更对其他代码区域的影响。根据我的经验，影响分析非常罕见。通常的做法似乎是进行变更，然后事后才发现影响。这在时间、金钱和声誉方面可能代价高昂。提前了解要好得多，这样可以分析和规划影响，而不是在将代码移动到测试（或更糟的生产）环境后才遇到问题。

当完成影响分析时，您或进行分析的人员应该对需要更改的对象有相当深入的了解。您不仅会知道哪些对象需要更改，还可以为每个要更改的对象创建测试用例，并为所有已更改的对象规划和估算回归测试。通过提前知道要测试哪些对象，您还能安排时间进行适当的性能测试，以确保变更不会对应用程序性能产生负面影响。

影响分析的问题在于，确实没有一种适用于所有情况的分析方法。在数据库应用程序中，可能发生多种类型的变更：数据变更（数据类型或数量的更改，例如添加新来源）、模式变更（数据库结构的更改）和代码变更。PL/SQL 书籍无法为您提供太多处理数据变更的方法；这非常特定于应用程序。对于模式变更，`USER_DEPENDENCIES` 视图仍然是查看模式中哪些对象引用其他对象的最佳选择。本节的其余部分将讨论代码变更（这可能与数据或模式变更相关，或是其结果）。

在我工作过的大多数组织中，进行影响分析的方式要么是假设你对应用程序及其所有使用情况拥有全知知识，要么是更改一个对象并查看它在下游对象中使哪些对象无效。这两种方法都很糟糕。首先，如果你已经在更改对象，那么你已经过了估算阶段，而这正是你真正想要开始这个过程的时候。其次，无法确切指示是什么导致了失效。这是一种高度繁琐且容易出错的方法。

使用数据字典中的可用数据来运行报告并获取与变更相关的精确数据列表，这不是更好的方法吗？然后，人类可以利用这种（大部分，希望是）自动化的数据收集。

开始进行影响变更分析的方法是将需求分解为组成部分。每个部分包含一个变更，该变更（通常）对应用程序有一些影响。下一步是确定所有将受到影响的单元。

这种分解可以在对象级别（例如包级别）进行。然而，在那个高度上，你会遇到如前面在 `USER_SOURCE` 和 `USER_DEPENDENCIES` 部分中描述的所有问题。理想情况下，你可以在不依赖代码逐行扫描的情况下，对源代码有一个非常细粒度的视图。`PL/Scope` 可以提供程序单元被其他程序单元调用的非常细粒度的视图。请注意，`PL/Scope` 对外部代码或从视图调用的代码没有帮助。对于视图支持，你会使用依赖视图；对于外部代码和 Java，你会使用数据库外部的工具。

我没有在本书中包含它，但我还经常提取相关信息并将其存储在我自己的表中。我会将父/子记录折叠为单条记录（例如，折叠行以组合 `package.function` 调用）。这样，我有一个多遍系统，允许使用更易于维护的查询来处理更大范围的程序。在这里解释我使用的完整系统超出了范围，但当你开始编写自己的查询时，请记住，你可以编写查询的一部分，保存这些结果，然后针对保存的数据编写查询。

本章提供的演示代码中有一个名为 `PLSCOPE_SUPPORT_PKG` 的包。这个包包含一些用于处理 `PL/Scope` 各个方面的有用代码。我将从过程 `PRINT_IMPACTS` 开始，它是一个重载过程。我首先使用的版本接受程序单元所有者（`P_OWNER`）、名称（`P_OBJECT`）和子单元名称（`P_UNIT`）作为参数。

让我们看一个例子。假设你计划更改 `HR.JOBS_API.VALIDATE_JOB` 过程，使其接受一个额外的参数或更改返回数据类型。你会想知道该函数的所有调用位置。对支持包的示例调用如下所示：

```sql
SQL> exec plscope_support_pkg.print_impacts('HR', 'JOBS_API', 'VALIDATE_JOB');
```

```
Impact to PACKAGE BODY HR.EMPLOYEE_API.CREATE_EMPLOYEE at line 40 and column 12
Impact to PROCEDURE HR.TEST_CALLS_PRC at line 8 and column 8
Impact to PROCEDURE HR.TEST_CALLS_PRC at line 4 and column 6
Impact to FUNCTION HR.TEST_CALLS_FNC at line 5 and column 10

PL/SQL procedure successfully completed.
```

从这个视图中，你可以看到三个对象受到影响，其中一个对象（`HR.TEST_CALLS_PRC`）被影响了两次。在这个小的应用程序中，查询 `USER_SOURCE` 或使用 `grep` 搜索代码仓库并目测结果可能同样容易。然而，在非常大的系统中，这种简洁性使分析变得容易得多。

如果你想在输出中看到源文本，你可以向 `PRINT_IMPACTS` 调用传递一个可选的 `BOOLEAN` 参数，如下所示：

```sql
SQL> set serveroutput on format wrapped
SQL> exec plscope_support_pkg.print_impacts('HR', 'JOBS_API', 'VALIDATE_JOB', TRUE);
```

```
Impact to PACKAGE BODY HR.EMPLOYEE_API.CREATE_EMPLOYEE at line 40 and column 12
    TEXT:
              IF NOT jobs_api.validate_job(p_job_t
          tle, p_department_name)

Impact to PROCEDURE HR.TEST_CALLS_PRC at line 8 and column 8
    TEXT:
              IF JOBS_API.VALIDATE_JOB('a', 'b')

Impact to PROCEDURE HR.TEST_CALLS_PRC at line 4 and column 6
    TEXT:
            IF JOBS_API.VALIDATE_JOB('a', 'b')

Impact to FUNCTION HR.TEST_CALLS_FNC at line 5 and column 10
    TEXT:
            RETURN JOBS_API.VALIDATE_JOB('a', 'b')

PL/SQL procedure successfully completed.
```

这段代码只是一个演示，但它展示了如何使用 `PL/Scope` 遍历标识符的层次结构，以找到关于你的代码的许多宝贵信息。这些影响分析过程的核心是来自 `PLSCOPE_SUPPORT_PKG.GET_IMPACT_INFO` 的这个查询。（注意，我简化了过程和主要查询，以便它们能更好地作为示例。一个可以处理嵌套块的示例会稍微复杂一些，对于本节的目的来说是不必要的。只需知道此代码是一个演示和起点，而不是一个完整的影响分析程序。）以下代码来自 `PLSCOPE_SUPPORT_PKG.GET_IMPACT_INFO`，是起始于第 54 行的 `FOR LOOP` 查询：


