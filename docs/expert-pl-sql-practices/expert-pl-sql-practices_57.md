# 缩进

如果说有什么编程话题能引起争议，那很可能是如何缩进代码的问题。所有语言的程序员几十年来一直就此争论不休。COBOL 程序员可能最轻松。至少在他们的语言中，许多关键的缩进是由语言规范规定的。

以下部分讨论了缩写 PL/SQL 代码的特定方面。总体目标是便于阅读代码并直观地识别逻辑块。我选择使用两个空格。有一点：避免使用制表符（tab），因为不同的软件开发环境和打印机会以不同的方式解释它们。

### 通用缩进

使用两个空格表示一个代码块是另一个代码块的子块，如下所示：

```
BEGIN
  l_author := 'Pierre Boulle';
  IF p_book_id = 123456 THEN
    FOR l_counter IN 1..100 LOOP
      ...
    END LOOP;
  END IF;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    -- 处理此错误
    ...
END;
```

### Select 语句

每个查询列占一行，每个变量占一行，语句的每个元素占一行。这种方法会占用一些空间，但便于阅读和理解整个语句。参见以下示例中对齐各种项目的有用方法：

```
SELECT editor
      ,publication_date
  FROM books
WHERE book_id = 12345
   OR (book_title      = 'Planet of the Apes'
       AND book_author = 'Pierre Boulle'
      );
```

![images](img/square.jpg) **注意** 此示例还展示了如何对齐括号以清楚地显示其起始和结束位置。也请注意等号（或者当编码变成绘画时！）的微妙对齐。

此示例展示了如何使用表别名：

```
SELECT b1.book_title
      ,b1.book_id
      ,b2.book_id
  FROM books b1
      ,books b2
 WHERE b1.book_title = b2.book_title
   AND b1.book_id <> b2.book_id;
```

### Insert 语句

每个列占一行，每个值占一行。始终指明列名以避免歧义。此示例展示了如何对齐：

```
INSERT INTO books (
            book_id
           ,title
           ,author
           )
    VALUES (
            12345
           ,'Planet of the Apes'
           ,'Pierre Boulle'
           );
```

### Update 和 Delete 语句

它们看起来非常相似。每个列和值占一行，语句的每个元素占一行。

```
UPDATE books
   SET publication_year = 1963
WHERE book_id = 12345;
```

```
DELETE FROM books
 WHERE book_id = 12345;
```

### If 语句

每个元素占一行。为了更好地阅读，在 `END IF` 关键字后添加注释很有用，特别是当它与匹配的 `IF` 之间间隔了许多行时。以下是一系列 `IF` 块的示例：

```
IF l_var IS NULL THEN
  ...
ELSE
  IF l_var > 0 AND l_var < 100 THEN
    ...

  ELSE
    ...
  END IF; -- l_var 在 0 和 100 之间
END IF; -- l_var 为空
```

![images](img/square.jpg) **注意** 在 `IF` 块中，确保你始终有一个 `ELSE` 部分，以避免未处理的情况，除非你确信根本无需做任何事情。那么，写一个简短的注释是件好事，只是为了记住 `ELSE` 并非被纯粹地遗忘。未显式处理的情况可能会使你的应用程序行为异常且难以调试。

### Case 语句

每个元素占一行，如下所示：

```
CASE l_var
  WHEN value_1 THEN
    -- 解释
    ...
  WHEN value_2 THEN
    -- 解释
    ...
  ELSE
    -- 解释
    ...
END CASE;
```

### 游标

使用小写命名游标，并使用后缀 *_cursor*。注意关键字在左侧的对齐方式，如前面的示例所示，这提高了语句的可读性。

```
CURSOR books_cursor IS
  SELECT book_id
        ,title
    FROM books
   WHERE registration_date >= TO_DATE('1971-01-01', 'YYYY-MM-DD')
   ORDER BY title;
```

### For…loop 语句

使用 *l_* 前缀命名循环中使用的变量。当循环遍历游标时，使用游标的名称和后缀 *_rec*。这样在阅读代码时，更容易理解值的来源，如此示例所示：

```
FOR l_book_rec IN books_cursor LOOP
  FOR l_counter IN 1..100 LOOP
    ...
  END LOOP; -- 计数器
  ...
END LOOP; -- 书籍列表
```

### 字符串拼接

每个被拼接的元素占一行。这样更易于阅读和修改。由多个部分构建字符串如下所示：

```
l_text := 'Today, we are '||TO_DATE(SYSDATE, 'YYYY-MM-DD')
                          ||' and the time is '
                          ||TO_DATE(SYSDATE, 'HH24:MI')
                          ;
```



### 动态代码

动态 SQL 可能为 SQL 或语句注入敞开大门。使用编码模板有助于强制使用绑定变量，这是应对此类威胁的解决方案之一。下面的示例展示了通过拼接字符串（其中一个是最终用户提供的，例如作为参数）来构建并执行语句可能存在危险。

假设有一个在图书馆注册图书预约的模块。它可能包含一些动态代码，例如因为数据要插入的表取决于图书的类型。它可能如下所示：

```sql
l_statement := 'BEGIN'
             ||'  INSERT INTO book_reservations ('
             ||'              res_date'
             ||'             ,res_book_id'
             ||'              )'
             ||'      VALUES ('
             ||               SYSDATE
             ||'             ,'
             ||               p_book_id
             ||'             );'
             ||'END;'
             ;
EXECUTE IMMEDIATE l_statement;
```

系统只是要求用户输入请求的图书 ID。对于一个善意且规范的用户，生成的语句将如下所示，其中 "123456" 是给定的输入：

```sql
BEGIN
  INSERT INTO book_reservations (
              res_date
             ,res_book_id
              )

      VALUES (
              SYSDATE
             ,123456
              );
END;
```

但是，如果最终用户怀有恶意，他或她可以利用相同的输入注入恶意代码。输入 "123456); delete from books where (1=1"，生成的语句就变成了这样：

```sql
BEGIN
  INSERT INTO book_reservations (
              res_date
             ,res_book_id
              )
      VALUES (
              SYSDATE
             ,123456
              );
  DELETE FROM books where (1=1);
END;
```

那么要执行的操作就会由两条语句组成：一条无害的语句，后面跟着一条清空表的极其恶意的指令。

以下是使用绑定变量重写的代码。使用绑定变量可以防止任何用户输入的字符串被解释为语句本身的一部分。SQL 或语句注入的风险得以消除。

```sql
l_statement := 'BEGIN'
             ||'  INSERT INTO book_reservations ('
             ||'              res_date'
             ||'             ,res_book_id'
             ||'              )'
             ||'      VALUES (:p_date'
             ||'             ,:p_book_id'
             ||'             );'
             ||'END;'
             ;
EXECUTE IMMEDIATE l_statement
  USING IN SYSDATE
       ,IN p_requested_book_id
       ;
```

### 包

尽管可以在数据库中编译独立的存储过程和函数，但在一个庞大、扁平的列表中搜索单个模块很快会变成一场噩梦。相反，使用包（package）按语义字段或用途对过程和函数进行分组。扫描一个命名正确且很可能包含你所需内容的包，比扫描整个方案中所有函数和过程的列表要快得多。

将直接处理数据的 `内核` 包与负责 `用户界面` 的包分开。这不仅有助于管理和使用，还能让你精细调整执行权限，例如对接口包拥有广泛访问权限，对用作 API 的内核包拥有更受限的访问权限，而对应用程序的核心部分则完全没有访问权限。

使用大写字母（避免使用双引号包围对象名，使其区分大小写）和有意义的名称来命名你的包。使用前缀来指示包包含的模块类型。例如，你可以使用以下前缀：

> `KNL_` 用于包含内核模块的包。
> 
> `GUI_` 用于包含处理用户界面的模块的包。

你的包名可能如下所示：

> `PACKAGE KNL_BOOKS;` 用于包含处理图书的模块的包。
> 
> `PACKAGE GUI_LIBRARY;` 用于包含处理图书馆用户界面的模块的包。

### 存储过程

存储过程（独立的或在包中的）应被视为执行给定任务的黑盒子。术语“黑盒子”意味着它们应该能处理任何类型的输入（有效或无效的），然后完成其工作，或者在出错时使数据库保持与调用时相同的状态。通常你希望过程能报告在完成工作的过程中遇到的任何问题。

最好让一个过程专注于一项单一任务。这是应用程序良好模块化的基础。拥有多个可以当作基础构件来使用和重用的小模块，比拥有少数庞大而笨重、难以重用的大模块要好。

不同类型的任务可以分离如下：检索记录、检查信息、插入或修改数据，由内核或 API 包处理；为用户界面生成显示内容，由接口包处理。

接下来，我将向你展示一种根据存储过程所执行的任务类型为其命名的方法。这种过程命名策略，结合包分离命名策略，将确保源代码更易于理解。此外，它还可以作为良好模块化的指南。

#### 命名

使用大写字母和有意义的名称。使用一个表示过程类型的前缀（这里的 `$` 符号是一个装饰性元素，用于在视觉上与包名区分开，并将类型前缀与名称分开）。例如，以下是一些前缀示例：

> `CHK$` 用于检查数据的过程。
> 
> `GET$` 用于从数据库检索信息或进行计算的过程。
> 
> `EXE$` 用于插入、修改或删除数据的过程。
> 
> `DSP$` 用于在用户界面中显示信息的过程。

以下是一些使用上述前缀列表开发的实际过程名称：

> `PROCEDURE CHK$BOOK_AVAILABITLIY;` 用于检查图书可用性的过程。
> 
> `PROCEDURE GET$BOOK_AUTHOR;` 用于检索图书作者的过程。
> 
> `PROCEDURE EXE$STORE_BOOK;` 用于存储图书数据的过程。
> 
> `PROCEDURE DSP$BOOK_INFO;` 用于在用户界面中显示图书信息的过程。

同时，养成在声明结束的 `END` 之后添加一个重复过程名称的注释的习惯。这样做可以简化阅读包含长模块列表的包的任务，正如你在这里看到的：

```sql
PROCEDURE CHK$TITLE_LENGTH (
                      p_title                      IN          VARCHAR2
                     ) IS
BEGIN
  ...
END; -- CHK$TITLE_LENGTH
```



## 参数

使用小写字母并以 `p_` 为前缀来命名参数。这样可以更容易地将其与过程体内的局部变量区分开来。每个参数占用一行，并且名称、输入/输出属性、类型和默认赋值值之间保持固定的间距，如下所示：

```sql
PROCEDURE GET$BOOK_AUTHOR (
                      p_book_id              IN      NUMBER   := NULL
                    , p_book_title           IN      VARCHAR2 := NULL
                    , p_author               OUT     VARCHAR2
                    );
```

请注意，每个参数使用一行使得代码易于阅读。在必要时添加或删除一个参数也很容易，特别是当逗号位于每行的开头时。

![images](img/square.jpg) `注意` 我还建议您将默认参数值指定为 `NULL`。否则，很难确定某个参数值是实际传递的，还是保留了其默认值。如果一个参数的默认值是 `X`，并且在给定的过程调用中传递的值也确实是 `X`，您的代码将无法知道该值 `X` 是传递给了过程，还是参数使用了默认值。如果您确实需要一个非空的默认值，在使用该参数时，总会有时间使用一个好用的 `NVL` 函数。

## 调用

始终在过程名前加上其包名，即使在同一个包内调用过程也是如此。这有助于阅读和调试，并将代码片段重新定位到另一个包中变得轻而易举。

始终使用 `=>` 语法来传递参数值。每行列出一个参数值。`=>` 语法使调用在参数顺序方面清晰无误；每个参数接收到它预期接收的值。此外，该语法在调用重载模块（有些人称之为多态模块）时也避免了任何歧义。这是一个例子：

```sql
KNL_BOOKS.GET$BOOK_AUTHOR (
                      p_book_id    => 123456
                    , p_author     => l_author
                    );
```

## 局部变量

使用小写字母并以 `l_` 为前缀来命名局部变量。这样可以更容易地将其与过程体内的参数区分开来。每个声明占用一行，名称和类型之间保持固定的间距，如下所示：

```sql
l_author                                                VARCHAR2(50);
```

## 常量

使用大写字母并以 `C_` 为前缀来命名您的常量，如下所示：

```sql
C_TITLE_HEADER                                          CONSTANT VARCHAR2(50) := 'Title';
```

当某个值（字符串或数字）在代码中多次使用时，立即创建一个常量。换句话说，避免使用硬编码的值。使用常量代替多次出现的硬编码值可以避免拼写错误以及花费数小时试图理解模块行为不如预期的原因。例如，假设您想检查用户界面中是否按下了某个按钮。为按钮名称使用一个常量并用于检查被点击的内容，可以确保界面和应用程序行为之间的一致性。这是一个例子：

```sql
IF p_button = 'Ok' THEN -- 糟糕的做法，如果按钮的名称是 'OK'（大写）
```

```sql
C_OK_BUTTON                                        CONSTANT VARCHAR2(50) := 'OK';
IF p_button = C_OK_BUTTON THEN -- 好得多！
```

![images](img/square.jpg) `注意` 为常量字符串指定过短的长度（例如 `VARCHAR2(1) := 'Hello'`）是可能的，且不会触发编译错误。该错误只有在调用给定的常量时才会发生。调试这一点非常困难，因为问题的根源不在任何模块的主体中。因此，我总是为常量字符串声明一个足够长的长度，比如 50，即使对于非常短的值也是如此（`VARCHAR2(50) := 'Y'`）。

## 类型

使用大写字母并以 `T_` 为前缀来命名您的类型，如下所示：

```sql
TYPE T_SEARCH_RESULT IS TABLE OF NUMBER
     INDEX BY BINARY_INTEGER;
```

## 全局变量

使用小写字母并以 `g_` 为前缀来命名全局变量，如下例所示：

```sql
g_book_record                                          BOOKS%ROWTYPE;
```

## 局部过程和函数

可以声明属于给定过程局部的过程或函数。在其名称中使用大写字母并以 `LOCAL$` 为前缀。一个局部过程看起来像这样：

```sql
PROCEDURE KNL$REGISTER_AUTHOR (
                         p_name                       IN      VARCHAR2
                        ) IS

  PROCEDURE LOCAL$COMPUTE_CODE_NAME(
                         p_local_name                 IN      VARCHAR2
                        ) IS
  BEGIN
    ...
  END; -- LOCAL$COMPUTE_CODE_NAME

  l_code_name                                             VARCHAR2(80);

BEGIN
      l_code_name := LOCAL$COMPUTE_CODE_NAME(
                         p_local_name => p_name
                        );
  ...
    END; -- KNL$REGISTER_AUTHOR
```

## 过程元数据

即使您使用软件版本管理工具（如 Subversion），在源代码中保留一些日期和姓名也可能是有用的。其他代码内文档也很有帮助。我建议在每个模块声明之前添加一个注释块，如下所示。记录谁进行了什么更改以及出于什么原因，可以在修改模块时提供帮助。它也作为系统技术文档的一部分。

```sql
/*-----------------------------------------------------------------------*/
/*                                                                       */
/* 模块     : MODULE_NAME                                                */
/* 目标     : 模块的简要描述。                                           */
/* 关键词   : 描述模块功能的一些关键词。                                 */
/*                                                                       */
/*-----------------------------------------------------------------------*/
/* 描述:                                                                 */
/*                                                                       */
/* 模块的详细描述：其目标。                                              */
/* 关于参数（输入和输出）的解释。                                        */
/* 过程如何工作，“技巧”等。                                              */
/*                                                                       */
/*-----------------------------------------------------------------------*/
/* 历史:                                                                 */
/*                                                                       */
/* YYYY-MM-DD : 名和姓 - 创建。                                         */
/*                                                                       */
/* YYYY-MM-DD : 名和姓 - 审查。                                         */
/*                                                                       */
/* YYYY-MM-DD : 名和姓                                                  */
/*              修改描述。                                               */
/*                                                                       */
/*-----------------------------------------------------------------------*/
```

## 函数

我的函数编码约定与过程相同。只有命名方式不同。我使用前缀 `STF$`（代表存储函数）来命名我的函数，如下所示：

```sql
FUNCTION STF$IS_BOOK_AVAILABLE (
                      p_book_id              IN      NUMBER
                    ) RETURN BOOLEAN;
```

函数通常设计为返回单个值。定义一个表示出错的常量可能很有用。请注意，函数也可以有 `OUT` 参数，但我认为这混淆了函数与过程的角色。因此，我通常将函数的使用限制在非常安全的过程或绝对需要它们的情况。


### 错误处理

健壮、可维护且可扩展的软件的关键要素之一是模块化和避免逻辑重复。一个强大的系统依赖于一系列基本的构件，每个构件都具有独特的功能。将这些构件组合在一起，便能实现更复杂的功能。如果一个模块失败，它不应导致调用它的模块崩溃。因此，错误处理的第一步是创建一个隔离舱，旨在防止任何损害扩散到失败的程序之外。第二步是向调用者报告此错误，以便其能被妥善处理并安全地传播到最高层级。第三步是确保在错误发生后系统状态保持一致；换句话说，就是没有操作被部分执行。

#### 错误捕获

PL/SQL 拥有一个强大的错误捕获机制，其形式为 `异常`。其中适用范围最广的是 `OTHERS`。正如其名，它旨在捕获所有未被其他方式捕获的错误。我强烈建议使用针对 `OTHERS` 的异常处理器来保护所有存储过程。这样做有机会记录或输出任何有价值的调试信息，例如哪个模块在哪些参数值下失败。这虽不能阻止应用程序崩溃，但你将知晓问题发生的位置和方式。更多细节请参见“错误报告”部分。

如果需要添加用户自定义异常，请使用大写字母、有意义的名称以及前缀 `L_*`。如果你愿意，也可以使用不同的前缀。我个人的约定是使用 `L_*`。

以下是一个展示我典型错误处理逻辑结构的框架过程：

```
PROCEDURE GET$BOOK_AUTHOR (
                   p_book_id              IN      NUMBER   := NULL
                  ,p_book_title           IN      VARCHAR2 := NULL
                  ,p_author                  OUT  VARCHAR2
                  ) IS

  L_MY_EXCEPTION                                EXCEPTION;

BEGIN
   ...
EXCEPTION
  WHEN L_MY_EXCEPTION THEN
    -- 特定错误的处理
    ...
  WHEN OTHERS THEN
    -- 通用错误的处理
    ...
END; -- GET$BOOK_AUTHOR
```

此外，虽然这本身不是编码约定，但请务必考虑检查所有输入参数。至少要确保不要完全信任它们。仅仅因为一个模块有一个名为 `p_book_id` 的参数，并不意味着传递给该参数的值总是一个有效的书籍 ID。

#### 错误报告

一旦你建立了一个模块系统，你就会希望在其中一个模块失败时能够知晓，以便正确处理问题。传输错误信息对于构建稳定的架构至关重要。

错误信息可以有两个方面：一个面向机器，一个面向人类。面向机器的错误信息基于易于解释的代码，例如 `(ORA-)01403` 代表“未找到数据”。面向人类的错误信息则基于解释特定错误的明文。我建议在特定表中创建你自己的错误集（代码和解释）。让你的错误特定于你的应用程序。例如，创建一条诸如“没有此标题的书籍”的消息，并将其与你选择的代码关联。这样的错误列表对程序员和最终用户都很有用。确保你的消息清晰且友好。

为了将错误从一个模块传播到另一个模块，我使用两个名为 `p_exitcode` 和 `p_exittext` 的输出参数。第一个用于机器；第二个用于人类。我确保在我编写的每一个存储过程中都存在这些参数。按照约定，退出代码等于 `0` 表示过程执行顺利。不同于 `0` 的代码表示发生了错误，其中的代码数字指示了具体是哪种错误。

一旦检测到问题就立即停止执行；继续下去不值得。你可能想使用一个名为 `L_PB_FATAL` 的通用异常（代表 *Local Problem Fatal*，但你可以选择任何戏剧性的名称），只要没有理由继续，就引发此异常。

以下是上一节中过程框架的增强版本。此版本的框架实现了我刚才描述的附加参数以及异常。

```
PROCEDURE GET$BOOK_AUTHOR (
                   p_book_id              IN      NUMBER   := NULL
                  ,p_book_title           IN      VARCHAR2 := NULL
                  ,p_author                  OUT  VARCHAR2
                  ,p_exitcode                 OUT  NUMBER
                  ,p_exittext                 OUT  VARCHAR2
                  ) IS

  L_PB_FATAL                                  EXCEPTION;

BEGIN
  -- 初始化
  p_exitcode := 0;
  p_exittext := NULL;
  ...
EXCEPTION
  WHEN L_PB_FATAL THEN
    IF p_exitcode = 0 THEN -- 避免遗留值
      p_exitcode := -1;    -- 返回非零值表示错误
    END IF;
  WHEN OTHERS THEN
    p_exitcode := SQLCODE;
    p_exittext := SUBSTR('Error in GET$BOOK_AUTHOR: '||SQLERRM, 1, 500);
END; -- GET$BOOK_AUTHOR
```

任何调用遵循此框架所示约定的程序，都应随后检查退出代码，并在出现问题时采取适当行动。然后，你可以决定是采取适当的紧急措施，还是停止执行并使用相同机制传播错误，如下所示：

```
PROCEDURE GET$BOOK_DATA (
                   p_book_id              IN      NUMBER
                  ,p_book_author              OUT  VARCHAR2
                  ,p_book_editor              OUT  VARCHAR2
                  ,p_exitcode                 OUT  NUMBER
                  ,p_exittext                 OUT  VARCHAR2
                  ) IS

  L_PB_FATAL                                  EXCEPTION;

BEGIN
  -- 初始化
  p_exitcode := 0;
  p_exittext := NULL;

  -- 获取书籍的作者
  KNL_BOOKS.GET$BOOK_AUTHOR (
                   p_book_id  => p_book_id
                  ,p_author   => p_book_author
                  ,p_exitcode => p_exitcode  -- 错误代码和文本的值
                  ,p_exittext => p_exittext  -- 传播给调用者
                  );
  IF p_exitcode <> 0 THEN
    RAISE L_PB_FATAL;
  END IF;

  -- 获取书籍的编辑者
  KNL_BOOKS.GET$BOOK_EDITOR (
  ...    
EXCEPTION
  WHEN L_PB_FATAL THEN
    IF p_exitcode = 0 THEN -- 避免遗留值
      p_exitcode := -1;    -- 返回非零值表示错误
    END IF;
  WHEN OTHERS THEN
    p_exitcode := SQLCODE;
    p_exittext := SUBSTR('Error in GET$BOOK_DATA: '||SQLERRM, 1, 500);
END; -- GET$BOOK_DATA
```

#### 错误恢复

遵循黑盒范式，一个执行插入、更新或删除信息的操作，应使数据库保持在一致状态。如果该操作失败，则必须回滚到进入该操作时系统所处的状态。请使用大写字母为保存点命名，并确保在操作的全局异常处理中包含回滚调用，如下所示：

```sql
PROCEDURE EXE$REGISTER_NEW_BOOK (
                     p_title                IN      VARCHAR2
                    ,p_author               IN      VARCHAR2
                    ,p_book_id                  OUT  NUMBER
                    ,p_exitcode                 OUT  NUMBER
                    ,p_exittext                 OUT  VARCHAR2
                    ) IS

  L_PB_FATAL                                    EXCEPTION;

BEGIN
  -- 初始化
  p_exitcode := 0;
  p_exittext := NULL;

  SAVEPOINT BEFORE_REGISTER_NEW_BOOK;

  IF p_title IS NULL OR p_author IS NULL THEN
    p_exitcode := 12345;  -- 无效新书数据的错误代码
    p_exittext := '缺少书名或作者信息';
    RAISE L_PB_FATAL;
  END IF;

  ...

EXCEPTION
  WHEN L_PB_FATAL THEN
    ROLLBACK TO BEFORE_REGISTER_NEW_BOOK;
    IF p_exitcode = 0 THEN -- 为避免值被遗漏
      p_exitcode := -1;
    END IF;
  WHEN OTHERS THEN
    ROLLBACK TO BEFORE_REGISTER_NEW_BOOK;
    p_exitcode := SQLCODE;
    p_exittext := SUBSTR('KNL_BOOKS.EXE$REGISTER_NEW_BOOK 中出错: '||SQLERRM, 1, 500);
END; -- EXE$REGISTER_NEW_BOOK
```

#### 测试优先，显示其次

如果你正在编写一个驱动用户界面的模块，最好先检查并获取所有必需的数据，然后再显示界面。这样，所有可能的错误在显示任何内容之前就被捕获了。你便可以轻松地避免在页面或屏幕中间出现“发生错误”之类的提示让用户看到。此外，如果你确实需要显示错误信息，你也可以在不使用户感到困惑的情况下进行。如果你开始显示数据，却中途被一个错误打断，用户可能会变得慌乱和沮丧。

### 总结

编码规范并非旨在解决你在实现系统或维护软件时可能遇到的所有问题。然而，为了标准化大型源代码库以及简化模块的维护和共享，它们非常方便。遵循编码标准是对你作为同行的程序员和项目负责人的尊重。作为一名程序员，你可能在开始时觉得使用它们很困难，因为它们可能与你自己的风格不匹配。相信我；只需要很短的时间，给定的一套标准就会成为你自然的编码方式。此外，一旦你开始管理大量模块，能够以一致的方式“即插即用”它们来构建复杂的机制，是多么令人愉快！作为一名项目负责人，你可能会发现将标准强加给整个团队很困难，但质量上的提升是值得的。

以下是一个最终模板。它将许多本章描述的规范总结在一个地方。

```sql
/*-----------------------------------------------------------------------*/
/*                                                                       */
/* 模块     : MODULE_NAME                                                */
/* 目标     : 模块的简短描述。                                           */
/* 关键词   : 描述模块功能的几个关键词。                                 */
/*                                                                       */
/*-----------------------------------------------------------------------*/
/* 描述:                                                                 */
/*                                                                       */
/* 模块的长描述：其目标。                                                */
/* 关于参数的解释（输入和输出）。                                        */
/* 操作如何工作，“技巧”等。                                              */
/*                                                                       */
/*-----------------------------------------------------------------------*/
/* 历史:                                                                 */
/*                                                                       */
/* YYYY-MM-DD : 名和姓 - 创建。                                          */
/*                                                                       */
/* YYYY-MM-DD : 名和姓 - 审查。                                          */
/*                                                                       */
/* YYYY-MM-DD : 名和姓                                                   */
/*              修改描述。                                                */
/*                                                                       */
/*-----------------------------------------------------------------------*/
PROCEDURE TYPE$MODULE_NAME(
                     p_param1              IN      VARCHAR2 -- 此处为 p_param1 添加注释
                    ,p_param2              IN      NUMBER   := NULL
                    ,p_exitcode               OUT  NUMBER
                    ,p_exittext               OUT  VARCHAR2
                    ) IS

-- 声明部分

CURSOR one_cursor IS
  SELECT *
    FROM table
   WHERE condition;

  L_PB_FATAL                                    EXCEPTION;
  l_variable                                    VARCHAR2(1);

BEGIN
  -- 初始化
  p_exitcode := 0;
  p_exittext := NULL;

  SAVEPOINT BEFORE_TYPE$MODULE_NAME;

  -- 检查输入
  PACKAGE.CHK$PARAM1 (
                     p_value    => p_param1
                    ,p_exitcode => p_exitcode
                    ,p_exittext => p_exittext
                    );
  IF p_exitcode <> 0 THEN     -- 原文为 p_exittext，此处根据上下文逻辑调整为 p_exitcode
     RAISE L_PB_FATAL;
  END IF;

  -- 收集所需数据
  BEGIN
    SELECT col
      INTO l_variable
      FROM table
     WHERE condition1
       AND condition2;
  EXCEPTION
     WHEN OTHERS THEN
       p_exitcode := 123456;
       p_exittext := '在 TYPE$MODULE_NAME 中获取信息时出错';
       RAISE L_PB_FATAL;
  END;

  -- 从游标获取数据
  FOR l_one_rec IN one_cursor LOOP    -- 原文为 FROM，应为 IN
    IF l_one_rec.col <> PACKAGE.C_CONSTANT THEN
      -- 关于上述测试的解释
      ...
    ELSE
      ...
    END IF;
  END LOOP; -- One cursor

  -- 显示
  ...

EXCEPTION
  WHEN L_PB_FATAL THEN
    ROLLBACK TO BEFORE_TYPE$MODULE_NAME;
    IF p_exitcode = 0 THEN          -- 为避免遗漏退出代码
      p_exitcode := -1;             -- 返回非 0 值表示错误
    END IF;
  WHEN OTHERS THEN
    ROLLBACK TO BEFORE_TYPE$MODULE_NAME;
    p_exitcode := SQLCODE;
    p_exittext := SUBSTR('PACKAGE.TYPE$MODULE_NAME 中出错: '||SQLERRM, 1, 500);
END TYPE$MODULE_NAME;
```

## 第 15 章

## 依赖关系与失效机制

**作者：Arup Nanda**

PL/SQL 包之间的依赖关系常常是应用程序错误的棘手来源。不熟悉依赖关系工作原理的数据库管理员和开发人员，可能会对一些偶发且无法复现、看似无因的错误感到困惑不解。例如，在执行你负责的一个包中的过程时，应用程序抛出以下错误：

`ORA-04068: 包的现有状态已被丢弃`

这个特定的过程之前已经成功执行过无数次。你完全可以发誓保证，自己很久没修改过这个包，最近几秒钟内肯定没有，但应用程序却报了这个错，客户订单失败，公司因此蒙受损失。

找不到明显的原因，你便责怪同事乱动了这个包，引发了一场激烈的争吵，事后立刻感到后悔。你的经理介入，冷静地让你给她重现错误。你手动重新执行了包中的过程，然后（惊喜，惊喜）它竟然毫无错误地执行成功了！在被你指责的同事那怒目而视的目光，以及经理那“你需要帮助”的眼神注视下，你此刻对究竟发生了什么感到彻底困惑。是蓄意破坏，Oracle Bug，运气不佳，还是都不是？

这个错误是什么？为什么你没做什么它自己就好了？问题并非源于玄学，而在于 PL/SQL 代码的依赖关系工作机制。包中引用的某个对象被修改了，导致该包失效。随后包被重新验证（或重新编译），并且必须重新初始化。

在开发应用程序时，理解依赖链和失效原因至关重要。前述情景就是一个设计不佳的应用程序代码如何导致服务中断的实例。本章将解释依赖关系，以及如何通过编码来减少失效，从而在运行时减少甚至可能消除对应用程序的干扰。

### 依赖链

首先，你必须理解依赖链及其对 PL/SQL 代码的影响。假设你有一个表 `ORDERS` 和一个用于更新 `QUANTITY` 列的过程 `UPD_QTY`。它们的定义如下（注意过程对表的依赖；这两个对象形成了一个依赖链）：

```sql
SQL> desc orders
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 ORDER_ID                                  NOT NULL NUMBER(5)
 CUST_ID                                   NOT NULL NUMBER(10)
 QUANTITY                                           NUMBER(10,2)

create or replace procedure upd_qty (
        p_order_id in orders.order_id%type,
        p_quantity in orders.quantity%type
) as
begin
     update orders
     set quantity = p_quantity
     where order_id = p_order_id;
end;
```

如果你删除了 `QUANTITY` 列会怎样？当然，这个过程将失去意义，因此 Oracle 合理地将其标记为无效。之后，你可能进行了另一项修改，要么将该列添加回表中，要么从过程中移除该列并替换为其他内容，使得该过程在语法上重新有效。如果你将列添加回了表中，可以通过执行以下命令重新编译该过程：

```sql
SQL> alter procedure upd_qty compile reuse settings;
```

![images](img/square.jpg) **提示** 重新编译时的 `REUSE SETTINGS` 子句是一个便捷且强大的工具。之前为过程设置的任何特殊编译时参数，如 `plsql_optimize_level` 或 `warning_level`，在重新编译后将保持不变。如果不使用该子句，则当前会话的设置将会生效。因此，除非你想更改这些设置，否则每次使用该子句都是一个好主意。

编译将检查过程中引用的所有组件是否存在。这种有效性确认仅在编译过程时进行——而不是在运行时。为什么？想象一下典型的存储代码库内容：长达数千行，包含数百个表、列、同义词、视图、序列和其他存储代码，如果在每次执行时都尝试验证每个对象，性能将会非常糟糕，使得此过程不可行。因此，验证在编译时完成，这很可能是一次性的操作。当一个对象（比如这个表）发生变化时，Oracle 无法立即确定变更是否会影响依赖对象（比如这个过程），因此它通过将该对象标记为失效来做出反应。对象之间的依赖关系在数据字典视图 `DBA_DEPENDENCIES`（或其对应视图 `USER_DEPENDENCIES`）中清晰可见。

```sql
SQL> desc dba_dependencies
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 OWNER                                     NOT NULL VARCHAR2(30)
 NAME                                      NOT NULL VARCHAR2(30)
 TYPE                                               VARCHAR2(18)
 REFERENCED_OWNER                                   VARCHAR2(30)
 REFERENCED_NAME                                    VARCHAR2(64)
 REFERENCED_TYPE                                    VARCHAR2(18)
 REFERENCED_LINK_NAME                               VARCHAR2(128)
 DEPENDENCY_TYPE                                    VARCHAR2(4)
```

`REFERENCED_NAME` 显示被引用的对象——当前对象所依赖的另一个对象。以下针对该表的查询列出了 `UPD_QTY` 的依赖项。每个这样的依赖项都代表 `UPD_QTY` 所依赖的一个对象。

```sql
select referenced_owner, referenced_name, referenced_type
from dba_dependencies
where owner = 'ARUP'
and name = 'UPD_QTY'
/
```


`REFERENCED_OWNER               REFERENCED_NAME                REFERENCED_TYPE`
`------------------------------ ------------------------------ ------------------`
`SYS                            SYS_STUB_FOR_PURITY_ANALYSIS   PACKAGE`
`ARUP                           ORDERS                         TABLE`

此输出显示过程 `UPD_QTY` 依赖于表 `ORDERS`，这符合我们的预期。如果该过程代码中引用了其他对象，它们也会在此列出。

![images](img/square.jpg) `注意` 有关此特定依赖关系的解释，请参见即将出现的侧栏“`SYS_STUB_FOR_PARITY_ANALYSIS` 是什么？”。

每当表 `ORDERS` 的结构发生改变时，过程的有效性就变得存疑，Oracle 会将其标记为无效。然而，存疑并不一定意味着无效。例如，当你向表中添加一个列时，这从技术上讲是对表的更改。这会使过程无效吗？

```sql
SQL> alter table orders add (store_id number(10));
```

```
Table altered.
```

如果你检查 `UPD_QTY` 过程的状态，是的，它会无效。

```sql
SQL> select status from user_objects where object_name = 'UPD_QTY';
```

```
STATUS
-------
INVALID
```

但该过程实际上并未引用新列，因此本不应受到影响。然而，Oracle 不一定知道这一点。为了安全起见，它会抢先将该过程标记为无效。此时，你可以重新编译该过程；它会编译成功。否则，下次调用该过程时，Oracle 会自动重新编译它。

让我们看看新的过程如何运作，包括自动重新编译。

```sql
SQL> exec upd_qty (1,1)
```

```
PL/SQL procedure successfully completed.
```

该过程执行成功，尽管它被标记为无效。这是如何做到的？它之所以能成功执行，是因为执行操作强制了过程的重新编译（和重新验证）。重新编译之所以成功，是因为该过程实际上并未引用底层表中已更改的组件——新列 `STORE_ID`。该过程编译成功并被标记为有效。你可以通过再次检查状态来确认这一点。

```sql
SQL> select status from user_objects
  2  where object_name = 'UPD_QTY';
```

```
STATUS
-------
VALID
```

到目前为止，这些事件似乎相当无害；这些强制重新编译处理了任何不影响程序有效性的更改，一切照常运行。你为什么应该关心？你应该非常关心。

### EXECUTED IMMEDIATE 是一种特殊情况

如果一个表是在 `execute immediate` 字符串内被引用的，那么依赖性检查不会捕获该表作为引用对象。例如，考虑以下代码：

```sql
create or replace procedure upd_qty as
begin
   execute immediate 'update orders set quantity = ...';
end;
```

在这种情况下，`ORDERS` 表将不会显示为 `UP_QTY` 的引用对象，这是不使用动态 SQL 的又一个理由（或者，根据你的观点，也是使用它的理由）。

存储代码的编译（以及依赖于它的其他对象，例如基于表的视图）会强制数据库中进行另一个称为解析的过程。解析类似于源代码的编译。当你执行一条 SQL 语句，例如 `select order_id from orders` 时，Oracle 不能直接执行该语句。它必须询问很多问题，其中一些是：

*   `orders` 是什么？是表、视图还是同义词？
*   表中是否确实存在名为 `order_id` 的*列*？
*   是否存在名为 `order_id` 的*函数*？
*   你是否有从该表（或函数）进行选择的*权限*？

在确保你具备所有必要的先决条件后，Oracle 会生成该 SQL 语句的可执行版本，并将其放入库缓存中。这个过程被称为*解析*。随后对 SQL 的执行将仅使用已解析的版本，而不再经历解析过程。在执行阶段，Oracle 必须确保组件（表 `ORDERS`、过程 `UPD_QTY` 等）没有被更改。它通过在这些对象以及其他结构（如游标）上放置一个*解析锁*来实现这一点。

解析非常消耗 CPU。频繁的解析和重新编译会占用服务器的 CPU 周期，剥夺其他有用进程应得的 CPU 资源。你可以通过检查一个名为 `v$mystat` 的视图中与解析相关的统计信息来审视其影响，该视图显示当前会话的统计信息。该视图仅以 `statistic#` 形式显示统计信息；它们的名称在另一个名为 `v$statname` 的视图中提供。以下是一个连接两个视图并显示解析相关统计信息的查询。将该查询保存为 `ps.sql` 以检查重新解析和重新编译的影响。

```sql
select name, value
from v$mystat s, v$statname n
where s.statistic# = n.statistic#
and n.name like '%parse%';
```

视图 `v$mystat` 是一个累积视图，因此其中的数值不显示当前数据；它们是累加的。要了解某段时间内的统计信息，你必须从该视图中选择两次并获取差值。要找出解析相关的统计信息，首先通过运行 `ps.sql` 收集统计信息，然后执行过程 `upd_qty()`，再次运行 `ps.sql` 收集统计信息，如下所示：

```sql
SQL> @ps
```

```
NAME                                VALUE
------------------------------ ----------
parse time cpu                           20
parse time elapsed                       54
parse count (total)                     206
parse count (hard)                       16
parse count (failures)                    1
```

```sql
SQL> exec upd_qty(1,1)
```

```
PL/SQL procedure successfully completed.
```

```sql
SQL> @ps
```

```
NAME                                VALUE
------------------------------ ----------
parse time cpu                           20
parse time elapsed                       54
parse count (total)                     208
parse count (hard)                       16
parse count (failures)                    1
```

注意这些数值。没有任何增加；换句话说，执行过程 `upd_qty` 并未导致解析。在一个*不同的*会话中，通过添加列来更改表，如下所示：

```sql
SQL> alter table orders add (amount number(10,2));
```

```
Table altered.
```

现在，在第一个会话中执行该过程并收集统计信息。

```sql
SQL> exec upd_qty(1,1)
```

```
PL/SQL procedure successfully completed.
```

```sql
SQL> @ps
```

```
NAME                                VALUE
------------------------------ ----------
parse time cpu                           26
parse time elapsed                       60
parse count (total)                     279
parse count (hard)                       19
parse count (failures)                    1
```

注意解析相关统计信息是如何增加的。硬解析（意味着 SQL 语句未在库缓存中找到，或者找到的任何版本不可用，因此必须重新解析）从 16 跳升到 19——表明发生了 3 次硬解析。总解析次数从 208 跳升到 279，增加了 71。由于有 3 次硬解析，另外 68 次是软解析，这意味着在库缓存中找到了结构并进行了重新验证，而不是从头开始解析。解析的 CPU 时间从 20 跳升到 26，表明解析消耗了 6 厘秒的 CPU 时间。如果没有解析的需求，这些 CPU 周期本可以用于执行其他有用的工作。这听起来可能不多，但考虑一下在正常的数据库系统中发生成千上万次此类情况，导致大量 CPU 消耗的场景。


