# 使用自定义模式指令

日期格式化是自定义模式指令的一个典型用例。Ben Newman 有一篇优秀的博文，题为“可复用的 GraphQL 模式指令^(²⁷)，”（[`https://dev-blog.apollodata.com/reusable-graphql-schema-directives-131fb3a177d1`](https://dev-blog.apollodata.com/reusable-graphql-schema-directives-131fb3a177d1)），其中相当详细地介绍了如何实现它们。

这里我将仅让你感受一下它的样子（参考 Ben Newman 的博文）。

首先，定义一个带有默认格式和一个参数的指令：

```
directive @formattableDate(
  defaultFormat: String = "mmmm d, yyyy"
) on FIELD_DEFINITION

scalar Date

type Query {
  today: Date @formattableDate
}
```

该指令可以被整合进一个模式中，并以不同的格式使用：

```
import { graphql } from "graphql";
import { makeExecutableSchema } from "graphql-tools";

const schema = makeExecutableSchema({
  typeDefs,
  schemaDirectives: {
    formattableDate: FormattableDateDirective
  }
});

graphql(schema, `query { today }`).then(result => {
  // 使用默认的 "mmmm d, yyyy" 格式记录：
  console.log(result.data.today);
});

graphql(schema, `query {
  today(format: "d mmm yyyy")
}`).then(result => {
  // 使用请求的 "d mmm yyyy" 格式记录：
  console.log(result.data.today);
});
```

如前所述，Ben Newman 的博文包含了实现此功能所需的所有细节。