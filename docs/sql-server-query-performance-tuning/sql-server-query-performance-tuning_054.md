# 第六章：查询性能指标

`SQL Server`性能下降的一个常见原因是繁重的数据库应用程序工作负载——即查询本身的性质和数量。因此，要分析系统瓶颈的原因，检查数据库应用程序的工作负载并识别对系统资源造成最大压力的`SQL`查询至关重要。为此，你可以使用`Extended Events`和其他`Management Studio`工具。

在本章中，我将涵盖以下主题：
* `Extended Events`的基础知识
* 如何使用`Extended Events`分析`SQL Server`工作负载并识别代价高的`SQL`查询
* 如何通过动态管理对象跟踪查询性能

## Extended Events

`Extended Events`在`SQL Server 2008`中引入，但由于没有`GUI`且设置代码相当复杂，它并未被广泛用于捕获性能指标。随着`SQL Server 2012`的推出，一个用于管理`Extended Events`的`GUI`被引入，消除了阻碍其成为收集查询性能指标以及其他指标和度量首选机制的最后一个问题。`SQL Profiler`之前是收集这些指标的最佳机制，现已弃用，并将在未来一两个版本中从产品中完全移除。同样不错的`Trace`事件也仍然可用，但将与`Profiler`一同逐步淘汰。因此，本书中的大多数示例将使用`Extended Events`。

`Extended Events`允许你执行以下操作：


