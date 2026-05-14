# 使用自定义模式指令

日期格式化是自定义模式指令的一个典型用例。Ben Newman 有一篇优秀的博文，题为“可复用的 GraphQL 模式指令^(²⁷)，”（[`https://dev-blog.apollodata.com/reusable-graphql-schema-directives-131fb3a177d1`](https://dev-blog.apollodata.com/reusable-graphql-schema-directives-131fb3a177d1))，其中相当详细地介绍了如何实现它们。

这里我将仅让你感受一下它的样子（参考 Ben Newman 的博文）。

首先，定义一个带有默认格式和一个参数的指令：

```
1   directive @formattableDate(
2     defaultFormat: String = "mmmm d, yyyy"
3   ) on FIELD_DEFINITION
4
5   scalar Date
6
7   type Query {
8     today: Date @formattableDate
9   }
```

该指令可以被整合进一个模式中，并以不同的格式使用：

```
1   import { graphql } from "graphql";
2   import { makeExecutableSchema } from "graphql-tools";
3
4   const schema = makeExecutableSchema({
5     typeDefs,
6     schemaDirectives: {
7       formattableDate: FormattableDateDirective
8     }
9   });
10
11   graphql(schema, `query { today }`).then(result => {
12     // 使用默认的 "mmmm d, yyyy" 格式记录：
13     console.log(result.data.today);
14   });
15
16   graphql(schema, `query {
17     today(format: "d mmm yyyy")
18   }`).then(result => {
19     // 使用请求的 "d mmm yyyy" 格式记录：
20     console.log(result.data.today);
21   });
```

如前所述，Ben Newman 的博文包含了实现此功能所需的所有细节。

