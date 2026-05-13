# 第 2 章 ■ 汇总与聚合数据

##### 2-10. 构建你自己的聚合函数

### 问题

你需要实现一个自定义聚合，以与 Oracle 现有的聚合函数和分组机制协同工作。你特别想要将多行中的字符串聚合到一行，使得每行的文本一个接一个地连接起来。你见过其他数据库支持此功能，但找不到 Oracle 的等效函数。

### 解决方案

Oracle 提供了一个框架，用于编写你自己的聚合函数，无论是提供 Oracle 中原本没有的聚合，还是为常见的聚合函数开发具有不同行为的你自己的定制版本。我们的配方创建了一个新的聚合函数`STRING_TO_LIST`，以及基于 Oracle 自定义聚合模板的支持类型定义和类型体。

以下类型定义定义了 Oracle 对自定义聚合类型要求的四个必需函数。

```sql
create or replace type t_list_of_strings as object (
  string_list varchar2(4000),
  static function odciaggregateinitialize
    (agg_context in out t_list_of_strings)
    return number,
  member function odciaggregateiterate
    (self in out t_list_of_strings,
     next_string_to_add in varchar2 )
    return number,
  member function odciaggregatemerge
    (self in out t_list_of_strings,
     para_context in t_list_of_strings)
    return number,
  member function odciaggregateterminate
    (self in t_list_of_strings,
     final_list_to_return out varchar2,
     flags in number)
    return number
);
/
```

我们将`STRING_LIST`参数限制在 4000，尽管 PL/SQL 支持高达 32000，以确保我们不传递 Oracle 纯 SQL 所支持的阈值。这四个必需函数中的每一个都实现了将一组字符串聚合成一个列表的各个阶段。

[www.it-ebooks.info](http://www.it-ebooks.info/)

