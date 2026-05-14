# 第 2 章 - BIDS 与 SSMS

### 代码片段示例

```
IF EXISTS
(
    SELECT *
    FROM sys.objects
    WHERE object_id = OBJECT_ID(N'[$SchemaName$].[$Tablename$]')
    AND type in (N'U')
)
DROP TABLE [$SchemaName$].[$Tablename$];
GO

CREATE TABLE $SchemaName$.$Tablename$
(
    $column1$ $datatype1$,
    $column2$ $datatype2$
);
GO

$end$]]>
</Code>

</Snippet>

</CodeSnippet>

</CodeSnippets>
```

### 代码片段关键组件

代码片段中的一些关键组件在清单 2-4 中突出显示，并在此详细说明：[www.it-ebooks.info](http://www.it-ebooks.info/)

**标题** 是 Management Studio 用来唯一标识你的自定义片段的名称。将片段添加到集合的向导如果检测到与同一文件夹路径中的现有片段发生冲突，将强制重命名该片段。

**字面量** 定义了代码片段中可以快速替换的部分。使用 Tab 键可以在代码中的字面量之间切换。字面量可以定义为默认值，该值可以设置为参数。将参数用作字面量的默认值，是“为模板参数指定值”实用工具能被用于代码片段的关键。

**代码** 包含要插入到脚本中的实际代码。代码块将按原样插入到 CDATA 块中。它将保留空白字符，并用指定的默认值替换字面量。

使用片段插入器插入代码片段后，你可以使用 Tab 键在查询中导航，方法是自动高亮显示替换点并用你自己的字符串替换它们。对于重用代码的更复杂片段，利用参数可能是一个更好的主意，如我们在清单 2-4 中所示。这使你能够利用 Management Studio 中的“为模板参数指定值”功能。

### 用于 SSIS 的查询

在 Management Studio 中创建的查询可以保存在文件系统上，以便轻松地并入 SSIS。例如，OLE DB 源组件在其编辑器中有一个“浏览”按钮，允许你将保存在文件系统上的查询作为源组件的 SQL 命令导入。仅当使用“SQL 命令”作为数据访问模式时，此选项才可用。

如果查找组件设置为使用 SQL 查询的结果，也可以导入查询。

通过此功能，开发人员可以更轻松地编写用于提取数据的 SQL，并根据需要直接将其导入到他们的 SSIS 包中。将此功能与 Management Studio 的代码片段功能相结合，复杂的连接和 where 子句可以存储起来，以便于在任何脚本中访问。

### 总结

SQL Server 12 通过其 Business Intelligence Development Studio 提供了整个商业智能工作所需的功能。本章介绍了工具库中一个重要工具——Integration Services——的所有新功能和增强功能。我们还介绍了维护和管理 SQL 套件的关键组件 Management Studio。最后，我们描述了所有这些组件如何共同构建一个端到端的解决方案，以及为提高开发人员代码复用性而进行的一些增强。第 3 章将引导你开发你的第一个包，并向你介绍一些你应该从一开始就养成的使用习惯的最佳实践。

[www.it-ebooks.info](http://www.it-ebooks.info/)

