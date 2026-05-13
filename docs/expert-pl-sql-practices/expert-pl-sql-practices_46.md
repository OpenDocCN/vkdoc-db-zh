# 作为模块的包及相关表

PL/SQL 明确地提供了包，可将其用作满足前述要求的模块。通过将包规范与其实施分离，支持信息隐藏。为确保客户端只需查阅规范，而不依赖于可能改变的实施细节，必须对包规范进行恰当的文档编写。对整个包以及每个接口元素（如子程序和类型）的语义和预期用途进行简要描述便已足够。目前没有类似于 `JavaDoc` 的标准 HTML API 生成工具可用于 `PL/SQL`，这不能成为不为包规范编写文档的借口。用户不会介意阅读包规范中的文档。事实上，在一项非正式调查中，大多数 `Java` 开发者告诉我，他们更倾向于查看 `Java` 源码中的 `JavaDoc`，而不是生成的 `HTML`。

信息隐藏甚至可以通过字面意义上地封装主体而非规范来实现。Oracle 对大多数公共 API 都采用此做法，例如 `dbms_sql`。不过要注意，封装的代码可以被反封装，在此过程中只会丢失注释。

包规范中暴露的每个元素都可以被同一模式中的任何其他程序使用，也可以被另一个已获得该包 `execute` 权限的模式中的程序使用。`PL/SQL` 不支持像 `Java` 那样的受保护导出，也不支持像 `Oberon-2` 那样的变量只读导出。这没有必要：可以通过不同的包来实现针对不同类型客户端的 API。

包还通过分离的、类型安全的编译来支持高效开发。客户端可以针对包规范进行编译，并且仅当包主体被修改时，它们也永远不会失效。随着 `11g` 中引入细粒度依赖跟踪，只有当包规范发生相关变更时，客户端才会失效。这种方法的缺点是，由于没有即时编译器（`PL/SQL` 没有），无法进行跨单元内联（除了独立过程，而 `PL/SQL` 目前也不支持独立过程）。

![images](img/square.jpg) **注意** 为保持接口精简，生产代码中的大多数过程应只在包主体中声明，而不通过规范导出。为了测试它们，可以使用条件编译在测试构建中将它们导出，正如 Bryn Llewellyn 在 `http://bit.ly/eXxJ9Q` 中所描述的那样。

要将模块化从 `PL/SQL` 代码扩展到表数据，每个表都应仅由单个包修改。同样，所有仅引用单个表的 `select` 语句都应包含在此单一包中。另一方面，引用与不同包相关联的多个表的 `select` 语句则允许打破这种一对一映射。可以使用视图来引入额外的抽象层。然而，为与不同包关联的两个表之间的每个连接都引入一个视图通常是不切实际的。

有多种方法可以检测违反前述表访问规则的情况。此处描述的所有方法都必须视为在测试期间发现错误的软件工程工具，而不是执行安全性的手段。除了触发器和细粒度审计方法外，都需要传递闭包（例如使用层次查询）来通过视图深入到底层表。

我将在一个章节中以摘要形式描述两种编译时方法，并在单独的小节下详细描述三种运行时方法。运行时方法更复杂，并且包含超出特定案例的令人感兴趣的技术。运行时方法的主要问题是它们需要能够触发所有相关 `SQL` 语句执行的测试用例。

### 通过静态分析检测表访问

有两种基于静态分析的编译时方法：

*   在源文本中搜索：如果工具良好，此方法通常出奇地快，除非存在大量视图，这些视图的出现也需要被搜索。然而，此方法需要 `PL/SQL` 解析器来实现自动化。
*   依赖关系：视图 `user_dependencies` 仅列出静态依赖关系，不区分 `DML` 和只读访问。`PL/Scope` 不包含关于 `SQL` 语句的信息。

### 通过探测共享池检测表访问

视图 `v$sql` 包含仍在共享池中最近执行的 `SQL`。`program_id` 列引用导致硬解析的 `PL/SQL` 单元。例如，以下语句显示所有对上述表 `abbr_reg` 发出过 `DML` 的 `PL/SQL` 源码。要同时看到 `select` 语句，只需移除对 `command_type` 的条件即可。

```sql
SQL> select ob.object_name
  2        ,sq.program_line#
  3        ,sq.sql_text
  4  from   v$sql       sq
  5        ,all_objects ob
  6  where  sq.command_type in (2 /*insert*/, 6 /*update*/, 7 /*delete*/, 189 /*merge*/)
  7     and sq.program_id = ob.object_id (+)
  8     and upper(sq.sql_fulltext) like '%ABBR_REG%';
```

```
OBJECT_NAME          PROGRAM_LINE# SQL_TEXT
-------------------- ------------- ------------------------------------------------------
ABBR_REG#                       10 INSERT INTO ABBR_REG(ABBR, TEXT, DESCN) VALUES(UPPER(:

1 row selected.
```

共享池探测方法有几个缺点。

*   只返回导致硬解析的 `PL/SQL` 单元。如果相同的 `SQL` 出现在多个单元中（尽管在手写代码中不应出现），你将找不到其他的。如果硬解析是由匿名块触发的，你将得不到任何相关信息。
*   你必须在 `SQL` 从共享池中刷出之前捕获它。关于 `SQL` 的 `Statspack`、`ASH` 和 `AWR` 视图不包含 `program_id` 列。
*   你需要使用近似的字符串匹配，这可能返回过多数据，因为 `DML` 的目标表仅在 `SQL` 文本中可见。来自 `select` 子句的表可以通过连接 `v$sql_plan` 来精确匹配。或者，你可以动态创建包含 `SQL` 文本的包装过程，从 `all_dependencies` 获取引用，然后再次删除包装过程。
*   `Truncate` 在 `SQL` 文本中被列为通用的 `lock table`。

### 使用触发器检测表访问

触发器是记录或阻止访问的另一种选择。以下触发器检查对表 `abbr_reg` 的所有 `DML` 操作是否来自包 `abbr_reg#` 或其调用的子程序：

```sql
create or replace trigger abbr_reg#b
before update or insert or delete or merge
on abbr_reg
begin
  if dbms_utility.format_call_stack not like
       '%package body% K.ABBR_REG#' || chr(10) /*UNIX EOL*/|| '%' then
    raise_application_error(-20999,'Table abbr_reg may only be modified by abbr_reg#.');
  end if;
end;
```

正如预期的那样，来自 `abbr_reg#` 的 `DML` 被允许，但直接的 `DML` 或来自另一个包的 `DML` 则不允许。

```sql
SQL> exec abbr_reg#.ins_abbr('descn', 'description')

PL/SQL procedure successfully completed.

SQL> insert into abbr_reg(abbr, text) values('reg', 'registry');
insert into abbr_reg(abbr, text) values('reg', 'registry')
            *
ERROR at line 1:
ORA-20999: Table abbr_reg may only be modified by abbr_reg#.
ORA-06512: at "K.ABBR_REG#B", line 3
ORA-04088: error during execution of trigger 'K.ABBR_REG#B'
```

代替调用开销大的 `dbms_utility.format_call_stack`，你可以在 `abbr_reg#` 中使用一个包主体全局变量，在访问表 `abbr_reg` 之前将该变量设置为 `true`，之后设置为 `false`，并在触发器中调用 `abbr_reg#` 中的一个过程来检查该变量是否为 `true`。



