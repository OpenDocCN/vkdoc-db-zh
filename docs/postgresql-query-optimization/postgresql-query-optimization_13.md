# 12. 函数

本章聚焦于 PostgreSQL 中使用最少且最常被误用的特性——对象，即函数。由于所有现代编程语言都包含用户自定义函数，人们常误以为数据库函数与之同出一辙，并认为如果掌握在应用编程语言中编写函数的时机与方法，便能直接将其应用于 PostgreSQL。这与事实相去甚远。

本章将探讨 PostgreSQL 函数与其他编程语言函数的差异，何时应该创建函数、何时不应创建，以及使用函数如何既能提升性能，又可能导致严重的性能下降。

在开始之前，我们先来探讨一个普遍存在的观点：使用函数会降低可移植性。这话虽有一定道理，但请考虑以下事实：

-  无论是 SQL 语句还是 ORM，都不是 100%可移植的；总是需要做一些（希望是较小的）适配工作。
-  为现有生产系统更换数据库始终是一个重大项目，绝不可能一蹴而就。对应用程序本身进行一些修改是不可避免的。相比之下，转换函数所带来的额外工作量相对较小。

## 函数创建

PostgreSQL 既有内置（内部）函数，也有用户自定义函数。在这一点上，它与其他编程语言并无不同。

### 内部函数

内部函数使用 C 语言编写，并与 PostgreSQL 服务器集成。对于 PostgreSQL 支持的每种数据类型，都存在一系列用于对该类型的变量或列值执行不同操作的函数。类似于命令式语言，这里有用于数学运算的函数、用于字符串操作的函数、用于日期时间操作的函数等等。可用函数和支持类型的列表随着每个新版本的发布而不断扩展。

清单 12-1 展示了一些内置函数的示例。

```
sin(x);
substr(first_name,1,1);
now();
Listing 12-1
内置函数示例
```

### 用户自定义函数

用户自定义函数是由用户创建的函数。PostgreSQL 支持三种用户自定义函数：

-  查询语言函数，即用 SQL 编写的函数
-  C 函数（用 C 或类似 C 的语言，如 C++编写）
-  过程语言函数，用受支持的过程语言（简称 PL）编写

一个`CREATE FUNCTION`命令如清单 12-2 所示。

```
CREATE FUNCTION function_name (par_name1 par_type1, ...)
RETURNS return_type
AS

LANGUAGE function_language;
Listing 12-2
CREATE FUNCTION 命令
```

从 PostgreSQL 的角度看，数据库引擎只捕获函数签名——函数名和参数列表（可能为空）、返回值类型（可能为`void`），以及一些规格说明，如用何种语言编写。对于除 SQL 外的所有过程语言，函数`主体`被包装在一个字符串字面量中，传递给一个了解该语言细节的特殊处理程序。该处理程序可以自行完成解析、语法分析、执行等所有工作，也可以作为 PostgreSQL 与现有编程语言实现之间的“粘合剂”。标准的 PostgreSQL 发行版支持四种过程语言：PL/pgSQL、PL/Tcl、PL/Perl 和 PL/Python。本书将只讨论用 PL/pgSQL 编写的函数。

### 过程语言简介

既然我们正在讨论函数，那么正式介绍编写这些函数所用的语言（或多种语言）似乎是个好主意。

到目前为止，本书中唯一使用的语言是 SQL，在代码片段中，除了`CREATE`操作符外，唯一使用过的操作符是`SELECT`。现在，是时候介绍过程语言了。在本书中，我们仅讨论 PostgreSQL 的原生过程语言`PL/pgSQL`。

用`PL/pgSQL`编写的函数可以包含任何 SQL 操作符（可能带有某些修改）、控制结构（`IF THEN ELSE`、`CASE`、`LOOP`）以及对其他函数的调用。

清单 12-3 展示了一个用`PL/pgSQL`编写的函数示例，该函数在可能的情况下将文本字符串转换为数值，如果字符串不代表数字则返回 null。

```
CREATE OR REPLACE FUNCTION text_to_numeric(input_text text)
RETURNS numeric AS
$BODY$
BEGIN
RETURN replace(input_text, ',', '')::numeric;
EXCEPTION WHEN OTHERS THEN
RETURN NULL::numeric;
END;
$BODY$
LANGUAGE plpgsql;
Listing 12-3
将文本转换为数值的函数
```

利用清单 12-2 中的信息，我们可以识别所有用户自定义函数的共同部分。函数名为`text_to_numeric`，它只有一个类型为`text`的参数`input_text`。

`RETURNS`子句定义了函数返回值的类型（`numeric`），`LANGUAGE`子句指定了函数所用的编写语言`plpgsql`。

现在，让我们更仔细地看看函数主体。

### 美元引用

在上一节中，我们提到函数主体表示为一个字符串字面量；然而，它并非以引号开始和结束，而是以`$BODY$`作为起止。PostgreSQL 中的这种表示法称为`美元引用`，在创建函数时尤其有用。确实，如果你有一个相对较大的函数文本，很可能需要使用单引号或反斜杠，那么你就需要在每次出现时将它们转义（双写）。使用美元引用，你可以使用两个美元符号，中间可能带有某些标签，来定义一个字符串字面量。

使用标签使得这种定义字符串常量的方式特别方便，因为你可以嵌套带有不同标签的字符串。例如，函数主体的开头可能如清单 12-4 所示。

```
$function$
DECLARE
V_error_message text:='Error:';
V_record_id integer;
BEGIN
...
v_error_message:=v_error_message||$em$Record can't be updated, #$em$||quote_literal(v_record_id);
...
END;
$function$
Listing 12-4
嵌套美元引用的用法
```

这里，我们对函数主体使用了带有`function`标签的美元引用。请注意，对于可用于函数主体的标签没有限制规则，包括空标签；唯一的要求是字符串字面量的结束标签与开始标签相同。这里的示例将使用不同的标签，以提醒读者标签并非预定义的。

函数主体以一个`DECLARE`子句开始，这是可选的，如果不需要变量可以省略。`BEGIN`（不带分号）表示语句部分的开始，而`END`关键字应该是函数体中的最后一条语句。

请注意，本节并非关于函数创建的全面指南。有关更多详细信息，请参阅 PostgreSQL 文档。

更多细节将在后续示例中介绍。


### 函数参数与函数输出：空函数

大多数情况下，函数会有一个或多个参数，但也可能没有参数。例如，内部函数 `now()` 用于返回当前时间戳，它没有任何参数。我们可以为任何函数参数指定默认值，以便在未显式传递特定值时使用。

此外，定义函数参数的方式有多种。在清单 12-3 的示例中，参数是 `命名的`，但它们也可以是位置参数（`$1`、`$2` 等）。有些参数可能被定义为 `OUT` 或 `INOUT`，而不是指定返回类型。同样，本章无意涵盖所有可能的规范，因为函数的性能并不依赖于所有这些规范变化。

最后，需要重点注意的是，函数可能不返回任何值；在这种情况下，函数被指定为 `RETURNS VOID`。这个选项之所以存在，是因为以前 PostgreSQL 不支持存储过程，所以将多条语句打包在一起的唯一方式就是在函数内部。

### 函数重载

与其他编程语言类似，PostgreSQL 中的函数可以是多态的，即多个函数可以使用相同的名称但具有不同的签名。这个特性被称为 `函数重载`。如前所述，函数由其名称和输入参数集定义；对于不同的输入参数集，返回类型可以不同，但出于显而易见的原因，两个函数不能同时共享相同的名称和相同的输入参数集。

让我们看一下清单 12-5 中的例子。在案例 #1 中，创建了一个计算特定航班乘客数量的函数。在案例 #2 中，创建了一个同名函数，用于计算特定日期从特定机场出发的乘客数量。

但是，如果您尝试运行代码片段 #3 来创建一个计算特定航班号在特定日期的乘客数量的函数，您将收到错误：

```
ERROR: cannot change name of input parameter "p_airport_code".
```

您可以创建一个名称相同但参数集和返回类型不同的函数；因此，您可以创建另一个同名函数，如案例 #4 所示。但是，如果您尝试创建一个名称相同、返回类型不同但参数相同的函数（案例 #5），数据库将抛出错误：

```
#1
CREATE OR REPLACE FUNCTION num_passengers(p_flight_id int) RETURNS integer;
#2
CREATE OR REPLACE FUNCTION num_passengers(p_airport_code text, p_departure date) RETURNS integer;
#3
CREATE OR REPLACE FUNCTION num_passengers(p_flight_no text, p_departure date) RETURNS integer;
#4
CREATE OR REPLACE FUNCTION num_passengers(p_flight_no text) RETURNS numeric;
#5
CREATE OR REPLACE FUNCTION num_passengers(p_flight_id int) RETURNS numeric;
清单 12-5
函数重载
```

```
ERROR: cannot change return type of existing function
```

请注意，这些函数的源代码有显著差异。清单 12-6 展示了 `num_passengers(integer)` 函数的源代码，清单 12-7 展示了 `num_passengers(text, date)` 函数的代码。

```
CREATE OR REPLACE FUNCTION num_passengers(p_flight_id int) RETURNS integer
AS
$$BEGIN
RETURN (
SELECT count(*)
FROM booking_leg bl
JOIN booking b USING (booking_id)
JOIN passenger p USING (booking_id)
WHERE flight_id=p_flight_id);
END;
$$ LANGUAGE plpgsql;
清单 12-6
num_passengers(int) 的源代码
```

```
CREATE OR REPLACE FUNCTION num_passengers(
p_airport_code text,
p_departure date) RETURNS integer
AS
$$BEGIN
RETURN (
SELECT count(*)
FROM booking_leg bl
JOIN booking b USING (booking_id)
JOIN passenger p USING (booking_id)
JOIN flight f USING (flight_id)
WHERE departure_airport=p_airport_code
AND scheduled_departure BETWEEN p_departure AND p_departure +1)
;
END;
$$ LANGUAGE plpgsql;
清单 12-7
num_passengers(text, date) 的源代码
```


## 函数执行

要执行一个函数，我们使用 `SELECT` 操作符。代码清单 12-8 展示了执行带有 `p_flight_id` 参数（设置为 13）的函数 `num_passengers` 的两种可能方式。

```
SELECT num_passengers(13);
SELECT * FROM num_passengers(13);
代码清单 12-8
函数执行
```

对于返回标量值的函数，两种语法都会产生相同的结果。复杂类型将在本章后面介绍。

同样值得注意的是，用户定义的标量函数可以像内置函数一样用在 `SELECT` 语句中。回顾代码清单 12-3 中的函数 `text_to_numeric`。你可能会疑惑，当 PostgreSQL 已经有三种将字符串转换为整数的方法时，为什么还需要创建用户定义的转换函数。记录一下，这三种方式是：

*   `CAST (text_value AS numeric)`
*   `text_value::numeric`（这是 `CAST` 的替代语法）
*   `to_number(text_value, '999999999999')`——使用内置函数

为什么需要自定义转换函数？对于上述列表中的任何方法，如果输入文本字符串包含数字以外的符号，尝试转换都会导致错误。

为了确保转换函数不会失败，我们在函数体中包含了 `异常处理部分`。该部分以 `EXCEPTION` 关键字开始；`WHEN` 关键字可以识别特定的异常类型。在本章中，我们将仅使用 `WHEN OTHERS` 形式，这意味着处理所有未包含在前面的 `WHEN` 条件中的异常类型。如果像代码清单 12-3 那样单独使用 `WHEN OTHERS`，则意味着所有异常都应以相同方式处理。

在代码清单 12-3 中，这意味着任何转换错误（或者实际上是任何错误）都不会导致函数失败，而是返回 `NULL`。为什么当传递“坏”参数时，函数不失败如此关键？因为这个函数被用在 `SELECT` 列表中。

在第 7 章中，我们创建了物态化视图 `passenger_passport`（参见代码清单 7-11）。该物态化视图的不同列应包含不同的数据类型，但由于在源数据中，所有这些字段都是文本字段，我们能做的很有限。现在，如果你想将 `passport_num` 作为数值类型选择，你的 `SELECT` 可能如下所示：

```
SELECT
passenger_id,
passport_num::numeric AS passport_number
FROM passenger_passport
```

如果在任何一个实例中，`passport_num` 列包含非数值（例如，空格或空字符串），那么整个 `SELECT` 语句将会失败。相反，我们可以使用自定义函数 `text_to_integer`：

```
SELECT
passenger_id,
text_to_numeric(passport_num) AS passport_number
FROM passenger_passport
```

让我们再创建一个用户定义函数 `text_to_date`，它将把包含日期的字符串转换为 `date` 类型——参见代码清单 12-9。

```
CREATE OR REPLACE FUNCTION text_to_date(input_text text)
RETURNS date AS
$BODY$
BEGIN
RETURN input_text::date;
EXCEPTION WHEN OTHERS THEN
RETURN null::date;
END;
$BODY$
LANGUAGE plpgsql;
代码清单 12-9
将文本转换为日期的函数
```

现在，我们可以在代码清单 12-10 中使用这两个函数。

```
SELECT
passenger_id,
text_to_integer(passport_num) AS passport_num,
text_to_date(passport_exp_date) AS passport_exp_date
FROM passenger_passport
代码清单 12-10
在 SELECT 列表中使用函数
```

尽管这个例子似乎是 PostgreSQL 中函数的完美用例，但实际上，就性能而言，这远非理想的解决方案——如下所述。

## 函数执行内部原理

本节解释了函数执行的一些细节，这些细节是 PostgreSQL 特有的。如果你之前有使用 Oracle 或 MS SQL Server 等 DBMS 的经验，你可能会对函数执行做出一些假设，而这些假设在 PostgreSQL 中并不成立。

第一个意外可能发生在你执行 `CREATE FUNCTION` 语句并收到类似以下的完成消息时：

```
CREATE FUNCTION
查询在 127 毫秒内成功返回。
```

读到这里，你可能会认为你的函数不包含任何错误。为了说明之后可能出错的地方，让我们编译代码清单 12-11 中的代码。如果你复制并执行此语句，你将收到创建成功消息。

```
CREATE OR REPLACE FUNCTION num_passengers(
p_airport_code text,
p_departure date) RETURNS integer
AS $$
BEGIN
RETURN (
SELECT
count(*)
FROM booking_leg bl
JOIN booking b USING (booking_id)
JOIN passenger p  USING (booking_id)
JOIN flight f USING (flight_id)
WHERE airport_code=p_airport_code
AND scheduled_departure BETWEEN p_date AND p_date +1);
END;
$$ LANGUAGE plpgsql;
代码清单 12-11
创建一个会顺利完成但包含错误的函数
```

然而，当你尝试执行这个函数

```
SELECT num_passengers('ORD', '2020-07-05')
```

……你将会收到错误消息：

```
错误：列 "airport_code" 不存在
```

哪里出错了？该函数使用了 `airport_code` 而不是 `departure_airport`。这是一个容易犯的错误，但你可能没想到，PostgreSQL 在你最初创建函数时永远不会告知你犯了这个错误。

现在，如果你更正这个错误并运行新的 `CREATE FUNCTION` 语句（参见代码清单 12-12），你将收到另一个错误：

```
CREATE OR REPLACE FUNCTION num_passengers(
p_airport_code text,
p_departure date) RETURNS integer
AS $$
BEGIN
RETURN (
SELECT
count(*)
FROM booking_leg bl
JOIN booking b USING (booking_id)
JOIN passenger p  USING (booking_id)
JOIN flight f USING (flight_id)
WHERE departure_airport =p_airport_code
AND scheduled_departure BETWEEN p_date AND p_date +1);
END;
$$ LANGUAGE plpgsql;
代码清单 12-12
创建一个函数：更正了一个错误，另一个错误仍然存在
```

```
错误：列 "p_date" 不存在
```

PostgreSQL 是对的，因为参数的名称是 `p_departure_date`，而不是 `p_date`。那么，为什么这个错误没有在之前报告呢？

在函数创建期间，PostgreSQL 仅执行初始的解析传递，在此期间仅检测到简单的语法错误。任何更深层次的错误要到执行时才会被发现。如果你刚从 Oracle 过来，并假设创建函数时它会被数据库引擎编译并以编译形式存储，这是个坏消息。函数不仅以源代码形式存储，而且更甚的是，与其他 DBMS 相比，函数是解释执行的，而不是编译执行的。

PL/pgSQL 解释器在第一次调用函数时（在每个会话内）解析函数的源文本并生成一个（内部的）指令树。即使到那时，函数中使用的单个 SQL 表达式和命令也不会立即被翻译。只有当执行路径到达特定命令时，才会分析它并创建一个 `预处理语句`。如果在同一会话中再次执行相同的函数，它将被重用。其含义之一是，如果你的函数包含一些条件代码（即 `IF THEN ELSE` 或 `CASE` 语句），如果这部分在执行期间未被触及，你甚至可能不会发现代码中的语法错误。我们见过这种令人不快的发现发生在函数投入生产很久之后。总之，当你创建一个 PL/pgSQL 函数时：

1.  不保存执行计划。
2.  不会检查表、列或其他函数的存在性。


