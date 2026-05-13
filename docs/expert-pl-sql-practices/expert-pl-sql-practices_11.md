# 动态 SQL：应对未知

作者：Michael Rosenblum

过去十年，我参加了在美国各地举办的众多 Oracle 会议。一次又一次，我听到演讲者们谈论如何“更好、更快、更经济”地构建系统。但当这些演讲者走下讲台，与你一对一讨论同样问题时，他们传递的信息却远没有那么乐观。经常被引用的、约 75%的大型 IT 项目失败率至今仍是现实。再加上那些“将失败宣布为成功”的案例（即没人有勇气承认错误），情况就更加明显：我们当代的软件开发流程正面临一场危机。

本章中，我将假设我们生活在一个稍好一点的世界：这里没有企业内部政治斗争，系统架构师清楚自己在做什么，开发者至少明白 OTN 的含义。但即使在这个改善后的世界里，系统开发过程中仍存在一些无法避免的固有风险：

*   清晰阐述待建系统的需求具有挑战性。
*   构建一个真正满足所有既定需求的系统很困难。
*   构建一个在短期内不需要大量修改的系统非常困难。
*   构建一个不会过时（迟早会过时）的系统是不可能的。

如果最后一点可以被视作常识，那么我的许多同事会对前三点强烈反对。然而，现实情况是，永远不会有 100%完美的分析、100%完整的需求集、100%足够且永远不需要升级的硬件，等等。在 IT 行业，我们必须接受一个事实：我们随时都必须预料到意外情况。

![images](img/square.jpg) `开发者信条` 整个开发过程的重点应该从我们已知的内容转向我们未知的内容。

不幸的是，未知的事情很多：

*   涉及哪些元素？例如，系统需要一个季度报告机制，但没有季度汇总表。
*   你该如何着手？这是 DBA 的噩梦：如果一个全局搜索屏幕包含来自不同表的数十个潜在条件，如何确保其性能足够好？
*   你能否继续？对于每一个限制，你通常至少有一个变通方案或“后门”。但如果这个后门的位置在下一个版本或更新中发生了变化呢？

幸运的是，回答这些（以及类似）问题有多种不同的方法。本章将讨论一种名为动态 SQL 的功能如何帮助解决前面提到的一些问题，以及你如何避免导致系统失败和过时的一些主要陷阱。

### 英雄登场

动态 SQL 的概念相当直接。这个功能允许你将代码（包括 SQL 和 PL/SQL）作为文本来构建，并在运行时处理它——仅此而已。动态 SQL 提供了一个程序在执行过程中编写另一个程序的能力。也就是说，理解这个功能所带来的潜在影响和可能性至关重要。这些问题将在本章中讨论。

![images](img/square.jpg) `注` 我所说的“处理”是指在任何编程语言中启动一个程序所需的整个事件链——解析/执行/[获取]（最后一步是可选的）。详细讨论这个主题超出了本章的范围，但了解每个步骤的基础知识对于正确使用动态 SQL 至关重要。

重要的是要认识到，有不同方式可以实现动态 SQL，需要讨论的有：

*   原生动态 SQL
*   动态游标
*   `DBMS_SQL`包

线上和出版物中有许多优秀的参考资料解释了这些方法各自的语法方面。本书的目的是展示最佳实践，而非提供参考指南，但强调每种动态 SQL 的关键点作为进一步讨论的共同基础是很有用的。

![images](img/square.jpg) `技术说明 #1` 尽管“动态 SQL”这个术语已被整个 Oracle 社区接受，但它并非百分之百精确，因为它涵盖了构建 SQL 语句和 PL/SQL 块。但“动态 SQL 和 PL/SQL”听起来太拗口，因此全文将使用动态 SQL。

![images](img/square.jpg) `技术说明 #2` 截至本文撰写时，Oracle 10g 和 11g 的使用大致相当。除非所描述的功能仅存在于 11g（此类情况会明确提及），否则此处使用的示例均与 10g 兼容。

### 原生动态 SQL

大约 95%的动态 SQL 实现都涉及以下`EXECUTE IMMEDIATE`命令的变体：

```
DECLARE
   v_variable_tx VARCHAR2(<Length>)|CLOB;
BEGIN
   v_variable_tx:='whatever_you_want';
   EXECUTE IMMEDIATE v_variable_tx [additional options];
END;
```
或

```
BEGIN
   EXECUTE IMMEDIATE 'whatever_you_want' [additional options];
END;
```
或

```
BEGIN
   EXECUTE IMMEDIATE 'whatever_you_want1'||'whatever_you_want2'||… [additional options];
END;
```

在 Oracle 10g 及之前版本中，`EXECUTE IMMEDIATE`可以处理的代码总长度（包括拼接结果）是 32KB（更大的代码集由`DBMS_SQL`包处理）。从 11g 开始，可以将`CLOB`作为输入参数传递。对架构师来说，一个很好的问题是：为什么会有人试图使用单个语句动态处理超过 32KB 的代码？但根据我的经验，这种情况确实存在。


##### 原生动态 SQL 示例 #1

具体的语法细节可能需要另写 30 页来解释，但既然本书是为更有经验的用户编写的，可以合理地期望读者知道如何使用文档。与其深入探讨 PL/SQL，不如提供一个关于**为什么**需要动态 SQL 的典型示例。为此，假设以下需求：

*   系统预计会有许多带有相关 LOV（值列表）查找表的查找字段，这些字段很可能在后续过程中扩展。与其单独构建每个 LOV，不如有一个集中式解决方案来处理所有内容。
*   所有 LOV 都应遵循相同的格式，即两列（ID/DISPLAY），第一列是查找键，第二列是文本。

这些需求非常适合使用动态 SQL：在运行时定义操作的重复模式，其中一些在初始编码时可能未知。以下代码在此场景中很有用：

```sql
CREATE TYPE lov_t IS OBJECT (id_nr NUMBER, display_tx VARCHAR2(256));
```

```sql
CREATE TYPE lov_tt AS TABLE OF lov_t;
```

```sql
CREATE FUNCTION f_getlov_tt (
          i_table_tx    VARCHAR2,
          i_id_tx       VARCHAR2,
          i_display_tx  VARCHAR2,
          i_order_nr    VARCHAR2,
          i_limit_nr    NUMBER:=100)
RETURN lov_tt
IS
    v_out_tt lov_tt := lov_tt();
    v_sql_tx VARCHAR2(32767);
BEGIN
    v_sql_tx:='SELECT lov_item_t '||
                    '(SELECT lov_t(' ||
                                dbms_assert.simple_sql_name(i_id_tx)||', '||
                      dbms_assert.simple_sql_name(i_display_tx)||') lov_item_t '||
                    ' FROM '||dbms_assert.simple_sql_name(i_table_tx)||
                       ' order by '||dbms_assert.simple_sql_name(i_order_nr)||
                ')' ||
             ' WHERE ROWNUM <= :limit';
    EXECUTE IMMEDIATE v_sql_tx BULK COLLECT INTO v_out_tt USING i_limit_nr;
    RETURN v_out_tt;
END;
```

```sql
SELECT * FROM TABLE(CAST(f_getlov_tt(:1,:2,:3,:4,:5) AS lov_tt))
```

这个示例包含了动态 SQL 的所有核心语法元素：

*   要执行的代码由 PL/SQL 字符串变量表示。
*   通过绑定变量传入或传出其他参数。绑定变量是逻辑占位符，在执行步骤与实际参数关联。
*   绑定变量用于值，而非结构元素。这就是为什么表名和列名被连接到语句中。
*   `DBMS_ASSERT`包有助于防止代码注入（因为你必须使用连接），它强制要求表名和列名是“简单的 SQL 名称”（没有空格，没有分隔符等）。
*   由于 SQL 语句有输出，输出被返回到匹配类型的 PL/SQL 变量（原生动态 SQL 允许用户定义的数据类型，包括集合）。

示例中的最后一条语句（调用`F_GETLOV_TT`的 SELECT 语句）应该提供给前端开发人员。这是他们集成到应用程序中所需的唯一信息。其他所有内容都可以由数据库开发人员处理，包括优化、权限、特殊处理逻辑等。对于这些项目中的每一项，都有一个单一的修改点和单一的控制点。这种“单点”概念是帮助最大限度地减少未来开发、调试和审计工作量的最关键特性之一。

##### 原生动态 SQL 示例 #2

示例#1 针对的是构建新系统的开发人员。示例#2 适用于维护现有系统的人员。

在某个时候，Oracle 决定更改删除基于函数的索引时的默认行为：所有引用拥有此类索引的表的 PL/SQL 对象都会自动失效。正如预期的那样，这一改变在许多批处理例程中造成了严重破坏。因此，Oracle 提供了一个通过设置跟踪事件来解决的变通方法。删除任何基于函数的索引的例程采用以下形式：

```sql
CREATE PROCEDURE p_dropFBIndex (i_index_tx VARCHAR2) is
BEGIN
    EXECUTE IMMEDIATE
        'ALTER SESSION SET EVENTS ''10624 trace name context forever, level 12''';
    EXECUTE IMMEDIATE 'drop index '||i_index_tx;
    EXECUTE IMMEDIATE
        'ALTER SESSION SET EVENTS ''10624 trace name context off''';
END;
```

这个示例说明了动态 SQL 的另一个关键元素。你不仅限于 DML。任何有效的 PL/SQL 匿名块或 SQL 语句，包括 DDL 或`ALTER SESSION`，都可以在运行时操作中执行。因此，DBA 不再需要他们最喜欢的脚本生成脚本的脚本，因为所有这些情况都可以在数据库中直接处理。

### 动态游标

动态游标的想法与执行动态 SQL 类似。但不是运行整个语句，而是打开一个游标并将 SELECT 语句作为参数传递。例如：

```sql
DECLARE
    v_cur SYS_REFCURSOR;
    v_sql_tx VARCHAR2(<长度>)|CLOB:='有效的 _SQL_ 查询';
    v_rec ...<记录类型>;
        或
    v_tt ...<记录集合>;
BEGIN
    OPEN v_cur FOR v_sql_tx [<附加参数>];

    FETCH v_cur INTO v_rec;
          或
    FETCH v_cur BULK COLLECT INTO v_tt [<限制 N>];

   CLOSE v_cur;
END;
```

根据我之前的经验，有两种情况动态游标可能派上用场。两个示例如下。

## 动态游标示例 #1

使用动态游标的第一个案例与那些大量使用 `REF CURSOR` 类型变量作为不同应用层之间通信机制的场景有关。不幸的是，在大多数情况下，这类变量是由中间层创建的，而数据库端专家的参与度往往极低。我见过太多案例，业务逻辑 100%有效，但为了解决直接功能问题而进行的实现，却造成了维护、调试甚至安全方面的噩梦！

修复此问题的正确方法是将构建 `REF CURSOR` 的过程下沉到数据库层，因为所有创建的逻辑在那里更容易控制（尽管，正确关闭所有打开的游标仍然是中间层开发人员的责任；否则系统容易发生“游标泄漏”）。基础层面的包装函数应如下所示：

```sql
CREATE FUNCTION f_getRefCursor_REF (i_type_tx VARCHAR2)
RETURN SYS_REFCURSOR
IS
  v_out_ref SYS_REFCURSOR;
  v_sql_tx VARCHAR2(32767);
BEGIN
    if i_type_tx = 'A' then
        v_sql_tx:=<some code to build query A>;  
    elsif i_type_tx = 'B' then
        v_sql_tx:=<some other code to build query B>;  
  ...
    END IF;
    OPEN v_out_ref FOR v_sql_tx;
    RETURN v_out_ref;
END;
```

当任务不仅仅是构建和执行查询，还要向其中传入一些绑定变量时，事情就变得稍微有趣一些了。解决这个问题有几种不同的方法。最简单的是创建一个包含多个全局变量和相应返回函数的包。因此，获取正确结果的整个过程包括：设置所有适当的变量，并在同一会话中立即调用 `F_GETREFCURSOR_REF`。然而，我刚才描述的解决方案并不完美，因为中间层通常以纯粹的无状态方式与数据库通信。在这种情况下，就不可能使用 `PL/SQL` 包变量。

![images](img/square.jpg) 注意：我所说的中间层无状态实现，是指每次数据库调用都获得一个独立的数据库会话（即使是在同一个逻辑操作的上下文中）。这个会话可以是从现有连接池中选取的，也可以是即时打开的，但实际来说，所有会话级资源在两次调用之间都应被视为“丢失”，因为无法确保后续调用会命中与前一次调用相同的会话。

尽管如此，Oracle 提供了足够的额外选项，可以使用对象集合或 `XMLType`（后者更加灵活）来克服这个限制。当然，这些方法都需要对查询进行一些不平凡的改动（如下面的示例所示），但结果是一个 100%抽象的查询构建器。

```sql
CREATE FUNCTION f_getRefCursor_ref
        (i_type_tx VARCHAR2:='EMP',
        i_param_xml XMLTYPE:= XMLTYPE(
                '<param col1_tx="DEPTNO" value1_tx="20" col2_tx="ENAME" value2_tx="KING"/>')
        )
RETURN SYS_REFCURSOR
IS
    v_out_ref SYS_REFCURSOR;
    v_sql_tx VARCHAR2(32767);
BEGIN
    IF i_type_tx = 'EMP' THEN
        SELECT              
            'WITH param AS  ('||
            '    SELECT   TO_NUMBER(EXTRACTVALUE (in_xml, ''/param/@value1_tx'') value1, '||
            '                       EXTRACTVALUE (in_xml, ''/param/@value2_tx'') value2 '||
            '    FROM (SELECT :1 in_xml FROM DUAL) '||
            '                             ) '||
            ' SELECT count(*)'||
            ' FROM scott.emp, '||
            '             param '||
            ' WHERE emp.'|| dbms_assert.simple_sql_name(
                                    EXTRACTVALUE (i_param_xml, '/param/@col1_tx')
                                            )||'=param.value1 '||
            'OR emp.'|| dbms_assert.simple_sql_name(
                                   EXTRACTVALUE (i_param_xml, '/param/@col2_tx')
                                    )||'=param.value2'
        INTO v_sql_tx  FROM DUAL;

    ELSIF i_type_tx = 'B' THEN
        v_sql_tx:='<queryB>';  
  ...
    END IF;
OPEN v_out_ref FOR v_sql_tx USING i_param_xml;
    RETURN v_out_ref;
END;
```

在此示例中，单个 `XML` 变量的参数代表一组对，其中在每组中，第一个参数定义应过滤哪些列（`EMP.DEPTNO` 和 `EMP.ENAME`），第二个参数设置此过滤器的值（`20` 和 `KING`）。列（`COL1_TX` 和 `COL2_TX`）在查询构建时进行评估，因为它们是结构元素，而值则作为 `XMLType` 的单个输入参数传递到创建的查询中。尽管最终的语法看起来有些复杂，但上面的示例证明，将所有与数据库相关的系统元素放在它们所属的位置（即数据库中）是可能的。

## 动态游标示例 #2

如果我们接受 `EXECUTE IMMEDIATE` 的主要限制，那么动态游标的另一个有效利用案例就变得不言自明了。它是一个单一的 `PARSE/EXECUTE/FETCH` 事件序列，无法停止或暂停。因此，在通过 `BULK COLLECT` 进行多行提取的情况下，总是存在尝试将过多元素加载到输出集合中的严重风险。当然，可以通过添加 `WHERE ROWNUM <=:1` 来缓解此风险，如初始示例所示。但这个选项也不完美，因为它不允许从同一个源继续读取。动态游标可以解决这个问题。

对于此示例，假设你需要编写一个模块，该模块将从表中读取前 N 个值，如果未到达字母表中间则停止，否则继续直到末尾。解决方案如下所示：

```sql
CREATE FUNCTION f_getlov_tt  (
        i_table_tx    VARCHAR2,
        i_id_tx          VARCHAR2,
        i_display_tx VARCHAR2,
        i_order_nr   VARCHAR2,
        i_limit_nr     NUMBER:=50)
RETURN lov_tt
IS
    v_out1_tt lov_tt := lov_tt();
    v_out2_tt lov_tt := lov_tt();
    v_sql_tx VARCHAR2(32767);
    v_cur SYS_REFCURSOR;
BEGIN
    v_sql_tx:='SELECT lov_t(' ||
                               dbms_assert.simple_sql_name(i_id_tx)||','||
                  dbms_assert.simple_sql_name(i_display_tx)||')'||

      ' FROM '||dbms_assert.simple_sql_name(i_table_tx)||
      ' ORDER BY '||dbms_assert.simple_sql_name(i_order_nr);

    OPEN v_cur FOR v_sql_tx;
    FETCH v_cur BULK COLLECT INTO v_out1_tt LIMIT i_limit_nr;
    IF v_out1_tt.count=i_limit_nr AND UPPER(v_out1_tt(i_limit_nr).display_tx)>'N' then
           FETCH v_cur BULK COLLECT INTO v_out2_tt;
           SELECT v_out1_tt MULTISET UNION v_out2_tt INTO v_out1_tt FROM DUAL;
    END IF;
    CLOSE v_cur;

    RETURN v_out1_tt;
END;
```

此实现允许程序获取第一组记录（`V_OUT1_TT`），然后暂停，并在从已打开的游标提取数据的过程中应用逻辑规则：只有当定义的规则成功时，才会填充第二组记录（`V_OUT2_TT`），并且它只需通过继续提取即可从同一查询中填充，无需额外成本。这种“停走式”正是使此示例有价值的原因。


