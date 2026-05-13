# Oracle 中的动态 SQL

### 性能与资源利用

除了安全风险，还有一个阻碍人们有效使用动态 SQL 的“拦路虎”：人们认为它在性能上代价过高。问题在于这种说法从未被完整阐述：所谓“代价高”是与什么相比？确实，直接运行同一条语句比将其包装在 `EXECUTE IMMEDIATE` 命令中要快。但这就像拿苹果和橘子作比较。单个任务运行所需的时间绝对无关紧要，因为唯一需要调优的关键点是模块的整体性能。

请再看一下上一节中使用动态 SQL 与仓库交互的示例。当然，这会带来一些开销，例如将结果预格式化为 HTML、使用封装模块、对底层函数进行动态调用等等。与手工构建的界面相比，这些因素中的每一个都会降低请求处理的速度。但关键问题依然存在：有人注意到这个减速吗？答案是“没有”，因为所有列出的“开销”最多只会导致毫秒级的时间消耗，相对于总处理时间而言可以忽略不计。

这意味着，实际需要评估的因素是，为使用动态 SQL 所付出的代价是否值得。每次的答案都将取决于对特定业务场景的详细分析。会有一些场景中这个代价是可以接受的（比如刚刚回顾的那个），而在另一些场景中代价会过高。下面的例子说明了后一种情况。

### 反模式

有时，最终用户会在应用程序中要求一个多选选项。这通常意味着需要对一个未定义的元素列表执行某项操作。由于列表的长度未知，使用文本字符串作为通信机制就显得非常诱人，如以下流程所示：

1.  前端代码将所有选中的 ID 连接成一个逗号分隔的字符串。
2.  该字符串被传递给 PL/SQL 函数。
3.  PL/SQL 函数利用动态 SQL 来构建 `IN` 子句。

以下是该流程最后一步所需的 PL/SQL 函数：

```sql
CREATE FUNCTION f_getSumSal_nr (i_empno_tx VARCHAR2)
RETURN NUMBER
IS
    v_out_nr NUMBER:=0;
    v_sql_tx VARCHAR2(2000);
BEGIN
    IF i_empno_tx IS NOT NULL THEN
        v_sql_tx:='SELECT sum(sal) FROM emp WHERE empno IN ('||i_empno_tx||')';
        EXECUTE IMMEDIATE v_sql_tx INTO v_out_nr;
    END IF;
    RETURN v_out_nr;
END;
```

不幸的是，这个看似干净的解决方案存在一些缺点，通常只在应用程序部署到生产环境一段时间后才会显现。以下列举几点：

*   对可传递对象的总数有限制：`32K/(ID 长度+1)` 或 `4K/(ID 长度+1)`，取决于该函数是仅在 PL/SQL 中使用还是作为 SQL 语句的一部分。将整个模块转换为 `CLOB` 而不是 `VARCHAR2` 代价非常高，因为 `CLOB` 的开销，特别是在物理 `I/O` 方面。
*   如果值的列表不只是由 ID 构成，而是某种文本，这就引出了文本中引号数量是否正确的问题。
*   由于列表中项目数量未知，可能会得到不同的执行计划。此外，此代码会非常容易受到列级统计信息变更的影响。
*   它不安全（传递 `SELECT empno FROM emp` 会立即提供不同级别的数据访问权限），但要使此模块防注入，需要进行非常复杂的字符串解析。

总的来说，根据我的经验，动态 SQL 在这种情况下产生的效果是负面的。替代方案如下所示：

```sql
CREATE TYPE id_tt IS TABLE OF NUMBER;

CREATE FUNCTION f_getSumSal_nr (i_tt id_tt)
RETURN NUMBER IS
    v_out_nr NUMBER:=0;
BEGIN
    IF i_tt IS NOT NULL AND i_tt.count>0 THEN
        SELECT sum(sal) INTO v_out_nr
        FROM emp
        WHERE empno IN (SELECT t.column_value FROM TABLE(CAST(i_tt as id_tt)) t);
    END IF;
    RETURN v_out_nr;
END;
```

这个解决方案使用一个对象集合来传递列表。这种集合的开销，通过执行常规 SQL 而非动态 SQL，并且无需担心数据长度/数据结构，得到了很好的补偿。



#### 比较动态 SQL 的不同实现

由于动态 SQL 存在不同的实现方式，哪种类型会更高效呢？接下来这个实际案例将说明为什么你需要了解 `DBMS_SQL` 包。这个任务的起因是终端用户希望系统能够根据一系列预定义的模板上传 CSV 文件。以下是该系统的一些需求和特点：

*   文件前缀定义了模板类型。列标题直接映射到表列（如果标题未注册，则将被忽略）。
*   文件中的一行代表一个包含 1 到 N 行的逻辑组。例如，一行可能同时包含操作记录和修正记录（你可以将其视为销售交易和取消交易——两者关联但独立）。
*   必须支持组级别验证。规则适用于整组行，而非每一次插入。

该解决方案最初的实现方式非常直接：读取一行，使用标题信息构建一个文本字符串形式的 `INSERT` 语句，然后执行 `EXECUTE IMMEDIATE` 命令。但当面对真实的数据量（单个文件超过 1 万行）时，数据库管理员开始抱怨 CPU 负载过高。同时，早期采用者也抱怨该模块的性能远低于最初预期（并且随着文件增大而恶化）。经过详细的数据库追踪，问题变得清晰：所有额外耗费的时间都源于解析 `INSERT` 语句。

请记住以下关于解析的信息：

*   不带绑定变量的 `EXECUTE IMMEDIATE`：N 次调用对应 N 次硬解析
*   带绑定变量的 `EXECUTE IMMEDIATE`：
    *   10g 及以下版本：N 次调用对应 1 次硬解析 + N-1 次软解析
    *   11g：仅当重复执行相同语句时，才进行一次硬解析。这是一种内部优化（不可见），并且如果存在多种调用类型，则不会生效。
*   `DBMS_SQL` 将解析和执行阶段分离：对于每种调用类型仅需一次硬解析，因为 `DBMS_SQL` 允许存储已解析的语句并多次复用。

由于大多数模板包含多种类型的 `INSERT` 语句，不幸的是 Oracle 11g 在这种情况下完全帮不上忙。然而，从 `EXECUTE IMMEDIATE` 切换到 `DBMS_SQL` 看起来很有希望，于是我进行了相应的修改。

第一步是准备 `INSERT` 语句，为每条语句分配自己的 `DBMS_SQL` 游标（已解析），并将这些游标存储在关联数组中以便后续访问，如下所示：

```
DECLARE
    TYPE integer_tt IS TABLE OF integer;
    v_cur_tt integer_tt;
    ...
BEGIN
    ...
    FOR r IN v_groupRow_tt.first..v_groupRow_tt.last LOOP
        v_cur_tt(r):=dbms_sql.open_cursor;
        FOR c IN c_cols(v_groupRow_tt(r)) LOOP
            FOR i IN v_header_tt.first..v_header_tt.last LOOP
                IF v_header_tt(i).text=c.name_tx THEN
                    v_col_tt(i):=c;
                    v_columnRow_tt(r||'|'||v_col_tt(i).name_tx):=v_col_tt(i).viewcol_tx;
                    v_col_tx:=v_col_tx||','||v_col_tt(i).viewcol_tx;
                    v_val_tx:=v_val_tx||',:'||v_col_tt(i).viewcol_tx;
                END IF;
            END LOOP;
        END LOOP;
        v_sql_tx:='INSERT INTO '||v_map_rec.view_tx||'('||v_col_tx||') VALUES('||v_val_tx||')';
        dbms_sql.parse(v_cur_tt(r),v_sql_tx,dbms_sql.native);
    END LOOP;
```

现在只需遍历上传文件的所有行，找到对应的行类型（从数组中），绑定变量，然后执行 `INSERT` 语句。

```
    FOR i IN 2..v_row_tt.count LOOP -- first row is a header
        FOR r IN v_groupRow_tt.first..v_groupRow_tt.last LOOP
            FOR c IN v_col_tt.first..v_col_tt.last LOOP
                IF v_columnRow_tt.exists(r||'|'||v_col_tt(c).name_tx)THEN
                    dbms_sql.bind_variable
                         (v_cur_tt(r),':'||v_col_tt(c).viewcol_tx, v_data_tt(c).text);
                END IF;
            END LOOP;
            v_nr:=dbms_sql.execute(v_cur_tt(r));
        END LOOP;
    END LOOP;
    …
END;
```

想象一下，当处理 6 万行数据集的总时间降低了 50 倍时，我是多么惊讶！这无疑让用户们非常高兴。

这个例子完美地说明了系统调优的复杂性。它不能简单地归结为底层技术分析，而必须始终包含对预期结果的全局视角。只有这种性能优化方法才能保证所选方案在此时此地是有效的。根本不存在普遍适用的最佳方法——只有特定情况下的最优解。

### 对象依赖性

使用动态 SQL 时，第三个主要顾虑（仅次于安全性和性能）源于将硬编码解决方案转换为动态方案会导致 Oracle 无法追踪不同数据库对象之间的依赖关系。这个论点很难反驳，因为在编译阶段自动收集的大量“免费”信息确实不复存在了。真正的问题在于：缺少这种信息对系统开发过程究竟有何具体影响？

### 负面影响

显然，可用数据减少并非好事。但在许多情况下，可以从略有不同的来源获取相同信息。让我们看看有哪些可用选项。以下是代码中丢失对象依赖记录可能导致的一些负面影响。

##### 没有依赖路径可追踪

能够清晰记录对象依赖性的唯一可靠方法是使用基于代码库的解决方案，并且只使用一种（且唯一一种）从其生成代码的方式。此外，我强烈建议你以类似于 Oracle 数据字典的方式对代码库进行建模。将表、列和 PL/SQL 对象显式引用，而不是深埋在文本中，是很好的做法。这样一来，整个依赖性问题就简化为将 Oracle 数据字典与你自己的进行比较，这应该不是什么大问题。

使用单一代码生成方法的需求也至关重要，因为它提供了 100% 的可预测性。由于将代码库转换为代码只有一种且唯一一种方式，所有操作都遵循相同的模式，不可能遇到意外的依赖关系。

##### 无法确定实际执行内容

由于许多参数要么由用户输入，要么是动态构建的，如果不实际遇到，很难预测可能的语法错误。这个问题更难解决，但“采样器”概念可能非常有效。以下是一些你可以采取的方法来掌控实际将执行的内容：

*   生成所有可能（或尽可能多）的待执行代码排列组合，并用这些代码创建 PL/SQL 模块。此方法有助于识别初始代码问题。
*   记录所有依赖关系，并保留一个引用相同对象集的简单模块。
*   如果采样模块变为无效状态，则表明代码存在问题。

#### 正面影响

任何特性（包括丢失对象依赖性）都可以被善用。以下是一些例子，说明缺乏依赖信息实际上如何能为你所用。




