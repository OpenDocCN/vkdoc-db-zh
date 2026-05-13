# 对象权限的授予

要访问另一个方案中的对象，需要授予相应的对象权限。首先，我在`data_access`方案中创建了一个表和一个过程，以便有一个具体的示例。

```
SQL> create table data_access.t(x number);
Table created.

SQL> create or replace procedure data_access.d(
  2    i_x                        number
  3  )
  4  is
  5  begin
  6    insert into t(x) values (abs(i_x));
  7  end d;
  8  /
```

![images](img/square.jpg) 注意：要创建这些对象，你必须使用具有“Create any table”和“Create any procedure”权限的用户登录，或者将`create session`、`table`和`procedure`权限授予`data_access`。

如果没有显式的授权，过程`data_access.d`无法从方案`business_logic`中的子程序访问：

```
SQL> create or replace procedure business_logic.b(
  2    i_x                        number
  3  )
  4  is
  5  begin
  6    data_access.d(trunc(i_x));
  7  end b;
  8  /
Warning: Procedure created with compilation errors.

SQL> show error
Errors for PROCEDURE BUSINESS_LOGIC.B:

LINE/COL ERROR
-------- ------------------------------------------------------
6/3      PL/SQL: Statement ignored
6/3      PLS-00201: identifier 'DATA_ACCESS.D' must be declared
```

在授予执行权限后，我可以成功编译该过程。

```
SQL> grant execute on data_access.d to business_logic;
Grant succeeded.

SQL> alter procedure data_access.d compile;
Procedure altered.
```

授权是一种灵活但属于底层的结构：

*   授权总是针对单个对象。每个接口包都需要单独授权。
*   授权总是授予特定的方案。仅仅因为`business_logic`可以访问`data_access.d`，并不意味着`presentation`也可以访问`data_access.d`。通过仅将`presentation`的权限授予`business_logic`而不授予`data_access`，你可以强制实施严格的分层，其中所有从`presentation`到`data_access`的调用都必须通过`business_logic`。授权提供了选择性的信息隐藏，而包要么将某个项导出给所有客户端，要么不导出给任何客户端。
*   如果两个方案需要在一组对象上获得相同的权限（例如，图 11-3 中的`Finance Logic`和`Payment Logic`对于`Business Framework`），则必须分别进行授权，因为在定义者权限过程中不考虑角色。
*   不可能授予一个方案（例如，包含生成代码的方案）访问另一个方案所有对象的权限。每个对象都需要单独授权。
*   对于表，`select`、`insert`、`update`和`delete`权限可以分别授予。因此，通过将 DML 限制在单个方案内，并允许其他模块直接读取以获得最佳性能，方案可以强制实施数据一致性。使用定义者权限过程访问其他方案中的对象没有性能开销。

由于授权是如此底层，并且一个大型应用程序可能需要成千上万的授权，因此最好从元模型生成授权，如图 11-4 所示。

![images](img/square.jpg) 注意：单个对象的所有权限可在视图`all_tab_privs`中查看。系统权限、角色权限和列权限可分别在`all_sys_privs`、`all_role_privs`和`all_column_privs`中找到。

