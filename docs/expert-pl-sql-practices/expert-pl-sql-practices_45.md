# PL/SQL 标识符命名规范与作用域捕获

## chk_ident 过程

```plsql
------------------------------------------------------------------------------
procedure chk_ident(
  i_ident          varchar2
 ,i_loc            varchar2
)
is
  l_start_pos      pls_integer := 1;
  l_end_pos        pls_integer := 1;
  l_abbr_cnt       pls_integer;
  l_part           varchar2(30);
  c_ident_len constant pls_integer := length(i_ident);
begin
  while l_start_pos < c_ident_len loop
    -- 确定下一个部分 --
    while     l_end_pos <= c_ident_len
          and substr(i_ident, l_end_pos, 1) not in ('_', '#', '$')
    loop
      l_end_pos := l_end_pos + 1;
    end loop;
    l_part := upper(substr(i_ident, l_start_pos, l_end_pos - l_start_pos));

    -- 检查该部分是否为已注册的缩写 --
    select count(*)
    into   l_abbr_cnt
    from   abbr_reg
    where  abbr = l_part;

    if l_abbr_cnt = 0 then
      dbms_output.put_line('标识符 ' || i_ident || ' 中存在未注册的部分 ' || l_part
                           || '，位置：' || i_loc || '。');
    end if;

    -- 初始化下一次循环的变量 --
    l_end_pos := l_end_pos + 1;
    l_start_pos := l_end_pos;
  end loop;
end chk_ident;

------------------------------------------------------------------------------
procedure chk_abbr
is
begin
  -- 使用 PL/Scope 的 PL/SQL --
  for c in (
    select name
          ,object_type
          ,object_name
          ,line
    from   user_identifiers
    where  usage = 'DECLARATION'
    order by object_name, object_type, line
  ) loop
    chk_ident(
      i_ident => c.name
     ,i_loc   => c.object_type || ' ' || c.object_name || '，行 ' || c.line
    );
  end loop;
  -- 其他项目：USER_OBJECTS, USER_TAB_COLUMNS 等 --
  -- ...
end chk_abbr;

end abbr_reg#;
```

由于构建这个检查器只需要大约一百行 PL/SQL 代码，因此没有理由不去构建一个。PL/Scope 使得这个过程比尝试从 `user_source` 或你的源代码存储库中解析源代码要容易得多。

![images](img/square.jpg) **注意** Oracle 在数据字典中对缩写的使用并不一致。例如，“index”这个术语在视图 `user_indexes` 和 `user_ind_columns` 以及 `user_policies` 的列 `idx` 中被完整拼写了一次，却被以两种不同的方式缩写。Oracle 不修复这个问题大概是因为公共 API 的向后一致性比缩写的一致性更重要。

### PL/SQL 标识符的前后缀

许多 PL/SQL 开发人员会给标识符添加前缀，以指示其作用域或类型。例如，局部变量以 `l_` 为前缀，常量以 `c_` 为前缀，输入参数以 `i_` 为前缀，类型以 `t_` 为前缀。为每个 PL/SQL 标识符添加前缀或后缀有两个很好的理由。两者都与避免作用域捕获有关。

PL/SQL 会自动将静态 SQL 语句中的 PL/SQL 变量转换为绑定变量。考虑以下函数，它应该从 Oracle 的 `demobld.sql` 脚本创建的 SCOTT 模式的 `emp` 表中返回指定员工编号的员工姓名。在这个例子中，我没有对参数 `empno` 和局部变量 `ename` 使用任何前缀。

```plsql
create or replace function emp#ename(empno emp.empno%type)
return emp.ename%type
is
  ename            emp.ename%type;
begin
  select ename
  into   ename
  from   emp
  where  empno = empno;

  return ename;
end emp#ename;
```

这个函数并没有实现我想要的功能。`where` 子句中 `empno` 的两个出现都指的是表列 `empno`，而不是输入参数，因为每个标识符都被解析为最局部的声明。在这种情况下，最局部的作用域是包含表 `emp` 的 SQL 语句。因此，`where` 条件等价于 `emp.empno = emp.empno`，这与 `emp.empno is not null` 相同。除非表 `emp` 中恰好有一条记录，否则该函数将抛出一个异常。

```sql
SQL> truncate table emp;
表已截断。
SQL> insert into emp(empno, ename) values (7369, 'SMITH');
已创建 1 行。
```

当表中只有一行时，该函数会返回该行的姓名，而不管实际的参数如何。我请求 `empno` 为 21 的员工姓名，却得到了 `empno` 为 7369 的员工的姓名。

```sql
SQL> select emp#ename(21) from dual;
EMP#ENAME(21)
-----------------------------------------------------------------
SMITH
已选择 1 行。
```

当表中有两行或更多行时，该函数总是返回 ORA-01422 错误。

```sql
SQL> insert into emp(empno, ename) values (7499, 'ALLEN');
已创建 1 行。
SQL> select emp#ename(7369) from dual;
select emp#ename(7369) from dual
       *
第 1 行出现错误:
ORA-01422: 实际提取的行数超出请求的行数
ORA-06512: 在 "K.EMP#ENAME", 第 5 行
```

你可以通过为输入参数 `empno` 添加前缀或后缀（例如 `i_empno`）来避免作用域捕获。让我们将此规则推广为：每个 PL/SQL 标识符都应该有一个在列名中未使用的前缀或后缀。前缀或后缀的最小长度是两个字节——即一个字母和一个作为分隔符的下划线。你可以使用这个前缀来传达额外的语义，例如变量的作用域或参数的模式，而无需浪费宝贵的三十字节限制中的另一个字节。

另一种替代前缀的方法是，在 SQL 语句中用声明块的名字来限定所有 PL/SQL 变量，例如 `empno = emp#ename.empno`。这种方法的优点是，当表中添加了列时，它可以防止不必要的失效，因为编译器知道不会发生作用域捕获。Bryn Llewellyn 在 `http://bit.ly/dSMfto` 中描述了 Oracle 11g 细粒度依赖跟踪的这个方面。我不使用这种方法，因为基于版本的重定义是防止在线升级期间失效的更好解决方案，而且我觉得语法很笨拙。

当然，为每个局部变量添加 `l_` 前缀并不能避免嵌套 PL/SQL 块中的作用域捕获。对所有嵌套块中的所有 PL/SQL 标识符使用完全限定符号可以解决这个问题，但代价是冗长。



## 避免 PL/SQL 标识符与关键字冲突

在 PL/SQL 标识符中包含前缀或后缀的第二个原因，是为了避免与 SQL 和 PL/SQL 中大量的关键字以及在`standard`和`dbms_standard`中声明的内置函数产生混淆。截至 Oracle 11gR2，有 1,844 个 SQL 关键字，例如`table`，以及内置函数的名称，例如`upper`，这些都列在`v$reserved_words`中。

```sql
SQL> select count(distinct keyword)
  2  from   v$reserved_words;

COUNT(DISTINCTKEYWORD)
----------------------
                  1844

1 row selected.
```

PL/SQL 关键字列在《PL/SQL 语言参考》的附录 D 中，但无法通过视图查看。如果不小心选择自己的标识符，就会遮蔽（shadow）Oracle 的实现。在下面的例子中，`upper`的实现（它返回小写参数而不是 Oracle 提供的同名标准函数）被执行了：

```sql
create or replace procedure test
  authid definer
is
  function upper(i_text varchar2) return varchar2
  is
  begin
    return lower(i_text);
  end upper;
begin
  dbms_output.put_line(upper('Hello, world!'));
end test;

SQL> exec test
hello, world!

PL/SQL procedure successfully completed.
```

这个小例子中显得有些人为的情况，在实际中很可能发生在一个由非原作者维护的大型包中。例如，当应用程序还在 10g 下运行时，原作者可能实现了一个函数`regexp_count`。在升级到 11gR2 后，新的维护者可能在包的某个其他地方添加了对`regexp_count`的调用，期望调用同名（但语义可能不同）的新增 SQL 内置函数。

Oracle 提供了警告来防止滥用关键字。不幸的是，警告通常处于禁用状态。如果你启用所有警告并重新编译测试过程，你会得到期望的错误或警告，这取决于你将`plsql_warnings`设置为`error:all`还是`enable:all`。

```sql
SQL> alter session set plsql_warnings='error:all';

Session altered.

SQL> alter procedure test compile;

Warning: Procedure altered with compilation errors.

SQL> show error
Errors for PROCEDURE TEST:

LINE/COL ERROR
-------- -------------------------------------------------------------------------------
4/12      PLS-05004: identifier UPPER is also declared in STANDARD or is a SQL built-in
```

总之，为每个 PL/SQL 标识符添加前缀或后缀可以解决作用域捕获（scope capture）的问题。我使用变量和参数的前缀来避免静态 SQL 中的作用域捕获，但我不费心为函数或过程添加前缀或后缀，因为与关键字的冲突很少见，而且带有参数的 SQL 函数不会发生作用域捕获。

为了区分列表及其元素，通常为列表（如关联数组）添加复数后缀。或者，使用后缀来指示类型，例如`_rec`表示记录，`_tab`表示关联数组类型，以此来区分元素和列表，同时以几个字符为代价传达额外的语义信息。我在本章中使用类型后缀表示法是为了清晰，因为我将比较使用不同类型的实现。我使用井号（#）作为包的后缀，以避免与同名表发生名称冲突。

**注意**：前缀和后缀的正确使用也可以通过 PL/Scope 轻松检查。Lucas Jellema 在 [`http://technology.amis.nl/blog/?p=2584`](http://technology.amis.nl/blog/?p=2584) 上有一个这方面的例子。

### 代码与数据的模块化

正确的模块化是开发团队可扩展性和应用程序可维护性的基础。模块化将“分而治之”的益处带入了软件工程。模块化的关键方面如下：

*   *可分解性*：必须能够将每个复杂问题分解为少量不太复杂的子问题，这些子问题可以分开处理。
*   *模块可理解性*：在一个大型应用程序中，必须能够独立理解任何部分，而无需了解其余应用程序的太多或全部信息。这种特性称为模块可理解性。“大规模编程”通常意味着一个人加入公司时，需要处理一个已有数百万行代码的应用程序。显然，只有应用程序满足模块可理解性，才能以高效的方式完成这项工作。
*   *模块连续性*：连续性有两个方面。首先，对规范的小改动必须仅导致一个或几个模块的改动。其次，模块中的改动必须易于证明不会在其他模块中引起回归。模块连续性对于临时补丁尤其重要，由于其频率和紧迫性，可能无法像主要版本那样经过充分测试。
*   *可重用性*：应用程序某部分问题的解决方案必须能在另一部分重用。更通用的可组合性目标（即用现有组件构建新的、可能非常不同的系统）通常只对基础框架有要求。

**注意**：可分解性和可组合性通常是相互冲突的目标。可分解性是通过自顶向下设计实现的。它导致特定的模块，这些模块通常不适合通用的组合。另一方面，可组合性基于自底向上设计，这导致通用设计，但这些设计对于特殊情况通常效率低下且成本过高（除非一个模块可以被重用多次，从而证明大的投入是合理的）。

通过遵守以下规则，可以用模块实现模块化的这些方面：

*   *信息隐藏（抽象）*：每个模块都明确地将其公共接口与其实现细节分离。这可以比作一台电视机：它的接口是带有几个按钮的遥控器，它的实现由复杂的电路和软件组成。电视观众不需要理解电视的实现。
*   *小接口*：接口尽可能小，以不限制实现的未来改进。
*   *少量接口和分层*：每个模块尽可能少地使用来自其他模块的接口，以减少接口需要以不兼容方式更改时的连带损害，并普遍提高模块连续性。大多数架构是分层的，其中较高层（如业务逻辑）的模块可以调用较低层（如数据访问）的模块，反之则不行。
*   *直接映射*：每个模块代表一个专门的业务概念，并包含执行单个任务及其相关数据和操作，并力求做好该任务。

这些通用需求和设计原则如何转化为 PL/SQL？模块在不同的粒度级别上映射到子程序、包、模式（schema）或它们中任何一项的集合。作为抽象单元的子程序在 PL/SQL 和大多数其他过程语言中是相同的，因此这里不作更详细的讨论。

我将讨论限制在技术模块化上，并忽略源代码版本控制、部署、定制、营销和许可方面的模块化，技术模块化通常是这些方面的前提条件。我从包开始，然后描述如何实现包含模式和不包含模式的更大模块。



