# 第 16 章 ■ 大对象

是使用 `SQL*Loader` 还是使用带目录对象的 PL/SQL 来加载你的 `BLOB` 和 `CLOB`，这取决于多种因素。如果你不需要任何过程逻辑来加载数据，并且你的开发人员更熟悉 `SQL*Loader`，那么这可能是最佳选择。如果要加载的数据无法通过 Oracle 目录对象访问，或者数据并不总是位于相同的文件系统位置，`SQL*Loader` 也是一个不错的选择。另一方面，如果 `LOB` 与 Oracle 数据库位于同一服务器上，并且所有目录对象都已就绪，那么使用 PL/SQL 调用来加载 `LOB` 比在客户端工作站上使用 `SQL*Loader` 更高效，因为后者必须先将数据从源位置复制到客户端位置，然后再复制回数据库服务器。

##### 16-4. 使用 HTTP 访问大对象

### 问题

你需要将 `LOB` 从远程服务器加载到你的数据库，但唯一可用于传输数据的协议是 HTTP（端口 80）。

图 16-1 显示了所需的数据、其目的地，以及将数据从源移动到目标的障碍。
`图 16-1. 在有防火墙限制的情况下在服务器间传输 LOB`

包含图像文件服务器的网络上的防火墙限制，阻止了从数据库服务器对图像服务器的任何直接访问，只能通过端口 80 访问源系统上用于提供图像的 Web 服务器前端。换句话说，从图像服务器获取图像的唯一方式是通过 HTTP。

### 解决方案

Apress.com 的图像必须迁移到另一个数据库服务器；这些服务器相隔数千英里，并且除了 HTTP（端口 80）协议外，服务器之间没有网络访问。你没有时间执行导出/导入到 DVD 或磁带，并将 DVD 或磁带运送到本地数据库服务器的物理位置。但是，你确实有一个本地数据库表，其中包含要传输的 URL 列表。

为了促进这种传输，请使用 Oracle PL/SQL 包 `UTL_HTTP` 和 `DBMS_LOB`，通过 HTTP 协议在网络上传输文件。在此解决方案中，表 `IMG_LIST` 包含保存图像的 URL 列表，表 `WEB_IMG` 包含每个图像的元数据以及图像本身。

以下是使用 HTTP 访问远程对象并将其复制到本地数据库表的 `LOB` 列中的代码：

```sql
-- create image table with list of URLs
-- and retrieve those images via HTTP
declare
    cursor img_list is -- image URLs to retrieve
        select img_url from img_list;
    r_img_list img_list%rowtype;
    lBlob blob; -- locator for current image
    l_http_request utl_http.req;
    l_http_response utl_http.resp;
    l_raw raw(32767); -- blob buffer
begin
    dbms_output.put_line('*** Begin image load.');
    open img_list;
    loop
        fetch img_list into r_img_list;
        exit when img_list%notfound;
        -- initialize image table row with empty blob
        insert into web_img(img_num, img_url, img_blb, ins_ts)
        values(web_img_seq.nextval, r_img_list.img_url, empty_blob(), systimestamp)
        returning img_blb into lBlob; -- save pointer for local use
        dbms_output.put_line('Attempt retrieval of: ' || r_img_list.img_url);
        -- attempt to retrieve image via HTTP
        l_http_request := utl_http.begin_request(r_img_list.img_url); -- request
        l_http_response := utl_http.get_response(l_http_request); -- response back
        dbms_output.put_line(' HTTP stat code: ' || l_http_response.status_code);
        dbms_output.put_line(' HTTP resp reason phrase: ' || l_http_response.reason_phrase);
        if l_http_response.status_code = 200 then /* image file found */
            dbms_lob.open(lBlob, dbms_lob.lob_readwrite);
            begin
                loop
                    utl_http.read_raw(l_http_response, l_raw, 32767);
                    dbms_lob.writeappend(lBlob, utl_raw.length(l_raw), l_raw);
                end loop;
            exception
                when utl_http.end_of_body then
                    utl_http.end_response(l_http_response);
```



