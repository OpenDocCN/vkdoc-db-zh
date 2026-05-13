# 第一部分 基本概念

![image](img/frontdot.jpg)

## 第 1 章 SQL 特性

本章讨论一系列对调优过程重要且相对独立的 SQL 特性，其中许多特性宣传得不太好。我将首先快速回顾什么是 SQL 语句以及用于引用它们的标识符。我的第二个主题是数组接口，它用于将大批量数据从客户端进程移动到数据库服务器。然后我将讨论因子子查询，它使阅读 SQL 语句变得容易得多。本章的第四个也是最后一个主题是回顾不同类型的内连接和外连接；我将解释如何编写它们、它们的用途，以及为什么重新排序外连接不如重新排序内连接那么容易。

### SQL 与声明式编程语言

用*声明式编程语言*编写的程序描述应该执行什么计算，但不描述如何计算。SQL 被认为是一种声明式编程语言。将 SQL 与*命令式编程语言*（如 C、Visual Basic 甚至 PL/SQL）进行比较，后者指定了计算的每个步骤。

这听起来是个好消息。你可以按自己喜欢的方式编写 SQL，只要语义正确，其他人或东西就会找到运行它的最优方式。在我们的情况下，那个“东西”是*基于成本的优化器*（CBO），在大多数情况下它做得相当不错。然而，尽管有理论支持，许多 SQL 语句中强烈暗示着一种算法。清单 1-1 使用`HR`示例模式就是这样一个例子。

**清单 1-1. SELECT 列表中的子查询**

```sql
SELECT first_name
        ,last_name
        , (SELECT first_name
             FROM hr.employees m
            WHERE m.employee_id = e.manager_id)
            AS manager_first_name
        , (SELECT last_name
             FROM hr.employees m
            WHERE m.employee_id = e.manager_id)
            AS manager_last_name
    FROM hr.employees e
   WHERE manager_id IS NOT NULL
ORDER BY last_name, first_name;
```

这个语句的意思是：*获取每位有经理的员工的名字和姓氏，并在每种情况下查找经理的名字和姓氏。按员工的姓氏和名字对结果行进行排序*。清单 1-2 看起来是一个完全不同的语句。

**清单 1-2. 使用连接代替 SELECT 列表**

```sql
 SELECT e.first_name
        ,e.last_name
        ,m.first_name AS manager_first_name
        ,m.last_name AS manager_last_name
    FROM hr.employees e, hr.employees m
   WHERE m.employee_id = e.manager_id
ORDER BY last_name, first_name;
```

这个语句说：*在`HR.EMPLOYEES`上执行自连接，只保留第一份副本的`EMPLOYEE_ID`与第二份副本的`MANAGER_ID`匹配的行。选择员工和经理的姓名并对结果排序*。尽管清单 1-1 和清单 1-2 之间存在明显差异，但它们产生相同的结果。实际上，因为`EMPLOYEE_ID`是`EMPLOYEES`的主键，并且存在从`MANAGER_ID`到`EMPLOYEE_ID`的引用完整性约束，所以它们在语义上是等效的。

在理想情况下，CBO 会解决所有问题，并以相同的方式执行这两个语句。事实上，截至 Oracle Database 12c，这些语句的执行方式完全不同。尽管 CBO 在每个版本中都在改进，但 SQL 语句的作者始终有责任以某种方式编写它们，以帮助 CBO 找到执行性能良好的执行计划，或者至少避免一个完全糟糕的计划。

### 语句与 SQL_ID

Oracle 数据库使用称为 SQL_ID 的东西来标识每个 SQL 语句。你在分析 SQL 性能时使用的许多视图，例如`V$ACTIVE_SESSION_HISTORY`，都与由 SQL_ID 标识的特定 SQL 语句相关。理解这些 SQL_ID 是什么，以及如何将 SQL_ID 与 SQL 语句的实际文本进行交叉引用，这一点很重要。

SQL_ID 是一个以字符串形式表示的 32 进制数，由 13 个字符组成，每个字符可以是数字或 22 个小写字母中的一个。例子可能是‘ddzxfryd0uq9t’。字母 e、i、l 和 o 不被使用，大概是为了限制转录错误的风险。SQL_ID 实际上是从 SQL 语句中的字符生成的哈希值。因此，假设大小写和空白被保留，相同的 SQL 语句在任何使用它的数据库上都将具有相同的 SQL_ID。

通常，清单 1-3 中的两个语句会被认为是不同的。

**清单 1-3. 涉及字面量的语句**

```sql
SELECT 'LITERAL 1' FROM DUAL;
SELECT 'LITERAL 2' FROM DUAL;
```

第一个语句的 SQL_ID 是‘3uzuap6svwz7u’，第二个是‘7ya3fww7bfn89’。

在 PL/SQL 块内部发出的任何 SQL 语句也有一个 SQL_ID。此类语句可能使用 PL/SQL 变量或参数，但更改变量的值不会改变 SQL_ID。清单 1-4 显示了一个类似于清单 1-3 的查询，但它是在 PL/SQL 块内部发出的。

**清单 1-4. 从 PL/SQL 发出的 SELECT 语句**

```sql
SET SERVEROUT ON

DECLARE
   PROCEDURE check_sql_id (p_literal VARCHAR2)
   IS
      dummy_variable   VARCHAR2 (100);
      sql_id           v$session.sql_id%TYPE;
   BEGIN
      SELECT p_literal INTO dummy_variable FROM DUAL;

      SELECT prev_sql_id
        INTO sql_id
        FROM v$session
       WHERE sid = SYS_CONTEXT ('USERENV', 'SID');

      DBMS_OUTPUT.put_line (sql_id);
   END check_sql_id;

BEGIN
   check_sql_id ('LITERAL 1');
   check_sql_id ('LITERAL 2');
END;
/
```

这个匿名块包括对嵌套过程的两次调用，该过程接受一个`VARCHAR2`字符串作为参数。该过程调用一个`SELECT`语句，然后从`V$SESSION`的`PREV_SQL_ID`列获取该语句的 SQL_ID 并输出它。该过程使用了与清单 1-3 中相同的两个字面量进行调用。然而，输出显示，在两种情况下都使用了相同的 SQL_ID，‘d8jhv8fcm27kd’。实际上，PL/SQL 在将`SELECT`语句提交给 SQL 引擎之前会稍微修改它。清单 1-5 显示了移除 PL/SQL 特有的`INTO`子句后的底层 SQL 语句。



清单 1-5。一个使用了绑定变量的 SQL 语句
```
SELECT :B1 FROM DUAL
```

其中的 `:B1` 部分就是所谓的 `绑定变量`，在 PL/SQL 中，每当使用变量或参数时就会用到它。从其他编程语言调用 SQL 时也会使用绑定变量。这个绑定变量只是一个实际值的占位符，它表明同一条语句可以重复使用，只需为占位符提供不同的值即可。我将在后文解释其重要性。

### 交叉引用语句与 SQL_ID

如果你有权访问运行 11.2 或更高版本数据库的 `SYS` 账户，则可以使用清单 1-6 中的方法来确定某条语句的 `SQL_ID`。

清单 1-6。使用 `DBMS_SQLTUNE_UTIL0` 确定语句的 `SQL_ID`
```
SELECT sys.dbms_sqltune_util0.sqltext_to_sqlid (
          q'[SELECT 'LITERAL 1' FROM DUAL]' || CHR (0))
  FROM DUAL;
```

清单 1-6 中查询的结果是 ‘3uzuap6svwz7u’，即清单 1-3 中第一条语句的 `SQL_ID`。关于清单 1-6，可以得出几点观察：
*   注意包含单引号的字符串本身是如何被引用的。该语法在 SQL 语言参考手册中有完整记录，非常有用，但许多有经验的 Oracle 专家却常常忽略它。
*   在调用函数之前，需要在文本末尾附加一个 NUL 字符。
*   你不需要访问当前工作数据库上的 `SYS` 账户来使用此函数。我经常远程工作，可以将一条语句放到我笔记本电脑上的 11.2 数据库中来获取 `SQL_ID`；请记住，`SQL_ID` 在所有数据库上都是相同的，与数据库版本无关！

这不是交叉引用 SQL 语句文本和 `SQL_ID` 的常规方法。我已经解释过如何使用 `V$SESSION` 的 `PREV_SQL_ID` 列来确定会话先前执行的 SQL 语句的 `SQL_ID`。正如你所想，`SQL_ID` 列与当前正在执行的语句相关。然而，最常用的确定语句 `SQL_ID` 的方法是查询 `V$SQL` 或 `DBA_HIST_SQLTEXT`。

`V$SQL` 包含有关当前正在运行或最近已完成的语句的信息。`V$SQL` 包含以下三列（以及其他列）：
*   `SQL_ID` 是语句的 `SQL_ID`。
*   `SQL_FULLTEXT` 是一个 `CLOB` 列，包含 SQL 语句的文本。
*   `SQL_TEXT` 是一个 `VARCHAR2` 列，包含可能是 `SQL_FULLTEXT` 截断版本的文本。

如果你使用自动工作负载存储库 (AWR) 的数据进行分析，那么你的 SQL 语句很可能已经从游标缓存中消失，使用 `V$SQL` 进行查找将不起作用。在这种情况下，你需要使用 `DBA_HIST_SQLTEXT`（它本身就是一个 AWR 视图）来执行查找。此视图与 `V$SQL` 略有不同，因为其 `SQL_TEXT` 列是 `CLOB` 列，没有 `VARCHAR2` 变体。

使用 `V$SQL` 或 `DBA_HIST_SQLTEXT`，你可以提供一个 `SQL_ID` 来获取相应的 `SQL_TEXT`，反之亦然。清单 1-7 显示了两个查询，用于搜索包含 ‘LITERAL1’ 的语句。

清单 1-7。从 `V$SQL` 或 `DBA_HIST_SQLTEXT` 中识别 `SQL_ID`
```
SELECT *
  FROM v$sql
 WHERE sql_fulltext LIKE '%''LITERAL1''%';

SELECT *
  FROM dba_hist_sqltext
 WHERE sql_text LIKE '%''LITERAL1''%';
```

![image](img/sq.jpg) **注意** 使用视图 `V$ACTIVE_SESSION_HISTORY` 以及以 `DBA_HIST_` 开头的视图需要企业版数据库并启用了诊断包。

清单 1-7 中的两个查询将为每个包含字符 ‘LITERAL1’ 的语句返回一行。你要找的查询，如果仍在共享池中，则会在 `V$SQL` 中；如果已被 AWR 捕获，则会在 `DBA_HIST_SQLTEXT` 中。

### 数组接口

`数组接口` 允许为绑定变量提供一组值。从性能角度来看，这一点极其重要，因为没有它，在客户端运行的代码可能需要进行大量的网络往返才能将数据数组发送到数据库服务器。尽管它是 SQL 中非常重要的一部分，但许多程序员和数据库管理员 (DBA) 却没有意识到它。其不为人知的一个原因是它不能直接从 SQL*Plus 中使用。清单 1-8 设置了几个表来帮助解释这个概念。

清单 1-8。为测试创建表 `T1` 和 `T2`
```
CREATE TABLE t1
(
   n1   NUMBER
  ,n2   NUMBER
);

CREATE TABLE t2
(
   n1   NUMBER
  ,n2   NUMBER
);

INSERT INTO t1
   SELECT object_id, data_object_id
     FROM all_objects
    WHERE ROWNUM <= 30;
```

清单 1-8 创建了表 `T1` 和 `T2`，每个表包含两列数值列：`N1` 和 `N2`。表 `T1` 已填充了 30 行数据，`T2` 是空的。你需要使用像 PL/SQL 这样的语言来演示数组接口，清单 1-9 包含两个示例。

清单 1-9。在 `DELETE` 和 `MERGE` 中使用数组接口
```
DECLARE
   TYPE char_table_type IS TABLE OF t1.n1%TYPE;

   n1_array   char_table_type;
   n2_array   char_table_type;
BEGIN
   DELETE FROM t1
     RETURNING n1, n2
          BULK COLLECT INTO n1_array, n2_array;

   FORALL i IN 1 .. n1_array.COUNT
      MERGE INTO t2
           USING DUAL
              ON (t2.n1 = n1_array (i))
      WHEN MATCHED
      THEN
         UPDATE SET t2.n2 = n2_array (i)
      WHEN NOT MATCHED
      THEN
         INSERT     (n1, n2)
             VALUES (n1_array (i), n2_array (i));
END;
/
```

清单 1-9 的 PL/SQL 块中的第一条 SQL 语句是一个 `DELETE` 语句，它将从 `T1` 删除的 30 行数据返回到两个数值数组中。该语句的 `SQL_ID` 是 ‘d6qp89kta7b8y’，其底层文本可以使用清单 1-10 中的查询检索。

清单 1-10。显示 PL/SQL 语句的底层文本
```
SELECT 'Output: ' || sql_text
  FROM v$sql
 WHERE sql_id = 'd6qp89kta7b8y';

Output: DELETE FROM T1 RETURNING N1, C2 INTO :O0 ,:O1
```

你可以看到，这次绑定变量 `:O0` 和 `:O1` 被用于输出。PL/SQL 中表示使用数组接口的 `BULK COLLECT` 语法，已从 PL/SQL 提交给 SQL 引擎的语句中移除了。

清单 1-9 中的 `MERGE` 语句也使用了数组接口，这次是用于输入。因为 `T2` 是空的，所以最终结果是，从 `T1` 删除的全部 30 行数据被插入到了 `T2` 中。其 `SQL_ID` 是 ‘2c8z1d90u77t4’，如果你从 `V$SQL` 中检索文本，会看到所有空白符已被压缩，所有标识符都以大写显示。这对于从 PL/SQL 发出的 SQL 来说是正常的。

### PL/SQL FORALL 语法

很容易认为 PL/SQL 的 `FORALL` 语法代表一个循环。它不是。它只是在将数组数据传递给数据操作语言 (DML) 语句时调用数组接口的一种方式，就像 `BULK COLLECT` 用于在检索数据时调用数组接口一样。

数组接口对于从应用服务器发出的代码尤为重要，因为它避免了客户端与服务器之间的多次往返，因此影响可能是巨大的。

### 子查询因式分解



