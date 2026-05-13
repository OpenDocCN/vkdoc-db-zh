# ILO 示例

当你的代码被正确插桩时，诊断性能问题的过程是怎样的？这里我展示一个来自我们在 Method R 公司自身应用开发中的案例研究。这个例子源于我们正在为一个客户所做的工作，该客户专门因为我们承诺交付现在和未来都快速的代码而与我们签约。

在许多情况下，ILO 是通过用 Java 或其他数据库外部语言编写的应用程序调用来实现的。在这个例子中，应用程序是用 Java 编写的，但所有数据库访问都是通过一个 PL/SQL API 完成的。每个 API 过程都使用 ILO，通过适当的模块和操作名称进行了插桩。

遵循 Method R 的方法，第一步是确定最需要性能改进的任务。根据最终用户的反馈，相关任务通过模块名称`assignment`被确定。

因为代码已经用 ILO 进行了插桩，所以第二步的测量变得微不足道。当我们遇到性能问题时，无需添加任何调试或日志记录代码。我们使用 ILO 为`assignment`任务发出性能数据并开启扩展 SQL 跟踪。在负载测试中执行生产代码后，我们使用以下查询分析了由 ILO 提供的响应时间数据：

```sql
SELECT
  ilo_module,
  ilo_action,
  SUM(elapsed_seconds)                                    total_duration,
  AVG(elapsed_seconds)                                    avg_duration,
  COUNT(*)                                                cnt
FROM ilo_run_vw
GROUP by ilo_module,
         ilo_action
ORDER BY SUM(elapsed_seconds) DESC;
```

```
                                            TOTAL DURATION   AVG DURATION      
ILO_MODULE      ILO_ACTION                    (SECONDS)      (SECONDS) COUNT
--------------  ---------------------- -------------- -------------- -----
assignment      SetLocationData              9.064287     0.09064287   100
SetLocationData get_device                   8.315280     0.08315280   100
SetLocationData get_location                 0.502014     0.00502014   100
SetLocationData update_location              0.036686     0.00036686   100
SetLocationData insert_reservation           0.035895     0.00035895   100
SetLocationData update_device                0.034592     0.00034592   100
SetLocationData merge_device                 0.031978     0.00031978   100
```

所提供的信息正是我们定位性能问题所需要的。在查询结果中，你可以看到`get_device`过程是响应时间的最大贡献者，在 9 秒的`SetLocationData`任务中占了超过 8 秒。借助此任务的扩展 SQL 跟踪数据，生成了一个性能剖析，进而导向了该过程的`优化`。此处省略优化的细节，以强调该方法的重要性。

在许多情况下，检查执行时间的差异可能是一种强大的性能分析技术。`get_device`过程的响应时间是本案例中我们主要关注点，但如果这不是一个当前的性能问题呢？我们可以使用 Oracle 的统计分析函数来查找差异最大的任务，如下所示：

```sql
SELECT
  ilo_module,
  ilo_action,
  VARIANCE(elapsed_seconds)                               variance,
  STDDEV(elapsed_seconds)                                 standard_deviation,
  ROUND(VARIANCE(elapsed_seconds)/AVG(elapsed_seconds),3) variance_to_mean_ratio,
  ROUND(STDDEV(elapsed_seconds)/AVG(elapsed_seconds),3)   coefficient_of_variation
FROM ilo_run_vw
GROUP by ilo_module,
         ilo_action
ORDER BY 5 DESC, 3 DESC;
```

```
                                                    STANDARD VARIANCE TO  COEFFICIENT
ILO_MODULE      ILO_ACTION               VARIANCE  DEVIATION  MEAN RATIO OF VARIATION
--------------  ---------------------- ---------- ---------- ----------- ------------
SetLocationData get_location           0.00004253 0.00652182       0.008        1.299
SetLocationData get_device             0.00033352 0.01826257       0.004        0.243
assignment      SetLocationData        0.00013663 0.01168871       0.002        0.129
SetLocationData update_location        5.8995E-09 0.00007680       0.000        0.209
SetLocationData insert_reservation     5.3832E-09 0.00007337       0.000        0.204
SetLocationData update_device          3.7374E-09 0.00006113       0.000        0.177
SetLocationData merge_device           7.8088E-09 0.00008837       0.000        0.276
```

查询结果显示，`get_location`和`get_device`两者都具有较高的方差与均值比率(VMR)和变异系数。如果这些过程每次调用执行的工作量相同，那么响应时间的差异就是一个值得深究的问题。关于在性能问题诊断中统计分析技术的进一步解释，我强烈推荐 Robyn Sands 在*Expert Oracle Practices*（Apress，2010）一书第 13 章中的工作。她在其中解释了上述输出中所示的变异系数和方差与均值比率计算的重要性。

以下代码摘录只是生成上述 ILO 数据的应用程序中的一小部分样本。我将代码放在这里是为了更清楚地展示插桩的简洁性。

```sql
PROCEDURE SetLocationData(…) IS
-- code omitted  

PROCEDURE get_location(…) IS
BEGIN
  ilo_wrapper.begin_task(module => 'assignment',
                         action => 'get_location',
                         comment => 'P_ID=' || p_id || ' p_key=' || p_key);

    -- code omitted

    ilo_wrapper.end_task(widget_count => p_data_count);
  EXCEPTION WHEN OTHERS THEN
    ilo_wrapper.end_task(error_num => SQLCODE);
  END get_location;

  PROCEDURE get_device() IS
  BEGIN --get_device
    ilo_wrapper.begin_task(module => 'assignment', action => 'get_device');

    -- code omitted

    ilo_wrapper.end_task;
  EXCEPTION WHEN OTHERS THEN
    ilo_wrapper.end_task(error_num => SQLCODE);
  END get_device;

ilo_wrapper.end_task(error_num => status_code, widget_count => v_cnt);

EXCEPTION WHEN OTHERS THEN
  ilo_wrapper.end_task(error_num => SQLCODE);
END SetLocationData;
```

这里真正的关键点是，性能优化，尤其是插桩，需要在你开发过程中的思维方式里占据首要位置。通过对代码进行插桩，我们能够排除所有与问题无关的代码部分。了解真正的问题是解决方案中最首要也是最重要的一步。

![images](img/square.jpg) **提示** 注意，对 ILO 的调用被包裹在一个名为`ilo_wrapper`的特定于应用程序的包中。这是一个好主意。如果你使用 ILO 或任何其他外部库，将该代码抽象出来是合理的。如果 ILO 的 API 发生变化，那么只需要更改你的包装器包即可与之匹配。



### 性能分析示例

在实践中，性能分析的过程是什么样的？这里有一个我在 SQL*Plus 中执行的示例，用来回答我之前提出的同一个问题：为什么程序 `p` 运行缓慢？这次我将使用 ILO 进行插桩，而不是使用 `DBMS_OUTPUT.PUT_LINE`。

```sql
CREATE OR REPLACE PROCEDURE q AS
  l_sqlcode NUMBER;
  l_sqlerrm VARCHAR2(512);
BEGIN
  ilo_task.begin_task(module => 'Procedure P',
                      action => 'q');
  FOR rec IN (SELECT * FROM V$SESSION) LOOP
    NULL;
  END LOOP;
  ilo_task.end_task;
EXCEPTION WHEN OTHERS THEN
  l_sqlcode := SQLCODE;
  l_sqlerrm := SQLERRM;
  ilo_task.end_task(error_num => l_sqlcode);
  -- handle exceptions here
END;
/
```

```sql
CREATE OR REPLACE PROCEDURE r AS
  l_sqlcode NUMBER;
  l_sqlerrm VARCHAR2(512);
BEGIN
  ilo_task.begin_task(module => 'Procedure P',
                      action => 'r');
  FOR rec IN (SELECT * FROM ALL_OBJECTS) LOOP
    NULL;
  END LOOP;
  ilo_task.end_task;
EXCEPTION WHEN OTHERS THEN
  l_sqlcode := SQLCODE;
  l_sqlerrm := SQLERRM;
  ilo_task.end_task(error_num => l_sqlcode);
  -- handle exceptions here
END;
/
```

```sql
CREATE OR REPLACE PROCEDURE s AS
  l_sqlcode NUMBER;
  l_sqlerrm VARCHAR2(512);
BEGIN
  ilo_task.begin_task(module => 'Procedure P',
                      action => 's');
  FOR rec IN (SELECT * FROM USER_TABLES) LOOP
    NULL;
  END LOOP;
  ilo_task.end_task;
EXCEPTION WHEN OTHERS THEN
  l_sqlcode := SQLCODE;
  l_sqlerrm := SQLERRM;
  ilo_task.end_task(error_num => l_sqlcode);
  -- handle exceptions here
END;
/
```

```sql
CREATE OR REPLACE PROCEDURE p AS
  l_sqlcode NUMBER;
  l_sqlerrm VARCHAR2(512);
BEGIN
  ilo_task.begin_task(module => 'Month End Posting',
                      action => 'p');
  q;
  r;
  s;
  ilo_task.end_task;
EXCEPTION WHEN OTHERS THEN
  l_sqlcode := SQLCODE;
  l_sqlerrm := SQLERRM;
  ilo_task.end_task(error_num => l_sqlcode);
  -- handle exceptions here
  raise;
END;
/
```

```sql
DELETE FROM ilo_run WHERE ilo_module IN ('Month End Posting', 'Procedure P');
```

```sql
EXEC ilo_timer.set_mark_all_tasks_interesting(true,true);
```

```sql
EXEC p;
```

```sql
EXEC ilo_timer.flush_ilo_runs;
```

```sql
set pages 1000
COLUMN ILO_MODULE FORMAT A20
COLUMN ILO_ACTION FORMAT A10
SELECT ILO_MODULE, ILO_ACTION, ELAPSED_SECONDS
FROM ilo_run_vw WHERE ilo_module IN ('Month End Posting', 'Procedure P') ORDER BY 3 DESC;
```

```
ILO_MODULE           ILO_ACTION ELAPSED_SECONDS
-------------------- ---------- ---------------
Month End Posting    p                 8.566376
Procedure P          r                 8.418068
Procedure P          s                  .135682
Procedure P          q                  .005708
```

SQL*Plus 脚本中的查询结果精确地向我展示了时间花费在哪里，这些时间与每个业务任务相关联，并通过模块和操作名称来标识。查询结果是一个性能剖析（假设我知道过程之间的层次关系，并且第一行代表了通常显示在底部的总计行），因此我可以轻松地看出，如果我的目标是优化响应时间，那么过程 `r` 需要我的关注。

除了将耗时信息存储在数据库中，ILO 还开启了扩展 SQL 跟踪，因此生成了一个跟踪文件。Oracle 的跟踪数据流允许我深入挖掘过程 `p`，查看构成响应时间的所有数据库和系统调用。由于模块和操作名称也会在跟踪文件中输出，我还能够专门深入挖掘仅对应于过程 `r` 的调用。

首先，我想看看每个模块/操作对的时间都花在了哪里。对跟踪文件中的响应时间数据进行汇总，结果如下：

```
Module:Action        DURATION       %  CALLS      MEAN       MIN       MAX
-------------------  --------  ------  -----  --------  --------  --------
Procedure P:r        8.332521   98.4%    545  0.015289  0.000000  1.968123
Procedure P:s        0.132008    1.6%      4  0.033002  0.000000  0.132008
Procedure P:q        0.000000    0.0%     16  0.000000  0.000000  0.000000
Month End Posting:p  0.000000    0.0%     66  0.000000  0.000000  0.000000
SQL*Plus:            0.000000    0.0%      2  0.000000  0.000000  0.000000
-------------------  --------  ------  -----  --------  --------  --------
TOTAL (5)            8.464529  100.0%    633  0.013372  0.000000  1.968123
```

在这个性能剖析中，为父过程 `p` 显示的数据并不包含其任何子过程的响应时间贡献。跟踪文件中显示的总耗时约为 8.46 秒。这与我的 ILO 查询中显示的值并不完全匹配，但这不成问题，因为这些数字足够接近我的分析。关于差异原因的更多信息，请参阅《**优化 Oracle 性能**》（O'Reilly，2003 年）。

![images](img/square.jpg) **注意** 在此示例中，我使用了 Method R 公司销售的 MR Tools 套件中的 `mrskew` 来生成汇总信息。Oracle 的 `tkprof` 和 Trace Analyzer 工具确实提供了一些聚合功能，但它们无法提供您所需的性能剖析。

接下来，我想看看在仅执行过程 `r` 时的时间花费情况。为此，我将深入挖掘跟踪文件数据，仅汇总模块和操作分别设置为 Month End Posting 和 `r` 时执行的调用：

```
Call        DURATION       %  CALLS      MEAN       MIN       MAX
----------  --------  ------  -----  --------  --------  --------
FETCH       8.332521  100.0%    542  0.015374  0.000000  1.968123
PARSE       0.000000    0.0%      1  0.000000  0.000000  0.000000
CLOSE       0.000000    0.0%      1  0.000000  0.000000  0.000000
EXEC        0.000000    0.0%      1  0.000000  0.000000  0.000000
----------  --------  ------  -----  --------  --------  --------
TOTAL (4)   8.332521  100.0%    545  0.015289  0.000000  1.968123
```

现在我清楚地看到数据库在花费的 8.33 秒期间做了什么。所有时间都花费在了 `FETCH` 数据库调用上。这并不奇怪，因为 `r` 所做的就是执行一个对 `ALL_OBJECTS` 的单一查询。但它的价值依然存在——我知道时间花在了哪里。但跟踪文件包含的详细信息远不止于此，因此进一步深入挖掘可以提供有价值的洞察。我想知道跟踪文件中的哪些行对我的过程 `r` 贡献了最大的时间。所有的 `FETCH` 调用持续时间都相同吗？以下是该问题的答案（同样由 `mrskew` 提供）：

```
Line Number  DURATION       %  CALLS      MEAN       MIN       MAX
-----------  --------  ------  -----  --------  --------  --------
       4231  1.968123   23.6%      1  1.968123  1.968123  1.968123
       2689  0.380023    4.6%      1  0.380023  0.380023  0.380023
       4212  0.316020    3.8%      1  0.316020  0.316020  0.316020
       4233  0.304019    3.6%      1  0.304019  0.304019  0.304019
       2695  0.156010    1.9%      1  0.156010  0.156010  0.156010
        281  0.144009    1.7%      1  0.144009  0.144009  0.144009
        362  0.100007    1.2%      1  0.100007  0.100007  0.100007
        879  0.064004    0.8%      1  0.064004  0.064004  0.064004
       1348  0.056003    0.7%      1  0.056003  0.056003  0.056003
       2473  0.052003    0.6%      1  0.052003  0.052003  0.052003
 532 others  4.792300   57.5%    532  0.009008  0.000000  0.044003
-----------  --------  ------  -----  --------  --------  --------
TOTAL (542)  8.332521  100.0%    542  0.015374  0.000000  1.968123
```


事实上，所有的 `FETCH` 调用并非具有相同的持续时间。在跟踪文件第 4231 行的那个单独的 `FETCH` 调用，几乎占用了 8.33 秒总响应时间的 24%。仪表化和性能分析使我能够精确地深入定位需要优化软件的代码位置。在这个经过简化的示例中，优化过程 `r` 就意味着优化查询 `SELECT * FROM ALL_OBJECTS`。这个例子中较长的 `FETCH` 调用是递归 SQL 导致的一个异常情况，但它很好地展示了这种方法。

### 总结

在软件开发生命周期中，你应该像对待任何其他软件语言一样，遵循相同的规范来对待 PL/SQL——尤其是仪表化这一规范。为了编写出快速的代码，并确保其在应用程序的整个生命周期中持续保持快速，你必须在开发期间对代码进行正确的仪表化。用于底层调试和性能分析的工具在开发期间是存在且有用的，但它们并不太适合在生产环境中使用。

当生产环境中出现性能问题时，分析师需要快速找出对响应时间贡献最大的代码区域。正确的仪表化将为性能分析提供数据，清晰地展示这些信息，从而成功地分析和解决问题。当你能够生成实际运行代码的性能分析，并清晰地指出花费时间最多的代码部分时，你就可以 `知道` 你正在衡量正确的指标，并且采用了正确的方式进行仪表化。向你的代码中添加良好的仪表化并不困难，而 Oracle 的仪表化库 `ILO` 能让你轻松完成这项工作。

## 第 14 章：编码规范与错误处理

**作者：Stephan Petit**

你能同时穿晚礼服和勃肯鞋吗？是的，可以！任何在粒子物理实验室待过一段时间的人都知道这在技术上是可能的。然而，有人可能会说这样的穿着有悖于优雅的惯例。你也可能喜欢将绿色和蓝色衣服搭配在一起。（我个人更喜欢混搭红色和黑色）。关于各自的品味，我们可以无休止地争论：每个人都对；每个人都错。与此同时，无论成文与否，规范都有助于共同生活和协作。它们有自己的历史和原因。

编写代码通常很有趣，我们都希望它尽可能好：执行效率高、结构紧凑，如果可能的话，还要有风格。但我们通常不只是为了自己而写代码。如果你曾经为了详细了解某段代码的功能、调试它或修改它，而去阅读别人写的代码，你可能想过“嗯，我不会那样做。”它能运行，但可能很难理解——就像听一个带着浓重口音的表亲说话一样。本章的目标是提出一种编写 PL/SQL 的方法。我将首先讨论编码规范的优势，然后提出具体的格式规范。接着，我会讨论错误处理，并提供错误处理规范的示例。最后，你会找到一个模板，总结了本章概述的规范。

![images](img/square.jpg) **注意** 我绝不声称我所概述的规范是唯一可行的，但它们在我日常工作中已证明是高效的。我认为理解每个规范的原因，并看到一种方法相对于另一种方法的好处，是很重要的。

### 为何需要编码规范？

为什么要费这个事？我们真的需要编码规范吗？让每个人自由发挥自己的艺术天赋，在变量命名、循环缩进等方面任其发挥，岂不是更好？这种自由听起来很诱人，但这是通往毁灭的道路——或者至少是导致你无法按时回家吃饭，因为你被困在办公室调试某些难以理解的代码。以下是建立结构并遵守一套良好规则所带来的好处：

> **快速理解**：遵循规范编写的代码更易于阅读，因此阅读速度更快。只需稍加练习，读者的眼睛就能立即识别代码中的模式。结构清晰可见，各种元素类型和逻辑块变得显而易见。它还有一个优势，就是能最大限度地减少误解的可能性。说“我喜欢巧克力”比说“我不否认我不讨厌巧克力”要好，即使两句话意思相同。
>
> **可靠性**：一旦你的代码完成，让同事审阅一下以找出可能的错误、不一致之处，并确保其逻辑清晰，是件好事。易于阅读和理解的代码能够得到更好的审阅，因此可能更可靠。根据我的经验，我知道当同事知道阅读我的代码不会因为它是我的古怪风格的体现而成为一种痛苦时，也更容易为我的工作找到审阅者。
>
> **可维护性**：随着时间的推移维护代码是一项挑战。当你几个月后重新审视自己的代码时，你可能会发现连自己的代码都难以理解。当整个软件包是由多人编写时，情况会更糟。确保在系统生命周期内使用统一的风格，对于维护工作大有裨益。
>
> **安全性**：编码规范可以提升系统的安全水平。例如，系统地使用绑定变量可以抵御代码注入威胁。这一点将在后续段落中更详细地描述。
>
> **可培训性**：当负责一个系统的团队中几乎每年有一半成员更换时，尽快培训每一位新成员至关重要。当每个人都使用相同的方式表达时，培训会更快。新成员可能不愿意学习一种他/她已经掌握的语言的新使用方式。要敢于推行整个团队使用的标准，并在可能的情况下，让新成员参与现有模块的开发。随处都能发现相同的编码风格，这很快会显现为一个优势。
>
> **编码速度**：为什么每次开始编写新模块时都要重新发明轮子？使用编码规范可以消除你可能会问自己的一些关于代码风格的问题。例如，在修改现有过程时，应该遵循原作者的风格，还是应该使用自己的风格？当每个人都使用相同的风格时，这种困境就消失了。
>
> **错误处理**：确保所有错误都能被优雅地捕获并正确处理，是交付优秀软件的另一个棘手部分。因此，拥有系统化的错误处理方法和标准化的错误报告方式非常重要。编码规范可以为软件编程的这一方面提供强大的解决方案。

### 格式

让我们首先看一下关于代码实际格式的详细规范。格式化是那些挑剔的人更偏好的规范的一个方面。敢于挑剔。遵循一些简单的规则可以使代码更易于阅读、重用和维护。

以下部分描述了一些多年来对我行之有效的格式规则。我遵循它们。我的团队也遵循它们。不过，请不要将它们奉为金科玉律。请随意采纳或调整它们，甚至发明你自己的！重要的是，大家步调一致。

#### 大小写

对所有 SQL 和 PL/SQL 关键字、过程名、函数名、异常名和常量名使用大写。

对 `变量`、注释、`表名` 和 `列名` 使用小写。


