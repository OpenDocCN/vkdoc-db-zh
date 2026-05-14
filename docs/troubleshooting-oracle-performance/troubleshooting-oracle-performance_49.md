# PL/SQL

PL/SQL 提供了不同的方法来执行 SQL 语句。主要分为**静态 SQL** 和**动态 SQL** 两大类。动态 SQL 又可以进一步分为三个子类：`EXECUTE IMMEDIATE`、`OPEN`/`FETCH`/`CLOSE` 以及 `dbms_sql` 包。所有这些方法中，唯一与解析相关的可用特性是使用绑定变量。实际上，语句重用和客户端语句缓存仅部分可用。它们并未在所有方法中实现。接下来的章节将描述这四类方法各自的特性。

## 注意

由于 PL/SQL 在数据库引擎中运行，谈论客户端语句缓存可能看起来有些奇怪。然而，从 SQL 引擎的角度来看，PL/SQL 是一个客户端。在该客户端中，先前讨论的客户端语句缓存概念已经实现。

本节提供的 PL/SQL 块示例摘自脚本 `ParsingTest1.sql`、`ParsingTest2.sql` 和 `ParsingTest3.sql`，这些脚本分别实现了测试用例 1、2 和 3。

## 静态 SQL

静态 SQL 集成在 PL/SQL 语言中。顾名思义，它是静态的，因此 SQL 语句必须在编译时完全已知。为此，如果 SQL 语句有参数，使用绑定变量是不可避免的。例如，使用静态 SQL 不可能编写一个重现测试用例 1 的代码片段。

你可以用两种方式编写静态 SQL。第一种基于隐式游标，因此无法控制游标生命周期。以下 PL/SQL 块展示了一个实现测试用例 2 的示例：

```sql
DECLARE
  l_pad VARCHAR2(4000);
BEGIN
  FOR i IN 1..10000
  LOOP
    SELECT pad INTO l_pad
    FROM t
    WHERE val = i;
  END LOOP;
END;
```

第二种方式基于显式游标。在这种情况下，可以对游标进行一些控制。然而，打开/解析/执行阶段被合并为单一操作（`OPEN`）。这意味着只有取数阶段可以被控制。以下 PL/SQL 块也展示了实现测试用例 2 的示例：

```sql
DECLARE
  CURSOR c (p_val NUMBER) IS SELECT pad FROM t WHERE val = p_val;
  l_pad VARCHAR2(4000);
BEGIN
  FOR i IN 1..10000
  LOOP
    OPEN c(i);
    FETCH c INTO l_pad;
    CLOSE c;
  END LOOP;
END;
```

从性能角度来看，这两种方法相似。虽然它们都防止了编写糟糕的代码（测试用例 1），但不允许编写非常高效的代码（测试用例 3）。这是因为无法完全控制游标。

为了解决这个问题，客户端游标缓存是可用的。在 Oracle9*i* 版本 9.2.0.4 之前，它始终启用，因为缓存游标的最大数量由初始化参数 `open_cursors` 决定。因此，每个可以打开的游标也可以被缓存。从 Oracle9*i* 版本 9.2.0.5 开始，缓存游标的数量由初始化参数 `session_cached_cursors` 决定。直到 Oracle Database 10*g* Release 1，该参数的默认值都是 0。但是，如果没有显式设置为 0，则默认的缓存游标数是 50。请注意，在这两种情况下，一个与客户端语句缓存无直接关系的初始化参数被“误用”来配置它！

## 原生动态 SQL：EXECUTE IMMEDIATE

从游标管理的角度来看，基于 `EXECUTE IMMEDIATE` 的原生动态 SQL 与使用隐式游标的静态 SQL 类似。换句话说，无法控制游标的生命周期。以下 PL/SQL 块展示了一个实现测试用例 2 的示例：

```sql
DECLARE
  l_pad VARCHAR2(4000);
BEGIN
  FOR i IN 1..10000
  LOOP
    EXECUTE IMMEDIATE 'SELECT pad FROM t WHERE val = :1' INTO l_pad USING i;
  END LOOP;
END;
```

由于无法控制游标，因此无法编写实现测试用例 3 的代码。为此，像静态 SQL 一样，使用了客户端游标缓存。然而，它仅从 Oracle Database 10*g* Release 1 开始可用。

## 原生动态 SQL：OPEN/FETCH/CLOSE

从游标管理的角度来看，基于 `OPEN`/`FETCH`/`CLOSE` 的原生动态 SQL 与使用显式游标的静态 SQL 类似。换句话说，只能控制取数阶段。以下 PL/SQL 块展示了一个实现测试用例 2 的示例：

```sql
DECLARE
  TYPE t_cursor IS REF CURSOR;
  l_cursor t_cursor;
  l_pad VARCHAR2(4000);
BEGIN
  FOR i IN 1..10000
  LOOP
    OPEN l_cursor FOR 'SELECT pad FROM t WHERE val = :1' USING i;
    FETCH l_cursor INTO l_pad;
    CLOSE l_cursor;
  END LOOP;
END;
```

由于无法完全控制游标，因此无法编写实现测试用例 3 的代码。此外，数据库引擎无法利用基于 `OPEN`/`FETCH`/`CLOSE` 的原生动态 SQL 进行客户端语句缓存。这意味着解决由使用此方法的代码引起的解析问题的唯一方法是用 `EXECUTE IMMEDIATE` 或 `dbms_sql` 包重写它。作为变通方案，你还应该考虑服务器端语句缓存。

## 动态 SQL：dbms_sql 包

`dbms_sql` 包提供了对游标生命周期的完全控制。在以下 PL/SQL 块（实现测试用例 2）中，请注意每个步骤是如何显式编码的：

```sql
DECLARE
  l_cursor INTEGER;
  l_pad VARCHAR2(4000);
  l_retval INTEGER;
BEGIN
  FOR i IN 1..10000
  LOOP
    l_cursor := dbms_sql.open_cursor;
    dbms_sql.parse(l_cursor, 'SELECT pad FROM t WHERE val = :1', 1);
    dbms_sql.define_column(l_cursor, 1, l_pad, 10);
    dbms_sql.bind_variable(l_cursor, ':1', i);
    l_retval := dbms_sql.execute(l_cursor);
    IF dbms_sql.fetch_rows(l_cursor) > 0
    THEN
      NULL;
    END IF;
    dbms_sql.close_cursor(l_cursor);
  END LOOP;
END;
```

由于可以完全控制游标，因此实现测试用例 3 没有问题。以下 PL/SQL 块展示了一个示例。请注意，准备游标（`open_cursor`、`parse` 和 `define_column`）和关闭游标（`close_cursor`）的过程被放置在循环之外，以避免不必要的软解析。

```sql
DECLARE
  l_cursor INTEGER;
  l_pad VARCHAR2(4000);
  l_retval INTEGER;
BEGIN
  l_cursor := dbms_sql.open_cursor;
  dbms_sql.parse(l_cursor, 'SELECT pad FROM t WHERE val = :1', 1);
  dbms_sql.define_column(l_cursor, 1, l_pad, 10);
  FOR i IN 1..10000
  LOOP
    dbms_sql.bind_variable(l_cursor, ':1', i);
    l_retval := dbms_sql.execute(l_cursor);
    IF dbms_sql.fetch_rows(l_cursor) > 0
    THEN
      NULL;
    END IF;
  END LOOP;
  dbms_sql.close_cursor(l_cursor);
END;
```

数据库引擎无法利用 `dbms_sql` 包进行客户端语句缓存。因此，为了优化一个因过多软解析（如测试用例 2）而性能不佳的应用程序，你必须修改它以重用游标（如测试用例 3 所做）。作为变通方案，你可以考虑服务器端语句缓存。



