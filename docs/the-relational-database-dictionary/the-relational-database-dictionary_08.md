# 关系数据库词典（扩展版）

**1**

[www.it-ebooks.info](http://www.it-ebooks.info/)

[www.it-ebooks.info](http://www.it-ebooks.info/)

![](img/00029.png)

### A

`A` 一种关系完备的、“精简指令集”形式的关系代数，仅有两个原始运算符——`REMOVE`（本质上是除一个属性外对所有属性的投影）和 `NOR` 或 `NAND`（参见相应词条）的代数类比物。这个名称是一个双重递归首字母缩略词：它代表 *ALGEBRA*（代数），而 *ALGEBRA* 本身又代表 *A Logical Genesis Explains Basic Relational Algebra*（一个逻辑起源解释基本关系代数）。正如这个扩展名称所示，它的设计方式强调了与谓词逻辑（参见）学科的密切关系和坚实基础。更多细节可以在 C.J.戴特（C. J. Date）和休·达文（Hugh Darwen）所著的《数据库、类型与关系模型：第三宣言》（*Databases, Types, and the Relational Model: The Third Manifesto*）（第 3 版，Addison-Wesley, 2006）一书中找到。

*注*：该书使用实心箭头 ◄ 和 ► 来界定 `A` 运算符名称，例如 ◄NOR►，以区分这些运算符与谓词逻辑或 **Tutorial D** 中或两者中具有相同名称的运算符，但此处特意省略了这些箭头。更重要的是，该书实际上并没有将 `NOR` 或 `NAND` 定义为原始的 `A` 运算符；相反，它将 `A` 定义为包含显式的 `NOT`、`OR` 和 `AND` 运算符。但它随后指出，(a) 移除 `OR` 或 `AND` 中的任何一个不会导致功能损失，并且 (b) `NOT` 和保留下来的 `OR` 或 `AND` 可以合并成一个运算符——`NOT` 和 `OR` 合并为 `NOR`，或 `NOT` 和 `AND` 合并为 `NAND`。因此，将 `NOR` 或 `NAND`（如同 `REMOVE`，参见）视为 `A` 的一个原始运算符并无大碍。

`Abelian group` *参见* group (mathematics)（群（数学））。

`absolute complement` *参见* complement (set theory)（补集（集合论））。

