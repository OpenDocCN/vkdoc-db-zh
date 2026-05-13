# 将 LOB 数据加载到对象列中

现在我们已经了解了如何加载到我们自己创建的简单表中，我们可能也会发现需要加载到一个包含 LOB 的复杂对象类型的表中。这在使用图像功能时最为常见。图像功能是通过一个复杂的对象类型 `ORDSYS.ORDIMAGE` 来实现的。我们需要能够告诉 `SQLLDR` 如何加载到这个类型中。

要将 LOB 加载到 `ORDIMAGE` 类型列中，我们必须更深入地了解 `ORDIMAGE` 类型的结构。使用我们要加载到的表，并在 `SQL*Plus` 中对该表执行 `DESCRIBE` 命令，我们发现我们有一个名为 `IMAGE` 的列，其类型为 `ORDSYS.ORDIMAGE`，我们最终希望加载到 `IMAGE.SOURCE.LOCALDATA` 中。以下示例仅在您安装并配置了 Oracle Text 的情况下才有效；否则，数据类型 `ORDSYS.ORDIMAGE` 将是一个未知类型：

```sql
SQL> create table image_load(
id number,
name varchar2(255),
image ordsys.ordimage);
Table created.
SQL> desc image_load
Name                                     Null?    Type
---------------------------------------- -------- ------------------------
ID                                                NUMBER
NAME                                              VARCHAR2(255)
IMAGE                                             ORDSYS.ORDIMAGE
SQL> desc ordsys.ordimage
Name                                             Null?    Type
------------------------------------------------ -------- ----------------
SOURCE                                                    ORDSYS.ORDSOURCE
HEIGHT                                                    NUMBER(38)
WIDTH                                                     NUMBER(38)
CONTENTLENGTH                                             NUMBER(38)
FILEFORMAT                                                VARCHAR2(4000)
...
SQL> desc ordsys.ordsource
Name                                             Null?      Type
------------------------------------------------ ---------- --------------
LOCALDATA                                                   BLOB
SRCTYPE                                                     VARCHAR2(4000)
SRCLOCATION                                                 VARCHAR2(4000)
SRCNAME                                                     VARCHAR2(4000)
UPDATETIME                                                  DATE
...
```

> 注意
>
> 您可以在 `SQL*Plus` 中执行 `SET DESC DEPTH ALL` 或 `SET DESC DEPTH <n>` 命令来一次性显示整个层次结构。考虑到描述 `ORDSYS.ORDIMAGE` 类型的输出会很长，我选择分步进行。

因此，加载数据的控制文件可能如下所示：

```
LOAD DATA
INFILE *
INTO TABLE image_load
REPLACE
FIELDS TERMINATED BY ','
( ID,
NAME,
file_name FILLER,
IMAGE column object
(
SOURCE column object
(
LOCALDATA LOBFILE (file_name) TERMINATED BY EOF
NULLIF file_name = 'NONE'
)
)
)
BEGINDATA
1,icons,icons.gif
```

我在这里引入了两个新的结构：

*   `COLUMN OBJECT`：这告诉 `SQLLDR` 这不是一个列名；相反，它是一个列名的一部分。它不映射到输入文件中的字段，而是用于构建加载期间要使用的正确对象列引用。在上面的文件中，我们有两个 column object 标签，一个嵌套在另一个里面。因此，将使用的列名是 `IMAGE.SOURCE.LOCALDATA`，这正是我们需要的。请注意，我们没有加载这两个对象类型的其他任何属性（例如，`IMAGE.HEIGHT`、`IMAGE.CONTENTLENGTH` 和 `IMAGE.SOURCE.SRCTYPE`）。我们很快会看到如何填充它们。

*   `NULLIF FILE_NAME = 'NONE'`：这告诉 `SQLLDR`，如果 `FILE_NAME` 字段包含单词 `NONE`，则将一个 `NULL` 加载到对象列中。

一旦加载了 Oracle Text 类型，通常需要使用 `PL/SQL` 对加载的数据进行后处理，以便 Oracle Text 对其进行操作。例如，对于前面的数据，您可能需要运行以下命令来正确设置图像的属性：

```sql
begin
for c in ( select * from image_load ) loop
c.image.setproperties;
end loop;
end;
/
```

`SETPROPERTIES` 是由 `ORDSYS.ORDIMAGE` 类型提供的一个对象方法，它处理图像本身，并使用适当的值更新对象的其余属性。

## 如何从存储过程中调用 SQLLDR？

简短的答案是：您无法做到。`SQLLDR` 不是一个 `API`；它不是可以调用的东西。`SQLLDR` 是一个命令行程序。您当然可以用 `Java` 或 `C` 编写一个运行 `SQLLDR` 的外部过程，但这与“调用” `SQLLDR` 不同。加载将在另一个会话中进行，并且不受您的事务控制。此外，您将必须解析生成的日志文件以确定加载是否成功以及成功程度（即，在错误终止加载之前加载了多少行）。我不建议从存储过程中调用 `SQLLDR`。

过去，您可能已经实现了自己类似 `SQLLDR` 的过程。例如，选项可能如下：

*   用 `PL/SQL` 编写一个迷你 `SQLLDR`。它既可以使用 `BFILES` 读取二进制数据，也可以使用 `UTL_FILE` 读取文本来进行解析和加载。

*   用 `Java` 编写一个迷你 `SQLLDR`。这可以比基于 `PL/SQL` 的加载器更复杂，并且可以利用许多可用的 `Java` 例程。

*   用 `C` 编写一个 `SQLLDR`，并将其作为外部过程调用。

我想通过讨论几个不是立即显而易见的话题来结束关于 `SQLLDR` 的话题。

## SQLLDR 注意事项

在本节中，我们将讨论使用 `SQLLDR` 时需要注意的一些事项。

### TRUNCATE 的行为似乎不同

`SQLLDR` 的 `TRUNCATE` 选项可能看起来与 `SQL*Plus` 或其他任何工具中的 `TRUNCATE` 工作方式不同。`SQLLDR` 假设您将使用类似数量的数据重新加载表，因此使用了 `TRUNCATE` 的扩展形式。具体来说，它执行以下操作：

```sql
truncate table t reuse storage
```

`REUSE STORAGE` 选项不会释放已分配的区间——它只是将它们标记为空闲空间。如果这不是期望的结果，您可以在执行 `SQLLDR` 之前截断表。

### SQLLDR 默认为 CHAR(255)

这个问题经常出现，我决定在本章中讨论两次。输入字段的默认长度是 255 个字符。如果您的字段长于此值，您将收到错误消息：

```
Record N: Rejected - Error on table T, column C.
Field in data file exceeds maximum length
```

这并不意味着数据无法放入数据库列；而是表明 `SQLLDR` 期望输入数据为 255 字节或更少，但它收到的数据量略多于此。解决方案很简单，就是在控制文件中使用 `CHAR(N)`，其中 `N` 足够大以容纳输入文件中最大的字段长度。请参阅前面“使用 SQLLDR 加载数据常见问题解答”部分中的第一个示例。

### 命令行覆盖控制文件

许多 `SQLLDR` 选项可以放在控制文件中，也可以在命令行上使用。例如，我可以使用 `INFILE FILENAME`，也可以使用 `SQLLDR ... DATA=FILENAME`。命令行会覆盖控制文件中的任何选项。您不能指望控制文件中的选项一定会被使用，因为执行 `SQLLDR` 的人可以覆盖它们。

