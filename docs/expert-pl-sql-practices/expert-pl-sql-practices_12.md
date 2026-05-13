# DBMS_SQL

`DBMS_SQL` 内置包是动态 SQL 思想的最初形态。尽管有传言称它可能被弃用，但该包随着 RDBMS 版本的更新持续进化。显然，保留 `DBMS_SQL` 有其充分理由，因为它提供了对**执行时定义的语句**所能达到的最低级别的控制。你可以将解析、执行、获取结果、定义输出以及处理绑定变量作为独立的命令随意执行。当然，这种粒度控制的代价是性能（但 Oracle 正在不断缩小这一差距）和复杂性。通常的经验法则是，除非你确定无法避免使用 `DBMS_SQL`，否则不要使用它。

以下是除了使用 `DBMS_SQL` 外别无选择的几种情况：

*   你需要超过 `EXECUTE IMMEDIATE` 的 32K 限制（在 Oracle 10g 及更低版本中）。
*   你有未知数量或类型的输入/输出参数（在所有版本中）。
*   你需要对流程进行细粒度控制（在所有版本中，但在 11g 中有更多可能性）。

第一种情况非常直接。你要么触及了限制，要么没有。然而，从开发角度看，后两种情况要有趣得多。本节接下来将通过示例说明，通过低级别访问系统逻辑可以实现什么。

在大量使用 `REF CURSOR` 变量的环境中，迟早会有人问这个问题：如何知道某个游标变量传递的确切查询是什么？在 Oracle 11g 之前，答案要么在源代码中，要么需要在 DBA 级别检查 SGA 的 `v$` 视图。但从 Oracle 11g 开始，可以双向转换 `DBMS_SQL` 游标和 `REF CURSOR`。这一特性使得以下解决方案成为可能：

```
CREATE PROCEDURE p_explainCursor (io_ref_cur IN OUT SYS_REFCURSOR)
IS
    v_cur      INTEGER := dbms_sql.open_cursor;
    v_cols_nr NUMBER := 0;
    v_cols_tt dbms_sql.desc_tab;
BEGIN
    v_cur:=dbms_sql.to_cursor_number(io_ref_cur);
    dbms_sql.describe_columns (v_cur, v_cols_nr, v_cols_tt);
    FOR i IN 1 .. v_cols_nr LOOP
        dbms_output.put_line(v_cols_tt (i).col_name);
    END LOOP;      
    io_ref_cur:=dbms_sql.to_refcursor(v_cur);
END;
```

```
SQL> DECLARE
  2    v_tx VARCHAR2(256):='SELECT * FROM dept';
  3    v_cur SYS_REFCURSOR;
  4  BEGIN
  5    OPEN v_cur FOR v_tx;
  6    p_explainCursor(v_cur);
  7    CLOSE v_cur;
  8  END;
  9  /
DEPTNO
DNAME
LOC
PL/SQL procedure successfully completed.
SQL>
```

此示例利用了过程 `DBMS_SQL.DESCRIBE_COLUMNS`，它接受任意已打开的游标并分析其结构。如果游标是为 SQL 查询打开的，信息包括输出中的列数、列名和列数据类型等。这让我们回到了管理 `UNKNOWN` 的概念。我们获取一些未知的内容，通过应用适当的工具，将这些潜在有用信息的原始材料转换为可用于解决实际业务需求的可靠数据。

### 动态思维示例

在正式介绍了本章的主角之后，现在有必要展示一个恰当应用动态思维的“黄金标准”案例。本节的示例来自一个实际的生产问题，完美地说明了高级数据库开发人员当前面临的问题概念级别。从一开始，这个任务就极具挑战性。

*   存在大约 100 个表，以层次结构描述一个人：客户 A 有电话 B，由参考人 C 确认，而 C 又有地址 D，等等。
*   整个客户及其相关子数据偶尔需要被克隆。
*   所有表都有一个从共享序列 (`OBJECT_SEQ`) 生成的单列合成主键；所有表通过外键链接。
*   数据模型会相当频繁地更改，因此不允许硬编码。请求必须即时处理，因此无法禁用约束或使用任何其他数据转换变通方法。

在这种情况下，克隆过程涉及什么？很明显，克隆根元素（客户）将需要一些硬编码，但其他所有内容在概念上代表沿着依赖树的分层遍历。克隆过程肯定需要某种快速访问存储（本例中是关联数组），用于保存旧/新 ID 对的信息（使用旧 ID 查找新 ID）。此外，还需要一个嵌套表数据类型来保存要传递到下一层级的主键列表。因此，除了主过程的声明外，还创建了以下类型：

```
CREATE PACKAGE clone_pkg
IS
    TYPE pair_tt IS TABLE OF NUMBER INDEX BY BINARY_INTEGER;
    v_Pair_t pair_tt;

    PROCEDURE p_clone (in_table_tx VARCHAR2, in_pk_tx VARCHAR2, in_id NUMBER);
END;
```

```
CREATE TYPE id_tt IS TABLE OF NUMBER;
```

注意，第二个类型 (`ID_TT`) 应创建为数据库对象，因为它将在 SQL 中被积极使用。第一个类型 (`PAIR_TT`) 仅在 PL/SQL 代码上下文中用于存储旧/新值；这就是为什么我在包 (`CLONE_PKG`) 中创建它并立即声明一个该类型的变量。

由于我试图识别潜在模式，我将忽略克隆客户的根过程（因为它将与其他所有过程不同）。同时，我将忽略向下第一层（属于客户的电话），因为它只涵盖了一个子情况（单个父级的多个子项）。让我们从向下两层开始，弄清楚如何克隆确认现有电话号码的参考人（假设新电话已经创建）。从逻辑上讲，操作流程如下：

1.  查找确认所有现有电话的所有参考人。
    1.  现有电话作为电话 ID 集合 (`V_OLDPHONE_TT`) 传递。
    2.  所有检测到的参考人被加载到暂存的 PL/SQL 集合 (`V_ROWS_TT`) 中。
2.  处理每个检测到的参考人。
    1.  存储所有检测到的参考人 ID (`V_PARENT_TT`)，这些 ID 将在层次树的后续层级中使用。
    2.  从序列中检索一个新的 ID，并在全局包变量中记录旧/新 ID 对。
    3.  在暂存集合 (`V_ROWS_TT`) 中，用新值替换主键（参考人 ID）和外键（电话 ID）。
3.  遍历暂存集合，并将新行插入到参考人表中。

实现这些步骤的代码如下：

```
DECLARE
TYPE rows_tt IS TABLE of REF%ROWTYPE;
    v_rows_tt rows_tt;
    v_new_id NUMBER;
    v_parent_tt id_tt:=id_tt();
BEGIN
    SELECT * BULK COLLECT INTO v_rows_tt
FROM REF t WHERE PHONE_ID in
        (SELECT column_value FROM TABLE (CAST (v_oldPhone_tt AS id_tt)));

    FOR i IN v_rows_tt.first..v_rows_tt.last LOOP
      v_parent_tt.extend;
      v_parent_tt(v_parent_tt.last):=v_rows_tt(i).REF_ID;
```


