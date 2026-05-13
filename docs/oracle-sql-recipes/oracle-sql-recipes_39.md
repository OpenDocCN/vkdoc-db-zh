# 第 16 章 ■ 大对象（LOB）

大对象，通常被称为 LOB，似乎是硬盘驱动器制造商的业务支柱。

无论是图像、文字处理文档、视频片段、整部电影还是 XML 文档，大型对象经常出现在网站上，供*希望立即获取*的客户下载。

将这些类型的对象存储在磁盘上只是故事的一半——它们还需要易于管理、组织和检索。大对象不仅需要能够轻松快速地访问，还需要进行备份，并与其关联的元数据保持一致。换句话说，具有高度冗余、可用性和可恢复性的 Oracle 数据库成为了存储大对象的理想平台。

大对象有多种不同类型。字符大对象（CLOB）包含各种可能超过 4000 字符（`VARCHAR2`列的最大大小）的基于文本的对象。

CLOB 包括文字处理文档、XML 文档、日志文件等，这些通常更方便地作为数据库行中的单个列进行访问。

相比之下，二进制大对象（BLOB）与特定的区域设置或字符集无关，代表的是通常需要某种查看器才能阅读的二进制信息。图像和视频片段是最常见的 BLOB 形式。解码 BLOB 内容的任务通常由中间件或客户端应用程序执行。BLOB 的类型（如 JPEG 或 MPEG）可以存储在 BLOB 本身内部，也可以作为 BLOB 所在行的另一列存储。

对于那些将大对象存储在数据库中不可行的应用程序，可以使用 BFILE。`BFILE`是一种数据类型，其元数据存储在数据库中，但实际内容存储在数据库外部的操作系统文件中。本质上，`BFILE`对象是指向操作系统文件的指针。

在本章中，我们将介绍一些涉及 CLOB、BLOB 和 BFILE 对象的任务。我们将展示如何将大对象加载到数据库中。我们将展示如何使用`SQL*Loader`执行批量加载。当然，我们还将展示如何查询大对象数据，以及如何使用 HTTP 协议检索大对象。

当没有其他便捷方式（例如从磁盘文件系统或通过其他文件传输协议如 FTP）加载大对象时，通过 HTTP 进行检索非常方便。

##### 16-1. 将大型文档加载到 CLOB 列

### 问题

您希望在数据库中管理大型文本文件。您的文件位于文件系统（本地文件系统、Windows 网络共享或 NFS 挂载目录）的某个目录中，并且需要将它们加载到数据库表中，该表包含每个文件的其他属性。

### 解决方案

创建一个目录对象、一个序列和一个表来保存您的 CLOB 文档，如下所示：
```
create directory lob_src as '/Download/Docs2Load';

create sequence doc_seq;

create table txt_docs
(
  doc_num number,
  doc_nm varchar2(100),
  doc_clb clob,
  ins_ts timestamp
);
```

接下来，创建一个匿名 PL/SQL 块（或存储过程），以创建对文件系统中文件的引用（LOB 指针），插入一行包含空 CLOB 列的数据到数据库中，然后使用内置的 PL/SQL 过程填充刚插入行中的 CLOB 列。应如下所示：
```
declare
  src_clb bfile;        /* 指向文件系统上的源 CLOB */
  dst_clb clob;         /* 表中的目标 CLOB */
  src_doc_nm varchar2(100) := 'Baseball Roster.doc';
  src_offset integer := 1;   /* 在源 CLOB 中的起始位置 */
  dst_offset integer := 1;   /* 在目标 CLOB 中的起始位置 */
  lang_ctx integer := dbms_lob.default_lang_ctx;
  warning_msg number;   /* 如果找到不可转换的字符，则返回警告值 */
begin
  src_clb := bfilename('LOB_SRC',src_doc_nm); -- 将指针分配给 Oracle 目录内的文件
  insert into txt_docs (doc_num, doc_nm, doc_clb, ins_ts)
    values(doc_seq.nextval, src_doc_nm, empty_clob(), systimestamp)
    returning doc_clb into dst_clb; -- 首先创建 LOB 占位列
  dbms_lob.open(src_clb, dbms_lob.lob_readonly);

  dbms_lob.loadclobfromfile
  (
    dest_lob     => dst_clb,
    src_bfile    => src_clb,
    amount       => dbms_lob.lobmaxsize,
    dest_offset  => dst_offset,
    src_offset   => src_offset,
    bfile_csid   => dbms_lob.default_csid,
    lang_context => lang_ctx,
    warning      => warning_msg
  );

  dbms_lob.close(src_clb);

  commit;

  dbms_output.put_line('已将 CLOB 写入表：' || src_doc_nm);
end;
```

该过程创建了一个指向文件系统上 CLOB 的指针`SRC_CLB`，初始化了包含 CLOB 的数据库行，并在 PL/SQL 变量`DST_CLB`中返回了一个指向新插入行中空 CLOB 列的指针。`DBMS_LOB.LOADCLOBFROMFILE`使用变量`SRC_CLB`访问文件系统中的 LOB，使用变量`DST_CLB`访问表中的 CLOB 列，并将文档从文件系统复制到 CLOB 列。最后，提交更改。

### 工作原理

各种文档和图像位于文件系统目录中，该目录包含以下文件：
```
[oracle@dw ~]$ ls -l /Download/Docs2Load
total 718
-rwxr-xr-x 1 root root  80896 Mar 15 22:59 Baseball Roster.txt
-rwxr-xr-x 1 root root  47104 Jan 25 17:40 Display Table.php
-rwxr-xr-x 1 root root  78336 Apr 11 23:20 My Antiques.docx
-rwxr-xr-x 1 root root 451638 May 30 22:08 Screen Capture.bmp
-rwxr-xr-x 1 root root  76800 Feb 16 23:02 Water Table Analysis.doc
[oracle@dw ~]$
```

在解决方案中，我们创建了一个 Oracle 目录对象`LOB_SRC`来引用操作系统文件目录。Oracle*目录对象*是物理文件目录的逻辑抽象。它有许多好处和用途；在此示例中，我们使用它从 PL/SQL 块访问大对象。从 DBA 的角度来看，它具有透明性和安全性的优势。DBA 可以更改目录对象引用的文件系统，而无需更改任何 SQL 或 PL/SQL 代码，因为目录对象名称保持不变。此外，DBA 可以向一个或多个用户授予目录对象的权限，从而控制谁可以读取或写入文件系统目录中的对象。

在`SRC_DOC_NM`变量中指定文件名并运行匿名块后，输出如下：
```
已将 CLOB 写入表：Baseball Roster.txt
```

现在您可以对表运行以下`SELECT`语句，看到 CLOB 已被加载：
```
SQL> select doc_num, doc_nm, ins_ts, length(doc_clb) from txt_docs;

DOC_NUM DOC_NM                     INS_TS                       LENGTH(DOC_CLB)
------------ ---------------------- ----------------------------- ------------------
         1 Baseball Roster.txt     07-JUN-09 07.52.41.229794 PM            80896
```

当您逐步进行时，PL/SQL 块中的步骤非常直接：
1. 使用`BFILENAME`函数创建指向磁盘上 CLOB 的指针。
2. 在数据库表中创建新行，并使用`EMPTY_CLOB`初始化 CLOB 列。
3. 使用`DBMS_LOB.OPEN`打开磁盘上的 CLOB。
4. 使用`DBMS_LOB.LOADCLOBFROMFILE`将 CLOB 从磁盘复制到表中。
5. 使用`DBMS_LOB.CLOSE`关闭源 CLOB。
6. 提交事务。

解决方案中展示了`DBMS_LOB.LOADCLOBFROMFILE`的大多数默认值。例如，您通常希望从源到目标从头开始复制（`SRC_OFFSET`和`DST_OFFSET`），使用默认语言上下文（`DBMS_LOB.DEFAULT_LANG_CTX`），并将整个 CLOB 复制到数据库列中（`DBMS_LOB.LOBMAXSIZE`）。

**注意** `DBMS_LOB.LOADCLOBFROMFILE`的替代方法是`DBMS_LOB.LOADFROMFILE`。它的参数较少，使用起来更简单；但是，它不支持多字节字符集。


