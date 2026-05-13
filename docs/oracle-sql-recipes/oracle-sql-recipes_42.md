# 第十六章 ■ 大型对象 (LOBs)

## 工作原理

此匿名过程逐行读取 URL 表中的记录，尝试获取每个 URL 的内容，并将一行数据写入新表。为每个 URL 的成功或失败状态都进行了记录。

该解决方案使用此 URL 列表，它们存储在 `IMG_LIST` 表中：
`select img_url from img_list;`

```
IMG_URL
http://apress.com/img/masthead_logo.gif
http://apress.com/img/dealofdaysanstimer_v2.gif
http://apress.com/img/AlphaProgram_140.gif
http://apress.com/img/mini_toe_logo.gif
http://apress.com/img/banner/1515banner.gif
已选择 5 行
```

以下是用于创建图像列表、目标图像表以及该图像表的序列生成器的 SQL：

`create table img_list ( img_url varchar2(255) );`

`create table web_img ( img_num number, img_url varchar2(255), img_blb blob, ins_ts timestamp );`

`create sequence web_img_seq;`

游标 `IMG_LIST` 检索每个 URL。对于每个 URL，该过程使用两个内置的 PL/SQL 包：`DBMS_LOB` 和 `UTL_HTTP`。本章之前的解决方案使用 `DBMS_LOB.LOADBLOBFROMFILE` 将二进制对象从文件系统复制到数据库表中的 `BLOB` 列。与此相反，我们在此使用 `DBMS_LOB.WRITEAPPEND`，在通过调用 `UTL_HTTP.READ_RAW` 接收到每个 32KB 的数据块后，以 32KB 的块为单位写入图像内容。

PL/SQL 包 `UTL_HTTP` 如其名所示：向网站发送 HTTP 请求并接收 HTTP 响应。对于每个 URL，该过程执行以下步骤：

1.  使用 `UTL_HTTP.BEGIN_REQUEST` 请求 URL。
2.  使用 `UTL_HTTP.GET_RESPONSE` 接收来自网站的响应。
3.  如果响应代码为 200（找到 URL），则打开表中的 `BLOB`，否则处理下一个 URL 并返回步骤 1。
4.  在一个本地 PL/SQL 块中，使用 `UTL_HTTP.READ_RAW` 以 32K 为块读取 URL 的内容，直到到达 URL 末尾，并使用 `DBMS_LOB.WRITE_APPEND` 将每个 32K 块追加到目标 `BLOB` 的末尾。
5.  关闭目标 `BLOB`。
6.  获取下一个 URL 进行步骤 1；如果没有更多 URL，则进行步骤 7。
7.  关闭游标，提交所有事务。

运行此过程的日志表明其中一个 URL 不存在：

```
*** 开始加载图像。
尝试获取: http://apress.com/img/masthead_logo.gif
HTTP 状态码: 200
HTTP 响应原因短语: OK
图像 URL 已加载: 16 http://apress.com/img/masthead_logo.gif
尝试获取: http://apress.com/img/dealofdaysanstimer_v2.gif
HTTP 状态码: 200
HTTP 响应原因短语: OK
图像 URL 已加载: 17 http://apress.com/img/dealofdaysanstimer_v2.gif
尝试获取: http://apress.com/img/AlphaProgram_140.gif
HTTP 状态码: 200
HTTP 响应原因短语: OK
图像 URL 已加载: 18 http://apress.com/img/AlphaProgram_140.gif
尝试获取: http://apress.com/img/mini_toe_logo.gif
HTTP 状态码: 404
HTTP 响应原因短语: Not Found
此 URL 无图像
尝试获取: http://apress.com/img/banner/1515banner.gif
HTTP 状态码: 200
HTTP 响应原因短语: OK
图像 URL 已加载: 20 http://apress.com/img/banner/1515banner.gif
*** 图像导入完成。
```

毫无疑问，您在使用喜欢的 Web 浏览器时，曾多次收到过 404 错误。

运行该过程后，`WEB_IMG` 表的内容如下：

`select img_num, img_url, length(img_blb) from web_img order by img_num;`



## 16-5\. 使外部大对象（BFILEs）对数据库可用

**问题**

你想在 Oracle 表中管理你的图像和其他二进制文件，但最终用户应用程序需要从文件系统访问这些文件，并且数据库表空间中没有足够的磁盘空间来将这些图像存储为 BLOB。

**解决方案**

你可以在 Oracle 表中使用 BFILE 指针来引用文件系统上的二进制对象。此外，你可以使用 `DBMS_LOB` 和 `UTL_FILE` 包来读取和管理这些对象。

创建一个表来保存图像元数据，并使用一个序列为该表提供主键：
```sql
create table web_img2
(
  img_num number primary key,
  img_url varchar2(255),
  img_blb bfile,
  ins_ts timestamp,
  constraint ak1_web_img2 unique(img_url)
);

create sequence web_img2_seq;
```

这里有一个过程，它接受一个 Oracle 目录名称和该目录中的一个文件，如果文件存在，则将其元数据加载到 `WEB_IMG2` 表中：
```sql
create or replace procedure load_bfile ( dir_name varchar2, src_img_nm in varchar2 ) is
  src_blb bfile; /* 指向文件系统上的源 BLOB */
  file_exists boolean; /* UTL_FILE.FGETATTR 的返回值 */
  file_len number;
  blksize binary_integer;
begin
  src_blb := bfilename(dir_name, src_img_nm);
  insert into web_img2 (img_num, img_nm, img_blb, ins_ts)
  values(web_img2_seq.nextval, src_img_nm, src_blb, systimestamp);
  -- 检查文件此刻是否存在
  utl_file.fgetattr(dir_name, src_img_nm, file_exists, file_len, blksize);
  if file_exists then
    commit;
    dbms_output.put_line('Wrote BFILE pointer to table: ' || src_img_nm);
  else
    rollback;
    dbms_output.put_line('BLOB ' || src_img_nm
                         || ' in directory '
                         || dir_name
                         || ' does not exist.');
  end if;
end;
```

**工作原理**

这个解决方案与本章前面一个将对象存储在数据库中的配方非常相似；在这个解决方案中，我们仍然将元数据存储在 `WEB_IMG2` 表中，但不是 BLOB 本身。相反，我们在操作系统文件系统中存储一个指向文件的指针。这确实方便了与需要在 Oracle 之外访问文件的其他应用程序共享文件，但它可能导致引用完整性（RI）问题。通常，在引用完整性中，一个数据库表引用另一个数据库表中的行，DBMS 约束阻止删除被引用的行。然而，这里表引用的是文件系统中的文件。如果另一个应用程序或管理员删除或移动了文件，数据库表中的 BFILE 指针仍然存在，但它已无效。因此，你可以使用 `UTL_FILE.FGETATTR` 来检查操作系统目录中是否存在 BFILE。

使用存在的文件运行此过程如下所示：
```sql
begin
  load_bfile('LOB_SRC','Screen Capture.bmp');
end;
```
输出：
```
Wrote BFILE pointer to table: Screen Capture.bmp
```

相反，尝试存储一个不存在的文件会产生以下输出：
```sql
begin
  load_bfile('LOB_SRC','Cake Boss.bmp');
end;
```
输出：
```
BLOB Cake Boss.bmp in directory LOB_SRC does not exist.
```

当你查询该表时，它会返回以下结果：
```sql
select * from web_img2;
```
```
IMG_NUM IMG_NM                 IMG_BLB INS_TS
------- ------------------- ------- --------------------------------
1       Screen Capture.bmp   (BFILE) 13-JUN-09 10.43.16.737554000 PM
```

你可能想知道如何解码 `IMG_BLB` 列中引用的 BFILE 内容。`DBMS_LOB.FILEGETNAME` 过程也可以帮助你。这里是一个返回 Oracle 目录名称和目录内文件名的函数：
```sql
create or replace function get_bfile_attrs (bfile_obj in bfile)
return varchar2 is
  dir_alias varchar2(30);
  file_name varchar2(100);
begin
  dbms_lob.filegetname(bfile_obj, dir_alias, file_name);
  return('BFILE: Directory=' || dir_alias || ' File=' || file_name);
end;
```
```sql
select img_num, img_nm, get_bfile_attrs(img_blb) from web_img2;
```
```
IMG_NUM IMG_NM                 GET_BFILE_ATTRS(IMG_BLB)
------- -------------------- --------------------------------------------------
1       Screen Capture.bmp   BFILE: Directory=LOB_SRC File=Screen Capture.bmp
```
你可以使用 `DBMS_LOB` 包将文件系统上的图像或 Word 文档复制到表中的 BLOB 或 CLOB 列；请参阅本章前面的配方。

## 16-6\. 删除或更新数据库表中的 LOB

**问题**

数据库表中的几个 LOB 要么已过时，要么需要替换。

**解决方案**

要删除 LOB，你可以将 LOB 列的内容设置为 `EMPTY_CLOB()` 或 `EMPTY_BLOB()`，如本例所示：
```sql
update web_img
set img_blb = empty_blob()
where img_num = 19;
```

虽然你也可以将列 `IMG_BLB` 设置为 `NULL`，但必须在重新插入 LOB 到该列之前将其设置为 `EMPTY_CLOB()` 或 `EMPTY_BLOB()`。允许 LOB 列具有 `NULL` 值使其与可以具有 `NULL` 的其他列的使用保持一致：你可以将值未知的 LOB 列与具有零长度空 LOB 的 LOB 区分开来。

更新 LOB 列的值与更新空 LOB 列相同——检索 LOB 列的指针并使用 `DBMS_LOB` 来填充该列。如果你知道 LOB 的哪一部分需要替换，可以使用 `DBMS_LOB.COPY` 的其他参数。以下是 `DBMS_LOB.COPY` 的参数：
```
DBMS_LOB.COPY
(
  dest_lob    IN OUT NOCOPY BLOB, -- 要更新的 BLOB
  src_lob     IN BLOB,           -- 包含新内容的 BLOB
  amount      IN INTEGER,        -- 要复制的字节数
  dest_offset IN INTEGER := 1,   -- 在目标中的起始位置
  src_offset  IN INTEGER := 1    -- 在源中的起始位置
);
```
正如你可能期望的那样，Oracle 为在源 LOB 和目标 LOB 中的起始位置提供了合理的默认值。

以下是如何将新 LOB 的第 10 个字节开始的 5000 个字节复制到现有 LOB 的第 125 个字节：
```sql
dbms_lob.copy (existing_blb, new_blb, 5000, 125, 10);
```

更新 CLOB 使用相同的过程（它被重载了，就像许多其他 Oracle PL/SQL 过程一样），只是 `DEST_LOB` 和 `SRC_LOB` 是 `CLOB` 类型，并将从源字符集转换为目标字符集。

**工作原理**

由于其潜在的大小，大对象列（BLOB 和 CLOB）在 `SELECT` 和 `DDL` 语句中有几个限制，包括以下内容：

- LOB 不能用于 `SELECT DISTINCT`、`ORDER BY` 或 `GROUP BY` 子句中
- 你不能使用 LOB 列作为 `JOIN` 列
- LOB 列不允许在 `UNION`、`INTERSECTION` 和 `MINUS` 中
- LOB 列不能有位图索引、唯一索引或 B 树索引——只能有基于函数的索引或域索引
- LOB 列不能是主键



