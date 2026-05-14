# 7-14. 塑造 XML 导出数据

## 问题

你希望在导出数据为 `XML` 时，根据你的要求塑造 `XML` 结构，而不是让 `SQL Server` 决定 `XML` 输出。

## 解决方案

学习使用 `FOR XML PATH` 的微妙之处，并在 `SELECT` 子句中用它来塑造 `XML`。

本节内容实际上由几个小节组成，因此我将独立展示几个 `XML` 输出技巧。

### 无最高级元素名

仅添加 `FOR XML PATH` 会给每条创建的记录添加一个 `<ROW>` 元素。要测试这一点，请运行以下 `T-SQL` 代码片段（`C:\SQL2012DIRecipes\CH07\TopLevelElementXML.sql`）：

```sql
SELECT
    C.ID
    ,C.ClientName
    ,C.Country
FROM dbo.Client C
FOR XML PATH;
```

这应该会从示例数据中产生以下输出（为发布目的已缩短）：

```xml
<row>
  <ID>3</ID>
  <ClientName>John Smith</ClientName>
  <Country>1</Country>
</row>
<row>
  <ID>4</ID>
  <ClientName>Bauhaus Motors</ClientName>
  <Country>2</Country>
</row>
```

### 定义最高级元素名

添加要使用的元素名称（代替 `<ROW>`）的方法如下（`C:\SQL2012DIRecipes\CH07\TopLevelElementName.sql`）：

```sql
SELECT
    C.ID
    ,C.ClientName
    ,C.Country
FROM dbo.Client C
FOR XML PATH('Client');
```

输出结果为（同样，已大幅缩短）：

```xml
<Client>
  <ID>3</ID>
  <ClientName>John Smith</ClientName>
  <Country>1</Country>
</Client>
<Client>
  <ID>4</ID>
  <ClientName>Bauhaus Motors</ClientName>
  <Country>2</Country>
</Client>
```

### 添加 ROOT 元素

为了生成结构更良好的 `XML`，你可以像下面这样添加一个根元素（`C:\SQL2012DIRecipes\CH07\RootElementXML.sql`）：

```sql
SELECT
    C.ID
    ,C.ClientName
    ,C.Country
FROM dbo.Client C
FOR XML PATH('Client'), ROOT('Root');
```



