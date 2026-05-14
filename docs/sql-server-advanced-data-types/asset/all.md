![](index-1_1.jpg)

![](index-1_2.png)

SQL Server

Advanced

Data Types

JSON, XML, and Beyond

—

Taking full advantage of SQL Server’s

ever-growing support for modern data

—

Peter A. Carter

**SQL Server Advanced**

**Data Types**

**JSON, XML, and Beyond**

**Peter A. Carter**

***SQL Server Advanced Data Types: JSON, XML, and Beyond***

Peter A. Carter

London, UK

ISBN-13 (pbk): 978-1-4842-3900-1 ISBN-13 (electronic): 978-1-4842-3901-8

[https://doi.org/10.1007/978-1-4842-3901-8](https://doi.org/10.1007/978-1-4842-3901-8)

Library of Congress Control Number: 2018955129

Copyright © 2018 by Peter A. Carter

This work is subject to copyright. All rights are reserved by the Publisher, whether the whole or part of the material is concerned, specifically the rights of translation, reprinting, reuse of illustrations, recitation, broadcasting, reproduction on microfilms or in any other physical way, and transmission or information storage and retrieval, electronic adaptation, computer software, or by similar or dissimilar methodology now known or hereafter developed.

Trademarked names, logos, and images may appear in this book. Rather than use a trademark symbol with every occurrence of a trademarked name, logo, or image, we use the names, logos, and images only in an editorial fashion and to the benefit of the trademark owner, with no intention of infringement of the trademark.

The use in this publication of trade names, trademarks, service marks, and similar terms, even if they are not identified as such, is not to be taken as an expression of opinion as to whether or not they are subject to proprietary rights.

While the advice and information in this book are believed to be true and accurate at the date of publication, neither the author nor the editors nor the publisher can accept any legal responsibility for any errors or omissions that may be made. The publisher makes no warranty, express or implied, with respect to the material contained herein.

Managing Director, Apress Media LLC: Welmoed Spahr

Acquisitions Editor: Jonathan Gennick

Development Editor: Laura Berendson

Coordinating Editor: Jill Balzano

Cover designed by eStudioCalamar

Cover image designed by Freepik (www.freepik.com)

Distributed to the book trade worldwide by Springer Science+Business Media New York, 233

Spring Street, 6th Floor, New York, NY 10013\. Phone 1-800-SPRINGER, fax (201) 348-4505, e-mail orders-ny@springer-sbm.com, or visit www.springeronline.com. Apress Media, LLC is a California LLC and the sole member (owner) is Springer Science+Business Media Finance Inc (SSBM Finance Inc). SSBM Finance Inc is a **Delaware** corporation.

For information on translations, please e-mail rights@apress.com, or visit www.apress.com/

rights-permissions.

Apress titles may be purchased in bulk for academic, corporate, or promotional use. eBook versions and licenses are also available for most titles. For more information, reference our Print and eBook Bulk Sales web page at www.apress.com/bulk-sales.

Any source code or other supplementary material referenced by the author in this book is available to readers on GitHub via the book’s product page, located at www.apress.

com/9781484239001\. For more detailed information, please visit www.apress.com/source-code.

Printed on acid-free paper

*Given the cover art, this book could*

*only be dedicated to my son, Reuben.*

**Table of Contents**

[About the Author ���������������������������������������������������������������������������������xi](#index_split_000.html#p10)

[About the Technical Reviewer �����������������������������������������������������������xiii](#index_split_000.html#p11)

[Introduction ����������������������������������������������������������������������������������������xv](#index_split_000.html#p12)

[Chapter 1:](#index_split_000.html#p13) [SQL Server Data Types ��������������������������������������������������������1](#index_split_000.html#p13)

[Numeric Data Types ����������������������������������������������������������������������������������������������1](#index_split_000.html#p13)

[Character Strings ������������������������������������������������������������������������������������������������10](#index_split_000.html#p22)

[Binary Data Types �����������������������������������������������������������������������������������������������14](#index_split_000.html#p26)

[Dates and Times �������������������������������������������������������������������������������������������������17](#index_split_000.html#p29)

[Miscellaneous Standard Data Types �������������������������������������������������������������������23](#index_split_000.html#p35)

[Summary of Advanced Data Types ����������������������������������������������������������������������25](#index_split_000.html#p37)

[Why Is Using the Correct Data Type Important? ��������������������������������������������������26](#index_split_000.html#p38)

[Summary�������������������������������������������������������������������������������������������������������������27](#index_split_000.html#p39)

[Chapter 2](#index_split_000.html#p41): [Understanding XML �����������������������������������������������������������29](#index_split_000.html#p41)

[Understanding XML ���������������������������������������������������������������������������������������������29](#index_split_000.html#p41)

[Well-Formed XML������������������������������������������������������������������������������������������������38](#index_split_000.html#p50)

[Understanding XSD Schemas �����������������������������������������������������������������������������43](#index_split_000.html#p55)

[XML Usage Scenarios in SQL Server �������������������������������������������������������������������46](#index_split_000.html#p58)

[Summary�������������������������������������������������������������������������������������������������������������47](#index_split_000.html#p59)

v

Table of Con

T

T

able of Con enT

en s

[Chapter 3:](#index_split_000.html#p61) [Constructing XML with T-SQL ��������������������������������������������49](#index_split_000.html#p61)

[Using FOR XML RAW �������������������������������������������������������������������������������������������49](#index_split_000.html#p61)

[Using FOR XML AUTO ������������������������������������������������������������������������������������������77](#index_split_000.html#p89)

[Using FOR XML PATH ������������������������������������������������������������������������������������������92](#index_split_001.html#p104)

[Using FOR XML EXPLICIT �����������������������������������������������������������������������������������101](#index_split_001.html#p113)

[Summary�����������������������������������������������������������������������������������������������������������111](#index_split_001.html#p123)

[Chapter 4](#index_split_001.html#p125): [Querying and Shredding XML �����������������������������������������113](#index_split_001.html#p125)

[Querying XML ����������������������������������������������������������������������������������������������������113](#index_split_001.html#p125)

[Using exist() �������������������������������������������������������������������������������������������������116](#index_split_001.html#p128)

[Using value( ) �����������������������������������������������������������������������������������������������118](#index_split_001.html#p130)

[Using query( ) ����������������������������������������������������������������������������������������������121](#index_split_001.html#p133)

[Using Relational Values in XQuery ���������������������������������������������������������������123](#index_split_001.html#p135)

[FLWOR ���������������������������������������������������������������������������������������������������������127](#index_split_001.html#p139)

[Modifying XML Data ������������������������������������������������������������������������������������133](#index_split_001.html#p145)

[Shredding XML ��������������������������������������������������������������������������������������������������141](#index_split_001.html#p153)

[Shredding XML with OPENXML( ) ����������������������������������������������������������������141](#index_split_001.html#p153)

[Shredding XML with Nodes �������������������������������������������������������������������������146](#index_split_001.html#p158)

[Using Schemas �������������������������������������������������������������������������������������������������152](#index_split_001.html#p164)

[Summary�����������������������������������������������������������������������������������������������������������155](#index_split_001.html#p167)

[Chapter 5](#index_split_001.html#p169): [XML Indexes ��������������������������������������������������������������������157](#index_split_001.html#p169)

[Preparing the Environment �������������������������������������������������������������������������������157](#index_split_001.html#p169)

[Clustered Indexes ���������������������������������������������������������������������������������������������159](#index_split_001.html#p171)

[Tables Without a Clustered Index ����������������������������������������������������������������160](#index_split_001.html#p172)

[Tables with a Clustered Index ����������������������������������������������������������������������161](#index_split_001.html#p173)

[Clustering the Primary Key ��������������������������������������������������������������������������162](#index_split_001.html#p174)

vi

Table of Con

T

T

able of Con enTs

[Performance Considerations for Clustered Indexes ������������������������������������164](#index_split_001.html#p176)

[Creating a Clustered Index ��������������������������������������������������������������������������165](#index_split_001.html#p177)

[Primary XML Indexes ����������������������������������������������������������������������������������������167](#index_split_001.html#p179)

[Creating Primary XML Indexes ��������������������������������������������������������������������168](#index_split_001.html#p180)

[Secondary XML Indexes ������������������������������������������������������������������������������������171](#index_split_001.html#p183)

[Creating Secondary XML Indexes ����������������������������������������������������������������172](#index_split_001.html#p184)

[Performance Considerations for XML Indexes ��������������������������������������������������175](#index_split_001.html#p187)

[Summary�����������������������������������������������������������������������������������������������������������179](#index_split_001.html#p191)

[Chapter 6](#index_split_001.html#p192): [Understanding JSON �������������������������������������������������������181](#index_split_001.html#p192)

[Understanding the JSON Format ����������������������������������������������������������������������181](#index_split_001.html#p192)

[JSON vs� XML ����������������������������������������������������������������������������������������������������188](#index_split_001.html#p199)

[JSON Usage Scenarios��������������������������������������������������������������������������������������195](#index_split_002.html#p206)

[n-Tier Applications with Rest APIs ���������������������������������������������������������������195](#index_split_002.html#p206)

[De-Normalizing Data �����������������������������������������������������������������������������������195](#index_split_002.html#p206)

[Config As Code ��������������������������������������������������������������������������������������������197](#index_split_002.html#p208)

[Analyzing the Log Data ��������������������������������������������������������������������������������199](#index_split_002.html#p210)

[Summary�����������������������������������������������������������������������������������������������������������199](#index_split_002.html#p210)

[Chapter 7](#index_split_002.html#p212): [Constructing JSON from T-SQL ����������������������������������������201](#index_split_002.html#p212)

[FOR JSON AUTO ������������������������������������������������������������������������������������������������201](#index_split_002.html#p212)

[Working with Root Nodes ����������������������������������������������������������������������������204](#index_split_002.html#p215)

[Working with NULL Values ���������������������������������������������������������������������������207](#index_split_002.html#p218)

[Using Column Aliases ����������������������������������������������������������������������������������212](#index_split_002.html#p223)

[Automatic Nesting ���������������������������������������������������������������������������������������215](#index_split_002.html#p226)

[FOR JSON PATH �������������������������������������������������������������������������������������������������224](#index_split_002.html#p235)

[Summary�����������������������������������������������������������������������������������������������������������228](#index_split_002.html#p239)

vii

Table of Con

T

T

able of Con enT

en s

[Chapter 8](#index_split_002.html#p240): [Shredding JSON Data ������������������������������������������������������229](#index_split_002.html#p240)

[OPENJSON() with Default Schema ��������������������������������������������������������������������229](#index_split_002.html#p240)

[Shredding a Column ������������������������������������������������������������������������������������232](#index_split_002.html#p243)

[Dynamic Shredding Based on Document Content ���������������������������������������235](#index_split_002.html#p246)

[OPENJSON( ) with Explicit Schema �������������������������������������������������������������������237](#index_split_002.html#p248)

[OPENJSON( ) with Path Expressions �����������������������������������������������������������������242](#index_split_002.html#p253)

[Shredding Data into Tables ��������������������������������������������������������������������������246](#index_split_002.html#p257)

[Summary�����������������������������������������������������������������������������������������������������������249](#index_split_002.html#p260)

[Chapter 9](#index_split_002.html#p262): [Working with the JSON Data Type ����������������������������������251](#index_split_002.html#p262)

[Querying JSON Data ������������������������������������������������������������������������������������������251](#index_split_002.html#p262)

[Using ISJSON( )��������������������������������������������������������������������������������������������251](#index_split_002.html#p262)

[Using JSON_VALUE( ) ����������������������������������������������������������������������������������254](#index_split_002.html#p265)

[Using JSON_QUERY( ) ����������������������������������������������������������������������������������261](#index_split_002.html#p272)

[Using JSON_MODIFY() ���������������������������������������������������������������������������������266](#index_split_002.html#p277)

[Indexing JSON Data ������������������������������������������������������������������������������������������271](#index_split_002.html#p282)

[Summary�����������������������������������������������������������������������������������������������������������276](#index_split_002.html#p287)

[Chapter 10: Understanding Spatial Data ������������������������������������������279](#index_split_002.html#p289)

[Understanding Spatial Data ������������������������������������������������������������������������������279](#index_split_002.html#p289)

[Spatial Data Standards �������������������������������������������������������������������������������������286](#index_split_002.html#p296)

[Well-Known Text ������������������������������������������������������������������������������������������287](#index_split_002.html#p297)

[Well-Known Binary ��������������������������������������������������������������������������������������289](#index_split_002.html#p299)

[Spatial Reference Systems �������������������������������������������������������������������������291](#index_split_002.html#p301)

[SSMS and Spatial Data �������������������������������������������������������������������������������������294](#index_split_002.html#p304)

[Summary�����������������������������������������������������������������������������������������������������������296](#index_split_002.html#p306)

viii

Table of Con

T

T

able of Con enTs

[Chapter 11: Working with Spatial Data �������������������������������������������299](#index_split_002.html#p308)

[Constructing Spatial Data ���������������������������������������������������������������������������������300](#index_split_002.html#p309)

[Querying Spatial Data ���������������������������������������������������������������������������������������307](#index_split_002.html#p316)

[Indexing Spatial Data ����������������������������������������������������������������������������������������332](#index_split_003.html#p341)

[Understanding Spatial Indexes ��������������������������������������������������������������������332](#index_split_003.html#p341)

[Creating Spatial Indexes ������������������������������������������������������������������������������334](#index_split_003.html#p343)

[Summary�����������������������������������������������������������������������������������������������������������340](#index_split_003.html#p349)

[Chapter 12: Working with Hierarchical Data and HierarchyID���������341](#index_split_003.html#p350)

[Hierarchical Data Use Cases �����������������������������������������������������������������������������342](#index_split_003.html#p351)

[Modeling Traditional Hierarchies �����������������������������������������������������������������������344](#index_split_003.html#p353)

[Modeling Hierarchies with HierarchyID �������������������������������������������������������������350](#index_split_003.html#p359)

[HierarchyID Methods �����������������������������������������������������������������������������������������356](#index_split_003.html#p365)

[Working with HierarchyID Methods �������������������������������������������������������������358](#index_split_003.html#p367)

[Indexing HierarchyID Columns ��������������������������������������������������������������������������377](#index_split_003.html#p386)

[Summary�����������������������������������������������������������������������������������������������������������382](#index_split_003.html#p391)

[Index �������������������������������������������������������������������������������������������������385](#index_split_003.html#p393)

ix

![](index-10_1.jpg)

**About the Author**

**Peter A.** **Carter** is a SQL Server expert with

more than 15 years’ experience in database

development, administration, and platform

engineering. He is currently a consultant,

based in London. Peter has written a number

of books across a variety of SQL Server topics,

including security, high availability, and

automation.

xi

![](index-11_1.jpg)

**About the Technical Reviewer**

**Ian Stirk** is a freelance SQL Server consultant

based in London. In addition to his day job, he

is an author, creator of software utilities, and

technical reviewer who regularly writes book

re[views for www.i-programmer.info](http://www.i-programmer.info/).

He covers every aspect of SQL Server and

has a specialist interest in performance and

scalability. If you require help with your SQL

Server systems, feel free to contact him at

ian_stirk@yahoo.com or [www.linkedin.com/](http://www.linkedin.com/in/ian-stirk-bb9a31)

[in/ian-stirk-bb9a31](http://www.linkedin.com/in/ian-stirk-bb9a31).

Ian would like to thank Peter Carter, Jonathan Gennick, and Jill

Balzano for making this book experience easier for him.

None of us stands alone, and with this in mind, Ian would like to

thank these special people: Pat Richards, Bhargava Ganti, Jon McCabe,

Nick Fairway, Aida Samuel, Paul Fuller, Vikki Singini, Rob Lee, Gerald

Hemming, and Jordy Mumba.

Ian’s fee for his work on this book has been donated to the Deworm the

World Initiativ[e (www.givingwhatwecan.org/report/deworm/).](http://www.givingwhatwecan.org/report/deworm/)

xiii

**Introduction**

*SQL Server Advanced Data Types* attempts to demystify the complex data types that are available to developers in modern versions of SQL Server.

Over the last couple of years, I have noticed that many SQL developers have heard of each of the complex data types available in SQL Server but often avoid using them, as they are not sure how these are best utilized. This

has led to suboptimal solutions being developed, such as an incident that I recently experienced in which a very good and seasoned SQL developer

implemented complex hierarchical logic using self joins, because he did

not feel confident implementing the HierarchyID data type.

That inspired me to write this book—to help SQL and other developers

responsible for writing T-SQL, as part of their applications, to better

understand the complex data types available in SQL Server and give them

the confidence to use these complex structures appropriately.

The book starts by exploring the simple, conventional data types that

are available in SQL Server and reminds readers why making the right

choices about data types can be so important. The book then moves on

to offer in-depth discussions about the complex data types in SQL Server, namely, XML, JSON, HierarchyID, GEOGRAPHY, and GEOMETRY. Many of the

code examples in the book are based on real-world problems and solutions

that I have encountered in my time as an SQL Server consultant in London.

Many of the code examples in this book use the WideWorldImporters

sample database. The GitHub repo for this database can be found at

github.com/Microsoft/sql-server-samples/tree/master/samples/

databases/wide-world-importers and the .bak file can be downloaded

from github.com/Microsoft/sql-server-samples/releases/download/

wide-world-importers-v1.0/WideWorldImporters-Standard.bak.

xv

**CHAPTER 1**

**SQL Server**

**Data Types**

SQL Server 2017 provides a wide range of basic data types that can store

data, such as character strings, numeric data, binary data, and dates. In this chapter, I will review each of the basic data types available in SQL

Server, looking at the differences between similar data types. I will also discuss the importance of using the correct data type for each use case.

**Tip** The following sections discuss the acceptable ranges for each data type. It is worth noting, however, that all data types can also be

set to NULL, indicating that the value is unknown.

**Numeric Data Types**

Some numeric data types in SQL Server are exact, while others are

approximate. If a data type is exact, it stores a number to a fixed precision.

If a data type is approximate, in many cases, it will store a very close

approximation of a number, instead of the number itself.

The numeric data types supported by SQL Server are listed in Table [1- 1](#index_split_000.html#p14).

© Peter A. Carter 2018

1

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_1](#index_split_000.html#p13)

ChApTEr 1 SQL SErvEr DATA TypES

**Exact/**

**Approximate**

Exact

Exact

Exact

Exact

Exact

**Range**

0 to 1

0 to 255

to 2^15 to 2^15 to 1

to 2^31 to 2^31 to 1

to 2^63 to 2^63 to 1

y 8 bit

**Storage Size**

1 byte for ever

columns in a table

1 byte

2 bytes

4 bytes

8 bytes

.

BIT

the string

ta type

respectively

can also be

*es*

, with no decimal

, with no decimal

, with no decimal

, with no decimal

*yp*

False

*ata T*

and

t can be set to 0 or 1,

*ic D*

True

g tha

*umer*

**Description**

A fla

values

inserted into a column of da

and are converted to 1 or 0,

Stores a whole number

points

Stores a whole number

points

Stores a whole number

points

Stores a whole number

points

***N***

 ****ype**

***Table 1-1.***

**Data T**

BIT

TINYINT

SMALLINT

INT

BIGINT

2

ChApTEr 1 SQL SErvEr DATA TypES

*d*) *ue*

*tin*

Exact

( *con*

to 10^38+

1 to 10^38 to 1

as

ge space

w:

The stora

required depends

on the precision,

belo

• 1-9 – 5 bytes

• 10-19 – 9 bytes

• 20-28 – 13 bytes

• 29-38 – 17 bytes

t

.

, you

The

is

while the scale

or example, F

NUMERIC

DECIMAL

DECIMAL

using

When

t can be stored,

w the decimal point.

Stores a number with fixed decimal

precision.

must specify precision and scale.

precision is the maximum number of

digits tha

is the maximum number of digits tha

follo

123.456 could be stored with a precision

of 6 and a scale of 3.

functionally equivalent to

DECIMAL

3

ChApTEr 1 SQL SErvEr DATA TypES

**Exact/**

**Approximate**

Exact

**Range**

to 10^38+1 to

10^38 to 1

as

ge space

w:

**Storage Size**

The stora

required depends

on the precision,

belo

• 1-9 – 5 bytes

• 10-19 – 9 bytes

• 20-28 - 13 bytes

• 29-38 – 17 bytes

t

.

, you

The

is

while the scale

or example, F

DECIMAL

NUMERIC

NUMERIC

using

*d*)

When

t can be stored,

*ue*

*tin*

w the decimal point.

**Description**

Stores a number with fixed decimal

precision.

must specify precision and scale.

precision is the maximum number of

digits tha

is the maximum number of digits tha

follo

123.456 could be stored with a precision

of 6 and a scale of 3.

functionally equivalent to

**( *con***

 ****ype**

***Table 1-1.***

**Data T**

NUMERIC

4

ChApTEr 1 SQL SErvEr DATA TypES

*d*)

te

te

*ue*

*tin*

Approxima

Approxima

( *con*

to

Single precision:

1.79E+308

to 2.23E to 308

Double precision:

2.23E to 308 to

1.79E+308

1.79E+308

to 2.23E to 308

as

precision

depends

precision

ge space

w:

(7 digits) – 4 bytes

(15 digits) – 8

bytes

The amount of

stora

required,

on the precision,

belo

• Single

• sDouble

4 bytes

te

ted to 24,

te

wn as double

pproxima

If a value between

(24)

single precision is

pproxima

it is transla

is used without

wn as single precision.

also kno

.

T accepts a single precision

FLOAT

with single precision is a

uses single precision and

FLOAT

REAL

FLOA

ym for

ted as 54,

If

REAL

FLOAT A ym for

Stores a number with a

precision.

value between 1 and 53.

1 and 24 is entered,

which is also kno

If a number between 25 and 54 is passed,

it is trea

precision.

specifying a precision,

used.

synon

Stores a number with a

precision.

is a synon

FLOAT

REAL

5

ChApTEr 1 SQL SErvEr DATA TypES

**Exact/**

**Approximate**

Exact

Exact

**Range**

to 214,748.3648 to

214,748.3647

to922,337,203,685,

477.5808 to

922,337,203,685,

477.5807

**Storage Size**

4 bytes

8 bytes

y

y

Currenc

Currenc

can be passed

can be passed

y values with a

.

y values with a

.

ver

ver

*d*)

ta type as a prefix but are not

ta type as a prefix but are not

such as £ or $,

such as £ or $,

*ue*

*tin*

**Description**

Stores small monetar

precision of four decimal places.

symbols,

into the da

stored by SQL Ser

Stores large monetar

precision of four decimal places.

symbols,

into the da

stored by SQL Ser

**( *con***

 ****ype**

***Table 1-1.***

**Data T**

SMALLMONEY

MONEY

6

ChApTEr 1 SQL SErvEr DATA TypES

**Tip** The caret (^) operator means to the power. For example,

2^15-1 means 2 to the power of 15 minus 1, which equates to

32,767.

To see how a number behaves, using each of the numeric formats, we

will use the number 2.5888712345678923456789 and convert it to each of

the numeric data types, to examine its behavior. In SQL Server, there are two functions that we can use to change the data type of a value: CAST and CONVERT.

The CAST function uses the syntax in Listing [1-1](#index_split_000.html#p19).

***Listing 1-1.*** CAST Syntax

CAST( expression AS datatype[(length)] )

The expression can be either a value that you wish to convert, a

column in a table, or any other valid expression. The datatype should be

the target data type for the expression. length is optional and based on

the target data type. For example, if you were using CAST to change the

data type of an INT to a DECIMAL, length would consist of the required

precision and scale. If you were casting a value to a string, the length would be the maximum length of the string.

The CONVERT function uses the syntax in Listing [1-2](#index_split_000.html#p19).

***Listing 1-2.*** CONVERT Syntax

CONVERT( datatype[(length)], expression [, style] )

The style option of the CONVERT function allows the developer to

supply additional information about how to translate the expression.

The style options that are available for numeric data types can be found

in Ta[ble 1-2.](#index_split_000.html#p20)

7

ChApTEr 1 SQL SErvEr DATA TypES

***Table 1-2.** Numeric Data Type Style Options*

**Style Code**

**Data Types**

**Output**

0

FLOAT & REAL

The default value for FLOAT and REAL.

A maximum of six digits. Uses scientific

notation, if required

1

FLOAT & REAL

Eight digits. Always uses scientific

notation

2

FLOAT & REAL

Sixteen digits. Always uses scientific

notation

3

FLOAT & REAL

Seventeen digits, with lossless conversion

0

SMALLMONEY & MONEY

The default value for SMALLMONEY and

MONEY. No comma separation, and two

digits to the right of the decimal point

1

SMALLMONEY & MONEY

Comma separation every three digits on

the left of the decimal point. Two digits to

the right of the decimal point

2

SMALLMONEY & MONEY

No comma separation. Four digits to the

right of the decimal point

126

SMALLMONEY & MONEY

Used when converting to character data

types. No comma separation. Four digits

to the right of the decimal point

The script in Listin[g 1-3](#index_split_000.html#p21) shows how to use the CAST function to translate the number 2.5888712345678923456789 to each numeric data type.

**Note** BIT is excluded, as the conversion does not make sense. If it were included, however, it would CAST to 1.

8

ChApTEr 1 SQL SErvEr DATA TypES

***Listing 1-3.*** Casting a Number to Each Data Type

SELECT

CAST(2.5888712345678923456789 AS TINYINT) AS 'TINYINT'

, CAST(2.5888712345678923456789 AS SMALLINT) AS

'SMALLINT'

, CAST(2.5888712345678923456789 AS INT) AS 'INT'

, CAST(2.5888712345678923456789 AS BIGINT) AS 'BIGINT'

, CAST(2.5888712345678923456789 AS DECIMAL(23,22)) AS

'DECIMAL'

, CAST(2.5888712345678923456789 AS DECIMAL(18,17)) AS

'DECIMAL ROUNDED'

, CAST(2.5888712345678923456789 AS NUMERIC(23,22)) AS

'NUMERIC'

, CAST(2.5888712345678923456789 AS FLOAT(24)) AS

'SINGLE FLOAT'

, CAST(2.5888712345678923456789 AS FLOAT(53)) AS

'DOUBLE FLOAT'

, CAST(2.5888712345678923456789 AS REAL) AS 'REAL'

, CAST(2.5888712345678923456789 AS SMALLMONEY) AS

'SMALLMONEY'

, CAST(2.5888712345678923456789 AS MONEY) AS 'MONEY'

9

![](index-22_1.png)

ChApTEr 1 SQL SErvEr DATA TypES

The results of the query in Listing [1-3](#index_split_000.html#p21) can be found in Figur[e 1-1.](#index_split_000.html#p22)

***Figure 1-1.** Results of casting to numeric data types*

**Character Strings**

SQL Server can store both Unicode and non-Unicode strings. Strings can

also be stored either as a fixed length or a variable length. The character data types supported by SQL Server are listed in Ta[ble 1-3.](#index_split_000.html#p23)

10

ChApTEr 1 SQL SErvEr DATA TypES

*d*) *ue*

*tin*

( *con*

**Variable/Fixed**

Fixed

variable

Fixed

**Maximum**

**Length**

8000

characters

2GB

4000

characters

**Storage Size**

Maximum string

length × 1 byte

The length of the

string stored × 1

byte

Maximum string

length × 2 bytes

If

the

with a

t is stored.

ta type

the da

, you must pass in a

CHAR

, you must pass in a value,

*es*

, you must either specify a

is specified,

*yp*

MAX

CHAR

te the maximum length of the string.

*ata T*

When using

depending on the string tha

VARCHAR

using

. When

MAX

te the maximum length of the string.

to indica

When

*haracter D*

**Description**

Stores a string of non-Unicode characters,

fixed length.

value,

If the string is shorter than the maximum length,

the string will be padded with whitespace.

Stores a string of non-Unicode characters with a

variable size,

When using

maximum length for the string or

specify

can store up to 2GB.

Stores a string of Unicode characters with a fixed

length.

to indica

the string is shorter than the maximum length,

string will be padded with whitespace.

***C***

 ****ype**

***Table 1-3.***

**Data T**

CHAR

VARCHAR

NCHAR

11

ChApTEr 1 SQL SErvEr DATA TypES

**Variable/Fixed**

variable

variable

variable

**Maximum**

**Length**

2GB

2GB

2GB

**Storage Size**

The length of the

string stored × 2

bytes

The length of the

string stored × 1

byte

The length of the

string stored × 2

bytes

t is

ta type

the da

, you must either

t should not be used.

t should not be used.

is specified,

VARCHAR

MAX

ta type tha

ta type tha

depending on the string tha

using

. When

ted da

ted da

*d*) *ue*

When

MAX

*tin*

**Description**

Stores a string of Unicode characters with a

variable size,

stored.

specify a maximum length for the string or

specify

can store up to 2GB.

A depreca

Stores non-Unicode strings with a variable length

A depreca

Stores Unicode strings with a variable length

*( *con**

 ***ype**

***Table 1-3.***

**Data T**

NVARCHAR

TEXT

NTEXT

12

![](index-25_1.png)

ChApTEr 1 SQL SErvEr DATA TypES

The script in Listin[g 1-4](#index_split_000.html#p25) uses the DATALENGTH system function to demonstrate the difference in storage size for a 15-character string, cast as each character data type.

***Listing 1-4.*** Examining the Storage Size of Strings

SELECT

DATALENGTH(CAST('My String Value' AS NCHAR(20))) AS

'NCHAR'

, DATALENGTH(CAST('My String Value' AS NVARCHAR(20))) AS

'NVARCHAR'

, DATALENGTH(CAST('My String Value' AS CHAR(20))) AS 'CHAR'

, DATALENGTH(CAST('My String Value' AS VARCHAR(20))) AS

'VARCHAR'

The results of Listing [1-4](#index_split_000.html#p25) can be found in Figur[e 1-2.](#index_split_000.html#p25)

***Figure 1-2.** Results of comparing string storage sizes*

13

ChApTEr 1 SQL SErvEr DATA TypES

**Binary Data Types**

SQL Server can store binary data, such as a word document or a photo,

using native binary data types. Binary data types are also used to store data that has been encrypted using a key or certificate. The binary data types available in SQL Server are detailed in Table [1-4](#index_split_000.html#p26).

***Table 1-4.** Binary Data Types*

**Data Type**

**Description**

**Storage Size**

**Maximum Variable/**

**Length**

**Fixed**

BINARY

Stores binary data with

Equal to

8000

Fixed

a fixed length. When

maximum

bytes

using BINARY, you must

number of

specify the data length in

allowed bytes

bytes. If the data stored is

shorter than the specified

length, it will be padded.

VARBINARY

Stores binary data with a

Equal to the

2GB

variable

variable length. When using actual number

VARBINARY, you must

of bytes stored,

specify either the maximum plus 2 bytes

length of the data in bytes

or specify MAX. When MAX

is specified, data up to a

maximum of 2GB can be

stored.

IMAGE

A deprecated data type

Equal to the

2GB

variable

that should not be used.

actual number

Stores binary data with a of bytes stored,

variable length

plus 2 bytes

14

ChApTEr 1 SQL SErvEr DATA TypES

**Tip** For detailed information about encrypting data, please refer to *Securing SQL Server* (Apress, 2016), which can be found at

[www.apress.com/gp/book/9781484222645.](http://www.apress.com/gp/book/9781484222645)

The style options available for BINARY data when using the CONVERT

function are detailed in Table [1-5](#index_split_000.html#p27).

***Table 1-5.** Style Options for BINARY Data*

**Style Code**

**Output**

0

The default value for binary data. Converts ASCII characters to binary

bytes, or vice versa

1

Converts a character string into binary data. validates that there are an even number of hexadecimal bytes and that the first character is 0x

2

Converts binary data into a character string. Each byte will be

converted into two hexadecimal characters. Data that overflows the

data type will be truncated. If the data is shorter than a fixed length

data type, it will be padded.

The script in Listin[g 1-5](#index_split_000.html#p27) demonstrates how to read a password that has been encrypted and stored in a VARBINARY column and convert it back to a

character string. The script first creates the required objects and encrypts the password.

***Listing 1-5.*** Decrypting an Encrypted Password and Converting It Back to a Character String

--Create a certificate that will encrypt the symmetric key

CREATE CERTIFICATE PasswordCert

ENCRYPTION BY PASSWORD = 'MySecurePa$$word'

15

ChApTEr 1 SQL SErvEr DATA TypES

WITH SUBJECT = 'Cert for securing passwords table' ;

--Create a symmetric key that will encrypt the password

CREATE SYMMETRIC KEY PasswordKey

WITH ALGORITHM = AES_128

ENCRYPTION BY CERTIFICATE PasswordCert ;

--Create a table to store the password

CREATE TABLE dbo.Passwords

(

Password VARBINARY(256)

) ;

--Open the symmetric key, so that it can be used

OPEN SYMMETRIC KEY PasswordKey

DECRYPTION BY CERTIFICATE PasswordCert

WITH PASSWORD = 'MySecurePa$$word' ;

--Encrypt a password and insert it into the table

INSERT INTO dbo.Passwords

SELECT ENCRYPTBYKEY(KEY_GUID('PasswordKey'), 'Pa$$w0rd') ;

--Decrypt and read the password

--The first column in the result set shows the password as the

decrypted value but still binary format

--The second column in the result set shows the password

decrypted and converted back to a character string

SELECT

DECRYPTBYKEY(Password) AS 'Decrypted Password In

Binary'

, CAST(DECRYPTBYKEY(Password) AS CHAR(8)) AS 'Decrypted

Password As Character String'

16

![](index-29_1.jpg)

ChApTEr 1 SQL SErvEr DATA TypES

FROM dbo.Passwords

--Close the symmetric key

CLOSE SYMMETRIC KEY PasswordKey ;

The results of the select statement against the table created in

Listin[g 1-5 c](#index_split_000.html#p27)an be found in Figure [1-3](#index_split_000.html#p29).

***Figure 1-3.** Results of converting binary data to a character string* **Dates and Times**

SQL Server can store accurate date and time information, including UTC

offsets. The details of each supported date and time data type can be found in Ta[ble 1-6.](#index_split_000.html#p30)

17

ChApTEr 1 SQL SErvEr DATA TypES

.003,

**y**

te

**Accurac**

Da

100 nanoseconds

rounded to .000,

or .007 seconds

One minute

ws:

as follo

**Storage Size**

3 bytes

Depending on the fractional

second precision,

• 0-2 – 3 bytes

• 3-4 – 4 bytes

• 5-7 – 5 bytes

8 bytes

4 bytes

*esyp*

. When using

to a maximum of 7.

*ata T*

te

the default is 7.

te and a time with fractional

te and a time to the grain of

*ime D*

, you can specify a fractional

*d Tn*

**Description**

Stores a da

Stores a time of day

TIME

second precision,

sIf omitted,

Stores a da

seconds

Stores a da

1 minute

*ate a*

***D***

 ****ype**

***Table 1-6.***

**Data T**

DATE

TIME

DATETIME

SMALLDATETIME

18

ChApTEr 1 SQL SErvEr DATA TypES

100 nanoseconds

100 nanoseconds

ws:

as follo

Depending on the fractional

second precision,

• 0-2 – 6 bytes

• 3-4 – 7 bytes

• 5-7 – 8 bytes

10 bytes

you

is

and

When

to a

to a

tes.

the

the default

with time

te and time,

DATETIME2

DATETIME

, you can specify

, you can specify

When using

If omitted,

If omitted,

te and time with

te than

te and time,

DATETIME2

wareness.

When storing a da

Stores a da

fractional seconds.

more accura

stores a larger range of da

using

a fractional second precision,

maximum of 7.

default is 7

Stores a da

zone a

DATETIMEOFFSET

a fractional second precision,

maximum of 7.

is 7.

can pass a UTC offset of -14 to +14

DATETIME2

DATETIMEOFFSET

19

ChApTEr 1 SQL SErvEr DATA TypES

Table [1-7](#index_split_000.html#p32) details allowable style options for date and time data types when using the CONVERT function.

***Table 1-7.** Date and Time Styles*

**Style Code**

**Standard**

**Input/Output**

0 or 100

Default for datetime and

mon dd yyyy hh:miAM (or pM)

smalldatetime

1

US

mm/dd/yy

2

ANSI

yy.mm.dd

3

British & French

dd/mm/yy

4

German

dd.mm.yy

5

Italian

dd-mm-yy

6

dd mon yy

7

Mon dd, yy

8 or 108

hh:mi:ss

9 or 109

Default style (100) + time

mon dd yyyy

(ms)

hh:mi:ss:mmmAM (or pM)

10

USA

mm-dd-yy

11

Japan

yy/mm/dd

12

ISO

yymmdd

13 or 113

dd mon yyyy hh:mi:ss:mmm(24h)

20 or 120

ODBC canonical

yyyy-mm-dd hh:mi:ss(24h)

21 or 121

ODBC canonical with time

yyyy-mm-dd hh:mi:ss.mmm(24h)

(ms) Default for time, date,

datetime2, and datetimeoffset

( *continued*)

20

ChApTEr 1 SQL SErvEr DATA TypES

***Table 1-7.*** ( *continued*)

**Style Code**

**Standard**

**Input/Output**

101

US

mm/dd/yyyy

102

ANSI

yyyy.mm.dd

1-3

British & French

dd/mm/yyyy

104

German

dd.mm.yyyy

105

Italian

dd-mm-yyyy

106

European default

dd mon yyyy

107

Mon dd, yyyy

110

USA

mm-dd-yyyy

111

Japan

yyyy/mm/dd

112

ISO

yyyymmdd

13 or 113

European

dd mon yyyy hh:mi:ss:mmm(24h)

default (106) + time (ms)

114

hh:mi:ss:mmm(24h)

126

ISO8601

yyyy-mm-ddThh:mi:ss.mmm

mmmm not displayed if 0

127

ISO8601 with time zone

yyyy-mm-ddThh:mi:ss.mmmZ

mmmm not displayed if 0

130

hijri

dd mon yyyy hh:mi:ss:mmmAM

mon is the hijri Unicode

representation of the month name

131

hijri

dd/mm/yyyy hh:mi:ss:mmmAM

mon is the hijri Unicode

representation of the month name

21

![](index-34_1.jpg)

ChApTEr 1 SQL SErvEr DATA TypES

The script in Listin[g 1-6 sho](#index_split_000.html#p34)ws how a date and time value will be displayed when cast to each of the date and time data types. The script uses the SYSUTCDATETIME function to retrieve the current system date and time.

***Listing 1-6.*** Casingt a Value to Each of the Date and Time Data Types SELECT

CAST(SYSUTCDATETIME() AS time(7)) AS 'TIME'

,CAST(SYSUTCDATETIME() AS date) AS 'DATE'

,CAST(SYSUTCDATETIME() AS smalldatetime) AS 'SMALLDATETIME'

,CAST(SYSUTCDATETIME() AS datetime) AS 'DATETIME'

,CAST(SYSUTCDATETIME() AS datetime2(7)) AS 'DATETIME2'

, CAST(SYSUTCDATETIME() AS datetimeoffset(7)) AS

'DATETIMEOFFSET' ;

The results of running the query in Listin[g 1-6 c](#index_split_000.html#p34)an be found in Figur[e 1-4.](#index_split_000.html#p34)

***Figure 1-4.** Results of casting a value to each date and time data type* 22

ChApTEr 1 SQL SErvEr DATA TypES

**Miscellaneous Standard Data Types**

SQL Server offers many other standard data types that can be used for

specialized purposes. A description of these data types can be found in

Table [1-8](#index_split_000.html#p35).

***Table 1-8.** Miscellaneous Data Types*

**Data Type**

**Description**

**Size**

**Allowable Usage**

CURSOR

Stores a cursor with

variable variable or stored

a variable or stored

procedure OUTPUT

procedure output

parameter

TIMESTAMP

Exposes a system-

8 bytes

Table, variable,

generated, unique binary

stored procedure

value. Used for versioning

parameter

a row within a table

UNIQUEIDENTIFIER

Stored a GUID (Globally

16 bytes Table, variable,

Unique Identifier)

stored procedure

parameter

SQL VARIANT

Can store data of multiple variable Table, variable,

data types. Can be used

stored procedure

when you do not know

parameter

the data type of the data

that will be input, but this

is a very bad practice

TABLE

Stores a result set within variable variable

a variable, to be used at a

later time

23

![](index-36_1.png)

ChApTEr 1 SQL SErvEr DATA TypES

The script in Listin[g 1-7](#index_split_000.html#p36) creates a value of the data type UNIQUEIDENTIFIER, which has been created using the NEWID system

function.

***Listing 1-7.*** Creating a UNIQUEIDENTIFIER

SELECT NEWID() ;

The results of running the query in Listin[g 1-7 c](#index_split_000.html#p36)an be found in Figur[e 1-5.](#index_split_000.html#p36)

***Figure 1-5.** Viewing a UNIQUEIDENTIFIER created with NEWID*

24

ChApTEr 1 SQL SErvEr DATA TypES

**Summary of Advanced Data Types**

SQL Server provides several advanced data types. These data types will be discussed throughout this book, but a summary can be found in Table [1-9](#index_split_000.html#p37).

***Table 1-9.** Advanced Data Types*

**Data Type**

**Description**

HIERARCHYID HierarchyID can be used to store hierarchical data, such as a BoM (bill of materials) of department structure, in a hexadecimal

form. The data type exposes numerous methods that allow

developers to easily navigate the hierarchy.

XML

Used to store data in native XML format. FLWOR (discussed in

Chapter [4) sta](#index_split_001.html#p125)tements can then be used against the data.

JSON

Stores data in a native JSON format. SQL Server provides functions

to convert data to and from JSON, extract values from JSON, and

modify JSON data.

GEOGRAPHY

Stores location data, using the round earth model. The data type

exposes a number of methods that can be used to interact with the

geospatial data.

GEOMETRY

Stores location data, using the flat earth model. The data type

exposes a number of methods that can be used to interact with the

geospatial data.

25

ChApTEr 1 SQL SErvEr DATA TypES

The available style options when using the CONVERT function with XML

data are detailed in Table [1-10](#index_split_000.html#p38).

***Table 1-10.** Style Options for XML Data*

**Style Code**

**Output**

0

The default value. Discards unneeded whitespace and does not allow

an internal DTD to be created

1

preserves all whitespace

2

Discards unneeded whitespace but allows limited DTD processing

3

preserves whitespace and allows limited DTD processing

**Why Is Using the Correct Data Type**

**Important?**

The most commonly used, and most commonly misused, data type in SQL

Server is INT. When developers must store whole numbers, INT is always

the default choice. Often, however, developers must only store small

values. An INT uses 4 bytes of data, whereas if a SMALLINT would suffice, only 2 bytes of storage would be used. In cases in which you have small

lookup tables, with less than 255 rows, a TINYINT could be used as the

primary key, only using 1 byte of storage per row.

You’re probably thinking to yourself at this point that storage is

relatively cheap these days. Do 2 or 3 bytes of data here and there really matter? To answer that question, you should not only think in terms of

storage but in terms of memory usage and application performance.

Imagine that you have a fact table containing 1 billion rows that

you must join to five dimension tables, each containing 30,000 rows.

If you have used the INT data type for the primary key in each of these

dimensions, that column would be 117KB in each of the dimension

26

ChApTEr 1 SQL SErvEr DATA TypES

tables, but the corresponding foreign key column in the fact table would

be 3.72GB! Now let’s multiply that by 5, so that each dimension table is

covered. We are now at 18.6GB. This is how much data SQL Server will

have to read from disk into memory before probably spooling out again,

to TempDB, in order to join the tables in your query. If we had used the

SMALLINT data type for our dimension tables instead, SQL Server would

only have to consume 9.3GB of data.

Another consideration when choosing data types is ensuring that

the same data type is used when the same data is stored in multiple

columns. For example, imagine that you have an ETL process that pulls

data from an OLTP database into a data warehouse. You may populate the

data warehouse by running queries that join or filter tables on columns

such as dates. If the data types are not consistent between columns—for

example, a date is stored as the DATE data type in one table and stored as a DATETIME2 in another column—then implicit casting operations will

occur. These have negative impacts on performance and even stop optimal

indexes from being used.

**Tip** If data type casting is required, I generally recommend

doing this manually, with the CAST or CONVERT functions, instead

of allowing implicit conversions. This can make code easier to

understand when debugging

**Summary**

SQL Server provides a large amount of data types out of the box that

will accommodate the needs of most development scenarios. Numeric

values can be stored as whole numbers, or with either exact or

approximate decimal precision. Character data can be stored with Unicode

or non- Unicode character sets. SQL Server can also store dates, times, or both. You also have the option of recording time zone information.

27

ChApTEr 1 SQL SErvEr DATA TypES

Values can be converted between different data types using either the

CAST or CONVERT functions. When using the CONVERT function, additional

translation information can be provided, via styles.

When developing using SQL Server, it is very important to use the

correct data type. When choosing a data type, a developer should use the

most limiting data type that will store all required values. If a data type with a more expansive range is erroneously used, then, as a database scales, a severe impact of memory utilization and performance may be witnessed.

28

**CHAPTER 2**

**Understanding XML**

XML (Extensible Markup Language) is a markup language, similar to

HTML, that was designed for the purpose of storing and transporting data.

Like HTML, XML consists of tags. Unlike HTML, however, these tags are

not predefined. Instead, they are defined by the document author. An XML

document has a tree structure, beginning with a root node and containing

child nodes (also known as child elements). Each element can contain

data but also 1..n attributes.

This chapter will assist you in your understanding of XML, by

explaining the structure of an XML document, before diving into the

difference between XML fragments and well-formed XML, XML schemas,

and how XML data is often used in SQL Server.

**Understanding XML**

Each attribute can contain data that describes the element. For example,

imagine that you require details of sales orders to be stored in XML format.

It would be sensible to assume that each sales order be stored in a separate element within the document. But what about that sales order properties,

such as order date, customer ID, product IDs, quantities, and prices?

These pieces of information could either be stored as child elements of the sales order element, or they could be stored as attributes of the sales order element. There are no set rules for when you should use child elements

© Peter A. Carter 2018

29

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_2](#index_split_000.html#p41)

Chapter 2 Understanding XML

or attributes to describe properties of an element. This choice is at the discretion of the document author.

The XML document in Listin[g 2-1](#index_split_000.html#p42) provides a sample XML document that holds the details of sales orders for a fictional organization. In this example, the document author has chosen to use an element to store each sales order and a nested element to store each line item of the order. The details of each sales order and line item are stored in attributes of the elements.

***Listing 2-1.*** Sales Orders Stored Using Attribute-Centric Approach

<SalesOrders>

<SalesOrder OrderDate="2013-03-07" CustomerID="57" OrderID="3168">

<LineItem StockItemID="176" Quantity="5" UnitPrice="240.00" />

<LineItem StockItemID="143" Quantity="108"

UnitPrice="18.00" />

<LineItem StockItemID="136" Quantity="3" UnitPrice="32.00" />

<LineItem StockItemID="92" Quantity="48" UnitPrice="18.00" />

</SalesOrder>

<SalesOrder OrderDate="2013-03-22" CustomerID="57" OrderID="4107">

<LineItem StockItemID="153" Quantity="40" UnitPrice="4.50" />

<LineItem StockItemID="36" Quantity="9" UnitPrice="13.00" />

<LineItem StockItemID="208" Quantity="108" UnitPrice="2.70" />

</SalesOrder>

<SalesOrder OrderDate="2013-04-09" CustomerID="57" OrderID="4980">

<LineItem StockItemID="102" Quantity="10" UnitPrice="35.00" />

<LineItem StockItemID="144" Quantity="24" UnitPrice="18.00" />

<LineItem StockItemID="79" Quantity="36" UnitPrice="18.00" />

<LineItem StockItemID="217" Quantity="10" UnitPrice="25.00" />

</SalesOrder>

<SalesOrder OrderDate="2016-01-09" CustomerID="57" OrderID="64608">

<LineItem StockItemID="156" Quantity="40" UnitPrice="15.00" />

<LineItem StockItemID="56" Quantity="7" UnitPrice="13.00" />

</SalesOrder>

30

Chapter 2 Understanding XML

<SalesOrder OrderDate="2016-05-25" CustomerID="57"

OrderID="73148">

<LineItem StockItemID="31" Quantity="7" UnitPrice="13.00" />

<LineItem StockItemID="103" Quantity="2" UnitPrice="35.00" />

</SalesOrder>

</SalesOrders>

The XML in Listing [2-1](#index_split_000.html#p42) can be generated by running the query in Listin[g 2-2 a](#index_split_000.html#p43)gainst the WideWorldImporters database.

***Listing 2-2.*** Generating Attribute-Centric XML

SELECT

SalesOrder.OrderDate

, SalesOrder.CustomerID

, SalesOrder.OrderID

, LineItem.StockItemID

, LineItem.Quantity

, LineItem.UnitPrice

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON LineItem.OrderID = SalesOrder.OrderID

WHERE SalesOrder.OrderID IN

(

3168,

4107,

4980,

64608,

73148

)

FOR XML AUTO, ROOT('SalesOrders') ;

31

Chapter 2 Understanding XML

**Note** the

FOR XML clause is used to convert the results of a query

into XML format. FOR XML is discussed in Cha[pter 3.](#index_split_000.html#p61)

The XML document in Listing [2-2](#index_split_000.html#p43) stores the same information as the document in Listing [2-3](#index_split_000.html#p44). This time, however, the document author has used an element-centric approach, instead of an attribute-centric

approach. Therefore, the data is stored in child elements, as opposed to

attributes.

***Listing 2-3.*** Sales Orders Stored Using Element-Centric Approach

<SalesOrders>

<SalesOrder>

<OrderDate>2013-03-07</OrderDate>

<CustomerID>57</CustomerID>

<OrderID>3168</OrderID>

<LineItem>

<StockItemID>176</StockItemID>

<Quantity>5</Quantity>

<UnitPrice>240.00</UnitPrice>

</LineItem>

<LineItem>

<StockItemID>143</StockItemID>

<Quantity>108</Quantity>

<UnitPrice>18.00</UnitPrice>

</LineItem>

<LineItem>

<StockItemID>136</StockItemID>

<Quantity>3</Quantity>

<UnitPrice>32.00</UnitPrice>

</LineItem>

32

Chapter 2 Understanding XML

<LineItem>

<StockItemID>92</StockItemID>

<Quantity>48</Quantity>

<UnitPrice>18.00</UnitPrice>

</LineItem>

</SalesOrder>

<SalesOrder>

<OrderDate>2013-03-22</OrderDate>

<CustomerID>57</CustomerID>

<OrderID>4107</OrderID>

<LineItem>

<StockItemID>153</StockItemID>

<Quantity>40</Quantity>

<UnitPrice>4.50</UnitPrice>

</LineItem>

<LineItem>

<StockItemID>36</StockItemID>

<Quantity>9</Quantity>

<UnitPrice>13.00</UnitPrice>

</LineItem>

<LineItem>

<StockItemID>208</StockItemID>

<Quantity>108</Quantity>

<UnitPrice>2.70</UnitPrice>

</LineItem>

</SalesOrder>

<SalesOrder>

<OrderDate>2013-04-09</OrderDate>

<CustomerID>57</CustomerID>

<OrderID>4980</OrderID>

33

Chapter 2 Understanding XML

<LineItem>

<StockItemID>102</StockItemID>

<Quantity>10</Quantity>

<UnitPrice>35.00</UnitPrice>

</LineItem>

<LineItem>

<StockItemID>144</StockItemID>

<Quantity>24</Quantity>

<UnitPrice>18.00</UnitPrice>

</LineItem>

<LineItem>

<StockItemID>79</StockItemID>

<Quantity>36</Quantity>

<UnitPrice>18.00</UnitPrice>

</LineItem>

<LineItem>

<StockItemID>217</StockItemID>

<Quantity>10</Quantity>

<UnitPrice>25.00</UnitPrice>

</LineItem>

</SalesOrder>

<SalesOrder>

<OrderDate>2016-01-09</OrderDate>

<CustomerID>57</CustomerID>

<OrderID>64608</OrderID>

<LineItem>

<StockItemID>156</StockItemID>

<Quantity>40</Quantity>

<UnitPrice>15.00</UnitPrice>

34

Chapter 2 Understanding XML

</LineItem>

<LineItem>

<StockItemID>56</StockItemID>

<Quantity>7</Quantity>

<UnitPrice>13.00</UnitPrice>

</LineItem>

</SalesOrder>

<SalesOrder>

<OrderDate>2016-05-25</OrderDate>

<CustomerID>57</CustomerID>

<OrderID>73148</OrderID>

<LineItem>

<StockItemID>31</StockItemID>

<Quantity>7</Quantity>

<UnitPrice>13.00</UnitPrice>

</LineItem>

<LineItem>

<StockItemID>103</StockItemID>

<Quantity>2</Quantity>

<UnitPrice>35.00</UnitPrice>

</LineItem>

</SalesOrder>

</SalesOrders>

The XML document in Listing [2-3](#index_split_000.html#p44) can be generated by running the query in Listin[g 2-4.](#index_split_000.html#p47)

***Listing 2-4.*** Generating Element-Centric XML

SELECT

SalesOrder.OrderDate

, SalesOrder.CustomerID

, SalesOrder.OrderID

35

Chapter 2 Understanding XML

, LineItem.StockItemID

, LineItem.Quantity

, LineItem.UnitPrice

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON LineItem.OrderID = SalesOrder.OrderID

WHERE SalesOrder.OrderID IN

(

3168,

4107,

4980,

64608,

73148

)

FOR XML AUTO, ELEMENTS, ROOT('SalesOrders') ;

**Note** the

FOR XML clause is discussed in Chapter [3](#index_split_000.html#p61).

The picture in Figure [2-1](#index_split_000.html#p49) calls out each noteworthy aspect of the attribute-centric document from Listin[g 2-1](#index_split_000.html#p42).

36

![](index-49_1.png)

Chapter 2 Understanding XML

***Figure 2-1.** Aspects of an XML document*

There are several things to note when looking at this XML document.

First, elements begin with the element name, encapsulated within angle

brackets. They end with the element name, preceded by a backslash and

enclosed in angle brackets. Any elements that fall between these two tags are child elements of the tag.

Attributes are enclosed in double quotation marks and reside within

the beginning tag of an element. For example, OrderID is an attribute of

the <SalesOrder> element.

It is acceptable to have repeating elements. You can see that

<SalesOrder> is a repeating element, as two separate sales orders

are stored in this XML document. The <SalesOrders> element is the

document’s root element and is the only element that is not allowed to

be complex. This means that it cannot have attributes and cannot be

repeating. Attributes can never repeat within an element. Therefore, if you require a node to repeat, you should use a nested element as opposed to

an attribute.

37

Chapter 2 Understanding XML

**Well-Formed XML**

SQL Server supports both XML fragments and well-formed XML. Using

well-formed XML is favorable, as there is a degree of validation. Therefore, you should always try to use well-formed XML, where possible. For an

XML document to be well-formed, certain requirements must be met, and

these requirements are listed following:

• XML documents must have a single, non-repeating root

element.

• XML elements must have a closing tag.

• XML elements must be properly nested, with closing

tags in the reverse order of opening tags.

• XML attribute values must be quoted with double-

quotation marks.

• Each attribute name must be unique within an

element.

**Tip** XML tags are case-sensitive.

Although SQL Server supports XML fragments (XML documents

that are not well-formed), the syntax must still be syntactically correct.

For example, consider the script in Listing [2-5](#index_split_000.html#p51). The XML document is syntactically incorrect, because the root node is missing a closing angle bracket.

38

Chapter 2 Understanding XML

***Listing 2-5.*** Syntactically Incorrect XML

DECLARE @Example XML ;

SET @Example =

'<SalesOrder OrderDate="2013-03-07" CustomerID="57"

OrderID="3168">

<LineItem StockItemID="176" Quantity="5" UnitPrice="240.00" />

<LineItem StockItemID="143" Quantity="108"

UnitPrice="18.00" />

<LineItem StockItemID="136" Quantity="3" UnitPrice="32.00" />

<LineItem StockItemID="92" Quantity="48" UnitPrice="18.00" />

</SalesOrder>

<SalesOrder OrderDate="2013-03-22" CustomerID="57"

OrderID="4107">

<LineItem StockItemID="153" Quantity="40" UnitPrice="4.50" />

<LineItem StockItemID="36" Quantity="9" UnitPrice="13.00" />

<LineItem StockItemID="208" Quantity="108" UnitPrice="2.70" />

</SalesOrder' ;

SELECT @Example ;

39

![](index-52_1.png)

Chapter 2 Understanding XML

Running the script in Listin[g 2-5 w](#index_split_000.html#p51)ill produce the error shown in Figur[e 2-2.](#index_split_000.html#p52)

***Figure 2-2.** XML syntax error*

If we were to correct the syntax error and rerun the script (Listin[g 2-6](#index_split_000.html#p52)), the script would run and return an XML document as the result set, even

though the document is not well-formed. It is not well-formed, because

there is not a root node.

***Listing 2-6.*** XML Fragment

DECLARE @Example XML ;

SET @Example =

'<SalesOrder OrderDate="2013-03-07" CustomerID="57"

OrderID="3168">

<LineItem StockItemID="176" Quantity="5" UnitPrice="240.00" />

<LineItem StockItemID="143" Quantity="108" UnitPrice="18.00" />

<LineItem StockItemID="136" Quantity="3" UnitPrice="32.00" /> 40

![](index-53_1.png)

Chapter 2 Understanding XML

<LineItem StockItemID="92" Quantity="48" UnitPrice="18.00" />

</SalesOrder>

<SalesOrder OrderDate="2013-03-22" CustomerID="57"

OrderID="4107">

<LineItem StockItemID="153" Quantity="40" UnitPrice="4.50" />

<LineItem StockItemID="36" Quantity="9" UnitPrice="13.00" />

<LineItem StockItemID="208" Quantity="108" UnitPrice="2.70" />

</SalesOrder>' ;

SELECT @Example ;

The results of running the script in Listin[g 2-6 ar](#index_split_000.html#p52)e shown in Figur[e 2-3.](#index_split_000.html#p53)

***Figure 2-3.** XML fragment*

To make the XML document returned, well-formed XML, we would

have to add the root node, as demonstrated in Listin[g 2-7](#index_split_000.html#p54).

41

Chapter 2 Understanding XML

***Listing 2-7.*** Well-Formed XML

DECLARE @Example XML ;

SET @Example =

'<SalesOrders>

<SalesOrder OrderDate="2013-03-07" CustomerID="57"

OrderID="3168">

<LineItem StockItemID="176" Quantity="5" UnitPrice="240.00" />

<LineItem StockItemID="143" Quantity="108" UnitPrice="18.00" />

<LineItem StockItemID="136" Quantity="3" UnitPrice="32.00" />

<LineItem StockItemID="92" Quantity="48" UnitPrice="18.00" />

</SalesOrder>

<SalesOrder OrderDate="2013-03-22" CustomerID="57"

OrderID="4107">

<LineItem StockItemID="153" Quantity="40" UnitPrice="4.50" />

<LineItem StockItemID="36" Quantity="9" UnitPrice="13.00" />

<LineItem StockItemID="208" Quantity="108" UnitPrice="2.70" />

</SalesOrder>

</SalesOrders>' ;

SELECT @Example ;

Newcomers to XML are often confused by the difference between a

well-formed XML document and a valid XML document. Even if an XML

document is well-formed, it is not considered valid, unless its components conform to the details provided in an associated schema. The original

form of the XML schema was known as a DTD (document type definition),

but now it is known as an XSD (XML schema definition). DTD schemas

are not supported by SQL Server, so the focus of this chapter will be on

understanding XSD schemas.

42

Chapter 2 Understanding XML

**Understanding XSD Schemas**

The format of an XML document can be defined by an XSD schema. An

XSD schema will define the document’s structure, including data types, if complex types (complex elements) are allowed, and how many times an

element must occur (or is limited to occurring) within a document. It also defines the sequence of elements and if an element is mandatory. The

main components of an XSD schema are as follows:

• Element declarations, defining the properties of an

elements:

• Element name

• Element default value

• Element type

• Elements integrity constraints

• Attribute declarations, defining the properties of an

attribute

• Attribute name

• Attribute default value

• Attribute type

• Attribute constraints

• Simple and complex types

• Model group and attribute group definitions

• Element particle and attribute use

If an element is bound to a primitive data type and does not include

child elements or attributes, it is a simple type. If an element has child elements, attributes, or other special properties, such as being bound to an ordered sequence, it must be defined as a complex type.

43

Chapter 2 Understanding XML

Model groups and attribute groups are named groups of nodes that can

be reused in multiple type definitions. Element particles and attribute uses define the complex properties of a node. For attributes, this might include the optionality of the node. For an element, this may also be minimum and maximum occurrences of the node.

Listin[g 2-8 sho](#index_split_000.html#p56)ws a schema declaration for the well-formed XML

document in Listing [2-7](#index_split_000.html#p54).

***Listing 2-8.*** XSD Schema

<xs:schema attributeFormDefault="unqualified"

elementFormDefault="qualified" xmlns:xs="http://www.

w3.org/2001/XMLSchema">

<xs:element name="SalesOrders">

<xs:complexType>

<xs:sequence>

<xs:element name="SalesOrder" maxOccurs="unbounded"

minOccurs="0">

<xs:complexType>

<xs:sequence>

<xs:element name="LineItem" maxOccurs="unbounded"

minOccurs="0">

<xs:complexType>

<xs:simpleContent>

<xs:extension base="xs:string">

<xs:attribute type="xs:short"

name="StockItemID" use="optional"/>

<xs:attribute type="xs:byte"

name="Quantity" use="optional"/>

<xs:attribute type="xs:float"

name="UnitPrice" use="optional"/>

</xs:extension>

44

Chapter 2 Understanding XML

</xs:simpleContent>

</xs:complexType>

</xs:element>

</xs:sequence>

<xs:attribute type="xs:date" name="OrderDate"

use="optional"/>

<xs:attribute type="xs:byte" name="CustomerID"

use="optional"/>

<xs:attribute type="xs:short" name="OrderID"

use="optional"/>

</xs:complexType>

</xs:element>

</xs:sequence>

</xs:complexType>

</xs:element>

</xs:schema>

**Tip** there are free online tools that will create an Xsd schema,

based on an XML document. For example, [FreeFormatter.com](http://freeformatter.com)

offers an Xsd generation tool, which can be found at

[www.freeformatter.com/xsd-generator.html.](http://www.freeformatter.com/xsd-generator.html) alternatively, Visual studio can also automatically generate schemas.

An XML document must be well-formed for it to be bound to a

schema. If an XML document is bound to a schema, it is called valid, or

typed, XML. SQL Server supports the use of XSD schemas through XML

schema collections, which are discussed in Chapt[er 4](#index_split_001.html#p125).

45

Chapter 2 Understanding XML

**XML Usage Scenarios in SQL Server**

The ability to work with XML in SQL Server can be very useful, in many

different cases. For example, imagine that you have a middleware

application that pulls sales orders from a web site and pushes them into an SQL Server database. SQL Server’s ability to work with XML data allows a

developer to use native SQL Server tooling to shred that XML sales order

into a relational table structure. XML documents can be shredded into

relational values by using the XQuery Nodes method or the OPENXML()

function. Both will be discussed in Chapt[er 4\. C](#index_split_001.html#p125)onversely, imagine that you must pass sales orders to a middleware system that requires an XML

document. This could also be achieved using SQL Server. You simply run a

query to retrieve the relational data and append the FOR XML clause, which will convert the data to XML. FOR XML will be discussed in Chapt[er 3.](#index_split_000.html#p61)

SQL Server’s support for schema validation also means that an

application can validate its data against a schema supplied by an SQL

Server developer before sending it across the network. SQL Server can

also validate a document against an XML schema before passing it to

an application. Essentially, you can create a data contract between the

different layers of an application, saving time and allowing for better error-handling.

In another use case, for storing data as XML in SQL Server, imagine

that you have an XML document that contains detailed product definitions

and descriptions. You could, of course, simply maintain this file as an XML

file in the file system, but storing the document in SQL Server allows you to query this XML document and join the results to relational information that is stored in an SQL Server database. XML documents can be queried

in SQL Server, using XQuery, which is discussed in Chapt[er 4](#index_split_001.html#p125).

46

Chapter 2 Understanding XML

Many SQL Server data structures are stored in XML format. For

example, the following SQL Server features all rely on XML data structures:

• Extended events

• Event information within DDL triggers

• Event notifications

• Data collectors

Native SQL Server support for XML allows developers and

administrators to interact with this data.

If you are reading this book and come from an application

development background, then, at this point, you might be thinking

“surely, everybody just uses JSON these days. Why bother with XML?”

JSON is a much more lightweight document format, which is discussed

in Chapters [6–](#index_split_001.html#p192)[9 of this b](#index_split_002.html#p262)ook. XML still has functionality that is not supported by JSON, however. For example, if you wish to use the following functionality, you will require XML, rather than JSON:

• Schema validation

• XPath

• Namespaces

• XLST

**Summary**

XML is a markup language like HTML. Instead of having predefined tags,

however, the tags are defined by the document author. An XML document

consists of elements and attributes and can store data in a semi-structured tree format.

47

Chapter 2 Understanding XML

SQL Server supports XML fragments, well-formed XML documents,

and valid (schema-bound) XML documents. To be well-formed, an XML

document must contain a single root node, and an XML document cannot

be schema-bound, unless it is well-formed. The phrases *XML document*

and *XML instance* are often used interchangeably (including in this chapter). Strictly speaking, however, XML document refers to XML that is

well-formed, whereas an XML instance could refer to either a well-formed

XML document or an XML fragment.

XML documents can be bound to a DTD (document type definition)

or an XSD (XML schema definition) schema. SQL Server only supports

storing XSD schemas, however. XSD schemas are implemented in SQL

Server, through an XML schema collection.

48

**CHAPTER 3**

**Constructing XML**

**with T-SQL**

T-SQL allows you to convert relational results sets into XML, by using the FOR

XML clause in your SELECT statement. There are four modes that can be used with the FOR XML clause; FOR XML RAW, FOR XML AUTO, FOR XML PATH, and

FOR XML EXPLICIT. This chapter will demonstrate how the FOR XML clause

works in RAW mode, AUTO mode, PATH mode, and EXPLICIT mode. As the

chapter progresses, we will move from simple to more complex examples.

**Using FOR XML RAW**

The simplest and easiest to understand of the FOR XML modes is FOR XML

RAW. This mode will transform each row in a relational result set into an element within a flat XML document. Consider the query in Listin[g 3-1](#index_split_000.html#p61), which extracts details of sales orders from the WideWorldImporters database.

***Listing 3-1.*** WideWorldImporters Sales Order Query

SELECT

SalesOrder.OrderDate

, Customers.CustomerName

, SalesOrder.OrderID

, LineItem.StockItemID

© Peter A. Carter 2018

49

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_3](#index_split_000.html#p61)

![](index-62_1.png)

Chapter 3 ConstruCting XML with t-sQL

, LineItem.Quantity

, LineItem.UnitPrice

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON LineItem.OrderID = SalesOrder.OrderID

INNER JOIN Warehouse.StockItems Product

ON Product.StockItemID = LineItem.StockItemID

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID = SalesOrder.CustomerID

WHERE customers.CustomerName = 'Agrita Abele' ;

This query extracts the details of orders placed by a customer called

Agrita Abele. A partial output of the query can be found in Figure [3-1](#index_split_000.html#p62).

***Figure 3-1.** WideWorldImporters sales order output*

50

Chapter 3 ConstruCting XML with t-sQL

If we were to add a FOR XML clause using RAW mode, the results would

be returned in the form of an XML fragment. The amended query in

Listin[g 3-2 w](#index_split_000.html#p63)ill return the XML document, instead of a relational result set.

***Listing 3-2.*** WideWorldImporters Sales Orders Using FOR XML RAW

SELECT

SalesOrder.OrderDate

, Customers.CustomerName

, SalesOrder.OrderID

, Product.StockItemName

, LineItem.Quantity

, LineItem.UnitPrice

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON LineItem.OrderID = SalesOrder.OrderID

INNER JOIN Warehouse.StockItems Product

ON Product.StockItemID = LineItem.StockItemID

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID = SalesOrder.CustomerID

WHERE customers.CustomerName = 'Agrita Abele'

FOR XML RAW ;

Listin[g 3-3 illus](#index_split_000.html#p63)trates the XML fragment that is returned.

***Listing 3-3.*** WideWorldImporters Sales Orders Using FOR XML RAW

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72787" StockItemName="Developer joke mug - when your hammer is C++ (Black)" Quantity="2" UnitPrice="13.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72787" StockItemName="Developer joke mug - when your hammer is C++ (White)" Quantity="9" UnitPrice="13.00" /> 51

Chapter 3 ConstruCting XML with t-sQL

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73350" StockItemName="DBA joke mug - you might be a DBA if (Black)" Quantity="4" UnitPrice="13.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73350" StockItemName="Developer joke mug - that's a hardware problem (Black)" Quantity="3" UnitPrice="13.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72770" StockItemName="&quot;The Gu&quot; red shirt XML

tag t-shirt (Black) 5XL" Quantity="120" UnitPrice="18.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72637" StockItemName="&quot;The Gu&quot; red shirt XML

tag t-shirt (Black) XXL" Quantity="24" UnitPrice="18.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73350" StockItemName="Halloween zombie mask (Light Brown) S" Quantity="24" UnitPrice="18.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72669" StockItemName="Halloween skull mask (Gray) M"

Quantity="60" UnitPrice="18.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72787" StockItemName="Furry gorilla with big eyes slippers (Black) XL" Quantity="9" UnitPrice="32.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73350" StockItemName="Tape dispenser (Blue)"

Quantity="90" UnitPrice="32.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72713" StockItemName="Tape dispenser (Red)"

Quantity="20" UnitPrice="32.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72770" StockItemName="Superhero action jacket (Blue) S" Quantity="8" UnitPrice="25.00" />

52

Chapter 3 ConstruCting XML with t-sQL

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73350" StockItemName="RC toy sedan car with remote control (Red) 1/50 scale" Quantity="2" UnitPrice="25.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72637" StockItemName="Superhero action jacket (Blue) 3XS" Quantity="1" UnitPrice="25.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72770" StockItemName="Black and orange handle with care despatch tape 48mmx100m" Quantity="96" UnitPrice="4.10" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72671" StockItemName="Alien officer hoodie (Black) 4XL" Quantity="6" UnitPrice="35.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72637" StockItemName="Furry animal socks (Pink) S"

Quantity="96" UnitPrice="5.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72669" StockItemName="Permanent marker black 5mm nib (Black) 5mm" Quantity="84" UnitPrice="2.70" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73340" StockItemName="Superhero action jacket (Blue) 5XL" Quantity="2" UnitPrice="34.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73356" StockItemName="Chocolate sharks 250g"

Quantity="192" UnitPrice="8.55" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72787" StockItemName="Novelty chilli chocolates 250g"

Quantity="240" UnitPrice="8.55" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72671" StockItemName="Packing knife with metal insert blade (Yellow) 9mm" Quantity="35" UnitPrice="1.89" /> 53

Chapter 3 ConstruCting XML with t-sQL

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72671" StockItemName="Shipping carton (Brown)

480x270x320mm" Quantity="150" UnitPrice="2.74" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72770" StockItemName="Shipping carton (Brown)

356x229x229mm" Quantity="175" UnitPrice="1.14" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72787" StockItemName="Ride on big wheel monster truck (Black) 1/12 scale" Quantity="4" UnitPrice="345.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72669" StockItemName="10 mm Anti static bubble wrap (Blue) 10m" Quantity="70" UnitPrice="26.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73340" StockItemName="32 mm Double sided bubble wrap 50m" Quantity="20" UnitPrice="112.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72669" StockItemName="Office cube periscope (Black)"

Quantity="20" UnitPrice="18.50" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73356" StockItemName="Ride on vintage American toy coupe (Black) 1/12 scale" Quantity="10" UnitPrice="285.00" /> The first thing that we should note about this XML instance is that it is an XML fragment, as opposed to a well-formed XML document, because

there is no root node. The <row> element cannot be the root node, because it repeats. This means that we cannot validate the XML against a schema.

Therefore, when using the FOR XML clause, you should consider using

the ROOT keyword. This will force a root element, with a name of your

choosing, to be created within the document. This is demonstrated in

Listin[g 3-4.](#index_split_000.html#p67)

54

Chapter 3 ConstruCting XML with t-sQL

***Listing 3-4.*** Adding a Root Node

SELECT

SalesOrder.OrderDate

, Customers.CustomerName

, SalesOrder.OrderID

, Product.StockItemName

, LineItem.Quantity

, LineItem.UnitPrice

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON LineItem.OrderID = SalesOrder.OrderID

INNER JOIN Warehouse.StockItems Product

ON Product.StockItemID = LineItem.StockItemID

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID = SalesOrder.CustomerID

WHERE customers.CustomerName = 'Agrita Abele'

FOR XML RAW, ROOT('SalesOrders') ;

Partial output of the resulting well-formed XML document can be

found in Listin[g 3-5.](#index_split_000.html#p67)

***Listing 3-5.*** Output of XML Document with Root Node

<SalesOrders>

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72787" StockItemName="Developer joke mug - when your hammer is C++ (Black)" Quantity="2" UnitPrice="13.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72787" StockItemName="Developer joke mug - when your hammer is C++ (White)" Quantity="9" UnitPrice="13.00" /> 55

Chapter 3 ConstruCting XML with t-sQL

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73350" StockItemName="DBA joke mug - you might be a DBA if (Black)" Quantity="4" UnitPrice="13.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73350" StockItemName="Developer joke mug - that's a hardware problem (Black)" Quantity="3" UnitPrice="13.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72770" StockItemName="&quot;The Gu&quot; red shirt XML

tag t-shirt (Black) 5XL" Quantity="120" UnitPrice="18.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72637" StockItemName="&quot;The Gu&quot; red shirt XML

tag t-shirt (Black) XXL" Quantity="24" UnitPrice="18.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73350" StockItemName="Halloween zombie mask (Light Brown) S" Quantity="24" UnitPrice="18.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72669" StockItemName="Halloween skull mask (Gray) M"

Quantity="60" UnitPrice="18.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72787" StockItemName="Furry gorilla with big eyes slippers (Black) XL" Quantity="9" UnitPrice="32.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73350" StockItemName="Tape dispenser (Blue)"

Quantity="90" UnitPrice="32.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72713" StockItemName="Tape dispenser (Red)"

Quantity="20" UnitPrice="32.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72770" StockItemName="Superhero action jacket (Blue) S" Quantity="8" UnitPrice="25.00" />

56

Chapter 3 ConstruCting XML with t-sQL

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73350" StockItemName="RC toy sedan car with remote control (Red) 1/50 scale" Quantity="2" UnitPrice="25.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72637" StockItemName="Superhero action jacket (Blue) 3XS" Quantity="1" UnitPrice="25.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72770" StockItemName="Black and orange handle with care despatch tape 48mmx100m" Quantity="96" UnitPrice="4.10" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72671" StockItemName="Alien officer hoodie (Black) 4XL" Quantity="6" UnitPrice="35.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72637" StockItemName="Furry animal socks (Pink) S"

Quantity="96" UnitPrice="5.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72669" StockItemName="Permanent marker black 5mm nib (Black) 5mm" Quantity="84" UnitPrice="2.70" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73340" StockItemName="Superhero action jacket (Blue) 5XL" Quantity="2" UnitPrice="34.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73356" StockItemName="Chocolate sharks 250g"

Quantity="192" UnitPrice="8.55" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72787" StockItemName="Novelty chilli chocolates

250g" Quantity="240" UnitPrice="8.55" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72671" StockItemName="Packing knife with metal

insert blade (Yellow) 9mm" Quantity="35" UnitPrice="1.89" /> 57

Chapter 3 ConstruCting XML with t-sQL

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72671" StockItemName="Shipping carton (Brown)

480x270x320mm" Quantity="150" UnitPrice="2.74" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72770" StockItemName="Shipping carton (Brown)

356x229x229mm" Quantity="175" UnitPrice="1.14" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"

OrderID="72787" StockItemName="Ride on big wheel monster

truck (Black) 1/12 scale" Quantity="4" UnitPrice="345.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72669" StockItemName="10 mm Anti static bubble wrap (Blue) 10m" Quantity="70" UnitPrice="26.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73340" StockItemName="32 mm Double sided bubble wrap 50m" Quantity="20" UnitPrice="112.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"

OrderID="72669" StockItemName="Office cube periscope (Black)"

Quantity="20" UnitPrice="18.50" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"

OrderID="73356" StockItemName="Ride on vintage American toy coupe (Black) 1/12 scale" Quantity="10" UnitPrice="285.00" />

</SalesOrders>

The other important thing to note about the document is that it is

completely flat. There is no nesting. This means that the document’s

granularity is at the level of line item, which does not make a lot of sense.

It is also worthy of note that all data is contained in attributes, as

opposed to elements. We can alter this behavior by using the ELEMENTS

keyword in the FOR XML clause. The ELEMENTS keyword will cause all data

to be contained within child elements, as opposed to attributes. This is

demonstrated in the modified query that can be found in Listing [3-6](#index_split_000.html#p71).

58

Chapter 3 ConstruCting XML with t-sQL

***Listing 3-6.*** Using the ELEMENTS Keyword

SELECT

SalesOrder.OrderDate

, Customers.CustomerName

, SalesOrder.OrderID

, Product.StockItemName

, LineItem.Quantity

, LineItem.UnitPrice

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON LineItem.OrderID = SalesOrder.OrderID

INNER JOIN Warehouse.StockItems Product

ON Product.StockItemID = LineItem.StockItemID

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID = SalesOrder.CustomerID

WHERE customers.CustomerName = 'Agrita Abele'

FOR XML RAW, ELEMENTS, ROOT('SalesOrders') ;

The well-formed XML document that is returned can be seen in

Listin[g 3-7.](#index_split_000.html#p71)

***Listing 3-7.*** Using the ELEMENTS Keyword

<SalesOrders>

<row>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72787</OrderID>

<StockItemName>Developer joke mug - when your hammer is C++

(Black)</StockItemName>

<Quantity>2</Quantity>

<UnitPrice>13.00</UnitPrice>

</row>

59

Chapter 3 ConstruCting XML with t-sQL

<row>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72787</OrderID>

<StockItemName>Developer joke mug - when your hammer is C++

(White)</StockItemName>

<Quantity>9</Quantity>

<UnitPrice>13.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73350</OrderID>

<StockItemName>DBA joke mug - you might be a DBA if

(Black)</StockItemName>

<Quantity>4</Quantity>

<UnitPrice>13.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73350</OrderID>

<StockItemName>Developer joke mug - that's a hardware

problem (Black)</StockItemName>

<Quantity>3</Quantity>

<UnitPrice>13.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72770</OrderID>

60

Chapter 3 ConstruCting XML with t-sQL

<StockItemName>"The Gu" red shirt XML tag t-shirt (Black) 5XL</StockItemName>

<Quantity>120</Quantity>

<UnitPrice>18.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72637</OrderID>

<StockItemName>"The Gu" red shirt XML tag t-shirt (Black) XXL</StockItemName>

<Quantity>24</Quantity>

<UnitPrice>18.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73350</OrderID>

<StockItemName>Halloween zombie mask (Light Brown) S</

StockItemName>

<Quantity>24</Quantity>

<UnitPrice>18.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72669</OrderID>

61

Chapter 3 ConstruCting XML with t-sQL

<StockItemName>Halloween skull mask (Gray)

M</StockItemName>

<Quantity>60</Quantity>

<UnitPrice>18.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72787</OrderID>

<StockItemName>Furry gorilla with big eyes slippers (Black)

XL</StockItemName>

<Quantity>9</Quantity>

<UnitPrice>32.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73350</OrderID>

<StockItemName>Tape dispenser (Blue)</StockItemName>

<Quantity>90</Quantity>

<UnitPrice>32.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72713</OrderID>

<StockItemName>Tape dispenser (Red)</StockItemName>

<Quantity>20</Quantity>

<UnitPrice>32.00</UnitPrice>

</row>

62

Chapter 3 ConstruCting XML with t-sQL

<row>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72770</OrderID>

<StockItemName>Superhero action jacket (Blue)

S</StockItemName>

<Quantity>8</Quantity>

<UnitPrice>25.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73350</OrderID>

<StockItemName>RC toy sedan car with remote control (Red)

1/50 scale</StockItemName>

<Quantity>2</Quantity>

<UnitPrice>25.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72637</OrderID>

<StockItemName>Superhero action jacket (Blue)

3XS</StockItemName>

<Quantity>1</Quantity>

<UnitPrice>25.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72770</OrderID>

63

Chapter 3 ConstruCting XML with t-sQL

<StockItemName>Black and orange handle with care despatch

tape 48mmx100m</StockItemName>

<Quantity>96</Quantity>

<UnitPrice>4.10</UnitPrice>

</row>

<row>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72671</OrderID>

<StockItemName>Alien officer hoodie (Black)

4XL</StockItemName>

<Quantity>6</Quantity>

<UnitPrice>35.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72637</OrderID>

<StockItemName>Furry animal socks (Pink) S</StockItemName>

<Quantity>96</Quantity>

<UnitPrice>5.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72669</OrderID>

<StockItemName>Permanent marker black 5mm nib (Black)

5mm</StockItemName>

<Quantity>84</Quantity>

<UnitPrice>2.70</UnitPrice>

</row>

64

Chapter 3 ConstruCting XML with t-sQL

<row>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73340</OrderID>

<StockItemName>Superhero action jacket (Blue)

5XL</StockItemName>

<Quantity>2</Quantity>

<UnitPrice>34.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73356</OrderID>

<StockItemName>Chocolate sharks 250g</StockItemName>

<Quantity>192</Quantity>

<UnitPrice>8.55</UnitPrice>

</row>

<row>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72787</OrderID>

<StockItemName>Novelty chilli chocolates

250g</StockItemName>

<Quantity>240</Quantity>

<UnitPrice>8.55</UnitPrice>

</row>

<row>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72671</OrderID>

65

Chapter 3 ConstruCting XML with t-sQL

<StockItemName>Packing knife with metal insert blade

(Yellow) 9mm</StockItemName>

<Quantity>35</Quantity>

<UnitPrice>1.89</UnitPrice>

</row>

<row>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72671</OrderID>

<StockItemName>Shipping carton (Brown) 480x270x320mm

</StockItemName>

<Quantity>150</Quantity>

<UnitPrice>2.74</UnitPrice>

</row>

<row>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72770</OrderID>

<StockItemName>Shipping carton (Brown) 356x229x229mm

</StockItemName>

<Quantity>175</Quantity>

<UnitPrice>1.14</UnitPrice>

</row>

<row>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72787</OrderID>

<StockItemName>Ride on big wheel monster truck (Black)

1/12 scale</StockItemName>

66

Chapter 3 ConstruCting XML with t-sQL

<Quantity>4</Quantity>

<UnitPrice>345.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72669</OrderID>

<StockItemName>10 mm Anti static bubble wrap (Blue) 10m

</StockItemName>

<Quantity>70</Quantity>

<UnitPrice>26.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73340</OrderID>

<StockItemName>32 mm Double sided bubble wrap 50m

</StockItemName>

<Quantity>20</Quantity>

<UnitPrice>112.00</UnitPrice>

</row>

<row>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72669</OrderID>

<StockItemName>Office cube periscope (Black)</StockItemName>

<Quantity>20</Quantity>

<UnitPrice>18.50</UnitPrice>

</row>

67

Chapter 3 ConstruCting XML with t-sQL

<row>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73356</OrderID>

<StockItemName>Ride on vintage American toy coupe (Black)

1/12 scale</StockItemName>

<Quantity>10</Quantity>

<UnitPrice>285.00</UnitPrice>

</row>

</SalesOrders>

You can see that the element-centric document still returns one

element, called <row>, per row in the relational result set. Instead of the data being contained in attributes, however, it is stored in the form of child elements. Each child element has been given the name of the column,

from which the data has been returned. The data is still flat, however.

There is no hierarchy based on logic or physical table structure.

It is possible to give the <row> element a more meaningful name. In our example, the most meaningful name would be <LineItem>. Listin[g 3-8](#index_split_000.html#p80)

demonstrates how we can use an optional argument in our FOR XML clause,

to generate this name for the element.

***Listing 3-8.*** Generating a Name for the <row> Element SELECT

SalesOrder.OrderDate

, Customers.CustomerName

, SalesOrder.OrderID

, Product.StockItemName

, LineItem.Quantity

, LineItem.UnitPrice

68

Chapter 3 ConstruCting XML with t-sQL

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON LineItem.OrderID = SalesOrder.OrderID

INNER JOIN Warehouse.StockItems Product

ON Product.StockItemID = LineItem.StockItemID

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID = SalesOrder.CustomerID

WHERE customers.CustomerName = 'Agrita Abele'

FOR XML RAW ('LineItem'), ELEMENTS, ROOT('SalesOrders') ;

The resulting XML document can be found in Listin[g 3-9.](#index_split_000.html#p81)

***Listing 3-9.*** Generating a Name for the <row> Element Results

<SalesOrders>

<LineItem>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72787</OrderID>

<StockItemName>Developer joke mug - when your hammer is

C++ (Black)</StockItemName>

<Quantity>2</Quantity>

<UnitPrice>13.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72787</OrderID>

<StockItemName>Developer joke mug - when your hammer is

C++ (White)</StockItemName>

<Quantity>9</Quantity>

<UnitPrice>13.00</UnitPrice>

</LineItem>

69

Chapter 3 ConstruCting XML with t-sQL

<LineItem>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73350</OrderID>

<StockItemName>DBA joke mug - you might be a DBA if

(Black)</StockItemName>

<Quantity>4</Quantity>

<UnitPrice>13.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73350</OrderID>

<StockItemName>Developer joke mug - that's a hardware

problem (Black)</StockItemName>

<Quantity>3</Quantity>

<UnitPrice>13.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72770</OrderID>

<StockItemName>"The Gu" red shirt XML tag t-shirt (Black) 5XL

</StockItemName>

<Quantity>120</Quantity>

<UnitPrice>18.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72637</OrderID>

70

Chapter 3 ConstruCting XML with t-sQL

<StockItemName>"The Gu" red shirt XML tag t-shirt (Black) XXL</StockItemName>

<Quantity>24</Quantity>

<UnitPrice>18.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73350</OrderID>

<StockItemName>Halloween zombie mask (Light Brown) S

</StockItemName>

<Quantity>24</Quantity>

<UnitPrice>18.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72669</OrderID>

<StockItemName>Halloween skull mask (Gray) M</StockItemName>

<Quantity>60</Quantity>

<UnitPrice>18.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72787</OrderID>

<StockItemName>Furry gorilla with big eyes slippers (Black)

XL</StockItemName>

<Quantity>9</Quantity>

<UnitPrice>32.00</UnitPrice>

</LineItem>

71

Chapter 3 ConstruCting XML with t-sQL

<LineItem>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73350</OrderID>

<StockItemName>Tape dispenser (Blue)</StockItemName>

<Quantity>90</Quantity>

<UnitPrice>32.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72713</OrderID>

<StockItemName>Tape dispenser (Red)</StockItemName>

<Quantity>20</Quantity>

<UnitPrice>32.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72770</OrderID>

<StockItemName>Superhero action jacket (Blue)

S</StockItemName>

<Quantity>8</Quantity>

<UnitPrice>25.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73350</OrderID>

<StockItemName>RC toy sedan car with remote control (Red)

1/50 scale</StockItemName>

72

Chapter 3 ConstruCting XML with t-sQL

<Quantity>2</Quantity>

<UnitPrice>25.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72637</OrderID>

<StockItemName>Superhero action jacket (Blue)

3XS</StockItemName>

<Quantity>1</Quantity>

<UnitPrice>25.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72770</OrderID>

<StockItemName>Black and orange handle with care despatch

tape 48mmx100m</StockItemName>

<Quantity>96</Quantity>

<UnitPrice>4.10</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72671</OrderID>

<StockItemName>Alien officer hoodie (Black)

4XL</StockItemName>

<Quantity>6</Quantity>

<UnitPrice>35.00</UnitPrice>

</LineItem>

73

Chapter 3 ConstruCting XML with t-sQL

<LineItem>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72637</OrderID>

<StockItemName>Furry animal socks (Pink) S</StockItemName>

<Quantity>96</Quantity>

<UnitPrice>5.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72669</OrderID>

<StockItemName>Permanent marker black 5mm nib (Black)

5mm</StockItemName>

<Quantity>84</Quantity>

<UnitPrice>2.70</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73340</OrderID>

<StockItemName>Superhero action jacket (Blue)

5XL</StockItemName>

<Quantity>2</Quantity>

<UnitPrice>34.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73356</OrderID>

<StockItemName>Chocolate sharks 250g</StockItemName>

74

Chapter 3 ConstruCting XML with t-sQL

<Quantity>192</Quantity>

<UnitPrice>8.55</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72787</OrderID>

<StockItemName>Novelty chilli chocolates 250g

</StockItemName>

<Quantity>240</Quantity>

<UnitPrice>8.55</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72671</OrderID>

<StockItemName>Packing knife with metal insert blade

(Yellow) 9mm</StockItemName>

<Quantity>35</Quantity>

<UnitPrice>1.89</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72671</OrderID>

<StockItemName>Shipping carton (Brown) 480x270x320mm

</StockItemName>

<Quantity>150</Quantity>

<UnitPrice>2.74</UnitPrice>

</LineItem>

75

Chapter 3 ConstruCting XML with t-sQL

<LineItem>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72770</OrderID>

<StockItemName>Shipping carton (Brown) 356x229x229mm

</StockItemName>

<Quantity>175</Quantity>

<UnitPrice>1.14</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-19</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72787</OrderID>

<StockItemName>Ride on big wheel monster truck (Black)

1/12 scale</StockItemName>

<Quantity>4</Quantity>

<UnitPrice>345.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72669</OrderID>

<StockItemName>10 mm Anti static bubble wrap (Blue) 10m

</StockItemName>

<Quantity>70</Quantity>

<UnitPrice>26.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73340</OrderID>

76

Chapter 3 ConstruCting XML with t-sQL

<StockItemName>32 mm Double sided bubble wrap 50m

</StockItemName>

<Quantity>20</Quantity>

<UnitPrice>112.00</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-18</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>72669</OrderID>

<StockItemName>Office cube periscope (Black)</

StockItemName>

<Quantity>20</Quantity>

<UnitPrice>18.50</UnitPrice>

</LineItem>

<LineItem>

<OrderDate>2016-05-27</OrderDate>

<CustomerName>Agrita Abele</CustomerName>

<OrderID>73356</OrderID>

<StockItemName>Ride on vintage American toy coupe (Black)

1/12 scale</StockItemName>

<Quantity>10</Quantity>

<UnitPrice>285.00</UnitPrice>

</LineItem>

</SalesOrders>

**Using FOR XML AUTO**

Unlike FOR XML RAW, for XML AUTO can return nested results. It is also

refreshingly simple to use, because it will automatically nest the data,

based on the joins within your query. The modified query in Listing [3-10](#index_split_000.html#p90)

uses AUTO mode to return a hierarchical XML document.

77

Chapter 3 ConstruCting XML with t-sQL

***Listing 3-10.*** Using FOR XML AUTO

SELECT

SalesOrder.OrderDate

, Customers.CustomerName

, SalesOrder.OrderID

, Product.StockItemName

, LineItem.Quantity

, LineItem.UnitPrice

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON LineItem.OrderID = SalesOrder.OrderID

INNER JOIN Warehouse.StockItems Product

ON Product.StockItemID = LineItem.StockItemID

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID = SalesOrder.CustomerID

WHERE customers.CustomerName = 'Agrita Abele'

FOR XML AUTO ;

The XML fragment that is returned by this query can be partially seen

in Listing [3-11](#index_split_000.html#p90).

***Listing 3-11.*** Using FOR XML AUTO Results

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Developer joke mug - when your

hammer is C++ (Black)">

<LineItem Quantity="2" UnitPrice="13.00" />

</Product>

78

Chapter 3 ConstruCting XML with t-sQL

<Product StockItemName="Developer joke mug - when your

hammer is C++ (White)">

<LineItem Quantity="9" UnitPrice="13.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="DBA joke mug - you might be a DBA

if (Black)">

<LineItem Quantity="4" UnitPrice="13.00" />

</Product>

<Product StockItemName="Developer joke mug - that's a

hardware problem (Black)">

<LineItem Quantity="3" UnitPrice="13.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="&quot;The Gu&quot; red shirt XML

tag t-shirt (Black) 5XL">

<LineItem Quantity="120" UnitPrice="18.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<Customers CustomerName="Agrita Abele">

79

Chapter 3 ConstruCting XML with t-sQL

<Product StockItemName="&quot;The Gu&quot; red shirt XML

tag t-shirt (Black) XXL">

<LineItem Quantity="24" UnitPrice="18.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Halloween zombie mask (Light Brown) S">

<LineItem Quantity="24" UnitPrice="18.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Halloween skull mask (Gray) M">

<LineItem Quantity="60" UnitPrice="18.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Furry gorilla with big eyes

slippers (Black) XL">

<LineItem Quantity="9" UnitPrice="32.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Tape dispenser (Blue)">

<LineItem Quantity="90" UnitPrice="32.00" />

80

Chapter 3 ConstruCting XML with t-sQL

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72713">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Tape dispenser (Red)">

<LineItem Quantity="20" UnitPrice="32.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Superhero action jacket (Blue) S">

<LineItem Quantity="8" UnitPrice="25.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="RC toy sedan car with remote

control (Red) 1/50 scale">

<LineItem Quantity="2" UnitPrice="25.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Superhero action jacket (Blue) 3XS">

<LineItem Quantity="1" UnitPrice="25.00" />

</Product>

</Customers>

</SalesOrder>

81

Chapter 3 ConstruCting XML with t-sQL

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Black and orange handle with care

despatch tape 48mmx100m">

<LineItem Quantity="96" UnitPrice="4.10" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72671">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Alien officer hoodie (Black) 4XL">

<LineItem Quantity="6" UnitPrice="35.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Furry animal socks (Pink) S">

<LineItem Quantity="96" UnitPrice="5.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Permanent marker black 5mm nib

(Black) 5mm">

<LineItem Quantity="84" UnitPrice="2.70" />

</Product>

</Customers>

</SalesOrder>

82

Chapter 3 ConstruCting XML with t-sQL

<SalesOrder OrderDate="2016-05-27" OrderID="73340">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Superhero action jacket (Blue) 5XL">

<LineItem Quantity="2" UnitPrice="34.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73356">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Chocolate sharks 250g">

<LineItem Quantity="192" UnitPrice="8.55" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Novelty chilli chocolates 250g">

<LineItem Quantity="240" UnitPrice="8.55" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72671">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Packing knife with metal insert

blade (Yellow) 9mm">

<LineItem Quantity="35" UnitPrice="1.89" />

</Product>

<Product StockItemName="Shipping carton (Brown)

480x270x320mm">

<LineItem Quantity="150" UnitPrice="2.74" />

</Product>

</Customers>

83

Chapter 3 ConstruCting XML with t-sQL

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Shipping carton (Brown) 356x229x229mm">

<LineItem Quantity="175" UnitPrice="1.14" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Ride on big wheel monster truck

(Black) 1/12 scale">

<LineItem Quantity="4" UnitPrice="345.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="10 mm Anti static bubble wrap

(Blue) 10m">

<LineItem Quantity="70" UnitPrice="26.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73340">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="32 mm Double sided bubble wrap 50m">

<LineItem Quantity="20" UnitPrice="112.00" />

</Product>

</Customers>

</SalesOrder>

84

Chapter 3 ConstruCting XML with t-sQL

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Office cube periscope (Black)">

<LineItem Quantity="20" UnitPrice="18.50" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73356">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Ride on vintage American toy coupe

(Black) 1/12 scale">

<LineItem Quantity="10" UnitPrice="285.00" />

</Product>

</Customers>

</SalesOrder>

You can see that when AUTO mode is used, the FOR XML clause has

automatically nested the data based on the JOIN clauses within the query.

Each element has been assigned a name, based on the table alias of the

table from which it was retrieved. Just as with RAW mode, we can use the ROOT

keyword to add a root node and make the document well-formed.

We can also make the document element-centric with the ELEMENTS keyword.

If you look closely at the document in Listing [3-11, y](#index_split_000.html#p90)ou will notice that the resulting hierarchy is not ideal. <Customers> is nested under

<SalesOrders>, and <LineItem> is nested under <Product>. In this document, however, you would naturally expect the hierarchy to be

<Customers> <SalesOrders> <LineItem> <Product>. To achieve this, we would have to rewrite the query, as per Listin[g 3-12](#index_split_001.html#p98).

85

Chapter 3 ConstruCting XML with t-sQL

***Listing 3-12.*** Rewriting the Query to Nest Data Correctly

SELECT

Customers.CustomerName

, SalesOrder.OrderDate

, SalesOrder.OrderID

, LineItem.Quantity

, LineItem.UnitPrice

, Product.StockItemName

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON LineItem.OrderID = SalesOrder.OrderID

INNER JOIN Warehouse.StockItems Product

ON Product.StockItemID = LineItem.StockItemID

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID = SalesOrder.CustomerID

WHERE customers.CustomerName = 'Agrita Abele'

FOR XML AUTO ;

Note that the order of columns has changed, so that the Customers

table is referenced before the SalesOrders table, which, in turn, is

referenced before the OrderLines table. The StockItems table is the last to be referenced. You will also notice that we did not have to change the order of the joins, to produce the document shown in Listin[g 3-13](#index_split_001.html#p98).

***Listing 3-13.*** Results of Rewritten Query

<Customers CustomerName="Agrita Abele">

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<LineItem Quantity="2" UnitPrice="13.00">

<Product StockItemName="Developer joke mug - when your

hammer is C++ (Black)" />

</LineItem>

86

Chapter 3 ConstruCting XML with t-sQL

<LineItem Quantity="9" UnitPrice="13.00">

<Product StockItemName="Developer joke mug - when your

hammer is C++ (White)" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<LineItem Quantity="4" UnitPrice="13.00">

<Product StockItemName="DBA joke mug - you might be a DBA

if (Black)" />

</LineItem>

<LineItem Quantity="3" UnitPrice="13.00">

<Product StockItemName="Developer joke mug - that's a

hardware problem (Black)" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<LineItem Quantity="120" UnitPrice="18.00">

<Product StockItemName="&quot;The Gu&quot; red shirt XML

tag t-shirt (Black) 5XL" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<LineItem Quantity="24" UnitPrice="18.00">

<Product StockItemName="&quot;The Gu&quot; red shirt XML

tag t-shirt (Black) XXL" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<LineItem Quantity="24" UnitPrice="18.00">

<Product StockItemName="Halloween zombie mask

(Light Brown) S" />

</LineItem>

87

Chapter 3 ConstruCting XML with t-sQL

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<LineItem Quantity="60" UnitPrice="18.00">

<Product StockItemName="Halloween skull mask (Gray) M" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<LineItem Quantity="9" UnitPrice="32.00">

<Product StockItemName="Furry gorilla with big eyes

slippers (Black) XL" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<LineItem Quantity="90" UnitPrice="32.00">

<Product StockItemName="Tape dispenser (Blue)" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72713">

<LineItem Quantity="20" UnitPrice="32.00">

<Product StockItemName="Tape dispenser (Red)" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<LineItem Quantity="8" UnitPrice="25.00">

<Product StockItemName="Superhero action jacket (Blue) S" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<LineItem Quantity="2" UnitPrice="25.00">

<Product StockItemName="RC toy sedan car with remote

control (Red) 1/50 scale" />

88

Chapter 3 ConstruCting XML with t-sQL

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<LineItem Quantity="1" UnitPrice="25.00">

<Product StockItemName="Superhero action jacket (Blue) 3XS" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<LineItem Quantity="96" UnitPrice="4.10">

<Product StockItemName="Black and orange handle with care

despatch tape 48mmx100m" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72671">

<LineItem Quantity="6" UnitPrice="35.00">

<Product StockItemName="Alien officer hoodie (Black) 4XL" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<LineItem Quantity="96" UnitPrice="5.00">

<Product StockItemName="Furry animal socks (Pink) S" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<LineItem Quantity="84" UnitPrice="2.70">

<Product StockItemName="Permanent marker black 5mm nib

(Black) 5mm" />

</LineItem>

</SalesOrder>

89

Chapter 3 ConstruCting XML with t-sQL

<SalesOrder OrderDate="2016-05-27" OrderID="73340">

<LineItem Quantity="2" UnitPrice="34.00">

<Product StockItemName="Superhero action jacket (Blue) 5XL" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73356">

<LineItem Quantity="192" UnitPrice="8.55">

<Product StockItemName="Chocolate sharks 250g" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<LineItem Quantity="240" UnitPrice="8.55">

<Product StockItemName="Novelty chilli chocolates 250g" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72671">

<LineItem Quantity="35" UnitPrice="1.89">

<Product StockItemName="Packing knife with metal insert

blade (Yellow) 9mm" />

</LineItem>

<LineItem Quantity="150" UnitPrice="2.74">

<Product StockItemName="Shipping carton (Brown)

480x270x320mm" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<LineItem Quantity="175" UnitPrice="1.14">

<Product StockItemName="Shipping carton (Brown)

356x229x229mm" />

</LineItem>

</SalesOrder>

90

Chapter 3 ConstruCting XML with t-sQL

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<LineItem Quantity="4" UnitPrice="345.00">

<Product StockItemName="Ride on big wheel monster truck

(Black) 1/12 scale" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<LineItem Quantity="70" UnitPrice="26.00">

<Product StockItemName="10 mm Anti static bubble wrap

(Blue) 10m" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73340">

<LineItem Quantity="20" UnitPrice="112.00">

<Product StockItemName="32 mm Double sided bubble wrap 50m" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<LineItem Quantity="20" UnitPrice="18.50">

<Product StockItemName="Office cube periscope (Black)" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73356">

<LineItem Quantity="10" UnitPrice="285.00">

<Product StockItemName="Ride on vintage American toy

coupe (Black) 1/12 scale" />

</LineItem>

</SalesOrder>

</Customers>

91

Chapter 3 ConstruCting XML with t-sQL

Nesting based on table joins is not always sufficient. In this example,

it would be possible for some <LineItem> elements to contain multiple

<Product> elements, which is obviously not correct. There cannot be more than one product per line on a sales order. This is because the primary key of the Sales.OrderLines table has not been included in the query, meaning that not every set of tuples returned from the Sales.OrderLines table

must be unique. The values for UnitPrice and OrderQty happened to be

repeated. If this occurred, FOR XML AUTO would group them. If the primary key had been included, the issue could not occur.

**Using FOR XML PATH**

Sometimes, you need more control over the shape of the resultant XML

document than can be provided by either RAW mode or AUTO mode. When

you have a requirement to define custom output, PATH mode can be used.

PATH mode offers great flexibility, as it allows you to define the location of each node within the resultant XML. This is achieved by specifying how

each column in the query maps to the XML, with the use of column names

or aliases.

If a column alias begins with the @ symbol, an attribute will be created.

If no @ symbol is used, the column will map to an element. Columns that

will become attributes must be specified before columns that will be

sibling nodes but defined as elements.

If you wish to define a node’s location in the hierarchy, you can use

the / symbol. For example, if you required the order date to appear nested under an element called <OrderHeader>, you could specify its column alias as '/OrderHeader/OrderDate' for the OrderDate column.

PATH mode allows you to create highly customized and complex

structures. For example, imagine that you are required to create an XML

document in the format displayed in Listin[g 3-14\. H](#index_split_001.html#p105)ere, you will note that there is a root node called <SalesOrders>. The next node in the hierarchy 92

Chapter 3 ConstruCting XML with t-sQL

is <Orders>. This is a repeating element, with a new occurrence for every order raised by the customer, which is the same as when we explored AUTO

mode. The difference is that each sales order has its own hierarchy. First, there is a section for generic order information, stored within a node called

<OrderHeader>. This element is the parent element for <CustomerName>,

<OrderDate>, and <OrderID>. These values are to be stored as simple elements.

There is also a section for the line details of the order, which are stored in a node called <OrderDetails>. This element contains a repeating child element named <Product>. This repeating element is the parent node for each <ProductName> (StockItemName) and <ProductID> (StockItemID) within the order, as well as the <Price> (unitprice) and <Qty> of each item.

These values are all stored as attributes of the <Product> element.

***Listing 3-14.*** Required Format of XML Output

<SalesOrders>

<Order>

<OrderHeader>

<CustomerName>Agrita Abele</CustomerName>

<OrderDate>2016-05-18</OrderDate>

<OrderID>72637</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="96" ProductName="&quot;The Gu&quot; red shirt XML tag t-shirt (Black) XXL" Price="18.00"

Qty="24" />

<Product ProductID="107" ProductName="Superhero action

jacket (Blue) 3XS" Price="25.00" Qty="1" />

<Product ProductID="138" ProductName="Furry animal socks (Pink) S" Price="5.00" Qty="96" />

</OrderDetails>

</Order>

93

Chapter 3 ConstruCting XML with t-sQL

<Order>

<OrderHeader>

<CustomerName>Agrita Abele</CustomerName>

<OrderDate>2016-05-18</OrderDate>

<OrderID>72669</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="147" ProductName="Halloween skull

mask (Gray) M" Price="18.00" Qty="60" />

<Product ProductID="206" ProductName="Permanent marker

black 5mm nib (Black) 5mm" Price="2.70" Qty="84" />

<Product ProductID="165" ProductName="10 mm Anti static bubble wrap (Blue) 10m" Price="26.00" Qty="70" />

<Product ProductID="3" ProductName="Office cube periscope (Black)" Price="18.50" Qty="20" />

</OrderDetails>

</Order>

<Order>

<OrderHeader>

<CustomerName>Agrita Abele</CustomerName>

<OrderDate>2016-05-18</OrderDate>

<OrderID>72671</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="105" ProductName="Alien officer

hoodie (Black) 4XL" Price="35.00" Qty="6" />

<Product ProductID="209" ProductName="Packing knife with metal insert blade (Yellow) 9mm" Price="1.89" Qty="35" />

<Product ProductID="183" ProductName="Shipping carton

(Brown) 480x270x320mm" Price="2.74" Qty="150" />

</OrderDetails>

</Order>

94

Chapter 3 ConstruCting XML with t-sQL

<Order>

<OrderHeader>

<CustomerName>Agrita Abele</CustomerName>

<OrderDate>2016-05-18</OrderDate>

<OrderID>72713</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="204" ProductName="Tape dispenser

(Red)" Price="32.00" Qty="20" />

</OrderDetails>

</Order>

<Order>

<OrderHeader>

<CustomerName>Agrita Abele</CustomerName>

<OrderDate>2016-05-19</OrderDate>

<OrderID>72770</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="99" ProductName="&quot;The Gu&quot; red shirt XML tag t-shirt (Black) 5XL" Price="18.00" Qty="120" />

<Product ProductID="110" ProductName="Superhero action

jacket (Blue) S" Price="25.00" Qty="8" />

<Product ProductID="196" ProductName="Black and orange handle with care despatch tape 48mmx100m" Price="4.10" Qty="96" />

<Product ProductID="181" ProductName="Shipping carton

(Brown) 356x229x229mm" Price="1.14" Qty="175" />

</OrderDetails>

</Order>

95

Chapter 3 ConstruCting XML with t-sQL

<Order>

<OrderHeader>

<CustomerName>Agrita Abele</CustomerName>

<OrderDate>2016-05-19</OrderDate>

<OrderID>72787</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="37" ProductName="Developer joke mug -

when your hammer is C++ (Black)" Price="13.00" Qty="2" />

<Product ProductID="36" ProductName="Developer joke mug -

when your hammer is C++ (White)" Price="13.00" Qty="9" />

<Product ProductID="133" ProductName="Furry gorilla with big eyes slippers (Black) XL" Price="32.00" Qty="9" />

<Product ProductID="220" ProductName="Novelty chilli

chocolates 250g" Price="8.55" Qty="240" />

<Product ProductID="75" ProductName="Ride on big wheel

monster truck (Black) 1/12 scale" Price="345.00" Qty="4" />

</OrderDetails>

</Order>

<Order>

<OrderHeader>

<CustomerName>Agrita Abele</CustomerName>

<OrderDate>2016-05-27</OrderDate>

<OrderID>73340</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="117" ProductName="Superhero action

jacket (Blue) 5XL" Price="34.00" Qty="2" />

<Product ProductID="164" ProductName="32 mm Double sided bubble wrap 50m" Price="112.00" Qty="20" />

</OrderDetails>

</Order>

96

Chapter 3 ConstruCting XML with t-sQL

<Order>

<OrderHeader>

<CustomerName>Agrita Abele</CustomerName>

<OrderDate>2016-05-27</OrderDate>

<OrderID>73350</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="21" ProductName="DBA joke mug - you might be a DBA if (Black)" Price="13.00" Qty="4" />

<Product ProductID="33" ProductName="Developer joke mug -

that's a hardware problem (Black)" Price="13.00" Qty="3" />

<Product ProductID="142" ProductName="Halloween zombie

mask (Light Brown) S" Price="18.00" Qty="24" />

<Product ProductID="205" ProductName="Tape dispenser

(Blue)" Price="32.00" Qty="90" />

<Product ProductID="59" ProductName="RC toy sedan car

with remote control (Red) 1/50 scale" Price="25.00"

Qty="2" />

</OrderDetails>

</Order>

<Order>

<OrderHeader>

<CustomerName>Agrita Abele</CustomerName>

<OrderDate>2016-05-27</OrderDate>

<OrderID>73356</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="225" ProductName="Chocolate sharks

250g" Price="8.55" Qty="192" />

97

Chapter 3 ConstruCting XML with t-sQL

<Product ProductID="74" ProductName="Ride on vintage American toy coupe (Black) 1/12 scale" Price="285.00" Qty="10" />

</OrderDetails>

</Order>

</SalesOrders>

Queries to build XML documents using FOR XML PATH can seem a little

complicated at first, but once you understand the principles, they become quite straightforward. Let’s examine each requirement and how we can

achieve the required shape.

First, we will require a root node called <SalesOrders>, and we will also need to rename the <row> element to <Orders>. This can be achieved in the same way as we did when we were exploring RAW mode, using the

optional argument for the <row> element and the ROOT keyword in the FOR

XML clause and the root node, as shown in Listing [3-15.](#index_split_001.html#p110)

***Listing 3-15.*** Creating a Root and Naming the <row> Element with FOR XML PATH

FOR XML PATH('Order'), ROOT ('SalesOrders') ;

The next requirement is to create a complex element called

<OrderHeader>, which will contain the <CustomerName>, <OrderDate>, and <OrderID> elements. This can be achieved by specifying a path to the element, as well as the element name. As demonstrated in Listin[g 3-16](#index_split_001.html#p110), when using this technique, the / character denotes a step down the

hierarchy.

***Listing 3-16.*** Creating Hierarchy Levels

SELECT

CustomerName 'OrderHeader/CustomerName'

, OrderDate 'OrderHeader/OrderDate'

, OrderID 'OrderHeader/OrderID'

98

Chapter 3 ConstruCting XML with t-sQL

To create the nested line items, we must use a subquery, which returns

the XML data type. For the process to work properly, we must use the TYPE

keyword in the FOR XML clause of the subquery. This will cause the results to be returned to the outer query in native XML, as it can be processed

server-side. Failure to use this keyword will result in some characters being replaced by control character sequences.

Because we want the values to be stored as attributes of an element

called <Product>, we will have to rename the <row> element and prefix our column aliases with the @ symbol. Finally, to ensure that the <Product> elements are nested under an element named <OrderDetails>, we will

use OrderDetails as the alias for the column returned by the subquery, as demonstrated in Listing [3-17.](#index_split_001.html#p111)

***Listing 3-17.*** OrderDetails Subquery

(

SELECT

LineItems2.StockItemID '@ProductID'

, StockItems.StockItemName '@ProductName'

, LineItems2.UnitPrice '@Price'

, Quantity '@Qty'

FROM Sales.OrderLines LineItems2

INNER JOIN Warehouse.StockItems StockItems

ON LineItems2.StockItemID = StockItems.Stock

ItemID

WHERE LineItems2.OrderID = Base.OrderID

FOR XML PATH('Product'), TYPE

) 'OrderDetails'

Listin[g 3-18 p](#index_split_001.html#p112)ulls together the aspects that we have discussed and returns an XML document in the required format.

99

Chapter 3 ConstruCting XML with t-sQL

***Listing 3-18.*** Putting It All Together

SELECT

CustomerName 'OrderHeader/CustomerName'

, OrderDate 'OrderHeader/OrderDate'

, OrderID 'OrderHeader/OrderID'

, (

SELECT

LineItems2.StockItemID '@ProductID'

, StockItems.StockItemName '@ProductName'

, LineItems2.UnitPrice '@Price'

, Quantity '@Qty'

FROM Sales.OrderLines LineItems2

INNER JOIN Warehouse.StockItems StockItems

ON LineItems2.StockItemID = Stock

Items.StockItemID

WHERE LineItems2.OrderID = Base.OrderID

FOR XML PATH('Product'), TYPE

) 'OrderDetails'

FROM

(

SELECT DISTINCT

Customers.CustomerName

, SalesOrder.OrderDate

, SalesOrder.OrderID

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON SalesOrder.OrderID = LineItem.OrderID

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID = SalesOrder.CustomerID

WHERE customers.CustomerName = 'Agrita Abele'

) Base

FOR XML PATH('Order'), ROOT ('SalesOrders') ;

100

Chapter 3 ConstruCting XML with t-sQL

**Using FOR XML EXPLICIT**

FOR XML EXPLICIT is the most complex of the FOR XML modes, but it also

provides ultimate flexibility. It uses the concept of tags and parents to control nesting. A tag designates a node’s position within the hierarchy, and parent indicates the tag of the node that resides above it in the

hierarchy. Therefore, the top-level node will have a tag of 1 and a parent of NULL. Each node is specified in a separate query, and the queries are

joined using UNION ALL.

For example, imagine that we want to expand the example in

Listin[g 3-18 s](#index_split_001.html#p112)o that we return an OrderDetails element that has two child elements. The first will contain the name of the salesperson who took the order. The second will be a repeating element, detailing the line items

within the order. We could achieve this by using the query in Listin[g 3-19.](#index_split_001.html#p113)

***Listing 3-19.*** Order Details with XML Explicit

SELECT

1 AS Tag

, 0 AS Parent

, SalesOrder.OrderID AS

[OrderDetails!1!SalesOrderID]

, SalesOrder.OrderDate AS

[OrderDetails!1!OrderDate]

, SalesOrder.CustomerID AS

[OrderDetails!1!CustomerID]

, NULL AS [SalesPerson!2!SalesPersonName]

, NULL AS [LineItem!3!LineTotal!ELEMENT]

, NULL AS [LineItem!3!ProductName!ELEMENT]

101

Chapter 3 ConstruCting XML with t-sQL

, NULL AS [LineItem!3!OrderQty!ELEMENT]

FROM Sales.Orders SalesOrder

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID = SalesOrder.CustomerID

WHERE customers.CustomerName = 'Agrita Abele'

UNION ALL

SELECT

2 AS Tag

, 1 AS Parent

, SalesOrder.OrderID

, NULL

, NULL

, People.FullName

, NULL

, NULL

, NULL

FROM Sales.Orders SalesOrder

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID = SalesOrder.CustomerID

INNER JOIN Application.People People

ON People.PersonID = SalesOrder.SalespersonPersonID

WHERE customers.CustomerName = 'Agrita Abele'

UNION ALL

SELECT

3 AS Tag

, 1 AS Parent

, SalesOrder.OrderID

, NULL

, NULL

, People.FullName

, LineItem.UnitPrice

102

Chapter 3 ConstruCting XML with t-sQL

, Product.StockItemName

, LineItem.Quantity

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON SalesOrder.OrderID = LineItem.OrderID

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID = SalesOrder.CustomerID

INNER JOIN Warehouse.StockItems Product

ON Product.StockItemID = LineItem.StockItemID

INNER JOIN Application.People People

ON People.PersonID = SalesOrder.SalespersonPersonID

WHERE customers.CustomerName = 'Agrita Abele'

ORDER BY

[OrderDetails!1!SalesOrderID]

, [SalesPerson!2!SalesPersonName]

, [LineItem!3!LineTotal!ELEMENT]

FOR XML EXPLICIT, ROOT('SalesOrders') ;

There are various things that we should note about the query in

Listin[g 3-19\. F](#index_split_001.html#p113)irst, each complex node is in its own query, and these queries are connected by using UNION ALL. Just like every use of UNION

ALL, each query must have the same number of columns. Therefore, NULL

values are used when a node does not exist within the complex node being

defined by the query. Also, note that the OrderID column is used in all

three queries. This column is used to define which salespeople, and line

details should be ordered under each sales order.

Because the SalesPerson and LineItem nodes are siblings, both the

second and third queries have their parent configured as 1, which is

the tag of the first query. This means that both will be nested under the OrderDetails node, at the same level of the hierarchy. The query defining the OrderDetails node has its parent configured as NULL. This denotes

103

Chapter 3 ConstruCting XML with t-sQL

that it is at the top of the hierarchy, with the exception of the root node, which is defined in the FOR XML clause.

As with other UNION ALL queries, where column names are derived

from the first query, the node names are taken from the first query. If you examine the column aliases in the first query, you will clearly see how

the structure of the node names is defined. The name parts are separated

with a !. The first name part defines the name of the complex node. The

second part describes the ID of the tag that is being named. The third part of the name defines the name of the node that will store the data within

the column. By default, FOR XML EXPLICIT is attribute-centric. Therefore, if you require a node to be an element, you must add a fourth part to the name, known as the ELEMENT directive.

The query in Listing [3-19 pr](#index_split_001.html#p113)oduces the XML document in Listin[g 3-20.](#index_split_001.html#p116)

***Listing 3-20.*** FOR XML EXPLICIT Results

<SalesOrders>

<OrderDetails SalesOrderID="72637" OrderDate="2016-05-18"

CustomerID="1061">

<SalesPerson SalesPersonName="Taj Shand" />

<LineItem>

<LineTotal>5.00</LineTotal>

<ProductName>Furry animal socks (Pink) S</ProductName>

<OrderQty>96</OrderQty>

</LineItem>

<LineItem>

<LineTotal>18.00</LineTotal>

<ProductName>"The Gu" red shirt XML tag t-shirt (Black)

XXL</ProductName>

<OrderQty>24</OrderQty>

</LineItem>

<LineItem>

104

Chapter 3 ConstruCting XML with t-sQL

<LineTotal>25.00</LineTotal>

<ProductName>Superhero action jacket (Blue) 3XS</ProductName>

<OrderQty>1</OrderQty>

</LineItem>

</OrderDetails>

<OrderDetails SalesOrderID="72669" OrderDate="2016-05-18"

CustomerID="1061">

<SalesPerson SalesPersonName="Hudson Onslow" />

<LineItem>

<LineTotal>2.70</LineTotal>

<ProductName>Permanent marker black 5mm nib (Black) 5mm

</ProductName>

<OrderQty>84</OrderQty>

</LineItem>

<LineItem>

<LineTotal>18.00</LineTotal>

<ProductName>Halloween skull mask (Gray) M</ProductName>

<OrderQty>60</OrderQty>

</LineItem>

<LineItem>

<LineTotal>18.50</LineTotal>

<ProductName>Office cube periscope (Black)</ProductName>

<OrderQty>20</OrderQty>

</LineItem>

<LineItem>

<LineTotal>26.00</LineTotal>

<ProductName>10 mm Anti static bubble wrap (Blue) 10m

</ProductName>

<OrderQty>70</OrderQty>

</LineItem>

</OrderDetails>

105

Chapter 3 ConstruCting XML with t-sQL

<OrderDetails SalesOrderID="72671" OrderDate="2016-05-18"

CustomerID="1061">

<SalesPerson SalesPersonName="Hudson Hollinworth" />

<LineItem>

<LineTotal>1.89</LineTotal>

<ProductName>Packing knife with metal insert blade

(Yellow) 9mm</ProductName>

<OrderQty>35</OrderQty>

</LineItem>

<LineItem>

<LineTotal>2.74</LineTotal>

<ProductName>Shipping carton (Brown) 480x270x320mm

</ProductName>

<OrderQty>150</OrderQty>

</LineItem>

<LineItem>

<LineTotal>35.00</LineTotal>

<ProductName>Alien officer hoodie (Black) 4XL</ProductName>

<OrderQty>6</OrderQty>

</LineItem>

</OrderDetails>

<OrderDetails SalesOrderID="72713" OrderDate="2016-05-18"

CustomerID="1061">

<SalesPerson SalesPersonName="Hudson Onslow" />

<LineItem>

<LineTotal>32.00</LineTotal>

<ProductName>Tape dispenser (Red)</ProductName>

<OrderQty>20</OrderQty>

</LineItem>

</OrderDetails>

106

Chapter 3 ConstruCting XML with t-sQL

<OrderDetails SalesOrderID="72770" OrderDate="2016-05-19"

CustomerID="1061">

<SalesPerson SalesPersonName="Hudson Onslow" />

<LineItem>

<LineTotal>1.14</LineTotal>

<ProductName>Shipping carton (Brown) 356x229x229mm

</ProductName>

<OrderQty>175</OrderQty>

</LineItem>

<LineItem>

<LineTotal>4.10</LineTotal>

<ProductName>Black and orange handle with care despatch

tape 48mmx100m</ProductName>

<OrderQty>96</OrderQty>

</LineItem>

<LineItem>

<LineTotal>18.00</LineTotal>

<ProductName>"The Gu" red shirt XML tag t-shirt (Black)

5XL</ProductName>

<OrderQty>120</OrderQty>

</LineItem>

<LineItem>

<LineTotal>25.00</LineTotal>

<ProductName>Superhero action jacket (Blue) S

</ProductName>

<OrderQty>8</OrderQty>

</LineItem>

</OrderDetails>

107

Chapter 3 ConstruCting XML with t-sQL

<OrderDetails SalesOrderID="72787" OrderDate="2016-05-19"

CustomerID="1061">

<SalesPerson SalesPersonName="Jack Potter" />

<LineItem>

<LineTotal>8.55</LineTotal>

<ProductName>Novelty chilli chocolates 250g</ProductName>

<OrderQty>240</OrderQty>

</LineItem>

<LineItem>

<LineTotal>13.00</LineTotal>

<ProductName>Developer joke mug - when your hammer is

C++ (White)</ProductName>

<OrderQty>9</OrderQty>

</LineItem>

<LineItem>

<LineTotal>13.00</LineTotal>

<ProductName>Developer joke mug - when your hammer is

C++ (Black)</ProductName>

<OrderQty>2</OrderQty>

</LineItem>

<LineItem>

<LineTotal>32.00</LineTotal>

<ProductName>Furry gorilla with big eyes slippers (Black)

XL</ProductName>

<OrderQty>9</OrderQty>

</LineItem>

<LineItem>

<LineTotal>345.00</LineTotal>

108

Chapter 3 ConstruCting XML with t-sQL

<ProductName>Ride on big wheel monster truck (Black) 1/12

scale</ProductName>

<OrderQty>4</OrderQty>

</LineItem>

</OrderDetails>

<OrderDetails SalesOrderID="73340" OrderDate="2016-05-27"

CustomerID="1061">

<SalesPerson SalesPersonName="Taj Shand" />

<LineItem>

<LineTotal>34.00</LineTotal>

<ProductName>Superhero action jacket (Blue) 5XL

</ProductName>

<OrderQty>2</OrderQty>

</LineItem>

<LineItem>

<LineTotal>112.00</LineTotal>

<ProductName>32 mm Double sided bubble wrap 50m

</ProductName>

<OrderQty>20</OrderQty>

</LineItem>

</OrderDetails>

<OrderDetails SalesOrderID="73350" OrderDate="2016-05-27"

CustomerID="1061">

<SalesPerson SalesPersonName="Sophia Hinton" />

<LineItem>

<LineTotal>13.00</LineTotal>

<ProductName>DBA joke mug - you might be a DBA if

(Black)</ProductName>

<OrderQty>4</OrderQty>

</LineItem>

109

Chapter 3 ConstruCting XML with t-sQL

<LineItem>

<LineTotal>13.00</LineTotal>

<ProductName>Developer joke mug - that's a hardware

problem (Black)</ProductName>

<OrderQty>3</OrderQty>

</LineItem>

<LineItem>

<LineTotal>18.00</LineTotal>

<ProductName>Halloween zombie mask (Light Brown) S

</ProductName>

<OrderQty>24</OrderQty>

</LineItem>

<LineItem>

<LineTotal>25.00</LineTotal>

<ProductName>RC toy sedan car with remote control (Red)

1/50 scale</ProductName>

<OrderQty>2</OrderQty>

</LineItem>

<LineItem>

<LineTotal>32.00</LineTotal>

<ProductName>Tape dispenser (Blue)</ProductName>

<OrderQty>90</OrderQty>

</LineItem>

</OrderDetails>

<OrderDetails SalesOrderID="73356" OrderDate="2016-05-27"

CustomerID="1061">

<SalesPerson SalesPersonName="Anthony Grosse" />

<LineItem>

<LineTotal>8.55</LineTotal>

<ProductName>Chocolate sharks 250g</ProductName>

<OrderQty>192</OrderQty>

</LineItem>

110

Chapter 3 ConstruCting XML with t-sQL

<LineItem>

<LineTotal>285.00</LineTotal>

<ProductName>Ride on vintage American toy coupe (Black)

1/12 scale</ProductName>

<OrderQty>10</OrderQty>

</LineItem>

</OrderDetails>

</SalesOrders>

**Summary**

The FOR XML clause can be used to construct XML data from a T-SQL

query. The FOR XML clause can be used in four modes: RAW, AUTO, PATH, and EXPLICIT.

When used in RAW mode, FOR XML will produce a flat XML instance,

with no nesting. It is very simple to use and allows for the XML instance to be either attribute-centric or element-centric. It can also produce

well- formed XML, by adding a root node. It provides very little control over formatting, however.

FOR XML AUTO is very simple to use and allows the developer to create

nested XML instances. Just as with FOR XML RAW, FOR XML AUTO can

produce either element-centric or attribute-centric documents, which can

be well-formatted with the addition of a root node. Data within the XML

document will be nested automatically, based on table joins. Nodes will

derive their names from table aliases.

When used in PATH mode, FOR XML provides much more granular

control over the layout of the XML document. Each column can be defined

as either an element or an attribute, by including or omitting the @ prefix.

Hierarchical position can be defined through column aliases, and complex

nesting requirements can be achieved through subqueries.

111

Chapter 3 ConstruCting XML with t-sQL

FOR XML EXPLICIT is the most complex of the FOR XML modes but also

the most powerful. It is implemented by defining each complex node in

a separate query, with the queries joined by using the UNION ALL clause.

Hierarchical positioning is defined through tag and parent columns,

which are always the first and second columns, respectively, in the SELECT

list. Column aliases allow for the declaration of node names and define if each node should be element-centric or attribute-centric.

112

**CHAPTER 4**

**Querying and**

**Shredding XML**

To allow developers to query and navigate XML documents from within

SQL Server, the XQuery language can be combined with T-SQL queries.

In this chapter, I will discuss how to use XQuery to filter, extract, and modify XML. I will also discuss shredding XML, which is the process of

converting XML data into relational results sets. Finally, an overview of how to bind an XSD schema to a column of data type XML is provided.

**Querying XML**

XQuery is a language for querying XML, in the same way that SQL is a

language for querying relational data. The language is built on XPath

but has been enhanced for better iteration and sorting. It also allows for the construction of XML. The XQuery standard is developed by the W3C

(World Wide Web Consortium), in conjunction with Microsoft and other

major relational database management system (RDBMS) vendors.

XQuery in SQL Server supports five methods against the XML data

type. An overview of these methods can be found in Table [4-1, and e](#index_split_001.html#p126)ach of the methods will be demonstrated throughout this chapter. XQuery

also supports the use of XSD schemas, to help interact with complex

documents and FLWOR (pronounced flower) statements. FLWOR is an

acronym of for, let, where, order by, and return.

© Peter A. Carter 2018

113

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_4](#index_split_001.html#p125)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Table 4-1.** XQuery Methods*

**Method**

**Description**

exist()

Checks for the existence of a node with a specified value within

an XML document. It returns 1 if the node exists with the specified

value and 0 if it does not.

modify()

Performs data modification statements against an XML document.

Acceptable DML actions are insert, delete, and replace value of.

nodes()

Returns a row set that contains copies of original XML instances. It

is used for shredding XML into a relational result set.

query()

This returns a subset of an XML document, in XML format.

value()

This returns a single scalar value from an XML document, mapped

to an SQL Server data type.

**Tip** The structure and benefits of XSD schemas are explained in

Chapter [2](#index_split_000.html#p41).

for statements allow you to iterate through a sequence of nodes.

let statements bind a sequence to a variable. where statements filter

nodes based on a Boolean expression. order by statements order nodes

before they are returned. return statements specify what should be

returned.

To demonstrate the use of the XQuery methods in this chapter, we

will create a table in the WideWorldImporters database, called Sales.

CustomerOrderSummary. This table can be created using the script in

Listin[g 4-1.](#index_split_001.html#p127)

114

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Listing 4-1.*** Creating the Sales.CustomerOrderSummary Table USE WideWorldImporters

GO

CREATE TABLE Sales.CustomerOrderSummary

(

ID INT NOT NULL IDENTITY,

CustomerID INT NOT NULL,

OrderSummary XML

) ;

INSERT INTO Sales.CustomerOrderSummary (CustomerID,

OrderSummary)

SELECT

CustomerID,

(

SELECT

CustomerName 'OrderHeader/CustomerName'

, OrderDate 'OrderHeader/OrderDate'

, OrderID 'OrderHeader/OrderID'

, (

SELECT

LineItems2.StockItemID

'@ProductID'

, StockItems.StockItem

Name '@ProductName'

, LineItems2.UnitPrice

'@Price'

, Quantity '@Qty'

FROM Sales.OrderLines LineItems2

INNER JOIN Warehouse.StockItems

StockItems

115

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

ON LineItems2.StockItemID

= StockItems.StockItemID

WHERE LineItems2.OrderID =

Base.OrderID

FOR XML PATH('Product'), TYPE

) 'OrderDetails'

FROM

(

SELECT DISTINCT

Customers.CustomerName

, SalesOrder.OrderDate

, SalesOrder.OrderID

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON SalesOrder.OrderID =

LineItem.OrderID

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID =

SalesOrder.CustomerID

WHERE customers.CustomerID = OuterCust.

CustomerID

) Base

FOR XML PATH('Order'), ROOT ('SalesOrders'), TYPE

) AS OrderSummary

FROM Sales.Customers OuterCust ;

**Using exist()**

The exist() method is used to check for the existence of a node with a

specified value. For example, please consider the script in Listin[g 4-2\. T](#index_split_001.html#p129)he query will programmatically check to see if the XML document containing

sales order details contains the Chocolate sharks 250g product. The first 116

![](index-129_1.jpg)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

portion of the XQuery defines the path to the node to be evaluated, in this case, the StockItemName attribute. We have defined that the node is an

attribute, by prefixing the node name with the @ symbol. We then specify

eq as the comparison operator, before supplying the value that we want

to validate the node against. If the criteria are met, exist() will return 1.

If the criteria are not met, it will return 0.

***Listing 4-2.*** Checking for the Existence of a Value

DECLARE @SalesOrders XML

SET @SalesOrders =

'<SalesOrder OrderDate="2016-05-27" OrderID="73356">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Chocolate sharks 250g">

<LineItem Quantity="192" UnitPrice="8.55" />

</Product>

</Customers>

</SalesOrder>' ;

SELECT @SalesOrders.exist('SalesOrder/Customers/Product[

(@StockItemName) eq "Chocolate sharks 250g"]') ;

The results of this script are displayed in Figur[e 4-1.](#index_split_001.html#p129)

***Figure 4-1.** Results of checking for the existence of a value*

117

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

You can see how the exist() method could easily be used in a WHERE

clause, against a column of data type XML. For example, consider the script in Listing [4-3](#index_split_001.html#p130), which uses the exist() method in a WHERE clause against the Sales.CustomerOrderSummary table in the WideWorldImporters

database, to return just the order summary for Tailspin Toys (Absecon, NJ).

You will notice that because we are evaluating the value of an element,

as opposed to an attribute, we have used the text() method to extract the value. The [1] denotes that a singleton value will be returned.

***Listing 4-3.*** Filtering a Table Using the exist() Method

SELECT

CustomerID

, OrderSummary

FROM WideWorldImporters.Sales.CustomerOrderSummary

WHERE OrderSummary.exist('SalesOrders/Order/OrderHeader/

CustomerName[(text()[1]) eq "Tailspin Toys (Absecon, NJ)"]') = 1 ; **Using value( )**

The value() method is used to extract a single, scalar value from an

XML document and map it to an SQL Server data type. For example,

consider the script in Listin[g 4-4](#index_split_001.html#p131). The script uses the same XML document as Listin[g 4-1 b](#index_split_001.html#p127)ut this time extracts the customer name from the document.

In the same way as when we used the exist() method, we will have to use

the @ symbol to prefix an attribute. We also must denote that a singleton value will be returned.

118

![](index-131_1.png)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Listing 4-4.*** Using the value() Method

DECLARE @SalesOrders XML ;

SET @SalesOrders =

'<SalesOrder OrderDate="2016-05-27" OrderID="73356">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Chocolate sharks 250g">

<LineItem Quantity="192" UnitPrice="8.55" />

</Product>

</Customers>

</SalesOrder>' ;

SELECT @SalesOrders.value('(/SalesOrder/Customers/

@CustomerName)[1]', 'nvarchar(100)') AS CustomerName ;

The results of this script are illustrated in Figur[e 4-2.](#index_split_001.html#p131)

***Figure 4-2.** Results of using the value() method*

The value() method can also be used against a table, to extract values

from each row. For example, the script in Listin[g 4-5](#index_split_001.html#p132) can be used to extract the customer name from each row in the Sales.CustomerOrderSummary table.

119

![](index-132_1.jpg)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Listing 4-5.*** Using the value() Method Against a Table

USE WideWorldImporters

GO

SELECT

CustomerID

, OrderSummary

, OrderSummary.value('(/SalesOrders/Order/OrderHeader/

CustomerName)[1]', 'nvarchar(100)') AS CustomerName

FROM Sales.CustomerOrderSummary ;

Partial results of this query can be found in Figure [4-3](#index_split_001.html#p132).

***Figure 4-3.** Results of using the value() method against a table* The value() method can also be used in the WHERE clause of a query.

For example, the query in Listin[g 4-6 is function](#index_split_001.html#p133)ally equivalent to the query in Listing [4-5](#index_split_001.html#p132). It has simply been rewritten using the value() method instead of the exist() method.

120

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Listing 4-6.*** Filtering a Table Using the value() Method

USE WideWorldImporters

GO

SELECT

CustomerID

, OrderSummary

FROM WideWorldImporters.Sales.CustomerOrderSummary

WHERE OrderSummary.value('(/SalesOrders/Order/OrderHeader/

CustomerName)[1]', 'nvarchar(100)') = 'Tailspin Toys (Absecon, NJ)' ;

**Using query( )**

The query() method is used to return an untyped XML document from an

XML document. For example, consider the script in Listin[g 4-7\. T](#index_split_001.html#p133)he query will extract the product details from the XML document in an XML format.

***Listing 4-7.*** Extracting Product Details

DECLARE @SalesOrders XML ;

SET @SalesOrders =

'<SalesOrder OrderDate="2016-05-27" OrderID="73356">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Chocolate sharks 250g">

<LineItem Quantity="192" UnitPrice="8.55" />

</Product>

</Customers>

</SalesOrder>' ;

SELECT @SalesOrders.query('/SalesOrder/Customers/Product') AS

ProductDetails ;

121

![](index-134_1.jpg)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

The results of this query can be seen in Figure [4-4](#index_split_001.html#p134).

***Figure 4-4.** Results of extracting product details*

To see how the query() method could be used when querying a

table, consider the script in Listing [4-8](#index_split_001.html#p134). This query uses all of the XQuery methods that have been discussed so far. The value() method is used

to extract the customer’s name and order ID, while the query() method

is used to extract details of the products that were ordered. The table is filtered using the exist() method.

***Listing 4-8.*** Using query(), value(), and exist() Against a Table USE WideWorldImporters

GO

SELECT

OrderSummary

, OrderSummary.value('(/SalesOrders/Order/OrderHeader/

CustomerName)[1]', 'nvarchar(100)') AS CustomerName

, OrderSummary.value('(/SalesOrders/Order/OrderHeader/

OrderID)[1]', 'int') AS OrderID

, OrderSummary.query('/SalesOrders/Order/OrderDetails/

Product') AS ProductsOrdered

FROM Sales.CustomerOrderSummary

WHERE OrderSummary.exist('SalesOrders/Order/OrderHeader/

CustomerName[(text()[1]) eq "Tailspin Toys (Absecon, NJ)"]') = 1 ; 122

![](index-135_1.jpg)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

This query returns the results shown in Figur[e 4-5.](#index_split_001.html#p135)

***Figure 4-5.** Results of using query(), value(), and exist() against* *a table*

**Using Relational Values in XQuery**

Relational values from T-SQL variables and columns can also be

passed into XQuery expressions. They can be used for filtering or even

constructing data, but they are read-only. Therefore, the XQuery expression cannot be used to modify the relational variable or column value.

To see this functionality in action, consider the query in Listing [4-9](#index_split_001.html#p135).

The query is functionally equivalent to the query in Listin[g 4-7\. T](#index_split_001.html#p133)his time, however, instead of using a hard-coded filter for the customer name, we

pass this in from a T-SQL variable. Queries that use T-SQL variables or

columns from tables are known as cross-domain queries.

***Listing 4-9.*** Parameterizing an exist() Method

USE WideWorldImporters

GO

DECLARE @CustomerName NVARCHAR(100) ;

SET @CustomerName = 'Tailspin Toys (Absecon, NJ)' ;

SELECT

123

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

OrderSummary

, OrderSummary.value('(/SalesOrders/Order/OrderHeader/

CustomerName)[1]', 'nvarchar(100)') AS CustomerName

, OrderSummary.value('(/SalesOrders/Order/OrderHeader/

OrderID)[1]', 'int') AS OrderID

, OrderSummary.query('/SalesOrders/Order/OrderDetails/

Product') AS ProductsOrdered

FROM Sales.CustomerOrderSummary

WHERE OrderSummary.exist('SalesOrders/Order/OrderHeader/

CustomerName[(text()[1]) eq sql:variable("@CustomerName") ]') = 1 ; T-SQL variables can also be used in the construction of XML. For

example, the script in Listin[g 4-10](#index_split_001.html#p136) generates a new column in the result set, which contains an XML fragment. This fragment has an element called

CustomerDetails. Within this element, you will notice an attribute called GoldCustomer. This is a flag that is configured from a T-SQL variable.

Note that when used inside the query() method, we have enclosed the

sql:variable statement inside double quotes and curly brackets.

***Listing 4-10.*** Constructing an XML Fragment and Passing a Value from a T-SQL Variable

USE WideWorldImporters

GO

DECLARE @Gold NVARCHAR(3)

DECLARE @CustomerName NVARCHAR(100)

SET @Gold = 'Yes'

SET @CustomerName = 'Tailspin Toys (Absecon, NJ)'

124

![](index-137_1.jpg)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

SELECT

OrderSummary

, OrderSummary.value('(/SalesOrders/Order/OrderHeader/

CustomerName)[1]', 'nvarchar(100)') AS CustomerName

, OrderSummary.value('(/SalesOrders/Order/OrderHeader/

OrderID)[1]', 'int') AS OrderID

, OrderSummary.query('/SalesOrders/Order/OrderDetails/

Product') AS ProductsOrdered

, OrderSummary.query('<CustomerDetails>GoldCustomer =

"{ sql:variable("@Gold") }" </CustomerDetails>') AS

CustomerDetails

FROM Sales.CustomerOrderSummary

WHERE OrderSummary.exist('SalesOrders/Order/OrderHeader/

CustomerName[(text()[1]) eq sql:variable("@CustomerName") ]') = 1 ; The results of this query can be found in Figur[e 4-6\. T](#index_split_001.html#p137)he resultant XML

fragment can be found in the CustomerDetails column.

***Figure 4-6.** Results of constructing an XML fragment and passing a* *value from a T-SQL variable*

125

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

When the XQuery expression is run against an XML column within a

table, relational data, stored in other columns, can also be passed to the XQuery expression. For example, the script in Listin[g 4-11](#index_split_001.html#p138) expands the generated CustomerDetails XML fragment to include the Customer ID

from the CustomerID column of the Sales.CustomerOrderSummary table.

***Listing 4-11.*** Constructing an XML Fragment Using a Relational Column

USE WideWorldImporters

GO

DECLARE @Gold NVARCHAR(3)

DECLARE @CustomerName NVARCHAR(100)

SET @Gold = 'Yes'

SET @CustomerName = 'Tailspin Toys (Absecon, NJ)'

SELECT

OrderSummary

, OrderSummary.value('(/SalesOrders/Order/OrderHeader/

CustomerName)[1]', 'nvarchar(100)') AS CustomerName

, OrderSummary.value('(/SalesOrders/Order/OrderHeader/

OrderID)[1]', 'int') AS OrderID

, OrderSummary.query('/SalesOrders/Order/OrderDetails/

Product') AS ProductsOrdered

, OrderSummary.query('<CustomerDetails> CustomerID =

"{ sql:column("CustomerID") }" GoldCustomer =

"{ sql:variable("@Gold") }" </CustomerDetails>') As CustomerDetails

FROM Sales.CustomerOrderSummary

WHERE OrderSummary.exist('SalesOrders/Order/OrderHeader/

CustomerName[(text()[1]) eq sql:variable("@CustomerName") ]') = 1 ; 126

![](index-139_1.jpg)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

The results of this query can be seen in Figure [4-7](#index_split_001.html#p139). Once again, pay attention to the CustomerDetails column, to see the resultant XML

fragment.

***Figure 4-7.** Results of constructing an XML fragment using a*

*relational column*

**FLWOR**

As previously mentioned, FLWOR stands for for, let, where, order by, and

return. These statements provide granular control, allowing a developer to navigate to, iterate over, filter, and present XML nodes exactly as required.

The for statement binds a variable to an input sequence. The let

statement is used to assign an XQuery expression to a variable, for use

within an iteration of FOR. The expression can return either atomic

values or a sequence of nodes. The let statement is optional. The where

statement is also optional but can be used to filter the results that are returned. The order by statement can be used optionally to order the

results of the FLWOR statement. The mandatory return statement specifies

what data will be returned.

For example, consider the XML document in Listin[g 4-12](#index_split_001.html#p140). This XML

document contains the first two orders from the XML document returned

by the OrderSummary column in Listin[g 4-11](#index_split_001.html#p138).

127

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Listing 4-12.*** XML Document for FLWOR Examples

DECLARE @XML XML = N'<SalesOrders>

<Order>

<OrderHeader>

<CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>

<OrderDate>2013-01-17</OrderDate>

<OrderID>950</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="119" ProductName="Dinosaur battery-

powered slippers (Green) M" Price="32.00" Qty="2" />

<Product ProductID="61" ProductName="RC toy sedan car with remote control (Green) 1/50 scale" Price="25.00" Qty="2" />

<Product ProductID="194" ProductName="Black and orange glass with care despatch tape 48mmx100m" Price="4.10" Qty="216" />

<Product ProductID="104" ProductName="Alien officer

hoodie (Black) 3XL" Price="35.00" Qty="2" />

</OrderDetails>

</Order>

<Order>

<OrderHeader>

<CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>

<OrderDate>2013-01-29</OrderDate>

<OrderID>1452</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="33" ProductName="Developer joke mug -

that's a hardware problem (Black)" Price="13.00" Qty="9" /> 128

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

<Product ProductID="121" ProductName="Dinosaur battery-

powered slippers (Green) XL" Price="32.00" Qty="1" />

</OrderDetails>

</Order>

<SalesOrders>’

If we wanted to iterate over each Product element within the second

order, we could use the FOR statement as follows in Listin[g 4-13](#index_split_001.html#p141).

Note that we use an internal variable, which we name $product, and

the in keyword to designate the path to the ProductName attribute. This

approach and syntax will be familiar to those of you who have worked with foreach loops in languages such as PowerShell.

The return statement is then used to append the ProductName

attribute’s value to the end of the $product string. Note that we use [2]

in square brackets, to denote that we are interested in the second order, which will be a singleton element (albeit a complex element containing

multiple other elements).

**Caution** The following code examples in this section expect a

variable called XML to be declared that contains the XML document in

Listing [4-12.](#index_split_001.html#p140) For space reasons, this variable is not explicitly declared or set within each code example.

***Listing 4-13.*** Using for to Iterate over Product Elements

SELECT @XML.query('

for $product in /SalesOrders/Order[2]/OrderDetails/

Product/@ProductName

return string($product)

') ;

129

![](index-142_1.jpg)

![](index-142_2.jpg)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

The results of this query are illustrated in Figur[e 4-8.](#index_split_001.html#p142)

***Figure 4-8.** Results of using for to iterate over product elements* The query in Listing [4-14 us](#index_split_001.html#p142)es a let statement, instead of a for iteration, to retrieve the name of the first product, within the first order, within the XML document. The return statement requires a singleton

value, and as let simply assigns a value, as opposed to iterating over

multiple values, we cannot return multiple products.

***Listing 4-14.*** Using let to Find a Product Name

SELECT @XML.query('

let $product := /SalesOrders/Order[1]/OrderDetails/

Product/@ProductName

return string($product[1])

') ;

The results of this query can be found in Figur[e 4-9.](#index_split_001.html#p142)

***Figure 4-9.** Results of using let to find a product name*

130

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

The query in Listing [4-15 com](#index_split_001.html#p143)bines both a for statement and a let statement to construct a new XML document containing the customer

name and product name for each product sold.

***Listing 4-15.*** Combining for and let

SELECT @XML.query('

for $product in /SalesOrders/Order/OrderDetails/Product/

@ProductName

let $customer := /SalesOrders/Order/OrderHeader/

CustomerName

return

<Customer>

{$customer[1]}

<OrderDetails>

{$product}

</OrderDetails>

</Customer>

') ;

The resultant XML document can be found in Listing [4-16.](#index_split_001.html#p143)

***Listing 4-16.*** Results of Combining for and let

<Customer>

<CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>

<OrderDetails ProductName="Dinosaur battery-powered slippers

(Green) M" />

</Customer>

<Customer>

<CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>

<OrderDetails ProductName="RC toy sedan car with remote

control (Green) 1/50 scale" />

</Customer>

131

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

<Customer>

<CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>

<OrderDetails ProductName="Black and orange glass with care

despatch tape 48mmx100m" />

</Customer>

<Customer>

<CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>

<OrderDetails ProductName="Alien officer hoodie (Black) 3XL" />

</Customer>

<Customer>

<CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>

<OrderDetails ProductName="Developer joke mug - that's a

hardware problem (Black)" />

</Customer>

<Customer>

<CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>

<OrderDetails ProductName="Dinosaur battery-powered slippers

(Green) XL" />

</Customer>

The query in Listing [4-17 enh](#index_split_001.html#p144)ances the query in Listing [4-15, t](#index_split_001.html#p143)o add a where statement, which will filter the products, so that only the dinosaur slippers are returned. The query also uses an order by statement, to

guarantee that the order of the results is by product name.

***Listing 4-17.*** Using where and order by

SELECT @XML.query('

for $product in /SalesOrders/Order/OrderDetails/Product/

@ProductName

let $customer := /SalesOrders/Order/OrderHeader/

CustomerName

132

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

where $product = "Dinosaur battery-powered slippers

(Green) M"

or $product = "Dinosaur battery-powered slippers

(Green) XL"

order by $product

return

<Customer>

{$customer[1]}

<OrderDetails>

{$product}

</OrderDetails>

</Customer>

') ;

**Modifying XML Data**

XML data can be modified using XQuery’s modify method. When using

modify, a developer has three options. You can use the insert option, the delete option, or the replace value of option. The replace value of

option replaces an existing value within an XML document.

To understand how the insert option works, consider the script in

Listin[g 4-18\. T](#index_split_001.html#p145)he script populates a variable with an empty order. The modify method is then used to add an order line to the sales order.

***Listing 4-18.*** Using the modify Insert

DECLARE @SalesOrder xml;

SET @SalesOrder = '

<Order>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

<OrderDate>2013-01-02</OrderDate>

133

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

<OrderID>121</OrderID>

</OrderHeader>

<OrderDetails>

</OrderDetails>

</Order>' ;

SET @SalesOrder.modify('

insert <Product ProductID="22" ProductName="DBA joke mug - it depends (White)" Price="13" Qty="6" />

into (/Order/OrderDetails)[1]') ;

SELECT @SalesOrder ;

The results of running this query can be seen in Listin[g 4-19.](#index_split_001.html#p146)

***Listing 4-19.*** Results of Using the modify Insert

<Order>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

<OrderDate>2013-01-02</OrderDate>

<OrderID>121</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="6" />

</OrderDetails>

</Order>

If Product elements already existed within the OrderDetails element,

however, you could gain granular control over where you would like the

element to be inserted by using the as first, as last, before, or after

options. For example, the query in Listin[g 4-20 w](#index_split_001.html#p147)ill insert the new Product element as the first element in the OrderDetails element.

134

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Listing 4-20.*** Inserting an Element As First

DECLARE @SalesOrder xml;

SET @SalesOrder = '

<Order>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

<OrderDate>2013-01-02</OrderDate>

<OrderID>121</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="6" />

</OrderDetails>

</Order>' ;

SET @SalesOrder.modify('

insert <Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" /> as first

into (/Order/OrderDetails)[1]') ;

SELECT @SalesOrder ;

The results of this query can be seen in Listing [4-21.](#index_split_001.html#p147)

***Listing 4-21.*** Results of Inserting an Element As First

<Order>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

<OrderDate>2013-01-02</OrderDate>

<OrderID>121</OrderID>

</OrderHeader>

135

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

<OrderDetails>

<Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="6" />

</OrderDetails>

</Order>

Alternatively, the query in Listin[g 4-22 demons](#index_split_001.html#p148)trates how you can insert an element after another, specific element. In this case, we will insert the Superhero Action Jacket after the USB Rocket Launcher.

***Listing 4-22.*** Inserting an Element After Another Element

DECLARE @SalesOrder xml;

SET @SalesOrder = '

<Order>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

<OrderDate>2013-01-02</OrderDate>

<OrderID>121</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="6" />

</OrderDetails>

</Order>' ;

DECLARE @ProductName NVARCHAR(200) ;

SET @ProductName = 'USB rocket launcher (Gray)' ;

136

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

SET @SalesOrder.modify('

insert <Product ProductID="111" ProductName="Superhero action jacket (Blue) M" Price="30" Qty="10" />

after (/Order/OrderDetails/Product[@ProductName =

sql:variable("@ProductName")])[1]') ;

SELECT @SalesOrder ;

This query uses sql:variable to pass the product’s name from an

T-SQL variable, using the techniques you learned in the “Using Relational Values in XQuery” section of this chapter. The results of the query can be seen in Listing [4-23.](#index_split_001.html#p149)

***Listing 4-23.*** Results of Inserting an Element After Another Element

<Order>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

<OrderDate>2013-01-02</OrderDate>

<OrderID>121</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />

<Product ProductID="111" ProductName="Superhero action

jacket (Blue) M" Price="30" Qty="10" />

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="6" />

</OrderDetails>

</Order>

137

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

The delete option can be used to remove nodes from an XML

document. For example, consider the script in Listing [4-24\. H](#index_split_001.html#p150)ere, we take the XML document produced by the query in Listing [4-22](#index_split_001.html#p148) and remove the last order line that we added to the document.

***Listing 4-24.*** Using the delete Option

DECLARE @SalesOrder xml;

SET @SalesOrder = '

<Order>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

<OrderDate>2013-01-02</OrderDate>

<OrderID>121</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />

<Product ProductID="111" ProductName="Superhero action

jacket (Blue) M" Price="30" Qty="10" />

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="6" />

</OrderDetails>

</Order>' ;

DECLARE @ProductName NVARCHAR(200) ;

SET @ProductName = 'Superhero action jacket (Blue) M' ;

SET @SalesOrder.modify('

delete (/Order/OrderDetails/Product[@ProductName =

sql:variable("@ProductName")])[1]') ;

SELECT @SalesOrder ;

138

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

You can see the resultant XML document in Listing [4-25.](#index_split_001.html#p151)

***Listing 4-25.*** Results of Using the delete Option

<Order>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

<OrderDate>2013-01-02</OrderDate>

<OrderID>121</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="6" />

</OrderDetails>

</Order>

The replace value of option performs an update on an existing node.

For example, the script in Listin[g 4-26](#index_split_001.html#p151) updates the quantity of DBA Joke Mugs ordered from six to ten. Once again, we bind SQL variables to allow

us to pass in the ProductName that we wish to update and the new quantity.

Note that we use replace value of to define the node that should be

updated and with to define the new value.

***Listing 4-26.*** Using replace value of

DECLARE @SalesOrder xml;

SET @SalesOrder = '

<Order>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

139

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

<OrderDate>2013-01-02</OrderDate>

<OrderID>121</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="6" />

</OrderDetails>

</Order>' ;

DECLARE @ProductName NVARCHAR(200) ;

SET @ProductName = 'DBA joke mug - it depends (White)' ;

DECLARE @Quantity INT ;

SET @Quantity = 10

SET @SalesOrder.modify('

replace value of (/Order/OrderDetails/Product[@Product

Name = sql:variable("@ProductName")]/@Qty)[1]

with "10"

') ;

SELECT @SalesOrder ;

The results of this query can be seen in Listing [4-27.](#index_split_001.html#p153)

140

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Listing 4-27.*** Results of Using replace value of

<Order>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

<OrderDate>2013-01-02</OrderDate>

<OrderID>121</OrderID>

</OrderHeader>

<OrderDetails>

<Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="10" />

</OrderDetails>

</Order>

**Shredding XML**

Shredding XML is the process of taking an XML result set and converting it to a relational result set. In SQL Server, there are two ways to achieve this: the OPENXML() function and the nodes() XQuery method.

**Shredding XML with OPENXML( )**

OPENXML() is a function that takes an XML document and converts it to a

relational result set. It accepts the parameters detailed in Table [4-2](#index_split_001.html#p154).

141

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Table 4-2.** OPENXML() Parameters*

**Parameter**

**Description**

idoc

A pointer to an internal representation of the XML document

rowpattern

The path to the lowest level of the XML document that should be

converted to rows

flags

An optional parameter that specifies the mapping between

the XML data and relational result set. The possible values are

detailed in T[able 4-3](#index_split_001.html#p154).

Table [4-3](#index_split_001.html#p154) details the valid flags values that can be passed to OPENXML().

***Table 4-3.** Flags Values*

**Value**

**Description**

0

Defaults to attribute-centric mapping

1

use attribute-centric mapping, unless XML_ELEMENTS is specified. If

XML_ELEMENTS is specified, attributes will be mapped first, followed by

elements.

2

element-centric mapping, unless XML_ATTRIBUTES is specified. If

XML_ATTRIBUTES is specified, elements will be mapped first, followed

by attributes.

8

XML_ATTRIBUTES and XML_ELEMENTS, combined with a logical OR

In addition to the function’s parameters, a WITH clause can also be

specified, which defines the schema of the result set to be returned. While the WITH clause is optional, not specifying the schema will cause an edge table to be returned. The schema is used to specify a relation column

name, a relational data type, and the path to the relevant node.

142

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

**Note** The flags will be overridden by specific mapping in the

WITH clause.

A limitation of the OPENXML() function is that it does not parse an XML

document itself. Instead, you must parse the document before calling

the function. This can be achieved using the sp_xml:preparedocument

system stored procedure. This procedure parses an XML document, using

the MSXML parser, and creates a handle to the parsed version of the

document that is ready for consumption.

After the OPENXML() function has completed, you should use the

sp_xml:removedocument system stored procedure to remove the parsed

document from memory. It is important to remember to tear down the

parsed document, or you could start running into memory issues, if the

sp_xml:preparedocument procedure is called frequently.

To see the OPENXML() function in action, see the script in Listing [4-28.](#index_split_001.html#p155)

This script passes a sales orders document into a variable, before using

OPENXML() to shred the data.

**Tip** The structure of the XML document has been changed from

previous examples in this chapter, to allow a broader demonstration

of the navigation of an XML document with OPENXML().

***Listing 4-28.*** Using OPENXML()

DECLARE @SalesOrder XML ;

DECLARE @ParseDoc INT ;

SET @SalesOrder = '

<SalesOrders>

<Order>

143

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

<OrderDate>2013-01-02</OrderDate>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

</OrderHeader>

<OrderDetails>

<Product ProductID="45" ProductName="Developer joke

mug - there are 10 types of people in the world (Black)"

Price="13" Qty="7" />

<Product ProductID="58" ProductName="RC toy sedan car with remote control (Black) 1/50 scale" Price="25" Qty="4" />

</OrderDetails>

</Order>

<Order>

<OrderDate>2013-01-02</OrderDate>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

</OrderHeader>

<OrderDetails OrderID = "122">

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="6" />

<Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />

<Product ProductID="111" ProductName="Superhero action

jacket (Blue) M" Price="30" Qty="10" />

<Product ProductID="116" ProductName="Superhero action

jacket (Blue) 4XL" Price="34" Qty="4" />

</OrderDetails>

</Order>

</SalesOrders>' ;

EXEC sp_xml:preparedocument @ParseDoc OUTPUT, @SalesOrder ;

SELECT *

144

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

FROM OPENXML(@ParseDoc, '/SalesOrders/Order/OrderDetails/Product')

WITH (

ProductID INT '@ProductID',

ProductName NVARCHAR(200) '@ProductName',

Quantity INT '@Qty',

OrderID INT '../@OrderID',

OrderDate DATE '../../OrderDate',

CustomerName NVARCHAR(200) '../../OrderHeader/CustomerName'

) ;

EXEC sp_xml:removedocument @ParseDoc ;

When you examine the WITH clause of the OPENXML() statement of this

script, you will see that we first specify the relational column name. By looking at the Quantity mapping, you will see that the relational names

do not have to match the names of the XML nodes. We then specify a

relational data type for the resultant value, before specifying a path to the node that we wish to retrieve. This is the most interesting aspect of the query. You will notice that we have used the rowpattern parameter

of the OPENXML() function to map down to the Product element, as the

lowest level. This means that the Product element is the starting point for our paths. The ProductID, ProductName, and Quantity attributes are all

attributes of the Product element. Therefore, we prefix these with the @

symbol, to designate that they are attributes, but no other path mapping

is required. The OrderID attribute is an attribute of the OrderDetails

element. Because the OrderDetails element is one level above the

Product element, we use the ../ syntax, to specify that we must navigate

up one level. The OrderDate element is two levels above the Product

element; therefore, we use ../ twice, to indicate we should move two

levels up the hierarchy. Also, note that because OrderDate is an element, not an attribute, we have not prefixed the node with an @ symbol. Finally, 145

![](index-158_1.jpg)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

to map to the CustomerName element, we first must navigate two levels up

the hierarchy, using the ../ syntax. We then must drop down into a sibling node (OrderHeader), to retrieve the CustomerName element.

The results of the query can be found in Figure [4-10\. N](#index_split_001.html#p158)ote that NULL

values have been returned for OrderID against two of the products. This is because no OrderID element was specified against the second sales order.

***Figure 4-10.** Results of using OPENXML()*

**Shredding XML with Nodes**

If you want to shred the data from a column instead of using a variable, you can avoid the need for iterative logic by shredding the data with XQuery.

Specifically, the nodes() method can be used to identify the nodes that

should be mapped to relational columns. This can be combined with the

value() method, to extract the data from the nodes. For example, consider the query in Listin[g 4-29. T](#index_split_001.html#p159)his query is functionally equivalent to the query in Listing [4-28](#index_split_001.html#p155) but uses the nodes() method instead of OPENXML().

146

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Listing 4-29.*** Using nodes() to Shred XML

DECLARE @SalesOrder XML ;

SET @SalesOrder = '

<SalesOrders>

<Order>

<OrderDate>2013-01-02</OrderDate>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

</OrderHeader>

<OrderDetails>

<Product ProductID="45" ProductName="Developer joke

mug - there are 10 types of people in the world (Black)"

Price="13" Qty="7" />

<Product ProductID="58" ProductName="RC toy sedan car with remote control (Black) 1/50 scale" Price="25" Qty="4" />

</OrderDetails>

</Order>

<Order>

<OrderDate>2013-01-02</OrderDate>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

</OrderHeader>

<OrderDetails OrderID = "122">

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="6" />

<Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />

<Product ProductID="111" ProductName="Superhero action

jacket (Blue) M" Price="30" Qty="10" />

147

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

<Product ProductID="116" ProductName="Superhero action

jacket (Blue) 4XL" Price="34" Qty="4" />

</OrderDetails>

</Order>

</SalesOrders>' ;

SELECT

TempCol.value('@ProductID', 'INT') AS ProductID

, TempCol.value('@ProductName', 'NVARCHAR(70)') AS

ProductName

, TempCol.value('@Qty', 'INT') AS Quantity

, TempCol.value('../@OrderID', 'INT') AS OrderID

, TempCol.value('../../OrderDate[1]', 'NVARCHAR(10)')

AS OorderDate

, TempCol.value('../../OrderHeader[1]/CustomerName[1]',

'NVARCHAR(15)') AS CustomerName

FROM @SalesOrder.nodes('SalesOrders/Order/OrderDetails/

Product') TempTable(TempCol) ;

When reviewing this script, you should pay particular attention to the

FROM clause. Here, we pass an XQuery expression into the nodes() method,

to define the grain. We then define arbitrary names for a table and column, which will contain the intermediate results set.

The nodes() method can also be used in conjunction with the query()

method, to shred an XML document into smaller XML documents. For

example, consider the query in Listing [4-30\. T](#index_split_001.html#p161)his query extracts the Product elements from the XML document, in XML format.

148

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Listing 4-30.*** Using the nodes() Method with the query() Method DECLARE @SalesOrder XML ;

SET @SalesOrder = '

<SalesOrders>

<Order>

<OrderDate>2013-01-02</OrderDate>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

</OrderHeader>

<OrderDetails>

<Product ProductID="45" ProductName="Developer joke

mug - there are 10 types of people in the world (Black)"

Price="13" Qty="7" />

<Product ProductID="58" ProductName="RC toy sedan car

with remote control (Black) 1/50 scale" Price="25"

Qty="4" />

</OrderDetails>

</Order>

<Order>

<OrderDate>2013-01-02</OrderDate>

<OrderHeader>

<CustomerName>Camille Authier</CustomerName>

</OrderHeader>

<OrderDetails OrderID = "122">

<Product ProductID="22" ProductName="DBA joke mug - it

depends (White)" Price="13" Qty="6" />

<Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />

<Product ProductID="111" ProductName="Superhero action

jacket (Blue) M" Price="30" Qty="10" />

149

![](index-162_1.jpg)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

<Product ProductID="116" ProductName="Superhero action

jacket (Blue) 4XL" Price="34" Qty="4" />

</OrderDetails>

</Order>

</SalesOrders>' ;

SELECT

TempCol.query('.') AS Product

FROM @SalesOrder.nodes('SalesOrders/Order/OrderDetails/

Product') TempTable(TempCol) ;

The results of this query can be seen in Figure [4-11](#index_split_001.html#p162).

***Figure 4-11.** Results of using the nodes() method with the query()* *method*

The biggest benefit of the nodes() method over OPENXML() is its

ease of use against a table. The CROSS APPLY operator can be used to

apply the nodes() method to multiple rows within a table. The query in

Listin[g 4-31 demons](#index_split_001.html#p163)trates this, by calling the nodes() method against the OrderSummary column in the Sales.CustomerOrderSummary table. The

query will return one row for every product on every sales order placed by a customer with the CustomerID of 814\. The results are then ordered by

the quantity of each product.

150

![](index-163_1.jpg)

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

***Listing 4-31.*** Using the nodes() Method Against a Table

USE WideWorldImporters

GO

SELECT

CustomerID

, TempCol.value('@Qty', 'INT') AS Quantity

, TempCol.value('@ProductName', 'NVARCHAR(70)') AS

ProductName

, TempCol.query('.') AS Product

FROM Sales.CustomerOrderSummary

CROSS APPLY OrderSummary.nodes('SalesOrders/Order/OrderDetails/

Product') TempTable(TempCol)

WHERE CustomerID = 841

ORDER BY TempCol.value('@Qty', 'INT') DESC ;

Partial results of this query can be seen in Figur[e 4-12](#index_split_001.html#p163).

***Figure 4-12.** Results of using the nodes() method against a table* 151

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

**Using Schemas**

As discussed in Chapt[er 3](#index_split_000.html#p61), an XML document can be bound to a schema, to ensure that its structure meets the client contract. In SQL Server, we can define an XSD schema by creating an XML SCHEMA COLLECTION. To

demonstrate this, imagine that we wanted to bind our OrderSummary

column, in the Sales.CustomerOrderSummary table, to a schema. The first

step would be to create the XSD SCHEMA COLLECTION. We could achieve this

using the code in Listin[g 4-32.](#index_split_001.html#p164)

***Listing 4-32.*** Creating an XML SCHEMA COLLECTION

USE WideWorldImporters

GO

CREATE XML SCHEMA COLLECTION OrderSummary AS

N'<?xml version="1.0" encoding="utf-16"?>

<xs:schema attributeFormDefault="unqualified"

elementFormDefault="qualified" xmlns:xs="http://www.

w3.org/2001/XMLSchema">

<xs:element name="SalesOrders">

<xs:complexType>

<xs:sequence>

<xs:element maxOccurs="unbounded" name="Order">

<xs:complexType>

<xs:sequence>

<xs:element name="OrderHeader">

<xs:complexType>

<xs:sequence>

<xs:element name="CustomerName"

type="xs:string" />

<xs:element name="OrderDate" type="xs:date" />

152

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

<xs:element name="OrderID"

type="xs:unsignedInt" />

</xs:sequence>

</xs:complexType>

</xs:element>

<xs:element name="OrderDetails">

<xs:complexType>

<xs:sequence>

<xs:element maxOccurs="unbounded"

name="Product">

<xs:complexType>

<xs:attribute name="ProductID"

type="xs:unsignedByte" use="required" />

<xs:attribute name="ProductName"

type="xs:string" use="required" />

<xs:attribute name="Price"

type="xs:decimal" use="required" />

<xs:attribute name="Qty"

type="xs:unsignedShort" use="required" />

</xs:complexType>

</xs:element>

</xs:sequence>

</xs:complexType>

</xs:element>

</xs:sequence>

</xs:complexType>

</xs:element>

</xs:sequence>

</xs:complexType>

</xs:element>

</xs:schema>' ;

153

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

**Tip** Because the schema does not specify a namespace, it will be

associated with the default empty string namespace. A namespace

can be added to the schema, using the <xs:schema> attribute

xmlns:ns=http://your-namespace.

We could bind our schema to the OrderSummary column by using the

query in Listin[g 4-33.](#index_split_001.html#p166)

***Listing 4-33.*** Binding a Schema to a Column

ALTER TABLE Sales.CustomerOrderSummary

ALTER COLUMN OrderSummary XML(OrderSummary) ;

We can also reference a schema when constructing or querying XML

data. The FOR XML clause includes a WITH XMLNAMESPACES option that can

be used to specify the target namespace of the resultant XML document,

and XQuery methods, such as query, can begin with a declare namespace

statement.

**Tip** A full discussion of the use of namespaces is beyond the scope of this book. however, examples of using WITH XMLNAMESPACES

can be found at [https://docs.microsoft.com/en-us/sql/t-](https://docs.microsoft.com/en-us/sql/t-sql/xml/with-xmlnamespaces?view=sql-server-2017)

[sql/xml/with-xmlnamespaces?view=sql- server- 2017,](https://docs.microsoft.com/en-us/sql/t-sql/xml/with-xmlnamespaces?view=sql-server-2017)

and examples of using a namespace with the query method can be

found at [https://docs.microsoft.com/en-us/sql/t-sql/](https://docs.microsoft.com/en-us/sql/t-sql/xml/value-method-xml-data-type?view=sql-server-2017)

[xml/value-method-xml-data-type?view=sql- server- 2017.](https://docs.microsoft.com/en-us/sql/t-sql/xml/value-method-xml-data-type?view=sql-server-2017)

154

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

**Summary**

XQuery is a language that can be used to query XML data when it is stored in SQL Server columns and variables. The exist() method can be used

to check for the existence of a node or a node containing a specific value.

The value() method can be used to extract a scalar value from an XML

document, and the query() method can be used to return a subset of an

XML document still in XML format.

FLWOR statements can be used to help navigate and iterate an XML

document. The for statement binds a variable to an input sequence. The

let statement is used to assign an XQuery expression to a variable. The

where statement can be used to filter the results that are returned. The

order by statement can optionally be used to order the results of the FLWOR

statement. The return statement specifies what data will be returned.

T-SQL variables and columns can be passed into XQuery statements.

When this technique is adopted, it is known as a cross-domain query.

It allows for values to easily be bound to XQuery statements, helping to

simplify logic and reduce duplicate code.

XML data can be modified by using the modify() method. This

method allows developers to use one of three options: insert, replace

value of, or delete. When inserting data, there are further options that

developers can take advantage of, to give them granular control over where the insert occurs. For example, you can choose to insert first, last, after a node, or before a node.

XML data can be converted into relational data (that is, shredded)

by using either the OPENXML() function or the nodes() XQuery method.

When the OPENXML() function is used, the XML document to be

shredded must first be parsed. This can be achieved with the sys.sp_

xml:preparedocument system-stored procedure. Once the document has

been shredded, the parse tree should be removed from memory, by using

the sys.sp_xml:removedocument system-stored procedure.

155

ChAPTeR 4 QueRyIng AnD ShReDDIng XML

When the nodes() XQuery method is used to shred data, it can be

used in conjunction with either the value() method, which will shred

the data into relational results, or the query() method, which will shred the data into smaller XML documents. You can also use nodes() with a

combination of both value() and query(). The biggest advantage of the

nodes() method is the ease with which it can be applied to a column,

using the CROSS APPLY operator.

156

**CHAPTER 5**

**XML Indexes**

As discussed in Chapters [3](#index_split_000.html#p61) [and 4](#index_split_001.html#p125), SQL Server allows you to store data in tables, in a native XML format, using the XML data type. Like other large object types, it can store up to a maximum of 2GB per tuple. Although

standard operators such as = and LIKE can be used against XML columns,

you also have the option of using XQuery expressions (discussed in this

chapter). They can be rather inefficient, however, unless you create XML

indexes.

XML indexes will outperform full-text indexes for most queries against

XML columns. SQL Server offers support for primary XML indexes and

three types of secondary XML indexes: PATH, VALUE, and PROPERTY. Each of

these indexes will be discussed in the following sections. First, however, I will briefly discuss clustered indexes, as a clustered index must exist on the table before you can create an XML index.

**Preparing the Environment**

Because the WideWorldImporters database has no tables that contain

native XML columns, we will create an OrderSummary table, for

demonstrations within this chapter. The table will contain three columns: an IDENTITY column (named ID), a CustomerID column, and an XML

column (called OrderSummary), which will contain a summary of all

orders that a customer has placed. The table can be created and populated using the script in Listing [5-1](#index_split_001.html#p170).

© Peter A. Carter 2018

157

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_5](#index_split_001.html#p169)

Chapter 5 XML IndeXes

***Listing 5-1.*** Creating an OrderSummary Table

USE WideWorldImporters

GO

CREATE TABLE Sales.OrderSummary

(

ID INT NOT NULL IDENTITY,

CustomerID INT NOT NULL,

OrderSummary XML

) ;

INSERT INTO Sales.OrderSummary (CustomerID, OrderSummary)

SELECT

CustomerID,

(

SELECT

CustomerName 'OrderHeader/CustomerName'

, OrderDate 'OrderHeader/OrderDate'

, OrderID 'OrderHeader/OrderID'

, (

SELECT

LineItems2.StockItemID

'@ProductID'

, StockItems.StockI

temName '@ProductName'

, LineItems2.UnitPrice

'@Price'

, Quantity '@Qty'

FROM Sales.OrderLines LineItems2

INNER JOIN Warehouse.StockItems

StockItems

158

Chapter 5 XML IndeXes

ON LineItems2.StockItemID

= StockItems.StockItemID

WHERE LineItems2.OrderID =

Base.OrderID

FOR XML PATH('Product'), TYPE

) 'OrderDetails'

FROM

(

SELECT DISTINCT

Customers.CustomerName

, SalesOrder.OrderDate

, SalesOrder.OrderID

FROM Sales.Orders SalesOrder

INNER JOIN Sales.OrderLines LineItem

ON SalesOrder.OrderID =

LineItem.OrderID

INNER JOIN Sales.Customers Customers

ON Customers.CustomerID =

SalesOrder.CustomerID

WHERE customers.CustomerID = OuterCust.

CustomerID

) Base

FOR XML PATH('Order'), ROOT ('SalesOrders'), TYPE

) AS OrderSummary

FROM Sales.Customers OuterCust ;

**Clustered Indexes**

A clustered index causes the data pages of a table to be logically stored in the order of the clustered index key. The clustered index key can be a single column or a set of columns. This is often the table’s primary key, but 159

![](index-172_1.jpg)

Chapter 5 XML IndeXes

this is not enforced, and there are some circumstances in which you would want to use a different column. This will be discussed in more detail later in this chapter.

**Tables Without a Clustered Index**

When a table exists without a clustered index, it is known as a heap. A heap consists of an IAM (index allocation map) page (or pages) and a series of data pages that are not linked together or stored in order. The only way

SQL Server can determine the pages of the table is by reading the IAM

page(or pages). When a table is stored as a heap, every time the table is accessed, SQL Server must read every single page in the table, even if you only want to return one row. The diagram in Figure [5-1](#index_split_001.html#p172) illustrates how a heap is structured.

***Figure 5-1.** Heap structure*

160

![](index-173_1.jpg)

Chapter 5 XML IndeXes

When data is stored on a heap, SQL Server must maintain a unique

identifier for each row. It does this by creating a RID (row identifier). Even if a table has nonclustered indexes, it is still stored as a heap, unless there is a clustered index. When nonclustered indexes are created on a heap, the RID is used as a pointer, so that nonclustered indexes can link back to the correct row in the base table. Nonclustered indexes store the RID with a

format of FileID: Page ID: Slot Number.

**Tables with a Clustered Index**

When you create a clustered index on a table, a B-Tree (balanced tree)

structure is created. This allows for more efficient search operations to be performed, by creating a tiered set of pointers to the data, as illustrated in Figure [5-2](#index_split_001.html#p173). The page at the top level of this hierarchy is called the root node. The bottom level of the structure is called the leaf level, and with a clustered index, the leaf level consists of the actual data pages of the table. There can be one or more intermediate levels of B-Tree structures, depending on the size of the table.

***Figure 5-2.** Clustered index structure*

161

Chapter 5 XML IndeXes

The diagram in Figur[e 5-2 sho](#index_split_001.html#p173)ws that while the leaf level is the data itself, the levels above contain pointers to the pages below them in the

tree. This allows for SQL Server to perform a seek operation. This is a

very efficient method of returning a small number of rows. It works by

navigating its way down the B-Tree, using the pointers to find the row(s) it requires. We can see that, if required, SQL Server can still scan all pages of the table, in order to retrieve the required rows. This is known as a

Clustered Index Scan. Alternatively, SQL Server may decide to combine

these two methods, to perform a range scan. Here, SQL Server will seek

the first value of the required range and then scan the leaf level, until it encounters the first value that is not required. SQL Server can do

this because the table is ordered by the index key, meaning that it can

guarantee that there will be no other matching values later in the table.

**Clustering the Primary Key**

The primary key of a table is often the natural choice for the clustered

index. In fact, by default, unless you specify otherwise, or unless a

clustered index already exists on the table, creating a primary key

will automatically generate a clustered index on that key. There are

circumstances in which the primary key is not the correct choice for the

clustered index. An example of this that I have witnessed is a third-party application that required the primary key of the table to be a GUID.

A GUID (globally unique identifier) is used to guarantee uniqueness across the entire network.

This introduces two major problems if the clustered index were to be

built on the primary key. The first is size. A GUID is 16 bytes long. When a table has nonclustered indexes, the clustered index key is stored in every nonclustered index. For unique nonclustered indexes, it is stored for every row at the leaf level, and for non-unique nonclustered indexes, it is also stored at every row in the root and intermediate levels of the index as well.

162

Chapter 5 XML IndeXes

When you multiply 16 bytes by millions of rows, this will drastically

increase the size of the indexes, making them less efficient.

The second issue is that when a GUID is generated, it is a random

value. Because the data in your table is stored in the order of the clustered index key, for good performance, you need the values of this key to

be generated in sequential order. Generating random values for your

clustered index key will result in the index becoming more and more

fragmented every time you insert a new row. Fragmentation will be

discussed later in this chapter.

There is a workaround for the second issue, however. SQL Server has a

function called NEWSEQUENTIALID() that will always generate a GUID value

higher than previous values generated on the server. Therefore, if you

use this function in the Default constraint of your primary key, you can

enforce sequential inserts.

**Caution** after the server has been restarted, NEWSEQUENTIALID()

can start with a lower value. this may lead to fragmentation.

If the primary key must be a GUID, or another wide column, such

as National Insurance Number, or a set of columns forming a natural

key, such as Customer ID, Order Date, and Product ID, it is highly

recommended that you create an additional column in your table. This

column could be an INT or BIGINT, depending on the number of rows you

expect the table to have, and could use either the IDENTITY property or

a SEQUENCE to create a narrow, sequential key that can be used for your

clustered index. I recommend ensuring a narrow key, as it will be included in all nonclustered indexes on the table. It will also use less memory when joining tables.

163

Chapter 5 XML IndeXes

**Caution** If you intend to use XML indexes, the clustered index must be created on the primary key.

**Performance Considerations for Clustered Indexes**

Because an IAM page lists the extents of a heap table in the order in which they are stored in the data file, as opposed to the order of the index key, a table scan of a heap may prove to be slightly faster than a clustered index scan, unless the clustered index has 0% fragmentation, which is rare.

Inserts into a clustered index may be faster than inserts into a heap,

when the clustered index key is ever-increasing. This is especially true

when there are multiple inserts happening in parallel, because a heap will experience more contention on system pages (GAM/SGAM/PFS) when

the database engine is looking for spaces to place the new data. If the

clustered index key is not ever-increasing, however, then inserts will lead to page splits and fragmentation. The knock-on effect is that inserts would be slower than they would be into a heap. A large insert into a heap may

also be faster, if you take out a table lock and take advantage of minimally logged inserts. This is because of reduced IO to the transaction log.

Updates that cause a row to relocate, due to a change in size, will be

faster when performed against a clustered index, as opposed to a heap.

This is, for the same reason as mentioned above for insert operations,

where there will be more contention against the system pages. When

updated, rows may change in size, for reasons such as updating a VARCHAR

column with a longer string. If the update to the row can be made in

place (without relocating the row), there is likely to be little difference in performance. Deletes may also be slightly faster into a clustered index

than into a heap, but the difference will be less noticeable than for update operations.

164

![](index-177_1.jpg)

Chapter 5 XML IndeXes

**Creating a Clustered Index**

With SSMS (SQL Server Management Studio), we can create a clustered

index on the ID column of the OrderSummary table, by expanding

Databases ➤ WideWorldImporters ➤ Tables ➤ Sales.OrderSummary in

Object Explorer, right-clicking the Indexes node, and selecting New index

➤ Clustered index. This will cause the New Index dialog box to be invoked, as shown in Figur[e 5-3.](#index_split_001.html#p177)

***Figure 5-3.** New Index dialog box*

**Caution** If you plan to follow later demonstrations in this chapter, do not execute the steps illustrated in F[igures 5-3 and](#index_split_001.html#p177) [5-4](#index_split_001.html#p178). also, a[void executing the script in Listing 5-2.](#index_split_001.html#p178) If you do create the index, you will have to drop it before running further examples.

165

![](index-178_1.jpg)

Chapter 5 XML IndeXes

On the General page of the dialog box, give the index a descriptive

name, then use the Add button, to select the column(s) that the index will be built on, as shown in Figur[e 5-4.](#index_split_001.html#p178)

***Figure 5-4.** Add columns dialog box*

Alternatively, this clustered index could be created using the script in

Listin[g 5-2.](#index_split_001.html#p178)

***Listing 5-2.*** Creating a Clustered Index

USE WideWorldImporters

GO

CREATE CLUSTERED INDEX [ClusteredIndex-OrderSummary-ID] ON

Sales.OrderSummary (ID) ;

GO

166

Chapter 5 XML IndeXes

**Note** advanced options for creating clustered indexes are beyond

the scope of this book, but further information can be found in *Pro*

*SQL Server Administration* (apress, 2015), available at [www.apress.](http://www.apress.com/gb/book/9781484207116)

[com/gb/book/9781484207116](http://www.apress.com/gb/book/9781484207116).

Because our XML indexes require the clustered index to be built on a

primary key, instead of executing the preceding script, we should instead run the script in Listing [5-3](#index_split_001.html#p179). This script will create a primary key on the ID

column and then a clustered index on the primary key.

***Listing 5-3.*** Creating a Primary Key and Clustered Index

USE WideWorldImporters

GO

ALTER TABLE Sales.OrderSummary ADD CONSTRAINT

PK_OrderSummary PRIMARY KEY CLUSTERED (ID) ;

**Primary XML Indexes**

A primary XML index is actually a multicolumn clustered index on an

internal system table called the Node table. This table stores a shredded representation on the XML objects within an XML column, along with the

clustered index key of the base table. This means that a table must have a clustered index before a primary XML index can be created. Additionally,

the clustered index must be created on the primary key and must consist of 32 columns or less.

The system table stores enough information that the scalar or XML

subtrees required by a query can be reconstructed from the index itself.

This information includes the node ID and name, the tag name and URI,

a tokenized version of the node’s data type, the first position of the node 167

![](index-180_1.jpg)

Chapter 5 XML IndeXes

value in the document, pointers to the long node value and binary value,

the nullability of the node, and the value of the base table’s clustered index key for the corresponding row.

Primary XML indexes can provide a performance improvement when

a query must shred scalar values from an XML document (or documents)

or return a subset of nodes from an XML document (or documents).

**Creating Primary XML Indexes**

To create a primary XML index using SSMS, drill through Databases ➤

WideWorldImporters ➤ Tables ➤ Sales.OrderSummary in Object Explore.

Then select New Index ➤ Primary XML Index from the context menu of

indexes. This will cause the General page of the New Index dialog box to be displayed. This is shown in Figure [5-5](#index_split_001.html#p180).

***Figure 5-5.** New Index dialog box (Primary XML)*

168

![](index-181_1.jpg)

Chapter 5 XML IndeXes

Here, we will give the index a descriptive name and then use the Add

button, to add the required XML column, as shown in Figure [5-6](#index_split_001.html#p181).

***Figure 5-6.** Add column dialog box (Primary XML)*

From the Options tab of the New Index dialog box (Figur[e 5-7), w](#index_split_001.html#p182)e can set the options detailed in Ta[ble 5-1.](#index_split_001.html#p182)

169

![](index-182_1.jpg)

Chapter 5 XML IndeXes

***Figure 5-7.** New Index dialog box—Options page (Primary XML)*

***Table 5-1.** Primary XML Index Options*

**Option**

**Description**

allow row Locks

specifies if row locks can be acquired when accessing

the index

allow page Locks

specifies if page locks can be acquired when accessing

the index

Maxdop

has no effect for building primary XML indexes, as this

operation is always single threaded

sort in tempdB

If specified, sort in tempdB will cause the intermediate result

set to be stored in tempdB, as opposed to the user database.

this could mean that the index is built faster.

( *continued*)

170

Chapter 5 XML IndeXes

***Table 5-1.*** ( *continued*)

**Option**

**Description**

Fill Factor

specifies a percentage of free space that will be left on each

index page at the lowest level of the index. the default is

0 (100% full), meaning that only enough space for a single

row will be left. specifying a percentage lower than 100,

for example, specifying 70, will leave 30% free space and

can reduce page splits, if there are likely to be frequent row

inserts.

pad Index

applies a fill factor (see preceding) to the intermediate levels

of a B-tree

Alternatively, to create the index via T-SQL, you could use the script in Listin[g 5-4.](#index_split_001.html#p183)

***Listing 5-4.*** Creating a Primary XML Index

USE WideWorldImporters

GO

CREATE PRIMARY XML INDEX [PrimaryXmlIndex-OrderSummary-

OrderSummary]

ON Sales.OrderSummary ([OrderSummary]) ;

GO

**Secondary XML Indexes**

Secondary XML indexes can only be created on XML columns that already

have a primary XML index. Behind the scenes, secondary XML indexes are

actually nonclustered indexes on the internal Node table. Secondary XML

indexes can improve query performance for queries that use specific types of XQuery processing.

171

Chapter 5 XML IndeXes

A PATH secondary XML index is built on the Node ID and VALUE

columns of the Node table. This type of index offers performance

improvements to queries that use path expressions, such as the exists()

XQuery method. A VALUE secondary XML index is the reverse of this and

is built on the VALUE and Node ID columns. This type of index will offer

performance improvements to queries that search for values, without

knowing the name of the XML element or attribute that contains the value

being searched for.

Finally, a PROPERTY secondary XML index is built on the clustered

index key of the base table, the Node ID, and the VALUE columns of the

Node table. This type of index performs very well if the query is trying to retrieve nodes from multiple tuples of the column.

**Creating Secondary XML Indexes**

To create a secondary XML index in SSMS, drill through Databases ➤

WideWorldImporters ➤ Tables ➤ OrderSummary in Object Explorer. Next,

select New Index ➤ Secondary XML Index from the context menu of the

Indexes node. This will cause the New Index dialog box to be displayed, as illustrated in Figur[e 5-8.](#index_split_001.html#p185)

172

![](index-185_1.jpg)

Chapter 5 XML IndeXes

***Figure 5-8.** New Index dialog box (Secondary XML)*

On the General tab of the New Index dialog box, we have first given the

index a descriptive name. Next, we select the appropriate primary XML

index from the Primary XML Index drop-down list. Finally, we select the

type of secondary XML index that we wish to create, from the Secondary

XML Index Type drop-down box. In this case, we have chosen to create a

PATH index.

Figur[e 5-9 illus](#index_split_001.html#p186)trates the Options tab of the New Index dialog box. For details of each option, please refer to Table [5-1](#index_split_001.html#p182).

173

![](index-186_1.jpg)

Chapter 5 XML IndeXes

***Figure 5-9.** New Index dialog box—Options tab (Secondary XML Index)* Alternatively, to create this index with T-SQL, you could use the script

in Listing [5-5](#index_split_001.html#p186).

***Listing 5-5.*** Creating a Secondary XML Index

USE WideWorldImporters

GO

CREATE XML INDEX [SecondaryXmlIndex-OrderSummary-Path]

ON Sales.OrderSummary (OrderSummary)

USING XML INDEX [PrimaryXmlIndex-OrderSummary-OrderSummary] FOR

PATH ;

GO

174

Chapter 5 XML IndeXes

**Performance Considerations for XML Indexes**

In order to discuss the performance of XML indexes, let’s write a query that is well-suited to the PATH secondary XML index that we have created on

the OrderSummary table. The query in Listing [5-6](#index_split_001.html#p187) runs a query against the OrderSummary table and returns all rows indicating customers who have

ordered the Chocolate echidnas 250g product, which has a StockItemID of

223\. The first part of the script removes unchanged pages from the buffer cache and drops the plan cache, making it a fair test. The middle part of the script turns on time statistics, so we can accurately tell how long the query took to run.

**Tip** performance will vary, based on the specification of your

server and how many resources are being consumed by concurrent

processes. You should always check performance within your own

environment.

***Listing 5-6.*** Return Rows Where Customers Have Ordered

StockItemID 23

--Clear buffer cache and plan cache

DBCC DROPCLEANBUFFERS

DBCC FREEPROCCACHE

GO

--Turn on IO statistics to appear with results

SET STATISTICS TIME ON

GO

--Run query

SELECT *

175

![](index-188_1.jpg)

Chapter 5 XML IndeXes

FROM Sales.OrderSummary

WHERE OrderSummary.exist('/SalesOrders/Order/OrderDetails/

Product/.[@ProductID = 223]') = 1 ;

The statistics shown in Figur[e 5-10](#index_split_001.html#p188) show that the query took 1.95

seconds to complete.

***Figure 5-10.** Query results with PATH index*

Now let’s use the script in Listing [5-7](#index_split_001.html#p188) to drop the PATH index and run the query again. This time, only the primary XML index is available for use.

***Listing 5-7.*** Run Query Without PATH Index

DROP INDEX [SecondaryXmlIndex-OrderSummary-Path] ON Sales.

OrderSummary ;

GO

DBCC DROPCLEANBUFFERS

DBCC FREEPROCCACHE

GO

176

![](index-189_1.jpg)

Chapter 5 XML IndeXes

SET STATISTICS TIME ON

GO

SELECT *

FROM Sales.OrderSummary

WHERE OrderSummary.exist('/SalesOrders/Order/OrderDetails/

Product/.[@ProductID = 223]') = 1 ;

This time, as we can see from the statistics in Figur[e 5-11](#index_split_001.html#p189), the query took more than 2.7 seconds to complete.

***Figure 5-11.** Query results without PATH index*

Finally, let’s use the script in Listin[g 5-8 t](#index_split_001.html#p190)o drop the primary XML

Index, and run the query again, with no XML index support.

177

Chapter 5 XML IndeXes

***Listing 5-8.*** Drop Primary XML Index and Rerun Query

DROP INDEX [PrimaryXmlIndex-OrderSummary-OrderSummary] ON

Sales.OrderSummary ;

GO

DBCC DROPCLEANBUFFERS

DBCC FREEPROCCACHE

GO

SET STATISTICS TIME ON

GO

SELECT *

FROM Sales.OrderSummary

WHERE OrderSummary.exist('/SalesOrders/Order/OrderDetails/

Product/.[@ProductID = 223]') = 1 ;

You will notice from the statistics in Figur[e 5-12](#index_split_001.html#p191) that the query execution time has now risen to more than 4 seconds. While our table

only has fewer than 700 rows and results you see will vary, depending on

the performance of your machine, this example shows why creating XML

indexes is so important.

178

![](index-191_1.jpg)

Chapter 5 XML IndeXes

***Figure 5-12.** Results of query with no XML indexes*

**Summary**

Specialized XML indexes can be created on XML columns, to improve

the performance of queries that rely on interrogating XML data. There are four types of XML Index: Primary, Secondary PATH, Secondary VALUE, and

Secondary PROPERTY.

A primary XML index cannot be created on an XML column, unless the

table has a clustered primary key (a clustered index built on a primary key column). A secondary XML cannot be created unless a primary XML index

already exists on the XML column. XML indexes can be created before a

table is populated with data, however.

Queries that interrogate XML columns can be quite inefficient and

perform poorly, unless the correct XML indexes are created to support

them. XML indexes will always be more efficient on XML columns than

full-text indexes will be. As demonstrated in this chapter, XML query

performance is significantly impaired if XML indexes are not created

appropriately.

179

**CHAPTER 6**

**Understanding JSON**

As SQL Server evolves, more and more nonrelational features are being

added to the product, blurring the lines between relational and NoSQL

technologies. JSON is an example of this. JSON (JavaScript object notation) is a document format, designed as a method of lightweight data interchange.

It is similar to XML, in the respect that it is a self-describing, hierarchical data-interchange format. Unlike XML, however, JSON tags are minimal,

making JSON documents shorter and both easier to read and quicker to parse.

In this chapter, I will introduce the JSON format. I will discuss the

structure of a JSON document and compare it to an XML document.

Finally, I will discuss usage scenarios for JSON data within SQL Server.

**Understanding the JSON Format**

The basic JSON syntax uses name/value pairs, separated by a colon.

The JSON object is then enclosed by braces. The name must be a string,

enclosed with double quotes, and the value must be

• A string (enclosed by double quotes)

• A number

• A nested JSON object

• A Boolean value

• An array (enclosed by square brackets)

• NULL

© Peter A. Carter 2018

181

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_6](#index_split_001.html#p192)

Chapter 6 Understanding JsOn

For instance, consider the simple example in Listing [6-1](#index_split_001.html#p193).

***Listing 6-1.*** Simple JSON Document

{ "FirstName" : "Pete" }

If multiple name/value pairs occur within a JSON document, they

are separated by a comma. For example, consider the JSON document in

Listin[g 6-2\. Y](#index_split_001.html#p193)ou will notice that the value for age is not enclosed in double quotes, because it is a number, as opposed to a string.

***Listing 6-2.*** Simple JSON Document with Multiple Name/Value Pairs

{ "FirstName" : "Pete" , "LastName" : "Carter" , "Age" : 38 }

If you were to represent as a JSON flat document a row set in a table,

the result would be an array of JSON objects. For example, consider the

query in Listin[g 6-3.](#index_split_001.html#p193)

***Listing 6-3.*** Top Vehicle Temperatures

USE WideWorldImporters

GO

SELECT TOP 3

VehicleRegistration

, ChillerSensorNumber

, Temperature

FROM Warehouse.VehicleTemperatures

ORDER BY Temperature DESC ;

182

![](index-194_1.jpg)

Chapter 6 Understanding JsOn

This query will produce the results displayed in Figur[e 6-1.](#index_split_001.html#p194)

***Figure 6-1.** Top vehicle temperatures*

If this result set were to be expressed as a JSON document, it would

look like the document in Listin[g 6-4.](#index_split_001.html#p194)

***Listing 6-4.*** Top Vehicle Temperatures Expressed As JSON

[

{

"VehicleRegistration": "WWI-321-A",

"ChillerSensorNumber": 1,

"Temperature": 5

},

{

"VehicleRegistration": "WWI-321-A",

"ChillerSensorNumber": 2,

"Temperature": 5

},

{

183

Chapter 6 Understanding JsOn

"VehicleRegistration": "WWI-321-A",

"ChillerSensorNumber": 1,

"Temperature": 5

}

]

You can see that the results are an array of JSON objects; therefore, the document is enclosed in square brackets. Each JSON object (representing

a single row in the table) is enclosed in braces and separated by commas.

Within each JSON object, comma separated name/value pairs represent

each column in the tabular representation of the results.

The Warehouse.VehicleTemperatures table, in the WideWorldImporters

database also includes a column with the JSON data type, which records

the full sensor data. Consider the query in Listin[g 6-5.](#index_split_001.html#p195)

***Listing 6-5.*** Vehicle Temperatures with Full Sensor Data

USE WideWorldImporters

GO

SELECT TOP 3

VehicleRegistration

, ChillerSensorNumber

, Temperature

, FullSensorData

FROM Warehouse.VehicleTemperatures

ORDER BY Temperature DESC ;

184

![](index-196_1.jpg)

Chapter 6 Understanding JsOn

This query returns the results displayed in Figur[e 6-2.](#index_split_001.html#p196)

***Figure 6-2.** Results of temperatures with full sensor data*

If we were to represent this result set as a JSON document, we would

have an array of JSON objects in which one of the objects is a nested JSON

object, as shown in Listin[g 6-6.](#index_split_001.html#p196)

***Listing 6-6.*** Vehicle Temperatures with Full Sensor Data Expressed As JSON

[

{

"VehicleRegistration": "WWI-321-A",

"ChillerSensorNumber": 1,

"Temperature": 5,

"FullSensorData": "{\"Recordings\": [{\"type\":\"Feature\",

\"geometry\": {\"type\":\"Point\",

\"coordinates\":[-107.9037602,43.1198494] },

\"properties\":{\"rego\":\"WWI-321- A\",

\"sensor\":\"1,\"when\":\"2016-05- 31T09:34:39\",

\"temp\":5.00}} ]"

},

{

"VehicleRegistration": "WWI-321-A",

"ChillerSensorNumber": 2,

"Temperature": 5,

185

Chapter 6 Understanding JsOn

"FullSensorData": "{\"Recordings\": [{\"type\":\"Feature\",

\"geometry\": {\"type\":\"Point\",

\"coordinates\":[-108.3927541,58.1174136] },

\"properties\":{\"rego\":\"WWI-321- A\",

\"sensor\":\"2,\"when\":\"2016-05- 31T09:44:35\",

\"temp\":5.00}} ]"

},

{

"VehicleRegistration": "WWI-321-A",

"ChillerSensorNumber": 1,

"Temperature": 5,

"FullSensorData": "{\"Recordings\": [{\"type\":\"Feature\",

\"geometry\": {\"type\":\"Point\",

\"coordinates\":[-88.2125713,56.0198938] },

\"properties\":{\"rego\":\"WWI-321- A\",

\"sensor\":\"1,\"when\":\"2016-05- 30T08:14:17\",

\"temp\":5.00}} ]"

}

]

**Tip** the nested JsOn objects contain a backslash before each

double quote, as an escape character.

You will notice, in this example, that the value for each

FullSensorData node is a JSON object nested inside the JSON object,

which represents a row within the tabular result set.

A root node can also be added to a JSON document, sometimes used

to represent the name of the object’s type or abstraction. This can help give the document context. Listing [6-7 sho](#index_split_001.html#p198)ws the same document as Listing [6-6](#index_split_001.html#p196), but with a root node added.

186

Chapter 6 Understanding JsOn

***Listing 6-7.*** Adding a Root Node

{

"VehicleTemperatures": [

{

"VehicleRegistration": "WWI-321-A",

"ChillerSensorNumber": 1,

"Temperature": 5,

"FullSensorData": "{\"Recordings\": [{\"type\":\"Feature\",

\"geometry\": {\"type\":\"Point\",

\"coordinates\":[-107.9037602,43.1198494] },

\"properties\":{\"rego\":\"WWI-321- A\",

\"sensor\":\"1,\"when\":\"2016-05- 31T09:34:39\",

\"temp\":5.00}} ]"

},

{

"VehicleRegistration": "WWI-321-A",

"ChillerSensorNumber": 2,

"Temperature": 5,

"FullSensorData": "{\"Recordings\": [{\"type\":\"Feature\",

\"geometry\": {\"type\":\"Point\",

\"coordinates\":[-108.3927541,58.1174136] },

\"properties\":{\"rego\":\"WWI-321- A\",

\"sensor\":\"2,\"when\":\"2016-05- 31T09:44:35\",

\"temp\":5.00}} ]"

},

{

"VehicleRegistration": "WWI-321-A",

"ChillerSensorNumber": 1,

"Temperature": 5,

187

Chapter 6 Understanding JsOn

"FullSensorData": "{\"Recordings\": [{\"type\":\"Feature\",

\"geometry\": {\"type\":\"Point\",

\"coordinates\":[-88.2125713,56.0198938] },

\"properties\":{\"rego\":\"WWI-321- A\",

\"sensor\":\"1,\"when\":\"2016-05- 30T08:14:17\",

\"temp\":5.00}} ]"

}

]

}

**JSON vs. XML**

Let’s compare a simple JSON document against an XML equivalent

and examine the differences. Consider the XML document in Listin[g 6-8,](#index_split_001.html#p199)

which represents the salespeople within the WideWorldImporters

database.

***Listing 6-8.*** Sales People—XML

<SalesPeople>

<SalesPerson>

<PersonID>2</PersonID>

<FullName>Kayla Woodcock</FullName>

<PreferredName>Kayla</PreferredName>

<LogonName>kaylaw@wideworldimporters.com</LogonName>

<PhoneNumber>(415) 555-0102</PhoneNumber>

<EmailAddress>kaylaw@wideworldimporters.com</EmailAddress>

</SalesPerson>

<SalesPerson>

<PersonID>3</PersonID>

<FullName>Hudson Onslow</FullName>

<PreferredName>Hudson</PreferredName>

188

Chapter 6 Understanding JsOn

<LogonName>hudsono@wideworldimporters.com</LogonName>

<PhoneNumber>(415) 555-0102</PhoneNumber>

<EmailAddress>hudsono@wideworldimporters.com</EmailAddress>

</SalesPerson>

<SalesPerson>

<PersonID>6</PersonID>

<FullName>Sophia Hinton</FullName>

<PreferredName>Sophia</PreferredName>

<LogonName>sophiah@wideworldimporters.com</LogonName>

<PhoneNumber>(415) 555-0102</PhoneNumber>

<EmailAddress>sophiah@wideworldimporters.com</EmailAddress>

</SalesPerson>

<SalesPerson>

<PersonID>7</PersonID>

<FullName>Amy Trefl</FullName>

<PreferredName>Amy</PreferredName>

<LogonName>amyt@wideworldimporters.com</LogonName>

<PhoneNumber>(415) 555-0102</PhoneNumber>

<EmailAddress>amyt@wideworldimporters.com</EmailAddress>

</SalesPerson>

<SalesPerson>

<PersonID>8</PersonID>

<FullName>Anthony Grosse</FullName>

<PreferredName>Anthony</PreferredName>

<LogonName>anthonyg@wideworldimporters.com</LogonName>

<PhoneNumber>(415) 555-0102</PhoneNumber>

<EmailAddress>anthonyg@wideworldimporters.com</EmailAddress>

</SalesPerson>

<SalesPerson>

<PersonID>13</PersonID>

<FullName>Hudson Hollinworth</FullName>

189

Chapter 6 Understanding JsOn

<PreferredName>Hudson</PreferredName>

<LogonName>hudsonh@wideworldimporters.com</LogonName>

<PhoneNumber>(415) 555-0102</PhoneNumber>

<EmailAddress>hudsonh@wideworldimporters.com</EmailAddress>

</SalesPerson>

<SalesPerson>

<PersonID>14</PersonID>

<FullName>Lily Code</FullName>

<PreferredName>Lily</PreferredName>

<LogonName>lilyc@wideworldimporters.com</LogonName>

<PhoneNumber>(415) 555-0102</PhoneNumber>

<EmailAddress>lilyc@wideworldimporters.com</EmailAddress>

</SalesPerson>

<SalesPerson>

<PersonID>15</PersonID>

<FullName>Taj Shand</FullName>

<PreferredName>Taj</PreferredName>

<LogonName>tajs@wideworldimporters.com</LogonName>

<PhoneNumber>(415) 555-0102</PhoneNumber>

<EmailAddress>tajs@wideworldimporters.com</EmailAddress>

</SalesPerson>

<SalesPerson>

<PersonID>16</PersonID>

<FullName>Archer Lamble</FullName>

<PreferredName>Archer</PreferredName>

<LogonName>archerl@wideworldimporters.com</LogonName>

<PhoneNumber>(415) 555-0102</PhoneNumber>

<EmailAddress>archerl@wideworldimporters.com</EmailAddress>

</SalesPerson>

<SalesPerson>

190

Chapter 6 Understanding JsOn

<PersonID>20</PersonID>

<FullName>Jack Potter</FullName>

<PreferredName>Jack</PreferredName>

<LogonName>jackp@wideworldimporters.com</LogonName>

<PhoneNumber>(415) 555-0102</PhoneNumber>

<EmailAddress>jackp@wideworldimporters.com</EmailAddress>

</SalesPerson>

</SalesPeople>

Each salesperson has an opening and closing tag element, containing

an opening and closing tag element, for each property related to a

salesperson. A root node, called SalesPeople, has also been added.

**Note** this XML document is element-centric. We could, of course,

also represent the salespeoples’ properties as attributes. please see

Chapter [3](#index_split_000.html#p61), for further details.

Let’s compare this XML to the JSON document in Listin[g 6-9](#index_split_001.html#p202).

***Listing 6-9.*** Sales People—JSON

{

"SalesPeople": [

{

"PersonID": 2,

"FullName": "Kayla Woodcock",

"PreferredName": "Kayla",

"LogonName": "kaylaw@wideworldimporters.com",

"PhoneNumber": "(415) 555-0102",

"EmailAddress": "kaylaw@wideworldimporters.com"

},

{

191

Chapter 6 Understanding JsOn

"PersonID": 3,

"FullName": "Hudson Onslow",

"PreferredName": "Hudson",

"LogonName": "hudsono@wideworldimporters.com",

"PhoneNumber": "(415) 555-0102",

"EmailAddress": "hudsono@wideworldimporters.com"

},

{

"PersonID": 6,

"FullName": "Sophia Hinton",

"PreferredName": "Sophia",

"LogonName": "sophiah@wideworldimporters.com",

"PhoneNumber": "(415) 555-0102",

"EmailAddress": "sophiah@wideworldimporters.com"

},

{

"PersonID": 7,

"FullName": "Amy Trefl",

"PreferredName": "Amy",

"LogonName": "amyt@wideworldimporters.com",

"PhoneNumber": "(415) 555-0102",

"EmailAddress": "amyt@wideworldimporters.com"

},

{

"PersonID": 8,

"FullName": "Anthony Grosse",

"PreferredName": "Anthony",

"LogonName": "anthonyg@wideworldimporters.com",

"PhoneNumber": "(415) 555-0102",

"EmailAddress": "anthonyg@wideworldimporters.com"

},

{

192

Chapter 6 Understanding JsOn

"PersonID": 13,

"FullName": "Hudson Hollinworth",

"PreferredName": "Hudson",

"LogonName": "hudsonh@wideworldimporters.com",

"PhoneNumber": "(415) 555-0102",

"EmailAddress": "hudsonh@wideworldimporters.com"

},

{

"PersonID": 14,

"FullName": "Lily Code",

"PreferredName": "Lily",

"LogonName": "lilyc@wideworldimporters.com",

"PhoneNumber": "(415) 555-0102",

"EmailAddress": "lilyc@wideworldimporters.com"

},

{

"PersonID": 15,

"FullName": "Taj Shand",

"PreferredName": "Taj",

"LogonName": "tajs@wideworldimporters.com",

"PhoneNumber": "(415) 555-0102",

"EmailAddress": "tajs@wideworldimporters.com"

},

{

"PersonID": 16,

"FullName": "Archer Lamble",

"PreferredName": "Archer",

"LogonName": "archerl@wideworldimporters.com",

"PhoneNumber": "(415) 555-0102",

"EmailAddress": "archerl@wideworldimporters.com"

},

193

Chapter 6 Understanding JsOn

{

"PersonID": 20,

"FullName": "Jack Potter",

"PreferredName": "Jack",

"LogonName": "jackp@wideworldimporters.com",

"PhoneNumber": "(415) 555-0102",

"EmailAddress": "jackp@wideworldimporters.com"

}

]

}

Instead of using elements, as in the XML document, the JSON

document consists simply of name/value pairs in an array of JSON objects.

A root node, called SalesPeople, has also been added.

The most obvious observation about the JSON document is that the

character count is much shorter, largely due to the lack of closing tags. The main consequence of this is that the document is easier to parse, and there is less information to be transferred between application tiers. Arguably, the document is also more human-readable. These advantages have made

JSON a very popular choice with application developers.

The main disadvantage of the JSON document is that it cannot be

bound to a schema in the way that XML can. The impact of this is that

although the document can be parsed, to ensure that it has valid syntax,

it isn’t possible (without custom code) to ensure that it meets the contract expected by the recipient before sending.

Other than the differences mentioned, despite their different

appearance, the documents are actually quite similar. They are both self-

describing, extensible documents that can be used for data-interchange.

Both formats are widely used and work with many REST APIs and web

service end points.

194

Chapter 6 Understanding JsOn

**Tip** rest is short for representational state transfer. it is an

architectural style of providing a uniform interface, often between

layers of an application. it provides a stateless approach, with client/

server separation.

**JSON Usage Scenarios**

There are many use cases for JSON data within SQL Server. The following

sections will introduce some of these potential uses.

**n-Tier Applications with Rest APIs**

Modern apps often have a lot of logic at the client side. The application tier of the application often has to have complex code, or even multiple

sublayers, to broker a conversation between the client and the back-end

RDBMS. This is because you will have to use an object relational mapper

to execute the query against the database, write these results into data

transfer object, and then serialize the results in JSON format before they can be sent to the client.

With JSON support in SQL Server, however, you can simply expose the

data from SQL Server to a REST API and return the data in JSON format,

meaning that the application tier can simply send the data as is to the

client. While there may be resistance to this approach from middle-tier

purists, it certainly allows architects to simplify the application design.

**De-Normalizing Data**

Using a normalized data model is perfect when high-frequency updates

are made to data. In a normalized model, data is separated into multiple

tables, which are joined together using primary and foreign key

195

Chapter 6 Understanding JsOn

constraints, with the intention of storing data only once. For example, if customers have multiple addresses, then core details about a customer,

such as name and phone number, may be stored in a table called

Customers. Their addresses may then be stored in a separate table called

CustomerAddresses, which contains a foreign key (CustomerID), which

joins to the primary key of the Customers table. This means that the core details about the customer are only stored once, as opposed to having to

repeat the details for every address.

**Tip** While a detailed discussion of normalization is beyond the

scope of this book, a full discussion can be found in *Expert Scripting* *and Automation for SQL Server DBAs* (apress, 2016).

A traditional normalized model can cause issues in some instances,

however. For example, performance can decrease when data is split

across multiple tables and joined together in a SELECT statement, owing

to the matching of primary and foreign key values that is required. Also, when data is updated across multiple tables, a transaction must be used,

to ensure a consistent update. This can lead to locking issues, in which

pessimistic isolation levels are in use, or IO performance issues, in which optimistic isolation levels are used.

**Tip** a full discussion of transaction isolation levels can be found in *Pro SQL Server Administration* (apress, 2015).

To work around this issue, data architects will sometimes use NoSQL

structures to store details of entities such as customers, so that logical entities can be stored as a single record. Coincidently, JSON is often

the format used for these records. This approach creates its own issues,

however, when the NoSQL data must be combined with data that is still

stored in a relational format.

196

Chapter 6 Understanding JsOn

JSON can help resolve these kinds of modeling challenges, by allowing

the JSON record to be stored in a table in SQL Server, meaning that

updates can be made to a single table, while the table can still easily be joined back to relational data.

**Config As Code**

In DevOps environments, there is a requirement to have infrastructure as

code, platforms as code, and config as code. Essentially, an entire virtual estate will be written in code, so that it is highly portable between data centers or, more commonly, between data centers and the cloud. It is also highly recoverable, in the event of a disaster, such as the loss of a data center.

SQL Server management has been slow to be incorporated into the

DevOps space, because there is a lack of crossover skills between SQL

Server and desired state configuration tooling, such as Chef and Puppet.

The DBA world is slowly starting to get on board, however, and as part of an SQL-Server-platform-as-code approach, a configuration management

database (CMDB) will often be used on a central management server, to

store the details of member servers (SQL Server VMs) within the estate.

A central management server (in this context) refers to an instance of SQL

Server that is used to help DBAs manage the rest of the SQL Server estate.

It will often be the master server in SQL Server Agent master/target job

configurations and may have other management features installed, such

as Management Data Warehouse, which is used as a central hub for SQL

Server monitoring.

When a platform-as-code approach is being used, however, a config-

as- code approach should be applied alongside. In the SQL Server world,

this involves being able to rebuild the CMDB from code, which is stored in a source control provider, such as GitHub of TFS.

197

Chapter 6 Understanding JsOn

Config as code for the SQL Server CMBD can easily be managed with a

circular process. The first step in this process is a Server Agent job, which will periodically run and export the data from the tables into JSON files in the operating system. These files will then be pushed to a source-control repository and checked in.

When a desired state configuration management tool such as Chef or

Puppet builds a new central management server, it will look, in the source control repository, to find the JSON files containing the configuration

data. It will create the config database and then repopulate the tables with the data from the repo. This provides an easily configurable RPO (recover point objective) for the config database, without the challenges that are often associated with enterprise backup management tools and the lead

times often associated with recovering data from tape robots. It also means that the core management utility for SQL Server can be easily moved to

the cloud or other data centers, helping to make the SQL Server estate

extremely portable.

**Tip** the principle of desired state management tools, such as

puppet and Chef, is that they run periodically on a server, with

a manifest that describes the desired state of the server. at the

Windows level, this may include code to disable the guest account

or ensure that user rights assignments are configured correctly.

at the sQL server level, it may check that a specific login exists or

that xp_cmdshell is disabled. First, the tool will check to see if a

resource is already configured as expected. if not, it will correct the

configuration. this means that if an unauthorized change is made to

a server, it will be corrected the next time the manifest is applied.

the result is that Windows engineers and dBas can also be sure of

the state of their servers.

198

Chapter 6 Understanding JsOn

**Analyzing the Log Data**

Devices such as sensors or RFID (radio frequency identification) can

generate very large amounts of data. This means that data architects will often choose to store this data in NoSQL solutions. Often, JSON is used as the file format for such logging.

When large logs are stored in JSON format, the SQL Server’s native

JSON support means that these logs can easily be read into SQL Server and analyzed using T-SQL, without any complex parsing requirements. This

can reduce the time to market for reports against log data.

**Summary**

JSON is a lightweight data-interchange format that is supported by many

REST APIs and web service end points. It is similar to XML, in that it is an extensible, self-describing, hierarchical document format, but it differs in the following ways:

• It cannot be validated against a schema.

• It does not have closing tags.

• It is shorter.

• It is easier to parse.

• It supports arrays.

JSON’s basic format is a series of name/value pairs, which are

separated by commas. JSON objects are always enclosed in braces, and

arrays are always enclosed in brackets. JSON objects can be nested inside other JSON objects. When this is the case, a backslash should be used to

escape characters, such as double quotes and braces.

199

Chapter 6 Understanding JsOn

JSON has many use cases in SQL Server, including the simplification

of REST APIs interacting with back-end databases. JSON is also a good

choice when NoSQL solutions must be integrated with SQL Server, such as

when device logs must be analyzed or NoSQL semi-structured data must

be stored alongside structured data. JSON is also very useful when DBAs or platform engineers implement config as code solutions.

200

**CHAPTER 7**

**Constructing**

**JSON from T-SQL**

Just as the FOR XML clause (see Chapt[er 3](#index_split_000.html#p61)) can be used to construct XML documents from relational results sets, the FOR JSON clause can be

used to construct JSON data from relational results sets. This is useful

for exchanging data between SQL Server and traditional programming

languages. Unlike FOR XML, however, FOR JSON only offers two modes: AUTO

and PATH. This chapter discusses the use of the FOR JSON clause in both

AUTO mode and PATH mode.

**FOR JSON AUTO**

FOR JSON AUTO is the simplest of the two FOR JSON modes. It can

automatically nest JSON data, based on table joins, or provide flat JSON

documents from a single table. To explain FOR JSON at the most basic level, consider the query in Listing [7-1](#index_split_002.html#p212), which returns the order dates, alongside customer, order, and salesperson keys, for customer 1060.

***Listing 7-1.*** Return Keys and Dates for Sales Orders

USE WideWorldImporters

GO

© Peter A. Carter 2018

201

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_7](#index_split_002.html#p212)

![](index-213_1.jpg)

Chapter 7 ConstruCting Json from t-sQL

SELECT

OrderID

, CustomerID

, SalespersonPersonID

, OrderDate

FROM Sales.Orders

WHERE CustomerID = 1060 ;

This query returns the results illustrated in Figure [7-1](#index_split_002.html#p213).

***Figure 7-1.** Results of keys and dates for sales orders*

If we wanted to return these results in JSON format, we could append

the FOR JSON clause, as demonstrated in Listing [7-2](#index_split_002.html#p214).

202

Chapter 7 ConstruCting Json from t-sQL

***Listing 7-2.*** Return Results As JSON

USE WideWorldImporters

GO

SELECT

OrderID

, CustomerID

, SalespersonPersonID

, OrderDate

FROM Sales.Orders

WHERE CustomerID = 1060

FOR JSON AUTO ;

This query will produce the results shown in Listin[g 7-3.](#index_split_002.html#p214)

***Listing 7-3.*** Results of Returning Data As JSON

[

{

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18"

},

{

"OrderID": 72738,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-19"

},

{

"OrderID": 72916,

"CustomerID": 1060,

203

Chapter 7 ConstruCting Json from t-sQL

"SalespersonPersonID": 6,

"OrderDate": "2016-05-20"

},

{

"OrderID": 73081,

"CustomerID": 1060,

"SalespersonPersonID": 8,

"OrderDate": "2016-05-24"

}

]

**Working with Root Nodes**

Because our results have no root node, an array wrapper (in the form of

square brackets) must be placed around the document. If we did not want

our results to be enclosed in square brackets, we could remove them by

amending our query, as shown in Listing [7-4](#index_split_002.html#p215).

***Listing 7-4.*** Removing an Array Wrapper

USE WideWorldImporters

GO

SELECT

OrderID

, CustomerID

, SalespersonPersonID

, OrderDate

FROM Sales.Orders

WHERE CustomerID = 1060

FOR JSON AUTO, WITHOUT_ARRAY_WRAPPER ;

204

Chapter 7 ConstruCting Json from t-sQL

This amended query will return the results shown in Listin[g 7-5.](#index_split_002.html#p216)

***Listing 7-5.*** Results Without an Array Wrapper

{

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18"

},

{

"OrderID": 72738,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-19"

},

{

"OrderID": 72916,

"CustomerID": 1060,

"SalespersonPersonID": 6,

"OrderDate": "2016-05-20"

},

{

"OrderID": 73081,

"CustomerID": 1060,

"SalespersonPersonID": 8,

"OrderDate": "2016-05-24"

}

Alternatively, we could resolve the issue by adding a root node to our

document. This can be achieved by using the amended query shown in

Listin[g 7-6, whic](#index_split_002.html#p217)h will cause a root node, SalesOrders, to be added to the resultant document.

205

Chapter 7 ConstruCting Json from t-sQL

**Caution** ROOT and WITHOUT_ARRAY_WRAPPER are not compatible

with each other. attempting to use both will cause the query to

generate an error.

***Listing 7-6.*** Adding a Root Node

USE WideWorldImporters

GO

SELECT

OrderID

, CustomerID

, SalespersonPersonID

, OrderDate

FROM Sales.Orders

WHERE CustomerID = 1060

FOR JSON AUTO, ROOT('SalesOrders') ;

This query produced the results illustrated in Listin[g 7-7.](#index_split_002.html#p217)

***Listing 7-7.*** Results of Adding a Root Node

{

"SalesOrders": [

{

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18"

},

{

"OrderID": 72738,

206

Chapter 7 ConstruCting Json from t-sQL

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-19"

},

{

"OrderID": 72916,

"CustomerID": 1060,

"SalespersonPersonID": 6,

"OrderDate": "2016-05-20"

},

{

"OrderID": 73081,

"CustomerID": 1060,

"SalespersonPersonID": 8,

"OrderDate": "2016-05-24"

}

]

}

**Working with NULL Values**

What if NULL values were returned by our dataset? Consider the example in Listin[g 7-8, wher](#index_split_002.html#p218)e the Comments column has been added to the SELECT list.

***Listing 7-8.*** Adding Comments to the SELECT List

USE WideWorldImporters

GO

SELECT

OrderID

, CustomerID

207

![](index-219_1.jpg)

Chapter 7 ConstruCting Json from t-sQL

, SalespersonPersonID

, OrderDate

, Comments

FROM Sales.Orders

WHERE CustomerID = 1060 ;

The results of this query are displayed in Figure [7-2](#index_split_002.html#p219).

***Figure 7-2.** Results of adding comments*

You will notice that there are no comments associated with any of the

orders returned. Let’s see how this affects the JSON, using the query in

Listin[g 7-9.](#index_split_002.html#p219)

***Listing 7-9.*** JSON with NULL Comments Added

USE WideWorldImporters

GO

SELECT

OrderID

, CustomerID

, SalespersonPersonID

, OrderDate

208

Chapter 7 ConstruCting Json from t-sQL

, Comments

FROM Sales.Orders

WHERE CustomerID = 1060

FOR JSON AUTO, ROOT('SalesOrders') ;

The results of this query can be found in Listin[g 7-10.](#index_split_002.html#p220)

***Listing 7-10.*** Results of JSON with NULL Comments

{

"SalesOrders": [

{

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18"

},

{

"OrderID": 72738,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-19"

},

{

"OrderID": 72916,

"CustomerID": 1060,

"SalespersonPersonID": 6,

"OrderDate": "2016-05-20"

},

{

"OrderID": 73081,

"CustomerID": 1060,

"SalespersonPersonID": 8,

209

Chapter 7 ConstruCting Json from t-sQL

"OrderDate": "2016-05-24"

}

]

}

You can see that the default behavior is to omit any name/value pairs

with NULL values from the results set. We can change this behavior, however, by using the INCLUDE_NULL_VALUES option, as demonstrated in Listin[g 7-11.](#index_split_002.html#p221)

***Listing 7-11.*** Including NULL Values in Results

USE WideWorldImporters

GO

SELECT

OrderID

, CustomerID

, SalespersonPersonID

, OrderDate

, Comments

FROM Sales.Orders

WHERE CustomerID = 1060

FOR JSON AUTO, ROOT('SalesOrders'), INCLUDE_NULL_VALUES ;

The results of this query can be found in Listin[g 7-12\. Y](#index_split_002.html#p221)ou will notice that, this time, the name/value pairs with NULL values have been included in the results.

***Listing 7-12.*** Results of Including NULL Values in Results

{

"SalesOrders": [

{

210

Chapter 7 ConstruCting Json from t-sQL

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18",

"Comments": null

},

{

"OrderID": 72738,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-19",

"Comments": null

},

{

"OrderID": 72916,

"CustomerID": 1060,

"SalespersonPersonID": 6,

"OrderDate": "2016-05-20",

"Comments": null

},

{

"OrderID": 73081,

"CustomerID": 1060,

"SalespersonPersonID": 8,

"OrderDate": "2016-05-24",

"Comments": null

}

]

}

211

Chapter 7 ConstruCting Json from t-sQL

**Using Column Aliases**

When considering the format of the JSON output, it is also possible to use column aliases, such as in Listin[g 7-13.](#index_split_002.html#p223)

***Listing 7-13.*** Using Column Aliases

USE WideWorldImporters

GO

SELECT

OrderID

, CustomerID

, SalespersonPersonID

, OrderDate

, Comments AS 'Orders Comments'

FROM Sales.Orders

WHERE CustomerID = 1060

FOR JSON AUTO, ROOT('SalesOrders'), INCLUDE_NULL_VALUES ;

This query returns the results shown in Listin[g 7-14](#index_split_002.html#p223).

***Listing 7-14.*** Results of Using Column Aliases

{

"SalesOrders": [

{

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18",

"Orders Comments": null

},

{

212

Chapter 7 ConstruCting Json from t-sQL

"OrderID": 72738,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-19",

"Orders Comments": null

},

{

"OrderID": 72916,

"CustomerID": 1060,

"SalespersonPersonID": 6,

"OrderDate": "2016-05-20",

"Orders Comments": null

},

{

"OrderID": 73081,

"CustomerID": 1060,

"SalespersonPersonID": 8,

"OrderDate": "2016-05-24",

"Orders Comments": null

}

]

}

You will notice that the comments key now has the name Orders

Comments, instead of Comments. This behavior is also true if you use dot-

separated aliases, such as the example in Listin[g 7-15](#index_split_002.html#p225).

213

Chapter 7 ConstruCting Json from t-sQL

***Listing 7-15.*** Using Dot-Separated Aliases

USE WideWorldImporters

GO

SELECT

OrderID

, CustomerID

, SalespersonPersonID

, OrderDate

, Comments AS 'Orders.Comments'

FROM Sales.Orders

WHERE CustomerID = 1060

FOR JSON AUTO, ROOT('SalesOrders'), INCLUDE_NULL_VALUES ;

This query returns the results illustrated in Listing [7-16.](#index_split_002.html#p225)

***Listing 7-16.*** Results of Using Dot-Separated Aliases

{

"SalesOrders": [

{

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18",

"Orders.Comments": null

},

{

"OrderID": 72738,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-19",

"Orders.Comments": null

},

214

Chapter 7 ConstruCting Json from t-sQL

{

"OrderID": 72916,

"CustomerID": 1060,

"SalespersonPersonID": 6,

"OrderDate": "2016-05-20",

"Orders.Comments": null

},

{

"OrderID": 73081,

"CustomerID": 1060,

"SalespersonPersonID": 8,

"OrderDate": "2016-05-24",

"Orders.Comments": null

}

]

}

You will notice that the key is simply given a dot-separated name.

This is important behavior to note, as when using FOR JSON PATH, dot-

separated aliases result in nested JSON objects. You will learn more about this as you progress through the chapter.

**Automatic Nesting**

So far, all our examples have focused on the behavior of FOR JSON AUTO,

when used against a single table. So now, let’s turn our attention to

how FOR JSON AUTO automatically nests data, based on table joins and

subqueries.

Consider the query in Listing [7-17\. T](#index_split_002.html#p227)his query inspects order 72646 in more detail, by enhancing it to retrieve the details of the stock items

ordered on each of the three orders placed.

215

Chapter 7 ConstruCting Json from t-sQL

***Listing 7-17.*** Added Order Line Details

USE WideWorldImporters

GO

SELECT

Orders.OrderID

, Orders.CustomerID

, Orders.SalespersonPersonID

, Orders.OrderDate

, Orders.Comments

, OrderLines.StockItemID

, OrderLines.UnitPrice

, OrderLines.Quantity

FROM Sales.Orders Orders

INNER JOIN Sales.OrderLines OrderLines

ON OrderLines.OrderID = Orders.OrderID

WHERE CustomerID = 1060

AND Orders.OrderID = 72646

FOR JSON AUTO, ROOT('SalesOrders'), INCLUDE_NULL_VALUES ;

The results of this query are illustrated in Listin[g 7-18](#index_split_002.html#p227).

***Listing 7-18.*** Results of Adding Order Line Details

{

"SalesOrders": [

{

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18",

"Comments": null,

"OrderLines": [

216

Chapter 7 ConstruCting Json from t-sQL

{

"StockItemID": 45,

"UnitPrice": 13,

"Quantity": 6

},

{

"StockItemID": 146,

"UnitPrice": 18,

"Quantity": 108

},

{

"StockItemID": 8,

"UnitPrice": 240,

"Quantity": 2

},

{

"StockItemID": 212,

"UnitPrice": 4.3,

"Quantity": 20

}

]

}

]

}

You will notice that a new key, the name of which matches the alias of

the second table, has been added to the representation of each order. The value is a nested array of JSON objects containing the name/value pairs,

which represent the columns returned from the second (based on the

order of columns within the SELECT list) table.

217

Chapter 7 ConstruCting Json from t-sQL

So, what if we added a nested join, so that we can retrieve the

product name from the Warehouse.StockItems table, as demonstrated

in Listing [7-19](#index_split_002.html#p229)?

***Listing 7-19.*** Using Nested Joins

USE WideWorldImporters

GO

SELECT

Orders.OrderID

, Orders.CustomerID

, Orders.SalespersonPersonID

, Orders.OrderDate

, Orders.Comments

, OrderLines.StockItemID

, OrderLines.UnitPrice

, OrderLines.Quantity

, Products.StockItemName

FROM Sales.Orders Orders

INNER JOIN Sales.OrderLines OrderLines

ON OrderLines.OrderID = Orders.OrderID

INNER JOIN Warehouse.StockItems Products

ON Products.StockItemID = OrderLines.StockItemID

WHERE CustomerID = 1060

AND Orders.OrderID = 72646

FOR JSON AUTO, ROOT('SalesOrders'), INCLUDE_NULL_VALUES ;

218

Chapter 7 ConstruCting Json from t-sQL

The results of this query can be found in Listin[g 7-20.](#index_split_002.html#p230)

***Listing 7-20.*** Results of Using Nested Joins

{

"SalesOrders": [

{

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18",

"Comments": null,

"OrderLines": [

{

"StockItemID": 45,

"UnitPrice": 13,

"Quantity": 6,

"Products": [

{

"StockItemName": "Developer joke mug - there

are 10 types of people in the world (Black)"

}

]

},

{

"StockItemID": 146,

"UnitPrice": 18,

"Quantity": 108,

"Products": [

{

"StockItemName": "Halloween skull mask

(Gray) S"

}

]

219

Chapter 7 ConstruCting Json from t-sQL

},

{

"StockItemID": 212,

"UnitPrice": 4.3,

"Quantity": 20,

"Products": [

{

"StockItemName": "Large replacement

blades 18mm"

}

]

},

{

"StockItemID": 8,

"UnitPrice": 240,

"Quantity": 2,

"Products": [

{

"StockItemName": "USB food flash drive -

dim sum 10 drive variety pack"

}

]

}

]

}

]

}

You will notice that an additional layer of nesting has been added to

the document, with an array, Products, containing the StockItemName

being added to each OrderLines array.

220

Chapter 7 ConstruCting Json from t-sQL

Just as nesting is automatic, based on table joins, the same applies

to subqueries. Consider the query in Listin[g 7-21](#index_split_002.html#p232), which includes two correlated subqueries in the SELECT list that return the name of the customer who placed the order and the name of the salesperson who sold it.

**Tip** nested structures can be processed more easily by programming languages.

***Listing 7-21.*** Using Subqueries to Nest Data

USE WideWorldImporters

GO

SELECT

Orders.OrderID

, (SELECT FullName FROM Application.People WHERE

PersonID = Orders.CustomerID FOR JSON AUTO) AS 'Customer'

, (SELECT FullName FROM Application.People WHERE

PersonID = Orders.SalespersonPersonID FOR JSON AUTO) AS

'SalesPerson'

, Orders.OrderDate

, Orders.Comments

, OrderLines.UnitPrice

, OrderLines.Quantity

, Products.StockItemName

FROM Sales.Orders Orders

INNER JOIN Sales.OrderLines OrderLines

ON OrderLines.OrderID = Orders.OrderID

INNER JOIN Warehouse.StockItems Products

ON Products.StockItemID = OrderLines.StockItemID

WHERE CustomerID = 1060

AND Orders.OrderID = 72646

FOR JSON AUTO, ROOT('SalesOrders'), INCLUDE_NULL_VALUES ;

221

Chapter 7 ConstruCting Json from t-sQL

The results of this query are detailed in Listin[g 7-22](#index_split_002.html#p233).

***Listing 7-22.*** Results of Nesting Data Using Subqueries

{

"SalesOrders": [

{

"OrderID": 72646,

"Customer": [

{

"FullName": "Konrads Sprogis"

}

],

"SalesPerson": [

{

"FullName": "Lily Code"

}

],

"OrderDate": "2016-05-18",

"Comments": null,

"OrderLines": [

{

"UnitPrice": 13,

"Quantity": 6,

"Products": [

{

"StockItemName": "Developer joke mug - there

are 10 types of people in the world (Black)"

}

]

},

{

222

Chapter 7 ConstruCting Json from t-sQL

"UnitPrice": 18,

"Quantity": 108,

"Products": [

{

"StockItemName": "Halloween skull mask

(Gray) S"

}

]

},

{

"UnitPrice": 4.3,

"Quantity": 20,

"Products": [

{

"StockItemName": "Large replacement

blades 18mm"

}

]

},

{

"UnitPrice": 240,

"Quantity": 2,

"Products": [

{

"StockItemName": "USB food flash drive -

dim sum 10 drive variety pack"

}

]

}

]

}

]

}

223

Chapter 7 ConstruCting Json from t-sQL

**FOR JSON PATH**

While the final example in the FOR JSON AUTO section of this chapter

returns all the required data for the sales order summary, the format of the document leaves something to be desired. The document is a lot longer

than necessary, because of unrequired nesting. For example, there can

only be one product name per line item, so the Products array should

ideally be a simple name/value pair in the OrderLines array. The same

principle applies to the customer and salesperson names.

Therefore, let’s imagine that we wanted to produce a JSON document

with the format illustrated in Listing [7-23, whic](#index_split_002.html#p235)h has a more concise, easier-to-read layout.

***Listing 7-23.*** Desired JSON Format

{

"SalesOrders": [

{

"OrderDetails": {

"OrderID": 72646,

"Customer": "Konrads Sprogis",

"SalesPerson": "Lily Code",

"OrderDate": "2016-05-18",

"Comments": null,

"LineItems": {

"UnitPrice": 13,

"Quantity": 6,

"ProductName": "Developer joke mug - there are

10 types of people in the world (Black)"

}

}

},

224

Chapter 7 ConstruCting Json from t-sQL

{

"OrderDetails": {

"OrderID": 72646,

"Customer": "Konrads Sprogis",

"SalesPerson": "Lily Code",

"OrderDate": "2016-05-18",

"Comments": null,

"LineItems": {

"UnitPrice": 18,

"Quantity": 108,

"ProductName": "Halloween skull mask (Gray) S"

}

}

},

{

"OrderDetails": {

"OrderID": 72646,

"Customer": "Konrads Sprogis",

"SalesPerson": "Lily Code",

"OrderDate": "2016-05-18",

"Comments": null,

"LineItems": {

"UnitPrice": 4.3,

"Quantity": 20,

"ProductName": "Large replacement blades 18mm"

}

}

},

{

"OrderDetails": {

"OrderID": 72646,

225

Chapter 7 ConstruCting Json from t-sQL

"Customer": "Konrads Sprogis",

"SalesPerson": "Lily Code",

"OrderDate": "2016-05-18",

"Comments": null,

"LineItems": {

"UnitPrice": 240,

"Quantity": 2,

"ProductName": "USB food flash drive - dim sum

10 drive variety pack"

}

}

}

]

}

We could easily achieve these results by using FOR JSON PATH, which

provides granular control over how the resultant document is formatted.

This control is provided through the use of column aliases, which allow

you to specify desired nesting, through dot separation. Consider the query in Listing [7-24](#index_split_002.html#p238). The query is similar to that in Listin[g 7-21](#index_split_002.html#p232), but there are two key differences to note. The first is that FOR XML AUTO has changed to FOR

XML PATH. The second is that each column has been given column aliases,

which designate not just the name of the node but its position within the document. The customer’s full name, for example, will be placed into a

node called Customer, which is nested directly under OrderDetails. The

ProductName node, in contrast, will be nested alongside the unit price

and quantity, under a node called LineItems. This causes a LineItems

key to be created with a value that consists of an array of JSON objects, including all nodes that are specified in the preceding text. The result is the document found in Listin[g 7-23.](#index_split_002.html#p235)

226

Chapter 7 ConstruCting Json from t-sQL

***Listing 7-24.*** Using FOR JSON PATH

USE WideWorldImporters

GO

SELECT

Orders.OrderID AS 'OrderDetails.OrderID'

, Customers.FullName AS 'OrderDetails.Customer'

, SalesPeople.FullName AS 'OrderDetails.SalesPerson'

, Orders.OrderDate AS 'OrderDetails.OrderDate'

, Orders.Comments AS 'OrderDetails.Comments'

, OrderLines.UnitPrice AS 'OrderDetails.LineItems.

UnitPrice'

, OrderLines.Quantity AS 'OrderDetails.LineItems.

Quantity'

, Products.StockItemName AS 'OrderDetails.LineItems.

ProductName'

FROM Sales.Orders Orders

INNER JOIN Sales.OrderLines OrderLines

ON OrderLines.OrderID = Orders.OrderID

INNER JOIN Warehouse.StockItems Products

ON Products.StockItemID = OrderLines.StockItemID

INNER JOIN Application.People Customers

ON Customers.PersonID = Orders.CustomerID

INNER JOIN Application.People SalesPeople

ON SalesPeople.PersonID = Orders.SalespersonPersonID

WHERE CustomerID = 1060

AND orders.OrderID = 72646

FOR JSON PATH, ROOT('SalesOrders'), INCLUDE_NULL_VALUES ;

227

Chapter 7 ConstruCting Json from t-sQL

**Summary**

The FOR JSON clause of a SELECT statement can be used to convert a result set into JSON format. It can be used in two modes: either AUTO or PATH.

When used in AUTO mode, FOR JSON will automatically nest data, based on

the joins within your query.

When used in PATH mode, FOR JSON gives the developer more control,

by allowing the location of each relational column to be defined within

the JSON result set, by using dot-separated column aliases, which control nesting.

The FOR JSON clause has multiple options for controlling the document

format. For example, a root node can be added to the document, by using

the ROOT option, or if no root node is present, the outer array wrapper can be removed by using the WITHOUT_ARRAY_WRAPPER option. You can also

specify if NULL values should be included in the resultant document, by

using the INCLUDE_NULL_VALUES option.

228

**CHAPTER 8**

**Shredding JSON Data**

In Chapter [7](#index_split_002.html#p212), I discussed how we can convert relational data into JSON

documents, but what if we had to shred a JSON document (just as you

learned to shred XML documents in Chapt[er 4](#index_split_001.html#p125)) into a relational dataset?

We could achieve this by using the OPENJSON() function. The OPENJSON()

function accepts a single JSON document as an input parameter and

outputs a tabular result set. The OPENJSON() function can be called either with or without specifying an explicit schema for the result set. OPENJSON() also supports the use of JSON path expressions. This chapter will examine each of these options.

**OPENJSON() with Default Schema**

In order to understand how the OPENJSON() function works with the

default schema, let’s examine the CustomFields column of the Application.

People table in the WideWorldImporters database. The query in Listin[g 8-1,](#index_split_002.html#p241)

returns the PersonID (the primary key of the table), the FullName column, and the CustomFields column, which contains a JSON document.

**Tip** Unlike the other data types discussed in this book, a JSON data type has not actually been created in SQL Server 2017. Instead, JSON

documents are stored in NVARCHAR columns, and JSON-aware functions

are called against the data, to parse and interact with the JSON.

© Peter A. Carter 2018

229

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_8](#index_split_002.html#p240)

![](index-241_1.jpg)

Chapter 8 ShreddINg JSON data

***Listing 8-1.*** Inspecting the Application.Person Table

USE WideWorldImporters

GO

SELECT

PersonID

, FullName

, CustomFields

FROM Application.People ;

You will notice, from the partial result set shown in Figure [8-1](#index_split_002.html#p241), that the CustomFields column contains a JSON document specifying each person’s

properties, such as their hire data (for staff), languages spoken, and their title.

***Figure 8-1.** Results of inspecting the Application.Person table* If we wanted to use OPENJSON() to shred the details of a specific user, we would first have to pass the JSON document into a variable, before passing the variable into the OPENJSON() function. This technique is demonstrated in Listing [8-2, whic](#index_split_002.html#p241)h returns the custom fields for Anthony Grosse.

***Listing 8-2.*** Shredding a Single JSON Document

USE WideWorldImporters

GO

DECLARE @CustomFields NVARCHAR(MAX) ;

230

![](index-242_1.jpg)

Chapter 8 ShreddINg JSON data

SET @CustomFields = (

SELECT CustomFields

FROM Application.People

WHERE FullName = 'Anthony Grosse'

) ;

SELECT *

FROM OPENJSON(@CustomFields) ;

The results of this script are illustrated in Figur[e 8-2.](#index_split_002.html#p242)

***Figure 8-2.** Results of shredding a single JSON document*

The key column contains the name of the name/value pair; the value

column contains the value of the name/value pair; and the type column

indicates the data type. Table [8-1](#index_split_002.html#p243) details the data types that can be returned.

231

Chapter 8 ShreddINg JSON data

***Table 8-1.** Data Types*

**Data Type ID**

**Data Type**

1

String

2

Number

3

Boolean

4

Array

5

Object

**Shredding a Column**

But what if we wanted to shred an entire column? The OPENJSON()

function only accepts a single JSON object, so we could not pass in values from multiple rows. Instead, we would have to use the OUTER APPLY

operator against the table.

The OUTER APPLY operator applies a function to every row in a result

set. If the function returns a NULL value, the row will be included in the result set. This contrasts with the CROSS APPLY operator, which also applies a function to every row in a result set but omits the row, if the applied function returns NULL.

The query in Listin[g 8-3](#index_split_002.html#p243) demonstrates how to use the OUTER APPLY

operator against the Application.People table, to shred the CustomFields

document.

***Listing 8-3.*** Using OUTER APPLY with OPENJSON()

USE WideWorldImporters

GO

SELECT PersonID, FullName, CustomFields, JSON.*

FROM Application.People

OUTER APPLY OPENJSON(CustomFields) JSON ;

232

![](index-244_1.jpg)

Chapter 8 ShreddINg JSON data

Partial results from this query are shown in Figur[e 8-3.](#index_split_002.html#p244)

***Figure 8-3.** Results of using OUTER APPLY with OPENJSON()*

You will notice that the results from the Application.Person table are

duplicated for each row returned from the OPENJSON() function. This is

known as a Cartesian product.

**Tip** If we had used CROSS APPLY instead of OUTER APPLY, the

results for PersonID 1 would have been omitted.

To turn this data into columns, to avoid rows being duplicated, you

could use the PIVOT operator. The PIVOT operator works by rotating

unique values from a column into separate columns. This could also be

described as changing rows to columns. It will then perform aggregations

on remaining columns, as required. The same could be achieved by using

multiple CASE statements, but the PIVOT operator is far more efficient.

The syntax of the PIVOT operator has an outer query, followed by two

subqueries. The first subquery contains the base query, while the second

contains the pivot specification. Because our values are often textual, and aggregation isn’t appropriate, we will use the MAX() aggregate function.

This is demonstrated with the query in Listing [8-4](#index_split_002.html#p245).

233

Chapter 8 ShreddINg JSON data

***Listing 8-4.*** Using PIVOT with OPENJSON()

USE WideWorldImporters

GO

SELECT

PersonID

, FullName

, [OtherLanguages]

, [HireDate]

, [Title]

, [PrimarySalesTerritory]

, [CommissionRate]

FROM (

SELECT

PersonID

, FullName

, JSON.[Key] AS JSONName

, JSON.value AS JSONValue

FROM Application.People

OUTER APPLY OPENJSON(CustomFields) JSON

) Src

PIVOT

(

MAX(JSONValue)

FOR JSONName IN ([OtherLanguages], [HireDate], [Title],

[PrimarySalesTerritory], [CommissionRate])

) pvt ;

234

![](index-246_1.jpg)

Chapter 8 ShreddINg JSON data

The partial results of this query can be seen in Figur[e 8-4.](#index_split_002.html#p246)

***Figure 8-4.** Results of using* PIVOT *with* OPENJSON() The limitation of using this approach is that you must know the name

of each key in the JSON document before writing the query. If any key

names are missed, or added later, the data will not appear in the result set.

This can be particularly challenging, as JSON documents cannot be bound

to a schema.

**Dynamic Shredding Based on Document Content**

The way to resolve the issue of not knowing the document contents at

development time is to use a dynamic PIVOT. This involves using dynamic

SQL to define the current list of JSON keys to pivot before the query is run.

This technique is demonstrated in Listin[g 8-5.](#index_split_002.html#p247)

**Tip** QUOTENAME() is a system function that delimits a value by

wrapping it in square brackets.

235

Chapter 8 ShreddINg JSON data

***Listing 8-5.*** Using Dynamic PIVOT with OPENJSON()

DECLARE @Columns NVARCHAR(MAX) ;

DECLARE @SQL NVARCHAR(MAX) ;

SET @Columns = ";

SELECT @Columns += ', p.' + QUOTENAME(JSONName)

FROM (

SELECT DISTINCT

JSON.[Key] AS JSONName

FROM Application.People p

CROSS APPLY OPENJSON(CustomFields) JSON

) AS cols ;

SET @SQL =

'SELECT

PersonID

, FullName

, ' + STUFF(@Columns, 1, 2, ") + '

FROM

(

SELECT

PersonID

, FullName

, JSON.[Key] AS JSONName

, JSON.value AS JSONValue

FROM Application.People

OUTER APPLY OPENJSON(CustomFields) JSON

) AS src

PIVOT

236

Chapter 8 ShreddINg JSON data

(

MAX(JSONValue) FOR JSONName IN ('

+ STUFF(REPLACE(@Columns, ', p.[', ',['), 1, 1, ")

+ ')

) AS p ;' ;

EXEC (@SQL) ;

**OPENJSON( ) with Explicit Schema**

When using OPENJSON() with an explicit schema, you are able to provide

control over the format of the result set that is returned. Instead of a three-column result set, a column will be returned for every column that you

have specified in the WITH clause. You can also specify the data type of each column. These data types are T-SQL data types, not JSON data types, so

types such as DATE or DECIMAL can be specified. For example, consider the script in Listing [8-6](#index_split_002.html#p248).

***Listing 8-6.*** Using OPENJSON() with an Explicit Schema

DECLARE @CustomFields NVARCHAR(MAX) ;

SET @CustomFields =

(

SELECT

CustomFields

FROM Application.People

WHERE PersonID = 2

) ;

SELECT *

FROM OPENJSON(@CustomFields)

WITH (

237

![](index-249_1.jpg)

Chapter 8 ShreddINg JSON data

HireDate DATETIME2

, Title NVARCHAR(50)

, PrimarySalesTerritory NVARCHAR(50)

, CommissionRate DECIMAL(5,2)

) ;

This query returns the results shown in Figur[e 8-5.](#index_split_002.html#p249)

***Figure 8-5.** Results of using OPENJSON() with an explicit schema* A slight complexity arises when one of the columns returned is a JSON

object. For example, consider the query in Listing [8-7, whic](#index_split_002.html#p249)h adds the OtherLanguages column to the query. As there is no specific JSON data type, we will use NVARCHAR(MAX), as it can be stored as NVARCHAR(MAX) in a table.

***Listing 8-7.*** Adding a JSON Column

DECLARE @CustomFields NVARCHAR(MAX) ;

SET @CustomFields =

(

SELECT

CustomFields

FROM Application.People

WHERE PersonID = 2

) ;

238

![](index-250_1.jpg)

Chapter 8 ShreddINg JSON data

SELECT *

FROM OPENJSON(@CustomFields)

WITH (

OtherLanguages NVARCHAR(MAX)

, HireDate DATETIME2

, Title NVARCHAR(50)

, PrimarySalesTerritory NVARCHAR(50)

, CommissionRate DECIMAL(5,2)

) ;

This query returns the results shown in Figur[e 8-6.](#index_split_002.html#p250)

***Figure 8-6.** Results of adding a JSON column*

So why has the OtherLanguages column returned NULL? We know that

the column exists, and that it contains data for PersonID 2, owing to the previous examples in this chapter. When returning a JSON object from

OPENJSON(), we must use additional syntax in the WITH clause, to specify

that the NVARCHAR actually represents a JSON object, as demonstrated in

Listin[g 8-8.](#index_split_002.html#p250)

***Listing 8-8.*** Correctly Returning a JSON Array or Object

DECLARE @CustomFields NVARCHAR(MAX) ;

SET @CustomFields =

(

239

![](index-251_1.jpg)

Chapter 8 ShreddINg JSON data

SELECT

CustomFields

FROM Application.People

WHERE PersonID = 2

) ;

SELECT *

FROM OPENJSON(@CustomFields)

WITH (

OtherLanguages NVARCHAR(MAX) AS JSON

, HireDate DATETIME2

, Title NVARCHAR(50)

, PrimarySalesTerritory NVARCHAR(50)

, CommissionRate DECIMAL(5,2)

) ;

The script will now return the results that we expect, as shown in Figur[e 8-7.](#index_split_002.html#p251)

***Figure 8-7.** Correctly returning JSON data*

When you must shred multiple rows, an explicit schema can also be

specified, when using the OUTER APPLY operator. Remember that the OUTER

APPLY operator will not remove rows that return NULL values, in the way

that CROSS APPLY does. This is demonstrated in Listin[g 8-9.](#index_split_002.html#p252)

240

Chapter 8 ShreddINg JSON data

***Listing 8-9.*** Using an Explicit Schema with OUTER APPLY

SELECT

PersonID

, FullName

, JSON.*

FROM Application.People

OUTER APPLY OPENJSON(CustomFields)

WITH (

OtherLanguages

NVARCHAR(MAX) AS JSON

, HireDate DATETIME2

, Title NVARCHAR(50)

, PrimarySalesTerritory

NVARCHAR(50)

, CommissionRate

DECIMAL(5,2)

) JSON ;

As you can see from the partial results in Figure [8-8](#index_split_002.html#p253), some of the need-to- pivot data has been eliminated. The issue remains, however, that you

must know every possible key in the JSON document before the query is

written. Therefore, if there is not a discrete set of possible values, you may still be required to use dynamic SQL.

241

![](index-253_1.jpg)

Chapter 8 ShreddINg JSON data

***Figure 8-8.** Results of using an explicit schema with OUTER APPLY*

**OPENJSON( ) with Path Expressions**

As well as the use of explicit schema, OPENJSON() also supports JSON path expressions. A path expression allows you to reference specific properties within a JSON document. For example, consider the JSON document in

Listin[g 8-10.](#index_split_002.html#p253)

**Tip** You may recognize this document, as we created it in Chapter [7.](#index_split_002.html#p212)

***Listing 8-10.*** Sales Orders with Root Node

{

"SalesOrders": [

{

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18"

},

242

![](index-254_1.jpg)

Chapter 8 ShreddINg JSON data

{

"OrderID": 72738,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-19"

},

{

"OrderID": 72916,

"CustomerID": 1060,

"SalespersonPersonID": 6,

"OrderDate": "2016-05-20"

},

{

"OrderID": 73081,

"CustomerID": 1060,

"SalespersonPersonID": 8,

"OrderDate": "2016-05-24"

}

]

}

If we used a basic OPENJSON() statement against this document, it would

return the entire SalesOrders array, as partially shown in Figure [8- 9](#index_split_002.html#p254).

***Figure 8-9.** Results of basic OPENJSON()*

243

![](index-255_1.jpg)

Chapter 8 ShreddINg JSON data

If we were to use a PATH statement, however, we could choose to only

return the *n* th item in this array. This would drastically alter the results set, as OPENJSON() would be able to map each item within the array element

to a relational column, meaning that a row for each key within the element would be returned, instead of a single row containing a JSON document,

as shown in Figur[e 8-10](#index_split_002.html#p255), which contains the results of shredding the first array element (OrderID 72646).

***Figure 8-10.** Results of shredding a single array element*

So, let’s look at how we can get to this result. First, we must understand that path expressions can be run in one of two modes: strict or lax. If you run a path expression in lax mode, and the path expression contains an

error, OPENJSON() will “eat the error” and return an empty result set. If you use strict mode, however, if the path expression contains an error,

OPENJSON() will throw an error message.

We now must understand the elements of the path itself. First, we use a $

to specify the context, followed by dot-separated, nested key names. Finally, we specify the array element number in square brackets. So, to produce the results in Figur[e 8-10, w](#index_split_002.html#p255)e would use the query in Listin[g 8-11](#index_split_002.html#p256).

244

Chapter 8 ShreddINg JSON data

***Listing 8-11.*** Using Path Expressions to Return a Single Array Element DECLARE @JSON NVARCHAR(MAX) ;

SET @JSON = '{

"SalesOrders": [

{

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18"

},

{

"OrderID": 72738,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-19"

},

{

"OrderID": 72916,

"CustomerID": 1060,

"SalespersonPersonID": 6,

"OrderDate": "2016-05-20"

},

{

"OrderID": 73081,

"CustomerID": 1060,

"SalespersonPersonID": 8,

"OrderDate": "2016-05-24"

}

]

}

' ;

245

Chapter 8 ShreddINg JSON data

SELECT *

FROM OPENJSON(@JSON, 'lax $.SalesOrders[0]') ;

You will notice, in this script, that after passing in the JSON document, we use the lax (or, alternatively, strict) keyword to specify the mode we will use. After a space comes the path expression itself. Here, we start with $, to set the context, and then point to the SalesOrders key. We then use square brackets to specify the array element that we wish to use.

**Tip** JSON path expressions always use zero-base arrays.

**Shredding Data into Tables**

You can now imagine how simple looping techniques could be used to

shred each element within an array. For example, consider the script

in Listing [8-12](#index_split_002.html#p257). This script will shred each of the array elements into a temporary table called Orders.

***Listing 8-12.*** Shredding Each Element into a Temporary Table DECLARE @JSON NVARCHAR(MAX) ;

SET @JSON = '{

"SalesOrders": [

{

"OrderID": 72646,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-18"

},

246

Chapter 8 ShreddINg JSON data

{

"OrderID": 72738,

"CustomerID": 1060,

"SalespersonPersonID": 14,

"OrderDate": "2016-05-19"

},

{

"OrderID": 72916,

"CustomerID": 1060,

"SalespersonPersonID": 6,

"OrderDate": "2016-05-20"

},

{

"OrderID": 73081,

"CustomerID": 1060,

"SalespersonPersonID": 8,

"OrderDate": "2016-05-24"

}

]

}

' ;

CREATE TABLE #Orders

(

OrderID INT,

CustomerID INT,

SalespersonPersonID INT,

OrderDate DATE

) ;

247

Chapter 8 ShreddINg JSON data

DECLARE @ArrayElement INT = 0 ;

DECLARE @path NVARCHAR(MAX) = 'lax $.SalesOrders[' + CAST(

@ArrayElement AS NVARCHAR) + ']' ;

WHILE @ArrayElement <=3

BEGIN

INSERT INTO #Orders (OrderID, CustomerID,

SalespersonPersonID, OrderDate)

SELECT

OrderID

, CustomerID

, SalespersonPersonID

, OrderDate

FROM OPENJSON(@JSON, @Path)

WITH( OrderID INT, CustomerID INT, SalespersonPersonID

INT, OrderDate DATE) ;

SET @ArrayElement = @ArrayElement + 1 ;

SET @path = 'lax $.SalesOrders[' + CAST(@ArrayElement

AS NVARCHAR) + ']' ;

END

SELECT * FROM #Orders ;

DROP TABLE #Orders ;

248

![](index-260_1.jpg)

Chapter 8 ShreddINg JSON data

***Figure 8-11.** Results of shredding multiple array elements*

The final SELECT statement in this script produces the results

illustrated in Figur[e 8-11](#index_split_002.html#p260).

**Caution** While I have used a WHILE loop in this example, I have

done so only because it provides a clear and easy example of how

path expressions can be used. I would never use a WHILE loop or

CURSOR in production code. there is always a way to achieve the

same results, using a set-based approach.

**Summary**

JSON data can be shredded into tabular results sets by using the OPENJSON() function. OPENJSON() can be used either with or without an explicit

249

Chapter 8 ShreddINg JSON data

schema. When a schema is not explicitly defined, OPENJSON(), using a WITH

clause, returns a standard row set, detailing the key (name), value, and

JSON data type ID of each node in the document.

When an explicit schema is supplied, OPENJSON() will return a

formatted result set, which contains a column for each specified in the

WITH clause. Using an explicit schema avoids the need to pivot the data

when you know every node in the document at development time. If the

list of columns is not discrete, however, dynamic SQL will be required, to build a list of possible results before processing.

OPENJSON() also supports JSON path expressions. Passing path

expressions to the function allows you to navigate to a specific item within an array, meaning that you can shred data to a more granular level. For

example, instead of shredding an array of JSON objects into a table, you

can use looping methodologies to shred the contents of each array element into relational data.

250

**CHAPTER 9**

**Working with the**

**JSON Data Type**

In this chapter, I will discuss the T-SQL functions that allow developers to query JSON data. I will then discuss how JSON data can be indexed.

**Querying JSON Data**

SQL Server has introduced the JSON_VALUE(), JSON_QUERY(), JSON_MODIFY(),

and ISJSON() functions to help developers interrogate and interact with

JSON data. The following sections will discuss each of these functions.

**Using ISJSON( )**

Because JSON data is stored in NVARCHAR(MAX) columns, as opposed to

using its own data type, it is very useful to ensure that a tuple contains a valid JSON document, before calling a JSON function against it. The

ISJSON() function will evaluate a string to check if it is a valid JSON

document. The function will return a value of 1 if the string is valid JSON

and 0 if it is not. Therefore, a common usage of the function is within an IF

statement. For example, consider the query in Listing [9-1](#index_split_002.html#p263).

© Peter A. Carter 2018

251

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_9](#index_split_002.html#p262)

![](index-263_1.jpg)

Chapter 9 Working With the JSon Data type

***Listing 9-1.*** Incorrectly Formatted JSON

DECLARE @JSON NVARCHAR(MAX) ;

SET @JSON = '{"I am not:"Correctly formatted"}' ;

SELECT *

FROM OPENJSON(@JSON) ;

Because the name of the key is missing a closing double quotation

mark, the query will fail, with the error shown in Figure [9-1.](#index_split_002.html#p263)

***Figure 9-1.** Error thrown by invalid JSON*

Instead of our script failing, we could instead use the ISJSON()

function in an IF statement, as demonstrated in Listing [9-2](#index_split_002.html#p263).

***Listing 9-2.*** Use ISJSON() in an IF Statement

DECLARE @JSON NVARCHAR(MAX) ;

SET @JSON = '{"I am not:"Correctly formatted"}' ;

IF ISJSON(@JSON) = 1

252

Chapter 9 Working With the JSon Data type

BEGIN

SELECT *

FROM OPENJSON(@JSON) ;

END

This time, the script will complete without errors, as the query against

the OPENJSON() function will never run.

**Tip** a full description of the usage of OPENJSON() can be found in Chapter [8](#index_split_002.html#p240).

The ISJSON() function could also be used in the WHERE clause of a

SELECT statement. For example, consider the script in Listin[g 9-3\. T](#index_split_002.html#p264)he script creates a simple temporary table and inserts two values into an

NVARCHAR(MAX) column. One of the values is valid JSON data, and the other is not. The script then calls the OPENJSON() function against the column.

***Listing 9-3.*** Filter Results That Are Not JSON

--Create a temp table

CREATE TABLE #JsonTemp

(

JSONData NVARCHAR(MAX)

) ;

--Populate temp table with one JSON value and one non-JSON value

INSERT INTO #JsonTemp

VALUES ('{"I am JSON":"True"}'),

('I am JSON - False') ;

--Call OPENJSON() against only rows where data is JSON

253

![](index-265_1.jpg)

Chapter 9 Working With the JSon Data type

SELECT JSON.*

FROM #JsonTemp Base

OUTER APPLY OPENJSON(Base.JSONData) JSON

WHERE ISJSON(Base.JSONData) = 1 ;

--Drop temp table

DROP TABLE #JsonTemp ;

Because the WHERE clause removes any rows that do not contain valid

JSON data, before the OPENJSON() function is applied, the script completes successfully and returns the results shown in Figur[e 9-2.](#index_split_002.html#p265)

***Figure 9-2.** Results of filtering non-JSON data*

**Using JSON_VALUE( )**

The JSON_VALUE() function can be used to return a single scalar value

from a JSON document. The function accepts two parameters. The first

parameter is the JSON document, from which to retrieve the data. The

second is a path expression to the value you wish to return. As described in Chapt[er 8, when us](#index_split_002.html#p240)ing path expressions with OPENJSON(), path expressions can be used in either lax mode or strict mode. When lax mode is used, if

254

Chapter 9 Working With the JSon Data type

there is an error in the path expression, NULL results will be returned, and no error will be raised. When used in strict mode, if there is an error in the path expression, an error will be thrown.

The value returned is always of data type NVARCHAR(4000). This means

that if the value exceeds 4000 characters, JSON_VALUE() will either return NULL or throw an error, depending on whether lax mode or strict mode has

been used.

To look more closely at the JSON_VALUE() function, let’s consider

the Warehouse.StockItems table in the WideWorldImporters database.

The CustomFields column of this table contains a JSON document that

includes a key called Tags, which has a value of an array, containing

product tags.

The script in Listin[g 9-4](#index_split_002.html#p267) will first populate a variable with the content of CustomFields for a single product. Subsequently, it will check that the variable contains a valid JSON document, by using the ISJSON() function,

before passing the document into the JSON_VALUE() function.

The path expression of the JSON_VALUE() function starts by

specifying that we wish to use the path expression in lax mode. It then

uses a $ to represent the context, before using a dot-separated path to

the node we wish to extract. Because the Tags node is an array, and the

JSON_VALUE() function can only return a scalar value, we will use square

brackets to denote the element within the array that we would like to

extract. This is mandatory syntax, even if there is only a single element in the array.

**Tip** the array is always zero-based.

255

![](index-267_1.jpg)

Chapter 9 Working With the JSon Data type

***Listing 9-4.*** UsingJSON_VALUE() Against a JSON Document

DECLARE @JSON NVARCHAR(MAX) ;

--The CustomFields column for StockItem ID 61 contains the

following JSON document:

--'{ "CountryOfManufacture": "China", "Tags": ["Radio Control","Realistic Sound"], "MinimumAge": "10" }'

SET @JSON = (SELECT CustomFields FROM Warehouse.StockItems

WHERE StockItemID = 61) ;

IF ISJSON(@JSON) = 1

BEGIN

SELECT JSON_VALUE(@Json,'lax $.Tags[0]') ;

END

This script produces the results illustrated in Figur[e 9-3.](#index_split_002.html#p267)

***Figure 9-3.** Results of using JSON_VALUE() against a JSON document* So, what if we want to use the JSON_VALUE() function against a column

in a table? Where it is a scalar function, we cannot use OUTER APPLY or

CROSS APPLY. Instead, we must include it in the SELECT list of our query.

This is demonstrated in Listing [9-5](#index_split_002.html#p268).

256

![](index-268_1.jpg)

Chapter 9 Working With the JSon Data type

***Listing 9-5.*** Using JSON_VALUE() in a SELECT List

USE WideWorldImporters

GO

SELECT

StockItemName

, JSON_VALUE(customfields,'lax $.Tags[0]')

FROM Warehouse.StockItems ;

You will notice that instead of passing in a variable, we simply pass in

the name of the column that contains the JSON document. Partial results

of this query can be found in Figure [9-4](#index_split_002.html#p268).

***Figure 9-4.** Results of using JSON_VALUE() against a table*

257

![](index-269_1.jpg)

Chapter 9 Working With the JSon Data type

Our original JSON document referred to Stock Item ID 61, which is a

remote-controlled car. We could also use JSON_VALUE in the WHERE clause

of our query, to filter the result set, so that only rows in which the first tag contains the value Radio Control are returned. This is demonstrated in

Listin[g 9-6, in whic](#index_split_002.html#p269)h we have also enhanced the query, to ensure that only valid JSON documents are returned, by using the ISJSON() function.

***Listing 9-6.*** Using JSON_VALUE() in a WHERE Clause

USE WideWorldImporters

GO

SELECT

StockItemName

, JSON_VALUE(CustomFields,'lax $.Tags[0]') AS Tag0

FROM Warehouse.StockItems

WHERE JSON_VALUE(CustomFields,'lax $.Tags[0]') = 'Radio Control'

AND ISJSON(CustomFields) = 1 ;

The results of this query are illustrated in Figur[e 9-5.](#index_split_002.html#p269)

***Figure 9-5.** Results of using JSON_VALUE() in a WHERE clause*

258

Chapter 9 Working With the JSon Data type

The third tag of remote-controlled car products denotes if the item

is vintage. There are two vintage cars in the product table. Therefore, in Listin[g 9-7, w](#index_split_002.html#p270)e will further filter the result set to include only products for which the third tag has a value of Vintage. We will also enhance the SELECT

list, to contain the first three tags in the array. Finally, we will change the JSON_VALUE() functions in the WHERE clause, to use strict path expressions, so that an error will result in the query failing.

***Listing 9-7.*** Enhancing the Query

USE WideWorldImporters

GO

SELECT

StockItemName

, JSON_VALUE(CustomFields,'lax $.Tags[0]') AS Tag0

, JSON_VALUE(CustomFields,'lax $.Tags[1]') AS Tag1

, JSON_VALUE(CustomFields,'lax $.Tags[2]') AS Tag2

FROM Warehouse.StockItems

WHERE JSON_VALUE(CustomFields,'strict $.Tags[0]') = 'Radio Control'

AND JSON_VALUE(CustomFields,'strict $.Tags[2]') = 'Vintage'

AND ISJSON(CustomFields) = 1 ;

Unfortunately, this time, even though the ISJSON() function is

ensuring that any non-valid JSON documents are not in the result set, the query returns the error shown in Figur[e 9-6\. T](#index_split_002.html#p271)his is because not all JSON

documents in the CustomFields column have a Tags key. From those that

do, not all documents have three tags. Therefore, the path expressions for two JSON_VALUE() calls in the WHERE clause are not valid. Because we have changed from lax mode to strict mode, the query fails.

259

![](index-271_1.jpg)

![](index-271_2.jpg)

Chapter 9 Working With the JSon Data type

***Figure 9-6.** Error thrown by the query*

If we were to change back to lax mode path expressions, the query

would return the results displayed in Figure [9-7](#index_split_002.html#p271).

***Figure 9-7.** Query results with lax mode path expressions*

With SQL Server 2017 and later versions, it is also possible to pass in a path expression as a variable. Therefore, the query in Listing [9-8](#index_split_002.html#p272) will return the same results as shown in Figure [9-7.](#index_split_002.html#p271)

260

Chapter 9 Working With the JSon Data type

**Tip** you must use SQL Server 2017 or later versions, to run the

quer[y in Listing 9-8.](#index_split_002.html#p272)

***Listing 9-8.*** Using a Variable As a Path

USE WideWorldImporters

GO

DECLARE @Path NVARCHAR(MAX) = 'lax $.Tags[2]' ;

SELECT

StockItemName

, JSON_VALUE(CustomFields,'lax $.Tags[0]') AS Tag0

, JSON_VALUE(CustomFields,'lax $.Tags[1]') AS Tag1

, JSON_VALUE(CustomFields,@Path) AS Tag2

FROM Warehouse.StockItems

WHERE JSON_VALUE(CustomFields,'lax $.Tags[0]') = 'Radio Control'

AND JSON_VALUE(CustomFields,'lax $.Tags[2]') = 'Vintage'

AND ISJSON(CustomFields) = 1 ;

**Using JSON_QUERY( )**

Unlike JSON_VALUE(), which returns a scalar value, JSON_QUERY() can

be used to extract a JSON object, or an array, from a JSON document.

For example, consider the script in Listing [9-9](#index_split_002.html#p273). The script uses the same JSON document that we used in Listin[g 9-4, whic](#index_split_002.html#p267)h extracted a single array element from the Tags array for Stock Item ID 61\. This time, however,

instead of extracting a single array element, we will extract the entire Tags array. Because we are extracting the entire array, there is no need to specify the array element number in square brackets, as we did when using JSON_

VALUE() against the document.

261

![](index-273_1.jpg)

Chapter 9 Working With the JSon Data type

***Listing 9-9.*** Using JSON_QUERY() Against a JSON Document

DECLARE @JSON NVARCHAR(MAX) ;

--The CustomFields column for StockItem ID 61 contains the

following JSON document:

--'{ "CountryOfManufacture": "China", "Tags": ["Radio Control","Realistic Sound"], "MinimumAge": "10" }'

SET @JSON = (SELECT CustomFields FROM Warehouse.StockItems

WHERE StockItemID = 61) ;

IF ISJSON(@JSON) = 1

BEGIN

SELECT JSON_QUERY(@Json,'lax $.Tags') ;

END

The results of this script are illustrated in Figur[e 9-8.](#index_split_002.html#p273)

***Figure 9-8.** Results of using JSON_QUERY() against a JSON document* 262

![](index-274_1.jpg)

Chapter 9 Working With the JSon Data type

As with JSON_VALUE(), if we want to use JSON_QUERY() against a

column in a table, we will use it in the SELECT list, as opposed to using a CROSS APPLY or OUTER APPLY operator. The difference between JSON_

VALUE() and JSON_QUERY() is demonstrated in Listing [9-10\. H](#index_split_002.html#p274)ere, we use the same query as in Listin[g 9-7 b](#index_split_002.html#p270)ut enhance it to include a column in the result set that includes the whole Tags array, using JSON_QUERY().

***Listing 9-10.*** Using JSON_QUERY() in a SELECT List

USE WideWorldImporters

GO

SELECT

StockItemName

, JSON_VALUE(CustomFields,'lax $.Tags[0]') AS Tag0

, JSON_VALUE(CustomFields,'lax $.Tags[1]') AS Tag1

, JSON_VALUE(CustomFields,'lax $.Tags[2]') AS Tag2

, JSON_QUERY(CustomFields,'lax $.Tags') AS TagsArray

FROM Warehouse.StockItems

WHERE JSON_VALUE(CustomFields,'lax $.Tags[0]') = 'Radio Control'

AND JSON_VALUE(CustomFields,'lax $.Tags[2]') = 'Vintage'

AND ISJSON(CustomFields) = 1 ;

This query returns the results shown in Figur[e 9-9.](#index_split_002.html#p274)

***Figure 9-9.** Results of using JSON_QUERY() in a SELECT list*

263

Chapter 9 Working With the JSon Data type

The JSON_QUERY() function can also be used in a WHERE clause.

Consider the query in Listing [9-11, whic](#index_split_002.html#p275)h has been rewritten, so that the JSON_QUERY() function is used to filter out any rows in which the JSON

document does contain an empty Tags array. You will notice that the query uses a mix of lax mode and strict mode.

***Listing 9-11.*** Using JSON_QUERY() in a WHERE Clause

USE WideWorldImporters

GO

SELECT

StockItemName

, JSON_VALUE(CustomFields,'lax $.Tags[0]') AS Tag0

, JSON_VALUE(CustomFields,'lax $.Tags[1]') AS Tag1

, JSON_VALUE(CustomFields,'lax $.Tags[2]') AS Tag2

, JSON_QUERY(CustomFields,'lax $.Tags') AS TagsArray

FROM Warehouse.StockItems

WHERE JSON_QUERY(CustomFields,'strict $.Tags') <> '[]'

AND ISJSON(CustomFields) = 1 ;

If only the document context ($) is passed to the path expression,

the entire JSON document will be returned. It is also worth noting that

a variable can be used to pass the path, from SQL Server 2017 onward,

just as it can for OPENJSON() and JSON_VALUE(). Both these concepts

are demonstrated in Listin[g 9-12](#index_split_002.html#p276), which uses a variable to pass only the document context, as the path expression, to an additional column in the

result set.

**Tip** you must be running SQL Server 2017 or later versions to run

the query in Listing [9-12](#index_split_002.html#p276).

264

![](index-276_1.jpg)

Chapter 9 Working With the JSon Data type

***Listing 9-12.*** Using Path Variables and Document Context

USE WideWorldImporters

GO

DECLARE @Path NVARCHAR(MAX) = '$' ;

SELECT

StockItemName

, JSON_VALUE(CustomFields,'lax $.Tags[0]') AS Tag0

, JSON_VALUE(CustomFields,'lax $.Tags[1]') AS Tag1

, JSON_VALUE(CustomFields,'lax $.Tags[2]') AS Tag2

, JSON_QUERY(CustomFields,'lax $.Tags') AS TagsArray

, JSON_QUERY(CustomFields, @Path) AS EntireDocument

FROM Warehouse.StockItems

WHERE JSON_QUERY(CustomFields,'strict $.Tags') <> '[]'

AND ISJSON(CustomFields) = 1 ;

The partial results of this query can be seen in Figur[e 9-10.](#index_split_002.html#p276)

***Figure 9-10.** Results of using path variables and document context* 265

Chapter 9 Working With the JSon Data type

**Using JSON_MODIFY()**

So far, all the JSON functions that we have examined have allowed us to

interrogate JSON documents. The JSON_MODIFY() function, however, as its

name suggests, allows us to modify the contents of the JSON document. To

explain this further, let’s once again use the CustomFields JSON document for Stock Item ID 61, as used in Listin[g 9-4 and L](#index_split_002.html#p267)isting [9-9](#index_split_002.html#p273).

The script in Listin[g 9-13](#index_split_002.html#p277) will modify the second element of the Tags array, so that the second element is updated to read 'Very Realistic

Sound'. The output of the function is the complete, modified document.

***Listing 9-13.*** Updating a Value

DECLARE @JSON NVARCHAR(MAX) ;

--The CustomFields column for StockItem ID 61 contains the

following JSON document:

--'{ "CountryOfManufacture": "China", "Tags": ["Radio Control","Realistic Sound"], "MinimumAge": "10" }'

SET @JSON = (SELECT CustomFields FROM Warehouse.StockItems

WHERE StockItemID = 61) ;

IF ISJSON(@JSON) = 1

BEGIN

SELECT JSON_MODIFY(@Json,'lax $.Tags[1]', 'Very

Realistic Sound') ;

END

This script returns the results in Figure [9-11.](#index_split_002.html#p278)

266

![](index-278_1.jpg)

Chapter 9 Working With the JSon Data type

***Figure 9-11.** Results of updating a value*

You can see how the output from this function could be used

subsequently to update a row in a table containing a JSON document, as

demonstrated in Listing [9-14. H](#index_split_002.html#p278)ere, instead of passing in a variable as the JSON document, we pass in the CustomFields column from the table.

***Listing 9-14.*** Using MODIFY_JSON() to Update a Row in a Table USE WideWorldImporters

GO

UPDATE StockItems

SET CustomFields = JSON_MODIFY(CustomFields,'lax

$.Tags[1]', 'Very Realistic Sound')

FROM Warehouse.StockItems

WHERE StockItemID = 61 ;

The MODIFY_JSON() function can also be used to add an element to an

array. Consider the query in Listin[g 9-15](#index_split_002.html#p279), which marks Stock Item ID 61

as being vintage, by adding an additional element to the Tags array. You

will notice that because we are adding an additional value to an array, as 267

Chapter 9 Working With the JSon Data type

opposed to updating an existing value, we have used the append keyword

at the beginning of the path expression. Note, too, that because we are

updating the array, rather than a single element, the array element in

square brackets is not included.

***Listing 9-15.*** Adding an Additional Array Element

USE WideWorldImporters

GO

UPDATE StockItems

SET CustomFields = JSON_MODIFY(CustomFields,'append lax

$.Tags', 'Vintage')

FROM Warehouse.StockItems

WHERE StockItemID = 61 ;

Let’s now use the query in Listing [9-16](#index_split_002.html#p279), to examine the updated record.

***Listing 9-16.*** Examining the Updated Record

USE WideWorldImporters

GO

SELECT

StockItemName

, JSON_VALUE(CustomFields,'lax $.Tags[0]') AS Tag0

, JSON_VALUE(CustomFields,'lax $.Tags[1]') AS Tag1

, JSON_VALUE(CustomFields,'lax $.Tags[2]') AS Tag2

, JSON_QUERY(CustomFields,'lax $.Tags') AS TagsArray

FROM Warehouse.StockItems

WHERE StockItemID = 61 ;

This query returns the results in Figur[e 9-12\. Y](#index_split_002.html#p280)ou will notice that the second array element now reads "Very Realistic Sound", and the array now contains a third element, marking the product as vintage.

268

![](index-280_1.jpg)

![](index-280_2.jpg)

Chapter 9 Working With the JSon Data type

***Figure 9-12.** Results of examining the modified row*

As you might expect, the MODIFY_JSON() function can also be used to

delete data. This is achieved by updating a value with a NULL. For example, imagine that marking Stock Item ID 61 as vintage was a mistake. We could

correct that mistake by using the query in Listing [9-17.](#index_split_002.html#p280)

***Listing 9-17.*** Deleting Data with MODIFY_JSON()

USE WideWorldImporters

GO

UPDATE StockItems

SET CustomFields = JSON_MODIFY(CustomFields,

'lax $.Tags[2]', NULL)

FROM Warehouse.StockItems

WHERE StockItemID = 61 ;

Rerunning the query in Listing [9-16 no](#index_split_002.html#p279)w returns the results shown in Figur[e 9-13.](#index_split_002.html#p280)

***Figure 9-13.** Results of reexamining the modified row*

269

Chapter 9 Working With the JSon Data type

Alternatively, we could delete the entire Tags array, by using the query

in Listing [9-18](#index_split_002.html#p281).

***Listing 9-18.*** Deleting the Tags Array

USE WideWorldImporters

GO

UPDATE StockItems

SET CustomFields = JSON_MODIFY(CustomFields,'lax

$.Tags',NULL)

FROM Warehouse.StockItems

WHERE StockItemID = 61 ;

The query in Listing [9-19 allo](#index_split_002.html#p281)ws us to examine the modified JSON

document. You will notice that the Tags array no longer exists.

***Listing 9-19.*** Examining the Modified Document

USE WideWorldImporters

GO

SELECT

StockItemName

, JSON_QUERY(CustomFields,'lax $') AS EntireDocument

FROM Warehouse.StockItems

WHERE StockItemID = 61 ;

270

![](index-282_1.jpg)

Chapter 9 Working With the JSon Data type

The results of this query are shown in Figure [9-14](#index_split_002.html#p282).

***Figure 9-14.** Examining the modified JSON document*

**Indexing JSON Data**

When querying a table and filtering, grouping, or ordering by properties

within a JSON document, you can improve the performance of your

queries by indexing the data. Because JSON doesn’t have its own data

type, as XML does, there are no JSON indexes, as there are XML indexes.

Instead, to index JSON properties, you must create a computed column

that includes a JSON_VALUE() call, which mirrors the logic in the query you are optimizing. You can then create an index on the computed column,

and SQL Server will use this index when optimizing the query.

**Note** actual query performance depends on many factors, including

system hardware, and other resource constraints, such as other

queries that may be running simultaneously. the performance

analysis in this section is meant to illustrate potential impacts, but

performance should always be tested on your own servers, under

realistic workloads.

Before demonstrating this technique, let’s first take a baseline of query performance against the Warehouse.StockItems table. We will do this by

turning on TIME STATISTICS in our session, before running the query

that we are trying to optimize. Prior to this, however, we will first copy 271

Chapter 9 Working With the JSon Data type

the data from the StockItems table to a new table. The reason for this is twofold. First, the StockItems table already has a number of indexes, which could potentially influence our results. The second reason is because the StockItems table is system-versioned with indexes. This means that when

we alter the table, to add computed columns, instead of a simple ALTER

TABLE script, we would be required to script out the data to a temp table, drop and re-create both the table and the archive table, and then script

the data back in. This amount of code would distract from how to add a

computed column, which is the point of this exercise. This is demonstrated in Listing [9-20](#index_split_002.html#p283).

**Tip** We use DBCC FREEPROCCACHE to drop any existing plans from

the plan cache. We then use DBCC DROPCLEANBUFFERS to remove

pages from the buffer cache that have not been modified. this helps

make it a fair test.

***Listing 9-20.*** Creating a Performance Baseline

USE WideWorldImporters

GO

--Copy data to a new table

SELECT *

INTO Warehouse.NewStockItems

FROM Warehouse.StockItems

--Clear plan cache

DBCC FREEPROCCACHE

--Clear buffer cache

272

![](index-284_1.jpg)

Chapter 9 Working With the JSon Data type

DBCC DROPCLEANBUFFERS

--Turn on statistics

SET STATISTICS TIME ON

SELECT

StockItemName

, JSON_VALUE(CustomFields,'lax $.Tags[0]') AS Tag0

, JSON_VALUE(CustomFields,'lax $.Tags[1]') AS Tag1

, JSON_VALUE(CustomFields,'lax $.Tags[2]') AS Tag2

, JSON_QUERY(CustomFields,'lax $.Tags') AS TagsArray

FROM Warehouse.NewStockItems

WHERE JSON_VALUE(CustomFields,'lax $.Tags[0]') = 'Radio Control' ;

The results in Figur[e 9-15 sho](#index_split_002.html#p284)w that the query took 6ms to execute.

***Figure 9-15.** Time statistics*

Let’s now create a computer column on the Warehouse.StockItems

table, using the same logic as in our WHERE clause. This can be achieved by using the script in Listing [9-21.](#index_split_002.html#p285)

273

Chapter 9 Working With the JSon Data type

***Listing 9-21.*** Creating a Computer Column

USE WideWorldImporters

GO

ALTER TABLE Warehouse.NewStockItems

ADD CustomFieldsTag0 AS JSON_VALUE(CustomFields,

'lax $.Tags[0]') ;

We can now index the computed column (which will also cause the

column to be persisted, as opposed to calculated on the fly, when queried), by using the script in Listing [9-22.](#index_split_002.html#p285)

***Listing 9-22.*** Indexing the Computed Column

USE WideWorldImporters

GO

CREATE NONCLUSTERED INDEX NCI_CustomFieldsTag0

ON Warehouse.NewStockItems(CustomFieldsTag0) ;

Let’s now check the performance of our query, once again, by using the

simplified script in Listing [9-23.](#index_split_002.html#p285)

***Listing 9-23.*** Checking Index Performance

USE WideWorldImporters

GO

--Clear plan chahe

DBCC FREEPROCCACHE

--Clear buffer cache

DBCC DROPCLEANBUFFERS

274

![](index-286_1.jpg)

Chapter 9 Working With the JSon Data type

--Turn on statistics

SET STATISTICS TIME ON

SELECT

StockItemName

, JSON_VALUE(CustomFields,'lax $.Tags[0]') AS Tag0

, JSON_VALUE(CustomFields,'lax $.Tags[1]') AS Tag1

, JSON_VALUE(CustomFields,'lax $.Tags[2]') AS Tag2

, JSON_QUERY(CustomFields,'lax $.Tags') AS TagsArray

FROM Warehouse.NewStockItems

WHERE JSON_VALUE(CustomFields,'lax $.Tags[0]') = 'Radio Control' ;

You can see from the time statistics shown in Figur[e 9-16 th](#index_split_002.html#p286)at the query executed in 3ms. That’s a 50% performance improvement!

***Figure 9-16.** Results of checking index performance*

275

Chapter 9 Working With the JSon Data type

**Tip** Because the newStockitems table is so small, there is a

chance that the query optimizer will choose not to use your index.

if this happens, you can force it to use the index, by adding the

WITH (INDEX(NCI_CustomFieldsTag0)) query hint. it is very

important to note, however, that on a general basis, the optimizer is

smart and rarely should be given hints. if you do need to use hints,

then you should always work with the optimizer, rather than against it.

For example, if the optimizer is incorrectly choosing a LOOP JOIN

physical operator, force it to use either MERGE JOIN or HASH JOIN.

Do not choose for it which is better!

**Summary**

SQL Server provides the ability to interrogate and modify JSON data with

the ISJSON(), JSON_VALUE(), JSON_QUERY(), and JSON_MODIFY() functions.

The ISJSON() function provides a simple validation that the document has

a valid JSON format. It returns 1 if the document is JSON and 0 if not.

The JSON_VALUE() function can be included in the SELECT list, WHERE

clause, ORDER BY clause, or GROUP BY clause of your query. It returns a

single scalar value from a JSON document that is passed to it, based on a path expression.

The JSON_QUERY() function can also be included in the SELECT list,

WHERE clause, ORDER BY, or GROUP BY, but instead of returning a single

scalar value, it returns a JSON object or array. As with JSON_VALUE(),

the object returned is based on a path expression that is passed to the

function.

276

Chapter 9 Working With the JSon Data type

The MODIFY_JSON() function can be used to update, insert, or delete

key values. A JSON document and a path expression are passed to the

function, and the complete modified document is returned, making it easy

to use in a standard UPDATE statement. The optional append keyword in the path expression is used to denote that the intention is to add an additional value to an array, as opposed to modifying an existing value. Updating a

key value with NULL deletes the key.

Query performance can be improved by indexing the properties of a

JSON document. This is achieved by creating a computed column, based

on the path expression that you wish to optimize. You can then create an

index on the computed column. This allows the query optimizer to use the

index when the column containing the JSON data is queried.

277

**CHAPTER 10**

**Understanding**

**Spatial Data**

Spatial data is data that describes a location. It is said that 80% of the world’s data has a spatial element to it, so you can see how, for many datatier applications, the ability to construct or interrogate spatial data may be critical. In this chapter, I will give an overview of spatial data and its implementation in SQL Server, before looking at the spatial standards that are used in the SQL Server implementation.

**Note** This chapter is intended to provide a theoretical overview of spatial data. Practical examples of usage can be found in Cha[pter 11](#index_split_002.html#p308).

**Understanding Spatial Data**

SQL Server provides two data types that are able to store spatial data:

GEOMETRY and GEOGRAPHY. These data types are implemented through CLR

(Common Language Runtime) and, behind the scenes, are .NET classes,

which provide a series of methods and properties that allow you to interact with the data.

© Peter A. Carter 2018

279

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_10](#index_split_002.html#p289)

ChAPTer 10 UndersTAnding sPATiAl dATA

**Caution** A geospatial object can be referred to as a geometry, with simple objects referred to as primitive geometries and collections of

objects referred to as geometry collections. Because geometry is also

the name of the data type that implements spatial data as a flat-earth

model, this can cause confusion. Therefore, please note that when

this chapter refers to a geospatial object, the word *geometry* will be used in lowercase. When referring to the data type, the word will be

in uppercase letters (GEOMETRY).

The GEOMETRY data type uses the Euclidean coordinates system, which

provides a flat-earth model. GEOMETRY is the correct data type to use when you are dealing with 2D objects or small objects that are not required to consider the curvature of the earth.

When using the GEOMETRY data type, objects are plotted using x and y

axes. For example, the point in Figure [10-1 is lo](#index_split_002.html#p291)cated (or plotted) at 5, 3.

280

![](index-291_1.png)

ChAPTer 10 UndersTAnding sPATiAl dATA

***Figure 10-1.** Plotting a point in a Euclidean system*

**Tip** A shape is referred to as a surface.

Similarly, the polygon (closed surface) in Figur[e 10-2](#index_split_002.html#p292) is drawn using the coordinates (-1 -1, -1 2, 2 2, 2 -1, -1 -1).

281

![](index-292_1.jpg)

ChAPTer 10 UndersTAnding sPATiAl dATA

***Figure 10-2.** Plotting a polygon in a Euclidean system*

The GEOGRAPHY data type, on the other hand, uses a round-earth model

(or, actually, multiple round-earth models) and is ideal for representing 3D

objects or large areas that must consider the curvature of the earth.

When working with data for a round-earth model, instead of x and y

coordinates, a point can be identified by providing longitude and latitude.

Lines of longitude, also called meridians, indicate the number of degrees east or west of a baseline of longitude, known as the prime meridian.

The prime meridian runs through the borough of Greenwich in London

and is regarded as the point where Eastern and Western Hemispheres

meet. Sometimes, longitude coordinates are referenced as positive and

negative values, with meridians east of the prime meridian being positive 282

![](index-293_1.jpg)

ChAPTer 10 UndersTAnding sPATiAl dATA

and meridians west of the prime meridian being negative. Other times,

meridians will always be referred to as positive numbers, with either

E (East) or W (West) being used to denote direction from the prime

meridian. This is illustrated in Figur[e 10-3](#index_split_002.html#p293).

***Figure 10-3.** Plotting longitude*

The equator is the central point between the North and South Poles.

Latitude indicates the number of degrees north or south of the equator.

Therefore, the equator has a latitude of 0 degrees. A line of latitude

represents all points on Earth that have the same degree of latitude. As

indicated in Figur[e 10-4](#index_split_002.html#p294), sometimes degrees of latitude are referred to as positive and negative values, with the Northern Hemisphere being positive values and the Southern Hemisphere being negative values. Other times,

latitude is always referenced with positive values, differentiated by a suffix of either north latitude or south latitude.

283

![](index-294_1.jpg)

ChAPTer 10 UndersTAnding sPATiAl dATA

***Figure 10-4.** Plotting latitude*

Both the GEOMETRY and GEOGRAPHY data types allow you to instantiate a

number of objects. These objects can be broken down into two categories.

The first category is single geometry instances, also known as primitive

geometries. The second category is geometry collections, also known as

multipart geometries. Ta[ble 10-1](#index_split_002.html#p295) details the primitive geometries that are supported by SQL Server.

284

ChAPTer 10 UndersTAnding sPATiAl dATA

***Table 10-1.** Primitive Geometries*

**Geometry**

**Description**

Point

A single point specified with a single set of coordinates

linestring

A path that joins multiple points, defined with multiple

sets of coordinates. A line string can either be enclosed or

unenclosed.*

Circularstring

A path, in the form of an arc, between multiple points. There

must be an odd number of points defined, with a minimum of

three. The points cannot all be on the same axis (or it will be

treated as a linestring).

CompoundCurve

A path that joins multiple sets of points. it can be defined by

using multiple linestrings and/or Circularstrings, to create

surfaces such as circles or semicircles.

Polygon

An enclosed surface defined by specifying the points that

map each corner of the polygon. each side of the polygon is

essentially a linestring.

CurvePolygon

An enclosed surface. This is similar to Polygon, except that it may

be defined using linestrings, Circularstring, or CompoundCurves,

instead of just linestrings, as is the case of Polygon.

**An enclosed surface is one in which the points used to define it join. Therefore, the* *first coordinates defined will also be the last coordinates defined.*

285

ChAPTer 10 UndersTAnding sPATiAl dATA

Table [10-2](#index_split_002.html#p296) defines the multipart geometries that are supported by SQL

Server.

***Table 10-2.** Multipart Geometries*

**Geometry**

**Description**

MultiPoint

A collection of 0 or more points

Multilinestring

A collection of 0 or more linestrings

MultiPolygon

A collection of 0 or more Polygons

geometryCollection

A collection of 0 or more geometries. Can include any

primitive or multipart geometries

In SQL Server 2017, a new surface type has been added, which is

applicable only to the GEOGRAPHY data type. FullGlobe represents the entire surface of the planet. Therefore, it has an area, but no borders.

**Tip** Practical examples of creating and working with primitive

geometries and geometry collections can be found in Chapter [11](#index_split_002.html#p308).

**Spatial Data Standards**

Spatial data in SQL Server has been implemented around the standards set

out by the Open Geospatial Consortium (OGC). These standards can be

found a[t www.opengeospatial.org/standards/gml](http://www.opengeospatial.org/standards/gml).

The following sections will provide an overview of the spatial surfaces

available within SQL Server and how they can be represented, using well-

known text and well-known binary. I will also discuss spatial reference

systems.

286

ChAPTer 10 UndersTAnding sPATiAl dATA

**Well-Known Text**

The OGC lays out two methods of referencing spatial data: well-known text (WKT) and well-known binary (WKB). I’ll first discuss WKT, and how each

geometry can be specified using this approach. Table [10-3 details the WK](#index_split_002.html#p297)T

for each primitive geometry.

***Table 10-3.** Primitive Geometry WKT*

**Geometry**

**WKT**

**Description**

Point

POinT(x y z m)

x indicates the x axis or

longitude; y indicates the

y axis or latitude;

z is optional and indicates the

elevation;

m is optional and indicates

the measure.

linestring

linesTring(POinT 1, POinT 2,

POinT n)

Circularstring

CirCUlArsTring(start POinT,

Anchor POinT 1, Anchor POinT n,

end POinT)

CompoundCurve

COMPOUndCUrVe((POinT

When an arc segment is a

1, POinT n),

Circularstring, the set of

CirCUlArsTring(start POinT, points must be prefixed with

Anchor POinT, end POinT))

the geometry type. When the

arc segment is a linestring,

there is no need to specify the

geometry type.

( *continued*)

287

ChAPTer 10 UndersTAnding sPATiAl dATA

***Table 10-3.*** ( *continued*)

**Geometry**

**WKT**

**Description**

Polygon

POlYgOn((POinT 1, POinT 2,

Polygon can also contain an

POinT n))

inner Polygon, by defining a

second set of points within

parentheses, separated by a

comma.

CurvePolygon

CUrVePOlYgOn((CirCUlArsTri A CurvePolygon can

ng(POinT 1, POinT 2, POinT n))

also contain an inner

CurvePolygon, by defining a

second set of points within

parentheses, separated by a

comma.

Table [10-4 details the WK](#index_split_002.html#p298)T for multipart geometries.

***Table 10-4.** Multipart Geometry WKT*

**Geometry**

**WKT**

MultiPoint

MUlTiPOinT((POinT 1),(POinT n))

Multilinestring

MUlTilinesTring((POinT 1, POinT n), (POinT 1, POinT n))

MultiPolygon

MUlTiPOlYgOn((POinT 1, POinT 2, POinT n),(POinT 1,

POinT 2, POinT n))

geometryCollection

geOMeTrYCOlleCTiOn(geOMeTrYTYPenAMe(POinT

n),geOMeTrYTYPenAMe(POinT n))

288

ChAPTer 10 UndersTAnding sPATiAl dATA

**Well-Known Binary**

WKB can also be used to represent a geometry. This binary data is usually represented as a hexadecimal value. When working with WKB, you can

control the order of bytes before specifying the surface type and, finally, the points specification. This section will discuss how each of the bytes in WKB are used.

The first byte indicates the byte order, in which 00 is big endian and 01

is little endian.

**Tip** Big endian means that the least significant byte in the sequence is stored last, and little endian means that the least significant byte is stored first.

The following four bytes represent the type of surface being defined.

Table [10-5](#index_split_002.html#p299) describes the integer codes for each of these bytes.

***Table 10-5.** WKB Surface Type Integer Codes*

**Type**

**2D**

**Z**

**M**

**ZM**

Point

0001

1001

2001

3001

linestring

0002

1002

2002

3002

Circularstring

0008

1008

2008

3008

CompoundCurve

0009

1009

2009

3009

Polygon

0003

1003

2003

3003

CurvePolygon

0010

1010

2010

3010

MultiPoint

0004

1004

2004

3004

Multilinestring

0005

1005

2005

3005

MultiPolygon

0006

1006

2006

3006

geometryCollection

0007

1007

2007

3007

289

ChAPTer 10 UndersTAnding sPATiAl dATA

The remaining bytes consist of eight-byte floats, which describe the

points. For example, imagine a POINT, with coordinates 4, 2\. Using big

endian, the hexadecimal representation would be 0x0101000000000000000

00010400000000000000040\. If we break this down into its constituent parts 01—Indicates the byte direction

01000000—Indicates the surface type

0000000000001040—Indicates the x axis

0000000000000040—Indicates the y axis

**Tip** each byte is represented by a two-character hexadecimal

reference.

Table [10-6](#index_split_002.html#p301) provides some examples of this.

290

ChAPTer 10 UndersTAnding sPATiAl dATA

***Table 10-6.** WKB Examples*

**WKT**

**WKB (Hexadecimal Representation—Big Endian)**

POinT(1 1)

0x0101000000000000000000F03F00000000000

0F03F

POinT(1 1 1 1)

0x00000000010F000000000000F03F0000000000

00F03F000000000000F03F000000000000F03F

linesTring(1 1, 2 2)

0x000000000114000000000000F03F0000000000

00F03F00000000000000400000000000000040

POlYgOn((1 1, 2 2, 2 1, 1 1))

0x00000000010404000000000000000000F03F00

0000000000F03F00000000000000400000000000

0000400000000000000040000000000000F03F00

0000000000F03F000000000000F03F0100000002

0000000001000000FFFFFFFF0000000003

geOMeTrYCOlleCTiOn

0x00000000010402000000000000000000F03

(POinT(1 1),POinT(2 2))

F000000000000F03F00000000000000400000

0000000000400200000001000000000101000

00003000000FFFFFFFF0000000007000000000-

000000001000000000100000001

**Spatial Reference Systems**

When working with the GEOGRAPHY data type, we must be aware that there

is not one single unified view of how to represent the earth. Instead, there are multiple reference systems, known as spatial reference systems. Each

system provides its own map projections. Each system is identified by an

SRID (spatial reference identifier). The SRIDs supported by SQL Server can be found by interrogating the sys.spatial_reference_systems system table, as demonstrated in Listin[g 10-1](#index_split_002.html#p302).

291

![](index-302_1.jpg)

ChAPTer 10 UndersTAnding sPATiAl dATA

**Tip** The earth cannot be modeled on a plane without being

distorted. A map projection defines the distortions that are acceptable

and unacceptable. each projection allows for different distortions.

***Listing 10-1.*** Discovering Spatial References

SELECT *

FROM sys.spatial_reference_systems ;

Partial results of this query can be found in Figure [10-5](#index_split_002.html#p302).

***Figure 10-5.** Results of discovering spatial reference systems*

You will notice that the table provides various details about each SR,

including its ID; the authority that develops and maintains it; the unit of measurement, along with the unit’s conversion factor, against the base

measure of meters; and, most interestingly, the WKT of the SR, as defined by the OGC.

292

ChAPTer 10 UndersTAnding sPATiAl dATA

Let’s examine the WKT for Microsoft’s implementation of SR (SRID

104001) , which is detailed in Listing [10-2.](#index_split_002.html#p303)

***Listing 10-2.*** WKT for SRID 104001

GEOGCS["Unit Sphere",

DATUM["Unit Sphere",

SPHEROID["Sphere", 1.0, 0.0]

],

PRIMEM["Greenwich",0.0],

UNIT["Degree", 0.0174532925199433]

]

Note that the WKT uses a JSON format (see Chapter[s 6–](#index_split_001.html#p192)[9 for a detaile](#index_split_002.html#p262)d discussion of JSON) and specifies map projection details, such as the

prime meridian and unit of measurement.

**Caution** it is important to note that while a single GEOGRAPHY

column can contain values based on multiple spatial reference

systems, it is not possible to compare these values. spatial functions

require that all values have the same srid, as they contain the

background information required for calculations. That said, however,

it is possible for a value to be converted to a different srid before a

comparison is performed.

293

![](index-304_1.jpg)

ChAPTer 10 UndersTAnding sPATiAl dATA

**SSMS and Spatial Data**

SQL Server Management Studio (SSMS) provides graphical results, to

assist developers when working with spatial data. For example, imagine

a simple script, such as Listing [10-3, whic](#index_split_002.html#p304)h declares a variable of type GEOMETRY and sets it to contain a simple CurvePolygon.

***Listing 10-3.*** Declaring a CurvePolygon with GEOMETRY

DECLARE @CurvePolygon GEOMETRY =

'CURVEPOLYGON(CIRCULARSTRING(1 3, 3 5, 4 7, 7 3, 1 3))' ;

SELECT @CurvePolygon ;

The normal results tab will return the WKB of the surface, in

hexadecimal notation, as shown in Figur[e 10-6.](#index_split_002.html#p304)

***Figure 10-6.** Results in WKB*

294

![](index-305_1.jpg)

ChAPTer 10 UndersTAnding sPATiAl dATA

You will notice, however, that a new tab, Spatial results, is present

between the Results and Messages tabs. This pane can be seen in

Figur[e 10-7.](#index_split_002.html#p305)

***Figure 10-7.** Spatial results tab—GEOMETRY*

You will notice that we can see a graphical representation of our surface, along with some basic controls that allow us to zoom in and out and also

flick between columns, if the result set contains multiple spatial columns. It is also possible to scroll around the object, by dragging the mouse.

Consider Listing [10-4\. Y](#index_split_002.html#p305)ou will notice that the script is very similar to Listin[g 10-3, ex](#index_split_002.html#p304)cept that we are now declaring a variable of type GEOGRAPHY, instead of GEOMETRY.

***Listing 10-4.*** Declaring a CurvePolygon with GEOGRAPHY

DECLARE @CurvePolygon GEOGRAPHY =

'CURVEPOLYGON(CIRCULARSTRING(1 3, 3 5, 4 7, 7 3, 1 3))' ;

SELECT @CurvePolygon ;

295

![](index-306_1.jpg)

ChAPTer 10 UndersTAnding sPATiAl dATA

Figur[e 10-8 sho](#index_split_002.html#p306)ws the Spatial results tab for this script. You will notice that the grid has changed to degrees, as opposed to Euclidean style

coordinates, to reflect that a round-earth model is now in use, and our

coordinates are actually longitude and latitude.

***Figure 10-8.** Spatial results tab—GEOGRAPHY*

**Summary**

SQL Server provides two data types for working with spatial data: GEOMETRY

and GEOGRAPHY. GEOMETRY allows you to work in a Euclidean (flat-earth)

model, while GEOGRAPHY allows you to work in a round-earth model. SQL

Server’s spatial implementation is based on standards set out by the Open Geospatial Consortium (OGC).

Spatial shapes, known as surfaces, can be defined by using either

well-known text (WKT) or well-known binary (WKB), which are standards

set out by the OGC. When working with WKT, the surface type is defined

by name, followed by its points specification. When working with WKB,

you can control the order of bytes before specifying the surface type

and, finally, the points specification. WKB is usually represented using

hexadecimal notation.

296

ChAPTer 10 UndersTAnding sPATiAl dATA

SQL Server supports the following primitive and multipart

geographies, as well as support for a FullGlobe geometry, when using

GEOGRAPHY in SQL Server 2017 or later versions:

• Point

• LineString

• CircularString

• CompoundCurve

• Polygon

• CurvePolygon

• MultiPoint

• MultiLineString

• MultiPolygon

• GeometryCollection

SQL Server supports many spatial reference systems. This is important

when working with GEOGRAPHY, as each SRID implements its own map

projections. Surfaces with conflicting SRIDs can be stored in the same

column but cannot be compared using spatial functions.

SQL Server Management Studio provides graphical results for spatial

queries. The Spatial results tab allows you to move around a surface,

zooming in and out. The grid will change between Euclidean coordinates

and latitude/longitude, depending on the data type of the results.

297

**CHAPTER 11**

**Working with**

**Spatial Data**

In Chapter [10](#index_split_002.html#p289), I discussed the concepts associated with spatial data, which SQL Server implements using the GEOMETRY and GEOGRAPHY data types.

In this chapter, I will examine how to work with these data types. First, I will discuss the methods that can be used to construct surfaces, before reviewing how to query spatial data. Finally, you will see how to design and create spatial indexes.

**Caution** A geospatial object can be referred to as a geometry, with simple objects referred to as primitive geometries and collections of

objects referred to as geometry collections. Because geometry is also

the name of the data type that implements spatial data as a flat-earth

model, this can cause confusion. Therefore, please note that when

this chapter refers to a geospatial object, the word *geometry* will be used in lowercase. When referring to the data type, the uppercase

word GEOMETRY will be used.

© Peter A. Carter 2018

299

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_11](#index_split_002.html#p308)

ChAPTer 11 WorKInG WITh SPATIAL DATA

**Constructing Spatial Data**

The GEOMETRY and GEOGRAPHY data types expose a number of methods that

can be used to interact with spatial data. Many of these methods form part of Open Geospatial Consortium (OGC) standards, while others are an

extension of that standard. Ta[ble 11-1](#index_split_002.html#p309) details the methods that can be used for constructing geometries, all of which are from OGC specifications and are exposed through both GEOMETRY and GEOGRAPHY.

***Table 11-1.** Methods for Constructing Geometries*

**Spatial Type**

**From WKT**

**From WKB**

Point

STPointFromText

STPointFromWKB

LineString

STLineFromText

STLineFromWKB

Polygon

STPolyFromText

STPolyFromWKB

Any Primitive Spatial Instance STGeomFromText

STGeomFromWKB

MultiPoint

STMPointFromText

STMPointFromWKB

MultiLineString

STMLineFromText

STMLineFromWKB

MultiPolygon

STMPolyFromText

STMPolyFromWKB

Any Multi Spatial Instance

STMGeomCollFromText STMGeomCollFromWKB

Each of these methods accepts two parameters. The first parameter

is the well-known text (WKT) or well-known binary (WKB) of the spatial

instance. The second parameter is the SRID (please refer to Chapt[er 10, for](#index_split_002.html#p289)

further details on SRIDs) that the spatial instance should use. When called against a column or variable of type GEOMETRY, 0 can be passed as the SRID, as map projections are not required in a Euclidean model (see Chapter [10).](#index_split_002.html#p289)

When called against a column or variable of type GEOGRAPHY, however, a

valid SRID must be used, and passing 0 will result in an error being thrown by the .NET framework.

300

![](index-310_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

The script in Listin[g 11-1](#index_split_002.html#p310) will create a LineString in a variable of type GEOMETRY, by using well-known-text.

***Listing 11-1.*** Creating a LineString with Well-Known Text

DECLARE @LineString GEOMETRY ;

SET @LineString = GEOMETRY::STLineFromText('LINESTRING(0 0, 4.5

5, 4.5 0, 0 0)', 0)

SELECT @LineString ;

The graphical results of this script can be found in Figur[e 11-1.](#index_split_002.html#p310)

***Figure 11-1.** Results of creating a LineString from WKT*

Alternatively, the same spatial instance could be created with

WKB, by using the script in Listin[g 11-2.](#index_split_002.html#p311)

301

ChAPTer 11 WorKInG WITh SPATIAL DATA

***Listing 11-2.*** Creating a LineString from a Well-Known Binary DECLARE @LineString GEOMETRY ;

SET @LineString = GEOMETRY::STLineFromWKB(0x01020000000400000

00000000000000000000000000000000000000000000012400000000000

001440000000000000124000000000000000000000000000000000000000000

0000000, 0);

SELECT @LineString ;

**Tip** An additional extended method exists for constructing

geometries. GeomFromGML allows you to instantiate an object, based

on the Geography Markup Language (GML). For example, the script

in Listing [11-3 will crea](#index_split_002.html#p311)te a LineString from GML. A full discussion of GML is beyond the scope of this book, but the oGC specification can

be found a[t www.opengeospatial.org/standards/gml](http://www.opengeospatial.org/standards/gml).

***Listing 11-3.*** Creating a LineString from GML

DECLARE @LineString GEOMETRY ;

SET @LineString =

GEOMETRY::GeomFromGml('<LineString xmlns="http://www.opengis.

net/gml">

<posList>0 0 4.5 5 4.5 0 0 0</posList> </LineString>', 0) ; SELECT @LineString ;

SQL Server also provides an extended method for creating a spatial

instance, by defining the x and y coordinates of a point. Listin[g 11-4](#index_split_002.html#p312)

demonstrates the use of the Point method against a GEOMETRY variable.

The method accepts three parameters. The first is the x axis coordinate, the second is the y axis coordinate, and the third is the SRID.

302

ChAPTer 11 WorKInG WITh SPATIAL DATA

***Listing 11-4.*** Using the Point Method

DECLARE @Point GEOMETRY ;

SET @Point = GEOMETRY::Point(10, 10, 0) ;

SELECT @Point ;

Spatial instances can also be instantiated by simply passing the well-

known text or well-known binary as a value. For example, consider the

script in Listing [11-5.](#index_split_002.html#p312)

***Listing 11-5.*** Passing Well-Known Text to a Variable

DECLARE @Polygon GEOMETRY ;

SET @Polygon = 'POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))' ;

SELECT @Polygon ;

303

![](index-313_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

The graphical results of this script are displayed in Figure [11-2](#index_split_002.html#p313).

***Figure 11-2.** Results of passing well-known text to a variable*

A spatial instance can be set to NULL, either by passing a NULL value

directly or by using the read-only Null property of the instance. This is demonstrated in Listing [11-6.](#index_split_002.html#p313)

***Listing 11-6.*** Setting an Instance As NULL

--SET Variable As NULL by passing A NULL Directly

DECLARE @Polygon GEOMETRY ;

SET @Polygon = NULL ;

SELECT @Polygon ;

304

![](index-314_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

--Set A Variable As NULL by using the read-only Null Property

SET @Polygon = GEOMETRY::[Null] ;

SELECT @Polygon ;

It is possible to check the validity of your well-known text or

well-known binary by using the STIsValid method. If the instance is not

valid, it can be made valid, by using the MakeValid method. For example,

consider the script in Listin[g 11-7](#index_split_002.html#p314), which instantiates a Polygon that overlaps itself.

***Listing 11-7.*** Validating and Correcting an Instance

DECLARE @Polygon GEOMETRY ;

SET @Polygon = GEOMETRY::STGeomFromText('POLYGON((0 0, 10 10,

10 0, 0 10, 0 0))', 0) ;

SELECT

@Polygon.STIsValid() AS IsValid

, @Polygon.MakeValid().ToString() AS Fixed

, @Polygon AS WKB ;

The results, shown in Figur[e 11-3, sho](#index_split_002.html#p314)w that the original value has returned 0, because it is not valid, and the fixed version has converted it to a MultiPolygon.

***Figure 11-3.** Results of validating and correcting an instance*

305

![](index-315_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

The graphical results of the query are shown in Figure [11-4.](#index_split_002.html#p315)

***Figure 11-4.** Graphical results of validating and correcting an* *instance*

It is worth noting that because of the way in which spatial data is

parsed, the use of STIsValid and MakeValid are somewhat limited. For

example, consider the script in Listing [11-8, whic](#index_split_002.html#p315)h attempts to instantiate a Polygon that is not enclosed.

***Listing 11-8.*** Attempt to Instantiate a Non-enclosed Polygon DECLARE @LineString GEOMETRY ;

SET @LineString = GEOMETRY::STGeomFromText('POLYGON((0 0, 10

10, 10 0, 0 10))', 0) ;

306

![](index-316_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

SELECT

@LineString.STIsValid() AS IsValid

, @LineString.MakeValid().ToString() AS Fixed

, @LineString AS WKB ;

Because the Polygon is not enclosed, it fails to parse, and the error,

illustrated in Figur[e 11-5](#index_split_002.html#p316), is thrown.

***Figure 11-5.** Error thrown from instantiating a non-enclosed*

*Polygon*

**Querying Spatial Data**

SQL Server provides numerous methods for interacting with spatial data.

An overview of these methods can be found in Ta[ble 11-2](#index_split_002.html#p317). The table details the name of each method, whether it is an OGC standard or a Microsoft

extension, and the data types the method works with. The final column

provides a brief description of the method’s ability.

307

ChAPTer 11 WorKInG WITh SPATIAL DATA

returns

tes a

wing for

tive.

tial instance,

g called

tial instance

allo

tial instance,

if present

if present

y of a spa

phy instance is less

and a bit fla

, which designa

of the spa

tial instance

denoting a distance from

wn binar

wn text of a spa

the tolerance,

t denotes if the tolerance is rela

returns an object representing all points,

tial object,

tial object representing the union of all points,

tive tha

**Description**

returns the well-kno

augmented with Z and M values,

returns the GML of a spa

returns the well-kno

augmented with Z and M values,

Accepts a single parameter

distance.

within the distance supplied,

against which the method was called

Accepts three parameters,

a spa

rela

a spa

whose distance from a geogra

than or equal to the distance specified,

the specified tolerance

&

&

&

&

&

**orks WithW**

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

*ds*

**OGC/Extended**

extended

extended

extended

extended

extended

*etho*

*al M*

***Spati***

 *****Table 11-2.***

**Method**

AsBinaryZM

AsGml

AsTextZM

BufferWithCurves

BufferWithTolerance

308

ChAPTer 11 WorKInG WITh SPATIAL DATA

*d*)

tion

*ue*

tive.

le and

*tin*

passed as

pproxima

s bounding

results may

( *con*

tial object the

ting a tolerance,

s bounding circ

tial object,

tial instance’

designa

returns 1 if the objects

t represents an a

ting if the tolerance is rela

phy instance

gainst.

gainst which the method was called,

returns 1 if an M value exists and 0

indicag, olygon tha

, intersects with the spa

tes if an M value has been specified for a

le

tial instance.

Accepts two parameters,

and a bit fla

returns a P

of the instance a

which contains no arc segments

returns the maximum angle (in degrees) between

the center point of an instance’

a point in the geogra

returns the center of a spa

circ

Attempts to determine if a spa

a parameter

method is called a

potentially intersect and 0 if they don’t.

be unreliable and return false positives.

Valida

spa

if it does not

&

&

&

GEOGRAPHY

GEOMETRY

GEOGRAPHY

only

GEOGRAPHY

only

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

Extended

extended

extended

extended

extended

CurveToLineWithTolerance

EnvelopeAngle

EnvelopeCenter

Filter

HasM

309

ChAPTer 11 WorKInG WITh SPATIAL DATA

tial

If the

. returns 1 if it

.

Null

[11-3](#index_split_003.html#p333)

, which denotes a spa

tial instance the method

Table

oint

the method returns 24400.

it returns an error code describing

These error codes and their

returns 1 if a Z value exists and 0 if

tial instance is

titude of a P

gainst is of the specified type or 0 if the

tes if an Z value has been specified for a

tes if a spa

tial instance.

returns 1 if the spa

**Description**

Valida

spa

it does not

Accepts a single parameter

type.

is called a

instance is a different type

Valida

is and 0 if it contains an object

If an object is valid,

object is not valid,

why it is not valid.

meanings can be found in

returns the la

&

&

&

&

**orks WithW**

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

only

**OGC/Extended**

extended

extended

extended

extended

extended

*d*) *ue*

*tin*

**( *con***

 *****Table 11-2.***

**Method**

HasZ

InstanceOf

IsNull

IsValidDetailed

Lat

310

ChAPTer 11 WorKInG WITh SPATIAL DATA

*d*)

tial

*ue*

*tin*

tial object

( *con*

tibility level,

gainst a spa

te result*

olygon

tial instance

t is not well-formed to

formed

t details a tolerance.

oint

gainst

tabase compa

returns the total number of

pproxima

tion of a P

t is well-

olygon.

olygon

eucker algorithm a

tial instance tha

gainst a P

tial instance tha

t the method is called a

returns the longitude of a P

returns the M value of the spa

Converts a spa

a spa

returns the minimum da

which provides support for the type of spa

tha

Called a

rings within the P

Accepts a single parameter tha

runs a Douglas-P

instance and returns the a

Changes the ring orienta

&

&

&

&

GEOGRAPHY

only

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

only

GEOGRAPHY

GEOMETRY

GEOGRAPHY

only

extended

extended

extended

extended

extended

extended

extended

Long

M

MakeValid

MinDbCompatibilityLevel

NumRings

Reduce

ReorientObject

311

ChAPTer 11 WorKInG WITh SPATIAL DATA

t the given

. returns

tial instance the

t is passed as a

t defines a 1-based

t the method is called

which resides a

gainst,

tial object tha

t represents the shortest distance

tial object tha

tial object as a parameter

WKB of an instance

WKT of an instance

returns a ring from the spa

**Description**

Accepts a single parameter tha

index.

method is called a

index number

Accepts a spa

a LineString tha

between the spa

against and the spa

parameter

returns the surface area of an instance

returns the

returns the

&

&

&

&

**orks WithW**

GEOGRAPHY

only

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

**OGC/Extended**

extended

extended

oGC

oGC

oGC

*d*) *ue*

*tin*

**( *con***

 *****Table 11-2.***

**Method**

RingN

ShortestLineTo

STArea

STAsBinary

STAsText

312

ChAPTer 11 WorKInG WITh SPATIAL DATA

*d*)

oint,

tive,

*ue*

and

method

*tin*

The

the

or example,

. Calling the

y.

( *con*

F

When nega

gainst must

tive.

olygon will return a

s border

otherwise,

, defining distance,

t distance of instance.

gainst the containing

olygon’

returns 1 if the value is

y of an instance.

gainst a P

Polygons;

tial instance is contained by

t the method is invoked a

of more

.

gainst a LineString will return a MultiP

one

NULL

tes if a spa

. Should be called a

returns the boundar

calling the method a

LineString marking the P

method a

with the start and end point of the LineString.

Accepts a single parameter

returns all points within tha

Distance can be positive or nega

the buffer will be interior to the boundar

returns the geometric center of an object.

instance tha

contain

returns

evalua

another

instance and accepts a single parameter for

the contained instance.

contained and 0 if it is not

&

&

GEOMETRY

only

GEOGRAPHY

GEOMETRY

GEOMETRY

only

GEOGRAPHY

GEOMETRY

oGC

oGC

oGC

oGC

STBoundary

STBuffer

STCentroid

STContains

313

ChAPTer 11 WorKInG WITh SPATIAL DATA

t

tial

gainst.

olygon tha

tial instance.

such as a

tial

.

t the method is

where a spa

CircularString,

t describes the

returns an instance

, which defines the index

tial instance,

ves

respectively

tial instance crosses the

returns 1 if the second

olygon.

y instances tha

te polygonal spa

t the method is called a

ved spa

veP

olygon,

or MultiLineString spa

ve of a LineString,

ve,

t should be returned,

tial instance.

pproxima

ve tha

gainst

from a cur

tes if a given spa

tial instance tha

tial instance crosses the first and 0 if it does not

tes an a

**Description**

returns the smallest possible convex P

contains the geometr

called a

Valida

spa

Accepts a single parameter tha

second spa

spa

returns the cur

CompoundCur

Accepts a single parameter

of the cur

instance has multiple cur

Crea

instance,

CircularString or Cur

of a LineString or P

&

&

&

**orks WithW**

GEOGRAPHY

GEOMETRY

GEOMETRY

only

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

**OGC/Extended**

oGC

oGC

oGC

oGC

*d*) *ue*

*tin*

**( *con***

 *****Table 11-2.***

**Method**

STConvexHull

STCrosses

STCurveN

STCurveToLine

314

ChAPTer 11 WorKInG WITh SPATIAL DATA

*d*) *ue*

tial

which is

*tin*

, which

tial

t is passed

( *con*

, which the

gainst

phy

tial instance,

t the method is

t does not reside within

tha

te shortest distance

tial instance tha

to intersect with the spa

. returns 1 if it is empty or 0 if

gainst,

oint used to define a spa

pproxima

tial instance tha

. Accepts a single parameter

t maximum number of dimensions of a

t the method is called a

tial instance,

gainst and a spa

tes if the intersection set between two

tes the a

tial instance

returns the portion of a geogra

method is called a

the boundaries of a second spa

passed as a parameter to the method

returns tha

spa

Valida

geometries is empty

it is not empty

is the spa

instance tha

Calcula

between the spa

called a

to the method as a parameter

returns the final P

instance

&

&

&

&

&

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

oGC

oGC

oGC

oGC

oGC

STDifference

STDimension

STDisjoint

STDistance

STEndpoint

315

ChAPTer 11 WorKInG WITh SPATIAL DATA

y collection.

loses the

as defined by

t enc

t the method

the method will

, which is the 1-based

tial instance,

olygon tha

olygon,

t the method has been called

tial instance tha

t is passed to the method as a

s exterior ring.

gainst a P

olygon’

gainst has the same points set as a

. returns 1 if the points sets are the same

tes if the spa

tial instance tha

tial instance tha

oGC

**Description**

returns a rectangular P

spa

against

Valida

is called a

spa

parameter

and 0 if they are not

When called a

return the P

returns a specific object from a geometr

Accepts a single parameter

index of the object to return

returns the type of a spa

the

&

&

&

**orks WithW**

GEOMETRY

only

GEOGRAPHY

GEOMETRY

GEOMETRY

only

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

**OGC/Extended**

oGC

oGC

oGC

oGC

oGC

*d*) *ue*

*tin*

**( *con***

 *****Table 11-2.***

**Method**

STEnvelope

STEquals

STExteriorRing

STGeometryN

STGeometryType

316

ChAPTer 11 WorKInG WITh SPATIAL DATA

*d*)

t

*ue*

tial

. returns

*tin*

t is

( *con*

t the method

tial instance

gainst which the

the method will

a

t covers the area tha

tial instance tha

olygon,

tial instance tha

ps with a spa

returns 1 if they are the same

s interior ring.

tial instance,

overla

tial instance is empty or contains an

gainst a P

tial instance tha

olygon’

gainst and a spa

pped by a spa

tes if the spa

tes if a spa

returns 1 if the instance is empty and 0 if it

t is passed to the method as a parameter

When called a

return the P

returns a spa

is overla

is called a

passed to the method as a parameter

Valida

method is called,

tha

1 if the instances intersect and 0 if they do not

Checks if the start point and end point of a spa

object are the same.

and 0 if they are not the same

Valida

object.

contains an object

&

&

&

&

GEOMETRY

only

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

oGC

oGC

oGC

oGC

oGC

STInteriorRingN

STIntersection

STIntersects

STIsClosed

STIsEmpty

317

ChAPTer 11 WorKInG WITh SPATIAL DATA

s

wing

wing

or 0 if 1 or

t end point)

t end point)

returns 0 if

tial instance meets the follo

tial instance meets the follo

tes if a spa

losed

tes if a spa

**Description**

Valida

criteria:

• Is a LineString

• Does not intersect itself (except a

• Multiple objects do not intersect each other (except

where the point of intersection is on both object’

boundaries)

• Is c

returns 1 if the conditions are all true,

more of the conditions are false

Valida

conditions:

• Does not intersect itself (except a

• Multiple objects do not intersect each other (except

where the point of intersection is on both objects’

boundaries)

returns 1 if both conditions are true.

either condition is false

**orks WithW**

GEOMETRY

only

GEOMETRY

only

**OGC/Extended**

oGC

oGC

*d*) *ue*

*tin*

**( *con***

 *****Table 11-2.***

**Method**

STIsRing

STIsSimple

318

ChAPTer 11 WorKInG WITh SPATIAL DATA

*d*) *ue*

or a

or a

*tin*

F

. F

one-

( *con*

tial object.

t were used to

primitive,

returns the number of

tial objects in a multi-

olygon contains

oints tha

tial instance is well-formed.

ways 0.

gainst simple,

olygon and returns the number of

tial instances.

t the P

tial instance

t a spa

this is the length of the perimeter

y. returns 1 if the instance is a primitive

y

gainst a P

tes tha

the length is al

ves in the instance

Valida

returns 1 if the instance is well-formed and 0 if it is

not well-formed

returns the total length of a spa

Polygon,

Point,

Must be called a

dimensional spa

cur

returns the number of spa

geometr

geometr

Called a

interior rings tha

returns the number of P

describe a spa

&

&

&

&

&

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOMETRY

only

GEOGRAPHY

GEOMETRY

oGC

oGC

oGC

oGC

oGC

oGC

STIsValid

STLength

STNumCurves

STNumGeometries

STNumInteriorRing

STNumPoints

319

ChAPTer 11 WorKInG WITh SPATIAL DATA

t

ttern

t the method

tial object

tial instance tha

tial instance

gainst which the

oGC Dimensionally

e-9IM) pa

. returns 1 if the objects

, which describes

ps a spa

within the interior of a

ted to a spa

trix used is the second

tion of an object within a

tial instance tha

y. returns the spa

y point,

tial instance a

ttern ma

gainst overla

t index

t tha

tes if the spa

p and 0 if they do not

tes if the spa

The pa

ted a

tial instance

t is passed as the first parameter to the method.

tionships are defined by the

trix.

**Description**

Valida

was called a

is passed in as a parameter

overla

Accepts a single parameter

a 1-based index loca

multipart geometr

loca

returns an arbitrar

spa

Valida

method is called is rela

tha

rela

extended 9 Intersection Model (D

ma

parameter passed to the method.**

&

&

**orks WithW**

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOMETRY

only

GEOMETRY

only

**OGC/Extended**

oGC

oGC

oGC

oGC

*d*) *ue*

*tin*

**( *con***

 **e

***Table 11-2.***

**Method**

STOverlaps

STPointN

STPointOnSurface

STRelat

320

ChAPTer 11 WorKInG WITh SPATIAL DATA

*d*) *ue*

oint

tial

.

*tin*

gainst

tial

a

( *con*

ludes all P

gainst which

tial instance,

which inc

tially touches a spa

tial instances

oint used to define a spa

tial instance a

tial instance the method was called

rID of the spa

tial instance,

tial instance passed as a parameter

tes if the spa

t are:

against

returns the S

which the method was called

returns the first P

instance

returns a spa

tha

• Within the spa

• Within a spa

• not within both spa

Valida

the method was called spa

instance passed to the method as a parameter

returns 1 if the instances touch and 0 if they do not

&

&

&

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOMETRY

only

oGC

oGC

oGC

oGC

STSrid

STStartPoint

STSymDifference

STTouches

321

ChAPTer 11 WorKInG WITh SPATIAL DATA

y,

tial

s x

tial instance

returns 1 if it

oint’

t provides the union

gainst which the

tial instance passed

. The resultant union

, which defines a spa

tial instance passed as a

returns the P

tial instance a

oint.

tial instance tha

tes if the spa

te

Valida

gainst a P

**Description**

returns a spa

between the spa

method was called and a spa

to the method as a parameter

could be either a primitive or multipart geometr

depending on the inputs.

Accepts a single parameter

object.

parameter is completely inside the spa

against which the method was called.

is and 0 if it is not

Called a

coordina

&

&

**orks WithW**

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

GEOMETRY

only

**OGC/Extended**

oGC

oGC

oGC

*d*) *ue*

*tin*

**( *con***

 *****Table 11-2.***

**Method**

STUnion

STWithin

STX

322

ChAPTer 11 WorKInG WITh SPATIAL DATA

s y

tial instance

oint’

*ve with fewer*

tial instance

returns the P

wn text of the spa

oint.

gainst a P

te

*ttempts to find a similar cur*

Called a

coordina

returns the well-kno

against which the method is called

returns the Z value of the spa

&

&

*t [https://link.springer.com/referenceworkentry/10.1](https://link.springer.com/referenceworkentry/10.1007/978-3-319-17885-1_298)*

GEOMETRY

only

GEOGRAPHY

GEOMETRY

GEOGRAPHY

GEOMETRY

*ve composed of line segments and a*

*.*

oGC

extended

extended

*trices can be found a*

*ttern ma*

*eucker algorithm takes a cur*

STY

ToString

Z

**A Douglas-P*

*points.*

***Details of the DE-9IM pa*

[*007%2F978-3-319-17885-1_298*](https://link.springer.com/referenceworkentry/10.1007/978-3-319-17885-1_298)

323

ChAPTer 11 WorKInG WITh SPATIAL DATA

Table [11-2](#index_split_002.html#p317) called out the IsValidDetailed, which has been implemented by Microsoft as an extended method. Ta[ble 11-3](#index_split_003.html#p333) details the error codes that can be thrown by the IsValidDetailed method, when a

spatial instance is not valid.

***Table 11-3.** IsValidDetailed Error Codes*

**Error Code Description**

24400

The instance is valid.

24401

The instance is not valid, but the reason is unknown.

24402

The instance is not valid, because a point is isolated, which is not valid for the object’s type.

24403

The instance is not valid, because some pair of polygon edges overlap.

24404

The instance is not valid because a polygon ring intersects itself or

another ring.

24405

The instance is not valid, because a polygon ring intersects itself or

another ring, and the ring number cannot be returned.

24406

The instance is not valid, because a curve degenerates to a point.

24407

The instance is not valid, because a polygon ring collapses to a line.

24408

The instance is not valid, because a polygon ring is not closed.

24409

The instance is not valid, because some portion of a polygon ring lies in the interior of a polygon.

24410

The instance is not valid, because a ring is the first ring in a polygon

but is not the exterior ring.

24411

The instance is not valid, because a ring lies outside the exterior ring of its polygon.

( *continued*)

324

ChAPTer 11 WorKInG WITh SPATIAL DATA

***Table 11-3.*** ( *continued*)

**Error Code Description**

24412

The instance is not valid, because the interior of a polygon is not

connected.

24413

The instance is not valid, because of two overlapping edges in a curve.

24414

The instance is not valid, because an edge of a curve overlaps an edge

of another curve.

24415

The instance is not valid, because a polygon has an invalid ring

structure.

24416

The instance is not valid, because in a curve, the edge is either a line or a degenerate arc with antipodal end points.

In addition to the methods detailed in Ta[bles 11-1](#index_split_002.html#p309) [and 11-2, ther](#index_split_002.html#p317)e are also a number of aggregation methods available when using the GEOMETRY

and GEOGRAPHY data types. These are detailed in Table [11-4.](#index_split_003.html#p334)

***Table 11-4.** Aggregation Methods*

**Method**

**Description**

CollectionAggregate

Creates a GeometryCollection from a set of spatial

objects

ConvexHullAggregate

returns the convex hull of a set of spatial objects

EnvelopeAggregate

returns a bounding spatial object for a set of spatial

objects

UnionAggregate

returns a union of a set of spatial objects

325

ChAPTer 11 WorKInG WITh SPATIAL DATA

Now that you are aware of the spatial methods that SQL Server

exposes, you can look at how they can be employed in practice, by using

the WideWorldImporters database. Imagine that we are responsible for

delivery routes, and we must plan a route in Alabama.

The first thing that we want to do is validate our data. Are all our

customers, with delivery locations marked as being in Alabama, actually

within the Alabama state border? The Sales.Customers table contains a

GEOGRAPHY column called DeliveryLocation, with each row containing a

Point object that maps their delivery address. We can use the STWithin

method against this column, passing in the Border column from the

Application.StateProvinces table, which maps the state border. This is

demonstrated in Listing [11-9.](#index_split_003.html#p335)

***Listing 11-9.*** Validating That All Addresses Are Within the Alabama State Border

DECLARE @StateBorder GEOGRAPHY = (

SELECT Border

FROM Application.StateProvinces

WHERE StateProvinceName = 'Alabama') ;

SELECT

Customer.CustomerName AS CustomerName

, City.CityName AS City

, Customer.DeliveryLocation.ToString() AS

DeliveryLocation

FROM SALES.Customers Customer

INNER JOIN Application.Cities City

ON City.CityID = Customer.DeliveryCityID

WHERE Customer.DeliveryLocation.STWithin(@StateBorder) = 1 ;

326

![](index-336_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

The results of this query (which are displayed in Figur[e 11-6) v](#index_split_003.html#p336)alidate that all 16 customers with Alabama delivery addresses actually reside in

Alabama.

***Figure 11-6.** Results of validating that all addresses are within the* *Alabama state border*

Next, we want to check the distance of each delivery location

from our depot and order the results by that distance. This will

help us plan the route. This technique is known as finding nearest

neighbors and is demonstrated in Listin[g 11-10](#index_split_003.html#p337), which calculates the distance and orders the results, by using the DeliveryLocation from

the Application.SystemParameters table, which marks the location of

WideWorldImporters.

327

ChAPTer 11 WorKInG WITh SPATIAL DATA

**Note** The WideWorldImporters depot is in California, but while the distances to the Alabama locations are vast, they still allow us to plan

the driver’s most sensible route.

***Listing 11-10.*** Finding Nearest Neighbors

DECLARE @StateBorder GEOGRAPHY = (

SELECT Border

FROM Application.StateProvinces

WHERE StateProvinceName = 'Alabama') ;

DECLARE @Office GEOGRAPHY = (

SELECT DeliveryLocation

FROM Application.SystemParameters) ;

DECLARE @MilesRatio INT = 0.000621371 ;

SELECT

Customer.CustomerName AS CustomerName

, City.CityName AS City

, Customer.DeliveryLocation.ToString() AS

DeliveryLocation

, Customer.DeliveryLocation.STDistance(@Office) * @

MilesRatio AS DeliveryDistanceMiles

FROM SALES.Customers Customer

INNER JOIN Application.Cities City

ON City.CityID = Customer.DeliveryCityID

WHERE Customer.DeliveryLocation.STWithin(@StateBorder) = 1

ORDER BY DeliveryDistanceMiles ;

328

![](index-338_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

The results of this query can be seen in Figure [11-7](#index_split_003.html#p338).

***Figure 11-7.** Results of finding nearest neighbors*

You will notice that in Listin[g 11-10](#index_split_003.html#p337), the distance is multiplied by 0.000621371\. This is because the distance is natively in meters. The unit of measurement of distance is associated with the SRID, so we can double

check this, by enhancing our query to expose the SRID, as shown in

Listin[g 11-11](#index_split_003.html#p339). This query exposes the STRid property of each spatial instance and uses this value to join to the sys.spatial_reference_systems system table, to expose the unit of measure column.

329

ChAPTer 11 WorKInG WITh SPATIAL DATA

***Listing 11-11.*** Exposing the SRID

DECLARE @StateBorder GEOGRAPHY = (

SELECT Border

FROM Application.StateProvinces

WHERE StateProvinceName = 'Alabama') ;

DECLARE @Office GEOGRAPHY = (

SELECT DeliveryLocation

FROM Application.SystemParameters) ;

DECLARE @MilesRatio INT = 0.000621371 ;

SELECT

Customer.CustomerName AS CustomerName

, City.CityName AS City

, Customer.DeliveryLocation.ToString() AS

DeliveryLocation

, Customer.DeliveryLocation.STDistance(@Office) *

@MilesRatio AS DeliveryDistanceMiles

, Customer.DeliveryLocation.STSrid AS SRID

, srid.unit_of_measure

FROM SALES.Customers Customer

INNER JOIN Application.Cities City

ON City.CityID = Customer.DeliveryCityID

INNER JOIN sys.spatial_reference_systems srid

ON srid.spatial_reference_id = Customer.

DeliveryLocation.STSrid

WHERE Customer.DeliveryLocation.STWithin(@StateBorder) = 1

ORDER BY DeliveryDistanceMiles ;

330

![](index-340_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

While we know that all deliveries are in the state of Alabama, we may

wish to discover how big our actual delivery area is. We can achieve this by creating an envelope and then calculating its area. This approach uses the EnvelopeAggregate extended method and the STArea method, as

demonstrated in Listing [11-12.](#index_split_003.html#p340)

***Listing 11-12.*** Calculating the Area of an Aggregate Envelope DECLARE @StateBorder GEOGRAPHY = (

SELECT Border

FROM Application.StateProvinces

WHERE StateProvinceName = 'Alabama') ;

SELECT

GEOGRAPHY::EnvelopeAggregate(DeliveryLocation).

STArea() AS AreaInSquareMetres

, GEOGRAPHY::EnvelopeAggregate(DeliveryLocation)

AS EnvelopeObject

FROM Sales.Customers

WHERE DeliveryLocation.STWithin(@StateBorder) = 1 ;

The results of this query can be seen in Figure [11-8](#index_split_003.html#p340).

***Figure 11-8.** Results of calculating the area of an aggregate*

*envelope*

331

![](index-341_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

The graphical results of the query in Listing [11-12](#index_split_003.html#p340) are illustrated in Figur[e 11-9.](#index_split_003.html#p341)

***Figure 11-9.** Graphical results of calculating the area of an aggregate* *envelope*

**Indexing Spatial Data**

The following sections provide an overview of spatial indexes, before

demonstrating how to create them.

**Understanding Spatial Indexes**

Spatial indexes are a special type of index, implemented in SQL Server,

that can improve the performance of certain queries against spatial data

types. Table [11-5 des](#index_split_003.html#p342)cribes the predicate patterns that can benefit from spatial indexes, when used in a WHERE or JOIN clause.

332

ChAPTer 11 WorKInG WITh SPATIAL DATA

***Table 11-5.** Queries That Can Benefit from Spatial Indexes*

**Data Type**

**Method**

**Operator**

GEOMETRY & GEOGRAPHY

STDistance

<

<=

GEOMETRY & GEOGRAPHY

STEquals

=

GEOMETRY & GEOGRAPHY

STIntersects

=

GEOMETRY only

STContains

=

GEOMETRY only

STOverlaps

=

GEOMETRY only

STTouches

=

GEOMETRY only

STWithin

=

Just as with traditional indexes, spatial indexes use a B-Tree structure

(see Chapter [5](#index_split_001.html#p169) for further information on B-Tree indexes), meaning that the spatial data must be represented in a linear order. To achieve this,

SQL Server decomposes space into a nested grid system, before building

an index.

The grid will have four layers. Each cell in the first (Level 1) grid will contain another (Level 2) grid, and so on. Each of the four grid layers can be given a separate density, with low density being defined as 4 × 4 cells, medium density as 8 × 8 cells, and high density as 16 × 16 cells. Each cell within the grid is numbered using a Hilbert space-filling curve algorithm.

**Tip** A full discussion of the hilbert space-filling curve is beyond the scope of this book, but further details can be found in many locations

online, including mathworld. [wolfram.com/HilbertCurve.html](http://wolfram.com/HilbertCurve.html).

333

ChAPTer 11 WorKInG WITh SPATIAL DATA

Once the grid system has been created, SQL Server reads the data from

the column row by row. For each row, it will associate the spatial object with each Level 1 cell that it touches. For each touched cell, it will then drop down to Level 2 and repeat the process. This then happens again for

Level 3 and Level 4, as required. This process is called tessellation. The output of the process is a set of touched cells, which can be stored in the index and subsequently used to calculate their spatial position, relative to other objects.

The tessellation process requires a bounding box, and this can behave

differently, depending on the tessellation system used. The tessellation

systems are data-type dependent and you will have the option of using

automatic coordinates for the bounding box or defining your own.

You may further configure the tessellation process, by defining how

many tessellation cells should be used. What this means is that you can

cap the maximum number of touched cells recorded for a single object.

It is worth noting, however, that this only affects Levels 2 through 4.

Level 1 will record as many cells as the object touches, regardless of your configuration.

**Creating Spatial Indexes**

To demonstrate the creation of a spatial index, we will create an index

on the Border column of the Application.StateProvinces table in the

WideWorldImporters database. We will use an auto grid system, and we will configure medium density for Levels 1 to 3, with high density for Level 4.

To create the index through SQL Server Management Studio, drill

through Databases ➤ WideWorldImporters ➤ Tables ➤ Application.

StateProvinces in Object Explorer. Then select New Index ➤ Spatial Index

from the context menu of the Indexes folder. This will cause the General

page of the New Index dialog bog to be invoked, as shown in Figure [11-10](#index_split_003.html#p344).

334

![](index-344_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

***Figure 11-10.** New Index dialog box—General page*

On this page of the dialog box, we have given the index a meaningful

name and used the Add button, to add the column that we wish to index. We will now progress to the Options page, which is illustrated in Figur[e 11- 11](#index_split_003.html#p345).

335

![](index-345_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

***Figure 11-11.** New Index dialog box—Options page*

The Options page will look familiar to anyone who has created a

traditional index. Table [11-6 details e](#index_split_003.html#p345)ach of the options available.

***Table 11-6.** Spatial Index Options*

**Option**

**Description**

Auto recompute

Specifies if statistics should be updated automatically when

Statistics

they are deemed out of date

Allow row Locks

Specifies if row locks can be acquired when accessing the index

Allow Page Locks Specifies if page locks can be acquired when accessing the index Maximum Degree has no effect for building primary spatial indexes, as this of Parallelism

operation is always single-threaded

( *continued*)

336

ChAPTer 11 WorKInG WITh SPATIAL DATA

***Table 11-6.*** ( *continued*)

**Option**

**Description**

Sort in TempDB

If specified, Sort in TempDB will cause the intermediate result

set to be stored in TempDB, as opposed to the user database.

This could mean that the index is built faster.

Fill Factor

Specifies a percentage of free space that will be left on each

index page at the lowest level of the index. The default is 0

(100% full), meaning that only enough space for a single row

will be left. Specifying a percentage lower than 100—for

example, specifying 70—will leave 30% free space, which can

reduce page splits, if there are likely to be frequent row inserts.

Pad Index

Applies a fill factor (see preceding) to the intermediate levels of

a B-Tree

On the Storage page, illustrated in Figure [11-12, y](#index_split_003.html#p347)ou can specify the filegroup that the index will be created on. Usually, it is best for indexes to be aligned with the same filegroup (or partition schema) as their table, for performance. From a maintenance perspective, it may be helpful to

store the index on a different filegroup when the table is partitioned. If you don’t, the index will have to be dropped before the table is repartitioned.

337

![](index-347_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

***Figure 11-12.** New Index dialog box—Storage page*

The Spatial page is shown in Figure [11-13\. T](#index_split_003.html#p348)his is where we can configure the spatial specific options of our index. In the General section of the page, we can choose our tessellation system—either automatic or

manual. If we select manual, the Grids area of the page will become active, and we can select the grid densities that we would like to use. If the manual geometry system is selected, the Bounding Box area of the page will also

become active, and we can specify our bounding box coordinates.

338

![](index-348_1.jpg)

ChAPTer 11 WorKInG WITh SPATIAL DATA

***Figure 11-13.** New Index dialog box—Spatial page*

**Tip** A geometry system must be used on a GEOMETRY column and

a geography system must be used on a GEOGRAPHY column.

Alternatively, we could use T-SQL to create the index. The script in

Listin[g 11-13](#index_split_003.html#p348) will create the same index as discussed previously.

***Listing 11-13.*** Creating a Spatial Index

USE WideWorldImporters

GO

CREATE SPATIAL INDEX [SI-Border]

ON Application.StateProvinces (Border)

USING GEOGRAPHY_GRID

WITH (GRIDS =(LEVEL_1 = MEDIUM,LEVEL_2 = MEDIUM,LEVEL_3 =

MEDIUM,LEVEL_4 = HIGH),

CELLS_PER_OBJECT = 16,

339

ChAPTer 11 WorKInG WITh SPATIAL DATA

PAD_INDEX = OFF,

STATISTICS_NORECOMPUTE = OFF,

SORT_IN_TEMPDB = OFF,

DROP_EXISTING = OFF,

ONLINE = OFF,

ALLOW_ROW_LOCKS = ON,

ALLOW_PAGE_LOCKS = ON, MAXDOP = 1)

ON [PRIMARY] ;

**Summary**

Spatial objects can be constructed by using various means. They can be

created using OGC standard methods, from either well-known text or

well- known binary, or they can be constructed by using a Microsoft

extended method from GML. SQL Server also supports passing

well-known text directly into a variable of a table’s cell.

A wealth of methods exist that allow developers to easily interact

with spatial data. Commonly used methods include STDistance, which

will return the distance between two geometries, and STWithin, which

will check if an instance resides within the same space as another. There are also aggregate methods that allow multiple instances to have union,

envelope, and convex hull applied to them as a group, as well as converting multiple instances to a single collection.

SQL Server provides spatial indexes, which can improve the performance

of certain types of queries. The index support for queries depends on the data type, the method in the WHERE or JOIN clause, and the arithmetic operator used.

Spatial indexes use B-Tree structures, but before the B-Tree is created,

a grid system of space is created, with four nested levels. The spatial

instances from the column are then read one by one into the grid system,

and their touched cells are numbered, using a Hilbert curve. This means

that spatial instances can be indexes in a linear fashion, but detailing their proximity to each other.

340

**CHAPTER 12**

**Working with**

**Hierarchical Data**

**and HierarchyID**

Modeling and working with data hierarchies have long been requirements

for SQL Server developers. Traditionally, hierarchical data has been

modeled using a self-join on a table, between two columns. One column

contains the ID of the hierarchical member, and the other, the ID of its

parent hierarchical member. Newer versions of SQL Server (2008 and

later versions) offer HierarchyID, however. HierarchyID is a data type

written in .NET and exposed in SQL Server. Using HierarchyID can offer

performance benefits and simplified code, compared to using a table

with a self-join. The data type exposes many methods that can be called

against the data, to allow developers to easily determine the ancestors and descendants of a hierarchical member, as well as determine other useful

information, such as the level of a specific hierarchical member within the hierarchy.

In this chapter, we will examine first the use cases for hierarchical data.

I will discuss how to model a traditional hierarchy, before explaining how we can remodel it using HierarchyID. Finally, we will look at the methods that are exposed against the HierarchyID data type.

© Peter A. Carter 2018

341

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8_12](#index_split_003.html#p350)

![](index-351_1.png)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

**Hierarchical Data Use Cases**

There are many data requirements that need a hierarchy to be maintained.

For example, consider an employee hierarchy, modeled from an organizational chart (see Figur[e 12-1). A h](#index_split_003.html#p351)uman resources department may have reporting requirements, to determine how many staff directly or indirectly report to a manager. It may also have to report on more complex requirements, such as how much revenue has been generated by staff reporting to each group head.

***Figure 12-1.** Sales department organization chart*

Another classic use case for hierarchical data is a bill of materials

(BoM). A BoM defines a hierarchy of parts that are required to produce

a product. For example, Figur[e 12-2](#index_split_003.html#p352) illustrates a simple BoM for a home computer. A computer manufacturer would have to maintain a hierarchy

of these parts for stock reporting.

342

![](index-352_1.png)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Figure 12-2.** Bill of materials*

343

![](index-353_1.png)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

In this chapter, we will be working with the example of a sales area

hierarchy. As illustrated in Figur[e 12-3](#index_split_003.html#p353), we will be modeling global sales regions. The hierarchy is ragged, meaning that there can be a varying

number of levels in each branch of the hierarchy.

***Figure 12-3.** Sales area hierarchy*

Maintaining a sales area hierarchy is important to many companies,

as it allows them to report on many factors, from revenue taken in each

region to the number of salespeople in each region to average revenue

per salesperson and region, etc. In this example, as well as in standard

sales regions, there are areas of the hierarchy that are aggregation areas.

This means that no salespeople exist, and sales are not directly taken for these regions. Instead, they are reporting levels, to allow lower levels of the hierarchy to be rolled up. These aggregation areas are highlighted in blue.

**Modeling Traditional Hierarchies**

Let’s look at how we might model a sales area hierarchy, using a traditional approach. To do this, consider the data in Ta[ble 12-1.](#index_split_003.html#p354)

344

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Table 12-1.** Sales Area Hierarchy*

**SalesAreaID ParentSalesAreaID SalesArea**

**CountOfSalesPeople SalesYTD**

**Name**

1

nUll

global Sales

nUll

nUll

2

1

europe

nUll

nUll

3

1

america

nUll

nUll

4

2

Uk

3

300,000

5

2

Western europe nUll

nUll

6

2

eastern europe nUll

nUll

7

3

Canada

4

350,000

8

3

USa

nUll

nUll

9

3

lataM

nUll

nUll

10

5

germany

3

150,000

11

5

France

2

100,000

12

6

hungary

1

50,000

13

6

Slovakia

2

80,000

14

8

eastern

4

140,000

15

8

Western

3

280,000

16

9

Brazil

1

100,000

17

9

argentina

2

70,000

18

14

new york

2

120,000

In the preceding table, the SalesAreaID column would be the primary

key of the table, and the ParentSalesAreaID column would be a foreign

key, which references the SalesAreaID column, creating what is known as a self-join.

345

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

**Note** the parentSalesareaiD column is NULL for the globalSales

area, because it does not have a parent. this is known as the root of

the hierarchy.

The SalesAreaName column describes the sales area, while the

CountOfSalesPeople and SalesYTD columns detail how each sales area

is performing. We will use these columns to explore how to work with

hierarchical data. You will notice that the CountOfSalesPeople and

SalesYTD columns are populated with NULL values for aggregation areas.

This is because they are “virtual” areas, where no salespeople are based.

This table can be created by using the script in Listing [12-1](#index_split_003.html#p355).

***Listing 12-1.*** Creating a Traditional Hierarchical Table

USE WideWorldImporters

GO

CREATE TABLE Sales.SalesAreaTraditionalHierarchy

(

SalesAreaID INT NOT NULL PRIMARY KEY,

ParentSalesAreaID INT NULL

REFERENCES Sales.Sales

AreaTraditionalHierarch

y(SalesAreaID),

SalesAreaName NVARCHAR(20) NOT NULL,

CountOfSalesPeople INT NULL,

SalesYTD MONEY NULL

) ;

INSERT INTO Sales.SalesAreaTraditionalHierarchy (

SalesAreaID

, ParentSalesAreaID

, SalesAreaName

346

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

, CountOfSalesPeople

, SalesYTD

)

VALUES

(1, NULL, 'GlobalSales', NULL, NULL),

(2, 1, 'Europe', NULL, NULL),

(3, 1, 'America', NULL, NULL),

(4, 2, 'UK', 3, 300000),

(5, 2, 'Western Europe', NULL, NULL),

(6, 2, 'Eastern Europe', NULL, NULL),

(7, 3, 'Canada', 4, 350000),

(8, 3, 'USA', NULL, NULL),

(9, 3, 'LATAM', NULL, NULL),

(10, 5, 'Germany', 3, 150000),

(11, 5, 'France', 2, 100000),

(12, 6, 'Hungary', 1, 50000),

(13, 6, 'Slovakia', 2, 80000),

(14, 8, 'Eastern', 4, 140000),

(15, 8, 'Western', 3, 280000),

(16, 9, 'Brazil', 1, 100000),

(17, 9, 'Agentina', 2, 70000),

(18, 14, 'New York', 2, 120000) ;

Let’s imagine that we must answer a business question, using this

traditional hierarchy. For example, we may have to answer the question,

“What is the total of SalesYTD for the America region?” To determine this, we would have to write a query that joins the table to itself and rolls up the SalesYTD column for all subregions under the America region.

The best way of achieving this is to use a recursive CTE (common

table expression). A CTE is a temporary result set that is defined within the context of a SELECT, UPDATE, INSERT, DELETE, or CREATE VIEW statement

and can only be referenced within that query. It is similar to using a

347

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

subquery as a derived table but has the benefit that it can be referenced multiple times and also reference itself. When a CTE references itself, it becomes recursive.

A CTE is declared using a WITH statement, which specifies the name of

the CTE, followed by the column list that will be returned by the CTE, in parentheses. The AS keyword is then used, followed by the body of the CTE, again within parentheses. This is shown, for our example, in Listing [12-2.](#index_split_003.html#p357)

**Tip** the queries in l[istings 12-2 to](#index_split_003.html#p357) [12-14 are not meant to be run](#index_split_003.html#p376)

separately. l[isting 12-15 brings them all together](#index_split_003.html#p377), into a useable script.

***Listing 12-2.*** CTE Structure

; WITH AreaHierarchy (SalesAreaID, SalesYTD, ParentSalesAreaID)

AS

(

[Body of CTE]

)

The definition of a recursive CTE has two queries, joined with a UNION

ALL clause. The first query is known as the anchor query and defines the

initial result set. The second query is known as the recursive query and

references the CTE. The first level of recursion will join to the anchor

query, and subsequent levels of recursion will join to the level of recursion immediately above them.

Therefore, in our example, the anchor query will have to return the

SalesAreaID of the America sales area. Because all queries joined with

UNION ALL must contain the same number of columns, our anchor query

must also return the ParentSalesAreaID and SalesYTD columns, even

though these values will be NULL.

348

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

The query in Listing [12-3 sho](#index_split_003.html#p358)ws the required anchor query.

***Listing 12-3.*** Anchor Query

SELECT

SalesAreaID

, SalesYTD

, ParentSalesAreaID

FROM Sales.SalesAreaTraditionalHierarchy RootLevel

WHERE SalesAreaName = 'America'

The recursive query will return the same column list as the anchor

query but will include a JOIN clause, which joins the ParentSalesAreaID

column in the recursive query to the SalesAreaID column of the CTE, as

demonstrated in Listing [12-4.](#index_split_003.html#p358)

***Listing 12-4.*** Recursive Query

SELECT

Area.SalesAreaID

, Area.SalesYTD

, Area.ParentSalesAreaID

FROM Sales.SalesAreaTraditionalHierarchy Area

INNER JOIN AreaHierarchy

ON Area.ParentSalesAreaID = AreaHierarchy.SalesAreaID

Following the declaration of this CTE, we will be able to run a SELECT

statement, which rolls up the SalesYTD, to return the total of SalesYTD

for the whole of America. The script in Listin[g 12-5 br](#index_split_003.html#p359)ings all these components together.

349

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Listing 12-5.*** Bringing It All Together

WITH AreaHierarchy (SalesAreaID, SalesYTD, ParentSalesAreaID)

AS

(

SELECT

SalesAreaID

, SalesYTD

, ParentSalesAreaID

FROM Sales.SalesAreaTraditionalHierarchy RootLevel

WHERE SalesAreaName = 'America'

UNION ALL

SELECT

Area.SalesAreaID

, Area.SalesYTD

, Area.ParentSalesAreaID

FROM Sales.SalesAreaTraditionalHierarchy Area

INNER JOIN AreaHierarchy

ON Area.ParentSalesAreaID = AreaHierarchy.SalesAreaID

)

SELECT SUM(SalesYTD)

FROM AreaHierarchy ;

**Modeling Hierarchies with HierarchyID**

When modeling hierarchical data using HierarchyID, there is no need

to perform a self-join against a table. Therefore, instead of having a

column that references its parent area’s primary key, we will instead

have a HierarchyID column, which defines each area’s position within

the hierarchy. To explain this concept further, let’s consider the data in Table [12-2](#index_split_003.html#p360).

350

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Table 12-2.** Sales Area Hierarchy with HierarchyID*

**SalesAreaID SalesAreaHierarchy SalesAreaName CountOfSalesPeople SalesYTD**

1

/

globalSales

nUll

nUll

2

/1/

europe

nUll

nUll

3

/2/

america

nUll

nUll

4

/1/1/

Uk

3

300,000

5

/1/2/

Western europe nUll

nUll

6

/1/3/

eastern europe nUll

nUll

7

/2/1/

Canada

4

350,000

8

/2/2/

USa

nUll

nUll

9

/2/3/

lataM

nUll

nUll

10

/1/2/1/

germany

3

150,000

11

/1/2/2/

France

2

100,000

12

/1/3/1/

hungary

1

50,000

13

/1/3/2/

Slovakia

2

80,000

14

/2/2/1/

eastern

4

140,000

15

/2/2/2/

Western

3

280,000

16

/2/3/1/

Brazil

1

100,000

17

/2/3/2/

argentina

2

70,000

18

/2/2/1/1/

new york

2

120,000

In Table [12-2, y](#index_split_003.html#p360)ou will notice that the hierarchy is represented by the format /[Node]/[Child Node]/[GrandchildNode]/, in which a row

containing just / is the root of the hierarchy. For example, we can see that the value /1/2/1/ for Germany tells us that Germany is a child of /1/2/

351

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

(Western Europe), which in turn is a child of /1/ (Europe). /1/ is the child of / (Global Sales), which is the root of the hierarchy.

We can create and populate this table by using the script in Listing [12-6](#index_split_003.html#p361).

***Listing 12-6.*** Create Table with HierarchyID

USE WideWorldImporters

GO

CREATE TABLE Sales.SalesAreaHierarchyID

(

SalesAreaID INT NOT

NULL PRIMARY KEY,

SalesAreaHierarchy HIERARCHYID NOT NULL,

SalesAreaName NVARCHAR(20) NOT NULL,

CountOfSalesPeople INT NULL,

SalesYTD MONEY NULL

) ;

INSERT INTO Sales.SalesAreaHierarchyID (

SalesAreaID

, SalesAreaHierarchy

, SalesAreaName

, CountOfSalesPeople

, SalesYTD

)

VALUES

(1, '/', 'GlobalSales', NULL, NULL),

(2, '/1/', 'Europe', NULL, NULL),

(3, '/2/', 'America', NULL, NULL),

(4, '/1/1/', 'UK', 3, 300000),

(5, '/1/2/', 'Western Europe', NULL, NULL),

(6, '/1/3/', 'Eastern Europe', NULL, NULL),

352

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

(7, '/2/1/', 'Canada', 4, 350000),

(8, '/2/2/', 'USA', NULL, NULL),

(9, '/2/3/', 'LATAM', NULL, NULL),

(10, '/1/2/1/', 'Germany', 3, 150000),

(11, '/1/2/2/', 'France', 2, 100000),

(12, '/1/3/1/', 'Hungary', 1, 50000),

(13, '/1/3/2/', 'Slovakia', 2, 80000),

(14, '/2/2/1/', 'Eastern', 4, 140000),

(15, '/2/2/2/', 'Western', 3, 280000),

(16, '/2/3/1/', 'Brazil', 1, 100000),

(17, '/2/3/2/', 'Agentina', 2, 70000),

(18, '/2/2/1/1/', 'New York', 2, 120000) ;

Even though we have inserted human-readable strings into the

SalesAreaHierarchyID column, SQL Server converts these strings and

stores them as hexadecimal values. This makes the column extremely

compact and efficient. The size of the HierarchyID column and that of

the INT column used for the ParentSalesAreaID column in the traditional

hierarchy can be compared using the query in Listin[g 12-7.](#index_split_003.html#p362)

***Listing 12-7.*** Comparing the Size of a Traditional Hierarchy to HierarchyID

USE WideWorldImporters

GO

SELECT

SUM(DATALENGTH(salesareahierarchy)) AS SizeOf

HierarchyID

, SUM(DATALENGTH(parentsalesareaid)) AS SizeOf

Traditional

FROM Sales.SalesAreaHierarchyID SalesAreaHierarchy

353

![](index-363_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

INNER JOIN sales.SalesAreaTraditionalHierarchy SalesAreaTraditional

ON SalesAreaHierarchy.SalesAreaID = SalesArea

Traditional.SalesAreaID ;

The results of this query are shown in Figure [12-4](#index_split_003.html#p363). You can see that the HierarchyID column is less than half the size of the INT column used for

the ParentSalesAreaID.

***Figure 12-4.** Results of size comparison*

**Note** in Cha[pter 1,](#index_split_000.html#p13) you learned that it is important to use the correct data type, and if i had chosen to use a SMALLINT for the

parentSalesareaiD column, the two columns would be about the

same size. this is, however, a minor example, provided for the

purpose of explaining HierarchyID, but if you are implementing

hierarchies on a large scale, this example is a fair representation.

If we run a normal SELECT statement against the SalesAreaHierarchyID

table, we can see the hexadecimal values in their raw form. For example,

consider the query in Listing [12-8](#index_split_003.html#p364).

354

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Listing 12-8.*** SELECT Statement Against HierarchyID Column USE WideWorldImporters

GO

SELECT

SalesAreaName

, SalesAreaHierarchy

FROM Sales.SalesAreaHierarchyID ;

The results of this query are displayed in Figure [12-5](#index_split_003.html#p365). You will notice that the contents of the SalesAreaHierarchy column are returned as

hexadecimal values, instead of human-readable strings. In order to view

the human-readable strings that we entered, we must use the ToString()

method, which is discussed in the “Working with HierarchyID Methods”

section of this chapter.

355

![](index-365_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Figure 12-5.** Results of SELECT statement against HierarchyID*

*column*

**HierarchyID Methods**

A number of methods are exposed against the HierarchyID data type,

allowing developers to quickly write efficient code when working with

hierarchies. Table [12-3](#index_split_003.html#p366) details these methods.

356

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Table 12-3.** Methods Exposed Against the HierarchyID Data Type*

**Method**

**Description**

GetAncestor

returns the ancestor of a hierarchy node. accepts a

parameter that defines how many levels up the hierarchy

the ancestor should be returned from. For example,

GetAncestor(1) will return the node’s parent, while

GetAncestor(2) will return the node’s grandparent.

GetDescendant

returns a child node iD for a given node in the hierarchy.

the GetDescendant() method is generally used in the

creation of two nodes. therefore, the method accepts two

parameters, both of type HierarchyID. the generated

node will sit between the two nodes specified.

GetLevel

returns the hierarchical level of the node

GetRoot

a static method that returns the root level of a hierarchy

IsDescendantOf

the IsDescendantOf() method accepts a single

parameter, of type HierarchyID, and returns 1 if a given

node is a descendant of the node passed as a parameter.

Parse

parses the string representation of a node, which is

passed as a parameter, to ensure it is valid. if valid, it

returns the hexadecimal representation. if invalid, it will

throw an error.

Read

reads the binary representation of SqlHierarchyId

from the Binaryreader and sets the SqlHierarchyId

object to that value. the Read() method can only be

called from SQlClr. it cannot be called from t-SQl. When

using t-SQl, you should use CAST or CONVERT instead.

( *continued*)

357

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Table 12-3.*** ( *continued*)

**Method**

**Description**

GetReparentedValue Used to move a node to a new parent. accepts two

parameters, the first being the original parent and the

second being the new parent

ToString

returns a string-formatted representation of a node within

the hierarchy

Write

Writes out a binary representation of SqlHierarchyId

to the BinaryWriter. For use with SQlClr only. When using

t-SQl, use CAST or CONVERT instead.

**Tip** HierarchyID methods are case-sensitive. For example, calling

tostring() will throw an error; calling ToString() will succeed.

**Working with HierarchyID Methods**

The following sections describe how to use the methods exposed against

the HierarchyID data type.

**Using ToString( )**

If you run a SELECT statement against a column with the HierarchyID data

type, the value returned will be a hexadecimal representation of the node.

To see a textual representation of the node, you must use the ToString()

method. For example, consider the query in Listin[g 12-9.](#index_split_003.html#p368)

358

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Listing 12-9.*** Using ToString()

USE WideWorldImporters

GO

SELECT

SalesAreaName

, SalesAreaHierarchy

, SalesAreaHierarchy.ToString() AS SalesArea

HierarchyString

FROM Sales.SalesAreaHierarchyID ;

The results of this query are displayed in Figure [12-6](#index_split_003.html#p369). You will see that the column becomes human-readable, once the ToString() method is

called against it.

359

![](index-369_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Figure 12-6.** Results of using ToString()*

**Using Parse( )**

The Parse() method is called implicitly when a string representation of

a node is inserted into a HierarchyID column. Essentially, the Parse()

method performs the reverse function of the ToString() method. It

attempts to convert a string formatted representation to the HierarchyID

representation. If it fails, an error is thrown. For example, consider the script in Listing [12-10.](#index_split_003.html#p370)

360

![](index-370_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Listing 12-10.*** Using the Parse() Method

--Returns Hexidecimal Representation Of Node

SELECT HierarchyID::Parse('/1/1/2/2/') ;

--Throws An Error Because Trailing / Is Missing

SELECT HierarchyID::Parse('/1/1/2/2') ;

The Results tab displayed by running this script can be seen in

Figur[e 12-7\. W](#index_split_003.html#p370)hile the first query displays the expected result, the second query returns no results.

***Figure 12-7.** Using Parse() Results tab*

361

![](index-371_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

Checking the Messages tab, displayed in Figure [12-8, w](#index_split_003.html#p371)ill detail the error thrown by the .NET framework.

***Figure 12-8.** Error thrown by .NET framework*

**Using GetRoot( )**

The GetRoot() method will return the root node of a hierarchy, as

demonstrated in Listing [12-11.](#index_split_003.html#p371)

***Listing 12-11.*** Using GetRoot()

USE WideWorldImporters

GO

SELECT

SalesAreaName

, SalesAreaHierarchy.ToString()

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaHierarchy = HierarchyID::GetRoot() ;

The results of this query can be viewed in Figure [12-9.](#index_split_003.html#p372)

362

![](index-372_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Figure 12-9.** Results of using GetRoot()*

**Using GetLevel( )**

The GetLevel() method allows you to determine at what level of the

hierarchy a particular node resides. For example, the query in Listin[g 12-12](#index_split_003.html#p372)

will return all nodes that reside on the bottom level of the hierarchy. In our case, this is just New York.

The subquery will determine the maximum level within the hierarchy,

and the outer query will return all sales areas that are at that level.

***Listing 12-12.*** Using GetLevel()

USE WideWorldImporters

GO

SELECT

SalesAreaName

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaHierarchy.GetLevel() =

(

SELECT

MAX(SalesAreaHierarchy.GetLevel())

FROM Sales.SalesAreaHierarchyID

) ;

363

![](index-373_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

The results of this query can be found in Figur[e 12-10](#index_split_003.html#p373).

***Figure 12-10.** Results of using GetLevel()*

**Read( ) and Write( )**

Because this chapter focuses on how to use HierarchyID within your

T-SQL code and the Read() and Write() methods are only applicable

to using SQLCLR (the technology that allows for managed objects to be

created inside SQL Server), a full description of the Read() and Write()

methods is beyond the scope of this book.

**Using GetDescendant( )**

Of course, a developer could insert a new node into the hierarchy, between existing nodes, but the GetDescendant() method helps a developer do

this pragmatically. The method accepts two parameters, both of which can

be NULL and represent existing children. The method will then generate a

node value, using the following rules:

• If the parent is NULL, then a NULL value will be returned.

• If the parent is not NULL, and both parameters are NULL,

the first child of the parent will be returned.

364

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

• If the parent and first parameter are not NULL, but the

second parameter is NULL, a child of parent greater than

the first parameter will be returned.

• If the parent and second parameter are not NULL but the

first parameter is NULL, a child of parent smaller than

the second parameter will be returned.

• If parent and both parameters are not NULL, a child of

parent between the two parameters will be returned.

• If the first parameter is not NULL and not a child of the

parent, an error is thrown.

• If the second parameter is not NULL and not a child of

the parent, an error is thrown.

• If the first parameter is greater than or equal to the

second parameter, an error is thrown.

For example, imagine that we want to create with the parent of America

a new sales area called Spain. We want the node value to be between

Canada and USA. We could achieve this with the query in Listing [12-13.](#index_split_003.html#p374)

**Tip** obviously, Spain should be included under Western europe, not under america. Don’t worry, this is a deliberate error, which we will

resolve in the “Using GetReparentedValue” section of this chapter.

***Listing 12-13.*** Generating a New Hierarchy Node

USE WideWorldImporters

GO

SELECT NewNode.ToString()

FROM

365

![](index-375_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

(

SELECT

SalesAreaHierarchy.GetDescendant(0x6AC0,0x6B40) AS

NewNode

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaName = 'America'

) NewNode ;

The results produced by this query are illustrated in Figure [12-11](#index_split_003.html#p375).

You will see that the lowest level contains a period (1.1). This is because Canada has a value of 1 and USA has a value of 2\. Therefore, to generate a node value between the two, an integer cannot be used. This guarantees

that a new node can always be inserted between two existing nodes.

***Figure 12-11.** Results of using GetDescendant()*

To programmatically add the Spain sales area to the hierarchy,

we could use the query in Listing [12-14.](#index_split_003.html#p376)

366

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Listing 12-14.*** Insert a New Node into the Hierarchy

USE WideWorldImporters

GO

INSERT INTO Sales.SalesAreaHierarchyID

(

SalesAreaID

, SalesAreaHierarchy

, SalesAreaName

, CountOfSalesPeople

, SalesYTD

)

SELECT

(SELECT MAX(SalesAreaID) + 1 FROM Sales.SalesArea

HierarchyID)

, SalesAreaHierarchy.GetDescendant

(0x6AC0,0x6B40)

, 'Spain'

, 2

, 200000

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaName = 'America' ;

**Using GetReparentedValue( )**

As you probably noticed in the “Using GetDescendant()” section, the

Spain sales area was incorrectly created under the America aggregation

area, as opposed to the Western Europe aggregation area. We can resolve

this issue by using the GetReparentedValue() method.

Consider the script in Listing [12-15\. F](#index_split_003.html#p377)irst, we declare two variables, with the type HierarchyID. These will be passed as parameters into the

GetReparentedValue() method. The @America variable is populated

367

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

with the sales area hierarchy node pertaining to the original parent sales area, and the @WesternEurope variable is populated with the sales area

hierarchy node pertaining to the target parent sales area.

***Listing 12-15.*** Use GetReparentedValue()

USE WideWorldImporters

GO

DECLARE @America HIERARCHYID =

(

SELECT SalesAreaHierarchy

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaName = 'America'

) ;

DECLARE @WesternEurope HIERARCHYID =

(

SELECT SalesAreaHierarchy

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaName = 'Western Europe'

) ;

SELECT Area.ToString()

FROM

(

SELECT SalesAreaHierarchy.GetReparentedValue(@America,

@WesternEurope) AS Area

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaHierarchy = 0x6B16

) NewNodePath ;

368

![](index-378_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

The results of this script can be seen in Figur[e 12-12\. Y](#index_split_003.html#p378)ou will notice that the leaf node value has remained the same, while the path (parent

nodes) have been changed, so that the node sits under the Western Europe

aggregation area.

***Figure 12-12.** Results of using GetReparentedValue()*

We could update the ancestry of the Spain sales area, in the Sales.

SalesAreaHierarchyID table, by using the script in Listin[g 12-16](#index_split_003.html#p378).

***Listing 12-16.*** Updating the Ancestry of the Spain Sales Area USE WideWorldImporters

GO

DECLARE @America HIERARCHYID =

(

SELECT SalesAreaHierarchy

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaName = 'America'

) ;

DECLARE @WesternEurope HIERARCHYID =

(

SELECT SalesAreaHierarchy

369

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaName = 'Western Europe'

) ;

UPDATE Sales.SalesAreaHierarchyID

SET SalesAreaHierarchy =

(

SELECT SalesAreaHierarchy.GetReparentedValue(@America,

@WesternEurope)

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaHierarchy = 0x6B16

)

WHERE SalesAreaHierarchy = 0x6B16 ;

**Using GetAncestor( )**

The GetAncestor() method can be used to return the ancestor node

of a given hierarchical node at the number of levels based on an input

parameter to the method. For example, consider the query in

Listin[g 12- 17](#index_split_003.html#p380). This query will return the grandparent of Spain, the parent of Spain, and Spain itself, by passing different parameters into the

GetAncestor() method.

**Caution** For this query to work as expected, you first must have

run the previous examples in the chapter. Specifically, the insert

and update queries in the “Using GetDescendant()” and “Using

GetReparentedValue()” sections, as well as listing [12-2](#index_split_003.html#p357), which creates and populates the table.

370

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Listing 12-17.*** Using GetAncestor()

USE WideWorldImporters

GO

SELECT

CurrentNode.SalesAreaName AS SalesArea

, ParentNode.SalesAreaName AS ParentSalesArea

, GrandParentNode.SalesAreaName AS GrandParentSalesArea

FROM Sales.SalesAreaHierarchyID Base

INNER JOIN Sales.SalesAreaHierarchyID CurrentNode

ON CurrentNode.SalesAreaHierarchy = Base.

SalesAreaHierarchy.GetAncestor(0)

INNER JOIN Sales.SalesAreaHierarchyID ParentNode

ON ParentNode.SalesAreaHierarchy = Base.

SalesAreaHierarchy.GetAncestor(1)

INNER JOIN Sales.SalesAreaHierarchyID GrandParentNode

ON GrandParentNode.SalesAreaHierarchy = Base.

SalesAreaHierarchy.GetAncestor(2)

WHERE Base.SalesAreaName = 'Spain' ;

**Using IsDescendantOf( )**

The IsDescendantOf() method evaluates if a node within the hierarchy

is a descendant (at any level) of a node that is passed to it as a parameter.

It is this method that we can use to rewrite the query in Listing [12-5](#index_split_003.html#p359), which rolled up the SalesYTD for all sales areas under the America aggregation

area, using a traditional hierarchy.

You will remember, that when using a traditional hierarchy, we had

to implement a recursive CTE, which rolled up the SalesYTD column, for

all hierarchical levels, which are descendants of America. When using

a HierarchyID column to maintain the hierarchy, however, our code is

greatly simplified, as demonstrated in Listin[g 12-18](#index_split_003.html#p381).

371

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Listing 12-18.*** Using IsDescendantOf()

USE WideWorldImporters

GO

SELECT

SUM(SalesYTD) AS TotalSalesYTD

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaHierarchy.IsDescendantOf(0x68) = 1 ;

Instead of a recursive CTE, the functionally equivalent code is a simple

query with a WHERE clause that filters hierarchical nodes, based on whether they are descendants of the America aggregation area. Listin[g 12-19](#index_split_003.html#p381) shows a more complex example. Here, we are parameterizing the sales area and

calculating not only the total SalesYTD but also the TotalSalesPeople and the regions AverageSalesPerSalesPerson.

***Listing 12-19.*** Parameterizing IsDescendantOf() Queries

USE WideWorldImporters

GO

DECLARE @Region NVARCHAR(20) = 'America' ;

DECLARE @RegionHierarchy HIERARCHYID =

(

SELECT SalesAreaHierarchy

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaName = @Region

) ;

SELECT

SUM(SalesYTD) AS TotalSalesYTD

, SUM(CountOfSalesPeople) AS TotalSalesPeople

372

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

, SUM(SalesYTD) / SUM(CountOfSalesPeople) AS

AverageSalesPerSalesPerson

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaHierarchy.IsDescendantOf(@RegionHierarchy) = 1 ;

You can clearly see how the Region parameter could be passed into a

stored procedure, so that this code could be accessed by an application.

Let us now add a SalesAreaName to the SELECT list, and group by this

column, as demonstrated in Listin[g 12-20](#index_split_003.html#p382).

***Listing 12-20.*** Adding a GROUP BY

USE WideWorldImporters

GO

DECLARE @Region NVARCHAR(20) = 'America' ;

DECLARE @RegionHierarchy HIERARCHYID =

(

SELECT SalesAreaHierarchy

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaName = @Region

) ;

SELECT

SUM(SalesYTD) AS TotalSalesYTD

, SUM(CountOfSalesPeople) AS TotalSalesPeople

, SUM(SalesYTD) / SUM(CountOfSalesPeople) AS

AverageSalesPerSalesPerson

, SalesAreaName

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaHierarchy.IsDescendantOf(@RegionHierarchy) = 1

GROUP BY SalesAreaName ;

373

![](index-383_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

The results of this query are illustrated in Figur[e 12-13](#index_split_003.html#p383).

***Figure 12-13.** Results of using IsDescendantOf() with GROUP BY*

The interesting behavior exposed by the results of this query is that

America is included. This is because HierarchyID regards America as

a descendant of itself. This does not create an issue for us, because the aggregations are not pre-calculated. However, in some instances, you may

have to exclude America from the result set. This can easily be achieved by adding an additional filter to the WHERE clause, as demonstrated in

Listin[g 12-21](#index_split_003.html#p383).

***Listing 12-21.*** Filtering the Current Node from Descendants USE WideWorldImporters

GO

DECLARE @Region NVARCHAR(20) = 'America' ;

DECLARE @RegionHierarchy HIERARCHYID =

(

374

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

SELECT SalesAreaHierarchy

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaName = @Region

) ;

SELECT

SUM(SalesYTD) AS TotalSalesYTD

, SUM(CountOfSalesPeople) AS TotalSalesPeople

, SUM(SalesYTD) / SUM(CountOfSalesPeople) AS

AverageSalesPerSalesPerson

, SalesAreaName

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaHierarchy.IsDescendantOf(@RegionHierarchy) = 1

AND SalesAreaHierarchy <> @RegionHierarchy

GROUP BY SalesAreaName ;

This query filters the result set to exclude the sales area, the

hierarchical node of which is equal to the hierarchical node that is being passed to the IsDescendantOf() method. This technique allows us to use

the WITH ROLLUP clause on the GROUP BY, in conjunction with wrapping

SalesAreaName in an ISNULL() function, to produce a subtotal row for the

whole of America. This is demonstrated in Listing [12-22.](#index_split_003.html#p384)

***Listing 12-22.*** Producing a Total Row for America

USE WideWorldImporters

GO

DECLARE @Region NVARCHAR(20) = 'America' ;

DECLARE @RegionHierarchy HIERARCHYID =

(

SELECT SalesAreaHierarchy

FROM Sales.SalesAreaHierarchyID

375

![](index-385_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

WHERE SalesAreaName = @Region

) ;

SELECT

SUM(SalesYTD) AS TotalSalesYTD

, SUM(CountOfSalesPeople) AS TotalSalesPeople

, SUM(SalesYTD) / SUM(CountOfSalesPeople) AS

AverageSalesPerSalesPerson

, ISNULL(SalesAreaName, @Region)

FROM Sales.SalesAreaHierarchyID

WHERE SalesAreaHierarchy.IsDescendantOf(@RegionHierarchy) = 1

AND SalesAreaHierarchy <> @RegionHierarchy

GROUP BY SalesAreaName WITH ROLLUP ;

The results of this query can be seen in Figure [12-14.](#index_split_003.html#p385)

***Figure 12-14.** Results of adding a total row*

376

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

**Indexing HierarchyID Columns**

There are no “special” index types that support HierarchyID, as there are for XML or geospatial data types. Instead, the performance of HierarchyID

columns can be improved by using traditional clustered and nonclustered

indexes. When creating indexes to support HierarchyID, there are two

strategies that can be employed, depending on the nature of the queries

that will use the indexes.

By default, creating an index on a HierarchyID column will create a

depth-first index. This means that descendants will be stored close to their parents. In our example, New York would be stored close to Eastern, which in turn would be stored close to USA, and so on. The script in Listing [12-23](#index_split_003.html#p386)

demonstrates how to create a clustered index on the SalesAreaHierarchy

column, which uses a depth-first approach.

**Caution** the script first drops the primary key on the SalesareaiD

column, which implicitly drops the clustered index on this column.

the primary key name, in this case, is system- generated, however.

therefore, to run this script, you must change the name of the

constraint, to reflect your own system.

***Listing 12-23.*** Creating a Depth-First Clustered Index

USE WideWorldImporters

GO

ALTER TABLE Sales.SalesAreaHierarchyID

DROP CONSTRAINT PK__SalesAre__DB0A1ED5D7B258FB ;

GO

CREATE CLUSTERED INDEX SalesAreaHierarchyDepthFirst

ON Sales.SalesAreaHierarchyID(SalesAreaHierarchy) ;

GO

377

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

We can see how SQL Server has organized this data,

by running the query in Listing [12-24\. T](#index_split_003.html#p387)his query uses the undocumented sys.physlocformatter() function to return the exact location of each

record, in the format FileID:PageID:SlotID.

***Listing 12-24.*** View Location of Rows

USE WideWorldImporters

GO

SELECT

SalesAreaName

, sys.fn_PhysLocFormatter(%%physloc%%) AS PhysicalLocation

FROM Sales.SalesAreaHierarchyID ;

This query returns the results shown in Figur[e 12-15](#index_split_003.html#p388). You can see that each node is stored under its parent. This is even true for Spain, despite us adding it after the other regions, which means its original location would have been the final used slot in the page.

378

![](index-388_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Figure 12-15.** Results of viewing row locations*

**Tip** the row’s physical location is likely to be different when you run the query yourself.

379

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

The other possible indexing strategy is a breadth-first technique.

Here, sibling nodes will be stored close to each other, instead of storing descendants close to each other. To implement a breadth- first indexing

strategy, we must add to our table an additional column that stores the

hierarchical level of each node. This column can be created and populated by using the script in Listing [12-25](#index_split_003.html#p389).

***Listing 12-25.*** Adding a Level Column to Support Breadth-First Indexing

USE WideWorldImporters

GO

ALTER TABLE Sales.SalesAreaHierarchyID ADD

SalesAreaLevel INT NULL ;

GO

UPDATE Sales.SalesAreaHierarchyID

SET SalesAreaLevel = SalesAreaHierarchy.GetLevel() ;

Using the script in Listing [12-26, w](#index_split_003.html#p390)e can now create a clustered index, which is first order by the hierarchical level of the node and then by the HierarchyID column. This will cause siblings to be stored close to one

another.

**Note** the script first drops the existing clustered index, because a table can only support a single clustered index.

380

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Listing 12-26.*** Creating a Breadth-First Index

USE WideWorldImporters

GO

DROP INDEX SalesAreaHierarchyDepthFirst ON Sales.

SalesAreaHierarchyID ;

GO

CREATE CLUSTERED INDEX SalesAreaHierarchyBredthFirst

ON Sales.SalesAreaHierarchyID(SalesAreaLevel,

SalesAreaHierarchy) ;

Listin[g 12-27](#index_split_003.html#p390) demonstrates how we can use the same technique as in Listin[g 12-24](#index_split_003.html#p387), to view the actual location of each node within the hierarchy.

This time, as well as returning the sales area name, we will also return the SalesAreaLevel for easy analysis.

***Listing 12-27.*** View Rows Location with a Breadth-First Strategy USE WideWorldImporters

GO

SELECT

SalesAreaName

, SalesAreaLevel

, sys.fn_PhysLocFormatter(%%physloc%%) AS PhysicalLocation

FROM Sales.SalesAreaHierarchyID ;

The results of this query can be found in Figur[e 12-16](#index_split_003.html#p391). You will notice that the order of rows has changed and that UK, Western Europe, Eastern

Europe, Canada, USA, and LATAM are now next to one another, as they are

all at Level 2 of the hierarchy.

381

![](index-391_1.jpg)

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

***Figure 12-16.** Results of viewing row locations in a breadth-first* *hierarchy*

**Summary**

HierarchyID is created as a .NET class and implemented as a data

type in SQL Server. Using HierarchyID over a traditional approach to

modeling hierarchies in SQL Server has the benefits both of reducing

code complexity and improving performance. The HierarchyID data type

exposes several methods that can be used by developers to easily navigate 382

Chapter 12 Working With hierarChiCal Data anD hierarChyiD

a hierarchy, insert new hierarchical nodes, or update existing nodes so that they sit under a new parent.

The two most commonly used methods, in my experience, are the

ToString() method, which allows a developer to format a hierarchical

node as a human-readable string representation, and IsDescendantOf(),

which performs an evaluation of hierarchical node lineage and returns 1

when a node is a descendant of an input parameter and 0 if it is not.

The Read() and Write() methods offer data type conversion

functionality to SQLCLR, but these are not implemented in T-SQL, as the

CAST and CONVERT functions can easily be used instead.

When indexing HierarchyID columns, either a depth-first strategy or a

breadth-first strategy can be applied. Depth-first is the default option and stores child nodes close to their parents. A breadth-first strategy requires an additional column in the table, which stores the node’s level with the hierarchy. This allows a multicolumn index to store sibling nodes close to one another. The indexing option that you choose should reflect the nature of the queries run against the HierarchyID column.

383

**Index**

**A**

NTEXT[, 12](#index_split_000.html#p24)

NVARCHAR[, 12](#index_split_000.html#p24)

Advanced data types

string storage

GEOGRAPHY[, 25](#index_split_000.html#p37)

sizes, [13](#index_split_000.html#p25)

GEOMETRY[, 25](#index_split_000.html#p37)

TEXT[, 12](#index_split_000.html#p24)

HIERARCHYID[, 25](#index_split_000.html#p37)

VARCHAR[, 11](#index_split_000.html#p23)

JSON[, 25](#index_split_000.html#p37)

Clustered indexes

XML, [25](#index_split_000.html#p37)

creating, [165–167](#index_split_001.html#p177)

Aggregation methods, [325](#index_split_003.html#p334)

description, [159](#index_split_001.html#p171)

inserts[, 164](#index_split_001.html#p176)

**B**

primary key, [162–163](#index_split_001.html#p174)

Bill of materials (B[oM), 342–343](#index_split_003.html#p351)

tables

Binary data types

B-Tree structure, [161–162](#index_split_001.html#p173)

BINARY, [14](#index_split_000.html#p26)

heap structure[, 160–161](#index_split_001.html#p172)

CONVERT function, [15](#index_split_000.html#p27)

updates, [164](#index_split_001.html#p176)

converting to character

Common table expression (CTE),

string, [17](#index_split_000.html#p29)

[347–348](#index_split_003.html#p356)

encrypted passwor[d, 15–16](#index_split_000.html#p27)

Configuration management

IMAGE, [14](#index_split_000.html#p26)

databas[e (CMDB), 197–198](#index_split_002.html#p208)

style options, [15](#index_split_000.html#p27)

CONVERT function, [7](#index_split_000.html#p19)

VARBINARY, [14](#index_split_000.html#p26)

**D**

**C**

Date and time data types

CAS[T function, 7,](#index_split_000.html#p19) [9–10](#index_split_000.html#p21)

casting[, 22](#index_split_000.html#p34)

Character data types

DATE[, 18](#index_split_000.html#p30)

CHAR[, 11](#index_split_000.html#p23)

DATETIME, [18](#index_split_000.html#p30)

NCHAR[, 11](#index_split_000.html#p23)

DATE[TIME2, 19](#index_split_000.html#p31)

© Peter A. Carter 2018

385

P. A. Carter, *SQL Server Advanced Data Types*,

[https://doi.org/10.1007/978-1-4842-3901-8](https://doi.org/10.1007/978-1-4842-3901-8)

Index

Date and time data types ( *cont.* )

**F**

DATETIMEOFFSET[, 19](#index_split_000.html#p31)

FOR JSON AUTO, SalesOrder

details[, 17,](#index_split_000.html#p29) [18](#index_split_000.html#p30)

automatic nesting

SMALLDATETIME[, 18](#index_split_000.html#p30)

adding order line details,

style options, [20–21](#index_split_000.html#p32)

[216–217](#index_split_002.html#p227)

TIME, [18](#index_split_000.html#p30)

nested joins, [218–220](#index_split_002.html#p229)

Document type definition (DTD),

subqueries to nest data,

[42,](#index_split_000.html#p54) [48](#index_split_000.html#p60)

[221–223](#index_split_002.html#p232)

column aliases[, 212–215](#index_split_002.html#p223)

**E**

NULL values[, 207–211](#index_split_002.html#p218)

exist() metho[d, 116–118](#index_split_001.html#p128)

return keys and dates[, 201–202](#index_split_002.html#p212)

Extensible Markup

return results[, 203](#index_split_002.html#p214)

Language (XML)

root nodes[, 204–207](#index_split_002.html#p215)

attribute-centric approach

FOR JSON PA[TH, 224–227](#index_split_002.html#p235)

aspect of, [36–37](#index_split_000.html#p48)

FOR XML AUTO, SalesOrder

generating, [31](#index_split_000.html#p43)

results, [78–85](#index_split_000.html#p90)

sales orders[, 30](#index_split_000.html#p42)

results of re-written query, [86–91](#index_split_001.html#p98)

[definition, 29](#index_split_000.html#p41)

rewriting query to nest data[, 86](#index_split_001.html#p98)

element-centric approach

root node, [85](#index_split_001.html#p97)

generating, [35](#index_split_000.html#p47)

FOR XML EXPLICIT, SalesOrder

sales orders[, 32–35](#index_split_000.html#p44)

NULL values[, 103](#index_split_001.html#p115)

SQL Server

Order Details[, 101–102](#index_split_001.html#p113)

data structures, [47](#index_split_000.html#p59)

UNION ALL[, 103–104](#index_split_001.html#p115)

FOR XML clause, [46](#index_split_000.html#p58)

results, [104](#index_split_001.html#p116)

JSON, [47](#index_split_000.html#p59)

FOR XML PATH, SalesOrder

XQuery Nodes

customized and complex

metho[d, 46](#index_split_000.html#p58)

structures[, 92](#index_split_001.html#p104)

OPENXML() function, [46](#index_split_000.html#p58)

hierarchy levels[, 98](#index_split_001.html#p110)

sales orders[, 46](#index_split_000.html#p58)

OrderDetails subquer[y, 99](#index_split_001.html#p111)

storing data, [46](#index_split_000.html#p58)

required format, [100](#index_split_001.html#p112)

well-formed ( *see* Well-formed

Root and <row> element, [98](#index_split_001.html#p110)

XML)

@ symb[ol, 92](#index_split_001.html#p104)

XSD schemas, [43–45](#index_split_000.html#p55)

XML output[, 92–98,](#index_split_001.html#p104) [100](#index_split_001.html#p112)

386

Index

FOR XML RAW,

HierarchyID

WideWorldImporters Sales

breadth-first indexing

Orders

addition[al column, 380](#index_split_003.html#p389)

ELEMENTS keywor[d, 59–68](#index_split_000.html#p71)

creating[, 380–381](#index_split_003.html#p389)

output[, 50](#index_split_000.html#p62)

viewing row locations,

quer[y, 49,](#index_split_000.html#p61) [51](#index_split_000.html#p63)

[381–382](#index_split_003.html#p390)

root node[, 54–58](#index_split_000.html#p66)

depth-first clustered index, [377](#index_split_003.html#p386)

<row> element[, 68–77](#index_split_000.html#p80)

GetAncest[or(), 370–371](#index_split_003.html#p379)

XML fragment[, 51–54](#index_split_000.html#p63)

GetDescendan[t(), 364–367](#index_split_003.html#p373)

GetLev[el(), 363–364](#index_split_003.html#p372)

**G**

GetReparentedValue(), [367–370](#index_split_003.html#p376)

GetRoot(), [362–363](#index_split_003.html#p371)

Geography Markup Language

IsDescendantOf(), [371–376](#index_split_003.html#p380)

(GML), [302](#index_split_002.html#p311)

methods, [356–358](#index_split_003.html#p365)

GetAncestor() method,

Parse(), [360–362](#index_split_003.html#p369)

[370–371](#index_split_003.html#p379)

Read() and Write(), [364](#index_split_003.html#p373)

GetDescendant() method,

ToString(), [358–360](#index_split_003.html#p367)

[364–367](#index_split_003.html#p373)

viewing row locations, [378–379](#index_split_003.html#p387)

GetLevel() method, [363–364](#index_split_003.html#p372)

GetReparentedValue() method,

**I**

[367–370](#index_split_003.html#p376)

GetRoot() metho[d, 362–363](#index_split_003.html#p371)

Indexing JSON data

Globally unique identifier (GUID),

ALTER TABLE script[, 272](#index_split_002.html#p283)

[162–163](#index_split_001.html#p174)

checking index performance,

[274–275](#index_split_002.html#p285)

computed column, [274](#index_split_002.html#p285)

**H**

JSON_VAL[UE(), 271](#index_split_002.html#p282)

Hierarchical data

performance baseline[, 272](#index_split_002.html#p283)

bill of materials (BoM),

StockItems table[, 272](#index_split_002.html#p283)

[342–343](#index_split_003.html#p351)

TIME STATISTICS, [271,](#index_split_002.html#p282) [273](#index_split_002.html#p284)

sales area ( *see* Sales area

WHERE clause, [273](#index_split_002.html#p284)

hierarchy)

IsDescendantOf() method,

sales department organization

[371–376](#index_split_003.html#p380)

chart[, 342](#index_split_003.html#p351)

IsValidDetailed error codes[, 324–325](#index_split_003.html#p333)

387

Index

**J, K, L**

modified ro[w, 268–269](#index_split_002.html#p279)

Tags array, [270](#index_split_002.html#p281)

JavaScript object notation (JSON)

updated recor[d, 268](#index_split_002.html#p279)

document

updating row, [267](#index_split_002.html#p278)

config as code[, 197–198](#index_split_002.html#p208)

updating value, [266–267](#index_split_002.html#p277)

log data, [199](#index_split_002.html#p210)

JSON_QUERY() function

name/value pairs[, 182,](#index_split_001.html#p193) [199](#index_split_002.html#p210)

JSON document,

normalized data[, 195–196](#index_split_002.html#p206)

[261–262](#index_split_002.html#p272)

n-Tier applications,

path variables and

Rest APIs, [195](#index_split_002.html#p206)

document context,

simple[, 182](#index_split_001.html#p193)

[264–265](#index_split_002.html#p275)

syntax[, 181](#index_split_001.html#p192)

SELECT list[, 263](#index_split_002.html#p274)

vehicle temperatures

WHERE clause[, 264](#index_split_002.html#p275)

full sensor data,

JSON_VALUE() function

[184–185](#index_split_001.html#p195)

JSON document[, 256](#index_split_002.html#p267)

root node, [186–188](#index_split_001.html#p197)

lax mode path

top[, 182–183](#index_split_001.html#p193)

expressions[, 260](#index_split_002.html#p271)

*vs.* XML, sales people[, 188](#index_split_001.html#p199)

NULL results, [255](#index_split_002.html#p266)

JSON data type

OPENJS[ON(), 254](#index_split_002.html#p265)

ISJSON() function

parameters, [254](#index_split_002.html#p265)

error[, 252](#index_split_002.html#p263)

path express[ion, 255](#index_split_002.html#p266)

filter results[, 253–254](#index_split_002.html#p264)

results[, 257](#index_split_002.html#p268)

IF statement, [251–252](#index_split_002.html#p262)

SELECT list[, 256–257](#index_split_002.html#p267)

incorrectly formatted

variable[, 260–261](#index_split_002.html#p271)

JSON, [252](#index_split_002.html#p263)

WHERE clause[, 258–259](#index_split_002.html#p269)

NVARCHAR(MAX)

columns, [251](#index_split_002.html#p262)

OPENJSON() function,

**M**

[253–254](#index_split_002.html#p264)

Miscellaneous data types

JSON_MODIFY() function

CURSOR[, 23](#index_split_000.html#p35)

array element, [268](#index_split_002.html#p279)

SQL VARIANT, [23](#index_split_000.html#p35)

deleting data, [269](#index_split_002.html#p280)

TABLE[, 23](#index_split_000.html#p35)

modified document,

TIMESTAMP, [23](#index_split_000.html#p35)

[270–271](#index_split_002.html#p281)

UNIQUEIDENTIFIER, [23–24](#index_split_000.html#p35)

388

Index

**N**

explicit schema

NVARCHAR(MA[X), 238–239](#index_split_002.html#p249)

Numeric data types

OUTER APPLY, [241–242](#index_split_002.html#p252)

BIGINT[, 2](#index_split_000.html#p14)

results[, 238](#index_split_002.html#p249)

BIT, [2](#index_split_000.html#p14)

returning JSON data[, 239–240](#index_split_002.html#p250)

CAST function, [7,](#index_split_000.html#p19) [9–10](#index_split_000.html#p21)

T-SQL data types[, 237](#index_split_002.html#p248)

CONVERT function, [7](#index_split_000.html#p19)

path expressions

DECIMAL, [3](#index_split_000.html#p15)

array element[, 244–245](#index_split_002.html#p255)

FLOAT[, 5](#index_split_000.html#p17)

results[, 243](#index_split_002.html#p254)

INT, [2](#index_split_000.html#p14)

sales orders, root node,

MONEY[, 6](#index_split_000.html#p18)

[242–243](#index_split_002.html#p253)

NUMERIC, [4](#index_split_000.html#p16)

REAL, [5](#index_split_000.html#p17)

SMALLINT, [2](#index_split_000.html#p14)

**P**

SMALLMONEY[, 6](#index_split_000.html#p18)

Parse() method, [360–362](#index_split_003.html#p369)

style options, [7–8](#index_split_000.html#p19)

Primary XML indexes

TINYINT, [2](#index_split_000.html#p14)

Add column dialog box, [169](#index_split_001.html#p181)

creating[, 171](#index_split_001.html#p183)

**O**

New Index dialog box[, 168](#index_split_001.html#p180)

Node table[, 167](#index_split_001.html#p179)

Open Geospatial Consortium

options, [169–171](#index_split_001.html#p181)

(OGC[), 296,](#index_split_002.html#p306) [300](#index_split_002.html#p309)

scalar/XML subtrees, [167](#index_split_001.html#p179)

OPENJSON() function

default schema

Application.Person table,

**Q**

[229,](#index_split_002.html#p240) [230](#index_split_002.html#p241)

Querying JSON data, *see* JSON

data types, [231–232](#index_split_002.html#p242)

data type

dynamic PIVOT,

query() method, [121–123](#index_split_001.html#p133)

[235–236](#index_split_002.html#p246)

OUTER APPLY operator,

[232–233](#index_split_002.html#p243)

**R**

PIVOT operator[, 233–235](#index_split_002.html#p244)

Radio frequency identification

shredding, single JSON

(RFID), [199](#index_split_002.html#p210)

document[, 230–231](#index_split_002.html#p241)

Read() and Write() methods[, 364](#index_split_003.html#p373)

389

Index

**S**

Spatial data[, 279](#index_split_002.html#p289)

aggregation methods[, 325](#index_split_003.html#p334)

Sales area hierarch[y, 344](#index_split_003.html#p353)

Alabama state border

anchor query, [349](#index_split_003.html#p358)

address validation,

components[, 349–350](#index_split_003.html#p358)

[326–327](#index_split_003.html#p335)

CTE[, 347–348](#index_split_003.html#p356)

area of aggregate envelope,

HierarchyID

[331–332](#index_split_003.html#p340)

area’s pos[ition, 350–351](#index_split_003.html#p359)

exposing SRID[, 329–330](#index_split_003.html#p338)

create table, [352](#index_split_003.html#p361)

finding nearest neighbors,

SELECT statement,

[327–329](#index_split_003.html#p336)

[354–356](#index_split_003.html#p363)

constructing geometries

size of[, 353–354](#index_split_003.html#p362)

GML[, 302](#index_split_002.html#p311)

recursive query, [349](#index_split_003.html#p358)

methods[, 300](#index_split_002.html#p309)

traditional approach[, 345–347](#index_split_003.html#p354)

NULL value[, 304](#index_split_002.html#p313)

Sales department organization

Point metho[d, 302–303](#index_split_002.html#p311)

chart[, 342](#index_split_003.html#p351)

STIsValid and MakeValid

Secondary XML indexes, [171–174](#index_split_001.html#p183)

methods, Polygon, [305–307](#index_split_002.html#p314)

Shredding JSON data

WKB, [302](#index_split_002.html#p311)

array elements, temporary

WKT, [300–301,](#index_split_002.html#p309) [303–304](#index_split_002.html#p312)

table, [246–249](#index_split_002.html#p257)

GEOGRAPHY data type,

OPENJSON() function ( *see*

[282–284](#index_split_002.html#p292)

OPENJSON() function)

GEOMETRY data type[, 280–282](#index_split_002.html#p290)

Shredding XML

indexing ( *see* Spatial indexes)

nodes(), [146–151](#index_split_001.html#p158)

IsValidDetailed error codes,

OPENXML()

[324–325](#index_split_003.html#p333)

flags values, [142](#index_split_001.html#p154)

methods, [307–323](#index_split_002.html#p316)

parameters[, 141–142](#index_split_001.html#p153)

multipart geometries, [286](#index_split_002.html#p296)

Product element[, 145](#index_split_001.html#p157)

primitive geometries, [284–285](#index_split_002.html#p294)

results, [146](#index_split_001.html#p158)

spatial reference systems,

sales orders document,

[291–293](#index_split_002.html#p301)

[143–144](#index_split_001.html#p155)

SSMS[, 294–296](#index_split_002.html#p304)

sp_xml:preparedocument

WKB[, 289–291](#index_split_002.html#p299)

procedure[, 143](#index_split_001.html#p155)

WKT[, 287–288](#index_split_002.html#p297)

WITH clause, [142,](#index_split_001.html#p154) [145](#index_split_001.html#p157)

390

Index

Spatial indexes

syntactically incorrect, [38–39](#index_split_000.html#p50)

B-Tree structure, [333](#index_split_003.html#p342)

syntax error, [40](#index_split_000.html#p52)

creating, [339](#index_split_003.html#p348)

Well-known binar[y (WKB), 300](#index_split_002.html#p309)

data types[, 332–333](#index_split_003.html#p341)

hexadecimal representation—

grid system[, 334](#index_split_003.html#p343)

big endian, [291](#index_split_002.html#p301)

New Index dialog box

hexadecimal value, [289](#index_split_002.html#p299)

General page, [334–335](#index_split_003.html#p343)

LineString, [302](#index_split_002.html#p311)

Options page, [335–337](#index_split_003.html#p344)

surface type integer codes[, 289](#index_split_002.html#p299)

Spatial page[, 338–339](#index_split_003.html#p347)

Well-known text (WKT)

Storage page, [337–338](#index_split_003.html#p346)

LineString, [301](#index_split_002.html#p310)

tessellation process, [334](#index_split_003.html#p343)

multipart geometry, [288](#index_split_002.html#p298)

T-SQL[, 339](#index_split_003.html#p348)

primitive geometr[y, 287–288](#index_split_002.html#p297)

Spatial reference identifier (SRID),

variable, [303–304](#index_split_002.html#p312)

[291,](#index_split_002.html#p301) [293](#index_split_002.html#p303)

SQL Server Management Studio

(SSMS), [294–296](#index_split_002.html#p304)

**X, Y, Z**

XML data[, 25](#index_split_000.html#p37)

**T, U**

style options[, 26](#index_split_000.html#p38)

XML indexes

Tessellation process, [334](#index_split_003.html#p343)

clustered ( *see* Clustered

ToString() metho[d, 358–360](#index_split_003.html#p367)

indexes)

OrderSummary table, [157–159](#index_split_001.html#p169)

**V**

PATH index

drop primary XML index

value() metho[d, 118–121](#index_split_001.html#p130)

and run query, [178](#index_split_001.html#p190)

Ordered StockItemID [23,](#index_split_000.html#p35) [175](#index_split_001.html#p187)

query execution time,

**W**

[178–179](#index_split_001.html#p190)

Well-formed XML

query results, [176–177](#index_split_001.html#p188)

components[, 42](#index_split_000.html#p54)

run query, [176](#index_split_001.html#p188)

fragment, [40–41](#index_split_000.html#p52)

primar[y, 167–171](#index_split_001.html#p179)

requirements, [38](#index_split_000.html#p50)

secondary, [171–174](#index_split_001.html#p183)

root node[, 41–42](#index_split_000.html#p53)

types[, 157](#index_split_001.html#p169)

391

Index

XML SCHEMA

XML document[, 127–128](#index_split_001.html#p139)

COLLECTION[, 152–154](#index_split_001.html#p164)

methods[, 113–114](#index_split_001.html#p125)

XML schema definition (XSD)

modify method

schemas, [43–45](#index_split_000.html#p55)

delet[e option, 138–139](#index_split_001.html#p150)

XQuery

inser[t option, 133–137](#index_split_001.html#p145)

[definition, 113](#index_split_001.html#p125)

replace value of option,

exist(), [116,](#index_split_001.html#p128) [118](#index_split_001.html#p130)

[139–141](#index_split_001.html#p151)

FLWOR statement

quer[y(), 121–123](#index_split_001.html#p133)

for and let statement[, 131](#index_split_001.html#p143)

relational values, T-SQL

product elements, [129–130](#index_split_001.html#p141)

variables[, 123–127](#index_split_001.html#p135)

product name, [130](#index_split_001.html#p142)

Sales.CustomerOrderSummary,

where and order by

[114–116](#index_split_001.html#p126)

statement, [132](#index_split_001.html#p144)

value(), [118–121](#index_split_001.html#p130)

392

# Document Outline

*   [Table of Contents](#index_split_000.html#p5)
*   [About the Author](#index_split_000.html#p10)
*   [About the Technical Reviewer](#index_split_000.html#p11)
*   [Introduction](#index_split_000.html#p12)
*   [Chapter 1: SQL Server Data Types](#index_split_000.html#p13)
    *   [Numeric Data Types](#index_split_000.html#p13)
    *   [Character Strings](#index_split_000.html#p22)
    *   [Binary Data Types](#index_split_000.html#p26)
    *   [Dates and Times](#index_split_000.html#p29)
    *   [Miscellaneous Standard Data Types](#index_split_000.html#p35)
    *   [Summary of Advanced Data Types](#index_split_000.html#p37)
    *   [Why Is Using the Correct Data Type Important?](#index_split_000.html#p38)
    *   [Summary](#index_split_000.html#p39)
*   [Chapter 2: Understanding XML](#index_split_000.html#p41)
    *   [Understanding XML](#index_split_000.html#p41)
    *   [Well-Formed XML](#index_split_000.html#p50)
    *   [Understanding XSD Schemas](#index_split_000.html#p55)
    *   [XML Usage Scenarios in SQL Server](#index_split_000.html#p58)
    *   [Summary](#index_split_000.html#p59)
*   [Chapter 3: Constructing XML with T-SQL](#index_split_000.html#p61)
    *   [Using FOR XML RAW](#index_split_000.html#p61)
    *   [Using FOR XML AUTO](#index_split_000.html#p89)
    *   [Using FOR XML PATH](#index_split_001.html#p104)
    *   [Using FOR XML EXPLICIT](#index_split_001.html#p113)
    *   [Summary](#index_split_001.html#p123)
*   [Chapter 4: Querying and Shredding XML](#index_split_001.html#p125)
    *   [Querying XML](#index_split_001.html#p125)
        *   [Using exist()](#index_split_001.html#p128)
        *   [Using value( )](#index_split_001.html#p130)
        *   [Using query( )](#index_split_001.html#p133)
        *   [Using Relational Values in XQuery](#index_split_001.html#p135)
        *   [FLWOR](#index_split_001.html#p139)
        *   [Modifying XML Data](#index_split_001.html#p145)
    *   [Shredding XML](#index_split_001.html#p153)
        *   [Shredding XML with OPENXML( )](#index_split_001.html#p153)
        *   [Shredding XML with Nodes](#index_split_001.html#p158)
    *   [Using Schemas](#index_split_001.html#p164)
    *   [Summary](#index_split_001.html#p167)
*   [Chapter 5: XML Indexes](#index_split_001.html#p169)
    *   [Preparing the Environment](#index_split_001.html#p169)
    *   [Clustered Indexes](#index_split_001.html#p171)
        *   [Tables Without a Clustered Index](#index_split_001.html#p172)
        *   [Tables with a Clustered Index](#index_split_001.html#p173)
        *   [Clustering the Primary Key](#index_split_001.html#p174)
        *   [Performance Considerations for Clustered Indexes](#index_split_001.html#p176)
        *   [Creating a Clustered Index](#index_split_001.html#p177)
    *   [Primary XML Indexes](#index_split_001.html#p179)
        *   [Creating Primary XML Indexes](#index_split_001.html#p180)
    *   [Secondary XML Indexes](#index_split_001.html#p183)
        *   [Creating Secondary XML Indexes](#index_split_001.html#p184)
    *   [Performance Considerations for XML Indexes](#index_split_001.html#p187)
    *   [Summary](#index_split_001.html#p191)
*   [Chapter 6: Understanding JSON](#index_split_001.html#p192)
    *   [Understanding the JSON Format](#index_split_001.html#p192)
    *   [JSON vs. XML](#index_split_001.html#p199)
    *   [JSON Usage Scenarios](#index_split_002.html#p206)
        *   [n-Tier Applications with Rest APIs](#index_split_002.html#p206)
        *   [De-Normalizing Data](#index_split_002.html#p206)
        *   [Config As Code](#index_split_002.html#p208)
        *   [Analyzing the Log Data](#index_split_002.html#p210)
    *   [Summary](#index_split_002.html#p210)
*   [Chapter 7: Constructing JSON from T-SQL](#index_split_002.html#p212)
    *   [FOR JSON AUTO](#index_split_002.html#p212)
        *   [Working with Root Nodes](#index_split_002.html#p215)
        *   [Working with NULL Values](#index_split_002.html#p218)
        *   [Using Column Aliases](#index_split_002.html#p223)
        *   [Automatic Nesting](#index_split_002.html#p226)
    *   [FOR JSON PATH](#index_split_002.html#p235)
    *   [Summary](#index_split_002.html#p239)
*   [Chapter 8: Shredding JSON Data](#index_split_002.html#p240)
    *   [OPENJSON() with Default Schema](#index_split_002.html#p240)
        *   [Shredding a Column](#index_split_002.html#p243)
        *   [Dynamic Shredding Based on Document Content](#index_split_002.html#p246)
    *   [OPENJSON( ) with Explicit Schema](#index_split_002.html#p248)
    *   [OPENJSON( ) with Path Expressions](#index_split_002.html#p253)
        *   [Shredding Data into Tables](#index_split_002.html#p257)
    *   [Summary](#index_split_002.html#p260)
*   [Chapter 9: Working with the JSON Data Type](#index_split_002.html#p262)
    *   [Querying JSON Data](#index_split_002.html#p262)
        *   [Using ISJSON( )](#index_split_002.html#p262)
        *   [Using JSON_VALUE( )](#index_split_002.html#p265)
        *   [Using JSON_QUERY( )](#index_split_002.html#p272)
        *   [Using JSON_MODIFY()](#index_split_002.html#p277)
    *   [Indexing JSON Data](#index_split_002.html#p282)
    *   [Summary](#index_split_002.html#p287)
*   [Chapter 10: Understanding Spatial Data](#index_split_002.html#p289)
    *   [Understanding Spatial Data](#index_split_002.html#p289)
    *   [Spatial Data Standards](#index_split_002.html#p296)
        *   [Well-Known Text](#index_split_002.html#p297)
        *   [Well-Known Binary](#index_split_002.html#p299)
        *   [Spatial Reference Systems](#index_split_002.html#p301)
    *   [SSMS and Spatial Data](#index_split_002.html#p304)
    *   [Summary](#index_split_002.html#p306)
*   [Chapter 11: Working with Spatial Data](#index_split_002.html#p308)
    *   [Constructing Spatial Data](#index_split_002.html#p309)
    *   [Querying Spatial Data](#index_split_002.html#p316)
    *   [Indexing Spatial Data](#index_split_003.html#p341)
        *   [Understanding Spatial Indexes](#index_split_003.html#p341)
        *   [Creating Spatial Indexes](#index_split_003.html#p343)
    *   [Summary](#index_split_003.html#p349)
*   [Chapter 12: Working with Hierarchical Data and HierarchyID](#index_split_003.html#p350)
    *   [Hierarchical Data Use Cases](#index_split_003.html#p351)
    *   [Modeling Traditional Hierarchies](#index_split_003.html#p353)
    *   [Modeling Hierarchies with HierarchyID](#index_split_003.html#p359)
    *   [HierarchyID Methods](#index_split_003.html#p365)
        *   [Working with HierarchyID Methods](#index_split_003.html#p367)
            *   [Using ToString( )](#index_split_003.html#p367)
            *   [Using Parse( )](#index_split_003.html#p369)
            *   [Using GetRoot( )](#index_split_003.html#p371)
            *   [Using GetLevel( )](#index_split_003.html#p372)
            *   [Read( ) and Write( )](#index_split_003.html#p373)
            *   [Using GetDescendant( )](#index_split_003.html#p373)
            *   [Using GetReparentedValue( )](#index_split_003.html#p376)
            *   [Using GetAncestor( )](#index_split_003.html#p379)
            *   [Using IsDescendantOf( )](#index_split_003.html#p380)
    *   [Indexing HierarchyID Columns](#index_split_003.html#p386)
    *   [Summary](#index_split_003.html#p391)
*   [Index](#index_split_003.html#p393)***************************