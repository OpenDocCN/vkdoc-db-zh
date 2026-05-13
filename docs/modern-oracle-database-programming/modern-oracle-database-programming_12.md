# 第三部分：Oracle 提供的功能

## 15. 实用的 APEX 包

Oracle Application Express (APEX) 是一个低代码开发平台，允许您构建可扩展、安全的企业应用程序，并可部署在任何地方。嗯，几乎是任何地方；APEX 引擎仍然需要一个 Oracle 数据库。APEX 是 Oracle 数据库的一个免费选项，从 Always Free 云层到 XE 再到企业版都可用。

所有 APEX 包在使用 APEX 时都非常有用。然而，即使您并非将 APEX 用于其设计初衷——创建前端应用程序，您仍然可以从其功能中受益。而这些包正是本主题讨论的对象。

因为 APEX 驻留在 Oracle 数据库内部，所以您可以将其与自己的代码无缝集成。APEX 包由 Oracle 维护。自行开发的解决方案则需要由您自己维护。

从 PL/SQL 调用 Web 服务可能相当具有挑战性，但使用 APEX 包，这变得轻而易举。使用 APEX 开箱即用的功能，可以轻松创建 ZIP 文件和解析 CSV 数据。


### APEX 可用性

只有当你的数据库中安装了 APEX 时，你才能利用它的功能。安装后，请保持其为最新版本，因为新功能和特性在不断加入，安全修复也在持续实施。

APEX 作为一个独立的模式（schema）存在于数据库中，因此检查 APEX 是否已安装的一个简单方法是查询 APEX 用户。

```sql
select username
from all_users
where username like 'APEX%'
```

该查询产生的结果会告诉你 APEX 模式是否存在，如果存在，还会显示已安装的版本。

当你安装了 22.1 版本时，结果如下：

```
APEX_220100
```

如果你从这个查询中没有得到任何结果，那仅仅意味着 APEX 未安装。安装 APEX 超出了本书的范围，请参考相关文档。始终推荐使用最新版本的 APEX。最新版本的文档可以通过以下 URL 找到。

```
https://apex.oracle.com/doc
```

对于特定版本的文档，你可以在 URL 后附加你正在查找的版本号，如下所示。

```
https://apex.oracle.com/doc212
```

由于本章重点介绍 APEX 提供的一些内置包，因此能够方便地访问文档非常有帮助。有一个 URL 可以直接带你进入 PL/SQL API 文档。

```
https://apex.oracle.com/api
```

注意

当数据库管理员（DBA）可能以“让 Web 服务器访问数据库风险太大”为由，导致无法充分利用 APEX 提供的全部功能时，或许可以请求*只*安装 APEX 包而*不*设置 Web 服务器。拥有 APEX 包可以使 PL/SQL 开发工作轻松许多。

### 调用 Web 服务

一个常见的需求是将本地数据库中的数据与不同数据源中可用的数据结合起来。想想天气信息、货币汇率、运费报价或交换发票数据。

实现此目标的一种方法是调用 Web 服务。以前，在与 Web 服务交互时，你必须使用 `UTL_HTTP` 并处理许多底层细节。Oracle APEX 使得调用 Web 服务变得非常简单。APEX 包同时支持 SOAP Web 服务和 REST 风格的 API。

本部分讨论的示例使用了 Ergast Developer API (`http://ergast.com/mrd/`)，该 API 提供了赛车运动的历史数据记录。通过此 Web 服务，可以获取 XML 或 JSON 格式的结果。我们更倾向于使用 JSON。

本章中的示例使用 Oracle Database 21c Express Edition。根据你的数据库设置，你可能需要你的 DBA 配置网络访问控制列表（ACL）。

注意

配置网络 ACL 超出了本书的范围。这在 *Oracle APEX 安装指南* 的第 [6] 章“安装和配置 Oracle APEX 与 Oracle REST 数据服务”中有所讨论。

在 `apex_web_service` 包中，有两个用于调用 REST 风格 Web 服务的过程；一个返回 `CLOB`，另一个返回 `BLOB`。在示例中，调用了 `CLOB` 变体，以便可以轻松检查结果。当你需要将 JSON 文档存储在数据库中时，请参考第 [11] 章。

调用 Web 服务所需的最低限度是 URL 和 HTTP 方法，本例中是 `GET` 请求。

```sql
declare
  l_response clob;
  l_part     varchar2(4000 char);
  l_offset   pls_integer;
begin
  l_response := apex_web_service.make_rest_request
                 (p_url         => 'http://ergast.com/api/f1/current.json'
                 ,p_http_method => 'GET'
                 );
```

该 URL 指定了要检索的内容：以 JSON 文档形式呈现的当前年份的一级方程式数据。如果从 URL 中省略 .json 扩展名，则会返回 XML。URL 语法由你所使用的 Web 服务决定。

调用 Web 服务后，最佳实践是验证调用是否成功。可以通过检查 `apex_web_service` 包中声明的全局变量 `g_status_code` 来获取此值。成功的响应可能与错误响应大相径庭，因此你的解析方法也必须不同，以便从失败的响应中提取有用信息。

响应状态码在 200 到 299 之间表示响应成功。

```sql
if apex_web_service.g_status_code between 200 and 299 then
```

为了显示响应的内容，让我们使用另一个 APEX 包 `apex_string` 来获取 `CLOB` 变量的各个部分，并用 `dbms_output` 显示它们。

```sql
while apex_string.next_chunk (
        p_str    => l_response
       ,p_chunk  => l_part
       ,p_offset => l_offset
       ,p_amount => 4000)
loop
  sys.dbms_output.put_line (l_part);
end loop;
```

代码清单 [15-1] 展示了完整的代码示例和输出的第一部分。完整的消息太长，无法在此全部显示。

```sql
declare
  l_response clob;
  l_part     varchar2(4000 char);
  l_offset   pls_integer;
begin
  l_response := apex_web_service.make_rest_request
                 (p_url         => 'http://ergast.com/api/f1/current.json'
                 ,p_http_method => 'GET'
                 );
  sys.dbms_output.put_line (apex_web_service.g_status_code);
  if apex_web_service.g_status_code between 200 and 299 then
    while apex_string.next_chunk (
            p_str    => l_response
           ,p_chunk  => l_part
           ,p_offset => l_offset
           ,p_amount => 4000)
    loop
      sys.dbms_output.put_line (l_part);
    end loop;
  end if;
end;
{"MRData":{"xmlns":"http:\/\/ergast.com\/mrd\/1.5","series":"f1","url":"http://ergast.com/api/f1/current.json","limit":"30","offset":"0","total":"22","RaceTable":{"season":"2022","Races":[{"season":"2022","round":"1",...
```

代码清单 15-1
从 Web 服务获取当前赛季的 F1 比赛数据



### 在 Oracle APEX 中处理 Web 服务请求头与 JSON 数据

当 Web 服务要求您指定请求头时，最初可能不太清楚该如何操作。

设置请求头是通过为全局变量 `g_request_headers` 赋值来完成的，该变量在 `apex_web_service` 包中定义。

清单 15-2 展示了如何设置用于 Bearer 令牌授权的请求头，将文本 “Bearer ”（带一个尾随空格）与局部变量 `l_bearer_token` 连接起来，以及一个包含在通过 HTTP GET 请求调用 Web 服务时发送的请求体内容类型的请求头。

```
apex_web_service.g_request_headers(1).name  := 'Authorization';
apex_web_service.g_request_headers(1).value := 'Bearer ' || l_bearer_token;
apex_web_service.g_request_headers(2).name  := 'Content-Type';
apex_web_service.g_request_headers(2).value := 'application/json';
```

清单 15-2 设置请求头

由于请求头是设置在全局变量中的，建议在 Web 服务调用后将其清除。这可以防止其他 Web 服务调用重用于特定 Web 服务的相同请求头。可以通过以下方式清除请求头。

```
apex_web_service.clear_request_headers;
```

虽然前述设置请求头的方法有效，但一个打包过程可以一步到位。

```
apex_web_service.set_request_headers (
    p_name_01  => 'Authorization'
   ,p_value_01 => 'Bearer ' || l_bearer_token
   ,p_name_02  => 'Content-Type'
   ,p_value_02 => 'application/json'
);
```

调用此过程的优点在于默认重置请求头；您无需担心它们。它们仅用于当前的 Web 服务调用。最多可以通过 `set_request_headers` 过程设置五个请求头。

`APEX_WEB_SERVICE` 包还能实现更多功能，例如基本认证、OAuth 认证、指定使用哪个 Oracle Wallet 以及向 Web 服务发送 multipart/form 请求体。

一个常见的需求是根据环境让 Web 服务指向多个不同的端点。对于开发环境，您希望指向端点 A，而生产环境则应指向端点 B。这可以通过条件编译实现（更多详情参见第 6 章）。

清单 15-3 是一个变量声明的示例，它根据 `environment_pkg` 中的静态布尔值指向不同的端点。

```
l_url constant varchar2(50) :=
$if environment_pkg.development
$then 'url to endpoint A'
$elsif environment_pkg.production
$then 'url to endpoint B'
$end;
```

清单 15-3 一个变量，不同环境指向不同端点

`environment_pkg` 中的静态布尔值声明如下所示。

```
development constant boolean := TRUE;
production  constant boolean := FALSE;
```

#### 从 JSON 获取数据

当使用 Oracle Database 12 或更高版本时，最好使用内置的 JSON 功能（参见第 11、12、13 和 14 章），而不是 APEX 的功能。内置 JSON 功能的性能远优于 `apex_json`。

您已经检索了一个包含 Formula 1 数据的 JSON 文档。清单 15-4 是此响应的一个格式化的小样本。

```
{
  "MRData": {
    "xmlns": "http:\/\/ergast.com\/mrd\/1.5",
    "series": "f1",
    "url": "http://ergast.com/api/f1/current.json",
    "limit": "30",
    "offset": "0",
    "total": "22",
    "RaceTable": {
      "season": "2022",
      "Races": [
        {
          "season": "2022",
          "round": "1",
          "url": "http:\/\/en.wikipedia.org\/wiki\/2022_Bahrain_Grand_Prix",
          "raceName": "Bahrain Grand Prix",
          // 为简洁起见省略
          // 此部分包含 22 场比赛的详细信息，例如赛道
        }
      ]
    }
  }
}
```

清单 15-4 格式化的 JSON 响应

在从此 JSON 文档中提取数据之前，您需要将 JSON 文档解析为内部格式。

```
apex_json.parse ( p_values => races
                 ,p_source => l_races
                );
```

其中 `races` 是类型为 `apex_json.t_values` 的局部变量，这是所需的内部格式。在局部变量中，`l_races` 是调用 Web 服务的响应，其数据类型为 CLOB。

本赛季的比赛数量存储在 `MRData` 对象的 `total` 属性中。因为此值预期是一个数字，所以可以使用 `get_number` 函数提取它。

```
apex_json.get_number (
    p_path   => 'MRData.total'
   ,p_values => races
);
```

使用点表示法，您指出了通往比赛总数的路径。同样，您可以提取位于 `RaceTable` 对象中更深层的赛季值。

```
apex_json.get_number (
    p_path   => 'MRData.RaceTable.season'
   ,p_values => races
);
```

注意

路径中的名称区分大小写。如果使用错误的大小写，则无法找到您要查找的节点。

所有赛季比赛的名称位于 `Races` 属性中的一个对象数组中。可以通过相应地调整路径来获取本赛季的第一场比赛名称。

```
apex_json.get_varchar2
   (p_path => 'MRData.RaceTable.Races[1].raceName'
   ,p_values => races
   );
```

清单 15-5 是一个完整的示例，包含对 Web 服务的调用并显示 JSON 文档中的不同值。

```
declare
    races      apex_json.t_values;
    l_races    clob;
    l_elements apex_t_varchar2;
begin
    l_races := apex_web_service.make_rest_request
                   (p_url         => 'http://ergast.com/api/f1/current.json'
                   ,p_http_method => 'GET'
                   );
    if apex_web_service.g_status_code between 200 and 299
    then
        apex_json.parse ( p_values => races
                         ,p_source => l_races
                        );
        sys.dbms_output.put_line (
            'Season: '||
            apex_json.get_number
                         (p_path   => 'MRData.RaceTable.season'
                         ,p_values => races)
        );
        sys.dbms_output.put_line (
            'Nr of races: '||
            apex_json.get_number
                         (p_path   => 'MRData.total'
                         ,p_values => races)
        );
        sys.dbms_output.put_line (
            'First Race: '||
            apex_json.get_varchar2
                         (p_path => 'MRData.RaceTable.Races[1].raceName'
                         ,p_values => races)
        );
    end if;
end;
Season: 2022
Nr of races: 22
First Race: Bahrain Grand Prix
```

清单 15-5 从 JSON 文档中提取值



### 基础空间功能

本章使用的 Web 服务还包含了赛道的坐标——纬度（lat）和经度（long）。这些值可以在以下 JSON 片段（并非完整的 JSON 文档）的 `Location` 对象中看到。

```
"Races": [
{
"season": "2022",
"round": "15",
"url": "http:\/\/en.wikipedia.org\/wiki\/2022_Dutch_Grand_Prix",
"raceName": "Dutch Grand Prix",
"Circuit": {
"circuitId": "zandvoort",
"url": "http:\/\/en.wikipedia.org\/wiki\/Circuit_Zandvoort",
"circuitName": "Circuit Park Zandvoort",
"Location": {
"lat": "52.3888",
"long": "4.54092",
"locality": "Zandvoort",
"country": "Netherlands"
}
},
```

以下表达式提取了第 15 场比赛（赞德沃特）的纬度，该值位于 `Location` 对象中，也是数组中的第 15 个对象。

```
apex_json.get_number (p_path => 'MRData.RaceTable.Races[15].Circuit.Location.lat'
,p_values => races)
```

经度的提取方式类似。

要使用 Oracle 数据库提供的所有空间函数，需要将坐标转换为正确的数据类型 `mdsys.sdo_geometry`。

以下代码片段展示了如何进行此转换，其中 `l_circuit` 是一个局部变量。

```
l_circuit := apex_spatial.point
(p_lon =>  apex_json.get_number
(p_path => 'MRData.RaceTable.Races[15].Circuit.Location.long'
,p_values => races)
,p_lat => apex_json.get_number
(p_path => 'MRData.RaceTable.Races[15].Circuit.Location.lat'
,p_values => races)
);
```

现在 JSON 文档中的值已经转换为真实的几何对象，让我们使用空间函数来计算荷兰 Oosterhout 与赞德沃特赛道之间的距离。清单 15-6 是一个完整的代码示例，其中用 `apex_spatial.` 构建了两个几何对象。`sdo_distance` 空间函数用于确定两者之间的距离（以公里为单位）。例如，你可以使用 Google Maps 查找你家乡的坐标。

```
declare
l_circuit  mdsys.sdo_geometry;
l_hometown mdsys.sdo_geometry;
races      apex_json.t_values;
l_races    clob;
begin
l_hometown := apex_spatial.point
(p_lon => 4.861316883708699
,p_lat => 51.64485385756768
);
l_races := apex_web_service.make_rest_request
(p_url         => 'http://ergast.com/api/f1/current.json'
,p_http_method => 'GET'
);
apex_json.parse (p_values => races
,p_source => l_races
);
l_circuit := apex_spatial.point
(p_lon =>  apex_json.get_number (p_path => 'MRData.RaceTable.Races[15].Circuit.Location.long'
,p_values => races)
,p_lat => apex_json.get_number (p_path => 'MRData.RaceTable.Races[15].Circuit.Location.lat'
,p_values => races)
);
sys.dbms_output.put_line (
'The distance from my hometown to the circuit is '
||sdo_geom.sdo_distance
(geom1 => l_hometown
,geom2 => l_circuit
,unit  => 'unit=KM')
||' km'
);
end;
The distance from my hometown to the circuit is 85.64951103008 km
清单 15-6
我的家乡距离赞德沃特赛道有多远？
```

### 文本处理实用工具

你已经学习了如何利用 `apex_string.next_chunk` 包函数来显示 CLOB，但还有许多其他很酷的函数值得研究。

我们个人最喜欢的功能之一是 `split`。清单 15-7 展示了该函数的示例。

```
select column_value
from apex_string.split
(p_str => 'this:should:be:shown:as:a:table'
,p_sep => ':')
COLUMN_VALUE

this
should
be
shown
as
a
table
清单 15-7
apex_string.split 用法示例
```

如清单 15-7 所示，`split` 函数将一个字符分隔的字符串（也可以是 CLOB）转换为一个嵌套表类型。

既然有 `split` 函数，那也一定有 `join` 函数。输入的嵌套表类型为 `apex_t_varchar2`。你可以在清单 15-8 中看到此功能的示例及其输出。

```
declare
l_ename_t apex_t_varchar2;
begin
select ename
bulk collect
into l_ename_t
from emp;
sys.dbms_output.put_line (
apex_string.join (p_table => l_ename_t
,p_sep   => '~'
)
);
end;
SMITH~ALLEN~WARD~JONES~MARTIN~BLAKE~CLARK~SCOTT~KING~TURNER~ADAMS~JAMES~FORD~MILLER
清单 15-8
apex_string.join 用法示例
```

必须指出的是，上述 `apex_string.join` 函数的功能也可以通过 SQL 函数 `listagg` 实现。`listagg` 函数能处理的最大长度取决于 `max_string_size` 初始化参数的设置。当 `listagg` 函数不适用（超出长度限制）时，`apex_string.join_clobs` 可以填补这个空缺。

#### 格式化消息

当你需要格式化消息时，`format` 函数非常有用。此函数允许你创建包含最多 19 个替换变量占位符的消息。清单 15-9 是一个带有替换参数的格式化错误消息示例。

```
raise_application_error
(-20000
,apex_string.format (p_message => q'{The combination %0, %1, and %2 is not allowed}'
,p0 => 'A'
,p1 => 'B'
,p2 => 'C'
)
);
Error report -
ORA-20000: The combination A, B, and C is not allowed
ORA-06512: at line 2
清单 15-9
一个带有变量的、格式良好的错误消息
```

此函数还提供了一个参数 `p_max_length`，用于防止消息变得过长。该参数的默认值为 1000。

当你需要保留多行消息的布局时，`p_prefix` 参数就非常有用。指定的前导前缀和空白将从每一行中移除。清单 15-10 展示了一个多行消息及其输出的示例。注意，所有的前导空白都被移除了，但结果文本按照预期进行了缩进。

```
begin
sys.dbms_output.put_line (
apex_string.format (
p_message => q'{Things you should do:
!   * Get quality sleep
!   * Eat Healthy
!   * Exercise regularly
!and all will be well}'
,p_prefix => '!'
));
end;
/
Things you should do:
* Get quality sleep
* Eat Healthy
* Exercise regularly
and all will be well
清单 15-10
保留多行消息的布局
```

#### 请在此处签上姓名首字母

APEX_STRING 包中一个实用的小工具是 `get_initials`。此函数将姓名转换为姓名首字母缩写。它输出每个单词的首字母（大写），数量上限由你指定。默认返回两个字母。如果单词数量超过 `p_cnt`，则返回 *前* `p_cnt` 个单词的首字母。清单 15-11 是此函数的示例。注意，输入的名字是小写，并指定了最多返回四个字母。由于输入字符串中只有两个单词，结果是两个大写字母。

```
begin
sys.dbms_output.put_line (
apex_string.get_initials (
p_str => 'alex nuijten'
,p_cnt => 4
)
);
end;
AN
清单 15-11
获取姓名首字母
```

一种更动态地获取给定姓名中所有首字母的方法是使用正则表达式来统计姓名中的单词数量，类似于下面的示例：

```
declare
l_str varchar2(140) := 'Willem-Alexander Claus George Ferdinand';
begin
sys.dbms_output.put_line (
apex_string.get_initials (
p_str => l_str
,p_cnt => regexp_count (l_str, '[[:punct:][:space:]]') + 1
)
);
end;
WACGF
```

### 解析数据

一个常见的需求是将文件（例如 Excel 或 CSV 文件）上传到数据库，然后解析它以获取内容并将其存储到数据库表中。这个需求可能相当具有挑战性，特别是当 Excel 文件中有多个工作表，或者需要转义某些值等情况时。这正是 `apex_data_parser` 大放异彩的地方，它可以让你的程序变得优雅。



### 业务案例

本节中的示例是一个业务案例的一部分。我们应用的用户可以将 CSV 文件上传到数据库。这些文件存储在一个 `BLOB` 列中。在此案例中，保存已上传文件的 `TEMP_FILES` 表包含表 15-1 中概述的列。

**表 15-1**

**TEMP_FILES 表中的列**

| 列名 | 用途 |
| --- | --- |
| ID | 用于识别要处理文件的主键 |
| Filename | 已上传文件的名称，应包含文件扩展名 |
| Mime_type | Mime 类型，在此案例中应为 "text/csv" |
| Blob_content | 以二进制格式存储的已上传文件 |

**注意**

文件扩展名被 `APEX_DATA_PARSER` 包用于识别应如何解析文件。在此案例中，文件名应以 `.csv` 为后缀。

这些文件包含支付数据，例如 `payment_date`、`payment_method`、`currency`、`amount` 等。CSV 文件中使用的分隔符可能各不相同；有时是逗号，有时是分号。有些文件的标题行带引号；有些则不带。

由于应用允许上传不同的文件，每个文件的结构可能差异很大。文件中的标题行决定了你应如何使用其中的数据。文件中可能包含比你需要的多得多的列，但这些列可以忽略。

对于每个文件，你都知道相关信息在哪里可以找到。

一个名为 `DESTINATION_TBL` 的目标表用于存储解析后的数据。

你需要处理许多变化；实现这一点绝非易事。

#### 让我们开始解析吧！

实现需求的第一步是确定你正在处理哪种变化的支付文件。

因为文件的变化种类是有限的，所以在此案例中，你可以为每个已知的标题行结构定义常量。清单 15-12 定义了两种不同标题的常量：一个是 Mollie 文件的，另一个是 PayPal 文件的。常量的数据类型是 `apex_t_varchar2`，这是一种嵌套表类型。即使文件中的每一列并不都用于提取数据，你仍然需要在常量中声明它们。如你所见，这两种文件的结构差异很大。

```sql
l_mollie_headers constant apex_t_varchar2
:= apex_t_varchar2
('DATE_'
,'PAYMENT_METHOD'
,'CURRENCY'
,'AMOUNT'
,'STATUS'
,'ID'
,'DESCRIPTION'
,'CONSUMER_NAME'
,'CONSUMER_BANK_ACCOUNT'
,'CONSUMER_BIC'
,'SETTLEMENT_CURRENCY'
,'SETTLEMENT_AMOUNT'
,'SETTLEMENT_REFERENCE'
,'AMOUNT_REFUNDED'
);
l_paypal_headers constant apex_t_varchar2
:= apex_t_varchar2
('DATE_'
,'TIME'
,'TIME_ZONE'
,'DESCRIPTION'
,'CURRENCY'
,'GROSS'
,'FEE'
,'NET'
,'BALANCE'
,'TRANSACTION_ID'
,'FROM_EMAIL_ADDRESS'
,'NAME'
,'BANK_NAME'
,'BANK_ACCOUNT'
,'SHIPPING_AND_HANDLING_AMOUNT'
,'SALES_TAX'
,'INVOICE_ID'
,'REFERENCE_TXN_ID'
);
```

*清单 15-12*
*用于不同文件头的常量*

下一步是从上传的 CSV 文件中提取第一行，即标题行。这个文件标题存储在一个局部变量中，也是一个 `apex_t_varchar2` 类型。

你可以使用 `APEX_DATA_PARSER` 包中的 `discover` 函数来识别你正在处理的文件。该函数返回一个 `CLOB` 值，其中包含文件的列配置信息。你可以通过将此配置信息传递给 `get_columns` 函数来获取有关列的信息，例如以下内容。

- 列位置
- 列名
- 数据类型
- 格式掩码

使用一个简单的 select 语句，你可以提取清单 15-13 中显示的信息。参数是 `P_ID`。它需要你想要处理的已上传文件的主键。

```sql
select column_name
bulk collect
into l_file_headers
from temp_files tf
cross
join table (apex_data_parser.get_columns
(apex_data_parser.discover
(p_content   => tf.blob_content
,p_file_name => tf.filename
)))
where tf.id = p_id;
```

*清单 15-13*
*从文件中提取列名*

**提示**

清单 15-13 中的 `table` 运算符自 Oracle Database 12.2 起已是可选的。你不再需要使用它。但它可以使你的代码更易读、更容易理解。

一旦你获得了文件的配置信息以及该文件中的列名，你就可以将其与预定义的文件模式（之前声明的常量）进行比较。

```sql
case l_file_headers
when l_mollie_headers
then
process_mollie;
when l_paypal_headers
then
process_paypal;
end case;
```

我们更倾向于使用搜索式 case 语句。当没有匹配项时（上传了未知文件），它会引发 `CASE_NOT_FOUND` 异常。这个异常可以被捕获，并向应用提供反馈，告知上传了错误的文件。

找到匹配项后，文件终于可以开始处理了。你需要为每种文件类型编写单独的 procedure 来提取相关信息。

你可以使用 `APEX_DATA_PARSER` 包中的 `parse` 函数从上传的文件中检索所有数据。最多支持 300 列。

清单 15-14 展示了如何使用 `parse` 函数将相关数据插入到 `DESTINATION_TBL` 中。

```sql
insert into destination_tbl
(date_
,payment_method
,currency
,amount
,status
,description
)
select to_date (col001, 'yyyy-mm-dd hh24:mi:ss')
,col002
,col003
,col004
,col005
,col007
from temp_files tf
cross
join table (apex_data_parser.parse
(p_content   => tf.blob_content
,p_file_name => tf.filename
,p_skip_rows => 1 -- 跳过标题行
))
where tf.id = p_id;
```

*清单 15-14*
*将数据从文件移动到表中*

这让你对 `apex_data_parser` 的功能有了初步了解，但它还可以更加灵活。在你从 `discover` 函数获取的文件配置信息中，还包含有关列中使用的格式掩码和小数字符的信息。这些信息可以将文件中的数据转换为数据库中的正确数据。

**提示**

使用 `validate_conversion` 函数，你可以确定一个表达式是否可以转换为指定的数据类型。此外，`to_date` 和 `to_number` 的 `default on conversion error` 子句在这方面也能提供帮助。

### 处理 ZIP 文件

上一节讨论了将文件上传到数据库，但对于应用用户来说，这可能相当繁琐。在这些情况下，如果用户能够上传包含所有需要上传到数据库的文件的 ZIP 文件，可能会更有益。

反之亦然。当用户想要从数据库下载多个文件时，下载单个 ZIP 文件比下载十个单独的文件更有效率。


## 16. 后台数据处理

处理数据可能相当耗时；例如，当一个文件被上传到数据库时，它需要被解析并加载到多个表中。对于最终用户而言，在处理完成之前，应用程序的使用会陷入停滞。
通常，文件（或任何其他类型的数据流入——如网络服务）的处理可以延迟到稍后进行。将进程转移到后台，可以将应用程序的控制权交还给用户，给人一种应用程序快速响应的感觉，从而改善用户体验。

### 资源密集型

数据处理可能是资源密集型的，这可能会妨碍应用程序的用户。在系统负载高峰期过后再处理数据将是有益的，因为它不会妨碍任何用户。
任何非时间依赖性的进程，例如通过外部网络服务生成文档或从数据库发送电子邮件，都可以卸载到后台进程。您最了解自己的系统，能够判断某个进程是否可以卸载到后台。

### 队列

Merriam-Webster 词典将*队列*定义为“一种数据结构，由记录列表组成，记录从一端添加，从另一端移除。”
在最基本的层面，消息被入队到队列中，每条消息由消费者出队并处理一次。消息会一直保留在队列中，直到被出队移除或消息过期。消息的生产者决定消息何时过期。消息不一定按照入队的顺序出队。
在 Oracle 数据库中，队列是一个功能齐全的消息传递解决方案。支持点对点、发布/订阅、持久性和非持久性消息传递。有多种方式可以与队列功能交互，包括 PL/SQL、JMS、JDBC、.NET、Python 和 Node.js。
Oracle 数据库提供两种类型的队列系统：事务性事件队列（`TEQ`）和高级队列（`AQ`）。`TEQ`是一个高性能的分区实现，每个队列支持多个事件流。`AQ`是一个基于磁盘的实现，适用于更简单的工作流用例。本章重点介绍`AQ`。

### 用例

通过应用程序上传几个包含最新详细信息的 CSV 文件，以保持一级方程式数据库的更新。需要存储许多细节，如单圈时间、进站信息、排位赛结果等。
处理文件中所有不同的结果需要相当长的时间，并且可能是资源密集型的。在这些文件被解析和处理时，应用程序无法响应，用户对此表示不满。
通过将文件的解析和处理推迟到稍后的时间点，可以将应用程序的控制权交还给用户。
从高层次来看，计划是将文件存储在数据库表中。包含新创建记录主键的消息被放置在队列上。完成此操作后，控制权即交还给用户。
稍后，检索队列上的消息，指示哪些文件仍需要处理。处理完成后，可以从队列中移除消息，并且可以从临时表中移除记录（但这对于流程并非必要，为了以防万一，保留它们可能有用）。
可以通过调度器作业以定时间隔读取队列中的消息，但`AQ`有更好的方法，将在后面探讨。

### 解压文件

上传的`ZIP`文件存入数据库后，从中提取文件是直接了当的。良好的做法是将文件上传到单独的表中。
要解压文件，首先需要根据`ZIP`文件的名称将其提取到本地的`BLOB`变量中。这个本地`BLOB`变量作为参数传递给`get_files()`函数。返回值是`apex_zip.t_files`数据类型，这是一个关联数组。它是`ZIP`文件内的文件名列表。
```sql
declare
l_zip_file      blob;
l_unzipped_file blob;
l_files         apex_zip.t_files;
begin
select blob_content
into l_zip_file
from zip_files
where filename = 'archive.zip';
l_files := apex_zip.get_files
(p_zipped_blob => l_zip_file);
```
下一步是从`ZIP`文件中获取实际的文件。为此，使用`get_file_content()`函数，它接受`ZIP`文件内部的文件名。此示例使用一个简单的`for 循环`将`ZIP`内的文件插入到之前创建的表中。
```sql
for i in 1 .. l_files.count
loop
l_unzipped_file := apex_zip.get_file_content
(p_zipped_blob => l_zip_file
,p_file_name   => l_files(i)
);
insert into temp_files
(filename
,blob_content
)
values
(l_files (i)
,l_unzipped_file
);
end loop;
end;
```
这就是解压`ZIP`文件并将文件存储到数据库表中所需的全部操作。
当`ZIP`文件包含文件夹时，文件名会以`dir/subdir/filename`的形式反映出来。

### 压缩文件

在数据库内部创建一个可供应用程序下载的`ZIP`文件，对用户可能很有用。
首先，必须检索要压缩到`ZIP`文件中的文件。对于此示例，使用了一个游标`for 循环`；它对 100 行数据进行隐式的批量获取。本地变量（数据类型为`BLOB`）就是`ZIP`文件。
```sql
declare
l_zip_file blob;
begin
for r in (select tf.filename
,tf.blob_content
from temp_files tf
)
loop
```
`add_file()`过程接受三个参数：本地变量（一个`ZIP`文件）、文件名和文件内容。
```sql
apex_zip.add_file (p_zipped_blob => l_zip_file
,p_file_name   => r.filename
,p_content     => r.blob_content
);
end loop;
```
要完成`ZIP`文件，您需要告知压缩进程不再向压缩包中添加文件。
```sql
apex_zip.finish (p_zipped_blob => l_zip_file);
```
下面的示例将结果存储在一个现有的表中。
```sql
insert into zip_files
(filename
,mime_type
,blob_content
)
values
('all_files.zip'
,'application/zip'
,l_zip_file
);
end;
```

### 小结

本章详细探讨了`APEX`包。即使您不使用`APEX`来开发应用程序，您仍然可以利用它的一些功能。
您学习了如何调用网络服务以及如何从`JSON`文档中提取信息。该`JSON`文档包含空间信息，因此可以使用`APEX`函数将它们转换为真正的空间几何对象。借助`apex_string`，格式化字符串和提取首字母缩写变得很容易。
使用`apex_data_parser`可以轻松解析来自文件的数据并提取相关信息。
最后，您了解了如何在数据库中压缩和解压文件。


### 设置

要使用 AQ，需要额外的权限。与 AQ 相关的权限在表 16-1 中概述。

#### 权限

| 操作 | 所需权限 |
| --- | --- |
| 创建、删除和监控自己的队列 | 对 `DBMS_AQADM` 的执行权限 |
| 创建、删除和监控任何队列 | 对 `DBMS_AQADM` 的执行权限和 `AQ_ADMINISTRATOR_ROLE` 角色 |
| 向自己的队列入队和出队 | 对 `DBMS_AQ` 的执行权限 |
| 向其他队列入队和出队 | 对 `DBMS_AQ` 的执行权限。该队列的所有者可以使用 `DBMS_AQADM.GRANT_QUEUE_PRIVILEGE` 授予权限 |
| 向任何队列入队和出队 | 对 `DBMS_AQ` 的执行权限。AQ 管理员必须使用 `DBMS_AQADM.GRANT_SYSTEM_PRIVILEGE` 授予 `ENQUEUE ANY QUEUE` 和 `DEQUEUE ANY QUEUE` 权限 |

对于此用例，需要 `dbms_aqadm` 和 `dbms_aq` 的执行权限。这允许你创建和删除队列，以及入队和出队消息。

如高层计划所述，需要一个暂存表来保存上传的文件。该表包含表 16-2 中概述的列。

#### 暂存表结构

| 列名 | 用途 |
| --- | --- |
| `FileID` | 主键，用于标识要处理的文件 |
| `Filename` | 上传文件的名称，应包含文件扩展名 |
| `Mime_type` | MIME 类型，在本例中应为 "text/csv" |
| `Blob_content` | 上传文件的二进制格式 |
| `Processed` | 一个 是/否 标志，指示文件是否已处理 |
| `Uploaded_ts` | 记录文件上传时间的时间戳，默认值：`systimestamp` |
| `Processed_ts` | 记录处理完成时间的时间戳 |

### 队列表与队列

要放置在队列中的消息（也称为*有效负载*）由一个对象类型定义。本例中，对象类型定义如下。

```
create noneditionable
type file_payload_ot as object
(fileid number)
/
```

这个用例非常直接，但你可以创建一个具有更多属性的对象类型。

基于此对象类型，可以使用 `DBMS_AQADM` 包创建队列表。

```
begin
sys.dbms_aqadm.create_queue_table (
queue_table        => 'file_process_qt'
, queue_payload_type => 'file_payload_ot'
);
end;
/
```

这基于之前创建的有效负载对象类型创建了一个队列表。

在此队列表中，需要一个队列来放置和移除消息。可以在单个队列表中有多个队列，但这超出了本用例的范围。队列通过以下语句创建并启动。

```
begin
sys.dbms_aqadm.create_queue
( queue_name  => 'file_process_q'
, queue_table => 'file_process_qt'
);
sys.dbms_aqadm.start_queue
( queue_name => 'file_process_q' );
end;
/
```

当队列启动后，它就准备好接受要入队的消息了。

这些操作之后，数据库中会新增几个支持队列功能的对象。

```
select object_name
, object_type
from   user_objects
where  object_name like 'AQ%'
/
OBJECT_NAME               OBJECT_TYPE
------------------------- -----------------------
AQ$FILE_PROCESS_QT        VIEW
AQ$_FILE_PROCESS_QT_E     QUEUE
AQ$_FILE_PROCESS_QT_F     VIEW
AQ$_FILE_PROCESS_QT_H     TABLE
AQ$_FILE_PROCESS_QT_I     INDEX
AQ$_FILE_PROCESS_QT_T     INDEX
6 rows selected.
```

### 入队与出队

通过调用 `dbms_aq.enqueue` 将消息放置到队列上。在入队消息时，可以设置许多属性，例如指定延迟或消息何时应过期，但对于简单的入队操作，默认设置完全足够。例如，默认情况下，入队跟随主事务。当入队消息的会话回滚事务时，消息将从队列中移除。此行为可以被覆盖；即使主事务回滚，消息也会保持入队状态。

入队的选项通过以下类型的变量设置。

```
sys.dbms_aq.enqueue_options_t
```

以下代码将一个消息入队。

```
declare
l_enqueue_options    sys.dbms_aq.enqueue_options_t;
l_message_properties sys.dbms_aq.message_properties_t;
l_msgid              raw( 16 );
l_payload            file_payload_ot;
begin
l_payload := file_payload_ot (1234);
sys.dbms_aq.enqueue
( queue_name         => 'file_process_q'
, enqueue_options    => l_enqueue_options
, message_properties => l_message_properties
, payload            => l_payload
, msgid              => l_msgid
);
end;
/
```

> 注意
>
> 入队和出队的 `queue_name` 参数指的是队列的名称，而不是队列表。

查询 `file_process_qt` 队列表时，消息会出现在结果集中。

```
select user_data
from file_process_qt
/
USER_DATA(FILEID)

FILE_PAYLOAD_OT(1234)
```

回滚事务会再次移除它。

```
rollback
/
Rollback complete.
select user_data
from file_process_qt
/
no rows selected
```

要从队列中出队一个消息，使用 `dbms_aq.dequeue` 过程。与入队过程类似，出队过程的默认设置很合理，在大多数情况下已经足够。默认情况下，出队过程跟随主事务。当消息出队后事务被回滚，消息不会从队列中移除。

出队选项通过以下类型的局部变量设置。

```
sys.dbms_aq.dequeue_options_t;
```

以下代码从队列中出队一个消息。

```
declare
l_dequeue_options    sys.dbms_aq.dequeue_options_t;
l_message_properties sys.dbms_aq.message_properties_t;
l_msgid              raw( 16 );
l_payload            file_payload_ot;
begin
sys.dbms_aq.dequeue
( queue_name         => 'file_process_q'
, dequeue_options    => l_dequeue_options
, message_properties => l_message_properties
, payload            => l_payload
, msgid              => l_msgid
);
sys.dbms_output.put_line
( '要处理的文件是 [' || l_payload.fileid || ']' );
end;
/
要处理的文件是 [1234]
```

当没有消息可供出队时，该过程会等待直到有消息出现。`wait` 的默认设置是*永远等待*。这可以通过将 `wait` 出队选项设置为 `no_wait` 或特定的秒数来覆盖。

```
l_dequeue_options.wait := dbms_aq.no_wait;
```

等待最长时间三秒。

```
l_dequeue_options.wait := 3;
```

当等待时间结束，或指定了 `no_wait` 且队列上没有消息时，会引发异常。

```
ORA-25228: timeout or end-of-fetch during message dequeue from BOOK.FILE_PROCESS_Q
```




### 包装过程

对于此用例，需要一个过程将文件存储在暂存表中，并在队列上放置一条消息。用于入队消息的包装过程定义如下。

```sql
procedure process_later (p_fileid in staging.fileid%type)
is
l_enqueue_options    sys.dbms_aq.enqueue_options_t;
l_message_properties sys.dbms_aq.message_properties_t;
l_msgid              raw( 16 );
l_payload            file_payload_ot;
begin
l_payload := file_payload_ot( p_fileid );
sys.dbms_aq.enqueue
( queue_name         => 'file_process_q'
, enqueue_options    => l_enqueue_options
, message_properties => l_message_properties
, payload            => l_payload
, msgid              => l_msgid
);
end process_later;
```

应用程序调用的用于将文件存储在暂存表中的过程定义如下。

```sql
procedure upload_file
( p_filename     in staging.filename%type
, p_mime_type    in staging.mime_type%type
, p_blob_content in staging.blob_content%type
)
is
l_fileid staging.fileid%type;
begin
insert into staging
( filename
, mime_type
, blob_content
, upload_ts
, processed)
values
( p_filename
, p_mime_type
, p_blob_content
, systimestamp
, 'N')
returning fileid
into l_fileid;
-- 当文件存储在表中后
-- 在队列上放置一条消息以
-- 在稍后时间启动处理
process_later( p_fileid => l_fileid );
end;
/
```

还需要一个过程来处理文件并将数据分发到正确的表中。对于本章，该过程的实现并不重要。实现了以下过程来模拟文件的处理。

```sql
procedure process_file (p_fileid in staging.fileid%type)
is
begin
sys.dbms_session.sleep( 3 ); -- 模拟耗时操作
update staging
set    processed = 'Y'
, processed_ts = systimestamp
where  fileid = p_fileid;
end process_file;
/
```

此过程首先等待三秒钟，之后将处理标志设置为 `'Y'`，并用当前时间戳填充 `processed_ts`。

### PL/SQL 回调通知

一旦文件被上传到暂存表，并且消息被放入队列，如何启动处理？

一种方法是安排一个作业运行，比如每天在指定时间运行。安排作业的缺点是，即使没有文件需要处理，作业也会运行。文件也可能在作业运行后上传，导致处理延迟。这也意味着即使系统在该时间非常繁忙，作业也会运行，从而妨碍系统用户。

#### 回调通知机制

队列有一个更好的技巧：PL/SQL 回调通知。当消息添加到队列时，回调通知可以启动一个 PL/SQL 过程。当消息入队时，回调过程会自动启动。当这种情况发生时，它由数据库处理，很可能是在数据库“安静”时。缺点是无法控制*何时*处理文件。

#### 设置回调

必须采取一些步骤来设置 PL/SQL 回调通知。首先，队列表必须允许多个消费者出队消息。消费者是读取队列中消息的代理。

不幸的是，队列表不能被修改以允许多个消费者。这只能在创建时指定。要删除队列表和队列，请执行以下代码。

```sql
begin
dbms_aqadm.stop_queue
( queue_name => 'file_process_q' );
dbms_aqadm.drop_queue
( queue_name => 'file_process_q' );
dbms_aqadm.drop_queue_table
( queue_table => 'file_process_qt' );
end;
/
```

队列在启动时无法被删除。必须先停止它。

以下代码创建一个新的队列表，启用了多消费者功能，并基于相同的对象类型。

```sql
begin
sys.dbms_aqadm.create_queue_table
( queue_table        => 'file_process_qt'
, queue_payload_type => 'file_payload_ot'
, multiple_consumers => true
);
end;
/
```

重新创建队列表后，也必须重建队列。

```sql
begin
sys.dbms_aqadm.create_queue
( queue_name  => 'file_process_q'
, queue_table => 'file_process_qt'
);
sys.dbms_aqadm.start_queue
( queue_name => 'file_process_q' );
end;
/
```

这些操作会导致数据库中创建额外的对象。

```sql
select object_name
, object_type
from   user_objects
where  object_name like 'AQ%'
/
OBJECT_NAME               OBJECT_TYPE
------------------------- -----------------------
AQ$FILE_PROCESS_QT        VIEW
AQ$FILE_PROCESS_QT_R      VIEW
AQ$FILE_PROCESS_QT_S      VIEW
AQ$_FILE_PROCESS_QT_E     QUEUE
AQ$_FILE_PROCESS_QT_F     VIEW
AQ$_FILE_PROCESS_QT_G     TABLE
AQ$_FILE_PROCESS_QT_H     TABLE
AQ$_FILE_PROCESS_QT_I     TABLE
AQ$_FILE_PROCESS_QT_L     TABLE
AQ$_FILE_PROCESS_QT_N     SEQUENCE
AQ$_FILE_PROCESS_QT_S     TABLE
AQ$_FILE_PROCESS_QT_T     TABLE
AQ$_FILE_PROCESS_QT_V     EVALUATION CONTEXT
13 行已选择。
```

#### 回调过程

在消费者可以订阅队列并指定要调用的 PL/SQL 例程之前，必须创建 PL/SQL 回调过程。PL/SQL 回调过程必须遵循以下签名。

```sql
procedure plsqlcallback(
context  IN  RAW
, reginfo  IN  SYS.AQ$_REG_INFO
, descr    IN  SYS.AQ$_DESCRIPTOR
, payload  IN  RAW
, payloadl IN  NUMBER
);
```

对于这个简单的用例，`descr` 参数包含了所有需要的信息。其他参数可用于更高级的用例。PL/SQL 回调过程的主要目的是出队消息并调用文件处理功能。

```sql
procedure process_file_callback( context  in raw
, reginfo  in sys.aq$_reg_info
, descr    in sys.aq$_descriptor
, payload  in raw
, payloadl in number)
is
l_dequeue_options    sys.dbms_aq.dequeue_options_t;
l_message_properties sys.dbms_aq.message_properties_t;
l_msgid              raw(16);
l_payload            file_payload_ot;
begin
-- 从队列中检索消息
l_dequeue_options.msgid := descr.msg_id;
l_dequeue_options.consumer_name := descr.consumer_name;
sys.dbms_aq.dequeue
( queue_name         => descr.queue_name
, dequeue_options    => l_dequeue_options
, message_properties => l_message_properties
, payload            => l_payload
, msgid              => l_msgid
);
-- 一旦消息出队，就可以在处理文件
-- 过程中完成实际工作，以解析和处理
-- 文件内容
process_file (p_fileid => l_payload.fileid);
-- 必须提交事务以
-- 确保消息从队列中
-- 被移除
commit;
end process_file_callback;
/
```

该过程应使用 `commit` 完成事务，以确保消息从队列中明确移除。

#### 注册订阅者和回调

有了这个 PL/SQL 回调通知，就可以创建一个消费者来订阅队列。订阅者的名称是 `file_process_subscriber`，并被添加到 `file_process_q`。

```sql
begin
sys.dbms_aqadm.add_subscriber
( queue_name => 'file_process_q'
, subscriber => sys.aq$_agent
( name     => 'file_process_subscriber'
, address  => null
, protocol => null
)
);
end;
/
```

最后一步是注册实际的回调过程。

```sql
begin
sys.dbms_aq.register
( reg_list =>
sys.aq$_reg_info_list
( sys.aq$_reg_info
( name      => 'file_process_q:file_process_subscriber'
, namespace => dbms_aq.namespace_aq
, callback  => 'plsql://process_file_callback'
, context   => null
)
)
, reg_count => 1
);
end;
/
```

`context` 参数可用于将信息（以 RAW 格式）传递给回调过程。在回调过程中，您可以使用此信息，例如，来区分队列的不同订阅者。对于此用例，由于只有一个订阅者，因此不需要这样做。




### 实际应用观察

一旦所有组件准备就绪，应用程序用于将文件上传到数据库的代码就会启动整个流程。由于文件尚未被解析和处理，控制权会立即返回给应用程序用户。

```
begin
upload_file
( p_filename     => 'latest.csv'
, p_mime_type    => 'text/csv'
, p_blob_content => :file
);
end;
/
```

前端应用程序管理着事务。用户通过发出 `commit` 或 `rollback` 命令来控制事务的结束。

一旦事务以 `commit` 结束，队列中的消息就会被存储，并且回调过程会被调用。

在使用上述例程上传文件后，可以在临时表中观察到以下内容。

```
select to_char (upload_ts, 'hh24:mi:ss.ff9')    as upload_ts
, to_char (processed_ts, 'hh24:mi:ss.ff9') as processed_ts
, processed_ts - upload_ts                 as elapsed_time
from   staging
/
UPLOAD_TS          PROCESSED_TS       ELAPSED_TIM
------------------ ------------------ -----------
12:28:54.873961000
```

等待几秒钟后，可以看到以下结果。

```
UPLOAD_TS          PROCESSED_TS       ELAPSED_TIME
------------------ ------------------ -------------------
12:28:54.873961000 12:28:58.108458000 +00 00:00:03.234497
```

当然，这很大程度上取决于数据库的“繁忙”或“空闲”程度。当数据库繁忙时，回调函数会延迟执行。没有 API 可以控制或定义并发 `PL/SQL` 通知出队进程的数量。这是因为 `AQ` 消息可能以突发式到达，固定数量的进程可能并不总是能高效工作。`AQ` 通知会根据总工作量和执行用户定义的 `PL/SQL` 回调过程所需的时间，自动启动合适数量的后台进程（又称 `EMON` 从属进程）。

当消息入队时，用户 `SYS` 会在后台启动一个新会话。模块是 `DBMS_SCHEDULER`，因此这个包必然涉及该机制。操作（action）被设置为类似“`AQ$_PLSQL_NTFN_`”后跟一个数字的内容。该操作信号涉及 `AQ` 和 `PL/SQL` 通知。这可以通过查询 `gv$session` 观察到。

```
select username
, module
, action
from   gv$session
/
USERNAME    MODULE              ACTION
----------- ------------------- -------------------------
SYS         DBMS_SCHEDULER      AQ$_PLSQL_NTFN_4107371948
```

### 异常队列

创建队列表时会自动添加一个异常队列。当在执行回调过程期间发生意外异常时，队列上的消息会被移至异常队列。

查询 `user_queues` 数据字典视图，可以看到 `file_process_qt` 队列表中有两个队列：一个是常规队列，一个是异常队列。

```
select name
, queue_table
, queue_type
from   user_queues
/
NAME                  QUEUE_TABLE     QUEUE_TYPE
--------------------- --------------- --------------------
FILE_PROCESS_Q        FILE_PROCESS_QT NORMAL_QUEUE
AQ$_FILE_PROCESS_QT_E FILE_PROCESS_QT EXCEPTION_QUEUE
```

出于演示目的，对最终从回调过程调用的 `process_file` 过程进行了修改，通过在过程体中包含以下行，使其 `总是` 引发异常。

```
raise zero_divide;
```

此异常在 `process_file` 过程中未被处理，并传播到调用它的 `process_file_callback` 过程。由于回调过程是 `无客户端` 调用的，因此没有客户端能观察到此异常。该消息被移至异常队列。

这不会立即发生；系统会多次尝试成功执行该过程。这些尝试的频率以及尝试之间的延迟时间可以在创建队列时进行控制。有两个参数控制这些设置：`max_retries` 和 `retry_delay`。如果未填写这些参数，系统将尝试执行回调过程六次，然后才将消息移至异常队列。每次尝试都会立即进行，没有延迟。

当连续查询队列表 `file_process_qt` 时，可以观察到消息从常规队列移动到异常队列的过程。

```
select qt.q_name
, qt.msgid
, to_char (systimestamp, 'hh24:mi:ss.ff9') as ts
from   file_process_qt qt
/
Q_NAME                MSGID                    TS
--------------------- ------------------------ ------------------
FILE_PROCESS_Q        ECA2CF1D94B3043FE0539518 13:45:55.842533000
>
Q_NAME                MSGID                    TS
--------------------- ------------------------ ------------------
FILE_PROCESS_Q        ECA2CF1D94B3043FE0539518 13:46:12.811844000
>
Q_NAME                MSGID                    TS
--------------------- ------------------------ ------------------
AQ$_FILE_PROCESS_QT_E ECA2CF1D94B3043FE0539518 13:46:15.315077000
```

一旦问题得到解决，异常情况消失，就可以将异常队列中的消息放回常规队列，这将再次触发回调过程的执行。

在可以从异常队列中出队消息之前，必须启动异常队列以进行出队操作。必须执行以下代码才能为出队操作启动异常队列。

```
begin
sys.dbms_aqadm.start_queue
( queue_name => 'AQ$_FILE_PROCESS_QT_E'
, enqueue    => false
, dequeue    => true
);
end;
/
```

无法为入队操作启动异常队列。

将消息从异常队列移动到常规队列，可以通过从异常队列出队消息，然后将它们入队到常规队列来完成。以下代码使用一个简单的 `for-loop` 结构，对异常队列上的所有消息执行此操作。

```
declare
l_dequeue_options    sys.dbms_aq.dequeue_options_t;
l_message_properties sys.dbms_aq.message_properties_t;
l_message_handle     raw(16);
l_payload            file_payload_ot;
l_enqueue_options    sys.dbms_aq.enqueue_options_t;
begin
for r in (select qt.msgid
from   file_process_qt qt
where  qt.q_name = 'AQ$_FILE_PROCESS_QT_E')
loop
l_dequeue_options.msgid := r.msgid;
sys.dbms_aq.dequeue
( queue_name         => 'AQ$_FILE_PROCESS_QT_E'
, dequeue_options    => l_dequeue_options
, message_properties => l_message_properties
, payload            => l_payload
, msgid              => l_message_handle
);
sys.dbms_aq.enqueue
( queue_name         => 'file_process_q'
, enqueue_options    => l_enqueue_options
, message_properties => l_message_properties
, payload            => l_payload
, msgid              => l_message_handle
);
end loop;
commit;
end;
/
```

在某些情况下，消息位于异常队列上且需要被移除。此时，可以使用 `DBMS_AQADM` 包中的 `purge_queue_table` 过程清除异常队列。

```
declare
l_purge_options sys.dbms_aqadm.aq$_purge_options_t;
begin
sys.dbms_aqadm.purge_queue_table
( 'file_process_qt'
, 'qtview.queue = ''AQ$_FILE_PROCESS_QT_E'''
, l_purge_options
);
end;
/
```



### 高级队列与 JSON

自 Oracle 数据库 21c 起，高级队列（AQ）也支持原生 JSON 负载。此版本引入了原生 JSON 数据类型。在此之前，处理 JSON 负载虽然可行，但始终是基于文本的；例如，使用 `sys.aq$_jms_text_message` 作为负载数据类型。

要创建处理 JSON 负载的队列表，请为 `queue_payload_type` 参数指定 JSON。

```sql
begin
sys.dbms_aqadm.create_queue_table (
queue_table        => 'json_qt'
,queue_payload_type => 'json'
);
end;
/
```

在队列表中创建和启动队列的操作与之前相同。

```sql
begin
sys.dbms_aqadm.create_queue (
queue_name  => 'jq'
,queue_table => 'json_qt'
);
sys.dbms_aqadm.start_queue (
queue_name => 'jq'
);
end;
/
```

当然，放入队列的消息应该是 JSON 格式。

```sql
declare
l_enqueue_options    sys.dbms_aq.enqueue_options_t;
l_message_properties sys.dbms_aq.message_properties_t;
l_msgid              raw (16);
l_payload            json;
begin
l_payload := json ('{"Hello" : "World"}');
sys.dbms_aq.enqueue
(queue_name         => 'jq'
,enqueue_options    => l_enqueue_options
,message_properties => l_message_properties
,payload            => l_payload
,msgid              => l_msgid
);
commit;
end;
/
```

当消息出队时，请记住负载是 JSON。这意味着在 `dbms_output` 中使用时，应将其视为 JSON 处理。在下面的示例中，返回值应转换为字符串。

```sql
set serveroutput on
declare
l_dequeue_options    sys.dbms_aq.dequeue_options_t;
l_message_properties sys.dbms_aq.message_properties_t;
l_msgid              raw(16);
l_payload            json;
begin
sys.dbms_aq.dequeue(
queue_name         => 'jq',
dequeue_options    => l_dequeue_options,
message_properties => l_message_properties,
payload            => l_payload,
msgid              => l_msgid
);
sys.dbms_output.put_line(
'Retrieved from the JSON-payload ['
|| json_value (l_payload, '$.Hello' returning varchar2 )
|| ']'
);
end;
/
Retrieved from the JSON-payload [World]
```

### 总结

本章通过一个常见用例探讨了如何提升用户体验，并讨论了高级队列（AQ）在其中发挥的作用。将处理任务卸载给后台进程，能够在控制权交还给用户时，让应用程序显得响应非常迅速。

生成 PDF 文档可能耗时且耗费资源。当此生成过程被卸载到后台进程时，就不会因过度消耗资源而妨碍任何用户。

由于回调过程在后台运行，必须监控任何导致消息传播到异常队列的异常，以确保该消息本应执行的工作得以完成。

本章还包含一个简短示例，展示了 Oracle 数据库 21c 中引入的原生 JSON 数据类型。

## 17. 内省 PL/SQL

了解正在执行的 PL/SQL 块中发生了什么可以提供有价值的信息。从 PL/SQL 块内部设置 client_identifier、module 和 action 有助于跟踪每个会话正在执行的操作。`DBMS_SESSION` 包中为此提供了支持性的过程和函数。

在 `DBMS_UTILITY` 包中，有可以在发生意外异常时识别调用堆栈的过程和函数。甚至可以使用本章同样涵盖的 `utl_call_stack` 来检索关于调用堆栈的更多信息。

### DBMS_SESSION

使用 `DBMS_SESSION`，您可以从 PL/SQL 中执行 `alter session` 命令。通常在 SQL*Plus 或 SQLcl 会话中进行的设置，现在可以通过您的 PL/SQL 过程和函数来完成。

### current_is_role_enabled

如果程序的某部分需要您启用某个角色，您可以使用 `dbms_session.current_is_role_enabled` 来检查指定角色当前是否已启用。您向函数提供要检查的角色名称。如果角色已启用，函数返回 `TRUE`；否则返回 `FALSE`。

```sql
function current_is_role_enabled(rolename varchar2)
return boolean;
```

### set_role

如果所需角色未启用但已授予您，您可以使用 `dbms_session.set_role` 在运行时设置角色。如果角色已经启用，再次发出 set role 命令会执行而不会报错。参数中的文本被附加到 "SET ROLE" SQL 语句后，然后作为动态 SQL 执行。

```sql
procedure set_role(role_cmd varchar2);
```

### set_nls

有时您的程序依赖于特定的 NLS 设置；例如，在将字符串转换为日期或反向转换时。如果您想确保 NLS 设置正确完成，而不依赖于用户的设置，可以使用 `dbms_session.set_nls` 来设置这些特定设置。该过程接受两个参数：参数名称和其值。

```sql
procedure set_nls(param varchar2, value varchar2);
```

调用此过程等效于执行 `alter session set <nls_parameter> = <value>`。

参数名称必须以 NLS 开头。如果值是文本字面量，则需要用单引号括起来。下面是一个示例。

```sql
set_nls('nls_date_format','''DD-MON-YY''')
```

如果参数需要很多引号，您可能希望使用替代引用法。

```sql
dbms_session.set_nls('nls_date_format', q'['DD-MON-YY']')
```

**注意**

使用 `execute immediate 'alter session ...'` 具有更好的性能。此外，`execute immediate` 允许在一条语句中指定多个参数，这比多次调用 `dbms_session.set_nls` 更高效。


#### 设置上下文

要使用上下文设置功能，您必须通过清单 17-1 中的 SQL 命令创建一个上下文。

```
create or replace context modp_ctx using modp_pkg
/
清单 17-1
创建应用程序上下文
```

如果您想在此上下文中设置或清除值，您不能直接调用 `DBMS_SESSION` 包。

```
begin
dbms_session.set_context( namespace => 'modp_ctx'
, attribute => 'author1'
, value     => 'Alex Nuijten'
);
dbms_session.set_context( namespace => 'modp_ctx'
, attribute => 'author2'
, value     => 'Patrick Barel'
);
end;
/
ORA-01031: 权限不足
ORA-06512: 在 "SYS.DBMS_SESSION", 第 141 行
ORA-06512: 在第 2 行
```

您必须创建一个包来访问 `DBMS_SESSION` 包中的过程。这可以是一个简单的包装器包，如清单 17-2 所示。正如您在清单 17-1 创建应用程序上下文的语句中所见，此包与上下文相关联。

```
create or replace package modp_pkg as
procedure set_context
( namespace_in in varchar2
, attribute_in in varchar2
, value_in     in varchar2
, username_in  in varchar2 default null
, client_id_in in varchar2 default null
);
procedure clear_context
( namespace_in in varchar2
, client_id_in in varchar2 default null
, attribute_in in varchar2 default null
);
procedure clear_all_context
( namespace_in in varchar2 );
end modp_pkg;
/
create or replace package body modp_pkg as
procedure set_context
( namespace_in in varchar2
, attribute_in in varchar2
, value_in     in varchar2
, username_in  in varchar2 default null
, client_id_in in varchar2 default null
) is
begin
dbms_session.set_context(namespace => namespace_in
,attribute => attribute_in
,value     => value_in
,username  => username_in
,client_id => client_id_in);
end set_context;
procedure clear_context
( namespace_in in varchar2
, client_id_in in varchar2 default null
, attribute_in in varchar2 default null
) is
begin
dbms_session.clear_context(namespace => namespace_in
,client_id => client_id_in
,attribute => attribute_in);
end clear_context;
procedure clear_all_context
( namespace_in in varchar2 ) is
begin
dbms_session.clear_all_context(namespace => namespace_in);
end clear_all_context;
end modp_pkg;
/
清单 17-2
用于与上下文通信的 modp_pkg
```

现在，当您尝试运行与之前相同的代码块时，只需调用包装器包，您就可以设置上下文值。

```
begin
modp_pkg.set_context( namespace_in => 'modp_ctx'
, attribute_in => 'author1'
, value_in     => 'Alex Nuijten'
);
modp_pkg.set_context( namespace_in => 'modp_ctx'
, attribute_in => 'author2'
, value_in     => 'Patrick Barel'
);
end;
/
PL/SQL 过程已成功完成
```

注意：包装器包可用于多个上下文。如果该包仅用于单个上下文，则最好删除 `namespace_in` 参数。

您可以使用简单的 select 语句检查上下文中的值，如下所示。

```
select sys_context('modp_ctx','author1') author1
, sys_context('modp_ctx','author2') author2
from   dual
/
AUTHOR1              AUTHOR2
-------------------- --------------------
Alex Nuijten         Patrick Barel
```

但您也可以在 `where` 子句的谓词中使用上下文值。

```
select 'author2 = Patrick Barel'
from   dual
where  sys_context('modp_ctx','author2') = 'Patrick Barel'
/
'AUTHOR2=PATRICKBAREL'

author2 = Patrick Barel
```

注意：此技术有时用于 *参数化视图*，其中上下文谓词用作过滤器。

#### 清除上下文

您可以调用 `dbms_session.clear_context` 过程来清除上下文中的属性。与 `set_context` 过程一样，您必须编写一个包装器过程来调用此过程。

```
begin
dbms_session.clear_context( namespace => 'modp_ctx'
, attribute => 'author1' );
dbms_session.clear_context( namespace => 'modp_ctx'
, attribute => 'author2' );
end;
/
ORA-01031: 权限不足
ORA-06512: 在 "SYS.DBMS_SESSION", 第 155 行
ORA-06512: 在第 2 行
```

您可以使用清单 17-2 所示的包来清除属性。

```
begin
modp_pkg.clear_context( namespace_in => 'modp_ctx'
, attribute_in => 'author1' );
modp_pkg.clear_context( namespace_in => 'modp_ctx'
, attribute_in => 'author2' );
end;
/
PL/SQL 过程已成功完成
```

#### 清除所有上下文

如果要清除上下文中的所有属性，请使用 `dbms_session.clear_all_context` 过程。与其他上下文过程一样，这不能直接调用，只能通过用户定义的包调用。

#### 设置标识符

当您想在 `v$session` 数据字典视图中标识连接的会话时，可以使用 `dbms_session.set_identifier` 过程。您可以在参数中设置任何内容，但仅使用前 64 个字符。

```
begin
dbms_session.set_identifier
( client_id => 'Whatever you want, whatever you need.' );
end;
/
```

如果您有权访问 `v$session` 视图，则可以看到该标识符。

```
select username
,client_identifier
from   v$session
/
USERNAME   CLIENT_IDENTIFIER
---------- ----------------------------------------
BOOK
BOOK       Whatever you want, whatever you need.
```

#### 清除标识符

要清除标识符，您可以使用空字符串调用 `set_identifier` 过程，但更有意义的做法是调用 `dbms_session.clear_identifier` 过程。

```
begin
dbms_session.clear_identifier;
end;
/
```

如您所见，标识符已被清除。

```
select username
,client_identifier
from   v$session
/
USERNAME   CLIENT_IDENTIFIER
---------- ----------------------------------------
BOOK
BOOK
```

#### 休眠

有时您希望会话暂停一小段时间。您可以使用 `dbms_session.sleep` 过程来实现这一点。

```
procedure sleep(seconds in number);
```

该参数也接受秒的小数部分。最大分辨率为百分之一秒；例如，1.5、1.01 和 0.99 都是合法值。

调用此过程比调用 `dbms_lock.sleep` 更可取，后者功能相同，但此过程避免了需要授予 `dbms_lock` 权限并暴露更敏感的方法。

### DBMS_APPLICATION_INFO

使用 `DBMS_APPLICATION_INFO` 中的过程来设置有关程序的运行时信息。您可以在通用的日志记录过程和函数中使用此信息。模块和操作也用于数据库工具。它们被记录用于跟踪和各种企业管理器报告，这与 `set_client_info` 不同，后者不被数据库使用，仅用于您的自定义代码。

#### 设置模块

此过程将当前模块的名称设置为一个新模块。在日志记录框架中使用此过程时，应注意以下事项：如果过程或函数终止，应将其设置为先前的值，这样当调用程序恢复时，它会被设置为正确的值。如果没有先前的值，则应将其设置为 `NULL`。

```
procedure set_module(module_name varchar2, action_name varchar2);
```

参数的最大长度为 64 字节。较长的名称将被截断。

#### 设置操作

此过程在当前模块内设置当前操作的名称。可以使用 `set_module` 过程设置模块。

```
procedure set_action(action_name varchar2);
```

参数的最大长度为 64 字节。较长的名称将被截断。


### DBMS_UTILITY

## read_module

此过程读取当前模块和操作字段中的值。

```sql
procedure read_module( module_name out varchar2
, action_name out varchar2);
```

如代码清单 17-3 所示，这是这些过程的一般用法。

```sql
declare
l_old_module_name varchar2(64);
l_old_action_name varchar2(64);
begin
sys.dbms_application_info.read_module
( module_name => l_old_module_name
, action_name => l_old_action_name
);
sys.dbms_application_info.set_module
( module_name => 'ModernOracleDatabaseProgramming'
, action_name => 'DemoDBMS_Application_Info'
);
-- do the real work of this program
--
-- at the end, reset the module and action
sys.dbms_application_info.set_module
( module_name => l_old_module_name
, action_name => l_old_action_name
);
end;
/
```

代码清单 17-3
模块和操作的用法

该实用程序包包含各种实用程序例程，如 `compile_schema`、`format_call_stack` 和 `expand_sql_text`。在升级应用程序后，`compile_schema` 非常有用，可确保在需要时重新编译代码库。检查与视图或嵌套视图相关的内容可以通过 `expand_sql_text` 轻松完成。这消除了手动检查源代码的需要。

## compile_schema

`compile_schema` 过程不是您希望在程序执行期间调用的过程，而是您希望在安装新版本代码库后调用的过程。它会编译指定模式中的所有过程、函数、包和触发器。由于没有必要重新编译已经有效的对象，因此建议将标志 `compile_all` 设置为 FALSE。

调用此过程后，您可以使用 `ALL_OBJECTS` 或 `USER_OBJECTS` 视图检查所有对象是否已成功编译。参数在表 17-1 中描述。

表 17-1

compile_schema 过程的参数

| 参数 | 是否必填 | 描述 |
| --- | --- | --- |
| schema | Y | 要（重新）编译的模式 |
| compile_all |   | 一个布尔标志，指示是否应编译所有模式对象，无论对象当前是否无效。默认值为 TRUE |
| reuse_settings |   | 一个布尔标志，指示应重用对象中的会话设置，还是应改用当前会话设置。默认值为 FALSE |

## format_call_stack

在调试期间，检索调用栈可能非常有用。`dbms_utility.format_call_stack` 过程返回当前调用栈的格式化字符串，包括私有包过程和函数的名称。此字符串的大小最多可达 2000 字节。为了演示这一点，请查看代码清单 17-4 中的包。

注意

与其编写自己的调用栈解析器，不如查看本章后面 `utl_call_stack` 中提供的功能。

```sql
create or replace package call_stack_demo is
procedure packageprocedure;
end call_stack_demo;
/
create or replace package body call_stack_demo is
procedure privateprocedure is
begin
dbms_output.put_line('-- call stack in PRIVATEPROCEDURE --');
dbms_output.put_line( dbms_utility.format_call_stack );
dbms_output.put_line('------------------------------------');
end privateprocedure;
procedure packageprocedure is
begin
dbms_output.put_line('-- call stack in PACKAGEPROCEDURE --');
dbms_output.put_line( dbms_utility.format_call_stack );
dbms_output.put_line('------------------------------------');
privateprocedure;
end packageprocedure;
end call_stack_demo;
/
```

代码清单 17-4
用于演示 dbms_utility.format_call_stack 的包

以下是调用包过程的输出。

```
begin
dbms_output.put_line( '-- call stack in ANONYMOUS BLOCK    --' );
dbms_output.put_line( dbms_utility.format_call_stack );
dbms_output.put_line( '--------------------------------------' );
call_stack_demo.packageprocedure;
end;
/
-- call stack in ANONYMOUS BLOCK    --
----- PL/SQL Call Stack -----
object  line   object
handle  number name
0x2138aa190  3 anonymous block

-- call stack in PACKAGEPROCEDURE --
----- PL/SQL Call Stack -----
object  line   object
handle  number name
0x6863c2600 13 package body BOOK.CALL_STACK_DEMO.PACKAGEPROCEDURE
0x2138aa190  5 anonymous block

-- call stack in PRIVATEPROCEDURE --
----- PL/SQL Call Stack -----
object  line   object
handle  number name
0x6863c2600  6 package body BOOK.CALL_STACK_DEMO.PRIVATEPROCEDURE
0x6863c2600 15 package body BOOK.CALL_STACK_DEMO.PACKAGEPROCEDURE
0x2138aa190  5 anonymous block

PL/SQL procedure successfully completed
```

## format_error_stack

当异常发生时，了解错误栈可能比检索调用栈更重要。有两个过程可以检索此信息，`dbms_utility.format_error_stack` 包含发生的错误代码和描述，而 `dbms_utility.format_error_backtrace` 不提供错误代码，但它返回完整的调用栈。它不是返回过程或函数名称，而是返回发生错误或调用下一个过程或函数的行号。为了演示这一点，请查看代码清单 17-5 中的包。

```sql
create or replace package body error_stack_demo is
procedure privateprocedure is
begin
raise no_data_found;
end privateprocedure;
procedure packageprocedure is
begin
privateprocedure;
exception
when others
then
dbms_output.put_line('-- error stack in PACKAGEPROCEDURE--');
dbms_output.put_line( dbms_utility.format_error_stack );
dbms_output.put_line('------------------------------------');
dbms_output.put_line('-error backtrace in PACKAGEPROCEDURE');
dbms_output.put_line( dbms_utility.format_error_backtrace );
dbms_output.put_line('------------------------------------');
end packageprocedure;
end error_stack_demo;
/
```

代码清单 17-5
用于演示错误栈过程的包

以下是调用包过程的输出。

```
begin
error_stack_demo.packageprocedure;
end;
/
-- error stack in PACKAGEPROCEDURE--
ORA-01403: no data found
ORA-06512: at "BOOK.ERROR_STACK_DEMO", line 5

-- error backtrace in PACKAGEPROCEDURE--
ORA-06512: at "BOOK.ERROR_STACK_DEMO", line 5
ORA-06512: at "BOOK.ERROR_STACK_DEMO", line 10

PL/SQL procedure successfully completed
```

根据您的需求，您可以使用这两个过程中的任何一个。



`comma_to_table` 通过 `comma_to_table``,` 可以将逗号分隔的字符串拆分为关联数组中的值。如果数据库表中的某一列包含逗号分隔的值，而你希望单独处理每个值，这将非常有用，尤其是当每行中的元素数量可能不同时。

**注意**

`dbms_utility.comma_to_table` 适用于标识符的逗号分隔列表，因此本节示例中使用的 `uncl_array` 是 `table of varchar2(227)`，其中 227 是 `"user"."name"."column"@link` 格式标识符可能的最大长度。你不能将 `comma_to_table` 用于任何通用值列表，最大长度仅限 227。如果需要更通用的解决方案，请查阅 `apex_string.split`（参见第 15 章）。

假设你有一个视图，其中一列包含某个车队的所有车手，如清单 17-6 所示。

```
create or replace view drivers_for_constructors
as
select dcr.constructorid                         as constructorid
, ctr.name                                  as name
, listagg( distinct( drv.driverref ), ',' ) as drivers
from   driverconstructors  dcr
join   f1data.constructors ctr
on ( dcr.constructorid = ctr.constructorid )
join   f1data.drivers      drv
on ( dcr.driverid      = drv.driverid )
group  by dcr.constructorid
, ctr.name
/
清单 17-6
包含每个车队所有车手的视图
```

有些车队只有两名车手，例如，因为他们只参加了一个赛季。有些车队则拥有大量车手，因为他们自一级方程式开始以来就一直参赛。

如果你想为每个车队分别显示车手，可以构建一个如清单 17-7 所示的脚本。

```
declare
l_tablen binary_integer;
l_tab dbms_utility.uncl_array;
l_drivername varchar2( 32767 );
begin
for rec in (select distinct ctr.name
, dfc.drivers
from   drivers_for_constructors dfc
join   driverconstructors as of period
for    drivercontract to_date( '20220320'
, 'YYYYMMDD') dcr
on     ( dfc.constructorid = dcr.constructorid )
join   f1data.constructors ctr
on     ( dfc.constructorid = ctr.constructorid )
)
loop
dbms_output.put_line( rec.name );
dbms_utility.comma_to_table( list   => rec.drivers
, tablen => l_tablen
, tab    => l_tab );
dbms_output.put_line(  'Nr of drivers: '
|| to_char( l_tablen )
);
for indx in 1 .. l_tablen
loop
select d.forename || ' ' || d.surname as drivername
into   l_drivername
from   f1data.drivers d
where  d.driverref = l_tab( indx );
dbms_output.put_line(  to_char( indx )
|| ') '
|| l_drivername
);
end loop;
end loop;
end;
/
Aston Martin
Nr of drivers: 3
1) Nico Hülkenberg
2) Lance Stroll
3) Sebastian Vettel
Mercedes
Nr of drivers: 5
1) Valtteri Bottas
2) Lewis Hamilton
3) Michael Schumacher
4) Nico Rosberg
5) George Russell
Red Bull
Nr of drivers: 12
1) Alexander Albon
2) David Coulthard
3) Robert Doornbos
4) Pierre Gasly
5) Christian Klien
6) Daniil Kvyat
7) Vitantonio Liuzzi
8) Max Verstappen
9) Sergio Pérez
10) Daniel Ricciardo
11) Sebastian Vettel
12) Mark Webber
>
Haas F1 Team
Nr of drivers: 6
1) Romain Grosjean
2) Esteban Gutiérrez
3) Kevin Magnussen
4) Nikita Mazepin
5) Mick Schumacher
6) Pietro Fittipaldi
PL/SQL procedure successfully completed
清单 17-7
显示每个车队的车手
```

**注意**

此代码未经过优化；它仅展示了 `dbms_utility.comma_to_table` 的用法。

## expand_sql_text

如果你想查看执行了什么 SQL，可以使用 `dbms_utility.``expand_sql_text`。它会递归地将输入 SQL 中的任何视图引用替换为视图的实际子查询。此过程在调试 SQL 宏时也非常有用。

```
procedure expand_sql_text( input_sql_text  in  clob,
output_sql_text out nocopy clob );
```

该视图返回某个车队的所有车手（参见清单 17-6）。使用此视图，你可以查询 2022 年锦标赛中各车队的所有车手。

```
select distinct c.name
, dfc.drivers
from   drivers_for_constructors dfc
join   driverconstructors as of period
for    drivercontract to_date('20220320', 'YYYYMMDD') dc
on     ( dfc.constructorid = dc.constructorid )
join   f1data.constructors c
on     ( dfc.constructorid = c.constructorid )
/
NAME                 DRIVERS
-------------------- ----------------------------------------
Aston Martin         hulkenberg,stroll,vettel
Mercedes             bottas,hamilton,michael_schumacher,
rosberg,russell
Red Bull             albon,coulthard,doornbos,gasly,klien,
kvyat,liuzzi,max_verstappen,perez,ricciardo
,vettel,webber
Alpine F1 Team       alonso,ocon
Ferrari              adamich,alboreto,alesi,alonso,amon,
arnoux,badoer,baghetti,bandini,barrichello,
bell,berger,bondurant,capelli,fisichella,
galli,gilles_villeneuve,giunti,ickx,
irvine,johansson,larini,lauda,leclerc,
mansell,mario_andretti,massa,merzario,
michael_schumacher,morbidelli,parkes,pironi,
prost,raikkonen,regazzoni,reutemann,
rodriguez,sainz,salo,scarfiotti,scheckter,
surtees,tambay,vaccarella,vettel,williams
Alfa Romeo           baldi,bottas,brambilla,cesaris,cheever,
depailler,giacomelli,giovinazzi,kubica,
mario_andretti,patrese,raikkonen,zhou
McLaren              alliot,alonso,andretti,berger,blundell,
bonnier,brundle,button,cesaris,charlton,
coulthard,donohue,emerson_fittipaldi,
gethin,giacomelli,gilles_villeneuve,
hailwood,hakkinen,hamilton,hobbs,hulme,hunt,
ickx,johansson,keke_rosberg,kevin_magnussen,
kovalainen,lauda,lunger,magnussen,mansell,
mass,mclaren,montoya,norris,oliver,perez,
piquet,prost,raikkonen,redman,revson,
ricciardo,rosa,sainz,scheckter,senna,
south,tambay,trimmer,vandoorne,villota,
watson,wurz
Williams             aitken,albon,ashley,barrichello,bottas,
boutsen,brise,brundle,bruno_senna,button,
cogan,coulthard,daly,damon_hill,
desire_wilson,frentzen,gene,heidfeld,
hulkenberg,ian_scheckter,jones,keegan,
keke_rosberg,kubica,laffite,latifi,lees,
lombardi,magee,maldonado,mansell,
mario_andretti,massa,merzario,migault,
montoya,nakajima,palmer,patrese,piquet,
pizzonia,prost,ralf_schumacher,regazzoni,
resta,reutemann,rosberg,russell,schlesser,
senna,sirotkin,stroll,villeneuve,vonlanthen,
webber,wurz,zanardi,zapico,zorzi
AlphaTauri           gasly,kvyat,tsunoda
Haas F1 Team         grosjean,gutierrez,kevin_magnussen,mazepin,
mick_schumacher,pietro_fittipaldi
10 rows selected
>
```

如果你想查看已执行的 SQL，请将该语句作为 `dbms_utility.``expand_sql_text` 过程的输入。



### UTL_CALL_STACK

要展示以下函数的运行结果，我们使用代码清单 17-8 中的代码模板。特定于不同函数的代码必须位于这些行之间。

```
-----8<-replace-start------
```

和

```
create or replace package utl_call_stack_demo
is
procedure public_procedure;
end utl_call_stack_demo;
/
create or replace package body utl_call_stack_demo is
procedure private_procedure is
l_subprogram utl_call_stack.unit_qualified_name;
procedure local_procedure_in_private_procedure is
l_subprogram utl_call_stack.unit_qualified_name;
procedure local_procedure_in_local_procedure is
l_subprogram utl_call_stack.unit_qualified_name;
begin
-----8<-replace-start------
null;
------8<-replace-end------
end local_procedure_in_local_procedure;
begin
-----8<-replace-start------
null;
------8<-replace-end------
dbms_output.put_line( 'Call the local in local procedure');
local_procedure_in_local_procedure;
end local_procedure_in_private_procedure;
begin
-----8<-replace-start------
null;
------8<-replace-end------
dbms_output.put_line( 'Call the local in private procedure');
local_procedure_in_private_procedure;
end private_procedure;
procedure public_procedure is
l_subprogram utl_call_stack.unit_qualified_name;
procedure local_procedure_in_public_procedure is
l_subprogram utl_call_stack.unit_qualified_name;
procedure local_procedure_in_local_procedure is
l_subprogram utl_call_stack.unit_qualified_name;
begin
-----8<-replace-start------
null;
------8<-replace-end------
end local_procedure_in_local_procedure;
begin
-----8<-replace-start------
null;
------8<-replace-end------
dbms_output.put_line( 'Call the local in local procedure');
local_procedure_in_local_procedure;
end local_procedure_in_public_procedure;
begin
-----8<-replace-start------
null;
------8<-replace-end------
dbms_output.put_line( 'Call the local procedure' );
local_procedure_in_public_procedure;
dbms_output.put_line( 'Call the private procedure' );
private_procedure;
end public_procedure;
end utl_call_stack_demo;
/
代码清单 17-8
utl_call_stack_demo 模板
```

```
------8<-replace-end------
```

### subprogram

函数 `utl_call_stack.subprogram` 返回一个可变数组，其中包含从你作为参数传递的层级开始的所有程序名称。层级 1 是当前程序，层级 2 是“父”程序，依此类推。可变数组是从底向上填充的，其中当前程序是数组中的最后一项。

为了演示此函数的工作原理，请将

```
-----8<-replace-start------
null;
------8<-replace-end------
```

替换为

```
-----8<-replace-start------
l_subprogram := utl_call_stack.subprogram( 1 );
dbms_output.put_line('subprograms ');
for indx in l_subprogram.first .. l_subprogram.last
loop
dbms_output.put_line(to_char(indx) || ')' ||
l_subprogram(indx));
end loop;
------8<-replace-end------
```

如果你调用公共过程，输出如下：

```
begin
utl_call_stack_demo_subprogram.public_procedure;
end;
/
subprograms
1)UTL_CALL_STACK_DEMO
2)PUBLIC_PROCEDURE
调用本地过程
subprograms
1)UTL_CALL_STACK_DEMO
2)PUBLIC_PROCEDURE
3)LOCAL_PROCEDURE_IN_PUBLIC_PROCEDURE
调用本地过程中的本地过程
subprograms
1)UTL_CALL_STACK_DEMO
2)PUBLIC_PROCEDURE
3)LOCAL_PROCEDURE_IN_PUBLIC_PROCEDURE
4)LOCAL_PROCEDURE_IN_LOCAL_PROCEDURE
调用私有过程
subprograms
1)UTL_CALL_STACK_DEMO
2)PRIVATE_PROCEDURE
调用私有过程中的本地过程
subprograms
1)UTL_CALL_STACK_DEMO
2)PRIVATE_PROCEDURE
3)LOCAL_PROCEDURE_IN_PRIVATE_PROCEDURE
调用本地过程中的本地过程
subprograms
1)UTL_CALL_STACK_DEMO
2)PRIVATE_PROCEDURE
3)LOCAL_PROCEDURE_IN_PRIVATE_PROCEDURE
4)LOCAL_PROCEDURE_IN_LOCAL_PROCEDURE
PL/SQL 过程已成功完成
```


### `concatenate_subprogram`

除了分别获取各个子程序名称外，你也可以方便地将子程序的变长数组（varray）连接成一个 `VARCHAR2` 字符串，其中包含以点号分隔的单元限定名称。`utl_call_stack.concatenate_subprogram` 过程的输入参数是一个变长数组，你可以使用 `utl_call_stack.subprogram` 过程来获取它。

以下是一个调用此过程的示例：

```
utl_call_stack.concatenate_subprogram
( utl_call_stack.subprogram( 1 ) );
```

为了演示此函数的功能，请将以下代码：

```
-----8<-replace-start------
null;
------8<-replace-end------
```

替换为：

```
-----8<-replace-start------
dbms_output.put_line
( utl_call_stack.concatenate_subprogram
( utl_call_stack.subprogram( 1 ) )
);
------8<-replace-end------
```

如果你调用公共过程，输出如下：

```
begin
utl_call_stack_demo_concatenate_subprogram.public_procedure;
end;
/
UTL_CALL_STACK_DEMO.PUBLIC_PROCEDURE
调用局部过程
UTL_CALL_STACK_DEMO.PUBLIC_PROCEDURE.LOCAL_PROCEDURE_IN_PUBLIC_PROCEDURE
调用局部过程中的局部过程
UTL_CALL_STACK_DEMO.PUBLIC_PROCEDURE.LOCAL_PROCEDURE_IN_PUBLIC_PROCEDURE.LOCAL_PROCEDURE_IN_LOCAL_PROCEDURE
调用私有过程
UTL_CALL_STACK_DEMO.PRIVATE_PROCEDURE
调用私有过程中的局部过程
UTL_CALL_STACK_DEMO.PRIVATE_PROCEDURE.LOCAL_PROCEDURE_IN_PRIVATE_PROCEDURE
调用局部过程中的局部过程
UTL_CALL_STACK_DEMO.PRIVATE_PROCEDURE.LOCAL_PROCEDURE_IN_PRIVATE_PROCEDURE.LOCAL_PROCEDURE_IN_LOCAL_PROCEDURE
PL/SQL 过程已成功完成。
```

### `owner`

`utl_call_stack.owner` 函数返回子程序的所有者名称。其参数与你用于 `utl_call_stack.subprogram` 函数的参数相同。此函数的返回值是单个 `VARCHAR2` 字符串，表示给定层级的所有者，而不是整个堆栈的变长数组。

### `unit_line`

使用 `utl_call_stack.unit_line`，你可以获取对程序进行调用的位置的行号。这些行号是调用堆栈中不同程序里的行号，而不仅仅是当前程序的行号。

为了演示此函数的功能，请将以下代码：

```
-----8<-replace-start------
null;
------8<-replace-end------
```

替换为：

```
-----8<-replace-start------
for i in 1 .. utl_call_stack.dynamic_depth loop
  if i > 1 then
    dbms_output.put('->');
  end if;
  dbms_output.put(to_char(utl_call_stack.unit_line(i)));
end loop;
dbms_output.put_line('');
------8<-replace-end------
```

如果你调用公共过程，输出如下：

```
begin
utl_call_stack_demo_unit_line.public_procedure;
end;
/
2->97
调用局部过程
2->107->81
调用局部过程中的局部过程
2->107->91->66
调用私有过程
2->109->44
调用私有过程中的局部过程
2->109->54->28
调用局部过程中的局部过程
2->109->54->38->13
PL/SQL 过程已成功完成。
```

### 摘要

有很多可用的工具包。使用 `DBMS_SESSION` 中的过程和函数，你可以确保设置了正确的角色以及 `NLS` 设置是正确的。为了便于调试，可以设置会话标识符。使用 `DBMS_UTILITY` 和 `UTL_CALL_STACK` 中的过程和函数，你可以精确查询程序当前的位置以及它是如何到达那里的。使用这些过程和函数来检测你的代码，以便在发生意外情况时，能够弄清楚是什么、何时以及为什么发生的。

## 18. 查看你需要看到的内容

如果你将数据存储在数据库中，它并不需要随时对所有人可见。有时你不想让人们看到你的数据。有时你希望某些群体看到你的数据，而其他群体看不到。有时你希望你的数据在特定时间段内可用。Oracle 数据库为这些问题提供了不同的机制。

### 时间有效性

有时我们数据库中的数据只在特定时间段内有效。你可以简单地为新时间段将行更新为新值。但如果你需要能够检索某个时间点的数据呢？这意味着需要引入历史表，其中记录了表的所有更新，然后创建查询来检索所需的数据。

### 设置

Oracle Database 12c (12.1) 引入了 `temporal validity`（时间有效性）的概念。通过在表中使用两个 DATE 或 TIMESTAMP 列，Oracle Database 可以确定一条记录是否应包含在查询的结果集中。这些列可以是已有的列，也可以在创建有效期时自动创建。

假设你创建了一个关于一级方程式赛车手及其合同的表。使用可用数据创建一个表。这并不意味着结果是完全准确的，但对本示例而言是可行的。

我们的假设如下：

*   车手为某车队参加的第一场比赛是合同开始日期。
*   车手为某车队参加的最后一场比赛是合同结束日期。
*   只显示 1965 年以来的合同（在此之前，可用数据不够准确）。

在这种情况下，你可以采用两种方法。一种是使用 `driverid` 和 `constructorid` 创建一个表，为合同添加一个有效期，这会自动添加开始和结束日期列，然后使用查询来填充表。

另一种可能性是创建表时就已经包含开始和结束日期列（例如，通过运行一个 `Create Table As Select` 语句），然后使用现有列添加合同的有效期。

这两种方法的主要区别在于，当列是自动创建时，它们被创建为不可见列，在描述表时不可见。新创建列的名称为 `<periodname>_start` 和 `<periodname>_end`。

在向表添加有效期时创建的检查约束会检查有效期的开始日期是否小于结束日期。如果任何一个值为 NULL，则该约束也被视为有效。

使用清单 [18-1] 中的脚本创建表并添加有效期。

```
create table driverconstructors as
with resultrace as
( select rsl.driverid
, rsl.constructorid
, rcs.year
, rcs.round
, rcs.race_date
from   f1data.results rsl
join   f1data.races   rcs
on     ( rsl.raceid = rcs.raceid )
where  1 = 1
and    rcs.race_date > to_date('19650101', 'YYYYMMDD')
)
, contractinfo as
( select rrc.driverid
, rrc.constructorid
, rrc.year
, rrc.round
, case
when lag( rrc.constructorid )
over( partition by rrc.driverid order by year
,round ) = rrc.constructorid
then
null
else
rrc.race_date
end begincontract
, case
when lead( rrc.constructorid )
over( partition by rrc.driverid order by year
,round ) = rrc.constructorid
then
null
else
rrc.race_date
end endcontract
from   resultrace rrc
)
, contracts as
( select cif.driverid
, cif.constructorid
, cif.year
, cif.round
, cif.begincontract
, cif.endcontract
, sum( case
when cif.begincontract is null
then

else

end )
over ( partition by driverid order by year
, round) contractnumber
from   contractinfo cif
)
, all_contracts as
( select con.contractnumber
, con.driverid
, con.constructorid
, min(con.begincontract) as startcontract
, max(con.endcontract) as endcontract
from   contracts con
group  by con.driverid
, con.constructorid
, con.contractnumber)
select act.driverid
, act.constructorid
, act.startcontract
, greatest( act.startcontract + 1 / 24 / 60 / 60
, act.endcontract) as endcontract
from   all_contracts act
/
```

**清单 18-1**
创建 `drivercontracts` 表

创建表后，使用两个现有列 `startcontract` 和 `endcontract` 向此表添加有效期。

```
alter table driverconstructors add period for drivercontract (startcontract, endcontract)
/
```

如果有效期的开始日期小于或等于指定日期，且有效期的结束日期大于指定日期，则该记录属于有效期内。日期的 NULL 值被视为无限小。对于开始日期，这意味着时间的起点；对于结束日期，这意味着时间的终点。

### “As of” 查询

以下查询允许你从表中仅检索在指定日期有效的记录。

```
select ... from <table_name> as of period for <period_name> <date_expression>
```

使用在 `DRIVERCONSTRUCTORS` 表上定义的周期，你可以查找在某个特定时间点有合同的车手。让我们查看 2022 赛季开始时的情况。

```
select d.forename as firstname
, d.surname  as lastname
, c.name     as constructorname
from   driverconstructors as of period
for    drivercontract to_date( '20220320', 'YYYYMMDD' ) dc
join   f1data.drivers d
on     (dc.driverid = d.driverid)
join   f1data.constructors c
on     (dc.constructorid = c.constructorid)
order  by constructorname, lastname, firstname
/
FIRSTNAME    LASTNAME     CONSTRUCTORNAME
------------ ------------ ----------------
Valtteri     Bottas       Alfa Romeo
Guanyu       Zhou         Alfa Romeo
Pierre       Gasly        AlphaTauri
Yuki         Tsunoda      AlphaTauri
Fernando     Alonso       Alpine F1 Team
Esteban      Ocon         Alpine F1 Team
Nico         Hülkenberg   Aston Martin
Lance        Stroll       Aston Martin
Sebastian    Vettel       Aston Martin
Charles      Leclerc      Ferrari
Carlos       Sainz        Ferrari
Kevin        Magnussen    Haas F1 Team
Mick         Schumacher   Haas F1 Team
Lando        Norris       McLaren
Daniel       Ricciardo    McLaren
Lewis        Hamilton     Mercedes
George       Russell      Mercedes
Sergio       Pérez        Red Bull
Max          Verstappen   Red Bull
Alexander    Albon        Williams
Nicholas     Latifi       Williams
21 rows selected
```

你可以为 “as of” 周期使用任何日期。因此，如果你想查看 1998 赛季开始时的车手，查询将如下所示。

```
select d.forename as firstname
, d.surname  as lastname
, c.name     as constructorname
from   driverconstructors as of period
for    drivercontract to_date( '19980308', 'YYYYMMDD' ) dc
join   f1data.drivers d
on     (dc.driverid = d.driverid)
join   f1data.constructors c
on     (dc.constructorid = c.constructorid)
order  by constructorname, lastname, firstname
/
FIRSTNAME    LASTNAME     CONSTRUCTORNAME
------------ ------------ ----------------
Pedro        Diniz        Arrows
Mika         Salo         Arrows
Giancarlo    Fisichella   Benetton
Alexander    Wurz         Benetton
Eddie        Irvine       Ferrari
Michael      Schumacher   Ferrari
Damon        Hill         Jordan
Ralf         Schumacher   Jordan
David        Coulthard    McLaren
Mika         Häkkinen     McLaren
Tarso        Marques      Minardi
Shinji       Nakano       Minardi
Esteban      Tuero        Minardi
Olivier      Panis        Prost
Jarno        Trulli       Prost
Jean         Alesi        Sauber
Johnny       Herbert      Sauber
Rubens       Barrichello  Stewart
Jan          Magnussen    Stewart
Ricardo      Rosset       Tyrrell
Toranosuke   Takagi       Tyrrell
Heinz-Harald Frentzen     Williams
Jacques      Villeneuve   Williams
23 rows selected
```

> **提示：**
> “as of”的一个典型用法是查询 `as of period for ... sysdate` 以检索当前有效的数据。


#### 查询间的版本 (Versions Between Queries)

如果你想查看某位车手曾效力于哪些车队，可以使用 `versions between` 构造。若想查看基米·莱科宁 (Kimi Räikkönen) 的合同记录，请使用以下查询。

```sql
select d.forename as firstname
, d.surname  as lastname
, c.name     as constructorname
, dc.startcontract
, dc.endcontract
from   driverconstructors versions period for drivercontract
between to_date( '19900101', 'YYYYMMDD' )
and     to_date( '20230101', 'YYYYMMDD' ) dc
join   f1data.drivers d
on     (dc.driverid = d.driverid)
join   f1data.constructors c
on     (dc.constructorid = c.constructorid)
where  d.surname = 'Räikkönen'
order  by dc.startcontract
, dc.endcontract
/
FIRSTNAME   LASTNAME    CONSTRUCTORNAME STARTCONTRACT ENDCONTRACT
----------- ----------- --------------- ------------- -----------
Kimi        Räikkönen   Sauber          04/03/2001    14/10/2001
Kimi        Räikkönen   McLaren         03/03/2002    22/10/2006
Kimi        Räikkönen   Ferrari         18/03/2007    01/11/2009
Kimi        Räikkönen   Lotus F1        18/03/2012    03/11/2013
Kimi        Räikkönen   Ferrari         16/03/2014    25/11/2018
Kimi        Räikkönen   Alfa Romeo      17/03/2019    12/12/2021
6 rows selected
```

你也可以查询哪些车手曾效力于某个特定车队。

```sql
select d.forename as firstname
, d.surname  as lastname
, c.name     as constructorname
, dc.startcontract
, dc.endcontract
from   driverconstructors versions period for drivercontract
between to_date( '19900101', 'YYYYMMDD' )
and    to_date( '20230101', 'YYYYMMDD' ) dc
join   f1data.drivers d
on     (dc.driverid = d.driverid)
join   f1data.constructors c
on     (dc.constructorid = c.constructorid)
where  c.name = 'Red Bull'
order  by dc.startcontract
, dc.endcontract
/
FIRSTNAME    LASTNAME   CONSTRUCTORNAME STARTCONTRACT ENDCONTRACT
------------ ---------- --------------- ------------- -----------
Christian    Klien      Red Bull        06/03/2005    10/09/2006
David        Coulthard  Red Bull        06/03/2005    02/11/2008
Vitantonio   Liuzzi     Red Bull        24/04/2005    29/05/2005
Robert       Doornbos   Red Bull        01/10/2006    22/10/2006
Mark         Webber     Red Bull        18/03/2007    24/11/2013
Sebastian    Vettel     Red Bull        29/03/2009    23/11/2014
Daniel       Ricciardo  Red Bull        16/03/2014    25/11/2018
Daniil       Kvyat      Red Bull        15/03/2015    01/05/2016
Max          Verstappen Red Bull        15/05/2016    03/07/2022
Pierre       Gasly      Red Bull        17/03/2019    04/08/2019
Alexander    Albon      Red Bull        01/09/2019    13/12/2020
Sergio       Pérez      Red Bull        28/03/2021    03/07/2022
12 rows selected
```

### DBMS_FLASHBACK_ARCHIVE

使用 `dbms_flashback_archive.enable_at_valid_time`，可以将有效性日期设置为一个固定日期，这样就不必在每个查询中添加 `as of period` 子句。该过程的参数在表 18-1 中描述。

表 18-1
enable_at_valid_time 过程参数

| 参数 (Parameter) | 必填 (Mandatory) | 描述 (Description) |
| --- | --- | --- |
| level | Y | 选项：− ALL 显示所有数据（默认值）− CURRENT 仅显示在当前时间戳有效的数据。数据会随时间推移而变得可见或不可见。− ASOF 显示在下一个参数设置的时间戳有效的数据。 |
| query_time |   | 仅在 level 为 ASOF 时使用仅显示有效数据 |

将闪回归档设置为 2022 赛季开始时。

```sql
begin
dbms_flashback_archive.enable_at_valid_time
( level      => 'ASOF'
, query_time => to_date( '20220320','YYYYMMDD' )
);
end;
/
```

你可以使用一个简单的查询（注意这里没有 `as of period` 子句）来获得与第一个示例相同的结果。

```sql
select d.forename as firstname
, d.surname  as lastname
, c.name     as constructorname
from   driverconstructors  dc
join   f1data.drivers      d
on     (dc.driverid = d.driverid)
join   f1data.constructors c
on     (dc.constructorid = c.constructorid)
order  by constructorname, lastname, firstname
/
FIRSTNAME    LASTNAME     CONSTRUCTORNAM
------------ ------------ --------------
Valtteri     Bottas       Alfa Romeo
Guanyu       Zhou         Alfa Romeo
Pierre       Gasly        AlphaTauri
Yuki         Tsunoda      AlphaTauri
Fernando     Alonso       Alpine F1 Team
Esteban      Ocon         Alpine F1 Team
Nico         Hülkenberg   Aston Martin
Lance        Stroll       Aston Martin
Sebastian    Vettel       Aston Martin
Charles      Leclerc      Ferrari
Carlos       Sainz        Ferrari
Kevin        Magnussen    Haas F1 Team
Mick         Schumacher   Haas F1 Team
Lando        Norris       McLaren
Daniel       Ricciardo    McLaren
Lewis        Hamilton     Mercedes
George       Russell      Mercedes
Sergio       Pérez        Red Bull
Max          Verstappen   Red Bull
Alexander    Albon        Williams
Nicholas     Latifi       Williams
21 rows selected
```

要恢复到“正常”操作，调用将 level 设置为 `ALL` 的 `dbms_flashback_archive.enable_at_valid_time`，或调用 `dbms_flashback_archive.disable_asof_valid_time` 过程。

```sql
begin
dbms_flashback_archive.disable_asof_valid_time;
end;
/
```

### 虚拟专用数据库 (Virtual Private Database)

虚拟专用数据库 (VPD) 使得每个终端用户都好像拥有自己的数据库，而实际上仍然使用单一的代码库和表集。假设一级方程式车队不允许查看其他车队的测试结果（实际情况并非如此，但在此示例中假设如此）。你可以在数据库中构建不同的模式，每个模式都有自己的表。当添加一场比赛的结果时，你必须将数据添加到正确的模式中。更简单的方法是在一个模式中创建一个表，并让 Oracle 数据库显示正确的数据。

当表是 VPD 的一部分时，Oracle 数据库会在底层向针对该表发出的任何 SQL 语句添加一个额外的谓词。这可以是从该表进行的单次 `select` 查询，也可以是当该表用于连接或子查询时。要定义这个额外的谓词，你必须先定义一个策略函数，然后将此函数与表关联。

#### 策略函数 (The Policy Function)

Oracle 数据库按顺序以两个参数 `schemaname` 和 `tablename` 调用此过程。VPD 使用的策略函数签名如下。

```sql
function 
(  in varchar2
,   in varchar2)
return varchar2
```

该函数必须返回一个有效的谓词，该谓词将被添加到针对此表发出的任何 SQL 语句中。

你可以在策略函数中执行任何操作，但它会被非常频繁地调用，如果函数需要很长时间才能完成，那么针对该表的每个 SQL 语句也会花费很长时间。

代码清单 18-2 展示了一个策略函数，用于确保一个车队只能看到其自身的数据。

```sql
create or replace function thisconstructoronly
( schemaname_in in varchar2
, tablename_in  in varchar2
) return varchar2
is
begin
return q'[constructorid = sys_context
( 'CONSTRUCTOR_CTX'
, 'CONSTRUCTORID'
)]';
end;
/
代码清单 18-2
策略函数 thisconstructoronly
```

此函数返回一个包含以下内容的 VARCHAR2。

```sql
constructorid = sys_context( 'CONSTRUCTOR_CTX', 'CONSTRUCTORID' )
```



#### 上下文

将外部设置传递给策略函数或其生成的谓词的最简单（也是最快）的方法是使用 `应用上下文`。

你可以使用以下命令创建一个上下文。

```sql
create or replace context constructor_ctx using context_pkg
/
```

然后，你需要构建一个包来在上下文中设置（或清除）属性。这些属性必须使用关联的包来设置。清单 18-3 展示了用于与上下文通信的包。

```sql
create or replace package context_pkg as
procedure set_constructor( constructorid_in in number );
procedure unset_constructor;
end;
/
create or replace package body context_pkg as
procedure set_constructor(constructorid_in in number)
is
begin
dbms_session.set_context( namespace => 'CONSTRUCTOR_CTX'
, attribute => 'CONSTRUCTORID'
, value     => constructorid_in
);
end set_constructor;
procedure unset_constructor
is
begin
dbms_session.clear_context( namespace => 'CONSTRUCTOR_CTX' );
end unset_constructor;
end context_pkg;
/
```

清单 18-3
用于与 `constructor_ctx` 通信的 `context_pkg`

#### 策略

现在，必须将这个谓词添加到针对该表的每条语句中。你需要创建一个 VPD 策略，将策略函数连接到表上。

#### add_policy

`dbms_rls.add_policy` 过程可以接受表 18-2 中描述的参数。

表 18-3
策略类型

| 类型 | 描述 |
| --- | --- |
| `DYNAMIC` | 此策略类型在用户每次访问受 VPD 保护的数据库对象时运行策略函数。 |
| `STATIC` | 此策略类型运行策略函数一次，然后将结果在 SGA 中共享。静态策略类型对实例中的所有用户强制执行相同的谓词。 |
| `SHARED_STATIC` | 你可以在多个表上使用相同的策略函数。 |
| `CONTEXT_SENSITIVE` | 如果本地应用上下文没有变化，Oracle 数据库在用户会话中不会重新运行策略函数。如果用户会话期间任何应用上下文属性发生变化（甚至不必被引用）；默认情况下，数据库会重新执行策略函数以确保其捕获到自初始解析以来谓词的所有更改。 |
| `SHARED_CONTEXT_SENSITIVE` | 当多个表使用相同的策略函数时使用此类型。 |

表 18-2
添加策略过程的参数

| 参数 | 是否必需 | 描述 |
| --- | --- | --- |
| `object_schema` | | 要保护的对象所在的模式。如果为 null，则假定为当前模式。 |
| `object_name` | Y | 要保护的对象的名称。 |
| `policy_name` | Y | 任意选择的名称。使用此名称来启用/禁用和删除策略。 |
| `function_schema` | | 策略函数所在的模式。如果为 null，则假定为当前模式。 |
| `policy_function` | Y | 策略函数的名称。 |
| `statement_types` | | 策略应用到的语句类型，`select`、`insert`、`update`、`delete` 或 `index`。默认为除 `index` 外的任何类型。 |
| `update_check` | | 根据策略检查更新或插入的值。根据策略，能够在语句执行后看到信息。 |
| `Enable` | | 策略是否启用？ |
| `static_policy` | | 策略是否总是产生相同的谓词？ |
| `policy_type` | | 从以下选项中选择：`DYNAMIC`、`STATIC`、`SHARED_STATIC`、`CONTEXT_SENSITIVE`、`SHARED_CONTEXT_SENSITIVE`。如果非 null，则覆盖 `static_policy`。（有关这些类型的说明，请参见表 18-3。） |
| `long_predicate` | | 返回的谓词最大长度为 4000 字节（默认）或 32K。 |
| `sec_relevant_cols` | | 安全相关列的列表。当引用其中一列时，会执行策略函数。 |
| `sec_relevant_cols_opt` | | 如果设置为 `dbms_rls.all_rows`，则返回所有行，但将安全相关列选项中指定的列置为 null。 |
| `namespace` | | 应用上下文命名空间的名称。 |
| `attribute` | | 应用上下文属性的名称。 |

如果你查询 constructors 表，你会看到所有构造器的信息。

```sql
select c.constructorid
, c.name
, c.nationality
from   f1data.constructors c
/
```

```
CONST NAME            NATIONALITY
----- --------------- ---------------
1 McLaren         British
2 BMW Sauber      German
3 Williams        British
4 Renault         French
5 Toro Rosso      Italian
6 Ferrari         Italian
7 Toyota          Japanese
8 Super Aguri     Japanese
9 Red Bull        Austrian
10 Force India     Indian
...
```

```sql
select cr.constructorresultsid as crid
, cr.constructorid        as cid
, cr.raceid               as rid
, cr.points               as points
from   f1data.constructorresults cr
/
```

```
CRID   CID   RID POINT
----- ----- ----- -----
1197     6   133    16
1198     3   133     4
1199    16   133     5
1200    15   133     1
1201    17   133     0
1202     1   133     0
1203    19   133     0
1204     4   133     0
1205    21   133     0
1206     7   133     0
...
```

```sql
select cs.constructorstandingsid as csid
, cs.constructorid          as cid
, cs.position               as pos
, cs.points                 as pnts
from   f1data.constructorstandings cs
where  rownum < 11
/
```

```
CSID   CID   POS  PNTS
----- ----- ----- -----
6242    15     6    18
6241     1     5    32
6240    16     3    67
6239     3     4    41
6238     4     2    79
6237     6     1   174
6256    18    10     1
6255    17     9     5
6254     7     7     8
6253    19     8     7
...
```

因为这三个表都包含构造器信息（`CONSTRUCTORS`、`CONSTRUCTORRESULTS` 和 `CONSTRUCTORSTANDINGS`），并且你希望对所有三个表使用相同的策略，所以添加三个共享静态策略。

```sql
begin
sys.dbms_rls.add_policy
( object_schema   => 'F1DATA'
, object_name     => 'CONSTRUCTORS'
, policy_name     => 'THISCONSTRUCTOR'
, function_schema => 'BOOK'
, policy_function => 'THISCONSTRUCTORONLY'
, policy_type     => sys.dbms_rls.shared_static
);
sys.dbms_rls.add_policy
( object_schema   => 'F1DATA'
, object_name     => 'CONSTRUCTORRESULTS'
, policy_name     => 'THISCONSTRUCTORRESULT'
, function_schema => 'BOOK'
, policy_function => 'THISCONSTRUCTORONLY'
, policy_type     => sys.dbms_rls.shared_static
);
sys.dbms_rls.add_policy
( object_schema   => 'F1DATA'
, object_name     => 'CONSTRUCTORSTANDINGS'
, policy_name     => 'THISCONSTRUCTORSTANDING'
, function_schema => 'BOOK'
, policy_function => 'THISCONSTRUCTORONLY'
, policy_type     => sys.dbms_rls.shared_static
);
end;
/
```

即使策略函数返回引用上下文的谓词，它也不依赖于上下文。即使上下文发生变化，谓词文本本身仍然相同。它会根据上下文中的不同值进行不同的求值。你可以通过调用我们的 `set_constructor` 过程来设置上下文。

```sql
begin
context_pkg.set_constructor( constructorid_in => 9 );
end;
/
```

当从表中选择数据时，谓词会自动添加到查询中，因此只显示红牛车队（Red Bull）的数据。


### 数据库策略与上下文管理

#### 查询示例

以下查询展示了构造函数（constructor）相关数据的检索。

```
select c.constructorid as cid
, c.name          as name
, c.nationality   as nationality
from   f1data.constructors c
/
CID NAME            NATIONALITY
----- --------------- ---------------
9 Red Bull        Austrian
select cr.constructorresultsid as crid
, cr.constructorid        as cid
, cr.raceid               as rid
, cr.points               as points
from   f1data.constructorresults cr
/
CRID   CID   RID POINT
----- ----- ----- -----
9     9    18     0
16     9    19     2
27     9    20     2
37     9    21     4
49     9    22     2
58     9    23     5
66     9    24     6
79     9    25     3
92     9    26     0
102     9    27     0
...
select cs.constructorstandingsid as csid
, cs.constructorid          as cid
, cs.position               as pos
, cs.points                 as pnts
from   f1data.constructorstandings cs
/
CSID   CID   POS  PNTS
----- ----- ----- -----
14     9     7     2
25     9     7     4
36     9     6     8
47     9     5    10
58     9     5    15
69     9     4    21
80     9     4    24
91     9     5    24
102     9     5    24
113     9     6    24
...
```

如果未设置或取消设置上下文，查询将不返回数据。

```
begin
context_pkg.unset_constructor;
end;
/
select c.constructorid as cid
, c.name          as name
, c.nationality   as nationality
from   f1data.constructors c
/
CID NAME            NATIONALITY
----- --------------- ---------------
select cr.constructorresultsid as crid
, cr.constructorid        as cid
, cr.raceid               as rid
, cr.points               as points
from   f1data.constructorresults cr
/
CRID   CID   RID POINT
----- ----- ----- -----
select cs.constructorstandingsid as csid
, cs.constructorid          as cid
, cs.position               as pos
, cs.points                 as pnts
from   f1data.constructorstandings cs
/
CSID   CID   POS  PNTS
----- ----- ----- -----
```

#### 策略管理

##### 启用策略

要启用或禁用策略，需使用 `dbms_rls.enable_policy` 过程。

`enable_policy` 过程可接受表 18-4 中描述的参数。

表 18-4

启用策略过程的参数

| 参数 | 是否必需 | 描述 |
| --- | --- | --- |
| object_schema | | 受保护对象所在的模式。如果为 null，则假定为当前模式 |
| object_name | Y | 受保护对象的名称 |
| policy_name | Y | 要启用或禁用的策略名称 |
| Enable | | 使用 TRUE 启用策略，FALSE 禁用策略 |

##### 删除策略

当不再需要策略或需要以不同选项重新创建策略时，可使用 `dbms_rls.drop_policy` 删除它。

`drop_policy` 过程可接受表 18-5 中描述的参数。

表 18-5

删除策略过程的参数

| 参数 | 是否必需 | 描述 |
| --- | --- | --- |
| object_schema | | 受保护对象所在的模式。如果为 null，则假定为当前模式 |
| object_name | Y | 受保护对象的名称 |
| policy_name | Y | 要启用或禁用的策略名称 |

##### 修改策略

可使用 `dbms_rls.alter_policy` 将某些属性与现有策略关联或取消关联。

`alter_policy` 过程可接受表 18-6 中描述的参数。

表 18-6

启用策略过程的参数

| 参数 | 是否必需 | 描述 |
| --- | --- | --- |
| object_schema | | 受保护对象所在的模式。如果为 null，则假定为当前模式 |
| object_name | Y | 受保护对象的名称 |
| policy_name | Y | 要启用或禁用的策略名称 |
| alter_option | | 关联的添加或移除 `dbms_rls.add_attribute_association` `dbms_rls.remove_attribute_association` |
| namespace | Y | 上下文的名称空间 |
| attribute | Y | 名称空间内的属性 |

### 复杂策略

你可以使用所有 SQL 技能来构建所需的任何谓词。但请注意，此谓词会被添加到语句中，并针对表中的每一行执行，而不仅仅是结果集中的行。因此，如果你构建的内容耗时很长，那么它将乘以表中的行数而耗时很长。尽量保持策略为纯 SQL。

如果希望构造函数（constructor）只看到其自身车手的圈速，可以使用清单 18-4 中显示的代码。

```
create or replace context constructor_ctx using context_pkg
/
create or replace package context_pkg as
procedure set_constructor( constructorid_in in number );
procedure unset_constructor;
end context_pkg;
/
create or replace package body context_pkg as
procedure set_constructor(constructorid_in in number) is
begin
dbms_session.set_context( namespace => 'CONSTRUCTOR_CTX'
, attribute => 'CONSTRUCTORID'
, value     => constructorid_in
);
End set_constructor;
procedure unset_constructor is
begin
dbms_session.clear_context( namespace => 'CONSTRUCTOR_CTX' );
end unset_constructor;
end context_pkg;
/
begin
context_pkg.set_constructor( constructorid_in => 9 );
end;
/
create or replace function lapsforthisconstructoronly
( schemaname_in in varchar2
, tablename_in  in varchar2
) return varchar2
is
begin
return q'[(raceid, driverid) in
(select r.raceid, rs.driverid
from   f1data.races r
join   f1data.results rs
on     (r.raceid = rs.raceid)
where  rs.constructorid =
sys_context( 'CONSTRUCTOR_CTX'
, 'CONSTRUCTORID' ) )]';
end lapsforthisconstructoronly;
/
select lapsforthisconstructoronly(  null, null ) from dual
/
THISCONSTRUCTORONLY(NULL,NULL)

(raceid, driverid) in
(select r.raceid, rs.driverid
from   f1data.races r
join   f1data.results rs
on     (r.raceid = rs.raceid)
where  rs.constructorid =
sys_context( 'CONSTRUCTOR_CTX'
, 'CONSTRUCTORID' ) )
begin
sys.dbms_rls.add_policy
( object_schema   => 'F1DATA'
, object_name     => 'LAPTIMES'
, policy_name     => 'THISCONSTRUCTORSLAPTIMES'
, function_schema => 'BOOK'
, policy_function => 'LAPSFORTHISCONSTRUCTORONLY'
, policy_type     => sys.dbms_rls.STATIC
);
end;
/
清单 18-4
构造函数只能看到其自身车手的信息
```

### 数据脱敏

在数据库中脱敏数据可确保数据仅由被授权者查看，从而使隐私规则和法规的合规性保持一致。脱敏是通过在将结果发送回调用进程时屏蔽（部分）数据来实现的。对持久化的数据不作任何更改。

假设你有一个 Web 应用程序，用于显示特定比赛中车手的排名。车手必须登录才能查看圈速。通过登录，会设置一个包含 `driverid` 的上下文。基于此上下文值已设置，时间和毫秒列会被显示；否则，这些列会被遮蔽。

首先，创建一个包含必要包的上下文，如清单 18-5 所示。

```
create or replace context driver_ctx using driver_pkg
/
create or replace package driver_pkg as
procedure set_driver( driverid_in in number );
procedure unset_driver;
end driver_pkg;
/
create or replace package body driver_pkg as
procedure set_driver(driverid_in in number) is
begin
sys.dbms_session.set_context
( namespace => 'driver_ctx'
, attribute => 'driverid'
, value     => driverid_in
);
end set_driver;
procedure unset_driver is
begin
sys.dbms_session.clear_context
( namespace => 'driver_ctx' );
end unset_driver;
end driver_pkg;
/
清单 18-5
车手上下文
```

## add_policy

要向表添加屏蔽策略，需要调用 `dbms_redact.add_policy` 过程。此过程的参数如表 **18-7** 所述。

**表 18-7**

`dbms_redact.add_policy` 参数

| 参数 | 是否必需 | 描述 |
| --- | --- | --- |
| `object_schema` | | 要保护的对象所在的模式。如果为 NULL，则假定为当前模式 |
| `object_name` | 是 | 要保护的对象的名称 |
| `policy_name` | 是 | 任意选择的名称，用于启用/禁用、更改和删除策略 |
| `column_name` | | 策略应用到的单个列名。如果要屏蔽多个列，请使用 `ALTER_POLICY` 过程添加额外的列 |
| `function_type` | 是 | 要使用的屏蔽函数类型。<br>可能的值有：<br>- `DBMS_REDACT.NONE`<br>- `DBMS_REDACT.FULL`（默认）<br>- `DBMS_REDACT.PARTIAL`<br>- `DBMS_REDACT.RANDOM`<br>- `DBMS_REDACT.REGEXP`<br>如果 `function_type` 是 `DBMS_REDACT.REGEXP`，则必须完全省略 `function_parameters`，且 `regexp_*` 参数必须定义数据屏蔽策略 |
| `function_parameters` | | 屏蔽函数的参数。<br>值取决于提供的 `function_type`：<br>– `DBMS_REDACT.NONE`：可以完全省略，默认为 NULL。<br>– `DBMS_REDACT.FULL`：可以完全省略，默认为 NULL。<br>– 部分字符屏蔽的掩码参数。<br>一个逗号分隔的列表，不同字段类型内容不同。<br>**字符数据类型**<br>− 输入格式：对每个可屏蔽的字符输入 `V`。对每个要使用格式化字符进行格式化的字符输入 `F`。确保每个字符都有对应的 `V` 或 `F` 值。<br>− 输出格式：对每个可能被屏蔽的字符输入 `V`。将输入格式中的每个 `F` 字符替换为用于显示输出的字符。<br>− 掩码字符：指定用于屏蔽的字符。<br>− 起始数字位置：指定屏蔽的起始 `V` 数字位置。<br>− 结束数字位置：指定屏蔽的结束 `V` 数字位置。`F` 位置不计入。<br>**数字数据类型设置**<br>− 掩码字符：指定要显示的字符。输入 0 到 9 的数字。<br>− 起始数字位置：指定屏蔽的起始数字位置，例如 1 表示第一位。<br>− 结束数字位置：指定屏蔽的结束数字位置。<br>**日期时间数据类型设置**<br>`m`：屏蔽月份。要省略屏蔽，请输入大写 `M`。<br>`d`：屏蔽月份中的日。要省略屏蔽，请输入大写 `D`。<br>`y`：屏蔽年。要省略屏蔽，请输入大写 `Y`。<br>`h`：屏蔽小时。要省略屏蔽，请输入大写 `H`。<br>`m`：屏蔽分钟。要省略屏蔽，请输入大写 `M`。<br>`s`：屏蔽秒。要省略屏蔽，请输入大写 `S`。<br>有关不同参数的更多信息，请参阅《Oracle Database Advanced Security Guide》文档。 |
| `expression` | | 使用 `sys_context` 的布尔表达式。如果计算结果为 TRUE，则执行屏蔽 |
| `enable` | | 数据屏蔽策略在创建时是否启用？默认值为 TRUE |
| `regexp_pattern` | | 正则表达式模式，最多 512 字节 |
| `regexp_replace_string` | | 正则表达式的替换字符串 |
| `regexp_position` | | 从 1 开始的整数，指定搜索必须开始的位置 |
| `regexp_occurrence` | | `0` - 替换所有匹配项。<br>`n` - 替换第 `n` 个匹配项 |
| `regexp_match_parameter` | | 更改默认匹配行为。可能的值是 `'i'`、`'c'`、`'n'`、`'m'`、`'x'` 的组合 |
| `policy_description` | | 屏蔽策略的描述 |
| `column_description` | | 被屏蔽列的描述 |

```sql
begin
sys.dbms_redact.add_policy
( object_schema       => 'f1data'
, object_name         => 'laptimes'
, policy_name         => 'maskedlaptime'
, column_name         => 'time'
, function_type       => dbms_redact.partial
, function_parameters => 'VFVVFVVVVV,V:VV.VVVVV,X,1,3'
, expression          => q'[sys_context( 'driver_ctx'
, 'driverid' ) is null]'
);
end;
/
```

## alter_policy

由于该表还包含毫秒时间列，因此需要将另一列添加到策略中进行屏蔽。为此使用 `dbms_redact.alter_policy` 过程。

`alter_policy` 过程可以采用与 `add_policy` 过程相同的参数，但多了一个 `action` 参数，该参数可以具有表 **18-8** 中描述的值之一。

**表 18-8**

`action` 参数的可能值

| 常量 | 描述 |
| --- | --- |
| `ADD_COLUMN` | 向屏蔽策略添加列。 |
| `DROP_COLUMN` | 从屏蔽策略中删除列。 |
| `MODIFY_EXPRESSION` | 修改屏蔽策略的表达式（该表达式计算布尔值：如果为 TRUE，则应用屏蔽，否则不应用）。 |
| `MODIFY_COLUMN` | 修改屏蔽策略中的列，以更改屏蔽函数类型 `function_type` 或函数参数 `function_parameters`。 |
| `SET_POLICY_DESCRIPTION` | 为屏蔽策略设置描述。 |
| `SET_COLUMN_DESCRIPTION` | 为列上执行的屏蔽设置描述。 |

```sql
begin
sys.dbms_redact.alter_policy
( object_schema       => 'f1data'
, object_name         => 'laptimes'
, policy_name         => 'maskedlaptime'
, action              => dbms_redact.add_column
, column_name         => 'milliseconds'
, function_type       => dbms_redact.partial
, function_parameters => '9,1,4'
);
end;
/
```

您可以通过设置上下文来测试策略。这会提供更清晰的数据。

```sql
begin
driver_pkg.set_driver( driverid_in => 830 );
end;
/
select *
from   f1data.laptimes
where  driverid = sys_context( 'driver_ctx'
, 'driverid' )
/
RACEID DRIVERID LAP POS TIME       MILLISECO
------ -------- --- --- ---------- ---------
942      830   1   8 2:04.666      124666
942      830   2   8 1:59.138      119138
942      830   3   8 1:59.055      119055
942      830   4   8 1:59.800      119800
942      830   5   7 2:07.565      127565
942      830   6   7 2:17.508      137508
942      830   7   7 2:17.854      137854
942      830   8   7 2:01.301      121301
942      830   9   7 1:57.250      117250
...
```

如果在不设置上下文或清除上下文的情况下查询计时表，则会获得屏蔽后的数据。

```sql
begin
driver_pkg.unset_driver;
end;
/
select *
from   f1data.laptimes
where  1=1
and    driverid = 830
/
RACEID DRIVERID LAP POS TIME       MILLISECO
------ -------- --- --- ---------- ---------
942      830   1   8 X:XX.666      999966
942      830   2   8 X:XX.138      999938
942      830   3   8 X:XX.055      999955
942      830   4   8 X:XX.800      999900
942      830   5   7 X:XX.565      999965
942      830   6   7 X:XX.508      999908
942      830   7   7 X:XX.854      999954
942      830   8   7 X:XX.301      999901
942      830   9   7 X:XX.250      999950
...
```

### enable_policy

使用 `enable_policy` 过程来启用策略。

`enable_policy` 函数可以采用表 **18-9** 中描述的参数。

**表 18-9**

启用策略过程的参数

| 参数 | 是否必需 | 描述 |
| --- | --- | --- |
| `object_schema` | | 要保护的对象所在的模式。如果为 NULL，则假定为当前模式 |
| `object_name` | 是 | 要保护的对象的名称 |
| `policy_name` | 是 | 要启用或禁用的策略的名称 |

## disable_policy

要禁用策略，请使用 `disable_policy` 过程。

`disable_policy` 函数可以接受表 18-10 中描述的参数。

表 18-10

禁用策略过程的参数

| 参数 | 必填 | 描述 |
| --- | --- | --- |
| `object_schema` |   | 受保护对象所属的模式。如果为 NULL，则假定为当前模式。 |
| `object_name` | Y | 受保护对象的名称。 |
| `policy_name` | Y | 您想要启用或禁用的策略的名称。 |

## drop_policy

如果不再需要该策略，可以使用 `dbms_redact.drop_policy` 过程将其删除。

```sql
begin
sys.dbms_redact.drop_policy
( object_schema       => 'f1data'
,object_name         => 'laptimes'
,policy_name         => 'maskedlaptime');
end;
/
```

### 总结

Oracle Database 提供了一整套功能，可以根据规则显示或隐藏数据。时间有效性允许您只查看特定时间点有效的数据。VPD 允许您拥有一个对不同用户可见的单一模式，就像他们在使用自己的模式和数据一样。Redaction 策略允许您向特定用户组隐藏（敏感）数据的某些部分。当用户需要看到部分（而非全部）数据时，Redaction 非常有用。典型的场景是在呼叫中心，他们会要求提供信用卡号、电话号码、社会安全号码（SSN）等的最后几位数字来验证您的身份。

## 19. 零停机时间升级应用程序

升级数据库应用程序总是需要停机时间。替换正在使用的 PL/SQL 对象在当时是不可能的。多年来，开发了许多技术来减少 Oracle Database 的停机时间，例如备用数据库、RAC、Streams、在线索引重建或在线表重定义。但有一件事仍然可能让用户的工作中断，那就是他们正在使用的实际数据库应用程序。为了克服这“最后一块拼图”，即能够在 PL/SQL 对象正在使用时替换它们，开发了基于版本的重定义（Edition-based redefinition）。

### 停机时间

本质上，有两种类型的停机时间：计划内的和计划外的。计划外的停机时间会意外发生，为此场景做准备非常困难。可以采取一些措施来减少计划外停机时间，但要消除它几乎是不可能的。

计划内的停机时间是预先知道的，Oracle 已经实现了相关功能来减少计划内停机时间。可以通过物理和/或逻辑备用数据库以及 RAC 来防止硬件故障。升级数据库软件本身不需要关闭数据库。当您拥有逻辑备用数据库和 Streams 时，可以在升级的同时保持数据库正常运行和可用。索引可以在线重建。表可以在线重定义，这允许数据库保持可用。

唯一需要停机的是*您的*自定义应用程序。在运行时替换 PL/SQL 当时是不可能的。某个 PL/SQL 对象只能有一个版本，替换它需要锁定该对象。该对象将无法再执行，并中断正常的业务流程。当 PL/SQL 对象被替换时，依赖它的对象可能会变为无效并需要重新编译。

基于版本的重定义（EBR）解决了这个问题，并允许您的自定义应用程序在使用中进行升级。通过 EBR，您可以在一个版本的“私密环境”中进行 PL/SQL 更改。当自定义应用程序完全更新后，可以将其发布给用户。在升级过程中使用应用程序的最终用户可以继续工作，就像什么都没发生一样，而创建新连接的用户则立即使用新的应用程序对象。

在最后一个用户断开与预升级应用程序的连接后，预升级应用程序可以被停用，只剩下升级后的应用程序可用。

### 定义

EBR 使用了一些贯穿本章的术语。表 19-1 概述了最常用的定义。

表 19-1

EBR 术语

| 术语 | 描述 |
| --- | --- |
| Edition | 一个标识非模式对象类型的名称，用于扩展标准命名解析。可版本化的对象通过版本、模式和对象名来标识。 |
| Editioning view | 一种特殊类型的视图，充当底层表的抽象层。因为 editioning view 存在于特定版本中，所以它反映了底层表的当前实现。editioning view 有一些特定的限制。 |
| Editionable object | 可版本化的模式对象。这包括同义词、视图和所有 PL/SQL 对象类型。 |
| Non-editionable object | 所有不可版本化的模式对象。这包括表或物化视图。版本不能拥有自己的非版本化对象的副本。非版本化对象由所有版本共享。 |
| Crossedition trigger | 一种可以跨越版本边界的特殊触发器。 |
| NE on E prohibition | 非版本化对象不能依赖于可版本化对象，因为版本化对象在名称解析期间是不可见的。但是，物化视图和虚拟列可以在名称解析期间指定一个评估版本。 |

### 概念

EBR 的目的非常直接：零停机时间打补丁或升级。如果您有条件可以安排停机时间，那么请务必这样做。这对所有相关方来说都要容易得多。

补丁和升级的区别如下。在理想情况下，程序的所有需求都在实际程序中实现。程序完全按照书面规范实现。当程序没有实现需求规范所规定的内容时，必须应用补丁来纠正这种情况，使实现与规范保持一致。补丁应使程序达到功能需求的标准。

升级的定义是在程序创建之后需求发生了变化。从这时起，文本中不再区分这两者。当提到升级时，也隐含了补丁，反之亦然。

当然，当您想要零停机时间升级时，数据库并不是应用程序中唯一涉及的部分。用于建立数据库连接的方法决定了连接需要连接到哪个版本的应用程序。这可以由负载均衡器或流量导向器完成。正在进行中的会话，即连接到预升级应用程序的会话，通过一台应用服务器连接。而建立新连接的会话则使用另一台应用服务器并连接到升级后的应用程序。

零停机时间必须在 Oracle Database、应用服务器和客户端应用程序中有意地进行设计。

实现零停机时间意味着应用程序应该始终对用户可用。这是一个巨大的挑战：使用预升级应用程序的用户不想停止他们正在做的事情来允许部署新的应用程序。想要使用升级后应用程序的用户不想等到预升级应用程序的用户完成工作后才连接到应用程序。因此，两个应用程序——预升级版本和升级后版本——需要同时处于活动状态，以便服务于使用预升级应用程序的用户，并建立到升级后应用程序的新连接。这被称为*热切换*。

能够在旧应用程序仍在使用时部署新应用程序，对于某些类型的应用程序来说可能很容易。部署新的源文件可能就足够了。

当涉及到数据库时，这就变得困难得多。需要回答一些基本问题。


#### 零停机升级的挑战

*当用户正在使用应用程序时，你该如何对其进行修改？*

对应用程序进行修改通常涉及多个被修改的对象，但你不能逐一修改它们。那样会使应用程序处于无效状态。此外，在有人使用时，数据库代码*完全不可能*进行编译。所有对象需要同时被更改，但这可能会中断升级前的应用程序。

一种可能是使用升级前的数据库应用程序对数据库进行一次*完整副本*，并进行所需更改以达到升级后的状态。这解决了第一个问题，即能够在私密环境中进行升级工作。

另一个选项是复制数据库模式，并对这个新模式进行更改。

两种选项都允许你在私密环境中更改对象，并为升级后的情况做好准备，但它们并未解决接下来的两个挑战，甚至可能使这些挑战变得更难。

*如何在两个应用程序（升级前和升级后）之间保持数据同步？*

正如你所想，这并非易事。原始数据库对象仍在使用中，数据可能被输入和更改。必须有一种机制，允许数据变更在升级前和升级后的应用程序之间传播。

当数据在升级前和升级后的应用程序之间共享时，同步问题并不简单。升级前的应用程序可以访问由升级后应用程序创建或更改的数据，反之亦然。对于所有包含数据的对象（如物化视图和索引）也是如此。

*如果数据结构发生变化怎么办？*

升级应用程序可能会改变数据结构，那么当数据在升级前和升级后的应用程序之间是共享的时候，你如何为每个应用程序定义不同的数据表示？

在升级前应用程序中执行的事务必须反映在升级后应用程序中。在热切换期间，反之亦然。升级后应用程序中的更改也必须反映在升级前应用程序中。

`EBR` 可以满足零停机要求的全部三个挑战。

#### 解决方案

解决这三个挑战的机制如下。

- 更改在新的版本中私密地进行。
- 通过使用版本化视图，实现对公共表的不同投影。
- `跨版本`触发器在热切换期间保持不同版本之间的数据变更同步。

除了 `EBR` 引入的这些结构外，Oracle 数据库的两个支持特性使解决方案更加完善。

第一个支持特性是非阻塞式 `DDL`。当你对表进行更改（如添加列）时，这是以非阻塞方式完成的。表上的当前事务不会阻止此更改。

另一个特性是细粒度的依赖跟踪。当你有一个包引用某个表时，两者之间存在依赖关系。过去，更改表（如添加列）会使包失效。现在情况已不同。依赖关系现在更加细粒度。因为没有引用到新列，所以包不会失效。向包规范添加新子程序也是如此。依赖对象不会失效。

实施 `EBR` 并不像按一下开关那么简单。`EBR` 的准备阶段非常重要，可能需要更改现有的数据库设计和源代码。

#### 依赖关系

在 Oracle 数据库中，有保存数据的对象，如表、物化视图和索引，也有代码对象，即所有的 `PL/SQL` 对象。

经验法则是：数据对象*不能*是可版本化的，而代码对象*可以*是可版本化的。

可能存在一个表，其中一列是基于用户定义类型（如对象类型）。因为用户定义类型是可版本化的而表不是，你可能会陷入一个不易解决的境地。当存在多个跨版本的表定义时，应使用哪个用户定义类型？这种困境称为 *`非版本化对象对版本化对象的禁止`*。一个非版本化对象不能依赖于一个可版本化对象。自 Oracle 数据库 12.1 起，一直有一个针对 `非版本化对象对版本化对象的禁止` 的变通方法。你可以豁免某些对象*永远成为*可版本化的，而不是让整个模式启用版本化。在表中使用用户定义类型的例子中，你会希望在 `create type` 语句中将用户定义类型定义为非版本化的。这样，你就不会遇到 `非版本化对象对版本化对象的禁止`。

此外，从 Oracle 数据库 12.1 开始，物化视图和虚拟列可以拥有额外的元数据，这些元数据提供了关于计算版本的信息。仅当物化视图或虚拟列使用了可版本化的 `PL/SQL` 函数调用时，拥有此额外元数据才相关。这些元数据需要通过执行 `CREATE` 和 `ALTER` 语句显式设置。

在 12.1 之前的版本中，公共同义词不能启用版本化，因为那会违反 `非版本化对象对版本化对象的禁止`。从 12.1 版本开始，Oracle 维护的名为 `PUBLIC` 的用户启用了版本化，但所有现有的公共同义词都被标记为非版本化的。创建新的公共同义词可以是版本化的。

在热切换期间，你也可以在新版本的私密环境中测试升级后的应用程序。当必须测试升级后应用程序中的 `DML` 时，请确保使用“假数据”，这些数据应易于识别且不参与业务。使用假数据，实际数据不会受到基于版本重定义操作的影响，并且在测试完成后可以轻松识别和删除。另一种方法是仅以只读模式进行测试，以防止删除版本时留下残余数据。

如果出现问题，或者功能不符合规格，你可以在升级后版本的私密环境中继续前进。目标是创建一个升级后的应用程序，而这只有在错误被纠正时才能实现。

当热切换期完成并且测试结束后，没有现有连接在使用升级前的应用程序，此时可以停用升级前版本。阻止用户连接到升级前的应用程序可能是第一步。用于在升级前和升级后应用程序之间保持数据同步的 `跨版本` 触发器不再需要。可以删除升级前版本中的对象，最终，可以从表中删除未使用的列。

如果 `EBR` 操作失败，需要移除最新版本，可以通过一条语句删除该版本。请记住，当从数据库中删除版本时，对表的更改不会被撤销。此外，对数据的更改也不会被撤销，应予以清除才能恢复到升级前的版本。



### 准备工作

要利用 EBR（版本隔离），必须采取一些步骤。自 11gR2 以来，每个 Oracle 数据库版本都有一个名为 `ORA$BASE` 的默认版本。即使无意使用 EBR，这个版本也始终存在。

第一步是在模式级别启用 EBR。这是一次性操作且不可逆。尽管不可逆，但这并不意味着你必须使用 EBR。

使用以下语句可为模式启用版本。

```sql
alter user book enable editions
/
```

在 Oracle 数据库 12.1 之前的版本中，整个模式会被启用版本。从该版本开始，可以将某些通常应为可版本化的对象标记为不可版本化。

以下语句可用于确定当前会话的版本。

```sql
select sys_context('userenv'
,'current_edition_name'
) "Current_Edition"
from   dual
/
Current_Edition

ORA$BASE
```

**提示**

使用 SQLcl 时，可以使用 `show edition` 命令来显示当前版本，这比检查 `userenv` 应用程序上下文中的当前版本名称要方便得多。

#### 权限

在上一节中，用户“book”已被启用版本，允许创建其他版本。版本之间存在层级关系，每个版本总是有 *一个* 父版本。唯一的例外是 `ORA$BASE`，或者当此版本最终被移除时，层级中的下一个版本。这个顶级版本称为 *根版本*。

为模式启用使用版本的功能并不意味着用户可以创建其他版本。要实现这一点，需要 `create any edition` 系统权限。除了此权限外，用户还需要在根版本上拥有 `use` 权限。

```sql
grant create any edition to book
/
```

当用户 book 创建一个版本时，`use` 权限会自动授予。此权限也可以显式授予其他用户。

```sql
grant use on edition  to 
/
```

只有在授予 `drop any edition` 系统权限时，才能删除根或叶版本。

```sql
grant drop any edition to book
/
```

连接到数据库时，也可以使用 `alter session` 语句来更改版本。

```sql
alter session set edition = 
/
```

从一种版本切换到另一种版本时，不能有未完成的事务。如果在事务未通过提交或回滚关闭时尝试更改版本，将会引发错误。

```sql
alter session set edition = r2
/
ERROR:
ORA-38814: ALTER SESSION SET EDITION must be first statement of transaction
```

### 多个复杂度级别

使用 EBR 存在多个复杂度级别。最简单的 EBR 是 *仅* 在版本之间更改 PL/SQL 对象。

下一级别是表结构也在版本之间更改，但 *无需* 同时保留多个版本可用。所有用户都会在未来指定的时间点迁移到升级后的版本。

最复杂的级别是涉及表结构更改，并且用户需要同时访问多个版本。这涉及在多个版本之间保持数据同步。

接下来的几个章节将讨论涉及的每个步骤，从最简单的形式开始：仅在版本之间更改 PL/SQL 对象。

#### 仅更改 PL/SQL 对象

EBR 的最简单形式是仅更改 PL/SQL 对象。数据模型保持不变。在以下示例中，在 `ORA$BASE` 版本中创建一个过程，通过提供名和姓作为参数来显示驾驶员的全名。

```sql
create or replace
procedure format_fullname (p_firstname in varchar2
,p_lastname in varchar2)
is
begin
dbms_output.put_line (p_firstname
||' '||
p_lastname
);
end format_fullname;
/
```

启用服务器输出后，这个简单的过程会将名和姓连接在一起显示，中间用空格分隔。

随着 EBR 的引入，数据字典扩展了许多同样反映版本信息的视图。这些数据字典视图带有 `_ae` 后缀，是 *all editions*（所有版本）的缩写。其中一个视图是 `user_objects_ae`。这个数据字典视图显示创建的 procedure `format_fullname` 存在于版本 `ORA$BASE` 中。

```sql
select object_name
, object_type
, edition_name
from   user_objects_ae
where  object_name = 'FORMAT_FULLNAME'
/
OBJECT_NAME     OBJECT_TYPE     EDITION_NAME
--------------- --------------- ---------------
FORMAT_FULLNAME PROCEDURE       ORA$BASE
```

**注意**

当上述查询显示 `edition_name` 列为空时，表示用户尚未启用版本。

下一步是创建一个名为 R1 的额外版本。

```sql
create edition r1 as child of ora$base
/
```

虽然严格来说不必指定哪个版本是父版本，但明确指定总是无妨。

创建版本并不意味着当前会话已切换到它。要将当前版本更改为新创建的版本，需要发出 `alter session` 命令。

```sql
alter session set edition = r1
/
select sys_context('userenv'
,'current_edition_name'
) "Current_Edition"
from   dual
/
Current_Edition

R1
```

除了使用 `alter session` 语句切换版本外，也可以在连接时指定版本。

```sql
conn /@ edition = r1
```

在 R1 版本中执行 `format_fullname` 过程会显示以下结果。

```sql
begin
format_fullname (p_firstname => 'Mick'
,p_lastname  => 'Schumacher'
);
end;
/
Mick Schumacher
```

这是如何实现的？过程 `format_fullname` 是从父版本 `ORA$BASE` 继承而来的。所有可版本化的对象都由子版本继承，并且可以像以前一样使用。

引入新版本的原因是为了改进 `format_fullname` 过程的功能。这可以通过非常熟悉的 `create or replace` 方法来完成。该过程的替换发生在版本 R1 的 *私有环境* 中，同时保持原始过程完整且有效。连接到 `ORA$BASE` 版本的用户不受 `format_fullname` 过程新版本的影响。

```sql
create or replace
procedure format_fullname (p_firstname in varchar2
,p_lastname  in varchar2)
is
begin
dbms_output.put_line (initcap (p_lastname)
||', '||
initcap (p_firstname)
);
end format_fullname;
/
```

过程 `format_fullname` 在当前版本中被替换，`user_objects_ae` 数据字典视图显示两个同名但存在于不同版本中的过程。

```sql
select object_name
, object_type
, edition_name
from   user_objects_ae
where  object_name = 'FORMAT_FULLNAME'
/
OBJECT_NAME     OBJECT_TYPE     EDITION_NAME
--------------- --------------- ---------------
FORMAT_FULLNAME PROCEDURE       ORA$BASE
FORMAT_FULLNAME PROCEDURE       R1
```

新版本的用户在调用该过程时会得到以下结果。

```sql
begin
format_fullname (p_firstname => 'Mick'
,p_lastname  => 'Schumacher'
);
end;
/
Schumacher, Mick
```

还有可能在升级后的版本中不再需要某个 PL/SQL 对象。可以通过发出 `drop` 语句来移除该 PL/SQL 对象。请记住，删除 PL/SQL 对象仅从当前版本中移除它。同名的过程仍然存在于 `ORA$BASE` 中。



删除过程后检查 `user_objects_ae` 数据字典视图，会显示以下结果。

```
select object_name
, object_type
, edition_name
from   user_objects_ae
where  object_name = 'FORMAT_FULLNAME'
/
OBJECT_NAME     OBJECT_TYPE     EDITION_NAME
--------------- --------------- ---------------
FORMAT_FULLNAME PROCEDURE       ORA$BASE
FORMAT_FULLNAME NON-EXISTENT    R1
```

前述结果显示，仍然有两个对名为 `format_fullname` 的对象的引用；一个在 `ORA$BASE` 中，另一个在 `R1` 中。请注意，`ORA$BASE` 中的对象是一个 `procedure`（过程），而 `R1` 中的另一个对象是 `non-existent`（不存在）。

一旦对象从某个版本中移除，它就不再被下一个子版本继承，因此也无法再被调用。

将过程更改为函数（用一个函数来格式化驾驶员全名更有意义）是通过组合使用 `drop`（这一步已经完成以删除过程）和 `create` 来实现函数功能。

```
create or replace
function format_fullname (p_firstname in varchar2
,p_lastname  in varchar2)
return varchar2
is
begin
return initcap (p_lastname)
||', '||
initcap (p_firstname);
end format_fullname;
/
```

这些操作在数据字典中产生如下结果。

```
select object_name
, object_type
, edition_name
from   user_objects_ae
where  object_name = 'FORMAT_FULLNAME'
/
OBJECT_NAME     OBJECT_TYPE     EDITION_NAME
--------------- --------------- ---------------
FORMAT_FULLNAME PROCEDURE       ORA$BASE
FORMAT_FULLNAME FUNCTION        R1
```

当仅涉及 PL/SQL 对象时，升级所需的仅此几个步骤。接下来要做的是使新版本对用户可用，并使旧版本不可用（即，停用旧版本）。这些主题将在下一节中介绍。

### 表变更：不要在版本间同步

有时，在向新版本推进的过程中，有必要更改表的结构。如前所述：表不能是可版本化的。不可能拥有同一张表的多个版本。有两种机制有助于在对表进行更改时保持应用程序在线：细粒度依赖跟踪和非阻塞 DDL。

细粒度依赖跟踪可防止在对表进行更改时对象失效。添加列不会使任何 PL/SQL 对象失效，因为 PL/SQL 对象中没有对该列的依赖关系。

由于非阻塞 DDL，添加列不会阻碍任何会话。

在每个版本中，都需要数据的不同表示形式来公开新添加的列。

EBR 引入了 `editioning view`（版本视图）的概念。顾名思义，它是一个视图，但一种特殊类型的视图。版本视图充当底层表的抽象层。因为版本视图是可版本化的，所以它可能在每个版本中具有不同的结构。由于抽象层，数据仍然存储在一个地方，即底层表。该表保存数据，是单一事实来源。索引和约束保留在表上。

引入版本视图需要一些停机时间，尤其是在第一次涉及表的基于版本的重定义操作中。这是一项一次性操作，用于将现有的非基于版本的数据库应用程序准备好，以便将来进行基于版本的升级。一旦完成此操作，它就为零停机时间升级做好了准备。

**注意**

表上使用的一些特性可能需要重新路由到版本视图，例如权限、触发器和虚拟专用数据库（`VPD`）。事先收集这些特性的脚本，可以大大简化切换到版本视图的过程。

在第一次 EBR 操作期间引入版本视图需要遵循六个步骤。

1.  重命名表。
2.  创建版本视图。
3.  将权限重新路由到版本视图。
4.  在版本视图上重新创建触发器。
5.  重新编译 PL/SQL 对象。
6.  重新应用 `VPD` 策略。

版本视图是实际表与应用程序之间的抽象层。应用程序很可能在整个源代码中引用实际表。在源代码中将实际表的名称更改为版本视图可能是一项相当艰巨的任务，并且容易出错。将版本视图命名为与原表相同的名字，对应用程序和源代码的影响最小。不幸的是，这是不可能的。不能有两个同名的对象。你可以通过首先重命名实际表，然后创建一个与原表同名的版本视图来解决这个困境。

今后，应用程序不应再直接访问表，创建一个与常规表名可区分的表名应有助于防止这种情况。以下命名约定引入了一个特殊字符（通常是下划线）并使表名区分大小写。

```
alter table drivers rename to "_drivers"
/
```

`_drivers` 表*只能*在名称带引号时使用，因为它区分大小写。这不是一种常见的表命名约定。具有特殊字符并区分大小写向开发人员表明此表是特殊的。

通过此操作，步骤一完成。下一步是引入与原表同名的版本视图。

```
create editioning view drivers
as
select driverid
, driverref
, driver_number
, code
, forename
, surname
, dob
, nationality
, url
from   "_drivers"
/
```

版本视图只能基于单个表，并且不能包含聚合或表达式。


#### 重新路由特权与触发器

接下来，将原本在实体表上的特权重新路由至版本化视图。这需要考虑诸如授予其他模式的 `select` 或 `insert` 等特权。

手头备有为原始表创建的特权脚本，能极大简化此过程。版本化视图已沿用了原始表的名称。通过重新执行原始的授权脚本，即可将特权授予该版本化视图。

下一步是将表级触发器从原始表重新创建到版本化视图上。拥有为原始表创建的原始脚本，使这一步变得轻而易举。由于不能存在两个同名对象，因此需要先删除当前与该表关联的触发器。

由于表已被重命名，源代码会变为无效状态，PL/SQL 对象需要重新编译。完成此重新编译步骤后，编译后的代码将引用版本化视图。

完成向 EBR（版本化）过渡的最后一步是重新应用 VPD（虚拟专用数据库）策略函数。若未使用 VPD，可跳过此步骤。

遵循所有这些步骤后，应用程序升级时便不再需要计划停机时间。新增功能可以在一个独立的版本中私下完成，而不会影响现有应用程序。

可以向表中添加列，且无需在不同版本间保持数据同步。由于该列在父版本中不存在，因此没有可展示数据的对应表示。

以下示例在 `_drivers` 表中添加了一个电话号码列，以演示如何增加额外列。

```sql
alter table "_drivers"
add (
phone varchar2(10)
)
/
```

由于采用了细粒度依赖跟踪，更改表不会使任何引用该对象的 PL/SQL 对象失效。目前数据库中没有代码引用这个新添加的列。

可以通过创建一个版本化视图，在某个特定版本中私下公开此列。

```sql
create edition r2 as child of r1
/
Edition R2 created.
alter session set edition = r2
/
Session altered.
create or replace
editioning view drivers
as
select driverid
, driverref
, driver_number
, code
, forename
, surname
, dob
, nationality
, url
, phone
from   "_drivers"
/
View created.
select forename
, surname
, phone
from   drivers
fetch  first 5 rows only
/
FORENAME             SURNAME              PHONE
-------------------- -------------------- ----------
Bernd                Schneider
Paolo                Barilla
Gregor               Foitek
Claudio              Langes
Gary                 Brabham
```

从结果中可见，电话列已显示，但尚无数据。可以创建或修改一个应用程序来公开电话号码，并维护此信息。

因为电话列是全新的，所以无需在多个版本间保持数据同步。旧版本不知道此列的存在，也不会使用它。

#### 涉及数据同步的表更改

最复杂的基于版本的定义练习发生在需要在多个版本间保持数据同步时。

在进行本节概述的步骤之前，应已完成前面练习中所述的实现版本化视图的六个步骤。

示例中，对 `_drivers` 表进行了如下更改：将列“forename”和“surname”分别重命名为“first_name”和“last_name”，并且它们的长度从 25 字节增加到 35 字节。

直接修改现有列非常容易，但这会影响现有应用程序。引用这些列的源代码会变为无效，并可能导致（部分）停机。这与零停机升级的目标背道而驰，因此修改现有列是不可接受的。

```sql
create edition r3 as child of r2
/
Edition R3 created.
alter session set edition = r3
/
Session altered.
alter table "_drivers"
add (
first_name varchar2 (35)
, last_name  varchar2 (35)
)
/
Table altered.
create or replace
editioning view drivers
as
select driverid
, driverref
, driver_number
, code
, first_name
, last_name
, dob
, nationality
, url
, phone
from   "_drivers"
/
```

向表中添加列是容易的部分，修改版本化视图以公开新列也是如此。因为底层表是一个不可版本化的对象，所以在哪个版本中进行更改并不重要。但常识告诉我们，不应在旧版本中执行 DDL 操作。

新列“first_name”和“last_name”不包含任何数据。在新情况下，这些列应获得与原始表中“forename”和“surname”相同的值。一个直接的 `update` 语句即可实现。但这样大规模的更新可能因锁机制而干扰正常业务。

如果应用程序用户只使用最新版本（本例中为 R3 版本），那么执行大规模更新可能就足够了。从那时起，用户只需处理“first_name”和“last_name”，旧列不再需要维护。

比直接使用 `update` 语句更好的方法是利用 `dbms_parallel_execute` 来最小化完成任务所需的时间。当涉及大量数据时，这一点尤其有用。

如果需要同时向用户提供两个版本（称为*热切换期*），则需要额外的机制来保持“forename”与“first_name”同步、“surname”与“last_name”同步。使用 R2 版本旧应用程序的用户看到并操作的是“forename”和“surname”，而使用 R3 版本新应用程序的用户使用的是“first_name”和“last_name”。

实现这一点的机制——在版本间保持数据同步——是使用*跨版本触发器*实现的。它们类似于普通的表触发器，是可版本化的，并且必须在新版本（R3）中创建。

```sql
create or replace trigger driver_R2_R3_Fwd_Xed
before insert or update on "_drivers"
for each row
forward crossedition
disable
begin
:new.first_name := :new.forename;
:new.last_name  := :new.surname;
end driver_R2_R3_Fwd_Xed;
/
```

该触发器创建在底层表上，而不是版本化视图上。触发器需要对每一行生效，“forward cross edition”指令表示当在父版本中执行任何 DML 时都会触发，无论哪个父版本。

**提示**

创建跨版本触发器时使用禁用模式。只有当触发器有效时，才应启用它。无效的触发器会阻止 DML 操作完成。

触发器的逻辑很简单：新添加的列用驱动程序名称的旧表示形式来填充。

当触发器有效后，即可启用它，以便在旧应用程序用户进行更改时保持数据同步。

```sql
alter trigger driver_R2_R3_Fwd_Xed enable
/
Trigger DRIVER_R2_R3_FWD_XED altered.
```

## 20. 选择正确的表类型

任何数据库的目的都是存储数据。为此，你需要在数据库中创建表。Oracle 数据库提供了许多不同的表类型，每种类型都有其自身的优点和缺点。

数据的存储方式直接影响到数据检索的效率。不难想象，当一起被查询的数据也被存储在一起时，其速度会比数据分散存储各处要快。高效的应用始于高效的设计。

本章讨论不同的表类型。选择并不仅限于传统的堆表。

### 验证跨版本触发器

以下更新展示了如何验证 `crossedition` 触发器的工作情况。

```sql
alter session set edition = r1
/
update drivers
set surname = surname
where driverid = 102
/
commit
/
alter session set edition = r3
/
select first_name
,last_name
from drivers
where driverid = 102
/
FIRST_NAME      LAST_NAME
--------------- ---------------
Ayrton          Senna
```

在较旧版本中由 DML 语句触及的记录会触发 `crossedition` 触发器，从而更新新添加的列。切换到较旧版本并执行批量更新可以同步数据，但缺点是这在软件安装过程中可能令人困惑且容易出错。

一个更好的替代方案是保留在新版本中，并以一种特殊的方式更新底层表来触发正向 `crossedition` 触发器。

**注意：** `Crossedition` 触发器在底层表被更新时不会触发。它们仅在更新版本化视图时才会触发。

使用 `dbms_sql`，你可以在更新底层表时触发 `crossedition` 触发器。在 `parse` 过程中，为 `apply_crossedition_trigger` 参数传入需要触发的 `crossedition` 触发器的名称。以下代码示例触发了 `crossedition` 触发器来更改所有新添加的列。所使用的更新语句是虚假的。

```sql
declare
c number := dbms_sql.open_cursor();
x number;
begin
dbms_sql.parse
( c                          => c
, language_flag              => dbms_sql.native
, statement                  => 'update "_drivers"
set   driverid = driverid'
, apply_crossedition_trigger => 'DRIVER_R2_R3_FWD_XED'
);
x := dbms_sql.execute(c);
dbms_sql.close_cursor(c);
commit;
end;
/
```

现在，从旧版本到新版本的同步已经实现，还需要添加相反方向的功能。用户可以在新应用程序中添加或更改数据，这需要反映到旧应用程序中。

反向 `crossedition` 触发器也创建在底层表上，就像正向 `crossedition` 触发器一样。在这种情况下，反向 `crossedition` 短语表示当在最新版本中进行更改时，触发器应触发。出于同样的原因，采用相同的策略以禁用状态创建触发器。

列长度存在差异。旧版本中的列比新版本的短。截断名称使其适配是不可接受的。一个人的姓名不应被截断，因此这个版本引入了更改。在热切换期间，升级前和升级后的应用程序都必须包含用户输入的有效数据。

反向 `crossedition` 触发器如下所示。

```sql
create or replace trigger driver_R3_R2_Rve_Xed
before insert or update of first_name, last_name on "_drivers"
for each row
reverse crossedition
disable
begin
if length (:new.first_name) > 25
or length (:new.last_name) > 25
then
raise_application_error (-20000
,'During the hot rollover it is not possible to enter more'||
' than twenty-five (25) characters for the First or Last Name'
);
else
:new.forename := :new.first_name;
:new.surname := :new.last_name;
end if;
end driver_R3_R2_Rve_Xed;
/
Trigger DRIVER_R3_R2_RVE_XED compiled
```

当触发器有效时，可以启用它。

```sql
alter trigger driver_R3_R2_Rve_Xed enable
/
Trigger DRIVER_R3_R2_RVE_XED altered.
```

### 退役旧版本

当 EBR 练习完成后，就不再需要维护应用程序的旧版本了。如果用户仍能访问旧应用程序，就有可能它还会被使用。这是不希望发生的。旧版本的应用程序应该被退役，以便旧应用程序对用户不可用。

虽然非常罕见，但多个版本并行运行较长时间是可能的。升级到新应用程序与旧应用程序退役之间的过渡应尽可能短。

撤销数据库中每个用户和角色对旧版本的使用权限，可以防止任何人建立新连接。应由具有权限的用户执行以下语句。

```sql
revoke use on edition r1 from book
/
```

版本也可以被完全移除，但仅当涉及根版本或叶版本时。同时具有父版本和子版本的版本不能被删除。

删除版本，尤其是叶版本，可能会留下不可版本化的更改，例如对底层表的更改。这也适用于已经发生的数据更改。

可以通过具有 `drop any edition` 权限的用户执行以下语句来删除子版本。

```sql
drop edition r3 cascade
/
Edition R3 dropped.
```

请记住，表更改无法撤销。该版本不能正被某个会话使用。否则将导致异常。

同样，父版本也可以被删除，尽管并非绝对必要。拥有多个已退役版本不会干扰性能，因为引用是在编译时解析的。建议移除预升级版本中的所有对象，因为它们不再需要，同时移除来自后升级版本的 `crossedition` 触发器。这只是一个良好的内务管理问题。

#### 更改默认版本

默认的 Oracle 数据库版本是 `ORA$BASE`。连接到数据库时，如果未指定版本，则使用默认版本。除了通过 `alter session` 命令更改版本外，也可以更改默认版本。通常，DBA 会处理以下情况。

```sql
alter database default edition = r1
/
```

这会立即更改整个数据库的默认版本。除了 `SYS` 之外，所有用户对先前默认版本的 `use` 权限都会被撤销。同时，`PUBLIC` 被授予 `use` 权限。

普通用户无法再连接到 `ORA$BASE`；`SYS` 仍然可以。这允许在需要时将默认版本更改回 `ORA$BASE`。

### 总结

本章概述了减少甚至可能消除计划内停机时间的解决方案。在 Oracle Database 11gR2 版本之前，升级自定义应用程序会导致计划内停机。EBR 解决了这个计划内停机问题。在版本的私密空间中部署升级后的应用程序是关键。升级完成后，用户会被切换到升级后的应用程序，而不会经历停机。

根据你的需求，你可以使用几个不同复杂度级别。从简单地在不同版本中替换 PL/SQL 对象，到创建多个带有 `crossedition` 触发器的版本化视图以保持跨版本的数据同步。本章也讨论了退役旧版本的方法。


### 堆表

堆表（heap table）正如其名所示，所有记录都存储在一个巨大的堆中，完全没有顺序。如果从这样的表中选择行，除非在 `select` 语句中包含 `order by` 子句，否则无法保证返回给调用进程的行顺序。即使重复执行相同的语句，它每次可能都返回相同顺序的行。但只要表发生变化（例如由于任何 DML 语句或数据库重启），这个顺序就可能大不相同。除非在 `select` 语句中显式包含 `order by` 子句，否则永远不要依赖记录返回的顺序。

