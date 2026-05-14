# 查询数据

为了验证 Cassandra 表是否已创建，接下来我们将运行一个 `SELECT` 语句从 `catalog` 表中选择列。添加一个方法 `select()` 来运行 `SELECT` 语句。

使用 `*` 选择所有列从 `catalog` 表中查询。运行 `SELECT` 语句作为测试，以确认我们添加的数据确实已被添加。

```
ResultSet results = session.execute("select * from datastax.catalog");
```

`ResultSet` 中的一行由 `Row` 类表示。遍历 `ResultSet` 以输出每个列的值。

```
for (Row row : results) {
        System.out.println("目录 ID: " + row.getString("catalog_id"));
        System.out.println("\n");
            System.out.println("期刊: " + row.getString("journal"));
            System.out.println("出版商: " + row.getString("publisher"));
            System.out.println("版次: " + row.getString("edition"));
            System.out.println("标题: " + row.getString("title"));
            System.out.println("作者: " + row.getString("author"));
            System.out.println("\n");
            System.out.println("\n");
        }
```

`CreateCassandraDatabase` 类如下所列。

```
package mongodb;

import com.datastax.driver.core.Cluster;
import com.datastax.driver.core.ResultSet;
import com.datastax.driver.core.Row;
import com.datastax.driver.core.Session;

public class CreateCassandraDatabase {
    private static Cluster cluster;
    private static Session session;

    public static void main(String[] argv) {
        cluster = Cluster.builder().addContactPoint("127.0.0.1").build();
        session = cluster.connect();
        createKeyspace();
        createTable();
        insert();
        select();
        session.close();
        cluster.close();
    }

    private static void createKeyspace() {
        session.execute("CREATE KEYSPACE IF NOT EXISTS datastax WITH replication "
                + "= {'class':'SimpleStrategy', 'replication_factor':1};");
    }

    private static void createTable() {
        session.execute("CREATE TABLE IF NOT EXISTS datastax.catalog (catalog_id text,journal text,publisher text, edition text,title text,author text,PRIMARY KEY (catalog_id, journal))");
    }

    private static void insert() {
        session.execute("INSERT INTO datastax.catalog (catalog_id, journal, publisher, edition,title,author) VALUES ('catalog1','Oracle Magazine', 'Oracle Publishing', 'November-December 2013', 'Engineering as a Service','David A.  Kelly') IF NOT EXISTS");
        session.execute("INSERT INTO datastax.catalog (catalog_id, journal, publisher, edition,title,author) VALUES ('catalog2','Oracle Magazine', 'Oracle Publishing', 'November-December 2013', 'Quintessential and Collaborative','Tom Haunert') IF NOT EXISTS");
    }

    private static void select() {
        ResultSet results = session.execute("select * from datastax.catalog");
        for (Row row : results) {
            System.out.println("目录 ID: " + row.getString("catalog_id"));
            System.out.println("\n");
            System.out.println("期刊: " + row.getString("journal"));
            System.out.println("出版商: " + row.getString("publisher"));
            System.out.println("版次: " + row.getString("edition"));
            System.out.println("标题: " + row.getString("title"));
            System.out.println("作者: " + row.getString("author"));
            System.out.println("\n");
            System.out.println("\n");
        }
    }
}
```



