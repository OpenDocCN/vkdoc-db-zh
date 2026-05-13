# 使用动态 SQL 实现通用克隆

有趣的是，如果仔细检查所生成的代码，会发现它包含三种元素：

*   传入的父主键列表（`V_OLDPHONE_TT`）。
*   一个主要的逻辑处理过程。
*   定义处理过程应作用对象的功能标识符：
    *   子表名称（`REF`）。
    *   子表的主键列名（`REF_ID`）。
    *   子表的外键列名（`PHONE_ID`）。

从技术角度讲，我试图构建一个具有结构化参数（表名、列名）和数据参数（父 ID）的模块，这正是利用动态 SQL 的完美案例，因为它能同时处理这两类参数。

由于层级结构的每一层都可以通过检查外键关系来完整表示，因此显然可以使用 Oracle 数据字典来遍历父-子树（目前，我将假设系统中不存在循环依赖）。思路非常直接：以表名作为输入，返回其子表列表（包含相应的子表主键列和指向父表的外键列）。虽然以下代码不包含任何动态 SQL，但它足够有用，值得展示（同样摘自`CLONE_PKG`包）：

```sql
TYPE list_rec IS RECORD
        (table_tx VARCHAR2(50), fk_tx VARCHAR2(50), pk_tx VARCHAR2(50));
TYPE list_rec_tt IS TABLE OF list_rec;

FUNCTION f_getChildrenRec (in_tablename_tx VARCHAR2)
RETURN list_rec_tt
IS
    v_out_tt list_rec_tt;
BEGIN
    SELECT fk_tab.table_name, fk_tab.column_name fk_tx, pk_tab.column_name pk_tx
    BULK COLLECT INTO v_Out_tt
    FROM
        (SELECT ucc.column_name, uc.table_name
         FROM user_cons_columns ucc,
              user_constraints uc
         WHERE ucc.constraint_name = uc.constraint_name
         AND   constraint_type = 'P') pk_tab,
         (SELECT ucc.column_name, uc.table_name
          FROM   user_cons_columns ucc,
                (SELECT constraint_name, table_name
                FROM user_constraints
                WHERE r_constraint_name = (SELECT constraint_name
                                           FROM user_constraints
                                           WHERE table_name = in_tablename_tx
                                           AND constraint_type = 'P'
                                           )
                   ) uc
          WHERE ucc.constraint_name = uc.constraint_name ) fk_tab
    WHERE pk_tab.table_name = fk_tab.table_name;
    RETURN v_out_tt;
END;
```

现在，我拥有了构建通用处理模块所需的所有信息片段，该模块将递归调用自身，直到达到父-子链的末端，如下所示。如定义所示，该模块将接收一个父主键 ID 集合和一个`CLONE_PKG.LIST_REC`类型的单一对象作为输入，该对象将描述要处理的父-子链接。

```sql
PROCEDURE p_process (in_list_rec clone_pkg.list_rec, in_parent_list id_tt)
IS
    v_execute_tx VARCHAR2(32767);
BEGIN
    v_execute_tx:=
        'DECLARE '||
        '    TYPE rows_tt IS TABLE OF '||in_list_rec.table_tx||'%rowtype;'||
        '    v_rows_tt rows_tt;'||
        '    v_new_id number;'||
        '    v_list clone_pkg.list_rec_tt;'||
        '    v_parent_list id_tt:=id_tt();'||
        'BEGIN '||
        '    SELECT * BULK COLLECT INTO v_rows_tt '||
        '    FROM '||in_list_rec.table_tx||' t WHERE '||in_list_rec.fk_tx||
        '       IN  (SELECT column_value FROM TABLE (CAST (:1 as id_tt)));'||
        '    IF v_rows_tt.count()=0 THEN RETURN; END IF;'||
        '    FOR i IN v_rows_tt.first..v_rows_tt.last LOOP '||
        '       SELECT object_Seq.nextval INTO v_new_id FROM DUAL;'||
        '       v_parent_list.extend;'||
        '       v_parent_list(v_parent_list.last):=v_rows_tt(i).'||in_list_rec.pk_tx||';'||
        '       clone_pkg.v_Pair_t(v_rows_tt(i).'||in_list_rec.pk_tx||'):=v_new_id;'||
        '       v_rows_tt(i).'||in_list_rec.pk_tx||':=v_new_id;'||
        '       v_rows_tt(i).'||in_list_rec.fk_tx||
                        ':=clone_pkg.v_Pair_t(v_rows_tt(i).'||in_list_rec.fk_tx||');'||
        '    END LOOP;'||
        '    FORALL i IN v_rows_tt.first..v_rows_tt.last '||
        '    INSERT INTO '||in_list_rec.table_tx||' VALUES v_rows_tt(i);'||
        '    v_list:=clone_pkg.f_getchildrenRec('''||in_list_rec.table_tx||''');'||
        '    IF v_list.count()=0 THEN RETURN; END IF;'||
        '    FOR l IN v_list.first..v_list.last LOOP '||
        '        clone_pkg.p_process(v_list(l),v_parent_list);'||
        '    END LOOP;'||
        'END;';
    EXECUTE IMMEDIATE v_execute_tx USING in_parent_list;
END;
```

最后一块需要的是一个入口点，它将接收一个根对象并克隆它。要启动整个过程，你需要知道起始表、其主键列以及要克隆的根元素的主键。

```sql
PROCEDURE p_clone (in_table_tx VARCHAR2, in_pk_tx VARCHAR2, in_id NUMBER)
IS
    v_new_id NUMBER;    
    PROCEDURE p_processRoot is
        v_sql_tx VARCHAR2(32767);
    BEGIN
      v_sql_tx:=
      'DECLARE '||
      '   v_row '||in_table_tx||'%ROWTYPE; '||
      '   v_listDirectChildren_t clone_pkg.list_rec_tt; '||
      '   v_parent_list id_tt:=id_tt(); '||
      '   v_old_id NUMBER:=:1; '||
      '   v_new_id NUMBER:=:2; '||
      'BEGIN '||
      '  SELECT * INTO v_row FROM '||in_table_tx||
      '    WHERE '||in_pk_tx||'=v_old_id;'||
      '    v_row.'||in_pk_tx||':=v_new_id;'||
      '    clone_pkg.v_Pair_t(v_old_id):=v_new_id;'||
      '    v_parent_list.extend;'||
      '    v_parent_list(v_parent_list.last):=v_old_id;'||
      '    INSERT INTO '||in_table_tx||' VALUES v_row;'||
      '    v_listDirectChildren_t:=clone_pkg.f_getChildrenRec(:3);'||
      '    FOR i IN v_listDirectChildren_t.first..v_listDirectChildren_t.last'||
      '    LOOP'||
      '        clone_pkg.p_process(v_listDirectChildren_t(i),v_parent_list); '||
      '     END LOOP;'||
      'END; ';
      EXECUTE IMMEDIATE v_sql_tx USING in_id,v_new_id,UPPER(in_table_tx);
    END;
BEGIN
    clone_pkg.v_Pair_t.delete;
    SELECT object_seq.nextval INTO v_new_id FROM DUAL;
    p_processRoot;
END;
```

剩下的唯一步骤就是将所有这些片段组装到一个包中，以创建一个完全通用的克隆模块。（完整的代码片段集可以在 Apress.com 上本书的目录页面下载）。

为什么前面的例子被认为是“黄金标准”？主要是因为该示例基于对代码中重复模式的分析，这些模式可以被泛化。识别这些模式是应用动态 SQL 解决日常问题并变得非常高效的关键技能之一。

### 安全问题

## 安全理念

从安全角度来看，“处理未知情况”需要一定程度的**健康的偏执**。任何优秀的开发者都应该假设，如果某样东西*可能*被滥用，那么它*终将*被滥用的几率就很大。同时，非常常见的情况是，“操作者失误”的后果比任何可想象的蓄意攻击都更具破坏性，这导向一个结论：一个系统不仅需要防范犯罪分子，还需要防范任何不幸的事件组合。

在使用动态 SQL 时，上述概念应转化为以下思想：在任何情况下，都不应该能动态生成任何非预期生成的代码。考虑到代码既包含结构元素（表、列等）也包含数据，适用以下规则：

*   结构元素不能替代数据被传入。
*   只能传入允许的结构元素。

## 两条指导原则

满足这两个条件的解决方案可以通过遵循以下两条规则来实现：

*   当应用用户输入纯数据元素（如列的值等）时，这些值必须通过绑定变量传递给动态 SQL。不应允许将任何值拼接到代码的结构部分。
*   当由于应用用户的操作需要更改代码的整体结构时，此类操作必须限于已知的存储库元素。整体系统安全性应通过以下角色分离来强制执行：
    *   普通用户无权更改存储库。
    *   能够更改存储库的人员是特别指定的管理员。
    *   任何管理员不能同时担任普通用户角色。

### 第一条规则：绑定变量

第一条规则很容易解释。因为绑定变量仅在查询结构解析后才被评估，意外的值（如著名的 `NULL OR 1=1`）根本无法影响任何事情，如下所示：

```sql
SQL> DECLARE
  2     v_tx VARCHAR2(256):='NULL OR 1=1';
  3     v_count_nr NUMBER:=0;
  4  BEGIN
  5     EXECUTE IMMEDIATE 'SELECT count(*) FROM emp WHERE ename = :1'
  6     INTO v_count_nr USING v_tx ;
  7     dbms_output.put_line('Bind: '||v_count_nr);
  8 
  9     EXECUTE IMMEDIATE 'select count(*) FROM emp WHERE ename = '||v_tx
 10     INTO v_count_nr;
 11     dbms_output.put_line('Inject: '||v_count_nr);
 12  END;
 13  /
Bind: 0
Inject: 14
PL/SQL procedure successfully completed.
SQL>
```

### 第二条规则：基于存储库的配置

第二条规则意味着使用存储库，这可能与大多数当代开发者的习惯相悖，但它为输入可用选项的人员和实际使用这些选项的人员之间提供了一个独特的角色分离机会。这种基于存储库的解决方案的另一个优势是部署成本。由于所有需要的更改都可以通过对存储库表执行 DML 来处理，因此没有服务中断，也没有停机时间。这些考虑有时可以真正挽救项目，如下文的真实世界案例所示。

## 真实案例研究

从前有一个经典的 3 层 IT 系统，即使是对前端进行最小的更改，通常也需要大约 4-6 小时的停机时间来部署（外加至少一天的准备工作）。新模块的需求至少每周来两次。这些需求非常简单，例如获取少量输入，执行相关例程，并报告结果。不幸的是，每个需求最初都必须作为一个新的屏幕单独编码，并通过常规机制部署。结果，公司里总是有一群不开心的人，要么是无法及时获得所需数据的用户，要么是每周都要经历数次为了添加一个简单屏幕而让整个系统停机的痛苦的维护团队。

将处理未知的概念应用到这个问题中，应该能使可用的替代方案更加清晰。让我们将信息分成两组：

*   已知：
    *   每个屏幕都必须部署到 Web。
    *   每个屏幕都基于一个单一的请求。
    *   每个请求最多接受五个简单参数。
    *   每个请求以文本形式返回一个摘要。
*   未知：
    *   标题信息（屏幕名称、备注、参数名称）
    *   参数的数据类型（包括可空性和可能的格式掩码）
    *   摘要的格式

通过以建议的结构阐明问题，现在所提议解决方案的格式就清晰了：

*   每个屏幕由存储库中的一行表示，具有以下属性集：
    *   通用名称（弹出屏幕的标题）
    *   最多 5 个参数，每个包括：
        *   标题
        *   必填/非必填标识
        *   数据类型（`NUMBER`/`DATE`/`TEXT`/`LOV`）
        *   可选的转换表达式（例如 UI 中的默认日期格式，因为 Web 上的所有内容都是基于文本的）
        *   值列表名称（用于 `LOV` 数据类型）
    *   相应函数的名称，格式如下：
        *   输入参数的顺序（和数量）必须与屏幕参数的顺序匹配
        *   函数应返回一个 `CLOB`
*   所有注册函数返回的 `CLOB` 必须是完全格式化的 HTML 页面，可立即在前端显示。
*   存储库中的所有活动只能由管理员访问，终端用户不可见。

因此，从系统的角度来看，操作的逻辑流程现在变得非常简单。

1.  用户从存储库中看到可用模块列表并选择一个。
2.  前端应用程序读取存储库，并动态构建一个弹出屏幕，包含适当的输入字段和必填指示器。如果输入字段的数据类型是值列表，该实用程序会请求通用的 `LOV` 机制提供现有的 `ID`/`DISPLAY` 对。
3.  用户输入所需内容并按下 `SUBMIT`。前端通过将正在使用的模块的存储库 ID 和所有用户输入的值传递给它，来触发主（伞形）过程。
4.  伞形过程构建一个真实的函数调用，传入输入的值，并将生成的 `CLOB` 返回给前端（已经格式化为 HTML）。
5.  前端显示生成的 HTML。

现在所有团队都会很高兴，因为从任何新模块被宣布为生产就绪，到它可从前端访问，只需要几秒钟。没有停机，也没有部署——只需复制一个新函数并在存储库中用一个 `INSERT` 语句注册该函数。

## 实现示例

为了说明所有这些“奇迹”是什么样子，示例的准备部分是构建一个满足所有格式要求的函数，创建一个存储库表，并在存储库中注册该函数：

```sql
CREATE FUNCTION f_getEmp_CL (i_job_tx VARCHAR2, i_hiredate_dt DATE)
RETURN CLOB
IS
    v_out_cl CLOB;
    PROCEDURE p_add(pi_tx VARCHAR2) IS
    BEGIN
        dbms_lob.writeappend(v_out_cl,length(pi_tx),pi_tx);
    END;
BEGIN
    dbms_lob.createtemporary(v_out_cl,true,dbms_lob.call);
    p_add('<html><table>');    
    FOR c IN (SELECT '<tr>'||'<td>'||empno||'</td>'||'<td>'||ename||'</td>'||'</tr>' row_tx
                     FROM emp
                     WHERE job = i_job_tx
                     AND hiredate >= NVL(i_hiredate_dt,add_months(sysdate,-36))
                     )
    LOOP
        p_add(c.row_tx);
    END LOOP;
    p_add('</table></html>');
    RETURN v_out_cl;
END;
```

