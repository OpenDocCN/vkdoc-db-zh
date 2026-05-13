# 访问其他方案中的对象

与包类似，方案引入了命名空间。引用另一个方案中的对象有两种选择。完全限定表示法要求在每次引用时都指定方案名称（例如前面示例中的`data_access.d`）。或者，你可以在方案`business_logic`中为过程`d`创建一个同义词，然后直接作为`d`访问该过程。

```
SQL> create or replace synonym business_logic.d for data_access.d;
Synonym created.
```

一些开发者更喜欢使用同义词的方法，以减少输入量并避免紧密耦合。由于接口应该少而明确，我更喜欢完全限定表示法，这可以防止名称冲突。

![images](img/square.jpg) 注意：如果一个方案包含与另一个方案同名的对象，那么使用完全限定表示法将无法引用该另一个方案中的对象，因为 PL/SQL 的作用域规则会将引用解析为该对象。例如，如果在方案`business_logic`中添加了一个名为`data_access`的表，那么除了通过同义词之外，你将无法在`business_logic`中从 PL/SQL 引用`data_access`中的任何对象。

