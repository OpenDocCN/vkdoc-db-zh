# 客户端与 REF 游标

请记住，一旦 `REF` 游标被返回给客户端，客户端便 `拥有` 了它。您的 PL/SQL 程序打开 `REF` 游标（有时基于从客户端接收的输入），然后将打开的结果集返回给客户端。清单 10-10 提供了一个 Java 程序的简要示例，该程序用于接收清单 10-9 中概述的 PL/SQL 程序返回的结果集。

## 清单 10-10. 用于处理 REF 游标的 Java 程序 (11.2.0.1)

```java
import java.sql.*;
import java.io.*;
import oracle.jdbc.driver.*;
class ProdConRefCursor
{
public static void main (String args [ ])
throws SQLException, ClassNotFoundException
{
   String query  = "BEGIN " + "product_pkg.get_concepts( ?, ?); " + "end;";
   DriverManager.registerDriver
        (new oracle.jdbc.driver.OracleDriver());
   Connection conn =
       DriverManager.getConnection
       ("jdbc:oracle:thin:@proximo-dev:1521:mozartora112dev", "mktg", "mktg");
   Statement trace = conn.createStatement();
   CallableStatement  cstmt = conn.prepareCall(query);
   cstmt.setInt(1, 37);
   cstmt.registerOutParameter(2 ,OracleTypes.CURSOR);
   cstmt.execute();
   ResultSet rset = (ResultSet)cstmt.getObject(2);
   for(int i = 0;  rset.next(); i++ )
        System.out.println( "rset " + rset.getString(1) );
   rset.close();
}
}
```

这是一个相当简单但直截了当的客户端接受 `REF` 游标返回结果集的示例。其关键部分是以下代码片段：

`cstmt.registerOutParameter(2 ,OracleTypes.CURSOR);`
`cstmt.execute();`
`ResultSet rset = (ResultSet)cstmt.getObject(2);`

您的返回类型是一个 `OracleTypes` 类型的游标。一旦收到游标，您就在客户端迭代每条记录，直到没有更多记录可供处理，如下所示：

`for(int i = 0;  rset.next(); i++ )`
`        System.out.println( "rset " + rset.getString(1) );`

最后，关闭 `REF` 游标是客户端的责任。它不能将 `REF` 游标传回给您的 PL/SQL 程序。因此，一旦它处理完 `REF` 游标返回的记录，只需使用以下语句关闭游标：

`rset.close();`

