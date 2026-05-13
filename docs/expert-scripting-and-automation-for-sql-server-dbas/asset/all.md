![](images/979-8-8688-1151-7_CoverFigure.jpg)

ISBN 979-8-8688-1150-0e-ISBN 979-8-8688-1151-7 [https://doi.org/10.1007/979-8-8688-1151-7](https://doi.org/10.1007/979-8-8688-1151-7) © Peter A. Carter 2016, 2025 This work is subject to copyright. All rights are solely and exclusively licensed by the Publisher, whether the whole or part of the material is concerned, specifically the rights of translation, reprinting, reuse of illustrations, recitation, broadcasting, reproduction on microfilms or in any other physical way, and transmission or information storage and retrieval, electronic adaptation, computer software, or by similar or dissimilar methodology now known or hereafter developed. The use of general descriptive names, registered names, trademarks, service marks, etc. in this publication does not imply, even in the absence of a specific statement, that such names are exempt from the relevant protective laws and regulations and therefore free for general use. The publisher, the authors and the editors are safe to assume that the advice and information in this book are believed to be true and accurate at the date of publication. Neither the publisher nor the authors or the editors give a warranty, expressed or implied, with respect to the material contained herein or for any errors or omissions that may have been made. The publisher remains neutral with regard to jurisdictional claims in published maps and institutional affiliations.

This Apress imprint is published by the registered company APress Media, LLC, part of Springer Nature.

The registered company address is: 1 New York Plaza, New York, NY 10004, U.S.A.

*This book is dedicated to Terri – the world’s greatest pig farmer.*

Acknowledgments

I want to offer a heartfelt thank-you to Adam Polka for an incredibly detailed technical review of this book. Throughout all of the years that we have collaborated, you have always demonstrated not just a one-in-a-million technical mind and the sharpest eye for detail I have ever met but also remarkable dedication and work ethic. Your review of this book was no different, and your efforts have had a big impact, as always.

About the Author About the Technical Reviewer

# 1. T-SQL Techniques for DBAs

In this chapter, we will discuss some fundamental techniques in T-SQL which will become useful in later chapters, when we discuss how to use PowerShell to automate your SQL Server enterprise. Specifically, we will discuss how to use the APPLY operator to call a function against rows within a result set. It is important to understand this technique in Chapter [8](#395785_2_En_8_Chapter.xhtml), when we begin to look at metadata-driven automation. We will then look at how XML (eXtensible Markup Language) and how the native XML data type can be harnessed by SQL Server DBAs. It is critical for DBAs to have a handle on the use of XML, due to the volume of information, such as Query Plans, that is stored in this format. Understanding basic XML in SQL Server is also key to understanding how to loop efficiently. To this end, we will explore how to efficiently iterate through multiple objects, using an XML technique. We will compare its efficiency to that of a traditional cursor. This is important, because although looping in PowerShell is always more efficient, there are times where the optimal approach to automation will involve multiple layers of looping, through the PowerShell and T-SQL layers. This will become apparent in Chapter [8](#395785_2_En_8_Chapter.xhtml), but the technique is used in multiple chapters throughout the book.

## Using the APPLY Operator

The T-SQL `APPLY` operator will be used in Chapter [8](#395785_2_En_8_Chapter.xhtml), when we explore metadata-driven automation. It allows you to call a table-valued function against every row in a result set returned by a query. There are two variations of the `APPLY` operator: `CROSS APPLY` and `OUTER APPLY`. When `CROSS APPLY` is used, the query will only return rows in which a result set has been produced by the table-valued function. When `OUTER APPLY` is used, no filter will be applied to the result set, and where no result is returned by the table-valued function, `NULL` will be returned, in each column of the table-valued function.

The `APPLY` operator can be very useful to DBAs when they are retrieving metadata from dynamic management views (DMV) and functions (DMF). For example, the query in Listing [1-1](#395785_2_En_1_Chapter.xhtml#PC1) will return a list of all sessions, detailing the session ID, the login time, the login name, and whether the process is a user or system process. It will then use OUTER APPLY to run the `sys.dm_exec_sql_text` dynamic management function against each row. This function returns a column called `text`, which is the SQL statement associated with the SQL handle in the `sql_handle` column in the `sys.dm_exec_requests` dynamic management view.

Note

The text column that is referenced in the `SELECT` list is the column returned by the `sys.dm_exec_sql_text` DMF.

```
SELECT
s.session_id
, s.login_time
, s.login_name
, s.is_user_process
, dest.[text]
FROM sys.dm_exec_sessions s
INNER JOIN sys.dm_exec_requests r
ON r.session_id = s.session_id
OUTER APPLY sys.dm_exec_sql_text(sql_handle) dest;
Listing 1-1
Using OUTER APPLY
```

You will notice that this query returns many rows in this result set with `NULL` values for the text column. This is because they are system processes, running background tasks for SQL Server, such as the Lazy Writer and the Ghost Cleanup Task.

Tip

We can be sure that they are system processes because of the `is_user_process` flag. We will not rely on the session ID being less than 50\. The assertion that all system processes have a session ID of less than 50 is widely believed, but also a fallacy, because there can potentially be more than 50 system sessions running in parallel.

If we use the `CROSS APPLY` operator for the same query, as shown in Listing [1-2](#395785_2_En_1_Chapter.xhtml#PC2), the only rows that will be returned will be rows where the result of applying the table-valued function is not `NULL`.

```
SELECT
s.session_id
, s.login_time
, s.login_name
, [text]
, s.is_user_process
FROM sys.dm_exec_sessions s
INNER JOIN sys.dm_exec_requests r
ON r.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(sql_handle) ;
Listing 1-2
Using APPLY
```

## XML for DBAs

XML (eXtensible Markup Language) is used extensively as a data format within SQL Server, for storing data that is critical for the successful DBA, such as Extended Events data, query plan data, and collector items, used by MDW (Management Data Warehouse). Therefore, it is important that DBAs have a grasp of the XML format and how to work with it, within SQL Server. Understanding how to use XML in SQL Server is also important to understanding how to loop efficiently. This will be discussed in the “Efficient Looping” section of this chapter. The following sections will provide an overview of the XML data type, how to generate XML documents, and how to query XML data.

### Understanding XML

XML is a markup language, similar to HTML, which was designed for the purpose of storing and transporting data. Like HTML, XML uses tags. Unlike HTML, however, these tags are not predefined. Instead, they are defined by the document author. An XML document has a tree structure, beginning with a root node and containing child nodes (also known as child elements). Each element can contain data but also attributes. Each attribute can contain information that describes the element. For example, imagine that you require details of sales orders to be stored in XML format. It would be sensible to assume that each sales order would be stored in a separate element within the document. But what about sales order properties, such as order date, customer ID, product IDs, quantities, and prices? These pieces of information could either be stored as child elements of the sales order element, or they could be stored as attributes of the sales order element. There are no set rules for when you should use child elements or attributes to describe properties of an element. This choice is at the discretion of the document author. The XML document in Listing [1-3](#395785_2_En_1_Chapter.xhtml#PC3) provides a sample XML document, which holds the details of sales orders for a fictional organization.

```

Listing 1-3
Sales Orders Stored in an XML Document
```

There are several things to note when looking at this XML document. First, elements (or nodes) begin with the opening tag, which is the element name, encapsulated within angle brackets. They end with the closing tag, which is the element name preceded by a backslash and enclosed in angle brackets. Any elements which fall between the opening and closing tags of another node are child elements of that node.

Attribute values are enclosed in double quotation marks and reside within the opening tag of an element. For example, `SalesOrderID` is an attribute of the `<SalesOrder>` element.

It is acceptable to have repeating elements. You can see that `<SalesOrder>` is a repeating element, as two separate sales orders are stored in this XML document. The `<SalesOrders>` element is the document’s root element and is the only element that is not allowed to be complex. This means that it cannot have attributes and cannot be repeating. Attributes can never repeat within an element. Therefore, if you require an attribute to repeat, you should use a nested element as opposed to an attribute.

The format of an XML document can be defined by an XSD (XML Schema Definition) schema. An XSD schema will define the document’s structure, including data types, if complex types (complex elements) are allowed, and how many times an element must occur (or is limited to occurring) within a document. It also defines the sequence of elements. A full description of XSD schemas can be found at `en.wikipedia.org/wiki/XML_Schema_(W3C)`.

Tip

An XML document requires a root element in order to be “well-formed.” An XML document without a root element is known as an XML fragment. This is important, because it is not possible to bind an XML fragment to an XSD schema. This means that the structure of the document, including data types, cannot be enforced.

So how does this map to DBA data? Well, take Listing [1-4](#395785_2_En_1_Chapter.xhtml#PC4) as an example. This shows part of an XML execution plan, which was produced by selecting the `database_id` column from `msdb.dbo.suspect_pages`.

```

Listing 1-4
Partial XML Execution Plan
```

While the execution plan is very simple, you can see that the author used nested elements, all of which have many attributes associated with them. Because the document conforms to a schema (typed), the root element, in this case `<ShowPlanXML>`, defines its schema association.

### Converting Result Sets to XML

T-SQL allows you to convert relational result sets into XML, by using the `FOR XML` clause in your `SELECT` statement. There are four modes that can be used with the `FOR XML` clause: `FOR XML RAW`, `FOR XML AUTO`, `FOR XML PATH`, and `FOR XML EXPLICIT`. The following sections will briefly discuss how the `FOR XML` clause works in `RAW` mode and `AUTO` mode, but we will mainly focus on `PATH` mode. `EXPLICIT` mode is beyond the scope of this book, as its functionality is very similar to `PATH` mode but is far more complex and, although helpful in some development scenarios, does not often prove useful for DBAs.

#### Using FOR XML RAW

The simplest and easiest to understand of the `FOR XML` modes is `FOR XML RAW`. This mode will transform each row in a relational result set into an element within a flat XML document. Consider the query in Listing [1-5](#395785_2_En_1_Chapter.xhtml#PC5), which extracts data from `sys.dm_exec_sessions` and `sys.dm_exec_requests`.

```
SELECT
ExecSessions.session_id
, ExecSessions.login_name
, ExecSessions.host_name
, ExecSessions.status
, ExecRequests.command
, DB_NAME(ExecRequests.database_id) AS DatabaseName
, ExecRequests.last_wait_type
FROM sys.dm_exec_sessions ExecSessions
INNER JOIN sys.dm_exec_requests ExecRequests
ON ExecSessions.session_id = ExecRequests.session_id
WHERE ExecSessions.is_user_process = 1
Listing 1-5
Extract Session Information
```

When running this query, your results will vary depending on your environment and what activity there currently is on the instance. When I ran it, it returned the results shown in Figure [1-1](#395785_2_En_1_Chapter.xhtml#Fig1).

![](images/395785_2_En_1_Chapter/395785_2_En_1_Fig1_HTML.jpg)

Table displaying database session information. Columns include session_id, login_name, host_name, status, command, DatabaseName, and task_wait_type. Two rows are shown: \\n\\n1\. session_id 52, login_name WIN-3B0U1U05D7\Administrator, host_name WIN-3B0U1U05D7, status running, command SELECT, DatabaseName master, task_wait_type SOS_WORK_DISPATCHER.\\n2\. session_id 55, login_name WIN-3B0U1U05D7\Administrator, host_name WIN-3B0U1U05D7, status running, command SELECT, DatabaseName master, task_wait_type ASYNC_NETWORK_IO.

Figure 1-1

Extract session information – results

If we were to add a `FOR XML` clause using `RAW` mode, the results would be returned in the form of an XML fragment. The amended query in Listing [1-6](#395785_2_En_1_Chapter.xhtml#PC6) will return the XML document instead of a relational result set.

```
SELECT
ExecSessions.session_id
, ExecSessions.login_name
, ExecSessions.host_name
, ExecSessions.status
, ExecRequests.command
, DB_NAME(ExecRequests.database_id) AS DatabaseName
, ExecRequests.last_wait_type
FROM sys.dm_exec_sessions ExecSessions
INNER JOIN sys.dm_exec_requests ExecRequests
ON ExecSessions.session_id = ExecRequests.session_id
WHERE ExecSessions.is_user_process = 1
FOR XML RAW
Listing 1-6
Extract Session Information Using FOR XML RAW
```

Listing [1-7](#395785_2_En_1_Chapter.xhtml#PC7) illustrates the XML document that is returned.

```

Listing 1-7
Extract Session Information FOR XML RAW Results
```

The first thing that we should note about this document is that it is an XML fragment, as opposed to a well-formed XML document, because there is no root node. The `<row>` element cannot be the root node, because it repeats. This means that we cannot validate the XML against a schema. Therefore, when using the `FOR XML` clause, you should consider using the `ROOT` keyword. This will force a root element, with a name of your choosing, to be created within the document. This is demonstrated in Listing [1-8](#395785_2_En_1_Chapter.xhtml#PC8).

```
SELECT
ExecSessions.session_id
, ExecSessions.login_name
, ExecSessions.host_name
, ExecSessions.status
, ExecRequests.command
, DB_NAME(ExecRequests.database_id) AS DatabaseName
, ExecRequests.last_wait_type
FROM sys.dm_exec_sessions ExecSessions
INNER JOIN sys.dm_exec_requests ExecRequests
ON ExecSessions.session_id = ExecRequests.session_id
WHERE ExecSessions.is_user_process = 1
FOR XML RAW, ROOT('Sessions')
Listing 1-8
Adding a Root Node
```

The resulting XML document can be seen in Listing [1-9](#395785_2_En_1_Chapter.xhtml#PC9).

```

Listing 1-9
Extract Session Information with Root Node Results
```

The other important thing to notice about the document is that it is completely flat. There is no nesting. This means that the document’s granularity is at the level of session, which, if there are multiple requests for a session, does not make a lot of sense.

It is also worth noting that all data is contained in attributes, as opposed to elements. We can alter this behavior by using the `ELEMENTS` keyword in the `FOR XML` clause. The `ELEMENTS` keyword will cause all data to be contained within child elements, as opposed to attributes. This is demonstrated in the modified query that can be found in Listing [1-10](#395785_2_En_1_Chapter.xhtml#PC10).

```
SELECT
ExecSessions.session_id
, ExecSessions.login_name
, ExecSessions.host_name
, ExecSessions.status
, ExecRequests.command
, DB_NAME(ExecRequests.database_id) AS DatabaseName
, ExecRequests.last_wait_type
FROM sys.dm_exec_sessions ExecSessions
INNER JOIN sys.dm_exec_requests ExecRequests
ON ExecSessions.session_id = ExecRequests.session_id
WHERE ExecSessions.is_user_process = 1
FOR XML RAW, ELEMENTS, ROOT('Sessions')
Listing 1-10
Extract Session Information As Elements
```

The well-formed XML document that is returned can be seen in Listing [1-11](#395785_2_En_1_Chapter.xhtml#PC11).

```

52
WIN-J38I01U06D7\Administrator
WIN-J38I01U06D7
running
SELECT
master
MEMORY_ALLOCATION_EXT

55
WIN-J38I01U06D7\Administrator
WIN-J38I01U06D7
running
SELECT
master
ASYNC_NETWORK_IO

Listing 1-11
Extract Session Information As Elements Results
```

You can see that the element-centric document still returns one element, called `<row>`, per row in the relational result set. Instead of the data being contained in attributes, however, it is stored in the form of child elements. Each child element has been given the name of the column, from which the data has been returned. The data is still flat, however. There is no hierarchy based on logic or physical table structure.

It is possible to give the `<row>` element a more meaningful name. In our example, the most meaningful name would be `<Session>`. Listing [1-12](#395785_2_En_1_Chapter.xhtml#PC12) demonstrates how we can use an optional argument in our `FOR XML` clause to generate this name for the element.

```
SELECT
ExecSessions.session_id
, ExecSessions.login_name
, ExecSessions.host_name
, ExecSessions.status
, ExecRequests.command
, DB_NAME(ExecRequests.database_id) AS DatabaseName
, ExecRequests.last_wait_type
FROM sys.dm_exec_sessions ExecSessions
INNER JOIN sys.dm_exec_requests ExecRequests
ON ExecSessions.session_id = ExecRequests.session_id
WHERE ExecSessions.is_user_process = 1
FOR XML RAW('Session'), ELEMENTS, ROOT('Sessions')
Listing 1-12
Generate a Name for the  Element
```

The partial resulting XML document can be found in Listing [1-13](#395785_2_En_1_Chapter.xhtml#PC13).

```

52
WIN-J38I01U06D7\Administrator
WIN-J38I01U06D7
running
SELECT
master
MEMORY_ALLOCATION_EXT

55
WIN-J38I01U06D7\Administrator
WIN-J38I01U06D7
running
SELECT
master
ASYNC_NETWORK_IO

Listing 1-13
Generate a Name for the  Element Results
```

#### Using FOR XML AUTO

Unlike `FOR XML RAW`, `FOR XML AUTO` is able to return nested results. It is also refreshingly simple to use, because it will automatically nest the data, based on the joins within your query. The modified query in Listing [1-14](#395785_2_En_1_Chapter.xhtml#PC14) uses `AUTO` mode to return a hierarchical XML document.

```
SELECT
ExecSessions.session_id
, ExecSessions.login_name
, ExecSessions.host_name
, ExecSessions.status
, ExecRequests.command
, DB_NAME(ExecRequests.database_id) AS DatabaseName
, ExecRequests.last_wait_type
FROM sys.dm_exec_sessions ExecSessions
INNER JOIN sys.dm_exec_requests ExecRequests
ON ExecSessions.session_id = ExecRequests.session_id
WHERE ExecSessions.is_user_process = 1
FOR XML AUTO
Listing 1-14
Using FOR XML AUTO
```

The XML fragment that is returned by this query can be seen in Listing [1-15](#395785_2_En_1_Chapter.xhtml#PC15).

```

Listing 1-15
Extract Session Information with FOR XML AUTO Results
```

You can see that when `AUTO` mode is used, the `FOR XML` clause has automatically nested the data based on the `JOIN` clauses within the query. Each element has been assigned a name based on the table alias of the table from which it was retrieved. Just as with `RAW` mode, we are able to use the `ROOT` keyword to add a root node, to make the document well-formed, or the `ELEMENTS` keyword to make the document element centric.

#### Using FOR XML PATH

Sometimes you need more control over the shape of the resultant XML document than can be provided by either `RAW` mode or `AUTO` mode. When you have a requirement to define custom heuristics, `PATH` mode can be used. `PATH` mode offers great flexibility, as it allows you to define the location of each node within the resultant XML. This is achieved by specifying how each column in the query maps to the XML, with the use of column names or aliases.

If a column alias begins with the `@` symbol, an attribute will be created. If no `@` symbol is used, the column will map to an element. Columns that will become attributes must be specified before columns that will be sibling nodes but defined as elements.

If you wish to define a node’s location in the hierarchy, you can use the `/` symbol. For example, if you require the host name to appear nested under an element called `<Session>`, you can specify that column alias as `'/Session/HostName'` for the `host_name` column.

`PATH` mode allows you to create highly customized and complex structures. For example, imagine that you have a requirement to create an XML document in the format displayed in Listing [1-16](#395785_2_En_1_Chapter.xhtml#PC16). Here, you will notice that there is a root node called `<Sessions>`, the next node in the hierarchy is `<UserSession>`. This is a repeating element, with a new occurrence for every active session on the instance. Within this element, the Session ID is displayed as an attribute, while the Login name, Host name, and Status are all child elements.

There is also a complex child element called `<Request>` which contains other child elements, containing the command type, the reason for the last wait, and Database. Database has its own child element, containing the Database name. This would be helpful if we wanted to add another tag, such as Database ID.

```

WIN-J38I01U06D7\Administrator
WIN-J38I01U06D7
running

SELECT

master

SOS_WORK_DISPATCHER

WIN-J38I01U06D7\Administrator
WIN-J38I01U06D7
running

SELECT

master

ASYNC_NETWORK_IO

Listing 1-16
Desired Results for FOR XML PATH
```

Queries to build XML documents using `FOR XML PATH` can seem a little complicated at first, but once you understand the principles, they become fairly straightforward. Let’s examine each requirement and how we can achieve the required shape.

First, we will need a root node called `<Sessions>`, and we will also have to rename the `<row>` element to `<UserSession>`. This can be achieved in the same way as when we were exploring `RAW` mode, using the optional argument for the `<row>` element and the `ROOT` keyword in the `FOR XML` clause and the root node, as shown in Listing [1-17](#395785_2_En_1_Chapter.xhtml#PC17).

```
FOR XML PATH('UserSession'), ROOT('Sessions')
Listing 1-17
Creating a Root and Naming the  Element with FOR XML PATH
```

Next, in the `SELECT` list, we simply need to specify where each of the tags should reside in our hierarchy and if they should be treated as elements or attributes. The technique for this is shown in Listing [1-18](#395785_2_En_1_Chapter.xhtml#PC18). Note that `SessionID` is prefixed with an `@` symbol to make it an attribute and that we have created the additional level of nesting for `DatabaseName`, simply by adding another level to the requested hierarchy.

```
SELECT
ExecSessions.session_id                      '@SessionID'
, ExecSessions.login_name                      'LoginName'
, ExecSessions.host_name                       'HostName'
, ExecSessions.status                          'Status'
, ExecRequests.command                         'Request/Command'
, DB_NAME(ExecRequests.database_id)            'Request/Database/Name'
, ExecRequests.last_wait_type                  'Request/LastWaitReason'
Listing 1-18
Defining the SELECT List
```

Pulling these techniques together, we have the complete query to generate our desired result set in Listing [1-19](#395785_2_En_1_Chapter.xhtml#PC19).

```
SELECT
ExecSessions.session_id '@SessionID'
, ExecSessions.login_name 'LoginName'
, ExecSessions.host_name 'HostName'
, ExecSessions.status 'Status'
, ExecRequests.command 'Request/Command'
, DB_NAME(ExecRequests.database_id) 'Request/Database/Name'
, ExecRequests.last_wait_type 'Request/LastWaitReason'
FROM sys.dm_exec_sessions ExecSessions
INNER JOIN sys.dm_exec_requests ExecRequests
ON ExecSessions.session_id = ExecRequests.session_id
WHERE ExecSessions.is_user_process = 1
FOR XML PATH('UserSession'), ROOT('Sessions')
Listing 1-19
The Final Query
```

## Extracting Values from XML Documents

When scripting, or even performance tuning, DBAs will often have to extract values from native XML. In order to demonstrate this, we will use the Query Store. The Query Store captures a history of query plans and their statistics, so that DBAs can easily compare the performance difference between different plans. As well as a graphical dashboard style tool, the Query Store also exposes plans and their metadata through a series of catalog views.

Tip

For a full discussion of Query Store, I strongly recommend the Apress title *Pro SQL Server 2022 Administration*, which can be purchased from [`https://link.springer.com/book/10.1007/978-1-4842-8864-1`](https://link.springer.com/book/10.1007/978-1-4842-8864-1).

Let’s imagine that we want to examine the most expensive query plans, which use a scan operation. This information can be retrieved by joining the sys.query_store_runtime_statistics catalog view with the sys.query_store plan catalog view. In order to retrieve the results that we require, however, we will have to interrogate the XML execution plan. This means that we will have to introduce the concepts of XQuery.

XQuery is a language for querying XML, in the same way that SQL is a language for querying relational data. The language is built on XPath and consists of XPath expressions to address a specific area of the document and FLOWR (pronounced flower) statements. FLOWR is an abbreviation of “For, Let, Where, Order by, and Return.”

For statements allow you to iterate through a sequence of nodes. Let statements bind a sequence to a variable. Where statements filter nodes based on a Boolean expression. Order by statements order nodes before they are returned. Return statements specify what should be returned.

The majority of DBAs (even those concerned with automation) rarely need to concern themselves with FLOWR statements. Therefore, a complete discussion of the topic is beyond the scope of this book. Instead, I will focus on the core methods that are of use for automation and scripting in SQL Server. These methods are query, value, exist, and nodes.

Note

The examples in the following sections use the showplan schema. Full details of this XSD can be found at [`http://schemas.microsoft.com/sqlserver/2004/07/showplan`](http://schemas.microsoft.com/sqlserver/2004/07/showplan).

### query Method

The XQuery query method can be used against columns or variables with the XML data type. It extracts part of the document. The extracted parts of the document will form an untyped XML document, even if the original document conforms to a schema (typed XML).

Calling the method uses the syntax `XMLDoc.query('/Path/To/Node')`. If the document is bound to an XSD, you can declare the namespace, prior to specifying the node path, with the syntax `XMLDoc.query(declare namespace alias="//domain.com/schemalocation"; //alias:node')`.

For example, the query in Listing [1-20](#395785_2_En_1_Chapter.xhtml#PC20) will return every column reference from each query plan within the Query Store. Within the query method, we first declare the namespace for Microsoft’s showplan schema. We can then easily specify the node that we wish to extract, as opposed to passing the node’s full location within the hierarchy.

```
SELECT
CAST(query_plan AS XML).query('declare namespace
qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
//qplan:ColumnReference') AS SQLStatement
FROM sys.query_store_plan ;
Listing 1-20
Using the query CROSS Method
```

### value and nodes Methods

The value method will extract a singleton value from an XML document, mapped to a SQL Server data type. The value method can be called against columns or variables of the XML data type in the format `XMLDoc.value('/path/to/node/@node[1]', 'int')`. Alternatively, you can also specify the schema that the XML document is bound to, in the format `XMLDoc.value('declare namespace alias=`[`http://domain.com/schemalocation;`](http://domain.com/schemalocation%3B) `(//alias:node[1]', 'int')`.

Tip

Because the value method can only return a singleton value, it is mandatory to specify an index value in square brackets, even if the node only occurs once within the document.

Because the value method can only return an atomic (singleton) value, it can be used in conjunction with the nodes method to shred nodes into relational data. The nodes method returns a rowset that contains logical copies of the instances of a node. The value method can then be cross-applied to the results of the nodes method, in order to return multiple values from the same XML document. The query in Listing [1-21](#395785_2_En_1_Chapter.xhtml#PC21) will return a list of tables and columns that are accessed within each plan in the Query Store.

The subquery within Listing [1-21](#395785_2_En_1_Chapter.xhtml#PC21) simply converts the query_plan column to native XML, so that the nodes method can be called against it. The outer query cross-applies the nodes method to the native XML version of the query plan and then runs the value method against the logical representation of the node instances that have been returned by the nodes method. Note that because the value method is being called against the results returned by nodes, rather than the original XML document, there is no need to specify a path for the attributes. This is because the results returned by nodes begin at the level of the <ColumnReference> element, and the @Table and @Column nodes are attributes of this element. Therefore, as far as the value method is concerned, the attributes are at the top level of the document.

Tip

When combining nodes with value, the schema is declared in the nodes method only. This is because the value method is run against the logical representation of instances, as opposed to the original XML document.

```
SELECT
plan_id
,nodes.query_plan.value('@Table[1]', 'nvarchar(128)') AS TableRef
,nodes.query_plan.value('@Column[1]', 'nvarchar(128)') AS ColumnRef
FROM
(
SELECT plan_id, CAST(query_plan AS XML) query_plan_xml
FROM sys.query_store_plan
) base
CROSS APPLY query_plan_xml.nodes('declare namespace
qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
//qplan:ColumnReference') as nodes(query_plan) ;
Listing 1-21
Using the value CROSS and nodes CROSS Methods
```

### exist Method

The exist method will run an XQuery expression against an XML document. It will then return a bit (Boolean) value to denote the success of the expression. 0 is returned when the result of the XPath expression is empty; 1 is returned if the result is not empty; and `NULL` is returned if the result is `NULL`.

The exist method is usually used in a `WHERE` clause. For example, you may have noticed that the queries in Listings [1-20](#395785_2_En_1_Chapter.xhtml#PC20) and [1-21](#395785_2_En_1_Chapter.xhtml#PC21) return many results for many internal queries run by SQL Server and also for DBA activity, such as accessing backup metadata. The query in Listing [1-22](#395785_2_En_1_Chapter.xhtml#PC22) extends the query in Listing [1-21](#395785_2_En_1_Chapter.xhtml#PC21) to filter out any results where the `@Schema` attribute of the `<ColumnReference>` element is equal to either `[sys]` or `[dbo]`. This will result if only plans that access user tables are considered.

```
SELECT
plan_id
,nodes.query_plan.value('@Table[1]', 'nvarchar(128)') AS index1
,nodes.query_plan.value('@Column[1]', 'nvarchar(128)') AS index2
FROM
(
SELECT plan_id, CAST(query_plan AS XML) query_plan_xml
FROM sys.query_store_plan
) base
CROSS APPLY query_plan_xml.nodes('declare namespace
qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
//qplan:ColumnReference') as nodes(query_plan)
WHERE query_plan_xml.exist('declare namespace
qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
//qplan:Object[@Schema="[sys]"]') = 0
AND query_plan_xml.exist('declare namespace
qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
//qplan:Object[@Schema="[dbo]"]') = 0 ;
Listing 1-22
Using the exist CROSS Method
```

### Pulling the Methods Together

In order to meet our original requirement, to examine the most expensive query plans, which use a scan operation, we will have to combine the value, query, and exist methods in a single query. The query in Listing [1-23](#395785_2_En_1_Chapter.xhtml#PC23) uses the exist method in the `WHERE` clause to examine the `<RelOp>` element of each plan and filter out any operators that do not involve a scan operation. The `WHERE` clause will also use the exist method to filter any operations that are not against user tables.

The `SELECT` list uses the value method to extract the `@Schema` and `@Table` attributes, to help us to identify the tables where we may be missing indexes. It also uses the query method to extract the entire `<Object>` element, giving us full details of the table, in XML format.

As well as using XQuery methods, the query also returns the whole plan in native XML format and plan statistics from the `sys.query_store_runtime_stats` catalog view. The query uses `TOP` and `ORDER BY` to filter out all but the five most expensive plans, based on the average execution time of the plan, multiplied by the number of times the plan has been executed.

```
SELECT TOP 5
CAST(p.query_plan AS XML)
, CAST(p.query_plan AS XML).value('declare namespace
qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
(//qplan:Object/@Schema)[1]','nvarchar(128)')
+ '.'
+ CAST(p.query_plan AS XML).value('declare namespace
qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
(//qplan:Object/@Table)[1]','nvarchar(128)') AS [Table]
, CAST(p.query_plan AS XML).query('declare namespace
qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
//qplan:Object') AS TableDetails
, rs.count_executions
, rs.avg_duration
, rs.avg_physical_io_reads
, rs.avg_cpu_time
FROM sys.query_store_runtime_stats rs
INNER JOIN sys.query_store_plan p
ON p.plan_id = rs.plan_id
WHERE CAST(p.query_plan AS XML).exist('declare namespace
qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
//qplan:RelOp[@LogicalOp="Index Scan"
or @LogicalOp="Clustered Index Scan"
or @LogicalOp="Table Scan"]') = 1
AND CAST(p.query_plan AS XML).exist('declare namespace
qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
//qplan:Object[@Schema="[sys]"]') = 0
AND CAST(p.query_plan AS XML).exist('declare namespace
qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
//qplan:Object[@Schema="[dbo]"]') = 0
ORDER BY rs.count_executions * rs.avg_duration DESC ;
Listing 1-23
Finding the Most Expensive Plans with Scan Operations
```

## Efficient Looping

One of the most common hosting standards enforced upon developers by DBAs is to disallow the use of cursors. At the same time, the majority of DBAs will use a cursor whenever they need to iterate through a set of objects. The argument is normally that their script only runs once per day, and writing a cursor is the quickest way to achieve their means. That argument, of course, has some merit, but I always feel that it is better to lead by example. There have been several times when I have had “the cursor debate” with development teams and won the argument by virtue of the fact that I have implemented my own scripts without the use of cursors, and developers should be able to do the same.

Of course, the main focus of this book is on PowerShell, and looping in PowerShell is far more efficient than looping in T-SQL. There are optimal techniques, however, which are demonstrated throughout this book, especially in Chapter [8](#395785_2_En_8_Chapter.xhtml), where having multiple layers of looping at the PowerShell and T-SQL levels is the logical approach.

Imagine that we need a script to kill all active sessions. This is a useful script for any DBA to have in their armory, as there will always be occasions when you have to disconnect all users quickly. The script in Listing [1-24](#395785_2_En_1_Chapter.xhtml#PC24) achieves this by using a cursor, but it will be slow and resource-intensive, especially in a highly transactional environment with many active sessions.

```
DECLARE @session_id INT ;
DECLARE C_Sessions CURSOR
FOR
SELECT Session_id
FROM sys.dm_exec_sessions
WHERE session_id  @@SPID
AND is_user_process = 1 ;
OPEN C_Sessions ;
FETCH NEXT FROM C_Sessions
INTO @Session_id ;
DECLARE @SQL NVARCHAR(MAX) ;
WHILE @@FETCH_STATUS = 0
BEGIN
SET @SQL = 'Kill ' + CAST(@Session_id AS NVARCHAR(4)) ;
EXEC(@SQL) ;
FETCH NEXT FROM C_Sessions
INTO @Session_id ;
END
CLOSE C_Sessions ;
DEALLOCATE C_Sessions ;
Listing 1-24
Kill All Sessions with a Cursor
```

A much more efficient way of achieving the same results (and adhering to your own hosting standards!) is to use a trick with `FOR XML`. The query in Listing [1-25](#395785_2_En_1_Chapter.xhtml#PC25) uses a subquery to populate a variable. The subquery returns the string `'Kill '` concatenated to the `Session_id` for each row that is returned in the result set. This is then converted into XML format, using `FOR XML PATH`. The `data()` method is used to strip out the XML tags and characters, leaving only the text. This text is then passed back up to the outer query, where it is implicitly converted into `NVARCHAR` by virtue of the data type of the variable. The variable can then be executed using dynamic SQL.

The benefit of this approach is that the query is only executed once, as opposed to once for every active session. This means less disk activity. There is also less memory used in this approach, compared to other techniques, such as `@SQL = @SQL +` approach.

```
DECLARE @SQL NVARCHAR(MAX) ;
SELECT DISTINCT @SQL =
(
SELECT 'Kill ' + CAST(session_id AS NVARCHAR(4)) AS [data()]
FROM sys.dm_exec_sessions
WHERE session_id  @@SPID
AND is_user_process = 1
FOR XML PATH('')
) ;
EXEC(@SQL) ;
Listing 1-25
Kill All Sessions Efficiently
```

## Summary

The DBA who is serious about reducing workload by scripting and automating tasks has to be familiar with some T-SQL techniques. While this book focuses on PowerShell, sometimes writing code at the T-SQL layer is the most optimal solution, meaning that your code will span both the PowerShell and T-SQL layers.

Using the `APPLY` operator allows you to apply a function to each row in a result set. When the `CROSS APPLY` operator is used, only rows that do not cause the function to return a `NULL` value will be returned. When the `OUTER APPLY` operator is used, all rows in the result set will be returned. DBAs can use this technique for applying Dynamic Management Functions to Dynamic Management Views.

XML is an ever increasingly important technology for a DBA to understand and be able to work with. The two main concepts relating to XML within SQL Server are converting XML to relational values and converting a result set to an XML document. The `FOR XML` clause of a `SELECT` statement can be used to produce XML documents from result sets. XQuery, which is based on XPath, can be used to extract values from XML documents.

Many DBAs still use cursors to iterate through objects, but using the XQuery `data()` method within a subquery provides a far more optimal solution. It also promotes good practice from developers, by showing that you lead by example.

# 2. PowerShell Fundamentals

PowerShell is a powerful scripting language, written by Microsoft. It uses the .NET Common Language Runtime (CLR) which provides massive functionality out of the box and is also entirely extensible. It is designed for automating and managing systems and is often used for the automation or maintenance of cloud environments, such as Azure and AWS. It is also the language of choice for managing platforms such as Microsoft Exchange and, of course, SQL Server. PowerShell Core Edition (v7) is supported not only on Windows but also on Mac OS and Linux. There is full support for many variants of Linux, including Red Hat, Ubuntu, and Debian, to name but a few.

In this chapter, we will refresh ourselves on the fundamentals of the PowerShell language. We will start with a short accelerator, which will help us be productive. We will then discuss some fundamental language elements such as comments, data types, variables, and flow control statements. Finally, we will look at an overview of PowerShell modules, which provide extensibility.

## Getting Started

To get started, we should understand that PowerShell can be used in two distinct ways. The first of these is a command-line shell. This provides administrators with the ability to run ad hoc commands (known as cmdlets) or simple scripts, using the PowerShell terminal. In a Windows environment, this terminal can be used instead of cmd shell and provides a much richer experience. Commands return .NET objects as opposed to plain text, and the pipeline (which we will discuss later in this chapter) can be used to chain commands together. The terminal itself offers command-line history and tab completion.

The second way in which PowerShell can be used is as a scripting language. When used as a scripting language, PowerShell provides a rich, extensible platform for creating automation, which allows administrators and DevOps Engineers to harness the power of .NET and a fully object-oriented language with classes, inheritance, methods, and properties. This means that the language is fully extensible through classes, functions, and modules.

### Execution Policy

If we plan to use PowerShell for automation on Windows, then we need to consider the execution policy. This is part of the PowerShell security strategy and determines if users can run scripts, load configuration files, and load profiles. Table [2-1](#395785_2_En_2_Chapter.xhtml#Tab1) details the execution policies that can be set on Windows.

Table 2-1

Execution Policies

| Policy | Description |
| --- | --- |
| **Undefined** | `The default policy in a Windows environment. Checks to see if a policy is defined in Group Policy. If undefined at all scopes, then it provides the equivalent of Restricted for Windows Clients or the equivalent of RemoteSigned for Windows Server` |
| **AllSigned** | `All scripts and configuration files must be signed by a trusted publisher. Even if the script was written on the local machine` |
| **Bypass** | `Nothing is blocked. There are no warnings or prompts` |
| **Default** | `Sets Windows desktops to Restricted and sets Windows Servers to RemoteSigned` |
| **RemoteSigned** | `If a script or configuration file has been downloaded, then it must be signed or explicitly unblocked. Locally created scripts can be executed without being signed` |
| **Restricted** | `Scripts cannot be run and configuration files cannot be loaded` |
| **Unrestricted** | `All scripts can be executed and all configuration files can be loaded. If the script has been downloaded from the Internet and is not signed, then the user will be prompted before execution. This is the default, for non-Windows systems` |

By default, when specifying an execution policy, it applies to the scope of `LocalMachine`. The `-Scope` parameter can be used, however, to specify `CurrentUser` or Process. If the scope of `Process` is used, then the policy only applies to the current PowerShell session.

On non-Windows operating systems, from PowerShell 6 onward, the default policy is `Unrestricted` and this cannot be overridden. On Windows, however, due consideration should be given to the appropriate execution policy, usually in collaboration with your organization’s Cyber Security team.

If you are planning on following examples in this book, and downloading the code samples from the code repository, I would recommend setting the execution policy to be `Unrestricted`, and this can be achieved using the command in Listing [2-1](#395785_2_En_2_Chapter.xhtml#PC1).

```
Set-ExecutionPolicy Unrestricted
Listing 2-1
Set Execution Policy
```

Tip

To set the execution policy, you must run the terminal as administrator.

### Standards

While there are exceptions, most commands in PowerShell follow the naming convention of Verb-Noun (noun in singular). The first part of the command’s name (“Verb”) describes the action that you will be performing. These are known as approved verbs and are used for the sake of consistency so that commands can be intuitively understood. Some of the most common verbs are listed below, but an exhaustive list can be found at learn.microsoft.com/en-us/powershell/scripting/developer/cmdlet/approved-verbs-for-windows-powershell-commands?view=powershell-7.4:

*   Add

*   Clear

*   Connect

*   Convert

*   Copy

*   Disable

*   Disconnect

*   Enable

*   Format

*   Get

*   Install

*   Invoke

*   New

*   Read

*   Remove

*   Rename

*   Reset

*   Revoke

*   Save

*   Select

*   Set

*   Show

*   Start

*   Stop

*   Test

*   Update

*   Wait

*   Write

This prefix is followed by a hyphen (`-`) and ends with a description of the resource that you will be performing the action upon (“Noun”). For example, `Get-ChildItem` can be used to list the contents of the current PowerShell location. This location could be a folder in the file system, or it could be a table inside a SQL Server instance. Navigating a SQL Server instance is discussed later in this chapter. The logical and consistent naming convention, combined with IntelliSense, makes learning and finding commands a relatively easy process in development environments such as Visual Studio Code.

### Aliases

To make learning PowerShell even easier, many commands have aliases, which map to DOS or Linux commands. This means that if you have any experience using Windows Command Prompt (which is based on DOS), or Bash, you can continue to use many commands that you are already familiar with.

As an example, let’s consider `Get-ChildItem`. As mentioned earlier, `Get-ChildItem` can be used to list the contents of a folder in the file system. Many new PowerShell users unfamiliar with this command will be familiar with `dir` from DOS or `ls` from LINUX. Therefore, PowerShell provides both of these aliases. For example, all of the commands in Listing [2-2](#395785_2_En_2_Chapter.xhtml#PC2) are functionally equivalent.

```
dir *.txt #Using the DOS alias
ls *.txt #Using the LINUX alias
gci *.txt #Using gci alias
Get-ChildItem *.txt #Using the PowerShell command
Listing 2-2
Using Aliases
```

Tip

Aliases can enhance productivity when you are operating interactively with the PowerShell terminal, but they are considered bad practice within scripts, because it makes the code more opaque and harder to read.

## Language Fundamentals

The following sections will discuss comments, data types, variables, piping and filtering, and flow control statements. PowerShell is a very large language, but the constructs contained in this section will give you an accelerator and provide you with the fundamentals you need for the rest of this book.

### Comments

PowerShell supports both inline comments and block comments. An inline comment is denoted by the `#` symbol. Anything to the right of this symbol is treated as a comment and is not executed. For example, the command in Listing [2-3](#395785_2_En_2_Chapter.xhtml#PC3) is functionally equivalent to the command in Listing [2-1](#395785_2_En_2_Chapter.xhtml#PC1).

```
Set-ExecutionPolicy -ExecutionPolicy Unrestricted #Set execution policy to Unrestricted
Listing 2-3
Inline Comments
```

Block comments begin with `<#` and end with `#>`. Anything between these character sequences is regarded as a comment and is not executed. The script in Listing [2-4](#395785_2_En_2_Chapter.xhtml#PC4) demonstrates the use of block comments.

```
Set-ExecutionPolicy  -ExecutionPolicy Unrestricted
Listing 2-4
Block Comments
```

### Data Types

PowerShell uses the data types provided by the .NET CLR. Every data type in .NET is derived from the `Object` data type. Therefore, the `Object` data type is the root of the type hierarchy, as well as being a data type in its own right.

The base types listed in Table [2-2](#395785_2_En_2_Chapter.xhtml#Tab2) are derived directly from the `Object` type. From these base types, literally hundreds of data types are derived. Some of these derived types are complex, such as `XML`, `DateTimeOffset` (which stores a date and time with an offset against UTC), and `Array`.

Table 2-2

Base Data Types

| Base Data Type | Description |
| --- | --- |
| `Byte` | 8-bit unsigned integer |
| `SByte` | Non-CLS-compliant 8-bit signed integer |
| `Int16` | 16-bit signed integer |
| `Int32` | 32-bit signed integer |
| `Int64` | 64-bit signed integer |
| `UInt16` | 16-bit unsigned integer |
| `UInt32` | Non-CLS-compliant 32-bit unsigned integer |
| `UInt64` | Non-CLS-compliant 64-bit unsigned integer |
| `Single` | 32-bit floating-point number |
| `Double` | 64-bit floating-point number |
| `Boolean` | Boolean/Bit value |
| `Char` | Unicode (16-bit) character |
| `Decimal` | Decimal (128-bit) value |
| `IntPtr` | A signed integer that has a 32-bit value on a 32-bit platform and a 64-bit value on a 64-bit platform |
| `UIntPtr` | Non-CLS-compliant, unsigned integer that has a 32-bit value on a 32-bit platform and a 64-bit value on a 64-bit platform |
| `String` | A string of Unicode characters |

### Variables

As with all languages, variables contain values that are unknown until runtime or that change during execution. They are critical to writing flexible, powerful, and maintainable code. They are used for purposes such as storing user inputs and looping. Each variable maps to a data type, and because variables are objects derived from a .NET class, various methods can be called against them. For example, the `ToString()` method can be called against an `Int32` to output the variable value as a string, as opposed to an integer.

A variable is always prefixed with a `$`, and when declaring a variable, you specify the data type in square brackets, immediately before the `$`, before finally declaring the name of the variable.

Tip

Always give variables meaningful names. If you give variables names such as `$i` or `$a`, they become as meaningless as naming them `$foo` and $`bar`. Giving variables meaningful names makes your code easier to read and maintain.

The script in Listing [2-5](#395785_2_En_2_Chapter.xhtml#PC5) first creates two variables. One is an integer called `$HelloWorldInt`, and the other is a string called `$HelloWorldText` and assigns values to each of the variables. The second part of the script clears the console of messages, before it outputs the results.

```
#Declare the variables and assign them values
[int]$HelloWorldInt = 123
[string]$HelloWorldText = "Hello World"
#Print the Results to the Console
Clear-Host
Write-Host "HelloWorldInt: " $HelloWorldInt
Write-Host "HelloWorldText: " $HelloWorldText
Listing 2-5
Declaring and Using Variables
```

We can also declare variables without specifying a data type. When you do this, PowerShell will use the data type that it considers to be most appropriate. Be warned, however, that PowerShell is not psychic! It can only guess what data type the variable should be. For example, consider the script in Listing [2-6](#395785_2_En_2_Chapter.xhtml#PC6). Here, we declare six variables without specifying a data type. We then use the `GetType()` method to extract the data type for each.

```
#Declare Variables In-line
$StringVariable1 = "Hello World"
$StringVariable2 = 123
$StringVariable3 = "123"
$Int32Variable = 123
$SingleVariable = 3.3
$XMLVariable = 'MyValue'
#Extract Data Types
Clear-Host
"StringVariable1: " + $StringVariable1.GetType()
"StringVariable2: " + $StringVariable2.GetType()
"StringVariable3: " + $StringVariable3.GetType()
"Int32Variable: " + $Int32Variable.GetType()
"SingleVariable: " + $SingleVariable.GetType()
"XMLVariable: " + $XMLVariable.GetType()
Listing 2-6
Declaring Variables Without Data Types
```

As you can see from the results, which are listed below, the data types are not as you might expect:

```
StringVariable1: string
StringVariable2: int
StringVariable3: string
Int32Variable: int
SingleVariable: double
XMLVariable: string
```

You can see that `$StringVariable`, `$StringVariable3`, and `$Int32Variable` have been assigned to the data types we wanted. We have not been so lucky with the other variables, however. `$StringVariable2` has been created with an `int` data type, because we did not enclose the assigned value within quotation marks. This is why `$StringVariable3` did not suffer the same issue. `$SingleVariable` has been created with a data type of `double`. This is not the end of the world, but it does mean that we are using more memory resource than we require. `$XMLVariable` has been created with the `string` data type. This means that we will be unable to use XML-specific methods and cmdlets, such as the `GetElementsByTagName` method and the `Select-Xml` cmdlet.

To summarize, it is important to remember the potential complications of not specifying a data type, and if the data type is important to your script, it should always be specified. An additional factor to consider is code maintenance. If it is not easy for you (or a team member) to reference the data type of a variable, it may slow you down when you come to troubleshoot, revise, or enhance the script.

### Piping and Filtering

If you have a T-SQL background, rather than a programming background, the idea of piping may be new to you. In PowerShell, piping is the process of passing the results of one command into another command. Piping is a key concept of PowerShell and one of the things that makes it so powerful.

A simple example of piping can be found in Listing [2-7](#395785_2_En_2_Chapter.xhtml#PC8). The first line in this script pulls a list of running processes from the operating system. It then passes the results of this command – as one or more objects, not just flat text – into a `Where-Object` cmdlet. `Where-Object` iterates over all received objects, applying the supplied filter script (in curly brackets). The filter script uses a special automatic PowerShell variable `$_`, which denotes the currently processed object, to compare each object’s `ProcessName` property against the string “*p*w*sh*” (the asterisks are used as wildcards). Non-matching objects are removed from the results.

```
# Return a list of running processes and then filter by the name of the process
# The chosen filter string will match executables of both Powershell 7 (pwsh) and 5 (powershell)
$process = Get-Process | Where-Object { $_.ProcessName -like "*p*w*sh*" }
# Print the filtered list of processes to the console
$process
Listing 2-7
Basic Piping
```

Without the ability to pipe, code would be much longer and harder to manage. In this scenario, we would need a separate statement, which looped through every process that was read into the `$process` variable. For a very simple example like this, that might not sound like the end of the world, but when we have very large, complex scripts, the extra overhead that would cause would become a burden.

### Controlling Flow

If your background is purely in SQL Server, then you will likely be averse to the thought of looping, as it is a very bad practice to loop in SQL Server, due to the set-based nature of the language. If you have any scripting or programming experience, you will already know how useful looping can be for iterating through objects, performing an action (or set of actions) a specified number of times, or performing an action (or set of actions) repeatedly, until a condition is met. PowerShell provides looping capability through a `for` statement, a `foreach` statement, and a `while` statement.

Caution

Of course, the statement asserting how useful looping can be does not hold true for T-SQL, as set-based operations are always far more efficient than looping. Please refer to Chapter [1](#395785_2_En_1_Chapter.xhtml) for more details.

Before we dive into how we can use loops, however, we need to ensure that we are familiar with the operators supported by PowerShell, as we will need these to control our loops. If you are familiar with T-SQL, then you are likely familiar with the concept of operators and know comparison operators such as `=` (equal to), `>` (greater than), and `<=` (less than or equal to).

Comparison operators available in PowerShell are listed in Table [2-3](#395785_2_En_2_Chapter.xhtml#Tab3), but an important differential to T-SQL is that in PowerShell `=` is only used for assignment, never for comparison. The equality comparative operator in PowerShell is `-eq`. It is important to remember this to avoid some nasty bugs in our code, which are difficult to find.

Table 2-3

Comparison Operators

| Operator | Description |
| --- | --- |
| `-contains` | Any item in the property’s value is equal to the value specified. This is useful if you must ensure that a specific value exists within an array |
| `-eq` | The property’s value is equal to the specified value |
| `-ge` | The property’s value is greater than or equal to the specified value |
| `-gt` | The property’s value is greater than the specified value |
| `-in` | The property’s value is equal to any of the elements of the array of values specified. The array of values can be specified as a comma-separated list |
| `-is` | The property’s value is of the specified data type. The data type specified must be enclosed in square brackets |
| `-isnot` | The data type of the property’s value is not the same as the specified data type. The data type specified must be enclosed in square brackets |
| `-le` | The property’s value is less than or equal to the specified value |
| `-lt` | The property’s value is less than the specified value |
| `-like` | The property’s value contains the specified value, following a wildcard match |
| `-match` | The property’s value matches the regex pattern provided |
| `-ne` | The property’s value is not equal to the value specified |
| `-notcontains` | No item in the property’s value is equal to the value specified. This is useful if you have to ensure that a specific value does not exist within an array |
| `-notin` | The property’s value is not equal to any of the elements of the array of values specified. The array of values can be specified as a comma-separated list |
| `-notlike` | The property’s value does not contain the specified value, following a wildcard match |
| `-notmatch` | The property’s value does not match the regex pattern provided |
| `-ccontains` | As per `-Contains`, but case-sensitive |
| `-ceq` | As per `-EQ`, but case-sensitive |
| `-cge` | As per `-GE`, but case-sensitive |
| `-cgt` | As per `-GT`, but case-sensitive |
| `-cin` | As per `-In`, but case-sensitive |
| `-cle` | As per `-LE`, but case-sensitive |
| `-clt` | As per `-LT`, but case-sensitive |
| `-clike` | As per `-Like`, but case-sensitive |
| `-cmatch` | As per `-Match`, but case-sensitive |
| `-cne` | As per `-NE`, but case-sensitive |
| `-cnotcontains` | As per `-NotContains`, but case-sensitive |
| `-cnotin` | As per `-NotIn`, but case-sensitive |
| `-cnotlike` | As per `-NotLike`, but case-sensitive |
| `-cnotmatch` | As per `-NotMatch`, but case-sensitive |

A `for` loop can be used for either repeating a set of actions a specified number of times or iterating through a subset of an array or collection. A `foreach` loop is useful when you have to iterate through every item in an array or collection, and a `while` loop is used to repeat a set of actions, until a condition is met.

Listing [2-8](#395785_2_En_2_Chapter.xhtml#PC9) demonstrates how a `for` loop could be used to implement error handling. The script will attempt to read the contents of a file three times, at 30-second intervals. The behavior of the for loop is defined within parentheses. There are three elements within the parentheses. The first initializes the variable that will control the loop. In our example, we create a variable named $i with an initial value of 1\. The second definition specifies the condition that has to be met before our loop terminates. We specify that we will continue to loop while $i is less than or equal to 3\. The final definition controls the increment of the looping variable. $i++ is shorthand for $i = $i + 1\. Therefore, in our example, the value of `$i` will increase by one on each iteration of the loop.

The code to be repeated in each iteration of the loop is enclosed within braces. Here, we use a `try...catch` block to attempt to read the file. If the operation succeeds, we will write the success to the console and then exit the loop via the break command. If the operation fails, execution moves to the `catch` block. Here, we write the failure to the console and then pause the execution of the script for 30 seconds.

Tip

Although failing to read the file will always generate an error, it will not usually cause the script to terminate. As we need the script to terminate for the flow to move to the `catch` block, we use the `-ErrorAction` parameter to force the script to terminate.

```
for ($i=1; $i -le 3; $i++) {
try {
Get-Content c:\ExpertScripting.txt -ErrorAction Stop
"Attempt " + $i + " Succeeded"
break
} catch {
"Attempt " + $i + " Failed"
Start-Sleep -s 30
}
}
Listing 2-8
Using a for Loop
```

The script in Listing [2-9](#395785_2_En_2_Chapter.xhtml#PC10) demonstrates how a `while` loop can be used to refine the code in Listing [2-8](#395785_2_En_2_Chapter.xhtml#PC9), so that the script will continue to loop (potentially infinitely) until the read of the file exists. The `while` loop continues as long as the condition defined within parentheses evaluates to the Boolean value of $true. In our example, since the condition is just $true, the `while` loop will continue infinitely. Once the contents of the file have been read successfully, however, the `break` command will exit the loop. This technique is particularly useful when you are scripting a middleware component, which requires a file to be written before continuing.

Note

If you are following along with the demonstrations, create a file named `ExpertScripting.txt` in the root of `c:\` while the script is running.

```
$i = 0
while ($true) {
try {
$i++
Get-Content c:\ExpertScripting.txt -ErrorAction Stop
"Attempt " + $i + " Succeeded"
break
} catch {
"Attempt " + $i + " Failed"
Start-Sleep -s 30
}
}
Listing 2-9
Using a while Loop
```

The script in Listing [2-10](#395785_2_En_2_Chapter.xhtml#PC11) demonstrates how a `foreach` loop can be used to ensure that all SQL Server services are running and start them if they are not.

```
# Populate a new variable with the details all SQL Server services
$services = Get-Service | Where-Object { $_.Name -like "*SQL*" -and $_.Status -eq "Stopped" }
# Start each service
foreach ($name in $services) {
Start-Service $name
}
Listing 2-10
Using a foreach Loop
```

The final control of flow concept that we will discuss is `IF`...`ELSE`. An `IF`...`ELSE` block allows you to branch your code, based on a condition. You can also nest `IF`...`ELSE` blocks to implement complex logic (although you should consider code maintenance when doing so). It is only possible to have one `IF` block. To implement multiple chained `IF` conditions, you can add multiple `ELSEIF` blocks. The `ELSE` block always comes last (if you choose to use one) and is a catchall for any occurrences in which none of your `IF` and `ELSEIF` conditions is met. The conditions of `IF` and `ELSEIF` blocks are enclosed within parentheses. The code block to execute is enclosed within braces. The script in Listing [2-11](#395785_2_En_2_Chapter.xhtml#PC12) demonstrates how to implement an `IF`...`ELSE` block to check the status of a service.

```
$Service = "SQLBrowser"
$ServiceDetails = Get-Service | Where-Object {$_.Name -eq $Service}
if ($ServiceDetails.Status -eq "Running") {
$Service + " is  working"
} elseif ($ServiceDetails.Status -eq "Stopped") {
$Service + " is not working. Please check the Event Log"
} elseif (-not $serviceDetails) {
$Service + " is not installed"
} else {
$Service + " is changing state"
}
Listing 2-11
Using IF, ELSEIF, and ELSE
```

## Modules

To make PowerShell fully extensible and able to manage virtually any application, it has been written using a modular approach, with modules having to be imported to allow users access to the cmdlets that they contain. Some of the core modules are imported automatically when a command is executed. For example, using the `Get-ExecutionPolicy` command causes the `Microsoft.PowerShell.Security` module to be imported.

A massive number of modules are available, providing the ability to use PowerShell to manage almost any platform or application. Some of these modules are provided by organizations. For example, AWS provides the AWSPowerShell module for managing its public cloud platform, and Microsoft provides the Az module for managing Azure.

There is also a vast number of community modules available, however, for example, the `posh-git` module which provides Git status summary and tab completion for Git command and the `powershell-yaml` module which provides YAML serialization and deserialization in PowerShell.

When modules are published on PowerShell Gallery, they can be downloaded and installed using the `Install-Module` command, as demonstrated in Listing [2-12](#395785_2_En_2_Chapter.xhtml#PC13), which installs the `Indented.IniFile` module, which was written by the excellent Indented Automation (which I strongly recommend you check out for all things PowerShell) and provides functionality for managing INI file content in PowerShell.

```
Install-Module Indented.IniFile
Listing 2-12
Install a PowerShell Module
```

### SqlServer Module

The SqlServer module is supplied as part of the SQL Server Management Tools download, but can also be installed using the `Install-Module` command. The module provides over 100 commands for interacting with SQL Server from a PowerShell session. This module will be discussed in depth in Chapter [4](#395785_2_En_4_Chapter.xhtml), but in this section, we will briefly discuss how the module can be used to navigate a SQL Server instance. The ability to navigate a SQL Server instance or even invoke PowerShell from inside SQL Server Management Studio can be a good break-through introduction for DBAs who have not used PowerShell before.

As well as navigating a folder structure with commands such as `Get-ChildItem` and `Set-Location`, PowerShell can also be used to navigate the SQL Server object hierarchy of an instance. You can connect PowerShell to the SQL Server Database Engine provider by using `Set-Location` to navigate to `SQLSERVER:\SQL`. The information returned by `Get-ChildItem` is dependent on the current location in the object hierarchy. Table [2-4](#395785_2_En_2_Chapter.xhtml#Tab4) details what information is returned from each level of the hierarchy.

Table 2-4

Details Returned at Each Hierarchy Level

| Location | Information Returned |
| --- | --- |
| `SQLSERVER:\SQL` | The name of the local machine |
| `SQLSERVER:\SQL\ComputerName` | The names of the database engine instances installed on the local machine |
| `SQLSERVER:\SQL\ComputerName\InstanceName` | Instance-level object types |
| `Lower levels` | Object types or objects contained within the current location |

Once you have navigated to an appropriate level of the hierarchy, you are able to use PowerShell to perform basic operations against objects at that level. For example, the script in Listing [2-13](#395785_2_En_2_Chapter.xhtml#PC14) will navigate to the `tables` namespace within the `AdventureWorks2022` database and rename the `dbo.DatabaseLog` table to `dbo.DatabaseLogPS`. The `dir` commands will display the original name and new name of the table.

Tip

The script assumes that you have a default instance of SQL Server installed (as opposed to a named instance) and that you have the AdventureWorks2022 database, which can be downloaded from learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure.

```
Set-Location SQLSERVER:\SQL\localhost\Default\Databases\AdventureWorks2022\Tables
dir | where { $_.name -like "*DatabaseLog*" }
Rename-Item -LiteralPath dbo.DatabaseLog -NewName DatabaseLogPS
dir | Where-Object { $_.name -like "*DatabaseLog*" }
Listing 2-13
Navigating Object Hierarchy and Renaming a Table
```

It is also possible to start PowerShell from within SQL Server Management Studio (SSMS). To do this, select Start PowerShell from the context menu of an object folder, within Object Explorer. This will cause the PowerShell CLI to be invoked, with the initial location matching the object folder that you used to invoke the CLI.

## Summary

PowerShell provides two methods of operation. The Command Line Interface can be used to immediately execute commands, in a similar manner to the Windows Command Prompt. This is useful for administrators who need to perform ad hoc actions or perform quick checks. The second method of operation is scripting. Administrators or DevOps Engineers can build complex scripts that involve many commands or are scheduled to run automatically.

The PowerShell language is implemented through modules to make it extensible. Each module contains cmdlets, which provide the functionality. Commonly used cmdlets provide aliases, which can shorten the length of the command you have to type and help new PowerShell users with DOS or Linux experience to learn the language. Aliases should be reserved for ad hoc commands, however. When used in scripts, they make the code opaque and hard to manage.

Each variable in PowerShell is an object, with a data type associated with it. Because the objects map to .NET classes, methods such as `ToString()` are available. PowerShell also provides a rich framework for controlling the execution flow of your script. This includes piping, which allows you to pass the results of one command directly into the next.

The `SqlServer` module provides support for SQL Server. Once the module has been loaded, it exposes a set of SQL Server–based cmdlets, which allow you to manage SQL Server from the command shell or through automated scripts. It also allows you to navigate the object hierarchy of the SQL Server instance and perform basic tasks against objects.

# 3. PowerShell DSC

Configuration management is a powerful methodology that can be used to configure your servers to your organization’s standards, but also keep them configured in a consistent manner, by enforcing the policies on a scheduled basis as they move through their life cycle.

In this chapter, we will discuss the concepts behind the configuration management approach, before drilling into the specific implementation of Desired State Configuration (DSC), which is Microsoft’s configuration management tool. Finally, we will explore how we can implement DSC to configure Windows Server in a SQL Server estate.

## Understanding Configuration Management

There are many ways that a server can be built and configured. We could manually configure a server, we could use a gold disk or a build script, but each of these approaches has limitations.

The limitations of configuring a server manually are fairly apparent. It is a mundane, time-consuming job, which often leads to human error, which results in an inconsistent estate. Even if the server is built to a good standard, its configuration will likely drift over time.

If we use a gold disk approach, which means building a single image, using sysprep and then deploying it multiple times, then we can address the issues described with a manual build, but we open up new issues. Specifically:

*   We have no flexibility. Every server has to be 100% identical. This means that if we need the slightest variation, we must either
    *   Have a huge selection of gold images, each of which must be maintained separately

    *   Use the gold image, but then make manual updates, which can lead to an inconsistent estate

*   While the estate may be consistent at the point each server is built, as time goes on, manual changes can lead to drift in configuration and, ultimately, an inconsistent estate.

If we use a scripted approach, then we will usually start with a very basic gold image and then run build scripts against the newly created server, which will perform the configuration. This approach removes the first issue with gold images. Specifically, it allows us to have a consistent build, which uses logic in the script to configure the server differently for different use cases. It does not resolve the second issue however. There is still the likelihood of configuration drift, as the server moves through its life cycle.

Configuration management addresses all of the build issues described above. It is a scripted approach, but instead of the script only being run once, at build time, the script is run on a schedule. Every time the script runs, it checks what the server’s configuration should be, and if anything has drifted, then it puts the configuration back into its desired state.

The logic used in configuration management is illustrated in Figure [3-1](#395785_2_En_3_Chapter.xhtml#Fig1).

![](images/395785_2_En_3_Chapter/395785_2_En_3_Fig1_HTML.jpg)

Flowchart illustrating a configuration evaluation process. It begins with "Start" leading to "Evaluate configuration Item 1." A decision diamond asks, "Is configuration item 1 in desired state?" with "Yes" leading to "Move to next configuration item" and "No" leading to "Configure as-per desired state," which loops back to the evaluation. The process continues to "Evaluate configuration Item 2."

Figure 3-1

Configuration management logic

This process can be used to configure any configuration within a server. It can install software, configure security setting, and even install and configure SQL Server (which will be discussed in Chapters [6](#395785_2_En_6_Chapter.xhtml) and [9](#395785_2_En_9_Chapter.xhtml)).

There are many configuration management tools available, including Ansible, Chef, and Puppet. This book, however, will focus on Microsoft DSC or Desired State Configuration, which is Microsoft’s implementation of configuration management.

## Understanding DSC

There are two broad ways that DSC can be implemented. The first is at the machine level. This approach is the simplest implementation and is best suited to small environments or testing. DSC is included in Windows Management Framework 5.1\. Therefore, we can get started with DSC, without the need to install any client software. This chapter will focus on this approach to help you understand the implementation in the most straightforward way.

The second way to implement it is by using a centralized management resource, which can manage multiple machines from a single location. This is most suitable for large enterprises as it significantly reduces management overheads when you have many servers.

The central management component can be a Pull Server, which is a server that runs the DSC-Service Windows Server feature, or it can be the guest configuration Azure Policy, which is compatible with Azure VMs and Arc-enabled servers outside of Azure.

Note

Windows Pull Server is still a supported feature, but it is no longer being developed. This means that Microsoft DSC is usually most appropriate for smaller organizations or large organizations which use either Azure or Arc-enabled servers. The DSC code has actually been branched, so that DSC 1.1 is still supported but not maintained. This can be used with PowerShell versions up to PowerShell 5.1 and can be used on non-Azure environments. DSC 2.0 is supported in PowerShell 7 and above, but is designed for use with Azure. At the time of writing, DSC 3.0 is in preview.

The two main concepts to understand in DSC are configurations and resources. Configurations provide a simple mechanism to describe the desired configuration of an item, and resources provide the mechanism of implementation of the settings described in the configurations (taking into account the current state of the item). Every DSC resource has three methods: Get, Test, and Set. The Get method is used to retrieve the current state of a resource. The Test method is used to determine if the server (known as a node in DSC) is currently configured in its desired target state. The Set method is used to configure the node to be compliant with its target state.

DSC provides a number of built-in resources, which can be used to configure items such as services, users, groups, registry keys, and Windows features. DSC is also fully extensible. We can create our own, custom resources, which can be used to configure any item that can be configured with PowerShell.

Resources can be written using either a class-based approach or a Microsoft Operations Framework (MOF) based approach. This chapter will focus on using out-of-the-box resources, but a guide to creating custom DSC resources can be found at the following location: learn.microsoft.com/en-us/powershell/dsc/resources/authoringresourceclass?view=dsc-1.1.

Configurations are the manifest of resources that should be applied. They are implemented through a special type of function, which lists the resources that should be applied to the node. Each resource is passed a minimum of two parameters. One of these parameters is the name of the target instance. The second is the action that should be applied to it. For example, if you had a resource that ensured that SQL Browser service was running, you would use the service resource and pass `Name = "SQLBrowser"` and `State = "Running"`. On the other hand, if you wanted to ensure that the file "C:\Logs\CustomSqlLog.log" file exists, you would use the file resource and pass the parameters `Name = "C:\Logs\CustomSqlLog.log"` and `Ensure = "Present"`. Local Configuration Manager is the engine that reads configurations and uses resources to ensure that the desired configuration is applied.

## Using DSC to Configure Windows Server

Now that we understand the key concepts, let’s put those concepts into practice to perform some simple Windows Server configuration, using DSC 1.1\. There are many configurations that you may wish to perform at the operating system level, such as configuring the operating system to be in line with the CIS benchmark or ensuring that the SQL Server Database Engine service is running. To keep things simple, in this example, however, we will look to ensure the following target configurations:

*   Ensure a folder exists that will store backups of certificates used by SQL Server

*   Ensure that Windows is configured to optimize background tasks

*   Ensure that the SQL Server Database Engine service is running

The code snippet in Listing [3-1](#395785_2_En_3_Chapter.xhtml#PC1) demonstrates how to define a resource that will ensure that a folder called CertificateBackups exists. The declaration starts by using the `File` keyword, which denotes the resource being declared is the DSC File resource. This is followed by the name of the resource. Providing a name is essential, as it uniquely identifies resources of the same type. Inside the curly brackets, we pass `Ensure = "Present"` which states that we want to ensure the folder exists. If we wanted to ensure that it doesn’t exist, we could pass `Ensure = "Absent"`. We pass `Type = "Directory"`. If we wanted to manage a file, instead of a folder, we could pass `Type = "File"`. We could then use `Content = "FooBar"` to insert the text into the file. Finally, we pass `DestinationPath = "C:\CertificateBackups"`, which defines which folder we want the resource to manage.

```
File CreateCertificateBackupsFolder {
Ensure           = "Present"
Type             = "Directory"
DestinationPath  = "C:\CertificateBackups"
}
Listing 3-1
Define a File Resource
```

The code snippet in Listing [3-2](#395785_2_En_3_Chapter.xhtml#PC2) demonstrates how to use a Registry DSC resource. This resource type is useful for configuring many Windows-level items, but here we will use it to ensure that Windows is configured to be optimized for background tasks. We don’t want the database engine having to wait, because a user clicking around Management Studio is being prioritized. This time, we use the Registry keyword to denote that this is a Registry DSC resource. Again, we give it a sensible name. Inside the curly brackets, we once again use `Ensure = “Present”` to indicate that the relevant string value should exist inside the registry key. We use `Key = "HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl"` to specify the path to the registry key that we want to manage. We use `ValueName = "Win32PrioritySeparation"` to denote the value within the key that should be present and `ValueData = 24` to define our desired value for the key.

```
Registry OptimizeForBackgroundServices {
Ensure      = "Present"
Key         = "HHKEY_LOCAL_MACHINE:\SYSTEM\CurrentControlSet\Control\PriorityControl"
ValueName   = "Win32PrioritySeparation"
ValueType   = 'Dword'
ValueData   = 24
}
Listing 3-2
Define a Registry Resource
```

Our final resource ensures that the database engine service of the Default SQL Server instance is running.

Tip

We will be discussing how to build SQL Server in Chapter [6](#395785_2_En_6_Chapter.xhtml), so if you want to follow along with the examples, but don’t have SQL Server installed yet, you can change the SQL Server Database Engine service to be any other service, which does exist.

Listing [3-3](#395785_2_En_3_Chapter.xhtml#PC3) demonstrates how to define this resource, by using the Service DSC resource. The declaration starts with the Service keyword to denote that we are using the Service DSC resource, and, again, we provide a sensible name. Inside the curly brackets, you will notice that, this time, we do not pass Ensure. When not present, Ensure defaults to Present. We pass `Name = "MSSQLSERVER"` to define the service we wish to manage. We pass `StartupType = "Automatic"` to specify that the service should always be configured to start automatically and `State = "Running"` to specify that the service should always be running. If the service is stopped, DSC will automatically start it for us.

```
Service SQLServerService
{
Name        = "MSSQLSERVER"
StartupType = "Automatic"
State       = "Running"
}
Listing 3-3
Define a Service Resource
```

Now that all of our resources are defined, we need to bring them together into a configuration file. The configuration file that we will use is shown in Listing [3-4](#395785_2_En_3_Chapter.xhtml#PC4). The first line uses the Configuration keyword, followed by a name for the configuration. The required DSC resources are imported from the PSDesiredStateConfiguration PowerShell module.

Next, we define the host or hosts that the configuration should run on. In our topology, we will be scheduling the configuration to run locally from Task Scheduler. Therefore, we define it as localhost. Inside the node definition, we have the definition of all of the resources which lay out our desired configuration.

Finally, the last line of the file calls the configuration that we have just defined. Think of the definition of the configuration a bit like defining a function. It’s not enough to just define what a function will do, it needs to actually be called off in order to be executed.

We need to save this file to our local computer. I have saved mine as C:\Scripts\WindowsConfig.ps1.

```
Configuration WindowsConfig {
Import-DscResource -ModuleName 'PSDesiredStateConfiguration'
Node 'localhost' {
File CreateCertificateBackupsFolder {
Ensure = "Present"
Type = "Directory"
DestinationPath = "C:\CertificateBackups"
}
Registry OptimizeForBackgroundServices {
Ensure      = "Present"
Key         = "HKEY_LOCAL_MACHINE:\SYSTEM\CurrentControlSet\Control\PriorityControl"
ValueName   = "Win32PrioritySeparation"
ValueType   = 'Dword'
ValueData   = 24
}
Service SQLServerService
{
Name        = "MSSQLSERVER"
StartupType = "Automatic"
State       = "Running"
}
}
}
WindowsConfig
Listing 3-4
Define the Configuration
```

Now that we have saved our configuration, we will run the configuration to generate the .mof file, as demonstrated in Listing [3-5](#395785_2_En_3_Chapter.xhtml#PC5).

Tip

If you have used a different path or filename for the configuration file, make sure you change the path in the following scripts.

```
C:\Scripts\WindowsConfig.ps1
Listing 3-5
Run the Configuration
```

The output of this command is shown below:

```
Mode          LastWriteTime               Length Name
------        --------------------        ------ -----------
-a----        10/11/2024     10:22        3796 localhost.mof
```

If you look in the file system, you will notice that when we ran the configuration it created a folder with the same name as the configuration and a .mof file inside it. This file is named after the node and contains configuration information for the system. The folder is shown in Figure [3-2](#395785_2_En_3_Chapter.xhtml#Fig2).

![](images/395785_2_En_3_Chapter/395785_2_En_3_Fig2_HTML.jpg)

File Explorer screenshot showing a directory path: This PC > Local Disk (C:) > Scripts > WindowsConfig. The directory contains a single file named "localhost.mof" with a modification date of 10/11/2024 at 10:22\. The file type is MOF File, and its size is 4 KB.

Figure 3-2

Configuration folder

So, let’s apply the configuration and see what happens. We can apply the configuration using the `Start-DscConfiguration` cmdlet, as shown in Listing [3-6](#395785_2_En_3_Chapter.xhtml#PC7). We use the `-Path` parameter to pass the folder that was created by executing the configuration, and we use the `-Verbose` switch, so we can see the details of what is happening.

Tip

Before running the script, I recommend stopping the SQL Server Database Engine service, ensuring that you do not have a folder called C:\CertificateBackups and making sure Windows is optimized for Programs. That will allow you to test your configuration has applied the desired state as expected.

```
Start-DscConfiguration -Path 'C:\Scripts\WindowsConfig' -Verbose -Wait
Listing 3-6
Apply the Configuration
```

The verbose output, when run on my test rig, is shown below:

```
VERBOSE: Perform operation 'Invoke CimMethod' with following parameters, ''methodName' = SendConfigurationApply,'className' = MSFT_DSCLocalConfigurationManager,'namespaceName' = root/Microsoft/Windows/DesiredStateConfiguration'.
VERBOSE: An LCM method call arrived from computer SQL2022-STANDAL with user sid S-1-5-21-2722909466-2614467794-2401863046-500.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Set      ]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Resource ]  [[File]CreateCertificateBackupsFolder]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Test     ]  [[File]CreateCertificateBackupsFolder]
VERBOSE: [SQL2022-STANDAL]:                            [[File]CreateCertificateBackupsFolder] The system cannot find the file specified.
VERBOSE: [SQL2022-STANDAL]:                            [[File]CreateCertificateBackupsFolder] The related file/directory is: C:\CertificateBackups.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Test     ]  [[File]CreateCertificateBackupsFolder]  in 0.0150 seconds.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Set      ]  [[File]CreateCertificateBackupsFolder]
VERBOSE: [SQL2022-STANDAL]:                            [[File]CreateCertificateBackupsFolder] The system cannot find the file specified.
VERBOSE: [SQL2022-STANDAL]:                            [[File]CreateCertificateBackupsFolder] The related file/directory is: C:\CertificateBackups.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Set      ]  [[File]CreateCertificateBackupsFolder]  in 0.0000 seconds.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Resource ]  [[File]CreateCertificateBackupsFolder]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Resource ]  [[Registry]OptimizeForBackgroundServices]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Test     ]  [[Registry]OptimizeForBackgroundServices]
VERBOSE: [SQL2022-STANDAL]:                            [[Registry]OptimizeForBackgroundServices] Registry key value 'HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl\Win32PrioritySeparation' of type 'String' does not contain data '24'
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Test     ]  [[Registry]OptimizeForBackgroundServices]  in 0.1100 seconds.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Set      ]  [[Registry]OptimizeForBackgroundServices]
VERBOSE: [SQL2022-STANDAL]:                            [[Registry]OptimizeForBackgroundServices] (SET) Set registry key value 'HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl\Win32PrioritySeparation' to '24' of type 'String'
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Set      ]  [[Registry]OptimizeForBackgroundServices]  in 0.0470 seconds.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Resource ]  [[Registry]OptimizeForBackgroundServices]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Resource ]  [[Service]SQLServerService]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Test     ]  [[Service]SQLServerService]
VERBOSE: [SQL2022-STANDAL]:                            [[Service]SQLServerService] Perform operation 'Query CimInstances' with following parameters, ''queryExpression' = SELECT * FROM Win32_Service WHERE Name='MSSQLSERVER','queryDialect' = WQL,'namespaceName
' = root\cimv2'.
VERBOSE: [SQL2022-STANDAL]:                            [[Service]SQLServerService] Operation 'Query CimInstances' complete.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Test     ]  [[Service]SQLServerService]  in 1.2030 seconds.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Set      ]  [[Service]SQLServerService]
VERBOSE: [SQL2022-STANDAL]:                            [[Service]SQLServerService] Service 'MSSQLSERVER' already exists. Write properties such as Status, DisplayName, Description, Dependencies will be ignored for existing services.
VERBOSE: [SQL2022-STANDAL]:                            [[Service]SQLServerService] Perform operation 'Query CimInstances' with following parameters, ''queryExpression' = SELECT * FROM Win32_Service WHERE Name='MSSQLSERVER','queryDialect' = WQL,'namespaceName
' = root\cimv2'.
VERBOSE: [SQL2022-STANDAL]:       [[Service]SQLServerService] Operation 'Query CimInstances' complete.
VERBOSE: [SQL2022-STANDAL]:       [[Service]SQLServerService] Service 'MSSQLSERVER' started.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Set      ]  [[Service]SQLServerService]  in 2.0040 seconds.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Resource ]  [[Service]SQLServerService]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Set      ]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Set      ]    in  3.7230 seconds.
VERBOSE: Operation 'Invoke CimMethod' complete.
VERBOSE: Time taken for configuration job to complete is 3.767 seconds
```

There is a lot of information there to digest, but the sections highlighted in bold text show that a set operation has happened for all three resources. So, let’s run the command in Listing [3-6](#395785_2_En_3_Chapter.xhtml#PC7) again and see what happens the second time. The output, as per my test rig, is shown below:

```
VERBOSE: Perform operation 'Invoke CimMethod' with following parameters, ''methodName' = SendConfigurationApply,'className' = MSFT_DSCLocalConfigurationManager,'namespaceName' = root/Microsoft/Windows/DesiredStateConfiguration'.
VERBOSE: An LCM method call arrived from computer SQL2022-STANDAL with user sid S-1-5-21-2722909466-2614467794-2401863046-500.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Set      ]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Resource ]  [[File]CreateCertificateBackupsFolder]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Test     ]  [[File]CreateCertificateBackupsFolder]
VERBOSE: [SQL2022-STANDAL]:                            [[File]CreateCertificateBackupsFolder] The destination object was found and no action is required.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Test     ]  [[File]CreateCertificateBackupsFolder]  in 0.0150 seconds.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Skip   Set      ]  [[File]CreateCertificateBackupsFolder]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Resource ]  [[File]CreateCertificateBackupsFolder]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Resource ]  [[Registry]OptimizeForBackgroundServices]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Test     ]  [[Registry]OptimizeForBackgroundServices]
VERBOSE: [SQL2022-STANDAL]:                            [[Registry]OptimizeForBackgroundServices] Found registry key value 'HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl\Win32PrioritySeparation' with type 'String' and data '24'
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Test     ]  [[Registry]OptimizeForBackgroundServices]  in 0.1090 seconds.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Skip   Set      ]  [[Registry]OptimizeForBackgroundServices]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Resource ]  [[Registry]OptimizeForBackgroundServices]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Resource ]  [[Service]SQLServerService]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Start  Test     ]  [[Service]SQLServerService]
VERBOSE: [SQL2022-STANDAL]:                            [[Service]SQLServerService] Perform operation 'Query CimInstances' with following parameters, ''queryExpression' = SELECT * FROM Win32_Service WHERE Name='MSSQLSERVER','queryDialect' = WQL,'namespaceName
' = root\cimv2'.
VERBOSE: [SQL2022-STANDAL]:                            [[Service]SQLServerService] Operation 'Query CimInstances' complete.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Test     ]  [[Service]SQLServerService]  in 1.1170 seconds.
VERBOSE: [SQL2022-STANDAL]: LCM:  [ Skip   Set      ]  [[Service]SQLServerService]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Resource ]  [[Service]SQLServerService]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Set      ]
VERBOSE: [SQL2022-STANDAL]: LCM:  [ End    Set      ]    in  1.4600 seconds.
VERBOSE: Operation 'Invoke CimMethod' complete.
VERBOSE: Time taken for configuration job to complete is 1.507 seconds
```

This time, the output is much shorter. You will notice from the entries highlighted in bold text that all three set operations have been skipped. This is because they are already configured in the desired state and no action needs to be performed. This illustrates the beauty of the configuration management approach.

The last step is to schedule this configuration to run. I normally recommend running the configuration every 30 minutes. That is, if any drift arises on a server, it will be corrected in no more than half an hour (plus the time it takes the configuration to execute).

Listing [3-7](#395785_2_En_3_Chapter.xhtml#PC10) demonstrates how to configure a task in Task Scheduler to apply the configuration every 30 minutes. The script sets multiple variables, which build up the parameters that will be required to execute the `Register-ScheduledTask` cmdlet at the end of the script.

Note

You will notice that we are using Windows PowerShell, rather than PowerShell 7\. This is because, as previously mentioned, usage of DSC without Azure is only supported by DSC 1.1 which shipped with Windows PowerShell 5.1\. PowerShell 7 shipped with DSC 2.0 which contains major changes, and as of PowerShell 7.2, the PSDesiredStateConfiguration module is no longer delivered with PowerShell but is available as a separate download.

```
$command = 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe'
$arguments = '-NoProfile -Command "Start-DscConfiguration -Path C:\Scripts\WindowsConfig" -Wait'
$actions = (New-ScheduledTaskAction -Execute $command -Argument $arguments )
$trigger = New-ScheduledTaskTrigger -Once -At 00:00 -RepetitionInterval (New-TimeSpan -Minutes 30)
$settings = New-ScheduledTaskSettingsSet
$principal = New-ScheduledTaskPrincipal -UserId 'NT AUTHORITY\SYSTEM' -RunLevel Highest -LogonType ServiceAccount
$task = New-ScheduledTask -Action $actions -Trigger $trigger -Settings $settings -Principal $principal
Register-ScheduledTask 'ApplyWindowsConfig' -InputObject $task
Listing 3-7
Create a Scheduled Task
```

## Summary

Configuration management is a very powerful technique for building servers and avoiding configuration drift as they move through their life cycle. DSC in the modern era is most suitable for use in environments that harness Azure or Azure Arc. It can still be used in non-Azure environments, however, by using the DSC 1.1 release.

DSC can be used to configure all aspects of Windows. In this chapter, we have examined how to use DSC to configure registry keys, folders, and to ensure that services are running. In Chapter [6](#395785_2_En_6_Chapter.xhtml), we will explore how to use DSC to install SQL Server, and in Chapter [9](#395785_2_En_9_Chapter.xhtml), we will explore how to configure SQL Server itself, using DSC.

DSC is fully extensible, and we can write our own DSC resources that will cover almost any possible configuration. The beauty of DSC is that it only performs configurations that need to be performed to avoid drift. Each resource will test the state of the resource and only perform the set action if the test evaluation fails.

# 4. Working with the sqlserver Module

The sqlserver is a PowerShell module, written and maintained by Microsoft, which allows DBAs to interact with and manage SQL Server instances from a PowerShell terminal. It can be manually downloaded from PowerShell Gallery ([`www.powershellgallery.com/packages/Sqlserver`](http://www.powershellgallery.com/packages/Sqlserver/21.1.18228)) or can be installed using the command in Listing [4-1](#395785_2_En_4_Chapter.xhtml#PC2).

```
Install-Module SqlServer
Listing 4-1
Install sqlserver Module
```

In this chapter, we will discuss how to work with the module, exploring some of the most useful cmdlets and their use cases, both for ad hoc administration and for automation. We will start by looking at the `Invoke-SqlCmd` cmdlet, before looking at other cmdlets that can be used for management and scripting.

## Working with Invoke-SqlCmd

Arguably, the most commonly used cmdlet in the sqlserver arsenal is `Invoke-SqlCmd`. This cmdlet mirrors the functionality of the SQLCMD command-line tool, allowing DBAs to run queries against a SQL Server instance, from a PowerShell terminal.

### Invoke-SqlCmd Basics

The most basic call of `Invoke-SqlCmd` involves passing two parameters: `ServerInstance`, which defines which SQL Server instance the query should be run against, and `Query` which supplies the query that should be executed. Listing [4-2](#395785_2_En_4_Chapter.xhtml#PC3) demonstrates using this basic call to return a list of database names from the default instance of the local server. Possible formats for the `ServerInstance` parameter include `ServerName` and `ServerName\InstanceName`. If you wish to run the query against a default instance of the local server, then you can use `localhost`, `.` or omit the `ServerInstance` parameter all together.

Tip

The following example assumes Windows authentication. If SQL authentication should be used, then the user credential will also need to be passed.

-TrustServerCertificate will be required if you are using v22 or above of the sqlServer module and do not have encryption configured.

```
Invoke-SqlCmd -ServerInstance localhost -Query "SELECT name FROM sys.databases" -TrustServerCertificate
Listing 4-2
Basic Call of Invoke-SqlCmd
```

### Using Invoke-SqlCmd for Scripting

So we now have a list of databases displayed on screen, which is great and certainly useful for ad hoc DBA activity. If we are planning to use `Invoke-SqlCmd` within a script, then there is a strong likelihood that we will want to pass the results of the query into a variable.

The script in Listing [4-3](#395785_2_En_4_Chapter.xhtml#PC4) demonstrates how to do this, but it also introduces a technique known as splatting. This technique builds up a hash table, which contains key/value pairs, which relate to the parameter names and values that we plan to pass to the command. When we execute the command, we can then simply pass the hash table, and PowerShell will expand the parameter names and values inside it. Functionally, it is equivalent to passing each parameter into the command in the usual way. The benefit of splatting is that it makes the commands more concise and easier to read. It is also easier to find and manage parameter values at a later date.

```
$Params = @{
ServerInstance = "localhost"
Query          = "SELECT name FROM sys.databases"
}
$Databases = Invoke-Sqlcmd @Params
Listing 4-3
Pass Invoke-SqlCmd Results to a Variable
```

It is important to note that the results of the `Invoke-SqlCmd` command are an object or an array of objects. By default, the output is as DataRows, although the `OutputAs` parameter can be used to change the output to a `DataSet` or `DataTables` format. This means that you need to work with the properties of the rich object, rather than the object itself. For example, consider the script in Listing [4-4](#395785_2_En_4_Chapter.xhtml#PC5), which passes the `$Databases` variable into a `foreach` loop and writes each object to screen.

Tip

foreach loops are introduced in Chapter [2](#395785_2_En_2_Chapter.xhtml).

```
$Params = @{
ServerInstance = "localhost"
Query          = "SELECT name FROM sys.databases"
}
$Databases = Invoke-Sqlcmd @Params
foreach ($Database in $Databases) {
Write-Host $Database
}
Listing 4-4
Passing Rich Objects to a foreach Loop
```

The results of running this script are displayed below:

```
System.Data.DataRow
System.Data.DataRow
System.Data.DataRow
System.Data.DataRow
System.Data.DataRow
```

Instead, we should work with the properties of the rich object. In this specific case, the column we are returning is called name; hence, we should work with the name property, as demonstrated in Listing [4-5](#395785_2_En_4_Chapter.xhtml#PC7).

```
$Params = @{
ServerInstance = "localhost"
Query          = "SELECT name FROM sys.databases"
}
$Databases = Invoke-Sqlcmd @Params
foreach ($Database in $Databases) {
Write-Host $Database.name
}
Listing 4-5
Working with Object Properties
```

This script will return the more helpful results, shown below:

```
master
tempdb
model
msdb
WideWorldImporters
```

This point brings me nicely on to discussing how we could use the data returned by `Invoke-SqlCmd`. In the examples above, we have been returning a list of databases that exist on our instance. Imagine that you have a script that will dynamically rebuild indexes, based on a fragmentation threshold, such as the script in Listing [4-6](#395785_2_En_4_Chapter.xhtml#PC9).

Tip

Creating a dynamic index rebuild script and other metadata-driven automation will be discussed in more detail in Chapter [7](#395785_2_En_7_Chapter.xhtml).

```
DECLARE @SQL NVARCHAR(MAX)
SET @SQL =
(
SELECT 'ALTER INDEX '
+ QUOTENAME(i.name)
+ ' ON ' + s.name
+ '.'
+ QUOTENAME(OBJECT_NAME(i.object_id))
+ ' REBUILD ; '
FROM sys.dm_db_index_physical_stats(DB_ID(),NULL,NULL,NULL,'DETAILED') ps
INNER JOIN sys.indexes i
ON ps.object_id = i.object_id
AND ps.index_id = i.index_id
INNER JOIN sys.objects o
ON ps.object_id = o.object_id
INNER JOIN sys.schemas s
ON o.schema_id = s.schema_id
WHERE index_level = 0
AND avg_fragmentation_in_percent > 25
FOR XML PATH('')
) ;
EXEC(@SQL) ;
Listing 4-6
Dynamic Index Rebuilds
```

Tip

Using FOR XML PATH to efficiently loop in T-SQL is explained further in Chapter [1](#395785_2_En_1_Chapter.xhtml).

A variable containing a list of databases will allow us to schedule this script to run against every database on our instance, without having to maintain a static list of databases. The technique for this is demonstrated in Listing [4-7](#395785_2_En_4_Chapter.xhtml#PC10).

```
$Params = @{
ServerInstance = "localhost"
Query          = "SELECT name FROM sys.databases"
}
$Databases = Invoke-Sqlcmd @Params
foreach ($Database in $Databases) {
$Params = @{
ServerInstance = "localhost"
Database       = $Database.name
InputFile      = "c:\scripts\IndexRebuild.sql"
}
Invoke-Sqlcmd @Params
}
Listing 4-7
Running a Script Against All Databases
```

There are a couple of key points to consider about this script. Firstly, note that we are using the `Database` parameter of the `Invoke-SqlCmd` cmdlet. We are passing in the `name` property of the `Database` object in each iteration of the `foreach` loop.

Secondly, while it is technically possible to pass the index rebuild script directly into the `Query` parameter of `Invoke-SqlCmd`, the script is rather large for this approach and will make our script hard to maintain. Therefore, we have saved the script in a file called `c:\scripts\IndexRebuild.sql` and are passing this file into the cmdlet, with the use of the `InputFile` parameter.

### Working with Parameters

Of course, while this script is already becoming rather useful, it would be even more useful if it could accept variables. In this case, the most obvious use case for a variable would be the fragmentation threshold, but we may also want to enhance the script to also rebuild indexes that are suffering from external fragmentation. Therefore, Listing [4-8](#395785_2_En_4_Chapter.xhtml#PC11) updates the `IndexRebuild.sql` script to include `avg_page_space_used_in_percent` and expect variables to be passed to the `WHERE` clause. Note that the format of the variables is the SQLCMD format of `$(Variable)`.

```
DECLARE @SQL NVARCHAR(MAX)
SET @SQL =
(
SELECT 'ALTER INDEX '
+ QUOTENAME(i.name)
+ ' ON ' + s.name
+ '.'
+ QUOTENAME(OBJECT_NAME(i.object_id))
+ ' REBUILD ; '
FROM sys.dm_db_index_physical_stats(DB_ID(),NULL,NULL,NULL,'DETAILED') ps
INNER JOIN sys.indexes i
ON ps.object_id = i.object_id
AND ps.index_id = i.index_id
INNER JOIN sys.objects o
ON ps.object_id = o.object_id
INNER JOIN sys.schemas s
ON o.schema_id = s.schema_id
WHERE index_level = 0
AND (avg_fragmentation_in_percent > $(Fragmentation)
OR avg_page_space_used_in_percent < $(PageSpace))
FOR XML PATH('')
) ;
EXEC(@SQL) ;
Listing 4-8
Update Script to Expect Variable
```

`Invoke-SqlCmd` allows SQLCMD variables to be passed, using the `VARIABLE='Value'` format, via the `Variable` parameter. This is demonstrated in Listing [4-9](#395785_2_En_4_Chapter.xhtml#PC12).

```
$Variables = "Fragmentation=25", "PageSpace=70"
$Params = @{
ServerInstance = "localhost"
Query          = "SELECT name FROM sys.databases"
}
$Databases = Invoke-Sqlcmd @Params
foreach ($Database in $Databases) {
$Params = @{
ServerInstance = "localhost"
Database       = $Database.name
InputFile      = "c:\scripts\IndexRebuild.sql"
Variable       = $Variables
}
Invoke-Sqlcmd @Params
}
Listing 4-9
Passing a Variable to Invoke-SqlCmd
```

In this script, we start by creating a string array, which contains each of our variables, which we want to pass. We then pass this array to the `Variable` parameter.

Tip

If we are running maintenance tasks such as index rebuilds, then it can be helpful to configure timeouts. `Invoke-SqlCmd` supports both `ConnectionTimeout` and `QueryTimeout` parameters, which allow you to specify timeout values in seconds.

### Outputting Errors

Invoke-SqlCmd has many parameters, which are less commonly used than those we have discussed so far, but no less useful. For example, imagine we had a bug in the query that we are using to return a list of database names. We could use the `OutputSqlErrors` parameter to return the error. This is demonstrated in Listing [4-10](#395785_2_En_4_Chapter.xhtml#PC13). We would expect this query to fail because the column in the `WHERE` clause should be `database_id`, not `db_id`. The first invocation of Invoke-SqlCmd will succeed, but the $Databases parameter will be empty. The second invocation of Invoke-SqlCmd will tell us the error.

```
$Params = @{
ServerInstance   = "localhost"
Query            = "SELECT name FROM sys.databases WHERE db_id = 1"
OutputSqlErrors = $false
}
$Databases = Invoke-Sqlcmd @Params
$Params = @{
ServerInstance   = "localhost"
Query            = "SELECT name FROM sys.databases WHERE db_id = 1"
OutputSqlErrors = $true
}
$Databases = Invoke-Sqlcmd @Params
Listing 4-10
Output Error Messages
```

The error returned by the second invocation is shown below:

```
Invoke-Sqlcmd : Invalid column name 'db_id'.
At line:15 char:14
+ $Databases = Invoke-Sqlcmd @Params
+              ~~~~~~~~~~~~~~~~~~~~~
+ CategoryInfo          : InvalidOperation: (:) [Invoke-Sqlcmd], SqlPowerShellSqlExecutionException
+ FullyQualifiedErrorId : SqlError,Microsoft.SqlServer.Management.PowerShell.GetScriptCommand
```

### Writing Results to a File

There is not a parameter which will allow you to send query results directly to a file; however, the same ends can be achieved by piping the results of Invoke-SqlCmd to the Out-File cmdlet. This is demonstrated in Listing [4-11](#395785_2_En_4_Chapter.xhtml#PC15), which writes the list of databases to a file called c:\scripts\DatabasesOut.txt.

```
$Params = @{
ServerInstance  = "localhost"
Query           = "SELECT name FROM sys.databases"
}
Invoke-Sqlcmd @Params | Out-File -FilePath "c:\scripts\DatabasesOut.txt"
Listing 4-11
Outputting Results to a File
```

Tip

For many use cases, it will be better to pipe the results into ConvertTo-Csv before piping into Out-File, but this does have the side effect of converting NULL values to System.Byte[ ].

### User Execution Context

All of the examples thus far have used Windows authentication. That is, the T-SQL scripts have been executed under the security context of the Windows user that is interactively running PowerShell and assume that Windows user has the appropriate permissions within SQL Server to execute the query.

This may not always be the case, however, especially if the script is being scheduled to run unattended, as opposed to being run interactively by a human. In these scenarios, it is possible to authenticate to SQL Server using SQL Server authentication, that is, a SQL Server Login and password.

To authenticate in this manner, you can use the `Username` and `Password` parameters, or you can use the `Credential` parameter, which passes the username and password as a `PSCredential` object. First, let’s examine the `Username` and `Password` parameters. Assuming that a SQL Login exists, called `Pete`, with a password of `Pa$$w0rd`, we can use the script in Listing [4-12](#395785_2_En_4_Chapter.xhtml#PC16) to retrieve a list of database names.

```
$Params = @{
ServerInstance  = "localhost"
Query           = "SELECT name FROM sys.databases"
Username        = "Pete"
Password        = 'Pa$$w0rd'
}
Invoke-Sqlcmd @Params
Listing 4-12
Using the Username and Password Parameters
```

This is a simple approach, but not very secure, as the username and password will be sent by clear text to the SQL Server instance. This might be OK if the command is being run from the local server, but clear text credentials should never be sent across the network. Therefore, an alternative would be to use the `Credential` parameter, as demonstrated in Listing [4-13](#395785_2_En_4_Chapter.xhtml#PC17).

```
$UserName = 'Pete'
$SecurePassword = ConvertTo-SecureString 'Pa$$w0rd' -AsPlainText -Force
$Credential = [PSCredential]::new($UserName, $SecurePassword)
$Params = @{
ServerInstance  = "localhost"
Query           = "SELECT name FROM sys.databases"
Credential      = $Credential
}
Invoke-Sqlcmd @Params
Listing 4-13
Authenticating with the Credential Parameter
```

This script begins by creating `$UserName` and `$Password` variables. The password is created as a secure string. These variables are then used to create a `PSCredential` object, which is passed to the `Credentia``l` parameter of `Invoke-SqlCmd`.

## Managing Security Objects

The sqlserver module provides more than just `Invoke-SqlCmd`, with tailored cmdlets for performing common administrative tasks. This includes a set of cmdlets for managing security objects, including Logins and Credentials. However, the module’s functionality for dealing with database users and permissions is somewhat limited; hence, we will need to interact directly with SMO (SQL Server Management Objects – the library that the sqlserver module is built upon) to manage these aspects. The following sections will discuss each of these topics.

Tip

A detailed discussion of each security object and their considerations is beyond the scope of this book, but can be found in the Apress title *Securing SQL Server*, which can be found here: [`www.apress.com/gb/book/9781484222652`](http://www.apress.com/gb/book/9781484222652).

### Working with Logins

New Logins can be created using the Add-SqlLogin cmdlet. In its most basic form, you can simply request a new SQL Server Login be created, with a given name, as demonstrated in Listing [4-14](#395785_2_En_4_Chapter.xhtml#PC18).

```
$params = @{
ServerInstance  = "localhost"
LoginName       = "Pete"
LoginType       = "SqlLogin"
DefaultDatabase = "WideWorldImporters"
Enable          = $true
GrantConnectSql = $true
}
Add-SqlLogin @params
Listing 4-14
Create a New Login
```

Tip

If you do not set `Enable = $true`, the Login will be created but left disabled. If you do not set `GrantConnectSql = $true`, then the Login will be created, but not granted permissions to connect to the Database Engine.

This will cause a password prompt to be displayed, as illustrated in Figure [4-1](#395785_2_En_4_Chapter.xhtml#Fig1), where you can enter the desired password for the Login.

Tip

If using a non-Windows operating system, the prompt will be displayed in the console, as opposed to a pop-up window.

While this is fine if you are running an ad hoc command, it is less helpful if you are trying to script processes. Therefore, you can alternatively pass a `PSCredential` object into the `LoginPsCredential` parameter, instead of using the `LoginName` parameter. This will allow both the username and password to be securely passed to SQL Server. The technique for this is demonstrated in Listing [4-15](#395785_2_En_4_Chapter.xhtml#PC19).

```
$UserName = 'Pete'
$Password = 'Pa$$w0rd'
# Convert to SecureString
$SecurePassword = ConvertTo-SecureString $Password -AsPlainText -Force
#Create The psCredential object
$LoginCredential = [PSCredential]::new($UserName, $SecurePassword)
$params = @{
ServerInstance      = "localhost"
LoginPsCredential   = $LoginCredential
LoginType           = "SqlLogin"
DefaultDatabase     = "WideWorldImporters"
Enable              = $true
GrantConnectSql     = $true
}
Add-SqlLogin @params
Listing 4-15
Create a Login with LoginPsCredential
```

![](images/395785_2_En_4_Chapter/395785_2_En_4_Fig1_HTML.jpg)

Screenshot of an SQL login password prompt. The window requests a password for a new login, displaying a username field with "Pete" selected. Below is a password input field. The window includes "OK" and "Cancel" buttons. An icon of keys is visible in the top left corner.

Figure 4-1

Password prompt

While this is fine if you are running an ad hoc command, it is less helpful if you are trying to script processes. Therefore, you can alternatively pass a

Just as if you were creating a Login using T-SQL or SQL Server Management Studio, it is possible to enforce domain password policy for a Login. This can be achieved with the help of the `EnforcePasswordPolicy`, `EnforcePasswordExpiration`, and `MustChangePasswordAtNextLogin` parameters.

These parameters form a hierarchy. It is not possible to set `EnforcePasswordExpiration` without setting `EnforcePasswordPolicy`. It is also not possible to set `MustChangePasswordAtNextLogin` without setting `EnforcePasswordExpiration`. Therefore, if you specify a parameter at a lower level of the hierarchy, then it silently implies the parameters above it.

For example, the script in Listing [4-16](#395785_2_En_4_Chapter.xhtml#PC20) only specifies `MustChangePasswordAtNextLogin`, but because this parameter is at the bottom of the hierarchy, `EnforcePasswordExpiration` and `EnforcePasswordPolicy` are also set.

```
$UserName = 'Pete'
$Password = 'Pa$$w0rd'
# Convert to SecureString
$SecurePassword = ConvertTo-SecureString $Password -AsPlainText -Force
#Create The psCredential object
$LoginCredential = [PSCredential]::new($UserName, $SecurePassword)
$params = @{
ServerInstance                  = "localhost"
LoginPsCredential               = $LoginCredential
LoginType                       = "SqlLogin"
DefaultDatabase                 = "WideWorldImporters"
Enable                          = $true
GrantConnectSql                 = $true
MustChangePasswordAtNextLogin   = $true
}
Add-SqlLogin @params
Listing 4-16
Enforcing Password Policies
```

The `Add-SqlLogin` cmdlet can also be used to create SQL Logins for Windows users, Windows groups, asymmetric keys, and certificates. This is achieved by changing the value of the `LoginType` parameter. For example, the script in Listing [4-17](#395785_2_En_4_Chapter.xhtml#PC21) will create a login for the AD User `POSHSQL\Pete`.

```
$params = @{
ServerInstance    = "localhost"
LoginName         = "POSHSQL\Pete"
LoginType         = "WindowsUser"
DefaultDatabase   = "WideWorldImporters"
Enable            = $true
GrantConnectSql   = $true
}
Add-SqlLogin @param
Listing 4-17
Create a Login for a Windows User
```

The sqlserver module also provides a `Get-SqlLogin` cmdlet which allows you to return information about a user. For example, the script in Listing [4-18](#395785_2_En_4_Chapter.xhtml#PC22) returns details of the SQL Login called Pete, which we created in an earlier example.

```
$params = @{
ServerInstance = "localhost"
LoginName      = "Pete"
}
Get-SqlLogin @params
Listing 4-18
Return Login Details
```

This command returns the results shown below:

```
Name                                        Login Type    Created
----                                        ----------    ----------------
Pete                                          SqlLogin      10/10/2020 10:33
```

The `Get-SqlLogin` command can be run with wildcard or regex searches, or if you omit the `LoginName` parameter, it will return all Logins that exist in the instance. This can be incredibly helpful for auditing, cleaning up old Logins, enforcing naming conventions, and checking if Logins exist. Of course, this command can also be added to scripts to help build automated administration routines.

With the help of another cmdlet, `Remove-SqlLogin`, let’s see how the `Get-SqlLogin` command can be used to clean up old Login objects. The script in Listing [4-19](#395785_2_En_4_Chapter.xhtml#PC24) will return all SQL Logins (filtered using the `LoginType` parameter to include SQL Logins only, not Windows users or Windows groups) and then pipe them into the `Remove-Login` cmdlet, which will drop them from the instance.

```
$params = @{
ServerInstance = "localhost"
LoginName      = "*Pete*"
Wildcard       = $true
LoginType      = "SqlLogin"
}
Get-SqlLogin @params | Remove-SqlLogin
Listing 4-19
Using Get-SqlLogin and Remove-SqlLogin to Clean Up Old Login Objects
```

### Working with Server Roles

So, we know how to create a SQL Login, but typically we would want to create a database User associated with the Login and assign our security principal permissions to SQL Server resources through server and database roles. Here, the sqlserver module lacks functionality, so instead we will need to interact directly with SMO (SQL Server Management Objects) to perform the required tasks.

The script in Listing [4-20](#395785_2_En_4_Chapter.xhtml#PC25) demonstrates how we can add our Login `Pete` to the `dbcreator` server role, so that the principal can create new databases. The script first creates an SMO object which maps to our SQL Server instance. We then create an object for the `dbcreator` server role, before finally adding our Login to the role.

```
using namespace Microsoft.SqlServer.Management.Smo
#Set variables
$ServerInstance = "localhost"
$LoginName = "Pete"
$Role = "dbcreator"
#Create an SMO instance of our SQL Instance
$Server = [Server]::new($ServerInstance)
#Traverse the instance for find the role
$ServerRole = $Server.Roles[$Role]
#Add the Login to the server role
$ServerRole.AddMember($LoginName)
Listing 4-20
Add Login to a Server Role
```

### Working with Database Users

We should now create a database User inside the `WideWorldImporters` database. Again, the sqlserver module is limited here, so we will need to interact with SMO. The script in Listing [4-21](#395785_2_En_4_Chapter.xhtml#PC26) uses a similar technique to the example above, where we added our Login to the `dbcreator` server role.

After setting our variable, the script starts by creating an SMO object that maps to our SQL Server instance. We then traverse the databases within this representation of our instance to create an SMO representation of our database. We then create an SMO object that represents our new database User, before using the `Create` method on that User object to create the Database user.

```
using namespace Microsoft.SqlServer.Management.Smo
#Set Variables
$ServerInstance = "localhost"
$LoginName = "Pete"
$UserName = "Pete"
$Database = "WideWorldImporters"
#Create an SMO instance of our SQL Instance
$Server = [Server]::new($ServerInstance)
#Create an SMO instance of our Database
$Database = $Server.Databases[$Database]
#Create an SMO objecty to represent our new user and map the appropriate Login
$DbUser = [Microsoft.SqlServer.Management.Smo.User]::New($Database, $UserName)
$DbUser.Login  = $LoginName
#Use the Create method to create the Database User
$DbUser.Create()
Listing 4-21
Create a Database User
```

Now that our User is created, we should add it to appropriate database roles to assign it permissions. In this instance, we would like to add the Login, Pete, to the `db_datareader` role. To do this, we modify the script in the previous example, so that this time, instead of creating the database user, it adds the user to the database role, using the `AddToRole` method of the User object, as demonstrated in Listing [4-22](#395785_2_En_4_Chapter.xhtml#PC27).

```
using namespace Microsoft.SqlServer.Management.Smo
#Set Variables
$ServerInstance = "localhost"
$UserName = "Pete"
$Database = "WideWorldImporters"
$Role = "db_datareader"
#Create an SMO instance of our SQL Instance
$Server = [Server]::new($ServerInstance)
#Create an SMO instance of our Database
$Database = $Server.Databases[$Database]
#Create an SMO objecty to represent our user
$DbUser = $Database.Users[$UserName]
#Use the Create method to create the Database User
$DbUser.AddToRole($Role)
Listing 4-22
Add a User to a Database Role
```

### Working with Credentials

A credential is an instance-level object in SQL Server that contains the details required to authenticate to resources outside of the SQL Server instance. A classic use case for this is working with SQL Server Agent jobs. By default, job steps within SQL Server Agent jobs run under the security context of the service account that runs the SQL Server Agent service. However, if job steps require elevated permissions, then it can help reduce the security footprint of the service account to run specific job steps under a different security context.

In this case, you can create a credential in SQL Server, which maps to a different AD account, with the appropriate permissions to perform the relevant task. This credential object can then be mapped to a SQL Server Agent proxy, which in turn is used to run relevant job steps.

Another use case for credentials is in hybrid cloud scenarios. For example, if you are performing backups to Azure Blob storage, then a credential must be used to store the Azure storage account name and access key.

The sqlserver module provides the `New-SqlCredential`, `Get-SqlCredential`, and `Remove-SqlCredential` cmdlets for managing credential objects in SQL Server. For example, the script in Listing [4-23](#395785_2_En_4_Chapter.xhtml#PC28) can be used to create a credential called `WinUser` which stores the authentication details for the Windows User `POSHSQL\Pete`.

The key thing to note about this script is that instead of providing the `ServerInstance` parameter to specify the SQL Server instance where the credential should be created, we provide the `Path` parameter. This parameter specifies the Windows Management Instrumentation (WMI) location of the instance, in the form `SQLServer:\SQL\[ServerName]\[InstanceName]`, where default is used to specify a `default` instance, or the name of the instance is used, for a named instance.

```
$Password = 'Pa$$w0rd'
# Convert to SecureString
$SecurePassword = ConvertTo-SecureString $Password -AsPlainText -Force
$params = @{
Name     = "WinUser"
Identity = "POSHSQL\Pete"
Secret   = $SecurePassword
Path     = "SQLServer:\SQL\localhost\default"
}
New-SqlCredential @params
Listing 4-23
Create a Credential
```

To demonstrate the use of the `Get-SqlCredential` and `Remove-SqlCredential` cmdlets, let’s use a similar scenario as we did for Logins. We will use the `Get-SqlCredential` cmdlet to return our `WinUser` credential and then pipe that into the `Remove-SqlCredential` cmdlet to drop them. This is demonstrated in Listing [4-24](#395785_2_En_4_Chapter.xhtml#PC29).

```
$params = @{
Name     = "WinUser"
Path     = "SQLServer:\SQL\localhost\default"
}
Get-SqlCredential @params | Remove-SqlCredential
Listing 4-24
Remove Old Credentials
```

## Working with Availability Groups

The sqlserver module provides a number of cmdlets for administering AlwaysOn Availability Groups. In this section, we will highlight the cmdlets that are available, but a full discussion of Availability Groups is beyond the scope of this book.

Tip

A full discussion of SQL Server AlwaysOn Availability Groups is beyond the scope of this book; however, full details can be found in the Apress title *SQL Server 2019 AlwaysOn*, which can be found here: link.springer.com/book/10.1007/978-1-4842-6479-9.

A list of cmdlets to support Availability Groups can be found by running the command in Listing [4-25](#395785_2_En_4_Chapter.xhtml#PC30).

```
Get-Command -Module sqlserver -Name  '*availability* | Select-Object -ExpandProperty Name
Listing 4-25
List Availability Group Cmdlets
```

The results of this command are shown below:

```
Add-SqlAvailabilityDatabase
Add-SqlAvailabilityGroupListenerStaticIp
Grant-SqlAvailabilityGroupCreateAnyDatabase
Join-SqlAvailabilityGroup
New-SqlAvailabilityGroup
New-SqlAvailabilityGroupListener
New-SqlAvailabilityReplica
Remove-SqlAvailabilityDatabase
Remove-SqlAvailabilityGroup
Remove-SqlAvailabilityReplica
Resume-SqlAvailabilityDatabase
Revoke-SqlAvailabilityGroupCreateAnyDatabase
Set-SqlAvailabilityGroup
Set-SqlAvailabilityGroupListener
Set-SqlAvailabilityReplica
Set-SqlAvailabilityReplicaRoleToSecondary
Suspend-SqlAvailabilityDatabase
Switch-SqlAvailabilityGroup
```

Tip

Get-Command is a useful helper function, irrespective of Availability Groups. Changing the value passed to the `–Name` parameter can be used to filter the results to any set of related commands.

Each cmdlet corresponds to a specific administrative task that is required to manage Availability Groups. The cmdlets beginning with the `New` verb are primarily responsible for the initial configuration of an Availability Group, alongside the `Add-SqlAvailabilityGroupListenerStaticIp` and `Grant-SqlAvailabilityGroupCreateAnyDatabase` cmdlets.

The other cmdlets are primarily concerned with post-implementation administration. For example, the script in Listing [4-26](#395785_2_En_4_Chapter.xhtml#PC32) will add the `WideWorldImporters` database to an existing Availability Group. The first part of the script seeds the database on the secondary by taking full and log backups to a share on the primary server and restoring them on the secondary. These tasks are performed using the useful `Backup-SqlDatabase` and `Restore-SqlDatabase` cmdlets. The second part of the script uses the `Add-SqlAvailabilityGroup` cmdlet to the Availability Group on each Replica.

Tip

Databases do not need to be joined to the replicas if automatic seeding is used.

```
$DatabaseBackupFile = "\\PrimaryServer.poshsql.com\backups\WideWorldImporters.bak"
$LogBackupFile = "\\PrimaryServer.poshsql.com\backups\WideWorldImporters.trn"
$AGPrimaryPath = "SQLSERVER:\SQL\PrimaryServer\default\AvailabilityGroups\WideWorldImportersAG"
$AGSecondaryPath = "SQLSERVER:\SQL\SecondaryServer\default\AvailabilityGroups\WideWorldImportersAG"
$PrimaryServerInstance = "PrimaryServer"
$SecondaryServerInstance = "SecondaryServer"
#Backup the database and log on the Primary
$BackupCommonParams = @{
Database        = "WideWorldImporters"
ServerInstance  = $PrimaryServerInstance
}
Backup-SqlDatabase @BackupCommonParams -BackupFile $DatabaseBackupFile
Backup-SqlDatabase @BackupCommonParams -BackupFile $LogBackupFile -BackupAction Log
#Restore the database and log on the Secondary
$RestoreCommonParams = @{
Database         = "WideWorldImporters"
ServerInstance   = $SecondaryServerInstance
}
Restore-SqlDatabase @RestoreCommonParams -BackupFile $DatabaseBackupFile -NoRecovery
Restore-SqlDatabase @RestoreCommonParams -BackupFile $LogBackupFile -RestoreAction Log -NoRecovery
#Add the database to the Availability Group
Add-SqlAvailabilityDatabase -Path $AGPrimaryPath -Database "WideWorldImporters"
Add-SqlAvailabilityDatabase -Path $AGSecondaryPath -Database "WideWorldImporters"
Listing 4-26
Add a Database to an Availability Group
```

Another great example of how the sqlserver module can be used to administer Availability Groups is the `Switch-AvailabilityGroup` cmdlet, which can be used to failover an Availability Group to another Replica. This functionality is demonstrated in Listing [4-27](#395785_2_En_4_Chapter.xhtml#PC33).

```
Switch-SqlAvailabilityGroup -Path "SQLSERVER:\Sql\SecondaryServer\default\AvailabilityGroups\WideWorldImportersAG"
Listing 4-27
Failover an Availability Group
```

If the Availability Group is in Asynchronous Mode, then there would be a risk of data loss. In this situation, the `AllowDataLoss` parameter must be used to ensure that the administrator running the command is aware of the potential consequences, as demonstrated in Listing [4-28](#395785_2_En_4_Chapter.xhtml#PC34).

```
Switch-SqlAvailabilityGroup -Path "SQLSERVER:\Sql\SecondaryServer\default\AvailabilityGroups\WideWorldImportersAG" –AllowDataLoss
Listing 4-28
Failover an Availability Group in Asynchronous Commit Mode
```

## Working with Miscellaneous Cmdlets

The sqlserver module also contains a number of cmdlets which do not fall into any of the sections above. For example, the `Invoke-SqlAssessment` cmdlet can be used to scan your SQL Server instance against a knowledge base of best practices to ensure that you are correctly hardened.

There are a number of cmdlets that allow you to administer SSAS (SQL Server Analysis Services), including functionality to process data in SSAS databases, which refreshes the aggregations and potentially underlying data from the source, relational databases. These can be useful for automating the administration of SSAS activities, as well as ad hoc actions.

There are a number of cmdlets that will also allow you to view information about SQL Server Agent, its jobs and job steps. For example, the command in Listing [4-29](#395785_2_En_4_Chapter.xhtml#PC35) will return information about all SQL Server Agent jobs on a given instance.

```
Get-SqlAgentJob –ServerInstance "localhost"
Listing 4-29
Return All SQL Server Agent Jobs
```

When run against an instance that is a client of SQL Server MDW (Management Data Warehouse), this command will return the following output:

```
Name                      Owner       Category             Enabled    CurrentRunStatus   DateCreated               LastModified              LastRunDuration
----                      -----       --------             -------    ----------------   -----------               ------------              ---------------
collection_set_1_nonca... sa          Data Collector       True       Idle               09/09/2020 16:01:38       07/10/2020 18:46:29       00:00:04
collection_set_2_colle... sa          Data Collector       True       Idle               09/09/2020 16:01:41       07/10/2020 18:46:30       20.18:26:41
collection_set_2_upload   sa          Data Collector       True       Idle               09/09/2020 16:01:41       07/10/2020 18:46:29       00:00:09
collection_set_3_colle... sa          Data Collector       True       Idle               09/09/2020 16:01:44       07/10/2020 18:46:30       20.17:43:04
collection_set_3_upload   sa          Data Collector       True       Idle               09/09/2020 16:01:44       07/10/2020 18:46:30       00:00:03
collection_set_6_nonca... sa          Data Collector       True       Idle               09/09/2020 16:16:50       07/10/2020 18:46:30       00:00:06
mdw_purge_data_[sysuti... sa          Data Collector       True       Idle               09/09/2020 16:01:17       07/10/2020 18:46:30       02:20:28
```

These cmdlets are slightly less useful than some other cmdlets, however, as the module only includes cmdlets to read the details of the jobs, not to update them. To update them, you would need to interact directly with SMO or use T-SQL within `Invoke-SqlCmd`.

The Read-SqlXEvent cmdlet can be used to read Extended Events and write them to the terminal. This cmdlet accepts either a connection string (to SQL Server) and the name of an Extended Events session, for reading live sessions, or can accept the name of a `FileName` parameter, if you wish to read events from an XEL file. For example, the command in Listing [4-30](#395785_2_En_4_Chapter.xhtml#PC37) can be used to read events from the live Extended Events session called LocksAndLatches_XE session to the terminal.

Tip

Further details regarding creating Extended Events sessions can be found in the Apress title *Pro SQL Server 2022 Administration*, which can be found here: link.springer.com/book/10.1007/978-1-4842-8864-1.

```
$params = @{
ConnectionString = "Server=localhost;Database=master;Trusted_Connection=True;"
SessionName      = "system_health"
}
Read-SQLXEvent @params
Listing 4-30
Reading an Extended Events Session
```

## Summary

The sqlserver module provides many cmdlets for performing common administrative tasks within SQL Server. However, there are some significant gaps in functionality. These can be worked around by interacting directly with SMO, the library that the sqlserver module is based upon. Alternatively, there is an open source module, updated by the community, called dbatools. This module plugs many of the gaps in functionality and will be discussed in Chapter [5](#395785_2_En_5_Chapter.xhtml) of this book.

The `Invoke-SqlCmd` cmdlet is a powerful tool, which allows you to run T-SQL statements from a PowerShell terminal or within PowerShell scripts. The results can be returned to variables, allowing you to write downstream logic, based on the results. For example, if you wish to perform an action on a set of databases, you can use `Invoke-SqlCmd` to retrieve a dynamic list of databases in an instance and then iterate over this list to perform the required administrative action.

The sqlserver module provides a set of cmdlets for managing SQL Server Logins. The `Add-SqlLogin` cmdlet can be used to create SQL Server Logins from Windows, with second-tier authentication, or from certificates and asymmetric keys, while the `Get-SqlLogin` cmdlet can be used to return information about a Login, and the `Remove-SqlLogin` can be used to drop logins from the instance. The module does not have the functionality to deal with database users, or permissions, however.

The module contains many miscellaneous cmdlets, including cmdlets for administering the Analysis Services database and retrieving information about SQL Server Agent jobs. A complete list of cmdlets and functions contained in the sqlserver module can be retrieved by running `Get-Command -module sqlserver`.

# 5. Working with DbaTools

As we discussed in Chapter [4](#395785_2_En_4_Chapter.xhtml), the Microsoft-maintained sqlserver PowerShell module is a great tool, allowing DBAs to easily interact with SQL Server from scripts or interactively from PowerShell; however, it does support limited functionality. The community-maintained DbaTools module addresses many of these gaps in functionality and provides an alternative to the sqlserver module for interacting with SQL Server from PowerShell.

The DbaTools module, at the time of writing, has 707 cmdlets and functions, but this number continues to grow. Therefore, it is not possible to discuss each of these in detail. Instead, after initially looking at the `Invoke-DbaQuery` function, which is equivalent to `Invoke-SqlCmd`, this chapter will focus primarily on the gaps in the sqlserver module that were discussed in Chapter [4](#395785_2_En_4_Chapter.xhtml). Specifically, we will look at how Logins and Users can be managed end to end and how SQL Server Agent artifacts can be managed. I would advise you, however, to review the list of commands available in DbaTools. A full list of commands can be found at dbatools.io/commands/, or you can use the script in Listing [5-1](#395785_2_En_5_Chapter.xhtml#PC1) to install the DbaTools module and list all available commands.

Tip

Please make sure you visit dbatools.io/commands/ and review the complete list of commands available in DbaTools.

```
# Install DbaTools
Install-Module dbatools
# List commands in the DbaTools module
Get-Command -Module dbatools
Listing 5-1
Install the DbaTools Module
```

DbaTools provides commands within the following categories:

*   Availability Groups

*   Backup and Restore

*   Community Tools

*   Connection Strings

*   Databases

*   Data Masking

*   Computer Management

*   Configuration

*   Support Tools

*   Update Watcher

*   DBCC

*   Detach and Attach

*   Diagnostics and Performance

*   Endpoints

*   Export

*   File System and Storage

*   FileStream

*   Finders

*   General

*   Log Shipping

*   Login and User Management

*   Mail and Logging

*   Max Memory

*   Migration

*   Mirroring

*   Network and Connectivity

*   Policy-Based Management

*   Registered Servers

*   Replication

*   Resource Governor

*   Security and Encryption

*   Server Management

*   Service Principal Names (SPNs)

*   Services

*   Snapshots

*   Sp_configure

*   SQL Agent

*   SQL Client Configuration

*   SQL Management Objects

*   SQL Server Integration Services (SSIS)

*   System Startup

*   Tempdb

*   Data Masking

*   Traces, Profiler, and Extended Events

*   Utilities

*   Windows Server Failover Cluster

*   Writing to SQL Tables

## Working with Invoke-DbaQuery

`Invoke-DbaQuery` offers equivalent functionality to the `Invoke-SqlCmd` command discussed in Chapter [4](#395785_2_En_4_Chapter.xhtml). Therefore, to explore the differences between the commands, we will use the same example; namely, we want to retrieve a list of databases and then pass that into a loop, which rebuilds indexes dynamically, based on their fragmentation level.

### Invoke-DbaQuery Basics

The most basic call of Invoke-DbaQuery involves passing two parameters: SqlInstance, which defines which SQL Server instance the query should be run against, and Query which supplies the query that should be executed. Listing [5-2](#395785_2_En_5_Chapter.xhtml#PC2) demonstrates using this basic call to return a list of database names from the default instance of the local server. Possible formats for the SqlInstance parameter include ServerName and ServerName\InstanceName. If you wish to run the query against a default instance of the local server, then you can use localhost, . or omit the SqlInstance parameter all together.

Tip

If you wish to follow along with the examples in this chapter, but you have not configured a secure TLS connection to your SQL Server instance, then you can run the following command to work around certificate failures when using DbaTools (you should not do this in a production environment): `Set-DbatoolsConfig -FullName sql.connection.encrypt -Value $false`.

```
$Params = @{
SqlInstance   = "localhost"
Query         = "SELECT name FROM sys.databases"
}
Invoke-DbaQuery @Params
Listing 5-2
Basic Call of Invoke-DbaQuery
```

To run the same query, but authenticate to SQL Server using a second-tier SQL Login, the SqlCredential parameter can be passed. This is demonstrated in Listing [5-3](#395785_2_En_5_Chapter.xhtml#PC3).

```
$UserName = 'Pete'
$SecurePassword = ConvertTo-SecureString 'Pa$$w0rd' -AsPlainText -Force
$Credential = [PSCredential]::new($UserName, $SecurePassword)
$Params = @{
SqlInstance   = "localhost"
Query         = "SELECT name FROM sys.databases"
sqlCredential = $Credential
}
Invoke-DbaQuery @Params
Listing 5-3
Authenticate with a SQL Login
```

### Working with Parameters

The biggest difference between `Invoke-DbaQuery` and `Invoke-SqlCmd`, in common usage of the commands, is how parameters are handled. As discussed in Chapter [4](#395785_2_En_4_Chapter.xhtml), when using `Invoke-SqlCmd`, parameters are passed into the Variable parameter as a string array of key-value pairs. With `Invoke-DbaQuery`, however, variables are passed to a parameter called `SqlParameters` as a hash table. This is an important advantage of `Invoke-DbaQuery`, as it allows for tidier, more readable, and easier-to-maintain code when dealing with multiple parameters.

The script in Listing [5-4](#395785_2_En_5_Chapter.xhtml#PC4) shows a T-SQL script that will be used to rebuild indexes, dynamically, based on levels of fragmentation that are passed from the calling invocation of `Invoke-DbaQuery`. Note that the format of the parameters is the same as native T-SQL parameters, as opposed to the SQLCMD parameter format. This can make testing and debugging scripts easier, as they can be run directly without switching to SQLCMD mode. This script will then be called by the PowerShell script in Listing [5-5](#395785_2_En_5_Chapter.xhtml#PC5).

Note

The T-SQL script will fail to execute if it’s run in isolation, because the `@Fragmentation` and `@PageSpace` parameters are not defined.

```
DECLARE @SQL NVARCHAR(MAX)
SET @SQL =
(
SELECT 'ALTER INDEX '
+ QUOTENAME(i.name)
+ ' ON ' + s.name
+ '.'
+ QUOTENAME(OBJECT_NAME(i.object_id))
+ ' REBUILD ; '
FROM sys.dm_db_index_physical_stats(DB_ID(),NULL,NULL,NULL,'DETAILED') ps
INNER JOIN sys.indexes i
ON ps.object_id = i.object_id
AND ps.index_id = i.index_id
INNER JOIN sys.objects o
ON ps.object_id = o.object_id
INNER JOIN sys.schemas s
ON o.schema_id = s.schema_id
WHERE index_level = 0
AND (avg_fragmentation_in_percent > @Fragmentation
OR avg_page_space_used_in_percent < @PageSpace)
FOR XML PATH('')
) ;
EXEC(@SQL) ;
Listing 5-4
T-SQL Script to Rebuild Indexes
```

The PowerShell script in Listing [5-5](#395785_2_En_5_Chapter.xhtml#PC5) will return a list of tables and pass them into a foreach loop that runs an index rebuild script against each database. The script passes two parameters into a T-SQL query, which is stored in a file. The location of the T-SQL script is passed through the `file` parameter. The key thing to note, however, is how the parameters are passed, using a hash table. This hash table is then passed into the `SqlParameters` parameter.

```
$Variables = @{
"Fragmentation" = "25"
"PageSpace"     = "70"
}
$Params = @{
SqlInstance = "localhost"
Query       = "SELECT name FROM sys.databases"
}
$Databases = Invoke-DbaQuery @Params
foreach ($Database in $Databases) {
$Params = @{
SqlInstance   = "localhost"
Database      = $Database.name
File          = "c:\scripts\IndexRebuild.sql"
SqlParameters = $Variables
}
Invoke-DbaQuery @Params
}
Listing 5-5
Using Parameters
```

### Error Handling

Commands within the DbaTools family attempt to help you handle errors by default. Specifically, when an error occurs, DbaTools will catch and parse the error and return a friendly error. For example, consider the command in Listing [5-6](#395785_2_En_5_Chapter.xhtml#PC6), which throws a divide by zero error.

```
Invoke-DbaQuery -SqlInstance localhost -Query "SELECT 1/0"
Listing 5-6
Producing a Friendly Error
```

The output of this error is shown below:

```
WARNING: [10:56:33][Invoke-DbaQuery] [localhost] Failed during execution | Divide by zero error encountered.
```

While this is certainly easier on the eye than a standard PowerShell exception being thrown, it can cause issues when you are scripting, as it can cause a script to fail silently. Therefore, in many circumstances, it is advisable to turn off this feature. This can be achieved by using the `–EnableException` parameter, as demonstrated in Listing [5-7](#395785_2_En_5_Chapter.xhtml#PC8).

```
Invoke-DbaQuery -SqlInstance localhost -Query "SELECT 1/0" –EnableException
Listing 5-7
Using EnableException Parameter
```

This will cause a standard exception to be thrown, as shown below:

```
Exception:
Line |
97974 |          throw $records[0]
|          ~~~~~~~~~~~~~~~~~
| Divide by zero error encountered.
```

### Running Queries Against Multiple Instances

Using Invoke-DbaQuery, it is very simple to run a query against multiple SQL Server instances. All you need to do is define a list of instance names and then pipe them into the Invoke-DbaQuery command. This is demonstrated in Listing [5-8](#395785_2_En_5_Chapter.xhtml#PC10), which returns a list of databases on each instance.

Note

The script assumes that you have a default SQL Server instance and an instance named `poshscripting`.

```
"localhost", "localhost\poshscripting" | Invoke-DbaQuery -Query "SELECT name FROM sys.databases"
Listing 5-8
Running a Query Against Multiple Servers
```

Looking at the results below, you will notice that the obvious issue is that it’s not possible to determine which instance each database resides within:

```
master
tempdb
model
msdb
AdventureWorks2022
master
tempdb
model
msdb
```

You could, of course, deal with this issue by adding the `@@SERVERNAME` to the SELECT list. `Invoke-DbaQuery` offers an alternative, however. You can simply specify the `–AppendServerInstance` parameter. The difference between the two techniques is apparent when dealing with a local server. Using the `@@SERVERNAME` technique will cause the actual server name to be returned. Using the `–AppendServerInstance` technique, however, will return whatever value you passed into `Invoke-DbaQuery`. This could be the server name, localhost, or an IP address. For example, consider the command in Listing [5-9](#395785_2_En_5_Chapter.xhtml#PC12).

```
$query = "SELECT name, @@SERVERNAME AS SQLInstance FROM sys.databases"
"localhost", "localhost\poshscripting" | Invoke-DbaQuery -Query $query –AppendServerInstance
Listing 5-9
Returning Server Name
```

The results (shown below) illustrate the potential difference between the results, with each technique:

```
name               SQLInstance                      ServerInstance
--------------     ---------------------------    -----------------------
master             WIN-J38I01U06D7                  localhost
tempdb             WIN-J38I01U06D7                  localhost
model              WIN-J38I01U06D7                  localhost
msdb               WIN-J38I01U06D7                  localhost
AdventureWorks2022 WIN-J38I01U06D7                  localhost
master             WIN-J38I01U06D7\POSHSCRIPTING    localhost\poshscripting
tempdb             WIN-J38I01U06D7\POSHSCRIPTING    localhost\poshscripting
model              WIN-J38I01U06D7\POSHSCRIPTING    localhost\poshscripting
msdb               WIN-J38I01U06D7\POSHSCRIPTING    localhost\poshscripting
```

#### Writing Results to a File

Just as with Invoke-SqlCmd, there is not a parameter which will allow you to send query results directly to a file; however, the same ends can be achieved by piping the results of Invoke-SqlCmd to the Out-File cmdlet. This is demonstrated in Listing [5-10](#395785_2_En_5_Chapter.xhtml#PC14), which writes the list of databases to a file called c:\scripts\DatabasesOut.csv.

```
$Params = @{
SqlInstance  = "localhost"
Query        = "SELECT name FROM sys.databases"
}
Invoke-DbaQuery @Params | ConvertTo-Csv | Out-File -FilePath "c:\scripts\DatabasesOut.csv"
Listing 5-10
Outputting Results to a File
```

## Managing Security Objects

In Chapter [4](#395785_2_En_4_Chapter.xhtml), we discussed how the SqlServer module can be used to create Logins and add them to roles. At the point, we wanted to start managing database users; however, we hit a gap in the functionality of the SqlServer module and had to use SMO directly to perform the required tasks.

The DbaTools module has a much richer set of commands for dealing with security objects. Therefore, in the following sections, we will walk through how we can use DbaTools to manage Logins and Users end to end. Specifically, we will explore how to create a Login, create a database User, add both the Login and the User to roles, rename a Login, delete a Login, and remove an orphaned User.

### Creating a Login

The `New-DbaLogin` command can be used for creating Logins. The simplest invocation of this command is to simply pass the name of the login to create via the `–Login` parameter and the name of the SQL Server instance where the Login should be created, via the `-SqlInstance` parameter. This is demonstrated in Listing [5-11](#395785_2_En_5_Chapter.xhtml#PC15).

Note

Change the server\login name to equate to your own server name and a local user that is present on your server.

```
$params = @{
SqlInstance     = "localhost"
Login           = "WIN-J38I01U06D7\Adam"
DefaultDatabase = "AdventureWorks2022"
}
New-DbaLogin @params
Listing 5-11
Create a Login
```

This script creates a login for an existing Windows user. If we wanted to create a SQL Login, then we would need to additionally pass the password for the Login as a SecureString. When creating a SQL Login, we can also, optionally, enforce domain password policies and expirations, just as we can with the `SqlServer` module. This is demonstrated in Listing [5-12](#395785_2_En_5_Chapter.xhtml#PC16).

```
$Password = 'Pa$$w0rd'
# Convert to SecureString
$SecurePassword = ConvertTo-SecureString $Password -AsPlainText -Force
$params = @{
SqlInstance      = "localhost"
Login            = "Adam"
SecurePassword   = $SecurePassword
DefaultDatabase  = "AdventureWorks2022"
}
New-DbaLogin @params -PasswordPolicyEnforced –PasswordExpirationEnabled
Listing 5-12
Create a SQL Login
```

When creating a SQL Login, we can also force a specific SID (Security Identifier) to be used. This is useful in scenarios such as configuration management (discussed in Chapter [9](#395785_2_En_9_Chapter.xhtml)), where we want to ensure that a Login always exists and is re-created automatically if it is unintentionally dropped.

SQL Server uses the SID to map Database Users to Logins; therefore, if the SID changes, the Database Users will be orphaned, even if the SQL Login has the same name and password. Using a specific SID removes this issue.

Tip

This issue does not occur for Logins created for Windows accounts. This is because the SID is stored in the domain (or the local server for a local Windows account). Every time you create a Login for a Windows user, it will have the same SID, even on different instances.

You can identify the SID of a SQL Login via the `sys.syslogins` table. The T-SQL query in Listing [5-13](#395785_2_En_5_Chapter.xhtml#PC17) demonstrates how to retrieve the SID of the SQL Login called `Adam`, which we created in Listing [5-12](#395785_2_En_5_Chapter.xhtml#PC16).

```
SELECT
name
, sid
FROM sys.syslogins
WHERE name = 'Adam'
Listing 5-13
Retrieve SID of Login
```

Listing [5-14](#395785_2_En_5_Chapter.xhtml#PC18) demonstrates how to create a Login with a specific SID. For this, we use the `–Sid` parameter of the `New-DbaLogin` command.

Note

The Login `Adam` already exists. Therefore, we have used the `–Force` parameter, which will cause the Login to be dropped and re-created, therefore avoiding a failure.

```
$Password = 'Pa$$w0rd'
# Convert to SecureString
[securestring]$SecurePassword = ConvertTo-SecureString $Password -AsPlainText -Force
$params = @{
SqlInstance     = "localhost"
Login           = "Adam"
SecurePassword  = $SecurePassword
DefaultDatabase = "AdventureWorks2022"
SID             = "0xC8105F39AEDBF947ACA0FA99544EBF33"
}
New-DbaLogin @params –Force
Listing 5-14
Create a Login with a Specific SID
```

### Managing Logins

DbaTools provides useful commands for managing Logins. In the following sections, we will explore how we can add a Login to a Server Role, rename, and drop Logins.

#### Adding a Login to a Server Role

Logins can be granted instance-wide permissions by adding them to Server Roles. This can be achieved with DbaTools by using the `Add-DbaServerRoleMember` command. For example, the command in Listing [5-15](#395785_2_En_5_Chapter.xhtml#PC19) adds the `Adam` Login to the `dbcreator` Server Role, allowing him to create new databases on the instance.

```
Add-DbaServerRoleMember -SqlInstance localhost -ServerRole dbCreator -Login Adam
Listing 5-15
Add a Login to a Server Role
```

Newer versions of SQL Server also allow you to create your own, custom server roles, and this functionality is also supported through DbaTools, by using the New-DbaServerRole command; however, at the time of writing, this command has very limited functionality, where a Server Role can be created, but permissions cannot be assigned to it. Therefore, we will not explore it in detail.

#### Renaming a Login

Imagine you inherit responsibility for an application and find that connections are made through a Login called `AppLogin`. We have all been there. More than being a little frustrating, you have migrated the application to a SQL instance, which hosts the databases for multiple applications. Therefore, the name simply isn’t appropriate, as troubleshooting will become confusing and complicated.

You can easily change the name of a Login by using the `Rename-DbaLogin` command. Listing [5-16](#395785_2_En_5_Chapter.xhtml#PC20) illustrates how to change the name of `AppLogin` to `SalesApp`, allowing you to differentiate between multiple application logins on the instance. The `-Login` parameter specifies the original Login name, with the `-NewLogin` parameter used to specify the desired name of the Login.

Caution

Remember that if you change the name of the application login in the SQL Server instance, it will also need to be updated in the application.

```
$params = @{
SqlInstance = "localhost"
Login       = "AppLogin"
NewLogin    = "SalesApp"
}
Rename-DbaLogin @params
Listing 5-16
Rename a Login
```

#### Drop a Login

A Login can be dropped by using the Remove-DBALogin command. To use this command, simply specify the name of the Login with the `–Login` parameter. This is demonstrated in Listing [5-17](#395785_2_En_5_Chapter.xhtml#PC21), where we drop the SalesApp Login.

```
$params = @{
SqlInstance = "localhost"
Login       = "SalesApp"
}
Remove-DbaLogin @params
Listing 5-17
Drop a Login
```

### Creating Database Users

DbaTools can be used to create Database Users as well as Instance Logins. The `New-DbaDbUser` command can be used for this purpose. In its most simple invocation, you can specify the name of the database via the `–Database` parameter and the name of the Login to which the user should be mapped via the `–Login` parameter, as illustrated in Listing [5-18](#395785_2_En_5_Chapter.xhtml#PC22), where we create a Database User associated with the `Adam` login, within the AdventureWorks2022 database.

```
$params = @{
SqlInstance = "localhost"
Database    = "AdventureWorks2022"
Login       = "Adam"
}
New-DbaDbUser @params
Listing 5-18
Create a Database User
```

You can also, optionally, use the –User parameter to specify the name of the Database User. You may choose to do this as a code-styling option, or it is also useful if you want your Database User to have a different name from the Login. For example, the script in Listing [5-19](#395785_2_En_5_Chapter.xhtml#PC23) creates a Database User called `PeteCarter` within the `AdventureWorks2022` database, which is mapped to the `Pete` Login.

```
$params = @{
SqlInstance = "localhost"
Database    = "AdventureWorks2022"
Login       = "Pete"
User        = "PeteCarter"
}
New-DbaDbUser @params
Listing 5-19
Create a Database User with a Specific Name
```

Additionally, the command can be used to copy users, either between databases within the same instance. For example, the command in Listing [5-20](#395785_2_En_5_Chapter.xhtml#PC24) creates the user Pete (which is based on the Windows Login Pete) from the AdventureWorks2022 database to the WideWorldImporters database in the same instance.

The Get-DbaDbUser command is used to get the details of all Database Users in the AdventureWorks2022 database. This is then piped into a Where-Object, which filters the results, so that only the Pete Database User is included. This is then stored in a variable called $user, which is used by the New-DbaDbUser command. The user is created in the database we specify.

Caution

It is important to note that while the Database User is copied, the permissions associated with that User are not copied with it.

```
$user = Get-DbaDbUser -SqlInstance localhost -Database AdventureWorks2022 | Where-Object name -eq "Pete"
$params = @{
Login = $user.Login
Database = 'WideWorldImporters'
SqlInstance = 'Localhost'
}
New-DbaDbUser @params
Listing 5-20
Copy a Database User
```

### Managing Database Users

DbaTools provides many useful commands for managing Database Users. In the following sections, we will explore how to add Users to database roles. We will also discuss how to drop Users and how to repair Users that have become orphaned.

#### Add a User to a Database Role

A User can be added to a database role using the `Add-DbaDbRoleMember` command. You will pass this command the name of the database via the `–Database` parameter, the role via the `–Role` parameter, and the name of the User to add to the role via the `–User` parameter. This is demonstrated in Listing [5-21](#395785_2_En_5_Chapter.xhtml#PC25), where the `PeteCarter` Login is added to the `db_owner` database role, within the `AdventureWorks2022` database.

```
$params = @{
SqlInstance = "localhost"
Database    = "AdventureWorks2022"
User        = "PeteCarter"
Role        = "db_owner"
}
Add-DbaDbRoleMember @params
Listing 5-21
Add a User to a Role
```

Tip

To avoid being prompted to confirm the action, you can append `-Confirm:$false` to the command.

A command called `New-DbaDbRole` exists, which can be used to create your own database roles. Like its server role equivalent, however, the functionality is limited to creating a role. It doesn’t allow for permissions to be assigned to that role. Therefore, we will not cover it here, and I recommend using `Invoke-DbaQuery` for creating both server and database roles.

#### Dropping Users and Removing Them from Roles

A User can be removed from a database role using the `Remove-DbaDbRoleMember` command. This is demonstrated in Listing [5-22](#395785_2_En_5_Chapter.xhtml#PC26), which removes the `PeteCarter` user from the `db_owner` database role, in the `AdventureWorks2022` database.

```
$params = @{
SqlInstance = "localhost"
Database    = "AdventureWorks2022"
User           = "PeteCarter"
Role           = "db_owner"
}
Remove-DbaDbRoleMember @params
Listing 5-22
Remove a User from a Database Role
```

If you need to drop a Database User entirely, this can be achieved by using the `Remove-DbaDbUser` command. This command requires the Database and User to be passed, as illustrated in Listing [5-23](#395785_2_En_5_Chapter.xhtml#PC27).

```
$params = @{
SqlInstance = "localhost"
Database    = "AdventureWorks2022"
User        = "PeteCarter"
}
Remove-DbaDbUser @params
Listing 5-23
Drop a Database User
```

## Managing Server Agent

In Chapter [4](#395785_2_En_4_Chapter.xhtml), we discussed how the SqlServer module enables you to create Credentials and also retrieve information about SQL Server Agent jobs, but how this is of limited use, as there are no commands provided to interact with or manage SQL Server Agent artifacts. Therefore, in the following sections, we will discuss how to manage SQL Server Agent end to end. Specifically, we will explore how to create a Credential, create a Proxy which maps to that Credential, and create a Job. We will also discuss how to retrieve Job information and how this information can be used to troubleshoot issues.

Note

This chapter assumes a familiarity with SQL Server Agent and its artifacts, such as Jobs, Job Steps, and Proxies. A full discussion and explanation of SQL Server Agent can be found in the Apress title *Pro SQL Server 2022 Administration*, which can be found here: link.springer.com/book/10.1007/978-1-4842-8864-1.

### Create a Credential

`New-DbaCredential` provides equivalent functionality of the `SqlServer` module’s `New-SqlCredential` command, which is discussed in Chapter [4](#395785_2_En_4_Chapter.xhtml) of this book. We will use the command to create a credential called `SQLAgentCredential`, which is mapped to the `Pete` Windows user and will be used as a security context for running SQL Server Agent jobs.

The command requires the name of the Credential to be passed via the `–Name` parameter. It also requires the identity that will be used (in this case, a Windows user) to be passed to the `–Identity` parameter and the secret (in this case, the password of the Windows user) to be passed to the `–SecurePassword` parameter.

The script in Listing [5-24](#395785_2_En_5_Chapter.xhtml#PC28) illustrates how to use the `New-DbaCredential` command to create the described Credential object.

```
$SecurePassword = ConvertTo-SecureString 'Pa$$w0rd' -AsPlainText -Force
$params = @{
SqlInstance     = "localhost"
Name            = "SqlAgentCredential"
Identity        = "Pete"
SecurePassword  = $SecurePassword
}
New-DbaCredential @params
Listing 5-24
Create a Credential
```

### Create a Proxy

A SQL Server Agent Proxy defines the security context under which a job step can run. Essentially, it provides a mapping of Job Step to Credential. In this example, we will create a Proxy called SQLAgentProxy, which maps to the SQLAgentCredential Credential and assign it the rights to run Job steps which are configured as the PowerShell or CmdExec Job step types.

To achieve this, we will use the `New-DbaAgentProxy` command and will pass in a name for the Proxy to the `–Name` parameter, the Credential that it will be mapped to, via the `-ProxyCredential` parameter, and finally specify the type of Job steps it can run, via the `–SubSystem` parameter, as demonstrated in Listing [5-25](#395785_2_En_5_Chapter.xhtml#PC29).

Tip

If passing the `–SubSystem` parameter directly in the command, we could simply pass a comma-separated list. Because we are using the splatting technique, we define the two values as an array of strings, which is then passed to the command.

This code is only valid on Windows-based machines.

```
$params = @{
SqlInstance     = "localhost"
Name            = "SqlAgentProxy"
ProxyCredential = "SQLAgentCredential"
SubSystem       = @("PowerShell","CmdExec")
}
New-DbaAgentProxy @params
Listing 5-25
Create a Proxy
```

A full list of valid SubSystems is as follows:

*   ActiveScripting

*   AnalysisCommand

*   AnalysisQuery

*   CmdExec

*   Distribution

*   LogReader

*   Merge

*   PowerShell

*   QueueReader

*   Snapshot

*   Ssis

Caution

The ActiveScripting SubSystem relates to running ActiveX Scripts. This is deprecated within SQL Server and should not be used.

### Create a Job

There are three aspects to creating a SQL Server Agent job with DbaTools. Firstly, you need to create the Job itself. You then need to create the Job Steps that are run by that Job, before finally attaching a Job schedule, which will define when the Job runs.

In this section, we will create a job called `DbaMaintenance`, which runs DBCC CHECKDB and an intelligent index rebuild process against each database on our instance. Because the jobs have logic built in, they will be PowerShell scripts, as opposed to T-SQL scripts. This will be discussed further in Chapters [10](#395785_2_En_10_Chapter.xhtml) and [11](#395785_2_En_11_Chapter.xhtml) of this book.

Creating the Job is achieved with the `New-DbaAgentJob` command. Its basic invocation requires a Job name via the `–Job` parameter and a job owner via the `–OwnerLogin` parameter. Creating a Job is demonstrated in Listing [5-26](#395785_2_En_5_Chapter.xhtml#PC30).

```
New-DbaAgentJob -SqlInstance localhost -Job DbaMaintenance -OwnerLogin sa
Listing 5-26
Creating a SQL Agent Job
```

Job steps can be created using the `New-DbaAgentJobStep` command. Let us first look at a simple invocation of this, before moving to a more complex example.

The script in Listing [5-27](#395785_2_En_5_Chapter.xhtml#PC31) creates the two required Job Steps. Here, we pass in the name of the step, the command that should be run, the relevant SubSystem, the Proxy name and the name of the Job that will run the steps, and the success/failure actions for each step.

We also pass the StepID. This is a sequential number, within a Job, defining the order in which the steps should run. Without this, every time you add a step, it moves into first position within the Job, giving you a slightly strange situation, where the steps are run in the reverse order of how they are defined in your code.

```
New-DbaAgentJobStep -SqlInstance localhost -Job DbaMaintenance -StepName CheckDB -ProxyName SQLAgentProxy -Command 'PowerShell "c:\scripts\CheckDB.ps1"' -Subsystem PowerShell -StepId 1 -OnSuccessAction GoToNextStep -OnFailAction GoToNextStep
New-DbaAgentJobStep -SqlInstance Localhost -Job DbaMaintenance -StepName IndexRebuild -ProxyName SQLAgentProxy -Command 'PowerShell "c:\scripts\IndexRebuild.ps1"' -Subsystem PowerShell -StepId 2 -OnSuccessAction GoToNextStep -OnFailAction GoToNextStep
Listing 5-27
Create Job Steps
```

While this works, it’s rather ugly and leads to a lot of code duplication. This is magnified when you consider that this example only uses two Job steps. Imagine if there were 20! We can work around this, however, by using a combination of splatting and looping in PowerShell, allowing us to define the majority of parameter values only once and also only have a single declaration of the command invocation. This technique is illustrated in Listing [5-28](#395785_2_En_5_Chapter.xhtml#PC32). Here, we initially use the splatting technique to define the common set of parameters. We then use a second splat, consisting of an array for each step definition. Finally, a `foreach` loop iterates over each step definition, calling `New-DbaAgentJobStep` for each step, in turn. The results are functionally equivalent, but this reduces code duplication and leads to more maintainable code.

```
$commonParams = @{
SqlInstance     = "localhost"
Job             = "DbaMaintenance"
Subsystem       = "PowerShell"
ProxyName       = "SQLAgentProxy"
OnSuccessAction = "GoToNextStep"
OnFailAction    = "GoToNextStep"
}
$steps = @(
@{ StepName = 'CheckDB';          Command = 'powershell "C:\scripts\CheckDB.ps1"';            StepID = 1 }
@{ StepName = 'RebuildIndexes'; Command = 'powershell "C:\scripts\RebuildIndexes.ps1"';   StepID = 2 }
)
foreach ($step in $steps) {
New-DbaAgentJobStep @step @commonParams
}
Listing 5-28
Improve the Code for Job Step Creation
```

The final stage is to add a schedule to the Job to define when the Job will run. This can be achieved using the New-DbaAgentSchedule command. The parameters that can be passed to this command are defined in Table [5-1](#395785_2_En_5_Chapter.xhtml#Tab1).

Table 5-1

New-DbaAgentSchedule Parameters

| Parameter | Description | Allowed Values |
| --- | --- | --- |
| Job | The name of the SQL Agent Job to which the schedule should be attached |   |
| Schedule | The name of the Schedule |   |
| Disabled | Specify if the schedule should be enabled or disabled | True, False (Default) |
| FrequencyType | Specify the frequency period of when the job should be executed | Once, OneTime, Daily, Weekly, Monthly, MonthlyRelative, AgentStart, AutoStart, IdleComputer, OnIdle |
| FrequencyInterval | The interval, within the frequency period, that the job should be executed. This can be day(s) of the week for jobs with a weekly `FrequencyType` or a day of the month for jobs with a `FrequencyType` of monthly | Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday, Weekdays, Weekend, Everyday, or numeric value from 1 to 31 (for monthly) or 1 to 365 (for daily) |
| FrequencySubdayType | For jobs that are executed multiple times a day, specifies the unit of time between each execution. For jobs that are executed only once within a day, use Time. Once specifies that the job will only execute once within its lifespan | Once, Time, Seconds, Second, Minutes, Minute, Hours, Hour |
| FrequencySubdayInterval | An integer specifying how many occurrences of the `FrequencySubdayType` should occur between each execution. For example, if `FrequencySubdayType` is set as `Minutes` and `FrequencySubdayInterval` is set as `5`, then the job will execute every 5 minutes |   |
| FrequencyRecurrenceFactor | If the `FrequencyType` is set to `Weekly`, `Monthly`, or `MonthlyRelative`, then an integer can be passed to this parameter to specify how many weeks or months should occur between each execution |   |
| FrequencyRelativeInterval | When `FrequencyInterval` is configured as `MonthRelative`, then this parameter is used to specify when, within the month, the job should be executed | First, Second, Third, Fourth, Last |
| StartDate | The date on which the schedule should begin as yyyyMMdd |   |
| EndDate | The date on which the schedule should end as yyyyMMdd |   |
| StartTime | The time when the schedule should begin. Acceptable formats are HHMMSS with a 24-hour clock |   |
| EndTime | The time when the schedule should end. Acceptable formats are HHMMSS with a 24-hour clock |   |

In our scenario, let’s imagine we would like our job to run every five minutes, every day. We want the schedule to start immediately and run indefinitely. Therefore, instead of specifying the `StartDate`, `StartTime`, `EndDate`, and `EndTime`, we will use the `–Force` parameter, which will add the current date as the start date and time as the current date/time and the end date and time as 31/12/9999 00:00:00. The script in Listing [5-29](#395785_2_En_5_Chapter.xhtml#PC33) demonstrates how to create this schedule.

Note

In reality, such a maintenance job would probably be scheduled to run once a day or, in some environments, perhaps even once a week. The schedule here is for illustrative purposes only.

```
$params = @{
SqlInstance = "localhost"
Job = "DbaMaintenance"
Schedule = "DailyEvery5Mins"
FrequencyType = "Daily"
FrequencyInterval = "Everyday"
FrequencySubdayType = "Minutes"
FrequencySubdayInterval = 5
}
New-DbaAgentSchedule @params -force
Listing 5-29
Create a Job Schedule
```

### Retrieve Job Information

DbaTools provides a number of commands for retrieving job information. This information can provide a very quick and easy mechanism to assist DBAs in troubleshooting issues. In this section, we will work through an example of this.

To begin, imagine that we are running our morning checks against a server. We want to check our SQL Agent Jobs, so we will return a list of Jobs, using the `Get-DbaAgentJob` command. By default, this command will return an object for each job, as demonstrated in Listing [5-30](#395785_2_En_5_Chapter.xhtml#PC34).

```
Get-DbaAgentJob -SqlInstance localhost
Listing 5-30
Return a List of SQL Agent Jobs
```

The results of this command will of course differ depending on what jobs you have on your instance. When I ran this command, I received the results below:

```
ComputerName    InstanceName SqlInstance     Name                    Category                OwnerLoginName CurrentRunStatus CurrentRunRetryAttempt Enabled LastRunDate         LastRunOutcome HasSchedule OperatorToEmail CreateDate
------------    ------------ -----------     ----                    --------                -------------- ---------------- ---------------------- ------- -----------         -------------- ----------- --------------- ----------
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 DbaMaintenance          [Uncategorized (Local)] sa                    Executing                      0    True 18/01/2021 08:37:00         Failed        True                 17/01/2021 16:36:29
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 syspolicy_purge_history [Uncategorized (Local)] sa                         Idle                      0    True 18/01/2021 07:09:47      Succeeded        True                 11/08/2020 13:10:41
```

The obvious thing to note from these results is that the `DbaMaintenance` job failed on its last execution. Therefore, we will retrieve further information about failed jobs, by using the `Get-DbaAgentJobHistory` command, to return the job history output. We will pipe the results of this command into a `Where-Object` command, so that we can filter for failed jobs, before finally piping into the `Format-Table` command to make the results easier to read. This is demonstrated in Listing [5-31](#395785_2_En_5_Chapter.xhtml#PC36).

```
Get-DbaAgentJobHistory -SqlInstance localhost | Where-Object status -eq "Failed" | Format-Table
Listing 5-31
Return Job History
```

The results of running this command are shown below:

```
ComputerName    InstanceName SqlInstance     Job            StepName      RunDate             StartDate               EndDate                 Duration Status OperatorEmailed Message
------------    ------------ -----------     ---            --------      -------             ---------               -------                 -------- ------ --------------- -------
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 DbaMaintenance (Job outcome) 18/01/2021 08:41:00 2021-01-18 08:41:00.000 2021-01-18 08:42:01.000 00:01:01 Failed                 The job failed.  JobManager tried to run a non-existent step (3) for job DbaMain...
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 DbaMaintenance (Job outcome) 18/01/2021 08:39:00 2021-01-18 08:39:00.000 2021-01-18 08:40:02.000 00:01:02 Failed                 The job failed.  JobManager tried to run a non-existent step (3) for job DbaMain...
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 DbaMaintenance (Job outcome) 18/01/2021 08:37:00 2021-01-18 08:37:00.000 2021-01-18 08:38:02.000 00:01:02 Failed                 The job failed.  JobManager tried to run a non-existent step (3) for job DbaMain...
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 DbaMaintenance (Job outcome) 18/01/2021 08:35:00 2021-01-18 08:35:00.000 2021-01-18 08:36:02.000 00:01:02 Failed                 The job failed.  JobManager tried to run a non-existent step (3) for job DbaMain...
```

We can see from these results which job is consistently failing in the same place. You will also note, however, that the error message is truncated. We can resolve this by piping the results of the Where-Object command into a Select-Object command, which expands the Message property. This is demonstrated in Listing [5-32](#395785_2_En_5_Chapter.xhtml#PC38).

```
Get-DbaAgentJobHistory -SqlInstance localhost | Where-Object status -eq "Failed" | Select-Object -ExpandProperty Message
Listing 5-32
Expanding the Message
```

The results of this command are shown below:

```
The job failed.  JobManager tried to run a non-existent step (3) for job DbaMaintenance.
The job failed.  JobManager tried to run a non-existent step (3) for job DbaMaintenance.
The job failed.  JobManager tried to run a non-existent step (3) for job DbaMaintenance.
The job failed.  JobManager tried to run a non-existent step (3) for job DbaMaintenance.
```

I can now categorically see that the job is failing, consistently, because it is trying to run a job step that doesn’t exist. This is clearly a bug in the job. If you cast your mind back to Listing [5-28](#395785_2_En_5_Chapter.xhtml#PC32), where we created the job steps, you may remember that we set common properties for the `OnSuccessAction` and `OnFailureAction` parameters to be `GoToNextStep` for all job steps. This included the final job step, meaning that when the final job step completes, it is trying to execute the “next job step,” which, of course, doesn’t exist.

We can resolve this issue by updating the final job step to either `QuitWithSuccess` or `QuitWithFailure`, depending on the outcome of the step. This can be achieved using the `Set-DbaAgentJobStep` command, as shown in Listing [5-33](#395785_2_En_5_Chapter.xhtml#PC40). Here, we pass in the name of the job to the `–Job` parameter, the name of the job step to the `–StepName` parameter, and the aspects of the job that we want to update. In our scenario, we will include the `-OnSuccessAction` and `-OnFailAction` parameters.

```
$params = @{
SqlInstance     = "localhost"
Job             = "DbaMaintenance"
StepName        = "RebuildIndexes"
OnSuccessAction = "QuitWithSuccess"
OnFailAction    = "QuitWithFailure"
}
Set-DbaAgentJobStep @params
Listing 5-33
Update a Job Step
```

## Summary

DbaTools provides a community-maintained alternative to Microsoft’s `SqlServer` PowerShell module. While it is not without its limitations, for example, the inability to assign permissions to roles that you have created, it provides far greater coverage of functionality than Microsoft’s offering. The commands within the module also offer greater consistency in the usage of commands, which makes code production faster.

The `Invoke-DbaQuery` command is the equivalent of Invoke-SqlCmd command in the `SqlServer` module. The functionality of both commands is aligned to a high degree. The biggest difference between the commands is how parameters are handled. `Invoke-DbaQuery` allows you to pass parameters in a hash table and then use them within your T-SQL scripts, as if they were declared within T-SQL. This can help make code easier to maintain.

DbaTools provides, at the time of writing, over 700 commands and functions, and I strongly recommend that you review a complete list of commands at dbatools.io/commands/. While it is only possible to scratch the surface of the available commands in the confines of this chapter, you can see that the raft of functionality provides the ability to manage artifacts, such as those associated with security and SQL Server Agent, in a very fast, easy, and flexible way, without having to rely on writing T-SQL scripts, within `Invoke-DbaQuery` commands.

# 6. Automating Installation

Automating an instance build not only reduces DBA effort and time to market for a new data-tier application, but it also reduces operational overhead, as it provides a consistent platform for all new data-tier applications across the enterprise.

A fully automated is more than just running setup.exe from the command line. We should also consider how we will optimize Windows and how we will test that our automated build succeeded.

There are multiple approaches that can be taken to automating the build of a SQL Server instance. For example, we could use sysprep, or in cloud, we could use an off-the-shelf image, with SQL Server already installed. These options have challenges, however, such as lack of control and flexibility.

Therefore, in this chapter, after initially reviewing how to install SQL Server from the command line, using setup.exe, we will focus on the two most flexible approaches to building an instance. Specifically, we will examine a scripted approach, followed by installing SQL Server using DSC. This second technique builds upon what you learned in Chapter [3](#395785_2_En_3_Chapter.xhtml), and I strongly recommend that you read that chapter first.

## Installing an Instance

Installing SQL Server from the command line involves running setup.exe from the PowerShell terminal. Setup.exe can be found in the root directory of the SQL Server installation media. When running setup.exe from the PowerShell terminal, you can use switches and parameters to pass in values, which will be used to configure the instance.

### Required Parameters

Although many switches and parameters are optional, some must always be included. When you are installing a stand-alone instance of the Database Engine, the parameters listed in Table [6-1](#395785_2_En_6_Chapter.xhtml#Tab1) are always required.

Table 6-1

Required Parameters

| Parameter | Usage |
| --- | --- |
| `/IACCEPTSQLSERVERLICENSETERMS` | Confirms that you accept the SQL Server license terms |
| `/SUPPRESSPRIVACYSTATEMENTNOTICE` | Prevents the privacy statement being displayed |
| `/ACTION` | Specifies the action that you want to perform, such as Install or Upgrade |
| `/FEATURES or /ROLE` | Specifies the features that you wish to install |
| `/INSTANCENAME` | The name to be assigned to the instance |
| `/SQLSYSADMINACCOUNTS` | The Windows security context(s) that will be given administrative permissions in the instance of the Database Engine |
| `/AGTSVCACCOUNT` | The account that will be used to run the SQL Agent service |
| `/AGTSVCPASSWORD` | If not a Group Managed Service Account (gMSA), this parameter is required to specify the password of the SQL Agent service account |
| `/SQLSVCACCOUNT` | The account that will be used to run the Database Engine service |
| `/SQLSVCPASSWORD` | If not a gMSA or virtual account, this parameter is required to specify the password of the Database Engine service account |
| `/qs` | Performs an unattended install. This is required on Windows Server Core since the installation wizard is not supported |

#### IACCEPTSQLSERVERLICENSETERMS Switch

Because `/IACCEPTSQLSERVERLICENSETERMS` is a simple switch that indicates your acceptance of the license terms, it does not require any parameter value be passed.

#### ACTION Parameter

When you perform a basic installation of a stand-alone instance, the value passed to the `/ACTION` parameter will be `install`; however, a complete list of possible values for the `/ACTION` parameter is shown in Table [6-2](#395785_2_En_6_Chapter.xhtml#Tab2).

Table 6-2

Values Accepted by the `/ACTION` Parameter

| Value | Usage |
| --- | --- |
| `Install` | Installs a stand-alone instance |
| `PrepareImage` | Prepares a vanilla stand-alone image, with no account-, computer-, or network-specific details |
| `CompleteImage` | Completes the installation of a prepared stand-alone image by adding account, computer, and network details |
| `Upgrade` | Upgrades an instance from SQL Server 2012, 2014, 2017, or 2019 |
| `EditionUpgrade` | Upgrades a SQL Server 2022 from a lower edition (such as Developer Edition) to a higher edition (such as Enterprise) |
| `Repair` | Repairs a corrupt instance |
| `RebuildDatabase` | Rebuilds corrupted system databases |
| `Uninstall` | Uninstalls a stand-alone instance |
| `InstallFailoverCluster` | Installs a failover clustered instance |
| `PrepareFailoverCluster` | Prepares a vanilla clustered image with no account-, computer-, or network-specific details |
| `CompleteFailoverCluster` | Completes the installation of a prepared clustered image by adding account, computer, and network details |
| `AddNode` | Adds a node to a failover cluster |
| `RemoveNode` | Removes a node from a failover cluster |

#### FEATURES Parameter

As shown in Table [6-3](#395785_2_En_6_Chapter.xhtml#Tab3), the `/FEATURES` parameter is used to specify a comma-delimited list of features that will be installed by setup, but not all features can be used on Windows Server Core.

Table 6-3

Acceptable Values of the `/FEATURES` Parameter

| Parameter Value | Use on Windows Core | Description |
| --- | --- | --- |
| `SQL` | NO | Full SQL Engine, including Full Text, Replication, and Data Quality Server |
| `SQLEngine` | YES | Database Engine |
| `FullText` | YES | Full Text search |
| `Replication` | YES | Replication components |
| `DQ` | NO | Data Quality Server |
| `PolyBase` | YES | PolyBase components |
| `AdvancedAnalytics` | YES | Machine Learning Solutions and In-Database R Services |
| `AS` | YES | Analysis Services |
| `DQC` | NO | Data Quality Client |
| `IS` | YES | Integration Services |
| `MDS` | NO | Master Data Services |
| `Tools` | NO | All client tools |
| `BC` | NO | Backward compatibility components |
| `Conn` | YES | Connectivity components |
| `DREPLAY_CTLR` | NO | Distributed Replay Controller |
| `DREPLAY_CLT` | NO | Distributed Replay Client |
| `SNAC_SDK` | YES | Client connectivity SDK |
| `SDK` | NO | Client tools SDK |
| `LocalDB` | YES | An execution mode of SQL Server express, which is used by application developers |
| `IS_MASTER` | YES | Integration Services Master, for use with scaled out SSIS deployments |
| `IS_WORKER` | YES | Integration Services Worker, for use with scaled out SSIS deployments |
| `AZUREEXTENSION` | YES | Integrate the instance with Azure ARC for centralized management |

Note

If you choose to install other SQL Server features, such as Analysis Services or Integration Services, then other parameters will also become required.

### Role Parameter

Instead of specifying a list of features to install, with the `/FEATURES` parameter, it is possible to install SQL Server in a predefined role, using the /ROLE parameter. The roles supported by the `/ROLE` parameter are detailed in Table [6-4](#395785_2_En_6_Chapter.xhtml#Tab4).

Table 6-4

Available Values for the `/ROLE` Parameter

| Parameter Value | Description |
| --- | --- |
| `SPI_AS_ExistingFarm` | Installs SSAS as a PowerPivot instance in an existing SharePoint farm |
| `SPI_AS_NewFarm` | Installs the Database Engine and SSAS as a PowerPivot instance in a new and unconfigured SharePoint farm |
| `AllFeatures_WithDefaults` | Installs all features of SQL Server and its components. I do not recommend using this option, except in the most occasional circumstances, as installing more features than are actually required increases the security footprint and resource utilization footprints of SQL Server |

### Basic Installation

When you are working with command-line parameters for setup.exe, you should observe the rules outlined in Table [6-5](#395785_2_En_6_Chapter.xhtml#Tab5) with regard to syntax.

Table 6-5

Syntax Rules for Command-Line Parameters

| Parameter Type | Syntax |
| --- | --- |
| Simple switch | `/SWITCH` |
| True/False | `/PARAMETER=true/false` |
| Boolean | `/PARAMETER=0/1` |
| Text | `/PARAMETER="Value"` |
| Multi-valued text | `/PARAMETER="Value1" "Value2"` |
| `/FEATURES` parameter | `/FEATURES=Feature1,Feature2` |

Tip

For text parameters, the quotation marks are only required if the value contains spaces. However, it is considered good practice to always include them.

Assuming that you have already navigated to the root directory of the installation media, then the command in Listing [6-1](#395785_2_En_6_Chapter.xhtml#PC1) provides PowerShell syntax for installing the Database Engine and Replication. It uses default values for all optional parameters, with the exception of the collation, which we will set to the Windows collation Latin1_General_CI_AS.

```
.\SETUP.EXE /IACCEPTSQLSERVERLICENSETERMS /ACTION="Install" /FEATURES=SQLEngine,Replication /INSTANCENAME="SCRIPTING" /SQLSYSADMINACCOUNTS="Administrator" /SQLCOLLATION="Latin1_General_CI_AS" /qs /SUPPRESSPRIVACYSTATEMENTNOTICE
Listing 6-1
Installing SQL Server from PowerShell
```

Note

If using the command prompt, instead of PowerShell, the leading `.\` characters are not required.

In this example, a SQL Server instance named SCRIPTING will be installed. Because service account parameters were not supplied, the Database Engine and the SQL Agent services will run under virtual service accounts. When installation begins, a pared-down, noninteractive version of the installation wizard will appear to keep you updated on progress.

### Smoke Tests

After performing an automated installation of an instance, it is always a good idea to perform some smoke tests. In this context, *smoke tests* refer to quick, high-level tests that ensure that the services are running and the instance is accessible.

The code in Listing [6-2](#395785_2_En_6_Chapter.xhtml#PC2) will use the PowerShell `Get-Service` cmdlet to ensure that the services relating to the SCRIPTING instance exist and to check their status. This script uses asterisks as wildcards to return all services that contain our instance name. This, of course, means that services such as SQL Browser will not be returned.

```
Get-Service -displayname *SCRIPTING* | Select-Object name, displayname, status
Listing 6-2
Checking Status of Services
```

You will notice that both the SQL Server and SQL Agent services have been installed. You can also see that the SQL Server service is started and the SQL Agent service is stopped. This aligns with our expectations, because we did not use the startup mode parameters for either service. The default startup mode for the SQL Server service is automatic, whereas the default startup mode for the SQL Agent service is manual.

The second recommended smoke test is to use `Invoke-Sqlcmd` to run a T-SQL statement which returns the instance name. To use the `Invoke-Sqlcmd` cmdlet (or any other SQL Server PowerShell cmdlets), the sqlserver PowerShell module needs to be installed. This module replaces the deprecated SQLPS module and contains many more cmdlets.

Once the sqlserver module has been installed, the script in Listing [6-3](#395785_2_En_6_Chapter.xhtml#PC3) can be used to return the name of the instance.

Note

This is also the query that is used by the IsAlive test, which is performed by a cluster. It has little system impact and just checks that the instance is accessible.

```
Invoke-Sqlcmd –serverinstance "localhost\SCRIPTING" -query -TrustServerCertificate  "SELECT @@SERVERNAME"
Listing 6-3
Checking If Instance Is Accessible
```

In this example, the `-serverinstance` switch is used to specify the instance name that you will connect to, and the `-query` switch specifies the query that will be run. You should see the query resolve successfully and returned the name of the instance.

### Optional Parameters

There are many switches and parameters that can optionally be used to customize the configuration of the instance that you are installing. The optional switches and parameters that you can use for the installation of the Database Engine are listed in Table [6-6](#395785_2_En_6_Chapter.xhtml#Tab6).

Tip

Account passwords should not be specified if the account being used is an MSA/gMSA. This includes the accounts for the Database Engine and SQL Server Agent services, which are otherwise mandatory.

Table 6-6

Optional Parameters

| Parameter | Usage |
| --- | --- |
| `/AGTSVCSTARTUPTYPE` | Specifies the startup mode of the SQL Server Agent service. This can be set to `Automatic`, `Manual`, or `Disabled` |
| `/BROWSERSVCSTARTUPTYPE` | Specifies the startup mode of the SQL Server Browser service. This can be set to `Automatic`, `Manual`, or `Disabled` |
| `/CONFIGURATIONFILE` | Specifies the path to a configuration file, which contains a list of switches and parameters so that they do not have to be specified inline when running setup |
| `/ENU` | Dictates that the English version of SQL Server will be used. Use this switch if you are installing the English version of SQL Server on a server with localized settings and the media contains language packs for both English and the localized operating system |
| `/FILESTREAMLEVEL` | Used to enable FILESTREAM and set the required level of access. This can be set to `0` to disable FILESTREAM, `1` to allow connections via SQL Server only, `2` to allow IO streaming, or `3` to allow remote streaming. The options from `1` to `3` build on each other, so by specifying level `3`, you are implicitly specifying levels `1` and `2` as well |
| `/FILESTREAMSHARENAME` | Specify the name of the Windows file share where FILESTREAM data will be stored. This parameter becomes required when `/FILESTREAMLEVEL` is set to a value of `2` or `3` |
| `/FTSVCACCOUNT` | The account used to run the Full-Text filter launcher service |
| `/FTSVCPASSWORD` | The password of the account used to run the Full-text filter launcher service |
| `/HIDECONSOLE` | Specifies that the console should be hidden |
| `/INDICATEPROGRESS` | When this switch is used, the setup log is piped to the screen during installation |
| `/IACCEPTPYTHONLICENSETERMS` | Must be specified if installing the Anaconda Python package, using /q or /qs |
| `/IACCEPTROPENLICENSETERMS` | Must be specified when installing Microsoft R package, using /q or /qs |
| `/INSTANCEDIR` | Specifies a folder location for the instance |
| `/INSTANCEID` | Specifies an ID for the instance. It is considered bad practice to use this parameter |
| `/INSTALLSHAREDDIR` | Specifies a folder location for 64-bit components that are shared between instances |
| `/INSTALLSHAREDWOWDIR` | Specifies a folder location for 32-bit components that are shared between instances. This location cannot be the same as the location for 64-bit shared components |
| `/INSTALLSQLDATADIR` | Specifies the default folder location for instance data |
| `/NPENABLED` | Specifies if Named Pipes should be enabled. This can be set to `0` for disabled or `1` for enabled |
| `/PID` | Specifies the PID for SQL Server. Unless the media is pre-pidded, failure to specify this parameter will cause the Evaluation Edition to be installed |
| `/PBENGSVCACCOUNT` | Specifies the account that will be used to run the PolyBase service |
| `/PBDMSSVCPASSWORD` | Specifies the password for the account that will run the PolyBase service |
| `/PBENGSVCSTARTUPTYPE` | Specifies the startup mode of the PolyBase. This can be set to `Automatic`, `Manual`, or `Disabled` |
| `/PBPORTRANGE` | Specifies a range of ports for the PolyBase service to listen on. Must contain a minimum of six ports |
| `/PBSCALEOUT` | Specifies if the Database Engine is part of a PolyBase scale-out group |
| `/SAPWD` | Specifies the password for the SA account. This parameter is used when `/SECURITYMODE` is used to configure the instance as mixed-mode authentication. This parameter becomes required if `/SECURITYMODE` is set to `SQL` |
| `/SECURITYMODE` | Use this parameter, with a value of `SQL`, to specify mixed mode. If you do not use this parameter, then Windows authentication will be used |
| `/SQLBACKUPDIR` | Specifies the default location for SQL Server backups |
| `/SQLCOLLATION` | Specifies the collation the instance will use |
| `/SQLMAXMEMORY` | Specifies the maximum amount of RAM that can be used by the database engine (mainly for the buffer cache) |
| `/SQLMINMEMORY` | Specifies the minimum amount of RAM that can be used by the database engine (mainly for the buffer cache) |
| `/SQLSVCSTARTUPTYPE` | Specifies the startup mode of the Database Engine service. This can be set to `Automatic`, `Manual`, or `Disabled` |
| `/SQLTEMPDBDIR` | Specifies a folder location for TempDB data files |
| `/SQLTEMPDBLOGDIR` | Specifies a folder location for TempDB log files |
| `/SQLTEMPDBFILECOUNT` | Specifies the number of TempDB data files that should be created |
| `/SQLTEMPDBFILESIZE` | Specifies the size of each TempDB data file |
| `/SQLTEMPDBFILEGROWTH` | Specifies the growth increment for TempDB data files |
| `/SQLTEMPDBLOGFILESIZE` | Specifies the initial size for the TempDB log file |
| `/SQLTEMPDBLOGFILEGROWTH` | Specifies the growth increment for TempDB log files |
| `/SQLUSERDBDIR` | Specifies a default location for the data files of user databases |
| `/SQLUSERDBLOGDIR` | Specifies the default folder location for log files of user databases |
| `/SQLSVCINSTANTFILEINIT` | Specifies that the Database Engine service account should be granted the Perform Volume Maintenance Tasks privilege. Acceptable values are **true** or **false** |
| `/TCPENABLED` | Specifies if TCP will be enabled. Use a value of `0` to disable or `1` to enable |
| `/UPDATEENABLED` | Specifies if Product Update functionality will be used. Pass a value of `0` to disable or `1` to enable |
| `/UPDATESOURCE` | Specify a location for Product Update to search for updates. A value of `MU` will search Windows Update, but you can also pass a file share or UNC |

### Product Update

The Product Update functionality replaces the deprecated slipstream installation functionality of SQL Server and provides you with the ability to install the latest CU (cumulative update) or GDR (General Distribution Release – a hotfix for security issues) at the same time you are installing the SQL Server base binaries. This functionality can save DBAs the time and effort associated with installing the latest update immediately after installing a SQL Server instance and can also help provide consistent patching levels across new builds.

Tip

From SQL Server 2017 onward, Service Packs are no longer released. All updates at CUs or GDRs.

In order to use this functionality, you must use two parameters during the command-line install. The first of these is the `/UPDATEENABLED` parameter. You should specify this parameter with a value of `1` or `True`. The second is the `/UPDATESOURCE` parameter. This parameter will tell setup where to look for the product update. If you pass a value of `MU` into this parameter, then setup will check Microsoft Update, or a WSUS service, or alternatively, you can supply a relative path to a folder or the UNC (Uniform Naming Convention) of a network share.

In the following example, we will examine how to use this functionality to install SQL Server 2022, with CU1 included, which will be located in a network share. When you download a GDR or CU, they will arrive wrapped in a self-extracting executable. This is extremely useful, because even if WSUS is not in use in your environment, once you have signed off on a new patching level, you can simply replace the CU within your network share; when you do, all new builds can receive the latest update, without you needing to change the PowerShell script that you use for building new instances.

The PowerShell command in Listing [6-4](#395785_2_En_6_Chapter.xhtml#PC4) will install an instance of SQL Server, named SCRIPTING2, and install CU1 at the same time, which is located on a file server.

Note

The account that you are using to run the installation will require permissions to the file share.

```
.\SETUP.EXE /IACCEPTSQLSERVERLICENSETERMS /ACTION="Install" /FEATURES=SQLEngine,Replication /INSTANCENAME="SCRIPTING2" /SQLSVCACCOUNT="MyDomain\SQLServiceAccount1" /SQLSVCPASSWORD="Pa$$w0rd" /AGTSVCACCOUNT="MyDomain\SQLServiceAccount1" /AGTSVCPASSWORD="Pa$$w0rd" /SQLSYSADMINACCOUNTS="MyDomain\SQLDBA" /UPDATEENABLED=1 /UPDATESOURCE="\\192.168.183.1\SQL2022_CU1\" /qs
Listing 6-4
Installing CU During Setup
```

The code in Listing [6-5](#395785_2_En_6_Chapter.xhtml#PC5) demonstrates how you can interrogate the difference between the two instances. The code uses invoke-sqlcmd to connect to the SCRIPTING2 instance and return the systems variable that contains the full version details of the instance, including the build number. The name of the instance is also included to help us easily identify the results.

```
$parameters = @{
ServerInstance = 'localhost\SCRIPTING2'
TrustServerCertificate = $true
Query               = "
SELECT
@@SERVERNAME
, @@VERSION
"
}
Invoke-sqlcmd @parameters | Format-List
Listing 6-5
Determining Build Version of Each Instance
```

### Using a Config File

The sample in Listing [6-6](#395785_2_En_6_Chapter.xhtml#PC6) is the content of a configuration file, which has been populated with all of the required parameters that are needed to install an instance named EXPERTSCRIPTING3\. It also contains the optional parameters to enable named pipes and TCP/IP, enables FILESTREAM at the access level where it can only be accessed via T-SQL, sets the SQL Agent service to start automatically, and configures the collation to be Latin1_General_CI_AS. In this .ini file, comments are defined with a semicolon at the beginning of the line.

```
; SQL Server 2022 Configuration File
[OPTIONS]
; Accept the SQL Server License Agreement
IACCEPTSQLSERVERLICENSETERMS
; Specifies a Setup work flow, like INSTALL, UNINSTALL, or UPGRADE.
; This is a required parameter.
ACTION="Install"
; Setup will display progress only, without any user interaction.
QUIETSIMPLE="True"
; Specifies features to install, uninstall, or upgrade.
FEATURES=SQLENGINE,REPLICATION
; Specify a default or named instance. MSSQLSERVER is the default instance for
; non-Express editions and SQLExpress is for Express editions. This parameter is
; required when installing the SQL Server Database Engine (SQL), Analysis
; Services (AS)
INSTANCENAME="EXPERTSCRIPTING3"
; Agent account name
AGTSVCACCOUNT="MyDomain\SQLServiceAccount1"
; Agent account password
AGTSVCPASSWORD="Pa$$w0rd"
; Auto-start service after installation.
AGTSVCSTARTUPTYPE="Automatic"
; Level to enable FILESTREAM feature at (0, 1, 2 or 3).
FILESTREAMLEVEL="1"
; Specifies a Windows collation or an SQL collation to use for the Database
; Engine.
SQLCOLLATION="Latin1_General_CI_AS"
; Account for SQL Server service: Domain\User or system account.
SQLSVCACCOUNT="MyDomain\SQLServiceAccount1"
; Password for the SQL Server service account.
SQLSVCPASSWORD="Pa$$w0rd"
; Windows account(s) to provision as SQL Server system administrators.
SQLSYSADMINACCOUNTS="MyDomain\SQLDBA"
; Specify 0 to disable or 1 to enable the TCP/IP protocol.
TCPENABLED="1"
; Specify 0 to disable or 1 to enable the Named Pipes protocol.
NPENABLED="1"
Listing 6-6
Configuration File for SCRIPTING3
```

Assuming that this configuration file had been saved as c:\SQL2022\configuration1.ini, then the code in Listing [6-7](#395785_2_En_6_Chapter.xhtml#PC7) could be used to run setup.exe from PowerShell.

```
.\setup.exe /CONFIGURATIONFILE="c:\SQL2022\Configuration1.ini"
Listing 6-7
Installing SQL Server Using a Configuration File
```

Although this is a perfectly valid use of a configuration file, you can actually be a little bit more sophisticated and use this approach to create a reusable script, which can be run on any server, to help you introduce a consistent build process. This is particularly useful if your Windows operational teams have not adopted the use of sysprep or use other methods to build servers.

In Listing [6-8](#395785_2_En_6_Chapter.xhtml#PC8), you will see another configuration file. This time, however, it only includes the static parameters that you expect to be consistent across your estate. Parameters that will vary for each installation, such as instance name and service account details, have been omitted.

```
;SQL Server 2022 Configuration File
[OPTIONS]
; Accept the SQL Server License Agreement
IACCEPTSQLSERVERLICENSETERMS
; Specifies a Setup work flow, like INSTALL, UNINSTALL, or UPGRADE.
; This is a required parameter.
ACTION="Install"
; Setup will display progress only, without any user interaction.
QUIETSIMPLE="True"
; Specifies features to install, uninstall, or upgrade.
FEATURES=SQLENGINE,REPLICATION
; Auto-start service after installation.
AGTSVCSTARTUPTYPE="Automatic"
; Level to enable FILESTREAM feature at (0, 1, 2 or 3).
FILESTREAMLEVEL="1"
; Specifies a Windows collation or an SQL collation to use for the Database Engine.
SQLCOLLATION="Latin1_General_CI_AS"
; Windows account(s) to provision as SQL Server system administrators.
SQLSYSADMINACCOUNTS="MyDomain\SQLDBA"
; Specify 0 to disable or 1 to enable the TCP/IP protocol.
TCPENABLED="1"
; Specify 0 to disable or 1 to enable the Named Pipes protocol.
NPENABLED="1"
Listing 6-8
Configuration File for SCRIPTING4
```

This means that to successfully install the instance, you will need to use a mix of parameters from the configuration file and also inline with the command that runs setup.exe, as demonstrated in Listing [6-9](#395785_2_En_6_Chapter.xhtml#PC9). This example assumes that the configuration in Listing [6-8](#395785_2_En_6_Chapter.xhtml#PC8) has been saved as C:\SQL2022\Configuration2.ini and will install an instance named EXPERTSCRIPTING3.

```
.\SETUP.EXE /INSTANCENAME="SCRIPTING4 /SQLSVCACCOUNT="MyDomain\SQLServiceAccount1" /SQLSVCPASSWORD="Pa$$w0rd" /AGTSVCACCOUNT="MyDomain\SQLServiceAccount1" /AGTSVCPASSWORD="Pa$$w0rd" /CONFIGURATIONFILE="C:\SQL2022\Configuration2.ini"
Listing 6-9
Installing SQL Server Using a Mix of Parameters and a Configuration File
```

## Automated Installation with Scripts

This approach gives us the benefit of having a consistent configuration file that we do not need to modify every time we build out a new instance. This idea can be taken even further, however. If we were to save our PowerShell command as a PowerShell script, then we could run the script and pass in parameters, rather than rewrite the command each time. This will give a consistent script for building new instances, which we can place under change control. The code in Listing [6-10](#395785_2_En_6_Chapter.xhtml#PC10) demonstrates how to construct a parameterized PowerShell script, which will use the same configuration file. The script assumes D:\ is the root folder of the installation media.

```
param(
[string]       $InstanceName,
[PSCredential] $SQLServiceAccountCredential,
[PSCredential] $AgentServiceAccountCredential
)
$params = @(
'/INSTANCENAME="{0}"' -f $InstanceName
'/SQLSVCACCOUNT="{0}"' -f $SQLServiceAccountCredential.Username
'/SQLSVCPASSWORD="{0}"' -f $SQLServiceAccountCredential.GetNetworkCredential().Password
'/AGTSVCACCOUNT="{0}"' -f $AgentServiceAccountCredential.Username
'/AGTSVCPASSWORD="{0}"' -f$AgentServiceAccountCredential.GetNetworkCredential().Password
'/CONFIGURATIONFILE="C:\SQL2022\Configuration2.ini"'
)
Start-Process -FilePath 'C:\SQLServerSetup\SETUP.EXE' -ArgumentList $params -Wait -NoNewWindow
Listing 6-10
PowerShell Script for Auto-install
```

Assuming that this script is saved as SQLAutoInstall1.ps1, the command in Listing [6-11](#395785_2_En_6_Chapter.xhtml#PC11) can be used to build an instance named SCRIPTING5\. This command runs the PowerShell script, passing in parameters, which are then used in the setup.exe command.

```
./SQLAutoInstall.ps1 -InstanceName 'SCRIPTING5' -SQLServiceAccount 'MyDomain\SQLServiceAccount1' -SQLServiceAccountPassword 'Pa$$w0rd' -AgentServiceAccount 'MyDomain\SQLServiceAccount1' -AgentServiceAccountPassword 'Pa$$w0rd'
Listing 6-11
Running SQLAutoInstall.ps1
```

### Enhancing the Installation Routine

You could also extend the SQLAutoInstall.ps1 script further and use it to incorporate the techniques that you learned in Chapter [1](#395785_2_En_1_Chapter.xhtml) for the configuration of operating system components and the techniques that you learned earlier in this chapter for performing smoke tests.

After installing an instance, the amended script in Listing [6-12](#395785_2_En_6_Chapter.xhtml#PC12), which we will refer to as SQLAutoInstall2.ps1, uses powercfg to set the High Performance power plan and `Set-ItemProperty` to prioritize background services over foreground applications. It then runs smoke tests to ensure that the SQL Server and SQL Agent services are both running and that the instance is accessible.

```
param(
[string]       $InstanceName,
[PSCredential] $SQLServiceAccountCredential,
[PSCredential] $AgentServiceAccountCredential
)
# Initialize ConnectionString variable
$serverName = $env:computername
$connectionString = '{0}\{1}' -f $serverName, $InstanceName
#Install the instance
$params = @(
'/INSTANCENAME="{0}"' -f $InstanceName
'/SQLSVCACCOUNT="{0}"' -f $SQLServiceAccountCredential.Username
'/SQLSVCPASSWORD="{0}"' -f $SQLServiceAccountCredential.GetNetworkCredential().Password
'/AGTSVCACCOUNT="{0}"' -f $AgentServiceAccountCredential.Username
'/AGTSVCPASSWORD="{0}"' -f$AgentServiceAccountCredential.GetNetworkCredential().Password
'/CONFIGURATIONFILE="C:\SQL2022\Configuration2.ini"'
)
Start-Process -FilePath 'C:\SQLServerSetup\SETUP.EXE' -ArgumentList $params -Wait -NoNewWindow
# Configure OS settings
Start-Process -FilePath 'C:\Windows\System32\powercfg.exe' -ArgumentList '-setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c' -Wait -NoNewWindow
Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl\ -Name Win32PrioritySeparation -Value 24
# Run smoke tests
Get-Service -DisplayName ('*{0}*' -f $InstanceName)
Invoke-SqlCmd -Serverinstance $connectionString -Query "SELECT @@SERVERNAME" -TrustServerCertificate
Listing 6-12
Enhanced PowerShell Auto-install Script
```

As well as passing variables into the setup.exe command, this script also uses the $InstanceName parameter as input for the smoke tests. The parameter can be passed straight into get-service cmdlet, with wildcards on either side. For invoke-sqlcmd, however, we need to do a little extra work. `Invoke-Sqlcmd` requires the full name of the instance, including the server name, or localhost, assuming that the script is always run locally. The script pulls the name of the server from the ComputerName environmental variable and then concatenates this with the $InstanceName variable, placing a \ between the two. This concatenated value populates the $ConnectionString variable, which can then be passed into the `-Serverinstance` switch.

### Production Readiness

Finally, you may wish to add some defensive coding to your script in order to make it production ready. Although PowerShell has try/catch functionality, it cannot be used here due to setup.exe being an external application, which does not return exit codes in a manner understood by PowerShell. Therefore, the most effective technique for ensuring the smooth running of this script is to enforce mandatory parameters.

The code in Listing [6-13](#395785_2_En_6_Chapter.xhtml#PC13) is a modified version of the script, which we will refer to as SQLAutoInstall3.ps1. This version of the script uses the Parameter keyword to set the Mandatory attribute to true for each of the parameters. This is important because if the person running this script were to omit any of the parameters, or if there was a typo in the parameter name, the installation would fail. This provides a fail-safe by ensuring that all of the parameters have been entered before allowing the script to run. The additional change that we have made in this script is to add annotations before and after each step, so that if the script does fail, we can easily see where the error occurred.

```
param(
[Parameter(Mandatory=$true)]
[string]       $InstanceName,
[PSCredential] $SQLServiceAccountCredential = (Get-Credential -Message 'Enter the SQL service account credential'),
[PSCredential] $AgentServiceAccountCredential = (Get-Credential -Message 'Enter the SQL Server Agent service account credential')
)
# Initialize ConnectionString variable
Write-Host 'Initialise variables...'
$serverName = $env:computername
$connectionString = '{0}\{1}' -f $serverName, $InstanceName
Write-Host 'Initialise variables complete'
#Install the instance
Write-Host 'Install the instance...'
$params = @(
'/INSTANCENAME="{0}"' -f $InstanceName
'/SQLSVCACCOUNT="{0}"' -f $SQLServiceAccountCredential.Username
'/SQLSVCPASSWORD="{0}"' -f $SQLServiceAccountCredential.GetNetworkCredential().Password
'/AGTSVCACCOUNT="{0}"' -f $AgentServiceAccountCredential.Username
'/AGTSVCPASSWORD="{0}"' -f$AgentServiceAccountCredential.GetNetworkCredential().Password
'/CONFIGURATIONFILE="C:\SQL2022\Configuration2.ini"'
)
Start-Process -FilePath 'C:\SQLServerSetup\SETUP.EXE' -ArgumentList $params -Wait -NoNewWindow
Write-Host 'Instance installation complete'
# Configure OS settings
Write-Host 'Configure OS settings...'
Start-Process -FilePath 'C:\Windows\System32\powercfg.exe' -ArgumentList '-setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c' -Wait -NoNewWindow
Write-Host 'High Performance power plan configured'
Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl\ -Name Win32PrioritySeparation -Value 24
Write-Host 'Optimise for background services configured'
# Run smoke tests
Write-Host 'Run smoke tests...'
Get-Service -DisplayName ('*{0}*' -f $InstanceName)
Write-Host 'Service running check complete'
Invoke-SqlCmd -Serverinstance $connectionString -Query "SELECT @@SERVERNAME" -TrustServerCertificate
Write-Host 'Instance accessibility check complete'
Listing 6-13
Auto-install Script with Defensive Code
```

The SQLAutoInstall3.ps1 script has been run, but without any parameters specified. In previous versions of the script, PowerShell would have gone ahead and executed the code, only for setup.exe to fail, since no values were specified for the required parameters. In this version, however, you can see that you will be prompted to enter a value for each parameter in turn. It is achieved in two different ways – either via the “Mandatory” attribute or by providing a default, which in this case happens to be a command that will prompt the user to enter the missing credential.

When running the script, you will notice that before and after each phase of the script execution, our annotations are shown. This can aid you in responding to errors, because you can easily see which command caused an issue before you even begin to decipher any error messages that may be displayed.

## Automated Installation with DSC

In Chapter [3](#395785_2_En_3_Chapter.xhtml), we discussed how to configure the Windows operating system using DSC. In this chapter, we will expand that configuration, so that it also installs our SQL Server instance. To achieve this, we will need to install the SQLServerDSC module, and this action can be performed using the command in Listing [6-14](#395785_2_En_6_Chapter.xhtml#PC14). This must be done in PowerShell 5.

```
Install-Module SqlServerDsc
Listing 6-14
Install the SQLServerDSC Module
```

We can now expand the configuration that we created in Chapter [3](#395785_2_En_3_Chapter.xhtml), with a resource that will ensure that a SQL Server instance called `DSCInstance` is created, using the Developer Edition. This resource is shown in Listing [6-15](#395785_2_En_6_Chapter.xhtml#PC15). You will notice that we pass the desired name of the instance, the source path of the SQL Server installation media, and the product key as strings. We pass `SQLENGINE` into the features parameter, but if we wanted to install multiple components, this string would consist of a comma-separated list. We also pass an array of Windows logins that we want to be added to the `sysadmins` fixed server role. In our case, we are just passing one login – `Administrator`.

Note

You should change the source path to reflect the location of your SQL Server installation media.

```
SqlSetup 'InstallInstance' {
InstanceName = 'DSCInstance'
Features = 'SQLENGINE'
SourcePath = 'C:\SQL Media'
SQLSysAdminAccounts = @('Administrator')
ProductKey = '22222-00000-00000-00000-00000'
}
Listing 6-15
InstallInstance Resource
```

Tip

The product key we pass is the product key for the Developer Edition, and it means that this is the edition that will be installed. If we passed out Enterprise or Standard Edition product keys, this would change the edition that is installed.

So, let’s examine Listing [6-16](#395785_2_En_6_Chapter.xhtml#PC16), which demonstrates what our updated configuration will look like, with the new resource added. There are two important things to note about the configuration. The first is that we have changed the service that the `SQLServerService` resource is managing, from the default instance to the `DSCInstance` instance, as this is the instance we will be working with. The second important aspect to note is that the syntax `DependsOn = '[SqlSetup]InstallInstance'` has been added to the end of the `SQLServerService` resource. This is because we do not want the configuration to try to ensure that the SQL Server Database Engine service is started, before the instance is installed. When the configuration is compiled, there is no guarantee of the order in which resources will be configured. Therefore, if there is a dependency between resources, we should always use `DependsOn` to ensure the correct order.

```
Configuration WindowsConfig {
Import-DscResource -ModuleName 'PSDesiredStateConfiguration'
Import-DscResource -ModuleName 'SqlServerDsc'
Node 'localhost' {
File CreateCertificateBackupsFolder {
Ensure = "Present"
Type = "Directory"
DestinationPath = "C:\CertificateBackups"
}
Registry OptimizeForBackgroundServices {
Ensure      = "Present"
Key         = " HKEY_LOCAL_MACHINE:\SYSTEM\CurrentControlSet\Control\PriorityControl"
ValueName   = "Win32PrioritySeparation"
ValueData   = 24
ValueType   = 'Dword'
}
SqlSetup 'InstallInstance' {
InstanceName        = 'DSCInstance'
Features            = 'SQLENGINE'
SourcePath          = 'C:\SQL Media'
SQLSysAdminAccounts = @('Administrator')
ProductKey          = '22222-00000-00000-00000-00000'
}
Service SQLServerService
{
Name        = 'MSSQL$DSCInstance'
StartupType = 'Automatic'
State       = 'Running'
DependsOn    = '[SqlSetup]InstallInstance'
}
}
}
WindowsConfig
Listing 6-16
Updated Configuration
```

Currently, this configuration will work, but it is rather inflexible. It will always install an instance with the same name, and it will always install the Developer Edition. To make it more flexible, we can parameterize the configuration, so that when the configuration is applied, we can pass an instance name and product key that is relevant to the node in question.

This parameterized version of the configuration is shown in Listing [6-17](#395785_2_En_6_Chapter.xhtml#PC17). You will notice that a param block has been added to the declaration, which accepts parameters for the instance name and edition. The `$SQLInstanceName` parameter has been given a value of `MSSQLSERVER`, which indicates a default instance. An `IF...ELSE IF...` block has also been added, which calculates which product key to use, based on the edition that has been passed to the configuration.

Note

For the purpose of this example, the product key for the Evaluation Edition has been used for both Enterprise and Standard Editions. These product keys should be substituted for your own Enterprise and Standard Edition product keys.

```
Configuration WindowsConfig {
param (
[string] $SqlInstanceName = 'MSSQLSERVER',
[Parameter(Mandatory)]
[ValidateSet('Developer', 'Standard', 'Enterprise')]
[string] $Edition
)
Import-DscResource -ModuleName 'PSDesiredStateConfiguration'
Import-DscResource -ModuleName 'SqlServerDsc'
if ($Edition -eq 'Developer') {
$ProductKey = '22222-00000-00000-00000-00000'
} elseif ($edition -eq 'Standard') {
$ProductKey = '00000-00000-00000-00000-00000'
} elseif ($edition -eq 'Enterprise') {
$ProductKey = '00000-00000-00000-00000-00000'
}
Node 'localhost' {
File CreateCertificateBackupsFolder {
Ensure = "Present"
Type = "Directory"
DestinationPath = "C:\CertificateBackups"
}
Registry OptimizeForBackgroundServices {
Ensure      = "Present"
Key         = "HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl"
ValueName   = "Win32PrioritySeparation"
ValueData   = 24
ValueType   = 'Dword'
}
SqlSetup 'InstallInstance' {
InstanceName        = $SqlInstanceName
Features            = 'SQLENGINE'
SourcePath          = 'C:\SQL Media'
SQLSysAdminAccounts = @('Administrator')
ProductKey          = $ProductKey
}
Service SQLServerService
{
Name        = 'MSSQL${0}' -f $SqlInstanceName
StartupType = 'Automatic'
State       = 'Running'
DependsOn    = '[SqlSetup]InstallInstance'
}
}
}
Listing 6-17
Parameterized Configuration
```

We can now pass parameters into Start-DscConfiguration to apply the configuration. The command in Listing [6-18](#395785_2_En_6_Chapter.xhtml#PC18) will apply the configuration, ensuring that the DSCInstance exists and uses the Developer Edition. The nuance here is that because we are passing in parameters, we must first dot-source the script to bring it into the same session as the calling context.

```
# dot source the configuration file
. C:\Scripts\WindowsConfig.ps1
# compile the modffile
WindowsConfig -Edition Developer -SqlInstanceName 'DSCInstance2'
# apply the configuration
Start-DscConfiguration -Path "C:\Scripts\WindowsConfig" -Verbose -Wait
Listing 6-18
Apply the Parameterized Configuration
```

In Chapter [9](#395785_2_En_9_Chapter.xhtml), we will explore how we can use DSC to configure our SQL Server instance, according to our organization’s best practice.

## Summary

Installing SQL Server is a monotonous task and subject to human error, which can lead to a divergent estate. The key to addressing this is automation. There are many ways in which a SQL Server installation can be automated, but in the modern workplace, we will usually opt for either a scripted approach or a configuration management approach.

A scripted approach involves using a PowerShell script to run setup.exe and then configure the instance to our requirements. We should also include smoke tests in the script to ensure that the instance is installed correctly.

Alternatively, we can use a configuration management approach, which will use a tool, such as PowerShell DSC, to manage the instance installation. This approach can be parameterized, just as a scripted approach can, and ensures that the instance is always present. In Chapter [9](#395785_2_En_9_Chapter.xhtml), we will explore how to use a configuration management approach to configure a SQL Server instance.

# 7. Metadata-Driven Automation

Metadata is data that describes other data. SQL Server exposes a large amount of metadata. This includes structural metadata, which describes every object, and descriptive metadata, which describes the data itself. Metadata is exposed through catalog views, information schema views, dynamic management views, dynamic management functions, system functions, and stored procedures. Metadata is the key to automation, as it allows you to create dynamic, intelligent scripts.

This chapter assumes that you are familiar with basic metadata objects, such as `sys.tables` and `sys.databases`. Instead, this chapter will focus on interrogating some less commonly known metadata objects that can prove very useful to DBAs. It will demonstrate how these metadata objects can be used to play a key role in automating maintenance routines. It is important to remember, however, that a discussion of all metadata objects available within SQL Server would be worthy of several volumes in its own right. Therefore, I suggest that you do your own research on SQL Server metadata and identify objects that will prove useful to your individual environment. A good place to start is the Dynamic Management Views and Functions page on MSDN, which can be found at [`https://msdn.microsoft.com/en-us/library/ms188754.aspx`](https://msdn.microsoft.com/en-us/library/ms188754.aspx).

Tip

This chapter will focus on writing the maintenance routines themselves. In Chapter [10](#395785_2_En_10_Chapter.xhtml), we will then focus on how we can centralize maintenance, across the SQL Server estate, so that we have a single code base to maintain.

## Creating Intelligent Routines

The following sections will demonstrate how SQL Server metadata can be used, in conjunction with PowerShell, to write intelligent routines that can be used to make your enterprise more consistent and better performing, with less DBA effort. We will discuss using Query Store metadata, dynamic index rebuilds, and enforcing policies that cannot be enforced using Policy-Based Management.

### Remove Ad Hoc Query Plans

Ad hoc query plans consume memory and can be of limited use. It is a good idea to remove ad hoc query plans if they are not being reused. The query in Listing [7-1](#395785_2_En_7_Chapter.xhtml#PC4) demonstrates how Query Store metadata can be used to clear unwanted ad hoc query plans from the cache.

The first part of the script runs a query against the `sys.databases` catalog view to return a list of databases on the instance. The Query Store can be enabled or disabled at the database level and cannot be enabled on either `master` or `TempDB` system databases. In SQL Server 2022, Query Store is enabled by default, but previously it was disabled by default. Therefore, the query filters out databases with a `database_id` of less than three and any databases where Query Store is disabled.

TWO WAYS TO PROCESS QUERY RESULTS

The output of Invoke-DbaQuery command is an object (or an array of objects) with a number of properties. There is a property for each column in the query result, as well as some additional properties.

Therefore, to use this result in further processing, the desired property will need to be extracted. This can be done in two different ways, each with pros and cons:

1.  Immediately – By piping the results into a Select-Object cmdlet, such as

    ```
    $databases = Invoke-DbaQuery @GetDatabaseParams | Select-Object -ExpandProperty Name
    ```

    This keeps $database simple; it ends up just an array of database names

2.  At the point of usage – By using a foreach loop to iterate through each object, such as

    ```
    foreach ($database in $databases.Name) { ... }
    ```

This approach means that at any point later you can make use of all the data returned by the query, including the metadata.

When PowerShell displays objects, it does not always show all their properties by default. To see all properties of an object, you can pipe it into Select-Object * such as

```
Invoke-DbaQuery @GetDatabaseParams | Select-Object *
```

The second part of the script passes the database names into a `foreach` loop and executes a SQL script against each one. This technique provides a useful mechanism for running the same command against every database in an instance, and I would highly recommend this approach, which uses the `FOR XML PATH('')` technique, described in Chapter [1](#395785_2_En_1_Chapter.xhtml), instead of the undocumented `sp_MSForeachdb` system stored procedure, which can be unreliable. We could also scale this script to run across all instances in the enterprise, by using the techniques discussed in Chapter [10](#395785_2_En_10_Chapter.xhtml).

```
$GetDatabaseParams = @{
SqlInstance = "localhost"
Query       = "
SELECT name
FROM sys.databases
WHERE database_id > 2
AND is_query_store_on = 1
"
}
$databases = Invoke-DbaQuery @GetDatabaseParams | Select-Object -ExpandProperty Name
$params = @{
SqlInstance = "localhost"t
Query       = "
DECLARE @SQL NVARCHAR(MAX)
SELECT @SQL = (
SELECT 'EXEC sp_query_store_remove_query '
+ CAST(qsq.query_id AS NVARCHAR(6)) + ';' AS [data()]
FROM sys.query_store_query_text AS qsqt
JOIN sys.query_store_query AS qsq
ON qsq.query_text_id = qsqt.query_text_id
JOIN sys.query_store_plan AS qsp
ON qsp.query_id = qsq.query_id
JOIN sys.query_store_runtime_stats AS qsrs
ON qsrs.plan_id = qsp.plan_id
GROUP BY qsq.query_id
HAVING SUM(qsrs.count_executions) = 1
AND MAX(qsrs.last_execution_time) < DATEADD (HH, -24, GETUTCDATE())
ORDER BY qsq.query_id
FOR XML PATH('')
) ;
EXEC(@SQL) ;
"
}
foreach ($database in $databases) {
Invoke-DbaQuery @params -Database $database
}
Listing 7-1
Removing Ad Hoc Query Plans
```

The query that is run against each database uses the `data()` method approach to iteration, to build a string that can be executed using dynamic SQL, before finally executing the script, using the `EXEC` statement. The metadata objects used by the query are described below.

The `sys.query_store_query_text` catalog view exposes the text and handle of each T-SQL statement in the store. It returns the columns detailed in Table [7-1](#395785_2_En_7_Chapter.xhtml#Tab1).

Table 7-1

Columns Returned by `sys.query_store_query_text`

| Column | Description |
| --- | --- |
| `query_text_id` | The primary key |
| `query_sql_text` | The T-SQL text of the query |
| `statement_sql_handle` | The SQL handle of the query |
| `is_part_of_encrypted_module` | Whether the SQL text is part of an encrypted programmable object |
| `has_restricted_text` | Whether the SQL text includes sensitive information that cannot be displayed, such as a password |

The `sys.query_store_query` catalog view joins to the `sys.query_store_query_text` catalog view on the `query_text_id` column in each view. It also joins to `sys.query_context_settings` on the `context_settings_id` column in each object. `sys.query_store_query` returns details of the queries, including aggregated statistics. The columns returned are detailed in Table [7-2](#395785_2_En_7_Chapter.xhtml#Tab2).

Table 7-2

Columns Returned by `sys.query_store_query`

| Column | Description |
| --- | --- |
| `query_id` | The primary key |
| `query_text_id` | Foreign key that joins to `sys.query_store_query_text` |
| `context_settings_id` | Foreign key that joins to `sys.query_context_settings` |
| `object_id` | The object ID of the programmable object, of which the query is a part. `0` represents ad hoc SQL |
| `batch_sql_handle` | The ID of the batch of which the query is a part. This field is only populated when the query uses table variables or temp tables |
| `query_hash` | The query tree represented as a Zobrist hash value |
| `is_internal_query` | Indicates if the query is a user or internal query:• `0` indicates a user query• `1` indicates an internal query |
| `query_parameterization_type` | The parameterization used for the query:• `0` indicates none• `1` indicates user parameterization• `2` indicates simple parameterization• `3` indicates forced parameterization |
| `query_parameterization_type_desc` | Description of the parameterization type |
| `initial_compile_start_time` | The date and time that the query was compiled for the first time |
| `last_compile_start_time` | The data and time of the most recent compilation of the query |
| `last_execution_time` | The date and time of the most recent execution of the query |
| `last_compile_batch_sql_handle` | The handle of SQL batch of which the query was a part on the most recent occurrence of the query being executed |
| `last_compile_batch_offset_start` | The offset of the start of the query from the beginning of the batch that is represented by `last_compile_batch_sql_handle` |
| `last_compile_batch_offset_end` | The offset of the end of the query from the beginning of the batch that is represented by `last_compile_batch_sql_handle` |
| `count_compiles` | A count of how many times that the query has been compiled |
| `avg_compile_duration` | The average length of time that it has taken the query to compile, recorded in microseconds |
| `last_compile_duration` | The duration of the most recent compilation of the query, recorded in microseconds |
| `avg_bind_duration` | The average time it takes to bind the query. Binding is the process of resolving every table and column name to an object in the system catalog and creating an algebrization tree |
| `last_bind_duration` | The time it took to bind the query on its most recent execution |
| `avg_bind_cpu_time` | The average CPU time that has been required to bind the query |
| `last_bind_cpu_time` | The amount of CPU time required to bind the query on its most recent compilation |
| `avg_optimize_duration` | The average length of time that the query optimizer takes to generate candidate query plans and select the most efficient |
| `last_optimize_duration` | The time it took to optimize the query during its most recent compilation |
| `avg_optimize_cpu_time` | The average CPU time that has been required to optimize the query |
| `last_optimize_cpu_time` | The amount of CPU time that was required to optimize the query on its most recent compilation |
| `avg_compile_memory_kb` | The average amount of memory used to compile the query |
| `last_compile_memory_kb` | The amount of memory used to compile the query during its most recent compilation |
| `max_compile_memory_kb` | The largest amount of memory that has been required on any compilation of the query |
| `is_clouddb_internal_query` | Indicates if the query is a user query or generated internally. This column only applies to the Azure SQL Database. It will always return `0` if the query is not run against the Azure SQL Database. When run on the Azure SQL Database• `0` indicates a user query• `1` indicates an internal query |

The `sys.query_store_plan` catalog view exposes details about the execution plans. The view joins to `sys.query_store_query` catalog view on the `query_id` column in each table. The columns returned by this view are detailed in Table [7-3](#395785_2_En_7_Chapter.xhtml#Tab3).

Table 7-3

Columns Returned by `sys.dm_query_store_plan`

| Column | Description |
| --- | --- |
| `plan_id` | The primary key |
| `query_id` | Foreign key that joins to `sys.query_store_query` |
| `plan_group_id` | The ID of the plan group that the plan is part of. This is only applicable to queries involving a cursor, as they require multiple plans |
| `engine_version` | The version of the database engine that was used to compile the plan |
| `compatibility_level` | The compatibility level of the database that the query references |
| `query_plan_hash` | The plan represented in an MD5 hash |
| `query_plan` | The XML representation of the query plan |
| `is_online_index_plan` | Indicates if the plan was used during an online index rebuild:• `0` indicates that it was not• `1` indicates that it was |
| `is_trivial_plan` | Indicates if the optimizer regarded the plan as trivial:• `0` indicates that it is not a trivial plan• `1` indicates that it is a trivial plan |
| `is_parallel_plan` | Indicates if the plan is parallelized:• `0` indicates that it is not a parallel plan• `1` indicates that it is a parallel plan |
| `is_forced_plan` | Indicates if the plan is forced:• `0` indicates it is not forced• `1` indicates that it is forced |
| `is_natively_compiled` | Indicates if the plan includes natively compiled stored procedures:• `0` indicates that it does not• `1` indicates that it does |
| `force_failure_count` | Indicates the number of times an attempt to force the plan has failed |
| `last_force_failure_reason` | The reason code for the failure in forcing the plan. Further details can be found in Table [7-4](#395785_2_En_7_Chapter.xhtml#Tab4) |
| `last_force_failure_reason_desc` | Text describing the reason for the failure in forcing the plan. Further details can be found in Table [7-4](#395785_2_En_7_Chapter.xhtml#Tab4) |
| `count_compiles` | A count of how many times that the query has been compiled |
| `initial_compile_start_time` | The date and time that the query was compiled for the first time |
| `last_compile_start_time` | The date and time of the most recent compilation of the query |
| `last_execution_time` | The date and time of the most recent end of the execution of the query |
| `avg_compile_duration` | The average length of time that it has taken the query to compile, recorded in microseconds |
| `last_compile_duration` | The duration of the most recent compilation of the query, recorded in microseconds |
| `plan_forcing_type` | Defines the plan forcing type, where 0 means none, 1 means manual, and 2 means auto |
| `plan_forcing_type_desc` | Provides the description associated with the plan_force_type |
| `has_compile_replay_script` | A flag which specifies if there is an optimized replay script associated with the plan |
| `is_optimized_plan_forcing_disabled` | A flag that specifies if optimized plan forcing was disabled for the plan |
| `plan_type` | The type of plan, where 0 means a compiled plan, 1 means a dispatcher plan, and 2 means a query variant plan |
| `plan_type_desc` | Provides the description associated with the plan_type column |

If plan forcing fails, then `last_force_failure_reason` and `last_force_failure_reason_desc` columns describe the reasons for the failure. Table [7-4](#395785_2_En_7_Chapter.xhtml#Tab4) defines the mapping between codes and reasons.

Table 7-4

Plan Forcing Failure Reasons

| Reason Code | Reason Text | Description |
| --- | --- | --- |
| `0` | `N/A` | No failure |
| `8637` | `ONLINE_INDEX_BUILD` | The query attempted to update a table while an index on the table was being rebuilt |
| `8683` | `INVALID_STARJOIN` | The star join included in the plan is invalid |
| `8684` | `TIME_OUT` | While searching for the force plan, the optimizer exceeded the threshold for allowed operations |
| `8689` | `NO_DB` | The plan references a database that does not exist |
| `8690` | `HINT_CONFLICT` | The plan conflicts with a query hint. For example, `FORCE_INDEX(0)` is used on a table with a clustered index |
| `8694` | `DQ_NO_FORCING_SUPPORTED` | The plan conflicts with the use of full-text queries or distributed queries |
| `8698` | `NO_PLAN` | Either the optimizer could not produce the forced plan, or it could not be verified |
| `8712` | `NO_INDEX` | An index used by the plan does not exist |
| `8713` | `VIEW_COMPILE_FAILED` | There is an issue with an indexed view that the plan uses |
| `3617` | `COMPILATION_ABORTED_BY_CLIENT` | The client aborted the query before compilation completed |
| `8675` | `OPTIMIZATION_REPLAY_FAILED` | There was a failure when executing the optimization replay script |
| Other values | `GENERAL_FAILURE` | An error not covered by the other codes |

The `sys.query_store_runtime_stats` catalog view exposes the aggregated runtime statistics for each query plan. The view joins to `sys_query_store_plan` using the `plan_id` column in each table and joins to the `sys.query_store_runtime_stats_interval` catalog view, using the `runtime_stats_interval_id` column in each object. The columns returned by the catalog view are within the aggregation interval. Important columns are described in Table [7-5](#395785_2_En_7_Chapter.xhtml#Tab5), but for brevity, not all are included.

Table 7-5

Columns Returned by `sys.query_store_runtime_stats`

| Column | Description |
| --- | --- |
| `runtime_stats_id` | The primary key |
| `plan_id` | Foreign key, joining to `sys.query_store_plan` |
| `runtime_stats_interval_id` | Foreign key, joining to `sys.query_store_runtime_stats_interval` |
| `execution_type` | Specifies the execution status:• `0` indicates a regular execution that finished successfully• `3` indicates the client aborted the execution• 4 indicates that the execution was aborted because of an exception |
| `execution_type_desc` | A textual description of the `execution_type` column |
| `first_execution_time` | The end date and time that the plan was first executed |
| `last_execution_time` | The end date and time of the most recent execution of the plan |
| `count_executions` | The number of times that the plan has been executed |
| `avg_duration` | The average length of time that the query has taken to execute, expressed in microseconds |
| `last_duration` | The time that the query took to execute on its most recent execution, expressed in microseconds |
| `min_duration` | The shortest length of time that the plan has taken to execute, expressed in microseconds |
| `max_duration` | The longest amount of time that the plan has taken to execute, expressed in microseconds |
| `stdev_duration` | The standard deviation of time spent executing the plan, expressed in microseconds |
| `avg_cpu_time` | The average amount of CPU time used by executions of the plan, expressed in microseconds |
| `last_cpu_time` | The amount of CPU time used by the most recent execution of the plan, expressed in microseconds |
| `min_cpu_time` | The minimum amount of CPU time used by any execution of the plan, expressed in microseconds |
| `max_cpu_time` | The maximum amount of CPU time used by any execution of the plan, expressed in microseconds |
| `stdev_cpu_time` | The standard deviation of CPU time used by executions of the plan, expressed in microseconds |
| `avg_logical_io_reads` | The average number of logical reads required by the plan |
| `last_logical_io_reads` | The number of logical reads required by the most recent execution of the plan |
| `min_logical_io_reads` | The smallest number of logical reads required by any execution of the plan |
| `max_logical_io_reads` | The maximum number of logical reads required by any execution of the plan |
| `stdev_logical_io_reads` | The standard deviation of logical reads required by executions of the plan |
| `avg_logical_io_writes` | The average number of logical writes required by the plan |
| `last_logical_io_writes` | The number of logical writes required by the most recent execution of the plan |
| `min_logical_io_writes` | The smallest number of logical writes required by any execution of the plan |
| `max_logical_io_writes` | The maximum number of logical writes required by any execution of the plan |
| `stdev_logical_io_writes` | The standard deviation of logical writes required by executions of the plan |
| `avg_physical_io_reads` | The average number of reads from disk required by the plan |
| `last_physical_io_reads` | The number of reads from disk required by the most recent execution of the plan |
| `min_physical_io_reads` | The smallest number of reads from disk required by any execution of the plan |
| `max_physical_io_reads` | The maximum number of reads from disk required by any execution of the plan |
| `stdev_physical_io_reads` | The standard deviation of reads from disk required by executions of the plan |
| `avg_clr_time` | The average CLR (Common Language Runtime) time required by executions of the plan, expressed in microseconds |
| `last_clr_time` | The CLR time required on the most recent execution of the plan, expressed in microseconds |
| `min_clr_time` | The smallest amount of CLR time required by any execution of the plan, expressed in microseconds |
| `max_clr_time` | The longest CLR time spent by any execution of the plan, expressed in microseconds |
| `stdev_clr_time` | The standard deviation of CLR time required by executions of the plan, expressed in microseconds |
| `avg_dop` | The average degree of parallelism used by execution of the plan |
| `last_dop` | The degree of parallelism used on the most recent execution of the plan |
| `min_dop` | The lowest degree of parallelism used by any execution of the plan |
| `max_dop` | The largest degree of parallelism used by any execution of the plan |
| `stdev_dop` | The standard deviation of the degree of parallelism used by executions of the plan |
| `avg_query_max_used_memory` | The average number of memory grants issued for any execution of the plan, expressed as a count of 8KB pages* |
| `last_query_max_used_memory` | The number of memory grants issued during the most recent execution of the plan, expressed as a count of 8KB pages* |
| `min_query_max_used_memory` | The smallest number of memory grants issued for any execution of the plan, expressed as a count of 8KB pages* |
| `max_query_max_used_memory` | The largest amount of memory grants issued for any execution of the plan, expressed as a count of 8KB pages* |
| `stdev_query_max_used_memory` | The standard deviation of memory grants issued during the executions of the plan, expressed as a count of 8KB pages* |
| `avg_rowcount` | The average number of rows affected by executions of the plan |
| `last_rowcount` | The number of rows affected by the most recent execution of the plan |
| `min_rowcount` | The minimum number of rows affected by any execution of the plan |
| `max_rowcount` | The largest number of rows affected by any execution of the plan |
| `stdev_rowcount` | The standard deviation of rows affected, across executions of the plan |

*Returns *0* for queries using natively compiled stored proceduresAll reads and writes are expressed as number of 8KB pages

### Dynamic Index Rebuilds

SQL Server does not provide out-of-the-box functionality for intelligently rebuilding indexes. If you use a Maintenance Plan to rebuild or reorganize all indexes in your database, every index will be rebuilt or reorganized, whether it needs it or not. Unfortunately, this can be a resource-intensive and time-consuming process, and it may be that only a fraction of your indexes need to be rebuilt.

If you have read Chapters [4](#395785_2_En_4_Chapter.xhtml) and [5](#395785_2_En_5_Chapter.xhtml), you will have noticed that we use a dynamic index rebuild script. This works around this issue, so you can use index-related metadata to evaluate the fragmentation level in each index and take the appropriate action (rebuild, reorganize, or leave alone). As mentioned in previous chapters, we will take the time here to explain how the script works.

The script uses the `sys.dm_db_index_physical_stats` dynamic management function. This function returns fragmentation statistics for indexes, with one row returned for every level of every index within scope (based on the input parameters). The parameters required to call this function are detailed in Table [7-6](#395785_2_En_7_Chapter.xhtml#Tab6).

Table 7-6

Parameters Required by `sys.dm_db_index_physical_stats`

| Parameter | Description |
| --- | --- |
| `Database_ID` | The ID of the database that the function will run against. If you do not know it, you can pass in `DB_ID('DatabaseName')` |
| `Object_ID` | The Object ID of the table that you want to run the function against. If you do not know it, pass in `OBJECT_ID('TableName')`. Pass `NULL` to run the function against all tables in the database |
| `Index_ID` | The index ID of the index you want to run the function against. `1` is always the ID of a table’s clustered index. Pass `NULL` to run the function against all indexes on the table |
| `Partition_Number` | The ID of the partition that you want to run the function against. Pass `NULL` if you want to run the function against all partitions or if the table is not partitioned |
| `Mode` | Acceptable values are `LIMITED`, `SAMPLED`, or `DETAILED`:• `LIMITED` only scans the non-leaf levels of an index• `SAMPLED` scans 1% of pages in the table, unless the table or partition has 10,000 pages or fewer, in which case `DETAILED` mode is used• `DETAILED` mode scans 100% of the pages in the table. For very large tables, `SAMPLED` is often preferred, due to the length of time it can take to return data in `DETAILED` mode |

Tip

This book assumes that you are familiar with B-Tree indexes and fragmentation (both internal and external). If you require a refresher, full details can be found in the Apress title *Pro SQL Server 2022 Administration*, which can be found at link.springer.com/book/10.1007/978-1-4842-8864-1, or *100 SQL Server Mistakes and How to Avoid Them* at manning.com/books/100-sql-server-mistakes-and-how-to-avoid-them.

The columns returned by `sys.dm_db_index_physical_stats` are detailed in Table [7-7](#395785_2_En_7_Chapter.xhtml#Tab7).

Table 7-7

Columns Returned by `sys.dm_db_index_physical_stats`

| Column | Description |
| --- | --- |
| `database_id` | The database ID of the database, in which the index resides |
| `object_id` | The object ID of the table or indexed view to which the index is associated |
| `index_id` | The ID of the index within the table or indexed view. The ID is only unique within its associated parent object. `0` is always the index ID of a heap, and `1` is always the index ID of a clustered index |
| `partition_number` | The partition number. This always returns `1` for non-partitioned tables and indexed views |
| `index_type_desc` | Text defining the index types. Possible return values are as follows:• `HEAP`• `CLUSTERED INDEX`• `NONCLUSTERED INDEX`• `PRIMARY XML INDEX`• `EXTENDED INDEX`• `XML INDEX` (Referring to a secondary XML index)There are also some index types used for internal use only. These are as follows:• `COLUMNSTORE MAPPING INDEX`• `COLUMNSTORE DELETEBUFFER INDEX`• `COLUMNSTORE DELETEBITMAP INDEX` |
| `alloc_unit_type_desc` | Text describing the type of allocation unit that the index level resides in |
| `index_depth` | The number of levels that the index consists of. If `1` is returned, the index is either a heap or LOB data or row overflow data |
| `index_level` | The level of the index that is represented by the row returned |
| `avg_fragmentation_in_percent` | The average percentage of external fragmentation within the index level (lower is better) |
| `fragment_count` | The number of fragments within the index level |
| `avg_fragment_size_in_pages` | The average size of each fragment, represented as a count of 8KB pages |
| `page_count` | For in-row data, the page count represents the number of pages at the level of the index that is represented by the rowFor a heap, the page count represents the number of pages within the `IN_ROW_DATA` allocation unitFor LOB or row overflow data, the page count represents the number of pages within the allocation unit |
| `avg_page_space_used_in_percent` | The average amount of internal fragmentation within the index level that the row represents (higher is better…usually) |
| `record_count` | For in-row data, the record count represents the number of records at the level of the index that is represented by the rowFor a heap, the record count represents the number of records within the `IN_ROW_DATA` allocation unitFor LOB or row overflow data, the record count represents the number of records within the allocation unit |
| `ghost_record_count` | A count of logically deleted rows that are awaiting physical deletion by the ghost cleanup task |
| `version_ghost_record_count` | The number of rows that have been retained by a transaction using the snapshot isolation level |
| `min_record_size_in_bytes` | The size of the smallest record, expressed in bytes |
| `max_record_size_in_bytes` | The size of the largest record, expressed in bytes |
| `avg_record_size_in_bytes` | The average record size, expressed in bytes |
| `forwarded_record_count` | The number of records that have forwarding pointers to other locations. Only applies to heaps. If the object is an index, `NULL` is returned |
| `compressed_page_count` | A count of the compressed pages within the index level represented by the row |
| `hobt_id` | This column only applies to columnstore indexes. It represents the ID of the heaps or B-Trees that are used to track internal columnstore data |
| `column_store_delete_buffer_state` | This column only applies to columnstore indexes. It represents the state of the columnstore delete buffer |
| `column_store_delete_buff_state_desc` | This column only applies to columnstore indexes. It is a textual description of the value returned in the `column_store_delete_buffer_state` column |
| `version_record_count` | The number of row version records being maintained for use with Accelerated Database Recovery |
| `inrow_version_record_count` | The number of version records kept in-row for use with Accelerated Database Recovery |
| `inrow_diff_version_record_count` | The number of version records kept in the form of differences for Accelerated Database Recovery |
| `total_inrow_version_payload_size_in_bytes` | The size of the indexes in-row version records. Expressed in bytes |
| `offrow_regular_version_record_count` | The number of version records kept outside of the data row |
| `offrow_long_term_version_record_count` | The number of records kept in the online index version store |

The script in Listing [7-2](#395785_2_En_7_Chapter.xhtml#PC5) demonstrates how you can use this dynamic management function to determine which indexes have more than 25% external fragmentation and rebuild only those indexes.

```
$GetDatabaseParams = @{
SqlInstance = "localhost"
Query       = "
SELECT name
FROM sys.databases
WHERE database_id > 4
"
}
$databases = Invoke-DbaQuery @GetDatabaseParams | Select-Object -ExpandProperty Name
$params = @{
SqlInstance = "localhost"
Query       = "
DECLARE @SQL NVARCHAR(MAX)
SET @SQL = (
SELECT 'ALTER INDEX '
+ QUOTENAME(i.name)
+ ' ON ' + s.name
+ '.'
+ QUOTENAME(OBJECT_NAME(i.object_id))
+ ' REBUILD ; '
FROM sys.dm_db_index_physical_stats(DB_ID(),NULL,NULL,NULL,'DETAILED') ps
INNER JOIN sys.indexes i
ON ps.object_id = i.object_id
AND ps.index_id = i.index_id
INNER JOIN sys.objects o
ON ps.object_id = o.object_id
INNER JOIN sys.schemas s
ON o.schema_id = s.schema_id
WHERE index_level = 0
AND avg_fragmentation_in_percent > 25
FOR XML PATH('')
) ;
EXEC(@SQL) ;
"
}
foreach ($database in $databases) {
Invoke-DbaQuery @params -Database $database
}
Listing 7-2
Dynamic Index Rebuilds
```

This script uses the XML `data()` method technique to build and execute a string that contains `ALTER INDEX` statements for every index in the specified database that has more than 25% external fragmentation at the leaf level. The script joins `sys.dm_db_index_physical_stats` to `sys.schemas` to obtain the name of the schema that contains the index. There is no direct join between these objects, so the join is implemented through an intermediate catalog view (`sys.objects`). The script also joins to `sys.indexes` to retrieve the `object_id`, which is then passed into the `OBJECT_NAME()` function to determine the name of the index.

### Enforcing Policies

Policy-Based Management (PBM) is a great tool for ensuring that your policies are met across the enterprise. It does have limitations, however. Specifically, some facets only allow you to report on policy violation but not actually to prevent the change that will cause the violation. Therefore, many DBAs will choose to enforce some of their key policies by using custom scripting.

Tip

A PBM Facet is a set of properties that are related to an area of SQL Server Management.

`xp_cmdshell` is a system-stored procedure that allows you to run operating system commands from within a SQL Server instance. There seems to be some debate between DBAs about whether this is a security hole or not, but I have always found it cut and dried. The argument for `xp_cmdshell` not being a security risk generally revolves around “Only my team and I are allowed to use it.” But what if the instance is compromised by a malicious user? In this case, you really do not want to reduce the number of steps hackers have to take before they can start spreading their attack out to the network.

Caution

If your organization wishes to comply with CIS hardening standards for SQL Server, then `xp_cmdshell` must be disabled.

PBM can report on `xp_cmdshell`, but it cannot prevent it being enabled. You can, however, create a PowerShell script that will check for `xp_cmdshell` being enabled and disable it, if required. The script can then be scheduled to run on a regular basis.

The script in Listing [7-3](#395785_2_En_7_Chapter.xhtml#PC6) demonstrates how to discover if the feature is incorrectly configured, by checking its status in the `sys.configurations` catalog view. It will then use `sp_configure` to reconfigure the setting. There are two calls of `sp_configure`, because `xp_cmdshell` is an advanced option, and the `show advanced options` property must be configured before the setting can be changed. The `RECONFIGURE` command forces the change.

Tip

In Chapter [10](#395785_2_En_10_Chapter.xhtml), we discuss techniques to allow useful scripts such as these to be applied to enterprise from a central location. However, for this particular configuration, you may wish to use the techniques discussed in Chapter [9](#395785_2_En_9_Chapter.xhtml) to make this part of your standard build, which will be applied through Desired State Configuration (DSC), or your configuration management tool of choice.

There are two commands to reconfigure the instance, following a change made by `sp_configure`. The first is `RECONFIGURE`. The second is `RECONFIGURE WITH OVERRIDE`. `RECONFIGURE` will change the running value of the setting, as long as the newly configured value is regarded as “sensible” by SQL Server. For example, `RECONFIGURE` will not allow you to disable contained databases when they exist on the instance. If you use `RECONFIGURE WITH OVERRIDE`, however, this action will be allowed, even though your contained databases will no longer be accessible. Even with this command, however, SQL Server will still run checks to ensure that the value you have entered is between the allowable minimum and maximum values for the setting. It will also not allow you to perform any operations that will cause fatal errors in the database engine. As an example of this, the procedure will not allow you to configure the `Min Server Memory (MB)` setting to be higher than the `Max Server Memory (MB)` setting.

```
$params = @{
SqlInstance = "localhost"
Query       = "
--Check if xp_cmdshell is enabled
IF (
SELECT value_in_use
FROM sys.configurations
WHERE name = 'xp_cmdshell'
) = 1
BEGIN
--Turn on advanced options
EXEC sp_configure 'show advanced options', 1 ;
RECONFIGURE
--Turn off xp_cmdshell
EXEC sp_configure 'xp_cmdshell', 0 ;
RECONFIGURE
END
"
}
Invoke-DbaQuery @params
Listing 7-3
Disable xp_cmdshell for an Instance
```

This approach can be used to configure many aspects of your enterprise’s surface area. In SQL Server 2022, the features that can be configured with this methodology are detailed in Table [7-8](#395785_2_En_7_Chapter.xhtml#Tab8).

Table 7-8

Settings That Can Be Changed with `sp_configure`

| Setting | Description |
| --- | --- |
| `recovery interval (min)` | Maximum recovery interval in minutes |
| `allow updates` | Allows updates to system tables |
| `user connections` | Number of user connections allowed |
| `Locks` | Number of locks for all users |
| `open objects` | Number of open database objects |
| `fill factor (%)` | Default fill factor percentage |
| `disallow results from triggers` | Disallows returning results from triggers |
| `nested triggers` | Allows triggers to be invoked within triggers |
| `server trigger recursion` | Allows recursion for server-level triggers |
| `remote access` | Allows remote access |
| `default language` | Default language |
| `cross db ownership chaining` | Allows cross `db` ownership chaining |
| `max worker threads` | Maximum worker threads |
| `network packet size (B)` | Network packet size |
| `show advanced options` | Shows advanced options |
| `remote proc trans` | Creates Distributed Transaction Coordinator (DTC) transaction for remote procedures |
| `c2 audit mode` | `c2` audit mode |
| `default full-text language` | Default full-text language |
| `two digit year cutoff` | Two-digit year cutoff |
| `index create memory (KB)` | Memory for index create sorts (KBytes) |
| `priority boost` | Priority boost |
| `remote login timeout (s)` | Remote login timeout |
| `remote query timeout (s)` | Remote query timeout |
| `cursor threshold` | Cursor threshold |
| `set working set size` | Sets working set size |
| `user options` | User options |
| `affinity mask` | Affinity mask |
| `max text repl size (B)` | Maximum size of a text field in replication |
| `media retention` | Tape retention period in days |
| `cost threshold for parallelism` | Cost threshold for parallelism |
| `max degree of parallelism` | Maximum degree of parallelism |
| `min memory per query (KB)` | Minimum memory per query (KBytes) |
| `query wait (s)` | Maximum time to wait for query memory (s) |
| `min server memory (MB)` | Minimum size of server memory (MB) |
| `max server memory (MB)` | Maximum size of server memory (MB) |
| `query governor cost limit` | Maximum estimated cost allowed by query governor |
| `lightweight pooling` | User mode scheduler uses lightweight pooling |
| `scan for startup procs` | Scan for startup stored procedures |
| `affinity64 mask` | Configures the Affinity64 mask (which schedulers can be used by the instance for servers with a large number of cores) |
| `affinity I/O mask` | Configures the Affinity I/O mask (which schedulers use for I/O operations) |
| `affinity64 I/O mask` | Configures the Affinity64 I/O mask (which schedulers can be used by the instance for I/O) |
| `transform noise words` | Transforms noise words for full-text query |
| `precompute rank` | Uses precomputed rank for full-text query |
| `PH timeout (s)` | DB connection timeout for full-text protocol handler (s) |
| `clr enabled` | CLR user code execution enabled in the server |
| `max full-text crawl range` | Maximum crawl ranges allowed in full-text indexing |
| `ft notify bandwidth (min)` | Number of reserved full-text notification buffers |
| `ft notify bandwidth (max)` | Maximum number of full-text notification buffers |
| `ft crawl bandwidth (min)` | Number of reserved full-text crawl buffers |
| `ft crawl bandwidth (max)` | Maximum number of full-text crawl buffers |
| `default trace enabled` | Enables or disables the default trace |
| `blocked process threshold (s)` | Blocked process reporting threshold |
| `in-doubt xact resolution` | Recovery policy for DTC transactions with unknown outcome |
| `remote admin connections` | Dedicated Admin Connections are allowed from remote clients |
| `common criteria compliance enabled` | Common Criteria compliance mode enabled |
| `EKM provider enabled` | Enables or disables EKM provider |
| `backup compression default` | Enables compression of backups by default |
| `filestream access level` | Sets the `FILESTREAM` access level |
| `optimize for ad hoc workloads` | When this option is set, plan cache size is further reduced for a single-use ad hoc OLTP workload |
| `access check cache bucket count` | Defaults hash bucket count for the access check result security cache |
| `access check cache quota` | Defaults quota for the access check result security cache |
| `backup checksum default` | Enables checksum of backups by default |
| `automatic soft-NUMA disabled` | Automatic soft-Non-uniform Memory Access (NUMA) is enabled by default |
| `external scripts enabled` | Allows execution of external scripts |
| `Agent XPs` | Enables or disables Agent XPs |
| `Database Mail XPs` | Enables or disables Database Mail XPs |
| `SMO and DMO XPs` | Enables or disables Server Management Objects (SMO) and Distributed Management Objects (DMO) extended stored procedures (XPs) |
| `Ole Automation Procedures` | Enables or disables Ole Automation Procedures |
| `xp_cmdshell` | Enables or disables command shell |
| `Ad Hoc Distributed Queries` | Enables or disables Ad Hoc Distributed Queries |
| `Replication XPs` | Enables or disables Replication XPs |
| `contained database authentication` | Enables contained databases and contained authentication |
| `hadoop connectivity` | Configures SQL Server to connect to external Hadoop or Microsoft Azure storage blob data sources through PolyBase |
| `polybase network encryption` | Configures SQL Server to encrypt control and data channels when using PolyBase |
| `remote data archive` | Allows the use of the `REMOTE_DATA_ARCHIVE` data access for databases |
| `ADR cleaner retry timeout (min)` | Specifies, in minutes, the amount of time accelerated database recover will spend exclusively retrying object lock acquisition, before stopping its attempted sweep. Defaults to 15 minutes |
| `ADR Preallocation Factor` | Accelerated Database Recovery pre-allocates blocks of pages to the persistent version store (PVS) to improve performance. This setting controls the number of blocks of pages that are pre-allocated, with a default of 4 |
| `allow filesystem enumeration` | Can be configured as 0 or 1, with a default of 1\. When configured as 0, it prevents the operating system from being browsed from within SQL Server |
| `allow polybase export` | Can be configured as 0 or 1\. When enabled, it allows the INSERT operation to be performed against Hadoop external tables. The default is 0 |
| `clr strict security` | Can be configured as 0 or 1\. The default is enabled. When disabled, a CLR assembly created with a permission set of SAFE may be able to access external resources. You should ensure this setting remains enabled, as the ability to disable it is provided for backward compatibility only |
| `column encryption enclave type` | Specifies if a secure enclave is available. When configured as 0, there is no secure enclave. When configured as 1, SQL Server will attempt to initialize a VBS (virtualization-based secure enclave) |
| `tempdb metadata memory-optimized` | Specifies if memory-optimized metadata feature set should be enabled for TempDB |
| `version high part of SQL Server` | Sets the version high part of SQL Server that model database copied for |
| `version low part of SQL Server` | Sets the version low part of SQL Server that model database copied for |
| `Data processed daily limit in TB` | Specifies the SQL on-demand data processed daily limit in TB |
| `Data processed weekly limit in TB` | Specifies the SQL on-demand data processed weekly limit in TB |
| `Data processed monthly limit in TB` | Specifies the SQL on-demand data processed monthly limit in TB |
| `ADR Cleaner Thread Count` | Sets the max number of threads ADR cleaner can assign |
| `hardware offload enabled` | Can be configured as 0 or 1\. Enables or disables hardware offloading on the server |
| `hardware offload config` | Can be configured as 0 or 1\. Enables or disables hardware offload accelerator |
| `hardware offload mode` | Sets the hardware offload accelerator mode |
| `backup compression algorithm` | Configures default backup compression algorithm |
| `polybase enabled` | Allows SQL Server to connect to external data sources |
| `Suppress recovery model errors` | Returns warning instead of errors for unsupported ALTER DATABASE SET RECOVERY statements |
| `Openrowset auto_create_statistics` | Enables or disables automatic creation of statistics for openrowset sources |

The `sys.configurations` catalog view returns the columns detailed in Table [7-9](#395785_2_En_7_Chapter.xhtml#Tab9).

Table 7-9

Columns Returned by `sys.configurations`

| Column | Description |
| --- | --- |
| `configuration_id` | The primary key |
| `name` | The name of the setting |
| `value` | The currently configured value of the setting |
| `minimum` | The minimum value that can be configured for the setting |
| `maximum` | The maximum value that can be configured for the setting |
| `value_in_use` | The value that is currently in use. This can differ from the value column, if the setting has been changed but the instance has not been reconfigured or restarted |
| `description` | A description of the setting |
| `is_dynamic` | Specifies if the setting is dynamic:• `1` indicates that the setting can be applied using `RECONFIGURE`• `0` indicates that the instance must be restarted |
| `is_advanced` | Specifies if the setting can only be changed when show advanced options is set to `1`:• `0` indicates it is not an advanced option• `1` indicates that it is an advanced option |

The parameters accepted by `sp_configure` are detailed in Table [7-10](#395785_2_En_7_Chapter.xhtml#Tab10).

Table 7-10

Parameters Accepted by `sp_configure`

| Parameter | Description |
| --- | --- |
| `@configname` | The name of the setting to change |
| `@configvalue` | The value to be configured for the setting |

## Summary

Discussion of all metadata objects available within SQL Server would be worthy of several volumes in its own right. Each area of metadata opens up opportunities for you to automate maintenance routines within your environment, which should be the ambition of every DBA. The configurable options within `sp_configure` alone will give you a small taste of how vast the possibilities are. I strongly encourage you to explore SQL Server metadata and how you can use it to automate the mundane, within your own enterprise.

There are some classic opportunities which will stand out to DBAs in any organization, however, such as the ability to rebuild only those indexes which require it, instead of rebuilding all indexes in every maintenance window. `sys.dm_db_index_physical_stats` can be harnessed to provide a mechanism for achieving this.

DBAs should always seriously consider enabling Query Store on their databases, and in SQL Server 2022, this is the default option. You may even choose to create a maintenance routine, using some of the techniques you have learned in this book, to have a centralized script that will run against every database in your enterprise, to enable the feature. Once enabled, a raft of possibilities opens up, such as the example provided here, of removing ad hoc plans from your instances. Many more complex possibilities exist, and some of these can be harnessed by querying the XML, in which the raw Query Store data is stored. In some of these scenarios, you will want to use the `CROSS` and `OUTER APPLY` operators in conjunction with this to pull useful metadata in the most efficient way. The APPLY operator will be particularly useful when you wish to add query text to your results. Please refer to Chapter [1](#395785_2_En_1_Chapter.xhtml) for a primer on how to use these features.

Scripts utilizing a combination of PowerShell and T-SQL can also be used to enforce policies, such as hardening standards, in scenarios where Policy-Based Management can report but not enforce the required standards. This is especially helpful in large enterprises, where it is challenging to track multiple servers. Chapter [10](#395785_2_En_10_Chapter.xhtml) will discuss how we can centralize our maintenance routines. Alternatively, you could use the techniques discussed in Chapter [9](#395785_2_En_9_Chapter.xhtml) to make it part of your standard build, so that it is enforced through configuration management.

# 8. Building an Inventory Database

You may be wondering why a book discussing PowerShell automation for DBA includes a chapter on building an inventory database. The answer to this is simple – an inventory database is critical to the success of your automation strategy. Despite being a relatively small and simple database, an inventory database will act as your hub for orchestrating your automation efforts throughout the enterprise.

Even if your company has a full-fledged Configuration Management Database (CMDB), which includes databases, the astute DBA will still have their own inventory system. There are two reasons for this. Firstly, it allows a DBA to have more control over their own database, limiting the chances of the data becoming stale or dirty.

From the perspective of PowerShell and automation, however, having full control over the system also means that you can easily extend and enhance it, so that you can also easily grow and extend your automation solution. Because the database is a DBA-owned SQL Server database, as opposed to a hidden database in a large CMDB, which can often only be accessed via APIs, it allows you to use the database to control your automated maintenance.

Your automated build will keep the database up to date by inserting the details of every new SQL Server instance that is built in your enterprise. The database can then feed automated maintenance routines that you create, for example, providing a list of SQL Server instances and databases that should be iterated by maintenance routines, such as DBCC CHECKDB, or index rebuilds.

Centralized maintenance techniques will be discussed in more detail in Chapter [10](#395785_2_En_10_Chapter.xhtml). In this chapter, we will focus on designing and building the inventory database itself. We will start by discussing an appropriate platform design for the database, before discussing the logical and physical database design. Finally, we will explore how to create the database.

## Inventory Database Platform Design

Before designing the inventory database itself, you should first consider the platform requirements of the database, such as availability. For example, imagine that your organization has four data centers, dispersed across the UK and the United States, as well as a presence in Azure. This is depicted in Figure [8-1](#395785_2_En_8_Chapter.xhtml#Fig1). You have decided to place the inventory database on a management instance in the UK. What will happen in the event of the `UK Primary` data center losing connectivity to the rest of the network? The SQL Server instances in `UK` will continue to see the inventory database, but instances in the other three data centers will not. If your inventory database is used as a hub to control automation, this scenario could pose real issues. If you have all hands on deck, firefighting a data center outage and potentially invoking DR strategies for dozens of data-tier applications, the last thing that you probably want to be worrying about is users complaining of performance issues across the rest of the enterprise because their maintenance jobs have not been able to run.

![](images/395785_2_En_8_Chapter/395785_2_En_8_Fig1_HTML.jpg)

Flow chart illustrating a cloud computing infrastructure using Azure. The diagram shows two main regions: West US and UK South, each with managed servers and databases. West US and UK South are connected to their respective primary and secondary sites, labeled as US Primary Site, US Secondary Site, UK Primary Site, and UK Secondary Site, each containing managed servers. The flow chart represents the distribution and management of servers and databases across different geographical locations.

Figure 8-1

Example topology

The resolution to this problem lays in redundancy. Specifically, SQL Server AlwaysOn Availability Groups can be used to synchronize your inventory database between two locations. You could either build an AlwaysOn Availability Group on-premise, spanning two data centers, or as in this example, if the organization has a presence in public cloud, you could build an Availability Group across two regions in Azure.

If you choose to host your inventory database in Azure, you could build the Availability Group on IaaS (Infrastructure as a Service), or you could make use of Azure’s PaaS (Platform as a Service) or DBaaS (Database as a Service) offerings.

If you choose to use IaaS, then you will build VMs in Azure and be responsible for manually configuring the Availability Group yourself. If you choose to use a PaaS offering, then you would be looking toward Azure Managed Instances. Here, you do not work with the VM directly. Instead, you are buying a SQL Server instance as a service. This reduces the administrative overhead, as it provides automatic patching, version updates, and automatic backups, as well as entirely removing the Windows administration. Auto-failover groups can also be used to manage geographical dispersement of your inventory database across Azure regions.

Caution

Azure SQL Database may not be suitable for your inventory database; although you could potentially use Elastic Jobs, it would be easier to manage with a Managed Instance.

Azure SQL Database is a database as a service offering from Azure. With this option, you purchase a database. This removes the requirement to administer both the Windows operating system and the SQL Server instance entirely. In our case, however, Azure SQL Database may not be a good choice, because we want to use our database as a hub for automation, which means creating SQL Server Agent jobs. These are not supported in Azure SQL Database, as they are instance-level artifacts.

Tip

A full discussion of AlwaysOn Availability Groups is beyond the scope of this book. However, full details can be found in the Apress title *SQL Server 2022 AlwaysOn*, which can be found at link.springer.com/book/10.1007/978-1-4842-8864-1.

## Inventory Database Logical Design

When designing any database, the first task should be to determine what data you need to store. Do not think about table design yet, just the data that you will need – in a flat list. For database developers, this involves conversations with business analysts or business stakeholders to determine requirements. For an application developer, this task will usually be handled by their chosen framework, such as Entity Framework. The DBA who wishes to create an inventory database, however, will play the role of both the developer and business stakeholder. Below is a list of attributes that you may wish to record. Of course, this list should be tailored to your individual requirements.

*   `Server name`

*   `Instance name`

*   `Port`

*   `IP Address`

*   `Service account name and password`

*   `Authentication mode`

*   `sa account name and password`

*   `Cluster flag`

*   `Windows version`

*   `SQL Server version`

*   `DR Server name`

*   `DR instance name`

*   `DR technology`

*   `Target RPO`

*   `Target RTO`

*   `Instance classification`

*   `Server cores`

*   `Instance cores`

*   `Server RAM`

*   `Instance RAM`

*   `Virtual flag`

*   `Hypervisor`

*   `SQL Server Agent account name and password`

*   `Application owner`

*   `Application owner e-mail`

### Normalization

Now that we have identified the data (known as attributes) that we need to store, we have to group these attributes together into related groups. Each of these groups is called an entity, and the attributes are, in fact, attributes of the resulting entities.

The process for modeling the entities is known as *normalization*. This is a process that was invented by Edgar F. Codd in 1970 and has endured the test of time, although additional normal forms were added over the subsequent decade.

There are eight normal forms in total, which are listed as follows:

*   1NF (first normal form)

*   2NF (second normal form)

*   3NF (third normal form – this is the last of the original normal forms, as defined by Edgar Codd)

*   BCNF (Boyce-Codd normal form, also known as 3.5 normal form)

*   4NF (fourth normal form)

*   5NF (fifth normal form)

*   6NF (sixth normal form)

*   DKNF (domain key normal form)

The normal forms BCNF to DKNF deal with special circumstances, and the majority of the time, if a database is in 3NF, it will also be in DKNF. For this reason, the following sections will discuss the first three normal forms only.

The reason for modeling a database with normalization is to reduce data redundancy, meaning that every piece of data should be stored only once, with the exception of a unique key, which is used to identify the data. This is less about saving storage space (although this has obvious cost and performance benefits) and more about avoiding database anomalies that may be caused by having to update data in more than one place. It also avoids the complexities and overheads associated with avoiding these anomalies.

Tip

Normalization is only suitable for OLTP (Online Transactional Processing) databases. Data Warehouses and Data Marts should be denormalized for performance. Therefore, they should be modeled using techniques such as Kimball methodology. An ODS (Operational Data Store) will also have to be modeled differently, usually based on the heterogeneous sources that it unites.

The objectives of modeling data in 1NF are to ensure that values are atomic, that repeating groups of attributes are moved to separate entities, and that all attributes are dependent on a key (also known as the prime attribute). The objective of modeling data into 2NF is to ensure that all non-prime attributes are dependent on all prime attributes within the entity. The objective of modeling data into 3NF is to ensure that no non-prime attributes are dependent on any other non-prime attributes. I like to remember this by using the phrase “first, second, and third normal forms ensure that all attributes are dependent on the key, the whole key, and nothing but the key.”

#### 1NF

The first step in our normalization journey is to move the required attributes into first normal form (1NF). To do this, we must ensure that each attribute is atomic (holds only one piece of information) and that each attribute is dependent upon a key. In this phase, we will also break out repeating groups to new entities. So, first, let’s break out the repeating groups. We will do this by starting with the first attribute (`Server name`) and then asking the following question about each other attribute: “Can we have more than one per server?” Table [8-1](#395785_2_En_8_Chapter.xhtml#Tab1) answers this question for each attribute.

Table 8-1

Can There Be More Than One per Server Name?

| Attribute | More Than One per Server? |
| --- | --- |
| `Instance name` | Yes |
| `Port` | Yes |
| `IP Address` | Yes |
| `Service account name and password` | Yes |
| `Authentication mode` | Yes |
| `sa account name and password` | Yes |
| `Cluster flag` | No |
| `Windows version` | No |
| `SQL Server version` | No (technically possible, but against most corporate policies) |
| `DR Server` | Yes |
| `DR instance` | Yes |
| `DR technology` | Yes |
| `Target RPO` | Yes |
| `Target RTO` | Yes |
| `Instance classification` | Yes |
| `Server cores` | No |
| `Instance cores` | Yes |
| `Server RAM` | No |
| `Instance RAM` | Yes |
| `Virtual flag` | No |
| `Hypervisor` | No |
| `SQL Server Agent account name and password` | Yes |
| `Application owner` | No |
| `Application owner e-mail` | No |

Any attribute for which the answer to the question is no will remain within the same entity as `Server name` in this phase of normalization. Therefore, we know that our first entity will contain the following attributes:

*   `Server name`

*   `Cluster flag`

*   `Windows version`

*   `SQL Server version`

*   `Server cores`

*   `Server RAM`

*   `Virtual flag`

*   `Hypervisor`

*   `Application owner`

*   `Application owner e-mail`

We should give this entity a meaningful name, so we will call it `Server`. We should now ensure that each attribute in the `Server` entity is dependent on a key. A key should uniquely identify any row, and `Server name` will fulfill this requirement. Therefore, we will make `Server name` the key (prime) attribute. We can see that each attribute contains only a single value, meaning that we are also fulfilling the requirement to be autonomous.

If we look at the remaining attributes, we can see that they naturally fall together into two groups: `Instance` and `DR Server`. Therefore, we will break the rest of the attributes out to new entities that align with these repeating groups.

First, let’s look at the `Instance` entity, which will contain the following attributes:

*   `Instance name`

*   `Port`

*   `IP Address`

*   `Service account name and password`

*   `Authentication mode`

*   `sa account name and password`

*   `Instance classification`

*   `Instance cores`

*   `Instance RAM`

*   `SQL Server Agent account name and password`

We will have to add the `Server name` attribute to the `Instance` entity, so that we can link the entities together. In this entity, we can also easily identify three attributes that do not contain atomic values: `sa account name and password`, `Service account name and password`, and `SQL Server Agent account name and password`. Each of these attributes contains two values: the account name and the password. Therefore, the columns should be broken up, giving the following set of attributes:

*   `Instance name`

*   `Server name`

*   `Port`

*   `IP Address`

*   `Service account name`

*   `Service account password`

*   `Authentication mode`

*   `sa account name`

*   `sa password`

*   `Instance classification`

*   `Instance cores`

*   `Instance RAM`

*   `Additional services`

*   `SQL Server Agent account name`

*   `SQL Server Agent account password`

In regard to a key, all attributes can be uniquely identified by the `Instance name`. Therefore, an instance name will become the key of the entity.

Tip

I have seen some environments in which instances always follow the naming convention SQL001, SQL002, etc. This pattern is then repeated on every server. In this scenario, the instance name will not be unique and cannot be used as a key. Therefore, either IP address would have to be used, or you would have to create a composite key consisting of `Server name` and `Instance name`.

The DR entity will consist of the following attributes:

*   `DR Server`

*   `DR instance`

*   `DR technology`

*   `Target RPO`

*   `Target RTO`

All of the values are atomic, so there is no need for us to split out any attributes. In this entity, there is no single attribute that can uniquely identify all other attributes. We will, therefore, have to construct a composite key for this entity. This key will consist of `DR Server` and `DR instance`. Again, we move the key of the `Server` entity to the `DR Server entity`, so that we can link the entities together. Figure [8-2](#395785_2_En_8_Chapter.xhtml#Fig2) illustrates our completed 1NF entity model.

![](images/395785_2_En_8_Chapter/395785_2_En_8_Fig2_HTML.jpg)

Table image displaying three sections: "Server," "Instance," and "DR Server." The "Server" section lists attributes like Server Name, Clustered Flag, Windows Version, SQL Server Version, Server Cores, Server RAM, Virtual Flag, Hypervisor, Application Owner, and Application Owner E-Mail. The "Instance" section includes Instance Name, Server Name, Port, IP Address, Service Account Name, Service Account Password, Authentication Mode, sa Account Name, sa Account Password, Instance Classification, Instance Cores, Instance RAM, SQL Server Agent Account Name, and SQL Server Agent Account Password. The "DR Server" section lists DR Server Name, Server Name, DR Instance, DR Technology, Target RPO, and Target RTO.

Figure 8-2

1NF entities

#### 2NF

The next step in the normalization process will move our entities into second normal form (2NF). To be in 2NF, the entities must already be in 1NF, and you must also ensure that no non-key attribute (also known as a non-prime attribute) is dependent on only part of the entity’s key. If it is, you must split the attribute(s) out to an additional entity.

As the `Server` and `Instance` entities all have a single attribute key, we know that these entities are already in second normal form. We do, however, have to review the `DR Server` entity, as this entity has a composite key. We should look at each non-prime attribute in turn and identify if the attribute is dependent on the whole of the composite key or just a subset of the key. Table [8-2](#395785_2_En_8_Chapter.xhtml#Tab2) shows this analysis.

Table 8-2

Key Dependencies

| Non-prime Attribute | Key Dependencies | 2NF Compliant? |
| --- | --- | --- |
| `DR technology` | `DR Instance name` | No |
| `Target RPO` | `DR Instance name` | No |
| `Target RTO` | `DR Instance name` | No |

As you can see, all three of the non-prime attributes can be identified without knowing the name of the DR server. This means that these columns should all be moved to a separate entity, which we will call `DR Instance`.

The only attributes that will remain in the `DR Server` entity are the key attributes. This may seem a little odd, but it has happened to avoid a many-to-many relationship between the `Server` entity and the `DR Instance` entity and is perfectly valid design.

#### 3NF

The final stage of normalization that I will cover in this book is how to move entities into third normal form (3NF). In order to be in 3NF, the entity should already be in 2NF, and no non-prime attributes of an entity should be dependent on any other non-prime attributes. If we find an attribute that is dependent on a non-prime attribute, this is known as a transitive dependency, and the attribute should be moved to a separate entity.

In the `Server`, `DR Server`, and `DR Instance` entities, there are no transitive dependencies. In the `Instance` entity, however, there are three attributes that are dependent on non-prime attributes. These are listed in Table [8-3](#395785_2_En_8_Chapter.xhtml#Tab3).

Table 8-3

Transitive Dependencies

| Attribute | Dependency |
| --- | --- |
| `sa account password` | `sa account name` |
| `Service account password` | `Service account name` |
| `SQL Server agent account password` | `SQL Server agent account name` |

Following the rules of normalization, we should move each of these attributes to its own entity. Using our understanding of the business rules, however, we know that in some cases, the same service account may be used for both the database engine service account and the SQL Server Agent service account. Therefore, we will move the service account attributes to a single entity, which we will call `Service Account`. We can then link the `Service Account` entity back to the `DR Instance` entity twice: once for the `Service Account Name` and again for the `SQL Server Agent Account Name`.

The sa account details do not fit with the service account details, as the sa account can never be shared between instances. Therefore, we will move the sa account details to a separate entity, called `sa Account`. We will move the primary key of both the `Service Account` and `sa Account` entities back up to the `Instance` entity, so that they can be linked together. Figure [8-3](#395785_2_En_8_Chapter.xhtml#Fig3) shows how the `Instance`, `sa Account`, and `Service Account` entities have been modeled.

![](images/395785_2_En_8_Chapter/395785_2_En_8_Fig3_HTML.jpg)

Diagram showing a flow chart with six labeled boxes: "Server," "Instance," "DR Server," "DR Instance," "Service Account," and "sa Account." Each box lists related attributes. "Server" includes details like server name and SQL version. "Instance" covers instance name, IP address, and authentication mode. "DR Server" and "DR Instance" focus on disaster recovery details. "Service Account" and "sa Account" list account names and passwords. The chart outlines server and account configurations.

Figure 8-3

Transitive dependencies

### Testing Normalization

We can test whether our normalization process has worked by creating a simple ERD (entity relationship diagram). This diagram will graphically depict the entities, along with the keys that join them. Where a key is unique, the line that connects the entity will be attached with a standard end. Where repeating key values can occur, the line will attach with a crow’s foot.

When we examine the completed ERD, we want to see every line that joins entities depicted as a standard line on one end and a crow’s foot on the other. This is known as a one-to-many relationship and will map directly to the Primary and Foreign Key constraints in your physical database tables.

If we discover many-to-many relationships, the likelihood is that we have missed an entity and will have to add an additional entity to our design that will work in the same way as our DR Server entity to resolve the many-to-many relationship into two one-to-many relationships.

On the other hand, if we discover a one-to-one relationship, we have almost certainly normalized the entities to a point that will add query complexity, without adding any value. In this case, we should denormalize our entities back to a point where all entities are joined together using a one-to-many relationship. An ERD for our normalized inventory database can be found in Figure [8-4](#395785_2_En_8_Chapter.xhtml#Fig4).

![](images/395785_2_En_8_Chapter/395785_2_En_8_Fig4_HTML.jpg)

Diagram of a database schema flow chart showing relationships between tables. The main tables include "Server," "Instance," "Service Account," "sa Account," "DR Server," and "DR Instance." Each table lists attributes such as "Server Name," "Cluster Flag," "Instance Name," "Service Account Name," and "DR Technology." Lines connect tables, indicating relationships, with keys labeled as "PK" for primary key and "FK" for foreign key. The chart visually represents the structure and connections within a database system.

Figure 8-4

Entity relationship diagram

You will notice that the ERD has identified an issue with our modeling. The `sa Account` entity is joined to the `Instance` entity with a one-to-one relationship. This means that we should denormalize the `sa Account` entity back into the `Instance` entity. The resulting ERD is displayed in Figure [8-5](#395785_2_En_8_Chapter.xhtml#Fig5).

![](images/395785_2_En_8_Chapter/395785_2_En_8_Fig5_HTML.jpg)

Diagram illustrating a database schema with five interconnected tables: Server, Instance, DR Server, Service Account, and DR Instance. Each table lists attributes and keys. The Server table includes attributes like Server Name and SQL Server Version. The Instance table contains Instance Name and IP Address. The DR Server table lists DR Server Name. The Service Account table includes Service Account Name. The DR Instance table features DR Instance and DR Technology. Lines indicate relationships between tables, with primary and foreign keys labeled.

Figure 8-5

Denormalized ERD

Tip

The preceding examples have demonstrated how to model a simple inventory database. When creating an inventory database to support your production environment, you may also wish to store details relating to HA technologies, such as AlwaysOn failover clusters or AlwaysOn Availability Groups. You may also wish to model Servers, DR Servers, and (potentially) HA Servers as supertype and subtypes. Further details of supertypes and subtypes can be found at [`https://msdn.micr``osoft.com/en-us/library/cc505839.aspx`](https://msdn.microsoft.com/en-us/library/cc505839.aspx).

## Inventory Database Physical Design

Now that we have a logical model for our database, we have to consider a physical design. The first consideration may be our attribute and entity names and how they will map to SQL Server–friendly object identifiers, which will not have to be encapsulated in square brackets. An object identifier will have to be encapsulated in square brackets when being referenced in code, if any of the following rules are broken:

1.  Identifiers may not contain spaces.

2.  Identifiers may not contain the following special characters:

    ```
    ~                              '
    -                              &
    !                              .
    {                              (
    %                              \
    }                              )
    ^                              `
    ```

3.  Identifiers may not be the same as a T-SQL keyword.

If identifiers do not meet these rules, they are known as delimited identifiers. Our `Server` and `Instance` entities conform to these rules, but our `DR Server` entity will become `DRServer`; our `DR Instance` entity will become `DRInstance`; and our `Service Account Entity` will become `ServiceAccount`. Table [8-4](#395785_2_En_8_Chapter.xhtml#Tab4) details the mapping of attribute names to column names.

Table 8-4

Removing Delimited Identifiers

| Attribute Name | Column Name |
| --- | --- |
| `Server name` | `ServerName` |
| `Instance name` | `InstanceName` |
| `Port` | `Port` |
| `IP Address` | `IPAddress` |
| `Service account name` | `ServiceAccountName` |
| `Service account password` | `ServiceAccountPassword` |
| `Authentication mode` | `AuthenticationMode` |
| `sa account name` | `saAccountName` |
| `sa account password` | `saPassword` |
| `Cluster flag` | `ClusterFlag` |
| `Windows version` | `WindowsVersion` |
| `SQL Server version` | `SQLVersion` |
| `DR Server Name` | `DRServerName` |
| `DR instance Name` | `DRInstanceName` |
| `DR technology` | `DRTechnology` |
| `Target RPO` | `TargetRPO` |
| `Target RTO` | `TargetRTO` |
| `Instance classification` | `InstanceClassification` |
| `Server cores` | `ServerCores` |
| `Instance cores` | `InstanceCores` |
| `Server RAM` | `ServerRAM` |
| `Instance RAM` | `InstanceRAM` |
| `Virtual flag` | `VirtualFlag` |
| `Hypervisor` | `Hypervisor` |
| `SQL Server Agent account name` | `SQLServerAgentAccountName` |
| `SQL Server Agent account password` | `SQLServerAgentAccountPassword` |
| `Application owner` | `ApplicationOwner` |
| `Application owner e-mail` | `ApplicationOwnerEMail` |

A data type is the only constraint that a column will always have, and care should be given when implementing this constraint. If the data type is too restrictive, some data will not fit into the column, and you will find yourself having to fix the issue by changing the column’s data type or length specification. If the data type is larger than it need be, however, you will find yourself using more server resources to satisfy disk and RAM requirements.

Table [8-5](#395785_2_En_8_Chapter.xhtml#Tab5) lists each column in our `Inventory` database and recommends a suitable data type. Where applicable, an explanation is given of the rationale behind the choice.

Table 8-5

Data Types

| Column | Data Type | Explanation |
| --- | --- | --- |
| `Server name` | `NVARCHAR(128)` | SQL Server uses an internal data type called sysname for storing object identifiers. The sysname data type is essentially a synonym for `NVARCHAR(128) NOT NULL`. Therefore, we will use `NVARCHAR(128)` as the data type for any object identifier, in order to keep consistency |
| `Instance name` | `NVARCHAR(128)` |   |
| `Port` | `NVARCHAR(8)` | Stored as a character string, to include the protocol |
| `IP Address` | `NVARCHAR(15)` |   |
| `Service account name` | `NVARCHAR(128)` |   |
| `Service account password` | `NVARCHAR(64)` |   |
| `Authentication mode` | `BIT` | `0` indicates Windows authentication`1` indicates mixed-mode authentication |
| `sa account name` | `NVARCHAR(128)` |   |
| `sa account password` | `NVARCHAR(64)` |   |
| `Cluster flag` | `BIT` |   |
| `Windows version` | `NVARCHAR(64)` | The version of Windows installed on the server, for example, “Windows Server 2022 Standard LSTC” |
| `SQL Server version` | `NVARCHAR(64)` | The version of SQL Server installed on the server, for example, “SQL Server 2022 Enterprise RTM CU2” |
| `DR Server name` | `NVARCHAR(128)` |   |
| `DR instance name` | `NVARCHAR(128)` |   |
| `DR technology` | `NVARCHAR(128)` |   |
| `TargetRPO` | `TINYINT` |   |
| `TargetRTO` | `TINYINT` |   |
| `Instance classification` | `TINYINT` | `1` indicates OLTP`2` indicates Data warehouse`3` indicates Mixed workload`4` indicates ETL |
| `Server cores` | `TINYINT` |   |
| `Instance cores` | `TINYINT` |   |
| `Server RAM` | `SMALLINT` |   |
| `Instance RAM` | `SMALLINT` |   |
| `Virtual flag` | `BIT` | `0` indicates physical server`1` indicates virtual machine |
| `Hypervisor` | `BIT` | `0` indicates VMWare`1` indicates Hyper-V |
| `SQL Server Agent account name` | `NVARCHAR(128)` |   |
| `SQL Server Agent account password` | `NVARCHAR(64)` |   |
| `Application owner` | `NVARCHAR(256)` |   |
| `Application owner e-mail` | `NVARCHAR(512)` |   |

We should now consider the design of our Primary Keys for each table. Are the logical “natural” keys that we have already identified sufficient, or should we create artificial keys, which are usually implemented through an `IDENTITY` column?

We can see that each of our Primary Key columns has the data type `NVARCHAR(128)`. Although these columns will uniquely identify each row in their respective tables, they are wide columns that will consume up to 256 bytes for each row.

The Primary Key will be replicated in each Foreign Key, and because the Primary Key is usually the column(s) that the table’s clustered index is built on, it is also likely to be replicated in all non-clustered indexes. Therefore, we want the Primary Key value to be as narrow as possible. If we add an artificial Primary Key to each table using the INT data type, each key value will only consume 4 bytes, as opposed to a possible 256.

Tip

Because we are taking the approach of using an artificial key, it is likely that each natural key will require a unique constraint.

## Creating the Inventory Database

In this section, we will use DbaTools to create the inventory database, which we have designed throughout this chapter. Our first step will be to create the shell database, which we can achieve using the `New-DbaDatabase` command, as shown in Listing [8-1](#395785_2_En_8_Chapter.xhtml#PC2). In this script, we configure the collation of the database, the database Owner, and the required recovery model, which in our case is Full, as it is an OLTP database and we want transaction log backups to be taken. We also configure the properties of the Primary data file.

Tip

`New-DbaDatabase` also has parameters that allow you to configure the properties of secondary data files, the properties of the transaction log, the default file group, and the file paths of the data and log files. Because we have not specified these values, SQL Server will use the configured values within the Model database.

```
$params = @{
SqlInstance        = "localhost"
Name               = "Inventory"
Collation          = "Latin1_General_CI_AS"
Owner              = "sa"
RecoveryModel      = "Full"
PrimaryFileSize    = "2048" #Specified in MBs
PrimaryFileGrowth  = "1024" #Specified in MBs
PrimaryFileMaxSize = "4096" #Specified in MBs
}
New-DbaDatabase @params
Listing 8-1
Create the Shell Database
```

Because our inventory database will include sa passwords for the managed instances, we need to consider encryption. The encryption hierarchy in SQL Server is depicted in Figure [8-6](#395785_2_En_8_Chapter.xhtml#Fig6).

![](images/395785_2_En_8_Chapter/395785_2_En_8_Fig6_HTML.jpg)

Flow chart illustrating a security architecture. The chart is divided into three main sections: "Operating System" with "DPAPI," "Instance" with "Service Master Key" and "Database Master Key (Copy)," and "Database" with "Database Master Key," "Certificates," "Symmetric Keys," and "Asymmetric Keys." An additional section labeled "EKM Module" includes "Certificates," "Symmetric Keys," and "Asymmetric Keys." The flow chart visually represents the hierarchy and relationship between different security components.

Figure 8-6

SQL Server encryption hierarchy

Tip

A full discussion of encryption is beyond the scope of this book. A full description, however, can be found in the Apress title *Pro SQL Server 2022 Administration*, which can be found at link.springer.com/book/10.1007/978-1-4842-8864-1.

For our use case, we will need to create a Database Master Key (the Service Master Key will be created automatically). We will then need to create a certificate, which is encrypted using the Database Master Key, before creating a symmetric key, which is encrypted using the certificate. This will allow for the sa passwords to be encrypted using the symmetric key, before being inserted into the database table.

We can create the Database Master Key by using the `New-DbaDbMasterKey` command, as shown in Listing [8-2](#395785_2_En_8_Chapter.xhtml#PC3). Here, we convert a password to a secure string and then pass that into the `New-DbaDbMasterKey` command as the password for the Master Key, along with the name of the database that it will be created in.

```
$Password = $SecurePassword = ConvertTo-SecureString 'Pa$$w0rd' -AsPlainText -Force
$params = @{
SqlInstance    = "localhost"
Database       = "Inventory"
SecurePassword = $Password
}
New-DbaDbMasterKey @params
Listing 8-2
Create a Database Master Key for the Inventory Database
```

It is important that we take a backup of the Database Master Key. Otherwise, if we lost the key, it would not be possible to recover our encrypted data. We can back up the key to the file system using the script in Listing [8-3](#395785_2_En_8_Chapter.xhtml#PC4).

```
$Password = $SecurePassword = ConvertTo-SecureString 'Pa$$w0rd' -AsPlainText -Force
$params = @{
SqlInstance    = "localhost"
Database       = "Inventory"
SecurePassword = $Password
Path           = "c:\Keys\"
}
Backup-DbaDbMasterKey @params
Listing 8-3
Back Up the Master Key
```

Sample output of this command is shown below. You will notice that it includes the filename that has been used when saving the key.

```
ComputerName : WIN-J38I01U06D7
InstanceName : MSSQLSERVER
SqlInstance  : WIN-J38I01U06D7
Database     : Inventory
Path         : C:\Keys\localhost-Inventory-20240128124555.key
Status       : Success
```

We can now create a certificate. This is demonstrated in Listing [8-4](#395785_2_En_8_Chapter.xhtml#PC6), which uses the `New-DbDbaCertificate` command to generate the certificate.

```
$Password = $SecurePassword = ConvertTo-SecureString 'Pa$$w0rd' -AsPlainText -Force
$params = @{
SqlInstance    = "localhost"
Database       = "Inventory"
SecurePassword = $Password
}
New-DbaDbCertificate @params
Listing 8-4
Create a Certificate
```

Just as we did with the Database Master Key, we should take a backup of the certificate to ensure that there is no data loss, should an issue with the certificate occur. This is demonstrated in Listing [8-5](#395785_2_En_8_Chapter.xhtml#PC7).

```
$Password = $SecurePassword = ConvertTo-SecureString 'Pa$$w0rd' -AsPlainText -Force
$params = @{
SqlInstance        = "localhost"
Database           = "Inventory"
EncryptionPassword = $Password
DecryptionPassword = $Password
Path               = "C:\Keys\"
}
Backup-DbaDbCertificate @params
Listing 8-5
Back Up the Certificate
```

The output of this command is shown below. You will notice that you are provided with the filenames used to create the certificate and private key files.

```
Certificate  : Inventory
ComputerName : WIN-J38I01U06D7
Database     : Inventory
InstanceName : MSSQLSERVER
Key          : C:\Keys\Inventory202401281255385538.pvk
Path         : C:\Keys\Inventory202401281255385538.cer
SqlInstance  : WIN-J38I01U06D7
Status       : Success
```

We can now create a symmetric key. This will be encrypted using our certificate and in turn will be used to encrypt data that we insert into the `saPassword` column of the `dbo.Instance` table. Listing [8-6](#395785_2_En_8_Chapter.xhtml#PC9) demonstrates how to create a symmetric key. Unfortunately, at the time of writing, the DbaTools command to do this (New-DbaEncryptionKey) looks for a certificate in the master database, as opposed to the user database. Therefore, the script is using Invoke-DbaQuery, passing in a T-SQL script to create the key.

Tip

Further details of Invoke-DbaQuery can be found in Chapter [5](#395785_2_En_5_Chapter.xhtml).

```
$params = @{
SqlInstance = "localhost"
Query = "
CREATE SYMMETRIC KEY Inventory_Key
WITH ALGORITHM = AES_128
ENCRYPTION BY CERTIFICATE Inventory
"
Database = "Inventory"
}
Invoke-DbaQuery @params
Listing 8-6
Create a Symmetric Key
```

Finally, we need to create the tables. Although there is a DbaTools command for creating tables (New-DbaDbTable), I personally find it rather cumbersome, with the column definitions needing to be defined in an array and then passed in. Therefore, we will perform this action using the `Invoke-DbaQuery` command, passing in a script that will create the required schema. This is demonstrated in Listing [8-7](#395785_2_En_8_Chapter.xhtml#PC10). Note that the `saPassword` column is created as a `VARBINARY(256)` instead of an `NVARCHAR(64)` to allow encrypted value to be stored.

```
$params = @{
SqlInstance = "localhost"
Database = “Inventory”
Query       = "
--Create the ServiceAccount table
CREATE TABLE [dbo].[ServiceAccount] (
[ServiceAccountID] INT NOT NULL IDENTITY PRIMARY KEY,
[ServiceAccountName] NVARCHAR(128) NOT NULL UNIQUE,
[ServiceAccountPassword] VARBINARY(256)
) ;
GO
--Create the Server table
CREATE TABLE [dbo].[Server] (
[ServerID] INT NOT NULL IDENTITY PRIMARY KEY,
[ServerName] NVARCHAR(128) NOT NULL UNIQUE,
[ClusterFlag] BIT NOT NULL,
[WindowsVersion] NVARCHAR(64) NOT NULL,
[SQLVersion] NVARCHAR(64) NOT NULL,
[ServerCores] TINYINT NOT NULL,
[ServerRAM] BIGINT NOT NULL,
[VirtualFlag] BIT NOT NULL,
[Hypervisor] BIT NULL,
[ApplicationOwner] NVARCHAR(256) NULL,
[ApplicationOwnerEMail] NVARCHAR(512) NULL
) ;
GO
--Create the DRServer table
CREATE TABLE [dbo].[DRServer] (
[DRServerID] INT NOT NULL IDENTITY ,
[DRServerName] NVARCHAR(128) NOT NULL
[ServerID] INT NOT NULL,
CONSTRAINT [FK_DRServer_ToServer]
FOREIGN KEY ([ServerID]) REFERENCES [Server]([ServerID]),
PRIMARY KEY ([DRServerID], [ServerID])
) ;
GO
--Create the DRInstance table
CREATE TABLE [dbo].[DRInstance] (
[DRInstanceID] INT NOT NULL IDENTITY PRIMARY KEY,
[DRInstanceName] NVARCHAR(128) NOT NULL UNIQUE,
[DRServerID] INT NOT NULL,
[ServerID] INT NOT NULL,
[DRTechnology] NVARCHAR(128) NOT NULL,
[TargetRPO] TINYINT NOT NULL,
[TargetPTO] TINYINT NOT NULL,
CONSTRAINT [FK_DRInstance_ToDRServer]
FOREIGN KEY ([DRServerID], [ServerID]) REFERENCES [DRServer]([DRServerID],[ServerID])
) ;
GO
--Create the Instance table
CREATE TABLE [dbo].[Instance] (
[InstanceID] INT NOT NULL IDENTITY  PRIMARY KEY,
[InstanceName] NVARCHAR(128) NOT NULL UNIQUE,
[ServerID] INT NOT NULL,
[Port] NVARCHAR(8) NOT NULL,
[IPAddress] NVARCHAR(15) NOT NULL,
[SQLServiceAccountID] INT NOT NULL,
[AuthenticationMode] BIT NOT NULL,
[saAccountName] NVARCHAR(128) NULL,
[saAccountPassword] VARBINARY(256) NULL,
[InstanceClassification] TINYINT NOT NULL,
[InstanceCores] TINYINT NOT NULL,
[InstanceRAM] BIGINT NOT NULL,
[SQLServerAgentAccountID] INT NOT NULL,
CONSTRAINT [FK_Instance_ToServer]
FOREIGN KEY ([ServerID]) REFERENCES [Server]([ServerID]),
CONSTRAINT [FK_Instance_SQL_ToServiceAccount]
FOREIGN KEY ([SQLServiceAccountID]) REFERENCES [ServiceAccount]([ServiceAccountID]),
CONSTRAINT [FK_Instance_Agent_ToServiceAccount]
FOREIGN KEY ([SQLServerAgentAccountID]) REFERENCES [ServiceAccount]([ServiceAccountID])
) ;
GO
"
}
Invoke-DbaQuery @params
Listing 8-7
Create the Schema
```

## Summary

A well-designed inventory database can assist DBAs in their automation effort by providing an automatically maintained, central repository of information that can in turn drive automated maintenance routines.

When designing the inventory database, you will have to consider both platform requirements and also the logical and physical design of the database tables. When designing the platform requirements of the database, you should consider HA/DR strategies and also placement of the inventory database to avoid data center isolation issues. Ideally, you will have geographical dispersion of the inventory database, as it is critical to offering a “low-ops” solution.

When performing the logical design of the database, you should model the data using normalization, which is a process invented in 1970 for the elimination of data duplication. You can then test your model by using an entity relationship diagram to ensure that all entities are joined with one-to-many relationships.

The physical design of the database will involve designing the physical tables that will store the data. This involves ensuring that there are no delimited identifiers, choosing appropriate data types, and defining any artificial keys that may be required.

You can create the database and required artifacts, such as encryption keys and certificates, as well as the database and tables using DbaTools. This keeps all of your code in PowerShell. There are some artifacts, such as symmetric keys and tables, however, which it may be better to create using the `Invoke-DbaQuery` command, due to limitations in the functionality-specific commands.

# 9. Implementing Configuration Management for SQL Server

In Chapter [3](#395785_2_En_3_Chapter.xhtml), we explored the concepts of configuration management and how we can use PowerShell DSC to configure the Windows operating system and prevent configuration drift. In Chapter [6](#395785_2_En_6_Chapter.xhtml), we expanded on this concept to examine how we can use DSC to ensure that our SQL Server instance is installed. In this chapter, we will explore the `SqlServerDsc` module further, exploring how we can use it to configure our SQL Server instance and prevent drift.

We will start by defining the requirements for our build, by thinking about some of the common aspects of a SQL Server instance that we may wish to configure during a SQL Server build and avoid drift during the instance’s life cycle. Then, we will explore how we can implement the appropriate DSC resources.

## Defining the Configuration Requirements

When defining a standard build for our organization, we should think not just about installing SQL Server but also how it should be configured. When thinking about this configuration, we should consider aspects such as security, performance, and reducing ongoing operations.

Tip

High availability is beyond the scope of this book, but it is another consideration for DSC, as the `SqlServerDsc` module has commands that can help add nodes to clusters or Availability Groups.

### Security Requirements

Security in SQL Server is a huge topic, which I cover in depth in the Apress book *Securing SQL Server*, which can be found at link.springer.com/book/10.1007/978-1-4842-2265-2\. For the purpose of this chapter, however, let’s pick three considerations. The first of these is the use of `xp_cmdshell`. This is a system stored procedure, which allows administrators to interact with the operating system from within SQL Server. Unfortunately, the nature of it makes it a prime target for nefarious activities. There are attack vectors, which means this procedure can be used to perform lateral movement attacks across the network, if a SQL Server instance is compromised.

The second configuration we will enforce is that OLE Automation is disabled. If OLE Automation Procedures are enabled, then the attack surface of the SQL Server instance is increased, and users are allowed to execute functions, external to SQL Server, which act within the security context of the database engine service. This is another prime target for attackers who wish to perform lateral movement attacks.

Finally, we will ensure that the built-in Windows `Administrators` role is not associated with a SQL Server Login. In very old versions of SQL Server, the `Administrators` group was automatically given a SQL Server Login and added to the `sysadmin` fixed server role. Over the last 15 years, however, this behavior has been changed, and it is now considered a poor practice. In fact, in SQL Server 2022, it’s no longer possible to add the local Administrators group as a login. Only designated SQL Server administrators should be given permissions to the SQL Server instance. If administrators who are not skilled in SQL Server attempt to perform actions in SQL Server, then this can lead to issues, even if their intentions are benign.

### Performance Requirements

Ensuring good performance is at the forefront of many DBAs’ minds. Many performance configurations will be dependent on the nature of the workload profile of the databases hosted in the instance. For example, OLTP databases, which expect a high percentage of `INSERTS` and `UPDATES`, may be configured differently to data warehouse databases, which are subject to nightly ETL runs, but the user experience is based mainly on large reads. There are some performance optimizations which **usually** hold true, regardless of the workload profile, however.

The first configuration that we will ensure for our instance is that the maximum degree of parallelism (MAXDOP) is configured optimally. This setting will determine the maximum number of cores that can be used by a single query. Setting MAXDOP to 0 allows for unlimited parallelization, whereas setting it to an integer value specifies the number of logical processors that can be used.

Although there are edge cases, such as servers with multiple NUMA nodes, the general rule of thumb is that MAXDOP should be configured as the number of cores in the server, up to a maximum of 8\. `SqlserverDsc` allows us to configure this value dynamically. This is very helpful, as it means that if we resize our server, DSC will automatically change the MAXDOP to ensure it remains appropriate.

The second configuration that we will enforce relates to how much memory is consumed by SQL Server. Microsoft recommends that the maximum amount of memory used by SQL Server (for the buffer cache) is 75% of the total memory in the server. Therefore, we will enforce this recommendation. Just as with MAXDOP, `SqlserverDsc` can enforce this dynamically to ensure best practice is still followed after a server is resized, without the need for administrator intervention.

Tip

If there are multiple instances installed on the server, then you should divide the 75% across those multiple instances. For example, if a server hosts three instances, you may wish to set each of them with a maximum memory of 25% to ensure that one instance does not starve the others of memory.

The next configuration relates to the minimum amount of memory that will be consumed by the SQL Server instance. By default, SQL Server adjusts memory usage up and down, depending on how much it needs. The trouble with this is that dynamic memory allocation in SQL Server is not the product’s strongest feature. Given that SQL Server should usually be the only application running on a server, and assuming that there is only a single instance, I recommend configuring the minimum memory to be the same as maximum memory, therefore preventing dynamic memory management.

Tip

SQL Server does not allocate the minimum memory to itself at service startup. It will always grow the memory usage as required. What the minimum memory setting does is set a limit on how much memory can be released.

### Reducing Ongoing Operations

The idea of automation is to prevent the need for manual tasks. Therefore, if we know up front that administration will be required, post install, it makes good sense to automate it up front to save us time later.

The first configuration that we will enforce, to save ourselves administration later, is to ensure that the DBA team has administrative access to the instance. This will involve creating a login for the Windows group called `DBATeam` and ensuring that this login is in the `sysadmin` fixed server role.

The second configuration that we will enforce relates to the central management server and configuration database that we discussed in Chapter [8](#395785_2_En_8_Chapter.xhtml). Specifically, we will ensure that a linked server exists, which provides access to the `Config` database on the central management server. This will allow us to run queries from the local instance, which pull back useful information. This could be the application owner, for example.

## Implementing the Configuration

In the following sections, we will examine how to create the resources for each of our requirements. The first section will look at implementing our security requirements. We will then discuss how to implement our performance and operational requirements. Finally, we will bring everything together into our configuration that we have been building through Chapters [3](#395785_2_En_3_Chapter.xhtml) and [6](#395785_2_En_6_Chapter.xhtml).

### Implementing Security Resources

Let’s look at how we can enforce each of our chosen security-related configurations using DSC. The first resource we will need should ensure that `xp_cmdshell` is disabled. We can achieve this using the resource definition in Listing [9-1](#395785_2_En_9_Chapter.xhtml#PC1). This resource uses the `SqlConfiguration` resource type from the `SqlserverDsc` module, and we have given it a name of `xpCmdshell`. You will notice that we are not passing an `Ensure` parameter to this resource. Instead, we pass the name of the option and the desired value of the option. In our case, this is `0`, which means disabled. We can also specify if we want the service to be restarted. This is because there are a small number of configuration options that will only take effect after a service restart.

Tip

When configuring a default instance with the `SqlserverDsc` module, we should pass `MSSQLSERVER` as the instance name. For a named instance, we will, of course, pass the instance name instead. In this scenario, our configuration (see Chapter [6](#395785_2_En_6_Chapter.xhtml)) is parameterized, so we will pass the `SqlInstanceName` variable, which has a default value of `MSSQLSERVER` but can be overridden when the MOF is compiled.

```
SqlConfiguration 'xpCmdshell' {
OptionName     = 'xp_cmdshell'
OptionValue    = 0
RestartService = $false
ServerName     = 'localhost'
InstanceName   = $SqlInstanceName
}
Listing 9-1
xpCmdshell Resource
```

Our next resource will ensure that OLE Automation is disabled in the instance. For this configuration, we will again use the `SqlConfiguration` resource. This time, however, we will pass in the `OptionName` of `Ole Automation Procedures`. Once again, we will pass in the `OptionValue` of `0` to denote that we want the feature to be disabled. This is illustrated in Listing [9-2](#395785_2_En_9_Chapter.xhtml#PC2).

```
SqlConfiguration 'OLEAutomation' {
OptionName     = 'Ole Automation Procedures'
OptionValue    = 0
RestartService = $false
ServerName     = 'localhost'
InstanceName   = $SqlInstanceName
}
Listing 9-2
OLEAutomation Resource
```

The final security-related resource that we will create will ensure that the Windows `Authenticated Users` group does not have a SQL Server Login. To do this, we will use the `SqlLogin` resource from the `SqlserverDsc` module. We will pass in the `Ensure` parameter, but instead of using `Ensure = 'Present'` as we have previously, we will use `Ensure = 'Absent'` to remove the Login if it exists.

We will use the Name parameter to provide the name of the Login that we wish to remove, and the `LoginType` parameter allows us to define if the Login is mapping to a Windows user, a Windows group, or a SQL Login. This is demonstrated in Listing [9-3](#395785_2_En_9_Chapter.xhtml#PC3).

```
SqlLogin 'AuthenticatedUsers' {
Ensure       = 'Absent'
Name         = 'NT AUTHORITY\Authenticated Users'
LoginType    = 'WindowsGroup'
ServerName   = 'localhost'
InstanceName = $SqlInstanceName
}
Listing 9-3
BuiltinAdministrators Resource
```

### Implementing Performance Resources

Let’s examine how we can perform our performance configurations using DSC. The first of these configurations ensures that MAXDOP is configured to an optimum value. With the `SqlMaxDop` resource, it is possible to pass a desired value for the MAXDOP, but instead of doing this, we will pass `DynamicAlloc = $true`. This is a useful feature, which uses logic contained in the code of the resource, which will calculate the ideal value based on the current size of the server and configure MAXDOP appropriately. This is demonstrated in Listing [9-4](#395785_2_En_9_Chapter.xhtml#PC4).

Tip

If your server has multiple NUMA nodes, then the recommendations are different, and the `SqlMaxDop` resource does not have logic to support this. Therefore, if the server has multiple NUMA nodes, I would recommend adding your own custom logic into the configuration.

```
SqlMaxDop 'DynamicSqlMaxDop' {
Ensure                  = 'Present'
DynamicAlloc            = $true
ServerName              = 'localhost'
InstanceName            = $SqlInstanceName
}
Listing 9-4
DynamicSqlMaxDop Resource
```

Next, we will consider the minimum and maximum memory requirements. We want to configure both of these settings to be 75% of the available memory in the server, which will leave enough memory for the operating system and avoid dynamic memory management in SQL Server.

We can configure both of these options using a single `SqlMemory` resource. This resource allows us to configure the minimum and maximum memory dynamically, using rules built into the resource, or it allows us to pass an arbitrary value in MB. In our case, however, we want to pass values as a percentage of total server memory. Therefore, we will use the `MaxMemoryPercent` and `MinMemoryPercent` parameters, as shown in Listing [9-5](#395785_2_En_9_Chapter.xhtml#PC5).

```
SqlMemory 'MinAndMaxMemory75Percent' {
Ensure               = 'Present'
DynamicAlloc         = $false
MaxMemoryPercent     = 75
MinMemoryPercent     = 75
ServerName           = 'localhost'
InstanceName         = $SqlInstanceName
}
Listing 9-5
MinAndMaxMemory75Percent Resource
```

### Implementing Operational Resources

The first configuration that we want to implement to reduce our operational overhead will ensure that a Login exists for the `DBATeam` Windows group and has administrative permissions on the instance. This will require two resources.

The first resource will ensure that the `DBATeam` login exists, using the `SqlLogin` resource, which we have already discussed in this chapter. The second resource will be a `SqlRole` resource, which ensures that the `DBATeam` login is added to the `sysadmin` fixed server role. In the declaration of the `SqlRole` resource, we need to pass `sysadmin` to the `ServerRoleName` parameter. The `Members` parameter accepts a comma-separated list of security principals that should be added to the role. Listing [9-6](#395785_2_En_9_Chapter.xhtml#PC6) demonstrates how to create these two resources.

Note

The script assumes that the `DBATeam` Windows group already exists. If you are following along with the examples in this chapter, then you should either create the `DBATeam` group in Windows or use a Windows group that already exists.

```
SqlLogin 'DBATeamLogin' {
Ensure           = 'Present'
Name             = 'MyDomain\DBATeam'
LoginType        = 'WindowsGroup'
ServerName       = 'localhost'
InstanceName     = $SqlInstanceName
}
SqlRole 'DBATeamSysadmin' {
Ensure           = 'Present'
ServerRoleName   = 'sysadmin'
MembersToInclude = 'MyDomain\DBATeam'
ServerName       = 'localhost'
InstanceName     = $SqlInstanceName
}
Listing 9-6
DBATeamLogin and DBATeamSysadmin Resources
```

Caution

This example uses MembersToInclude. If we were to use the Members property instead, then it would overwrite existing group membership, rather than append additional users.

The final resource that we will configure will ensure that a linked server exists, which will allow us to run queries against our central management server. The trouble is that a resource does not exist in the `SqlServerDsc` module, which allows us to manage linked servers.

Therefore, we will use an advanced resource, known as the `SqlScriptQuery` resource. This resource allows us to define our own `Get`, `Test`, and `Set` queries. So, before looking at the PowerShell script to create the resource, let’s first look at each of the T-SQL queries that will be run by the resource.

The first of these scripts is the Get script. Listing [9-7](#395785_2_En_9_Chapter.xhtml#PC7) demonstrates how to write a `Get` query for our use case. The query reads the `sys.sysservers` catalog view and returns the svrname where the svrname is `CENTRALMGMT`. It then converts the results of this query to JSON and returns it to DSC.

Note

The scripts in this section require a default SQL Server instance to be hosted on a server named `CENTRALMGMT`. You should change these scripts to point to a SQL Server instance in your environment.

```
SELECT srvname
FROM sys.sysservers
WHERE srvname = 'CENTRALMGMT'
FOR JSON AUTO
Listing 9-7
Get Query
```

The test query can be found in Listing [9-8](#395785_2_En_9_Chapter.xhtml#PC8). This query checks to see if a server with the name `CENTRALMGMT` exists in the `sys.sysservers` catalog view. If the server exists, the script succeeds and returns a message stating the linked server was found. If it fails, it returns an error message.

```
IF (SELECT COUNT(srvname) FROM sys.sysservers WHERE srvname = 'CENTRALMGMT') = 0
BEGIN
RAISERROR ('Did not find the CENTRALMGMT linked sever', 16, 1)
END
ELSE
BEGIN
PRINT 'Found the CENTRALMGMT linked server'
END
Listing 9-8
Test Query
```

Finally, the Set query uses the `sp_addlinkedserver` system stored procedure to create a linked server and then the `sp_addlinkedserverlogin` system stored procedure to configure pass-through authentication. This means that queries made against the linked server will be made using the security context of the security principal that ran the query. This script can be seen in Listing [9-9](#395785_2_En_9_Chapter.xhtml#PC9).

```
USE master
GO
EXEC master.dbo.sp_addlinkedserver
@server = 'CENTRALMGMT',
@srvproduct='SQL Server'
GO
EXEC master.dbo.sp_addlinkedsrvlogin
@rmtsrvname = 'CENTRALMGMT',
@locallogin = NULL , @useself = 'True'
GO
Listing 9-9
Set Query
```

The `SqlScriptQuery` resource that will run our custom scripts is shown in Listing [9-10](#395785_2_En_9_Chapter.xhtml#PC10). As well as the `Get`, `Set`, and `Test` scripts, we also pass a parameter called `QueryTimeout`, which specifies the maximum duration any of the three scripts can run, before they fail with a timeout.

Tip

You will notice that the string terminators (herestrings) for each of the custom queries are not indented. This is because string terminators must always be placed at the beginning of the line, when working with the `SqlserverDsc` resources.

```
SqlScriptQuery 'CentralMgmtLinkedServer' {
Id                   = 'CentralMgmtLinkedServer'
ServerName           = 'localhost'
InstanceName         = $SqlInstanceName
GetQuery             = @'
SELECT srvname
FROM sys.sysservers
WHERE srvname = 'CENTRALMGMT'
FOR JSON AUTO
'@
TestQuery            = @'
IF (SELECT COUNT(srvname) FROM sys.sysservers WHERE srvname = 'CENTRALMGMT') = 0
BEGIN
RAISERROR ('Did not find the CENTRALMGMT linked sever', 16, 1)
END
ELSE
BEGIN
PRINT 'Found the CENTRALMGMT linked server'
END
'@
SetQuery             = @'
USE master
GO
EXEC master.dbo.sp_addlinkedserver
@server = 'CENTRALMGMT',
@srvproduct='SQL Server'
GO
EXEC master.dbo.sp_addlinkedsrvlogin
@rmtsrvname = 'CENTRALMGMT',
@locallogin = NULL , @useself = 'True'
GO
'@
QueryTimeout         = 30
}
Listing 9-10
CentralMgmtLinkedServer Resource
```

### Bringing It All Together

The only thing that remains is to bring all of our resources together into the configuration that we have built during the course of Chapters [3](#395785_2_En_3_Chapter.xhtml) and [6](#395785_2_En_6_Chapter.xhtml). This complete configuration can be seen in Listing [9-11](#395785_2_En_9_Chapter.xhtml#PC11). You will notice that we have added a `DependOn` clause to the `DBATeamSysadmin` resource, which makes it dependent on the `DBATeamLogin` resource, as, of course, this resource will fail if the Login doesn’t already exist. We will make all other SQL Server configuration resources dependent on the `SQLServerService` resource, which ensures that SQL Server Database Engine service is started. This is because each of these resources would fail if the database engine service was stopped.

```
Configuration WindowsConfig {
param (
[string] $SqlInstanceName = 'MSSQLSERVER',
[Parameter(Mandatory)]
[ValidateSet('Developer', 'Standard', 'Enterprise')]
[string] $Edition
)
Import-DscResource -ModuleName 'PSDesiredStateConfiguration'
Import-DscResource -ModuleName SqlServerDsc
if ($Edition -eq 'Developer') {
$ProductKey = '22222-00000-00000-00000-00000'
} elseif ($edition -eq 'Standard') {
$ProductKey = '00000-00000-00000-00000-00000'
} elseif ($edition -eq 'Enterprise') {
$ProductKey = '00000-00000-00000-00000-00000'
}
$serviceName = if ($SqlInstanceName -eq 'MSSQLSERVER') {
'MSSQLSERVER'
} else {
'MSSQL${0}' -f $SqlInstanceName
}
Node 'localhost' {
#OS Resources
File CreateCertificateBackupsFolder {
Ensure = "Present"
Type = "Directory"
DestinationPath = "C:\CertificateBackups"
}
Registry OptimizeForBackgroundServices {
Ensure      = "Present"
Key         = "HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl"
ValueName   = "Win32PrioritySeparation"
ValueData   = 24
ValueType   = 'Dword'
}
#Install SQL Server
SqlSetup 'InstallInstance' {
InstanceName        = $SqlInstanceName
Features            = 'SQLENGINE'
SourcePath          = 'C:\SQL Media'
SQLSysAdminAccounts = @('Administrator')
ProductKey          = $ProductKey
}
Service SQLServerService {
Name        = $serviceName
StartupType = "Automatic"
State       = "Running"
DependsOn   = '[SqlSetup]InstallInstance'
}
#Security Resources
SqlConfiguration 'xpCmdshell' {
OptionName     = 'xp_cmdshell'
OptionValue    = 0
RestartService = $false
ServerName     = 'localhost'
InstanceName   = $SqlInstanceName
DependsOn      = '[Service]SQLServerService'
}
SqlConfiguration 'OLEAutomation' {
OptionName     = 'Ole Automation Procedures'
OptionValue    = 0
RestartService = $false
ServerName     = 'localhost'
InstanceName   = $SqlInstanceName
DependsOn      = '[Service]SQLServerService'
}
SqlLogin 'BuiltinAdministrators' {
Ensure       = 'Absent'
Name         = 'BUILTIN\Administrators'
LoginType    = 'WindowsGroup'
ServerName   = 'localhost'
InstanceName = $SqlInstanceName
DependsOn    = '[Service]SQLServerService'
}
#Performance Resources
SqlMaxDop 'Set_SqlMaxDop_ToAuto' {
Ensure                  = 'Present'
DynamicAlloc            = $true
ServerName              = 'localhost'
InstanceName            = $SqlInstanceName
DependsOn               = '[Service]SQLServerService'
}
SqlMemory 'MinAndMaxMemory75Percent' {
Ensure               = 'Present'
DynamicAlloc         = $false
MaxMemoryPercent     = 75
MinMemoryPercent     = 75
ServerName           = 'localhost'
InstanceName         = $SqlInstanceName
DependsOn            = '[Service]SQLServerService'
}
#Operational Resources
SqlLogin 'DBATeamLogin' {
Ensure               = 'Present'
Name                 = 'MyDomain\DBATeam'
LoginType            = 'WindowsGroup'
ServerName           = 'localhost'
InstanceName         = $SqlInstanceName
DependsOn            = '[Service]SQLServerService'
}
SqlRole 'DBATeamSysadmin' {
Ensure               = 'Present'
ServerRoleName       = 'sysadmin'
Members              = 'MyDomain\DBATeam'
ServerName           = 'localhost'
InstanceName         = $SqlInstanceName
DependsOn            = '[SqlLogin]DBATeamLogin'
}
SqlScriptQuery 'CentralMgmtLinkedServer' {
Id                   = 'CentralMgmtLinkedServer'
ServerName           = 'localhost'
InstanceName         = $SqlInstanceName
GetQuery             = @'
SELECT srvname
FROM sys.sysservers
WHERE srvname = 'CENTRALMGMT'
FOR JSON AUTO
'@
TestQuery            = @'
IF (SELECT COUNT(srvname) FROM sys.sysservers WHERE srvname = 'CENTRALMGMT') = 0
BEGIN
RAISERROR ('Did not find the CENTRALMGMT linked sever', 16, 1)
END
ELSE
BEGIN
PRINT 'Found the CENTRALMGMT linked server'
END
'@
SetQuery             = @'
USE master
GO
EXEC master.dbo.sp_addlinkedserver @server = 'CENTRALMGMT', @srvproduct='SQL Server'
GO
EXEC master.dbo.sp_addlinkedsrvlogin @rmtsrvname = 'CENTRALMGMT', @locallogin = NULL , @useself = 'True'
GO
'@
QueryTimeout         = 30
DependsOn            = '[Service]SQLServerService'
}
}
}
Listing 9-11
The Final Configuration
```

## Summary

Many people think of DSC as a powerful tool for building operating systems and avoiding configuration drift at the operating system level. SQL Server DBAs should also consider it a useful tool for building and avoiding configuration drift in SQL Server.

The `SqlserverDsc` module provides many resources, which we can use to configure all aspects of SQL Server, including security, performance, and availability. Some of these resources, such as the `SqlMaxDop` resource, have logic written into the resources that avoid us having to pass arbitrary values and, instead, dynamically calculate the most appropriate value for us. This is very useful because it allows us to resize our servers without having to worry about changing the values passed within our DSC configurations.

The `SqlScriptQuery` resource allows us to define our own `Get`, `Test`, and `Set` queries. This makes the module fully extensible, as it means that we can cater to literally any configuration requirement in SQL Server, even if an out-of-the-box resource does not exist to perform the configuration.

When using the `SqlScriptQuery` resource, the `Get` query should return the status of the resource to DSC as JSON, and we can achieve this by using a `FOR JSON` clause in the query. The `Test` query should check to see if the resource is configured in our desired state. If it is, we should let the query succeed, but if not, we should raise an error, which will result in the `Set` query being executed. The `Set` query simply consists of the T-SQL statements required to configure the resource to our requirements.

# 10. Centralizing Maintenance

One of the main goals of introducing automation and scripting techniques to your SQL Server environment is the ability to centralize your maintenance. This is a key step toward obtaining a low-ops model, where a DBA’s time is spent adding value proposition to the business, instead of mundane tasks, such as checking the status of SQL Server Agent jobs, across their enterprise.

In this chapter, we will first discuss the design considerations for implementing a centralized maintenance system, which is built on the inventory database that we discussed in Chapter [8](#395785_2_En_8_Chapter.xhtml). We will then explore how to create the maintenance engine.

## Designing Centralized Maintenance

There are multiple ways in which you can build a centralized maintenance system for your SQL Server Enterprise. One of the options is to use SQL Server Agent Master and Target servers, which allow target servers to pull jobs from a centralized master.

This is an approach that I previously advocated, but in recent years, however, the limitations and lack of flexibility in this approach have led me to recommend creating a custom maintenance scheduling system, which can easily be integrated with your automated build. Assuming that you use a configuration management approach, you can also easily perform activities, such as changing maintenance schedules, just by updating parameters in your build. Therefore, this chapter will focus on using the custom maintenance system approach.

As depicted in Figure [10-1](#395785_2_En_10_Chapter.xhtml#Fig1), for this approach, we will use a SQL Server instance as a centralized management system. The instance will host our inventory database, which will be enhanced to contain a set of tables and views, which will define which maintenance routines will be executed on which servers and when. The instance will also host a SQL Server Agent job, which runs continuously and calls off maintenance scripts against servers, based on their defined schedules. The PowerShell scripts that are called off by the job will also be stored on our centralized server. This gives the benefit of only having to update a single job, or a set of maintenance scripts on a single server, instead of potentially hundreds of servers across the enterprise. At the same time, we maintain the flexibility of running maintenance job at a different time on each server.

![](images/395785_2_En_10_Chapter/395785_2_En_10_Fig1_HTML.jpg)

Flow chart illustrating a maintenance process. The "Central Maintenance Instance" contains an "Inventory Database" and a "SQL Agent Maintenance Job." Arrows connect the database and maintenance job, indicating data flow. A separate arrow links "Maintenance Scripts" to the maintenance job. The process outputs to "Managed Servers," depicted as a stack of server icons. Key elements include inventory management, SQL maintenance, and server management.

Figure 10-1

Target maintenance routine architecture

Tip

The ability to run jobs at different times across different servers is absolutely key. If we consider backups, as a classic example, many servers will require backups to happen at specific times to avoid their maintenance or ETL windows. But even without considering windows of opportunity, in environments, such as a private cloud, where all SQL Instances run on shared resources, it would be undesirable to run all backup jobs at the same time, due to the risk of resource contention on the underlying hardware.

We now need to consider the table structure that we will need to add to the inventory database to support the solution. At a high level, this is depicted in Figure [10-2](#395785_2_En_10_Chapter.xhtml#Fig2). Here, the MaintenanceWindows table will key off of the Server ID from the Server and Instance tables and will define the windows in which maintenance routines can run for each server across the enterprise.

![](images/395785_2_En_10_Chapter/395785_2_En_10_Fig2_HTML.jpg)

Flow chart depicting a database schema. At the top is "dbo.MaintenanceWindows (View)" connected to two tables: "dbo.MaintenanceWindows (Table)" and "dbo.MaintenanceTasks (Table)." Both tables are linked to "dbo.Instance (Table)" and "dbo.Server (Table)" respectively, with cardinality indicators showing many-to-one relationships.

Figure 10-2

High-level design of tables and views

The MaintenanceTasks table will define which maintenance tasks should run against each instance, giving us the ability to disable specific maintenance jobs from specific servers, as well as providing the interval between the executions of each maintenance task. Again, this table will key from the Server and Instance tables.

Finally, the MaintenanceSchedule view is built across the two tables mentioned above and calculates the next time that a maintenance task should run against each server. This view will be the entry point for the SQL Server Agent job that will call off our maintenance scripts.

So now let’s take a look at the configuration data we should store in each of these objects. The suggested column layout of the MaintenanceWindows table is detailed in Table [10-1](#395785_2_En_10_Chapter.xhtml#Tab1).

Tip

The example column layouts in this chapter are meant to provide a base template for creating a scheduling engine; however, you should consider adapting them to the unique needs of your organization.

We would like to make our scheduling options as flexible and as easy as possible. Therefore, we will want to accommodate for a DBA to use set schedules of Daily or Weekly or to express the schedule in minutes. Our MaintenanceTasks table, which will be at the granularity of Server, Instance, and Task will allow for this, by containing a schedule column, where we can add values such as Daily and Weekly. It will then have a computed column, which translates this value to a number of minutes. This will allow our code to easily determine how often the task should run against a given server and instance.

Table 10-1

`MaintenanceWindows` Columns

| Column | Description |
| --- | --- |
| MaintenanceWindowID | Artificial Primary Key of the table |
| ServerID | Foreign Key from Server table. A unique constraint should be added across the ServerID and InstanceID columns |
| InstanceID | Foreign Key from Instance table. A unique constraint should be added across the ServerID and InstanceID columns |
| DayOfWeekNumber | A number from 1 to 7 which designates the day of the week. Allows for an instance to have a different schedule on different days |
| StartTime | The start time of the maintenance window |
| EndTime | The end time of the maintenance window |

We would like to make our scheduling options as flexible and as easy as possible. Therefore, we will want to accommodate for a DBA to use set schedules of Daily or Weekly or to express the schedule in minutes. Our MaintenanceTasks table, which will be at the granularity of Server, Instance, and Task will allow for this, by containing a

The suggested column layout for the MaintenanceTasks table is defined in Table [10-2](#395785_2_En_10_Chapter.xhtml#Tab2).

Table 10-2

`MaintenanceTasks` Columns

| Column | Description |
| --- | --- |
| MaintenanceTaskID | Artificial Primary Key of the table |
| ServerID | Foreign Key from Server table. A unique constraint should be added across the ServerID and InstanceID columns |
| InstanceID | Foreign Key from Instance table. A unique constraint should be added across the ServerID and InstanceID columns |
| Task | The name of the maintenance task |
| LastExecDate | The date and time that the task was last executed |
| Schedule | The schedule. In our example, this can be Daily, Weekly, or the number of minutes between task executions |
| ScheduleMinutes | An integer value that expresses the schedule in minutes |
| InProgress | A flag that specifies if the task is in progress |
| LastExecStatus | The status of the task, the last time it was executed on the instance |
| TaskDisabled | A flag that specifies if the task is disabled for the specific instance |

The MaintenanceSchedules view will be the entry point for the code, so it should contain all columns that will be useful to minimize potential code duplication within our maintenance tasks. The suggested column layout for the view is defined in Table [10-3](#395785_2_En_10_Chapter.xhtml#Tab3).

Table 10-3

`MaintenanceSchedules` Columns

| Column | Description |
| --- | --- |
| ServerInstance | The combined server\instance name of each target instance |
| Task | The name of the maintenance task |
| LastExecDate | The date and time that the task was last executed on the instance |
| NextExecDate | The date and time that the task is next due to be executed on the instance |
| InProgress | A flag specifying if the task is currently in progress |
| TaskDisabled | A task specifying if the task is disabled |

Our SQL Server Agent job should be scheduled to run every minute. The job will have a job step for every common maintenance task, each of which will call a PowerShell script. The PowerShell script will use the data surfaced in the MaintenanceSchedule view to determine which instances the task should run against and then execute the task against each instance in turn.

Therefore, as we will have a job step per maintenance task, we need to define the list of maintenance tasks that we want to run against our environment. In our example, we will run the following maintenance tasks:

*   Full backup of all databases

*   Remove old backup files

*   Update statistics

Tip

Of course, the list of maintenance tasks that you want to create will depend on the requirements of your own organization and enterprise. For example, you will almost certainly want a dynamic index rebuild capability, and the script to achieve this can be found in Chapter [7](#395785_2_En_7_Chapter.xhtml). The possibilities are endless, and I encourage you to think broadly. For example, you could consider including transaction log backups for databases in full recovery model, columnstore index maintenance, or even snapshot maintenance.

## Implementing Centralized Maintenance

Now that we have put some thought into the design of our centralized maintenance solution, we will now look at how to implement it. In the following sections, we will discuss how to create the tables and view, how to create the PowerShell scripts, and finally how to create the SQL Server Agent job.

### Create the Tables and View

The first table that we will create is the `MaintenanceWindows` table. This table is relatively straightforward as it does not contain any computed columns. We will define the artificial key for the table, and we will define two foreign keys against the server and instance tables, with a unique constraint across these two columns, plus DayOfWeekNumber to avoid the same `server\instance` inadvertently being added twice for the same day. The script in Listing [10-1](#395785_2_En_10_Chapter.xhtml#PC1) demonstrates how to create this table.

```
Set-DbatoolsInsecureConnection -SessionOnly
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       = "
CREATE TABLE dbo.MaintenanceWindows (
MaintenanceWindowID  INT    NOT NULL    PRIMARY KEY CLUSTERED,
ServerID             INT    NOT NULL    FOREIGN                                       KEY REFERENCES Server(ServerID),
InstanceID           INT    NOT NULL    FOREIGN                                        KEY REFERENCES Instance(InstanceID),
DayOfWeekNumber      INT    NOT NULL,
StartTime            TIME   NOT NULL,
EndTime              TIME   NOT NULL
) ;
CREATE UNIQUE NONCLUSTERED INDEX ServerID_InstanceID_Day
ON dbo.MaintenanceWindows(ServerID, InstanceID, DayOfWeekNumber) ;
"
}
Invoke-DbaQuery @params
Listing 10-1
Create the MaintenanceWindows Table
```

Next, we will use the script in Listing [10-2](#395785_2_En_10_Chapter.xhtml#PC2) to create the `MaintenanceTasks` table. This table will once again have a unique constraint, created across the `ServerID`, `InstanceID`, and `Task` columns. Also, note the definition of the `ScheduleMinutes` column. This is a computed column, which changes the granularity of all schedule types into minutes.

```
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       = "
CREATE TABLE dbo.MaintenanceTasks (
MaintenanceTaskID INT            NOT NULL    PRIMARY KEY CLUSTERED,
ServerID          INT            NOT NULL    FOREIGN                                           KEY REFERENCES Server(ServerID),
InstanceID        INT            NOT NULL    FOREIGN                                       KEY REFERENCES Instance(InstanceID),
Task              NVARCHAR(32)   NOT NULL,
LastExecDate      DATETIME       NULL,
Schedule          NVARCHAR(8)    NOT NULL,
ScheduleMinutes   AS (CASE WHEN [Schedule]='Daily' THEN (1440) WHEN [Schedule]='Weekly' THEN (10080) ELSE [Schedule] END) PERSISTED,
InProgress        BIT            NOT NULL,
LastExecStatus    NCHAR(7)       NOT NULL,
TaskDisabled      BIT            NOT NULL
) ;
CREATE UNIQUE NONCLUSTERED INDEX ServerID_InstanceID
ON dbo.MaintenanceTasks(ServerID, InstanceID, Task) ;
"
}
Invoke-DbaQuery @params
Listing 10-2
Create the MaintenanceTasks Table
```

Finally, we will create the `MaintenanceSchedules` view (Listing [10-3](#395785_2_En_10_Chapter.xhtml#PC3)), which will be based on the underlying tables, and calculate when each maintenance task should run against each server\instance. Pay particular attention to the `NextExecDate` column, which is calculated to provide this functionality. Also, pay particular attention to the `StartTime` and `EndTime` columns. These columns are derived from the `MaintenanceWindows` table, with the join to this table being based on the day number of the day on which the query is executed. This is achieved by evaluating the `DayOfWeekNumber` column against the WEEKDAY daypart.

```
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       =
"
CREATE VIEW dbo.MaintenanceSchedule
AS
SELECT
S.ServerName + '\' + I.InstanceName AS ServerInstance
, MT.Task
, MT.LastExecDate
, CASE
WHEN LastExecDate IS NULL
THEN GETDATE()
ELSE DATEADD(MINUTE,ScheduleMinutes,LastExecDate)
END AS NextExecDate
, MT.InProgress
, MT.TaskDisabled
, MW.StartTime
, MW.EndTime
, S.ServerID
, I.InstanceID
FROM dbo.MaintenanceTasks MT
INNER JOIN dbo.Server S
ON S.ServerID = MT.ServerID
INNER JOIN dbo.Instance I
ON I.InstanceID = MT.InstanceID
INNER JOIN dbo.MaintenanceWindows MW
ON MW.ServerID = s.ServerID
AND MW.InstanceID = I.InstanceID
AND (SELECT DATEPART(dw,GETDATE())) = MW.DayOfWeekNumber
"
}
Invoke-DbaQuery @params
Listing 10-3
Create the MaintenanceSchedules View
```

### Create the PowerShell Scripts

Before diving into the script implementation for each of the specific tasks, we should add some metadata to our database that we can use to test our framework and scripts. The script in Listing [10-4](#395785_2_En_10_Chapter.xhtml#PC4) inserts some sample data into our tables for this purpose. If you are following along with the examples, then you should change the server and instance details to match your own infrastructure.

```
$params = @{
SqlInstance = "localhost"
Database = "Inventory"
Query       = "
--Insert Server/Instance Details
INSERT INTO dbo.Server (
ServerName
,ClusterFlag
,WindowsVersion
,SQLVersion
,ServerCores
,ServerRAM
,VirtualFlag
,ApplicationOwner
,ApplicationOwnerEMail)
VALUES (
'SQL2022-STANDAL'
,0
,'Windows Server 2022 Standard'
,'SQL Server 2022 Developer Edition'
,4
,16
,1
,'Peter Carter'
,'pete@ExpertScripting.com'
)
GO
INSERT INTO dbo.ServiceAccount
(ServiceAccountName)
VALUES
('SQLServiceAccount')
GO
INSERT INTO dbo.Instance (
InstanceName
,ServerID
,Port
,IPAddress
,SQLServiceAccountID
,AuthenticationMode
,InstanceClassification
,InstanceCores
,InstanceRAM
,SQLServerAgentAccountID
)
VALUES (
'Expert'
,1
,1434
,'127.0.0.1'
,1
,0
,1
,2
,6
,1
),
(
'Scripting'
,1
,1435
,'127.0.0.1'
,1
,0
,1
,2
,6
,1
)
GO
--Insert Maintenance Details
INSERT INTO dbo.MaintenanceWindows (
MaintenanceWindowID
,ServerID
,InstanceID
,DayOfWeekNumber
,StartTime
,EndTime
)
VALUES (
1
,1
,1
,1
,'00:01'
,'05:00'
),
(
2
,1
,1
,2
,'00:01'
,'05:00'
),
(
3
,1
,1
,7
,'06:01'
,'18:00'
),
(
4
,1
,2
,6
,'00:01'
,'05:00'
),
(
5
,1
,2
,7
,'06:01'
,'18:00'
)
GO
INSERT INTO [dbo].[MaintenanceTasks] (
[MaintenanceTaskID]
,[ServerID]
,[InstanceID]
,[Task]
,[LastExecDate]
,[Schedule]
,[InProgress]
,[LastExecStatus]
,[TaskDisabled]
)
VALUES (
1
,1
,1
,'FullBackup'
,'1900-01-01'
,'Daily'
,0
,'Not Run'
,0
),
(
2
,1
,1
,'RemoveOldBackups'
,'1900-01-01'
,'Daily'
,0
,'Not Run'
,0
),
(
3
,1
,1
,'UpdateStats'
,'1900-01-01'
,'Daily'
,0
,'Not Run'
,0
),
(
4
,1
,2
,'FullBackup'
,'1900-01-01'
,'Daily'
,0
,'Not Run'
,0
),
(
5
,1
,2
,'RemoveOldBackups'
,'1900-01-01'
,'Daily'
,0
,'Not Run'
,1
),
(
6
,1
,2
,'UpdateStats'
,'1900-01-01'
,'Daily'
,0
,'Not Run'
,1
)
"
}
Invoke-DbaQuery @params
Listing 10-4
Insert Sample Data
```

So, let’s start by creating a script that will run full backups against our User databases. The script starts by generating a list of maintenance tasks that it needs to run against each server/instance. It creates this list by querying the MaintenanceSchedule view and filtering out any maintenance tasks that are already in progress and tasks that have been disabled. It also includes a filter to ensure that the current date and time is greater than or equal to the time that the task is next due to run. The next filter ensures that the current time is within the start and end times of when the job can be executed. This is achieved by casting the current date/time as time and evaluating this against the `StartTime` and `EndTime` columns. The final filter removes all tasks which are not equal to `FullBackup`.

The results of this query are passed into a foreach loop, which connects to each SQL Server instance in turn and generates a list of user databases on that instance. The `MaintenanceTasks` table is then updated to indicate that the task is in progress for the given server\instance.

The list of databases is then passed into a child foreach loop, where each database, in turn, is backed up. The backup statement is generated by using the name of the given database.

Once all user databases on the instance are backed up, the child foreach loop exits, and the `MaintenanceTasks` table is updated again. This time, to reflect that the task is no longer in progress for the given server. The `LastExecTime` and `LastExecStatus` columns are also updated to avoid the task being run again until required. The loop then repeats for every server that is due to have full backups, as shown in Listing [10-5](#395785_2_En_10_Chapter.xhtml#PC5).

```
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       = "
SELECT
ServerInstance
,Task
,LastExecDate
,NextExecDate
,InProgress
,TaskDisabled
,StartTime
,EndTime
,ServerID
,InstanceID
FROM dbo.MaintenanceSchedule ms
WHERE InProgress = 0
AND TaskDisabled = 0
AND GETDATE() >= NextExecDate
AND CAST(GETDATE() AS TIME) BETWEEN StartTime AND EndTime
AND Task = 'FullBackup'
"
}
$ServerTasks = Invoke-DbaQuery @params
foreach ($Server in $ServerTasks) {
$params = @{
SqlInstance = $Server.ServerInstance
Database    = "Master"
Query       = "
SELECT name FROM sys.databases WHERE database_id > 4
"
}
$databases = Invoke-DbaQuery @params
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       = "
UPDATE dbo.MaintenanceTasks
SET InProgress = 1
WHERE ServerID = '" + $Server.ServerID + "' AND InstanceID = '" + $Server.InstanceID + "' AND Task = 'FullBackup'
"
}
Invoke-DbaQuery @params
foreach ($database in $databases) {
$params = @{
SqlInstance = $Server.ServerInstance
Database    = "Master"
Query       = "
BACKUP DATABASE " + $database.name + " TO DISK = 'C:\Backups\" + $database.name + ".bak'
"
}
Invoke-DbaQuery @params
}
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       = "
UPDATE dbo.MaintenanceTasks
SET InProgress = 0, LastExecDate = GETDATE(), LastExecStatus = 'Success'
WHERE ServerID = '" + $Server.ServerID + "' AND InstanceID = '" + $Server.InstanceID + "' AND Task = 'FullBackup';
"
}
Invoke-DbaQuery @params
}
Listing 10-5
Full Backup Script
```

Tip

If you are following along, then save this script as c:\scripts\fullbackup.ps1\. Later, we will call this script from a SQL Agent job.

The next script we need is illustrated in Listing [10-6](#395785_2_En_10_Chapter.xhtml#PC6). This script will remove old backup files. You will notice that a `$limit` variable is created that is populated with the current date/time minus three days. The `$path` variable builds the path to the share, using the split function to remove the instance name. The `Get-ChildItem` cmdlet is then used to build a list of files to delete. This list is then piped into the `Remove-Item` cmdlet.

For this example, we will assume that we should delete backups that are older than three days. This makes the assumption that there is an enterprise backup tool offloading the .bak files to tape. If there isn’t, then you will likely want to retain the backups for much longer.

The script uses the same logic as the FullBackup script we created earlier. The only difference is that the tasks being scheduled and the script being called are `RemoveOldBackups`.

Tip

The script assumes that the volume the backup files are stored on is shared. If this is not the case, then assuming you have PowerShell Remoting configured, you can use the `Invoke-Command` cmdlet to invoke the script on the remote server.

```
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       = "
SELECT
ServerInstance
,Task
,LastExecDate
,NextExecDate
,InProgress
,TaskDisabled
,StartTime
,EndTime
,ServerID
,InstanceID
FROM dbo.MaintenanceSchedule ms
WHERE InProgress = 0
AND TaskDisabled = 0
AND GETDATE() >= NextExecDate
AND CAST(GETDATE() AS TIME) BETWEEN StartTime AND EndTime
AND Task = 'RemoveOldBackups'
"
}
$ServerTasks = Invoke-DbaQuery @params
foreach ($Server in $ServerTasks) {
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       = "
UPDATE dbo.MaintenanceTasks
SET InProgress = 1
WHERE ServerID = '" + $Server.ServerID + "' AND InstanceID = '" + $Server.InstanceID + "' AND Task = 'RemoveOldBackups'
"
}
Invoke-DbaQuery @params
foreach ($database in $databases) {
$limit = (Get-Date).AddDays(-3)
$path = "\\" + $Server.ServerInstance.split('\')[0] + "\Backups\"
Get-ChildItem -Path $path | Where-Object { $_.CreationTime -lt $limit } | Remove-Item
}
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       = "
UPDATE dbo.MaintenanceTasks
SET InProgress = 0, LastExecDate = GETDATE(), LastExecStatus = 'Success'
WHERE ServerID = '" + $Server.ServerID + "' AND InstanceID = '" + $Server.InstanceID + "' AND Task = 'RemoveOldBackups';
"
}
Invoke-DbaQuery @params
}
Listing 10-6
Remove Old Backups Script
```

The final script we need for this example is illustrated in Listing [10-7](#395785_2_En_10_Chapter.xhtml#PC7). This script is used to update statistics. The script follows the same pattern as the two previous maintenance tasks. It loops around each instance and then each database within the instance. The script itself is a simple execution of the sp_`updatestats` stored procedure.

```
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       = "
SELECT
ServerInstance
,Task
,LastExecDate
,NextExecDate
,InProgress
,TaskDisabled
,StartTime
,EndTime
,ServerID
,InstanceID
FROM dbo.MaintenanceSchedule ms
WHERE InProgress = 0
AND TaskDisabled = 0
AND GETDATE() >= NextExecDate
AND CAST(GETDATE() AS TIME) BETWEEN StartTime AND EndTime
AND Task = 'UpdateStatistics'
"
}
$ServerTasks = Invoke-DbaQuery @params
foreach ($Server in $ServerTasks) {
$params = @{
SqlInstance = $Server.ServerInstance
Database    = "Master"
Query       = "
SELECT name FROM sys.databases
"
}
$databases = Invoke-DbaQuery @params
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       = "
UPDATE dbo.MaintenanceTasks
SET InProgress = 1
WHERE ServerID = '" + $Server.ServerID + "' AND InstanceID = '" + $Server.InstanceID + "' AND Task = 'UpdateStats'
"
}
Invoke-DbaQuery @params
foreach ($database in $databases) {
$params = @{
SqlInstance = $Server.ServerInstance
Database    = $database.name
Query       = "
EXEC sp_UpdateStatistics
"
}
Invoke-DbaQuery @params
}
$params = @{
SqlInstance = "localhost"
Database    = "Inventory"
Query       = "
UPDATE dbo.MaintenanceTasks
SET InProgress = 0, LastExecDate = GETDATE(), LastExecStatus = 'Success'
WHERE ServerID = '" + $Server.ServerID + "' AND InstanceID = '" + $Server.InstanceID + "' AND Task = 'UpdateStatistics';
"
}
Invoke-DbaQuery @params
}
Listing 10-7
Update Statistics
```

Each of the scripts should be saved to a folder that the SQL Agent service account (or Proxy account) can access. For this example, I have saved them to the c:\scripts folder on the central management server. I have named the files FullBackups.ps1, RemoveOldBackups.ps1, and UpdateStatistics.ps1, respectively.

Tip

There are various enhancements that could be made, if appropriate for your environment. For example, you could consider breaking the granularity of the MaintenanceTasks table down to the level of database to allow you to schedule different databases on the same instance to be backed up at different times. You could also add LastExecStatus to the MaintenanceSchedule view, which would allow you to add retry logic, if a task fails.

### Create the SQL Server Agent Job

Now that we have each of the scripts that we would like to run against our SQL Server instances, we can create a SQL Agent job on the central management server that will run them. Because the scripts themselves determine which instances they need to run against, it means that the job can be scheduled to run once a minute. If there are no tasks for the script to execute, it will simply exit and allow the job to move to the next job step.

The script in Listing [10-8](#395785_2_En_10_Chapter.xhtml#PC8) creates the SQL Agent Job, the schedule, and the required steps. The job is configured to move to the next step, regardless of success or failure of the previous step. This avoids a problem with one script stopping the other tasks from running.

Note

Where we have used " inside the query, we have escaped them using "".

```
$params = @{
SqlInstance  = "localhost"
Query        = "
USE msdb
GO
DECLARE @jobId BINARY(16)
EXEC  msdb.dbo.sp_add_job @job_name='SQLMaintenance',
@enabled=1,
@notify_level_eventlog=0,
@notify_level_email=2,
@notify_level_page=2,
@delete_level=0,
@owner_login_name='SQL2022-STANDAL\Administrator', @job_id = @jobId OUTPUT
SELECT @jobId
GO
EXEC msdb.dbo.sp_add_jobserver @job_name='SQLMaintenance', @server_name = 'SQL2022-STANDALONE'
GO
USE msdb
GO
EXEC msdb.dbo.sp_add_jobstep @job_name='SQLMaintenance', @step_name='FullBackups',
@step_id=1,
@cmdexec_success_code=0,
@on_success_action=3,
@on_fail_action=3,
@retry_attempts=0,
@retry_interval=0,
@os_run_priority=0, @subsystem='PowerShell',
@command='powershell ""C:\scripts\FullBackups.ps1""'
GO
USE msdb
GO
EXEC msdb.dbo.sp_add_jobstep @job_name='SQLMaintenance', @step_name='RemoveOldBackups',
@step_id=2,
@cmdexec_success_code=0,
@on_success_action=3,
@on_fail_action=3,
@retry_attempts=0,
@retry_interval=0,
@os_run_priority=0, @subsystem='PowerShell',
@command='powershell ""C:\scripts\RemoveOldBackups.ps1""'
GO
USE msdb
GO
EXEC msdb.dbo.sp_add_jobstep @job_name='SQLMaintenance', @step_name='UpdateStatistics',
@step_id=3,
@cmdexec_success_code=0,
@on_success_action=1,
@on_fail_action=2,
@retry_attempts=0,
@retry_interval=0,
@os_run_priority=0, @subsystem='PowerShell',
@command='powershell ""C:\scripts\UpdateStatistics.ps1""'
GO
USE msdb
GO
EXEC msdb.dbo.sp_update_job @job_name='SQLMaintenance',
@enabled=1,
@start_step_id=1,
@notify_level_eventlog=0,
@notify_level_email=2,
@notify_level_page=2,
@delete_level=0,
@owner_login_name='SQL2022-STANDAL\Administrator'
GO
USE msdb
GO
DECLARE @schedule_id int
EXEC msdb.dbo.sp_add_jobschedule @job_name='SQLMaintenance', @name='MaitenanceSchedule',
@enabled=1,
@freq_type=4,
@freq_interval=1,
@freq_subday_type=4,
@freq_subday_interval=1,
@freq_relative_interval=0,
@freq_recurrence_factor=1,
@active_start_date=20241020,
@active_end_date=99991231,
@active_start_time=0,
@active_end_time=235959
GO
"
}
Invoke-DbaQuery @params
Listing 10-8
Create the SQL Agent Job
```

## Summary

Creating a centralized maintenance engine provides the ultimate flexibility in how and when maintenance tasks will be executed across your SQL Server estate. A maintenance engine sits on a centralized management server and allows for maintenance tasks to be executed across the enterprise but controlled from a single location.

The engine itself consists of tables and views that store information about which maintenance routines should be executed at which times on which server. This is followed by simple logic within PowerShell scripts that determines where a task should be run. These tasks are then called off by a SQL Agent Job, which runs every minute.

This approach has endless possibilities, and I encourage you to use this approach as a basis for developing your own maintenance scheduler, which meets the needs of your organization.

# 11. Automating Routine Maintenance and Break/Fix Scenarios

The more quickly a DBA team can respond to a request from the business, the more value it can bring to the organization. In Chapter [10](#395785_2_En_10_Chapter.xhtml), we focused on the centralization of maintenance routines, but there are many more mundane aspects of a DBA’s role that can be automated.

In this chapter, we will look at two aspects of this. This first aspect is automating ad hoc maintenance, which will allow us to respond more rapidly to business requirements. This chapter will focus specifically on environment refreshes, but the techniques can be applied to many scenarios.

The second aspect of automation we will address is break/fix automation. This can greatly improve service to the business, by resolving issues automatically, without needing to wait for a DBA to do it. This chapter will focus on resolving log space issues, but, once again, the techniques can be applied to almost any scenario.

So far, this book has focused primarily on PowerShell. In this chapter, however, we will move away from PowerShell to demonstrate other methods of achieving automation with SQL Server. Specifically, we will examine the use of SSIS and SQL Agent Alerts. The chapter also demonstrates configuring a SQL Agent Job through the GUI. It is useful to understand this because, in this book, you have already learned how to create a SQL Agent job through code, but sometimes the fastest way to create a SQL Agent Job is to use the GUI and then script it, so that you can save it to source control.

## Automating Ad Hoc Routine Maintenance

The techniques for building centralized maintenance you learned in Chapter [10](#395785_2_En_10_Chapter.xhtml) are great for routine maintenance that runs on a schedule. But what about ad hoc tasks? These can’t be scheduled to run on a regular basis, but that doesn’t prevent us from implementing automation to take the heavy lifting out of the requests, thus allowing us to respond more quickly and also freeing us up for higher value activity.

Let’s take environment refreshes as an example. If you have in-house application development teams, it is common for them to periodically request that their development environment be refreshed with the latest cut of production data. In this scenario, providing that there are no data privacy issues, we may respond by backing up the database and restoring it to the development environment or detaching the database, copying the data files, and reattaching it in both locations.

In other environments, however, especially where there is a heightened need for security, the process can be a lot more complex.

For example, I once worked with a FTSE 100 company that was tightly controlled by regulators, and when developers required an environment refresh, there was no way that they could be given access to live data. Therefore, the process of refreshing the environment consisted of the following steps:

1.  Back up the production database.

2.  Put the development server into SINGLE USER mode, so that developers could not access it.

3.  Restore the database onto the development server.

4.  Remove database users without a login.

5.  Obfuscate the data.

6.  Put instance back into MULTI USER mode.

In this section, we will re-create this process for the AdventureWorks2022 database, which we will copy from an instance called ESPROD1 to an instance called ESPROD2\. This process is depicted in the workflow that can be found in Figure [11-1](#395785_2_En_11_Chapter.xhtml#Fig1).

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig1_HTML.jpg)

Flow chart illustrating a database obfuscation process. It begins with "Start" in a green oval, followed by steps: "Backup Production Database," "Put Development Server into SINGLE USER Mode," "Restore Database to Development Server," and "Remove Database Logins." The process continues with "Obfuscation," which includes "Identify Data to be Obfuscated" and "Obfuscate Data." Finally, it ends with "Put Development Server into MULTI USER Mode" and "Stop" in a pink oval. Arrows connect each step sequentially.

Figure 11-1

Environment refresh workflow

We will parameterize the process, so that it can be easily modified for any database. Again, we have the choice of orchestrating this workflow using either PowerShell or SSIS. For this kind of workflow, however, I would recommend that SSIS be considered as the most appropriate tool.

### Creating the SSIS Package

We will now use SQL Server Data Tools (SSDT) to create a SQL Server Integration Services Project, which we will name EnvironmentRefresh. We will give the same name to the package that is automatically created within the project.

Our first task will be to create three project parameters that will accept the name of the Production Server\Instance, the name of the Development Server\Instance, and the name of the database to be refreshed. This is illustrated in Figure [11-2](#395785_2_En_11_Chapter.xhtml#Fig2).

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig2_HTML.jpg)

A table displaying server information with columns labeled "Name," "Data type," "Value," "Sensitive," "Required," and "Description." The rows list "ProductionServer," "DevelopmentServer," and "DatabaseName," all with "String" as the data type. The "Sensitive" and "Required" columns are marked "False" for each entry.

Figure 11-2

Creating project parameters

The next step will be to create two OLEDB Connection Managers, by right-clicking in the Connection Managers window and using the New OLEDB Connection dialog box. We should name one Connection Manager ProductionServer and the other DevelopmentServer.

Once the Connection Managers have been created, we can use the Expression Builder to configure the ServerName property of each Connection Manager to be dynamic, based on the ProductionServer and DevelopmentServer project parameters that we created. Figure [11-3](#395785_2_En_11_Chapter.xhtml#Fig3) illustrates using Expression Builder to configure the ServerName property of the ProductionServer Connection Manager.

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig3_HTML.jpg)

Expression Builder interface screenshot showing a window with sections for Variables and Parameters, and Functions. The left panel lists system variables like `$Project::DatabaseName`, `$Project::DevelopmentServer`, and `$Project::ProductionServer`. The right panel includes categories like Mathematical Functions, String Functions, and Date/Time Functions. The Expression field contains `@[$Project::ProductionServer]`. Buttons for "Evaluate Expression," "OK," and "Cancel" are at the bottom.

Figure 11-3

Expression Builder

We must now create three package variables, which will hold the SQL statements that will be run to back up the database, restore the database, and obfuscate the database. We will enter the appropriate expressions to build the SQL statements within the variables, as illustrated in Figure [11-4](#395785_2_En_11_Chapter.xhtml#Fig4). It is important to set the EvaluateAsExpression property of the variables to true (although in newer versions, this happens automatically if you paste in an expression). This can be configured in the Properties window, when the variable is in scope.

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig4_HTML.jpg)

A table displaying three rows with columns labeled Name, Scope, Data type, Value, and Expression. The names are "BicyclistStatement," "AutomobileStatement," and "ChocolateMilkStatement," all within the "Environment" scope and of "String" data type. Each row contains complex expressions and values related to database operations and string manipulations.

Figure 11-4

Creating package variables

The expression stored in the BackupSQLStatement variable is detailed in Listing [11-1](#395785_2_En_11_Chapter.xhtml#PC1).

```
"BACKUP DATABASE " +  @[$Project::DatabaseName] + " TO DISK = 'C:\\Backups\\" + @[$Project::DatabaseName] + ".bak'"
Listing 11-1
BackupSQLStatement Expression
```

The expression stored in the RestoreSQLStatement variable can be found in Listing [11-2](#395785_2_En_11_Chapter.xhtml#PC2).

```
"RESTORE DATABASE " +  @[$Project::DatabaseName] + " FROM DISK = '\\\\ESPROD2\\backups\\" + @[$Project::DatabaseName] + ".bak' WITH REPLACE; ALTER DATABASE " + @[$Project::DatabaseName] + " SET SINGLE_USER WITH ROLLBACK IMMEDIATE;"
Listing 11-2
RestoreSQLStatement Expression
```

The expression stored in the ObfuscateDataStatement variable can be found in Listing [11-3](#395785_2_En_11_Chapter.xhtml#PC3).

```
"DECLARE @SQL NVARCHAR(MAX) ;
SET @SQL = (SELECT CASE t.name WHEN 'int' THEN 'UPDATE ' + SCHEMA_NAME(o.schema_id) + '.' + QUOTENAME(OBJECT_NAME(nkc.object_id)) + ' SET ' +  QUOTENAME(c.name) + ' = CHECKSUM(' + c.name + '); '
WHEN 'money' THEN 'UPDATE ' + QUOTENAME(SCHEMA_NAME(o.schema_id)) + '.' + QUOTENAME(OBJECT_NAME(nkc.object_id)) + ' SET ' +  QUOTENAME(c.name) + ' = CHECKSUM(' + c.name + '); '
WHEN 'nvarchar' THEN 'UPDATE ' + QUOTENAME(SCHEMA_NAME(o.schema_id)) + '.' + QUOTENAME(OBJECT_NAME(nkc.object_id)) + ' SET ' +  QUOTENAME(c.name) + ' = LEFT(RTRIM(CONVERT(nvarchar(255), NEWID())), ' + CAST(c.max_length / 2 AS NVARCHAR(10)) + '); '
WHEN 'varchar' THEN 'UPDATE ' + QUOTENAME(SCHEMA_NAME(o.schema_id)) + '.' + QUOTENAME(OBJECT_NAME(nkc.object_id)) + ' SET ' +  QUOTENAME(c.name) + ' = LEFT(RTRIM(CONVERT(nvarchar(255), NEWID())), ' + CAST(c.max_length AS NVARCHAR(10)) + '); '
WHEN 'text' THEN 'UPDATE ' + QUOTENAME(SCHEMA_NAME(o.schema_id)) + '.' + QUOTENAME(OBJECT_NAME(nkc.object_id)) + ' SET ' +  QUOTENAME(c.name) + ' = LEFT(RTRIM(CONVERT(nvarchar(255), NEWID())), ' + CAST(c.max_length AS NVARCHAR(10)) + '); '
WHEN 'ntext' THEN 'UPDATE ' + QUOTENAME(SCHEMA_NAME(o.schema_id)) + '.' + QUOTENAME(OBJECT_NAME(nkc.object_id)) + ' SET ' +  QUOTENAME(c.name) + ' = LEFT(RTRIM(CONVERT(nvarchar(255), NEWID())), ' + CAST(c.max_length AS NVARCHAR(10)) + '); '  END
FROM
(
SELECT object_id, column_id
FROM sys.columns
EXCEPT --Exclude foreign key columns
SELECT parent_object_id, parent_column_id
FROM sys.foreign_key_columns
EXCEPT --Exclude check constraints
SELECT parent_object_id, parent_column_id
FROM sys.check_constraints
) nkc
INNER JOIN sys.columns c
ON nkc.object_id = c.object_id
AND nkc.column_id = c.column_id
INNER JOIN sys.objects o
ON nkc.object_id = o.object_id
INNER JOIN sys.types t
ON c.user_type_id = t.user_type_id
AND c.system_type_id = t.system_type_id
INNER JOIN sys.tables tab
ON o.object_id = tab.object_id
WHERE is_computed = 0  --Exclude computed columns
AND c.is_filestream = 0 --Exclude filestream columns
AND c.is_identity = 0 --Exclude identity columns
AND c.is_xml_document = 0 --Exclude XML columns
AND c.default_object_id = 0 --Exclude columns with default constraints
AND c.rule_object_id = 0 --Exclude columns associated with rules
AND c.encryption_type IS NULL --Exclude columns with encryption
AND o.type = 'U' --Filter on user tables
AND t.is_user_defined = 0 --Exclude columns with custom data types
AND tab.temporal_type = 0 --Exclude temporal history tables
FOR XML PATH('')
) ;
EXEC(@SQL) ;
ALTER DATABASE " +  @[$Project::DatabaseName] + " SET MULTI_USER ;"
Listing 11-3
ObfuscateDataStatement Expression
```

We will now create an Execute SQL task, which will be used to back up the production database. As you can see in Figure [11-5](#395785_2_En_11_Chapter.xhtml#Fig5), we will use the Execute SQL Task Editor dialog box to configure the task to run against the production server. We will configure the SQL source type as a variable and then point the task to our BackupSQLStatement variable.

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig5_HTML.jpg)

Screenshot of the "Execute SQL Task Editor" window in a software application. The interface includes a navigation pane on the left with options: General, Parameter Mapping, Result Set, and Expressions. The main section displays settings for executing an SQL task, including General, Options, Result Set, and SQL Statement. Key details include a connection type of "OLE DB," a connection to "ProductionServer," and a source variable "User::BackupSQLStatement." Buttons at the bottom include "Build Query," "Parse Query," "OK," "Cancel," and "Help."

Figure 11-5

Configuring backup task

We will now add a second Execute SQL Task, which we will configure to restore the database onto the Development Server and place it into Single User mode. The Execute SQL Task Editor for this task is illustrated in Figure [11-6](#395785_2_En_11_Chapter.xhtml#Fig6). Here, you will notice that we have configured the task to run against our Development Server. We have configured the SQL source as a variable and pointed the task to our RestoreSQLStatement variable.

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig6_HTML.jpg)

Screenshot of the "Execute SQL Task Editor" window. The interface includes a navigation pane on the left with options: General, Parameter Mapping, Result Set, and Expressions. The main section displays settings for a task named "Restore Database" with options like TimeOut set to 0, CodePage 1252, and TypeConversionMode allowed. The ResultSet is set to None. SQL Statement settings include ConnectionType as OLE DB, Connection as DevelopmentServer, and SQLSourceType as Variable. Buttons at the bottom include Build Query, Parse Query, OK, Cancel, and Help.

Figure 11-6

Configuring the restore task

The next Execute SQL Task will be used to remove any existing database users without a login. It is possible to create logins directly at the database level, as opposed to database users, which must map to logins at the Instance level. This feature is part of contained database functionality, which simplifies administration when using technologies such as AlwaysOn Availability Groups. In our scenario, however, database users without a login can cause an issue, because instead of being orphaned on the instance of the lower environment, unauthorized users could connect if they have a database login and know the password. Therefore, we will remove all such logins.

Figure [11-7](#395785_2_En_11_Chapter.xhtml#Fig7) illustrates how we will use the Execute SQL Task Editor dialog box to configure this process. You will note that we have configured the Connection property to run the query against the Development Server; we have configured the SQL source as direct input; and we have typed the query directly into the SQLStatement property. We have not used a variable, as, this time, we will explore an alternative way of configuring the task to be dynamic.

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig7_HTML.jpg)

Screenshot of the "Execute SQL Task Editor" window in a software application. The window displays various configuration options for running SQL statements. Key sections include "General," "Options," "Result Set," and "SQL Statement." The "ConnectionType" is set to "OLE DB," and "Connection" is "DevelopmentServer." The "SQLSourceType" is highlighted as "Direct input," with a sample SQL statement shown. Buttons at the bottom include "Browse," "Build Query," "Parse Query," "OK," "Cancel," and "Help."

Figure 11-7

Configuring remove database users without a login task

The expression entered into the SQLSourceType property can be found in Listing [11-4](#395785_2_En_11_Chapter.xhtml#PC4).

```
"USE " +  @[$Project::DatabaseName] + "
DECLARE @SQL NVARCHAR(MAX)
SET @SQL = (
SELECT 'DROP USER ' + QUOTENAME(name) + ' ; '
FROM sys.database_principals
WHERE type = 'S'
AND authentication_type = 0
AND principal_id > 4
FOR XML PATH('')
) ;
EXEC(@SQL)"
Listing 11-4
Remove Database Users Without a Login
```

We will make this task dynamic by configuring an expression on the SQLStatement property of the task, as illustrated in Figure [11-8](#395785_2_En_11_Chapter.xhtml#Fig8).

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig8_HTML.jpg)

Screenshot of the "Property Expressions Editor" window. It displays a table with columns labeled "Property" and an expression. The property shown is "SqlStatementSource" with an expression: `"USE " + @[Project::DatabaseName] + " DECLARE @SQL NVARCHAR(MAX)`. Buttons at the bottom include "Delete," "OK," and "Cancel."

Figure 11-8

Configuring expression on SQLStatement property

The final Execute SQL task will be used to obfuscate the data and place it back in Multi User mode. We will use the Execute SQL Task Editor dialog box to configure the connection to be made to the Development Server and configure the SQL source as a variable. We will then point the task to the ObfuscateDataSQLStatement variable. The dynamic techniques that we have used to create the package mean that when we run the package (probably as a manual execution of a SQL Server Agent job), we can pass in any Production Server, Development Server, and database. The metadata-driven process will obfuscate textual data, integers, and money data in any database. If the need arises, you can also easily amend the script to obfuscate only certain data, such as a specific schema, within the database.

All of the required tasks have now been created and configured. To clean up the package, join each task in sequence, using success precedence constraints. Because there are four Execute SQL Tasks, I also strongly recommend renaming them to give them intuitive names. This is always a best practice, but even more important, when there are so many tasks of the same type, as by default, they will be named with a meaningless, sequential number. The final control flow is depicted in Figure [11-9](#395785_2_En_11_Chapter.xhtml#Fig9).

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig9_HTML.jpg)

Flow chart illustrating a database management process. The sequence includes four steps: "Backup Database," "Restore Database," "Remove Database Logins," and "Obfuscate Data," connected by arrows indicating the flow direction. The chart is set against a dark background, with each step enclosed in a rectangular box.

Figure 11-9

Completed control flow

## Automating Break/Fix Scenarios

Automating responses to break/fix scenarios is the pinnacle of your automation efforts. When implemented well, it can significantly reduce the TCO (Total Cost of Ownership) of the SQL Server estate.

If you attempt to design an entire break/fix automation solution up front, you are destined to fail. First, the cost and effort will be prohibitive. Second, each enterprise is not only unique but also changes over time. Therefore, break/fix automation should be implemented as CSI (continuous service improvement).

The approach that I recommend is to use a simple formula to decide which break/fix scenarios you should create automated responses for. The formula I recommend is “If issue occurs x times in n weeks and (time to fix X n) > estimated automation effort.”

The values that you will plug into this formula are dependent on your environment. An example would be “If issue occurs 5 times in 4 weeks and (time to fix X 20) > 4 hours.” The formula calculates if you will recoup your automation effort, within a three-month period. If you will recover your effort in three months, then it is almost certainly worth the up-front effort to automate your response.

If your company uses a sophisticated ticketing system, such as ServiceNow or ITSM 365, you will be able to report on problems. A problem is a ticket that is repeatedly raised. For example, your ticketing system may be configured so that if a ticket is raised for the same issue five times in four weeks, it becomes a problem ticket. This allows effective root-cause analysis, but you will instantly see the synergy with our formula. Reporting on problem tickets against the DBA team’s queue can be a great starting point when looking for candidate break/fix scenarios to automate.

The following sections will discuss how to design and implement an automated response to 9002 errors. 9002 errors occur if there is no space to write to the transaction log, and the log is unable to grow.

### Designing a Response to 9002 Errors

Before creating a process to respond to 9002 errors, we will first create a process flow. This will assist with the coding effort and also act as ongoing documentation of the process.

Figure [11-10](#395785_2_En_11_Chapter.xhtml#Fig10) displays the process flow that we will follow when writing our code.

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig10_HTML.jpg)

Flow chart illustrating a decision-making process for database management. It begins with "Start" in a green oval, leading to a decision diamond asking, "Is DB using SIMPLE recovery model?" If "Yes," it proceeds to "Issue a CHECKPOINT," then "Shrink Transaction Log," and ends at "Stop" in a red oval. If "No," it asks, "Was delayed truncation to HA/DR?" If "Yes," it leads to "e-mail DBA Team," then "Shrink Transaction Log," and ends at "Stop." If "No," it goes to "Backup Transaction Log," then "Shrink Transaction Log," and ends at "Stop."

Figure 11-10

Responding to 9002 error process flow

### Automating a Response to 9002 Errors

We will use a SQL Server Agent alert to trigger our process if a 9002 error occurs. To create a new alert, drill through SQL Server Agent in SQL Server Management Studio and select New Alert from the context menu of Alerts. This will cause the New Alert dialog box to be displayed. On the General page of the dialog box, illustrated in Figure [11-11](#395785_2_En_11_Chapter.xhtml#Fig11), we will give the alert a name and configure it to be triggered by SQL Server error number 9002.

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig11_HTML.jpg)

Screenshot of a "New Alert" configuration window for SQL Server. The alert is named "RespondTo9002" and is enabled. It is a SQL Server event alert for all databases, triggered by error number 9002\. The severity is set to "001 - Miscellaneous System Information." The server is "ESASSMGMT1" with a connection to "ESASSMGMT1\Pete." Options to view connection properties and progress status as "Ready" are visible.

Figure 11-11

New Alert – General page

On the Response page of the dialog box, we will check the option to Execute Job and use the New Job button to invoke the New Job dialog box. Save the job, then on the General page of the New Job dialog box, we will define a name for our new Job, as illustrated in Figure [11-12](#395785_2_En_11_Chapter.xhtml#Fig12).

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig12_HTML.jpg)

Screenshot of a "New Job" window in a software application. The interface includes fields for "Name," "Owner," "Category," and "Description," with the name set to "RespondTo9002" and owner as "ESASSMGMT1\Pete." The category is "[Uncategorized (Local)]." A checkbox labeled "Enabled" is checked. The left panel shows options like "General," "Steps," "Schedules," "Notifications," and "Targets." Connection details display the server as "ESASSMGMT1" and a link to "View connection properties." The progress status is "Ready."

Figure 11-12

New Job – General page

On the Steps page of the New Job dialog box, we will use the New button to invoke the New Job Step dialog box. On the General page of the New Job Step dialog box, we will give our job step a name and configure it to execute the script in Listing [11-5](#395785_2_En_11_Chapter.xhtml#PC5).

```
--Create a table to store log entries
CREATE TABLE #ErrorLog
(
LogDate          DATETIME,
ProcessInfo      NVARCHAR(128),
Text             NVARCHAR(MAX)
) ;
--Populate table with log entries
INSERT INTO #ErrorLog
EXEC('xp_readerrorlog') ;
--Declare variables
DECLARE @SQL NVARCHAR(MAX) ;
DECLARE @DBName NVARCHAR(128) ;
DECLARE @LogName NVARCHAR(128) ;
DECLARE @Subject        NVARCHAR(MAX) ;
--Find database name where error occured
SET @DBName = (
SELECT TOP 1
SUBSTRING(
SUBSTRING(
Text,
35,
LEN(Text)
),
1,
CHARINDEX(
'''',
SUBSTRING(Text,
35,
LEN(Text)
)
)-1
)
FROM #ErrorLog
WHERE Text LIKE 'The transaction log for database%'
ORDER BY LogDate DESC
FOR XML PATH('')
) ;
--Find name of log file that is full
SET @LogName = (
SELECT name
FROM sys.master_files
WHERE type = 1
AND database_id = DB_ID(@DBName)
) ;
--Kill any active queries, to allow clean up to take place
SET @SQL = (
SELECT 'USE Master; KILL ' + CAST(s.session_id AS NVARCHAR(4)) + ' ; '
FROM sys.dm_exec_requests r
INNER JOIN sys.dm_exec_sessions s
ON r.session_id = s.session_id
INNER JOIN sys.dm_tran_active_transactions at
ON r.transaction_id = at.transaction_id
WHERE r.database_id = DB_ID(@DBName)
) ;
EXEC(@SQL) ;
--IF recovery model is SIMPLE
IF (SELECT recovery_model FROM sys.databases WHERE name = @DBName) = 3
BEGIN
--Issue a CHECKPOINT
SET @SQL = (
SELECT 'USE ' + @DBName + ' ; CHECKPOINT'
) ;
EXEC(@SQL) ;
--Shrink the transaction log
SET @SQL = (
SELECT 'USE ' + @DBName + ' ; DBCC SHRINKFILE (' + @LogName + ' , 1)'
) ;
EXEC(@SQL) ;
--e-mail the DBA Team
SET @Subject = (SELECT '9002 Errors on ' + @DBName + ' on Server ' + @@SERVERNAME)
EXEC msdb.dbo.sp_send_dbmail
@profile_name = 'ESASS Administrator',
@recipients = 'DBATeam@ESASS.com',
@body = 'A CHECKPOINT has been issued and the Log has been shrunk',
@subject = @Subject ;
END
--If database in full recovery model
IF (SELECT recovery_model FROM sys.databases WHERE name = @DBName) = 1
BEGIN
--If reuse delay is not because of replication or mirroring/availability groups
IF (SELECT log_reuse_wait FROM sys.databases WHERE name = @DBName) NOT IN (5,6)
BEGIN
--Backup transaction log
SET @SQL = (
SELECT 'BACKUP LOG '
+ @DBName
+ ' TO  DISK = ''C:\Backups\'
+ @DBName
+ '.bak'' WITH NOFORMAT, NOINIT,  NAME = '''
+ @DBName
+ '-Full Database Backup'', SKIP ;'
) ;
EXEC(@SQL) ;
--Shrink the transaction log
SET @SQL =  (
SELECT 'USE ' + @DBName + ' ; DBCC SHRINKFILE (' + @LogName + ' , 1)'
) ;
EXEC(@SQL) ;
--e-mail the DBA Team
SET @Subject = (SELECT '9002 Errors on ' + @DBName + ' on Server ' + @@SERVERNAME) ;
EXEC msdb.dbo.sp_send_dbmail
@profile_name = 'ESASS Administrator',
@recipients = 'DBATeam@ESASS.com',
@body = 'A Log Backup has been issued and the Log has been shrunk',
@subject = @Subject ;
END
--If reuse delay is because of replication or mirroring/availability groups
ELSE
BEGIN
--e-mail DBA Team
SET @Subject = (SELECT '9002 Errors on ' + @DBName + ' on Server ' + @@SERVERNAME) ;
EXEC msdb.dbo.sp_send_dbmail
@profile_name = 'ESASS Administrator',
@recipients = 'DBATeam@ESASS.com',
@body = 'DBA intervention required - 9002 errors due to HA/DR issues',
@subject = @Subject ;
END
END
Listing 11-5
Respond to 9002 Errors
```

Tip

This script assumes that you have Database Mail configured. A full discussion of Database Mail is beyond the scope of this book. Further details can be found at [`https://learn.microsoft.com/en-us/sql/relational-databases/database-mail/configure-database-mail?view=sql-server-ver16`](https://learn.microsoft.com/en-us/sql/relational-databases/database-mail/configure-database-mail?view=sql-server-ver16).

This is illustrated in Figure [11-13](#395785_2_En_11_Chapter.xhtml#Fig13).

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig13_HTML.jpg)

Screenshot of a "New Job Step" window in SQL Server Management Studio. The window includes fields for "Step name," "Type," and "Run as," with "RespondTo9002" and "Transact-SQL script (T-SQL)" filled in. The "Database" is set to "master." The "Command" section contains SQL code for creating a table named "#ErrorLog" with columns "LogDate," "ProcessInfo," and "Text," and inserting log entries. The left panel shows "Select a page" options, and the "Connection" section displays server details. Buttons for "Open," "Select All," "Copy," "Paste," and "Parse" are visible.

Figure 11-13

New Job Step – General page

After exiting the New Job Step dialog box and the New Job dialog box, our new job will automatically be selected in the Execute job drop-down list on the Response page of the New Alert dialog box, as shown in Figure [11-14](#395785_2_En_11_Chapter.xhtml#Fig14).

![](images/395785_2_En_11_Chapter/395785_2_En_11_Fig14_HTML.jpg)

Screenshot of a "New Alert" configuration window in a server management interface. The window includes options to execute a job titled "RespondTo9002 (Uncategorized Local)" with buttons for "New Job" and "View Job." There is a section to notify operators, though no operators are listed. The left panel shows navigation options like General, Response, and Options, with the Response tab selected. Connection details display "Server: ESASSMGMT1" and "Connection: ESASSMGMT1\Pete." A progress indicator at the bottom shows "Ready."

Figure 11-14

Response page

We do not need to configure anything on the Options page of the New Alert dialog box. You should be aware of the options that can be configured on this page. A very useful option is the ability to provide a delay between responses. Configuring this option can help you avoid the possibility of “false” responses being fired while a previous response is still in the process of resolving an issue.

## Summary

Automating routine maintenance tasks can save a DBA’s valuable time and also improve the service to the business. Of course, it may not be possible to get to a position where 100% of maintenance is automated, because there are always exceptions. It is a good idea to aim for an 80/20 split of automated vs. exception. As well as automating tasks within the database engine, such as intelligent index rebuilds, as you have learned in previous chapters, it is also possible to automate tasks that have operating system elements, such as patching.

The more break/fix scenarios that you can automate, the better and more proactive your service to the business will become. It is not feasible to attempt to automate responses to all break/fix scenarios in one go, however. This is because problems will change and vary over time and because such a large-scale effort would likely be cost-prohibitive. Instead, you should look to add automated break/fix scenarios as a continuous service improvement (CSI) program. It is helpful to use the problem management functionality of your ticket system to help define candidate issues for automation.

Index A Add-SqlLogin cmdlet Ad hoc routine maintenance Ad hoc commands Ad hoc query plans Aggregation interval Aliases APPLY operator Automated installation Automating ad hoc maintenance break/fix scenarios 9002 errors installation SeeSQL Server installation multiple approaches scripts Automation See alsoAutomating AUTO mode Availability Groups Azure Azure SQL Database B Break/fix scenarios C Centralized maintenance column layout configuration data inventory database PowerShell Scripts schedule column scheduling system tables and view target CertificateBackups CLR SeeCommon Language Runtime (CLR) Comments Common Language Runtime (CLR) Comparison operators Config database Configuration management ongoing operations operational resources performance requirements performance resources security requirements security resources Continuous service improvement (CSI) program Controlling flow Credential Credentials CROSS APPLY operator CSI program SeeContinuous service improvement (CSI) program D Database as a Service (DBaaS) Database users addittion role dropping users Data-tier applications Data types DateTimeOffset DBaaS SeeDatabase as a Service (DBaaS) DBATeam DBATeamSysadmin resource DbaTools database users installation SQL Server Agent artifacts DependOn clause Desired state configuration (DSC) configuration management configurations implementation management component process Windows Server DMF SeeDynamic management functions (DMF) DMV SeeDynamic management views (DMV) Dropping users DSC SeeDesired state configuration (DSC) Dynamic management functions (DMF) Dynamic management views (DMV) E ELSE block ELSEIF blocks EndTime Error handling 9002 errors Errors EvaluateAsExpression Execute SQL Task Execution context Execution policy Exist method eXtensible Markup Language (XML) description execution plan extracting values FOR XML AUTO FOR XML PATH FOR XML RAW sales order values extraction F FEATURES parameter Filtering First normal form (1NF) FLOWR statements Forcing failure reasons FOR XML modes FOR XML PATH(‘’) technique G Get-Command-module sqlserver Get-DbaDbUser command Get-ExecutionPolicy command Get-SqlLogin command Gold disk approach H $HelloWorldText Hierarchy level I, J, K IaaS SeeInfrastructure as a Service (IaaS) IDENTITY column Infrastructure as a Service (IaaS) Instance installation ACTION Parameter command line parameters FEATURES parameter IACCEPTSQLSERVERLICENSETERMS installation progress optional parameters product update role parameter Inventory database centralized maintenance techniques creation database logical design physical design platform design See alsoNormalization Invoke-DbaQuery command error handling parameters queries against multiple instances Invoke-SqlCmd execution context Out-File cmdlet parameters scripting ITSM 365 L LastExecTime Logins dropping renaming server role Looping M MaintenanceSchedules Columns MaintenanceSchedule view MaintenanceTasks MaintenanceTasks Columns MaintenanceTasks table MaintenanceWindows table MaintenanceWindows Columns Many-to-many relationships MAXDOP SeeMaximum degree of parallelism (MAXDOP) Maximum degree of parallelism (MAXDOP) Metadata-driven automation ad hoc query plans automation dynamic index policies enforcement SQL Server Microsoft Operations Framework (MOF) Modules Install-Module command modular approach SqlServer MOF SeeMicrosoft Operations Framework (MOF) N .NET CLR New-DbaAgentJob command New-DbaAgentJobStep command New-DbaAgentSchedule command New-DbaDbMasterKey command Nodes method Normalization first normal form (1NF) second normal form (2NF) testing third normal form (3NF) NVARCHAR(128) O 1NF SeeFirst normal form (1NF) Operational resources Optional parameters OUTER APPLY operator Out-File cmdlet Out-of-the-box functionality OutputSqlErrors parameter P PaaS SeePlatform as a Service (PaaS) PATH mode PBM SeePolicy-based management (PBM) Piping Platform as a Service (PaaS) Policy-based management (PBM) PowerShell aliases comments configuration management controlling flow data types description DSC SeeDesired state configuration (DSC) execution policy modules object-oriented language piping & filtering scripts standards variables PowerShell Get-Service cmdlet Process flow, 9002 errors Product updation Proxy Q Query method Query plans Query Store R Remove-DbaDbUser command Remove-DBALogin command Retrieving job information Role parameter S SalesOrderID Scripts auto-install installation routine PowerShell production readiness Scripting Second normal form (2NF) Security objects availability groups credentials database user DbaTools module login creation miscellaneous Cmdlets server roles sqlserver module working with logins working with server roles Security requirements Security resources Server Agent credential job creation Proxy retrieving job information Server roles Service Account ServiceNow Shell Database Smoke tests SMO SeeSQL Server Management Objects (SMO) sp_configure SQLAgentCredential SQL Agent job SQLAgentProxy SqlConfiguration resource SqlMaxDop resource SQL Server Agent SQL Server AlwaysOn SqlServerDsc module SQL Server Enterprise SQL Server installation ACTION parameter required parameters SQL Server Management Objects (SMO) SQL Server Management Studio (SSMS) Sqlserver module Invoke-SqlCmd SQL Task SQL Server Analysis Services (SSAS) SSIS package SSAS SeeSQL Server Analysis Services (SSAS) SSMS SeeSQL Server Management Studio (SSMS) Standards StartTime sys.configurations sys.dm_db_index_physical_stats sys.dm_query_store_plan sys.query_store_query catalog sys.query_store_runtime_stats T, U Task Scheduler TempDB system Testing normalization Third normal form (3NF) 3NF SeeThird normal form (3NF) ToString() method Transitive dependencies T-SQL techniques APPLY operator looping values extraction XML data type XML for DBAs 2NF SeeSecond normal form (2NF) V Value method Variables W WHERE clause Windows Server Windows Server Core Installation configuration file auto-install script PROSQLADMINCONF2 SQLAutoInstall.ps1 SQLPROSQLADMINCONF1 instance SeeInstance Installation X, Y, Z XML SeeeXtensible Markup Language (XML) XML Schema Definition (XSD) xp_cmdshell XQuery XSD SeeXML Schema Definition (XSD)