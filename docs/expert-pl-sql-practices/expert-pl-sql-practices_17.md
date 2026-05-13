# 并行处理与大数据相关概念

### 影响并行处理的定律

解决问题的技术（或`算法`）由计算机科学家研究和实现，常常引出所谓的`计算定律`。其中一些定律与过去几年计算机架构和数据增长的趋势相互作用，影响了我们对并行处理的思考。

*   `摩尔定律`（1965）指出，可以容纳在集成电路上的晶体管数量每两年翻一番，并意味着计算机微处理器的速度和性能也将随之翻倍。从 1980 年代中期到大约 2004 年，这种`加速`基本成立，因为处理器频率扩展保持了步伐。然而，功耗开始成为问题，导致了需要并行才能充分利用该定律的`多核`处理器的出现。
*   `阿姆达尔定律`（1967）指出，在并行计算中使用多个处理器的程序，其加速受程序中顺序部分所需时间的限制。阿姆达尔定律对并行处理产生了冷却效应，因为它指出程序不可能比其顺序组件的执行速度更快。这意味着通过并行化程序所能获得的加速量是有限的，通常只能让程序比它们的串行对应物快 10 到 20 倍。
*   `古斯塔夫森定律`（1988）指出，具有非常大、重复数据集的问题可以有效地并行化，几乎是尴尬地并行（具有极小的顺序组件且易于划分为小的、独立单元的问题正式称为`尴尬并行`）。

这些定律之间的紧张关系在 2000 年代中期与另一个行业趋势——所谓“大数据”的兴起——相遇。

### 大数据的兴起

直到大约 2007 年，数据增长保持相当稳定和线性。然而，随着技术使捕获、存储和发送信息变得更容易，数据增长曲线出现了加速。2011 年，世界产生的数据量将是 2006 年的十倍。其中大部分数据实际上是机器生成的，因为传感器和基于网络活动的捕获成本已经下降。我们过去可能认为千兆字节的数据是巨大的，而现在我们则以太字节、拍字节和艾字节的数据来衡量。

淹没在数据海洋中，我们开始看到，试图以串行方式利用这些数据是不可行的。一天只有 86，400 秒，如果我们一次处理一个比特，就需要极高的速度来处理这种数据。阿姆达尔定律旨在预测固定问题大小和顺序部分占比 5%到 10%的加速情况，根本无法为处理这种以新速率增长的大型问题提供一个框架。10 到 20 倍的加速根本不够；我们需要超越串行加速，寻找处理越来越大的数据集的方法。古斯塔夫森定律包含了人们希望在相对较短的时间内解决越来越大的问题的观点，再加上 2004 年处理器频率加速的衰退，导致了人们对并行编程重新产生了兴趣。

## 并行处理与分布式处理

在这一点上，你可能想知道如何在数据库环境中实现并行处理。你可能接触过一些较新的数据处理框架，它们使用许多通过网络连接在一起的廉价独立计算机来执行数据分析任务，听起来像是并行处理。毕竟，我们从自然中得到的例子似乎是这种架构的完美匹配。

这类环境通常被归类为分布式计算环境，其特点是缺乏对共享内存区域的访问，因此需要通过某种网络进行消息传递来通信。在分布式环境中，由于系统需要通过这种（与本地处理速度相比）缓慢的网络协调它们的活动，因此存在显著的开销。此外，在分布式系统中，假定每个进程只能看到一部分数据——没有一个进程可以访问整个数据集。这限制了分布式算法的适用性。当数据已经分布在节点之间，或者数据集如此之大以至于分割和分发数据的开销任务（包括节点间通信开销）与整个数据集大小相比微不足道时，它们工作良好。即便如此，并行处理和分布式处理之间存在显著重叠。一个并行系统可以被描述为紧耦合的分布式系统，而一个分布式系统可以被描述为松耦合的并行系统。

好消息是，大多数分布式算法很容易转化为并行过程，因此如果你遇到一个听起来很有前景的分布式过程，你应该能够修改它以在并行架构上运行。

### 并行硬件架构

有几种硬件架构可能适用于并行处理。了解哪些架构特性使得系统不太适合并行过程的要求也是有用的。

*   `单计算机，单处理器，单核`：作为基本情况，这种计算机很少遇到，但值得注意的是，单核、单处理器、独立的计算机一次只能做一件事。无法在此类系统上实现并行过程。
*   `单计算机，单处理器，多核`：如果系统将多个内核作为独立处理器暴露给操作系统，那么就有可能并行利用这些内核。通常，这些内核共享对芯片上高速内存缓存的访问，因此可以轻松共享协调信息。
*   `单计算机，多处理器，单核或多核`：通常称为对称多处理系统（SMP），这类计算机一直是开放系统的主力。这些系统的处理器可能不共享本地高速内存，但所有处理器都通过内存总线共享对主计算机内存的访问。这种共享内存访问使处理器能够协调它们的活动。在其中一些系统中，处理器被分组在包含大量内存的处理器板上。如果该内存是在处理器板之间共享的，则这种访问可能是`非均匀`的，导致内存访问时间不均。
*   `SMP 计算机的集群系统`：集群系统通常采用某种高速专用系统互连网络来促进计算机节点之间的通信。这类计算机集群可能共享也可能不共享磁盘访问。当它们共享磁盘访问时，它们被称为`共享一切`架构。当它们不共享时，则被称为`无共享`架构，更类似于分布式系统。在某些集群变体中，高速互连实际上允许从所有其他节点远程访问每个单独计算机节点的内存。虽然这种远程内存访问比本地内存访问慢得多，但它比使用磁盘子系统在节点之间传输信息要快得多。从版本 9 开始，使用缓存融合的`Oracle Real Application Clustering`技术就利用了这一功能。
*   `SMP 计算机的分布式或网络化系统`：这种架构的共享资源有限，因为每台计算机都被设计为独立的。虽然这种架构比集群系统具有更高的容错能力，但它依赖于消息传递进行活动协调，更适用于分布式算法。



### 明确目标

现在，你已经对并行处理有了基本了解，知道它与分布式处理的区别，以及可能支持并行程序的底层计算机架构，接下来你需要理解你的问题集，以及你希望通过并行处理实现什么。首先需要考虑的事情之一是，你是想加快现有流程的处理速度，还是想处理规模不断增长的、多变的数据集。

#### 加速

加速的概念是：将一个以串行方式运行的过程修改为并行运行，并评估其运行速度提升了多少。它的定义是串行运行时间与并行运行时间的比值。

`加速比 = 串行运行时间 / 并行运行时间`

理想情况下，你希望并行化工作的效率尽可能高。例如，如果你编写的并行程序使用了五个处理器而不是一个，你可能期望该程序的完成时间是串行程序的 1/5。不幸的是，将数据集和任务分割成可由并行程序处理的片段，以及汇总结果的活动，都会带来开销。这些任务增加了并行程序所需的时间，导致加速比相对于应用于该问题的处理器数量而言，并非 100% 高效。如果这些任务占串行运行时间的 10%，那么最大可能的加速比就只有 10 倍。这就是阿姆达尔定律的基础。

#### 扩展

扩展的概念是在相对恒定的时间窗口内处理越来越大的任务。它可以定义为并行进程在时间窗口内可处理的任务规模与串行进程可处理的任务规模之比。

`扩展度 = 并行完成的任务总规模 / 串行完成的任务规模`

理想情况下，你希望每增加一个处理器，就能增加需要处理的任务规模。例如，如果你编写了一个使用四个处理器可以处理 100 个任务项的并行程序，那么你希望该程序在增加第五个处理器时能够处理 125 个任务项。100% 的可扩展性意味着，每增加一个新的处理器或计算节点，你能完成的增量任务项数量不会减少；这通常只适用于增量任务集完全独立于所有其他任务项的情况。如果情况并非如此，你将获得略低于 100% 的可扩展性。然而，在许多情况下，任务项独立是可能的，你应该努力识别这些情况。这对于考虑如何设计系统以处理所需处理数据量级的增长特别有用。思考如何在合理固定的时间内处理数据集规模成倍增加的问题，将迫使你考虑如何将处理过程并行化。这就是古斯塔夫森定律的基础。

### 并行度

当将串行任务拆分为并行任务项时，你可以决定希望同时运行多少个任务项。这种并发级别被称为 `并行度`，通常缩写为 `DOP`。

你可能会认为，直接将 `DOP` 设置为并行任务项的数量是个好主意；例如，如果你可以将一个串行进程划分为 12 个任务，你可能希望一次性运行所有 12 个任务。然而，在某些情况下，你可能希望一次只运行三个任务，分四批进行，也许是为了在并行任务运行时减少系统的资源消耗。你应该仔细考虑希望系统在处理并行任务时达到多繁忙的程度。如果你决定将 `DOP` 设置为小于任务项的数量，那么剩余的任务项将会等待——这种情况被称为 `并行排队`。

### 并行处理的候选工作负载

现在你已经掌握了特定进程的目标（加速或扩展），是时候看看你正在处理何种工作负载，以及将它们转换为并行进程将对系统整体产生什么影响。

重要的是要理解，将任何串行进程转换为并行进程都会在并行进程运行期间增加系统的资源消耗。这很容易通过引言中杂货店的例子想象出来。想象一个商店一次只允许一个人进入，你会发现任何给定时间段内，商店都不太繁忙。但如果你允许购物者并行购物，你可以看到在特定的时间窗口内，商店要繁忙得多。另一种看待这个问题的方式是，在串行进程运行时测量系统的 CPU 利用率等指标，并将其乘以你为任务考虑的并行度；你需要确保有足够的 CPU 来处理你期望的并行工作负载。在考虑将并行性引入已运行的系统时，这一点尤其重要。

### 并行性与 OLTP

联机事务处理系统通常具有高吞吐量、高并发性、执行最小工作量的短事务的特点。这些工作单元通常是独立的，例如单个客户下的订单。在大多数情况下，客户的活动彼此独立，因此可以并行运行。

虽然这并非严格意义上的并行处理——因为你不是将一个大任务分解成更小的独立任务单元——但 OLTP 活动确实通过允许多个任务同时并发运行，实现了更高的系统资源利用率。设计良好的 OLTP 系统能充分利用 SMP 和集群数据库服务器上的所有 CPU 及 CPU 核心，几乎没有闲置的 CPU。然而，你可能会倾向于进一步将单个 OLTP 事务分解为可并行化的子任务。

例如，假设你有一个在线购物网站，部分客户向你批量采购；他们可能是批发商。这些客户的购物车中可能包含成千上万的商品——远多于普通消费者。处理这些购物车中的商品（例如计算每件商品的应税性质）可能需要比你期望更长的时间；你希望他们的体验与普通客户的体验相似。如果可能，你可能希望并行处理购物车中的商品，或许采用较低的并行度，以使购物车“感觉”比实际小 4-5 倍。这当然是可能的，但你需要小心，不要让额外的并发工作负载压垮你的系统。检查你系统的 CPU 利用率，看看是否能处理与你的并行算法相关的额外并发量。对于超大型购物车，采用较低的并行度可能是可行的。

```pseudo
if (cart.size() > threshold) {
    parallel_process(cart.items, degree_of_parallelism=4);
}
```


### 并行性与非 OLTP 工作负载

非 OLTP 工作负载，例如大型报告、统计分析（或许是为了衡量客户流失率等指标）或定期的全量数据处理活动（如为每位客户生成定期账单或向特定客户进行群发邮件），通常都非常适合并行处理活动。

这类活动可能包括并行检查大量数据以识别待处理的特定项目、构建中间汇总结果或批量更新多行数据。它们也可能执行诸如并行转换大量数据等活动。通常，这类活动会在专门为此类处理保留的系统或时间窗口中运行，因此有充足的 CPU 可用。如果串行作业运行时间很长，但系统的整体 CPU 利用率仍然很低，或者显示实际使用的可用处理器非常少，那么你就知道这些系统已经具备了并行开发的成熟条件。

### MapReduce 编程模型

MapReduce 是一种用于并行处理大型数据集的编程模型。该编程模型简单明了，其原语被定义为以独立方式处理数据的`块`（chunks），从而能够并行处理这些数据块。该模型首先通过“分片”（sharding）或将数据拆分成集合，传递给 map 函数。一个 `map` 函数通常接收一组输入数据，并产生一组键/值对，然后这些键/值对可以交给一组 `reduce` 函数，由它们对映射后的列表执行聚合操作。该模型因其通用的原语而广受欢迎，这些原语已被用于执行与文本扫描和过滤相关的各种任务。

许多新的数据库系统支持 MapReduce 原语，以此作为执行大规模聚合和过滤任务的一种方式。如果你熟悉这种范式，我将在接下来的几页中带你了解如何在 PL/SQL 中实现它。

### 在考虑 PL/SQL 之前

请务必记住，数据库通常可以自动并行化许多 SQL DML 和 DDL 操作，而无需任何特殊的 PL/SQL 编程。在尝试手动并行化你的存储过程之前，你可能想先研究一下这些选项，看看自动并行化是否就能提供你所期望的收益。在接下来的例子中，你将利用自动并行化的一些架构基础，但首先，让你熟悉一下可用的功能可能会很有帮助。Oracle 将此架构称为 `并行执行`（parallel execution），它可以应用于几种不同类型的操作。

-   `并行查询`：涉及全表和分区扫描的大型查询可以让这些扫描并行执行，包括表扫描、索引快速全扫描和分区索引范围扫描。连接方法如嵌套循环、排序合并连接、哈希连接以及星型转换也可以并行化。包括 `GROUP BY`、`NOT IN`、`UNION`、`UNION ALL`、`CUBE` 和 `ROLLUP` 在内的聚合和集合处理方法同样可以并行化。
-   `并行 DML`：`INSERT AS SELECT`、`UPDATE`、`DELETE` 和 `MERGE` 操作都可以并行执行，包括使用单个 DML 语句向多个表插入数据的能力。
-   `并行 DDL`：`CREATE TABLE AS SELECT`、`CREATE INDEX`、`REBUILD INDEX`、`REBUILD INDEX PARTITION` 以及 `MOVE`、`SPLIT` 或 `COALESCE PARTITION` 都可以并行运行。

一如既往，在考虑并行替代方案之前，你应该花时间确保你的串行操作尽可能高效。虽然并行执行可能会带来更快的执行时间，但你应该警惕与加速和纵向扩展相关的效率问题，正如本章前面所讨论的那样。由于大多数并行化技术的效率都低于 100%，你会给系统引入额外的资源消耗，因此你需要确保这种额外消耗是值得的。这对于通常以接近 100%利用率运行的系统尤其重要；你可能想看看能否通过运行更高效的串行类型操作来弥补并行化的开销。

### 可用于并行活动的进程

既然你已决定尝试在 PL/SQL 中实现并行流程，那么了解你的执行选项就很有用了。由于数据库通过通常映射到单独连接和操作系统进程的会话来支持并发活动，你需要熟悉哪些操作系统进程可以支持你的存储过程。有三种可用选项。

-   `并行执行服务器`：这些进程有时被称为并行查询从属进程，由数据库自动配置以支持 SQL 语句的并行执行。通常由参数 `parallel_min_servers` 和 `parallel_max_servers` 设置，为每个数据库实例建立一组操作系统进程池。
-   `作业队列进程`：使用 `DBMS_JOB` 或 `DBMS_SCHEDULER` 包，你可以提交 PL/SQL 例程，由服务器作业队列进程以异步方式运行。虽然无法保证任何时间点有多少进程可用，但最大数量由 `job_queue_processes` 参数控制。
-   `用户会话进程`：通过手动创建自己的、建立到数据库连接的操作系统进程，你可以创建自己的并行处理支持框架。你将需要在数据库外部或使用数据库数据结构来控制并行度和任务管理。

在本章中，你将只研究使用 `并行执行服务器`（通过并行流水线函数）来支持你的并行 PL/SQL 例程。

### 将并行执行服务器用于 MapReduce

到目前为止，你已经看到 Oracle（特别是企业版）为针对表的并行操作提供了几种不同的选项，特别是在并行查询领域，其中表数据被分配到并行执行服务器中进行过滤、连接和排序。虽然这个选项看起来很有吸引力，但如何将其与 PL/SQL 结合使用呢？

答案在于数据库能够将返回值的 PL/SQL 函数视为一个伪表；这种能力被称为 PL/SQL `流水线函数`（pipelined functions），首次出现在 Oracle 9i 中。在 Oracle9i 之前，可以将 PL/SQL 函数 `CAST`（转换）为集合并从中选择，但该过程需要在返回任何数据之前在内存中实例化整个结果集。保存完整结果集所需的内存使得它对于你考虑用于并行化的大型问题而言变得不可行。这个问题的答案就是流水线函数。


## 管道化表函数

管道化表函数使你能够使用 `PL/SQL` 以编程方式生成数据，指定你希望在每一行中获得的内容，并使数据库能够像查询任何其他表一样从你的函数中进行选择。你是通过使用数据库对象类型或 `PL/SQL` 记录类型来声明记录的结构，然后创建一个描述该记录“表”的集合类型来实现这一点的。

对于你的 `MapReduce` 示例，你将定义一个键/值对记录，然后定义一个描述此类记录列表的集合类型，如下所示：

```sql
create or replace package map_reduce_type as
  type key_value_pair is record (
    key_item    varchar2(32),
    value_item  number
  );
  type key_value_pairs is table of key_value_pair;
end map_reduce_type;
/
```

你将尝试使用并行管道化函数解决的具体问题是统计字符串列表中字母的出现次数。对于你的字符串列表，你将使用标准 Oracle 数据库安装中的超过 100,000 个列名；从该列表中，你将尝试确定哪个字母是最常见的。

在 `MapReduce` 模型中，你将首先将每个字符串分解为其组成字母，并“发出”每个字符串中的字母列表，每个字母的计数为 1（你的 `Map` 函数）。一旦你完成了字母的映射，你将把它们交给一个 `Reduce` 函数进行计数。让我们深入这个例子，看看它是如何工作的：

首先声明你的 `Map` 函数，表明它将使用管道化函数返回一个键/值对列表。你的函数将接受一个输入参数：一个要分解成单个字母的字符串。代码如下：

```sql
create or replace package map_letter_count as
  function result_set (p_input in varchar2)
    return map_reduce_type.key_value_pairs pipelined;
end map_letter_count;
/
```

你的函数简单地接收输入字符串，逐个字符遍历它，并为每个字母“管道传输”一个值为 1 的行作为输出，像这样：

```sql
create or replace package body map_letter_count as

  function result_set (p_input in varchar2)
    return map_reduce_type.key_value_pairs pipelined is

    l_key_value_pair map_reduce_type.key_value_pair;

    begin

      for i in 1..length(p_input) loop

        l_key_value_pair.key_item := substr(p_input,i,1);
        l_key_value_pair.value_item := 1;

        pipe row(l_key_value_pair);

      end loop;

      return;

    end result_set;

end map_letter_count;
/
```

现在，你可以通过在 `SELECT` 语句中使用此函数并将其转换为表来检索结果。

```sql
select * from table(
map_letter_count.result_set('Hello, world!')
);
```

```
KEY_ITEM                     VALUE_ITEM
---------------------------- ----------------------
H                            1
e                            1
l                            1
l                            1
o                            1
,                            1
                             1
w                            1
o                            1
r                            1
l                            1
d                            1
!                            1

13 rows selected
```

到目前为止，你已经看到了如何使用管道化函数接收输入参数，并使用 `pipe row` 构造将其转换为 `MapReduce` 键/值对列表。这很好，但目前你的函数仅限于处理单个字符串。让我们看看如何扩展它以处理大量的字符串集合。

为此，你可以修改你的 `Map` 函数，使其以数据游标的形式接受字符串或文档列表作为输入，如下所示：

```sql
create or replace package map_letter_count as
  function result_set (p_documents in sys_refcursor)
    return map_reduce_type.key_value_pairs pipelined;
end map_letter_count;
/
```

要设置你的字符串列表，基于 `DBA_TAB_COLUMNS` 创建一个 `documents` 表；为了使你的测试易于管理，将其限制为 10 行。

```sql
create table documents
as
select rownum doc_id, column_name text
from dba_tab_columns
where rownum <= 10;
```

现在你的 `Map` 函数将依次遍历每个文档以执行其映射函数。

```sql
create or replace package body map_letter_count as

  function result_set (p_documents in sys_refcursor)
    return map_reduce_type.key_value_pairs pipelined is

    type document_type is record (
      doc_id number,
      text   varchar2(4000)
    );
    l_document       document_type;
    l_key_value_pair map_reduce_type.key_value_pair;

    begin

      fetch p_documents into l_document;
      loop
        exit when p_documents%notfound;

        for i in 1..length(l_document.text) loop

           l_key_value_pair.key_item := substr(l_document.text,i,1);
           l_key_value_pair.value_item := 1;

           pipe row(l_key_value_pair);

        end loop;

        fetch p_documents into l_document;

      end loop;

      return;

    end result_set;

end map_letter_count;
/
```

现在从你的 `Map` 函数中进行选择，会返回表中所有文档的映射键和值。

```sql
select * from table(
map_letter_count.result_set(
cursor(select doc_id, text from documents)
)
);
```

```
KEY_ITEM                     VALUE_ITEM
---------------------------- ----------------------
D                            1
_                            1
O                            1
B                            1
J                            1
#                            1
O                            1
R                            1
D                            1
E                            1
R                            1
#                            1
C                            1
…snip…
O                            1
L                            1
U                            1
D                            1
P                            1
R                            1
I                            1
O                            1
R                            1
I                            1
T                            1
Y                            1
S                            1
T                            1
A                            1
T                            1
E                            1

63 rows selected
```

针对整个数据集运行此操作会产生超过 100 万行。

```sql
drop table documents;
create table documents
as
select rownum doc_id, column_name text from dba_tab_columns;

select count(*)
from
(
select * from table(
map_letter_count.result_set(
cursor(select doc_id, text from documents)
)
)
);
```

```
COUNT(*)
----------------------
1132707
```

然而，这整个过程是以串行方式运行的。

```sql
select * from v$pq_sesstat;
```

```
STATISTIC                      LAST_QUERY             SESSION_TOTAL
------------------------------ ---------------------- ----------------------
Queries Parallelized           0                      0
DML Parallelized               0                      0
DDL Parallelized               0                      0
DFO Trees                      0                      0
Server Threads                 0                      0
Allocation Height              0                      0
Allocation Width               0                      0
Local Msgs Sent                0                      0
Distr Msgs Sent                0                      0
Local Msgs Recv'd              0                      0
Distr Msgs Recv'd              0                      0

11 rows selected
```

试图强制它并行运行并不奏效，正如你所看到的：



```sql
select count(*)
from
(
select /*+ parallel */ * from table(
map_letter_count.result_set(
cursor(select doc_id, text from documents)
)
)
);
```

```
COUNT(*)
----------------------
1132707
```

```sql
select * from v$pq_sesstat;
```

```
STATISTIC                      LAST_QUERY             SESSION_TOTAL
------------------------------ ---------------------- ----------------------
Queries Parallelized           0                      0
DML Parallelized               0                      0
DDL Parallelized               0                      0
DFO Trees                      0                      0
Server Threads                 0                      0
Allocation Height              0                      0
Allocation Width               0                      0
Local Msgs Sent                0                      0
Distr Msgs Sent                0                      0
Local Msgs Recv'd              0                      0
Distr Msgs Recv'd              0                      0
```

现在你已经定义好了 Map 函数，如何通过 PL/SQL 管道函数（pipelined functions）使其并行运行呢？答案是在函数声明时使用 `PARALLEL_ENABLE` 构造。

```sql
create or replace package map_letter_count as
  function result_set (p_documents in sys_refcursor)
    return map_reduce_type.key_value_pairs pipelined
      parallel_enable (partition p_documents by any);
end map_letter_count;
/
```

`PARALLEL_ENABLE` 告诉 Oracle，这个 PL/SQL 函数可以针对数据块并行执行。为了向函数的并行实例提供这些数据块，你需要以游标（cursor）的形式定义一个数据集（此处使用了通用的 `sys_refcursor`）。通过为 `PARALLEL_ENABLE` 参数使用 `PARTITION` 选项，你告诉数据库“分片”或“拆分”数据。`BY ANY` 子句告诉数据库，你不在意数据如何拆分，只需选取大小基本相等的子集即可。

然后，你修改 Map 函数，从分配到的子集中检索每个项目（通过 `FETCH` 从游标获取），并将每个项目处理成目标键/值对。

```sql
create or replace package body map_letter_count as

  function result_set (p_documents in sys_refcursor)
    return map_reduce_type.key_value_pairs pipelined
    parallel_enable (partition p_documents by any) is

    type document_type is record (
      doc_id number,
      text   varchar2(4000)
    );
    l_document       document_type;
    l_key_value_pair map_reduce_type.key_value_pair;

    begin

      fetch p_documents into l_document;
      loop
        exit when p_documents%notfound;

        for i in 1..length(l_document.text) loop

           l_key_value_pair.key_item := substr(l_document.text,i,1);
           l_key_value_pair.value_item := 1;

           pipe row(l_key_value_pair);

        end loop;

        fetch p_documents into l_document;

      end loop;

      return;

    end result_set;

end map_letter_count;
/
```

现在，当你告诉服务器并行运行此函数时，你会看到它启动了多个函数实例来执行 Map 操作。

```sql
select count(*)
from
(
select /*+ parallel */ * from table(
map_letter_count.result_set(
cursor(select doc_id, text from documents)
)
)
);
```

```
COUNT(*)
----------------------
1132707
```

```
STATISTIC                      LAST_QUERY             SESSION_TOTAL
------------------------------ ---------------------- ----------------------
Queries Parallelized           1                      1
DML Parallelized               0                      0
DDL Parallelized               0                      0
DFO Trees                      1                      1
Server Threads                 2                      0
Allocation Height              2                      0
Allocation Width               1                      0
Local Msgs Sent                56                     56
Distr Msgs Sent                0                      0
Local Msgs Recv'd              56                     56
Distr Msgs Recv'd              0                      0
```

```
11 rows selected
```

既然你已完成 Map 函数，现在来讨论 Reduce 函数。如前所述，Reduce 函数通常接收 Map 函数的结果，并使用聚合原语（aggregation primitives）将其“归约”。在你的例子中，你将让 Reduce 函数遍历 Map 的结果，并汇总每个字母出现的次数。你还会在函数中引入一个细微的错误，以强调 MapReduce 编程模型的一个方面，以及如何使用 PL/SQL 并行管道函数选项来解决它。你将从函数定义开始，因为你希望 Reducer 并行运行，所以你将使用 `PARALLEL_ENABLE` 选项，并让它像之前一样对 Map 函数的结果进行分区。

```sql
create or replace package reduce_letter_count is
  function result_set (p_key_value_pairs in sys_refcursor)
    return map_reduce_type.key_value_pairs pipelined
      parallel_enable (partition p_key_value_pairs by any);
end reduce_letter_count;
/
```

你的 Reduce 函数定义相当简单，因为我们知道 Reducer 总是处理来自 Map 函数的键/值对。你的 Reduce 代码也会很简单。你只需逐个处理来自 Map 函数的结果，累加值，并在切换到新字母时发出一个结果。代码如下：

```sql
create or replace package body reduce_letter_count as

  function result_set (p_key_value_pairs in sys_refcursor)
    return map_reduce_type.key_value_pairs pipelined
      parallel_enable (partition p_key_value_pairs by any) is

    l_in1_key_value_pair map_reduce_type.key_value_pair;
    l_out_key_value_pair map_reduce_type.key_value_pair;

    begin

      fetch p_key_value_pairs into l_in1_key_value_pair;
      l_out_key_value_pair.key_item := l_in1_key_value_pair.key_item;
      l_out_key_value_pair.value_item := 0;

      loop
        exit when p_key_value_pairs%notfound;

        if l_out_key_value_pair.key_item = l_in1_key_value_pair.key_item
        then

          l_out_key_value_pair.value_item :=
            l_out_key_value_pair.value_item +
            l_in1_key_value_pair.value_item;

        else

          pipe row(l_out_key_value_pair);

          l_out_key_value_pair.key_item := l_in1_key_value_pair.key_item;
          l_out_key_value_pair.value_item := 1;

        end if;

        fetch p_key_value_pairs into l_in1_key_value_pair;

      end loop;

      pipe row(l_out_key_value_pair);

      return;

    end result_set;

end reduce_letter_count;
/
```

现在，为了测试你的 Reduce 函数，你将回头定义一个仅针对单个字符串工作的单一 Map。

```sql
create or replace package single_map_letter_count as
  function result_set (p_input in varchar2)
    return map_reduce_type.key_value_pairs pipelined;
end single_map_letter_count;
/
```

```sql
create or replace package body single_map_letter_count as

  function result_set (p_input in varchar2)
    return map_reduce_type.key_value_pairs pipelined is

    l_key_value_pair map_reduce_type.key_value_pair;

    begin

      for i in 1..length(p_input) loop
```


