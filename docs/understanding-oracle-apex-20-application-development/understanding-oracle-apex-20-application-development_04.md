# 示例应用

随着本书讲解 APEX 应用构建器的各个部分，它将指导你开发一个名为`Employee Demo`的小型应用。我鼓励你在阅读的同时构建自己的应用版本。你可以通过访问 URL `apex.oracle.com/pls/apex/f?p=91392:1` 来运行我的应用版本。你也可以从 Apress 网站下载应用源代码并将其导入到你自己的工作区。

与许多书中的示例应用不同，这个应用本身并没有做什么特别有趣的事情。相反，每个页面的构建都是为了说明一种或多种技术。有些页面具有相似的功能，旨在展示不同实现技术之间的权衡。

`Employee Demo` 应用使用了每个 APEX 工作区都可用的 `DEPT` 和 `EMP` 数据库表。`DEPT` 表列出了公司的部门，而 `EMP` 表列出了这些部门中的员工。它们的列如下所示：

```
DEPT(DeptNo, DName, Loc)
EMP (EmpNo, EName, Job, Mgr, HireDate, Sal, Comm, DeptNo)
```

`DEPT` 表的键是 `DeptNo`，`EMP` 表的键是 `EmpNo`。每个表都有一个内置序列用于为这些键生成唯一值，以及一个关联的插入触发器。如果你向其中一个表插入记录时省略了键值，触发器将自动从相应的序列生成一个键值。

`Employee Demo` 应用假设 `EMP` 表已被修改，增加了一个类型为 `char(1)` 的额外列 `OffSite`。如果员工在部门办公室工作，该列的值将为 'N'，如果员工在异地工作，则为 'Y'。供你参考，以下是向 `EMP` 表添加此新列所需的 SQL 代码。

```
alter table EMP
add OffSite char(1);
```

修改表结构后，你还需要为每个现有员工分配一个 `OffSite` 值。在我的 `Employee Demo` 应用中，员工 SCOTT、ALLEN、WARD 和 TURNER 在异地工作；其他员工在办公室工作。第 1 章描述了如何在工作区中导入这些表（如果它们尚不存在），并讨论了进行这些修改所需的 APEX 工具。

