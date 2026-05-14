# 第一部分

## 基于存储过程的数据库框架

## 1. 存储过程基础

IT 组织持续面临的最常见问题之一，是找到开发和部署流程到生产环境所需的工作量与生产控制操作员的效率和有效性之间的适当平衡。开发和部署的工作量，与生产控制的有效性，似乎是截然对立的。让开发和部署流程变得更容易，通常意味着生产控制需要更多的工作和人工干预。真正需要考虑的问题是，在哪里“感受痛苦”更好？将工作量推向开发方可能会降低吞吐量，但能最大限度地减少生产流程层面出现问题的风险。本章以及接下来的几章将重点讨论使用存储过程执行的流程。在本章中，你将获得关于存储过程的基本介绍，这是后续章节的基础。你将看到如何创建子存储过程，并且我们会为你提供一个模板，你可以在自己的工作中用它来创建类似的过程。

## 框架的必要性

当仅处于开发模式时，构建从头到尾完成所有任务的大型、单体式过程是极其容易（且令人印象深刻）的。它们就是众所周知的“黑盒”，输入一些东西，经过一系列复杂变换，然后产生期望的结果。这种方式在没有问题或只是业务需求变更需要进行修改时，效果很好。当一个过程做了这么多事情时，修改完成后测试它需要付出什么代价？即使变更只影响该过程的 10%，也必须测试整个过程。实现这一点需要什么？如果需要进行多个复杂的修改呢？对于多个开发人员协同处理变更并协调他们的工作，难度会变得有多大？

现在，设想将那个单体拆分为多个过程，每个过程执行一个工作单元。当需要变更并进行测试时，可以将工作量隔离为仅执行该工作单元所必需的部分，验证也集中在过程的结果上。而且，由于现在是多个过程，多个开发人员通常可以同时进行修改，他们的工作可以互不干扰。随着时间的推移，这种方法可以证明更具成本效益和效率。

这就是框架的用武之地，它帮助组织和管理流程，为开发提供最大的灵活性，并能将维护工作量降到最低（而维护工作有时直到成为明显问题时才会被考虑）。框架为组装和执行流程提供了一致的方法。它还提倡以小型工作单元编写代码，这些单元有可能被混合、匹配和重用。它增加了开发和部署流程的复杂性，但可以减少生产调度的工作量。框架还可以为管理流程的执行提供更大的灵活性。

## 框架演示

为了开始分析框架概念，我们需要一个流程。我们接下来的示例展示了一个为针对示例模式运行每日流程而构建的框架。该流程的细节对于示例来说并不重要。只需考虑任何生产系统都可能有需要每天完成的任务，而接下来就是一个让这些日常流程得以实现的框架。

此外，示例中还包含一个每月流程。正如系统可能需要每天完成某些任务一样，通常也需要每月完成一次某些事情。在设计这样的系统时，必须考虑每日流程和每月流程在它们的调度时间重合时——在我们的示例中是每月的 1 日——的执行顺序。

为了本书的目的，我们开发了一个简单的过程（注意：所有描述的代码可在 [entdna.​com](http://www.entdna.com) 下载。你也可以从本书在 [Apress.​com](http://apress.com) 上的目录页面找到代码链接）。下载示例代码使你能够在自己的机器上跟随即将进行的示例操作。



### 一个示例模式

清单 1-1 展示了创建名为 `FWDemo` 的模式所需的代码，该模式将包含演示所需的所有内容。此外，还有创建名为 `FWDemo.ProcessLog` 表的代码。虽然在所有过程中都包含向此表写入的模式确实增加了一些复杂性和开销，但它提供的监控和故障排查能力足以弥补前期的努力。

```sql
print 'FWDemo Schema'
If Not Exists(Select name
From sys.schemas
Where name="FWDemo")
begin
print ' - Creating FWDemo schema'
declare @sql varchar(255) = 'Create Schema FWDemo'
exec(@sql)
print ' - FWDemo schema created'
end
Else
print ' - FWDemo schema already exists.'
print ''
GO
IF  EXISTS (SELECT * FROM sys.objects
WHERE object_id = OBJECT_ID(N'FWDemo.ProcessLog')
AND type in (N'U'))
DROP TABLE FWDemo.ProcessLog
GO
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [FWDemo].ProcessLog NOT NULL,
[ProcessLogMessage] nvarchar     NOT NULL,
[CreateDate]        [smalldatetime]     NOT NULL
)
GO
SET ANSI_PADDING OFF
GO
清单 1-1
模式和日志表创建
```

你是否有可以用于学习目的的 SQL Server 实例？以管理员身份（例如 `sa` 用户）连接到该实例。然后，在 SQL Server Management Studio (SSMS) 中，打开“新建查询”窗口，复制清单 1-1 中的代码，并执行它以创建本章及后续章节中使用的示例模式。

### 日常流程

清单 1-2 展示了将构成我们演示的“日常流程”的两个存储过程的创建代码。我们在示例中提供两个过程是因为这很常见，而且有两个过程可以让我们展示如何使后续过程的执行依赖于前面过程的成功——因为需要执行一系列过程并在出错时停止或采取其他措施，正是我们大多数人面临的现实场景。

这些过程（以及我们将要使用的所有其他过程）都可以被编译和执行以供你自行测试。你会注意到每个过程中都有一些被注释掉的代码（以 `--` 开头的行），这些代码可以被调用（移除 `--`，然后重新编译）以创建错误条件。这种创建错误条件的能力允许测试成功和未成功的完成情况，随着我们在后续章节中进行演示迭代，这将变得更加重要。

为了本次练习，我们将为日常流程声明一条业务规则：`FWDemo.DailyProcess1` 必须成功完成，然后才能执行 `FWDemo.DailyProcess2`。接着，`FWDemo.DailyProcess2` 必须成功完成，日常流程才能被视为成功执行。

```sql
If Exists(Select s.name + '.' + p.name
From sys.procedures p
Join sys.schemas s
On s.schema_id = p.schema_id
Where s.name = 'FWDemo'
And p.name = 'DailyProcess1')
begin
print ' - Dropping FWDemo.DailyProcess1 stored procedure'
Drop Procedure FWDemo.DailyProcess1
print ' - FWDemo.DailyProcess1 stored procedure dropped'
end
GO
CREATE PROCEDURE FWDemo.DailyProcess1
AS

--
-- Purpose: This procedure is part of the Stored Procedure Framework Demo.
--
-- NOTE: An Error situation can be created for testing/demo purposes by
--    un-commenting the Error code in the body of the procedure.  To return
--    to a procedure with a successful execution, re-comment the code or       --    recompile the original.
--

SET NOCOUNT ON
/*********************************************/
/*  Log the START of the procedure to the process log  */
/*********************************************/
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
Values ('Procedure FWDemo.DailyProcess1 - STARTING',
GETDATE()
)
DECLARE @RetStat int
SET @RetStat = 0
/******************************************/
/*  Force an ERROR CONDITION for this procedure  */
/******************************************/
--INSERT INTO FWDemo.ProcessLog (
--    ProcessLogMessage,
--    CreateDate
--)
--VALUES ('Procedure FWDemo.DailyProcess1 - Problem Encountered',
--    GETDATE()
--)
--SET @RetStat = 1
/****************************************************/
/*  Log the COMPLETION of the procedure to the process log     */
/****************************************************/
IF @RetStat = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.DailyProcess1 - COMPLETED',
GETDATE()
)
END
ELSE
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.DailyProcess1 - ERROR',
GETDATE()
)
END
RETURN @RetStat
GO
If Exists(Select s.name + '.' + p.name
From sys.procedures p
Join sys.schemas s
On s.schema_id = p.schema_id
Where s.name = 'FWDemo'
And p.name = 'DailyProcess2')
begin
print ' - Dropping FWDemo.DailyProcess2 stored procedure'
Drop Procedure FWDemo.DailyProcess2
print ' - FWDemo.DailyProcess2 stored procedure dropped'
end
GO
CREATE PROCEDURE FWDemo.DailyProcess2
AS

--
-- Purpose: This procedure is part of the Stored Procedure Framework Demo.
--
-- NOTE: An Error situation can be created for testing/demo purposes by
--    un-commenting the Error code in the body of the procedure.  To return
--    to a procedure with a successful execution, re-comment the code or
--    recompile the original.
--

SET NOCOUNT ON
/*********************************************/
/*  Log the START of the procedure to the process log  */
/*********************************************/
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
Values ('Procedure FWDemo.DailyProcess2 - STARTING',
GETDATE()
)
DECLARE @RetStat int
SET @RetStat = 0
/******************************************/
/*  Force an ERROR CONDITION for this procedure  */
/******************************************/
--INSERT INTO FWDemo.ProcessLog (
--    ProcessLogMessage,
--    CreateDate
--)
--VALUES ('Procedure FWDemo.DailyProcess2 - Problem Encountered',
--    GETDATE()
--)
--SET @RetStat = 1
/****************************************************/
/*  Log the COMPLETION of the procedure to the process log     */
/****************************************************/
IF @RetStat = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.DailyProcess2 - COMPLETED',
GETDATE()
)
END
ELSE
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.DailyProcess2 - ERROR',
GETDATE()
)
END
RETURN @RetStat
GO
清单 1-2
日常流程存储过程
```

在 SSMS 的“新建查询”窗口中，连接到 `FWDemo` 模式并执行清单 1-2 中的代码。该代码创建了两个共同构成日常流程的存储过程。有了这些过程后，你就可以将注意力转向下一个问题，即安排这些过程以实际每天运行。




### 执行日常流程

既然环境已搭建完毕且流程也已创建，现在让我们转向执行环节。运维人员需要设置或安排这些流程的运行，并监控是否有错误状况产生，或者如果使用的调度工具支持此功能，则需设置好执行的先后顺序规则。从基本意义上讲，我们现在已拥有一个可供投产的**日常流程**。清单 1-3 展示了可用于执行**日常流程**的语句，以及一条可运行的 `SELECT` 语句，用于查看写入到 `FWDemo.ProcessLog` 的输出。你会注意到我们按降序对输出进行了排序。这将把最新的消息显示在顶部，避免了需要向下滚动才能找到当前执行消息的麻烦，一旦日志内容变得非常多时，这会使查看变得容易得多。

```
EXECUTE FWDemo.DailyProcess1
EXECUTE FWDemo.DailyProcess2
SELECT ProcessLogID
,ProcessLogMessage
,CreateDate
FROM FWDemo.ProcessLog
ORDER BY ProcessLogID desc
清单 1-3
日常流程执行语句与流程日志 SELECT 语句
```

### 纳入月度流程

现在是时候为我们的生产流程再添加一个层级了。在清单 1-4 中，有用于创建另外两个存储过程的代码，它们将组成一个**月度流程**。这些过程的操作方式与我们的日常流程过程相同，并且它们也关联着一些业务规则。首先，**月度流程**将在每月的第一天运行。其次，它将在**日常流程**成功执行后运行。第三，`FWDemo.MonthlyProcess1` 必须成功完成，然后 `FWDemo.MonthlyProcess2` 才能被执行。

```
If Exists(Select s.name + '.' + p.name
From sys.procedures p
Join sys.schemas s
On s.schema_id = p.schema_id
Where s.name = 'FWDemo'
And p.name = 'MonthlyProcess1')
begin
print ' - Dropping FWDemo.MonthlyProcess1 stored procedure'
Drop Procedure FWDemo.MonthlyProcess1
print ' - FWDemo.MonthlyProcess1 stored procedure dropped'
end
GO
CREATE PROCEDURE FWDemo.MonthlyProcess1
AS

--
-- Purpose: This procedure is part of the Stored Procedure Framework Demo.
--
-- NOTE: An Error situation can be created for testing/demo purposes by
--    un-commenting the Error code in the body of the procedure.  To return
--    to a procedure with a successful execution, re-comment the code or
--    recompile the original.
--

SET NOCOUNT ON
/*********************************************/
/*  Log the START of the procedure to the process log  */
/******************************** ************/
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
Values ('Procedure FWDemo.MonthlyProcess1 - STARTING',
GETDATE()
)
DECLARE @RetStat int
SET @RetStat = 0
/******************************************/
/*  Force an ERROR CONDITION for this procedure  */
/******************************************/
--INSERT INTO FWDemo.ProcessLog (
--    ProcessLogMessage,
--    CreateDate
--)
--VALUES ('Procedure FWDemo.MonthlyProcess1 - Problem Encountered',
--    GETDATE()
--)
--SET @RetStat = 1
/****************************************************/
/*  Log the COMPLETION of the procedure to the process log     */
/****************************************************/
IF @RetStat = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.MonthlyProcess1 - COMPLETED',
GETDATE()
)
END
ELSE
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.MonthlyProcess1 - ERROR',
GETDATE()
)
END
RETURN @RetStat
GO
If Exists(Select s.name + '.' + p.name
From sys.procedures p
Join sys.schemas s
On s.schema_id = p.schema_id
Where s.name = 'FWDemo'
And p.name = 'MonthlyProcess2')
begin
print ' - Dropping FWDemo.MonthlyProcess2 stored procedure'
Drop Procedure FWDemo.MonthlyProcess2
print ' - FWDemo.MonthlyProcess2 stored procedure dropped'
end
GO
CREATE PROCEDURE FWDemo.MonthlyProcess2
AS

--
-- Purpose: This procedure is part of the Stored Procedure Framework Demo.
--
-- NOTE: An Error situation can be created for testing/demo purposes by
--    un-commenting the Error code in the body of the procedure.  To return
--    to a procedure with a successful execution, re-comment the code or
--    recompile the original.
--

SET NOCOUNT ON
/*********************************************/
/*  Log the START of the procedure to the process log  */
/********************************************/
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
Values ('Procedure FWDemo.MonthlyProcess2 - STARTING',
GETDATE()
)
DECLARE @RetStat int
SET @RetStat = 0
/******************************************/
/*  Force an ERROR CONDITION for this procedure  */
/******************************************/
--INSERT INTO FWDemo.ProcessLog (
--    ProcessLogMessage,
--    CreateDate
--)
--VALUES ('Procedure FWDemo.MonthlyProcess2 - Problem Encountered',
--    GETDATE()
--)
--SET @RetStat = 1
/****************************************************/
/*  Log the COMPLETION of the procedure to the process log     */
/****************************************************/
IF @RetStat = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.MonthlyProcess2 - COMPLETED',
GETDATE()
)
END
ELSE
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.MonthlyProcess2 - ERROR',
GETDATE()
)
END
RETURN @RetStat
GO
清单 1-4
月度流程存储过程
```

运维人员需要运行或调度的执行语句如清单 1-5 所示。使用前面介绍的 `SELECT` 语句来监控流程执行的进度。

```
EXECUTE FWDemo.MonthlyProcess1
EXECUTE FWDemo.MonthlyProcess2
清单 1-5
月度流程执行语句
```

从运维角度来看，我们现在引入了更多的复杂性。流程的设置/调度现在增加了额外的优先级层，还有一个时间因素。调度和监控变得更加关键，并且为了确保一切顺利运行，需要关注的地方也更多了。

### 总结

至此，我们已构建了一个包含两个日常流程和两个月度流程的系统。系统中存在依赖关系，要求所有先前流程成功执行后，才能启动下一个流程的执行。这些依赖关系必须在常规生产中的流程调度期间，以及在故障排查和问题修复过程中加以应用。所有这些“活动部件”不仅增加了复杂性，还因为需要外部关注而给系统带来了更多的薄弱点。

## 2. 使用存储过程实现自动化

既然我们已经了解到为了执行第 1 章中定义的流程，日复一日需要协调的所有部分，现在让我们引入**控制器**的概念。控制器是我们演示框架概念中最基础的元素。接下来的示例将展示控制器如何使你的工作更轻松，同时减少运维人员需要处理的开销。



## 一个日常处理控制器

就我们的目的而言，控制器是一个存储过程，它本身执行一个流程的所有元素，并能处理该流程内的业务规则。`清单 2-1` 展示了创建名为 `FWDemo.DailyProcessController` 的存储过程的代码。此存储过程执行我们的两个 `日常处理` 存储过程。它记录自身的进度，并检查每个存储过程的返回状态，以确保每个过程都成功完成。

```
If Exists(Select s.name + '.' + p.name
From sys.procedures p
Join sys.schemas s
On s.schema_id = p.schema_id
Where s.name = 'FWDemo'
And p.name = 'DailyProcessController')
begin
print ' - 正在删除 FWDemo.DailyProcessController 存储过程'
Drop Procedure FWDemo.DailyProcessController
print ' - FWDemo.DailyProcessController 存储过程已删除'
end
GO
CREATE PROCEDURE FWDemo.DailyProcessController
AS

--
-- 目的：此过程是存储过程框架演示的一部分。
--          它是日常处理存储过程的控制器。
--

SET NOCOUNT ON
/*********************************************/
/*  将过程的 开始 记录到处理日志  */
/*********************************************/
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
Values ('过程 FWDemo.DailyProcessController - 开始',
GETDATE()
)
DECLARE @RetStat int
SET @RetStat = 0
/*****************************************/
/*  执行 DailyProcess1 存储过程  */
/*****************************************/
EXEC @RetStat = FWDemo.DailyProcess1
IF @RetStat  0
GOTO EndController
/*****************************************/
/*  执行 DailyProcess2 存储过程  */
/*****************************************/
EXEC @RetStat = FWDemo.DailyProcess2
IF @RetStat  0
GOTO EndController
/****************************************************/
/*  将过程的 完成 状态记录到处理日志      */
/****************************************************/
EndController:
IF @RetStat = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('过程 FWDemo.DailyProcessController - 已完成',
GETDATE()
)
END
ELSE
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('过程 FWDemo.DailyProcessController - 错误',
GETDATE()
)
END
RETURN @RetStat
GO
清单 2-1
日常处理控制器存储过程
```

借助 `清单 2-1` 中的控制器，运维人员的执行工作变得简单多了。无需再同时调度多个存储过程并处理调度中的优先规则，只需设置好控制器的执行，现在监控成功完成情况也更容易。`清单 2-2` 展示了用于执行 `日常处理` 控制器的代码。一如既往，`FWDemo.ProcessLog` 显示了执行的进度。

```
EXECUTE FWDemo.DailyProcessController
清单 2-2
日常处理控制器执行语句
```

使用控制器模式的另一个优点是，当需要向流程中添加新的存储过程和/或新的业务规则时。如果没有控制器，新的存储过程需要编译，而运维人员负责在流程中的正确位置执行它，并在设置过程中考虑新的业务规则要求。这是在开发/测试结束后需要发生的许多步骤，因此是出错的机会。有了控制器，开发和测试将不仅包括新的存储过程，还包括对控制器存储过程的必要修改，以执行新的存储过程。然后，业务规则成为开发/测试工作的一部分，部署到生产环境就变得像编译新的存储过程和新版本的控制器存储过程一样简单。由于控制器执行已经调度好，运维人员无需进行其他更改。我们在部署到生产环境时消除了许多手动干预，从而减少了出错的机会。

## 月度处理控制器

接下来，我们将为我们的 `月度处理` 添加一个控制器。`清单 2-3` 包含名为 `FWDemo.MonthlyProcessController` 的存储过程的代码。此存储过程执行两个月度处理存储过程，并在 `FWDemo.ProgressLog` 中记录进度的同时处理流程内的优先规则。运维人员只需执行此控制器即可启动流程。

```
If Exists(Select s.name + '.' + p.name
From sys.procedures p
Join sys.schemas s
On s.schema_id = p.schema_id
Where s.name = 'FWDemo'
And p.name = 'MonthlyProcessController')
begin
print ' - 正在删除 FWDemo.MonthlyProcessController 存储过程'
Drop Procedure FWDemo.MonthlyProcessController
print ' - FWDemo.MonthlyProcessController 存储过程已删除'
end
GO
CREATE PROCEDURE FWDemo.MonthlyProcessController
AS

--
-- 目的：此过程是存储过程框架演示的一部分。
--          它是月度处理存储过程的控制器。
--

SET NOCOUNT ON
/*********************************************/
/*  将过程的 开始 记录到处理日志      */
/*********************************************/
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
Values ('过程 FWDemo.MonthlyProcessController - 开始',
GETDATE()
)
DECLARE @RetStat int
SET @RetStat = 0
/********************************************/
/*  执行 MonthlyProcess1 存储过程      */
/********************************************/
EXEC @RetStat = FWDemo.MonthlyProcess1
IF @RetStat  0
GOTO EndController
/********************************************/
/*  执行 MonthlyProcess2 存储过程  */
/********************************************/
EXEC @RetStat = FWDemo.MonthlyProcess2
IF @RetStat  0
GOTO EndController
/****************************************************/
/*  将过程的 完成 状态记录到处理日志    */
/****************************************************/
EndController:
IF @RetStat = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('过程 FWDemo.MonthlyProcessController - 已完成',
GETDATE()
)
END
ELSE
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('过程 FWDemo.MonthlyProcessController - 错误',
GETDATE()
)
END
RETURN @RetStat
GO
清单 2-3
月度处理控制器存储过程
```

`清单 2-4` 包含用于执行新的 `月度处理控制器` 的语句。

```
EXECUTE FWDemo.MonthlyProcessController
清单 2-4
月度处理控制器执行语句
```

然而，这里没有涉及的是控制器之间的业务优先规则，即 1) `月度处理` 在每月第一天执行，以及 2) 在 `日常处理` 成功完成后执行。但是，随着控制器模式的引入，运维人员需要处理的这些细节已经变得简单得多。

### 总结

控制器融合了由业务规则定义的依赖关系，并协调流程的执行。这样做有两个主要优点。首先，它使调度变得容易得多，因为依赖关系是内置的。其次，在排查流程问题时，它便于随时访问和参考流程流。

## 3. 存储过程协调器

基于第 2 章中的控制器模式，框架的下一个逻辑进展是协调器的概念。它基本上是控制器的控制器，可以解决控制器之间的优先规则问题。

### 协调器

控制器本身就是一种流程，与我们已构建的每日和每月流程一样。我们可能会发现不同控制器的处理之间存在依赖关系。这正是协调器的用武之地。协调器可以像控制器一样精确地处理这些依赖关系。代码清单 3-1 展示了创建名为 `FWDemo.ProcessOrchestrator` 的存储过程的代码。此存储过程执行 `每日控制器` 存储过程，并根据日期（如适用）执行 `每月控制器` 存储过程。

```sql
IF EXISTS (SELECT s.name + '.' + p.name
FROM sys.procedures p
JOIN sys.schemas s
ON s.schema_id = p.schema_id
WHERE s.name = 'FWDemo'
AND p.name = 'ProcessOrchestrator')
BEGIN
PRINT ' - 正在删除 FWDemo.ProcessOrchestrator 存储过程'
DROP PROCEDURE FWDemo.ProcessOrchestrator
PRINT ' - FWDemo.ProcessOrchestrator 存储过程已删除'
END
GO
CREATE PROCEDURE FWDemo.ProcessOrchestrator
@RunDate smalldatetime = null
AS

--
-- 目的：此过程是存储过程框架演示的一部分。                        --    它是执行控制器过程的协调器过程。
--

SET NOCOUNT ON
/*******************************************************/
/*  将过程开始信息记录到过程日志  */
/*******************************************************/
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('过程 FWDemo.ProcessOrchestrator - 开始',
GETDATE()
)
IF @RunDate is null
SET @RunDate = GETDATE()
DECLARE @RetStat int
SET @RetStat = 0
/******************************************/
/*  执行每日流程控制器  */
/******************************************/
EXEC @RetStat = FWDemo.DailyProcessController
IF @RetStat <> 0
GOTO EndOrchestrator
/**********************************************************************/
/*  如果 @RunDate 是某月的第一天，则执行每月流程控制器                    */      /*                                                                    */
/**********************************************************************/
IF DATEPART(DAY, @RunDate) = 1
BEGIN
EXEC @RetStat = FWDemo.MonthlyProcessController
IF @RetStat <> 0
GOTO EndOrchestrator
END
/************************************************************/
/*  将过程完成信息记录到过程日志  */
/************************************************************/
EndOrchestrator:
IF @RetStat = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('过程 FWDemo.ProcessOrchestrator - 已完成',
GETDATE()
)
END
ELSE
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('过程 FWDemo.ProcessOrchestrator - 错误',
GETDATE()
)
END
RETURN @RetStat
GO
代码清单 3-1
流程协调器存储过程
```

正如我们已经见过的各种存储过程一样，协调器通过记录到 `FWDemo.ProcessLog` 来显示执行进度。返回状态用于检查控制器是否成功完成。此外，还包括用于确定是否为每月第一天的代码，这是执行 `月度流程` 的优先规则之一。有一个名为 `@RunDate` 的输入参数，其默认值为 `null`。`null` 值导致存储过程使用当前日期进行处理。然而，为了便于试验演示系统，可以在执行时提供日期来设定 `@RunDate` 的值。代码清单 3-2 展示了协调器的执行语句，分别用于使用当前日期和覆盖日期的情况。从所有实际目的来看，运维人员需要做的全部事情就是设置/安排 `FWDemo.ProcessOrchestrator` 的执行，其他一切都由已构建的模式处理。

```sql
--如果运行日期是当前日期：
EXECUTE FWDemo.ProcessOrchestrator
--如果运行日期不是当前日期：
EXECUTE FWDemo.ProcessOrchestrator
@RunDate = '<CCYYMMDD>'
--注意：<CCYYMMDD> 的格式为 CCYYMMDD
代码清单 3-2
流程协调器执行语句
```

### 流程快速概览

至此，让我们回顾一下目前已构建的内容。有一个 `每日流程`，由两个存储过程组成；还有一个 `每月流程`，也由两个存储过程组成。每日流程控制器 `FWDemo.DailyProcessController` 执行每日流程，并确保存储过程 `FWDemo.DailyProcess1` 成功完成后再执行 `FWDemo.DailyProcess2` 存储过程。每月流程控制器 `FWDemo.MonthlyProcessController` 执行每月流程，并确保存储过程 `FWDemo.MonthlyProcess1` 成功完成后再执行 `FWDemo.MontlyProcess2` 存储过程。包裹这一切的是 `FWDemo.ProcessOrchestrator`，它执行每日流程控制器，并在成功完成后检查运行日期是否为某月的第一天，如果是，则执行每月流程控制器。单个执行语句驱动整个流程。

### 问题排查初探

到目前为止，我们的注意力一直集中在如何成功完成已构建的流程上。所以，是时候进行现实检验了。我们如何处理错误情况？如果序列中的某个流程出错并未成功完成，我们该怎么办？这种情况有时会伴随着深夜的电话，需要采取一些行动。于是便开始了深入调查和问题排查。查看 `FWDemo.ProcessLog` 是开始识别问题位置和原因以进行纠正的绝佳起点。一旦确定了解决方案并纠正了问题，我们如何完成序列中剩余的步骤？按照我们当前的模式，唯一的选择似乎是解构出错的流程并手动完成它。之后，我们在模式的最高层级找到一次执行，并从那里手动执行。

例如，生产流程的存储过程 `FWDemo.DailyProcess2` 出现问题。我们研究并纠正了问题。此时，需要手动执行 `FWDemo.DailyProcess2`。成功完成后，我们知道 `FWDemo.DailyProcessController` 中的所有内容都已完成。下一个最高层级是 `FWDemo.ProcessOrchestrator` 存储过程。在每日流程控制器执行后，协调器会检查 `@RunDate` 参数中的值是否等于某月的第一天。如果是这种情况，那么我们需要手动执行 `FWDemo.MonthlyProcessController`。如果不是某月的第一天，则无需其他操作。在我们的系统中，这相当简单。但实际上，解决方案可能包含更多变动部分和大量手动干预。流程越复杂，问题解决就越复杂。

### 总结

协调器不过是更高级别的控制器。它管理多个控制器的执行，同时整合控制器之间可能存在的任何依赖关系。它可以进一步简化调度，同时提供更完整的流程视图，有助于成功进行问题补救。

## 4. 基于存储过程的元数据驱动框架

为了增加灵活性和对框架内所有流程执行的控制，可以在环境中添加元数据。它可以标识存储过程、控制器和协调器，以及它们之间的关系。元数据将用作驱动流程执行的通信媒介。


### 构建元数据

#### 简介

如果我们打算创建元数据，那么就需要一个地方来存储它。为此，我们可以创建一个表或一组表。本章示例的元数据存储在一组用于控制执行的表中。第一个表是 `FWDemo.ProcessProcedure`，如代码清单 4-1 所示。

#### 创建元数据表

```sql
IF EXISTS (SELECT * FROM sys.objects
WHERE object_id = OBJECT_ID(N'FWDemo.ProcessProcedure')
AND type in (N'U'))
DROP TABLE FWDemo.ProcessProcedure
GO
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [FWDemo].ProcessProcedure NOT NULL,
[ProcessProcedureName]  nvarchar     NOT NULL
)
GO
SET ANSI_PADDING OFF
GO
```
代码清单 4-1
创建 ProcessProcedure 表

此表标识了框架中所有可用于执行的存储过程。这些存储过程构成了构建块，它们被收集在一起，形成我们所谓的框架应用程序。为了建立应用程序，我们使用 `FWDemo.Application` 表，如代码清单 4-2 所示。

```sql
IF EXISTS (SELECT * FROM sys.objects
WHERE object_id = OBJECT_ID(N'FWDemo.Application')
AND type in (N'U'))
DROP TABLE FWDemo.Application
GO
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [FWDemo].Application NOT NULL,
[ApplicationName] nvarchar     NOT NULL
)
GO
SET ANSI_PADDING OFF
GO
```
代码清单 4-2
创建 Application 表

现在需要确定哪些存储过程将包含在每个已定义的应用程序中。为此，我们使用名为 `FWDemo.ApplicationProcessProcedure` 的表，如代码清单 4-3 所示。该表包含定义进程执行的元数据。正如我们在第 1 章所讨论的，构建小的工作单元存储过程可以实现代码重用。使用元数据表可以将单个存储过程关联到多个应用程序。数据归档存储过程就是一个很好的例子，说明这在何处可能有所帮助。

```sql
IF EXISTS (SELECT * FROM sys.objects
WHERE object_id = OBJECT_ID(N'FWDemo.ApplicationProcessProcedure')
AND type in (N'U'))
DROP TABLE FWDemo.ApplicationProcessProcedure
GO
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [FWDemo].ApplicationProcessProcedure NOT NULL,
[ApplicationID]         [int]  NOT NULL,
[ProcessProcedureID]    [int]  NOT NULL,
[ExecutionOrder]        [int]  NOT NULL,
[Active]                [bit]  NOT NULL
)
GO
SET ANSI_PADDING OFF
GO
```
代码清单 4-3
创建 ApplicationProcessProcedure 表

有一个名为 `ExecutionOrder` 的列，它将决定其关联的存储过程将在进程序列中的哪个位置执行。此外，`Active` 列指示存储过程是否已启用执行。它用于禁用那些因某种原因不需在进程中执行的过程。如果您想临时跳过某个过程，这非常有用，本章稍后将进行讨论。

#### 加载元数据

元数据表创建完成后，让我们为进程加载数据。代码清单 4-4 显示了将所有存储过程加载到 `FWDemo.ProcessProcedure` 表中的代码。由于 `ProcessProcedureID` 是 `IDENTITY` 列，其值将自动生成，因此只插入了名称。

```sql
--Load First Daily Procedure
INSERT INTO FWDemo.ProcessProcedure (ProcessProcedureName)
VALUES ('FWDemo.DailyProcess1')
GO
--Load Second Daily Procedure
INSERT INTO FWDemo.ProcessProcedure (ProcessProcedureName)
VALUES ('FWDemo.DailyProcess2')
GO
--Load First Monthly Procedure
INSERT INTO FWDemo.ProcessProcedure (ProcessProcedureName)
VALUES ('FWDemo.MonthlyProcess1')
GO
--Load Second Monthly Procedure
INSERT INTO FWDemo.ProcessProcedure (ProcessProcedureName)
VALUES ('FWDemo.MonthlyProcess2')
GO
```
代码清单 4-4
向 ProcessProcedure 表加载数据

代码清单 4-5 包含将演示用的应用程序定义到 `FWDemo.Application` 表中的代码。对于我们的基本框架，我们定义了每日和每月进程。同样，`IDENTITY` 列 `ApplicationID` 的值将自动生成。

```sql
--Load Application for Daily Process
INSERT INTO FWDemo.Application (ApplicationName)
VALUES ('DailyProcessControllerMD')
GO
--Load Application for Monthly Process
INSERT INTO FWDemo.Application (ApplicationName)
VALUES ('MonthlyProcessControllerMD')
GO
```
代码清单 4-5
向 Application 表加载数据

#### 关联存储过程与应用程序

为了将所有部分组合在一起，代码清单 4-6 包含了将存储过程与其目标应用程序关联起来的代码。请注意执行顺序所使用的值。使用一个刻度范围内的值（本例中是十的间隔）可以在最初定义的执行顺序之间留出间隔。这样，如果以后添加一个需要在现有存储过程之间执行的新过程，就有空间添加一个值，而无需重新编号现有的执行顺序值。另外，请注意 `Active` 列使用了值 `"1"`，这会在应用程序内启用该过程的执行。值 `"0"` 将在应用程序的上下文中禁用该过程。

```sql
DECLARE @AppID    int,
@ProcID     int
/********************************/
/*  Daily Process Application   */
/********************************/
SET @AppID = (SELECT ApplicationID
FROM FWDemo.Application
WHERE ApplicationName = 'DailyProcessControllerMD')
SET @ProcID = (SELECT ProcessProcedureID
FROM FWDemo.ProcessProcedure
WHERE ProcessProcedureName = 'FWDemo.DailyProcess1')
INSERT INTO FWDemo.ApplicationProcessProcedure (
ApplicationID,
ProcessProcedureID,
ExecutionOrder,
Active
)
VALUES (@AppID,
@ProcID,
10,
1
)
SET @ProcID = (SELECT ProcessProcedureID
FROM FWDemo.ProcessProcedure
WHERE ProcessProcedureName = 'FWDemo.DailyProcess2')
INSERT INTO FWDemo.ApplicationProcessProcedure (
ApplicationID,
ProcessProcedureID,
ExecutionOrder,
Active
)
VALUES (@AppID,
@ProcID,
20,
1
)
/**********************************/
/*  Monthly Process Application   */
/**********************************/
SET @AppID = (SELECT ApplicationID
FROM FWDemo.Application
WHERE ApplicationName = 'MonthlyProcessControllerMD')
SET @ProcID = (SELECT ProcessProcedureID
FROM FWDemo.ProcessProcedure
WHERE ProcessProcedureName = 'FWDemo.MonthlyProcess1')
INSERT INTO FWDemo.ApplicationProcessProcedure (
ApplicationID,
ProcessProcedureID,
ExecutionOrder,
Active
)
VALUES (@AppID,
@ProcID,
10,
1
)
SET @ProcID = (SELECT ProcessProcedureID
FROM FWDemo.ProcessProcedure
WHERE ProcessProcedureName = 'FWDemo.MonthlyProcess2')
INSERT INTO FWDemo.ApplicationProcessProcedure (
ApplicationID,
ProcessProcedureID,
ExecutionOrder,
Active
)
VALUES (@AppID,
@ProcID,
20,
1
)
```
代码清单 4-6
为框架加载元数据


### 元数据就绪的控制器

现在，所有的元数据都已就绪。然而，为了能够利用这些元数据执行操作，需要新的控制器来使用它们。代码清单 4-7 展示了新的每日和每月控制器的代码。你会注意到，这些新过程的名称都带有“MD”后缀。

```sql
IF EXISTS (SELECT s.name + '.' + p.name
FROM sys.procedures p
JOIN sys.schemas s
ON s.schema_id = p.schema_id
WHERE s.name = 'FWDemo'
AND p.name = 'DailyProcessControllerMD')
BEGIN
PRINT ' - Dropping FWDemo.DailyProcessControllerMD stored procedure'
DROP PROCEDURE FWDemo.DailyProcessControllerMD
PRINT ' - FWDemo.DailyProcessControllerMD stored procedure dropped'
END
GO
CREATE PROCEDURE FWDemo.DailyProcessControllerMD
AS

--
-- Purpose: This procedure is part of the Stored Procedure Framework Demo.                    --    It is the Metadata Controller version for the Daily Process Stored       --    Procedures.
--

SET NOCOUNT ON
/*******************************************************/
/*  Log the START of the procedure to the process log  */
/*******************************************************/
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.DailyProcessControllerMD - STARTING',
GETDATE()
)
DECLARE @RetStat int
SET @RetStat = 0
/***************************************************************/
/*  Get and Execute the Active DailyProcess Stored Procedures  */
/***************************************************************/
DECLARE @DailyProcName  nvarchar(255)
,@AppID     int
,@AppPPID   int
DECLARE @DailyCursor as CURSOR
SET @DailyCursor = CURSOR FOR
SELECT pp.ProcessProcedureName, a.ApplicationID,
app.ApplicationProcessProcedureID
FROM FWDemo.Application a
JOIN FWDemo.ApplicationProcessProcedure app
ON a.ApplicationID = app.ApplicationID
JOIN FWDemo.ProcessProcedure pp
ON app.ProcessProcedureID = pp.ProcessProcedureID
WHERE a.ApplicationName = 'DailyProcessControllerMD'
AND app.Active = 1
ORDER BY app.ExecutionOrder
OPEN @DailyCursor
FETCH NEXT FROM @DailyCursor INTO @DailyProcName, @AppID, @AppPPID
WHILE @@FETCH_STATUS = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Executing ' + @DailyProcName + ' ApplicationID = ' +
convert(varchar(10), @AppID) +
' ApplicationProcessProcedureID = ' + convert(varchar(10),
@AppPPID),
GETDATE()
)
EXEC @RetStat = @DailyProcName
IF @RetStat  0
BEGIN
CLOSE @DailyCursor
DEALLOCATE @DailyCursor
GOTO EndController
END
FETCH NEXT FROM @DailyCursor INTO @DailyProcName, @AppID, @AppPPID
END
CLOSE @DailyCursor
DEALLOCATE @DailyCursor
/************************************************************/
/*  Log the COMPLETION of the procedure to the process log  */
/************************************************************/
EndController:
IF @RetStat = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.DailyProcessControllerMD - COMPLETED',
GETDATE()
)
END
ELSE
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.DailyProcessControllerMD - ERROR',
GETDATE()
)
END
RETURN @RetStat
GO
IF EXISTS (SELECT s.name + '.' + p.name
FROM sys.procedures p
JOIN sys.schemas s
ON s.schema_id = p.schema_id
WHERE s.name = 'FWDemo'
AND p.name = 'MonthlyProcessControllerMD')
BEGIN
PRINT ' - Dropping FWDemo.MonthlyProcessControllerMD stored procedure'
DROP Procedure FWDemo.MonthlyProcessControllerMD
PRINT ' - FWDemo.MonthlyProcessControllerMD stored procedure dropped'
END
GO
CREATE PROCEDURE FWDemo.MonthlyProcessControllerMD
AS

--
-- Purpose: This procedure is part of the Stored Procedure Framework Demo.
--    It is the Metadata Controller version for the Monthly Process Stored
--    Procedures.
--

SET NOCOUNT ON
/*******************************************************/
/*  Log the START of the procedure to the process log  */
/*******************************************************/
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.MonthlyProcessControllerMD - STARTING',
GETDATE()
)
DECLARE @RetStat int
SET @RetStat = 0
/*****************************************************************/
/*  Get and Execute the Active MonthlyProcess Stored Procedures  */
/*****************************************************************/
DECLARE @MonthlyProcName nvarchar(255)
,@AppID     int
,@AppPPID   int
DECLARE @MonthlyCursor as CURSOR
SET @MonthlyCursor = CURSOR FOR
SELECT pp.ProcessProcedureName, a.ApplicationID,
app.ApplicationProcessProcedureID
FROM FWDemo.Application a
JOIN FWDemo.ApplicationProcessProcedure app
ON a.ApplicationID = app.ApplicationID
JOIN FWDemo.ProcessProcedure pp
ON app.ProcessProcedureID = pp.ProcessProcedureID
WHERE a.ApplicationName = 'MonthlyProcessControllerMD'
AND app.Active = 1
ORDER BY app.ExecutionOrder
OPEN @MonthlyCursor
FETCH NEXT FROM @MonthlyCursor INTO @MonthlyProcName, @AppID, @AppPPID
WHILE @@FETCH_STATUS = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES (
'Executing ' + @DailyProcName + ' ApplicationID = ' +
convert(varchar(10), @AppID) +
' ApplicationProcessProcedureID = ' + convert(varchar(10),
@AppPPID),
GETDATE()
)
EXEC @RetStat = @MonthlyProcName
IF @RetStat  0
BEGIN
CLOSE @MonthlyCursor
DEALLOCATE @MonthlyCursor
GOTO EndController
END
FETCH NEXT FROM @MonthlyCursor INTO @MonthlyProcName, @AppID, @AppPPID
END
CLOSE @MonthlyCursor
DEALLOCATE @MonthlyCursor
/*************************************************************/
/*  Log the COMPLETION of the procedure to the process log   */
/*************************************************************/
EndController:
IF @RetStat = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.MonthlyProcessControllerMD - COMPLETED',
GETDATE()
)
END
ELSE
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.MonthlyProcessControllerMD - ERROR',
GETDATE()
)
END
RETURN @RetStat
GO
Listing 4-7
New metadata controllers
```

如你所见，这些新版本的控制器使用了一个游标来检索所有与各自应用程序相关的活动存储过程，并按执行顺序排序。过程名称以及所有关联键的值被加载到变量中。存储过程名称用于引导执行，而键值则用于写入 `FWDemo.ProcessLog` 的消息中，以跟踪执行情况并有助于监控或故障排除。该游标循环遍历每个存储过程，直到所有过程都执行完毕或发生错误。


### 支持元数据的协调器

随着控制器新版本的就绪，协调器也需要一个新版本（同样以“MD”为后缀）来执行它们。代码清单 4-8 展示了新协调器的代码。请注意，创建控制器和协调器的独立版本是为了保留原始版本并使其继续运行，以确保框架演示的所有方面保持完整和可用。

```sql
IF EXISTS (SELECT s.name + '.' + p.name
FROM sys.procedures p
JOIN sys.schemas s
ON s.schema_id = p.schema_id
WHERE s.name = 'FWDemo'
AND p.name = 'ProcessOrchestratorMD')
BEGIN
PRINT ' - 正在删除 FWDemo.ProcessOrchestratorMD 存储过程'
DROP PROCEDURE FWDemo.ProcessOrchestratorMD
PRINT ' - FWDemo.ProcessOrchestratorMD 存储过程已删除'
END
GO
CREATE PROCEDURE FWDemo.ProcessOrchestratorMD
@RunDate smalldatetime = null
AS

--
-- 目的：此存储过程是存储过程框架演示的一部分。
--      它是执行元数据版本的控制器存储过程的协调器存储过程。
--

SET NOCOUNT ON
/*******************************************************/
/*           将过程开始记录到过程日志                   */
/*******************************************************/
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
Values ('Procedure FWDemo.ProcessOrchestratorMD - STARTING',
GETDATE()
)
IF @RunDate is null
SET @RunDate = GETDATE()
DECLARE @RetStat int
SET @RetStat = 0
/*******************************************/
/*           执行 DailyProcessControllerMD */
/*******************************************/
EXEC @RetStat = FWDemo.DailyProcessControllerMD
IF @RetStat <> 0
GOTO EndOrchestratorMD
/************************************************************************/
/*  如果 @RunDate 是某月的第一天，则执行 MonthlyProcessControllerMD */
/************************************************************************/
IF DATEPART(DAY, @RunDate) = 1
BEGIN
EXEC @RetStat = FWDemo.MonthlyProcessControllerMD
IF @RetStat <> 0
GOTO EndOrchestratorMD
END
/************************************************************/
/*           将过程完成记录到过程日志                       */
/************************************************************/
EndOrchestratorMD:
IF @RetStat = 0
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.ProcessOrchestratorMD - COMPLETED',
GETDATE()
)
END
ELSE
BEGIN
INSERT INTO FWDemo.ProcessLog (
ProcessLogMessage,
CreateDate
)
VALUES ('Procedure FWDemo.ProcessOrchestratorMD - ERROR',
GETDATE()
)
END
RETURN @RetStat
GO
```

**代码清单 4-8** 新的元数据协调器

与原始版本一样，唯一需要设置/计划执行的是带有可选 `@RunDate` 参数覆盖的协调器。代码清单 4-9 展示了执行命令。

```sql
-- 如果运行日期是当前日期：
EXECUTE FWDemo.ProcessOrchestratorMD
-- 如果运行日期不是当前日期：
EXECUTE FWDemo.ProcessOrchestratorMD
@RunDate = ''
-- 注意：<rundate> 格式为 CCYYMMDD
```

**代码清单 4-9** 过程协调器的元数据版本执行语句

### 利用元数据进行故障排除

这是我们已构建系统最基本的元数据设置。那么，像我们在第 3 章最终遇到的错误情况下，如何完成执行呢？假设 `FWDemo.DailyProcess2` 未能成功执行。代码清单 4-10 显示了来自 `FWDemo.ProcessLog` 的消息，指出了错误发生的位置。

```
ProcessLogID      ProcessLogMessage                                                                                  CreateDate
39                Procedure FWDemo.ProcessOrchestratorMD – ERROR                         2018-09-17 20:47:00
38                Procedure FWDemo.DailyProcessControllerMD – ERROR                    2018-09-17 20:47:00
37                Procedure FWDemo.DailyProcess2 – 遇到问题                               2018-09-17 20:47:00
36                Procedure FWDemo.DailyProcess2 – 启动中                                 2018-09-17 20:46:00
35                正在执行 FWDemo.DailyProcess2 ApplicatonID = 1                          2018-09-17 20:46:00
ApplicatonProcessProcedureID = 2
34                Procedure FWDemo.DailyProcess1 – 已完成                                2018-09-17 20:46:00
33                Procedure FWDemo.DailyProcess1 – 启动中                                 2018-09-17 20:45:00
32                正在执行 FWDemo.DailyProcess1 ApplicatonID = 1                          2018-09-17 20:45:00
ApplicatonProcessProcedureID = 1
31                Procedure FWDemo.DailyProcessControllerMD – 启动中              2018-09-17 20:45:00
30                Procedure FWDemo.ProcessOrchestratorMD – 启动中                   2018-09-17 20:45:00
```

**代码清单 4-10** 过程日志的输出

一旦确定解决方案，我们可以从消息中看出 `FWDemo.DailyProcess1` 已经成功运行，因此无需重新运行。我们在消息中看到 `ApplicationProcessProcedureID` 值是 1。我们可以使用代码清单 4-11 中的代码来禁用 `FWDemo.DailyProcess1`，以便在应用程序执行时它不会运行。

```sql
-- 为重启禁用存储过程的执行
UPDATE FWDemo.ApplicationProcessProcedure
SET Active = 0
WHERE ApplicationProcessProcedureID = 1    -- 或使用过程日志中的值
```

**代码清单 4-11** 禁用执行的代码

此时，可以像之前一样重新启动计划任务，执行将从 `FWDemo.DailyProcess2` 开始，如代码清单 4-12 所示。任务完成后，只需使用类似代码清单 4-13 中的代码重置 `FWDemo.DailyProcess1` 的 `Active` 列。

```sql
-- 为下一次执行启用已禁用的存储过程
UPDATE FWDemo.ApplicationProcessProcedure
SET Active = 1
WHERE ApplicationProcessProcedureID = 1    -- 或使用过程日志中的值
```

**代码清单 4-13** 启用已禁用存储过程执行的代码


### 总结

`元数据`被用作`编排器`和`控制器`执行的路线图。目前，大多数情况都可以通过操作控制执行的`元数据`来处理，无需手动拆解和执行。任何添加到现有`控制器`的新`存储过程`，或添加到现有`编排器`的新`控制器`，只需通过部署待编译的`存储过程`并添加相应的`元数据`即可纳入。在大多数情况下（即使不是全部），现有的`存储过程`无需任何更改。

