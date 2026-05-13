# 第 16 章 ■ 大型对象

##### 16-2. 将图像数据加载到 BLOB 列

### 问题

你有各种 JPG、PNG、GIF 和 BMP 图像文件需要在数据库中进行管理和备份，并且需要一种方法来执行初始加载到数据库中。

### 解决方案

加载二进制大对象比加载基于文本的大对象稍微简单一些。你不必担心字符集转换——或者根本无需关心字符集；一图胜千言，数据库中的一个图像文件意味着你不需要存储包含那一千个单词的等效文本文档！

为你的图像表的唯一标识符创建一个序列，以及表本身：

```sql
create sequence img_seq;

create table image
(
  img_num number,
  img_nm varchar2(100),
  img_blb blob,
  ins_ts timestamp
);
```

接下来，运行一个匿名块，将图像`SCREEN CAPTURE.BMP`加载到你的数据库表中：

```sql
declare
  src_blb bfile; /* point to source BLOB on file system */
  dst_blb blob;  /* destination BLOB in table */
  src_img_nm varchar2(100) := 'Screen Capture.bmp';
  src_offset integer := 1; /* where to start in the source BLOB */
  dst_offset integer := 1; /* where to start in the target BLOB */
begin
  src_blb := bfilename('LOB_SRC',src_img_nm);
  insert into image (img_num, img_nm, img_blb, ins_ts)
    values(img_seq.nextval, src_img_nm, empty_blob(), systimestamp)
    returning img_blb into dst_blb;
  dbms_lob.open(src_blb, dbms_lob.lob_readonly);
  dbms_lob.loadblobfromfile
  (
    dest_lob => dst_blb,
    src_bfile => src_blb,
    amount => dbms_lob.lobmaxsize,
    dest_offset => dst_offset,
    src_offset => src_offset
  );
  dbms_lob.close(src_blb);
  commit;
  dbms_output.put_line('Wrote BLOB to table: ' || src_img_nm);
end;
```

### 工作原理

将二进制对象加载到`BLOB`列中，与将文本文件加载到`CLOB`列中非常相似。

然而，与文本文档不同，你很少会遇到只将图像文件的一部分复制到`BLOB`列的情况。这是由于大多数图像文件的结构特性决定的。你要么加载整个图像，要么完全不加载。

查询表`IMAGE`，你可以看到刚刚加载的图像文件的元数据：

```sql
select img_num, img_nm, ins_ts, length(img_blb) from image;
```

```
IMG_NUM IMG_NM                  INS_TS                        LENGTH(IMG_BLB)
------- ----------------------- ----------------------------- ---------------
      1 Screen Capture.bmp     07-JUN-09 10.25.02.984755 PM           451638
```

为了进一步自动化这个过程（假设你处理的仍然是相对较少的对象），你可以创建一个存储过程来代替使用匿名过程。你的参数至少应包含要加载的文件名。以下是这样一个过程的示例：

```sql
create or replace procedure load_blob ( src_img_nm in varchar2 )
is
  src_blb bfile; /* point to source BLOB on file system */
  dst_blb blob;  /* destination BLOB in table */
  src_offset integer := 1; /* where to start in the source BLOB */
  dst_offset integer := 1; /* where to start in the target BLOB */
begin
  ... /* same code as in the anonymous block version */
end;
```

从另一个存储过程的循环或匿名块中调用该过程很容易。例如：

```sql
begin
  load_blob('Screen Capture.bmp');
end;
```

你也可以考虑对目录对象进行参数化，并为图像文件的描述创建一个新列并进行参数化。通常，文件名本身并不是图像内容的很好描述！

> **注意** 本方案使用了与本章开头`CLOB`示例相同的目录对象。正如你可能推断的那样，如果你不必担心字符转换，你可以将所有对象，无论是`CLOB`还是`BLOB`，都视为二进制对象来处理。

## 16-3. 使用 SQL*Loader 批量加载大型对象

### 问题

你想从本地文件系统加载数千个二进制对象，并希望尽量减少手动干预以及加载对象所需的时间。

### 解决方案

之前的解决方案使用了 Oracle 目录对象和 PL/SQL 过程。如果二进制文件可以从 Oracle 客户端访问，你可以使用`SQL*Loader`轻松地加载`BLOB`。以下是一个名为`load_lobs.ctl`的`SQL*Loader`控制文件：

```
load data
infile load_lobs.list
append into table image
fields terminated by ','
trailing nullcols
(
  img_num char,
  img_nm char(100),
  img_file_nm filler char(100),
  img_blb lobfile(img_file_nm) terminated by EOF,
  ins_ts "systimestamp"
)
```

这是所引用的数据文件`load_lobs.list`，包含了`BLOB`的位置：

```
101,Water Table Analysis,E:\Download\Docs2Load\Water Table Analysis.doc
102,My Antiques,My Antiques.docx
103,Screen Capture for Book,Screen Capture.bmp
```

你的`SQL*Loader`命令行将类似于这样：

```
sqlldr userid=rjb/rjb@recipes control=load_lobs.ctl
```

### 工作原理

对`LOB`使用`SQL*Loader`与对传统数据类型使用`SQL*Loader`非常相似，区别在于`SQL*Loader`的`INFILE`包含一个指向文件系统中`LOB`位置的指针。

你像在典型的`SQL*Loader`作业中一样指定其他表列。然而，对于`BLOB`数据，你需要指定两列：`BLOB`的位置和表中将容纳该`BLOB`的列。在解决方案中，`IMG_FILE_NM`列被标记为`FILLER`，这意味着`IMG_FILE_NM`不是一个真实的列，而是在稍后加载`BLOB`数据时会被引用。包含`BLOB`列的`SQL*Loader`行如下：

```
img_blb lobfile(img_file_nm) terminated by EOF,
```

`LOBFILE`子句引用了包含文件系统中`BLOB`位置的`SQL*Loader`列。注意在例子中，你可以提供一个绝对文件位置（`E:\Download\Docs2Load\Water Table Analysis.doc`）或一个相对于你运行`SQL*Loader`的目录的文件名。

最后，子句`TERMINATED BY EOF`告诉`SQL*Loader`在文件结尾处停止读取`BLOB`。在其他场景中，你可能只想读取`CLOB`或`BLOB`的一部分，直到特定的分隔符，你可以用`TERMINATED BY`子句中的一个常量以及一个可选的最大大小来指定，如下例所示：

```
paragraph_clb lobfile(chapter_file_nm) char(2000) terminated by '[PARA]'
```

从解决方案中的`BLOB`被加载到`IMAGE`表后，表的内容如下所示：

```sql
select img_num, img_nm, ins_ts, length(img_blb) from image;
```

```
IMG_NUM IMG_NM                  INS_TS                        LENGTH(IMG_BLB)
------- ----------------------- ----------------------------- ---------------
    101 Water Table Analysis    11-JUN-09 10.49.44.154432000 PM        76800
    102 My Antiques            11-JUN-09 10.49.44.154432000 PM        78336
    103 Screen Capture for Book 11-JUN-09 10.49.44.154432000 PM       451638
3 rows selected
```

正如你可能预期的那样，你可以在同一个`SQL*Loader`作业中加载多个`BLOB`和`CLOB`列，只需要确保你的`LOBFILE`定位器被定义为`FILLER`列（如果它们不是表中实际列的值）。

这种方法非常有效，即使客户端与数据库实例位于不同的平台上也是如此。在提供的解决方案中，客户端和`BLOB`对象位于 Microsoft Windows 平台上（MS-Word 的`DOC`和`DOCX`文件本质上是二进制文件），而包含带有`BLOB`列的表的数据库则位于 Linux 平台上。请记住，当我们使用`BLOB`大型对象类型时，我们不必担心平台字符集转换问题。



