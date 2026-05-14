# 自定义聚合函数 stragg

早在 Oracle 11.2 版本问世之前，人们在著名的 Tom Kyte 的 [*http://asktom.oracle.com*](http://asktom.oracle.com) 网站上经常提出的一个问题是如何进行字符串聚合。于是 Tom 开发了一个名为 `stragg` 的自定义聚合函数作为答案，多年来被许多人使用。在此，我将展示一个我在此基础上添加了若干改进的版本。

#### 注意事项

您可能会在数据库的 `SYS` 模式中发现一个名为 `stragg` 的函数。这是一个基于 C 库、随 `dbms_xmlindex` 包一起安装的、鲜为人知的函数。它没有文档记录，专为 XML 索引实现中的某些特定任务而设计。**请勿使用它！** 无法保证它的工作方式，并且很容易在不知情的情况下以不受支持的方式调用它，从而导致错误或错误结果。

Oracle 数据 cartridges 接口 (`ODCI`) 是一组用于实现相当底层功能的接口函数，这些功能的使用方式与内置函数非常相似。它主要由库作者（例如使用 C 语言）实现特殊功能时使用，但也可以用于在纯 `PL/SQL` 中实现的更简单的情况。

由于本书主要是关于 `SQL` 的，我不会浪费纸张将整个实现打印在书中。因此，我将分段展示如代码清单 10-6 所示的创建语句，但跳过主体的大部分内容。

```
SQL> create or replace type stragg_expr_type as object (
2     element    varchar2(4000 char)
3   , delimiter  varchar2(4000 char)
4   , map member function map_func return varchar2
5  );
6  /
Listing 10-6
用于实现自定义聚合的类型、类型体和函数
```

Tom Kyte 最初的 `stragg` 函数只是简单地聚合一个 `varchar2`，并且硬编码了所使用的分隔符，因为聚合函数无法创建多个参数。我改为聚合一个对象类型 `stragg_expr_type`，允许我将所需的分隔符作为对象的第二个属性传递。

```
SQL> create or replace type body stragg_expr_type
2  as
3     map member function map_func return varchar2
4     is
5     begin
6        return element || '|' || delimiter;
7     end map_func;
8  end;
9  /
```

我在对象类型中实现了一个 `map member function`，因为它允许数据库判断两个对象是否相同。这反过来使得我的聚合函数能够支持 `distinct` 关键字，而这正是 `listagg` 直到 19c 版本才具备的功能之一。

```
SQL> create or replace type stragg_type as object
2  (
3     aggregated varchar2(4000)
4   , delimiter  varchar2(4000)

6   , static function ODCIAggregateInitialize(
7        new_self    in out stragg_type
8     ) return number

10   , member function ODCIAggregateIterate(
11        self        in out stragg_type
12      , value       in     stragg_expr_type
13     ) return number

15   , member function ODCIAggregateTerminate(
16        self        in     stragg_type
17      , returnvalue out    varchar2
18      , flags       in     number
19     ) return number

21   , member function ODCIAggregateMerge(
22        self        in out stragg_type
23      , other_self  in     stragg_type
24     ) return number
25  );
26  /
```

然后我定义了将要实现实际聚合的类型 `stragg_type`。我使用的两个属性用于内部实现。这四个函数由 `ODCI` 接口规定，必须按所示命名，并且参数列表必须完全一致（参数名称可以不同，但参数的顺序和类型必须匹配）：

*   `ODCIAggregateInitialize` 类似于构造函数，在聚合开始时由数据库显式调用，因此我在此创建该对象的一个新实例。
*   `ODCIAggregateIterate` 由数据库对每个要聚合的字符串调用，因此我在此将分隔符和字符串添加到 `aggregated` 属性中。（在原始的 `stragg` 中，`value` 参数只是一个 `varchar2`；这里我传递的是一个 `stragg_expr_type` 类型的值。）
*   `ODCIAggregateTerminate` 在聚合结束时由数据库调用，当它需要结果时，我在此返回聚合后的字符串。
*   如果数据库决定将聚合工作拆分成多个部分（例如，在并行查询中），每个部分都会调用 `ODCIAggregateInitialize` 来获取一个对象，然后通过 `ODCIAggregateIterate` 进行聚合。最后，每个部分都将拥有一个对象，其中一些字符串聚合在 `aggregated` 属性中——数据库随后将调用 `ODCIAggregateMerge` 来合并内容，因此在此函数中，我将 `other_self` 对象的 `aggregated` 内容附加到 `self` 对象中。

以上是对函数中需要实现的内容的文字描述，然后我只需要在类型体中编写代码即可。

```
SQL> create or replace type body stragg_type
2  is
...
54  end;
55  /
```

关于在类型体中实现这四个函数的代码，请参见配套脚本 *practical_fill_schema.sql*。

```
SQL> create or replace function stragg(input stragg_expr_type )
2     return varchar2
3     parallel_enable aggregate using stragg_type;
4  /
```

创建了用于实现的对象类型后，最后要做的就是创建聚合函数 `stragg` 本身。输入参数的数据类型必须与 `ODCIAggregateIterate` 函数的 `value` 参数匹配，返回数据类型必须与 `ODCIAggregateTerminate` 函数的 `returnvalue` 参数匹配。

`aggregate using stragg_type` 告诉数据库这是一个由对象类型 `stragg_type` 实现的自定义聚合函数，因此当数据库使用此函数执行聚合时，它将调用该类型的 `ODCI∗` 函数。关键字 `parallel_enable` 指定数据库可以使用并行化，因为我已实现了 `ODCIAggregateMerge`。

创建了这些对象后，我现在可以在代码清单 10-7 中使用我的自定义聚合函数。

```
SQL> select
2     max(brewery_name) as brewery_name
3   , stragg(
4        stragg_expr_type(product_name, ',')
5     ) as product_list
6  from brewery_products
7  group by brewery_id
8  order by brewery_id;
Listing 10-7
使用 stragg 自定义聚合函数
```

由于我使用 `ODCI` 接口将此函数声明为聚合函数，我可以在第 3-5 行像使用任何内置聚合函数一样使用 `stragg`。输入数据类型是 `stragg_expr_type`，因此我使用类型构造函数，传入产品名称和逗号作为分隔符。

#### 注意

使用对象类型向聚合函数传递分隔符的技巧很有效，但这确实需要我作为开发人员有一点自律精神，因为我需要确保分隔符是一个常量。原则上，我可以在每一行传递不同的分隔符值，但这会在实现中导致问题。我已尝试实现为使用首次调用 `ODCITableIterate` 时传入的分隔符，但在并行化的情况下，会有多次来自不同行的 `ODCITableIterate` 调用。因此，确保分隔符值是常量非常重要——最安全的方式是使用字面量。

代码清单 10-7 的输出与我从 `listagg` 和 `collect` 获得的输出几乎相同，但不一定完全一样：

```
BREWERY_NAME        PRODUCT_LIST
Balthazar Brauerei  Monks and Nuns,Der Helle Kumpel,Hercule Trippel
Happy Hoppy Hippo   Hazy Pink Cloud,Ghost of Hops,Summer in India
Brewing Barbarian   Coalminers Sweat,Pale Rider Rides,Hoppy Crude Oil,Reindeer Fuel
```


`product_list` 列中的产品名称是相同的——其值结果完全一致。但使用此自定义聚合函数`stragg`时，分隔符字符串中产品的顺序是不确定的——我无法为其实施 `order by` 子句。

这里需要注意的是，如果我在调用 `stragg` 时添加 `distinct` 子句，其行为会有所不同：

```
...
4        distinct stragg_expr_type(product_name, ',')
...
```

突然之间，产品列表中的啤酒按字母顺序排序了：

```
BREWERY_NAME        PRODUCT_LIST
Balthazar Brauerei  Der Helle Kumpel,Hercule Trippel,Monks and Nuns
Happy Hoppy Hippo   Ghost of Hops,Hazy Pink Cloud,Summer in India
Brewing Barbarian   Coalminers Sweat,Hoppy Crude Oil,Pale Rider Rides,Reindeer Fuel
```

这是数据库为了获取唯一值而必须对产品名称进行排序所导致的副作用。但这并不能保证总是像这里看到的这样有序并正常工作——数据库可能会找到一种方法，例如，使用哈希函数来执行 `distinct`，那时结果就会变得非常无序。

## 聚合函数 xmlagg

既然你已经看过了 `listagg`、`collect` 和 `stragg`——如果这还不够，下面的清单展示了使用 `xmlagg` 进行字符串聚合的第四种方法。

```
SQL> select
2     max(brewery_name) as brewery_name
3   , rtrim(
4        xmlagg(
5           xmlelement(z, product_name, ',')
6           order by product_id
7        ).extract('//text()').getstringval()
8      , ','
9     ) as product_list
10  from brewery_products
11  group by brewery_id
12  order by brewery_id;
清单 10-8
使用 xmlagg 并从 xml 中提取文本
```

审视一下我在这里使用的表达式，其工作原理如下：

*   在第 5 行，我创建了一个名为 `z` 的 XML 元素（名称无关紧要），其中包含产品名称和逗号的连接。
*   在第 4 和第 6 行使用 `xmlagg`，我创建了一个 XML 片段，它是前面创建的 `z` XML 元素的聚合——按产品 id 排序。
*   在第 7 行，我移除了片段中的 XML 标签，只保留文本值。
*   此时聚合的文本末尾多了一个逗号，因此我使用第 3 和第 8 行的 `rtrim` 将其去掉。

所有这些加在一起，使得清单 10-8 的输出与清单 10-3 和 10-5 中的 `listagg` 和 `collect` 的输出完全一致。

那么这个 `z` XML 元素是怎么回事？它的目的是什么？嗯，如果我只做 `xmlagg(xmlelement(...` 而跳过 `extract` 和 `rtrim`，对于 Balthazar Brauerei 来说，输出将是这样的：

```
Monks and Nuns,Hercule Trippel,Der Helle Kumpel,
```

你可以看到一系列 `z` 元素的 XML 开始和结束标签，每个都包含一个产品名称和逗号。我用于 XML 标签的实际名称无关紧要，所以尽可能短也可以，因为当我对其执行 `extract('//text()')` 时，它会被剥离掉：

```
Monks and Nuns,Hercule Trippel,Der Helle Kumpel,
```

现在你可以明白为什么需要 `rtrim` 来移除末尾的逗号了。

创建 `z` XML 元素是在使用 `xmlagg` 时的一种良好做法。但实际上还有一种替代方法，可以让你免于执行 `extract` 来剥离 XML 标签。

函数 `xmlparse` 接收文本形式的 XML 并将其转换为 `XMLType` 数据类型。通常它会检查是否是良好的 XML，但它也支持关键字 `wellformed`，通过它你告诉数据库“相信我，这是良构的 XML，你不需要检查它。” 因此，我可以用 `xmlparse` 替换 `xmlelement` 的使用，从而跳过使用 `extract`：

```
...
4        xmlagg(
5           xmlparse(content product_name || ',' wellformed)
6           order by product_id
7        ).getstringval()
...
```

这将直接给我输出，没有 XML 标签，准备好使用 `rtrim` 函数去掉最后一个逗号：

```
Monks and Nuns,Hercule Trippel,Der Helle Kumpel,
```

当你有我展示过的其他替代方案时，为什么还要考虑使用 `xmlagg`？部分原因是在较旧的数据库中，使用 `xmlagg` 就可以进行字符串聚合，而无需安装自己的数据类型，这很方便；部分原因是它是处理非常长的聚合的方式之一，就像我马上要展示给你的那样。

### 当结果不适合 VARCHAR2 时

到目前为止我展示的字符串聚合，如果聚合输出的长度超过了 `varchar2` 的最大长度——通常是 4000 字节，但如果你的数据库 `max_string_size` 设置为 `extended`，则可能是 32,767 字节——它们都将失败。

如果你需要更大的输出该怎么办？为了向你展示这一点，我将使用 `monthly_sales` 表并将其与 `products` 表连接。

我有 10 种产品中每一种的 3 年月度销售数据，因此此表中有 360 行。想象一下，我需要以固定长度格式输出每一行的产品名称——也就是说，每个产品名称用空格填充，使其恰好占满 20 个字符，且不使用任何分隔符。结果是一个 7200 字符的单一字符串。

在下面的清单中，我尝试使用 `listagg` 生成此字符串——由于我使用了 `group by`，我应该得到一行输出，其中包含一个包含这 7200 个字符、固定长度的 360 个产品名称列表的列。

```
SQL> select
2     listagg(rpad(p.name, 20)) within group (
3        order by p.id
4     ) as product_list
5  from products p
6  join monthly_sales ms
7     on ms.product_id = p.id;
Error starting at line : 1 in command -
Error report -
ORA-01489: result of string concatenation is too long
清单 10-9
使用 listagg 时出现 ORA-01489 错误
```

但在我的数据库中失败了，因为 `varchar2` 最长只能为 4,000 字节。为了解决这个问题，我有不同的选择。


#### 仅获取结果的第一部分

有时我实际上并不需要获取整个结果；只需获取能放入一个`varchar2`的内容，并附带一个指示，说明还有更多无法显示的内容即可。在 12.2 版本中，`listagg` 函数得到了增强，正好提供了此功能，如我所示于清单 10-10。

```sql
SQL> select
2     listagg(
3        rpad(p.name, 20)
4        on overflow truncate '{more}' with count
5     ) within group (
6        order by p.id
7     ) as product_list
8  from products p
9  join monthly_sales ms
10     on ms.product_id = p.id;
清单 10-10
在 listagg 中抑制错误
```

与清单 10-9 相比，我只是添加了第 4 行：

*   关键字 `on` 和 `overflow` 用于指定如果聚合结果变得太长而无法放入一个 `varchar2` 时，数据库应执行的操作。默认是 `on overflow error`，即清单 10-9 中显示的错误。
*   `truncate` 指定不引发错误，而是只返回能放入 `varchar2` 的内容并截断其余部分。请注意，它绝不会在列表中的字符串中间截断——导致溢出的那个字符串（使得输出无法容纳）将被整体截断。
*   如果结果被截断，字面量 `'{more}'` 将被附加到结果后。如果不指定字面量，默认是省略号（三个点）`'...'`。
*   `with count` 会导致被截断的元素数量（而非字符数）被附加到结果后。默认是 `without count`。

添加第 4 行使得清单 10-10 能够无错误地运行，并给我如下输出，这是一个几乎 4000 个字符长的单个字符串（此处为节省篇幅省略了大部分）：

```
PRODUCT_LIST
Coalminers Sweat    Coalminers Sweat    ...[[3880 个字符已移除]]...Der Helle Kumpel    Der Helle Kumpel    {more}(162)
```

因此，对于只需知道有更多内容无法容纳的情况，这是对 `listagg` 的一个很好的增强。但如果情况并非如此呢？那么我还有其他可能性。

#### 尝试通过减少数据来使其适应

可能存在这样的情况：使用 `listagg` 无法容纳的原因是数据不唯一，而你实际上不需要看到重复数据的每个单独出现——一次就足够了。当你的数据库是 19c 或更高版本时，你可以进行 `distinct` 字符串聚合，使得较少的出现次数可能适应一个 `varchar2`。

清单 10-11 类似于清单 10-9；我只是在 `listagg` 函数调用中添加了关键字 `distinct`，这是 19c 版本中的一个新特性。

```sql
SQL> select
2     listagg(distinct rpad(p.name, 20)) within group (
3        order by p.id
4     ) as product_list
5  from products p
6  join monthly_sales ms
7     on ms.product_id = p.id;
清单 10-11
使用 distinct 减少数据
```

由于此案例中的 7200 字符字符串包含大量重复项，使用 `distinct` 给我一个仅有 200 个字符的字符串：

```
PRODUCT_LIST
Coalminers Sweat    Der Helle Kumpel    Ghost of Hops       Hazy Pink Cloud     Hercule Trippel     Hoppy Crude Oil     Monks and Nuns      Pale Rider Rides    Reindeer Fuel       Summer in India
```

如果我没有 19c 数据库，我可以使用一个带有 `select distinct` 的内联视图，然后在该内联视图的结果上执行 `listagg` 聚合。

对于唯一数据集能使聚合结果足够小的情况，`listagg` 在 19c 或更高版本中支持此功能。但也可能存在你确实需要聚合结果大于 `varchar2` 的情况——这时你需要一个 `clob`。

#### 使用 CLOB 代替 VARCHAR2

使用 clob 的一种方法是使用前面展示的 `collect` 函数，然后创建一个函数 `name_coll_type_to_clob`，而不是我展示的 `name_coll_type_to_varchar2`。我将此留给你作为练习，因为如果你愿意尝试，不需要做太多更改。

但在清单 10-12 中，我将展示如何使用内置函数 `xmlagg` 聚合到一个 `clob`——这样你就不需要创建任何自己的函数。

```sql
SQL> select
2     xmlagg(
3        xmlparse(
4           content rpad(p.name, 20) wellformed
5        )
6        order by product_id
7     ).getclobval() as product_list
8  from products p
9  join monthly_sales ms
10     on ms.product_id = p.id;
清单 10-12
使用 xmlagg 聚合到 clob
```

这非常类似于我在清单 10-8 中所做的，只是在第 7 行使用了 `getclobval()` 而不是 `getstringval()`。这确实是将 `xmltype` 转换为 `clob` 而非 `varchar2` 所需的全部操作，结果就是我想要的 7200 字符字符串（此处显示时省略了大部分）：

```
PRODUCT_LIST
Coalminers Sweat    Coalminers Sweat    ...[[7120 个字符已移除]]...Pale Rider Rides    Pale Rider Rides
```

如果我的数据库是 18c 或更高版本，我可以使用 `json_arrayagg` 作为 `xmlagg` 的替代品，获得与清单 10-12 相同的输出。我在清单 10-13 中展示了一个例子。

```sql
SQL> select
2     json_value(
3        replace(
4           json_arrayagg(
5              rpad(p.name, 20)
6              order by product_id
7              returning clob
8           )
9         , '","'
10         , ''
11        )
12      , '$[0]' returning clob
13     ) as product_list
14  from products p
15  join monthly_sales ms
16     on ms.product_id = p.id;
清单 10-13
使用 json_arrayagg 聚合到 clob
```

如果你没有创建自己的 `name_coll_type_to_clob` 并且数据库中安装了 APEX，你还有一个 APEX 函数可以使用，如我在清单 10-14 中所示。

```sql
SQL> select
2     apex_string.join_clob(
3        cast(
4           collect(
5              rpad(p.name, 20)
6              order by p.id
7           )
8           as apex_t_varchar2
9        )
10      , ''
11      , 12 /* dbms_lob.call */
12     ) as product_list
13  from products p
14  join monthly_sales ms
15     on ms.product_id = p.id;
清单 10-14
使用 apex_string.join_clob 聚合到 clob
```

这是一个可以像本章前面展示的 `apex_string.join` 一样使用的函数。由于 `apex_string.join_clob` 返回一个临时 clob，它与 `apex_string.join` 相比有一个额外的参数，用于指示临时 clob 的生命周期，接受与 `dbms_lob.createtemporary` 相同的值。在第 11 行，我指定该 clob 仅在调用期间存在。

在未来的 `listagg` 实现可能实现 `clob` 支持之前，`xmlagg`、`json_arrayagg` 和 `apex_string.join_clob` 都是有效可用的方法。数据库中的 JSON 功能通常从版本到版本都进行了优化，因此在最新的数据库版本中，JSON 函数通常是最快的解决方案。

### 经验教训

我展示了内置和自定义的字符串聚合方法，使你能够：

*   在除了它不起作用的特殊情况外，使用内置的 `listagg` 函数作为首选方法。
*   创建一个嵌套表类型和一个函数（或使用 APEX 内置函数）来使用 `collect` 聚合函数作为替代方案。
*   使用自定义创建的聚合函数 `stragg`。
*   使用各种内置函数在 `varchar2` 和 `clob` 中执行字符串聚合。

所有这些方法在特殊情况下都可能有用，但我通常的建议是，如果可以的话，坚持使用 `listagg`。内置功能通常比你自己构建的任何东西性能更好——除非情况非常特殊。

