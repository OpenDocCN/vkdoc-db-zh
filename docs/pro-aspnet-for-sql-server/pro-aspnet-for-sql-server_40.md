# 生成的数据访问层

清单 8-5 中生成的代码仅仅是一个返回给调用方的字符串，用于创建一个新的 `CodeSnippetCompileUnit` 类。目前，你只需知道这个类读取代码的字符串值，并将其传递给 `AssemblyBuilder` 类，该类将代码生成到内存中。

**注意：** .NET 平台目前正在进行一个新的动态语言运行时 (DLR) 的开发。它将支持诸如 IronPython 和 IronRuby 之类的语言，你随后可以在生产环境中选择使用它们作为 C# 和 VB 之外的编程语言。许多开发者认为这是一个极好的消息，因为他们觉得这些语言能用更少的代码做更多的事情。随着这些技术逐渐成熟并成为 .NET 平台的核心部分，未来几年这个领域值得关注。

#### CodeDom 命名空间

为 .NET 平台生成代码可通过 `CodeDom` 命名空间（具体指 `System.CodeDom`）方便地获得支持。该命名空间将代码表示为一个文档对象模型，可以像操作 XML 一样轻松地操作它。你可以以一种语言中立的方式创建新的类，并向它们添加字段、属性和方法，然后将其生成到诸如 C# 和 VB 等受支持的语言中。

在清单 8-5 中，代码是作为字符串组装的。你可以使用 `CodeDom` 完成完全相同的工作，这将为你提供更好的结构。通过将代码组装为跨越多行的带引号的字符串，你必须小心地在每个语句末尾添加分号并关闭每个代码块。当使用 `CodeDom` 以编程方式组装代码时，这些顾虑就消失了。

从一个名为 `AbcBuildProvider` 的新自定义构建提供程序开始，你将定义一种使用 `.abc` 扩展名的文件的 XML 文件格式。`Web.config` 文件的编译部分现在包含了额外的构建提供程序，如清单 8-6 所示。

### 清单 8-6. 为 .abc 文件添加构建提供程序

```xml
<compilation debug="true">
  <buildProviders>
    <add extension=".abc" type="Chapter08.BuildProviders.AbcBuildProvider"/>
    <add extension=".xyz" type="Chapter08.BuildProviders.XyzBuildProvider"/>
  </buildProviders>
</compilation>
```

新的文件格式将定义一个自定义数据类，构建提供程序将使用每个源文件中定义的字段列表来生成该类。清单 8-7 显示了一个示例文件。

### 清单 8-7. Person.abc

```xml
<?xml version="1.0"?>
<dataClass>
  <fields>
    <add name="FirstName" type="System.String"/>
    <add name="LastName" type="System.String"/>
    <add name="BirthDate" type="System.DateTime"/>
    <add name="Location" type="System.String"/>
  </fields>
</dataClass>
```

与 `XyzBuildProvider` 类似，这个新的构建提供程序将使用源生成的类的名称。示例文件 `Person.abc` 将创建一个名为 `Person` 的类，其包含一组由文件定义的字段和属性。当构建提供程序和网站被编译后，`Person` 类应该对网站可用。

从也为之前的构建提供程序创建的 `GetGeneratedCode` 方法开始，你将调整它以返回一个 `CodeCompileUnit` 而不是字符串，让它将源文件解析为 XML 流，并处理在 `fields` 元素内定义的每一个字段。清单 8-8 显示了新方法。

### 清单 8-8. AbcBuildProvider 的 GetGeneratedCode 方法

```csharp
private CodeCompileUnit GetGeneratedCode()
{
    CodeCompileUnit code = new CodeCompileUnit();
    CodeNamespace ns = new CodeNamespace(GetNamespace());
    CodeNamespaceImport import = new CodeNamespaceImport("System");
    ns.Imports.Add(import);
    code.Namespaces.Add(ns);

    string className = GetClassName();
    CodeTypeDeclaration generatedClass =
        new CodeTypeDeclaration(className);
    generatedClass.IsPartial = true;
    ns.Types.Add(generatedClass);

    XmlDocument document = new XmlDocument();
    using (Stream stream = OpenStream())
    {
        document.Load(stream);
        XmlNode rootNode = document.SelectSingleNode(RootPath);
        if (rootNode != null)
        {
            ProcessFieldNodes(generatedClass, rootNode);
        }
    }
    return code;
}
```

新的 `GetGeneratedCode` 方法中引用了几种新类型。像 `CodeNamespace`、`CodeNamespaceImport` 和 `CodeTypeDeclaration` 这样的类型都是 `CodeCompileUnit` 的组成部分，用于创建生成的类。`CodeNamespace` 类型是包裹生成类的命名空间。为 `System` 添加了一个 `CodeNamespaceImport` 到该命名空间中。你可以根据需要添加其他的导入，就像在 Visual Studio 中编辑源文件时一样。最后，`CodeTypeDeclaration` 用于 `generatedClass` 的实例，该实例将保存在清单 8-9 所示的 `ProcessFieldNodes` 方法中定义的字段。

### 清单 8-9. ProcessFieldNodes 方法

```csharp
private void ProcessFieldNodes(
    CodeTypeDeclaration generatedClass, XmlNode rootNode)
{
    XmlNodeList fieldNodes = rootNode.SelectNodes(FieldsAddPath);
    foreach (XmlNode addFieldNode in fieldNodes)
    {
        XmlNode nameNode = addFieldNode.SelectSingleNode("@name");
        XmlNode typeNode = addFieldNode.SelectSingleNode("@type");
        string propertyName = nameNode.Value;
        string fieldName = GetFieldName(propertyName);
        Type fieldType = Type.GetType(typeNode.Value);

        // private field
        CodeMemberField field = new CodeMemberField(fieldType, fieldName);
        generatedClass.Members.Add(field);

        AttachProperty(generatedClass, propertyName, fieldName, fieldType);
    }
}
```

`ProcessFieldNodes` 方法首先遍历 XML 源文件中根节点下的所有字段节点。你需要访问的两个节点是 `add` 元素中名为 `name` 和 `type` 的属性。这些引用是通过诸如 `@name` 和 `@type` 这样的 XPath 引用来完成的。在 `AbcBuildProvider` 的顶部定义了两个常量来帮助遍历这个 XML 文档。这些 XPath 常量如清单 8-10 所示。

### 清单 8-10. XPath 常量

```csharp
public const string RootPath = "/dataClass";
public const string FieldsAddPath = "fields/add";
```

第一个 XPath 常量 `RootPath` 访问第一个 XML 节点，然后允许第二个 XPath 常量 `FieldsAddPath` 访问所有保存我们想要使用的属性的 `add` 元素。在清单 8-9 中，读取了这两个节点，稍后用于设置 `propertyName`、`fieldName` 和 `fieldType` 变量的值。然后创建一个新的 `CodeMemberField` 实例并将其添加到 `generatedClass` 的 `Members` 列表中。这些都是私有成员变量，通过清单 8-11 所示的 `AttachedProperty` 方法定义的属性来访问。

### 清单 8-11. AttachProperty 方法

```csharp
private static void AttachProperty(
    CodeTypeDeclaration generatedClass,
    string propertyName,
    string fieldName,
    Type fieldType)
{
    // public property
    CodeMemberProperty prop = new CodeMemberProperty();
    prop.Name = propertyName;
    prop.Type = new CodeTypeReference(fieldType);
    prop.Attributes = MemberAttributes.Public;

    CodeFieldReferenceExpression fieldRef;
    fieldRef = new CodeFieldReferenceExpression();
    fieldRef.TargetObject = new CodeThisReferenceExpression();
    fieldRef.FieldName = fieldName;

    // property getter
    CodeMethodReturnStatement ret;
    ret = new CodeMethodReturnStatement(fieldRef);
    prop.GetStatements.Add(ret);

    // property setter
    CodeAssignStatement assign = new CodeAssignStatement();
    assign.Left = fieldRef;
    assign.Right = new CodePropertySetValueReferenceExpression();
    prop.SetStatements.Add(assign);

    generatedClass.Members.Add(prop);
}
```



`AttachProperty` 方法接收必要的变量，用以引用该属性将要访问的相关私有成员变量。该属性使用 `CodeMemberProperty` 类型，结合 `CodeFieldReferenceExpression`、`CodeMethodReturnStatement` 和 `CodeAssignStatement` 等类来创建。此方法首先创建属性实例，创建对字段的引用，然后向该属性添加 getter 和 setter 语句。
最后，该属性被添加到 `generatedClass` 的 `Members` 集合中。
核心工作就是这些。还有几个小的私有方法来辅助这个过程。

### GetNamespace 方法
`GetNamespace` 方法从配置中获取字符串值。
```
private string GetNamespace()
{
    string ns = ConfigurationManager.AppSettings["AbcNamespace"];
    if (String.IsNullOrEmpty(ns))
    {
        ns = "Abc";
    }
    return ns;
}
```
`AbcNamespace` 的配置应添加到网站中，值为 `Chapter08.Website`；否则将默认使用 `Abc`。

### GetFieldName 方法
`GetFieldName` 方法根据所有先前代码示例中遵循的命名约定，将属性名转换为字段名，即在字段名前加下划线。
```
private string GetFieldName(string propertyName)
{
    return "_" +
           propertyName.Substring(0, 1).ToLower() +
           propertyName.Substring(1);
}
```
现在，当网站与构建提供程序一起编译时，它将生成一个名为 `Person` 的新类，其中包含前面清单 8-7 中显示的所有字段。构建成功运行后，你可以打开对象浏览器查看新生成的类。图 8-2 展示了包含所有已生成字段和属性的 `Person` 类。
原始源文件是十行 XML，而它生成的类却超过 60 行代码，这些代码我都不用自己写。你可以更进一步：不用 XML 文件直接定义字段，而是让 XML 数据指向数据库中的一个表或存储过程，由它们提供生成所有这些字段和属性所需的全部详细信息。几个流行的数据访问层项目正是这么做的。本章稍后将介绍的 `SubSonic` 和 `Blinq` 都从数据库读取架构，并根据发现的信息生成类。我将向你展示这两个项目提供的功能。但首先，考虑一下你会如何生成所有这些代码。虽然仅用 `CodeDom` 你也能非常快速地完成，但你应该考虑使用模板。

## 模板
虽然 `CodeDom` 为你提供了一种结构化的方式来生成代码，但使用 `CodeDom` 可能很繁琐且难以维护。`CodeDom` 也让人感觉不自然。作为开发者，我们更习惯看起来像代码的代码，而不是 `CodeDom` 用来描述构成代码的各个部分的衍生形式。当你浏览一组使用 `CodeDom` 生成代码的方法时，你会看到对 `CodeTypeDeclaration` 和 `CodeMemberProperty` 的引用。当 `CodeDom` 代码很多时，它可能难以理解，也难以在脑中拼凑起来。
因此，当需要生成大量代码时，将代码段放入模板文件中会很有帮助。这些模板文件可以以尽可能接近实际代码的形式读取。生成代码时，该过程会读取这些模板，并将结果字符串加载到 `CodeCompileUnit` 中以生成类。
ASP.NET 对页面和用户控件使用模板，这些内容主要由 HTML 以及特定于 ASP.NET 控件的特殊指令组成。如果不这样做，你就必须像早期 Web 开发那样，从源文件中拼凑所有的 HTML。
前面章节中从 XML 文件生成所有字段只需要大约 100 行 `CodeDom` 代码。这相当不错，但它也只定义了字段和属性。
创建调用存储过程的方法（可能带有多个输入参数）将需要使用 `CodeDom` 完成大量更多的工作。而且，正如你所看到的，调用存储过程是非常样板化的代码，特别是在使用企业库（Enterprise Library）时，你可以从模板中获得很多价值。
我将要 review 的第一个代码生成器是 `SubSonic`，它包含一个丰富的模板系统。

#### SubSonic
`SubSonic` 是一个开源项目，部分托管在 CodePlex 上，因其提供的许多引人注目的功能而广受欢迎。最近，`SubSonic` 发布了 2.0 版本，增加了对企业库的支持以及为多个数据库生成代码的能力。更多详情请访问 http://www.subsonicproject.com。这里，我将重点介绍它如何使用模板生成代码以及如何从数据库读取详细信息来生成代码。
> **注意** CodePlex (http://www.codeplex.com) 是微软提供的最新的代码存储库。它以 Team System 作为源代码控制和工作项跟踪系统。有一个免费的支持 Team System 的 Visual Studio 版本可用于与 CodePlex 一起工作。CodePlex 也支持与 Windows 资源管理器集成的 Subversion 客户端 TortoiseSVN 的基本桥接。
你需要下载 `SubSonic` 项目文件用于本节，并将它们放入如图 8-3 所示的公共工具文件夹中。你将从此位置引用命令行工具以及 `SubSonic` 程序集。如果你想编译 `SubSonic` 项目，还需要安装 SQL Server SDK。
运行 `SubSonic` 是通过一个名为 `SubCommander` 的命令行工具完成的，它处理多个命令以生成代码和数据库脚本。`generate` 命令识别许多参数。最重要的参数与 `/config` 和 `/out` 开关一起使用。清单 8-14 展示了一个用于生成数据访问层代码的示例命令。

```console
sonic.exe /config Website\Web.config /out Website\App_Code\Generated
```
生成的类可以包含在你的项目中，项目可以是一个网站或一个类库。最初，`SubSonic` 创建时，它专门用于 ASP.NET 2.0 网站，因为它利用了构建提供程序（Build Provider）模型，而该模型仅适用于网站。为了扩展生成代码的适用范围，增加了将类生成到源文件的功能。将代码生成限制在网站内，阻止了数据访问层作为其他类型应用程序（如桌面和控制台应用程序）的依赖项被使用。一些开发人员也对数据访问层随着数据库变化而动态变化感到不适。
> **注意** 有关 `code generate` 命令参数的完整文档以及演练视频，请参阅 `SubSonic` 网站 (http://www.subsonicproject.com)。

### SubSonic 模板
`SubSonic` 中的模板系统镜像了 ASP.NET 使用的模板系统。文件的格式看起来很像 `.aspx` 标记文件，但它不是生成 HTML，而是为你的数据访问层生成代码。图 8-4 展示了一个用于生成类的示例模板。



.NET 框架本身并不提供处理这些模板文件的基础设施，除非作为 ASP.NET 运行时的一部分。为了让这些模板能在 SubSonic 中用于代码生成，该项目创建了一个解析器和代码处理器。它读取嵌入在程序集中的模板文件，进行解析、处理，生成输出代码，然后保存到源文件中。所有用于生成可工作代码的模板代码都包含在源代码下载中，以便您可以根据需要进行调整。如果您不想使用嵌入式模板，也可以为自己的模板指定一个替代位置。`/templateDirectory` 参数用于覆盖标准模板处理。为了使用“外部世界”的值处理模板，一些占位符在解析和处理模板时被替换。在图 8-3 中，您可以看到 `providerName` 和 `tableName` 变量被设置为 `#PROVIDER#` 和 `#TABLE#`，这些是占位符，会在模板处理的早期步骤中被替换。这些变量在模板生成代码的其余部分中被使用。

模拟 .aspx 模板处理的优势在于，它不仅支持占位符替换，还支持循环。清单 8-15 展示了类模板中的一个循环，用于生成与数据库表列匹配的所有属性。

`清单 8-15`. 用于表属性的模板循环

```
<%
foreach(TableSchema.TableColumn col in cols){
string propName = col.PropertyName;
string varType = Utility.GetVariableType(col.DataType, col.IsNullable, lang);
%>
[XmlAttribute("<%=propName%>")]
public <%=varType%> <%=propName%>
{
get { return GetColumnValue[<]<%= varType%>>; }
set
{
MarkDirty();
SetColumnValue("<%=col.ColumnName %>", value);
}
}
<%
}
%>
```

为每个表类生成的属性将遵循清单 8-15 的模板，其中包含了调用 `MarkDirty` 方法。如果所有代码都是手动编写的，那个简单的调用可能是一个乏味的细节。而在这里，调用 `MarkDirty` 是自动的，因此每次属性值更改时都能可靠地调用它。此示例来自标准模板，但您可以将其更改为检查新值是否与当前值不同，并仅在适当时调用 `MarkDirty`。如果您现在想这样做，可以将标准模板复制到您自己的模板目录，并调整此模板使其类似于清单 8-16。

8601Ch08CMP2 8/24/07 3:57 PM Page 245

第 8 章 ■ 生成的数据访问层

**245**

`清单 8-16`. 调整后的表属性模板

```
<%
foreach(TableSchema.TableColumn col in cols){
string propName = col.PropertyName;
string varType = Utility.GetVariableType(col.DataType, col.IsNullable, lang);
%>
[XmlAttribute("<%=propName%>")]
public <%=varType%> <%=propName%>
{
get { return GetColumnValue[<]<%= varType%>>; }
set
{
if (!GetColumnValue[<]<%= varType%>> ➥
.Equals(value)) {
MarkDirty();
SetColumnValue("<%=col.ColumnName %>", value);
}
}
}
<%
}
%>
```

现在，类只会在属性值更改时被标记为已修改。因为模板中的任何内容都可以更改，所以您可以进行更多改动。从 SubSonic 生成的代码还利用了部分类，这通常用于 ASP.NET 页面后台代码文件。

■ `注意` SubSonic 能够很好地生成非常一致的代码，使得 `MarkDirty` 方法在属性值更改时总是被调用。类似地，您可以使用 Spring Framework 进行所谓的依赖注入，以向您的方法和属性添加类似行为，而无需生成代码甚至重新编译现有代码。（参见 http://www.springframework.net。）

**部分类**


