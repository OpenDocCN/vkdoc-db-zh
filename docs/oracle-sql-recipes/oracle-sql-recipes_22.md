# 第 2 章 ■ 汇总与聚合数据

```sql
create or replace type body t_list_of_strings is
  static function odciaggregateinitialize
    (agg_context in out t_list_of_strings)
    return number is
  begin
    agg_context := t_list_of_strings(null);
    return odciconst.success;
  end;

  member function odciaggregateiterate
    (self in out t_list_of_strings,
     next_string_to_add in varchar2 )
    return number is
  begin
    self.string_list := self.string_list || ' , ' || next_string_to_add;
    return odciconst.success;
  end;

  member function odciaggregatemerge
    (self in out t_list_of_strings,
     para_context in t_list_of_strings)
    return number is
  begin
    self.string_list := self.string_list || ' , ' || para_context.string_list;
    return odciconst.success;
  end;

  member function odciaggregateterminate
    (self in t_list_of_strings,
     final_list_to_return out varchar2,
     flags in number)
    return number is
  begin
    final_list_to_return := ltrim(rtrim(self.string_list, ' , '), ' , ');
    return odciconst.success;
  end;
end;
/
```

有了类型和类型体，我们构建将在实际 SQL 语句中使用的函数来生成自定义聚合数据。

```sql
create or replace function string_to_list
  (input_string varchar2)
  return varchar2
  parallel_enable
  aggregate using t_list_of_strings;
/
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

