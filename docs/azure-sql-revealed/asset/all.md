![](images/979-8-8688-0974-3_CoverFigure.jpg)

ISBN 979-8-8688-0973-6e-ISBN 979-8-8688-0974-3 [https://doi.org/10.1007/979-8-8688-0974-3](https://doi.org/10.1007/979-8-8688-0974-3) © The Editor(s) (if applicable) and The Author(s), under exclusive license to APress Media, LLC, part of Springer Nature 2021, 2024 This work is subject to copyright. All rights are solely and exclusively licensed by the Publisher, whether the whole or part of the material is concerned, specifically the rights of translation, reprinting, reuse of illustrations, recitation, broadcasting, reproduction on microfilms or in any other physical way, and transmission or information storage and retrieval, electronic adaptation, computer software, or by similar or dissimilar methodology now known or hereafter developed. The use of general descriptive names, registered names, trademarks, service marks, etc. in this publication does not imply, even in the absence of a specific statement, that such names are exempt from the relevant protective laws and regulations and therefore free for general use. The publisher, the authors and the editors are safe to assume that the advice and information in this book are believed to be true and accurate at the date of publication. Neither the publisher nor the authors or the editors give a warranty, expressed or implied, with respect to the material contained herein or for any errors or omissions that may have been made. The publisher remains neutral with regard to jurisdictional claims in published maps and institutional affiliations.

This Apress imprint is published by the registered company APress Media, LLC, part of Springer Nature.

The registered company address is: 1 New York Plaza, New York, NY 10004, U.S.A.

*This book is dedicated to all of the customers of Microsoft who have made the journey to migrate to Azure.*

Foreword

If you told me in late 2018 that I would one day write the foreword of a book written by Bob Ward, I would have laughed and probably added a “Yeah, right.” At the time, I was reading Bob’s book, *Pro SQL Server on Linux*. Actually, I was pouring over the book, trying to learn as much about SQL Server as I could. I tried all the exercises, sometimes multiple times to make sure I really grasped both the *problem* and the *solution*, which is fortunately a way of storytelling that Bob often employs. Understanding the scenarios was critical for me to understand why and how the SQL team has built various aspects of the product.

Why was I pouring over *Pro SQL Server on Linux*? In 2018, I transferred internally at Microsoft to join the SQL Server and Azure SQL product group. I didn’t know anything about SQL Server or managing databases. Nothing. Asad Khan, Vice President of SQL at Microsoft, took a chance on me. Bob’s *Pro SQL Server on Linux* book was amazing because Bob “read the room.” He took the extra time to explain the fundamental concepts of SQL Server, because Linux folks don’t necessarily know SQL Server. I learned so much from using the book as a tool. It set me up for success.

It was no surprise in 2019 when Asad’s charter grew to include not just SQL Server but also Azure SQL and migrations. It was then that Asad asked me to look at migrations to Azure SQL. Bob, Buck Woody, and I together built SQL Ground to Cloud, which was a set of resources (slides, labs) meant to help you understand all things SQL Server 2019, Azure SQL, and migrations. We did a global “tour” where we attended many SQL Saturdays, events, and Microsoft offices, to enable both internals at Microsoft and customers. Between July and December of 2019, we were never home for more than three days at a time. During all this, Bob wrote *SQL Server 2019 Revealed*.

For anyone who has done any presentation before, you know that practice helps immensely. I think something that makes Bob’s books both informative and digestible is how much experience Bob has delivering the messages. For example, prior to the launch of his *SQL Server 2019 Revealed* book, Bob had, no exaggeration, likely presented the content over 50 times to over 100,000 people. This gave Bob the opportunity to observe what resonates, hear all questions, and tune his message accordingly. The impact is that any book you pick up and read by Bob, including this one, is easily consumable and often directly relevant for the problems you are likely facing today.

By the end of 2019, I realized that we needed a way to bring SQL Server people *with us* to the cloud and Azure SQL. I had a hope that we could deliver a similar roadshow to SQL Ground to Cloud but more focused on translating SQL Server knowledge to Azure. I remember having an old-fashioned cocktail (a Bob staple) with Bob after the SQL Server 2019 product release, and he told me he was going to start looking at Azure SQL. It was a dream come true. Bob Ward, a trusted SQL Server guru in the community, was going to study Azure SQL. So I asked Bob if he would be willing to take the results of his close examination of Azure SQL and partner with me on the effort of building something called “Azure SQL fundamentals.” Bob took a chance on this with me, though I’m not sure he knew exactly what he signed up for. Together, we produced a 61-episode video series with over 500K views, a learning path with six modules ([`https://aka.ms/azuresqlfundamentals`](https://aka.ms/azuresqlfundamentals)) with over 1M views, and a highly polished deck that we (or anyone, we made it public) could use to present one module at a time, or a one-day workshop.

During this time, I got to work very closely with Bob. Bob was always hyper-focused on understanding the exact details and reasons around any differences between Azure SQL and SQL Server. There were hundreds of emails, meetings, and Teams messages that Bob drove with program managers, engineers, ex-SQL team members, and the field to understand every difference and every nuance. He also used this phrase “Trust, but verify,” and I think that actually landed in this book. Bob has immense respect and rapport, and it was very cool to witness how he professionally and respectfully gets to understanding both the *why* and the *how*. Bob then translates that into relatable *problems* and *solutions*.

During all this, similar to with the craziness of his *SQL Server 2019 Revealed* efforts, Bob wrote *Azure SQL Revealed*, which you are now reading the second edition of. This is where Bob has captured and distilled all the conversations and learnings into one comprehensive piece. If you have any history of working with SQL Server, Azure SQL, or Azure in general, I promise this will be an easy-to-read book and an actually entertaining and interesting read. If you read the first edition, you can be sure that Bob has made sure every detail was verified or updated, and every new capability up to the very last day that Bob could submit edits has been added (we move at the speed of the cloud!).

I mentioned how Asad and Bob have taken chances on me, but I think it’s even more important that we take chances on ourselves – that we believe in ourselves. My call to action is to take a chance on yourself with this book. Treat it as a tool, not a book. Do the exercises, make sure you understand the “meat and potatoes” (if you don’t recognize this phrase from Bob, you will soon). If you do, the opportunities and doors that will open for you professionally and even personally may surprise you. This is just the beginning.

Anna Hoffman

Principal Group PM Manager and Host of Data Exposed, Microsoft

Introduction

I can’t believe it has been four years since the first edition of *Azure SQL Revealed* was released. I took off writing in 2023, but at the PASS Data Community Summit in November 2023, I met Shaul Elson from Apress. He asked me about whether I was interested in writing another book. My first instinct was the second edition of *Azure SQL Revealed* because I had lived and breathed through four years of enhancements and changes. I knew it would be compelling to update it all in a new edition of this book.

The first edition of this book was written during the pandemic all from my home in North Richland Hills, Texas, in 2020\. But 2024 was a massive travel year for me both professionally and personally, so it was fun to look back in all the places where I wrote bits and pieces of this book in planes, cars, trains, hotels, and convention centers in Austin, Las Vegas (multiple times), Orlando, Atlanta, Dallas, Farnborough (UK), London, Dublin and Cork (Ireland), Ft. Lauderdale, Monterey (California), Seattle, Redmond (Microsoft Campus), Washington DC, New York City, Belize, Chicago, Denver, and Genesee (Colorado).

The intention of this edition of the book is the same as the first edition: I want SQL professionals from developers to administrators to architects who know SQL Server to learn Azure SQL. I always listen to feedback, positive and negative. I read quite a bit of feedback from reviews of the first edition. All of them were extremely positive with a few constructive comments that I think helped shape a few changes in this book.

Keep in mind these things as you read the chapters of the book:

*   The first chapter is “pure history.” Every book I’ve written starts with a history. Azure SQL has a long history, so this chapter can seem long. If you don’t care about that, then skip this chapter and go back to it later.

*   The second chapter is all fundamentals. If you believe you have a good grasp of fundamentals of “what is Azure and Azure SQL?” then go right to the remaining chapters. However, I believe it is worth your time to look over. You never know what you might find you didn’t know.

*   I like to include references for my books. I want my readers to have further material that reinforce my words. However, this may lead some to grow weary of seeing URL links in the book. In this edition, they are still there, but I removed several from the first edition that I think did not add value. For readers of the electronic version of the book, these links provide an easy “go-to” reference to get more information. If you are reading the printed copy, sometimes it may seem to “get in the way,” but I believe you will find these reference points helpful to extend the knowledge you need.

*   I went through every chapter of the first edition and completely modified it to ensure all the latest details were up to date, including new figures, removing features that no longer exist, and adding new capabilities.

*   There are not as many sample scripts as in my other books. Many of the samples are guided exercises using the Azure Portal. I do recommend a free workshop with labs I have built you can use at [`https://aka.ms/cloudsqlworkshop`](https://aka.ms/cloudsqlworkshop) to extend your hands-on experience.

With these details in mind, here is my guide on how to best use and read this book.

Chapter [3](#496204_2_En_3_Chapter.xhtml) is a complete end-to-end coverage of SQL Server in Azure Virtual Machine which provides a great “lift and shift” approach to migrate SQL Server to Azure. The rest of the book focuses on the two PaaS services: Azure SQL Managed Instance and Azure SQL Database.

Chapters [4](#496204_2_En_4_Chapter.xhtml) and [5](#496204_2_En_5_Chapter.xhtml) are guides to deploy and configure both Azure SQL Managed Instance and Azure SQL Database. Even if you have already been through the deployment experience, I guarantee you will find hidden gems in these chapters.

Chapters [6](#496204_2_En_6_Chapter.xhtml)–[8](#496204_2_En_8_Chapter.xhtml) are the core of knowledge of Azure SQL, covering security, performance, and availability. These are what I call the “meat and potatoes” of SQL, so I think you will like seeing what is both the same and different. I think you will also like the new scenarios in these chapters using technologies like managed identities, Copilots, and Database Watcher.

Chapter [9](#496204_2_En_9_Chapter.xhtml) covers a range of topics to extend your knowledge not covered anywhere else in the book, like how to schedule jobs, use data virtualization, or support your own issues with Azure SQL.

I completely replaced Chapter [10](#496204_2_En_10_Chapter.xhtml) from the first edition and included topics related to what I call *Beyond RDBMS*, including Generative AI applications and integration with Microsoft Fabric.

If you want any sample scripts from the book, see any errata found after publishing, or keep up with any important updates, bookmark the link [`https://aka.ms/azuresqlbook2`](https://aka.ms/azuresqlbook2).

My personal philosophy is to “empower others to make a difference,” so that is my ultimate goal for the readers of this book: I want to empower you with the knowledge to learn Azure SQL, so you can be successful in the cloud. Welcome to the world’s database: Azure SQL.

Bob Ward

North Richland Hills, Texas

September 2024

Acknowledgments

Anything is possible with God, and I believe it is his gift of grace through his son Jesus Christ that I have a hope of everlasting life. I thank God every day for giving me gifts of intelligence and to carry forward a message of grace and truth in everything I do. God has given me in my life the most amazing partner and wife anyone could ask for in Ginger. She has endured all of those seemingly endless days and nights of me working on this book, our travels together while I wrote it, and an incredible patience hearing me talk about this passion of mine. Thank you, Ginger, and all you have done, all with a loving heart. I thank God as well for my two sons Troy and Ryan who show me all the time what better men they are than me. I couldn’t ask any more from God for being a father. Seeing their independence gives me such immense joy. But I still also love the phone call “Hey dad, can I ask you something….” Thank you to my daughter-in-law Blair Ward who is always saying how she knows a “famous author.” I don’t tell her, but it makes me feel pretty special.

I want to thank my technical reviewer Joe Sack, who also reviewed the first edition and is a colleague of mine at Microsoft. Joe is the ultimate professional and one of the nicest people you will meet. Joe also reviewed most of this book under incredible tight deadlines but never complained once. I want to thank Anna Hoffman for writing the foreword. When I asked Anna to do this, she acted surprised, but she was the first person I thought of when I was writing the book this year. Anna not only has become an industry presence, but she is one of the most talented people I work with at Microsoft. I loved her personal style in the foreword.

I want to thank my contacts at Apress: Welmoed Spahr, Shaul Elson, and Gryffin Winkler. You all were so easy and great to work with and showed amazing patience and flexibility for me to finish out the book and write in my style. Thank you Apress for continuing to give me opportunities to share my knowledge.

I also want to thank all the members of the Azure SQL leadership, product management, engineering, marketing, and sales team at Microsoft that all help power the “world’s database.” There are so many people at Microsoft behind the scenes that make this incredible service better each day.

And finally, thanks to you, the community who support and help us promote the amazing story of SQL Server and Azure SQL. Many of you have made the journey to migrate to Azure, and some are still in that process or thinking about it. Thank you all #sqlfamily for putting your passion and energy into everything that is SQL.

About the Author About the Technical Reviewer

# 1. SQL Server Rises to the Clouds

In late 2005, Microsoft as a company was humming (I’m a little biased here) in the enterprise space and so was the SQL Server product. In October of 2005, we were close to releasing SQL Server 2005 (codename Yukon) which was unfortunately five years in the making (that is a story for another book – just ask Paul Randal). I was in Microsoft support in those days, and despite the delay in getting SQL Server 2005 to market, I was very proud of the release. Windows, Windows Server, Office, and Xbox 360 were all popular products from Microsoft.

In October of 2005, an architect new to Microsoft named Ray Ozzie sent an internal email to several executives at Microsoft (which eventually was sent to all employees, including a 12-year veteran named Bob Ward) called *The Internet Services Disruption* (the email leaked to the Web fairly quickly, which you can read at [`https://www.cnet.com/news/ozzie-memo-internet-services-disruption/`](https://www.cnet.com/news/ozzie-memo-internet-services-disruption/)). I remember hearing about the email leak and some of its contents as an employee but didn’t pay much attention. Wasn’t the Internet just for email and web browsing? In that email, Ray Ozzie painted a picture of Microsoft becoming a *cloud provider* vs. just a “traditional software company.” Microsoft only really had a few “Internet services” offerings at the time which included the legendary Hotmail email service (which had existed since 1997), the Bing search service, and Xbox Live. The email from Ray Ozzie painted a picture for something far bigger.

One of the key statements from this email was “…All Business Groups have been asked to develop their plans to embrace this mission and create new service offerings that deliver value to customers and utilize the platform capabilities that we have today and are building for the future.” Little did I know how much behind the scenes work would kick in within the SQL Server team to develop plans for this statement.

Ray Ozzie became the Chief Software Architect of Microsoft in the summer of 2006 (taking the role held by Bill Gates), and this email would set the stage for what would become known as ***Azure***. SQL Server was destined to become a huge part of it.

## CloudDB

In early 2006, Paul Flessner, Vice President of the Data Storage and Platform division of Microsoft, decided to step down as the leader of SQL Server and turn over the reins to Ted Kummert. When Ted took over to lead SQL Server, a project was already underway to look at cloud services led by Technical Fellow Peter Spiro, who was a chief architect for several SQL Server releases, including SQL Server 7.0, 2000, and 2005\. Peter formed a team which included several engineers. Among them was an architect still at Microsoft today: Ajay Kalhan. The team embarked on a project to build a cloud-based service to host databases. They called it ***CloudDB***. As Ted tells it, “We needed to build a cloud version of SQL. Our goal was to build a *serverless* or Platform as a Service (PaaS) SQL. A customer wouldn’t worry about a server or VM, just a database.”

To build a cloud-based database service, the team needed to build out a robust design to support the concept of hosting multiple customers or “databases” isolated from each other using shared resources. This concept is called *multi-tenant*.

Note

The term *tenant* can mean many things in the cloud. For CloudDB in the beginning, a tenant referred to a database owned by a customer. You will see throughout this book the word tenant, but I’ll be clear about the scope of what I mean when using the term.

According to Ajay Kalhan, from the beginning the CloudDB team started working out designs to incorporate concepts such as failure detection, logical master (think of a “metadata” master not physical), load balancing, and deployment. Early designs even looked at the idea of a “key value store” instead of traditional relational database concepts. Not long after the team was building out the design for CloudDB, Ted assigned David Campbell to also work on the project and lead the team toward a true mission of “SQL Server in the Cloud.”

The team believe it needed an internal customer to help *dogfood* the project and prove they could host customers. That internal customer would become a public-facing cloud service called **Exchange Hosted Archive (EHA)** (an email archive solution in the cloud predating Office 365). For this internal customer, early designs to support multi-tenants (which in this case even though there was one internal customer, that customer serviced the needs of multiple customers) used a concept called *silos* where a SQL Server could host multiple databases, but tenants were partitioned within the database itself. EHA became one of the first ***Software as a Service*** (**SaaS**) services at Microsoft to use our cloud-based database service. Think of SaaS as purchasing software on a subscription basis and using the software from a hosted solution, like in Azure. You just focus on using an application hosted somewhere other than your computers. Since SQL Server hosted the back-end databases, the team forked the codebase of SQL Server 2005 to use for the service.

While the CloudDB team was working on their project with a goal to support EHA and other customers, another team at Microsoft was chartered by Ray Ozzie to look at how to host *compute services* in the cloud.

## The Red Dog

In 2006, Ray Ozzie enlisted Microsoft veteran Amitabh Srivastava to lead a “Cloud OS” project in the attempt to move forward the “Internet services disruption” he had talked about a year ago. One of the first actions Amitabh took was to bring out of retirement Dave Cutler, the “father” of DEC VMS and Windows NT operating systems. As part of their initial project work, Srivastava and Cutler visited groups at Microsoft that were providing “cloud services,” including Xbox Live, Hotmail, and Bing. On one of the trips to visit Hotmail in San Jose, California, the team drove by a club called the Pink Poodle. It was Dave Cutler who famously said, “Maybe we should name our project the Pink Poodle?” The project team all agreed that would not go over well, so named the project instead “Red Dog.” The name stuck (you can read more about the great history of the beginning of Red Dog at ([`https://www.wired.com/2008/11/ff-ozzie/?currentPage=7`](https://www.wired.com/2008/11/ff-ozzie/%253FcurrentPage%253D7) and [`https://www.zdnet.com/article/how-the-red-dog-dream-team-built-a-cloud-os-from-scratch/`](https://www.zdnet.com/article/how-the-red-dog-dream-team-built-a-cloud-os-from-scratch/)).

From the beginning, the Red Dog team did things differently at Microsoft to build the “cloud OS.” They built their own “data center” in the heart of the Microsoft campus, even taking reserve power from neighboring buildings. Their goals were ambitious and still resonate today. Their main overall goal was to *build a cloud service for developers to build scalable web applications*. They also had a massive theme from the beginning: *reliability*. As Dave Cutler said back in 2008, “One of the things you did not ask is why aren’t we saying more about Azure and in the process filling the marketplace with sterling promises for the future. The answer to this is simply that the RD group is very conservative, and we are not anywhere close to being done. We believe that cloud computing will be very important to Microsoft’s future, and we certainly don’t want to do anything that would compromise the future of the product. We are hypersensitive about losing people’s data. We are hypersensitive about the OS or hypervisor crashing and having properties experience service outages. So, we are taking each step slowly and attempting to have features 100% operational and solidly debugged before talking about them. The opposite is what Microsoft has been criticized for in the past and the RD dogs hopefully have learned a new trick.”

The Red Dog and CloudDB teams were marching together as separate projects (ironically on the same campus only buildings apart) to support cloud services for web applications and hosted databases in the cloud. These projects were on a path to come together in 2007 and 2008 for the launch of a unified cloud service.

## The Azure Services Platform

In October of 2008 at the Microsoft Professional Developers Conference (PDC) in Los Angeles, California, Ray Ozzie announced **Windows Azure**. The PDC was the precursor to today’s Microsoft //Build conference ([`https://en.wikipedia.org/wiki/Build_(developer_conference)`](https://en.wikipedia.org/wiki/Build_%2528developer_conference%2529). PDC was a huge event for Microsoft for developers.

Windows Azure was launched as part of the **Azure Services Platform**. Figure [1-1](#496204_2_En_1_Chapter.xhtml#Fig1) shows a snapshot of the Azure Services Platform offerings.

![](images/496204_2_En_1_Chapter/496204_2_En_1_Fig1_HTML.jpg)

Figure 1-1

The Azure Services Platform in 2008

The Red Dog team had been cranking away since 2006 with the goal of releasing a cloud service for developers. Ray Ozzie called Windows Azure a “new Windows offering at the web tier of computing” (watch the video for yourself at [`https://www.zdnet.com/article/ray-ozzie-announces-windows-azure/`](https://www.zdnet.com/article/ray-ozzie-announces-windows-azure/)). He also called Azure “Windows in the cloud.” Microsoft now would offer customers Windows on your laptop (at that time, it was Windows Vista), servers for your enterprise (Windows Server), and Windows in the cloud (Azure).

Note

I sought out many folks at Microsoft on why our cloud service was named Azure. As Buck Woody tells the story, “Azure means clear blue sky with no clouds. The name just seemed right without using the word cloud in our name.”

Like the goal of the CloudDB project, when Windows Azure was first released, the goal was all about scalability and availability targeting web applications in the form of a *Platform as a Service* (**PaaS**). Think of PaaS as purchasing a platform to host your application or database based on a subscription where the platform is *managed* by a provider, like Azure. With PaaS, you are typically abstracted from a host computer or virtual machine. Therefore, *cloud services* was the first service in Windows Azure. This type of service was known internally as PaaS V1.

Note

Cloud services are still offered today in Azure. You can read more about cloud services at [`https://azure.microsoft.com/services/cloud-services/`](https://azure.microsoft.com/services/cloud-services/). This has been replaced though in modern apps with Azure App Service which you can read more about at [`https://azure.microsoft.com/services/app-service/`](https://azure.microsoft.com/services/app-service/).

Even though a cloud service application ran in one or more virtual machines, the idea was to support easy-to-scale web applications in the cloud where developers didn’t focus on the details of the virtual machine but more on the application. Developers at this time for Windows were used to the Internet Information Server (IIS) feature of Windows Server. While developers didn’t have to worry as much about deploying and configuring IIS, they typically had to have an administrator within their organization. While developers had some access to the virtual machine native OS environment for cloud services, that access was limited. It would be a few years later that Microsoft would introduce the concept of *Infrastructure as a Service* (**IaaS**) through Azure Virtual Machine. Think of IaaS as purchasing an infrastructure to host your virtual machine based on a subscription. You worry about the guest VM, and the provider manages the host, hardware, networking, and storage.

One of the other promises of PaaS and cloud services is to create an easy-to-use concept of application deployment, configuration, and updates. Furthermore, providing capabilities for scalability, built-in high availability, and load balancing made the concept of cloud services extremely appealing to web developers. These same concepts you will see are a part of the appeal as well for Azure SQL and databases.

To host PaaS cloud services, an *underlying hosting system* had to be built. The Windows Azure team took the designs from the Red Dog project to build this hosting system to support deployment, networking, high availability, scale, and security, as cloud services abstracted all these details from the developer. This software hosting system was known as *Windows Fabric*. Providing the underlying hosting system for services consumed by users is the *power of the cloud*. I found this interesting slide from a talk at the PDC 2008 conference that talks about all the details required for someone to run their own *fabric* in a data center as seen in Figure [1-2](#496204_2_En_1_Chapter.xhtml#Fig2).

![](images/496204_2_En_1_Chapter/496204_2_En_1_Fig2_HTML.jpg)

Figure 1-2

Building your own fabric in a data center

This slide speaks volumes for what a fabric must support for cloud services at scale. A highly available *fabric controller (**FC)* is key to the system. The FC maintains a graph of the inventory of what it manages: computer, virtual machines, load balancers, and switches with edges being objects like network cables. One key to the fabric system is the use of a declarative model, so the FC takes what you declare and implements it. Very early on, Windows Fabric in Azure had concepts of high availability such as fault and update domains (I’ll describe the importance of these in Chapters [2](#496204_2_En_2_Chapter.xhtml) and [3](#496204_2_En_3_Chapter.xhtml) of the book).

Windows Fabric is today known as **Service Fabric**. The usage of Service Fabric is also exposed to applications to host their own services in a Service Fabric cluster. You can read more about Azure Service Fabric at [`https://azure.microsoft.com/services/service-fabric/`](https://azure.microsoft.com/services/service-fabric/).

Note

As you read more about Service Fabric in this chapter in the book, you will likely see some similarity to another *fabric system* called Kubernetes. If you want to read more about the differences between these two systems, this blog post is a good place to start: [`https://techcommunity.microsoft.com/t5/azure-developer-community-blog/service-fabric-and-kubernetes-community-comparison-part-1-8211/ba-p/337421`](https://techcommunity.microsoft.com/t5/azure-developer-community-blog/service-fabric-and-kubernetes-community-comparison-part-1-8211/ba-p/337421).

To round out the set of *Azure Services*, Microsoft announced the data platform or **SQL services**, thus beginning the first public announcement of the journey that would become Azure SQL.

## The Road to SQL Azure

A big part of the announcement for Windows Azure at PDC in 2008 involved data. Since the CloudDB project in 2006, Peter Spiro, David Campbell, Ajay Kalhan, Tomas Talius, and the rest of the team had built out a set of cloud services for SQL Server to host ***multi-tenant databases*** (or multiple customers in a shared set of SQL Servers).

The first name announced at PDC 2008 was **SQL Data Services (****SDS)**. While the news of this service made buzz in the industry, so many customers were focused on on-premises enterprise deployments, and our team overall were heavily focused on SQL Server (e.g., shipping SQL Server 2008 code-named Katmai). But internally, the leadership of the company was making a major push for the cloud but not just because they were “told to do this.” Ted Kummert said “We were believers. We believed PaaS was the future, but we were early in the industry for a service like this.”

### SQL Data Services

SQL Data Services was announced as part of a broader set of services called ***SQL services*** which would include DataSync, Reference Data, Reporting, Data Mining, and ETL as seen in Figure [1-3](#496204_2_En_1_Chapter.xhtml#Fig3).

![](images/496204_2_En_1_Chapter/496204_2_En_1_Fig3_HTML.jpg)

Figure 1-3

SQL services at PDC in 2008

This image came from a slide from a talk presented by David Campbell back at PDC in 2008.

Note

It is interesting to see our intention was to also provide “data warehouse” services which we do today with Azure Synapse (and Microsoft Fabric) and “ETL” which is now Azure Data Factory. “Reporting” never really panned out in SQL services (but there were attempts), but Microsoft eventually created a very powerful Reporting service called Power BI.

SQL Data Services embodied the ability for developers to host databases for their applications and be completely abstracted from the details of computers, virtual machines, and SQL Server itself. Basically, you create a database; populate it with tables, data, and indexes; and then just start using it. No machine, operating system, or machine installation required.

The other concept that SDS provided was “database as a utility” or “pay as you go service.” That was really the same concept across all of Windows Azure. It represented a shift for customers to use a subscription service to pay for database usage (and the compute and storage that went behind it) vs. a license for SQL Server.

The team learned a quick lesson when it introduced the programming interface as REST API instead of T-SQL. REST stands for **Representational State Transfer** and is a common protocol used for web services. Customer feedback quickly changed that model (but REST API interfaces remain to this day for many aspects of Azure SQL, which you will see throughout the book). You can see from this blog post in March 2009 ([`https://web.archive.org/web/20140411144147/http://blogs.msdn.com/b/sqlazure/archive/2009/03/10/9469228.aspx`](https://web.archive.org/web/20140411144147/http:/blogs.msdn.com/b/sqlazure/archive/2009/03/10/9469228.aspx)) that the SDS team needed to provide developers and users a “relational data experience” which includes programming interfaces through Tabular Data Stream (TDS). Translation: T-SQL. Other basic features you expect from a SQL Server database had to be there, including indexes, stored procedures, triggers, views, etc.

Since the SDS and Windows Azure teams were innovating at the same time, the SDS team had to figure out a hosting system for databases and SQL Server. Windows Fabric was being built as the SDS team was innovating. The decision was made to use a hosting system that already existed at Microsoft instead of using Windows Azure. That platform was called *AutoPilot* (you can read more about the AutoPilot system at `https://www.microsoft.com/en-us/research/publication/autopilot-automatic-data-center-management/?from=http%3A%2F%2Fresearch.microsoft.com%2Fapps%2Fpubs%2Fdefault.aspx%3Fid%3D64604`) built by the team running the Bing Search Service. AutoPilot was effectively a platform to provision “bare-metal” computers in a scaled fashion. SDS *clusters* would physically be colocated with Windows Azure clusters, but SDS managed their own systems.

AutoPilot just provided software services to deploy and maintain applications on bare-metal servers at scale. The SDS team had to build their own set of services for fault tolerance, high availability, and connectivity. The SDS team built their own *fabric* to deploy, run, scale, and maintain SQL Server instances to host their customer databases. The original design of silos was replaced by “database per tenant” model called *partitions* (not the same as SQL Server partitions), but multiple databases could be hosted on a single SQL Server. Each bare-metal server could also host multiple SQL Server instances.

The other piece of this design to support the concept of a “database service” was to abstract users from the SQL Server instance itself (even though SQL Server instances were used to host databases). Thus, the concept of a “master node” was built into the service to host metadata about the “data nodes.” These data nodes had the concept of replicas and fabric controllers. In addition, a front-end service was built where applications would connect instead of connecting to the back-end SQL Server. This would be the early design of what is now known as *Gateway Server* for Azure SQL.

Figure [1-4](#496204_2_En_1_Chapter.xhtml#Fig4) shows the original design of the SDS hosting system.

![](images/496204_2_En_1_Chapter/496204_2_En_1_Fig4_HTML.jpg)

Figure 1-4

Original SDS hosting design for SQL service databases

Fabric processes help coordinate with the overall cluster for failover purposes. So early on, we needed to provide

*   The ability to isolate customers with our partition concept but share SQL Servers for density

*   Failover logic within the fabric

*   Replica sets of data. Sound familiar? (Kind of like an Availability Group)

*   Access for our databases to underlying storage and networking across the data center but abstracted from users

*   A logical “master” database for application databases to support logins and store other metadata

*   The ability to collect metrics to gain insight into telemetry and health

*   *Watchdog* processes for health detection

The team learned a lot in these early days. Ted Kummert described the challenges of now not just enhancing and building the software but owning all aspects of a “live service,” including hardening, quality, availability, development velocity, telemetry, outages, security, and even things like Cost of Goods Sold (COGS) and capacity planning. These early learnings would eventually allow Microsoft to scale to the levels the original team had dreamed about. As Ted described it “…we were now not just evolving a codebase, but we were evolving as a team and our capabilities all at the same time. It was both an exhilarating and humbling experience.”

Another important event in Microsoft’s company history happened in 2008\. Steve Ballmer then asked a leader within the company to reinvent another cloud service, the Bing Search Service. That leader was a man named Satya Nadella. According to Satya in his book *Hit Refresh*, “Ultimately, Bing would prove to be a great training ground for building the hyper-scale, cloud-first services that permeate Microsoft today.”

### SQL Azure Is Born

It was a massive effort to move to a market release. Along the way, SQL Data Services just didn’t have the right name for many. Therefore, in the summer of 2009, while the service was still in Community Technology Preview (CTP), a brand name change was made from SQL Data Services to **SQL Azure**. The SQL Azure name is still what many call Azure SQL today (just ask Conor Cunningham). The programming and usage model were the same as SDS (except T-SQL and the TDS protocol were adopted instead of REST), the hosting was the same, but the name SQL Azure was the go-to market brand.

On February 1, 2010, it all became official. Windows Azure was officially launched and, along with it, the first truly PaaS relational database service in the industry, SQL Azure (you can read the official blog announcement at [`https://blogs.microsoft.com/blog/2010/02/01/windows-azure-general-availability/`](https://blogs.microsoft.com/blog/2010/02/01/windows-azure-general-availability/)). Along with the announcement was a new logo (changing the current SQL Server 2008 logo from red to blue) as seen in Figure [1-5](#496204_2_En_1_Chapter.xhtml#Fig5).

![](images/496204_2_En_1_Chapter/496204_2_En_1_Fig5_HTML.jpg)

Figure 1-5

The original SQL Azure logo

In order to interact with Windows Azure, the team had to also snap into a user interface experience called a *portal*. The first version of the Windows Azure portal used HTML, but quickly after this, a new portal experience based on Microsoft Silverlight was adopted. This also included a separate Silverlight “administration” experience for SQL Azure.

Figure [1-6](#496204_2_En_1_Chapter.xhtml#Fig6) shows an example SQL Azure management portal based on Silverlight.

![](images/496204_2_En_1_Chapter/496204_2_En_1_Fig6_HTML.jpg)

Figure 1-6

The SQL Azure Management Portal

When Windows Azure was launched, the concept of an *Azure data center* was introduced to our customers. A data center is a physical set of buildings in a specific geographic location where Microsoft hosted Azure services. The names of the data centers were based on a geographical region (we have since shifted to a concept of *regions* and data centers which I’ll explain later in this chapter and in other places in the book). At the original launch of SQL Azure, customers could deploy databases in data centers with names of North Central US, South Central US, East Asia, and North Europe.

The original SQL Azure had some interesting characteristics:

*   We launched with a business model that had two *editions*: Web and Business. The basic difference was the maximum database size: 1GB for Web and 10GB for Business (as you will see in this book, you can create sizes much larger than this now). We quickly bumped this up to 50GB by June 2010.

*   In order to deploy a database, you would deploy first a *logical database server*. Multiple databases could be associated with a logical server. The logical server also contained other metadata such as logins and firewall rules for security.

*   Any table in a database was required to have a clustered index.

*   We used our own internal “replica system” but ensured that we always kept three replicas available. We also automated processes like backups and kept multiple copies.

*   We updated the software for SQL Server through a concept called a Service Update (SU) and made announcement about these updates in blog posts (an example blog post for a service update can be found at [`https://web.archive.org/web/20140420195848/http://blogs.msdn.com/b/sqlazure/archive/2010/02/17/9965464.aspx`](https://web.archive.org/web/20140420195848/http:/blogs.msdn.com/b/sqlazure/archive/2010/02/17/9965464.aspx)).

*   We introduced the concept of a Service-Level Agreement (SLA) to ensure a level of database availability.

*   Early on, we developed an integrated experience with the popular tool, SQL Server Management Studio (SSMS).

*   Customers struggled with concepts like application retry logic, new error messages, logical master, throttling, and inequality with the T-SQL surface area of SQL Server.

Note

If you want to step back in time and see some older blogs about SQL Azure, visit [`https://web.archive.org/web/20140410165353/http://blogs.msdn.com/b/sqlazure/default.aspx?PageIndex=1`](https://web.archive.org/web/20140410165353/http:/blogs.msdn.com/b/sqlazure/default.aspx%253FPageIndex%253D1) and traverse the links at the bottom of the page.

In these early days for both Windows and SQL Azure, it was even a new world within Microsoft. Buck Woody worked on the original Windows Azure teams. He told me that working on Azure was in a group at Microsoft called “Incubation” – a startup-like culture. “One of the most interesting parts of that,” he said, “was seeing everything getting built in the technology, and in the business side. It was probably the best MBA I ever got.” In Incubation, you were “walled off” from the rest of Microsoft, having your own engineers, sales, marketing, and the like. When the product showed a profit and all the business processes were established, it “graduated” to the rest of the field at Microsoft. Some products graduated, and others didn’t – so there was a lot of pressure to succeed.

### The SAWA Project

To this point in time, SQL Azure still was deployed and ran using the AutoPilot cluster system with SQL Server instances hosted on bare-metal servers (Brian Chamberlain, one of the principal engineers for Azure SQL, calls this internally SQL Azure v1).

We knew as a team we needed to snap into the Windows Azure hosting system which uses virtual machines to deploy services. We needed to take more advantage of what Windows Azure offers for deployment, networking, and storage without making wholesale changes to the SQL Azure architecture. Therefore, a project was born called **SAWA** (SQL Azure on Windows Azure). Brian calls this SQL Azure v2\. In order to help abstract the team from having to deploy on both AutoPilot and SAWA systems, we built a software layer code-named **Blackbird**.

The SAWA project was important because it would allow the team to eventually become a full-fledged Azure service taking advantage of everything internally that Windows Azure provides to services. But for a few years, the team operated and managed SQL Azure Databases deployed on both AutoPilot and SAWA. Users didn’t see the difference. The service still looked and behaved the same.

For the next few years, Windows Azure offered compute services through cloud services and database services through SQL Azure. The SQL Azure team had also added other engineering leaders to the team, including Nigel Ellis and Peter Carlin. It was the beginning of the journey, but Microsoft leadership was behind the scenes already working on changes and bigger things to push Azure further in the public cloud.

## The Virtual Machine Initiative

When Windows Azure was first released, one of the primary target solutions were scalable web applications in the cloud. Therefore, cloud service was the first compute service in Windows Azure. As I described in the section above on Windows Azure, cloud service applications ran in virtual machines and had the ability to store data in SQL Azure or in Azure Blob Storage using APIs. But application developers did not have access to any local storage or full virtual machine access. The concept of a virtual machine in the cloud as a service, also called **Infrastructure as a Service (****IaaS****)**, had been introduced by Amazon in 2006 called Amazon Elastic Compute Cloud (EC2) as part of their overall Amazon Web Services (AWS) suite. EC2 was literally a virtual machine where you can deploy your choice of operating system and application.

For many, cloud services in Windows Azure were still thought of as Platform as a Service (PAAS) since developers didn’t really have access to the entire guest VM or concepts like local storage. Our Windows Azure team knew we needed something to compete with EC2 and make IaaS a big part of Azure.

In 2011, Microsoft decided to make a bold move in leadership. Steve Ballmer wanted to make big bets in the cloud, including Azure. He asked Satya Nadella to move from his current position leading the Bing team to run the division at Microsoft called Server Tools and Business (STB). This was the chief enterprise software group at Microsoft that ran Windows Server, SQL Server, and Windows Azure. As part of his role in taking over STB, Satya did several key things. First, he hired Scott Guthrie to lead the engineering efforts for Azure. Scott is now the leader of Cloud and Enterprises, also known as C&E, which used to be STB. Second, he hired Jason Zander to lead the Azure core infrastructure team. And third, he empowered Azure CTO Mark Russinovich to build the road map for the future. And one of the first bold moves of this team was to launch into preview Azure Virtual Machine (VM) to provide a true IaaS service offering for Windows Azure.

One of the first key workloads to showcase Azure Virtual Machine would be SQL Server. I remember these early days of Azure VM as I was assigned in Microsoft support to look at the supportability of SQL Server in this environment. It was at that point I met the lead program manager for SQL Server on this effort, Guy Bowerman.

When Guy joined the SQL team around June of 2010, he found out about cloud services with worker and web roles, but also saw we had announced the concept of a *VM role*. A VM role allowed a user to upload a virtual machine image (VHD) and run their VM. However, the VM role didn’t provide the richness of a true IaaS solution. The VM role, for example, did not provide local persistent storage for the operating system or attached persistent drives.

Throughout 2011 and 2012, the Windows Azure team worked with groups like SQL Server to launch a new Azure Virtual Machine preview program. Azure Virtual Machine was officially launched in April of 2013\. Azure Virtual Machine was a significant move for Microsoft. “Opening up” the virtual machine now allowed users to deploy the operating system of their choice, including Linux (this move would set the stage for a little project you may have heard about called Helsinki or SQL Server on Linux).

The new Azure Virtual Machine platform provided all types of benefits for running SQL Server, including a dedicated “OS disk,” a temporary disk for storing files like tempdb, and persistent storage for SQL Server database and log files called *data disks*. Even though the choices were limited, there were various virtual machine *sizes* customers could choose to deploy SQL Server, including the number of CPUs, memory, number of disks, and maximum capacity. In addition to providing these choices for the virtual machine configuration, the Windows Azure team provided a system where teams like SQL Server could provide customer choices for a fully deployed virtual machine with SQL Server preinstalled through a concept called *gallery images* or a *marketplace*. Now a user could choose a virtual machine configuration and a version of SQL Server, make a few other choices, click a button, and within 10–15 minutes have access to a fully deployed SQL Server in a virtual machine hosted in Azure. You could then use a program like Remote Desktop, connect into the VM, and off you went. In addition, Azure Virtual Machine services included SLAs and Availability Sets (update and fault domains).

The initial launch of Azure Virtual Machine used the same Windows Fabric that hosted cloud services. The SQL Server team was effectively an “internal customer” of Windows Azure to deploy virtual machines. The main interface and system in place for the SQL team to deploy was called RDFE (Red Dog Front End). This system later affectionally became known as *classic* virtual machines. Today, the classic system has been replaced with the Azure Resource Manager (ARM) system, which you will learn more about in various places in the book.

While the initial Azure Virtual Machine classic system was popular with customers, it presented issues for the SQL Server team. Disk performance stood out as an issue, and I remember in the early days of Azure VM as a Microsoft support engineer working with customers on trying to solve these problems. In addition, using RDFE presented some challenges to deploy virtual machines with SQL Server and provide robust programming interfaces to deploy and manage virtual machines.

Still the service was popular and important to the success of SQL Server in Azure. Now customers who didn’t feel that SQL Azure could meet their requirements had another choice. They could still host a SQL Server in the public cloud in Azure with virtual machines. As Guy told me, “The SQL Server on Azure VMs offering proved to be one of the most popular and successful offerings after the announcement of Azure VM.”

## Becoming Azure SQL Database

By the summer of 2012, Microsoft started branding SQL Azure as **Windows Azure SQL Database**. There was no official branding news that I could ever find. My research and internal discussions on our teams were that we just decided to start calling the service **SQL Database** to highlight the fact that the service is all about “Database as a Service” abstracting the details of SQL Server instance from the user.

In 2014, Microsoft changed the branding of Windows Azure to **Microsoft Azure**, or just Azure, so the current name of Azure SQL Database came to life.

Note

The branding of Microsoft Azure was significant to the future of Azure. Windows was and still is a dominant force for operating systems. However, since the launch of Azure Virtual Machine, Linux virtual machines have become incredibly popular. With the branding of Microsoft Azure or just Azure, we were sending a signal to the industry that Azure is more than just a Windows cloud.

As SQL Azure started to mature, other engineers from the “traditional” SQL Server team started to come onboard, including Conor Cunningham. One of Conor’s goals was to work directly with customers to make them successful with SQL Azure. This included internal customers such as **Team Foundation Services** (TFS). Conor to this day still works with TFS (which has morphed into Azure DevOps) and their success with Azure SQL. According to Conor, “They are one of our best ISVs using the platform and they help us make SQL Azure better every day.”

There were also important external customers who wanted to harness the power of Azure. One of the largest and most notable customers Conor and many on the team worked with was **Pottermore**. Around 2012, all the *Harry Potter* movies had been released, but the popularity of the books and movies was massive. Therefore, the Pottermore company decided to build a website experience for fans. And they chose Windows Azure and SQL Azure to power the website experience (see the full story at [`https://news.microsoft.com/2012/06/06/pottermore-new-website-based-on-the-hugely-popular-harry-potter-books-uses-windows-azure-to-scale-up-to-1-billion-page-views-in-first-two-weeks/`](https://news.microsoft.com/2012/06/06/pottermore-new-website-based-on-the-hugely-popular-harry-potter-books-uses-windows-azure-to-scale-up-to-1-billion-page-views-in-first-two-weeks/)).

Note

Pottermore (see [`https://en.wikipedia.org/wiki/Pottermore`](https://en.wikipedia.org/wiki/Pottermore)) is actually a company, and the previous Pottermore website is now officially wizardingworld.com.

Pottermore was an interesting test for the SQL Azure team. This project involved many databases and concurrent users. As Conor tells it, “We clearly didn’t want to disappoint all the Harry Potter fans of the world, but we also learned a lot about how to design things that scale.” As with any innovation, projects like these were ambitious but also became a foundation of knowledge to improve the future.

Microsoft support also experienced big shifts to deal with the cloud. My longtime colleague and friend Keith Elmore (the famous author of the popular ostress tool), formerly in CSS but now in the engineering team with me, was involved in Azure support from the beginning. He told me that supporting Azure had some interesting challenges but also opened new possibilities. Keith said, “There was a major shift in how we could troubleshoot. We no longer had to ask customers to capture a lot of troubleshooting data as we did in the on-premises environment but could immediately access a large set of Azure telemetry to assess the general nature of their problem, and often narrow down the specific issue.” Supporting the cloud presented a new expectation from customers to deliver a solution fast. Microsoft now owned the “back end” so data that customers normally had completely control over obtaining was not possible for them in Azure. According to Keith, “the expectation of resolving a large percentage of your support cases in a day or less was radical. The team rallied around how we could organize ourselves and leverage this Azure telemetry to resolve most issues in one day.” You will learn about some of the resources Azure provides customers for support and troubleshooting later in this book.

### The Sterling (SAWAv2) Project

By 2013 and 2014 timeframe, Azure SQL Database had many different successful customers, but the legacy architecture even running on SAWA was starting to show its age through different problems and customer experiences like TFS and Pottermore:

*   As much as we made changes in the infrastructure, one thorny problem cropped up all the time, called the *noisy neighbor* problem. A customer could consume resources for one database that could adversely affect another. We needed a solution where each tenant (database) had their own SQL Server instance. This would allow us to use features like SQL Server Resource Governor.

*   We also felt a lot of pressure to open more of the T-SQL surface area for SQL Azure customers. Since multiple tenants shared a SQL Server instance, this was a major problem. For example, how do we present a Dynamic Management View (DMV) of just your database when they by design show anything on the underlying instance?

*   We also needed a model where customers would expect more predicable performance since they had no way of choosing things like the number of CPUs, memory, or I/O speed.

*   Our codebase for Azure SQL Database was still forked from SQL Server 2005 (yikes!). For the SQL team to become truly *cloud first*, we needed to merge the codebase of SQL Azure and SQL Server.

*   We still used local disks for everything (including user databases, telemetry, etc.) with our own custom replication system. Even with SAWA, we were using custom hardware to support large disk needs as all our storage was local using spindles (non-SSD drives). We needed to move toward hardware generations that were aligned with all of Windows Azure. However, the generation of hardware at this time only supported very small local SSD drives. Therefore, we needed a strategy that allowed for “remote” databases using Azure Storage.

*   *Windows Fabric* which hosts and powers most Azure services internally has many built-in capabilities to support deployment, scaling, networking, storage, high availability, and fault tolerance. We knew to enable new models and options like Azure Storage for databases, Azure SQL Database needed to become a *WinFab application* (or Windows Fabric application).

As early as 2011, team members, including Morgan Oslake, were looking to solve the noisy neighbor problem with concepts like *resource reservations* and *node isolation*. Resource reservations were implemented with a concept called a **Service-Level Objective (SLO)**. Even today, you can see the term SLO in some of the diagnostics for Azure SQL. This work led to several innovations which would shape the future. As Morgan tells it, “The initial solution also set the stage for enabling true scaling elasticity. Incidentally, this project is also where IO Resource Governor was born in partnership with Microsoft Research and the solution was eventually integrated into the SQL Server boxed product.”

As SQL Server 2014 was being developed and launched, the Azure SQL Database team internally started working on a project called **Sterling** (also known as project Dearborn or SAWAv2). At this time, SQL Azure was known as *v11* (the @@VERSION string had 11.x in it).

Note

The name of Sterling comes from an interesting source. As Peter Carlin tells it, “The name of the project was supposed to be after Stirling, a well-known efficient heat engine invented by Robert Stirling in the 1800s. Ironically, the project name got misspelled to Sterling, but the name stuck.”

For the team, Sterling effectively became a rewrite of the architecture of the service while still maintain the principles of a database service for customers. The next generation of Azure SQL Database would also get a “version bump” to highlight this new architecture called **v12**. v12 also included a merge of the SQL Server codebase. Code fixes and new features could be done in a single branch that would be used for both SQL Server and Azure SQL Database. The v12 name was confusing for customers because it did not line up to a specific version of SQL Server. We named it v12 because with this new architecture we opened more SQL Server features like columnstore and instance-level diagnostics. We didn’t want to break applications, so we changed the major version to v12 (which corresponded to the SQL Server version number 12.x of SQL Server 2014). Since this time, Azure SQL Database has become a *versionless* SQL Server, which I’ll describe more in Chapter [5](#496204_2_En_5_Chapter.xhtml) of the book.

Rohan Kumar, the former Vice President of Azure Data Engineering, was leading engineering efforts in Azure SQL around this period. He says about the codebase merge, “Probably the most important decision technically we made was to unify the codebase.” In fact, Rohan was assigned to lead this project, which took some 18 months while we were maintaining and running the service and delivering releases of SQL Server at high quality.

The Sterling architecture involved running the SQL Server instance (and other needed programs) as a WinFab application or effectively as worker roles in the Windows Azure nomenclature. One interesting aspect to the deployment of Azure SQL Database for Sterling was that only a single virtual machine is used per host with one or more SQL Server instances, each hosting a tenant database.

Note

You will see later in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book that Azure Managed Instance can combine more than one virtual machine per host.

Figure [1-7](#496204_2_En_1_Chapter.xhtml#Fig7) comes from a diagram Peter Carlin built to show the primary difference between the SAWA (and AutoPilot) and Sterling architectures from a SQL Server instance, logical server, and database perspective.

![](images/496204_2_En_1_Chapter/496204_2_En_1_Fig7_HTML.jpg)

Figure 1-7

Sterling architecture database isolation

In this diagram, a node is a virtual machine or server. An instance is a SQLSERVR.EXE instance. The cluster is a logical collection of machines hosting virtual machines. A logical server is really a set of metadata describing the collection of databases.

Notice in the “Old Service” (SAWA and AutoPilot) that databases from two different logical servers could be deployed on a single SQL instance. With Sterling, each instance was reserved for a single logical server, but multiple instances could (but doesn’t have to) be deployed on a single virtual machine (or node).

Note

Today, almost every Azure SQL Database has its own dedicated SQL instance with a few exceptions, which I’ll cover in Chapter [4](#496204_2_En_4_Chapter.xhtml).

Using Windows Fabric also allowed us to provide better resource governance closer to the operating system, provide direct connections to the back-end SQL Servers, and leverage Azure Storage for databases and backups. Windows Fabric also provided architectures for fault tolerance and networking in the form of *clusters*, *nodes*, and *rings*. You will learn more about these concepts as well as other aspects of the Azure SQL Database architecture throughout the rest of this book. You will also learn in this chapter and the rest of the book how we have created other deployment options based on the Sterling architecture, including Elastic Database, Managed Instance, Hyperscale, and Serverless. After a long journey, Azure SQL Database V12 became generally available in the spring of 2015.

In addition to work on Sterling, Microsoft Azure had pivoted to a *new portal experience* based on HTML (moving away from Silverlight) for all Azure services, including Azure SQL Database. Figure [1-8](#496204_2_En_1_Chapter.xhtml#Fig8) is a snapshot of Scott Guthrie showing off the new portal in one of his blog post archives.

![](images/496204_2_En_1_Chapter/496204_2_En_1_Fig8_HTML.jpg)

Figure 1-8

The Windows Azure admin portal

Tip

Given Scott Guthrie’s long role in Azure, the archives of his blog are incredible to tour the history of Azure! You can find them at [`https://weblogs.asp.net/scottgu/archive`](https://weblogs.asp.net/scottgu/archive).

### New Editions, the DTU, and Previews

Independent of the new V12 architecture, we also realized the current editions of Azure SQL Database – Web and Business – were outdated both from a payment model and a predictable performance and choice perspective.

The SLO work that had started before Sterling ultimately led to a preview of a new edition called **Premium**. In some cases, Premium customers were isolated to a single node to provide maximum performance. Along with the Sterling architecture, the concept of resource reservations and node isolation paves the way for a new edition suite and a self-service method to choose *sizes*.

In April of 2014, we announced a new pricing model based on new editions, **Basic**, **Standard**, and **Premium**, and concept to materialize sizes called *performance tiers*. Performance tiers offered granularity within an edition for maximum database size and a SLO. To support a SLO, we introduced a new concept called a **Database Transaction Unit (****DTU****)**. We started this new model with a preview of these editions and tiers in April 2014\. By this time, we allowed up to a 500GB for Premium editions.

See the blog post and video of Scott Guthrie talking about these new models at [`https://weblogs.asp.net/scottgu/azure-99-95-sql-database-sla-500-gb-db-size-improved-performance-self-service-restore-and-business-continuity`](https://weblogs.asp.net/scottgu/azure-99-95-sql-database-sla-500-gb-db-size-improved-performance-self-service-restore-and-business-continuity).

A DTU was a logical concept of measurement for a combined resource usage of CPU, I/O, and memory. The idea was to provide a metric to obtain more predictable performance and pay for resource usage. You will learn more details about SLO, tiers, and DTU throughout the rest of the book. By spring of 2015, we had also announced the retirement of Web and Business editions and had fully rolled out Basic, Standard, and Premium editions.

With these new editions, we also introduced the concept of self-service database restore (point-in-time restore or PITR) and active geo-replication (our way of introducing Always On Availability Group replicas in the cloud).

Another concept introduced during these years (and across all of Azure) was **private** and **public preview**. The SQL Server team had already shifted to the term Community Technology Preview (CTP) vs. “beta” builds before they released a version of SQL Server. New Azure services and enhancements to existing services started rolling out in previews. Since Microsoft hosted the software in the public cloud, they had the ability to *provision* specific customer subscriptions to use specific services and even features for services. A private preview required a customer to sign up to gain access to a new service or enhancement. A private preview was often free, limited in availability, and involved direct interaction between engineering and a customer (and the customer was in a non-disclosure agreement with us). A public preview was often open to the public. For a new service, it was sometimes free but most often involved a significantly discounted price. In most cases for a new feature for a service, public preview was free. Previews allowed Azure teams to move agile and fast, gain customer adoption and feedback very quickly, and eventually move to *General Availability (**GA)*. You will learn in this book as a customer how to keep up with previews and announcements for Azure services and enhancements.

Note

You will read terms in the book like “went GA” or “went public preview” in the book to note when a service or feature was released.

The preview system was also a great example of a new approach for the SQL team, namely, a *cloud-first* approach. Cloud first means that the SQL team could build out a new service or feature first in the cloud and then eventually allow that feature to appear in the next major release of SQL Server. Previews combined with a merged SQL codebase allowed for these types of motions to happen.

As Peter Carlin tells it, with the cloud-first approach, “…we use service telemetry to learn what is wrong and needs to be improved, use that telemetry as we build and refine it via iterative deployments and then when we know it works well for the scenarios.”

### Intelligent Performance and the MDCS

By the 2013 timeframe, the SQL engineering team had hired resources outside of Redmond. Microsoft had built a development center in Serbia called the Microsoft Developer Center Serbia (MDCS). You can read more about MDCS at [`https://www.microsoft.com/sr-latn-rs/mdcs`](https://www.microsoft.com/sr-latn-rs/mdcs). Our Azure SQL Database team assigned engineers from MDCS to form a data science team. One of the first tasks for this team was to investigate how to provide *value-added services* for Azure. Azure SQL Database was launched as a true PaaS service. Abstracting the developer from the details of SQL Server was important, and providing built-in HA and predictable performance were critical. However, our team wanted to see what type of additional services we could offer customers as part of the platform.

Performance is perhaps one of the toughest problems within SQL Server to solve given how vague an issue can be (ever heard of “it is just slow”). Add to that the open nature of T-SQL and databases (bad indexes + bad queries = poor performance). Engineers Vladimir Ivanovic and Miroslav Grbic embarked on a project to see how machine learning at scale could improve performance for Azure SQL Database. As Miodrag Radulovic tells it, “The original intention was to leverage data science and ML at Azure scale to find a way to improve customer experience of using Azure SQL Database. Performance optimization was identified as one of the areas where the team could deliver pretty impactful improvements, especially for those novice users who are not that skilled in perf optimizing SQL Server engine.” Vladimir specifically said that index recommendations for performance were an area the team felt important to tackle. SQL Server had technologies to assist with index recommendations, namely, Dynamic Management Views and a tool called Database Tuning Advisor (DTA). Vladimir says, “We picked index recommendations since the technology was already partially available via missing indexes DMV and also DTA, and our initial analyses showed that a significant number of SQL DB customers could benefit from this.”

The work for this project took some time to get it right. The work started in 2014 and was released for public preview in July of 2015 known as *Database Tuning Advisor*. The functionality would use machine learning combined with existing SQL Server resources (including the Query Store) to recommend and even auto create/drop indexes. In January of 2016, the experience went GA and was named Automatic Tuning. You will learn more about the details of Automatic Tuning in Chapter [7](#496204_2_En_7_Chapter.xhtml) of this book.

### Advanced Data Security and the ILDC Team

In addition to adding services for Azure SQL Database for performance, the team wanted to create new experiences for security. Microsoft had formed the first Research and Development Center outside the United States in 1991 in Israel called the Microsoft Israel Development Center (ILDC). You can read more about the ILDC at [`https://www.microsoftrnd.co.il/`](https://www.microsoftrnd.co.il/).

In 2014, the Azure SQL team turned to the ILDC to look more into security to form a group called the *Azure Security Center for SQL*. When first formed, this group, according to one of the original members Ron Matchoro, was chartered to look at security topics like auditing, data masking, vulnerability assessments, and threat protection techniques.

This work led to several innovations for both Azure SQL and SQL Server. The team first landed the concept of *Dynamic Data Masking* in Azure SQL in 2015 and in the SQL Server 2016 release (read more about Dynamic Data Masking today at [`https://learn.microsoft.com/sql/relational-databases/security/dynamic-data-masking`](https://learn.microsoft.com/sql/relational-databases/security/dynamic-data-masking)).

The team then accelerated further by enhancing auditing (SQL Server already had a concept called SQL Server Audit) for Azure SQL, delivering on a method to perform vulnerability assessments (which would also land in SQL Server Management Studio). The ILDC also invested in a concept for data classification which now exists in Azure SQL and SQL Server.

Perhaps the biggest area of investment was in *threat protection*. The concept was to use the power of the cloud to detect possible threats to an Azure SQL deployment and alert administrators. This included concepts like detecting *SQL injection* attacks. This capability went GA with Azure SQL in 2017 as **Advanced Threat Protection (ATP)**. In 2019, the team grouped together a series of capabilities, including ATP, vulnerability assessment, and data classification called **Advanced Data Security (ADS)**. These capabilities would get rebranded and expanded into **Microsoft Defender for Cloud** which includes coverage for SQL, ground to cloud. You will learn more about Microsoft Defender in Chapter [6](#496204_2_En_6_Chapter.xhtml) of this book. Today, the ILDC continues to work with our teams in Redmond to deliver new security capabilities for Azure SQL.

### A Pane for the Future Called Ibiza

Microsoft also decided around the 2014 timeframe the current Azure portal experience needed a new look (yes, again). Project **Ibiza** was a new Azure portal with a completely new look and design. This was effectively the fourth generation of the Azure portal. The Ibiza portal was also known early on as the “Preview Portal.” This new portal used a concept called *blades* as a user interface experience. Users reported very early on this portal was more reliable, faster, and an overall better user experience that included a dashboard and pins and supported all the various Azure services.

Figure [1-9](#496204_2_En_1_Chapter.xhtml#Fig9) shows the Ibiza portal at launch in 2015\. Today it is simply known as the **Azure Portal** (accessed by almost anyone from portal.azure.com).

![](images/496204_2_En_1_Chapter/496204_2_En_1_Fig9_HTML.jpg)

Figure 1-9

The original Ibiza portal

This portal serves as the *foundation* of the current Azure portal but with many enhancements and a different look and feel which you will see throughout the chapters of this book.

### A New Engineering Model for Azure

Even since the launch of SQL Azure, a unique experience existed within Microsoft. For SQL Server, the engineering team mostly focused on designing and building a new release. They absolutely take in customer feedback as they build new features or resolve problems, but their lenses mostly come from feedback forums or Microsoft technical support.

With the launch of a service, the engineering team now owned the operation of SQL Server in the cloud. They built the software but also managed the operation of a data center. While other teams owned the overall operation of data centers, the SQL Azure engineering team owned the health, cost, and operations of the Azure SQL Database service. This involved all types of proactive monitoring for health and reliability. But it also meant the team owned updating all the software behind the scenes that powered the service.

SQL Azure engineers were now involved in *Live Site* experiences, which today can be thought of as a *DevOps model*. If an outage occurred, the SQL Azure engineering team was directly involved in resolving the problem. These experiences over time drove innovation and automation. Much of the functionality behind the scenes that are part of the Azure SQL Database ecosystem were built to avoid manual intervention of problems. Even new features introduced both for Azure SQL Database and SQL Server came from Live Site experiences for the team ensuring the service was healthy, applications were performant, and databases were reliable and available. Peter Carlin describes the benefits of Live Site: “...basically everything we have built in the last 5 years are driven by learnings from live site. In many ways, we didn’t know how to run our own product, and once we realized how hard we had made it, could make the changes to make it much easier to operate – benefiting all SQL DBAs.”

As Rohan Kumar tells it, “One of our biggest challenges was team culture. We needed to create a team that could not only build great software but also operationally run it.”

By 2015, Azure SQL Database had a robust architecture for the future with v12 and innovation to add value for both performance and security. As new customers built applications with the service, feedback and Live Site experiences drove future innovation, both for the architecture and new deployment models.

## Bending Azure SQL Database

As we announced the introduction of Basic, Standard, and Premium editions and were phasing out Web and Business editions, we had some customers, mostly ISVs, that brought us a dilemma. Web and Business editions charged only for storage, not for compute, but there was no predictable performance. Basic, Standard, and Premium editions were paid by DTU, not just storage. Some ISVs wanted to host many databases, sometimes 1000s, to support their application, many of them Software as a Service (SaaS) applications. The new edition model with DTUs would now become cost prohibitive. Not all 1000 databases needed the same DTU capacity, but more importantly, the needed DTU usage could vary widely across these databases. The only way to support all these databases to provide required performance would be to pay for the maximum DTU needed for any database.

Around the 2014 timeframe, program managers Morgan Oslake and Tobias Ternstrom were assigned to come up with a solution where according to Morgan, “We (SQL DB) needed a price-perf optimized solution for SaaS ISVs with apps containing 10s, 100s, 1000s or more databases.” Tobias proposed the project named *Malmo* (as Morgan recalls, “…the name was motivated by the observation that Malmo, the city in Sweden, was growing rapidly and bursting in population size. In any case, a concept of Malmo is to more efficiently accommodate bursting episodes of multi-database apps”). The team moved fast to a private preview of the capability for databases to be grouped together called an **elastic pool**. We moved to public preview in April of 2015 and General Availability in 2016\. Read the announcement at [`https://azure.microsoft.com/updates/azure-sql-database-supports-large-numbers-of-databases-for-saas-providers/`](https://azure.microsoft.com/updates/azure-sql-database-supports-large-numbers-of-databases-for-saas-providers/).

Elastic pools allowed a developer to group databases together in a pool and consume and pay for usage as an elastic DTU or *eDTU*. You will learn more about how elastic pools work in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book. Having elastic pool as an offering also helped pave the way to deprecate and remove the Web and Business edition model.

With new editions, the Sterling architecture, DTUs, and elastic pools, many of what customers needed to adopt Azure SQL Database were in place. However, some customers using the SQL Server “box” product were resistant. As we polled and talked to these customers, we discovered the *surface area* of Azure SQL Database didn’t meet their core requirements. By 2016, we decided we needed to develop another option to enable more customers to adopt Azure SQL.

## Lifting Customers to the Cloud

With the success of the MDCS team working on Automatic Tuning, our team turned to them to work on another project to help reduce the friction from migrating SQL Server instances to Azure. One of the leaders in MDCS, Drazen Sumic, told me the origin of the project. He said “Lindsey Allen, one or our leaders in Azure SQL, was on a flight back from Microsoft Ignite in 2016 after receiving tons of feedback on Azure SQL and came up with an idea to *lift* customers to the cloud.” By December of 2016, the MDCS team was working on project *CloudLifter*.

By 2016, Azure Virtual Machine offered many choices to deploy a full instance of SQL Server. However, a user must still own the management and every aspect of the guest OS and SQL Server. No Azure service existed to provide some of the benefits of PaaS but also *feel* like a SQL Server instance. That is what Lindsey proposed to Drazen and the MDCS team. The team had to find a way to deploy and *expose* a SQL Server instance within the PaaS architecture of Azure, including integration with Service Fabric. Users would then connect to the SQL Server instance and use it just like a regular SQL Server on-premises or in Azure Virtual Machine. In addition, the new service still needed to provide the benefits of PaaS, such as built-in high availability and SLAs.

The team spent almost the entire 2017 year in a private preview program for a new service called **Azure SQL Database Managed Instance** (or often called just Managed Instance). Many folks in MDCS worked on this project, including Borko Novakovic, Jovan Cukalovic, Branko Kokanovic, and Milan Novakovic. Public preview of Managed Instance landed in March of 2018, and General Availability for the first set of tiers was announced at the Microsoft Ignite conference in September 2018 (you can read the announcement at [`https://azure.microsoft.com/blog/azure-sql-database-managed-instance-general-purpose-tier-general-availability/`](https://azure.microsoft.com/blog/azure-sql-database-managed-instance-general-purpose-tier-general-availability/)).

Even though many folks in Redmond were instrumental in this project, including Lindsey Allen, Peter Carlin, and Alexander Vorobyov, this project was an important milestone for the MDCS team for SQL. According to Drazen, “Yes, this was the largest project to date to be driven from the Serbia team. We’re proud of it, and thankful for the trust. Previous efforts were also important (e.g. Query Store for the SQL 2016 wave) but were smaller features compared to this one.”

Managed Instance solved another aspect for increased cloud adoption, but the ability to handle very large enterprise workloads was still an issue for the team.

## Project Socrates Goes Hyper

In the fall of 2015, Rohan Kumar, who effectively owned engineering teams for both SQL Server and Azure SQL at the time, was holding a meeting with many in his senior staff talking about a recent Live Site incident involving a customer when he asked the question to the room: “If we had to build an architecture for Azure SQL Database from scratch what would that look like?” It was not like the current Sterling architecture was not good. In fact, the Sterling architecture had allowed Azure SQL to grow significantly. However, if we wanted to truly run very large-sized mission-critical workloads, the team thought something different might be needed. Something we could build on top of the existing Sterling architecture. “How can we provide no-limits scale to SQL in the cloud” was the mission Rohan gave the team.

One of the people in that room was Cristian Diaconu, one of the principal engineers who had been instrumental in the Hekaton project for SQL Server (In-Memory OLTP). Cristian talks about those early meetings with Rohan: “So Rohan kept at it saying that he wants us to think about building for the longer term, with more architectural durability and something that we’d be hanging our name to because it was differentiated in the industry.”

That meeting led to a series of discussions about a possible new architecture with engineering leaders like Hanuma Kodavalla, Tomas Talius, Donald Kossman, Justin Levandovski, Phil Bernstein, Peter Byrne, Peter Carlin, and eventual engineering leader Naveen Prakash.

Note

You can see a more comprehensive list of team members and contributors from the whitepaper written for this project at [`https://www.microsoft.com/en-us/research/uploads/prod/2019/05/socrates.pdf`](https://www.microsoft.com/en-us/research/uploads/prod/2019/05/socrates.pdf).

By May of 2016, the team had funding to move forward with their designs into a full-fledged project. They called it *Socrates* (Cristian says that “...as I was meeting a lot of folks with a ton more experience doing this than I, so it dawned on me that all I had were questions – hence Socrates”).

The Socrates concept was to build a very scalable architecture in Azure through separation of services like logging and caching services (e.g., page servers). The original Socrates architecture can be seen in Figure [1-10](#496204_2_En_1_Chapter.xhtml#Fig10).

![](images/496204_2_En_1_Chapter/496204_2_En_1_Fig10_HTML.jpg)

Figure 1-10

The Socrates architecture

The team moved quickly to turn their design into working code. By December of 2016, they had a working prototype. However, this architecture was not a trivial project to get right, so it took until September of 2018 to launch a public preview. The name of the new offer would be called **Azure SQL Database Hyperscale** (we technically call this ***Hyperscale service tier***). By May of 2019, Hyperscale was a generally available service (you can read the announcement at [`https://azure.microsoft.com/updates/azure-sql-database-hyperscale-support-for-single-databases-is-now-available/`](https://azure.microsoft.com/updates/azure-sql-database-hyperscale-support-for-single-databases-is-now-available/)).

Hyperscale literally put Azure SQL on the map at a new level even within Microsoft. Watch Rohan Kumar demonstrate Hyperscale at the keynote for the first time with Satya Nadella at the Microsoft Inspire 2019 conference ([`https://youtu.be/WtoU8gugP5g`](https://youtu.be/WtoU8gugP5g)). Hyperscale has grown in leaps and bounds since that time and has become the “default” choice for many developers for Azure SQL Database. You will see Hyperscale show up throughout the rest of this book.

## Azure SQL Today

The evolution of Azure SQL from CloudDB to Hyperscale was an amazing journey for the SQL team and Microsoft. February 1, 2020, marked the official tenth year of Windows and SQL Azure.

By this time, we had released three significant capabilities: the **vCore purchasing model**, **service tiers**, and the **serverless** database. The vCore purchasing model makes it clearer on how many CPU resources your application is using and being billed. Service tiers classified feature capabilities and resource limits and are called General Purpose, Business Critical, and Hyperscale (only for Azure SQL Database). The serverless database, which you will learn more about in the next chapter, provides a new method for *autoscale* compute and is now an important foundation of the future of Azure SQL Database.

In the four years since that time, we have also delivered enhancements for Hyperscale, more cores, more storage, new hardware generations, new Azure Storage options for virtual machines, more redundancy options, and new monitoring experiences. You will learn about all of these as you traverse the rest of this book.

In addition, our team during this time had to move internally off the original RDFE architecture to deploy our Azure SQL services to the modern ARM architecture or PaaS v2\. Moving to this architecture provided us the same benefits that modern virtual machines have, including scale sets, faster disks, load balancers, multi-NIC networking, and multiple Availability Zones, among others. This was a massive project but done all abstracted from users of Azure SQL services.

Over the last two years, we have built an acceleration of new capabilities for Azure SQL, especially for developers, which includes AI and new integrations with Microsoft Fabric. We have also created a new method to connect to the cloud called Azure Arc. These are all beyond the traditional thought of a relational database, which is why the final chapter of the book, Chapter [10](#496204_2_En_10_Chapter.xhtml), is aptly named “Beyond RDBMS.”

Azure itself has grown from a few data centers in three countries to 60 regions available in 140 countries worldwide (with 18 new regions in progress or planned).

Tip

If you want to see an interactive visual of Azure regions across the globe, visit [`https://datacenters.microsoft.com/globe`](https://datacenters.microsoft.com/globe).

Azure experiences include a robust cross-platform command-line interface called **az** CLI (you will see the usage of az CLI throughout the rest of this book) and enhancements to the Azure portal.

What started as shared databases on bare-metal servers supporting only a maximum of 10GB is now a powerful force in the industry and the future of data services in the cloud called **Azure SQL**.

Azure and Azure SQL are poised for the future for even bigger things. Azure SQL can be an excellent destination for SQL Server in the cloud. I believe it, which is why I decided to write this book. This book is intended to help you navigate how to make your journey with Azure SQL successful. The first step in that road to success is to further understand the fundamentals of Azure and Azure SQL. What do I mean when I say the phrase Azure SQL? What are all the options for Azure SQL? When and why would I choose one over the other? Read on to get answers to those fundamental questions and more.

## Summary

In this chapter, you learned the amazing journey of a small project at Microsoft to build a cloud database service to the world’s industry-leading database of Azure SQL today.

You saw the history of how Azure SQL started as a database service but expanded into virtual machines and managed instances. You learned from the creators of the service how and why they built SQL for the cloud.

Understanding the origins of Azure SQL can be fun. It is like reading a story. But now you need to start learning about what Azure SQL is and how to use it. The next chapter is your path to learning the fundamentals of Azure and Azure SQL.

# 2. What Is Azure SQL?

“Bob, what is the cloud?” I vividly remember my beautiful and talented wife Ginger asking me this question when I walked in our kitchen one evening. I paused for a second, getting ready to present my incredible and thoughtful answer, and said “Well, you see the cloud is….” Fifteen minutes later (as Ginger recalls – I thought it was just a few minutes), Ginger politely interrupted me and said “Uh…I was kind of looking for the CliffsNotes answer?” I was taken aback. How can anyone define the cloud as something so simple when it is such a complex topic and provides so much. I couldn’t let this go, so I spent the next few days researching a simpler answer that still defined the cloud. But I also wanted to make sure Ginger knew the answer to the question “**What is Azure?**”

Turns out Microsoft has an answer for this in the documentation at [`https://azure.microsoft.com/overview/what-is-azure/`](https://azure.microsoft.com/overview/what-is-azure/). **Azure** is defined as “…an ever-expanding set of cloud services to help your organization meet your business challenges. It’s the freedom to build, manage, and deploy applications on a massive, global network using your favorite tools and frameworks.” I proudly showed this to my wife, thinking I have now provided the simple answer she wanted. With a twinkle in her eye (she does that), she said “That is impressive of course but what?” Leave it to my CEO, Satya Nadella, to save the day for me. He says simply “**Azure is the world’s computer**” (see the exact quote from the keynote he presented at Build 2018 at [`https://news.microsoft.com/speeches/satya-nadella-build-2018/`](https://news.microsoft.com/speeches/satya-nadella-build-2018/)).

I then showed this quote to my wife. She responded “Yes. I got it. If you don’t have your own computer, just use Azure. Just use the cloud” (maybe we should hire her). That is what Azure is in a nutshell. Azure defines and provides the promise of **cloud computing**, including the following:

*   Scale out instead of having to scale up

*   Add and remove capacity on demand

*   Pay for what you use as you go

*   Double down on automation to reduce cost

Note

Microsoft provides a free training course on the concepts of cloud computing at [`https://learn.microsoft.com/training/paths/microsoft-azure-fundamentals-describe-cloud-concepts/?WT.mc_id=azureportalcard_Category_overview_-inproduct-azureportal`](https://learn.microsoft.com/training/paths/microsoft-azure-fundamentals-describe-cloud-concepts/%253FWT.mc_id%253Dazureportalcard_Category_overview_-inproduct-azureportal). You should also know that this course is part of a suite of free Azure training (400+) you can find at [`https://learn.microsoft.com/training/browse/?products=azure`](https://learn.microsoft.com/training/browse/%253Fproducts%253Dazure).

Before I define “What is Azure SQL?” I feel it is important for you to know more about some of the basic elements of Azure as I’ll use terms from this section throughout the rest of the book.

## The Azure Ecosystem

To properly understand all the elements of Azure SQL and go through examples in the book, it is important to know certain aspects of Azure independent of Azure SQL. I call this the ***Azure ecosystem***. This includes Azure accounts and subscriptions, interfaces such as the Azure Portal and APIs, resource management, regions, and Service-Level Agreements (SLAs). This section is not a comprehensive discussion on the topic. I’ll provide you enough information (with references), so you can understand terms and systems that Azure provides as you deploy and use Azure SQL services.

### Azure Accounts, Tenants, and Subscriptions

One of the most fundamental concepts you need to get started with Azure SQL is an Azure **account**. An Azure account is required to do anything with Azure. You may be working for a company that has already created an account for you based on your organization. If not, you can create one yourself by following the documentation at [`https://azure.microsoft.com/free`](https://azure.microsoft.com/free).

Note

While you can deploy Azure SQL resources with a free trial account, the credits for this account may not be enough to complete all examples in this book. The good news is that we have free offers and trial specific to Azure SQL.

Accounts are **identities** used to assign owners or members of one or more **subscriptions**. Azure accounts are members of an Azure **tenant**. A tenant is usually created by an organization and is exposed as a **Microsoft Entra** (formerly Azure Active Directory) directory. Each account is unique in a tenant but can be a member or owner of multiple subscriptions within the tenant.

Subscriptions are used to manage billing and Azure resources. In this book, I will not focus on the details of billing in Azure, but I will talk about various aspects of how Azure SQL services are charged. You can read more about management costs and billing in Azure at [`https://learn.microsoft.com/azure/cost-management-billing/cost-management-billing-overview`](https://learn.microsoft.com/azure/cost-management-billing/cost-management-billing-overview).

Subscriptions are a very important concept in Azure. Azure resources and access rights to resources are managed through a subscription. You will learn how to access resources and use an Azure subscription throughout this book. It is possible in your organization you are members of many different subscriptions but have different levels of access within each subscription. You will find out that some Azure resource names are unique within the scope of a subscription (while other names are unique to a scope of an Azure resource group or global to the entire Azure system).

Subscriptions also provide a very convenient method to organize resources within an organization (such as production vs. non-production). There are also various **subscription offers** that determine how you pay for Azure services. The most basic subscription offer is **Pay As You Go** which means you pay for resources in your subscription on a monthly or sometimes hourly basis. There are other offers, including Free Trials, Enterprise Agreement (EA), Cloud Service Provider (CSP), Enterprise Dev/Test, Pay-As-You-Go Dev/Test, and Monthly Azure Credits for Visual Studio subscribers. You can learn about all these options at [`https://azure.microsoft.com/pricing/purchase-options`](https://azure.microsoft.com/pricing/purchase-options).

Note

I wrote a blog post covering various ways to save costs with Azure SQL at [`https://www.microsoft.com/en-us/sql-server/blog/2024/02/15/how-sql-developers-can-maximize-savings`](https://www.microsoft.com/en-us/sql-server/blog/2024/02/15/how-sql-developers-can-maximize-savings).

Azure also has the concept of **Management Groups** which allow you to organize and set policies and access to a group of subscriptions in your organization.

One important security concept in the Azure ecosystem that applies to accounts, management groups, and subscriptions is Azure *Policy*. **Azure** **Policy** is a system that allows you or your organization to establish policies for management groups and subscriptions (or at a lower level like resource groups, which you will read about in the section below called “Azure Resource Manager (ARM)”). Example policies are built into Azure, and others can be created by you. An example of a built-in policy is to require all Azure SQL Databases to enable Transparent Data Encryption (TDE). You can read more about Azure policies at [`https://learn.microsoft.com/azure/governance/policy/overview`](https://learn.microsoft.com/azure/governance/policy/overview).

### The Azure Portal

If Azure is the world’s computer, then the **Azure Portal** is the *user interface* for the world’s computer. You may have read in Chapter [1](#496204_2_En_1_Chapter.xhtml) the interesting evolution of the Azure Portal over the years. Today, the Azure Portal is available through a web browser ([`https://portal.azure.com`](https://portal.azure.com)), which is available on all popular web browsers, or a mobile application ([`https://azure.microsoft.com/features/azure-portal/mobile-app/`](https://azure.microsoft.com/features/azure-portal/mobile-app/)).

Figure [2-1](#496204_2_En_2_Chapter.xhtml#Fig1) shows the **Home** page from the latest Azure Portal experience for my account.

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig1_HTML.jpg)

Figure 2-1

The Azure Portal home page

Note

In this book, I’ll be showing various screenshots of the Azure Portal. Microsoft employees by default use a *preview* of the Azure Portal to help test the latest updates. For the most part, your view of the Azure Portal should look similar, but there could be some differences.

There are several features of the Azure Portal I think you will find important as you see it used in this book and in your own experiences. Figure [2-2](#496204_2_En_2_Chapter.xhtml#Fig2) highlights some of these.

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig2_HTML.jpg)

Figure 2-2

Highlights to use the Azure Portal

Following the number sequences in Figure [2-2](#496204_2_En_2_Chapter.xhtml#Fig2), consider these Azure Portal features:

1.  **Dashboard** – Select the dashboard to see important resources you have *pinned* (you can pin almost anything in Azure to your dashboard).

2.  **Customize** your dashboard – Select Edit to move your pinned resources around and organize them in your dashboard. You can have multiple dashboards.

3.  **Search** – You will use this often to search for and find resources and Azure services.

4.  **Notifications** – As deploy and manage Azure resources, notifications are provided for progress, failures, and completion events.

5.  **Copilot** – Use this to launch the Microsoft Copilot in Azure.

6.  **Quick create** an Azure resource – Choose this to quickly create a resource from a popular Azure service.

7.  **Azure Cloud Shell** – Open a command prompt for PowerShell or bash within the portal experience.

I’m a “command line” person by nature (most of us who worked in computers from the 1980s are). The Azure Cloud Shell is simply amazing! The Azure Cloud Shell comes with your subscription, accessible via the portal, has many tools preinstalled (e.g., sqlcmd), and comes with free storage for a personal “hard drive” for your files. Figure [2-3](#496204_2_En_2_Chapter.xhtml#Fig3) shows an example Azure Cloud Shell for my account using PowerShell.

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig3_HTML.jpg)

Figure 2-3

The Azure Cloud Shell

You can read and learn more about the Azure Cloud Shell at [`https://azure.microsoft.com/features/cloud-shell/`](https://azure.microsoft.com/features/cloud-shell/).

Note

Microsoft offers a free online training course on how to use the Azure Portal at [`https://learn.microsoft.com/training/modules/tour-azure-portal/?WT.mc_id=azureportalcard_Category_Overview_-inproduct-azureportal`](https://learn.microsoft.com/en-us/training/modules/tour-azure-portal/%253FWT.mc_id%253Dazureportalcard_Category_Overview_-inproduct-azureportal).

### The Azure Marketplace

To deploy or consume an Azure service, you use the **Azure Marketplace**. The marketplace is a collection of services you can deploy and consume. Microsoft provides many of the services in the marketplace as a *publisher* (not all but many Microsoft services start with the word Azure). The marketplace also has solutions provided by other publishers, known as partners.

To see a list of services in the Azure Marketplace provided by Microsoft, you can choose the *QuickStart* Create Resource option in the portal as seen in Figure [2-4](#496204_2_En_2_Chapter.xhtml#Fig4).

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig4_HTML.jpg)

Figure 2-4

The Azure Marketplace in the portal

At this point, you can search the marketplace with keywords, pick from categories, or choose from popular services. If you select **See All**, you will get a list of all Azure services in the marketplace by all publishers organized by categories.

Azure services published by Microsoft are also known as **products**. You can see a list of Azure products on the Azure documentation site from [`https://azure.microsoft.com/`](https://azure.microsoft.com/) as seen in Figure [2-5](#496204_2_En_2_Chapter.xhtml#Fig5).

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig5_HTML.jpg)

Figure 2-5

Azure products

Notice the new AI-powered assistant at the bottom of this page to help you find answers about Azure*.*

### Azure API and CLI

Almost every Azure service has its own Application Programming Interface (API), protocol (TDS, for example, for Azure SQL), and tools (sqlcmd or SSMS for Azure SQL).

However, a common thread across all Microsoft Azure services is a Representational State Transfer (REST) API. **Azure REST APIs** are service endpoints that support sets of HTTP operations (methods), which provide create, retrieve, update, or delete access to the service’s resources. You can think of REST APIs as a *low-level core layer* of API across different Azure services and base functionality (like Azure Resource Manager). You can find more information about the fundamentals of Azure REST API at [`https://learn.microsoft.com/rest/api/azure`](https://learn.microsoft.com/rest/api/azure).

Note

You will see later in this book Azure SQL provides a unique method to integrate with REST API in Azure Services.

Your API usage with Azure services most likely will be at a higher layer using programming languages such as .NET, Java, or Python. Or you may use the Azure Command-Line Interface (CLI) called **az** CLI. The az CLI is a cross-platform program used for many different types of operations across a wide variety of Azure services and core operations such as accounts and subscriptions. You can get all the latest information on the az CLI at [`https://learn.microsoft.com/cli/azure/?view=azure-cli-latest`](https://learn.microsoft.com/cli/azure/%253Fview%253Dazure-cli-latest). The az CLI uses Azure REST APIs under the covers.

Tip

You can see a trace of Azure REST API from the az CLI by using the --debug option for any az CLI command.

Azure also provides a series of **PowerShell** cmdlets for managing Azure services. You can go to the central hub for Azure PowerShell at [`https://learn.microsoft.com/powershell/azure`](https://learn.microsoft.com/powershell/azure).

### Azure Resource Manager (ARM)

I’ve described an Azure service as something you consume or *deploy*. An instance of an Azure service you have deployed can be thought of as a *resource*. Technically, some Azure services result in multiple resources being created as part of a deployment. For example, you might deploy an Azure Virtual Machine, and the result is a virtual machine resource deployed plus other resources associated with the VM such as networks and storage.

The system that supports the management of Azure services as resources is called the **Azure Resource Manager (ARM)**. ARM is not a service you consume. Rather, it is a system to support your deployment and management of services across Azure.

Every system like ARM is based on *interfaces*. ARM exposes an interface both in and outside of Azure. You might remember the RDFE interface from Chapter [1](#496204_2_En_1_Chapter.xhtml) that was used previously in Azure (also referred to as the *classic* model). ARM is today the primary system used by tools and APIs, including the Azure Portal. Figure [2-6](#496204_2_En_2_Chapter.xhtml#Fig6) from the Azure documentation at [`https://learn.microsoft.com/azure/azure-resource-manager/management/overview#consistent-management-layer`](https://learn.microsoft.com/azure/azure-resource-manager/management/overview%2523consistent-management-layer) shows the interfaces of ARM.

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig6_HTML.jpg)

Figure 2-6

Azure Resource Manager interfaces

ARM provides many benefits of consistency and management across all Azure services, including

*   **Resource groups**

    One of the objects in the Azure ecosystem you will use often is **resource groups**. Resource groups are a logical collection of resources you can manage as a unit. Every Azure resource (e.g., a virtual machine or database) must exist in a resource group and can only be a member of one resource group. Resource groups have metadata, including a region location (but resources in a group can be in different regions).

    A resource group is always associated with a specific Azure subscription. Remember that I mentioned that Azure subscriptions are associated with an account and possibly management groups. You can therefore apply Azure policies at the account, management group, subscription, resource group, or resource level.

    One nice advantage of managing resources at the group level is deletion. If you delete a resource group, all resources associated with the group are also deleted. You could therefore associate all resources for a proof of concept (POC) project with a resource group and then delete the group when you are done, thereby avoiding any unnecessary costs.

*   **Role-Based Access Control (RBAC)**

    RBAC is an authorization system that provides access management to Azure resources. RBAC is based on security principals, roles, and scope. Security principals are objects like users associated with an Azure subscription. Think of security principles like SQL principles such as logins. Roles are collections of permissions based on types such as owners or contributors. Scope is the level of access such as management groups, subscriptions, resource groups, or resources. You will see examples of RBAC throughout this book, including Azure SQL–specific RBAC objects. You can read more about RBAC at [`https://learn.microsoft.com/azure/role-based-access-control/overview`](https://learn.microsoft.com/azure/role-based-access-control/overview).

*   **Locks**

    Azure locks allow you to prevent users from accidentally deleting or modifying critical resources. For example, you could create a resource group and apply a lock that prevents any user from deleting resources in the group. Valid lock types are CanNotDelete and ReadOnly.

    One good example my colleague Joe Sack pointed out to me is that locks can prevent users from accidentally dropping databases through the Azure portal (but not through tools like SSMS).

    You can read more about locks at [`https://learn.microsoft.com/azure/azure-resource-manager/management/lock-resources`](https://learn.microsoft.com/azure/azure-resource-manager/management/lock-resources).

*   **Templates**

    ARM is a *declarative* system. You use ARM by stating your intentions, and ARM does the rest. You declare your intentions through interfaces such as the Azure Portal or APIs. ARM also provides an interface mechanism to declare your intentions through a JSON file called a **template**. Templates provide a method to deploy resources consistently, repeatedly, and at scale. I’ll show you example templates you can use with Azure SQL throughout the book. You can read more about templates at [`https://learn.microsoft.com/azure/azure-resource-manager/templates/overview`](https://learn.microsoft.com/azure/azure-resource-manager/templates/overview).

Note

Azure supports other popular frameworks for managing resources at scale such as Bicep ([`https://learn.microsoft.com/azure/azure-resource-manager/bicep`](https://learn.microsoft.com/azure/azure-resource-manager/bicep)) and Terraform ([`https://learn.microsoft.com/azure/developer/terraform/overview`](https://learn.microsoft.com/azure/developer/terraform/overview)).

*   **Activity Logs**

    As you use the Azure Portal or APIs to interact with ARM to deploy and manage resources, your activities or operations are logged in a store called the **Activity Log**. Each subscription has a single Activity Log that records operations against resources associated with the subscription. Subscription-level events in the Activity Log are general (like an Azure Policy action) or specific to a resource (like updating an Azure SQL Database). You can read more about Azure Activity Logs at [`https://learn.microsoft.com/azure/azure-monitor/essentials/activity-log`](https://learn.microsoft.com/azure/azure-monitor/essentials/activity-log).

### Azure Monitor

SQL Server professionals are used to looking at metrics and logged events in tools like Windows Performance Monitor, Windows Event Log, Linux systemd logs, or Grafana dashboards. You will see in this book that Azure Virtual Machine will give you access to all these tools *inside* the guest operating system.

However, given Azure is the hosting system for all your Azure resources, it would be nice to have a centralized system for hosting, viewing, and analyzing *metrics* and *logs* of events for all your Azure resources. That in a nutshell is what **Azure Monitor** provides.

You will learn more about how to use Azure Monitor with Azure SQL services throughout this book. You can read more about Azure Monitor at [`https://learn.microsoft.com/azure/azure-monitor/overview`](https://learn.microsoft.com/azure/azure-monitor/overview).

Note

A new service called **Database Watcher** is in public preview at the time of the writing of this book which provides amazing monitoring capabilities for Azure SQL Database and Azure SQL Managed Instance ***at scale***. You will learn more about Database Watcher in Chapter [7](#496204_2_En_7_Chapter.xhtml) of this book.

### Azure Regions and Data Centers

Where does the world’s computer exist? While it is perhaps fun to think of Azure existing literally in the clouds, Azure services are hosted in physical locations called **data centers**. You might have remembered in the first chapter of the book that Azure started with four data centers.

Today, data centers are not the choice you make to deploy Azure resources. Instead, data centers are organized around the globe into **regions**. Regions are a group of data centers connected through a low-latency network. You will see in this book as you deploy Azure SQL services you will choose a region as the target for deployment. At the time of writing this book, Azure has 60+ regions worldwide available in 140 countries. Some regions have special purposes for specific customers, such as *government* or *national* regions. For example, Azure provides special regions throughout the United States called Azure Government regions (you can learn more about Azure Government at [`https://azure.microsoft.com/global-infrastructure/government/`](https://azure.microsoft.com/global-infrastructure/government/)).

A **geography** is a market of two or more regions that preserve *data residency* and compliance boundaries. To ensure further resiliency, regions are sometimes located close enough to be *paired* but far enough in distance for scenarios like natural disasters.

In addition, within some regions, one or more data centers are grouped in an **Availability Zone** to prove further high availability for Azure resources. Each zone has independent power, cooling, and networking. You can learn more about regions, geographies, and zones at [`https://azure.microsoft.com/global-infrastructure/regions/`](https://azure.microsoft.com/global-infrastructure/regions/). An Availability Zone is a key component to make your SQL deployment in cloud *highly available*. You will learn more all about this in Chapter [8](#496204_2_En_8_Chapter.xhtml).

### Azure Trust, Privacy, and Compliance

If you are thinking about moving to the cloud, you are no doubt thinking about trust, privacy, and compliance. Can I trust my data in Azure? Does Azure meet compliance for my local government and industry?

Azure has well over 100+ compliance certifications that span government regulations and industry standards. Start reading at [`https://azure.microsoft.com/overview/trusted-cloud/compliance/`](https://azure.microsoft.com/overview/trusted-cloud/compliance/) to find out all the compliance standards for Azure.

Let’s say you are concerned about privacy. Azure not only has features to help you maintain privacy and protection (learn more about Azure SQL offerings in Chapter [6](#496204_2_En_6_Chapter.xhtml) of the book), but also we at Microsoft maintain privacy standards to meet your requirements.

If your organization is concerned about trusting your data in Azure, I recommend you start with [`https://azure.microsoft.com/overview/trusted-cloud/`](https://azure.microsoft.com/overview/trusted-cloud/) and dive into how Microsoft protects your assets in our data centers and how you can use Azure tools and features to protect your deployments.

### Azure Service-Level Agreement (SLA)

Formal documents called **Service-Level Agreements** (SLAs) capture the specific terms that define the performance standards that apply to Azure. SLAs describe Microsoft’s commitment to providing Azure customers with specific performance and availability standards. There are SLAs for individual Azure products and services. SLAs also specify what happens if a service or product fails to perform to a governing SLA’s specification. Azure SQL has specific SLAs that apply to availability and performance, which you will learn about throughout this book. SLAs are one of the major reasons to consider using Azure because it is a promise from Microsoft to keep your data available. You can read more about the Azure SLA at [`https://azure.microsoft.com/blog/understanding-and-leveraging-azure-sql-database-sla`](https://azure.microsoft.com/blog/understanding-and-leveraging-azure-sql-database-sla).

## What Is Azure SQL?

Azure SQL represents a suite of Azure services for SQL Server in the cloud. It is both a collection and a *brand*, including virtual machines, managed instances, and database services. It spans both Infrastructure as a Service (IaaS) and Platform as a Service (PaaS). If Azure is the world’s computer, then **Azure SQL is the world’s database**.

### IaaS vs. PaaS

I mentioned concepts like SaaS, PaaS, and IaaS in Chapter [1](#496204_2_En_1_Chapter.xhtml) of the book, but they are worth reviewing here again.

**Infrastructure as a Service (IaaS)** is a computing system hosted by a cloud service provider like Microsoft Azure. Users of IaaS deploy and manage a virtual machine, while Azure provides the hosting infrastructure for the hardware, including host server, storage, and networking. Azure IaaS services are surfaced through Azure Virtual Machines. Azure SQL has specific options for IaaS customers.

**Platform as a Service (PaaS)** provides all the benefits of IaaS with additional benefits of cloud services usually abstracted from an underlying operating system or virtual machine. The concept is for the developer or user to focus on the application or in the case of Azure SQL a database or instance instead of details of the OS or VM. Typically, a PaaS service provides other benefits to the user, including built-in scalability, high availability, and security. Azure SQL provides two PaaS services, including Managed Instances and Databases.

One other pivot to the cloud service model is **Software as a Service (SaaS)**. A SaaS user takes advantage of using application software hosted by a cloud service provider. Many SaaS providers use IaaS, PaaS, or some combination of cloud services to support their service. Azure IaaS or PaaS services make the perfect system for many SaaS vendors. SaaS applications were the early pioneers of cloud services in the form of email systems like Hotmail. Today, Microsoft SaaS applications include the famous Microsoft 365 application suite.

### The Azure SQL Lineup

With the concepts of the Azure ecosystem and IaaS/PaaS in mind, Azure SQL consists of the following Azure services: SQL Server on Azure Virtual Machines, Azure SQL Managed Instance, and Azure SQL Database. Figure [2-7](#496204_2_En_2_Chapter.xhtml#Fig7) shows a visual of the Azure SQL *lineup*.

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig7_HTML.jpg)

Figure 2-7

The Azure SQL lineup

Note

Life in the cloud moves fast. Microsoft is looking for ways to update services for SQL in the cloud, including new branding and form factors to run SQL. Stay up to date with all the latest news for Azure SQL at our blog at [`https://devblogs.microsoft.com/azure-sql`](https://devblogs.microsoft.com/azure-sql).

There is a purpose to the order of the Azure SQL lineup left to right. As you move left to right, the *friction* is higher to migrate an existing SQL Server application to Azure, and your *control* over all aspects to the underlying SQL Server is less. However, this is not a negative thing. As you move right, you *increase capabilities for* *PaaS* *or managed capabilities* for databases. There are benefits to abstracting the details of a SQL Server as you will see when you learn all about Azure SQL Databases. As you read about each of these options for Azure SQL, keep in mind these common threads:

*   The same SQL Server engine powers each of these options.

*   The same T-SQL language you know and love works with each of these options.

*   The tools and APIs that you use today with SQL Server all work with each of these options.

#### SQL Virtual Machines

Officially called **SQL Server on Azure** **Virtual Machines**, this is your IaaS option to deploy SQL Server in Azure. You should think of this option as the *same as you are using SQL Server in a virtual machine today* except Azure hosts your virtual machine. Azure manages the host servers and hardware system. It provides interfaces for you to deploy a complete virtual machine running Windows or Linux and your choice of SQL Server. SQL Server is a complete edition (Enterprise, etc.) as you would deploy in a virtual machine in Hyper-V or VMware. Your responsibility is to manage all aspects of the guest operating system and SQL Server environment. However, because the virtual machine runs in Azure, there are benefits to assist you in managing SQL Server and the virtual machine. Chapter [3](#496204_2_En_3_Chapter.xhtml) is a complete discussion of SQL Server on Azure Virtual Machines.

#### Managed Instances

Officially called **Azure SQL** **Managed Instance**, this is an Azure service one level *up* from Azure Virtual Machine. Managed Instance is a *full* SQL Server database engine *instance* deployed as a PaaS service, hence the term *managed*. Azure provides the host server, hardware system, and virtual machines, allowing you to focus on deploying and managing a SQL Server instance and set of databases. You will see throughout the rest of this book the benefits of Azure PaaS for Managed Instance especially in the areas of security, scale, and high availability.

#### Databases

Officially known as **Azure SQL Database**, this is an Azure PaaS service up another level from managed instance. Azure provides a method to deploy one or more databases and takes care of the host, hardware system, virtual machine, and SQL Server instance. You will see that Azure SQL Database provides many different deployment options to meet unique database application scenarios. Azure SQL Database provides PaaS benefits in the areas of security, scale, intelligent performance, and high availability.

## Azure SQL Managed Instance

I described the history and purpose behind Azure SQL Managed Instance, or **Managed Instance**, in Chapter [1](#496204_2_En_1_Chapter.xhtml) of the book: make it easier to lift SQL Server applications to Azure (a.k.a. CloudLifter). You should absolutely think of a Managed Instance like a SQL Server ***Database Engine instance*** today you install on-premises. The install or deploy experience will be completely different as you will learn in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book. The experience of setting up the infrastructure to connect and network with the instance will also be different. However, once you have completed these tasks, your experience with Managed Instance will *feel* like a SQL Server instance today.

Today, when you install SQL Server, you have a single *database engine instance* of SQL Server running. You can deploy multiple databases in the instance or deploy multiple instances.

For a Managed Instance, you also will have a database engine instance running in the Azure infrastructure after a successful deployment. You then are free to perform instance-level configuration tasks, create databases, load data, and start connecting and use the instance. You are abstracted from the infrastructure details of how a managed instance is deployed. However, it can be important to know some aspects of how a managed instance is deployed in the Azure infrastructure. You will learn more about the architecture of a managed instance deployment in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book.

### Managed Instance Capabilities

Since a managed instance is like a **database engine instance** **of SQL Server**, the *surface area* of the database engine is almost 100% like SQL Server. If these capabilities of SQL Server are important to you, then a Managed Instance may be the best choice:

*   SQL Server Agent jobs

*   Database Mail

*   Cross-database transactions

*   SQL Server Replication

*   Linked servers

*   SQL CLR

*   DTC transactions

*   Resource Governor

*   Service Broker

These capabilities are not exposed with Azure SQL Database.

That is not the complete list, and we are constantly adding new features to get Managed Instance as close to a complete 100% SQL Server database engine instance as possible. You can see the latest complete list of differences between Managed Instance and SQL Server at [`https://aka.ms/azuresqlmitsqldiff`](https://aka.ms/azuresqlmitsqldiff).

Note

This is an exhaustive list of T-SQL differences. Consider in some cases it may not matter. For example, the T-SQL syntax to create and manage Availability Groups doesn’t exist for managed instance. However, this may be fine for your requirements since managed instance can automatically deploy and manage an Availability Group for you.

There are major benefits in using a managed instance because it is a PaaS service:

*   Built-in high availability (including Availability Groups), automated backups, point-in-time restore, and recovery of deleted databases. But you also can perform your own native backup (COPY_ONLY) and restore.

*   A 99.99% uptime SLA.

*   HADR across Azure regions with auto-failover groups.

*   Zone redundancy.

*   Free licensing for disaster recovery.

*   Security isolation with virtual network integration and private link networks.

*   Microsoft Entra Integration.

*   Windows Authentication.

*   Simple and easy-to-use scaling options for resources.

*   A *versionless* SQL Server constantly updated with the latest updates and features but also a *versioned* instance to maintain compatibility with SQL Server 2022.

Note

You will learn more details on what a versionless and versioned SQL Server means in Chapter [5](#496204_2_En_5_Chapter.xhtml) of this book.

*   Abstraction from the details and maintenance of the host and virtual machine environment.

*   A free trial offer for Managed Instance.

*   Integration with Azure Monitor and Database Watcher.

*   Set up an online migration strategy or compatibility with SQL Server 2002 using a Managed Instance link.

*   PaaS security capabilities such as Microsoft Defender for Azure SQL. You will learn more about Microsoft Defender for Azure SQL in Chapter [6](#496204_2_En_6_Chapter.xhtml) of this book.

As you go throughout this book, you will be learning in more detail about many capabilities of a managed instance.

### Managed Instance Options and Limits

When you install a SQL Server in a virtual machine or bare-metal server, you typically are provided preconfigured CPUs, memory, and storage resources (size and speed).

When you deploy a managed instance, you will have similar choices to make to meet your resource needs. These choices start with a concept called a **service tier**. For Managed Instance, the two service tier choices are **General Purpose** and **Business Critical**.

These tier choices will dictate your performance capabilities, resource limits, and in some cases feature capabilities. You will see more details about how to pick these tiers in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book (and you will also see the same service tier names, plus more, with Azure SQL Database). Here is a quick tour of each of these service tiers for an Azure SQL Managed Instance.

#### General Purpose

General Purpose is a service tier choice for most Managed Instance deployments, hence the term *general*. General Purpose service tier supports *vCores* from 4 to 128 (remember numbers like these can change over time as we add more capabilities for Azure). You should think of vCore as a logical CPU for your Managed Instance. Managed Instances are charged based on your vCore choice (at a fixed cost per hour).

Your choice of vCore affects other capacity choices or limits for General Purpose, including maximum memory, maximum storage for databases for the instance, and resource rates, such as Input/Output per Second (IOPS) or Log Write throughput.

Note

In early 2024, Microsoft introduced a new form of General Purpose called *Next-gen*. The Next-gen General Purpose service tier uses a new architecture to provide more vCores (128), lower I/O latency, larger transaction I/O throughput, more max storage, more databases per instance, and most importantly the ability to control IOPS separately from your vCore choices. You can read the announcement about Next-gen at [`https://aka.ms/sqlminextgen`](https://aka.ms/sqlminextgen).

General Purpose tiers also provide *basic* built-in high availability using Azure Storage (think Failover Cluster Instance), which you will learn more about in Chapter [8](#496204_2_En_8_Chapter.xhtml) on availability.

Even though a managed instance has an almost full feature set of a SQL Server, a General Purpose service tier does not support In-Memory OLTP. One of the reasons behind this is the storage requirements for In-Memory OLTP that can be only met on local storage.

Note

Columnstore indexes are available with General Purpose service tiers.

You can read all the capacities and limits for a General Purpose Managed Instance, including Next-gen, at [`https://aka.ms/sqlmiservicetiers`](https://aka.ms/sqlmiservicetiers).

#### Business Critical

Business Critical has all the maximum capacities you need for a managed instance. Business Critical choices have the same number of vCores as General Purpose but have these unique characteristics:

*   Built-in high availability with replicas using Always On Availability Groups (one free read replica is included)

*   Lower latency I/O using local storage

*   In-Memory OLTP

*   Higher IOPS and I/O throughput rates than General Purpose (but Next-gen has increased its limits significantly)

It is important to know that today since Business Critical uses local storage, your maximum database size is less than a General Purpose service tier.

There are a few limits for managed instances for a given subscription in a region (e.g., total number of vCores). You can read about these limits at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits?view=azuresql#regional-resource-limitations`](https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits%253Fview%253Dazuresql%2523regional-resource-limitations).

### Managed Instance Pools

One of the interesting aspects of Managed Instances is the deployment architecture, which is a *virtual cluster* built for a new managed instance. You will learn more about the architecture of a Managed Instance in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book. For now, know that this type of deployment can result in a much longer time (but we have made this faster than in the past with a concept called fast provisioning) to deploy new managed instances than an Azure SQL Database (for a list of expected times, see our documentation at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-overview)). In addition, because of the nature of deployment, the smallest CPU choice for a Managed Instance is 4 vCores.

Therefore, we allow the concept of **Managed Instance pools** which is in public preview at the time of the writing of this book. A Managed Instance pool provides many of the same capabilities as a Managed Instance except they allow for smaller instance deployments (2 vCores) and provide a method to deploy instances much faster. You will learn more about the architecture differences between a Managed Instance and a Managed Instance pool in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book. To read more now about a Managed Instance pool, you can read the documentation at [`https://aka.ms/sqlmipools`](https://aka.ms/sqlmipools).

### Managed Instance vs. SQL Server on Azure Virtual Machine

As you have read about SQL Server in an Azure Virtual Machine and Azure SQL Managed Instance, it might be clear to you the main differences between these Azure SQL options. However, let me provide a quick summary, so you can make decisions on which option you might choose.

SQL Server on Azure Virtual Machines is your best choice:

*   If you need to **migrate quickly** to Azure from an existing SQL Server installation. This is really a *lift and shift* of your SQL Server (which most likely is already installed in a VM) to a different VM hosting system. You will learn more in Chapter [3](#496204_2_En_3_Chapter.xhtml) about optimizing this experience and configuration.

*   If you need **full SQL Server box** capabilities like filestream, simple recovery databases, or SQL Server Analysis Services (SSAS).

*   If you want **complete control of the operating system and SQL Server**. This includes SQL Server versions back to SQL Server 2008 and a choice of Windows, Linux, or Containers.

    **Tip** Before you think you absolutely need full control of the version of SQL Server, *carefully study all the capabilities of a Managed Instance*. It may have everything you need for your SQL Server. Moving away from having to depend on a version of SQL Server or having to manage the guest OS can be a good thing!

*   **Capacity can be a factor.** If you need more than 128 vCores, 870GB memory, or a database bigger than 32TB. In addition, Managed Instance may limit things like transaction log rates or other I/O rates depending on your deployment choices. SQL Server on Azure Virtual Machine has no limits on I/O rates within the engine. The only limit would be the I/O rate that applies to storage for the virtual machine.

If these factors are not critical for your requirements, there are big advantages to use Azure SQL Managed Instance since this is a database engine instance with the full SQL Server database engine capabilities combined with the power of PaaS.

### Customers Using Managed Instance

One question I often receive about our technologies is “Who else is using this?” For Managed Instance, it is a good question. Is anyone using Managed Instance and why?

One of these customers is **Komatsu**. Komatsu manufactures construction and industry machines in multiple countries and has embraced a plan to go through a digital transformation. Komatsu has a large SQL Server on-premises footprint and chose Azure SQL Managed Instance as a cloud approach to modernize their data platform. Not only does using Azure SQL Managed Instance fit into their overall digital transformation using the cloud, but they have found both cost and performance improvements. According to Komatsu, “We determined Azure SQL Database Managed Instance was the best choice for us in terms of scalability, cost, and performance.… We’ve seen a 49 percent cost reduction and 25 to 30 percent performance gains.”

You can read the full customer story at [`https://customers.microsoft.com/story/komatsu-australia-manufacturing-azure`](https://customers.microsoft.com/story/komatsu-australia-manufacturing-azure).

## Azure SQL Database

As you read in Chapter [1](#496204_2_En_1_Chapter.xhtml) of the book, **Azure SQL Database** (formerly SQL Azure) is *where it all started*. Even though a database for Azure SQL Database deployment is installed on an actual SQL Server instance, the concept is to abstract you away from details of the instance and focus on the database. An Azure SQL Database is sometimes called a **single database**. It doesn’t mean you won’t get exposed to the feel of a SQL Server as you will see throughout the book.

### Azure SQL Database Capabilities

Azure SQL Database offers you the most complete PaaS capabilities, including the *most options for deployment of the managed Azure SQL offerings*. Like an Azure Managed Instance, an Azure SQL Database gives you access to core SQL Server database engine capabilities. However, not all database engine instance features are available. For example, columnstore indexes are available with an Azure SQL Database, but you cannot create SQL Server Agent jobs.

There are major benefits in using an Azure SQL Database because it is a complete PaaS service:

*   A free database offer (not a trial but for the lifetime of your subscription). Learn more at [`https://aka.ms/freedboffer`](https://aka.ms/freedboffer).

*   Built-in high availability (including Availability Groups), automated backups, long-term backup retention, point-in-time restore, and recovery of deleted databases.

*   Up to 99.995% uptime SLA (read more at [`https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Service`](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Service)).

*   Zone redundancy.

*   HADR across Azure regions with active geo-replication and auto-failover groups.

*   Virtual network integration and security isolation with private link support.

*   Microsoft Entra Integration.

*   Simple and easy-to-use scaling options for resources.

*   A *versionless* SQL Server constantly updated with the latest updates, optimizations, fixes, and features.

Note

You will learn more details on what a versionless SQL Server means in Chapter [5](#496204_2_En_5_Chapter.xhtml) of this book.

*   Abstraction from the details and maintenance of the host, virtual machine environment, and SQL Server instance.

*   Intelligent performance capabilities such as Automatic Tuning. You will learn more about Automatic Tuning in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book.

*   Integration with Azure Monitor and Database Watcher.

*   Azure Portal visualizations for query performance analysis.

*   PaaS security capabilities such as Microsoft Defender for Azure SQL. You will learn more about Microsoft Defender for Azure SQL in Chapter [6](#496204_2_En_6_Chapter.xhtml) of this book.

You will learn more about these capabilities and others throughout the rest of the book.

### Azure SQL Database Options and Limits

You have similar but more choices to deploy an Azure SQL Database than a Managed Instance along with different sets of resource limits. Even though you will see more details of how to pick these options and various resource limits in Chapter [4](#496204_2_En_4_Chapter.xhtml), it is worth reviewing these choices as you evaluate whether Azure SQL Database is the right option for you.

Figure [2-8](#496204_2_En_2_Chapter.xhtml#Fig8) shows a high-level decision flow for Azure SQL Database options.

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig8_HTML.jpg)

Figure 2-8

Azure SQL Database options

Notice that any option can be part of an elastic pool, but some options like Serverless only apply to General Purpose and Hyperscale vCore service tiers. Let’s explore the options and each part of this decision process.

#### Elastic Pool or Single Database

You will decide if you want just a single database or to include your database in an *elastic pool*. I’ll discuss more on elastic pools later in this section. Note that Serverless is not available for an elastic pool.

#### DTU vs. vCore

One of the first decisions you will make is a concept called a *purchasing option* for Azure SQL Database. Even though this option will also dictate your resource capacity and limit, it greatly affects how you pay for the Azure SQL Database service.

One option you can choose is called a **Database Transaction Unit (DTU)**. As I described in Chapter [1](#496204_2_En_1_Chapter.xhtml) of the book, we introduced the DTU concept as a logical concept of measurement for a combined resource usage of CPU, I/O, and memory. Your choices of resource limits and capacities for the DTU are called Basic, Standard, and Premium (with multiple levels for each of these). One nice advantage of the DTU model is simplicity. You pick a level of performance you want, and we worry about providing you the necessary cores and memory to meet the needs of your application.

Microsoft recommends for most customers to use the vCore model, so I won’t spend any other time in this book on the DTU model. You can read more about the DTU model should you want to learn more at [`https://learn.microsoft.com/azure/azure-sql/database/service-tiers-dtu`](https://learn.microsoft.com/azure/azure-sql/database/service-tiers-dtu).

The vCore model is very similar to the model for Azure SQL Managed Instance and the recommended purchasing model. In fact, using the Azure Portal will show you the vCore options by default. The vCore model gives you more choice of choosing different resources you pay for, including CPU and storage. In addition, the vCore model allows you to take advantage of cost savings options such as Azure Hybrid Benefit (AHB) licensing (which you can read more about at [`https://learn.microsoft.com/azure/azure-sql/azure-hybrid-benefit`](https://learn.microsoft.com/azure/azure-sql/azure-hybrid-benefit)) and Reserved Capacity (which you can read more about at [`https://learn.microsoft.com/azure/azure-sql/database/reserved-capacity-overview`](https://learn.microsoft.com/azure/azure-sql/database/reserved-capacity-overview)).

What is different about the vCore model from an Azure SQL Managed Instance is that you now have three *service tier options*: **General Purpose**, **Business Critical**, and **Hyperscale**.

#### General Purpose Service Tier

Like a Managed Instance, General Purpose is a service tier choice for many Azure SQL Database deployments, hence the term *general*. Unlike a Managed Instance, Azure SQL Database supports two General Purpose options called *compute tiers*: **Provisioned and** **Serverless**.

General Purpose provisioned service tiers support *vCores* from 2 to 128\. You should think of vCore as a logical CPU for your Azure SQL Database. Azure SQL Databases are charged based on your vCore choice (at a fixed cost per hour).

Your choice of a provisioned vCore affects other capacity choices or limits for General Purpose, including maximum memory, maximum storage for the database, and resource rates such as IOPS, I/O latency, and Transaction Log Write throughput. You can see the details of these limits for a General Purpose provisioned service tier at [`https://aka.ms/sqldbgplimits`](https://aka.ms/sqldbgplimits).

There is another choice for General Purpose provisioned tier called a **hardware configuration**. Hardware configuration options for General Purpose provisioned tiers are Gen5, Fsv2-series, and DC-series. This choice can also affect resource options such as the number of vCores, processor speed, and security options like secure enclaves.

General Purpose tiers also provide *basic* built-in high availability using Azure Storage (think Failover Cluster Instance), which you will learn more about in Chapter [8](#496204_2_En_8_Chapter.xhtml) on availability.

Like Azure SQL Managed Instance, engine features like In-Memory OLTP are not available for General Purpose (but columnstore indexes are supported).

#### Serverless

The other compute tier option for General Purpose is Serverless. You may remember the story behind Serverless from Chapter [1](#496204_2_En_1_Chapter.xhtml) of the book. The vCore model still applies to Serverless but in a different way.

You will choose a *range* of vCores which includes a min and a max value (the min can be < 1 vCore). Azure SQL Database will *auto-scale* your application to the number of vCores required by the application within this range. One of the advantages of Serverless is that you pay for resource compute usage *by the second* vs. a fixed vCore cost per hour with provisioned.

Note

You may need to dive a bit more into how we calculate the billing for serverless. If so, use our documentation at [`https://azure.microsoft.com/pricing/details/sql-database/single/`](https://azure.microsoft.com/pricing/details/sql-database/single/) and look at the FAQ section for the topic **How is the compute bill calculated in serverless?**

In addition, Serverless supports the concept of *pausing* the database when it is not in use. If activity for the database is *inactive* within an **auto-pause delay** interval (which you can configure), no compute charges are billed, only storage costs. This provides an amazing cost savings opportunity for a user who has an application that does not use Azure SQL Database 24/7\. We found some customers who are building new applications have natural complete idle times or natural times of different compute scale needs. Serverless provides a natural fit for those customers. Although the capabilities of Serverless are the same as Provisioned, the resource capacities and limits are different. We have seen the Serverless model as a popular choice for many applications using Azure SQL Database.

You can see a nice comparison to decide between Serverless and Provisioned at [`https://aka.ms/serverlessvsprovisioned`](https://aka.ms/serverlessvsprovisioned).

#### Business Critical

Just like a Managed Instance, Business Critical service tiers are designed for applications that need maximum performance and availability. The differentiators for Business Critical are

*   Built-in availability with replicas using Always On Availability Groups, including read scale-out (with free one read-only replica)

*   High-speed low-latency I/O using local storage

*   In-Memory OLTP

*   Higher IOPS and I/O throughput rates than General Purpose

#### Hyperscale

As you read in Chapter [1](#496204_2_En_1_Chapter.xhtml) of the book, the Socrates project was a milestone of innovation for the Azure SQL engineering team which launched as Hyperscale. Looking back, the name Hyperscale may not have been our best choice. This is because the name has led to a perception that Hyperscale is only for the largest (and most expensive) database deployments in Azure.

Note

You should know that at the time of the writing of this book, we have been considering a “rebranding” of the name Hyperscale so that developers and users know Hyperscale can be for any application.

There are three innovations for Hyperscale that I believe can now allow developers and users to consider it as the default service tier choice for Azure SQL Database:

1.  We have removed all Azure SQL feature restrictions for Hyperscale that previously existed in the first edition of this book (a few remain).

2.  We changed the pricing model for compute for Hyperscale to a much *more affordable price*. You can read more about this new pricing model at [`https://techcommunity.microsoft.com/t5/azure-sql-blog/azure-sql-database-hyperscale-lower-simplified-pricing/ba-p/3982209`](https://techcommunity.microsoft.com/t5/azure-sql-blog/azure-sql-database-hyperscale-lower-simplified-pricing/ba-p/3982209).

3.  Serverless and elastic pools are now supported by Hyperscale.

Consider these facts about why Hyperscale could be your best service tier choice over General Purpose or Business Critical:

*   You now have an auto-scale compute and auto-scale storage with Hyperscale. You can now start with the smallest database and workload and have it auto-scaled to the size you need with no code changes.

*   Hyperscale supports the largest database size for SQL in the cloud. The current documented maximum size of a Hyperscale database is **100TB** and **1TB** transaction log. Automatic database sizing for Hyperscale database is also extremely fast.

Note

We built the Hyperscale architecture so that it could theoretically support *limitless* database and transaction log sizes.

*   Hyperscale now supports the largest vCore and memory sizes you need. Furthermore, the time required for **scaling operations is constant**. I’ll talk more about scaling in Chapter [7](#496204_2_En_7_Chapter.xhtml) of the book.

*   Read-scale is truly *scale*. You can get **up to 30 named replicas**.

*   **Restore** operations are **incredibly fast regardless of size**. Hyperscale uses snapshot backups of database files to provide fast backups and very fast restores.

*   Your **transaction log throughput is independent** of your vCore configuration or the fact that Hyperscale has replicas. This is because the replica architecture doesn’t rely on Always On Availability Group technology.

*   **I/O operations can be extremely fast** especially if they are served by caching systems implemented with Hyperscale.

Being a lifelong SQL Server engine internals geek, I believe Hyperscale is one of the most innovative engineering efforts for the SQL Team (right along the amazing story of SQL Server on Linux, which you can read more about in *Pro SQL Server on Linux*) in its history. Talking about Hyperscale just in this section doesn’t give it due justice, so you will read more about Hyperscale throughout this book. But take a few minutes and watch this video where Kevin Farlee, my former colleague in the SQL Server engineering team who has now retired, shows the Microsoft Mechanics team why Hyperscale is so special at [`https://youtu.be/Z9AFnKI7sfI`](https://youtu.be/Z9AFnKI7sfI).

#### Elastic Pool Databases

As I told the story in Chapter [1](#496204_2_En_1_Chapter.xhtml) of the book, Project Malmo turned into the popular option for software vendors called **SQL elastic pools**. Elastic pools are simply the concept of grouping a collection of Azure SQL Databases for the purpose of lowering cost and management of databases at scale.

Think of a scenario where you are a Software as a Service (SaaS) vendor and want to use Azure SQL Database. You decide to build a database for each customer to partition and isolate their performance and experience. You may need to deploy thousands of databases.

Here is your problem. How do you decide which pricing tier to use for each database for each customer when they may not all have the same usage across time? You have two choices:

*   Provision all databases with the maximum resource capacity (such as vCores) you think may be needed. In this scenario, you may be overpaying across all databases.

*   Provision all the databases with a lower resource capacity which can save you money but might cause issues for you to have to scale up and down specific databases depending on their usage leading to downtime.

Elastic pools allow you to provision (serverless is not available) a service tier with a vCore choice (DTU is also possible) and database size that will be used across a group of databases that are added to the pool. This way, you pay for the resources of the entire pool, and databases can share resources across the pool.

Databases in an elastic pool can be used with DTU or vCore deployment options like General Purpose, Business Critical, and Hyperscale service tiers (but do not support serverless). I’ll cover more about how to deploy and use an elastic pool in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book.

### Azure Database Fleet Manager

While elastic pools can be helpful to provide simplified management of costs and performance for many databases, you may need more. Microsoft has announced a new method to deploy and manage databases and elastic pools at a *very large* scale called **fleet manager**.

Fleet manager is a concept born within Microsoft for our groups that need to deploy and manage databases at scale such as Dynamics 365\. Fleet managers use concepts called a **fleetspace**, **fleet**, and **tier** to allow you to deploy and manage a large number of databases or elastic pools across multiple regions using a policy approach. This approach allows you to manage your fleet in a consistent and flexible manner, including security, schema changes, intelligent resource rebalancing, HADR configuration, and “smart” defaults.

At the time of the writing of this book, fleet manager was in preview and requires a sign-up to participate. You can read the blog announcement at [`https://aka.ms/azuresqlfleet`](https://aka.ms/azuresqlfleet). Stay up to date with all the latest announcements for Azure SQL Database as this capability moves into public preview and eventually General Availability at [`https://aka.ms/whatsnewsqldb`](https://aka.ms/whatsnewsqldb).

### Azure SQL Database vs. Azure SQL Managed Instance

As you have read and seen all the options for Azure SQL Managed Instance and Azure SQL Databases, you might have enough information to start making choices. Many customers choose a combination of these options depending on their application needs.

As you make this decision, consider these major factors as differences between an Azure Managed Instance and Azure SQL Database:

*   If you need a **database > 32TB**, **Hyperscale** today is your only choice.

*   If your deployment requires **database engine instance features** such as SQL Server Agent jobs, Database Mail, Resource Governor, or Transactional Replication, **Azure Managed Instance** is your only choice.

*   If your application is very **sensitive to I/O latency**, **Business Critical** service tiers or **Hyperscale** may be your best choice.

*   If you need the **highest level of availability and resiliency**, then the **Business Critical** service tier or Azure SQL Database **Hyperscale** is your best choice.

*   If your application has very **intermittent usage patterns**, including periods of almost complete idleness, then **Serverless** may be a very good choice for you.

As you look at these factors and still may not be sure whether an Azure SQL Managed Instance or Azure SQL Database fits your requirements, then I would first recommend you look at Azure SQL Database. Azure SQL Database has most of the core database engine features you likely need and provides the highest level of managed service capabilities. I’ve also found that Azure SQL Database has the fastest deployment and scaling operations across various deployment options.

You still may find that Azure SQL Database does not have the feature set you need. Therefore, I highly recommend you look at this feature comparison in the documentation (which we are always updating) at [`https://learn.microsoft.com/azure/azure-sql/database/features-comparison`](https://learn.microsoft.com/azure/azure-sql/database/features-comparison).

A really nice comparison between service tiers across Azure SQL Database General Purpose, Business Critical, Hyperscale, and Azure SQL Managed Instance can be found at [`https://aka.ms/azuresqlservicetiers`](https://aka.ms/azuresqlservicetiers).

### Customers Using Azure SQL Database

Azure SQL Database has been a popular customer choice across a single database, Hyperscale, Serverless, and elastic pools.

**BKW Energie** quickly realized as they built a digital geographic information system (GIS) platform for their energy production grid that **Hyperscale** was the only cloud database platform that provided them with the scale they needed along with the power of SQL, including built-in graph database capabilities. You can read the full story about BKW Energie at [`https://customers.microsoft.com/story/825080-bkw-energie-energy-azure`](https://customers.microsoft.com/story/825080-bkw-energie-energy-azure).

The true power of **Serverless** is cost savings for applications that don’t need a high level of compute usage 24/7\. **CampBrain** is a software company that offers a service to help camps run their operations. When CampBrain discovered Microsoft Azure SQL Database **serverless**, a compute tier for databases with intermittent, unpredictable usage, it cleared one of its biggest hurdles: managing compute costs despite the highs and lows created by extreme usage, plus reducing performance management complexity and risk. You can read the full CampBrain customer story at [`https://customers.microsoft.com/story/779861-campbrain_professionalservices_azure_canada`](https://customers.microsoft.com/story/779861-campbrain_professionalservices_azure_canada).

## Interfaces for Azure SQL

At the beginning of this chapter, I described the various interfaces in the Azure ecosystem: portal, API, and CLI. Azure SQL integrates with all these interfaces but also with traditional SQL Server interfaces.

### Azure Portal

We have created an Azure SQL experience in the Azure Portal to snap to the Azure SQL lineup. Figure [2-9](#496204_2_En_2_Chapter.xhtml#Fig9) shows the Azure SQL options after using search in the portal.

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig9_HTML.jpg)

Figure 2-9

Azure SQL services and marketplace

If you choose Azure SQL services, you will see a list of resources you have created for Azure SQL (including registered virtual machines). If you choose the Azure SQL marketplace option, you will see choices to create Azure SQL services as seen in Figure [2-10](#496204_2_En_2_Chapter.xhtml#Fig10).

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig10_HTML.jpg)

Figure 2-10

Azure SQL deployment options

You will also see that after you have deployed Azure SQL resources, there are specific integrations for virtual machines, instances, and databases. You will see the examples throughout the rest of the book.

As I pointed out earlier, the Azure Portal has a new Copilot experience to help you deploy and manage Azure resources. Our team has introduced a new context-aware Copilot experience as part of that framework called **Microsoft Copilot skills in Azure SQL Database**. In my opinion, this experience is industry differentiating. For example, with this new “SQL Copilot,” you can solve complex performance problems quickly in the context of your database using natural language prompts. Figure [2-11](#496204_2_En_2_Chapter.xhtml#Fig11) shows an example of this experience.

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig11_HTML.jpg)

Figure 2-11

Microsoft Copilot skills in Azure SQL Database

You will see examples of how to use this Copilot experience in several chapters throughout the rest of the book. You can get started quickly at [`https://aka.ms/sqlcopilot`](https://aka.ms/sqlcopilot).

In addition, we also now have a new experience in the **Query Editor** through the Azure Portal to help generate T-SQL statements using natural language prompts. Figure [2-12](#496204_2_En_2_Chapter.xhtml#Fig12) shows an example of using this new experience.

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig12_HTML.jpg)

Figure 2-12

Query Editor with natural language to SQL

I’ll show you examples of how to use this AI-assisted experience in Chapter [10](#496204_2_En_10_Chapter.xhtml) of the book.

Note

At the time of the writing of this book, the **natural language to SQL in the Azure portal** **query editor** feature was in preview. The final interface for how this will work in the Query Editor may change by the time it becomes generally available.

### az CLI

There are specific options for the az CLI cross-platform utility for virtual machines, managed instances, and databases. Don’t forget az is installed by default in the Azure Cloud Shell. Here are some documentation resources to get you started with the az CLI for Azure SQL:

*   Here is a reference for how to use the az for SQL Server on Virtual Machines: [`https://learn.microsoft.com/cli/azure/sql/vm`](https://learn.microsoft.com/cli/azure/sql/vm).

*   You can find a list of az options specifically for managed instances at [`https://learn.microsoft.com/cli/azure/sql/mi`](https://learn.microsoft.com/cli/azure/sql/mi).

*   You can find a list of az options specifically for databases at [`https://learn.microsoft.com/cli/azure/sql`](https://learn.microsoft.com/cli/azure/sql).

Don’t forget about ARM templates. You can find many examples of ARM templates specific to Azure SQL. A great resource is *QuickStart* templates all found on GitHub. To see the complete gallery, visit [`https://github.com/Azure/azure-quickstart-templates`](https://github.com/Azure/azure-quickstart-templates).

### PowerShell

The Azure PowerShell cmdlet suite includes specific options for Azure SQL:

*   The following link contains an example of how to use Azure PowerShell to provision a SQL Server on Virtual Machines: [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/create-sql-vm-powershell`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/create-sql-vm-powershell).

*   You can find a complete list of Azure PowerShell cmdlets for Azure Managed Instance and Databases at [`https://learn.microsoft.com/powershell/module/az.sql`](https://learn.microsoft.com/powershell/module/az.sql).

*   You can find examples for PowerShell for Azure SQL Database at [`https://learn.microsoft.com/azure/azure-sql/database/powershell-script-content-guide?view=azuresql&tabs=single-database`](https://learn.microsoft.com/azure/azure-sql/database/powershell-script-content-guide%253Fview%253Dazuresql%2526tabs%253Dsingle-database).

### REST API

It may be rare you use the REST API interfaces for Azure.

*   If you do, there are some specific REST API operations for Azure SQL Database which you can see at [`https://learn.microsoft.com/rest/api/sql`](https://learn.microsoft.com/rest/api/sql).

*   You can find REST API references for Azure SQL Managed Instance at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/api-references-create-manage-instance?view=azuresql#rest-api-create-and-manage-managed-instances`](https://learn.microsoft.com/azure/azure-sql/managed-instance/api-references-create-manage-instance%253Fview%253Dazuresql%2523rest-api-create-and-manage-managed-instances).

*   You can also manage Azure Virtual Machines through REST which you can read about at [`https://learn.microsoft.com/rest/api/compute/virtual-machines`](https://learn.microsoft.com/rest/api/compute/virtual-machines).

### TDS and T-SQL

All Azure SQL services are powered by the SQL Server Engine. This means all Azure SQL services support the Tabular Data Stream (TDS) protocol (the full TDS specification can be found at [`https://aka.ms/tdsprotocol`](https://aka.ms/tdsprotocol)).

This also means that all Azure SQL services support a variety of programming languages, drivers, and providers. Use the website [`https://aka.ms/sqldev`](https://aka.ms/sqldev) to learn how to develop programs for SQL Server and Azure SQL. In some cases, a provider may provide specific options only applicable to Azure SQL. For example, the .Net provides modules for Azure SQL Management which you can read at [`https://learn.microsoft.com/dotnet/api/microsoft.azure.management.sql.models`](https://learn.microsoft.com/dotnet/api/microsoft.azure.management.sql.models).

Because all Azure SQL services are powered by the SQL Server database engine, the engine programming language for all is T-SQL. As described throughout this chapter, not all Azure SQL options support all 100% T-SQL statements supported in SQL Server. Here is a summary guide for you to use:

*   SQL Server on Azure Virtual Machine has 100% support of T-SQL.

*   Azure SQL Managed Instance is close to 100% T-SQL support. You can read the differences at [`https://aka.ms/azuresqlmitsqldiff`](https://aka.ms/azuresqlmitsqldiff).

*   Azure SQL Database has the largest difference in T-SQL support from SQL Server, but *most core database engine* T-SQL statements are supported. The documentation provides a guide for you at [`https://aka.ms/azuresqldbtsqldiff`](https://aka.ms/azuresqldbtsqldiff).

In some cases, there are new T-SQL options only applicable to a Managed Instance or Database. For example, CREATE DATABASE has the option to specific service tiers for Azure SQL.

Tip

The Microsoft documentation for the T-SQL reference has a concept called “Applies To.” This guide will help steer you whether a T-SQL statement applies to SQL Server, Azure SQL Database, or both. You can read about this guidance at [`https://learn.microsoft.com/sql/t-sql/language-reference?view=sql-server-ver15#applies-to-references`](https://learn.microsoft.com/sql/t-sql/language-reference%253Fview%253Dsql-server-ver15%2523applies-to-references).

### SQL CLI

All common SQL Server Command-Line Interfaces (CLIs) support all options for Azure SQL, including **sqlcmd** and **bcp**. You can read about these tools at [`https://aka.ms/sqlcmdtools`](https://aka.ms/sqlcmdtools).

Note

A new version of sqlcmd known as “go-sqlcmd” is now available in preview. Check out the new tool at [`https://github.com/microsoft/go-sqlcmd`](https://github.com/microsoft/go-sqlcmd).

### SQL Server Management Studio (SSMS)

**SSMS** is perhaps the most popular SQL Server tool in the world. SSMS fully supports all options for Azure SQL. In fact, SSMS detects the specific Azure SQL type and only provides options that work for that type. For example, if you connect SSMS to Azure SQL Database, it will not show you options in Object Explorer (e.g., SQL Server Agent) that are not supported for Azure SQL Database. Download the latest SSMS version (typically updated monthly) at [`https://aka.ms/ssms`](https://aka.ms/ssms).

When you connect with SSMS to an Azure SQL Database, the icon in Object Explorer will look like Figure [2-13](#496204_2_En_2_Chapter.xhtml#Fig13).

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig13_HTML.jpg)

Figure 2-13

SSMS Object Explorer connected to Azure SQL Database

When you connect with SSMS to an Azure SQL Managed Instance, the icon in Object Explorer will look like Figure [2-14](#496204_2_En_2_Chapter.xhtml#Fig14).

![](images/496204_2_En_2_Chapter/496204_2_En_2_Fig14_HTML.jpg)

Figure 2-14

SSMS Object Explorer connected to Azure SQL Managed Instance

### Visual Studio Tools

The popular **SQL Server Data Tools (SSDT)** fully supports Azure SQL Database. You can read more about SSDT at [`https://visualstudio.microsoft.com/vs/features/ssdt/`](https://visualstudio.microsoft.com/vs/features/ssdt/). **Visual Studio Code** supports an extension called **mssql** which supports Azure SQL Database and Managed Instance. You can see an example of using the mssql extension with Azure SQL Database at [`https://learn.microsoft.com/azure/azure-sql/database/connect-query-vscode?view=azuresql`](https://learn.microsoft.com/azure/azure-sql/database/connect-query-vscode%253Fview%253Dazuresql).

## Beyond RDBMS

In 2023, I came to realize that Azure SQL Database was becoming far more than a traditional Relational Database Management System (RDBMS). I came up with a branding to describe these capabilities, calling it *Beyond RDBMS*.

Chapter [10](#496204_2_En_10_Chapter.xhtml) will describe all these capabilities in more detail but consider that Azure SQL Database (as well as SQL Server in Virtual Machines and Azure SQL Managed Instance) has built into the database (with no add-ons required):

*   A native JSON data type and T-SQL functions

*   A graph database

*   Spatial data types

*   Columnstore indexes

*   Intelligent Query Processing

*   A built-in blockchain with Ledger

*   A new Regular Expression (RegEx) set of T-SQL functions

*   Memory-optimized tables

In addition, a new set of capabilities both in and outside of SQL has been built, including

*   Calling REST API inside the SQL engine

*   Connecting and executing SQL with GraphQL

*   Deep integration with Azure Functions

*   A complete DevOps development framework using containers, database projects, and GitHub actions

*   The ability to build new Generative AI applications

*   Integration with Microsoft Fabric

In Chapter [10](#496204_2_En_10_Chapter.xhtml), you will read more about how to use and take advantage of these new features to do things with SQL you may never thought possible.

## Summary

Azure SQL is the world’s database. With services for virtual machines, managed instances, and databases, there is just about every option you need, all powered by the same database engine for one of the world’s most popular database products, SQL Server.

Azure SQL is fully integrated with the Azure ecosystem, including the Azure Portal and popular interfaces and tools.

SQL Server on Azure Virtual Machines provides you the ultimate level of control while taking advantage of the benefits of the Azure system. Azure SQL Managed Instances provide you a database engine instance combined with the power of a PaaS service, including built-in high availability. Azure SQL Database provides you the most deployment options for a modern cloud application, including serverless and Hyperscale. Azure SQL Database gives you maximum PaaS capabilities, including built-in HADR and Automatic Tuning.

The rest of this book is a deep dive into the Azure SQL lineup. Your first stop in that journey is SQL Server on Azure Virtual Machines.

# 3. SQL Server on Azure Virtual Machine

In Chapter [2](#496204_2_En_2_Chapter.xhtml), I described the Azure SQL lineup which includes SQL Server on Azure Virtual Machine (VM). SQL Server on Azure VM represents the primary IaaS deployment option for SQL in Azure.

In this chapter, I will cover all aspects of deploying, configuring, optimizing, and managing SQL Server on Azure Virtual Machine.

You will go through several examples in this chapter. You will need the following to complete these examples:

*   An Azure subscription.

*   A minimum of Contributor role access to the Azure subscription.

*   Access to the Azure Portal (web or Windows application).

*   Installation of the az CLI. You can also use the Azure Cloud Shell instead since az is already installed.

## Deploying

As a long-time SQL Server user, I have never really used the term *deploy*. I have always used the term install or setup. Deploy is a term I will use for the rest of this book to talk about installing a SQL Server on Azure Virtual Machine, Managed Instance, or Database. There is a good reason to snap to the term deploy. When you create a resource for an Azure service (whether that is through the Portal or CLI), Azure Resource Manager will create a **deployment**. You will learn how to view information about deployment history in this chapter and in Chapter [4](#496204_2_En_4_Chapter.xhtml) of this book.

The basic process to deploy SQL Server on Azure Virtual Machine is

*   ✓ Decide to use a SQL Server gallery image or “deploy on your own.”

*   ✓ Choose a resource group, region, availability, and other options.

*   ✓ Choose a virtual machine size, admin account, and port rule.

*   ✓ Optionally supply other configuration choices including specific SQL Server choices and storage.

*   ✓ Deploy it!

### Pricing

Before I get into the details of deployment, you should understand more about how you pay for Azure Virtual Machines. Since you are deploying in Azure, you are paying Microsoft a fee on a regular basis (billed monthly) for resource usage such as compute and storage. This is referred to as *pay as you go*. In addition, you will pay for the license for the operating system (if that OS requires a paid license such as Windows Server) and the license for SQL Server. You will have options to *bring your own license* (BYOL) for both the OS and SQL Server to utilize licenses you have already paid for or to use those licenses for a concept called Azure Hybrid Benefit (AHB). You will also learn that there are other ways in this chapter to save money such as Reserved Instances and stopping the VM when you do not need it. To help you, check out this nifty website called a pricing calculator at [`https://azure.microsoft.com/pricing/calculator/?service=virtual-machines`](https://azure.microsoft.com/pricing/calculator/%253Fservice%253Dvirtual-machines).

### SQL Server Gallery Images

To deploy a SQL Server on Virtual Machine, you can choose from a set of preinstalled images of an operating system/SQL Server version/edition combination called *gallery images*. There are all types of choices here. You can read a good FAQ resource on SQL Server images at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/frequently-asked-questions-faq?view=azuresql#images`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/frequently-asked-questions-faq%253Fview%253Dazuresql%2523images).

Think of using a SQL Server gallery image like a *sysprep installed SQL Server*. You can also choose to deploy a virtual machine with an operating system from a gallery image and then install SQL Server from your own media inside the guest OS of the VM. I call this method “deploy on your own.” I will discuss this option in the section below titled “Deploy on Your Own.”

Let’s explore using the Azure Portal for a deployment of SQL Server 2022 on Azure Virtual Machines using a gallery image. Follow along with your own subscription or read through these examples and deploy later. I also have some interesting labs to point you to at the end of the chapter.

In your Azure Portal, search for Azure SQL in the **Marketplace** using the Search box at the top of your portal as in Figure [3-1](#496204_2_En_3_Chapter.xhtml#Fig1).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig1_HTML.jpg)

Figure 3-1

Azure SQL from the Marketplace

You are now presented choices on the type of Azure service to deploy. Use the drop-down list under SQL virtual machines to choose **SQL Server 2022 Enterprise on Windows Server 2022** as in Figure [3-2](#496204_2_En_3_Chapter.xhtml#Fig2).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig2_HTML.jpg)

Figure 3-2

SQL Server 2012 gallery image from Azure SQL

For the purposes of this example, I am going to choose SQL Server 2022 Enterprise. This allows me to show you a few configuration choices in the rest of this chapter. You could have easily picked Free SQL Server 2022 Developer on Windows Server 2022\. For this image choice, the SQL Server license is free, but you can only use the VM for development purposes.

Choose this option and click the **Create** button. You are now presented a screen for a series of required fields and choices to deploy the VM as seen in Figure [3-3](#496204_2_En_3_Chapter.xhtml#Fig3).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig3_HTML.jpg)

Figure 3-3

The initial top of the create SQL VM screen for Basics

Notice at the top of the screen some Copilot for Azure choices. In addition, there are a series of names (or sections) indicating several choices you can make during deployment. The first screen is known as **Basics**. Not all fields in all sections are required. The Basics screen has the most required fields. You can click any of these sections or use the buttons at the bottom to navigate through a series of choices.

Some of your defaults will be different (e.g., your Azure subscription will be listed as a default; if you have more than one subscription, change it here to the desired target subscription for the VM deployment), and you will be making various choices. Let us walk through all your choices during the deploy process and create screens.

### Resource Group, Region, and Availability

Your first several options are required fields to provide, including a resource group, name, and Azure region. In addition, you have an option to choose an availability option.

#### Resource Group

As I have described in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book, a resource group is a great way to organize and manage Azure resources together. You will see in this chapter that, by default, a virtual network is created for a new resource group you specify, and the VM is automatically added to the virtual network. For the purposes of this exercise, choose **Create New** and give it a name. I called my VM **bwsqlvmsrg**.

#### Virtual Machine Name

This is both the name of the Azure resource and the guest host name inside the VM (you can change the host name in the VM after deployment). Note some special characters are not allowed, and the name must be no longer than 15 characters. This name must be unique within the resource group. For this exercise, I put in the name **bwsql2022**.

#### Region

This is the region where Azure data centers are located. I described Azure regions and data centers in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book. There are several factors for the region you choose, including available VM sizes, compliance, price, and latency to users and applications. You can read more about choosing the right region at [`https://www.cloudelicious.net/azure-region-and-datacenter-find-your-best-match/`](https://www.cloudelicious.net/azure-region-and-datacenter-find-your-best-match/). Note it is also possible that your subscription does not support certain Azure regions. Check with the owner of your subscription if you don’t see a choice you need. I will choose Central US for my region.

#### Availability Options

This is an optional field during deployment. This choice will only make sense to choose if you plan to connect your SQL VM to other VMs for HADR purposes (such as Availability Groups, Failover Cluster Instance, etc.). Unfortunately, it is best to decide which option to use here during deployment. For the purposes of this example, leave this as “No infrastructure redundancy required.” I will discuss more when to use certain options here under the section titled “HADR” later in this chapter.

Note

This does not mean you do not get basic high availability for a *stand-alone* SQL Server. Azure can handle migrating your VM to a host within a data center for failures (using live migration). In addition, there is built-in redundancy for Azure Storage for your databases.

#### Security Type

Azure Virtual Machines offer new options to provide a higher-level security for the VM. This can be especially important for SQL Server as it can provide a level of security for data at rest and in-memory for SQL inside the VM. This security type is known as a **confidential virtual machine**, supported by AMD processors. Another security type is **trusted launch virtual machines** which protects the virtual machines with features like secure boot. For the purposes of this example, I won’t use these choices, so I’ll pick **Standard**.

#### Image

This will be filled in with the choice you made from the Azure SQL screen. It is possible for you to change it at this point before you deploy. Leave the choice SQL Server 2022 Enterprise Windows Server 2022 for now.

Note

You can get a complete list of SQL Server gallery images available in a specific region using the following PowerShell command (substitute in your region name of choice):

**Get-AzVMImageOffer -Location ‘Central US’ -Publisher ‘MicrosoftSQLServer’**

#### VM Architecture

Azure Virtual Machines support both x64 and Arm64 architectures. SQL Server today is only supported for x64.

#### Spot Instance

An Azure Spot Instance is a concept to save you money on virtual machines that are not used consistently. This is a great concept but probably not applicable to your use of SQL Server in Azure Virtual Machine.

After making these choices, my screen looks like Figure [3-4](#496204_2_En_3_Chapter.xhtml#Fig4).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig4_HTML.jpg)

Figure 3-4

Specifying resource group, region, availability, and other choices for Basics

You will notice in my selection that the image includes the words **Gen2**. Generation 2 (Gen2) virtual machines are the default selection for our SQL Server images. I highly recommend you stay with Gen2 images as it provides capabilities that are not possible with Gen1 VMs. You can read more at [`https://learn.microsoft.com/azure/virtual-machines/generation-2`](https://learn.microsoft.com/azure/virtual-machines/generation-2).

### Virtual Machine Sizes

One of the most important decisions you will make is what *size* to make the virtual machine for SQL Server in Azure. Size is effectively a combination of CPU, memory, and I/O choices. You make these choices today in your data center when you deploy SQL Server to a virtual machine or bare metal. You select how many CPUs, CPU speed, RAM, size of disks, and disk speed. So you need to be able to do the same thing with Azure.

Azure Virtual Machine sizes are categorized by *Family* and are known with a letter designation (B, D, E, …). If you are just trying to get started with SQL Server on Azure Virtual Machine, my recommendation is to pick a reasonable size to test out SQL Server vs. cost. If you are choosing a size for production, you need to carefully review size options. As you can imagine, the higher the *horsepower*, the higher the cost.

As Azure data centers bring in new models of computers to support virtual machine and Azure resources, new sizes support different hardware *generations*. Therefore, you will see virtual machine sizes like Ev4 or Ev5\. VM sizes are comprised of several options in the form of a series of letters and numbers that make up the characteristics of VM performance and resources. I call it the “alphabet of virtual machine choices.” Let me show you through an example. Using the portal screen I’ve shown above for Basics, you can scroll down to see the size chosen by default. I will then use the **Select size** option as seen in Figure [3-5](#496204_2_En_3_Chapter.xhtml#Fig5) to bring up a list of choices.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig5_HTML.jpg)

Figure 3-5

Choosing a VM size

You should now be presented with a screen that gives you choices on VM sizes. These tables can look daunting, but it gives you a feel of why you would pick a specific size because it dictates the number of CPUs, RAM, disks, IOPS, etc., you get for your VM deployment. Figure [3-6](#496204_2_En_3_Chapter.xhtml#Fig6) shows results after I have typed in the “Search by VM size…” the string v5 which shows me some of the latest hardware for VMs.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig6_HTML.jpg)

Figure 3-6

VM size choices for Azure

I scrolled down after using this search and pointed out the choice I’ll use for this VM deployment called E4bds_v5\. Let me use this example to point out what this screen shows about my selection and what some of the letter options mean.

First, you can see at the top columns that show for your choice a **Type** and then the maximum resources you get for vCPU, RAM, Data Disks, Max IOPS, Local Storage, Premium Disk Support, and estimated cost. The Type of **Memory Optimized** refers to a category of VM Size families that have the best memory to vCPU ratio because the number of vCPUs can affect how much RAM you get. For SQL Server, RAM is an important resource so the more memory per vCPU can benefit SQL Server workloads. You can see all the VM types available at [`https://aka.ms/azurevmsizes`](https://aka.ms/azurevmsizes).

I am going to choose E4bds_v5 because it has plenty of CPUs and RAM for my purposes and enables a few options I need which I will describe shortly. If you look at the VM size choice, here is what each option means:

*   **E** = E is the E-series family which is one of the recommended VM sizes for SQL production workloads. The E-series has some of the latest innovations, latest processors, and optimized workloads that need memory like SQL Server.

*   **4** = Number of vCPUs for this choice.

*   **b** = Choosing the b option gives you a higher I/O to core ratio in the form of IOPS. If you look at the VM choice E4ds_v5, you can see it has a max IOPS of 6400 where the b option bumps this to 12100 with only a slightly higher cost.

*   **d** = The VM supports local storage. Local storage is a fast, local SSD drive for the VM. This is a critical choice for SQL Server since you can place tempdb on this drive and not have to worry about I/O performance. The local SSD drive is also known as a “scratch” drive since if a failover occurs for the VM to another host, any data is lost on this drive, but for tempdb that doesn’t matter since we recreate it on SQL Server startup. The amount of local storage you get is defined by the number of vCPUs you choose.

*   **s** = Premium managed disks are supported. You will learn more about why premium managed disks are important for SQL Server in the section later in this chapter titled “Maximizing Storage Performance.”

*   **v5** = This is the family version. Microsoft is always innovating on VM choices, so any new major innovation (usually a choice for the latest hardware) comes in the form of a new family version. At the time of the writing of this book, the v6 for the E-series was in preview. Read more at [`https://learn.microsoft.com/azure/virtual-machines/esv6-edsv6-series`](https://learn.microsoft.com/azure/virtual-machines/esv6-edsv6-series).

Tip

If you are looking for VM choices that use AMD processors, look for the letter **a** in the VM size. For example, the equivalent AMD choice for my above selection is E4ads_v5\. It is important to note that the b option for higher I/O to core ratio is not available today for AMD.

You can learn more about the latest Microsoft recommendations for VM sizes for SQL Server at [`https://aka.ms/sqlIaaSSizing`](https://aka.ms/sqlIaaSSizing).

Note

You can change the VM size after you have deployed it without having to delete and recreate it. This is called *resizing* a VM. There are limits here, and it will require some downtime, but you can read more about resizing VMs at [`https://learn.microsoft.com/azure/virtual-machines/sizes/resize-vm`](https://learn.microsoft.com/azure/virtual-machines/sizes/resize-vm).

### Accounts, Port, and OS Licensing

To finish off the default screen to deploy SQL Server in a virtual machine, you must provide an administrator account and password. For Windows, this becomes the local admin account and for Linux a root user. For a SQL Server gallery image deployment on Windows, this local admin account is automatically added to the sysadmin role (just like you would add yourself during SQL Server setup). The password for the admin account can be between 12 and 123 characters and must be a strong password with at least 3 of the following characters: 1 lower case, 1 upper case, 1 number, and 1 special character.

The next choice is to decide whether any ports for the virtual machine will be opened and available for inbound traffic. By default, port 3389 which is used by the Remote Desktop Connection (RDP) protocol is selected (for Linux, this would be port 22 which is the default port for the ssh protocol). Using this option is the most flexible but not the most secure option. I will leave this option for now, but later in this chapter in the section called “Connecting to Your VM,” I will talk about using different security options.

The final choice involves licensing for Windows deployments. When you choose a SQL Server gallery image with Windows, your license will be a *pay as you go* subscription for both SQL Server and Windows. This basically means you pay each month for a fixed cost of using Windows and SQL Server. You also pay cost for resource usage based on CPU usage and storage. If you own existing licenses for Windows Server, you can apply those licenses to your Azure Virtual Machine cost. You can read more about using Azure Hybrid Benefit for Windows Server at [`https://aka.ms/azurehybridbenefit`](https://aka.ms/azurehybridbenefit).

Figure [3-7](#496204_2_En_3_Chapter.xhtml#Fig7) shows the rest of the portal screen to fill out these options.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig7_HTML.jpg)

Figure 3-7

Choosing account, port, and license

At this point in the deployment process, you can click **Review+Create** and deploy your VM with other default options. However, let’s look at some of the other sections and why you might want to use them, especially the SQL Server Settings.

### Making Configuration Choices As Part of Deploy

Let us look at the other options you can select as part of deploying SQL Server for Azure Virtual Machine.

#### OS Disks

Select **Next: Disks >** to see the options as seen in Figure [3-8](#496204_2_En_3_Chapter.xhtml#Fig8).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig8_HTML.jpg)

Figure 3-8

Disk options for Azure Virtual Machine during deployment

The disk options presented here are for encryption at the host level and options for the disk supporting the operating system and other system files. While for all SQL Server deployments I recommend you choose the default options provided here, including premium managed disks, it is worth investigating the option called **Encryption at Host**, which will encrypt all disks at rest on the host. I will talk more about different types of managed disks you can use with SQL Server for Azure Virtual Machine in the section called “Maximizing Storage Performance.” I will recommend you take advantage of managed disks with SQL Server for the simple advantage of fault tolerance, but you may want to read more about unmanaged and ephemeral disks. You can read more about managed disks at [`https://learn.microsoft.com/azure/virtual-machines/managed-disks-overview`](https://learn.microsoft.com/azure/virtual-machines/managed-disks-overview).

Note

If you install just an operating system image from the marketplace (e.g., Windows Server), you would get options here to add data disks. For SQL Server marketplace images, you will be able to add data disks in the configuration section below called “SQL Server Settings” or after you have deployed.

#### Networking

Now click **Next: Networking >**. Here, you are going to be presented with a set of choices to configure various aspects of networking for your virtual machine as seen in Figure [3-9](#496204_2_En_3_Chapter.xhtml#Fig9).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig9_HTML.jpg)

Figure 3-9

Networking options during deploy for Azure Virtual Machine

The two options you may want to consider changing are the virtual network and subnet if this virtual machine needs to be included in an *existing* Azure virtual network. Notice in this example, a new virtual network will be created, including a name that contains my new VM. If you would have selected an existing resource group on the first screen to deploy, the virtual network associated with that resource group (if one exists) would be selected. I will discuss more about networking options for Azure Virtual Machine in the section later in this chapter called “Networking.”

The only other option I want to call out on this screen that I believe applies to your SQL Server on Azure Virtual Machine deployments is **accelerated networking**. Accelerated networking can be extremely beneficial for SQL Server deployments where client applications must communicate across a virtual network to SQL Server. Therefore, you may want to choose a VM size (or a VM size with the required number of CPUs) that supports accelerated networking. If you plan to just deploy SQL Server in a virtual machine and do everything inside the virtual machine, you do not need to worry about this option. You can read more about the benefits of accelerated networking at [`https://learn.microsoft.com/azure/virtual-network/accelerated-networking-overview`](https://learn.microsoft.com/azure/virtual-network/accelerated-networking-overview).

Note

You can later change the size of the virtual machine and then enable accelerated networking using PowerShell. You can read more about how to do this at [`https://learn.microsoft.com/azure/virtual-network/create-vm-accelerated-networking-powershell`](https://learn.microsoft.com/azure/virtual-network/create-vm-accelerated-networking-powershell).

There is one other option I’ve not shown here called **Load Balancing options**. While this option may be helpful for some application scenarios, for SQL Server a load balancer is not needed even for Always On Availability Groups (AG). I’ll discuss more about how to set up connectivity for an AG in the section titled “HADR” later in this chapter.

#### Management

Select **Next: Management >**. Figure [3-10](#496204_2_En_3_Chapter.xhtml#Fig10) shows options that may be interesting to your deployment and use of SQL Server with Azure Virtual Machine.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig10_HTML.jpg)

Figure 3-10

Management options during deploy for Azure Virtual Machine

I highly recommend you consider **Microsoft Defender for Cloud**. My subscription has Defender enabled, so it is automatically enabled for my VM. Microsoft Defender for Cloud provides security capabilities for your virtual machine as well as SQL-specific features. Microsoft Defender for Cloud provides advanced threat protection and vulnerability assessments specific to potential security issues for a virtual machine in the cloud. In addition, Microsoft Defender for Cloud provides capabilities specific to SQL Server on Azure Virtual Machine. I’ll discuss more about how to configure this and its benefits in the section titled “Configuring an Azure Virtual Machine” later in this chapter.

Tip

At the time of the writing of this book, Microsoft has released a **Copilot** **for Security in Defender for Cloud**. Give it a try when you enable Defender at [`https://learn.microsoft.com/azure/defender-for-cloud/copilot-security-in-defender-for-cloud`](https://learn.microsoft.com/azure/defender-for-cloud/copilot-security-in-defender-for-cloud).

**System-assigned managed identity** is an interesting option allowing you to create an identity in Microsoft Entra to use for authentication without having to put credentials in your code or application. One nice example to use this capability with Azure Virtual Machine is to connect to an Azure SQL Database from a virtual machine using Microsoft without prompting for any passwords. You can read more about managed identities at [`https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview`](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview).

**Enable auto-shutdown** provides an option where the virtual machine will be shut down daily on the time of your choosing. Why would you ever select this for a SQL Server deployment? The primary reason is that you have a SQL Server deployment, and you have some downtime you can afford and want to save money. When a virtual machine is shut down, you only pay for your licensing and storage costs. I will talk more about stopping and starting virtual machines in the section titled “Configuration” later in this chapter.

**Guest OS updates** provides an option to have critical updates automatically applied to the OS inside the VM. There is a new option called Azure Update Manager you should consider because it allows you to apply even noncritical updates for both Windows Server and SQL Server. I’ll describe this capability more in the section titled “Configuration” later in this chapter.

#### Monitoring

Figure [3-11](#496204_2_En_3_Chapter.xhtml#Fig11) shows options for monitoring.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig11_HTML.jpg)

Figure 3-11

Monitoring options during deploy for Azure Virtual Machine

**Boot diagnostics** are very helpful to see the serial output screen of your virtual machine on the host. This is like seeing the boot screen using a tool like Hyper-V Manager. You can read more about boot diagnostics at [`https://learn.microsoft.com/azure/virtual-machines/boot-diagnostics`](https://learn.microsoft.com/azure/virtual-machines/boot-diagnostics).

I personally recommend you select **OS guest diagnostics**. This will allow you to see performance information from the guest OS in systems like Azure Monitor and even get alerts. I will discuss how to use this in the section titled “Performance Monitoring” later in this chapter.

If you deploy many Azure Virtual Machines, you should consider enabling virtual machine insights. This provides methods to monitor key information about the health and performance of your virtual machines *at scale*. Learn more at [`https://learn.microsoft.com/azure/azure-monitor/vm/vminsights-overview`](https://learn.microsoft.com/azure/azure-monitor/vm/vminsights-overview).

#### Advanced

With these options selected, click **Next: Advanced >**. As the name implies, these are advanced options that may interest you as seen in Figure [3-12](#496204_2_En_3_Chapter.xhtml#Fig12).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig12_HTML.jpg)

Figure 3-12

Advanced options for Azure VM

**Extensions** are applications that run inside the VM that can provide post-deployment and automation tasks. The SQL Server gallery images have an extension used to configure SQL Server automatically if you use a gallery image as I’m doing here. There could be other extensions you want to explore at [`https://learn.microsoft.com/azure/virtual-machines/extensions/features-windows`](https://learn.microsoft.com/azure/virtual-machines/extensions/features-windows).

There are other advanced options as you can see in Figure [3-13](#496204_2_En_3_Chapter.xhtml#Fig13).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig13_HTML.jpg)

Figure 3-13

Other Advanced options for Azure VM

**Host group** defines an *Azure Dedicated Host* used for your virtual machine. The major benefit of an Azure Dedicated Host is a *Dedicated Physical Server*: the server is exclusively used by your organization, ensuring that no other customers share the same hardware. **Capacity reservations** allow you to reserve compute capacity in a specific Azure region or Availability Zone without a long-term commitment. I will discuss more about these topics in a later section of this chapter titled “Reserved Instances, Dedicated Hosts, and Capacity Reservations.”

**Proximity placement groups** are an interesting concept in Azure that allows you to request multiple Azure resources be located as close as possible within an Azure data center to provide the lowest network latency possible. This may be an option to consider if the application to connect to SQL Server in Azure Virtual Machine will be hosted in Azure.

Note

I would not recommend using proximity groups for a HA solution for SQL Server in Azure Virtual Machines such as Availability Groups. In this scenario, you will use a concept called Availability Sets or Availability Zones. I will discuss these concepts later in the chapter in a section titled “HADR.”

Click **Next: SQL Server Settings >** to see the next set of options.

#### SQL Server Settings

Up to this point, all the configuration choices are independent of SQL Server. This screen provides configuration choices for deployment specific to SQL Server. If you know the SQL Server setup experience, we have provided in the latest releases the ability to “configure when you install” options. This means we have included options you normally configure after installation into the process itself. We are now trying to provide this same experience in Azure Virtual Machine but also include options that allow an optimal configuration for Azure.

Let us look at these choices in more detail. Figure [3-14](#496204_2_En_3_Chapter.xhtml#Fig14) shows choices for networking, security, and storage.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig14_HTML.jpg)

Figure 3-14

SQL Server Settings for network, security, and storage

Note

As of the time of writing this book, SQL Server Linux images do not support SQL Server Settings during deployment.

**Networking** provides a choice on how to expose the SQL Server instance default TCP port 1433\. Think of this choice like using a firewall. *Private* means any Azure source within the virtual network of the VM can access this port. This is the default and one I recommend you use. *Local* means the only access is allowed inside the VM. *Public* means the TPC port is exposed on the Internet. As tempting as it may be to use the public option so you can connect with a tool like SSMS from your laptop to this deployed VM, I do not recommend using this option.

Note

The first time I deployed a SQL Server in Azure VM using the public option, I immediately got attacked from outside intruders trying to log in using the sa account guessing password. I found this out when I saw my ERRORLOG flooded with login failed messages. This occurred almost immediately after deploying the VM.

The choice for **SQL Authentication** is identical to enabled mixed mode security for SQL Server. Even though the sa login is disabled by default if you choose to enable this option, you will be prompted for a SQL login that will be granted SQL Server sysadmin rights for your deployment.

**Azure Key Vault integration** is a choice you might want to enable, but do not worry, you can enable this post deployment. Azure Key Vault integration may be helpful to ease your use of Azure Key Value for scenarios like Transparent Data Encryption (TDE). You can read more about Azure Key Vault integration with SQL Server in Azure Virtual Machine at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/azure-key-vault-integration-configure`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/azure-key-vault-integration-configure).

**Storage configuration** is perhaps one of the most important choices you will make for SQL Server in Azure Virtual Machine especially for performance. Let’s look at what choices you can make and how we provide a guided experience to configuration storage for your databases.

Note

There is a new experience in preview to use Azure Premium SSD v2 Storage. I’ll discuss more about this option in the section titled “Maximizing Storage Performance” later in this chapter.

If you click Change configuration, you will see options such as seen in Figure [3-15](#496204_2_En_3_Chapter.xhtml#Fig15).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig15_HTML.jpg)

Figure 3-15

Configuring storage for SQL Server in Azure Virtual Machine

This screen is divided into three sections: (1) database file storage, (2) transaction file storage, and (3) tempdb storage.

For database and transaction files, you are provided options for location, managed disk type, disk size type, and number of disks. For databases, you also have the option to have system databases (master, model, and msdb) moved off the OS drive and to the drive where database files are placed. Unless you are just doing some basic testing, I always recommend the following:

*   Choose to have the data and log on separate drives.

*   Use the Premium SSD disk type.

*   Choose a disk size type that is large enough to handle the necessary size, IOPS, and throughput for the I/O SQL Server needs for your application.

Premium SSD disk types come in various sizes with a designation of P<n> from sizes 4GB (P1) to 32TB (P80). The performance in terms of IOPS and throughput increases with larger sizes. One problem you may face is trying to align the IOPS and throughput you need with your minimum size requirement. This may require you to *overprovision* the size (or multiple disks) to get the performance you need (as I mentioned earlier, there is a new disk type called Premium SSD v2 in preview that can help here; you will learn about this in the section titled “Maximizing Storage Performance” later in the chapter). You can read more about the various Premium SSD disk types at [`https://learn.microsoft.com/azure/virtual-machines/disks-types#premium-ssds`](https://learn.microsoft.com/azure/virtual-machines/disks-types%2523premium-ssds).

For tempdb, notice how the recommendation is to place it on the local SSD drive and recommend number of files and sizes for both data and log, very much like SQL Server setup does today.

Notice the warning at the bottom of the screen. It says the configured disks have a performance vector that is greater than what the VM size supports. This is an important warning because it means the VM size you have selected doesn’t support enough IOPS and/or throughput even though you selected storage that meets your requirements. This is a hard problem to detect when it occurs, so getting the VM size right now to meet your storage performance requirements is important. You will need to go back to the Basics screen first and change the VM size and then come back to this screen to make sure the warning is gone.

Below storage configuration is the ability to change **SQL Server instance settings** as seen in Figure [3-16](#496204_2_En_3_Chapter.xhtml#Fig16).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig16_HTML.jpg)

Figure 3-16

Choosing SQL Server instance settings

Click **Change SQL instance settings** and you will get a screen like Figure [3-17](#496204_2_En_3_Chapter.xhtml#Fig17).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig17_HTML.jpg)

Figure 3-17

Additional options for SQL Server Settings

You can see on this screen various options you can choose during SQL Server setup (remember that when you deploy SQL Server in Azure VM using the portal, we automatically run setup for you to install everything!), such as server collation, MAXDOP, max and min memory, and optimize for ad hoc workload settings. In addition, we provide you two popular options for performance optimization: **lock pages in memory** and **instant file initialization**. You can configure all of these after you deploy, but having the opportunity to do this now just saves you time and effort.

After you select OK, you have other choices you can make as seen in Figure [3-18](#496204_2_En_3_Chapter.xhtml#Fig18).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig18_HTML.jpg)

Figure 3-18

Additional SQL Server Settings

**SQL Server License** allows you to testify that you have an existing SQL Server license that allows you to apply it for Azure Hybrid Benefit (AHB) like the choice for Windows. This can be a significant cost savings to you when using SQL Server for Azure Virtual Machine.

Note

If you do not see this option available, it is because you picked a SQL Server image choice where this option cannot be or is already selected. For example, images that start with the title BYOL (this means Bring Your Own License) already imply you are just using our own license. Developer Edition is free, so no license applies.

We also announced in 2019 new licensing benefits for HA and DR scenarios when using Azure Virtual Machine. You can read more at [`https://cloudblogs.microsoft.com/sqlserver/2019/10/30/new-high-availability-and-disaster-recovery-benefits-for-sql-server/`](https://cloudblogs.microsoft.com/sqlserver/2019/10/30/new-high-availability-and-disaster-recovery-benefits-for-sql-server/).

**Automated patching** provides specific configurations on when to deploy important and critical updates for Windows and SQL Server. Other updates for Windows and SQL Server will depend on how you configure Windows Update inside the VM. As I mentioned earlier, there is a new option called Azure Update Manager you should consider because it allows you to apply even noncritical updates for both Windows Server and SQL Server. I’ll describe this capability more in the section titled “Configuration” later in this chapter.

**Automated backup** uses the managed backups to Azure capability that was shipped as part of SQL Server 2016. Your needs may vary, but many users might find this option very useful to provide a simple automated backup solution for SQL Server. When you enable this option, you have several choices to configure how backups are executed.

Note

Automated patching and automated backup options may not show up as enabled immediately after deployment. The SQL IaaS extension runs in the background after deployment.

**R Services (Advanced Analytics)** is an option to install the SQL Server Machine Learning services feature as part of deployment. I will describe in the next section called “Deploy!” what exactly is installed with the SQL Server gallery image.

Click **Next: Tags >** to see the last option.

#### Tags

The last option before you deploy your virtual machine is to potentially use a *tag* as seen in Figure [3-19](#496204_2_En_3_Chapter.xhtml#Fig19). Tags are a concept supported by the Azure ecosystem to assign a string value to a resource in Azure, like a virtual machine, to organize your resources.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig19_HTML.jpg)

Figure 3-19

Assigning tags to an Azure resource

Click **Next: Review + create >** to validate and deploy the virtual machine.

### Deploy!

The portal will take all your options, perform validation steps, and then present you the ability to create the virtual machine. Figure [3-20](#496204_2_En_3_Chapter.xhtml#Fig20) shows some interesting information on the final validation screen before you click Create.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig20_HTML.jpg)

Figure 3-20

Validation before creating the virtual machine

At the top of this screen, you can see the size of the VM you chose plus estimated costs per hour. Also, notice *Terms of use* and *Privacy policy*. The End User Licensing Agreement (EULA) that comes with SQL Server still applies to Azure Virtual Machine since this is a fully licensed SQL Server. However, since you are deploying the virtual machine in Azure, there are terms and privacy policies you should review. You can read more about Azure terms of use at [`https://azure.microsoft.com/support/legal`](https://azure.microsoft.com/support/legal). Privacy is a very important topic, and since Microsoft is hosting your virtual machine in the cloud, you need to understand all the details of what information Microsoft collects. Read more at [`https://privacy.microsoft.com/privacystatement`](https://privacy.microsoft.com/privacystatement). Your usage of a gallery image for the operating system and/or SQL Server also has terms called Azure Marketplace Terms. You can read more at [`https://azure.microsoft.com/support/legal/marketplace-terms/`](https://azure.microsoft.com/support/legal/marketplace-terms/).

You will also notice on this screen a warning about allowing the RDP port to be exposed to the Internet. You will learn more about how to control access and limit any issues in the section below titled “Connecting to Your VM.” If you scroll down on this screen, you will see all the details of the options you chose to deploy the virtual machine. Notice also at the bottom of the screen the option to **Download a template for automation**. I will discuss using an option for automation with a template in a section later in this chapter called “Using a CLI with ARM Templates, Bison, and Terraform.”

Click the **Create** button when you are ready to launch! The deployment of the virtual machine is asynchronous, so you can even exit the portal, and the deployment is done in the background. However, if you leave the portal screen open, you can track progress live. Within seconds of clicking create, my screen looks like Figure [3-21](#496204_2_En_3_Chapter.xhtml#Fig21) to track the progress of the deployment.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig21_HTML.jpg)

Figure 3-21

Azure Virtual Machine deployment in progress

Not only does the main screen refresh as the virtual machine is being created, but you can click the Notifications icon on the title bar of the portal to also track progress. In my example, in about ten minutes my deployment was complete. Figure [3-22](#496204_2_En_3_Chapter.xhtml#Fig22) shows all the details, including status from the Notifications icon.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig22_HTML.jpg)

Figure 3-22

Deployment complete for SQL Server in Azure Virtual Machine

You will notice for Deployment details that it is not just a virtual machine deployed but many different resources, including network interfaces, disks, storage accounts, virtual networks, and security groups. If you want more details about the deployment, you can click **More events in the Activity Log**.

If you click **Go to resource**, you will be presented with the overview screen for the virtual machine. There is another view of this virtual machine from a SQL Server perspective I’ll cover later. Let’s take a tour on how to navigate the *overview* screen for an Azure Virtual Machine.

### Navigating an Azure Virtual Machine in the Portal

You will find yourself often using several aspects to the overview screen for Azure Virtual Machine and all Azure resources in this book. Let us examine the main areas of the overview screen as seen in Figure [3-23](#496204_2_En_3_Chapter.xhtml#Fig23).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig23_HTML.jpg)

Figure 3-23

The overview screen for Azure Virtual Machine

1.  **Service Menu**

    Each Azure resource provides a series of options to manage the resource. The very top of the Service Menu is the same for all resources. You will want to use the **Overview** option to “find your way back” if you are deep into using *blades or sections* for details on the virtual machine. This is super important as I often have hit the X at the top right of a screen and completely lost the context of my VM. By clicking Overview on the Service Menu, I can keep that context.

    Under Overview, we will see options to look for Activity Log entries specific to this VM, the ability to control access *outside* the VM using RBAC, set new tags, or diagnose problems. Underneath this is a series of category options that you can expand such as Connect to connect to the VM, Network to configure network options, or Settings to configure specific aspects of the VM. There are other options as well you can explore. I will not go over each option here but will use several of these options throughout the rest of the chapter.

2.  **Command Bar**

    These are buttons that allow you to operate the virtual machine. Every Azure resource has buttons unique to the resource. The most common buttons you will use are Connect, Start, Restart, and Stop. I’ll cover more the significance of stopping the VM using this option in the section later in this chapter titled “Stopping vs. Deallocating.”

3.  **Essentials**

    Technically, this is part of the Working Pane. This area displays key information about your virtual machine, including resource group, region, status, OS version, and VM size, but also lets you navigate to certain aspects of the virtual machine such as virtual network.

4.  **Working Pane**

    The Working Pane includes sections to see detailed properties of the VM, monitoring with health events, alerts, and performance metrics, enabling capabilities like Windows Admin Center, recommendations, and training tutorials.

Explore more about how to navigate around the portal at [`https://learn.microsoft.com/azure/azure-portal/azure-portal-overview#getting-around-the-portal`](https://learn.microsoft.com/azure/azure-portal/azure-portal-overview%2523getting-around-the-portal).

### Connecting to Your VM

Now that you have deployed, one of the first things you will want to do is connect to the VM. For a Windows VM, the most popular way to connect and use the VM is with the Remote Desktop Protocol (RDP). You saw in the VM deployment example that I chose the option to open the RDP port (3389) to the Internet.

To use RDP, click the Connect button in the Command Bar and choose Connect. You see the options to connect as in Figure [3-24](#496204_2_En_3_Chapter.xhtml#Fig24).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig24_HTML.jpg)

Figure 3-24

Connecting with RDP to Azure Virtual Machine

You can connect via RDP by selecting **Download RDP file**. The RDP file will be copied to your *Downloads* folder on your client machine. The public IP address for the VM is also posted here if you just want to take any valid RDP client and connect directly to the VM. Once you connect, just like any VM or Windows computer you will be prompted to enter your credentials (which you supplied as part of deployment), which by default is a member of the Administrators group.

Note

If you notice under Download RDP file, there are options to help you troubleshoot any RDP connectivity issues which I highly recommend you look at, should you encounter any issues.

If you use **Select**, you can also see on this screen a recommendation to enable *just-in-time access* to the VM to improve security. Just-in-time access is a way to gain RDP access to the VM “on-demand” so that the RDP port is not opened to the Internet when you are not using the VM. This is a way to allow you to connect to the VM from any client yet limit the exposure of the RDP port to everyone. You can learn more about just-in-time access at [`https://learn.microsoft.com/azure/defender-for-cloud/just-in-time-access-usage`](https://learn.microsoft.com/azure/defender-for-cloud/just-in-time-access-usage).

The other method you can see on this screen is to use a more secure method called Azure Bastion, which you can learn more about at [`https://azure.microsoft.com/services/azure-bastion/`](https://azure.microsoft.com/services/azure-bastion/). I’ve run into customers where their company blocks them from using RDP on their laptop, so this browser-based method is an excellent, more secure option to use to connect to the VM.

You may also connect with standard SQL tools like SSMS from another Azure VM that you have deployed in the same Azure virtual network. I often will deploy another “client” Azure Virtual Machine in the same virtual network, so I can privately connect using SQL tools to my SQL Server deployment.

### Exploring the SQL Server Installation

I thought you might find it interesting to know exactly what we install and how we configure the SQL Server instance when you deploy with SQL Server gallery images.

#### What Is Installed

A SQL Server gallery image installs the entire database engine, SQL Server Analysis Services, SQL Server Integration Services (SSIS), MDS, and DQS. In addition, we install the following engine features:

*   SQL Server Agent

*   SQL Server Replication

*   Full-text search

We also install client, tools, SDK, and SQL Server Management Studio. We do not install Polybase by default. As I have described earlier in this chapter, you can install R Services as part of the deployment based on an option you choose.

Note

Your license includes the ability to deploy SQL Server Reporting Services, but like SQL Server, we don’t install that. You can deploy this yourself using [`https://learn.microsoft.com/sql/reporting-services/install-windows/install-reporting-services`](https://learn.microsoft.com/sql/reporting-services/install-windows/install-reporting-services). In addition, customers with SA agreements using Enterprise Edition have license rights to install Power BI Report Server ([`https://www.microsoft.com/power-platform/products/power-bi/report-server`](https://www.microsoft.com/power-platform/products/power-bi/report-server)).

For a SQL Server gallery image, we copy the SQL Server media, so you can install or remove any features you want. You can find all the setup files in the **C:\SQLServerFull** folder in the VM. Figure [3-25](#496204_2_En_3_Chapter.xhtml#Fig25) shows an example of what features are installed by default using a SQL Server 2022 gallery image.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig25_HTML.jpg)

Figure 3-25

SQL Server features installed by default

#### What Is Configured

The following is a list of configuration choices we make as part of a SQL Server gallery image installation:

*   SQL Server, SQL Server Agent, full-text, and SSIS services are set to automatic and are running after deployment. Service SIDs are used for all services.

*   The VM admin account you specify during deployment becomes a SQL sysadmin.

*   The filestream feature is disabled by default (but can be enabled and supported).

*   Always On Availability Groups is disabled by default (but can be enabled and supported).

*   The TCP/IP protocol is enabled for SQL Server (even for Developer Edition).

*   We typically have a recent Cumulative Update (and/or security update) installed, but it may not be the exact latest Cumulative Update. You can read more about how we keep our images up to date at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-server-on-azure-vm-iaas-what-is-overview?view=azuresql#lifecycle`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-server-on-azure-vm-iaas-what-is-overview%253Fview%253Dazuresql%2523lifecycle).

*   CEIP is enabled by default, but you can disable it. Read more at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-server-on-azure-vm-iaas-what-is-overview?view=azuresql#customer-experience-improvement-program-ceip`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-server-on-azure-vm-iaas-what-is-overview%253Fview%253Dazuresql%2523customer-experience-improvement-program-ceip).

*   Microsoft SQL Server VSS Writer is installed by default.

### Deploy on Your Own

Today when I deploy a SQL Server instance in a Hyper-V virtual machine (often just on my laptop), I go through the process of choosing certain options for the VM (like number of logical CPUs, memory, location of disks) and then install an operating system like Windows Server or Linux (typically from an .ISO file). When this is done, I usually connect to the VM with Remote Desktop (Windows) or ssh (Linux). For Windows, I will download the installation media for SQL Server to my local drive and then “copy and paste” it into a folder in the VM. I then run setup and am off and running to deploy SQL Server. For Linux, I just make sure my VM is connected to the Internet and run the SQL Server on Linux installation which downloads packages as part of the installation process.

This same process is almost identical to the “deploy on your own” option for SQL Server on Azure Virtual Machine, with one exception. You will configure the VM and deploy an operating system by *using one of the gallery images just for the operating system*. You can use the portal, az CLI, or PowerShell to choose your VM size and operating system, very similar to using a SQL Server gallery image except all the options for SQL Server will not be part of the process.

This technique is one I sometimes use because I have complete control of how and what is installed with SQL Server vs. what Microsoft chooses from a gallery image. If you use this method, there is one downside. You do not immediately get to take advantage of the options you get with a SQL gallery image, such as automated backups and security updates. Fortunately, there is a solution to use this customized method for deployment but take advantage of automated features and licensing options. This solution is called the **SQL Server IaaS Agent Extension** (which is installed by default when using a gallery image). I will discuss more about this capability in the section titled “SQL Server IaaS Agent Extension” later in the chapter.

### Using a CLI with ARM Templates, Bison, and Terraform

Azure provides other ways to deploy SQL Server on Azure Virtual Machine besides the Azure portal using a command-line interface (CLI) with the **az sql vm** or **az vm** CLI. There are also PowerShell commands **New-AzVM** and **New-AzSqlVM**. The problem is that all of these methods don’t support a way to configure optimal storage settings as I have described earlier with the portal. You would have to manually configure this after you deployed.

Azure provides a mechanism for any Azure resource to automate the use of Azure CLI tools with a concept called an *ARM template*. An ARM template is a JSON file that describes in a declarative fashion the configuration and infrastructure of a deployment.

While you can try to learn how to build templates from scratch, I recommend you get an example. You can download and edit an example of an Azure template to create a new SQL Server in Azure Virtual Machine with configurable storage settings at [`https://learn.microsoft.com/samples/azure/azure-quickstart-templates/sql-vm-new-storage`](https://learn.microsoft.com/samples/azure/azure-quickstart-templates/sql-vm-new-storage). Azure templates not only configure all the properties to create the virtual machine but support *parameters*, so you can use a template and then supply the parameters you want to customize the deployment at runtime. The link for this template also includes a file to support deploying a SQL Server VM on Azure with Bison.

Terraform can also be used to deploy SQL Server on Azure VM. You can read more at [`https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/mssql_virtual_machine`](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/mssql_virtual_machine).

### Reserved Instances, Dedicated Hosts, and Capacity Reservations

To save money on Azure Virtual Machines, you can prepay for a period (one to three years) and then save on the compute costs of the Azure Virtual Machine as you deploy. This option is called **Azure Reserved VM Instances**. If you plan to deploy many SQL Server Azure Virtual Machines, you probably should investigate this option. Learn more about Reserved Instances at [`https://azure.microsoft.com/pricing/reserved-vm-instances/`](https://azure.microsoft.com/pricing/reserved-vm-instances/).

**Capacity Reservations** allow you to reserve compute capacity in an Azure region or Availability Zone for any duration of time. This is particularly useful for ensuring that you have the necessary resources available for critical workloads, even during peak demand periods. Capacity Reservations are more flexible than Reserved Instances because there is no major long-term commitment. They are available almost immediately, and you have full control how you target your reservation for critical workloads. However, Reserved Instances, because of their long-term commitment, are less expensive than Capacity Reservations. You can learn more about Capacity Reservations at [`https://learn.microsoft.com/azure/virtual-machines/capacity-reservation-overview`](https://learn.microsoft.com/azure/virtual-machines/capacity-reservation-overview).

You might remember there was an option during the deployment example in the portal called a *dedicated host*. An **Azure Dedicated Host** allows you to reserve physical servers dedicated to you and your organization. While normal virtual machines are dedicated to your deployment, you are typically sharing the underlying hosts with other users. A dedicated host may give you the option you need for specific compliance requirements. In addition, you have more control over maintenance events of the infrastructure hosting your VM deployment. You can read more about dedicated hosts at [`https://learn.microsoft.com/azure/virtual-machines/dedicated-hosts`](https://learn.microsoft.com/azure/virtual-machines/dedicated-hosts). The largest virtual machine in this group is the Standard_E104ids_v5 that offers 104 vCores and 672 GiBs of memory. This virtual machine is notable because it’s isolated which means it’s guaranteed to be the only virtual machine running on the host and therefore is isolated from other customer workloads.

### Migration to Azure

You might be considering migration of an existing SQL Server installation to Azure Virtual Machine. Here are the options that might help your migration plans.

#### Azure Arc Migration Assessment

I briefly mentioned in Chapter [1](#496204_2_En_1_Chapter.xhtml) the concept of Azure Arc to connect a SQL Server to the cloud. While I talk more about Azure Arc in Chapter [10](#496204_2_En_10_Chapter.xhtml) of the book, I’ll mention here there is an innovation (in preview at the time of the writing of this book) to help you assess your current SQL Server deployment to see if it is compatible to migration to the cloud, including what are the right deployment decisions and resource choices. Learn more about how to use this capability at [`https://learn.microsoft.com/sql/sql-server/azure-arc/migration-assessment`](https://learn.microsoft.com/sql/sql-server/azure-arc/migration-assessment).

#### Restoring a Database

Since SQL Server in Azure Virtual Machine is a full SQL Server engine, one simple way to migrate an existing SQL Server is to just restore a backup of existing databases.

Since the target SQL Server is in the Azure infrastructure, you have a few ways to do this:

*   Copy the backup file with “copy and paste” over RDP. RDP clients allow you to copy and paste files, and I have done this a bunch of times for Azure VM. Of course, this only makes sense for small backup files (although I have done this with backup files even 1GB in size).

*   Back up the database to Azure Storage using the **backup to URL** functionality within SQL Server. Then in the Azure VM, restore the backup from the same Azure Storage account. You can read more about SQL Server backup to URL at [`https://learn.microsoft.com/sql/relational-databases/backup-restore/sql-server-backup-to-url`](https://learn.microsoft.com/sql/relational-databases/backup-restore/sql-server-backup-to-url).

*   Use **Azure Files**. Think of this like creating your own file share in the cloud. You can use az CLI or tools to copy your backup file to the Azure file share. Then you can mount the Azure file share within the Azure VM (i.e., it will look like a network share in the VM). Read more about how to do this at [`https://learn.microsoft.com/azure/storage/files/storage-files-introduction`](https://learn.microsoft.com/azure/storage/files/storage-files-introduction). I have used this method several times for both Azure VMs for Windows and Linux.

*   For large backup files, consider the **Azure Import/Export service**. You literally will securely ship a hard drive to Microsoft (we can supply you the method with an Azure Data Box Disk), and we will import this to Azure Storage or Azure Files. You can read more about this at [`https://learn.microsoft.com/azure/import-export/storage-import-export-service`](https://learn.microsoft.com/azure/import-export/storage-import-export-service).

#### Azure Migrate and Azure Database Migration Service

Let’s say you wanted a more online migration experience to Azure Virtual Machine. You have that option using the **Azure Database Migration Service** (DMS). DMS is a managed service in Azure that orchestrates a migration to Azure SQL, including Azure Virtual Machine. One of the options for this migration is to do an online migration for SQL Server to Azure VM. DMS will use log shipping technology to take a series of continuous database full and transaction log backups and restore them to an already deployed SQL Server on Azure VM. See an example of how to use this service at [`https://learn.microsoft.com/data-migration/sql-server/virtual-machines/database-migration-service?tabs=online-with-extension`](https://learn.microsoft.com/data-migration/sql-server/virtual-machines/database-migration-service%253Ftabs%253Donline-with-extension).

If you want to assess and migrate SQL Server ***at scale***, consider using **Azure Migrate**. Azure Migrate is an Azure service and set of client software agents to assess and migrate SQL Server running in virtual machines and bare-metal servers to Azure. Azure Migrate can coordinate with Azure Database Migration Service to help you perform offline and/or online migrations. Get started at [`https://learn.microsoft.com/azure/migrate/how-to-create-azure-sql-assessment`](https://learn.microsoft.com/azure/migrate/how-to-create-azure-sql-assessment).

#### Using Azure Migrate Server Migration

What if you would like to migrate the installation of an entire physical server or virtual machine to Azure instead of just your data? Azure supports this concept with a service called **Azure Migrate Server Migration**. This is a true *lift and shift* operation. Think of this like taking a snapshot of your machine or VM and creating an entire VM from the snapshot. Then you configure and optimize after the fact. You can read more about Server Migration at [`https://learn.microsoft.com/azure/migrate/migrate-services-overview#azure-migrate-server-migration-tool`](https://learn.microsoft.com/azure/migrate/migrate-services-overview%2523azure-migrate-server-migration-tool). A great example of how this works for VMware installation can be seen at [`https://aka.ms/mechanicsazuremigrate`](https://aka.ms/mechanicsazuremigrate) with legendary Windows guru Jeff Woolsey.

### Deploying SQL Server on Linux with Azure Virtual Machine

Up to this point, my examples and discussion have centered around deploying SQL Server in Azure Virtual Machines based on Windows Server.

The options and deployment process are very similar for SQL Server on Linux with some notable differences:

*   The gallery images are offered for Linux distributions Ubuntu, Red Hat Enterprise Server, and SUSE. You pay for licenses if the Linux distribution requires it (Ubuntu is a free license).

*   We do not support SQL Server Settings during deployment. The SQL Server IaaS Agent extension is supported on Ubuntu distributions but provides limited support for licensing. Learn more at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/linux/sql-server-iaas-agent-extension-linux`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/linux/sql-server-iaas-agent-extension-linux).

After using a SQL Linux gallery image, you will find the mssql-server, mssql-tools, and mssql-ha packages have been installed.

SQL Agent is not enabled, but you can do this yourself. You can enable SQL Server Agent on Linux with the mssql-conf script which you can read more about at [`https://learn.microsoft.com/sql/linux/sql-server-linux-setup-sql-agent`](https://learn.microsoft.com/sql/linux/sql-server-linux-setup-sql-agent).

Post deployment, you can use documented methods to install other SQL Server Linux packages. For example, you can read [`https://learn.microsoft.com/sql/linux/sql-server-linux-setup-machine-learning`](https://learn.microsoft.com/sql/linux/sql-server-linux-setup-machine-learning) to learn how to install SQL Server Machine Learning Services for Linux.

### Deploying SQL Server Containers

There is no special gallery image or process to deploy SQL Server containers in Azure Virtual Machine. You will use the same process you use today to deploy a SQL Server container in a virtual machine. You can read more about this process at [`https://aka.ms/sqlcontainers`](https://aka.ms/sqlcontainers).

**Azure Kubernetes Service** (AKS) provides a method to deploy containers at scale using the power of Kubernetes. You can see a tutorial of how to do this at [`https://learn.microsoft.com/sql/linux/quickstart-sql-server-containers-azure`](https://learn.microsoft.com/sql/linux/quickstart-sql-server-containers-azure).

## SQL Server IaaS Agent Extension

You have seen examples in this chapter of special options and advantages of deploying a SQL Server in Azure Virtual Machine with a gallery image, including licensing, configuration, and automation. This is all made possible through the **SQL Server IaaS Agent Extension** using the **SQL Virtual Machine Resource Provider**. The resource provider is software running in Azure that enables specific SQL capabilities integrated with Azure Resource Manager. The extension is a set of agents running in your virtual machine that use the resource provider.

If you deploy using a SQL Server gallery image, you just take advantage of what the resource provider does, and no action is required from you. However, if you deploy on your own and want these advantages, including licensing, you will need to register your VM as a SQL Azure VM.

First, you need to ensure your Azure subscription has the Microsoft.SqlVirtualMachine resource provider registered. You can do this from the Azure Portal as part of your Azure subscription. Or you can use the following PowerShell command:

```
Register-AzureRmResourceProvider -ProviderNamespace Microsoft.SqlVirtualMachine
```

Next, you need to register your virtual machine with the resource provider. Registering your VM installs the SQL Server IaaS Agent Extension. You can read about all of the steps at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-agent-extension-manually-register-single-vm`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-agent-extension-manually-register-single-vm).

Once your VM is registered or if you used a SQL gallery image, your VM is now considered both an Azure Virtual Machine resource and a **SQL virtual machine**. This means your VM will show up when searching for **Azure SQL** resources. Additionally, the portal will show additional properties as seen in Figure [3-26](#496204_2_En_3_Chapter.xhtml#Fig26) based on the deployment I made earlier in this chapter.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig26_HTML.jpg)

Figure 3-26

A registered SQL Azure Virtual Machine

Note

One easy way to get to this overview page is to go to the Command Menu of your virtual machine and select SQL Server Configuration under Settings. Then click SQL Server Configuration.

You can see this screen looks like an Azure Virtual Machine but information and options specific to SQL Server.

You can see the Essentials pane has information about the VM but also SQL Server, including version, edition, license type, and OS details. Underneath the Working Pane is a graph showing storage for database and log files and features you can enable for SQL Server in Azure VM. The Command Bar has an option to delete (unregister) the SQL Server Azure Virtual Machine and delete the underlying VM.

The Service Menu (expanded in Figure [3-26](#496204_2_En_3_Chapter.xhtml#Fig26)) has options specific to SQL Server running on Azure Virtual Machine, including the following two main categories. I think you will find several of these options extremely valuable in helping you manage SQL Server on Azure Virtual Machines.

### Settings

Under Settings, you have the following options.

#### Configure

This option allows you to specify the license for SQL (Pay As You Go, Azure Hybrid Benefit, or DR).

I’ve described Azure Hybrid Benefit earlier in this chapter. Pay As You Go is the default billing method to be billed for the SQL License per month. DR is a new option to offer a free SQL Server license if the SQL Server in the VM is used only for disaster recovery purposes.

In addition, you can specify which edition of SQL Server you are using for the VM.

#### Storage

This option is worth looking at as it provides analysis, metrics, assessments, and configuration of storage as it relates to SQL Server for the VM as seen in Figure [3-27](#496204_2_En_3_Chapter.xhtml#Fig27).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig27_HTML.jpg)

Figure 3-27

Storage assistance for SQL Server in Azure VM

#### Updates

This is another nice option to manage not just security updates but other updates for Windows and Cumulative Updates for SQL Server. You can even manage and schedule updates at scale. This capability is based on the Azure Update Manager. Learn more at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/azure-update-manager-sql-vm`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/azure-update-manager-sql-vm).

#### Backups

This feature allows you to enable and disable automatic backups for SQL Server in the VM. This capability uses a feature that has been in the product for several releases using SQL Server Agent and backup to URL. You can learn more at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/automated-backup`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/automated-backup). While I do recommend you look at this feature, I found in the cloud customers typically build their own automated backup system (but use Azure Storage with backup to URL) or use the Azure Backup service.

#### High Availability

This option helps you configure Always On Availability Groups with SQL Server on Azure VM. Learn more about this option in the section titled “HADR” later in this chapter.

#### SQL Best Practices Assessment

This option is a tool that allows you to scan the configuration of your SQL Server deployment and provide advice on settings that may be not optimal or follow best practices based on the common knowledge of the SQL community and Microsoft. You can run an assessment on demand or on a schedule. There is some great advice from this tool, so I recommend you give it a spin and see how it can help you. You can see the complete set of rules it uses at [`https://github.com/microsoft/sql-server-samples/blob/master/samples/manage/sql-assessment-api`](https://github.com/microsoft/sql-server-samples/blob/master/samples/manage/sql-assessment-api) and learn more about the feature at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-assessment-for-sql-vm`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-assessment-for-sql-vm).

### Security

Security options include integration with services like Microsoft Entra and Microsoft Defender for Cloud.

#### Security Configure

This option allows you to enable integration with **Azure Key Vault** which can be a very nice option to use with features like Transparent Data Encryption (Bring Your Own Key).

You can also enable integration with **Microsoft Entra** (formerly Azure Active Directory). This option requires SQL Server 2022 and is an excellent solution to provide an alternative for authentication instead of using basic or SQL Authentication, especially because you don’t need a domain controller like you do for Windows Authentication. One great use for this capability is to use a managed identity, so you can build *passwordless* applications connecting to SQL Server. Learn more at [`https://learn.microsoft.com/sql/relational-databases/security/authentication-access/azure-ad-authentication-sql-server-overview`](https://learn.microsoft.com/sql/relational-databases/security/authentication-access/azure-ad-authentication-sql-server-overview).

#### Microsoft Defender for Cloud

This capability used to be known as Advanced Data Security but is now wrapped under the Microsoft Defender umbrella. The capabilities here are still the same:

*   Vulnerability Assessment

    Scan for any configuration settings that could lead to a possible security vulnerability (e.g., sa account enabled) based on known industry standards (e.g., FedRAMP).

*   Advanced Threat Protection

    Alert to any threats detected like brute force login attacks (SQL auth enabled and someone tries to hack the sa password) and SQL injection attacks.

You can learn more about Microsoft Defender for Cloud and SQL on Azure VM at [`https://learn.microsoft.com/azure/defender-for-cloud/defender-for-sql-usage`](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-sql-usage).

Tip

There are two new capabilities from the Defender team worth looking at, including the **Data Security Dashboard** ([`https://learn.microsoft.com/azure/defender-for-cloud/data-aware-security-dashboard-overview`](https://learn.microsoft.com/azure/defender-for-cloud/data-aware-security-dashboard-overview)) and a **Copilot experience** ([`https://learn.microsoft.com/azure/defender-for-cloud/copilot-security-in-defender-for-cloud`](https://learn.microsoft.com/azure/defender-for-cloud/copilot-security-in-defender-for-cloud)).

There is one other option on the Service Menu worth looking at called **SQL Server IaaS Agent Extension Settings**. This can help you repair any issues with the extension and also set up auto upgrades.

In addition to using the portal, you can use the **az sql vm** CLI. This allows you to register your VM or change SQL Server properties. You can read all these options at [`https://learn.microsoft.com/cli/azure/sql/vm`](https://learn.microsoft.com/cli/azure/sql/vm).

## Configuring an Azure Virtual Machine

Inside the VM, your SQL Server configuration is completely identical to a virtual machine you deploy in your data center. There are some aspects to configuration of the virtual machine from the Azure infrastructure worth looking at.

### Stopping vs. Deallocating

There could be several reasons to shut down your virtual machine in Azure. If you shut down the virtual machine inside the VM (e.g., Windows shutdown), the VM is stopped, but compute resources are reserved for the VM, which means you are still billed for compute. The VM status will show as **Stopped** in the portal and CLI interfaces. To shut down the VM and ensure you are not billed for compute, you need to use the interfaces outside the VM (such as Stop in the Command Menu of the Azure Portal or az CLI) to stop the VM. In this case, the status of the VM is listed as **Stopped (deallocated)**. Depending on your requirements, you can even automate the process of starting and stopping VMs using Azure interfaces. Read more at [`https://techcommunity.microsoft.com/t5/educator-developer-blog/azure-virtual-machine-auto-shutdown/ba-p/379342`](https://techcommunity.microsoft.com/t5/educator-developer-blog/azure-virtual-machine-auto-shutdown/ba-p/379342).

A related topic is maintenance windows for the infrastructure hosting virtual machines. Read more about updates and maintenance of Azure Virtual Machine hosts at [`https://learn.microsoft.com/azure/virtual-machines/maintenance-and-updates`](https://learn.microsoft.com/azure/virtual-machines/maintenance-and-updates).

### Resizing

What if you did not pick the right Azure Virtual Machine size during deployment? You can use the Azure portal (choose Size from the Service Menu) or CLI interfaces to change the size of the virtual machine. This process is called *resizing* an Azure Virtual Machine. Resizing is much like you being able to change the resources available (CPU, storage, memory, etc.) for a VM within your data center. While you can resize a running Azure VM, a reboot of the VM is required for the resize operation. The sizes available to your choice to resize depend on whether your VM is running or stopped (deallocated). This difference is because there may be only certain sizes available on the current host of your VM (the running case). To see the full complete list of VM sizes, I recommend you stop (deallocate) your VM first. Read the overall Azure Virtual Machine resize story at [`https://azure.microsoft.com/blog/resize-virtual-machines/`](https://azure.microsoft.com/blog/resize-virtual-machines/).

You can use Azure Migrate to move Azure Virtual Machines to other regions. Read more at [`https://learn.microsoft.com/azure/site-recovery/azure-to-azure-tutorial-migrate`](https://learn.microsoft.com/azure/site-recovery/azure-to-azure-tutorial-migrate). In addition, to change the resource group or subscription of your virtual machine, read the documentation at [`https://learn.microsoft.com/azure/azure-resource-manager/management/move-resource-group-and-subscription`](https://learn.microsoft.com/azure/azure-resource-manager/management/move-resource-group-and-subscription).

### Security (RBAC)

Role-Based Access Control (RBAC) allows you to assign privileges to other Azure accounts to have permissions to manage the Azure Virtual Machine. Think of this as how to control security *outside the VM*. You manage this through the **Access Control (IAM)** option on the Service Menu. This is a common option for many Azure resources. Learn more at [`https://learn.microsoft.com/azure/role-based-access-control/quickstart-assign-role-user-portal`](https://learn.microsoft.com/azure/role-based-access-control/quickstart-assign-role-user-portal).

### Other Config Options

There are other options to configure your Azure Virtual Machine using the portal from the Service Menu or CLI interfaces.

Examples include resetting the password for the admin user, viewing security recommendations, and redeploying the VM to another host. Read more about some of these options at [`https://learn.microsoft.com/azure/virtual-machines/windows/tutorial-config-management`](https://learn.microsoft.com/azure/virtual-machines/windows/tutorial-config-management).

## Maximizing Storage Performance

Based on our experience with customers, getting the right size for your Azure Virtual Machine is the number one factor in ensuring you have the right performance you need for SQL Server (except for standard SQL Server performance optimization practices). Let’s go into more detail as why this choice is so important.

### Best Practices

You saw earlier in this chapter the example of how a SQL Server gallery image will create additional managed *data disks* based on Azure Premium Storage. To achieve your best, I/O performance, *and availability* SQL Server on Azure Virtual Machine, you should keep these principles in mind, some of which I have mentioned as I showed you how to configure storage in the Azure Portal:

*   Use the Azure Portal to help guide your choices as it includes warnings if the VM size doesn’t support your I/O choices. It also helps you easily configure multiple disks (which use Windows Storage Spaces behind the scenes), separate your data and log storage, move system databases off the OS drive, and configure tempdb storage.

*   Do not create any user databases on the OS disk.

*   Put tempdb on the local SSD drive which is also called temporary storage. If your VM fails over for any reason to a different host, the contents of tempdb are lost, but who cares since tempdb is recreated at startup time.

*   Separate your database and log files on different *managed* disks as you would with any SQL Server production deployment.

*   Use premium managed disks at minimum (also called Premium SSD). One option for extreme low latency is Ultra disks. Ultra disks are more expensive but for SQL production workload could be a needed choice for heavy write workloads where the latency of the transaction log is important. Learn more about how to configure this at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/storage-migrate-to-ultradisk`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/storage-migrate-to-ultradisk).

### Premium SSD v2

As you saw from the examples of configuring storage via the Azure Portal, it is possible you need to overprovision storage to achieve your IOPS and throughput needs. This is because these resource numbers are tied to the size of premium managed disks.

Premium SSD v2 for SQL (now in preview) provides a new method to control your IOPS and throughput independent of the size of disks.

We have a new Azure portal experience to use this new option. However, as of the time of the writing of this book, it requires you to deploy your VM with an Availability Zone, even for a single SQL instance. While this will not cause any specific issue for you, deploying an Azure VM in a single Availability Zone does not really gain you any extra availability. Availability Zones are typically only needed for virtual machines for HA scenarios like Always On Availability Groups.

If you make this choice on the Basics screen of the portal (under Availability Options), you will see a new option under the Storage Configuration under SQL Server Setting that looks like Figure [3-28](#496204_2_En_3_Chapter.xhtml#Fig28).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig28_HTML.jpg)

Figure 3-28

Choosing the new Premium SSD v2 storage for SQL

When you select Change configuration, you now see a new portal experience to control size, IOPS, and throughput independently as seen in Figure [3-29](#496204_2_En_3_Chapter.xhtml#Fig29).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig29_HTML.jpg)

Figure 3-29

The new Premium SSD v2 storage experience

Learn more details at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/storage-configuration-premium-ssd-v2`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/storage-configuration-premium-ssd-v2).

### Learn More

You of course must balance these choices with cost. All the managed disks are based on Azure Blob Storage. An entire chapter would be needed to give you all the internals of how Azure Blob Storage and managed disks work. For now, look at these resources:

*   Details on Azure managed disk types can be found at [`https://learn.microsoft.com/azure/virtual-machines/disks-types`](https://learn.microsoft.com/azure/virtual-machines/disks-types). You will see in these tables what sizes, IOPS, and throughput each type supports.

*   We have guidance in our documentation specific to choosing the right storage for SQL Server for Azure VM at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/performance-guidelines-best-practices-checklist`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/performance-guidelines-best-practices-checklist).

*   This documentation provides nice guidance on choosing the right sizes for Azure Premium Storage along with VM size. You can read this at [`https://learn.microsoft.com/azure/virtual-machines/premium-storage-performance`](https://learn.microsoft.com/azure/virtual-machines/premium-storage-performance).

*   An independent study on performance for SQL Server on Azure Virtual Machine was published by GigaOm. Learn more at [`https://techcommunity.microsoft.com/t5/sql-server-blog/sql-server-on-azure-vms-the-best-price-performance-gets-even/ba-p/3727091`](https://techcommunity.microsoft.com/t5/sql-server-blog/sql-server-on-azure-vms-the-best-price-performance-gets-even/ba-p/3727091).

One major key point I’ve mentioned before but worth repeating is to **match the right VM size to meet your storage needs** as VM sizes have limits on the number of data disk, total IOPS, and throughput across all disks. Then provision the data disks you need for database and transaction log files within that total IOPS.

## Performance Monitoring

Since SQL Server is deployed inside a virtual machine, you should use all the normal techniques available to you to monitor performance, including SQL tools such as Dynamic Management Views (DMVS) and operating system tools such as Windows Performance Monitor or Linux tools.

Having said that, Azure Monitor provides integrated performance metrics for virtual machines, including SQL Server performance counters (for Windows only).

### Azure Metrics

You can get rich metrics from Azure available in the Monitoring option in the Service Menu. By default, you get metrics for the host of the VM. Most likely though, you will want metrics for *inside* (a.k.a. guest) the VM. In order to enable this, you will need to select Diagnostic Settings from the Service Menu of the VM (not the SQL Virtual Machine), provision a new Azure Storage account in the same region as the VM (see these instructions on how to create one quickly: [`https://learn.microsoft.com/azure/storage/common/storage-account-create`](https://learn.microsoft.com/azure/storage/common/storage-account-create)), and then select **Enable guest-level monitoring**.

After you enable guest-level monitoring, you will be presented with various options, including the ability to add specific counters for the guest VM. Under the Performance Counters option, you can select SQL Server counters. Once you have applied this, you can now use the Metric option in the Service Menu in the portal to see fundamental metrics like you would inside the VM with Windows Performance Monitor. Here is an example from my VM as seen in Figure [3-30](#496204_2_En_3_Chapter.xhtml#Fig30).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig30_HTML.jpg)

Figure 3-30

Azure guest VM metrics for a virtual machine

You can also select the **Logs** option from the Service Menu to view Azure Metrics by using Kusto Query Language (KQL) queries. This allows you to view metrics from Log Analytics that can span more than the 14-day default of using Azure Metrics from the portal. Learn more at [`https://learn.microsoft.com/azure/azure-monitor/vm/tutorial-monitor-vm-guest#view-logs`](https://learn.microsoft.com/azure/azure-monitor/vm/tutorial-monitor-vm-guest%2523view-logs).

### Insights

When you selected **Enabled guest-level monitoring**, Azure Monitor started collecting metrics in a Log Analytics workspace. By choosing the **Insights** option from the Service Menu, you can look at another visualization of these metrics. You can see an example of these metrics in Figure [3-31](#496204_2_En_3_Chapter.xhtml#Fig31).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig31_HTML.jpg)

Figure 3-31

Insights for Azure Virtual Machine using Log Analytics

You can go even farther using a concept called Workbooks, which you can read more about at [`https://learn.microsoft.com/azure/azure-monitor/vm/vminsights-workbooks`](https://learn.microsoft.com/azure/azure-monitor/vm/vminsights-workbooks).

## Networking

When I deployed the Azure Virtual Machine earlier in this chapter, several types of network resources were created in the resource group, including a virtual network, network interface, public IP address, and network security group.

The public IP address is important as it allows you to use tools like Remote Desktop to connect to the virtual machine. The network interface is the interconnection between the VM and the virtual network. The network security group (NSG) provides an Access Control List (ACL) for rules to allow or deny network traffic to the VM (like a firewall). You can read the complete picture of networks and Azure Virtual Machines at [`https://learn.microsoft.com/azure/virtual-network/network-overview#network-security-groups`](https://learn.microsoft.com/azure/virtual-network/network-overview%2523network-security-groups).

Previously, I connected to the virtual machine using the Remote Desktop Client (RDP) which used the public IP address and access to the RDP port (3389). There could be other methods you want to use to connect to the virtual machine.

I recommend against enabling the SQL Server port 1433 to be open to the public Internet. However, that is the only method for you to connect a SQL Server client application or tool (like SSMS) to the SQL Server instance in the Azure Virtual Machine unless you run the application or tool in a computer that is connected to the virtual network of the VM. One method to do this is to deploy the application or another VM in Azure and join the same Azure virtual network. You can also deploy in another Azure virtual network and set up a configuration called virtual network peering, which you can read about at [`https://learn.microsoft.com/azure/virtual-network/virtual-network-peering-overview`](https://learn.microsoft.com/azure/virtual-network/virtual-network-peering-overview).

Finally, you may find yourself wanting to connect on-premises resources such as applications or other computer or virtual machines to the virtual network for the Azure VM. You can read more about the details of setting up a configuration like this at [`https://learn.microsoft.com/azure/architecture/reference-architectures/hybrid-networking`](https://learn.microsoft.com/azure/architecture/reference-architectures/hybrid-networking).

## HADR

No matter where you deploy SQL Server, almost everyone wants high availability at some level and the ability to execute, when necessary, disaster recovery techniques. Azure provides methods to deliver several choices for HADR.

### Azure Storage

In the example in this chapter where I showed you how to deploy SQL Server in Azure Virtual Machine, I used *data disks* based on premium managed disks from Azure Storage. By default, managed disks have built-in redundancy called **Locally Redundant Storage (LRS)**. LRS maintains three copies of data within the data center region. You can read more about Azure Storage redundancy at [`https://learn.microsoft.com/azure/storage/common/storage-redundancy`](https://learn.microsoft.com/azure/storage/common/storage-redundancy). You can actually make your disk zone redundant, but because the data is not replicated across zones, you could see a performance impact.

### Backups

Since you are running SQL Server in the Azure infrastructure, you will no doubt want to store backups of SQL Server database and transaction logs within Azure. You could store your backups using T-SQL to a separate data disk using managed disks. However, another option is to store your backups to Azure Storage accounts using the *backup to URL* capability of SQL Server. You can read more about SQL Server backup to URL at [`https://learn.microsoft.com/sql/relational-databases/backup-restore/sql-server-backup-to-url`](https://learn.microsoft.com/sql/relational-databases/backup-restore/sql-server-backup-to-url). I mentioned earlier the option of using automatic backups from the Azure Portal which uses backup to URL. An advantage of using URL backups is you can configure your Azure Storage account to use a higher level of availability called **geo-redundant storage (GRS)**. View the Service-Level Agreement (SLA) for Azure Storage accounts at [`https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services`](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services%253Flang%253D1).

Another option is to integrate backups for SQL Server with the **Azure Backup** service. Azure Backup offers a streaming-based service to back up SQL Server to Azure Storage. Azure Backup involves a VM extension that will use the Virtual Device Interface (VDI) APIs with SQL Server for backups. Read more about using Azure Backup with SQL Server at [`https://learn.microsoft.com/azure/backup/backup-azure-sql-database`](https://learn.microsoft.com/azure/backup/backup-azure-sql-database).

The final option for backups in Azure uses **file snapshot backups**. File snapshot backups can be extremely fast. They require storing SQL Server data files directly to an Azure Storage account vs. a managed disk. You can read more about SQL Server file snapshot backups in Azure at [`https://learn.microsoft.com/sql/relational-databases/backup-restore/file-snapshot-backups-for-database-files-in-azure`](https://learn.microsoft.com/sql/relational-databases/backup-restore/file-snapshot-backups-for-database-files-in-azure).

One other concept to consider is **Azure File Share**. Azure File Shares are like a network file share you might have in your organization but available to any virtual machine (think of an SMB-based network file share). Any file stored on this file share is available even if the VM is not available. An option with Azure File Shares is called a **premium file share**, which can support up to 100TB.

Tip

Since you have premium managed disk attached to the VM, you could also back up your databases to one of those disks locally (like backup to the G: drive). However, these disks are only available when attached to a VM. Using the other methods listed in this section allows you to access your backups independent of the VM being available.

### Go Further with Azure Availability

Azure Virtual Machine is part of the Azure infrastructure that provides for availability during unplanned and planned events. For example, if an unplanned hardware problem occurs, Azure can use live migration technologies to failover your virtual machine to a healthy host. Azure also may update the underlying hosts of your virtual machine which may or may not require a reboot. You can read more about unplanned and planned downtime for Azure Virtual Machines at [`https://docs.microsoft.com/azure/virtual-machines/windows/manage-availability#understand-vm-reboots---maintenance-vs-downtime`](https://docs.microsoft.com/azure/virtual-machines/windows/manage-availability%2523understand-vm-reboots%252D%252D-maintenance-vs-downtime).

Azure offers more options to go further with high availability for Azure Virtual Machines which I’ve mentioned earlier in this book but worth repeating:

1.  **Availability Set** – Spreads your VMs across multiple fault and update domains in the same data center. Basically, this makes sure your VMs are spread across different racks and switches. The SLA is 99.95%.

2.  **Availability Zone** – Think of this as spreading an Availability Set across data centers within a region. The SLA is 99.99% but can cost more.

Unfortunately, you need to make this choice when you deploy. You can change it later, but you must migrate your VM to do it.

These choices are required if you plan to use an HADR solution such as an Always On Failover Cluster Instance or Availability Group. It does not make sense to use either of these options for a single VM deployment.

### Always On Failover Cluster Instance

An Always On Failover Cluster Instance (FCI) provides high availability for SQL Server through shared storage. Since this is a virtual machine, using an FCI will be very much like a normal SQL Server. Having said that, consider these important concepts to use to make the deployment of an FCI successful in Azure:

*   You will need to deploy any VM in the cluster using an Availability Set or Availability Zone.

*   For shared storage for the cluster, consider **Azure Shared Disks**. Azure Shared Disks are specifically designed to share managed disks across virtual machines for scenarios like clusters. See more at [`https://learn.microsoft.com/azure/virtual-machines/disks-shared`](https://learn.microsoft.com/azure/virtual-machines/disks-shared).

*   Every Windows Failover Cluster Solution (WFCS) needs a witness for quorum. Azure provides a specific type of witness for the cloud called a **cloud witness**. Think of a cloud witness as a *managed witness*. You create the cloud witness, and it handles everything you need. Behind the scenes, it uses a VM and Azure Storage. Learn more at [`https://learn.microsoft.com/windows-server/failover-clustering/deploy-cloud-witness?tabs=windows#what-is-cloud-witness`](https://learn.microsoft.com/windows-server/failover-clustering/deploy-cloud-witness%253Ftabs%253Dwindows%2523what-is-cloud-witness).

*   You need some type of network abstraction for listeners. Azure has the ability for you to create a **multi-subnet virtual network** to avoid the need for load balancers or distributed network names. Learn more at [`https://learn.microsoft.com/sql/sql-server/failover-clusters/windows/sql-server-multi-subnet-clustering-sql-server`](https://learn.microsoft.com/sql/sql-server/failover-clusters/windows/sql-server-multi-subnet-clustering-sql-server).

For a complete tutorial to build an FCI, start here and then follow the next step with each article: [`https://learn.microsoft.com/sql/sql-server/failover-clusters/install/before-installing-failover-clustering`](https://learn.microsoft.com/sql/sql-server/failover-clusters/install/before-installing-failover-clustering).

### Always On Availability Groups

Always On Availability Groups (AG) increase your high availability RPO (Recovery Point Objective) and RTO (Recovery Time Object) using replicas. It is the flagship HADR capability for SQL Server using a non-shared storage concept with replicas.

We know that setting up an AG on Azure can be a daunting process. In fact, the complete tutorial to set up an AG with auto failover on Windows is documented at [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/availability-group-manually-configure-tutorial-single-subnet`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/availability-group-manually-configure-tutorial-single-subnet).

Fortunately, our team built a new experience in the Azure Portal to make this easier. Assuming you first set up a Windows Domain Controller and a virtual network and then use a new Azure portal experience to deploy the rest of the solution, including failover clustering and an AG.

If you can remember earlier in the chapter, I showed you the experience to pick an Azure SQL solution like in Figure [3-32](#496204_2_En_3_Chapter.xhtml#Fig32).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig32_HTML.jpg)

Figure 3-32

Choosing the HA option when deploying a SQL Azure VM

Notice the selection of high availability here. After picking your image choice and selecting Create, you will have a similar experience as I have shown you earlier in this chapter, except now you have choices to deploy *multiple VMs* and configure your cluster and AG.

For example, as you can see in Figure [3-33](#496204_2_En_3_Chapter.xhtml#Fig33), you have choices in the Basics screen to deploy multiple VMs.

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig33_HTML.jpg)

Figure 3-33

Basics for deploying a SQL HA deployment

In the Networking screen, you have options to set up a multi-subnet, including Listener IP addresses. Notice also an option for WSFC and Credentials. When you select SQL Server Settings, you now have options to configure the AG to deploy as you see in Figure [3-34](#496204_2_En_3_Chapter.xhtml#Fig34).

![](images/496204_2_En_3_Chapter/496204_2_En_3_Fig34_HTML.jpg)

Figure 3-34

SQL Server Settings to configure an AG for SQL Server Azure VM

To go through this complete experience, see [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/availability-group-azure-portal-configure`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/availability-group-azure-portal-configure).

One other scenario to consider is to set up a secondary replica for an AG in Azure from an on-premises AG primary replica. You can get more guidance on how to set this up at [`https://learn.microsoft.com/previous-versions/azure/virtual-machines/windows/sqlclassic/virtual-machines-windows-classic-sql-onprem-availability`](https://learn.microsoft.com/previous-versions/azure/virtual-machines/windows/sqlclassic/virtual-machines-windows-classic-sql-onprem-availability). And if you set up an async secondary replica to be *passive*, you can tag this VM for HA purposes and avoid SQL licensing costs.

### SQL Server and Linux High Availability

SQL Server on Linux supports Always On Failover Cluster Instance and Availability Groups functionality through the database engine. Automatic failover capabilities are often supported through Linux packages like Pacemaker, DH2i, or HPE ServiceGuard.

To see the complete story of deploying an Always On Availability Group on Linux, see [`https://learn.microsoft.com/sql/linux/sql-server-linux-availability-group-ha`](https://learn.microsoft.com/sql/linux/sql-server-linux-availability-group-ha).

## Microsoft Innovations and Azure Boost

Microsoft is always looking to improve the infrastructure that hosts Azure Virtual Machines. One of these innovations is called **Azure Boost**.

Azure Boost is a system designed by Microsoft that offloads server virtualization processes traditionally performed by the hypervisor and host OS onto purpose-built software and hardware.

The great thing about Azure Boost is that you just need to choose a virtual machine size that supports Boost (which you can find at [`https://learn.microsoft.com/azure/azure-boost/overview#current-availability`](https://learn.microsoft.com/azure/azure-boost/overview%2523current-availability)), and all the goodness is delivered behind the scenes on hosts.

The main benefits of using a virtual machine size that supports Boost are faster networking, faster storage with lower latency, higher IOPS/throughput, a more secure host, and higher performance because CPU resources can be more dedicated to the guest virtual machines. Geek out on all the details at [`https://aka.ms/azureboost`](https://aka.ms/azureboost).

## Summary

You have learned in this chapter how to deploy and configure SQL Server with Azure Virtual Machine looking at many different options as part of deployment. You have also learned unique aspects of security, performance, and HADR related to SQL Server and Azure VM. Azure VM provides a great infrastructure for running SQL Server and is the easiest way to migrate your SQL Server to the cloud.

Try out your knowledge of what you have learned in this chapter by going through the following labs in my Cloud Workshop for the SQL Professional GitHub repo ([`https://aka.ms/cloudsqlworkshop`](https://aka.ms/cloudsqlworkshop)):

[`https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/03_Deploy_SQL_AzureVM`](https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/03_Deploy_SQL_AzureVM)

[`https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/04_Manage_and_Optimize_SQL_AzureVM`](https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/04_Manage_and_Optimize_SQL_AzureVM)

Now that you have studied SQL Server on Azure VM, it is time to show you how to deploy and configure Azure SQL Database and Managed Instance.

# 4. Deploying Azure SQL

Deploying an Azure SQL Managed Instance or Database is a different but similar experience than deploying SQL Server on Azure Virtual Machine. The experience is the same because you can use the Azure portal and CLI. The difference is that Azure is *managing the virtual machine* and infrastructure, so there are several of the options you pick for a virtual machine you do not have to worry about.

In this chapter, you will learn the options and process to deploy and connect to an Azure SQL Managed Instance and Database. You will also learn the options to migrate existing databases into Azure SQL. In addition, you will learn some implementation details of the architecture used to host Azure SQL Managed Instances and Databases.

You have the option to follow along with the examples in this chapter. You will need the following to complete these examples:

*   An Azure subscription.

*   A minimum of Contributor role access to the Azure subscription. You can read more about Azure built-in roles at [`https://learn.microsoft.com/azure/role-based-access-control/built-in-roles`](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles).

*   Access to the Azure Portal (web or Windows application).

*   Installation of the **az** CLI (see [`https://learn.microsoft.com/cli/azure/install-azure-cli`](https://learn.microsoft.com/cli/azure/install-azure-cli) for more details). You can also use the Azure Cloud Shell instead since az is already installed. You can read more about the Azure Cloud Shell at [`https://azure.microsoft.com/get-started/azure-portal/cloud-shell`](https://azure.microsoft.com/get-started/azure-portal/cloud-shell).

*   You will run some T-SQL in this chapter, so install a tool like SQL Server Management Studio (SSMS) at [`https://aka.ms/ssms`](https://aka.ms/ssms).

I also have a link to self-paced labs you can go through at the end of this chapter.

## Pre-deployment Planning

Before you jump into deploying an Azure SQL Database or Managed Instance, I recommend you spend some time doing some pre-deployment planning. Reviewing your choices and making a few informed decisions will save you time and effort.

### New Deployment or Migration

One of the first decisions to make is whether you plan to migrate an existing database or instance or deploy a new database or instance. The process of deploying will be the same, but migrating implies you need to assess your current SQL Server instance, database, or other database environments before you deploy. Your assessment will give you guidance on what type of deployment choice you need to make based on your current requirements. You must decide what possible changes must be made to your application, schema, scripts, or other aspects to your current deployment with SQL Server or other database platforms. You also must consider how to migrate your actual data into the new deployment. This chapter will include sections specifically geared around what consideration and tools you can use to migrate to Azure SQL Managed Instance and Database. Here are some great resources for you to consider as you think about migration:

*   **Azure cloud migration and modernization center** – Here is a central resource of information and access to Microsoft experts to migrate any resources to Azure. Check out [`https://azure.microsoft.com/solutions/migration`](https://azure.microsoft.com/solutions/migration).

*   **The Microsoft Data Migration Guide** – Get step-by-step guidance on migrating to Azure for any type of data. Learn more at [`https://aka.ms/datamigration`](https://aka.ms/datamigration).

*   **SQL Server enabled by Azure Arc migration assessment** – In preview at the time of the writing of this book, this capability allows you to connect your existing SQL Server to Azure with Arc and get a *continuous* assessment on what is the best solution to migrate to Azure, including resource (sizing) recommendations. Learn more at [`https://learn.microsoft.com/sql/sql-server/azure-arc/migration-assessment`](https://learn.microsoft.com/sql/sql-server/azure-arc/migration-assessment).

*   **Azure** **Database Migration Service (DMS)** – This Azure service will allow to perform both offline and online migrations to Azure SQL Managed Instance and Database (offline only). Learn more at [`https://learn.microsoft.com/azure/dms/dms-overview`](https://learn.microsoft.com/azure/dms/dms-overview).

*   **Azure Migrate** – Use this service to assess and migrate (using DMS) at *scale*. Learn more at [`https://aka.ms/azuremigrate`](https://aka.ms/azuremigrate).

*   **Managed Instance link** – While DMS provides an online migration solution, it uses a log shipping approach. A possible better online migration path is Azure SQL Managed Instance link. This allows you to migrate a SQL Server 2016, 2019, or 2022 instance to Azure SQL Managed Instance using the power of Always On Availability Groups. Learn more at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/managed-instance-link-feature-overview?view=azuresql#migrate-to-azure`](https://learn.microsoft.com/azure/azure-sql/managed-instance/managed-instance-link-feature-overview%253Fview%253Dazuresql%2523migrate-to-azure).

### Making Deployment Choices

Whether you are migrating or creating a new deployment, you have several choices to make that are worth the time to plan out. Chapter [2](#496204_2_En_2_Chapter.xhtml) of this book is invaluable to go back and read as it describes choices and differences between Azure SQL Managed Instance and Azure SQL Database.

Having said that, let us review some important choices at a high level that can affect your decision-making:

*   If you need SQL Server instance features like SQL Server Agent, Database Mail, and cross-database queries, Managed Instance is the choice you need to make.

*   If your database size is > 32TB, *your only choice* as of the time of writing this book is Azure SQL Database Hyperscale.

Past these two choices, either Azure SQL Managed Instance or Database likely meets your needs. However, as I called out in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book, there can be advantages in using Azure SQL Database because Microsoft will manage both the infrastructure and the SQL Server instance to let you focus on the database. Furthermore, Azure SQL Database can offer you more options such as Serverless compute and elastic pools.

The options you will pick as you deploy a Managed Instance or Database can be thought of the following as listed in Figure [4-1](#496204_2_En_4_Chapter.xhtml#Fig1) (credits to my colleague Anna Hoffman for this diagram).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig1_HTML.jpg)

Figure 4-1

Deployment choices for Azure SQL

#### Deploying for Free

If you are looking for ways to deploy Azure SQL for free for development or proof of concept (POC) scenarios, consider the following options:

##### The Azure SQL Managed Instance Free Trial

I have heard so many customers ask me how they can do tests for Azure SQL Managed Instance to migrate a SQL Server without having to spend a lot of money. We now have a free 12-month trial for Azure SQL Managed Instance to now give you this capability. This is available for any Azure subscription. Learn more at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/free-offer`](https://learn.microsoft.com/azure/azure-sql/managed-instance/free-offer).

##### The Azure SQL Database Free Offer

SQL Server Developer Edition is so popular because developers can develop and test their application for free. We now have a similar option for Azure SQL Database, and it is not a trial. For the lifetime of your Azure subscription, you can now deploy a free Azure SQL Database. Learn more at [`https://aka.ms/freedboffer`](https://aka.ms/freedboffer).

#### Deployment Methods

You can deploy an Azure Managed Instance or Database using the Azure Portal or through a CLI with the az utility, PowerShell, or even REST APIs (az rest can be used to help you).

If you are just trying out Azure SQL or doing a POC, you can easily use the Azure Portal. However, for a repeatable process to deploy (imagine if you needed to redeploy a Managed Instance exactly as it was originally configured), then a script using a CLI is a better option. Remember you also can use Azure templates to help automate deployments. You can read more about using Azure templates for Azure SQL at [`https://learn.microsoft.com/azure/azure-sql/database/arm-templates-content-guide?view=azuresql&tabs=single-database`](https://learn.microsoft.com/azure/azure-sql/database/arm-templates-content-guide%253Fview%253Dazuresql%2526tabs%253Dsingle-database) and [`https://learn.microsoft.com/azure/azure-sql/database/arm-templates-content-guide?view=azuresql&tabs=managed-instance`](https://learn.microsoft.com/azure/azure-sql/database/arm-templates-content-guide%253Fview%253Dazuresql%2526tabs%253Dmanaged-instance).

You also have the option to use Bicep to deploy Azure SQL Managed Instance ([`https://learn.microsoft.com/azure/azure-sql/managed-instance/create-bicep-quickstart`](https://learn.microsoft.com/azure/azure-sql/managed-instance/create-bicep-quickstart)) or Azure SQL Database ([`https://learn.microsoft.com/azure/azure-sql/database/single-database-create-bicep-quickstart`](https://learn.microsoft.com/azure/azure-sql/database/single-database-create-bicep-quickstart)).

Terraform is also supported both for Azure SQL Managed Instance ([`https://learn.microsoft.com/azure/azure-sql/managed-instance/instance-create-terraform`](https://learn.microsoft.com/azure/azure-sql/managed-instance/instance-create-terraform)) and Azure SQL Database ([`https://learn.microsoft.com/azure/azure-sql/database/single-database-create-terraform-quickstart`](https://learn.microsoft.com/azure/azure-sql/database/single-database-create-terraform-quickstart)).

Let me show you another method. I mentioned the Copilot for Azure experience in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book. Here is an example in Figure [4-2](#496204_2_En_4_Chapter.xhtml#Fig2) of how to ask Copilot to help provide an example of how to deploy an Azure SQL Database using the az CLI.

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig2_HTML.jpg)

Figure 4-2

Using Copilot in Azure to help deploy an Azure SQL Database

Note

You can see here that after providing responses Copilot provides suggestions for other things you may want to do. At this point, I have not deployed a database, so Copilot has no *context*. You will see later in the book more sophisticated experiences after you deploy the database. However, as you go through this chapter, you can ask Copilot different questions related to topics you see in the chapter. It can look up concepts in the documentation and provide examples in many cases faster than you probably can search for yourself.

#### Deployment Option

I discussed earlier whether you will consider Azure SQL Managed Instance or Database. Within each of these options, there is a choice of using a *pool*. As I mentioned in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book, Azure SQL Managed Instance offers a *Managed Instance pool* which could be a better fit for a smaller, cost-effective Managed Instance. Deployment times are also much faster with pools. You can read more about Managed Instance pools at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/instance-pools-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/instance-pools-overview). I’ll discuss the use of Managed Instance pools in several places later in this chapter.

Note

Managed Instance pools at the time of the writing of this book went through a public preview *refresh*. It was in public preview for a long time, so many thought it would never make General Availability, but it is on the right path now in my opinion.

As I mentioned in both Chapters [1](#496204_2_En_1_Chapter.xhtml) and [2](#496204_2_En_2_Chapter.xhtml) of this book, Azure SQL Database provides an option called an *elastic pool*. An elastic pool can be a good choice if you plan to use Azure SQL Database to host many databases. ISVs and Software as a Service (SaaS) developers often look at this choice to save costs and manage databases more efficiently. I’ll discuss more about elastic pools in several places later in the chapter.

#### Region

Choosing a specific Azure region can be important just as I described in Chapter [3](#496204_2_En_3_Chapter.xhtml) on virtual machines. You need to make sure your deployment options are available in your Azure region choice.

You may have certain compliance and security requirements which also dictate what region you choose.

You may be implementing specific HADR options like Availability Zones, geo-replication, or auto-failover groups and have specific regions in mind to make those deployment successful. Remember I talked about in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book the concept called *paired regions*, which are important to make these geo-region choices.

If you need to move an Azure SQL Managed Instance or Database to another region, read a checklist in our documentation at [`https://learn.microsoft.com/azure/azure-sql/database/move-resources-across-regions`](https://learn.microsoft.com/azure/azure-sql/database/move-resources-across-regions).

Azure SQL Database is a “Ring 0” service which means it gets deployed in every region as a default service. Managed Instance is not exactly at that status yet, but it is generally available in all regions.

In addition, you need to consider where your application will be hosted and latency requirements between where the application will be hosted and your Azure SQL deployment. Consider performance and proximity to other services. I was chatting with my colleague Anna Hoffman on this topic. She said “…but I feel that it’s not just where the application is – where are the users? Where should the application be? If you have geo-replication or auto failover groups, how do you build a globally available solution?”

#### Purchasing Model

For Azure SQL Database only, you will need to choose a *purchasing model*. The choices are DTU or vCore. I explained these models and the history behind them in Chapters [1](#496204_2_En_1_Chapter.xhtml) and [2](#496204_2_En_2_Chapter.xhtml) of the book. While the DTU model may be a choice for you, I recommend the vCore model. You have more choices with vCore and licensing options such as Azure Hybrid Benefit (AHB). The vCore model gives you more transparency and control over the underlying resources (CPU, memory, storage), making it easier to match your database’s performance needs.

If you select the DTU model and want to move to the vCore model at a later date, consult the documentation at [`https://learn.microsoft.com/azure/azure-sql/database/migrate-dtu-to-vcore`](https://learn.microsoft.com/azure/azure-sql/database/migrate-dtu-to-vcore).

#### Service Tier (SLO)

If Azure SQL Managed Instance is your deployment option, then you will need to select a *service tier* (we also refer to this as the Service-Level Objective or SLO) of General Purpose (GP) or Business Critical (BC). I described these service tier options in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book. While resource limits and performance may differ, one of the primary differences with these tiers is how availability works which you will learn more about in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book. One notable difference between Business Critical is that it supports In-Memory OLTP capabilities. Remember in Chapter [2](#496204_2_En_2_Chapter.xhtml) I talked about the “NextGen” of GP for Managed Instance. Be sure to look at that option because it gives you more options and more powerful storage performance. A comparison between GP and BC for Managed Instance can be found at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits`](https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits).

Tip

You will see later in this chapter that it can often take time to deploy a Managed Instance. Changing between GP and BC is possible but could result in significant downtime. You can learn more about specific times at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-overview). Managed Instance does support a concept called fast provisioning for GP service tiers with certain configurations. You can even provision them in 30 minutes. Learn more at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-overview?view=azuresql#fast-provisioning`](https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-overview?view=azuresql#fast-provisioning).

If Azure SQL Database is your deployment option, then you also have the choice of General Purpose (GP) vs. Business Critical (BC) service tiers. In addition, you have the choice of *Hyperscale*. If you choose General Purpose or Hyperscale, you also have the choice of *Provisioned* vs. *Serverless*. This is also called a *compute model* *or tier*. I covered all these options in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book, but I’ll also cover this topic in this chapter.

**Azure SQL Database Hyperscale** I believe will start becoming your ***default*** choice for databases. I talked about the history of Hyperscale (Project Socrates) in Chapter [1](#496204_2_En_1_Chapter.xhtml) of the book. If you read the first edition of this book, you probably walked away with the impression that Hyperscale is only for the largest database deployments. And that is our fault not yours for that impression. Several things have changed since 2020 for Hyperscale:

*   Hyperscale now supports serverless.

*   Hyperscale now supports elastic pools.

*   The price of Hyperscale in late 2023 was ***significantly reduced***. Learn more about this at [`https://azure.microsoft.com/updates/general-availability-lower-simplified-pricing-for-azure-sql-database-hyperscale-expected-in-december-2023`](https://azure.microsoft.com/updates/general-availability-lower-simplified-pricing-for-azure-sql-database-hyperscale-expected-in-december-2023).

One key point about Hyperscale is that storage is autoscaled! You never specify how much database space you need. We just auto grow per your needs.

**I believe Hyperscale now becomes a starting point for anyone** and just let auto-scaling for computer (serverless) and storage flex as your application needs grow or change. Read all the details about Hyperscale at [`https://learn.microsoft.com/azure/azure-sql/database/service-tier-hyperscale`](https://learn.microsoft.com/azure/azure-sql/database/service-tier-hyperscale).

And speaking of Serverless (which I talked about in Chapter [1](#496204_2_En_1_Chapter.xhtml) of the book), it is a unique option we have created for auto-scale and scenarios where your database may not always be utilized. It provides a new cost-effective way to deploy and use an Azure SQL Database. Read more about Serverless at [`https://learn.microsoft.com/azure/azure-sql/database/serverless-tier-overview`](https://learn.microsoft.com/azure/azure-sql/database/serverless-tier-overview).

Switching between GP and BC for Azure SQL Database is typically significantly faster than with Managed Instance. You can also switch between Serverless and Provisioned easily. For Hyperscale, you can switch back to a GP service tier. Learn more at [`https://learn.microsoft.com/azure/azure-sql/database/manage-hyperscale-database`](https://learn.microsoft.com/azure/azure-sql/database/manage-hyperscale-database).

#### Hardware

Even though for Azure SQL we abstract you from the infrastructure and virtualization used for the deployment, we provide options for a *hardware generation*.

For Azure SQL Managed Instance, you have three choices:

*   The Standard series is called Gen5 (it used to be Gen4 as we continually improve our generation of hardware). It has great choices for the number of vCores and memory per vCore.

*   The premium-series uses a newer generation of processors and offers more vCores and a higher memory per vCore.

*   The memory-optimized premium-series is like the premium-series but offers a higher memory per vCore. This option for General Purpose doesn’t support as many vCores as the premium-series.

Keep up with all the details at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits?view=azuresql#hardware-configuration-characteristics`](https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits%253Fview%253Dazuresql%2523hardware-configuration-characteristics).

For Azure SQL Database, you have the following options:

*   The Standard series is also called Gen5 (it used to be Gen4 as we continually improve our generation of hardware). It has great choices for the number of vCores and memory per vCore.

*   The Fsv2-series has the fastest CPU speeds, but you cannot provision as many vCores as you can for Gen5\. This choice is only available for the General Purpose service tier.

*   The DC-series only supports 8 vCores but is critical for security as it supports the Intel Software Guard Extension (Intel SGX) for secure enclaves used with Always Encrypted.

*   Hyperscale supports premium and premium memory-optimized choices for more vCores and a higher memory to vCore ratio.

Keep up with all the details for these choices at [`https://learn.microsoft.com/azure/azure-sql/database/service-tiers-sql-database-vcore?view=azuresql&tabs=azure-portal#hardware-configuration`](https://learn.microsoft.com/azure/azure-sql/database/service-tiers-sql-database-vcore%253Fview%253Dazuresql%2526tabs%253Dazure-portal%2523hardware-configuration).

#### vCore and Storage Sizes

Once you have figured these options, you have choices on storage size. The DTU mode for Azure SQL Database has a DTU number you can choose (and a data size). For the vCore purchasing model, you have both number of vCores and database storage max size to select (remember you don’t need this for Hyperscale). There are a few differences in how these options work depending on your other choices. I will describe these differences as I walk you through the deployment process in the rest of this chapter.

#### Price

Just like with Azure Virtual Machine, take advantage of the Azure Pricing Calculator to plug in some of these choices to get an idea of your costs. This includes using Azure Hybrid Benefit. You can find the pricing calculator for Azure SQL Managed Instance at [`https://azure.microsoft.com/pricing/details/azure-sql/sql-managed-instance/single/`](https://azure.microsoft.com/pricing/details/azure-sql/sql-managed-instance/single/) and Azure SQL Database at [`https://azure.microsoft.com/pricing/calculator/?service=sql-database`](https://azure.microsoft.com/pricing/calculator/%253Fservice%253Dsql-database).

Tip

To make sure to find Azure SQL Database and Managed Instance, search for Azure SQL Database. From there, you can change type for Managed Instance.

Figure [4-3](#496204_2_En_4_Chapter.xhtml#Fig3) shows an example of the pricing calculator for Azure SQL Database.

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig3_HTML.jpg)

Figure 4-3

The pricing calculator for Azure SQL Database

### Consider Resource Limits

Your choices from deployment options, service tier, and sizes can affect your resource limits. These are the following resource limits to consider as you make these choices. I call these out because these may not be obvious as you deploy through the Azure Portal:

*   Max memory

*   Max log size

*   Log rate governance

*   IOPS and I/O latency

*   Max size of tempdb

*   Max concurrent workers

*   Backup retention

I will discuss more about log rate governance and IOPS and I/O latency in Chapter [7](#496204_2_En_7_Chapter.xhtml) of this book. For now, keep these concepts in mind as they can affect the performance of applications such as those that are heavy transaction log workloads.

To see the specific limits for Azure SQL Managed Instance, please see these well-documented tables at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits`](https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits). To see the specific limits for Azure SQL Database, please see the table at [`https://learn.microsoft.com/azure/azure-sql/database/resource-limits-vcore-single-databases`](https://learn.microsoft.com/azure/azure-sql/database/resource-limits-vcore-single-databases).

You should also know that there are overall Azure SQL limits per subscription per region. You can view what these limits are at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits`](https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits) and [`https://learn.microsoft.com/azure/azure-sql/database/resource-limits-logical-server`](https://learn.microsoft.com/azure/azure-sql/database/resource-limits-logical-server).

It is possible to make a request to Microsoft to increase your subscription limits. This is called a *quota increase request*. Read more at [`https://learn.microsoft.com/azure/azure-sql/database/quota-increase-request`](https://learn.microsoft.com/azure/azure-sql/database/quota-increase-request).

## Deploying Azure SQL Managed Instance

Similar to the process I documented in Chapter [3](#496204_2_En_3_Chapter.xhtml) for a virtual machine, deploying an Azure SQL Managed Instance through the Azure Portal starts by using the Azure SQL option from the Azure Marketplace (I showed you this view in Figure 3-1).

Using the three Azure SQL choices, you would select SQL Managed Instance and single instance, then click **Create**.

### Walking Through a Deployment

Let us walk through each screen with deployment options of a Managed Instance and then deploy it through the Azure Portal. I’m going to use the new NextGen General Purpose service tier as an example to walk through the deployment experience. You will see a familiar but slightly different experience as you saw in Chapter [3](#496204_2_En_3_Chapter.xhtml) for SQL Server on Azure Virtual Machine.

As I mentioned earlier in the chapter about pre-deployment planning, I highly recommend you take the time to review all the options I’m going to show you. This is because some of the choices you will make can only be made at deployment and cannot be modified (which would require you to migrate afterward). In addition, some choices can be modified (like service tier) but may require some downtime.

#### Basics

Figure [4-4](#496204_2_En_4_Chapter.xhtml#Fig4) shows the fields you complete on the top of the Basics screen, including choosing Subscription, Resource Group, Instance Name (this becomes part of the server name and @@SERVERNAME of the instance), Region, Pool, and Compute+Storage.

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig4_HTML.jpg)

Figure 4-4

Basics top of screen to deploy Azure SQL Managed Instance

I like the fact that the estimated cost is listed on the right side as I make all my choices. This is a common experience we provide for Azure SQL.

Notice the option called **Compute + Storage**. This is where you will choose Service Tier and Size (vCores + Storage). Notice for me the default is General Purpose with 8 vCores and 256GB max storage. Click **Configure Managed Instance** to see what your options are, which should look like Figure [4-5](#496204_2_En_4_Chapter.xhtml#Fig5).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig5_HTML.jpg)

Figure 4-5

Top of screen for Compute+Storage for Azure SQL Managed Instance

This is an important screen to make choices for your deployment, so let’s look at these options.

At the top of this screen, you can see you can choose General Purpose (GP), the default, or Business Critical (BC), and some of the resource limits and performance expectations are listed with each. From pre-deployment planning, remember there are other significant reasons to choose BC, including

*   Access to the In-Memory OLTP feature

*   Higher availability because BC uses local storage and a replica architecture

Next, you will notice I have enabled the NexGen GP option. As this feature becomes generally available, I expect this to be the default GP behavior.

Now I can choose the hardware generation which I’ve described these choices earlier in the chapter.

Now I can choose my “size” options, including vCores, max storage, and IOPS (this is one of the new features of NextGen).

Your hardware generation choice dictates your possible max vCore values which can be up to 128.

The maximum storage value is called the *maximum instance size* and is the maximum size allowed for **all** database and transaction log files associated with databases for the managed instance. You should think of this as the maximum storage drive for your databases. The maximum storage size is different depending on your vCore choice and choice of GP and BC. GP allows for up to 32TB, and BC allows for up to 16TB.

Note

I discussed the architecture of Business Critical deployments in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book and will elaborate more on this architecture in Chapter [8](#496204_2_En_8_Chapter.xhtml). The reason for the lower storage limit for BC is the fact that databases are stored on local SSD drives vs. Azure Storage.

Tempdb max sizes are dependent on vCore selections but are counted toward the overall maximum storage limit. In fact, all databases, including system databases, count toward the overall max instance storage size.

You can run the following query after you deploy your managed instance (in any database context) to see how much space your databases are taking up in relation to the max storage for the instance:

```
SELECT TOP 1 used_storage_gb = storage_space_used_mb/1024,
max_storage_size_gb = reserved_storage_mb/1024
FROM sys.server_resource_stats ORDER BY start_time DESC;
```

Scrolling down, you can see the rest of the options to choose from in Figure [4-6](#496204_2_En_4_Chapter.xhtml#Fig6).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig6_HTML.jpg)

Figure 4-6

Other choices for Compute+Storage for Azure SQL Managed Instance

Your first choice is zone redundancy. That option is disabled during the preview of NextGen GP but will be available once this is generally available. Zone redundancy uses Availability Zones to make your deployment more highly available. I highly recommend you consider this option for your deployments. I will discuss this concept more in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book.

You also have a choice to save money using your existing SQL Server licenses with Azure Hybrid Benefit (AHB). I’ll pause and say the Hybrid failover rights apply to a Managed Instance that is used for disaster recovery. Learn more in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book.

You also have choices for the redundancy of your backups. Unless you have specific data residency requirements, I recommend geo-redundant backups which replicate your backups in Azure Storage to known paired regions. I’ll discuss more about backup redundancy and costs in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book.

Click **Apply** after you make any changes (Apply is only enabled if you change the defaults; you can click the X in the upper right-hand corner to get back to the Basics screen).

Tip

Hitting Apply will not deploy the Managed Instance yet, but take your time to get your choices here as close to correct as possible. Why? You can change them later, but Managed Instance changes to tiers and sizes can be a long operation.

We still have one more choice on the Basics screen as you can see in Figure [4-7](#496204_2_En_4_Chapter.xhtml#Fig7).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig7_HTML.jpg)

Figure 4-7

Authentication choices for Azure SQL Managed Instance during deployment

This option allows you to choose Microsoft Entra authentication only, mixed, or just SQL Authentication. If you choose any Entra option, you must specify an Entra account. These choices become a member of the sysadmin role for the instance. If you have never used Microsoft Entra authentication, it will require some learning for you. It is very similar to using Windows Authentication on SQL Server, but there are several differences. This capability is the most secure method and even supports passwordless applications.

Setting this up though for Azure SQL Managed Instance has one possible roadblock (which is why I didn’t use it here). You must have special permissions in your Microsoft Entra system to use this option (I don’t at Microsoft by default have this permission so have a special process to request it). Read more at [`https://learn.microsoft.com/azure/azure-sql/database/authentication-aad-configure?view=azuresql&tabs=azure-powershell#provision-microsoft-entra-admin-sql-managed-instance`](https://learn.microsoft.com/azure/azure-sql/database/authentication-aad-configure%253Fview%253Dazuresql%2526tabs%253Dazure-powershell%2523provision-microsoft-entra-admin-sql-managed-instance).

#### Networking

Click **Next: Networking >** to review your networking choices. Your screen should look like Figure [4-8](#496204_2_En_4_Chapter.xhtml#Fig8).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig8_HTML.jpg)

Figure 4-8

Azure SQL Managed Instance Networking Options

##### Virtual Network

You can see at the top of the screen that a new virtual network will be created to host the Azure SQL Managed Instance. One of the advantages of Managed Instance is that it is deployed in a private virtual network. You could deploy your own Azure virtual network first (you can use the Azure portal or CLI) and select that virtual network on this deployment screen. If you choose to use your own virtual network, you must configure it a specific way, which you can read at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/vnet-existing-add-subnet`](https://learn.microsoft.com/azure/azure-sql/managed-instance/vnet-existing-add-subnet)`.`

Tip

Did you know you can deploy multiple instances in the same virtual network? Doing this will speed up subsequent deployments, but make sure this meets your security and networking needs.

##### Connection Type

Notice on my screen I have chosen a connection type of *Redirect*. The default is *proxy*. A proxy connection requires that any connection to the Managed Instance by a tool or application (a connection to the TDS port 1433) must always go through a *gateway*. A redirect connection type uses the gateway to find the direct virtual private IP address of the node containing the Managed Instance. All subsequent traffic flows directly to the node. You can read more about these connection types at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/connection-types-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/connection-types-overview). Proxy can be more secure, but redirect can be faster. The virtual network and included subnet will have all the appropriate Network Security Group (NSG) rules applied for these types if you choose to create the virtual network as part of this deployment step.

##### Public Endpoint

You have the option to enable TDS traffic on a public endpoint. The public endpoint will be enabled on port 3342 (and get redirected in the virtual network to the node instance port 1433). While I do not recommend using this option for production, it is one of the quickest ways to get connected to a Managed Instance. You can read more about the public endpoint for Managed Instance at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/public-endpoint-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/public-endpoint-overview).

You also can control the minimum TLS connection encryption for the instance. I would always recommend using the latest TLS version. The only reason you may want to use a different one is for older applications where that version is not supported. Learn more at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/minimal-tls-version-configure`](https://learn.microsoft.com/azure/azure-sql/managed-instance/minimal-tls-version-configure).

Notice that accelerated networking is automatically enabled for the Managed Instance.

#### Security

Click **Next: Security >** to make further security selections as seen in Figure [4-9](#496204_2_En_4_Chapter.xhtml#Fig9).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig9_HTML.jpg)

Figure 4-9

Security choices for Azure SQL Managed Instance

I’ve mentioned **Microsoft Defender** in previous chapters, and the same capabilities (but customized for Managed Instance) are available for vulnerability assessments and advanced threat protection.

You can set up your own user-defined **managed identity** for *passwordless* applications. By default, you get a system-defined managed identity.

If you want to use **Windows Authentication** in your applications against Azure SQL Managed Instance, you can enable an option for a Service Principal. If you want to avoid changing our application to use Microsoft Entra authentication, consider this option. Learn more at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/winauth-azuread-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/winauth-azuread-overview).

**Transparent Data Encryption** (TDE), which is encryption at rest for the storage of your databases and transaction logs, is enabled by default (you need to enable this for each individual database). This option allows you to set up your own key for encryption.

#### Additional Settings

Click **Next: Additional settings >** to enable a few additional options for the deployment as seen in Figure [4-10](#496204_2_En_4_Chapter.xhtml#Fig10).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig10_HTML.jpg)

Figure 4-10

Additional settings for Managed Instance

The **collation** here is like setting a collation for a SQL Server. It is important to know that you cannot change the instance collation after it is supplied here.

Note

You *can* set and change *database* collations on Managed Instance after deployment. Just keep in mind having different database collations can cause issues for cross-database queries or temporary table operations.

The **time zone** is the recognized time zone by the SQL Server engine on the Managed Instance node. I have changed this to my local time zone, but it can be UTC or whatever time zone you want to choose. You cannot change this after deployment.

The Managed Instance can be part of a **failover group** which we will talk more about in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book. You will use that option when you review those topics later. For now, leave it to the default of No.

**Maintenance window** allows you to define a different timeframe when scheduled updates must be applied to your deployment. The default is any data between 5pm and 8am based on your time zone, or you can configure this for 10pm to 6am either Monday–Thursday or Friday–Sunday. You can change the maintenance window after you deploy.

**Update policy** is a relatively new concept. You can choose to keep your Managed Instance at a version compatible with SQL Server 2022\. I’ll show you in Chapter [8](#496204_2_En_8_Chapter.xhtml) how you can take advantage of this for an online disaster recovery capability. However, if you choose this option, you will not be able to get new features that typically come from a versionless deployment (but you will have Cumulative Updates automatically applied). The first option ensures you are always up to date, but you will not be able to restore a backup or participate in an online disaster recovery scenario with SQL Server 2022.

I will choose the SQL Server 2022 update policy, so I can show you these new compatibility features later in the book.

#### Tags

Click **Next: Tags >** to define a tag like how I described in Chapter [3](#496204_2_En_3_Chapter.xhtml) with Azure Virtual Machines.

#### Deploy!

Click **Next: Review + create >** to view the final screen before deploying. Like with the deployment for Azure Virtual Machine, this screen shows estimated costs, terms of use, privacy policy, a review of all the options you have chosen, and the option to download an Azure template that describes these deployment options.

You will also see a warning that based on my choices the deployment could take as much as 6 hours to complete. For my deployment, it was completed in about 1 hour and 45 minutes. If you want to track more information during your deployment, you can use the Azure Portal ([`https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-monitor`](https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-monitor)).

So, click **Create** to deploy, leave this screen open to see the progress, and read on to the next section about deploying with a CLI and then some architecture and implementation details you might find interesting why the deployment runs.

### Deploying with a CLI

An Azure Managed Instance can be deployed with command-line interfaces (CLI) through the **az sql mi** ([`https://learn.microsoft.com/cli/azure/sql/mi`](https://learn.microsoft.com/cli/azure/sql/mi)) command interfaces or through **New-AzSQLInstance** PowerShell cmdlet ([`https://learn.microsoft.com/powershell/module/az.sql/new-azsqlinstance`](https://learn.microsoft.com/powershell/module/az.sql/new-azsqlinstance)).

I went down the path to build an example with az sql mi and found that I needed to run several az CLI commands to create the virtual network, subnet, and all associated settings. Therefore, I only recommend you use the az sql mi CLI with *Azure templates*. An example template can be found at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/create-template-quickstart?view=azuresql&tabs=azure-cli`](https://learn.microsoft.com/azure/azure-sql/managed-instance/create-template-quickstart%253Fview%253Dazuresql%2526tabs%253Dazure-cli).

Tip

Just like I showed you how to use Copilot in Azure to create a database using the az CLI, it can also give you tips and examples on how to do this for Azure SQL Managed Instance. It also offers options to help you configure a virtual network and subnets.

PowerShell requires you to set up the virtual network and other context before executing New-AzSQLInstance. There is a good tutorial on using PowerShell at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/scripts/create-configure-managed-instance-powershell`](https://learn.microsoft.com/azure/azure-sql/managed-instance/scripts/create-configure-managed-instance-powershell).

As I mentioned earlier in this chapter, you have options to deploy an Azure SQL Managed Instance with both Bicep and Terraform.

### Implementation Details

Note

These implementation details may change over time as we change and improve the service. I offer up some of these details, so you can understand how we build, manage, and run the service.

Azure SQL Managed Instances are deployed on nodes (virtual machines) powered by Azure Service Fabric in a concept called a *ring* or *virtual cluster*. A virtual cluster is a dedicated set of isolated virtual machines that run in a virtual network subnet.

When you deploy your first managed instance in a new virtual network (subnet) as I did in my example, you are in fact deploying an entire virtual cluster. This explains why the initial deployment can take so long to complete. You can deploy other managed instances in the *same virtual network subnet*, and the deployment is slightly faster. Read more about specific expected times at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-overview).

A managed instance is a full SQL Server engine database instance deployed in a dedicated virtual machine in the virtual cluster. Microsoft will decide how to deploy these virtual machines on various nodes of the cluster. It is possible that a node may have one virtual machine with an instance or multiple virtual machines. As per the promise of Platform as a Service (PaaS), Microsoft abstracts you from those details. Your interface with the Managed Instance is through either standard SQL Server interfaces such as T-SQL or Azure interfaces such as the portal, CLI, or REST API. You will never directly access the underlying virtual machines.

This architecture also explains why certain management operations such as scaling vCores can also take a long period of time as some of these operations require a deployment of a new virtual cluster with either attaching files from Azure Storage or reseeding a replica.

I described some of the architecture of General Purpose (GP) vs. Business Critical (BC) tiers in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book. I will describe them further in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book. Either of these service tiers use the same virtual cluster architecture just with different storage and HA implementations.

**Resource limits** for a Managed Instance such as memory limits, max storage size, and others are enforced through several mechanisms depending on your choices for deployment. For example, memory limits are enforced with Windows Job Objects (and you cannot configure “max server memory”). You can read more about *Windows Job Objects* at [`https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects`](https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects). Storage capacity (or max size) is enforced by the File Server Resource Manager (FSRM), which you can read more about at [`https://learn.microsoft.com/windows-server/storage/fsrm/fsrm-overview`](https://learn.microsoft.com/windows-server/storage/fsrm/fsrm-overview).

A *Managed Instance pool deployment* can be far faster because an instance pool can be a set of SQL Server instances running in the same virtual machine. Isolation and resource limits are applied using Windows Job Objects. Figure [4-11](#496204_2_En_4_Chapter.xhtml#Fig11) represents a visual comparing managed instance and pools. This figure comes from the documentation at [`https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/instance-pools-overview?view=azuresql#architecture`](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/instance-pools-overview%253Fview%253Dazuresql%2523architecture).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig11_HTML.jpg)

Figure 4-11

The Managed Instance and pool architecture

### Connecting and Verifying Deployment

Once your deployment is complete, your screen should look like Figure [4-12](#496204_2_En_4_Chapter.xhtml#Fig12).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig12_HTML.jpg)

Figure 4-12

Deployment complete for Managed Instance

Let’s look at a few details on this screen:

1.  If you want a summary of the deployment, you can select Operation Details, which among other things can show you the duration of the deployment.

2.  You can get more details about each step of the deployment by selecting More events in the Activity Log.

3.  You can select Go to Resource to see your completed deployment in the Azure Portal.

If you click **Go to Resource**, you will see a screen like in Figure [4-13](#496204_2_En_4_Chapter.xhtml#Fig13).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig13_HTML.jpg)

Figure 4-13

The Overview screen for Azure SQL Managed Instance

One of the first steps you will want to take at this point is try and connect to the new instance and verify the deployment. I like to verify my SQL Server installations with a set of simple T-SQL queries.

#### Connect to a Managed Instance

There are several different methods to connect to your Managed Instance, and you can see all of them at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/connect-application-instance`](https://learn.microsoft.com/azure/azure-sql/managed-instance/connect-application-instance).

What I personally like to do is connect with an Azure Virtual Machine *inside* the virtual network that was created as part of the deployment, so I don’t have to enable the public endpoint. I call this virtual machine a ***jumpbox***. I’ll use RDP to connect to the VM, but in the VM, I use tools like SSMS to connect to the private network for the instance.

I’ll create a new Azure Virtual Machine (I just use a SQL Server Developer Edition from the marketplace) in the same resource group as my Managed Instance. During deployment of the VM, I’ll choose the virtual network created for the Managed Instance (usually vnet-<MI name>, so the name was vnet-bwsqlnextgeni). On this screen, I choose to manage subnet configuration. This allows me to create a new subnet called default (and use the default choices). This becomes the private subnet for applications like my VM to connect to the virtual network of the Managed Instance.

My network settings look like Figure [4-14](#496204_2_En_4_Chapter.xhtml#Fig14).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig14_HTML.jpg)

Figure 4-14

Networking settings for VM jumpbox to Managed Instance

With my jumpbox now deployed, I will use SSMS inside the VM to connect to the Managed Instance. When you connect to a Managed Instance, the server’s name becomes a URL like this (which you can find on the Overview screen in the portal for the Managed Instance).

##### bwsqlminextgen.b243ea7f888c.database.windows.net

Figure [4-15](#496204_2_En_4_Chapter.xhtml#Fig15) shows an example of SSMS connected to an Azure SQL Managed Instance.

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig15_HTML.jpg)

Figure 4-15

SSMS connected to Azure SQL Managed Instance through a jumpbox

#### Verify the Deployment

Notice that Object Explorer in SSMS looks almost identical to a SQL Server except for the Fully Qualified Domain Name (FQDN) for the server name.

To verify the installation of SQL Server, I often use a few techniques, including running queries against system catalog views and DMVs, and look at the ERRORLOG.

##### Examining the ERRORLOG

You do not have access to the file system for the virtual machine hosting the Managed Instance. To look at the ERRORLOG, we will need a tool like SSMS or T-SQL.

You can use Object Explorer in SSMS to view the ERRORLOG, but I prefer just T-SQL, so I can execute `sp_readerrorlog` to look at the current ERRORLOG file. I must warn you that we dump all types of extra information in the ERRORLOG for a Managed Instance (yes, even more than a SQL Server), and it can take several seconds to execute. My colleague Dimitri Furman wrote a blog post with some sample code to filter down the ERRORLOG for a Managed Instance. You can view this at [`https://techcommunity.microsoft.com/t5/datacat/azure-sql-db-managed-instance-sp-readmierrorlog/ba-p/305506`](https://techcommunity.microsoft.com/t5/datacat/azure-sql-db-managed-instance-sp-readmierrorlog/ba-p/305506).

There are a few key messages I look at startup in the ERRORLOG and found these in my Managed Instance:

```
SQL Server detected 1 sockets with 4 cores per socket and 8 logical processors per socket, 8 total logical processors; using 8 logical processors based on SQL Server licensing. This is an informational message; no user action is required.
SQL Server is starting at normal priority base (=7). This is an informational message only. No user action is required.
Detected 44645 MB of RAM. This is an informational message; no user action is required.
```

I can see that eight logical processors were detected, which is what I expect given I deployed an 8-vCore instance.

The memory detected is how much memory the SQL Server engine detects from the host or VM. You will see in several places in the book how Azure will use Windows Job Objects to limit the memory visible to SQL Server to enforce resource limits per service tier and vCores. You will find for this Managed Instance the job object will not allow SQL Server to access all the memory as shown here in the ERRORLOG. In fact, you should never rely on what the ERRORLOG shows but instead on the DMV sys.dm_os_job_object, which I will show you how to use in the next section.

##### Verification Queries

Whenever I install a SQL Server, I typically use a few T-SQL queries as a *sanity check*. Let us look at these and the results from the Managed Instance compared to SQL Server:

```
SELECT @@version
Microsoft SQL Azure (RTM) - 12.0.2000.8   May 28 2024 14:55:16   Copyright (C) 2022 Microsoft Corporation
```

I explained in Chapter [1](#496204_2_En_1_Chapter.xhtml) of the book describing the history of Azure SQL why v12 was a monumental moment for the service. Since that time, we have not changed the major version of 12.

Basically, the major version of Azure SQL Managed Instance has no meaning. It does not line up with any major version of SQL Server since Azure SQL Managed Instance is *versionless*. I will discuss more the concept of versionless in Chapter [5](#496204_2_En_5_Chapter.xhtml) of the book. Just know that Microsoft strives to keep instances and databases of Azure SQL up to date with all the right changes and fixes, so you do not worry about applying updates.

Note

Even though the version doesn’t change, the timestamp of @@VERSION is when we have last updated the instance. It is not a guarantee, but I’ve never seen this not be the case.

```
SELECT database_id, name, compatibility_level FROM sys.databases;
database_id    name        compatibility_level
1              master      160
2              tempdb      160
3              model       160
4              msdb        160
```

This looks normal (I have not created any user databases) except the mssqlsystemresource database is not listed as with SQL Server (it does exist as you can see in the ERRORLOG). Notice the compatibility level is set to 160 which is the dbcompat for SQL Server 2022.

```
SELECT name, object_id, type_desc FROM sys.objects;
```

The results from this query are what you would normally think from a master database on a SQL Server with system tables at the top of the list.

```
SELECT * FROM sys.dm_os_schedulers;
```

Since we have deployed an 8-vCore Managed Instance, I would expect eight VISIBLE ONLINE schedulers, which is the case. I also expect to see a few HIDDEN ONLINE schedulers which are in the result of this query.

```
SELECT * FROM sys.dm_os_sys_info;
```

This is a DMV that provides system information about SQL Server. I can see from the output of this DMV the number of CPUs detected, the amount of memory detected, the target (max server memory), the total memory used by the engine, and the actual worker thread max (1640 for 8 vCores) and that the conventional memory model is used. I will talk more about locked pages in Chapter [5](#496204_2_En_5_Chapter.xhtml) of the book.

```
SELECT * FROM sys.dm_os_process_memory;
```

This DMV shows OS-related memory information, including whether locked or large pages are used (they are not for Azure SQL Managed Instance or Database). I normally just use this as a sanity check that enough memory is available for SQL Server in a VM or server.

```
SELECT * FROM sys.dm_exec_requests;
```

This is one of the most common DMVs in the world to check the state of what is running on a SQL Server. I run this just to make sure all the normal background processes are running, including LAZY WRITER, RECOVERY WRITER, LOCK MONITOR, etc., and that I can see active queries.

```
SELECT SERVERPROPERTY('EngineEdition');
```

Per the documentation at [`https://learn.microsoft.com/sql/t-sql/functions/serverproperty-transact-sql`](https://learn.microsoft.com/sql/t-sql/functions/serverproperty-transact-sql), the value of eight is for a Managed Instance.

There are also two new DMVs specific to Azure not found in SQL Server:

```
SELECT * FROM sys.dm_user_db_resource_governance;
```

This DMV is really intended to show you resource limits for a specific Azure SQL Database, but it also works for Managed Instance (you will see a row for all databases, including system databases except for tempdb). You can view limits like memory, max storage, log rates, etc. You can read the documentation for this DMV at [`https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-user-db-resource-governor-azure-sql-database`](https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-user-db-resource-governor-azure-sql-database). Note that the docs say this is mostly for internal use, which means it might change in the future.

Note

There is an undocumented DMV called **sys.dm_instance_resource_governance** which shows resource limits at the instance level.

```
SELECT * FROM sys.dm_os_job_object;
```

This is a DMV specific to Azure that shows resource limits Azure applies to a Managed Instance using Windows Job Objects. The specific column I look at is **memory_limit_mb** which shows me the true amount of memory the Managed Instance has access to. I talked about Windows Job Objects in the section “Implementation Details.”

You may be asking at this point what is so special about Managed Instance since from the perspective of using a tool like SSMS it feels like a SQL Server running in an Azure Virtual Machine. This is the point of a Managed Instance. We want you to have the feel of a SQL Server instance, but not worry about the details you might have to consider with a virtual machine. And since Azure SQL Managed Instance (MI) is a PaaS service, you will see the benefits of using MI especially when it comes to a versionless SQL Server, predictable performance, built-in high availability, and disaster recovery.

## Migrating to Azure SQL Managed Instance

I discussed earlier in this chapter some of the considerations and tools for migrating to Azure SQL Managed Instance.

Let me add more details on some considerations should you consider when migrating:

*   When you are migrating a database protected by Transparent Data Encryption to a managed instance using native restore option, the corresponding certificate from the on-premises or Azure VM SQL Server needs to be migrated *before* database restore.

*   Restore of system databases is not supported. **To migrate instance-level objects** (stored in master or msdb databases), we recommend scripting them out and run T-SQL scripts on the destination instance. For example, you can use SSMS to script out SQL Server Agent jobs to T-SQL.

*   Database compatibility is fully supported by Azure SQL Managed Instance, including all current supported versions of SQL Server. I have many customers that choose to keep their current dbcompat when migrating to Azure and then test and update it later. Read more about dbcompat at [`https://aka.ms/dbcompat`](https://aka.ms/dbcompat).

*   While you can always restore a full backup (offline) to Azure SQL Managed Instance, you have online choices as well, including a log-shipping approach with **Azure Database Migration Service** (DMS) and a more granular approach using **Managed Instance Link** using Always On Availability Groups. Managed Instance link works even if you don’t have an AG deployed on-premises. Learn more about how to use Managed Instance link at [`https://aka.ms/milink`](https://aka.ms/milink).

*   Application changes should be minimal for a migration to Managed Instance except using the new server name (which you saw in the previous part of this chapter on how to connect) and possibly the authentication method. A new capability to use Windows Authentication and mitigate changes you need in your application for authentication is now available. Learn more at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/winauth-azuread-setup`](https://learn.microsoft.com/azure/azure-sql/managed-instance/winauth-azuread-setup).

## Deploying an Azure SQL Database

Deploying an Azure SQL Database is both different and like Azure SQL Managed Instance. The experience is similar as you will use the Azure Portal or CLI (or even T-SQL) to deploy a database but different since you are…well deploying a database, not an instance of SQL Server. Since you are deploying just a database, you get more options as I have described in this book, including Serverless, elastic pools, and Hyperscale.

The basic process to deploy Azure SQL Database is

*   ✓ Decide to deploy a single database or elastic pool

*   ✓ Choose a resource group and region

*   ✓ Choose an existing or new logical database server

*   ✓ Choose your purchasing model, compute model, service tier, hardware configuration, and sizes

*   ✓ Optionally supply other configuration choices

*   ✓ Deploy it!

### Deployment and Options

To get started, I will use the **Azure SQL** screen to deploy by searching for Azure SQL in the marketplace as I showed you in Chapter [3](#496204_2_En_3_Chapter.xhtml) with SQL Server on Azure Virtual Machine but the single database on the left-hand side and click **Create**. I will show you how to deploy a single database which will allow me to also deploy a database server (and I will describe what and why you need a database server).

As we did for Azure SQL Managed Instance, let’s walk through all the options to deploy the database through the Azure Portal.

#### Basics

If you choose a single database, you are presented with a Basics screen like Azure SQL Managed Instance as seen in Figure [4-16](#496204_2_En_4_Chapter.xhtml#Fig16).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig16_HTML.jpg)

Figure 4-16

Basics screen to deploy an Azure SQL Database

You can see from my screen that I have already created a new resource group, defined a database name, and have the option to first create a new database (or logical) server.

Note

Notice the option at the top of this screen to choose the free database offer at [`https://aka.ms/freedboffer`](https://aka.ms/freedboffer).

You may be wondering why you need a database server when the promise of Azure SQL Database is “you own the database; Azure will manage everything else.”

A **database server**, also known as a *logical server*, is a collection of metadata stored in the Azure infrastructure used to organize one more Azure SQL Databases. It is *not* a single SQL Server instance on a physical server. A database server contains a **logical master database** just like a true SQL Server instance. Notice the region is associated with the logical server, not the database. Any database created for the logical server will be hosted in the region of the server. All connectivity and networking will be associated with the logical server. In fact, you could create a logical server first, connect to that server, and use T-SQL CREATE DATABASE to create Azure SQL Databases. The login and password you supply for the logical server becomes a login in the logical master database, which is a server-level principal who is effectively a *server admin* for all databases.

You can see my choices in Figure [4-17](#496204_2_En_4_Chapter.xhtml#Fig17) to create the logical server, including your region and preferred authentication method. In this case, I’m going to set up both Microsoft Entra and SQL Authentication and supply an admin for each.

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig17_HTML.jpg)

Figure 4-17

Creating a logical server for Azure SQL Database

Once you click **OK** for the logical server (it will be created as part of the deployment), you can choose if you want to make this part of an elastic pool, but I won’t do that here. You can create a logical server on its own first and then create databases for it. In this scenario, I’m creating a new database along with a logical server and have the database placed with that new server.

Now I have more choices to complete the Basics screen as seen in Figure [4-18](#496204_2_En_4_Chapter.xhtml#Fig18).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig18_HTML.jpg)

Figure 4-18

Completing Basics screen choices for Azure SQL Database

The Workload environment is a convenient method to choose a lower service tier and vCore size for Development or choose the Production option which will default to Hyperscale. I’ll choose Production.

You then have a choice to configure several aspects of your deployment, including service tier and other resource choices. You can select **Configure database** to make these choices. Before I show you those options, you also have a choice on backup storage redundancy options. This is like the choice for Azure SQL Managed Instance. Unless you have data residency requirements, I recommend you pick a geo-redundant option. Geo-zone redundancy has the highest level of redundancy and cost, but a zone redundant option is required if you choose zone redundancy as a deployment option. In fact, I’ll choose a zone redundant option for the deployment, and this option will change for my backup option.

Figure [4-19](#496204_2_En_4_Chapter.xhtml#Fig19) shows the options when you select Configure database.

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig19_HTML.jpg)

Figure 4-19

Compute and storage options for Azure SQL Database

Let’s go through each of these options.

##### Service Tier

I mentioned earlier your choices of General Purpose (GP), Business Critical (BC), and Hyperscale.

##### Compute Tier

For GP and Hyperscale, you can choose Serverless or Provisioned. I’ll show you more about Serverless later in the chapter.

##### Hardware Configuration

This is where you can choose Gen5, FSv2, DC, or Premium Hardware choices.

##### vCores

Based on your service tier and hardware configuration, you can choose how many vCores you need (which you can scale up or down later). You can have as many as 128 vCores.

##### HA Replicas and Zone Redundancy

Hyperscale allows up to 30 named replicas, so you can choose your number of replicas here. I’ll cover more about Hyperscale Availability in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book. You don’t have to have any replicas for Hyperscale, but if you choose zone redundancy, you need to have at least one replica.

##### Other Choices

*   If you have selected GP or BC tiers, you need to choose a max storage size. You can have up to 4TB for GP and for BC. For Hyperscale, there is no maximum storage, so you don’t need to make a choice.

*   For GP and BC service tiers, you can choose Azure Hybrid Benefit for licensing. Hyperscale’s new pricing structure doesn’t need this choice.

I realize there are so many choices here; it is hard to keep up. I created this slide for a presentation I did that you might find useful as seen in Figure [4-20](#496204_2_En_4_Chapter.xhtml#Fig20).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig20_HTML.jpg)

Figure 4-20

Azure SQL Database deployment choices

#### Networking

Click **Next: Networking >** to see your choices for connectivity and network security as seen in Figure [4-21](#496204_2_En_4_Chapter.xhtml#Fig21).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig21_HTML.jpg)

Figure 4-21

Networking choices when deploying an Azure SQL Database

Unlike Azure SQL Managed Instance, Azure SQL Database is not part of a private virtual network. You have three choices during deployment:

*   **No access** – Deploy the database but do not allow any connectivity until you are ready to make your choice.

*   **Public endpoint** – Expose connectivity of the logical server and/or database to the public within Azure or to the Internet (or both).

*   **Private endpoint** – This is a new addition to Azure SQL Database to make it very secure. This allows you to deny public access to your server and/or database and only allow private connectivity within defined virtual networks in and outside of Azure.

For now, I will select the public endpoint and set **Allow Azure services and resources access this server** to Yes. This allows me to deploy an Azure Virtual Machine (no matter what virtual network) and connect to this database or to connect with SQL client tools on the client computer where I am currently deploying the browser. I will show you how to tighten up the security of this model in Chapter [6](#496204_2_En_6_Chapter.xhtml) of the book.

I also have choices here like an Azure SQL Managed Instance for connection policy (Proxy or Redirect) and a minimum TLS version.

#### Security

Click **Next: Security >** to set several security-related options as you can see in Figure [4-22](#496204_2_En_4_Chapter.xhtml#Fig22).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig22_HTML.jpg)

Figure 4-22

Security choices for Azure SQL Database

Like Azure SQL Managed Instance, you can enable **Microsoft Defender** for SQL and get capabilities like Vulnerability Assessments and Advanced Thread Protection.

**Ledger** is a feature that provides tamper-evident record of changes in the database, like a blockchain inside SQL. You will learn more about Ledger in Chapter [10](#496204_2_En_10_Chapter.xhtml) of the book.

Server identity allows you to configure a **managed identity** for *passwordless* applications using Microsoft Entra.

**Transparent Data Encryption** (TDE), encryption at rest, is on by default for Azure SQL Database. This option gives you the ability to Bring Your Own Key (BYOK).

Your last option is to configure **Secure Enclaves** for Always Encrypted. Always Encrypted is an end-to-end encryption capability for SQL. Learn more about Secure Enclaves at [`https://learn.microsoft.com/sql/relational-databases/security/encryption/always-encrypted-enclaves`](https://learn.microsoft.com/sql/relational-databases/security/encryption/always-encrypted-enclaves).

#### Additional Settings

Click **Next: Additional settings >** to see more options for the deployment. Figure [4-23](#496204_2_En_4_Chapter.xhtml#Fig23) shows these additional options.

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig23_HTML.jpg)

Figure 4-23

Additional settings for the deployment of Azure SQL Database

Your first choice is to either create a blank database or create a database based on a backup of a geo-replicated Azure SQL Database or from the sample AdventureWorksLT (LT stands for light). You can learn more about how to restore from a geo-replicated backup at [`https://learn.microsoft.com/azure/azure-sql/database/recovery-using-backups`](https://learn.microsoft.com/azure/azure-sql/database/recovery-using-backups).

Your next choice is *database collation*. Since I chose a sample database, the collation is already decided. For a new blank database, it is important to choose this during deployment because you cannot change it later.

#### Tags

Click **Next: Tags >** to define an optional tag for the deployment.

#### Deploy It!

Click **Next: Review + create >** to see the final validation screen as you can see from any Azure resource deployment. Just like with Managed Instance, you can see estimated costs, terms of use, privacy policy, a review of your choices, and the ability to download an Azure template.

Click **Create** to see the deployment take off. Just like with Managed Instance if you do not leave this screen, you will see the progress of your deployment in both the notification area of the portal and on your main screen.

In my example, the deployment only took a matter of minutes as shown in Figure [4-24](#496204_2_En_4_Chapter.xhtml#Fig24).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig24_HTML.jpg)

Figure 4-24

A completed deployment of Azure SQL Database

Just like a Managed Instance deployment, you can click More events in the Activity Log to see the sequence of deploying all resources.

Click **Go to resource** to see the Overview screen on the database like Figure [4-25](#496204_2_En_4_Chapter.xhtml#Fig25).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig25_HTML.jpg)

Figure 4-25

The Overview screen of an Azure SQL Database

Just like with a virtual machine and Managed Instance, the portal shows a Service Menu, Command Bar, Working Pane, and Monitoring Pane. While this looks like other Azure resources, most of this information is specific to Azure SQL Database. We will use many of these options throughout the rest of the book as you explore security, performance, availability, and other features. I’ll discuss some specific configuration choices for the database in Chapter [5](#496204_2_En_5_Chapter.xhtml) of the book.

### Deploying Serverless

Let’s first look at how to deploy a Serverless database for Hyperscale by looking at the unique difference on specifying vCores.

On the Overview screen of the database you just deployed, click the server name. This brings you to a screen like Figure [4-26](#496204_2_En_4_Chapter.xhtml#Fig26).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig26_HTML.jpg)

Figure 4-26

Overview of a logical server for Azure SQL Database

Click **+ Create Database** in the Command Menu to create a new database for this logical server.

On the Basics screen for this new database, I’ll use a name of **bwsqldbserverless** and then select Configure database. I’ll then use choices like Figure [4-27](#496204_2_En_4_Chapter.xhtml#Fig27) to choose a Serverless deployment.

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig27_HTML.jpg)

Figure 4-27

Deploying a serverless Hyperscale database

Notice here with Serverless I can now choose a min and max vCore range. My database can scale up and down automatically based on my resource needs. And the key factor for Serverless is that I only pay for the vCores I use per second with the min vCore as the min fixed cost.

All my other options are the same as my first database deployment. Just like I can change the number of fixed vCores for a Provisioned database, I can change the min and max for serverless later. I’ll discuss later about configuring a Serverless database in Chapter [5](#496204_2_En_5_Chapter.xhtml) of the book.

Note

A General Purpose service tier serverless database also has an option for auto-pause, but that is not available at the time of the writing of this book for Hyperscale.

### Deploying an Elastic Pool

Let’s explore also how to deploy an elastic pool of two Hyperscale databases by looking at the different options for deploying the pool and each database in the pool.

Using the same method as I did previously to deploy a serverless database, I’ll create a new database for the same logical server, but as you can see in Figure [4-28](#496204_2_En_4_Chapter.xhtml#Fig28), choose options to put the new database into a new elastic pool.

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig28_HTML.jpg)

Figure 4-28

Creating a database in an elastic pool

You have the option to create the pool on its own first. In this scenario, I created a database and a pool at the same time and placed this new database in the pool.

Now that I have one database in my pool, let’s add another one, but this time one that has a different resource allocation (since this is one of the reasons I’ll use a pool).

On the overview page of the database I just deployed, sqldb1, you will see the elastic pool listed which is called **bwsqldbpool**. Click this and you will see the Overview screen for the elastic pool as you can see in Figure [4-29](#496204_2_En_4_Chapter.xhtml#Fig29).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig29_HTML.jpg)

Figure 4-29

Overview of an elastic pool

Using the Create button in the command window, I can create another database. What you will see in that experience is that you don’t specify a service tier, vCPUs, or other choices. This is because you define these choices at the *pool level*, not the database level.

You can see what databases belong to the pool in the Azure portal. In the same Overview screen as you see in Figure [4-29](#496204_2_En_4_Chapter.xhtml#Fig29), click the databases under Elastic Databases in the Essentials pane, and you will get a screen like Figure [4-30](#496204_2_En_4_Chapter.xhtml#Fig30).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig30_HTML.jpg)

Figure 4-30

Listing databases for an elastic pool

You can also use the catalog view **sys.database_service_objectives** to find which databases belong to a specific elastic pool. I’ll show you how to configure resource selection for elastic pools in Chapter [5](#496204_2_En_5_Chapter.xhtml) of the book.

Here are some references for elastic pools worth reviewing:

*   Read an **overview of elastic pools** at [`https://learn.microsoft.com/azure/azure-sql/database/elastic-pool-overview`](https://learn.microsoft.com/azure/azure-sql/database/elastic-pool-overview).

*   Read about **guidance** to pick the right sizes for the pool at [`https://learn.microsoft.com/azure/azure-sql/database/elastic-pool-overview?view=azuresql#how-do-i-choose-the-correct-pool-size`](https://learn.microsoft.com/azure/azure-sql/database/elastic-pool-overview%253Fview%253Dazuresql%2523how-do-i-choose-the-correct-pool-size).

### Deploying with a CLI

An Azure SQL Database can be deployed with command-line interfaces (CLI) through the **az sql db** ([`https://learn.microsoft.com/cli/azure/sql/db`](https://learn.microsoft.com/cli/azure/sql/db)) command interfaces or through **New-AzSQLDatabase** PowerShell cmdlet ([`https://learn.microsoft.com/powershell/module/az.sql/new-azsqldatabase`](https://learn.microsoft.com/powershell/module/az.sql/new-azsqldatabase)).

Unlike Managed Instance, it is easier to use the az CLI for Azure SQL Database without an Azure template because I only have to create the logical server and then I can create the database (I don’t have to create a virtual network and all the components first).

Tip

It is easy to create a database with a very small az CLI command like the following command:

`az sql db create -n sqldbiseasy -g bwsqldbrg -s bwsqllogicalserver`

and within a few minutes I have deployed a General Purpose 2 vCore database. This assumes I’ve already created the resource group and logical server, but it shows how simple it can be to create a new database.

You can also deploy Azure SQL Database using Bicep ([`https://learn.microsoft.com/azure/azure-sql/database/single-database-create-bicep-quickstart`](https://learn.microsoft.com/azure/azure-sql/database/single-database-create-bicep-quickstart)) and Terraform ([`https://learn.microsoft.com/azure/azure-sql/database/single-database-create-terraform-quickstart`](https://learn.microsoft.com/azure/azure-sql/database/single-database-create-terraform-quickstart)).

In addition, as I showed earlier in this chapter, you can use **Copilot in Azure** to get easy install steps to deploy an Azure SQL Database using the az CLI.

### Implementation Details

Note

These implementation details may change over time as we change and improve the service. I offer up some of these details, so you can understand how we build, manage, and run the service.

In Chapter [1](#496204_2_En_1_Chapter.xhtml) of this book, I covered the incredible history of how we have built architecture to power millions of databases for Azure SQL Database. Let me give you a few more details about how we implement Azure SQL Database behind the scenes.

Note

As I interviewed many people in the Microsoft engineering team behind the scenes about Azure SQL Database even as I write this chapter, we are looking into how to make our implementation more efficient. Therefore, it is possible even some of these details could be a bit outdated by the time you read this chapter. That is the *speed of the cloud* and an author’s nightmare!

#### Dedicated Rings and Instances

Unlike Managed Instance, we pre-deploy rings dedicated to hosting Azure SQL Databases. With only a few exceptions, each database is hosted by a dedicated SQL Server instance (exceptions being “subcore” DTU options and elastic pools). This allows us to provide better isolation for a customer and keep the “just worry about the database” model while opening some instance-level surface area (e.g., DMVs and columnstore indexes). We may provision these instances on the same VM or node, but those details are abstracted from you provided we keep to our Service Level Agreement (SLA) and objectives for performance.

All the rings and instances are powered and managed using **Azure Service Fabric**. This is the same fabric software that you can build your own microservices. The Azure Service Fabric architecture is well documented at [`https://learn.microsoft.com/azure/service-fabric/service-fabric-architecture`](https://learn.microsoft.com/azure/service-fabric/service-fabric-architecture).

#### The Logical Server

As I stated earlier in this chapter, a database or logical server is just a metadata concept. We provide an interface to a server and a master database. But, when you query various aspects of the master database, we may be pulling data from other stores or files within the service to show you the information. I remember getting this question from Anna Hoffman as we built out our now famous Azure SQL for Beginners series ([`https://aka.ms/azuresql4beginners`](https://aka.ms/azuresql4beginners)). What I told Anna is that with SQL Server a master database has its own metadata and the physical location of the other databases on the server, which is why you can “switch” database context very easily (i.e., USE <database>). In this case of Azure SQL Database and a logical server, the master contains metadata stored on gateway servers and locations of nodes where user databases exist for the server. It is kind of difficult to switch context to another server easily with a USE statement.

#### Storage, Compute, and Gateways

You will see in Chapter [8](#496204_2_En_8_Chapter.xhtml) for availability more details on the architecture behind the scenes on how we implement high availability (HA) for General Purpose, Business Critical, and Hyperscale service tiers. We achieve certain HA capabilities by either using Azure Storage or local storage with technologies like Always On Availability Groups.

In each of these cases, one of the key components for connectivity is a *gateway*. Gateways are nodes that basically route traffic to nodes hosting SQL Server databases. I mentioned the use of redirect vs. proxy connection types with a Managed Instance earlier in the chapter. The same concept will apply with an Azure SQL Database. Gateways are critical to connectivity in that they provide abstractions to an application for connectivity no matter where the node for the database lives, which you will learn more about in Chapter [6](#496204_2_En_6_Chapter.xhtml) on security and Chapter [8](#496204_2_En_8_Chapter.xhtml) on availability.

#### Serverless

Serverless compute involves several interesting technologies we implement within the standard deployment of a SQL Server. Many of these details are described in our documentation at [`https://learn.microsoft.com/azure/azure-sql/database/serverless-tier-overview`](https://learn.microsoft.com/azure/azure-sql/database/serverless-tier-overview).

Since storage and compute are separated for a Serverless deployment, *pausing* a database is not that difficult since no application is connected. We just need to keep around enough state information that when a new login is made, we can connect the database to an instance and “warm up” the application.

Autoscaling is more interesting. We need to scale up or down the database CPU resources without application disruption. And if we can scale on a node that can meet the new scaling demand, there is no disruption. However, if we cannot meet that demand, we may need to use an Azure Load Balancer to keep the application connected if possible until a new node is found to meet demand, but there can be a disconnection when the new node is brought up.

Memory management is also different in that we must deploy memory policies to reclaim memory for the SQL Server instance when CPU or cache utilization is low. Think of this concept as though we are signaling SQL Server there is external memory pressure and lowering the target for memory. Autoscaling and memory management for Serverless is described more at [`https://learn.microsoft.com/azure/azure-sql/database/serverless-tier-overview?view=azuresql&tabs=general-purpose#autoscaling`](https://learn.microsoft.com/azure/azure-sql/database/serverless-tier-overview%253Fview%253Dazuresql%2526tabs%253Dgeneral-purpose%2523autoscaling).

#### Hyperscale

Hyperscale is a unique implementation for a database within the same architecture of databases (dedicated rings and nodes) implemented much differently than General Purpose or Business Critical service tiers.

A Hyperscale deployment involves a series of nodes for compute, logging, and caching combined with Azure Storage. I talked about this architecture in Chapter [1](#496204_2_En_1_Chapter.xhtml) of the book, but it is worth showing you again as seen in Figure [4-31](#496204_2_En_4_Chapter.xhtml#Fig31) (which is directly from the documentation at [`https://learn.microsoft.com/azure/azure-sql/database/service-tier-hyperscale?view=azuresql#distributed-functions-architecture`](https://learn.microsoft.com/azure/azure-sql/database/service-tier-hyperscale%253Fview%253Dazuresql%2523distributed-functions-architecture)).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig31_HTML.jpg)

Figure 4-31

The Hyperscale architecture

I will show you more of the working parts of this architecture in Chapter [8](#496204_2_En_8_Chapter.xhtml) of this book, but let me stop and call out a few key components:

*   Separate compute and storage

Just like General Purpose, we store database files on Azure Storage. But notice here we use Azure Standard Storage. Speed to access the database files is not as important because of the caching system.

*   The *caching* system

We use a combination of page servers (actual nodes that host database pages) and buffer pool caches (think SSD drives that extend the buffer pool) on page servers and compute nodes.

*   Log Service

For Hyperscale, any logged changes are still in a log cache on the primary compute node. However, when log changes must be flushed to disk (a commit), these I/O requests are *redirected* to another node where a component called the Log Service runs (this is also called Xlog). The Log Service is responsible for ensuring changes are stored locally (called the *landing zone*) and are refreshed to the caching system, secondary replicas, and eventually to Azure Storage.

*   Decoupled replicas

In a way, Hyperscale provides you with the best of both the General Purpose and Business Critical tiers. The actual database and transaction log files are stored in Azure Storage (which has its own redundancy), but we also have replicas for extremely fast high availability.

However, the secondary replica system does not use Always On Availability Groups. In fact, the primary and secondary replicas are not aware of each other. Secondary replicas use a log change methodology but are fed changes from the Log Service. Commits on the primary replica can proceed once the Log Service has hardened the changes.

High availability is even allowed with no secondary replicas. How? If the primary node has an issue, we can deploy a new primary replica on a new node and use page servers or even the database files on Azure Storage because it is decoupled. Having said this, RTO is much faster with the presence of secondary replicas. The secondary replica system (because you can have four of these) provides the best read-scale option for Azure SQL Database.

*   *Hyperfast* backup and restore

Because most of the data access comes from the caching system, reading database pages from the database files is rare with a *warm* system. This allows us to use snapshot backups for database files. Snapshot backups are extremely fast since we just copy the files to another storage location. And the other amazing story is restore. Restoring a database snapshot is crazy fast!

#### Resource Governance

To meet the SLA requirements for Azure SQL Database, we must put some resource limits on the usage of the database. I have described some of these limits in this chapter. Behind the scenes, we use these technologies to enforce these limits.

##### SQL Server Resource Governor

Azure SQL Managed Instance allows user defined workload groups and pools. Azure SQL Database uses Resource Governor behind the scenes to enforce certain limits. Moving to a dedicated SQL Server instance was a key driver in allowing us to use Resource Governor.

##### Engine enhancements

The engine has been enhanced in Azure to detect the generation of a certain size and rate of transaction log records and govern the application if necessary. The primary signal this governance is happening is seeing a wait type of LOG_RATE_GOVERNOR. You can read more about log rate governance at [`https://learn.microsoft.com/azure/azure-sql/database/resource-limits-logical-server?view=azuresql#transaction-log-rate-governance`](https://learn.microsoft.com/azure/azure-sql/database/resource-limits-logical-server?view=azuresql#transaction-log-rate-governance).

##### Windows Job Objects

I have mentioned this technology before in this chapter. Windows Job Objects allow us to control resource usage on the SQL Server engine process to ensure we properly enforce resource limits like memory.

##### File Source Resource Manager (FSRM)

FSRM provides a mechanism, so we can properly enforce storage maximum sizes outside of what we control through SQL Server file size limits.

We have created a nice blog post talking about how we enforce resource limits using these technologies and why we use them at [`https://azure.microsoft.com/blog/resource-governance-in-azure-sql-database/`](https://azure.microsoft.com/blog/resource-governance-in-azure-sql-database/).

### Connecting and Verifying Deployment

Once you deploy an Azure SQL Database, you will likely want to quickly connect and verify aspects of the deployment.

#### Connecting to Azure SQL Database

Earlier in this chapter, I deployed my logical server and Azure SQL Database, allowing public endpoint access with options to access through Azure services and with a firewall rule for the client IP address where I deployed through the portal. I will show you how to connect using both techniques.

Because I used the option Allow Azure services and resources to access this server when I deployed the logical server for the database, I can use an Azure Virtual Machine or even sqlcmd from the Azure Cloud Shell to connect to this server and database.

To connect with Azure Cloud Shell, you need to find the name of the logical server for the deployment. There are many ways to do this through the portal. You can simply look at your resource groups or resources from the home of the portal and find the server **bwsqllogicalserver** (or your name).

Figure [4-32](#496204_2_En_4_Chapter.xhtml#Fig32) shows the Working Pane of the server with the server name to use when connecting with a SQL Server tool or application.

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig32_HTML.jpg)

Figure 4-32

Find the server name to connect

Notice I clicked next to the server name to copy the Fully Qualified Domain Name (FQDN) to the clipboard. You will find the FQDN is a combination of the logical server name and .database.windows.net.

Note

Azure SQL Database also supports the concept of a DNS alias, which you can read about at [`https://learn.microsoft.com/azure/azure-sql/database/dns-alias-overview`](https://learn.microsoft.com/azure/azure-sql/database/dns-alias-overview).

I can now bring up the Azure Cloud Shell (you can use the home page of the portal or [`https://shell.azure.com`](https://shell.azure.com)).

Since sqlcmd is installed in the cloud shell, I can use a syntax like Figure [4-33](#496204_2_En_4_Chapter.xhtml#Fig33).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig33_HTML.jpg)

Figure 4-33

Using sqlcmd from the Azure Cloud Shell

I could also connect inside a VM in Azure with a tool like SSMS, but let me show you another way.

I also configured the logical server for a firewall rule for the IP address of the computer when I was using the Azure Portal. In the Working Pane of the logical server, I can select **Show firewall settings** to see the following information. Figure [4-34](#496204_2_En_4_Chapter.xhtml#Fig34) shows this firewall setting along with other network options.

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig34_HTML.jpg)

Figure 4-34

Configure firewall rules for Azure SQL Database

A firewall rule is very much like a firewall rule you would configure in Windows Server or Linux. If you have used SQL Server before you know that by default, we do not open the firewall rules in the OS for port 1433 by default. The firewall rule above is opening access to the gateway for this logical server for this specific IP address. Firewall rules can be specified at the logical server or database level. You can read more about firewall rules for Azure SQL Database at [`https://learn.microsoft.com/azure/azure-sql/database/firewall-configure`](https://learn.microsoft.com/azure/azure-sql/database/firewall-configure). I will talk more about using a more secure method to connect to Azure SQL Database called **Private Link** in Chapter [6](#496204_2_En_6_Chapter.xhtml) of the book.

Since my client IP address is in the firewall rule, I can use a tool like SQL Server Management Studio to connect to the logical server from my client laptop as seen in Figure [4-35](#496204_2_En_4_Chapter.xhtml#Fig35).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig35_HTML.jpg)

Figure 4-35

Connecting to Azure SQL Database with SSMS

You can see here I used Microsoft Entra authentication since I added my Microsoft Entra account as an admin when I deployed the logical server.

Once I hit connect, I get the familiar Object Explorer, and expanding the list of databases, I see all the databases I have deployed in this chapter. I can right-click the server, select New Query, and try to switch database context to one of my databases as seen in Figure [4-36](#496204_2_En_4_Chapter.xhtml#Fig36).

![](images/496204_2_En_4_Chapter/496204_2_En_4_Fig36_HTML.jpg)

Figure 4-36

Trying to switch database context for Azure SQL Database

You can first see some differences for Object Explorer (OE), including the color of the icon (Azure blue) for the server name. Notice also there are not as many choices in OE as there are with SQL Server or Managed Instance (because this is not a full SQL Server instance).

When I connected with SSMS specifying no options, I was put in the context of the logical master for the logical server. I tried to change the database context with the familiar T-SQL USE statement, but it failed. Why?

If you think about T-SQL USE on a SQL Server, the engine switches context to a database stored on the instance. For an Azure SQL Database logical server, the databases are on separate SQL Server instances that can be spread across various rings in the Azure region. The USE statement is not built to redirect connections to different servers.

Therefore, for SSMS, you can either specify the database to connect to before you hit the Connect button (use the Options button) or switch database context using the drop-down box (which does change the connection context).

#### Verifying Deployment

While you can use the Activity Log to verify the deployment of the database, there is no method to see the ERRORLOG behind the SQL Server instance. Therefore, you can use several T-SQL queries to examine the deployment.

Note

Even though there are some queries that make sense to run in the context of the logical master, I ran these queries in the context of my database bwsqldb.

```
SELECT @@version;
Microsoft SQL Azure (RTM) - 12.0.2000.8   Jun 19 2024 16:01:48   Copyright (C) 2022 Microsoft Corporation
```

This is the exact same result as with Azure SQL Managed Instance indicating a versionless SQL Server.

```
SELECT database_id, name, compatibility_level FROM sys.databases;
database_id    name                compatibility_level
1              master              160
5              bwsqldb             160
6              bwsqldbserverless   160
7              bwsqldb1            160
8              bwsqldb2            160
9              sqldbiseasy         160
```

You will always see master and only your database from the context of a user database. But note that this is still the logical master not the physical master on the SQL Server instance hosting the database. Notice for Azure SQL Database we have moved to a default dbcompat of 160 (but you can change back if you want to).

```
SELECT name, object_id, type_desc FROM sys.objects;
```

Since this database was built on the sample AdventureWorksLT, I have ~233 rows from this catalog view, including system tables.

```
SELECT * FROM sys.dm_os_schedulers;
```

This is one of the DMVs we can expose since we are running on a dedicated SQL Server instance. I deployed an 8-vCore Hyperscale database, so as I would expect, I get eight ONLINE schedulers.

Note

**sys.dm_os_sys_info** is supported, but **sys.dm_os_process_memory** is not with Azure SQL Database.

```
SELECT * FROM sys.dm_exec_requests;
```

This is one of the most common DMVs in the world to check the state of what is running on a SQL Server. I run this just to make sure all the normal background processes are running, including LAZY WRITER, RECOVERY WRITER, LOCK MONITOR, etc.

Here is the interesting twist on this DMV for Azure SQL Database. This will show you requests for the instance for your database, not requests for other databases on your logical server (because they are deployed on other instances).

```
SELECT SERVERPROPERTY('EngineEdition');
```

Per the documentation at [`https://learn.microsoft.com/sql/t-sql/functions/serverproperty-transact-sql`](https://learn.microsoft.com/sql/t-sql/functions/serverproperty-transact-sql), the value of five is a SQL Database.

There are also two new DMVs specific to Azure not found in SQL Server:

```
SELECT * FROM sys.dm_user_db_resource_governance;
```

This DMV is really intended to show you resource limits for a specific Azure SQL Database. You can view limits like memory, max storage, log rates, etc. You can read the documentation for this DMV at [`https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-user-db-resource-governor-azure-sql-database`](https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-user-db-resource-governor-azure-sql-database).

```
SELECT * FROM sys.dm_os_job_object;
```

This is a DMV specific to Azure that shows resource limits Azure applies to the database using Windows Job Objects. The specific column I look at is **memory_limit_mb** which shows me the true amount of memory the database has access to. I talked about Windows Job Objects in the section “Implementation Details.”

Note

I do not recommend you rely on any results of running these queries in the context of the logical master. Even though you might see results, they do not mean anything since the logical master is not a true physical master database.

## Migrating to Azure SQL Database

Migrating to Azure SQL Database involves the same process as with a Managed Instance of assessment and planning, migration, application changes, and post-migration.

While the steps are the same, you will find several differences that are important:

*   First and foremost, you cannot restore a SQL Server backup to Azure SQL Database, so all your choices will feel like an “export/import” type of experience.

*   Azure SQL Database has more restrictions on features, so you may find your assessment is going to find more problems you need to take care of before migrating. An example is a feature like Service Broker which is supported in Managed Instance but not in Azure SQL Database. Let me give you a simple example. When I first tried to migrate the example WideWorldImporters ([`https://github.com/Microsoft/sql-server-samples/releases/tag/wide-world-importers-v1.0`](https://github.com/Microsoft/sql-server-samples/releases/tag/wide-world-importers-v1.0)) to Azure SQL Database, I ran into a bunch of problems because certain features used in the sample didn’t work in Azure SQL Database. Therefore, I needed to use the Standard WideWorldImporters found at [`https://github.com/Microsoft/sql-server-samples/releases/download/wide-world-importers-v1.0/WideWorldImporters-Standard.bacpac`](https://github.com/Microsoft/sql-server-samples/releases/download/wide-world-importers-v1.0/WideWorldImporters-Standard.bacpac).

*   Migrating to Azure SQL Database involves migrating your schema (all your definitions) first and then the data. You can use Azure Database Migration Service (DMS) to do this. Read more about how to do this at [`https://learn.microsoft.com/data-migration/sql-server/database/database-migration-service`](https://learn.microsoft.com/data-migration/sql-server/database/database-migration-service). Like Managed Instance, consider using a database compatibility level that matches your current SQL Server installation and then move later to the latest dbcompat level. Learn more about dbcompat at [`https://aka.ms/dbcompat`](https://aka.ms/dbcompat).

    You can also load your data into Azure SQL Database using SSIS packages, Azure Data Factory, bcp, or a BACPAC file. You should know that minimal logging for bulk import is not supported in Azure SQL Database.

*   The SQL Server enabled by Azure Arc Assessment ([`https://learn.microsoft.com/sql/sql-server/azure-arc/migration-assessment`](https://learn.microsoft.com/sql/sql-server/azure-arc/migration-assessment)) along with Azure Migrate assessments ([`https://aka.ms/azuremigrate`](https://aka.ms/azuremigrate)) can be helpful to see if your database configuration is compatible to migrate to Azure SQL Database or whether you will need to consider changes. One common example I see often is cross-database queries, which are not supported in Azure SQL Database.

Like Managed Instance, the application needs to change the connection string and possibly authentication method. Depending on what T-SQL features and language constructs are used, further application changes may be needed. To be more thorough, please look over this documentation for T-SQL differences: [`https://docs.microsoft.com/en-us/azure/azure-sql/database/transact-sql-tsql-differences-sql-server`](https://docs.microsoft.com/en-us/azure/azure-sql/database/transact-sql-tsql-differences-sql-server).

Use the rest of the chapters in the book to guide you on any post-migration changes you need to make to fully take advantage of security, performance, and availability in Azure.

## Summary

In this chapter, you learned how to go through a deployment exercise to make the best choices possible to deploy Azure SQL Managed Instance or Database. You learned the details of deployment for both a Managed Instance and Database along with some interesting implementation details.

You learned how to connect and run verification queries against a Managed Instance and Database deployment. You also learned migration techniques and tools to migrate an existing SQL Server to Azure SQL Managed Instance and Database.

The Cloud Workshop for the SQL Professional ([`https://aka.ms/cloudsqlworkshop`](https://aka.ms/cloudsqlworkshop)) has some great hands-on labs to try out deployment for Azure SQL Managed Instance and Azure SQL Database. Give these a spin for a step-by-step experience:

[`https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/05_Deploy_AzureSQLMI`](https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/05_Deploy_AzureSQLMI)

[`https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/07_Deploy_Manage_Optimize_AzureSQLDB`](https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/07_Deploy_Manage_Optimize_AzureSQLDB)

Now that you have deployed, learn more in the next chapter about how to make configuration choices and compare these choices to configuring a SQL Server instance or database.

# 5. Configuring Azure SQL

Now that you have deployed your Azure SQL Managed Instance or Database, there could be some configuration changes you may want to make before or after moving into production.

In this chapter, we will explore configuration options for Azure SQL Managed Instance and Database as it compares to configuration options for SQL Server. In addition, I will discuss some of the configuration choices you could normally make for SQL Server that are restricted for Azure SQL and why. I will also spend time toward the end of the chapter explaining space management, various techniques to load data, and why Azure SQL is referred to as versionless.

You can try out some of the methods I describe in the chapter. I will use the existing deployment I did in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book when discussing configuration choices and restrictions. For you to try out any of the techniques or commands I use in this chapter, you will need

*   An Azure subscription.

*   A minimum of Contributor role access to the Azure subscription. You can read more about Azure built-in roles at [`https://learn.microsoft.com/azure/role-based-access-control/built-in-roles`](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles).

*   Access to the Azure Portal.

*   A deployment of an Azure SQL Managed Instance and an Azure SQL Database.

*   Installation of the **az** CLI (see [`https://learn.microsoft.com/cli/azure/install-azure-cli`](https://learn.microsoft.com/cli/azure/install-azure-cli) for more details). You can also use the Azure Cloud Shell instead since az is already installed. You can read more about the Azure Cloud Shell at [`https://azure.microsoft.com/features/cloud-shell/`](https://azure.microsoft.com/features/cloud-shell/).

*   You will run some T-SQL in this chapter, so install a tool like SQL Server Management Studio (SSMS) at [`https://aka.ms/ssms`](https://aka.ms/ssms).

## Configuring Azure SQL Managed Instance

There are several options you can choose from to configure the instance of a SQL Server. Let us look at a few of these compared to configuring an Azure SQL Managed Instance.

### sp_configure

One of the ways to configure SQL Server at the instance level is with the system procedure **sp_configure**. This stored procedure is supported to use with Azure SQL Managed Instance with these exceptions:

*   Any configuration value that requires a server restart is not supported (since we do not offer an interface for you to restart the instance). For example, if you try to change the **scan for startup procs** configuration option, you will get an error:

    ```
    Msg 5869
    Changes to server configuration option scan for startup procs are not supported in SQL Database Managed Instances
    ```

*   Advanced configuration values are supported.

*   Some options are not supported because we do not allow that level of configuration or are enforced in a different manner through resource limits. For example, if I try to change the **max server memory** value, I get the following error:

    ```
    Msg 5870
    Changes to server configuration option max server memory (MB) are not supported in this edition of SQL Server.
    ```

### Trace Flags

Azure SQL Managed Instance sets a predetermined set of global trace flags ON which you can see with **DBCC TRACESTATUS** (there are some 31g trace flags set as of the time of the writing of this book).

Some flags we set cannot be turned of, such as trace flag 1800 (which is used for disk sector size alignments), and some you can turn off and on. To see a list of trace flags you can turn on and off, please see [`https://learn.microsoft.com/sql/t-sql/database-console-commands/dbcc-traceon-transact-sql?view=sql-server-ver15#remarks`](https://learn.microsoft.com/sql/t-sql/database-console-commands/dbcc-traceon-transact-sql%253Fview%253Dsql-server-ver15%2523remarks).

Session-level trace flags are not supported.

Note

We have been striving for some time to eliminate the need for trace flags, but clearly some are still needed. The addition of commands like ALTER DATABASE SCOPED CONFIGURATION is an example of configuration methods to use instead of trace flags.

### Tempdb

The max size limit of tempdb is configurable by changing your deployment option or size. For a General Purpose deployment, you get 24GB per vCore but limited to a max of ~2TB. For a Business Critical deployment, you are limited by the maximum storage size of the instance. The transaction log is capped at 120GB, but since Accelerated Database Recovery is ON by default and we regularly back up the transaction log, I do not foresee a problem with you running out of space.

You can also configure the following properties of tempdb in Azure SQL Managed Instance:

*   The number of tempdb data files (max of 128). A new instance is deployed by default with 12 data files and 1 transaction log file.

*   The autogrow increment for data and log files (as with SQL Server, this should be the same size for all data files).

*   The logical names of the tempdb files can be changed.

While all these changes are persisted, the only caveat here is that the initial size of all files is 16MB. You could change the size of the files, but anytime a restart or failover occurs, the initial size will be reset to 16MB, so don’t rely on changes to file size.

### Master and Model

Like SQL Server, you can configure the size of the master database and even add objects to it (but that is usually not recommended on SQL Server anyway).

The same concept applies to model. You can configure the model database size so that new databases will take on that size. Furthermore, you can add objects to model which will be picked up by new user databases.

### Configuring Edition

You can use the SQL Server setup program (or mssql-conf with Linux) to change the edition of SQL Server (e.g., Standard to Enterprise).

For Azure SQL Managed Instance, you can change the deployment service tier (General Purpose or Business Critical) or size (vCores or max storage) using the Azure Portal or CLI interfaces such as **az sql mi update** or PowerShell **Set-AzSqlInstance**.

Note

As I have mentioned to this point in the book, deploying or changing the deployment for a Managed Instance can be a time-consuming operation.

### Compute and Storage

You learned about the various options for compute and storage that come with your choice of service tier in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book.

Just remember that for Managed Instance, changes to vCores, hardware configuration, and/or max storage could result in a management operation that takes some time. For example, scaling storage for a General Purpose service tier can take as fast as five minutes. However, scaling vCores for any service tier can take an hour. In addition, many operations that affect a Business Critical service tier may require a reseeding of the database to another node. Downtime is minimal since any change doesn’t affect your application until a “cutover” takes place. Remember the NextGen for the General Purpose service tiers includes the ability to configure IOPS separate from compute and storage.

The complete list of all operations and expected durations can be found at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/management-operations-overview).

### Networking Configuration

Once you have deployed a Managed Instance, you may need to make some adjustments to the network configuration for the instance.

You have the option to **move the Managed Instance to another subnet**, either within the current virtual network or to a different one. Think of this like provisioning a new virtual cluster with your instance and all databases moved and then removing the existing cluster. Like other management operations, this could be a lengthy process, but your downtime is minimal. Learn all the details at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/vnet-subnet-move-instance`](https://learn.microsoft.com/azure/azure-sql/managed-instance/vnet-subnet-move-instance).

You can **enable the public endpoint** (which I left disabled during deploy) and **change the connection type** to Proxy (which is the default, but I chose Redirect during deployment).

In addition, there is another option that was offered during deployment, which is the **minimum TLS version**. Transport Layer Security (TLS) is a protocol for encrypting connections and is supported by SQL Server and Azure SQL. TLS 1.2 is the minimal supported version recommended today as it includes fixes for some vulnerabilities found in previous versions. However, be careful with this setting as it could break an application using drivers that can’t use that TLS version. You can read more about minimal TLS version at [`https://learn.microsoft.com/azure/azure-sql/database/connectivity-settings?view=azuresql&tabs=azure-portal#minimum-tls-version`](https://learn.microsoft.com/azure/azure-sql/database/connectivity-settings%253Fview%253Dazuresql%2526tabs%253Dazure-portal%2523minimum-tls-version).

Networking options for Managed Instance can also be configured using **az sql mi update** and PowerShell **Set-AzSqlInstance**.

I will discuss the ability to add private endpoint connections in Chapter [6](#496204_2_En_6_Chapter.xhtml) of the book.

### Maintenance

You can use the **Settings and Maintenance Updates** option to configure maintenance windows, set up alerts for maintenance activities, and control the update policy.

As I described in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book, Azure SQL Managed Instance supports a **maintenance window**. This allows you to define a different timeframe when scheduled maintenance is required on your deployed instance. Instead of the default, which is 5pm to 8am on any day of the week, you can choose different day ranges and timeframes.

In addition, you can set up **alerts** to notify you in advance when scheduled maintenance activities are required.

Finally, you can configure the **update policy**, which I described in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book. This allows you to maintain compatibility with SQL Server 2022 (allowing you to restore backups to SQL Server 2022 and set up online disaster recovery) or use the option for “Always-up-to-date” which is the versionless engine for continuous updates.

Regardless of the configured update policy, all instances will continue receiving updates and features that don’t require changes to the SQL engine, such as the following features: zone redundancy, instance stop and start, and fast provisioning.

Important

If you choose the Always-up-to-date policy, you cannot go back to the SQL Server 2022 policy.

### Start and Stop

One of the cool features that the Azure SQL Managed Instance team built since the first edition of this book is the ability to start and stop the instance, including on a schedule.

You can see how to configure this in the Azure Portal as in Figure [5-1](#496204_2_En_5_Chapter.xhtml#Fig1).

![](images/496204_2_En_5_Chapter/496204_2_En_5_Fig1_HTML.jpg)

Figure 5-1

Starting and stopping an Azure SQL Managed Instance

This feature can be a huge cost savings because you do not pay for compute when the instance is stopped (but you still pay for storage). However, starting a stopped instance could take up to 20 minutes.

### Configuring Databases

Once you deploy an Azure SQL Managed Instance, you can now add databases or configure existing databases using the Azure Portal, tools like SSMS, or the T-SQL CREATE DATABASE or ALTER DATABASE statements.

The CREATE DATABASE T-SQL syntax is very simple since you do not specify files or any WITH options. For example, for the Managed Instance I deployed, I can run the following T-SQL statement to create a new database:

```
CREATE DATABASE gocowboys;
```

The new database will take on the properties of the model database just like SQL Server. The one difference is that we automatically create a memory-optimized filegroup called XTP, which you can see if you query the **sys.filesgroups** catalog view. We create the XTP filegroup even if you are using the General Purpose service tier which does not support In-Memory OLTP. This makes it easier to support this feature if you move to Business Critical.

This database will also take on the properties set by model and is like SQL Server except for two key options that are on by default: **Query Store** and **Accelerated Database Recovery**.

Once a database is created, you can use the ALTER DATABASE statement to make various changes to options or to add/remove files and filegroups. One of the nifty things we have done with the T-SQL documentation is to allow you to choose a product to see the exact syntax support for statements like ALTER DATABASE. Figure [5-2](#496204_2_En_5_Chapter.xhtml#Fig2) shows the documentation reference for ALTER DATABASE (which is at [`https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql?view=azuresqldb-mi-current`](https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql?view=azuresqldb-mi-current)) after I selected Managed Instance:

![](images/496204_2_En_5_Chapter/496204_2_En_5_Fig2_HTML.jpg)

Figure 5-2

ALTER DATABASE reference for Managed Instance

Most of the SET options you normally see for SQL Server are available for Managed Instance. One notable exception is ACCELERATED_DATABASE_RECOVERY. We turn on this option by default for your database and do not allow you to disable it. Why? It is important for this option to be on for us to meet SLA requirements and ensure you do not have issues with running out of transaction log space.

One of the most significant advances for Managed Instance since the first edition of this book is to **allow 500 databases** per instance.

One of the key options for a database is the *database compatibility level*, and Managed Instance supports setting the “dbcompat” with ALTER DATABASE. Compat levels 90 through 160 are currently supported for Managed Instance databases. Read more about dbcompat at [`https://aka.ms/dbcompat`](https://aka.ms/dbcompat).

Note

If you restore a backup from an existing SQL Server database or Managed Instance, we will retain the dbcompat level of that database when it was backed up.

Even though you do not have to worry about physical file placement, you do have the ability to add database files and change file sizes up to the maximum instance storage limit. You can also create filegroups. A good example of why you might want to increase the number of files or size is to improve I/O performance. My colleague Jovan Popovic has a very nice blog post describing how to do this at [`https://medium.com/azure-sqldb-managed-instance/increasing-data-files-might-improve-performance-on-general-purpose-managed-instance-tier-6e90bad2ae4b`](https://medium.com/azure-sqldb-managed-instance/increasing-data-files-might-improve-performance-on-general-purpose-managed-instance-tier-6e90bad2ae4b).

Having said that, the new NextGen Azure SQL Managed Instance doesn’t require you to use multiple files to increase I/O performance because you can control IOPS separately. NextGen also just gives you lower I/O latency, higher IOPS, and better I/O throughput. Check out this blog post to learn more: [`https://techcommunity.microsoft.com/t5/azure-sql-blog/introducing-azure-sql-managed-instance-next-gen-gp/ba-p/4092647`](https://techcommunity.microsoft.com/t5/azure-sql-blog/introducing-azure-sql-managed-instance-next-gen-gp/ba-p/4092647).

Note

SQL Server Managed Studio (SSMS) also supports changing database options and file options as you can with SQL Server.

It is also important to know that the ALTER DATABASE SCOPED CONFIGURATION T-SQL statement is supported for Azure SQL Managed Instance just like SQL Server. You can read more at [`https://learn.microsoft.com/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql`](https://learn.microsoft.com/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql).

## Configuring Azure SQL Database

Since deploying an Azure SQL Database is effectively creating a database, you may want to configure some settings about the database from the deployment. You may also want to perform some configuration on the logical server associated with the database, including creating a new database on the logical server.

### Creating New Databases

The T-SQL CREATE DATABASE statement is supported when connected to a logical server (in any database context). You can view the full syntax of CREATE DATABASE for Azure SQL Database at [`https://learn.microsoft.com/sql/t-sql/statements/create-database-transact-sql?view=azuresqldb-current`](https://learn.microsoft.com/sql/t-sql/statements/create-database-transact-sql?view=azuresqldb-current).

The documentation reference above shows all the possible options you can use for EDITION and SERVICE_OBJECTIVE. In this case, the EDITION is used to decide a DTU model, General Purpose, Business Critical, or Hyperscale. The SERVICE_OBJECTIVE is used to select options like Serverless, hardware generation, and number of vCores.

Note

Remember that creating a database for Azure SQL Database with CREATE DATABASE is a deployment. For SQL Server, we create new files and add metadata to the master database. For Azure SQL Database, we are building a new deployment (i.e., a dedicated instance for the database) to host the database and storing information in our control plan (e.g., gateways). This means the CREATE DATABASE is an asynchronous operation but can be tracked with the DMV **sys.dm_operation_status**. You must query this DMV from the logical master, and we retain history for one hour.

You will notice that you only have two options when creating a new database: collation and deployment options. Deployment options allow you to choose a purchasing model, service tier, compute model, and size.

If you remember, in Chapter [4](#496204_2_En_4_Chapter.xhtml), I deployed several databases using different options for the deployment. Connecting to the logical server **bwsqllogicalserver** I used in the Chapter [4](#496204_2_En_4_Chapter.xhtml) examples, I can create a new Hyperscale, 8-vCore database with the following T-SQL syntax:

```
CREATE DATABASE bwhyperscale
(EDITION = 'Hyperscale', SERVICE_OBJECTIVE = 'HS_Gen5_8');
```

Remember this is a deployment, so it can take longer than you might be used to for a SQL Server database creation. In this example, it took about four minutes for this T-SQL statement to complete. And notice since we are using Hyperscale, we don’t have to specify a max storage option.

The CREATE DATABASE statement for Azure SQL Database also supports the AS COPY OF option to create a database as a copy of another database even from another logical server.

Any database that is created has the following options turned on by default:

*   SNAPSHOT_ISOLATION_STATE

*   READ_COMMITTED_SNAPSHOT

*   AUTO_CREATE_STATISTICS

*   AUTO_UPDATE_STATISTICS

*   FULL RECOVERY

*   CHECKSUM

*   TDE

*   QUERY_STORE

*   ACCELERATED_DATABASE_RECOVERY

One option you may not notice is set on if you examine **sys.databases** is *stale page detection*. Stale page detection assists Azure SQL Database to find possible data integrity problems due to I/O problems in the infrastructure. I will discuss more data integrity checks provided by Azure SQL in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book.

### Altering Databases

For any database that is created, you can use the T-SQL ALTER DATABASE statement to modify certain SET options. You can use this documentation reference to see what options you can turn on and off at [`https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql-set-options?view=azuresqldb-current`](https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql-set-options?view=azuresqldb-current). You do not have the ability to change any properties of the files behind this database. To change the maximum size of the database, you would use the MAXSIZE parameter. You can also change the EDITION or SERVICE_OBJECTIVE with ALTER DATABASE or the Azure Portal. Any execution of ALTER DATABASE is also asynchronous and can be tracked with **sys.dm_operation_status**.

One of the key options for a database is the *database compatibility level*, and Azure SQL Database supports setting the “dbcompat” with ALTER DATABASE. Compat levels 90 through 160 are currently supported for Azure SQL Database. Read more about dbcompat at [`https://aka.ms/dbcompat`](https://aka.ms/dbcompat).

It is also important to know that the ALTER DATABASE SCOPED CONFIGURATION T-SQL statement is supported for Azure SQL Database just like SQL Server. This is a very important option for you to control various methods to control various database behaviors. You can read more at [`https://learn.microsoft.com/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql`](https://learn.microsoft.com/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql).

The **az sql db** CLI also supports the ability to change options about the deployment of the database as documented at [`https://learn.microsoft.com/cli/azure/sql/db?view=azure-cli-latest#az-sql-db-update`](https://learn.microsoft.com/cli/azure/sql/db?view=azure-cli-latest#az-sql-db-update). PowerShell also provides the **Set-AzSQLDatabase**, which is documented at [`https://learn.microsoft.com/powershell/module/az.sql/set-azsqldatabase`](https://learn.microsoft.com/powershell/module/az.sql/set-azsqldatabase).

The Azure Portal supports changing several options about the database deployment from the Service Menu, Command Bar, and Working Pane.

### Network Configuration

When we deployed an Azure SQL Database in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book, we were able to specify a few networking options for the logical server, including public endpoint access, allowing Azure services, connection types, and firewall rules. The keyword here is logical server. Even though you can technically set up firewall rules at the database level, most of your networking configuration happens at the logical server context.

As you can see from Figure [5-3](#496204_2_En_5_Chapter.xhtml#Fig3), for an existing logical server you can configure settings for public access, private access, and connectivity.

![](images/496204_2_En_5_Chapter/496204_2_En_5_Fig3_HTML.jpg)

Figure 5-3

Configuring networking for a logical server

**Public access** allows you to configure virtual networks and firewall configurations.

We will cover more about **private access**, including private endpoints, in Chapter [6](#496204_2_En_6_Chapter.xhtml) of the book.

**Connectivity** allows you to change settings like outbound connections, connection types, and TLS.

**Outbound connections** allow you to create firewall rules for any connections from the server to other sources like Azure Storage or other logical servers. Features impacted by this option include auditing, OPENROWSET, and REST APIs (which you will learn more about in Chapter [10](#496204_2_En_10_Chapter.xhtml) of the book).

Connection policy is like the connection type for a Managed Instance: Proxy or Redirect. Notice here for Azure SQL Database there is the choice of default. Default uses a policy to use a Proxy connection type if the connection originates outside of Azure while Redirect if the connection is within Azure.

The Redirect connection policy can be much faster for application latency because the gateway is used to redirect traffic to the direct node for the Azure SQL Database. My colleague Anna Hoffman has a nice example to how much faster Redirect can be over Proxy at [`https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/02-DeployAndConfigure`](https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/02-DeployAndConfigure).

Minimal TLS version is exactly like the TLS requirements as I described earlier in this chapter for a Managed Instance.

### Configuring Serverless

I showed you in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book the concept of deploying a Serverless Azure SQL Database. Figure [5-4](#496204_2_En_5_Chapter.xhtml#Fig4) shows the configuration choices for vCores for a General Purpose Serverless database.

![](images/496204_2_En_5_Chapter/496204_2_En_5_Fig4_HTML.jpg)

Figure 5-4

Configure a serverless database

You can see from your choices you can control a range of vCores, including a minimum and maximum. The minimum can be as low as 0.5 and the maximum can be 80\. You will be only charged for the vCores you use per second (with a minimum charge for the min vCore setting). For General Purpose service tier, you also have the choice of auto-pause. If we detect your use of the database is idle, we can pause your compute, and you won’t be charged for any compute during idle time (just storage). Then when you connect to the database again, we will attach your database to a new SQL Server instance and start it up.

### Configuring Elastic Pools

I showed you how to deploy a database in an elastic pool in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book. You can configure an elastic pool like a database, *at the pool level*, including setting vCores, storage, and redundancy.

One of the promises of elastic pool is to share resources across many databases, so you don’t have to worry about managing minimum and maximum resources across them. However, it could be that you want to make sure within the pool that no one database tries to maximize too many resources.

Figure [5-5](#496204_2_En_5_Chapter.xhtml#Fig5) shows how you can configure the minimum and maximum vCores for each database in the pool.

![](images/496204_2_En_5_Chapter/496204_2_En_5_Fig5_HTML.jpg)

Figure 5-5

Configuring per database resources for an elastic pool

## Configuration Restrictions

While there are several options to configure an Azure SQL Managed Instance and Database, I thought it is important to call out some of the configuration options you can make for SQL Server both at the instance and database levels *but are restricted* in Azure SQL. When possible, I will call out why certain options are restricted or are not needed in a PaaS environment.

### Azure SQL Managed Instance Restrictions

Many of the configuration choices at a SQL Server level are done using the SQL Server Configuration Manager (Windows) or mssql-conf (Linux), T-SQL, or tools like SSMS. Since I have worked with SQL Server for many years, I thought it would be interesting for you to know what options exist in these tools and interfaces and why they are restricted for Azure SQL Managed Instance and Databases.

#### Start and Stop Services

You can stop and start a Managed SQL Instance, but you cannot do this for an Azure SQL Database (remember a serverless database does have an auto-pause feature).

Note

You will learn in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book how to manually fail over an Azure SQL Database using the PowerShell command **Invoke-AzSqlDatabaseFailover**.

#### Instant File Initialization

There is no interface for you to enable Instant File Initialization (IFI), which you can read about at [`https://learn.microsoft.com/sql/relational-databases/databases/database-instant-file-initialization`](https://learn.microsoft.com/sql/relational-databases/databases/database-instant-file-initialization). However, there are two important points why I do not think this is a big factor:

*   IFI is not supported or used when the database has Transparent Data Encryption (TDE) enabled. TDE is enabled by default for databases for Azure SQL Managed Instance and Database.

*   I did some testing for “autogrow” scenarios and could never cause any major performance bottlenecks.

#### Locked Pages

There is no interface to enable locked pages (or large pages) as with SQL Server. However, we may choose to enable this in some situations in our deployments. The key for you is you shouldn’t have to worry about this. We will control memory for the OS and this setting in coordination if we need to use it.

#### FILESTREAM and Availability Groups

SQL Server Configuration Manager allows you to enable the FILESTREAM and Always On Availability Group features. FILESTREAM is not supported by Azure SQL. Always On Availability Groups (AG) are used behind the scenes for Business Critical service tiers, but you cannot set up or configure an AG on your own.

#### Server Collation

You can set the collation for the SQL Server instance during deployment of a Managed Instance but cannot change it after that. You can still set collations for databases or columns in tables like SQL Server.

Azure SQL Database allows you to specify a database collation during deployment, but you cannot change it later. You can also define collations at the column level.

#### Startup Parameters

SQL Server Configuration Manager allows you to set certain startup parameters for the SQL Server engine as documented at [`https://learn.microsoft.com/sql/database-engine/configure-windows/database-engine-service-startup-options`](https://learn.microsoft.com/sql/database-engine/configure-windows/database-engine-service-startup-options). None of these options are exposed to modify for Azure SQL. In my experience, most of these options are used for certain “edge” scenarios that you won’t need or can’t use (like starting SQL Server in single user mode). Having said that as I said earlier in this chapter, you can set some trace flags for Azure SQL Managed Instance.

#### ERRORLOG Configuration

SSMS allows you to configure the number of ERRORLOG files and maximum size. These options are not supported to configure for Azure SQL Managed Instance because it requires a restart of the instance. Furthermore, while it might be nice to have more than the default of six ERRORLOG files, the maximum size of the ERRORLOG value is generally used to control disk space, but that is not a concern with Managed Instance.

Note

The system procedure sp_cycle_errorlog is supported on Managed Instance. However, ERRORLOG files are stored on the local node, so they are not saved on a failover.

#### Error Reporting and Customer Feedback

SQL Server provides a way to configure error reporting and customer feedback as documented at [`https://learn.microsoft.com/sql/sql-server/usage-and-diagnostic-data-configuration-for-sql-server`](https://learn.microsoft.com/sql/sql-server/usage-and-diagnostic-data-configuration-for-sql-server). This configuration is not possible for Azure SQL but does not apply.

Note

We do use telemetry to improve Azure SQL. Take a look at this article where the legendary Conor Cunningham explains in more detail: [`https://redmondmag.com/articles/2018/02/14/qa-lead-sql-architect-part-1.aspx`](https://redmondmag.com/articles/2018/02/14/qa-lead-sql-architect-part-1.aspx).

If you have any concerns about what information about your deployment is collected or used by Microsoft, I recommend you read over privacy and other legal documents at [`https://azure.microsoft.com/support/legal/`](https://azure.microsoft.com/support/legal/).

#### ALTER SERVER CONFIGURATION

ALTER SERVER CONFIGURATION was introduced several releases ago with SQL Server as a new way to configure the instance vs. sp_configure.

ALTER SERVER CONFIGURATION is not supported with Azure SQL, but there are no options where it makes sense to configure an instance for Azure SQL given most of these options are done automatically or do not apply.

#### “Mixed Mode” Security

Azure SQL Managed Instance and Database both support a similar concept to “mixed mode” security from SQL Server. You can set up authentication for your instance or database to be

*   Microsoft Entra authentication only

*   Both SQL and Microsoft Entra authentication

*   SQL Authentication only

Here are a few notes about these options:

*   If you have already allowed SQL Authentication and change to Microsoft Entra authentication only, no SQL logins are removed, just disabled.

*   Azure SQL Managed Instance does allow Windows Authentication to help with application compatibility.

#### Logon Auditing

The SQL Server engine for years has supported the ability to track successful, failed, both (and none) login attempts. This tracking is written to the ERRORLOG and is configured typically through SSMS. While for Managed Instance, the options exist to configure this tracking of logins, they do not take effect because it requires a restart of SQL Server. Failed logins, which is the default, is tracked in the ERRORLOG.

You can perform your own audit of logins with Extended Events or SQL Audit for Azure SQL. I will show you more about auditing for Azure SQL in Chapter [6](#496204_2_En_6_Chapter.xhtml) of the book.

#### Server Proxy Account

SSMS supports the configuration of a server proxy account for the system procedure **xp_cmdshell**. This configuration is not supported for Managed Instance since xp_cmdshell is not supported for Azure SQL Managed Instance or Database.

#### Database Restrictions

Many of the common options to configure a database through ALTER DATABASE are allowed for Managed Instance just like SQL Server. One notable exception is that you cannot disable ACCELERATE_DATABASE_RECOVERY. That is because we rely on ADR technology to ensure the promised SLA for availability.

### Azure SQL Database Restrictions

All the restrictions listed in the previous section for Managed Instance also apply to Azure SQL Database.

A few other notable restrictions you may have guessed since Azure SQL Database does not expose a full SQL Server instance:

*   sp_configure is not supported.

*   DBCC TRACEON is not supported. In fact, if you run this command, it fails with Msg 2571 (i.e., no permissions).

*   You can use DBCC TRACESTATUS to see what trace flags we enable even if you cannot turn them on or off (the list is pretty much the same as is used with Managed Instance).

ACCELERATED_DATABASE_RECOVERY is also enabled by default and cannot be disabled. The same applies to Query Store, but you can disable Query Store from writing data and make it read only.

## Azure SQL Space Management

One other topic related to Azure SQL configuration is space management. While we have abstracted the physical placement of database and transaction log files for both Azure SQL Managed Instance and Database, *sizes* are still a factor you must manage and consider. Because Azure SQL Managed Instance and Database provide different deployment surface areas, space management can be different between each option.

### Azure SQL Managed Instance Space Management

As I have described in Chapter [4](#496204_2_En_4_Chapter.xhtml) for deployment, the maximum size for data for a Managed Instance is the overall size for all database storage. Your choice of vCores and options like Business Critical affect the maximum storage size for the instance.

With SQL Server, when a database file has a limit on size, you may encounter Msg 1105 to indicate you have run out of space. This error could occur if you also run out of disk space on the filesystem hosting your database. For a Managed Instance, you can also encounter Msg 1105 if you have run out of space on the maximum size of your database. However, since we enforce maximum storage limits for all databases, you may also encounter an error before you run out of space in your database like this:

```
Msg 1133
The managed instance has reached its storage limit. The storage usage for the managed instance cannot exceed (%d) MBs.
```

To be clear what this means, it is possible for you to hit this error before hitting the maximum size of an individual database because collectively you have hit the limit for all databases (including system databases).

### Azure SQL Database Space Management

Azure SQL Database manages space differently than Managed Instance. The maximum size as specified during deployment is the *possible* maximum size of a single database file.

If you deploy an Azure SQL Database, the maximum size is typically reflected in the **max_size** column of **sys.database_files** for a single data file. However, for larger maximum database size values (e.g., 1TB), we may not set the max_size to that value but grow as data is added to the eventual max_size.

If you run out of space in the database, you will get a quota error instead of the traditional 1105 error like this:

```
Msg 40544, Level 17, State 2, Line 12
The database '' has reached its size quota. Partition or delete data, drop indexes, or consult the documentation for possible resolutions.
```

The transaction log is handled differently. While you configure the maximum data storage size, we automatically add 30% above this size for the transaction log. Note that we can in some cases increase this number, but you will never be charged for more than the 30% additional space for the log. Do not use the max_size in sys.database_files for the maximum size of the transaction log. It does not accurately reflect the quota. In all my testing, I never ran out of transaction log space due to two factors:

*   We regularly and often back up the transaction log (remember we use full recovery).

*   We have Accelerated Database Recovery (ADR) enabled by default, and you cannot disable it. With ADR, the log can be truncated even in the presence of active transactions.

The one exception to space management for Azure SQL Database is the Hyperscale service tier. A Hyperscale database has a 100TB limit, but theoretically we believe it could be unlimited. When you deploy a Hyperscale database, we create multiple database files that total around 40GB. Then as you add data, we automatically grow the database. The transaction log has a limit of 1TB, but we back up the log regularly and rely also on ADR, so I do not think you will run out of space.

## Loading Data

Whether you deployed a new Managed Instance or Database or migrated from an existing instance or database, you inevitably will want to load data. Just like a SQL Server installation, you have several tools and options to load data. In addition, you have new services available to you in Azure.

### Keep These in Mind

Just like a SQL Server, you may need **additional resources** when importing large amounts of data. For Azure SQL Managed Instance or Database, keep in mind any additional vCore or size requirements for importing data. I have described how long scaling can take for Managed Instance, so keep this in mind as you deploy.

Several of the data loading techniques involve loading a *file*. While you can always use these techniques to load files from on-premises environment, any tool or command run in the Azure infrastructure should be done using Azure Storage or Azure Files. Some of your data may not be *cloud-born*, so consider using tools like **AzCopy** to load your files into Azure Storage or Azure Files. You can read more about AzCopy at [`https://learn.microsoft.com/azure/storage/common/storage-use-azcopy-files`](https://learn.microsoft.com/azure/storage/common/storage-use-azcopy-files). If your files are very large, take a look at our documentation on guidance and tools at [`https://learn.microsoft.com/azure/storage/common/storage-solution-large-dataset-moderate-high-network`](https://learn.microsoft.com/azure/storage/common/storage-solution-large-dataset-moderate-high-network).

I have mentioned already in the book that Azure SQL Managed Instance and Database both use the FULL recovery model, and this cannot be changed (we need this to meet SLA requirements). This means that **minimal logging is not supported** for bulk operations (except for tempdb). Therefore, you should ensure you use certain techniques like *batch sizes*. A batch for bulk import is a defined unit of transaction. A batch size of 10,000 means 10,000 rows are imported for a transaction. Typically, using large batch sizes can improve bulk performance and should be used for Azure SQL. However, you should also keep in mind that if you use too large of a batch size, you might be throttled by *log rate governance*. You can read more about log rate governance at [`https://learn.microsoft.com/azure/azure-sql/database/resource-limits-logical-server?view=azuresql#transaction-log-rate-governance`](https://learn.microsoft.com/azure/azure-sql/database/resource-limits-logical-server?view=azuresql#transaction-log-rate-governance). If your intended target for data is columnstore, you should consider directly importing into a clustered columnstore index. You can read more about data loading guidance for columnstore at [`https://learn.microsoft.com/sql/relational-databases/indexes/columnstore-indexes-data-loading-guidance`](https://learn.microsoft.com/sql/relational-databases/indexes/columnstore-indexes-data-loading-guidance).

Azure does charge on certain types of network traffic called **inbound** and **outbound**. Any inbound traffic (e.g., bulk import data from your on-premises environment into Azure) is free.

Outbound traffic can be charged if that traffic travels between Azure regions. If you use bulk import from an Azure Virtual Machine or Azure Data Factory Service from one region into an Azure SQL Database in another region, there could be outbound network charges. You can read more at [`https://azure.microsoft.com/pricing/details/bandwidth`](https://azure.microsoft.com/pricing/details/bandwidth).

### bcp

The bulk copy program (bcp) is perhaps the most traditional and most popular tool in the history of SQL Server to export and import data. The bcp program runs on Windows, Linux, and MacOS computers. You can get all the latest information about bcp at [`https://learn.microsoft.com/en-us/sql/tools/bcp-utility`](https://learn.microsoft.com/en-us/sql/tools/bcp-utility).

Since bcp is a program that reads a file and *bulk imports* data into a SQL Server or Azure SQL, you must run the bcp program on a computer that can access the file and connect to Azure SQL. That could be on a computer on-premises or Azure Virtual Machine. In any of these scenarios, you can use bcp against a file that is hosted on your on-premises environment, local to the Azure Virtual Machine storage (which may be data disks in Azure Storage), or in Azure Storage. Since bcp does not support a direct path to Azure Storage (i.e., URL), you can use Azure Files.

One other option is to use bcp from the Azure Cloud Shell. bcp in the Azure Cloud Shell runs in the Azure infrastructure (think of it like running in a temporary VM), so provided you have network connectivity set up correctly, you can bulk import into Azure SQL Managed Instance or Database. The only provision here is that the cloud shell cannot join a virtual network, so you would have to set up a public endpoint access. The Azure Cloud Shell supports persistence of files through a concept called a *clouddrive* which you can read more about at [`https://learn.microsoft.com/azure/cloud-shell/persisting-shell-storage`](https://learn.microsoft.com/azure/cloud-shell/persisting-shell-storage). You can copy files into your clouddrive and then use bcp to import the file into Azure SQL.

bcp uses *Bulk APIs* behind the scenes, so you can also write applications that use the Bulk APIs. Here is an example of Bulk APIs with ODBC at [`https://learn.microsoft.com/sql/relational-databases/native-client-odbc-extensions-bulk-copy-functions/sql-server-driver-extensions-bulk-copy-functions`](https://learn.microsoft.com/sql/relational-databases/native-client-odbc-extensions-bulk-copy-functions/sql-server-driver-extensions-bulk-copy-functions).

### BULK INSERT and OPENROWSET

The T-SQL BULK INSERT and OPENROWSET (using the BULK option) statements support bulk import of data from a file. One nice advantage of these commands is that they run in the context of the SQL Server engine.

For SQL Server, it is common to copy a file to drive or network share that the computer hosting SQL Server can access. Then you use BULK INSERT to reference that file to import. The problem is that for Azure SQL Managed Instance and Database, you do not have access to the underlying node file system. Therefore, BULK INSERT and OPENROWSET statements have been enhanced to support Azure Blob Storage (even from an on-premises SQL Server).

Here is how this works. You create an EXTERNAL DATA SOURCE (like Polybase) to reference an Azure Storage account. The BULK INSERT and OPENROWSET statements have been enhanced to support a DATA_SOURCE parameter. I’ll discuss external data sources more in Chapter [9](#496204_2_En_9_Chapter.xhtml) of the book.

Now you can connect directly to your Azure SQL Managed Instance or Database with your favorite SQL tool (remember it could be sqlcmd in the Azure Cloud Shell) and bulk import data having the SQL Server engine do the work.

Here is an example syntax on how to do this with BULK INSERT from the documentation at [`https://learn.microsoft.com/sql/t-sql/statements/bulk-insert-transact-sql?view=sql-server-ver15#f-importing-data-from-a-file-in-azure-blob-storage`](https://learn.microsoft.com/sql/t-sql/statements/bulk-insert-transact-sql?view=sql-server-ver15#f-importing-data-from-a-file-in-azure-blob-storage). This requires you first to create an Azure Storage account and container.

```
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'YourStrongPassword1';
GO
--> Optional - a DATABASE SCOPED CREDENTIAL is not required because the blob is configured for public (anonymous) access!
CREATE DATABASE SCOPED CREDENTIAL MyAzureBlobStorageCredential
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = '******srt=sco&sp=rwac&se=2017-02-01T00:55:34Z&st=2016-12-29T16:55:34Z***************';
-- NOTE: Make sure that you don't have a leading ? in SAS token, and
-- that you have at least read permission on the object that should be loaded srt=o&sp=r, and
-- that expiration period is valid (all dates are in UTC time)
CREATE EXTERNAL DATA SOURCE MyAzureBlobStorage
WITH ( TYPE = BLOB_STORAGE,
LOCATION = 'https://****************.blob.core.windows.net/invoices'
, CREDENTIAL= MyAzureBlobStorageCredential --> CREDENTIAL is not required if a blob is configured for public (anonymous) access!
);
BULK INSERT Sales.Invoices
FROM 'inv-2017-12-08.csv'
WITH (DATA_SOURCE = 'MyAzureBlobStorage');
```

### SQL Server Integration Services (SSIS)

SSIS packages are one of the most popular methods today for Extract, Transform, and Load (ETL) applications with SQL Server.

Azure SQL Managed Instance and Database can always be targets for data loading with SSIS packages no matter where you run the SSIS package with the SSIS runtime. This means you can build SSIS packages and run them on-premises in your data center or in Azure Virtual Machine. Here is a simple tutorial on how to use SSIS to load data into Azure SQL Database: [`https://learn.microsoft.com/sql/integration-services/load-data-to-sql-database-with-ssis`](https://learn.microsoft.com/sql/integration-services/load-data-to-sql-database-with-ssis).

There are other methods to execute an SSIS package in Azure SQL itself (which you could use to have packages that import data into Azure SQL Managed Instance or Database).

#### Azure SSIS

There is no specific service called *Azure SSIS*, but I have coined this term to refer to services in Azure to host and execute SSIS packages. There are two components to make Azure SSIS a reality:

*   **Azure SSIS Integration Runtime (SSIS IR)**

    Azure Data Factory (ADF) is a service in Azure that provides ETL capabilities. ADF supports an *integration runtime* compute environment. One of the choices for an integration runtime environment is SSIS. Effectively, ADF will host compute nodes that allow you to run SSIS packages. This compute environment can also be connected to a virtual network paving the way for connectivity to Azure SQL resources. The steps to deploy SSIS IR can be found at [`https://learn.microsoft.com/azure/data-factory/create-azure-ssis-integration-runtime`](https://learn.microsoft.com/azure/data-factory/create-azure-ssis-integration-runtime).

*   **SSIS Catalog Database (SSISDB) in Azure**

    Although not required to use SSIS IR, you can host the catalog database for SSIS called *SSISDB* in Azure SQL Database or Managed Instance. This way, you can execute packages that are stored in Azure SQL Database or Managed Instance leaving all your ETL packages, managed, and execution in the cloud. These packages can access any data source a normal SSIS package could include on-premises. Think of this as a way of executing everything in Azure and *pulling* data from on-premises. The easiest method to create the SSISDB catalog in Azure SQL is to provision it when deploying SSIS IR. You will need to have already deployed an Azure SQL Database Logical Server or Managed Instance to deploy the SSISDB in Azure. Check out this tutorial at [`https://learn.microsoft.com/azure/data-factory/tutorial-deploy-ssis-packages-azure`](https://learn.microsoft.com/azure/data-factory/tutorial-deploy-ssis-packages-azure).

    Once you have deployed SSIS IR and SSISDB for Azure, you basically have a compute structure to run packages and a catalog to host them (without you having to provision your own VM).

Note

You still can execute packages stored in a file system such as Azure Files.

You now have the option to execute these packages using several methods:

*   Execute packages with **SQL Server Data Tools (SSDT)**. Learn more at [`https://learn.microsoft.com/azure/data-factory/how-to-invoke-ssis-package-ssdt`](https://learn.microsoft.com/azure/data-factory/how-to-invoke-ssis-package-ssdt).

*   Execute packages using SQL Server Agent on a Managed Instance. Learn more at [`https://learn.microsoft.com/azure/data-factory/how-to-invoke-ssis-package-managed-instance-agent`](https://learn.microsoft.com/azure/data-factory/how-to-invoke-ssis-package-managed-instance-agent).

*   Execute packages with the Azure-enabled version of dtexec called **AzureDTExec** (Windows only). Learn more at [`https://learn.microsoft.com/azure/data-factory/how-to-invoke-ssis-package-azure-enabled-dtexec`](https://learn.microsoft.com/azure/data-factory/how-to-invoke-ssis-package-azure-enabled-dtexec).

*   Execute an **Execute SSIS Package activity** as part of an Azure Data Factory (ADF) pipeline. Learn more at [`https://learn.microsoft.com/azure/data-factory/how-to-invoke-ssis-package-ssis-activity`](https://learn.microsoft.com/azure/data-factory/how-to-invoke-ssis-package-ssis-activity). You can also use ADF to run a stored procedure activity. Read more at [`https://learn.microsoft.com/azure/data-factory/how-to-invoke-ssis-package-stored-procedure-activity`](https://learn.microsoft.com/azure/data-factory/how-to-invoke-ssis-package-stored-procedure-activity).

Using SSIS IR and SSISDB in Azure could be a chapter in itself! To find out more details about other options, including migrating existing SSIS packages to Azure, read the following documentation at [`https://learn.microsoft.com/sql/integration-services/lift-shift/ssis-azure-lift-shift-ssis-packages-overview`](https://learn.microsoft.com/sql/integration-services/lift-shift/ssis-azure-lift-shift-ssis-packages-overview).

### BACPAC

A BACPAC file (bacpac extension) is a file that includes schema and data for a database. You can use a BACPAC file generated through tools like SSMS, SqlPackage, or PowerShell to import into Azure SQL. The import process in this case *creates a full database* with the schema and data in the file. You can store your BACPAC file in Azure Storage or in a local file depending on what tool and where you are importing the data.

Azure SQL Database supports importing to a new database using the Azure Portal (the BACPAC file would be in Azure Storage), the SqlPackage tools, SSMS, or PowerShell. Figure [5-6](#496204_2_En_5_Chapter.xhtml#Fig6) shows the option from the Command Bar of a logical server to import a database with a BACPAC file.

![](images/496204_2_En_5_Chapter/496204_2_En_5_Fig6_HTML.jpg)

Figure 5-6

Importing a database from the Azure Portal

For Azure SQL Managed Instance, you can use SSMS or SqlPackage, but you can also do a T-SQL RESTORE of a backup. BACPAC files are limited in 200GB, so if you need larger sizes you will need to use bulk copy options. A full database migration can be done using the Azure Database Migration Service.

To read more about using BACPAC files to import a new database in Azure SQL, consult our docs at [`https://learn.microsoft.com/azure/azure-sql/database/database-import`](https://learn.microsoft.com/azure/azure-sql/database/database-import).

### Database Copy

Azure SQL Database supports the ability to create a new database by making a transactional consistent copy from another deployed database. You can use the Azure Portal, PowerShell, az CLI, or the T-SQL CREATE DATABASE statement. One nice aspect to this capability is that you can copy a database to a different logical server. The portal also allows you to configure the deployment option for the target database.

Figure [5-7](#496204_2_En_5_Chapter.xhtml#Fig7) shows that the copy database capability can be accessed from the Command Bar in the Azure Portal from the context of an existing database.

![](images/496204_2_En_5_Chapter/496204_2_En_5_Fig7_HTML.jpg)

Figure 5-7

Copying a database using the Azure Portal

This is the same as using the CREATE DATABASE…AS COPY option I mentioned earlier in the chapter. You can read more about how to copy a database for Azure SQL Database at [`https://learn.microsoft.com/azure/azure-sql/database/database-copy`](https://learn.microsoft.com/azure/azure-sql/database/database-copy).

### RESTORE to Managed Instance

Since Azure SQL Managed Instance is a full SQL Server instance with almost all surface area including T-SQL available to you, you can perform a *native* restore of a SQL Server backup to the instance.

This means you can back up a database from a SQL Server, copy the backup file to Azure Storage, and then restore the database using the RESTORE DATABASE statement to the Azure Managed Instance.

At this time, only a full database backup can be restored. There is a key difference in RESTORE with Managed Instance from SQL Server. A RESTORE is an asynchronous operation. You can drop the connection and the restore option is done in the background. You can use the DMVs **sys.dm_operation_status** to check on the deployment.

To learn more about how to use RESTORE with Azure SQL Managed Instance check out documentation at [`https://learn.microsoft.com/sql/t-sql/statements/restore-statements-transact-sql`](https://learn.microsoft.com/sql/t-sql/statements/restore-statements-transact-sql).

### Spark Connector

Spark is a technology that can often be used for ETL operations. Microsoft supports a Spark connector that allows you to export and import data to and from Azure SQL Database and Managed Instance. It supports bulk operations that can be very fast and even use Microsoft Entra authentication.

Anywhere you can run Spark and connect to Azure SQL, you can use this connector. Learn more at [`https://learn.microsoft.com/azure/azure-sql/database/spark-connector`](https://learn.microsoft.com/azure/azure-sql/database/spark-connector). Get examples on GitHub at [`https://github.com/microsoft/sql-spark-connector`](https://github.com/microsoft/sql-spark-connector).

### Azure Data Factory (ADF)

As mentioned earlier in the chapter, Azure Data Factory (ADF) is a cloud service built for data integration. You can build *pipelines* to orchestrate data integration activities much like SSIS. ADF can be as simple or complex as you want or need. You can see a simple example of using ADF to copy data from Azure Blob Storage to Azure SQL Database at [`https://learn.microsoft.com/en-us/azure/data-factory/tutorial-copy-data-portal`](https://learn.microsoft.com/en-us/azure/data-factory/tutorial-copy-data-portal).

While ADF uses compute integration runtime environments for execution, you should think of ADF as *PaaS service for data integration* (it includes an SLA). The ADF team likes to think of their service as **Code-free ETL as a Service**.

If you do not have a lot of investment in SSIS packages already and need to perform ETL or just data movement operations in Azure on a consistent basis, I highly encourage you to consider ADF. Start by reading the introduction at [`https://learn.microsoft.com/azure/data-factory/introduction`](https://learn.microsoft.com/azure/data-factory/introduction).

## Updating Azure SQL

I have mentioned in this book that Azure SQL Managed Instance and Azure SQL Database are *versionless*. Versionless means that we do not release major versions of SQL Server in Azure SQL and then let you adopt them. We are continuously updating the software that powers Azure SQL to give you the latest updates and enhancements. **You are completely unburdened from keeping up and updating both the operating system and SQL Server.**

### Maintenance of Azure SQL

But what exactly do the latest updates mean in comparison to SQL Server? For SQL Server (as of SQL Server 2017), we release major versions, Cumulative Updates (CU), and General Distribution Releases (GDR). Customers download and apply the updates they would like to use.

When we release a new major version, for example, SQL Server 2019, it includes new features and capabilities, but it also includes a series of bug fixes that includes fixes from all CU and GDR builds from the previous major release plus a set of fixes we believe makes sense to include in the major release.

For bug fixes (and some minor enhancements that are not necessarily features), we are constantly keeping these fixes up to date in our main branch of the source code. These changes are pushed into Azure SQL on a frequent basis and often available earlier to our customers vs. having to wait for a CU, GDR, or even major version release. This is a major advantage in running in Azure. We are constantly testing these fixes and changes but effectively you are getting a constant flow of bug fixes only found in a major version release.

We do not document or publish how often we roll out updates to SQL Server and other components that power Azure SQL. The process of rolling these updates is called a *train*. A train contains updates to all the components that support Azure SQL, including OS, SQL Server, Service Fabric components, and other software we use to power Azure SQL. We do not roll out trains to every Azure Region at one time. Rather, these trains roll out across regions. If we detect any issue in rolling out a train in an early phase, we can easily roll it back. Since every Azure SQL deployment has built-in availability, when we update nodes and instances, we can fail over the deployment and ensure your data is available by the SLA agreement associated with your deployment. Concepts like Availability Zones also help provide further availability. I will discuss more about Availability Zones in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book. Resource Health options in the Azure Portal or through REST APIs can provide you with information about whether a failover occurred because of *deployments*. Deployments are planned maintenance events that affect availability.

Note

It is possible the timing of a Cumulative Update for SQL Server may result in a fix landing in a CU before it rolls out in a train for Azure SQL. Unless the fix in the CU has a regression, that fix will make it eventually in a deployment train.

As you have seen in this book, you also have options on maintenance windows to control when updates are scheduled and alerts to notify you in advance of these maintenance operations that could affect uptime.

Note

If we have a major impacting customer problem, we may notify a specific customer that their deployment may be affected, so we can correct their problem.

Part of the improvements we are working on will provide advance maintenance notification to customers, so they can plan for these events. Furthermore, we are also investigating the ability for customers to select a custom schedule for a maintenance event.

Our work in the cloud often generates innovation. One very cool capability we have rolled out to reduce the number of scenarios where maintenance of Azure SQL requires a restart of SQL Server (and therefore a failover) is called **hot patching**. You can read more about how hot patching improves availability in Azure SQL in this blog post by my colleague Hans Olav Norheim at [`https://azure.microsoft.com/blog/hot-patching-sql-server-engine-in-azure-sql-database/`](https://azure.microsoft.com/blog/hot-patching-sql-server-engine-in-azure-sql-database/).

### Update Policy for Azure SQL Managed Instance

I’ve described in the book the concept of an update policy for Azure SQL Managed Instance with two choices:

*   Always up to date

    This is the possibility to receive continuous updates to Managed Instance, including new features. Any database you back up for this choice cannot be restored to SQL Server 2022 (but can be restored to other always-up-to-date managed instances).

*   SQL Server 2022

    This is the choice to use if you want to be able to restore a backup to SQL Server 2022 or use online disaster recovery with SQL Server 2022 (which is discussed further in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book). With this choice, your instance will still have cumulative updates automatically applied as they are available for SQL Server 2022 but not new features that are only available for an always-up-to-date instance. You can restore a backup from this Managed Instance to another SQL Server 2022 or always-up-to-date Managed Instance (when you use this latter choice, you cannot restore back).

### New Features and Capabilities in Azure SQL

In general, over the last few years, we have tried to adopt a *cloud-first mentality* for SQL Server. We build a new capability and roll it out in Azure SQL first through a private (limited customers and usually requires a sign-up) and public preview (not fully generally available, but customers can try it out) program. Eventually, this capability goes into General Availability (GA).

Then based on the timing of a new major version of SQL Server, that capability is included in that major version. A great example of this is Intelligent Query Processing (IQP), which you can read about at [`https://learn.microsoft.com/sql/relational-databases/performance/intelligent-query-processing`](https://learn.microsoft.com/sql/relational-databases/performance/intelligent-query-processing). We rolled out IQP for Azure SQL customers before it was released in SQL Server 2019 and 2022.

I’ve created two shortcuts for you to keep up to date with all the latest news for Azure SQL Database and Managed Instance:

[`https://aka.ms/whatsnewsqldb`](https://aka.ms/whatsnewsqldb)

[`https://aka.ms/whatsnewsqlmi`](https://aka.ms/whatsnewsqlmi)

## Summary

In this chapter, you learned the various methods, capabilities, techniques, and restrictions for configuring Azure SQL Managed Instance and Database after you deploy. You learned both configuration options specific to Azure SQL and ones for the database engine and databases using familiar techniques like T-SQL.

You also learned some interesting aspects of space management which can be important as you manage the database in Azure SQL.

In this chapter, we covered the various methods to load data from both on-premises and in Azure to Azure SQL. Finally, you learned the details of how Azure SQL is updated and why it is referred to as a versionless database.

Now that you have learned how to deploy and configure Azure SQL, it is time to jump into the first category of the *meat and potatoes* of Azure SQL: security. In the next chapter, we will cover the security capabilities for Azure SQL as it compares to SQL Server.

# 6. Securing Azure SQL

Now that you have deployed and configured your Azure SQL Managed Instance or Database, you will want to ensure you have done all the right things to fully secure your Azure SQL deployment. Azure SQL has all the capabilities that come with SQL Server for security plus more.

My colleague Anna Hoffman asked me why I keep calling security, performance, and availability the *meat and potatoes* of SQL Server and Azure SQL. I can’t take credit. This phrase comes from my longtime colleague, the famous Conor Cunningham. Conor and I have together presented several demos at the PASS Summit Keynote. One time I was thinking of doing some pretty leading edge demos, and Conor stopped me and said “Bob, who is nice, but our customers expect you and I to showcase the core innovation of the engine. Things like security, performance, and availability. You know, the meat and potatoes of SQL Server.” The phrase has stuck ever since. Oh, by the way, I still also look to build those leading edge demos.

Note

For those who don’t know the phrase, it is one we use in Texas to mean something fundamental or core. A basic meal consists of meat and potatoes. Not a problem for me since I love both.

In this chapter, we will explore all the capabilities that you normally use to secure a SQL Server and compare it with Azure SQL. You will also learn about unique capabilities and methods you will use to secure your Azure SQL Managed Instance and Database deployments.

This chapter is organized into the following categories for security:

**Networking Security**

In this section, learn the several ways to provide the most secure connections over various network options, including firewalls, virtual networks, and private endpoints.

**Authentication and Access**

In this section, you will learn the most secure methods to provide authentication and access, including Azure Role-Based Access Control (RBAC), Microsoft Entra, Row-Level Security, and Microsoft Purview.

**Data Protection**

In this section, you will learn how to protect your data, including encryption methods, data masking, and Ledger (a blockchain inside of SQL).

**Security Management**

In this section, you will learn about concepts like auditing, Microsoft Defender, and data classification.

I will use the deployments I did in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book when discussing security. For you to try out any of the techniques or commands I use in this chapter, you will need

*   An Azure subscription.

*   A minimum of Contributor role access to the Azure subscription. You can read more about Azure built-in roles at [`https://learn.microsoft.com/azure/role-based-access-control/built-in-roles`](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles).

*   Access to the Azure Portal.

*   The deployment of an Azure SQL Managed Instance and an Azure SQL Database similar as I did in Chapter [4](#496204_2_En_4_Chapter.xhtml).

*   To connect to Managed Instance, you will need a *jumpbox* or virtual machine in Azure to connect. I showed you how to do this in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book. One simple way to do this is to create a new Azure Virtual Machine and deploy it to the same virtual network as the Managed Instance (you will use a different subnet than the Managed Instance).

*   To connect to Azure SQL Database, you can use your on-premises client using firewall rules, but this chapter also shows you how to connect using a private endpoint in a virtual network (and with a Microsoft Entra managed identity), so you will need an Azure Virtual Machine. For my example, I will use the virtual machine I created in Chapter [3](#496204_2_En_3_Chapter.xhtml) of the book, called **bwsql2022**.

*   Installation of the **az** CLI (see [`https://learn.microsoft.com/cli/azure/install-azure-cli`](https://learn.microsoft.com/cli/azure/install-azure-cli) for more details). You can also use the Azure Cloud Shell instead since az is already installed. You can read more about the Azure Cloud Shell at [`https://azure.microsoft.com/features/cloud-shell/`](https://azure.microsoft.com/features/cloud-shell/).

*   You will run some T-SQL in this chapter, so install a tool like SQL Server Management Studio (SSMS) at [`https://aka.ms/ssms`](https://aka.ms/ssms).

## Network Security

Most administrators who install SQL Server use a private network within a company infrastructure. Firewalls block incoming traffic within this network. In addition, operating systems provide firewalls to protect ports for applications like SQL Server. In fact, if you have installed SQL Server before, you know that by default the firewall for port 1433 is blocked for both Windows and Linux. You generally must take action to add an exception for this port for remote connectivity to SQL Server.

Azure SQL is no different except that you have options to allow connectivity to the SQL Server instance behind the scenes as a public endpoint on the Internet or private within a virtual network.

I will admit to you as you review the information in this section of the chapter that you should absolutely consult networking experts in your organization to configure Azure SQL Managed Instance or Database for the requirements you need.

### Azure SQL Managed Instance Network Security

As you saw in Chapter [4](#496204_2_En_4_Chapter.xhtml) to deploy Azure SQL Managed Instance, a virtual network, a public endpoint, and private connectivity are baked into the Managed Instance experience.

Figure [6-1](#496204_2_En_6_Chapter.xhtml#Fig1) (this comes right from the documentation at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/connectivity-architecture-overview?view=azuresql#communication-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/connectivity-architecture-overview?view=azuresql#communication-overview)) is a great visualization of the options to connect.

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig1_HTML.jpg)

Figure 6-1

Networking options for Azure SQL Managed Instance

Let’s explore each of these options in more detail.

#### Public Endpoint

You can enable a public endpoint for an Azure SQL Managed Instance, but it is the least secure. The big advantage of this method is that any SQL client can connect to the endpoint with little configuration required. If you decide to do this, the endpoint is on port 3342, which cannot be changed, not 1433. Additionally, you can use Network Security Group (NSG) rules to effectively set up a firewall on the port. Read more at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/public-endpoint-configure`](https://learn.microsoft.com/azure/azure-sql/managed-instance/public-endpoint-configure).

#### VNet-Local Endpoint

This is the most common option to connect to Managed Instance. I showed you one method I prefer is to set up an Azure Virtual Machine (or it could be a cloud application) in the same virtual network as the Managed Instance but in a different subnet. This was the example I used in Chapter [4](#496204_2_En_4_Chapter.xhtml) to use a jumpbox.

However, it is also possible to connect using a different virtual network, inside or outside of Azure. **VNet peering** allows you to connect from a resource in one virtual network that is peered to another. Check out this blog post for instructions to connect to a Managed Instance with VNet peering: [`https://techcommunity.microsoft.com/t5/azure-database-support-blog/connect-to-azure-sql-database-managed-instance-with-virtual/ba-p/369077`](https://techcommunity.microsoft.com/t5/azure-database-support-blog/connect-to-azure-sql-database-managed-instance-with-virtual/ba-p/369077). You can read more about virtual network peering at [`https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview`](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview). Virtual network peering can occur within a region or across regions.

If you would like to connect to the Managed Instance from on-premises, you will need to use an **Azure Virtual Network Gateway**. There are several options to connect your on-premises environment with a virtual network gateway: Point-to-Site (P2S), Site-to-Site (S2S), and ExpressRoute. ExpressRoute is by far the fastest (but the most expensive) way to connect with a gateway. You can learn about all of these gateway options at [`https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-vpngateways`](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-vpngateways). As I’ve said earlier in this chapter, if networking is not your area of expertise (it is not mine), then I would consult networking engineers in your organization on the best option.

#### Private Endpoint

The most secure option to connect is a private endpoint. Private endpoints establish secure, isolated connectivity between a service and multiple virtual networks without exposing your service’s entire network infrastructure. I like the documentation’s analogy when it says “…is like extending a physical network cable from a computer running Azure SQL Managed Instance to another virtual network.”

There is a bit more configuration to set this up, and there are some limitations such as you cannot use this for failover groups, distributed transactions, and Managed Instance link.

You can set up a private link using the Private endpoint connections option in the Service Menu under Security in the context of the Azure SQL Managed Instance in the Azure Portal. See the complete steps at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/private-endpoint-overview?view=azuresql&tabs=separate-vnets#create-a-private-endpoint-in-a-virtual-network`](https://learn.microsoft.com/azure/azure-sql/managed-instance/private-endpoint-overview?view=azuresql&tabs=separate-vnets#create-a-private-endpoint-in-a-virtual-network).

I’ll walk through a detailed example of how to use a private endpoint in the next section of the chapter for Azure SQL Database.

#### Other Considerations

It is important to know that besides the SQL Server endpoint (the standard TCP port to connect and run queries), an Azure SQL Managed Instance has a *management endpoint*. Since a Managed Instance is deployed in its own virtual cluster, various services outside the cluster but within Azure (such as deployment within Resource Manager) must be able to access the cluster. That access is through the management endpoint. The management endpoint is a public endpoint protected by firewalls. This means when you use the portal or CLIs to manage the Managed Instance (e.g., scaling operations), you are connecting to this endpoint. Read more about how the management endpoint is protected at [`https://docs.microsoft.com/en-us/azure/azure-sql/managed-instance/connectivity-architecture-overview#management-endpoint`](https://docs.microsoft.com/en-us/azure/azure-sql/managed-instance/connectivity-architecture-overview#management-endpoint).

I showed during deployment in Chapter [4](#496204_2_En_4_Chapter.xhtml) the connection type of Proxy vs. Redirect. Even though the SQL Server endpoint is in a private virtual network, technically a Proxy connection is more secure because all traffic is routed through a *gateway*. With a redirect connection, the connection is first made to the gateway, and then all subsequent traffic goes directly to the Managed Instance node. Redirect can be much faster, and since a private endpoint is being used, I recommend this option. Any use of a public endpoint always uses a proxy connection. You can learn more about these policies at [`https://docs.microsoft.com/en-us/azure/azure-sql/database/connectivity-architecture#connection-policy`](https://docs.microsoft.com/en-us/azure/azure-sql/database/connectivity-architecture#connection-policy).

### Azure SQL Database Network Security

Network security for Azure SQL Database (any deployment option) is a bit different than Managed Instance because when you deploy a database, we do not have a dedicated private virtual cluster. Rather, all database deployments share virtual clusters (rings) in Azure regions. Remember that when I talk about these network security options, they apply to the logical server for all databases. The only exception to this rule is that you can configure firewall rules specific to a database different than the logical server.

This doesn’t mean you can’t be protected because you can have a private endpoint to your database deployment as you will see in this section of the chapter.

#### Using the Public Endpoint

In Chapter [4](#496204_2_En_4_Chapter.xhtml) for deployment, I showed you some of the connection options for a database deployment, which includes

*   **Allow access to Azure services** – This option allows any Azure resource (e.g., VM, application, or cloud shell) to access the public endpoint for the database deployment.

*   **Firewall rules** – This option allows you to create specific firewall rules for client computers outside of Azure. I used this technique in Chapter [4](#496204_2_En_4_Chapter.xhtml) to connect to the logical server I deployed with my laptop and SQL Server Management Studio (SSMS).

Figure [6-2](#496204_2_En_6_Chapter.xhtml#Fig2) shows a network connectivity diagram of how both Azure resource and on-premises computers can be connected to the logical server with a public endpoint.

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig2_HTML.jpg)

Figure 6-2

Connecting to a logical server with a public endpoint

You can see in this figure Azure VM resources that are connected through the Allow access to Azure services and on-premises computers connecting through a firewall rule. Notice the IP addresses of the Azure Virtual Machines are using their public IP address because even they are using a public endpoint connection within Azure.

Note

To be clear, you could use Allow access to Azure services and turn OFF all other firewall rules. While this is not a private endpoint scenario, it does prevent any connection unless it comes *within* Azure to connect to the logical server.

Notice in this figure the name of the logical server is [mysqldbsrv.​database.​windows.​net](http://mysqldbsrv.database.windows.net) (this is from an example that is not the logical server I used). That is the name of the logical server, but how does the public endpoint get resolved on the Internet? Notice underneath this name in the figure is a public IP address and a network name of [westus1-a.​control.​database.​windows.​net](http://westus1-a.control.database.windows.net). This name is part of the DNS name of the node for the gateways when connecting to the logical server.

Since Allow access to Azure services is enabled, let’s use the Azure Virtual Machine I created in Chapter [3](#496204_2_En_3_Chapter.xhtml), **bwsql2022**, to examine connectivity properties to the logical server I deployed in Chapter [4](#496204_2_En_4_Chapter.xhtml).

I used RDP to connect into the virtual machine (I noted earlier in the chapter I installed SSMS and Azure Data Studio in the VM). I then connected using SSMS to the Azure Logical Server I deployed in Chapter [4](#496204_2_En_4_Chapter.xhtml) called [**bwsqllogicalserver.database.windows.net**](http://bwsqllogicalserver.database.windows.net). **dm_exec_connections** is a DMV for SQL Server that can provide key information about connections to the server. Therefore, I ran the following T-SQL statement from SSMS:

```
SELECT client_net_address FROM sys.dm_exec_connections
WHERE session_id = @@SPID;
```

And the returned result is the following:

```
52.173.38.9
```

This IP address is the *public IP address* of the Azure Virtual Machine, bwsql2012\. This proves that the VM is connecting to the logical server over a public endpoint. However, the VM has access (without allowing the connection through a firewall) because I used the option **Allow access to Azure services**.

Another interesting way to look at the public endpoint aspect to Azure SQL Database is to examine the DNS infrastructure of the logical server. You can use the **nslookup** command to do this (nslookup is available by default on Windows and Linux operating systems; see more at [`https://learn.microsoft.com/windows-server/administration/windows-commands/nslookup`](https://learn.microsoft.com/windows-server/administration/windows-commands/nslookup)).

From my Azure Virtual Machine, I then ran nslookup from PowerShell like the following:

```
nslookup bwsqllogical.database.windows.net
```

I received the following results:

```
Server:  UnKnown
Address:  168.63.129.16
Non-authoritative answer:
Name:    cr8.cEntralus1-a.control.database.windows.net
Address:  13.89.169.20
Aliases:  bwsqllogicalserver.database.windows.net
dataslice7.cEntralus.database.windows.net
dataslice7cEntralus.trafficmanager.net
```

The top result is IP address 168.63.129.16\. It turns out this address is a special virtual IP address used for Azure communications, so within Azure your address will always say this (see more information at [`https://learn.microsoft.com/en-us/azure/virtual-network/what-is-ip-address-168-63-129-16`](https://learn.microsoft.com/en-us/azure/virtual-network/what-is-ip-address-168-63-129-16)).

The results at the bottom show the DNS hierarchy of the logical server which includes a DNS server within the *control ring* (gateways) of Azure.

Also, note that a ping is blocked, but it shows how a public endpoint is attempted to be accessed:

```
ping bwsqllogicalserver.database.windows.net
Pinging cr8.cEntralus1-a.control.database.windows.net [13.89.169.20] with 32 bytes of data:
Request timed out
```

There is a third option to secure the connectivity to access the logical server. Let’s say you want to turn off Allow access to Azure services but don’t want to have to use a fixed IP address for a firewall rule. You can use a **virtual network service endpoint** to allow only specific Azure sources in a virtual network (which could include on-premises connections) to connect to the logical server. This is still a public endpoint connection but strictly limited to resources from a specific Azure virtual network. Read more about how to use a virtual network service endpoint at [`https://learn.microsoft.com/azure/azure-sql/database/vnet-service-endpoint-rule-overview`](https://learn.microsoft.com/azure/azure-sql/database/vnet-service-endpoint-rule-overview).

Let’s use a different technique to tighten up the security of the network connectivity to the Azure logical server, a private link.

#### Using Private Link

Let’s say you do not want to allow any public endpoint access to your Azure SQL Databases regardless of whether connections come from within or outside of Azure. The Azure team has created a concept called **private link** to allow PaaS services like Azure SQL Database to restrict access only through a private endpoint. You can read an overview about private link at [`https://learn.microsoft.com/azure/private-link/private-link-overview`](https://learn.microsoft.com/azure/private-link/private-link-overview).

Let’s look at a new variation of Figure [6-2](#496204_2_En_6_Chapter.xhtml#Fig2) using a private link. Figure [6-3](#496204_2_En_6_Chapter.xhtml#Fig3) shows how a private link provides a private endpoint for Azure SQL Database.

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig3_HTML.jpg)

Figure 6-3

Private link with Azure SQL Database

A private link will expose a private endpoint in an existing Azure virtual network. Notice in this diagram the DNS name for the logical server is no longer in a public DNS hierarchy.

Let’s see how to implement a private link connection using the existing deployment of Azure SQL Database from Chapter [4](#496204_2_En_4_Chapter.xhtml) and the Azure portal. From my logical server (mine is called **bwsqllogicalserver**), I can create a private endpoint in the Azure Portal as you see in Figure [6-4](#496204_2_En_6_Chapter.xhtml#Fig4).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig4_HTML.jpg)

Figure 6-4

Creating a private endpoint for Azure SQL Database

You will be given a series of screens to give the endpoint a name, pick the virtual network (in this case, the virtual network is the one for the Azure VM **bwsql2022** I deployed in Chapter [3](#496204_2_En_3_Chapter.xhtml)), and DNS (which I just used the defaults). The deployment for this link will take a few minutes.

Note

The private endpoint must be in the same region as the Azure virtual network you choose. Remember the endpoint now becomes a resource in that VNet. However, your client connection could be in another virtual network using VNet peering or VNet gateways as seen in Figure [6-3](#496204_2_En_6_Chapter.xhtml#Fig3).

Figure [6-5](#496204_2_En_6_Chapter.xhtml#Fig5) shows my deployed private endpoint associated with this logical server.

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig5_HTML.jpg)

Figure 6-5

A deployed private endpoint for Azure SQL Database

The endpoint is its own resource and is associated both with the virtual network for my Azure VM and this logical server (it can be used with other logical servers as well). If you select the private endpoint, you can drill into the network interface and see the private IP address. To show how I can use just this endpoint, I can also use the Azure Portal to disable any public endpoint through the same networking option off the Service Menu for this logical server.

Now I’ll go back to my Azure VM bwsql2022 and try to connect again with SSMS. My connection now works. If I run the following T-SQL statement:

```
SELECT client_net_address FROM sys.dm_exec_connections
WHERE session_id = @@SPID;
```

the result is the following:

```
10.0.0.4
```

This is the ***private IP address*** (before it was the public IP address) of the Azure Virtual Machine bwsql2022\. Let’s try to ping the logical server this time:

```
ping bwsqllogicalserver.database.windows.net
Pinging bwsqllogicalserver.privatelink.database.windows.net [10.0.0.5] with 32 bytes of data:
Request timed out
```

Notice the private IP address of the server and the new DNS name which is not in the public hierarchy.

Also, note the output for nslookup:

```
nslookup bwsqllogical.database.windows.net
Server:  UnKnown
Address:  168.63.129.16
Non-authoritative answer:
Name:    bwsqllogicalserver.privatelink.database.windows.net
Address:  10.0.0.5
Aliases:  bwsqllogicalserver.database.windows.net
```

Read more in our documentation at using Azure Private Link for Azure SQL Database at [`https://learn.microsoft.com/azure/azure-sql/database/private-endpoint-overview`](https://learn.microsoft.com/azure/azure-sql/database/private-endpoint-overview). For anyone wanting to integrate their on-premises environment, pay special attention to this documentation page: [`https://learn.microsoft.com/azure/azure-sql/database/private-endpoint-overview?view=azuresql#on-premises-connectivity-over-private-peering`](https://learn.microsoft.com/azure/azure-sql/database/private-endpoint-overview?view=azuresql#on-premises-connectivity-over-private-peering).

Tip

Private link connections for Azure SQL Database currently only support Proxy connection types. We had left the connection policy for our Azure SQL Database as default, so if a connection uses a private link which is inside of Azure, it will not use Redirect but Proxy.

Go further with your knowledge of Azure SQL and Network Security by watching videos from Anna Hoffman and Rohit Nayak as part of the Data Exposed Channel at [`https://www.youtube.com/playlist?list=PL3EZ3A8mHh0xtbf4Cr2yR4-xsUtELwPjw`](https://www.youtube.com/playlist?list=PL3EZ3A8mHh0xtbf4Cr2yR4-xsUtELwPjw).

## Authentication and Access

You have now successfully deployed an Azure SQL Managed Instance and Database(s) and connected using a secure network architecture. When you deployed both Azure SQL services, you specified an *admin*, which is a SQL login and password and possibly a Microsoft Entra Admin. The next steps are to set up and configure other logins and users just like you would a SQL Server deployment. I call this process setting up *authentication*. Then you will want to grant access to users to the objects they need based on the requirements of your application and business.

Before we talk about the details of authentication and access for Azure SQL, let’s review first the authentication and access for Azure SQL resources *outside* of SQL but in the Azure infrastructure called Azure Role-Based Access Control (RBAC).

### Azure Role-Based Access Control (RBAC)

I’ve mentioned the concept of Azure RBAC in previous chapters in the book. Today when you deploy SQL Server on Windows or Linux, you must have certain rights and privileges to install SQL Server. For example, on Windows, most people use local administrator accounts when installing SQL Server.

At this point in the book, I’ve listed in several chapters the requirements to deploy Azure SQL such as the *Contributor role*. An Azure account user that is part of the Contributor role has the permissions to manage everything except grant access to resources to other accounts (that access is reserved for members of the Owner or User Access Administrator roles).

Therefore, if you are assigned the Contributor role for your Azure subscription, you should have the rights to deploy Azure SQL Managed Instances and Databases.

It is possible you want to set up a system for your organization so that some Azure users have the right to deploy or manage Azure SQL Managed Instances and Databases but not access the resources. Think of an administrator that only deploys or manages resources but doesn’t have access to the underlying SQL Server.

Azure provides the following built-in roles for these purposes.

#### SQL DB Contributor

Members of this role can deploy and manage Azure SQL Databases but not access them.

#### SQL Server Contributor

Members of this role can deploy and manage Azure SQL logical servers and databases but not access them.

#### SQL Security Manager

Members of this role can manage security policies of Azure SQL logical servers and databases but not access them.

#### SQL Managed Instance Contributor

Members of this role can deploy and manage Azure SQL Managed Instances but not access them.

You can read more about Azure built-in roles at [`https://learn.microsoft.com/azure/role-based-access-control/built-in-roles/databases`](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles/databases).

### Authentication for Azure SQL Managed Instance

When you deploy SQL Server on Windows, the default authentication *mode* is Windows Only. This means only Windows users can log in to SQL Server. **Mixed mode security** allows both SQL and Windows logins (Linux requires mixed mode). Azure SQL Managed Instance forces mixed mode security, and you cannot change this.

When you deploy an Azure SQL Managed Instance, you specify an **Administrator Account** as I showed you in Chapter [4](#496204_2_En_4_Chapter.xhtml). This can be a SQL Authentication account, Microsoft Entra account, or both. This account will be added as a member of the sysadmin role. The sa login is disabled by default, but you can enable and use it (but I don’t recommend it).

With this sysadmin login, you can add other SQL logins, assign them to roles (even sysadmin) just like SQL Server. You can also create users in the database and map them to logins just like SQL Server.

Note

You cannot change the SQL admin once you deploy. You can reset the password for the admin through the Azure portal, az CLI, or PowerShell.

#### Microsoft Entra Authentication

We provide the ability for you to add **Microsoft Entra logins** to your Managed Instance. When you use Windows or domain authentication with SQL Server, you are using behind the scenes Active Directory Domain Services. SQL Server on Linux even supports this concept. Azure provides the same type of service through Microsoft Entra.

The first step in using Microsoft Entra (Entra) authentication for a Managed Instance is to provision an administrator for the Managed Instance from an Entra user. To use Entra with Managed Instance, you first need to create an Entra directory. It is possible when you sign in to the Azure Portal you are already part of an Entra directory for your organization. For me, that is the case at Microsoft.

To set up an Entra admin for a Managed Instance, you must have “Administrator” rights for your Entra to grant read permissions. I don’t have these at Microsoft. The documentation shows the process to configure this Entra admin at [`https://learn.microsoft.com/azure/azure-sql/database/authentication-aad-configure`](https://learn.microsoft.com/azure/azure-sql/database/authentication-aad-configure). This covers both the portal and CLI options to configure an Entra admin for Managed Instance.

The new Entra admin will become a member of the sysadmin server role for the Managed Instance. Now you can use the T-SQL CREATE LOGIN statement to create new logins based on Entra users. The FROM EXTERNAL PROVIDER clause provides this capability. The documentation shows an example of this syntax at [`https://learn.microsoft.com/sql/t-sql/statements/create-login-transact-sql?view=azuresqldb-mi-current`](https://learn.microsoft.com/sql/t-sql/statements/create-login-transact-sql?view=azuresqldb-mi-current) like the following T-SQL statement:

```
CREATE LOGIN [bob@contoso.com] FROM EXTERNAL PROVIDER;
```

#### Windows Authentication

Many SQL applications use Windows Authentication to connect to SQL Server. We now have an option for those applications to still use Windows Authentication, so they will not have to change that authentication method with Azure SQL Managed Instance.

This process still uses Microsoft Entra behind the scenes along with service principals to help implement Kerberos authentication.

Windows Authentication for Microsoft Entra principals on managed instances is available for virtual machines joined to Active Directory, Microsoft Entra ID, or hybrid Microsoft Entra ID – a hybrid Microsoft Entra user identity exists both in Microsoft Entra ID and Active Directory and can access a managed instance in Azure using Microsoft Entra Kerberos.

The only issue you might run into with this method is that logins are still created in the engine with the CREATE LOGIN FROM EXTERNAL PROVIDER as you would with Microsoft Entra. Learn how to set this up at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/winauth-azuread-setup`](https://learn.microsoft.com/azure/azure-sql/managed-instance/winauth-azuread-setup).

Note

At the time of the writing of this book, a new feature has been released for preview called **Native Windows Principals**. I highly encourage you to take a look at this feature because it can provide a way for you to use the familiar CREATE LOGIN FROM WINDOWS T-SQL statement. This feature uses a new concept called an authentication metadata mode. Take a look at the details at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/native-windows-principals`](https://learn.microsoft.com/azure/azure-sql/managed-instance/native-windows-principals).

### Authentication for Azure SQL Database

When I showed you how to deploy an Azure SQL Database in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book, I supplied a **server admin login** through both SQL Authentication and Microsoft Entra. I used the SQL Authentication login to connect to the logical server. This admin account is a *server-level principal* for the logical server and is mapped as **dbo** in all databases.

Note

You cannot change the SQL Authentication admin once you deploy. You can reset the password for the admin through the Azure portal, az CLI, or PowerShell. You can change the Microsoft Entra admin after you deploy.

If you would like to create other logins that have admin capabilities (but not full server admin), you should use the CREATE LOGIN T-SQL statement to create standard SQL logins for the logical server in the context of the logical master database. You can then create a user in context of the logical master database and assign this user to two special roles for Azure SQL Database using ALTER ROLE:

*   **dbmanager** – Users assigned to this role can create and manage databases and will be mapped to the dbo of that database, so has full database owner permissions.

*   **loginmanager** – Users assigned to this role can create new logins in the context of the logical master but are not mapped to the dbo role of databases.

You can now use the standard process with SQL Server to create SQL logins and map them to users in any database they need access. You can assign users to roles and even create new roles just like SQL Server.

Note

One complexity with using logins is that when you choose a failover option like geo-replication, you must create the login on the secondary server manually.

#### Using Contained Users

You can also create contained database users that don’t require a login. This concept has been around for a while with SQL Server using contained databases. Azure SQL Database is in a way a contained database. Contained users are also called **user accounts**.

The CREATE USER T-SQL statement supports contained users using the WITH PASSWORD clause. One advantage of a contained user is that the information is stored in the database and therefore replicated as part of a geo-replication failover deployment.

#### Microsoft Entra Authentication

Like Managed Instance, Azure SQL Database supports Microsoft Entra authentication. Like using Windows Authentication for SQL Server, Entra authentication can be the most secure and best method to use with Azure SQL Database. You can create an Entra server admin (in addition to the SQL server admin you created during deployment) for the logical server. You can then create contained users based on an Entra account. You can even create users based on Entra groups.

One option you can set for authentication is Microsoft Entra authentication only as I’ve done here in Figure [6-6](#496204_2_En_6_Chapter.xhtml#Fig6) for the database I deployed.

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig6_HTML.jpg)

Figure 6-6

Microsoft Entra options for a logical server

This is very similar to using Windows Authentication only with SQL Server. Like SQL Authentication, I can create other logins and contained users using Microsoft Entra accounts. You should know that PowerShell also supports creating an Entra admin using the **Set-AzSqlServerActiveDirectoryAdministrator** cmdlet, and az CLI supports **az sql server ad-admin create**.

Connected as the Entra admin to this user database, I could then create a contained user based on another Entra account like the following (this is a hypothetical account in Microsoft Entra):

```
CREATE USER [troyward@microsoft.com] FROM EXTERNAL PROVIDER;
GO
```

I will then give this user access to read data by adding them to the db_datareader role:

```
ALTER ROLE db_datareader ADD MEMBER [troyward@microsoft.com];
GO
```

Tip

If you want to know the various connection string options to use Microsoft Entra authentication in your application, see this table in our documentation: [`https://learn.microsoft.com/en-us/sql/connect/ado-net/sql/azure-active-directory-authentication`](https://learn.microsoft.com/en-us/sql/connect/ado-net/sql/azure-active-directory-authentication). You will notice the legacy of the name Azure Active Directory still exists.

#### Managed Identities

One interesting option for using Microsoft Entra authentication is **managed identities**. A managed identity is an object in the Microsoft Entra directory that can “act” like a user but doesn’t require a password. Therefore, using this option to authenticate is often called *passwordless* authentication.

Let’s say, for example, you will use SSMS in an Azure Virtual Machine to connect to an Azure SQL Database. You can now create a managed identity in Entra and assign that identity to the virtual machine. You can also enable an Azure SQL Database with the same identity and create a login based on the identity.

Now SSMS can be used in the VM to log in with the identity without a password. In fact, any application can use a token to gain access to the identity-based login in the VM. This is because the identity is assigned to the boundary of the VM itself.

Figure [6-7](#496204_2_En_6_Chapter.xhtml#Fig7) shows a visualization of how this can work.

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig7_HTML.jpg)

Figure 6-7

Using a passwordless managed identity to connect to Azure SQL Database

Let me show how this works using my deployed Azure VM bwsql2022 and my logical server bwsqllogicalserver:

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig8_HTML.jpg)

Figure 6-8

Enabling a managed identity for a logical server

1.  I used the following documentation steps to create a user-assigned managed identity: [`https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities`](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities). I called this managed identity **bwadwidentity**.

    **Note** There is also a concept of a system-assigned managed identity which you can enable directly with a logical server. However, it is confined to use of just that server. I like user-defined managed identities because I can use them with all types of Azure resources.

2.  I can now enable this managed identity for my logical server as you see in Figure [6-8](#496204_2_En_6_Chapter.xhtml#Fig8) from the Azure Portal by selecting Add and then choosing my already created identity. I also had to make this the primary identity of the server and click Save.

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig9_HTML.jpg)

Figure 6-9

Assigning a managed identity to an Azure VM

1.  Now I need to create a login for this identity. What I’ll do instead is make this identity a user of a specific database. So, I’ll connect with SSMS using my Entra admin account and run the following T-SQL statement in the context of the bwhyperscale database:

    ```
    CREATE USER [bwadwidentity] FROM EXTERNAL PROVIDER;
    ALTER ROLE db_datareader ADD MEMBER [bwadwidentity];
    GO
    ```

2.  Now I need to assign the identity to my Azure Virtual Machine. Using the Azure Portal, I can navigate to the virtual machine called bwsql2022 and assign the identity as you see in Figure [6-9](#496204_2_En_6_Chapter.xhtml#Fig9).

Note

You can also assign a managed identity to an application such as Azure App Services to connect to the Azure SQL Database*.*

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig10_HTML.jpg)

Figure 6-10

Overview of a managed identity

1.  Now let’s try to connect with SSMS. The SSMS interface is trickier to use this method as you can’t just supply the identity name. Now find the bwidentity resource in Azure (you can just click the identity name like in Figure [6-9](#496204_2_En_6_Chapter.xhtml#Fig9)). You should see on the Overview screen something like Figure [6-10](#496204_2_En_6_Chapter.xhtml#Fig10).

You need to copy the value of the client ID. That is what you will use to log in with SSMS.

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig11_HTML.jpg)

Figure 6-11

Connect with a managed identity in SSMS

1.  Inside the VM, launch SSMS and select your connection like the one in Figure [6-11](#496204_2_En_6_Chapter.xhtml#Fig11).

    The user-assigned identity is the client ID of the identity from the previous step. **Before you hit connect**, you must select the Connection Properties tab and put in your database name (mine is bwhyperscale) in the Connect to database field.

    You can now just use this like any other Azure SQL Database. The same concept applies for Azure SQL Managed Instance except you would be creating a login for the instance.

    Application developers don’t have to use this special ID. They have APIs from Azure to obtain tokens based on the identity assignment to the application or virtual machine.

### Set Up and Configure Access

Now that you have created logins and users, what do you do now? You do what you do for SQL Server. You grant access and permissions to objects within your database to meet your application requirements.

This could involve creating schemas, roles, and grant or revoking specific permissions. To get a primer on permissions for SQL Server, refer to the docs at [`https://learn.microsoft.com/sql/relational-databases/security/permissions-database-engine`](https://learn.microsoft.com/sql/relational-databases/security/permissions-database-engine).

One of the nice additions to SQL access since the first edition of this book is a set of more *granular* server roles. These new fixed server roles have a specific naming convention of **##MS_<role>##**. They allow you to use the concept of *least privilege* instead of having to overprovision a user to a role. For example, the role **##MS_ServerStateReader##** has permissions to view system catalog states from the logical server or any database but has no other permissions than this. You can see all the new fixed server roles at [`https://learn.microsoft.com/azure/azure-sql/database/security-server-roles`](https://learn.microsoft.com/azure/azure-sql/database/security-server-roles).

Don’t forget that Row-Level Security (RLS) is supported in Azure SQL just like SQL Server. Read about RLS at [`https://learn.microsoft.com/sql/relational-databases/security/row-level-security`](https://learn.microsoft.com/sql/relational-databases/security/row-level-security). You can read more about authentication and access for Azure SQL Database at [`https://learn.microsoft.com/azure/azure-sql/database/logins-create-manage`](https://learn.microsoft.com/azure/azure-sql/database/logins-create-manage).

### Microsoft Purview

Consider this scenario. You are trying to set up access for logins and access to data across multiple Managed Instances or databases at *scale*. It would be nice to use a central place to control and manage this access. Along come **Microsoft Purview DevOps** policies to help address this scenario.

Consider this situation. You need to have someone in your organization lead all your performance monitoring across many of your Azure SQL Databases and across many logical servers. Instead of creating a separate login and granting them access to perform this task (remember we have new server fixed roles for this), you can enable Purview to govern your logical server(s) and then set up a policy for which Microsoft Entra accounts will have access to a Performance Monitoring role (we also support a SQL Security Audit role) *in one place*.

Now these Microsoft Entra accounts will be able to log in to these logical servers and perform the tasks of “performance monitoring.” The actual permissions of this role can be found at [`https://learn.microsoft.com/purview/how-to-policies-devops-authoring-generic#role-definition-detail`](https://learn.microsoft.com/purview/how-to-policies-devops-authoring-generic#role-definition-detail). And if you want to remove this access, you can do this in Purview through the policy vs. having to go to each logical server. Read about how to get started at [`https://learn.microsoft.com/purview/how-to-policies-devops-azure-sql-db`](https://learn.microsoft.com/purview/how-to-policies-devops-azure-sql-db).

Note

Purview DevOps policies are also supported for Azure SQL Managed Instance and SQL Server 2022.

## Data Protection

Ensuring you have set up proper authorization to connect and access data is just the first step. You need to protect your data for all aspects of your deployment, including connections, data at rest, data end-to-end, and ensuring only the right people can view important data. Azure SQL has all the capabilities to protect data just like SQL Server.

### Encrypting Connections

Like SQL Server, Azure SQL supports encryption of connections through the Transparent Layer Security (TLS) protocol (you can read about TLS at [`https://en.wikipedia.org/wiki/Transport_Layer_Security`](https://en.wikipedia.org/wiki/Transport_Layer_Security)).

By default, Azure SQL Managed Instance enforces encryption for connections. Tools and applications should enable an encrypted connection for a Managed Instance to avoid client/server negotiation. In addition, you can force a minimal TLS version for a client connection. The latest TLS version, 1.2, fixes some known security vulnerability, so you may consider requiring this version. You can set the minimal version through the Azure portal, through PowerShell (**Set-AzInstance**), or the az CLI (**az sql mi update**). You can read more about TLS and Managed Instance at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/minimal-tls-version-configure`](https://learn.microsoft.com/azure/azure-sql/managed-instance/minimal-tls-version-configure).

Azure SQL Database forces encrypted connections whether the client or application enables it. You can verify this by examining the **encrypt_option** column for **sys.dm_exec_connections**, and seeing it will always be a value of TRUE for any user TCP connection. Azure SQL Database also offers enforcement of a minimal TLS version (1.0, 1.1, and 1.2) through the Azure portal, PowerShell (**Set-AzSqlServer**), and az CLI (**az sql server update**). You can read more at [`https://learn.microsoft.com/azure/azure-sql/database/connectivity-settings?view=azuresql&tabs=azure-portal#minimal-tls-versio`](https://learn.microsoft.com/azure/azure-sql/database/connectivity-settings?view=azuresql&tabs=azure-portal#minimal-tls-versio).

Tip

Because Azure SQL Database forces an encrypted connection, the best practice is to enable this for your client tool or application. This speeds up connection time since the server must negotiate with the client to set the encryption if not set by the client.

SQL Server 2022 now supports TLS 1.3\. I anticipate this TLS version to be supported for Azure SQL soon.

### Transparent Data Encryption (TDE)

Transparent Data Encryption (TDE) is an *encryption at rest* technology that has been in use with SQL Server for many releases. The concept is that the SQL Server engine will encrypt and decrypt data to the files for the database as data is written and read from disk. This way, the data in the file is encrypted to protect any *offline* attempt to access the files of the database. Azure SQL Managed Instance and Database enable this option **by default**.

You might wonder why you would need this encryption option since you or anyone doesn’t have access to files in the underlying virtual machines of Azure SQL. Enabling TDE by default is just another mechanism in Azure’s commitment to a *defense in depth* methodology to protect your data. Many use TDE with SQL Server deployments in their own data center to protect from an unexpected intrusion to access database files outside the engine. The same holds true for Azure even though the Azure ecosystem has many protection mechanisms in place for data centers.

For Azure SQL Managed Instance, TDE is on by default for the instance, which means all databases created for the instance are enabled by TDE. You cannot disable this option for the instance, but you can individually disable TDE through ALTER DATABASE or tools like SSMS. One option you do have for a Managed Instance is to control what keys are used for encryption for TDE. By default, Azure SQL Managed Instance uses a service-managed key, which means Azure SQL manages a certificate for the key (rotates the key and protects it with a root key within Azure).

Azure SQL Database also supports configuring TDE for databases with ALTER DATABASE (ENCRYPTION option) but also allows you to enable and disable TDE through the Azure portal, PowerShell (**Set-AzSqlDatabaseTransparentDataEncryption**), and az CLI (**az sql db tde set**). The default key management is also a service-managed key at the logical server level.

#### Bring Your Own Key (BYOK)

Many of our customers want complete control of the keys used for encryption (compliance and trust are an example). SQL Server provides a method to use an Extensible Key Management (EKM) provider to protect the Database Encryption Key (DEK) used to encrypt data with TDE. One of the EKM providers allowed is Azure Key Vault. This allows keys used for encryption to be stored outside of SQL Server.

Azure SQL provides a similar mechanism affectionately called *Bring Your Own Key* (BYOK). You may also see this referred to as *customer-managed key* (CMK). The mechanism to use Azure Key Vault for BYOK is referred to as a **TDE protector**. Azure Key Vault is a service in Azure to help you centrally store and manage secrets and keys. As stated in the introductory documentation for Azure Key Vault at [`https://learn.microsoft.com/azure/key-vault/general/overview`](https://learn.microsoft.com/azure/key-vault/general/overview), Azure Key Vault is “Secrets and keys are safeguarded by Azure, using industry-standard algorithms, key lengths, and hardware security modules (HSMs). The HSMs used are Federal Information Processing Standards (FIPS) 140-2 Level 2 validated.”

Authorization to create a key vault and create and manage keys is done through Microsoft Entra. Azure Key Vault keys can be set at an instance or logical server level and apply to all databases in the instance or logical server. In addition, Azure SQL Database now supports the key to be set at the database level. I like the diagram in this blog post at [`https://azure.microsoft.com/blog/announcing-transparent-data-encryption-tde-with-customer-managed-keys-for-managed-instance`](https://azure.microsoft.com/blog/announcing-transparent-data-encryption-tde-with-customer-managed-keys-for-managed-instance) to show how Azure Key Vault BYOK works as seen in Figure [6-12](#496204_2_En_6_Chapter.xhtml#Fig12).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig12_HTML.jpg)

Figure 6-12

Using Azure Key Vault for BYOK with TDE

Let me show you how I can easily use BYOK with an Azure SQL Database. I will navigate in the Azure Portal to my database called **bwhyperscale** and then select Data Encryption from the Security option in the Service menu as you see in Figure [6-13](#496204_2_En_6_Chapter.xhtml#Fig13).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig13_HTML.jpg)

Figure 6-13

Configure BYOK for TDE for Azure SQL Database

You can see I’ve selected Database level customer-managed key (CMK). I can now select Change key to use a screen to create a new key in Azure Key Vault and apply that to my database encryption. I can even use a managed identity to manage the key which could be used across multiple Azure resources.

This seems easy enough, but there are several considerations when you use BYOK with Azure SQL:

*   COPY_ONLY backups for Managed Instance are only supported when you use BYOK (because you have the keys to restore) as opposed to backups encrypted by the service-managed TDE given the certificate is inaccessible.

*   Your key vault and Azure SQL deployment must belong to the same Microsoft Entra tenant.

*   Like any scenario where you manage keys, you should back them up regularly. Learn more about Azure Key Vault backups at [`https://learn.microsoft.com/azure/key-vault/general/backup`](https://learn.microsoft.com/azure/key-vault/general/backup).

*   There are considerations for BYOK with high availability. Read more at [`https://learn.microsoft.com/azure/azure-sql/database/transparent-data-encryption-byok-overview?view=azuresql&viewFallbackFrom=sql-server-ver15#high-availability-with-customer-managed-tde`](https://learn.microsoft.com/azure/azure-sql/database/transparent-data-encryption-byok-overview?view=azuresql&viewFallbackFrom=sql-server-ver15#high-availability-with-customer-managed-tde).

### Always Encrypted

Always Encrypted is a technology based on work from Microsoft research used to provide end-to-end encryption for SQL applications. It was introduced in SQL Server 2016 and has all the same capabilities in Azure SQL. Just like with SQL Server, keys for Always Encrypted can be stored in Azure Key Vault. Read the entire story about how Always Encrypted works and how to set it up at [`https://learn.microsoft.com/sql/relational-databases/security/encryption/always-encrypted-database-engine`](https://learn.microsoft.com/sql/relational-databases/security/encryption/always-encrypted-database-engine).

One unique aspect to Always Encrypted that is new since the first edition of this book is support for ***secure enclaves*** for Azure SQL Database. This option provides a faster and more robust option for Always Encrypted scenarios. This is because much of the decryption for SQL Server happens within the engine using the secure enclave. This is also why we now have the DC-series hardware option for Azure SQL Database since it behind the scenes uses the Intel SGX chip architecture. Read more at [`https://learn.microsoft.com/sql/relational-databases/security/encryption/always-encrypted-enclaves`](https://learn.microsoft.com/sql/relational-databases/security/encryption/always-encrypted-enclaves).

### Dynamic Data Masking (DDM)

One other method to protect your data is to control which users have access to view *sensitive* data. Many applications provide this type of protection by *masking* data in the display layers of their application. For example, a web application may display a phone number as XXX-XXX for some users and the full phone number for others. The problem with this approach is that the application must be modified if any rules change on the masks used or which users can see data or masked data.

SQL Server provides a method to control masking of data at the database layer instead of the application. Then *any* application or tool would only see data based on masking rules defined with T-SQL. This feature is called Dynamic Data Masking (DDM) and was introduced in SQL Server 2016\. You can read the full documentation of DDM at [`https://learn.microsoft.com/sql/relational-databases/security/dynamic-data-masking`](https://learn.microsoft.com/sql/relational-databases/security/dynamic-data-masking). Azure SQL supports DDM through T-SQL statements as referenced in the documentation. In addition, Azure SQL Database allows you to manage masks and permissions through the Azure portal as seen in Figure [6-14](#496204_2_En_6_Chapter.xhtml#Fig14) for a database I created based on the sample AdventureWorksLT.

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig14_HTML.jpg)

Figure 6-14

Dynamic data masking through the Azure portal

You can see using the Azure portal will provide recommendations on columns to mask based on the column names (e.g., LastName or EmailAddress) I have in my database.

### SQL Ledger

If you read *SQL Server 2022 Revealed*, I wrote about a new feature in SQL Server that is a blockchain *inside* the database. The key concept here is that Ledger provides a *tamper-evident record* of changes in the database.

The best way to describe SQL Ledger is with a diagram as you see in Figure [6-15](#496204_2_En_6_Chapter.xhtml#Fig15).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig15_HTML.jpg)

Figure 6-15

SQL Ledger

SQL Ledger allows you to create a table using T-SQL with an additional syntax WITH LEDGER = ON, and now automatically we will track all changes to the table using the technology of temporal tables. We will build a history table of your changes and a view that allows you to see the current table along with the historical changes.

How is this different than temporal tables? It is the Database ledger, which is a series of system internal tables that track

*   Which SQL principal made the change

*   A crypto hash of the actual changes

Now you can track what the change was, when the change was made, who made the change, and make sure no one has tampered internally with the data, for example, if someone tried to hack the system and “fake out” an UPDATE to look like the original value and hide the change.

The Database ledger keeps track of these hashes in a chain, hence a blockchain. This has advantages over traditional blockchain systems since all the data is kept inside the SQL Server database. No application logic is required here since everything is built into SQL.

In addition, a *digest* can be stored, which is an independent hash on top of the Database ledger. This provides an independent method to verify the Database ledger itself is valid.

One additional option not shown in Figure [6-15](#496204_2_En_6_Chapter.xhtml#Fig15) is that you can make a ledger table APPEND_ONLY which means only T-SQL INSERT statements are allowed. This is almost similar to a “journal” along with the other benefits of Ledger.

Azure SQL Database offers a few unique options as you can see in Figure [6-16](#496204_2_En_6_Chapter.xhtml#Fig16).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig16_HTML.jpg)

Figure 6-16

Azure SQL Database ledger options

You can from the Azure Portal enable all tables to use ledger by default, verify the Database ledger for all ledger tables, and enable the digest to be stored automatically in Azure Storage or Azure Confidential Ledger.

Ledger is available in Azure SQL Database, Azure SQL Managed Instance, and SQL Server 2022 both in Azure VM and on-premises. Learn everything you need to know about SQL Ledger at [`https://aka.ms/sqlledger`](https://aka.ms/sqlledger).

## Security Management

After you have configured security for authentication, access, and data protection, you will want to manage and audit specific aspects of security. This includes monitoring activity, using SQL auditing, and checking for security issues with Microsoft Defender for Cloud.

### Monitoring Activities Outside of SQL

When you deploy and manage SQL Server on Windows and Linux, the operating system provides several different methods to audit activity outside of SQL Server. You may have other methods within your data center to audit this type of activity.

The Azure ecosystem provides this same type of audit capabilities. You may have seen in Chapter [4](#496204_2_En_4_Chapter.xhtml) after deploying Azure SQL Managed Instance and Database the **Activity Log** for these types of activities. Think of this as a way to audit Azure RBAC.

The Activity Log is a *platform log* supported by the Azure ecosystem for all subscriptions. In fact, the Activity Log is a record of all events for your Azure subscription and includes events specific to Azure SQL Managed Instance and Database. Basically, any operation you perform against an Azure SQL resource that is *outside* of SQL Server is recorded in the Activity Log. I’ve used the Windows Event Log for many years, and I like to think of the Activity Log as the Event Log of Azure.

The Azure portal provides an excellent way to view Activity Log entries specific to an Azure resource. For example, if I navigate to my logical server **bwsqllogicalserver** in the Azure portal, then select Activity log from the Resource menu, and then change the timeframe for a week, I see entries like in Figure [6-17](#496204_2_En_6_Chapter.xhtml#Fig17).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig17_HTML.jpg)

Figure 6-17

The Activity Log for Azure SQL Database logical server

You can see some of the log entries show activities like updating firewall rules or setting up private endpoints.

By default, Azure Activity Log entries are kept for 90 days (and they are rolled over). If you want to keep Activity Log entries longer than this, you can create a Log Analytics workspace. A Log Analytics workspace also gives you more capabilities to query and visualize Activity Log entries. On this screen, there is an option at the top for Diagnostic settings. This allows you to create a Log Analytics workspace and add Activity Log entries to it. You also have the option to send Activity Log entries to Event Hub for streaming. To read more about using the Activity Log in general, go to [`https://learn.microsoft.com/azure/azure-monitor/essentials/activity-log`](https://learn.microsoft.com/azure/azure-monitor/essentials/activity-log).

### Auditing Azure SQL Managed Instance

Since Azure SQL Managed Instance is very much like a full SQL Server instance, many of the familiar tools and features for auditing are available to you.

#### Tracking Logins

Since I can remember, almost every SQL Server version tracks failed logins in the ERRORLOG of SQL Server. A failed login looks like this in the ERRORLOG for a Managed Instance:

```
Error: 18456, Severity: 14, State: 7.
Login failed for user 'sa'. Reason: An error occurred while evaluating the password. [CLIENT: 10.1.0.4]
```

SQL Server provides the ability to turn off this tracking or also track successful logins. That capability is not available for a Managed Instance (even though SSMS gives you the appearance it is allowed) because it requires a restart of SQL Server which you don’t have access to do.

Since Azure SQL Managed Instance gives you full access to Extended Events, you can use these events to track logins: **process_login_finish, login_event, and login**. Extended Events for Azure SQL Managed Instance supports all events, actions, and targets. File targets must use Azure Blob Storage since you don’t have access to the underlying OS file system.

#### SQL Server Audit

SQL Server Audit is a capability that has been in SQL Server in several releases to audit and track instance and database activity. SQL Server Audit is fully supported with Azure SQL Managed Instance with a few exceptions:

*   Audit files are stored in Azure Blob Storage. Read more about how to do this at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/auditing-configure?view=azuresql#set-up-auditing-for-your-server-to-azure-storage`](https://learn.microsoft.com/azure/azure-sql/managed-instance/auditing-configure?view=azuresql#set-up-auditing-for-your-server-to-azure-storage).

*   The option to shut down SQL Server on an audit failure is not supported (but continue and fail options are supported).

If you have never used SQL Server Audit, look through the documentation at [`https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-database-engine`](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-database-engine).

SQL Server Audit produces files to track activity based on the Extended Event format (SQL Server Audit uses Extended Event sessions behind the scenes). Azure SQL Managed Instance also allows you to produce audit events to Azure Monitor Logs and Event Hub. The option TO EXTERNAL MONITOR has been added to the CREATE SERVER AUDIT T-SQL statement.

See an example of how to configure SQL Server Audit to Azure logs or Event Hub at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/auditing-configure?view=azuresql#set-up-auditing-for-your-server-to-event-hubs-or-azure-monitor-logs`](https://learn.microsoft.com/azure/azure-sql/managed-instance/auditing-configure?view=azuresql#set-up-auditing-for-your-server-to-event-hubs-or-azure-monitor-logs).

### Auditing Azure SQL Database

Auditing activity for Azure SQL Database is provided through metrics in Dynamic Management Views (DMVs) and Azure Metrics. In addition, the SQL Server Audit capability is exposed as a feature called **SQL Database auditing**.

#### Tracking Connections

Azure SQL Database provides a DMV called **sys.event_log** that can be queried *in the context of the logical master database* of the logical server. This DMV shows information collected in five-minute aggregate intervals for connectivity metrics. This DMV doesn’t track individual successful or failed connection but rather connectivity metrics across all databases (including the logical master) for the logical server. This DMV is not supported in Azure SQL Managed Instance.

Examples of what you can view for this DMV include

*   Number of successful connections

*   Number of failed connections due to invalid login name

*   Number of failed connections due to blocked firewall rule

While this information is stored across all databases, you can use Azure Metrics and Logs to capture aggregate numbers for failed connections, connections blocked by firewall rules, and successful connections.

Figure [6-18](#496204_2_En_6_Chapter.xhtml#Fig18) shows how I’ve added the number of successful connections for one of my Azure SQL Databases.

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig18_HTML.jpg)

Figure 6-18

Tracking successful connections with Azure Metrics

#### SQL Database Auditing

Since you don’t have access to the underlying SQL Server instance for an Azure SQL Database, you don’t have access to use the T-SQL statement CREATE SERVER AUDIT to use SQL Server Audit capabilities.

Therefore, we have created interfaces *outside* of Azure SQL Database to audit database and logical server activities. We call this **SQL Database auditing**. Read the complete documentation at [`https://learn.microsoft.com/azure/azure-sql/database/auditing-overview`](https://learn.microsoft.com/azure/azure-sql/database/auditing-overview). SQL Database Auditing can be enabled through the Azure portal, PowerShell (**Set-AzSqlDatabaseAudit** and **Set-AzSqlServerAudit**), and az CLI (**az sql db audit-policy**).

You can direct SQL Database auditing to an Azure Storage account, a Log Analytics workspace, or Event Hub for streaming.

Let’s look at an example of creating an audit for the logical server and direct the audit to a Storage Account **and** Log Analytics workspace (and explain why you might want to use one vs. the other or both). Auditing the logical server will audit all activities for all databases.

Note

You could also create a separate audit specific to each database, but when you audit a logical server, all activities for all databases go into that audit as well.

I first created a new Log Analytics workspace per the steps you can do at [`https://learn.microsoft.com/azure/azure-monitor/logs/quick-create-workspace`](https://learn.microsoft.com/azure/azure-monitor/logs/quick-create-workspace). I called this **bwsqldbloganalytics**.

I started the process by navigating to my logical server **bwsqllogicalserver**, selected Auditing from the Service Menu, and was presented with the top of the screen like in Figure [6-19](#496204_2_En_6_Chapter.xhtml#Fig19).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig19_HTML.jpg)

Figure 6-19

First part of configuring auditing for Azure SQL Database

I selected the option to Enable auditing and then selected Create new to create a new Azure Storage account to store for auditing. I also chose Managed Identity to use for authentication to create a more secure access to audit storage.

Scrolling down, I have more options to choose as you can see in Figure [6-20](#496204_2_En_6_Chapter.xhtml#Fig20).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig20_HTML.jpg)

Figure 6-20

Second part of configuring SQL Audit for Azure SQL Database

I’ve added the Log Analytics workspace I created earlier. In addition, I’ve configured an option to audit Microsoft support operations to the same Log Analytics workspace (it can be a different workspace or even different Azure Storage). This option really shows how serious Microsoft is about transparency. This means you can audit any action taken by Microsoft support against your deployment if you should contact them for any issue. You can read more about this option at [`https://learn.microsoft.com/azure/azure-sql/database/auditing-microsoft-support-operations`](https://learn.microsoft.com/azure/azure-sql/database/auditing-microsoft-support-operations). I clicked **Save** to persist my choices.

Note

I also have the choice to enable auditing at the database level. If I enable auditing for a logical server, all databases have auditing enabled.

SQL Server Audit has a concept called an action group which defines what activities are audited. SQL Database auditing has by default the following action groups enabled:

*   BATCH_COMPLETED_GROUP – Audit all successful SQL statements

*   SUCCESSFUL_DATABASE_AUTHENTICATION_GROUP – Audit a successful login to a database

*   FAILED_DATABASE_AUTHENTICATION_GROUP – Audit a failed connection to a database

All the possible action groups to use can be found at [`https://learn.microsoft.com/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions`](https://learn.microsoft.com/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions). You can use the PowerShell cmdlet **Set-AzSqlDatabaseAudit** to enable other action groups.

With SQL Server, you typically can view an audit through SSMS or using the system function **sys.fn_get_audit_file**. Turns out you can do the same thing for each database as part of the logical server.

I just turned on auditing, so I don’t expect much activity. I’ll connect with SSMS as I showed you earlier in this chapter with the managed identity I enabled and created a login, and I’ll connect on my laptop with the Microsoft Entra ID I set up as the admin. To view audit logs, I’ll navigate to my database **bwhyperscale**, then select Auditing from the Security section of the Service menu. And then select View Audit Logs at the top of the screen. My results are in Figure [6-21](#496204_2_En_6_Chapter.xhtml#Fig21).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig21_HTML.jpg)

Figure 6-21

SQL Audit records for an Azure SQL Database

You can see from these results my different connections. Notice the GUID values for the managed identity as well as my Microsoft Entra admin account. You can click Run in Query Editor to see the SQL query behind these results which uses the **sys.fn_get_audit_file** system procedure to read from the Azure Storage account I configured for auditing and is the same result as the Audit records in the previous screen. You can also use the Log Analytics option in the Command Menu to use different techniques to view audit results. This option can provide a more visual representation for your security audit. Learn more at [`https://learn.microsoft.com/azure/azure-sql/database/auditing-analyze-audit-logs?view=azuresql#analyze-logs-using-log-analytics`](https://learn.microsoft.com/azure/azure-sql/database/auditing-analyze-audit-logs?view=azuresql#analyze-logs-using-log-analytics).

### Microsoft Defender for SQL

I’ve mentioned earlier in this book a few times about Microsoft Defender for SQL (which is technically under the umbrella of Microsoft Defender for Cloud). Microsoft Defender for Cloud provides insights into security vulnerabilities and possible attacks against all your Azure resources (and for other clouds which is why this service was rebranded Microsoft vs. Azure). Microsoft Defender for Cloud is a paid service and can be enabled per resource or at the subscription level. Once it is enabled, you can get an overview of all possible insights from the Microsoft Defender for Cloud dashboard (in the first edition of this book, this was called the Azure Security Center) as you see in Figure [6-22](#496204_2_En_6_Chapter.xhtml#Fig22).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig22_HTML.jpg)

Figure 6-22

Microsoft Defender for Cloud Overview

It looks like a daunting amount of information which is why I like to first focus on Recommendations and Security Alerts. In addition, there is a new Copilot experience to analyze details like Recommendations and Alerts as you can see in Figures [6-23](#496204_2_En_6_Chapter.xhtml#Fig23) and [6-24](#496204_2_En_6_Chapter.xhtml#Fig24).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig24_HTML.jpg)

Figure 6-24

Using Copilot for Azure with Microsoft Defender

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig23_HTML.jpg)

Figure 6-23

Microsoft Defender for Cloud Recommendations

Tip

This Copilot experience is part of the new Copilot for Security which has even more capabilities than what you see here for Defender. You can learn more at [`https://www.microsoft.com/security/business/ai-machine-learning/microsoft-copilot-security`](https://www.microsoft.com/security/business/ai-machine-learning/microsoft-copilot-security).

For Azure SQL, there are two primary categories of capabilities you can use with Microsoft Defender: **Vulnerability Assessment** and **SQL Threat Protection**.

#### Vulnerability Assessment

Another aspect to securing your data is to proactively monitor and check for any known security vulnerabilities. But what are known vulnerabilities? Azure SQL has a knowledge base of rules we have built (based on industry standards from [`https://www.cisecurity.org/cis-benchmarks/`](https://www.cisecurity.org/cis-benchmarks/)) to scan your Azure SQL Managed Instances and Databases for configurations that might be considered vulnerable. I like to think of a vulnerability assessment like a virus checker which uses a *scan* method to look for possible issues. I’m using a method called the *express configuration* for a Vulnerability Assessment. This new method, which is now the default, doesn’t require any storage to store the results (the classic configuration requires this).

Figure [6-25](#496204_2_En_6_Chapter.xhtml#Fig25) shows a Vulnerability Assessment on my logical server.

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig25_HTML.jpg)

Figure 6-25

Vulnerability Assessment on a logical server

On this screen, I can see findings, basic check information, what benchmark they were associated with, and severity. Notice on the command menu I can initiate a manual scan (Vulnerability Assessments are typically scheduled once a week) and see scan history. The scan is lightweight and safe. It takes a few seconds to run and is entirely read-only. It doesn’t make any changes to your database.

If I click a specific rule, I can drill into more detail as you can see in Figure [6-26](#496204_2_En_6_Chapter.xhtml#Fig26).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig26_HTML.jpg)

Figure 6-26

Details of a Vulnerability Assessment finding

This finding is based on the fact that I have exposed the public ending for this logical server. I’m using firewall rules but allowing any Azure resource to connect.

On the left-hand side, you can get more information about the finding. On the right-hand side, you have a script to remediate the problem and the query that was used to detect the problem.

Below are options to add the results as part of the *baseline* or remove them. If I add the results in their current state, that means it is not an issue to expose the public endpoint with firewall rules and allow Azure resources. So subsequent scans will not have this finding show up. If I want to change this later, I can remove this from the baseline, and it will show up again. Another nice feature of Vulnerability Assessments is that you can configure the system to email you with findings for on-demand scans and weekly scheduled scans.

You can also use PowerShell (e.g., **Get-AzSqlDatabaseVulnerabilityAssessmentScanRecord**) to show and manage Vulnerability Assessments.

You can learn more about Vulnerability Assessments at [`https://learn.microsoft.com/azure/defender-for-cloud/sql-azure-vulnerability-assessment-overview`](https://learn.microsoft.com/azure/defender-for-cloud/sql-azure-vulnerability-assessment-overview).

Tip

One other new feature for Defender I think you should explore is the Data Security Dashboard. This capability helps you view possible security vulnerabilities and threats focused on sensitive data you may have. Learn more at [`https://learn.microsoft.com/azure/defender-for-cloud/data-aware-security-dashboard-overview`](https://learn.microsoft.com/azure/defender-for-cloud/data-aware-security-dashboard-overview).

#### SQL Threat Protection

I’m not a fan of the term here because SQL Threat Protection doesn’t prevent problems, it *detects* them. Examples include brute-force login attacks, logins from unusual locations, or potential SQL injection attacks.

SQL Threat Protection uses Extended Events and machine learning technology to detect and alert you to certain types of suspicious activities. An example of a suspicious activity is code designed for a *SQL injection*. You can read all the various rules and alerts detected at [`https://learn.microsoft.com/azure/defender-for-cloud/alerts-sql-database-and-azure-synapse-analytics`](https://learn.microsoft.com/azure/defender-for-cloud/alerts-sql-database-and-azure-synapse-analytics).

While we don’t document the details of how we detect all rules, I can show you how to simulate a SQL injection using SSMS to see an alert.

Using SSMS from my laptop, I’ll create a new query connection using SSMS, but I will use the Additional Connection Parameters of SSMS (from the Options button) to put in this string:

```
Application Name=webappname
```

I also used the Connect to database to connect to the **bwadw** database (this database uses the sample AdventureWorksLT). In the Query Editor window, I put in this query and executed it:

```
SELECT * FROM SalesLT.Customer WHERE CustomerID like '' or 1 = 1 --' and family = 'test1';
```

Note

We filter out SSMS as an application because no one sends in injections from a tool like SSMS. Therefore, I used a different application name to simulate a real application sending a query that looks like it could be a SQL injection attack.

Within a matter of seconds, I can navigate to my database in the Azure Portal, select Microsoft Defender for Cloud from the Security section of the Service Menu, and then select “Check for alerts on this resource in Microsoft Defender for Cloud”; you will see a screen like the one in Figure [6-27](#496204_2_En_6_Chapter.xhtml#Fig27).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig27_HTML.jpg)

Figure 6-27

A SQL injection attack detected

You can click View full details to see more about the source of the attack and more details on why this is considered an injection attack.

If you are wondering why this is an attack, take a closer look at the SQL query I used:

```
SELECT * FROM SalesLT.Customer WHERE CustomerID like '' or 1 = 1 --' and family = 'test1';
```

In this case, the parser stops processing the query when it sees the comment --. So, someone who crafts this query is attempting to see all customers. Imagine if the input for the WHERE clause came from an application web input field. If the developer dynamically concatenates the query and the input string, a hacker can try to use this technique to see data they likely should not (it can get far worse). SQL injection is an interesting security topic, and we have documentation at [`https://docs.microsoft.com/sql/relational-databases/security/sql-injection`](https://docs.microsoft.com/sql/relational-databases/security/sql-injection) to explain how dangerous this can be and how to avoid them.

Like a Vulnerability Assessment, you can configure your system to receive emails upon any threat detected. Microsoft takes this so seriously; anytime I try a test like this, I get an email from our company security team making sure I’ve seen this and that all is OK with my database!

SQL Threat Detection also has interfaces for PowerShell (e.g., **Set-AzSqlDatabaseThreatDetectionPolicy**) and az CLI (e.g., **az sql db threat-policy**).

### Data Classification

Let’s say you would like to classify the tables in your database into categories to look for access to data for policies like General Protection Data Regulation (GDPR) or Personally Identifiable Information (PII). Data classification for Azure SQL provides a method to classify your columns based on detected pattern match of names or allows you to tag them manually.

I can do this from the Azure Portal as you can see in Figure [6-28](#496204_2_En_6_Chapter.xhtml#Fig28).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig28_HTML.jpg)

Figure 6-28

Data classification for Azure SQL

If I choose all these recommendations and select Accept selected recommendations, I can use the Overview pane to get a report for data classification as you can see in Figure [6-29](#496204_2_En_6_Chapter.xhtml#Fig29).

![](images/496204_2_En_6_Chapter/496204_2_En_6_Fig29_HTML.jpg)

Figure 6-29

Data classification overview

One of the nice aspects to this feature is that access to data that is marked as classified can be seen in SQL auditing. Now you can track who has tried to access data that you have marked as classified with these information types and labels.

Having shown you this, this capability can now be integrated with Microsoft Purview. To read more about how to use Microsoft Purview for data classification, see the documentation at [`https://learn.microsoft.com/azure/azure-sql/database/data-discovery-and-classification-overview?view=azuresql#enabling-access-control-for-sensitive-data-using-microsoft-purview-information-protection-policies-public-preview`](https://learn.microsoft.com/azure/azure-sql/database/data-discovery-and-classification-overview?view=azuresql#enabling-access-control-for-sensitive-data-using-microsoft-purview-information-protection-policies-public-preview).

## Summary

In this chapter, you learned how Azure SQL security is just like SQL Server in many ways. You learned how to secure your network and authenticate logins and users, including with Microsoft Entra. You learned how to protect your data with various encryption techniques. You learned all the audit capabilities you can use outside and in Azure SQL. Furthermore, you learned to go further with the cloud using Microsoft Defender for Cloud and data classification.

You have learned in this book that the speed of the cloud helps us innovate and adapt quickly. Security is always part of that innovation. According to Joachim Hammer, Principal Group PM Manager for Security, “We continue to invest in a three-pronged approach to ensure Azure SQL meets the most stringent security requirements as well as regulatory compliances in the industry. These areas include State-of-the-art, built-in Security Controls, Trust and Compliance, and Threat detection and assessment.”

There are some self-paced labs for Microsoft Entra authentication for Azure SQL Database you can try yourself in the Cloud Workshop at [`https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/07_Deploy_Manage_Optimize_AzureSQLDB`](https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/07_Deploy_Manage_Optimize_AzureSQLDB).

In the next chapter, we will explore and dive deep into the second major aspect to the core of Azure SQL: performance.

# 7. Monitoring and Tuning Performance for Azure SQL

You now have seen how to secure your Azure SQL deployment. Another aspect to ensure you have the best possible database for your application is understanding how to scale, configure, monitor, and tune performance. If you know SQL Server, here is some good news. The engine that powers Azure SQL is the same one for SQL Server! This means that about any performance capability you need exists for Azure SQL. It also means that many of the same tasks and skills you use for SQL Server apply to Azure SQL. In this chapter, we will explore all the capabilities and tasks you normally use to monitor and tune performance for a SQL Server and compare it with Azure SQL. You will also learn some specific capabilities and techniques unique to Azure to give you the best possible performance.

This chapter will contain examples for you to try out and use as you read along. For you to try out any of the techniques, commands, or examples I use in this chapter, you will need

*   An Azure subscription.

*   A minimum of Contributor role access to the Azure subscription. You can read more about Azure built-in roles at [`https://learn.microsoft.com/azure/role-based-access-control/built-in-roles`](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles).

*   Access to the Azure Portal.

*   A deployment of an Azure SQL Managed Instance and/or an Azure SQL Database as I did in Chapter [4](#496204_2_En_4_Chapter.xhtml). The Azure SQL Database I deployed uses the AdventureWorks sample which will be required to use for some of the examples.

*   To connect to Managed Instance, you will need a *jumpbox* or virtual machine in Azure to connect. I showed you how to do this in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book. One simple way to do this is to create a new Azure Virtual Machine and deploy it to the same virtual network as the Managed Instance (you will use a different subnet than the Managed Instance).

*   To connect to Azure SQL Database, I am going to use the Azure VM I deployed in Chapter [3](#496204_2_En_3_Chapter.xhtml), called **bwsql2022**, and configured for a private endpoint in Chapter [6](#496204_2_En_6_Chapter.xhtml) (you could use another method as long as you can connect to the Azure SQL Database).

*   Installation of the **az** CLI (see [`https://learn.microsoft.com/cli/azure/install-azure-cli`](https://learn.microsoft.com/cli/azure/install-azure-cli) for more details). You can also use the Azure Cloud Shell instead since az is already installed. You can read more about the Azure Cloud Shell at [`https://azure.microsoft.com/features/cloud-shell/`](https://azure.microsoft.com/features/cloud-shell/).

*   You will run some T-SQL in this chapter, so install a tool like SQL Server Management Studio (SSMS) at [`https://aka.ms/ssms`](https://aka.ms/ssms). I installed SSMS in the **bwsql2022** Azure Virtual Machine I deployed in Chapter [3](#496204_2_En_3_Chapter.xhtml).

*   For this chapter, I have script files you can use for several of the examples. You can find these scripts in the **ch7_performance** folder for the source files included for the book. I will also use the very popular tool ostress.exe for exercises in this chapter which comes with the Replay Markup Language (RML) Utilities. You can download RML from [`https://aka.ms/ostress`](https://aka.ms/ostress). Make sure to put the folder where RML gets installed in your system path (which is by default C:\Program Files\Microsoft Corporation\RMLUtils).

Tip

Since the first edition of this book, I built some new performance troubleshooting videos and example labs specifically targeted for Azure SQL Database. You can find the videos at [`https://aka.ms/performanceseries`](https://aka.ms/performanceseries) and the example labs at [`https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting`](https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting). I think you will find these a nice complement to the examples in this chapter (with even more performance scenarios).

## Performance Capabilities

Since the engine that powers Azure SQL is the same as SQL Server, about any performance capability is available to you. Having said that, it is important to cover a few important areas that are similar and different that can affect your ability to ensure maximum performance for your Azure SQL deployment. This includes maximum capacities, indexes, In-Memory OLTP, partitions, SQL Server 2022 performance enhancements, and new Azure SQL intelligent performance capabilities.

### Max Capacities

When you choose a platform to install SQL Server, you typically *size* the resources you need. In many cases, you plot out the maximum capacities you will need for resources such as CPU, memory, and disk space. You may also ensure you have the correct performance capabilities for I/O with regard to IOPS and latency.

In Chapters [4](#496204_2_En_4_Chapter.xhtml) and [5](#496204_2_En_5_Chapter.xhtml) of the book, I showed you all the options to choose, deploy, and configure your Azure SQL Managed Instance and Azure SQL Database deployments. To ensure you have the best performance, you need to keep these capacities in mind with Azure SQL:

*   Azure SQL Managed Instance can support *up to* 128 vCores, ~870GB of memory, and a maximum storage of 32TB, depending on your service tier and hardware configuration choice.

*   Azure SQL Database can support up to 128 vCores, 830GB memory, and a 4TB database depending on your service tier and hardware configuration choice. The Hyperscale deployment option for Azure SQL Database can support up to 100TB database and unlimited transaction log space.

*   Your decision on deployment options such as number of vCores affects other resource capacities whether it be a Managed Instance or Database deployment. For example, the number of vCores for a General Purpose Azure SQL Database affects the maximum memory, maximum database size, maximum transaction log size, and maximum log rate, among others.

Let us stop here to help you get oriented. How can you see a chart or table to figure out the limits for all these choices?

For a Managed Instance, start with this documentation page which shows differences for hardware configurations: [`https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits?view=azuresql#hardware-configuration-characteristics`](https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits?view=azuresql#hardware-configuration-characteristics). Then continue to scroll down this page to see other limits such as vCores, memory, storage, and others organized by service tier.

What about Azure SQL Database? You can view a table for capacities and limits based on vCores at [`https://learn.microsoft.com/azure/azure-sql/database/resource-limits-vcore-single-databases`](https://learn.microsoft.com/azure/azure-sql/database/resource-limits-vcore-single-databases). You will see there are some differences if you choose a Serverless compute tier. Bookmark these documentation links. I use them all the time.

Keep in mind that some limits like memory are enforced by Windows Job Objects. I mentioned this implementation in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book. Use the DMV **sys.dm_os_job_object** to see the true limits for memory and other resources for your deployment.

Tip

Once you deploy, you get to see your resource limits with DMVs like **sys.dm_user_db_resource_governance**.

What if you make the wrong choice and need more capacity? The good news is that you can make changes for Azure SQL Managed Instance and Database to get more (or less) without any database migration required. You will see an example of this later in this chapter. Just remember that a change for Managed Instance can take a significant amount of time.

In addition, for Azure SQL Database, you can switch back and forth between General Purpose and Business Critical service tiers (note that there could be some time required to do this for database seeding). If you move from General Purpose to Hyperscale, you can move back one time. However, if you start with Hyperscale, you cannot move back to General Purpose (and can never move to Business Critical).

### Indexes

Anyone who works with SQL Server knows that without proper indexes, it is difficult to obtain the query performance you need.

Every type of index option you can use in SQL Server is available to you with Azure SQL, including clustered, non-clustered, online, and resumable indexes. You can read an index primer at [`https://learn.microsoft.com/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described`](https://learn.microsoft.com/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described) and details on online indexes at [`https://learn.microsoft.com/sql/relational-databases/indexes/perform-index-operations-online`](https://learn.microsoft.com/sql/relational-databases/indexes/perform-index-operations-online) and resumable online indexes at [`https://learn.microsoft.com/sql/relational-databases/indexes/guidelines-for-online-index-operations?view=sql-server-ver16#resumable-index-considerations`](https://learn.microsoft.com/sql/relational-databases/indexes/guidelines-for-online-index-operations?view=sql-server-ver16#resumable-index-considerations). Resumable online indexes are very compelling since you can pause and resume large index builds and even continue an index build or rebuild at the point where it failed.

**Columnstore indexes** are nothing short of amazing. I continue to see customers who just do not take advantage of this capability. Columnstore index can accelerate read query performance by 100x for the right workload. Columnstore indexes are supported in every deployment option you choose with Azure SQL. One myth about columnstore is that it is *only* an in-memory technology. The truth is that columnstore indexes perform best when they fit in memory and use compression, so more will fit in your memory limits. However, a columnstore index does not have to all fit in memory. To get a start on columnstore indexes, see the documentation at [`https://learn.microsoft.com/sql/relational-databases/indexes/columnstore-indexes-overview`](https://learn.microsoft.com/sql/relational-databases/indexes/columnstore-indexes-overview).

### In-Memory OLTP

In SQL Server 2014 (and greatly enhanced in SQL Server 2016), we introduced a revolutionary capability for high-speed transactions called In-Memory OLTP (codename Hekaton). In-Memory OLTP is available for Azure SQL Managed Instance and Databases if you choose the Business Critical service tier.

Memory-optimized tables are the mechanism to use In-Memory OLTP. Memory-optimized tables are truly *in-memory* as they must completely fit in memory. The memory available for storing memory-optimized tables is a subset of the memory limits of your Business Critical service tier. The number of vCores for your deployment determines what percentage of memory is available for memory-optimized tables. Hyperscale supports a subset of In-Memory OLTP objects, including memory-optimized table types, table variables, and natively compiled modules but not memory-optimized tables.

Note

Memory-optimized tables require a memory-optimized filegroup. Azure SQL creates this filegroup for any databases even if it is not a Business Critical (BC) service tier. This way, if you move to BC, the filegroup is set up for memory-optimized tables.

New to In-Memory OLTP? Start with our documentation at [`https://learn.microsoft.com/sql/relational-databases/in-memory-oltp/overview-and-usage-scenarios`](https://learn.microsoft.com/sql/relational-databases/in-memory-oltp/overview-and-usage-scenarios).

### Partitions

Partitions are often used with SQL Server for tables with many rows to improve performance by subdividing data by a column in the table. Consider these points for partitions and Azure SQL:

*   Partitions are supported for Azure SQL Database and Managed Instance.

*   You can only use filegroups with partitions with Azure SQL Managed Instance. (Remember Azure SQL Database only has a primary partition and filegroup, while Managed Instance supported user-defined filegroups).

Need a primer for partitions? Start with this documentation page at [`https://learn.microsoft.com/sql/relational-databases/partitions/partitioned-tables-and-indexes`](https://learn.microsoft.com/sql/relational-databases/partitions/partitioned-tables-and-indexes).

Note

There are some interesting *partitioning* techniques with Azure SQL Database not associated with SQL partitions you may want to look at as a developer. Read more at [`https://learn.microsoft.com/azure/architecture/best-practices/data-partitioning-strategies#partitioning-azure-sql-database`](https://learn.microsoft.com/azure/architecture/best-practices/data-partitioning-strategies#partitioning-azure-sql-database).

### SQL Server 2022 Enhancements

SQL Server 2022 includes several enhancements related to performance that are also available in Azure SQL Managed Instance and Azure SQL Database:

*   **Intelligent Query Processing** (IQP) improvements

    IQP ([`https://aka.ms/iqp`](https://aka.ms/iqp)) is a series of capabilities built into the query processor to make your T-SQL queries run faster or maintain performance with no code changes.

    Azure SQL Managed Instance and Database include all the features for IQP from SQL Server 2017 and 2019\. In addition, IQP capabilities from SQL Server 2022 are available in Azure SQL. Learn more in the last section of the chapter titled “Intelligent Performance.”

*   **Query Store hints**

    Plan guides have been an effective way to force or shape a query plan independent of the query sent from an application. But there are limits to this feature, and it can be complex to use.

    Query Store hints allow you to “add” query hints to any incoming query from an application to shape or change the query plan. These hints are persisted in the Query Store, allowing you to force a query plan based on a series of hints. Some new IQP capabilities use Query Store hints behind the scenes. Learn more about [`https://aka.ms/querystorehints`](https://aka.ms/querystorehints).

Note

It is important to know that some “hidden gem” capabilities like merry-go-round scans and buffer pool ramp-up are all used behind the scenes for all editions for Azure SQL. Learn more at [`https://learn.microsoft.com/sql/relational-databases/reading-page`](https://learn.microsoft.com/sql/relational-databases/reading-page).

### Intelligent Performance

Over the past few releases of SQL Server, we have been striving to provide built-in capabilities to enhance performance without you making application changes. Our goal is to use data and automation to make smart decision to make your queries run faster. We call this *intelligent performance*. These capabilities exist in Azure SQL, but we go further in the cloud. We use the power of the cloud to offer even more. You learn more details about intelligent performance for Azure SQL in the final section of this chapter (IQP is part of that story).

## Configuring and Maintaining for Performance

In Chapter [5](#496204_2_En_5_Chapter.xhtml) of this book, I described many of the options to configure an Azure SQL Managed Instance and Database. There are some configuration options that can affect performance worth diving deeper into. This includes the tempdb database, configuring database options, files and filegroups, max degree of parallelism, and Resource Governor. In addition, it is worth reviewing the various tasks you would go through to maintain indexes and statistics for database for Azure SQL as compared to SQL Server.

### Tempdb

The tempdb database is an important shared resource used by applications. Ensuring the right configuration of tempdb can affect your ability to deliver consistent performance. Tempdb is used the same with Azure SQL like SQL Server, but your ability to configure tempdb is different, including placement of files, the number and size of files, tempdb size, and tempdb configuration options.

In Azure SQL Database, tempdb files are always automatically stored on local SSD drives, so I/O performance should not be an issue.

SQL Server professionals often use more than one database file to partition allocations for tempdb tables. For Azure SQL Database, the number of files are scaled with the number of vCores (e.g., 2 vCores = 4 files) with a max of 16\. The number of files is not configurable through T-SQL against tempdb but by *changing the deployment option*. The maximum size of the tempdb database is scaled per number of vCores.

I described in Chapter [5](#496204_2_En_5_Chapter.xhtml) that you get some configuration options for tempdb in Azure SQL Managed Instance, including expanding the number of files, the logical name of the files, and the autogrow increment size value.

Tempdb database option MIXED_PAGE_ALLOCATION is set to OFF and AUTOGROW_ALL_FILES is set to ON. This cannot be configured, but they are the recommended defaults as with SQL Server.

### Database Configuration

As I described in Chapter [5](#496204_2_En_5_Chapter.xhtml), about every database configuration option is available to you with Azure SQL as it is with SQL Server through ALTER DATABASE and ALTER DATABASE SCOPED configuration. Consult the documentation at [`https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql`](https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql) and [`https://learn.microsoft.com/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql`](https://learn.microsoft.com/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql). You will see later in this chapter there are new options specific to Azure SQL from ALTER DATABASE.

For performance, one database option that is not available to change is the *recovery model* of the database. The default is full recovery and cannot be modified. I often see the community compare OLTP performance for SQL Server and Azure SQL but don’t use full recovery which can have an impact on performance of scenarios like bulk insert. This ensures your database can meet Azure Service-Level Agreements (SLAs). Therefore, minimal logging for bulk operations is not supported. Minimal logging for bulk operations is supported for tempdb.

### Files and Filegroups

SQL Server professionals often use files and filegroups to improve I/O performance through physical file placement. Azure SQL does not allow users to place files on specific disk systems. However, Azure SQL has resource commitments for I/O performance with regard to rates, IOPS, and latencies, so abstracting the user from physical file placement can be a benefit.

Azure SQL Database only has one database file (Hyperscale may have several), and the size is configured through Azure interfaces. There is no functionality to create additional files, but again you do not need to worry about this given IOPS and I/O latency commitments.

Note

Hyperscale has a unique architecture and may create one or more files upon initial deployment depending on your vCore choice. For example, for an 8-vCore deployment, I’ve seen Hyperscale create multiple files totaling 40GB. This implementation may change, and you shouldn’t rely on it. Hyperscale simply creates the files and size it needs to meet your requirements.

Azure SQL Managed Instance supports adding database files and configuring sizes but not physical placement of files. The number of files and file sizes for Azure SQL Managed Instance can be used to improve I/O performance. However, as I mentioned in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book, the NextGen General Purpose service tier uses a different I/O architecture so that you have control over I/O performance without having to add files or change sizes. In addition, user-defined filegroups are supported for Azure SQL Managed Instance for manageability purposes such as use with partitions and using commands like DBCC CHECKFILEGROUP.

### Max Degree of Parallelism

Max degree of parallelism (MAXDOP), which can affect the performance of individual queries, works the same in the engine for Azure SQL as SQL Server. The ability to configure MAXDOP may be important to deliver consistent performance in Azure SQL. You can configure MAXDOP in Azure SQL like SQL Server using the following techniques:

*   ALTER DATABASE SCOPED CONFIGURATION to configure MAXDOP is supported for Azure SQL.

*   sp_configure for “max degree of parallelism” is supported for Managed Instance.

*   MAXDOP query hints are fully supported, including Query Store hints.

*   Configuring MAXDOP with Resource Governor is supported for Managed Instance.

Read more about MAXDOP at [`https://learn.microsoft.com/sql/database-engine/configure-windows/configure-the-max-degree-of-parallelism-server-configuration-option`](https://learn.microsoft.com/sql/database-engine/configure-windows/configure-the-max-degree-of-parallelism-server-configuration-option).

You will learn more in the section titled “Intelligent Performance” in this chapter a feature of IQP called **DOP feedback** which can assist you in dynamically allowing the SQL engine to adjust the DOP of a query.

### Resource Governor

Resource Governor is a feature in SQL Server that can be used to control resource usage for workloads through I/O, CPU, and memory. While Resource Governor is used behind the scenes for Azure SQL Database, it is supported for Azure SQL Managed Instance for user-defined workload groups and pools. If you would like to use Resource Governor in Azure SQL Managed Instance, consult our documentation at [`https://learn.microsoft.com/sql/relational-databases/resource-governor/resource-governor`](https://learn.microsoft.com/sql/relational-databases/resource-governor/resource-governor).

### Maintaining Indexes

Unfortunately, indexes for SQL don’t just maintain themselves, and they do occasionally need maintenance. In fairness, index maintenance (specifically rebuild or reorganization) does not have a single answer. I have seen many customers perform too often a rebuild or reorganization when it is not necessary. Likewise, there can be many times where these operations can help performance. You might consider looking at our documentation on index fragmentation as one reason why index maintenance can make sense: [`https://learn.microsoft.com/sql/relational-databases/indexes/reorganize-and-rebuild-indexes`](https://learn.microsoft.com/sql/relational-databases/indexes/reorganize-and-rebuild-indexes).

Note

I’m not telling the complete truth. For Azure SQL, there is a solution here that can help with decisions on building or dropping indexes. But I will not get too far ahead. The tale of that story is at the end of the chapter.

Indexes for SQL Server occasionally need to be reorganized and sometimes rebuilt. Azure SQL supports all the options you have for SQL Server to reorganize and rebuild indexes, including online and resumable indexes.

Online and resumable index operations can be extremely important to maintain maximum application availability. Read all about these capabilities at [`https://learn.microsoft.com/sql/relational-databases/indexes/guidelines-for-online-index-operations`](https://learn.microsoft.com/sql/relational-databases/indexes/guidelines-for-online-index-operations).

### Maintaining Statistics

Correct statistics can be the lifeblood for query performance. SQL Server offers options to automatically keep statistics up to date based on database modification, and Azure SQL supports all those options. Our documentation has a very detailed explanation on how statistics are used for query performance at [`https://learn.microsoft.com/sql/relational-databases/statistics/statistics`](https://learn.microsoft.com/sql/relational-databases/statistics/statistics).

One interesting aspect to automatic statistics updates is a database scoped configuration we specifically introduced for Azure SQL to help improve application availability. You can read about this in great detail from a blog post from my colleague Dimitri Furman at [`https://techcommunity.microsoft.com/t5/azure-sql-database/improving-concurrency-of-asynchronous-statistics-update/ba-p/1441687`](https://techcommunity.microsoft.com/t5/azure-sql-database/improving-concurrency-of-asynchronous-statistics-update/ba-p/1441687).

## Monitoring and Troubleshooting Performance

If you want to ensure you have the best performance for a SQL application, you need to learn how to monitor and troubleshoot performance scenarios. Azure SQL comes with the performance tools and capabilities of SQL Server to help you with this task. This includes tools from the Azure ecosystem as well as capabilities built into the SQL Server engine that powers Azure SQL.

In this part of the chapter of the book, you will learn not just monitoring capabilities but how to apply them to performance scenarios for Azure SQL, including examples.

### Monitoring Tools and Capabilities

Azure provides tools outside of SQL integrated with the Azure ecosystem called Azure Monitor.

Are you used to using Dynamic Management Views (DMV) and Extended Events? Azure SQL has what you need. Do you need to debug query plans? Azure SQL has all the capabilities of SQL Server, including Lightweight Query Profiling and *showplan* details.

Query Store has become the bedrock for performance tuning, and it is on by default in Azure SQL. The Azure portal includes visualizations, such as Query Performance Insight, to view Query Store data without needing tools like SSMS.

In addition, since the first edition of this, two new breakthrough tools can make a dramatic difference in saving you time and effort: Database Watcher and Microsoft Copilot skills in Azure SQL Database.

All this lines up to be a formidable set of tools and capabilities to help you monitor and troubleshoot performance for Azure SQL.

#### Azure Monitor

Azure Monitor is part of the Azure ecosystem, and Azure SQL is integrated to support Azure Metrics, Alerts, and Logs. Azure Monitor data can be visualized in the Azure Portal or accessed by applications through Azure Event Hub or APIs. An example of why Azure Monitor is important is accessing resource usage metrics for Azure SQL outside of SQL Server tools much like Windows Performance Monitor. Read more about how to use Azure Monitor with Azure SQL in the Azure portal at [`https://learn.microsoft.com/azure/azure-sql/database/monitor-tune-overview?view=azuresql#monitoring-and-tuning-capabilities-in-the-azure-portal`](https://learn.microsoft.com/azure/azure-sql/database/monitor-tune-overview?view=azuresql#monitoring-and-tuning-capabilities-in-the-azure-portal).

#### Dynamic Management Views (DMVs)

Azure SQL provides the same DMV infrastructure as with SQL Server with a few differences. DMVs are a crucial aspect to performance monitoring since you can view key SQL Server performance data using standard T-SQL queries. Information such as active queries, resource usage, query plans, and resource wait types are available with DMVs. Learn more details about DMVs with Azure SQL later in this chapter.

#### Extended Events (XEvent)

Azure SQL provides the same Extended Events infrastructure as with SQL Server. Extended Events is a method to trace key events of execution within SQL Server that powers Azure SQL. For performance, Extended Events allow you to trace the execution of individual queries. Learn more details about Extended Events with Azure SQL later in this chapter.

#### Lightweight Query Profiling

Lightweight Query Profiling is a capability to examine the query plan and running state of an active query. This is a key feature to debug query performance for long-running statements as they are running. This capability cuts down the time for you to solve performance problems vs. using tools like Extended Events to trace query performance. Lightweight Query Profiling is accessed through DMVs and is on by default for Azure SQL just like SQL Server 2022\. Read more about Lightweight Query Profiling at [`https://learn.microsoft.com/sql/relational-databases/performance/query-profiling-infrastructure?view=sql-server-ver15#lwp`](https://learn.microsoft.com/sql/relational-databases/performance/query-profiling-infrastructure?view=sql-server-ver15#lwp).

#### Query Plan Debugging

In some situations, you may need additional details about query performance for an individual T-SQL statement. T-SQL SET statements such as SHOWPLAN and STATISTICS can provide these details and are fully supported for Azure SQL as they are for SQL Server. A good example of using SET statements for query plan debugging can be found at [`https://learn.microsoft.com/sql/t-sql/statements/set-statistics-profile-transact-sql`](https://learn.microsoft.com/sql/t-sql/statements/set-statistics-profile-transact-sql). In addition, looking at plans in a graphical or XML format is always helpful and completely works for Azure SQL. Learn more at [`https://learn.microsoft.com/sql/relational-databases/performance/display-the-estimated-execution-plan`](https://learn.microsoft.com/sql/relational-databases/performance/display-the-estimated-execution-plan).

#### Query Store

Query Store is a historical record of performance execution for queries stored in the user database. Query Store is on by default for Azure SQL and is used to provide capabilities such as Automatic Plan Correction and Automatic Tuning. SQL Server Management Studio (SSMS) reports for Query Store are available for Azure SQL. These reports can be used to find top resource consuming queries including query plan differences and top wait types to look at resource wait scenarios.

Query Store is also now used in certain Intelligent Query Processing (IQP) scenarios to store feedback information used to improve performance.

I will show you an example of using the Query Store in this chapter with Azure SQL. If you have never seen or used Query Store, start reading at [`https://learn.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store`](https://learn.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store).

#### Performance Visualization in Azure Portal

For Azure SQL Database, we have integrated Query Store performance information into the Azure Portal through visualizations. This way, you can see some of the same information for Query Store as you would with a client tool like SSMS by using the Azure Portal with an option called **Query Performance Insight**. I’ll show you an example of using these visuals in the portal later in the chapter. For now, to get started using it, check out our documentation at [`https://learn.microsoft.com/azure/azure-sql/database/query-performance-insight-use`](https://learn.microsoft.com/azure/azure-sql/database/query-performance-insight-use).

#### Database Watcher

In all honesty, we have made a few attempts at a monitoring solution in Azure that provides rich and visual experiences to monitor Azure SQL, but none of them have stuck.

I believe Database Watcher, a new cloud service for monitoring SQL, will be different (in preview at the time of the writing of this book) because of the following:

*   The service is separate from SQL using built-in telemetry though metadata like DMVs and Query Store and persisting the information a Kusto cluster.

*   The service comes by default with a rich set of visualizations that provide just about anything a SQL professional will need. This is why I call it my “perfmon in the cloud.”

*   The service allows you to collect data across multiple Azure SQL Managed Instances and Databases. And some of the visualizations allow you to look across these objects. For example, you can see top queries across database (effectively across Query Stores).

*   The release of this service was managed by my colleague Dimitri Furman whom I consider one of the top experts for SQL within Microsoft. And you will see its qualities shine in this chapter.

Figure [7-1](#496204_2_En_7_Chapter.xhtml#Fig1) shows the overall architecture of Database Watcher.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig1_HTML.jpg)

Figure 7-1

Database Watcher architecture

The service connects to targets, which are your Azure SQL Managed Instances and Databases. It collects data from these targets and stores in a Kusto cluster. The service comes with some very rich visualizations, but you can also use K-SQL or T-SQL to query the cluster. In addition, you can store the telemetry in Microsoft Fabric real-time analytics to build solutions like Power BI reports.

You can get started with Database Watcher at [`https://aka.ms/dbwatcher`](https://aka.ms/dbwatcher) but also see Anna Hoffman and I talk about this at one of her famous Data Exposed shows at [`https://learn.microsoft.com/shows/data-exposed/database-watcher-your-perfmon-in-the-cloud-data-exposed`](https://learn.microsoft.com/shows/data-exposed/database-watcher-your-perfmon-in-the-cloud-data-exposed).

#### Microsoft Copilot Skills in Azure SQL Database

Around the later summer of 2023, I found myself involved in a new project at Microsoft for an AI Assistant for SQL under the umbrella of Copilot for Azure. The concept was to provide a *sidecar* chat experience in the Azure Portal. Around this time, Microsoft had already launched several Copilot experiences, including Microsoft Edge, mobile apps, and Microsoft 365\. A Copilot is an example of a GenAI application in the form of a prompt application using language models such as GPT.

For our time, I remember being in early meetings on the project, and one of the goals was to build a rich experience *using the context* of an Azure SQL Database. If you navigated in the Azure Portal to your database, you could chat with Copilot using *skills* that were **specific to your database**. I remember telling our team that SQL has such rich built-in telemetry, including catalog views, DMVs, Extended Events, and Query Store, that we could build something truly industry differentiating.

In one of our first meetings, they asked me about some scenarios we should try to solve. Performance was at the top of my list. Why? Because performance troubleshooting involves precision for something that is normally imprecise. In fact, I distinctly remember telling the team I should be able to say to Copilot “My database is slow,” and it would use our engine telemetry based on the permissions of the user to narrow down the problem quickly. I showed them the “Running vs. Waiting” concept that I used in the Special Edition: Performance Troubleshooting Azure SQL for Beginners ([`https://aka.ms/performanceseries`](https://aka.ms/performanceseries)). I told the team to go through all the examples in the videos (for which the code can be found at [`https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting`](https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting)). I said that if I can chat with Copilot to solve these types of scenarios, we will have built something no one else has inside or outside of Microsoft.

The first look at this project was when Anna Hoffman and I had one of our famous duels in the PASS Summit Keynote in 2023 (she always seems to win?), which you can view at [`https://youtu.be/M80Ze7nyvVc`](https://youtu.be/M80Ze7nyvVc). And the project accelerated when the famous Joe Sack rejoined Microsoft and took the lead PM role for the project (who just happens to also be the tech reviewer for this book). The entire development team, led by the incredible Katherine Lin, pushed hard to get to a preview of the product which we unveiled at SQLBits in March 2024 (you can watch Joe and I present this at SQLBits at [`https://sqlbits.com/sessions/event2024/Welcome_to_the_world_of_SQL_Copilots`](https://sqlbits.com/sessions/event2024/Welcome_to_the_world_of_SQL_Copilots)).

Joe has built a very nice architecture diagram, showing how this technology works as seen in Figure [7-2](#496204_2_En_7_Chapter.xhtml#Fig2).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig2_HTML.jpg)

Figure 7-2

The “SQL Copilot” architecture

You can get started with Microsoft Copilot skills in Azure SQL Database at [`https://aka.ms/sqlcopilot`](https://aka.ms/sqlcopilot).

Copilot can do more than just help you with performance. To first show you how Copilot understands your database context, I used a prompt like in Figure [7-3](#496204_2_En_7_Chapter.xhtml#Fig3).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig3_HTML.jpg)

Figure 7-3

SQL Copilot database context

Figure [7-4](#496204_2_En_7_Chapter.xhtml#Fig4) shows the results.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig4_HTML.jpg)

Figure 7-4

SQL Copilot database context results

You can see that Copilot “understands” your current database context. Notice it also provides suggestions for other possible related prompts you may want to use. Figure [7-5](#496204_2_En_7_Chapter.xhtml#Fig5) shows what types of possible skills “SQL Copilot” can do in the Azure Portal.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig5_HTML.jpg)

Figure 7-5

SQL Copilot skills

You can see many capabilities exist more than just performance, but that is an area where this capability shines.

Note

Copilot in Azure “understands” SQL even if you are not in the context of your database, but its responses will be more from documentation searches. As you can see from Figure [7-5](#496204_2_En_7_Chapter.xhtml#Fig5), you can also use Copilot to help you search Microsoft documentation even in the database context.

I’ll show you some examples of using both Database Watcher and Microsoft Copilot skills in Azure SQL Database to solve some performance problems later in this chapter. I’ll also show you another part of the Copilot story in Chapter [10](#496204_2_En_10_Chapter.xhtml) of the book, which is natural language to SQL (NL2SQL).

### Dive into DMVs and Extended Events

Dynamic Management Views (DMVs) and Extended Events (XEvent) have been the *bedrock* of diagnostics, including performance monitoring and troubleshooting for SQL Server for many years. I can truthfully tell you that DMV and XEvent technology all started with the brains of folks like Slava Oks and Conor Cunningham so many years ago. Many on the engineering team have worked, molded, and shaped these technologies, but I remember being there from the beginning with Slava and my colleague for many years Robert Dorr and Keith Elmore working on these technologies when we were in Microsoft support together. DMVs and XEvent are very important technologies to support performance monitoring and troubleshooting for Azure SQL because Azure SQL is powered by the SQL Server engine, and the SQL Server engine powers Azure SQL Managed Instance and Database.

Let’s dive a bit deeper into what DMV and XEvent capabilities are the same and new for Azure SQL vs. SQL Server.

#### DMVs Deep Dive

Let’s dive deeper into DMV for Azure SQL vs. SQL Server across Azure SQL Managed Instance and Database.

##### Azure SQL Managed Instance

All DMVs for SQL Server are available for Managed Instance. Key DMVs like **sys.dm_exec_requests** and **sys.dm_os_wait_stats** are commonly used to examine query performance.

One DMV is specific to Azure called **sys.server_resource_stats** and shows historical resource usage for the Managed Instance. This is an important DMV to see resource usage since you do not have direct access to OS tools like Performance Monitor. You can learn more about sys.server_resource_stats at [`https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-server-resource-stats-azure-sql-database`](https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-server-resource-stats-azure-sql-database).

##### Azure SQL Database

*Most* of the common DMVs you need for performance, including **sys.dm_exec_requests** and **sys.dm_os_wait_stats**, are available. It is important to know that these DMVs only provide information specific to the database and not across all databases for a logical server.

**sys.dm_db_resource_stats** is a DMV specific to Azure SQL Database and can be used to view a history of resource usage for the database. Use this DMV similar to how you would use sys.server_resource_stats for a Managed Instance. I will show you how to use this DMV in an example later in this chapter. For now, you can read more at [`https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-db-resource-stats-azure-sql-database`](https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-db-resource-stats-azure-sql-database).

**sys.elastic_pool_resource_stats** is like sys.dm_db_resource_stats but can be used to view resource usage for elastic pool databases.

##### DMVs You Will Need

There are a few DMVs worth calling out that you will need to solve certain performance scenarios for Azure SQL, including the following:

*   **sys.dm_io_virtual_file_stats** is important for Azure SQL since you don't have direct access to operating system metrics for I/O performance per file.

*   **sys.dm_os_performance_counters** is available for both Azure SQL Database and Managed Instance to see SQL Server common performance metrics. This can be used to view SQL Server performance counter information that is typically available in Windows Performance Monitor.

*   **sys.dm_instance_resource_governance** can be used to view resource limits for a Managed Instance. You can view this information to see what your expected resource limits should be without using the Azure portal.

*   **sys.dm_user_db_resource_governance** can be used to see common resource limits per the deployment option, service tier, and size for your Azure SQL Database deployment. You can view this information to see what your expected resource limits should be without using the Azure portal. I’ll show you an example of looking at this DMV in an example. For now, you can read more at [`https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-user-db-resource-governor-azure-sql-database`](https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-user-db-resource-governor-azure-sql-database).

##### DMVs for Deep Troubleshooting

These DMVs provide deeper insight into resource limits and resource governance for Azure SQL. They are not meant to be used for common scenarios but might be helpful when looking deep into complex performance problems:

*   **sys.dm_os_out_of_memory_events**

*   **sys.dm_user_db_resource_governance_internal (Managed Instance only)**

*   **sys.dm_resource_governor_resource_pools_history_ex**

*   **sys.dm_resource_governor_workload_groups_history_ex**

*Geek out* with these DMVs. The first DMV was introduced since the first edition of this book and can help to troubleshoot out of memory errors. The last three DMVs provide historical information across time (right now about 30 minutes). Be warned when using these DMVs. We kind of built these for our internal purposes to debug issues with Azure to look at problems like background activity vs. user load. So don’t be surprised if we change these to suit our needs to ensure we provide a great database service.

#### XEvent at Your Service

Extended Events (XEvent) was introduced as the new tracing mechanism for SQL Server in SQL Server 2005 to replace SQL Trace. XEvent today supports some 1800+ trace points in the SQL Server engine. XEvent powers other capabilities, including SQL Audit and SQL Threat Detection for Microsoft Defender.

##### Extended Events for Azure SQL Managed Instance

Extended Events can be used for Azure SQL Managed Instance just like SQL Server by creating sessions and using events, actions, and targets. Keep these important points in mind when creating Extended Event sessions:

*   All events, targets, and actions are supported.

*   File targets are supported with Azure Blob Storage since you don't have access to the underlying operating system disks.

*   Some specific events are added for Managed Instance to trace events specific to the management and execution of the instance.

You can use SSMS or T-SQL to create and start sessions. You can use SSMS to view Extended Event session target data or the system function **sys.fn_xe_file_target_read_file**.

##### Extended Events for Azure SQL Database

Extended Events can be used for Azure SQL Database just like SQL Server by creating sessions and using events, actions, and targets. Keep these important points in mind when creating Extended Event sessions:

*   Most commonly used events and actions are supported. For example, the fundamental event sql_batch_completed is available to you. Azure SQL Database offers ~400 events vs. SQL Server (and Managed Instance) which has around 1800\. Use the DMV **sys.dm_xe_objects** to find out all objects available to you.

*   File, ring_buffer, and counter targets are supported.

*   File targets are supported with Azure Blob Storage since you don't have access to the underlying operating system disks. Here is a blog from the Azure Support team for a step-by-step process to set up Azure Blob Storage as a file target: [`https://techcommunity.microsoft.com/t5/azure-database-support-blog/extended-events-capture-step-by-step-walkthrough/ba-p/369013`](https://techcommunity.microsoft.com/t5/azure-database-support-blog/extended-events-capture-step-by-step-walkthrough/ba-p/369013).

You can use SSMS or T-SQL to create and start sessions. You can use SSMS to view Extended Event session target data or the system function **sys.fn_xe_file_target_read_file**.

One of the nice additions for Extended Events since the first edition of this book is the ability to use XE Profiler in SSMS for Azure SQL. Figure [7-6](#496204_2_En_7_Chapter.xhtml#Fig6) shows “live” Extended Event sessions with an Azure SQL Database.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig6_HTML.jpg)

Figure 7-6

XE Profiler with Azure SQL Database

It is important to know that any Extended Events fired for your sessions are specific to your database and not across the logical server. Therefore, we have a new set of catalog views such as **sys.database_event_sessions** (definitions) and DMVs such as **sys.dm_xe_database_sessions** (active sessions). Take a look through our documentation for a complete list of differences for XEvent between Azure SQL Database and SQL Server: [`https://learn.microsoft.com/azure/azure-sql/database/xevent-db-diff-from-svr`](https://learn.microsoft.com/azure/azure-sql/database/xevent-db-diff-from-svr).

### Performance Scenarios

In a galaxy, far, far away when I was in Microsoft Support, my longtime friend Keith Elmore was considered our expert on performance troubleshooting (he still is as far as I’m concerned). As we trained other support engineers, Keith came up with an idea that most SQL performance problems could be categorized as either **Running** or **Waiting**.

Note

Keith’s work led to a report called the Performance Dashboard reports. That report is now part of the Standard Reports for SQL Server Management Studio. Unfortunately, the report relies on some DMVs which are not exposed for Azure SQL Database. However, the reports will work for Managed Instance.

One way to look at this concept is with Figure [7-7](#496204_2_En_7_Chapter.xhtml#Fig7).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig7_HTML.jpg)

Figure 7-7

The Running vs. Waiting for SQL performance

Let’s take a look at more of the details of this figure from the perspective of performance scenarios.

Note

As you look at DMVs in this section, remember that for Azure SQL Database you are only looking at results for a specific database, not across all databases for the logical server.

#### Running vs. Waiting

Running or waiting scenarios can often be determined by looking at overall resource usage. For a standard SQL Server deployment, you might use tools such as Performance Monitor in Windows or top in Linux. For Azure SQL, you can use the following methods.

##### Azure Portal/PowerShell/Alerts

Azure Monitor has integrated metrics to view resource usage for Azure SQL. You can also set up alerts to look for resource usage conditions such as high CPU percent. Since we have integrated some Azure SQL performance data with Azure Monitor, having alerts is a huge advantage to snapping into the ecosystem. Read more about how to set up alerts with Azure Metrics at [`https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-create-metric-alert-rule`](https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-create-metric-alert-rule).

##### sys.dm_db_resource_stats

For Azure SQL Database, you can look at this DMV to see CPU, memory, and I/O resource usage for the database deployment. This DMV takes a snapshot of this data every 15 seconds. The reference for all columns in this DMV can be found at [`https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-db-resource-stats-azure-sql-database`](https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-db-resource-stats-azure-sql-database). I’ll use this DMV in an example later in this section.

Note

A DMV called **sys.resource_stats** works within the logical master to review resource stats for up to 14 days across all Azure Databases associated with the logical server. Learn more at [`https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-resource-stats-azure-sql-database`](https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-resource-stats-azure-sql-database).

##### sys.server_resource_stats

This DMV behaves just like sys.dm_db_resource_stats, but it used to see resource usage for the Managed Instance for CPU, memory, and I/O. This DMV also takes a snapshot every 15 seconds. You can find the complete reference for this DMV at [`https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-server-resource-stats-azure-sql-database`](https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-server-resource-stats-azure-sql-database).

#### Using Copilot to Get Started

One thing missing from Figure [7-7](#496204_2_En_7_Chapter.xhtml#Fig7) is Microsoft Copilot skills in Azure SQL Database. Copilot uses the resources listed above to help you narrow down a “Running vs. Waiting” problem using prompts.

Figure [7-8](#496204_2_En_7_Chapter.xhtml#Fig8) shows an example of using a prompt to find out the details of a performance problem with an Azure SQL Database and possible causes.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig8_HTML.jpg)

Figure 7-8

Using Copilot to narrow down a performance problem

I’ll show you some examples of Copilot detecting both a *running* and a *waiting* problem with this type of prompt. I’ll also use Copilot later in this chapter to see how it can help with some specific scenarios.

#### Running

If you have determined the problem is high CPU utilization, this is called a *running scenario*. A running scenario can involve queries that consume resources through compilation or execution. Further analysis to determine a solution can be done by using these tools.

##### Query Store

Query Store was introduced with SQL Server 2016 and has been one of the most game-changing capabilities for performance analysis. Use the Top Consuming Resource reports in SSMS, Query Store catalog views, or Query Performance Insight in the Azure Portal (Azure SQL Database only) to find which queries are consuming the most CPU resources. Do you need a primer for Query Store? Start with our documentation at [`https://learn.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store`](https://learn.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store).

##### sys.dm_exec_requests

This DMV has become perhaps the most popular DMV to use for SQL Server in history. This DMV displays a snapshot of all current active requests, which could be a T-SQL query or background task. Use this DMV in Azure SQL to get a snapshot of the state of active queries. Look for queries with a state of RUNNABLE and a wait type of SOS_SCHEDULER_YIELD to see if you have enough CPU capacity. Get the complete reference for this DMV at [`https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-requests-transact-sql`](https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-requests-transact-sql).

##### sys.dm_exec_query_stats

This DMV can be used much like Query Store to find top resource consuming queries but is only available for query plans that are cached where Query Store provides a persistent historical record of performance. This DMV also allows you to find the query plan for a cached query. Get the complete reference at [`https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-query-stats-transact-sql`](https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-query-stats-transact-sql).

Since Query Store is not yet available for readable secondaries (this is coming I promise), this DMV could be useful for those scenarios.

##### sys.dm_exec_procedure_stats

This DMV provides information much like sys.dm_exec_query_stats except the performance information can be viewed at the stored procedure level. Get the complete reference at [`https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-procedure-stats-transact-sql`](https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-procedure-stats-transact-sql).

Once you determine what query or queries are consuming the most resources, you may have to examine whether you have enough CPU resources for your workload or debug query plans with tools like Lightweight Query Profiling, SET statements, Query Store, or Extended Event tracing.

##### Microsoft Copilot Skills in Azure SQL Database

You saw earlier in the chapter I showed you a prompt “My database is slow” to help narrow down a performance problem. Figure [7-9](#496204_2_En_7_Chapter.xhtml#Fig9) shows an example of where this prompt helped narrow down a high CPU problem.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig9_HTML.jpg)

Figure 7-9

Copilot detects a high CPU problem

Notice the details on how much CPU was consumed for an extended period of time and the query that was causing most of the CPU consumption. You can also see I was guided to use another prompt to optimize the query. When I selected that, you can see what happened in Figure [7-10](#496204_2_En_7_Chapter.xhtml#Fig10).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig10_HTML.jpg)

Figure 7-10

Copilot detects a missing index

Copilot has used built-in SQL telemetry to show a missing index would reduce CPU and increase performance. If you want to try out your own scenario and see Copilot detect this, use the example at [`https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting/highcpu_missingindex`](https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting/highcpu_missingindex).

I’ve also used Copilot to detect scenarios like anti-pattern queries. The full set of documented scenarios can be found at [`https://aka.ms/sqlcopilot`](https://aka.ms/sqlcopilot) (we are always looking to increase the skills of this Copilot – we are just getting started!).

I’ll show another *running* problem later in this chapter with an end-to-end scenario and see how Copilot can help.

##### Database Watcher

I’ve mentioned Database Watcher as a new cloud service to give you incredible insights into your Azure SQL workload. One capability Database Watcher can help you with is a high CPU scenario via a heatmap. You can see this example from Figure [7-11](#496204_2_En_7_Chapter.xhtml#Fig11), a heatmap of a high CPU scenario for a database target.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig11_HTML.jpg)

Figure 7-11

Database Watcher heatmap to show a high CPU scenario

You can then drill into the “hot” database to see more details. But Database Watcher can also show you the query causing the problem when looking at “Top Queries” across all targets as you can see in Figure [7-12](#496204_2_En_7_Chapter.xhtml#Fig12).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig12_HTML.jpg)

Figure 7-12

Database Watcher top queries

Besides being able to see top queries across databases (Query Stores), you can see in this example the top query for the database adw1 is “red,” indicating it has high CPU, and there is a suggestion that an index is missing.

You can set up Database Watcher ([`https://aka.ms/dbwatcher`](https://aka.ms/dbwatcher)) and use Copilot with this example to see a high CPU missing index scenario: [`https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting/highcpu_missingindex`](https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting/highcpu_missingindex). You will see how to use these tools with a different high CPU scenario later in this chapter.

#### Waiting

If your problem doesn't appear to be a high CPU resource usage, it could be the performance problem involves waiting on a resource. Scenarios involving waiting on resources include

*   **I/O waits** – This includes wait types such as PAGEIOLATCH latches (wait on database I/O) and WRITELOG (wait on transaction log I/O). These waits can still be seen in Azure SQL just like SQL Server. Long duration for these indicates an I/O bottleneck (which could mean you need to make changes to a different service tier or increase vCores).

*   **Lock waits** – These waits show up as standard “blocking” problems.

*   **Latch waits** – This includes PAGELATCH (“hot” page) or even just LATCH (concurrency on an internal structure).

*   **Buffer pool limits** – If you run out of buffer pool resources, you might run into unexpected PAGEIOLATCH waits.

*   **Memory grants** – A high number of concurrent queries that need memory grants or large grants (could be from overestimation) could result in RESOURCE_SEMAPHORE waits.

*   **Plan cache eviction** – If you don’t have enough plan cache and plans get evicted, this could lead to higher compile times (which could result in higher CPU) or RUNNABLE status with SOS_SCHEDULER_YIELD because there is not enough CPU capacity to handle compiles. You also might see them waiting on locks for schema to compile queries.

To perform analysis on waiting scenarios, you typically look at the following tools.

##### sys.dm_os_wait_stats

Use this DMV to see what the top wait types for the database or instance are. This can guide you on what action to take next depending on the top wait types. Remember that for Azure SQL Database, these are just waits for the database, not across all database on the logical server. You can view the complete reference at [`https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-os-wait-stats-transact-sql`](https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-os-wait-stats-transact-sql).

Note

There is a DMV specific to Azure SQL Database called **sys.dm_db_wait_stats** (it also works with Managed Instance, but I don’t recommend using it given you are looking at the instance), which only shows waits specific for the database. You might find this useful, but sys.dm_os_wait_stats will show all waits for the dedicated instance hosting your Azure SQL Database.

##### sys.dm_exec_requests

Use this DMV to find specific wait types for active queries to see what resource they are waiting on. This could be a standard blocking scenario waiting on locks from other users.

##### sys.dm_os_waiting_tasks

Queries that use parallelism use multiple tasks for a given query, so you may need to use this DMV to find wait types for a given task for a specific query.

##### Query Store

Query Store provides reports and catalog views that show an aggregation of the top waits for query plan execution. The catalog view to see waits in Query Store is called **sys.query_store_wait_stats**, which you can read more about at [`https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-query-store-wait-stats-transact-sql`](https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-query-store-wait-stats-transact-sql). It is important to know that a **wait of CPU** is equivalent to a **running** problem.

Tip

Extended Events can be used for any running or waiting scenarios but requires you to set up an Extended Event session to trace queries and can be considered a *heavier* method to debug a performance problem.

##### Microsoft Copilot Skills in Azure SQL Database and Database Watcher

Like for a high CPU (running) problem, Copilot and Database Watcher can also help with waiting problems. I took the example at [`https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting/waiting_blocking`](https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting/waiting_blocking) and prompted Copilot “My database is slow.” Figure [7-13](#496204_2_En_7_Chapter.xhtml#Fig13) shows the results.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig13_HTML.jpg)

Figure 7-13

SQL Copilot detects a blocking problem

Here is the power of Copilot, and it is taking the “Running vs. Waiting” approach. I sent a prompt for a very vague request, and it was able to use built-in telemetry to check for high CPU. When it didn’t find this, it checked for a waiting problem and found this blocking chain. Many blocking problems are due to an open transaction left open by a session, so I sent a prompt right after this as part of a *conversation* with Copilot “Does session 110 have an open transaction?” and Figure [7-14](#496204_2_En_7_Chapter.xhtml#Fig14) shows the results.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig14_HTML.jpg)

Figure 7-14

SQL Copilot detects an open transaction causing blocking

I can terminate the lead session but will also have to investigate the cause of the application causing blocking. What if I also want to see if there is a trend to this problem? This is where Database Watcher is incredible.

Database Watcher can capture much of its insights across time. One of these is SQL Activity, including blocking chains. Figure [7-15](#496204_2_En_7_Chapter.xhtml#Fig15) shows an example.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig15_HTML.jpg)

Figure 7-15

Database Watcher historical SQL activity

You can see from this chart there is a trend where blocking persists and then ends but comes back. I can click a bar on this chart to see the details as you see in Figure [7-16](#496204_2_En_7_Chapter.xhtml#Fig16).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig16_HTML.jpg)

Figure 7-16

Database Watcher historical blocking chain

#### Performance Example

Let’s look at an example of a performance scenario to show how to use tools and capabilities I’ve discussed in this section to identify a performance scenario. I’ll use the following resources for this exercise:

*   The logical server **bwsqllogicalserver** as well as the database **bwadw** I’ve deployed earlier. This database was deployed as a Hyperscale, Gen5, 2 vCore database.

*   The Azure VM called **bwsql2022**. I left my security settings from Chapter [6](#496204_2_En_6_Chapter.xhtml), so this VM has access to the logical server and database.

*   I’ll use SQL Server Management Studio (SSMS) to run some queries and look at Query Store reports.

    **Tip** If you connect with SSMS to an Azure SQL Database logical server and with SSMS choose a specific database, Object Explorer will only show you the logical master and your database. If you connect to the logical master with a server admin account, Object Explorer will show you all databases.

*   I’ll use the Azure portal to view Azure Metrics.

*   I’ll also set up Database Watcher for this target database (get started quickly at [`https://learn.microsoft.com/azure/azure-sql/database-watcher-quickstart`](https://learn.microsoft.com/azure/azure-sql/database-watcher-quickstart)) and use Microsoft Copilot skills in Azure SQL Database which is enabled by default in my subscription (if you don’t see it enabled, check the following documentation: [`https://learn.microsoft.com/azure/azure-sql/copilot/copilot-azure-sql-overview?view=azuresql#enable-microsoft-copilot-in-your-azure-tenant`](https://learn.microsoft.com/azure/azure-sql/copilot/copilot-azure-sql-overview?view=azuresql#enable-microsoft-copilot-in-your-azure-tenant)).

*   For this chapter, I have script files you can use for several of the examples. You can find scripts for this example in the **ch7_performance\monitor_and_scale** folder for the source files included for the book. I will also use the very popular tool ostress.exe for exercises in this chapter which comes with the RML Utilities. You can download RML from [`https://www.microsoft.com/en-us/download/details.aspx?id=4511`](https://www.microsoft.com/en-us/download/details.aspx?id=4511). Make sure to put the folder where RML gets installed in your system path (which is by default C:\Program Files\Microsoft Corporation\RMLUtils).

Let’s go through an example in a step-by-step fashion:

1.  Set up to monitor Azure SQL Database with a DMV query.

    **Tip** To open a script file in the context of a database in SSMS, click the database in Object Explorer and then use the File/Open menu in SSMS.

2.  Launch SQL Server Management Studio (SSMS) and load a query in the context of the database to monitor the Dynamic Management View (DMV) **sys.dm_exec_requests** from the script **dmexecrequests.sql** which looks like the following:

    ```
    SELECT er.session_id, er.status, er.command, er.wait_type, er.last_wait_type, er.wait_resource, er.wait_time
    FROM sys.dm_exec_requests er
    INNER JOIN sys.dm_exec_sessions es
    ON er.session_id = es.session_id
    AND es.is_user_process = 1;
    ```

3.  Load another query to observe resource usage.

    In another session for SSMS in the context of the database, load a query to monitor a Dynamic Management View (DMV) unique to Azure SQL Database called **sys.dm_db_resource_stats** from a script called **dmdbresourcestats.sql**:

```
SELECT * FROM sys.dm_db_resource_stats;
```

This DMV will track overall resource usage of your workload against Azure SQL Database such as CPU, I/O, and memory.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig17_HTML.jpg)

Figure 7-17

Viewing CPU utilization from Azure Metrics

1.  Edit the workload script.

    Edit the script **sqlworkload.cmd** (which will use the ostress.exe program).

    I’ll substitute my server, database, and password. The script will look like this (with your password):

    ```
    ostress.exe -Sbwsqllogicalserver.database.windows.net -itopcustomersales.sql -Usqladmin -dbwadw -P -n10 -r20 -q -T146
    ```

2.  Examine the T-SQL query we will use for the workload. You can find this T-SQL batch in the script **topcustomersales.sql**:

    ```
    DECLARE @x int
    DECLARE @y float
    SET @x = 0;
    WHILE (@x < 10000)
    BEGIN
    SELECT @y = sum(cast((soh.SubTotal*soh.TaxAmt*soh.TotalDue) as float))
    FROM SalesLT.Customer c
    INNER JOIN SalesLT.SalesOrderHeader soh
    ON c.CustomerID = soh.CustomerID
    INNER JOIN SalesLT.SalesOrderDetail sod
    ON soh.SalesOrderID = sod.SalesOrderID
    INNER JOIN SalesLT.Product p
    ON p.ProductID = sod.ProductID
    GROUP BY c.CompanyName
    ORDER BY c.CompanyName;
    SET @x = @x + 1;
    END
    GO
    ```

    This database is not large, so the query to retrieve customer and their associated sales information ordered by customers with the most sales shouldn't generate a large result set. It is possible to tune this query by reducing the number of columns from the result set, but these are needed for demonstration purposes of this activity. You will note in this query I don’t return any results to the client but assign values to a local variable. This will put all the CPU resources to run the query to the server.

3.  Now let’s run the workload and observe its performance and results from queries we loaded earlier. Run the workload by executing the **sqlworkload.cmd** script from a command shell or PowerShell. The script uses ostress to simulate ten concurrent users running the T-SQL batch.

4.  Now use the DMVs that you loaded to observe performance while this runs. First, run the query from **dmexecrequests.sql** five or six times in the query window from SSMS. You will see several users have status = RUNNABLE and last_wait_type = SOS_SCHEDULER_YIELD. This is a classic signature of not having enough CPU resources for a workload.

5.  Observe the results from the query **dmdbresourcestats.sql**. Run this query a few times and observe the results. You will see several rows with a value for **avg_cpu_percent** close to 100%. sys.dm_db_resource_stats takes a snapshot every 15 seconds of resource usage.

6.  Let the workload complete and take note of its duration. For me, it measured around 12–13 minutes to complete.

7.  Use Azure Monitor and metrics.

    Let’s look at this performance scenario through the lens of Azure Monitor and metrics. I’ll navigate to my database using the Azure portal. In the monitoring pane is an area called Monitoring and Key metrics. After my workload has run, my chart looks like Figure [7-17](#496204_2_En_7_Chapter.xhtml#Fig17).

    You can see on this screen the blue line showing high CPU (the lines to the left of the sustained high CPU was another brief test I did before the example). Notice the red line showing less than 100%. This is the CPU for the instance hosting the database. This number shows there is room to scale for my database which is a sign that I might not have enough resources to run this workload.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig18_HTML.jpg)

Figure 7-18

SSMS Query Store Top Consuming Queries

1.  Let’s use the Query Store now to dive deeper into the performance of the queries in this workload. In SSMS in the Object Explorer, load the Top Resource Consuming Queries as seen in Figure [7-18](#496204_2_En_7_Chapter.xhtml#Fig18).

    The default metric is Total Duration, so I changed this to CPU. You can see the query I ran for this example is the one consuming all the CPU.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig19_HTML.jpg)

Figure 7-19

Query Wait Statistics report from SSMS

1.  Query Store reports in SSMS also include top queries by waits. In this case, the CPU problem *is also a waiting problem* on CPU resources as I mentioned earlier. Figure [7-19](#496204_2_En_7_Chapter.xhtml#Fig19) shows this.

    Now consider the evidence. The workload consumes CPU resources for the database at 100% (in reality, it is more than 100%, but we are capped). The status of many requests is RUNNABLE, and the top wait type for the workload is SOS_SCHEDULER_YIELD. If the query cannot be changed, then the most likely scenario is that you don’t have enough CPU resources for your workload. Later in this chapter, we will use Azure interfaces to make this query run faster.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig23_HTML.jpg)

Figure 7-23

Wait statistics for a query in Database Watcher

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig22_HTML.jpg)

Figure 7-22

Query details in Database Watcher

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig21_HTML.jpg)

Figure 7-21

Top queries from Database Watcher

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig20_HTML.jpg)

Figure 7-20

Database Watcher resource utilization

1.  Let’s look now at what Database Watcher shows.

    What does Database Watcher show for this situation, and can it also help me narrow down the problem?

    The heatmap that you have seen earlier in this chapter looked green for this database over the last hour, and the CPU only showed < 50%. That is because the workload only hit 100% CPU for a 13-minute window. If I select the database from the heatmap, I see a screen like Figure [7-20](#496204_2_En_7_Chapter.xhtml#Fig20).

    I like this view because I see the same information as I see in Azure Monitor metrics, but I get all in the same chart information about other resources like Workers and I/O. And even though I could see a chart like this in Azure Metrics, I wasn’t able to drill down to top queries. You can see I can select Top Queries from this screen and see results like in Figure [7-21](#496204_2_En_7_Chapter.xhtml#Fig21).

    I see that one query is marked red (which indicates it is taking up a high amount of CPU).

    I can click this query and see the details as in Figure [7-22](#496204_2_En_7_Chapter.xhtml#Fig22).

    If I scroll down from here, I get wait statistics specific to this query like in Figure [7-23](#496204_2_En_7_Chapter.xhtml#Fig23).

    Notice the majority of wait time for the query is for the CPU just like Query Store reports revealed.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig25_HTML.jpg)

Figure 7-25

SQL Copilot detects resource limits

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig24_HTML.jpg)

Figure 7-24

SQL Copilot analyzing slow performance

1.  What about Copilot? Can it help?

    You have seen some of the capabilities of “SQL Copilot.” What did it respond to in this situation with the prompt “My database is slow”? Figure [7-24](#496204_2_En_7_Chapter.xhtml#Fig24) shows the answer.

    Copilot has detected a high CPU situation with a known query, but there are no scenarios like missing indexes to explain any problem.

    I then responded (yes, you can have a conversation with Copilot!) with “Am I hitting any resource limits?” (Notice the phrase “hitting” here. Language models just get this.) Figure [7-25](#496204_2_En_7_Chapter.xhtml#Fig25) shows the response.

    You can see the response is that the problem is not a query tuning issue but that I just don’t have enough resources for the workload. Notice part of the suggestion is to use Serverless.

### Azure SQL–Specific Performance Scenarios

Based on the Running vs. Waiting scenario, there are some scenarios which are specific to Azure SQL.

#### Log Governance

Azure SQL can enforce resource limits on transaction log usage called *log rate governance*. This enforcement is often needed to ensure resource limits and to meet promised SLA. Log governance may be seen from the following **wait types**:

*   LOG_RATE_GOVERNOR – Waits for Azure SQL Database

*   POOL_LOG_RATE_GOVERNOR – Waits for elastic pools

*   INSTANCE_LOG_GOVERNOR – Waits for Azure SQL Managed Instance

*   HADR_THROTTLE_LOG_RATE* – Waits for Business Critical and geo-replication latency

Log rate governance is enforced inside the SQL Server engine before transaction log blocks are submitted for I/O. The documentation has a good description of how this works at [`https://learn.microsoft.com/azure/azure-sql/database/resource-limits-logical-server?view=azuresql#transaction-log-rate-governance`](https://learn.microsoft.com/azure/azure-sql/database/resource-limits-logical-server?view=azuresql#transaction-log-rate-governance). Scaling your deployment to a different service tier or vCore choice can give you more log rate for your application.

#### Worker Limits

SQL Server uses a worker pool of threads but has limits on the maximum number of workers. Applications with a large number of concurrent users may need a certain number of workers. Keep these points in mind on how worker limits are enforced for Azure SQL Database and Managed Instance:

*   Azure SQL Database has limits based on service tier and size. If you exceed this limit, a new query will receive an error like

    ```
    Msg 10928
    The request limit for the database is  and has been reached.
    ```

*   Azure SQL Managed Instance worker pool limit is based on the number of vCores ((105 * vCores) + 800). Since this is a full SQL Server instance, this behaves like the “max worker thread” setting for SQL Server (you can see the calculated value in the DMV **sys.dm_os_sys_info**). If you run out of worker threads, you might see THREADPOOL waits. At the time of the writing of this book, we are discussing about not letting users change this configuration value via sp_configure, so I don’t recommend you try to change this. We have not seen a customer at this time where this default value doesn’t meet the needs of an application workload.

#### Business Critical (BC) HADR Waits

Let’s say you deploy a Business Critical service tier for Azure SQL Managed Instance or Azure SQL Database. Now you start running transactions that modify data and therefore require logged changes.

You look at a DMV like sys.dm_exec_requests and see wait types like HADR_SYNC_COMMIT. What? This wait type is only seen when you deploy a sync replica for an Always On Availability Group (AG).

Turns out Business Critical service tiers use an AG behind the scenes. Therefore, it is not surprising to see these wait types normally, but it may surprise you if you are monitoring wait types.

You can also see HADR_DATABASE_FLOW_CONTROL and HADR_THROTTLE_LOG_RATE_SEND_RECV waits as part of Log Governance to ensure we can meet your promised SLA.

#### Hyperscale Scenarios

I’ve talked about the Hyperscale architecture briefly in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book. I’ll go even deeper in Chapter [8](#496204_2_En_8_Chapter.xhtml). While Hyperscale has log rate limits just like other deployment options, there are cases where we must govern transaction log generation due to a page server or replica getting significantly behind (which would then affect our ability to deliver our SLA). When this occurs, you may see wait types that start with the word **RBIO_**.

Even though we don’t dive into the details of how to diagnose various aspects of the Hyperscale architecture in this book, there are interesting capabilities for you to take advantage of. For example, reads from page servers are now available in DMVs like **sys.dm_exec_query_stats**, **sys.dm_io_virtual_file_stats**, and **sys.query_store_runtime_stats**. In addition, the I/O statistics in sys.dm_io_virtual_file_stats apply to RBEX cache and page servers since these are the I/O files that mostly affect Hyperscale performance.

Get all the details for Hyperscale performance diagnostics at [`https://learn.microsoft.com/azure/azure-sql/database/hyperscale-performance-diagnostics`](https://learn.microsoft.com/azure/azure-sql/database/hyperscale-performance-diagnostics).

## Accelerating and Tuning Performance

You have seen performance capabilities for Azure SQL, including monitoring tools. You have also seen an example of how to apply your knowledge of monitoring and a performance scenario to recognize a possible performance bottleneck. Let’s apply that knowledge to learn how to accelerate and tune performance in the areas of scaling CPU capacity, I/O performance, memory, application latency, and SQL Server performance tuning best practices.

### Scaling CPU Capacity

Let’s say you ran into the performance problem with high CPU as I showed you in the previous exercise *in your data center*. What would you do? If you were running SQL Server on a bare-metal server, you would have to potentially acquire more CPUs or even move to another server. For a virtual machine, you may be able to reconfigure the VM to get more vCPUs, but what if the host server didn’t support that? You are possibly facing a scenario to migrate your database to another VM on another host. Ouch.

For Azure SQL, you have the ability to *scale* your CPU resources with very simple operations from the Azure portal, az CLI, PowerShell, and even T-SQL. And you can do all of this with no database migration required.

For Azure SQL Database, there will be some small downtime to scale up your CPU resources. It is possible with larger database sizes, this downtime could be longer especially if we need to move your deployment to another host with enough resources for your request. We also have to ensure your replicas have the same new resources for Business Critical service tiers. Hyperscale provides a more constant scaling motion regardless of database size.

Azure SQL Managed Instance can be a concern for duration of scaling. We may need to build a new virtual cluster, so scaling operations can be significantly longer. This is something to keep in mind and is why deploying with the right resources for Managed Instance can be important. Managed Instance pools are much faster but still significantly longer than Azure SQL Database in most cases.

Azure SQL Database Serverless compute tier provides the concept of autoscaling as I described in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book.

Let’s go back where we left off in our exercise where we determined it is likely we didn’t have enough CPU resources for our workload (remember Copilot helped us determine that is the problem). Let’s scale it up and see if workload performance improves.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig27_HTML.jpg)

Figure 7-27

Scaling vCores for an Azure SQL Database

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig26_HTML.jpg)

Figure 7-26

Changing the pricing tier for an Azure SQL Database

1.  Look at scaling options in the Azure portal.

    I’ll navigate to my database in the Azure portal and select Pricing tier as seen in Figure [7-26](#496204_2_En_7_Chapter.xhtml#Fig26).

    You are now presented with a screen to make changes to your deployment. I showed you a screen like this in Chapter [4](#496204_2_En_4_Chapter.xhtml) as I described all the options after you deploy. My options look like Figure [7-27](#496204_2_En_7_Chapter.xhtml#Fig27) where I can use a slider bar to increase the number of vCores for my deployment.

Tip

There are other methods to scale the database besides the Azure Portal, including PowerShell, az CLI, and even T-SQL using ALTER DATABASE. To use these options though you need to learn more about the concept called a Service Level Objective (SLO) or ServiceObjective for short. I’ve mentioned this concept earlier in the book. This is because to change vCores you need to choose the combination of service tier and vCore. You can use the T-SQL ALTER DATABASE documentation to see all possible values for the service objective at [`https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql?view=azuresqldb-current&preserve-view=true&tabs=sqlpool`](https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql?view=azuresqldb-current&preserve-view=true&tabs=sqlpool).

In my scenario, I would need to use the HS_Gen5_16 option for the SERVICE_OBJECT argument for ALTER DATABASE.

In this case, the scaling should only take a few minutes, and the portal will show a message “Scaling is completed.”

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig28_HTML.jpg)

Figure 7-28

Azure Metrics after scaling database

1.  Let’s run the workload again.

    Let’s run the workload again to see if there is any performance differences. I’ll use the same scripts, queries, and SSMS reports as I did in the previous example in the chapter.

    Run the script **sqlworkload.cmd** again from the command prompt.

2.  Observe resource usage with **sys.dm_db_resource_stats**.

    Just as you did before, running this query several times should show a lower overall CPU usage for the database. This is a pure CPU-driven workload, so you could still see numbers in the 90% but for a far less sustained period of time. The key here is that with 16 vCores you should be able to run other queries since more CPU will be available.

3.  Observe active queries with **sys.dm_exec_requests**.

    You should see more RUNNING requests and less SOS_SCHEDULER_YIELD waits.

4.  Observe the overall workload duration.

    The overall duration dropped now from 12–13 minutes to less than 2 minutes.

5.  Let’s take a brief look at Azure Metrics to see what they look like now.

    Instead of just looking at the Monitoring graph on the Overview screen, I clicked it, which allows me to dive deeper into metrics and even change the time interval. My metrics now looks like Figure [7-28](#496204_2_En_7_Chapter.xhtml#Fig28).

    You can see here that even after moving to 16 vCores the workload uses a lot of CPU. However, it is not at 100% completely, so there is room for other workloads to use CPU. In the example with 2 vCores, our workload was technically using more than 100%, which meant it had to wait for CPU resources.

What would happen if we were to use the **Serverless** compute tier option for our workload? Remember Serverless offers the ability to autoscale workloads and also pause idle compute.

I deployed a new Serverless database with a min vCore = 2 and max vCore = 16\. Turns out in most cases (not guaranteed), a Serverless database is deployed with the number of SQL Schedulers = max vCores. So provided the Serverless database is not paused, running the same workload as in this example gives you approximately the same performance as the scaled Hyperscale 16 vCore deployment. Here is the big advantage of Serverless over the Provisioned deployment. Let’s say over a period of two hours, this workload only consumes compute for 15 minutes of the 120 minutes. For the Provisioned, you will pay for compute for the entire 120 minutes. For a Serverless deployment, you would pay for the 15 minutes of compute usage for 8 vCores, and for the remaining 90 minutes, you would pay for the equivalent compute usage for the min vCores. In addition, if you have AutoPause enabled, you will not pay for any compute costs for the last 60 minutes of that two-hour period (this is because the smallest time before a Serverless deployment is paused if idle is one hour).

Figure [7-29](#496204_2_En_7_Chapter.xhtml#Fig29) shows an example of CPU utilization for a Serverless deployment and below it a graph of actual compute billed. Notice the highest average CPU billed is during high compute utilization. The rest of the billing is for the min vCore (2 vCore) when the database is idle.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig29_HTML.jpg)

Figure 7-29

Azure Metrics for Serverless for CPU and billing

Note

General Purpose service tier supports auto-pause when the database is idle after a period. When the database is paused, you are not billed for any compute. At the time of the writing of this book, Hyperscale does not support auto-pause.

**The Cloud SQL Workshop** also has labs to show you how to scale with serverless, which you can try at [`https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/07_Deploy_Manage_Optimize_AzureSQLDB`](https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/07_Deploy_Manage_Optimize_AzureSQLDB).

### Tuning I/O Performance

I/O performance can be critical to SQL Server applications and queries. Azure SQL abstracts you from physical file placement, but there are methods to ensure you get the I/O performance you need.

**Input/Output per Second (IOPS)** may be important to your application. Be sure you have chosen the right service tier and vCores for your IOPS needs. Understand how to measure IOPS for your queries on-premises if you are migrating to Azure (hint: look at Disk Transfers/sec in Performance Monitor). If you have restrictions on IOPS, you may see long I/O waits. Scale up vCores or move to Business Critical or Hyperscale if you don't have enough IOPS. Remember for Azure SQL Managed Instance, you have seen already in this book that the new NextGen General Purpose service tier allows you to control IOPS separately from your vCore selection.

**I/O latency** is another key component for I/O performance. For faster I/O latency for Azure SQL Database, consider Business Critical or Hyperscale. For faster I/O latency for Managed Instance, move to Business Critical or use the new NextGen General Purpose service tier that I’ve mentioned in this chapter and previous chapters in this book: [`https://learn.microsoft.com/azure/azure-sql/managed-instance/service-tiers-next-gen-general-purpose-use`](https://learn.microsoft.com/azure/azure-sql/managed-instance/service-tiers-next-gen-general-purpose-use).

Configuration isn’t your only choice. Improving transaction log latency may require you to use multi-statement transactions. Learn more at [`https://learn.microsoft.com/azure/azure-sql/performance-improve-use-batching`](https://learn.microsoft.com/azure/azure-sql/performance-improve-use-batching).

In the first edition of this book, I had a lab showing you I/O latency with the General Purpose service tier for Azure SQL Database. Because of the price reduction of Hyperscale, you can see I use that service tier in this book. When I tried my labs from the first edition using Hyperscale, there was no I/O latency or issue as I saw with General Purpose, so I simply took out these examples. The **Cloud SQL Workshop** does have an example of I/O latency with Azure SQL Managed Instance you can try at [`https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/06_Manage_and_Optimize_AzureSQLMI`](https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/06_Manage_and_Optimize_AzureSQLMI).

### Increasing Memory or Workers

Memory is also an important resource for SQL Server performance, and Azure SQL is no different. The total memory available to you for buffer pool, plan cache, columnstore, and In-Memory OLTP is all dependent on your deployment choice. As I described earlier in this chapter, your highest memory capacity comes from an Azure SQL Database Hyperscale service tier using the new memory-optimized premium series at around 830GB. For a Managed Instance, you can get around 870GB of memory using the memory-optimized premium-series. Also, keep in mind that In-Memory OLTP, which is only available for Business Critical service tiers, has a maximum memory as a subset of the overall maximum memory.

One key statement about memory that holds true for SQL Server or Azure SQL: If you think you don’t have enough memory, be sure you have an optimal database and query design. You may think you are running out of buffer pool after you scan a massive table. Maybe indexes should be deployed to enhance performance of your query and use less memory. Columnstore indexes are compressed, so they use far less memory than traditional indexes.

Note

The Hyperscale vCore choice not only affects the amount of memory available to the compute nodes but also the size of the RBEX cache, which can also affect performance.

I’ve described worker limits in this chapter already, which is set to a maximum value for Azure SQL Database and Managed Instance. As with SQL Server, running out of workers may be an application problem. A heavy blocking problem for all users may result in an error or THREADPOOL wait when the real solution is to fix the blocking problem.

### Improving Application Latency

Even if you configure your deployment for all your resource needs, applications may introduce latency performance issues. Be sure to follow these best practices with Azure SQL applications:

*   Use a redirect connection type instead of proxy.

*   Optimize "chatty" applications by using stored procedures or limiting the number of query round trips through techniques like batches.

*   Optimize transactions by grouping them vs. singleton transactions.

Take a look at this documentation page for tuning applications for Azure SQL Database: [`https://learn.microsoft.com/azure/azure-sql/database/performance-guidance`](https://learn.microsoft.com/azure/azure-sql/database/performance-guidance).

### Tune like It Is SQL Server

Azure SQL is still SQL Server. So in addition to everything you have seen so far in this section and in the next section, there is almost never a substitute for ensuring you tune your SQL Server queries and look at the following:

*   Proper index design

*   Using batches

*   Using stored procedures

*   Proper query design (e.g., Do you really need to scan one million rows?)

*   Parameterize queries to avoid too many cached ad hoc queries

*   Process results in your application quickly and correctly (avoid the dreaded ASYNC_NETWORK_IO waits)

## Intelligent Performance

I mentioned earlier in this chapter our intention to build into the database engine intelligent capabilities based on data and your application workload to get you faster with **no code changes**.

Let’s take a look in more detail at the areas of Intelligent Query Processing (IQP), Automatic Plan Correction, and Automatic Tuning.

### Intelligent Query Processing

In SQL Server 2017, we enhanced the query processor to adapt to query workloads and improve performance when you used the latest database compatibility level. We called this Adaptive Query Processing (AQP). We went a step further in SQL Server 2019 and rebranded it as **Intelligent Query Processing (IQP)**. IQP is a suite of new capabilities built into the query processor and enabled using the latest database compatibility level. In SQL Server 2022, we expanded these capabilities even further, in some cases enhancing previous IQP capabilities. I called the next wave of features for SQL Server 2022 IQP “NextGen.” One of the great aspects of these features is that they are all cumulative. Figure [7-30](#496204_2_En_7_Chapter.xhtml#Fig30) shows the latest set of IQP capabilities along with a list of the previous features.

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig30_HTML.jpg)

Figure 7-30

Azure SQL IQP

You can see that two of the features Approximate Percentile and Optimized Plan Forcing are available to any application in SQL Server 2022 or Azure SQL independent of the dbcompat level. If you are using dbcompat 140 or higher, we have enhancements available for Memory Grant feedback. And then dbcompat 160 lights up new features to help you with common problems like parameter sniffing, cardinality estimation, and “out of control” DOP queries.

In addition, like in previous releases, all optimizer hotfixes are rolled up if you enable dbcompat 160 (which is now the default for Azure SQL Database). And then at the bottom middle of the figure is a list of features we enabled in SQL Server 2017 and 2019 based on your dbcompat level.

There is no feature “checkbox” to enable these capabilities. They all come built-in to the query processor, but some require a specific dbcompat level or ALTER DATABASE statement (e.g., DOP feedback requires both). You will also see in the figure an icon for the Query Store. This indicates that the Query Store is required to also enable the feature. This is because in some cases we need the Query Store to persist feedback like Memory Grant, CE, or DOP feedback.

I covered this topic extensively in the books *SQL Server 2019 Revealed* and *SQL Server 2022 Revealed*. You can try out several of these features yourself with examples from these books. These include the following:

*   You can try out IQP features introduced in SQL Server 2019, such as Memory Grant Feedback for Row Store, Table Variable Deferred Compilation, Batch Mode for Row Store, Scalar UDF Inlining, and Approximate Count Distinct at [`https://github.com/microsoft/bobsql/tree/master/sql2019book/ch2_intelligent_performance/iqp`](https://github.com/microsoft/bobsql/tree/master/sql2019book/ch2_intelligent_performance/iqp).

*   You can try out IQP features introduced in SQL Server 2022, such as Optimized Plan Forcing, Persisted Memory Grant Feedback, and Percentiles at [`https://github.com/microsoft/bobsql/tree/master/sql2022book/ch04_builtinqueryintelligence`](https://github.com/microsoft/bobsql/tree/master/sql2022book/ch04_builtinqueryintelligence).

*   You can try out other IQP features introduced in SQL Server 2022, such as Parameter Sensitive Plan Optimization, CE Feedback, and DOP Feedback at [`https://github.com/microsoft/bobsql/tree/master/sql2022book/ch04_builtinqueryintelligence.`](https://github.com/microsoft/bobsql/tree/master/sql2022book/ch04_builtinqueryintelligence)

In addition, the documentation covers this topic extensively at [`https://aka.ms/iqp`](https://aka.ms/iqp). Here is what Joe Sack said, who is not only the technical reviewer of this book but also the Group Lead for IQP, back in the first edition about the significance of IQP for Azure SQL. He said back then

> *“Over the last four years, the query processing team delivered two waves of Intelligent QP features – all with the objective to improve workload performance automatically with minimal changes to application code. Today we’re already seeing millions of databases and billions of queries using IQP features. Just as one example, we already have millions of unique query execution plans being executed hundreds of millions of times per day that use the memory grant feedback feature. In Azure SQL on a daily basis, this ends up preventing terabytes of query spills and petabytes worth of overestimations for user queries. The end result is improved query execution performance and workload concurrency”.*

This area of improving our query processor to help your application is significant for Azure SQL. As Joe tells it for the future “We have a long-term plan and active engineering investments to keep alleviating the hardest query processing problems that customers face at-scale. We look at a myriad of signals in order to prioritize features – including telemetry, customer support case volume, customer engagements and SQL community member feedback….” This statement back then proved to be true as you can see from the new capabilities we introduced in SQL Server 2022.

### Automatic Plan Correction

In 2017, I stood on stage with Conor Cunningham at the PASS Summit and showed off an amazing piece of technology for SQL Server 2017 to solve a performance problem using automation with Query Store. Query Store has such rich data; why not use it with automation?

What I showed on stage was a demonstration of a **query plan regression** problem that can be automatically fixed.

Note

You can see the code I used for this demonstration at [`https://github.com/microsoft/bobsql/tree/master/demos/sqlserver/autotune`](https://github.com/microsoft/bobsql/tree/master/demos/sqlserver/autotune).

A query plan regression occurs when the same query is recompiled and a new plan results in worse performance. A common scenario for query plan regression is *parameter-sensitive plans* (PSP), also known as parameter sniffing.

SQL Server 2017 and Azure SQL Database introduced the concept of **Automatic Plan Correction** (APC) by analyzing data in the Query Store. When the Query Store is enabled with a database in SQL Server 2017 (or later) and in Azure SQL Database, the SQL Server engine will look for query plan regressions and provide recommendations. You can see these recommendations in the DMV **sys.dm_db_tuning_recommendations**. These recommendations will include T-SQL statements to manually force a query plan when performance was "in a good state."

If you gain confidence in these recommendations, you can enable SQL Server to force plans automatically when regressions are encountered. Automatic Plan Correction can be enabled using ALTER DATABASE using the AUTOMATIC_TUNING argument.

For Azure SQL Database, you can also enable Automatic Plan Correction through *Automatic Tuning options* in the Azure Portal or REST APIs. You can read more about these techniques in the documentation. Automatic Plan Correction recommendations are always enabled for any database where Query Store is enabled (which is the default for Azure SQL Database and Managed Instance). Automatic Plan Correction (FORCE_PLAN) is enabled by default for Azure SQL Database as of March 2020 for **new** databases.

You can read more about Automatic Plan Correction at [`https://learn.microsoft.com/sql/relational-databases/automatic-tuning/automatic-tuning?view=sql-server-ver16#automatic-plan-correction`](https://learn.microsoft.com/sql/relational-databases/automatic-tuning/automatic-tuning?view=sql-server-ver16#automatic-plan-correction).

### Automatic Tuning

Technically, Automatic Plan Correction is part of a suite of services to use automation to improve query performance with no code changes called **Automatic Tuning**. Automatic Plan Correction works in SQL Server, Azure SQL Managed Instance, and Azure SQL Database.

In Chapter [1](#496204_2_En_1_Chapter.xhtml) of this book, I talked about the history of how Automatic Tuning was created. Azure SQL Database offers a unique feature of Automatic Tuning to help automate creating and dropping indexes called **automatic indexing**.

Note

Today automatic indexing is not available for Azure SQL Managed Instance.

This capability is known as **Automatic Tuning for Azure SQL Database** (also known in some parts of the documentation as SQL Database Advisor). These services run as background programs analyzing performance data from an Azure SQL Database and are included in the price of any database subscription. Automatic Tuning will analyze data from telemetry of a database, including the Query Store and Dynamic Management Views, to *recommend* indexes to be created that can improve application performance. Additionally, you can enable Automatic Tuning services to *automatically create* indexes that it believes will improve query performance. Automatic Tuning will also monitor index changes and recommend or automatically drop indexes that do not improve query performance. Automatic Tuning for Azure SQL Database takes a conservative approach to recommend indexes. This means that recommendations that may show up in a DMV like **sys.dm_db_missing_index_details** or a query showplan may not show up immediately as recommendations for Automatic Tuning. Automatic Tuning services monitor queries over time and use machine learning algorithms to make recommendations to truly affect query performance.

One downside to Automatic Tuning for index recommendations is that it does not account for any overhead performance an index could cause insert, update, or delete operations.

Note

You can read an excellent paper for how automatic indexing is built by our engineering team at [`https://www.microsoft.com/research/uploads/prod/2019/02/autoindexing_azuredb.pdf`](https://www.microsoft.com/research/uploads/prod/2019/02/autoindexing_azuredb.pdf).

One additional scenario in preview for Automatic Tuning for Azure SQL Database is parameterized queries. Queries with non-parameterized values can lead to performance overhead because the execution plan is recompiled each time the non-parameterized values are different. In many cases, the same queries with different parameter values generate the same execution plans. These plans, however, are still separately added to the plan cache. The process of recompiling execution plans uses database resources, increases the query duration time, and overflows the plan cache. These events, in turn, cause plans to be evicted from the cache. This SQL Server behavior can be altered by setting the forced parameterization option on the database (this is done by executing the ALTER DATABASE T-SQL statement using the PARAMETERIZATION FORCED option). Automatic Tuning can analyze a query performance workload against a database over time and recommend forced parameterization for the database. If over time, performance degradation has been observed, the option will be disabled.

Let’s see an example of automatic indexing in action. I’ll use a database I deployed based on the AdventureWorks example to show this capability in this chapter called bwadw (now a Hyperscale 16 vCore database, but you don’t have to have 16 vCores for this example). You can try this out yourself using the scripts found in the **ch7_performance\tuning_recommendations**. You will need the ostress utility again for this demo.

Run the following steps connected to the database:

*   Run the **order_rating_ddl.sql** script to create the table.

*   Edit the **order_rating_insert.cmd** file to put in your logical server, database, admin login, and password. Execute this script. This script uses the **order_rating_insert.sql** script to insert rows into the table.

*   Now edit the **query_order_rating.cmd** script to put in your logical server, database, admin login, and password. Execute this script. **Caution:** This script is designed to run for at least 12 hours. It uses the **query_order_rating.sql** script to execute queries repeatedly against a table with no index.

Here is the main issue when using these scripts. It takes time and patience. Why? Our algorithms don’t just recommend indexes based on a single query and single execution. We look at query workloads over time and for frequent executions to decide if an index makes sense. Therefore, when you try this yourself, you will need to let this script run to completion (it runs thousands of iterations). When I did this within 12 hours, I saw the information I’m about to show you from the Azure portal.

After running the workload and waiting for 12 hours, I can see a chart on my performance and recommendations highlighted from **Performance overview** in the Azure portal similar to Figure [7-31](#496204_2_En_7_Chapter.xhtml#Fig31).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig31_HTML.jpg)

Figure 7-31

Performance overview for Azure SQL Database

Let’s now get a closer look at the queries behind the high CPU utilization by selecting **Query Performance Insights** as you can see in Figure [7-32](#496204_2_En_7_Chapter.xhtml#Fig32).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig32_HTML.jpg)

Figure 7-32

Query Performance Insights for Azure SQL Database

The first query is consuming all the CPU, so I can select it and I will get a screen like Figure [7-33](#496204_2_En_7_Chapter.xhtml#Fig33).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig33_HTML.jpg)

Figure 7-33

Query details from performance insights

This provides me historical performance and the text of the query. I could click Recommendations at the top of this screen, but instead let’s go back to the Azure Portal. Here, I can select **Performance Recommendations** like in Figure [7-34](#496204_2_En_7_Chapter.xhtml#Fig34).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig34_HTML.jpg)

Figure 7-34

Performance Recommendations for Azure SQL Database

On this screen, you can see one of the recommendations is to create a new index and the Impact is High (which means it could have a high positive impact on improving performance).

I can click the Create Index action and get more details about the recommended index as you see in Figure [7-35](#496204_2_En_7_Chapter.xhtml#Fig35).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig35_HTML.jpg)

Figure 7-35

Create index recommendation

On this screen, I can get details of how the index should be built, its impact on performance, and expected storage space required to build it. I can at this point select Apply at the top of the screen to create the index. I can also select View script to get the details of the T-SQL command as you see in Figure [7-36](#496204_2_En_7_Chapter.xhtml#Fig36).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig36_HTML.jpg)

Figure 7-36

Create index script

You can see an online index is the default method used for automatic indexing.

Let’s say that you like these recommendations and even want to consider having Azure find and create these *automatically*. You can select **Automatic Tuning** from the Azure Portal to change these options as you see in Figure [7-37](#496204_2_En_7_Chapter.xhtml#Fig37).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig37_HTML.jpg)

Figure 7-37

Automatic Tuning options for Azure SQL Database

You have options to force plan (automatic plan correction), create, and drop indexes. You can set these options at the subscription, logical server, or database level. The DROP index is a nice feature here because it could be that data changes in such a way that an index is no longer needed and could be hurting performance.

You also have another way to see recommendations for indexes on the Overview screen for the database (or even across databases on the logical server overview) just like Figure [7-38](#496204_2_En_7_Chapter.xhtml#Fig38).

![](images/496204_2_En_7_Chapter/496204_2_En_7_Fig38_HTML.jpg)

Figure 7-38

Database notifications

You can also view Automatic Tuning options through the catalog view **sys.database_automatic_tuning_options**. You can view all the columns for this catalog view at [`https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-database-automatic-tuning-options-transact-sql`](https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-database-automatic-tuning-options-transact-sql).

## Summary

To deliver the best performance for your application, you need the capabilities and monitor tools that are tried and proven from SQL Server. Azure SQL gives you that and more, including capabilities and tools specific to Azure.

Azure SQL gives you the controls and options to accelerate and tune performance, including the ability to scale easily with no database migration required or deploy a Serverless Hyperscale database for autoscale compute and storage.

Finally, Azure SQL comes with intelligent performance capabilities built into the query processor and services that leverage the power of Query Store from your database.

In the next chapter, we will explore and dive deep into the final core engine capability of Azure SQL: availability.

# 8. Availability for Azure SQL

You have now made the journey from deploy to configure to security and then to scale, monitor, and tune performance. The final piece of the *core* of Azure SQL is **availability**. I have rarely talked to a SQL Server customer over the years that didn’t need their database to be highly available. I also almost never talk to a customer who doesn’t care about being able to recover from a disaster. Therefore, this chapter is really about **High Availability and Disaster Recovery (HADR)** for Azure SQL. I will tell you that in my opinion after now assessing and using Azure SQL over this last year, the built-in capabilities for HADR for Azure SQL are one of the great stories of the service. In fact, I believe after you go through this chapter you will be convinced that looking at Azure SQL as a target for your deployment is worth it just on the HADR capabilities alone.

In this chapter, we will spend the majority of time diving into the details of HADR capabilities, including backup and restore, built-in availability, extending HADR with Azure, and database availability and consistency. Then I’ll finish the chapter on how to monitor HADR for your deployment.

This chapter will contain examples for you to try out and use as you read along. For you to try out any of the techniques, commands, or examples I use in this chapter, you will need

*   An Azure subscription.

*   A minimum of Contributor role access to the Azure subscription. You can read more about Azure built-in roles at [`https://learn.microsoft.com/azure/role-based-access-control/built-in-roles`](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles).

*   Access to the Azure Portal.

*   I will use several Azure SQL Databases and a Managed Instance I deployed earlier in this book. I also deployed a new Azure SQL Business Critical service tier database that uses the AdventureWorksLT sample for one of the examples.

*   To connect to Managed Instance, you will need a *jumpbox* or virtual machine in Azure to connect. I showed you how to do this in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book. One simple way to do this is to create a new Azure Virtual Machine and deploy it to the same virtual network as the Managed Instance (you will use a different subnet than the Managed Instance).

*   To connect to Azure SQL Database, I’m going to use the Azure VM I deployed in Chapter [3](#496204_2_En_3_Chapter.xhtml), called **bwsql2022**, and configured for a private endpoint in Chapter [6](#496204_2_En_6_Chapter.xhtml) (you could use another method as long as you can connect to the Azure SQL Database).

*   Installation of the **az** CLI (see [`https://learn.microsoft.com/cli/azure/install-azure-cli`](https://learn.microsoft.com/cli/azure/install-azure-cli) or more details). You can also use the Azure Cloud Shell instead since az is already installed. You can read more about the Azure Cloud Shell at [`https://azure.microsoft.com/features/cloud-shell/`](https://azure.microsoft.com/features/cloud-shell/).

*   Installation of Azure PowerShell. Use the following documentation on how to install Azure PowerShell for your client: [`https://learn.microsoft.com/powershell/azure/install-azure-powershell`](https://learn.microsoft.com/powershell/azure/install-azure-powershell). I installed Azure PowerShell in my Azure VM and any client where I might do examples from this chapter.

*   You will run some T-SQL in this chapter, so install a tool like SQL Server Management Studio (SSMS) at [`https://aka.ms/ssms`](https://aka.ms/ssms). I installed both SSMS in the bwsql2022 Azure Virtual Machine.

*   For this chapter, I have script files you can use for some of the examples. You can find these scripts in the **ch8_availability** folder for the source files included for the book. I will also use the very popular tool ostress.exe for exercises in this chapter which comes with the RML Utilities. You can download RML from [`https://aka.ms/ostress.exe`](https://aka.ms/ostress.exe). Make sure to put the folder where RML gets installed in your system path (which is by default C:\Program Files\Microsoft Corporation\RMLUtils).

*   I also use Database Watcher and Microsoft Copilot skills in Azure SQL Database in this chapter, but this is optional for you to use.

## HADR Capabilities

I want to first review with you the amazing HADR capabilities that come with Azure SQL before we dive deeper with examples into each topic.

### Automatic Backups and Point-in-Time Restore

Azure SQL is SQL Server, so the full complement of BACKUP and RESTORE functionality is possible. However, the promise of PaaS is to provide **managed capabilities**. Therefore, Azure SQL provides an automated backup system for both Managed Instance and Databases to meet your Recovery Point Object (RPO) and historical data needs. In fact, for Azure SQL Database, you are completely abstracted from the BACKUP T-SQL statement. Managed Instance will allow a COPY_ONLY backup to Azure Storage.

All backups from Azure SQL are kept on separate storage from your database and log files with automated geo-redundant mirrors. Azure SQL also offers a long-term backup retention option.

Azure SQL will use full, differential, and log backups supporting a complete point-in-time restore interface. In addition, you have these restore capabilities:

*   Restore deleted databases.

*   Managed Instance supports the RESTORE T-SQL statement from Azure Blob Storage which could be from an on-premises SQL Server backup or a COPY_ONLY backup from a Managed Instance.

*   Remember as mentioned earlier in the book, a versioned Managed Instance allows you to restore a backup to SQL Server 2022.

### Built-In Availability

You may be used to using an Always On Failover Cluster Instance (FCI) or Always On Availability Group (AG) with SQL Server to give you high availability and achieve a desired Recovery Time Object (RTO).

As part of **every** Azure SQL deployment, you get a complete built-in availability system, just by deploying Azure SQL. This is included in your Azure subscription and fees for your deployment.

As you will see in this chapter, a General Purpose deployment will behave similar to FCI, and Business Critical will be similar to AGs. Hyperscale uses a unique architecture that will feel like a combination of both. In all cases, the power of the Azure Service Fabric is used for automatic failover capabilities.

### Azure Redundancy for High Availability

Why rely on a single data center when you have three? Azure SQL can integrate with a capability called **zone redundancy**. Each zone is a set of one or more data centers (so actually more than three) within an Azure region that has independent power, cooling, and networking. Azure SQL can deploy a ***high availability*** solution across zones to provide you even more availability should there be a failure in a particular data center.

All Azure SQL deployments are created as part of an **Azure Availability Set** which includes using different fault and update domains. **Fault domains** define the group of virtual machines that share a common power source and network switch. **Update domains** are groups of virtual machines and underlying physical hardware that can be rebooted at the same time. Only one update domain is rebooted at a time. A rebooted update domain is given 30 minutes to recover before maintenance is initiated on a different update domain. You can read more about this concept at [`https://learn.microsoft.com/azure/virtual-machines/availability#configure-multiple-virtual-machines-in-an-availability-set-for-redundancy`](https://learn.microsoft.com/azure/virtual-machines/availability#configure-multiple-virtual-machines-in-an-availability-set-for-redundancy).

### Geo-replication and Failover Groups

You might want to provide even higher levels of availability by synchronizing your deployments across Azure regions. Azure SQL provides two methods for this capability called geo-replication and failover groups. We will go into each option in this chapter and why you may want to choose one vs. the other.

### Database Availability and Consistency

With SQL Server, you are used to using various techniques to make your database available and checking consistency. Azure SQL eliminates the need for heavy “emergency” recovery options and provides many built-in consistency checks. I’ll explore more in this chapter specific comparisons for database availability, recovery, and consistency for Azure SQL as compared to SQL Server.

### Disaster Recovery with Managed Instance Link

Managed Instance link provides the ability to perform an online migration from SQL Server 2016, 2019, or 2022 to Managed Instance using the power of Availability Groups. Since a versioned Managed Instance is compatible with SQL Server 2022, you can also build a disaster recovery architecture between SQL Server 2022 and Managed Instance. This is effectively building a Distributed Availability Group (DAG) between SQL Server and Managed Instance.

### Free Passive DR Replicas

If you configure a *replica* for Azure SQL Managed Instance or Database using technologies like Managed Instance link, geo-replication, or failover groups, you have the ability to declare the secondary replica as *passive*. This means you will only use the secondary for disaster recovery purposes. If you choose this option, you can save costs because you only pay for compute and storage of the secondary. This is also known as a *standby replica*.

### SQL Server Replication

SQL Server Replication has been a popular method to provide a level of availability and database synchronization over many SQL Server releases. I won’t dive into the details of using SQL Server Replication in this chapter, but point out these capabilities:

*   Azure SQL Managed Instance gives you the full capabilities to set up a transaction or snapshot replication system, including publisher, distribution, and subscribers. A subscriber can be another Managed Instance database, an Azure SQL Database, or even a SQL Server in Azure VM or on-premises. Read more about Azure SQL Managed Instance and Replication at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/replication-transactional-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/replication-transactional-overview).

*   An Azure SQL Database can be a subscriber from an on-premises SQL Server, SQL Server in Azure VM, or Managed Instance for transaction and snapshot replication. This might be an interesting migration option when moving to Azure SQL Database because it can provide a type of “online” migration strategy. Read more at [`https://learn.microsoft.com/azure/azure-sql/database/replication-to-sql-database`](https://learn.microsoft.com/azure/azure-sql/database/replication-to-sql-database).

## Backup and Restore

Imagine you needed to set up an automated backup system for your SQL Server deployment. You basically want a system to abstract even other DBAs from worrying about performing backups. You want these backups to run regularly; use a combination of full, differential, and log backups; and be placed on storage separate from your database for full protection. And you would also like the storage for your backups to be mirrored even across data centers in your company.

Guess what? When you deploy an Azure SQL Managed Instance or Azure SQL Database, we just do all of this by default…and more. Let’s look at various aspects to the automated backup system and how to use restore with these backups.

### Automatic Backups

When you deploy an Azure SQL Database or create a new database for an Azure SQL Managed Instance deployment, we monitor this activity and kick in the following schedule of backup activity:

*   A full database backup once a week.

*   A differential backup every 12 or 24 hours.

*   A transaction log backup every 5–10 minutes. The actual frequency of log backups is based on the number of vCores and database activity.

Note

We may vary the implementation of how we do this. The concept is that we give you point-in-time restore and meet your RPO objective.

All backups are done using standard T-SQL statements in the background and are stored separately from your data and log files (even if they are on Azure Storage). One of the options you have for your backups is a ***redundancy*** option. These options include the following.

#### Locally Redundant Storage (LRS)

Your backup files are copied synchronously three times within the physical data center. While this option does not provide you the best protection, it is a choice for customers due to data residency requirements.

#### Zone-Redundant Storage (ZRS)

This option is similar to LRS except the synchronous copies of the backup files are spread across three Availability Zones. This option is available for Azure SQL Managed Instance and Azure SQL Database independent of the zone redundancy of the deployment.

#### Geo-redundant Storage (GRS)

This is the default option for any Azure SQL Managed Instance or Azure Database. Your backup files are locally redundant but also copied asynchronously to a single physical location in another region (which uses LRS).

Figure [8-1](#496204_2_En_8_Chapter.xhtml#Fig1) shows an example of the redundancy of these backups.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig1_HTML.jpg)

Figure 8-1

Geo-redundant backups

#### Geo-Zone-Redundant Storage (GZRS)

This means that backup files are copied across three Azure Availability Zones in the primary region and copied asynchronously to a single physical location in a different region (which uses LRS in the secondary region). This option is only available for Azure SQL Managed Instance and (even if the instance is not zone redundant) for Azure SQL Database Hyperscale (when it is enabled for zone redundancy).

Geo-redundant options offer you the best recovery options because you can restore from a backup even if the region for your instance or database is not available. ZRS and GZRS are the only options for a Hyperscale database that is zone redundant. You cannot change the backup redundancy for Hyperscale after database creation. Read more about geo-redundancy for Azure Storage at [`https://learn.microsoft.com/azure/storage/common/storage-redundancy#redundancy-in-a-secondary-region`](https://learn.microsoft.com/azure/storage/common/storage-redundancy#redundancy-in-a-secondary-region).

Note

Geo and zone redundant backup storage options can incur more costs. Learn more at [`https://learn.microsoft.com/azure/azure-sql/database/automated-backups-overview?view=azuresql&tabs=single-database#backup-storage-costs`](https://learn.microsoft.com/azure/azure-sql/database/automated-backups-overview?view=azuresql&tabs=single-database#backup-storage-costs).

When you deploy or create a new database, we schedule a full database backup almost immediately. We perform integrity checks on your backups using CHECKSUM and restore techniques. Read the complete story of automated backups for Azure SQL at [`https://learn.microsoft.com/en-us/azure/azure-sql/database/automated-backups-overview`](https://learn.microsoft.com/en-us/azure/azure-sql/database/automated-backups-overview).

#### Backup Retention

By default, we keep enough backup files to allow you to perform a point-in-time restore (PITR) within the last 7 days at any point in time. For Azure SQL Database, you can change this retention to 1 day or up to 35 days. This is called **short-term backup retention**. You have the same option for any database created for a Managed Instance.

Retention policies affect how far back you can restore from a point in time but also how much storage space your backups consume. You can configure the short-term retention policy for Azure SQL Database backups through the Azure portal, with the az CLI (**az sql db ltr-policy**), or PowerShell (**Set-AzSqlDatabaseBackupShortTermRetentionPolicy**).

Azure SQL offers a concept called **long-term backup retention (LTR)**. This allows you to keep backups for up to **10 years**. All LTR backups that are kept *are full database backups*.

Backup retention can be configured for databases for Managed Instance using the Azure portal as seen in Figure [8-2](#496204_2_En_8_Chapter.xhtml#Fig2) for one of the databases I deployed on my Managed Instance.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig2_HTML.jpg)

Figure 8-2

Configuring backup retention for a Managed Instance database

The portal only allows configuration on a database level, so for the instance you may want to use scripts for automation. Therefore, you can also manage the short-term retention policy for Managed Instance backups with the az CLI (**az sql midb short-term-retention-policy**) and PowerShell (**Set-AzSqlInstanceDatabaseBackupShortTermRetentionPolicy**).

Figure [8-3](#496204_2_En_8_Chapter.xhtml#Fig3) shows how to configure backup retention for an Azure SQL Database through the logical server in the Azure Portal.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig3_HTML.jpg)

Figure 8-3

Backup retention for Azure SQL Database

We have a chart in our documentation that can help you sketch out a schedule for LTR at [`https://learn.microsoft.com/azure/azure-sql/database/long-term-retention-overview?view=azuresql#how-long-term-retention-works`](https://learn.microsoft.com/azure/azure-sql/database/long-term-retention-overview?view=azuresql#how-long-term-retention-works).

#### Automated Backups

You can also configure LTR for Azure SQL Database using the az CLI (**az sql db ltr-policy**) or PowerShell (**Set-AzSqlDatabaseBackupLongTermRetentionPolicy**).

Note

If you use geo-replication or failover groups, which I will discuss later in this chapter, you can configure LTR for those databases, but LTR backups are not taken unless that database becomes a primary.

#### Backup Storage Consumption and Costs

As part of your deployment, you get free storage for backups equivalent to the maximum size of your database or Managed Instance storage size. This includes the space for all full, differential, and log backups. Even though we compress all backups, the size you need will depend on the size of your data, how many changes you make (affects size of differential and log backups), and your number of backup retention days.

If you exceed the backup storage you get for free with your managed instance maximum storage size or maximum Azure SQL Database size, you can incur extra costs for backups.

In most cases, we have found customers that use the default retention period of seven days rarely incur extra charges. For Azure SQL, you can track if you are using extra space that is being charged on your subscription by using the Azure portal and viewing billing information with your subscription. Learn more at [`https://learn.microsoft.com//azure/azure-sql/database/automated-backups-overview?view=azuresql&tabs=single-database#backup-storage-costs`](https://learn.microsoft.com//azure/azure-sql/database/automated-backups-overview?view=azuresql&tabs=single-database#backup-storage-costs). If you look at this documentation page, you can see that geo and zone redundant backup options can incur more costs.

For Azure SQL Database, Azure Metrics allow you to track and even get alerts on backup storage consumed for full, differential, and log backups. Figure [8-4](#496204_2_En_8_Chapter.xhtml#Fig4) shows an example of using Azure Metrics through the portal to see what is available.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig4_HTML.jpg)

Figure 8-4

Metrics for backup storage consumption for Azure SQL Database

Some tips to help you on consuming backup storage space:

*   Reduce the retention period to the shortest possible number of days per your requirements.

*   The larger modifications you make, the larger the space needed for differential and log backups. For example, reduce unnecessary index rebuilds to avoid large modifications.

*   It is possible that you can increase your maximum storage size, so you get more backup space, but the increase in storage size could cost less than backup storage costs.

### Point-in-Time Restore (PITR)

Now that you know about the automated backup strategy we use, you may have a need to use backups to perform a restore. In some cases, for SQL Server, you may run into a situation where an incident has occurred that affects availability like a database owner dropping a table.

Since we deploy a combination of full, differential, and log backups, we allow you to pick a *point in time* and restore back to that state using these backups. This concept is called **point-in-time restore (PITR)** and is available in SQL Server (provided you deploy the right backup strategy).

If you look at the SQL Server documentation at [`https://learn.microsoft.com/sql/relational-databases/backup-restore/restore-a-sql-server-database-to-a-point-in-time-full-recovery-model`](https://learn.microsoft.com/sql/relational-databases/backup-restore/restore-a-sql-server-database-to-a-point-in-time-full-recovery-model), the method to restore to a point in time is to use a series of backups to restore from a log backup using the T-SQL RESTORE statement. For Azure SQL, PITR is supported for the automated backups we create for Azure SQL Managed Instance and Databases. Therefore, to perform a point-in-time restore, you must use Azure interfaces such as the portal, az CLI, or PowerShell.

Note

Even though the RESTORE statement is supported for a Managed Instance, your syntax is limited and you cannot perform a point-in-time restore. This is only to restore full backups from the COPY_ONLY option or from a SQL Server.

Azure SQL Managed Instance supports PITR through the Azure portal (navigate to the database in the portal and select the Restore option from the Command Bar), az CLI (**az sql midb restore**), and PowerShell (**Restore-AzSqlInstanceDatabase**). A PITR operation is asynchronous and creates a *new database (in fact, a new deployment)* based on the date and time you select for the restore. Your date and time choices are based on your backup day retention.

Azure SQL Database also supports PITR through the Azure portal, az CLI (**az-sql-db-restore**), and PowerShell (**Restore-AzSqlDatabase**).

The **key concept** for PITR for Azure SQL is you never restore on top of an existing database but initiate a new database deployment by restoring from an automatic backup.

Let’s look at an example of how to use PITR for the database I deployed called **bwadw** earlier in the book. I created this database some time back as I was writing the chapters of the book, so by now at least seven days have gone by for a series of backups. Let’s go through an exercise where I accidentally drop a table in my database and then use PITR to restore the database to a new database name before the drop, so I can merge that data back into my current database.

1.  **Drop the table**

    I’ll use the Azure VM **bwsql2022** I deployed in Chapter [3](#496204_2_En_3_Chapter.xhtml) and have used in the last two chapters to connect with SSMS and run the following T-SQL statement against the **bwadw** database connected as the server admin I used during deployment (remember it is based on the AdventureWorksLT sample):

    ```
    DROP TABLE SalesLT.OrderRating;
    GO
    ```

    **Note** This is the table I created as part of an example in Chapter [7](#496204_2_En_7_Chapter.xhtml). I left around 125,000 rows in the table before I dropped it. So, if we restore it correctly, we should have it back with 125,000 rows.

2.  **Restore the database to the time before the DROP**

    I just dropped the table, so I should be able to go back to an earlier point when the table was created and data existed. I can just pick one hour earlier to restore from this point.

    **Tip** You can use SQL Audit to go back and find the DROP TABLE statement in a real-world scenario where you don’t know when the table was dropped.

You can use the Azure Portal to choose to start the Restore process as seen in Figure [8-5](#496204_2_En_8_Chapter.xhtml#Fig5).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig5_HTML.jpg)

Figure 8-5

Starting the PITR process for an Azure SQL Database

Notice the earliest restore point is listed on the Overview screen.

Note

The earliest point in time is based on the first transaction log backup after the database is created. A full backup must first complete before the log backup is taken. **Here is a tip.** Use our new **Copilot** and use this prompt **“How far back in time can I go for a point-in-time restore?”**

After I select Restore, I’m presented with a screen where I can fill out the time I want to restore to as seen in Figure [8-6](#496204_2_En_8_Chapter.xhtml#Fig6).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig6_HTML.jpg)

Figure 8-6

Creating a new database based on PITR

You will notice where I can select a date and time for the restore which is effectively an hour before I executed the DROP statement. The rest of the screen if you scroll down allows you to choose the name of the database and other options as you would if you deployed a new database (e.g., service tier). The default database name created (which you can change) is the original database name, including the details of the restore point. For me, my database name was bwadw_2024-08-25T20-28Z.

You need to set some expectations on time here to deploy. This is not just a new database. We must do all the things to deploy a new database **and** restore a full database backup and a series of differential and log backups up to the time you selected. For me, this restore took around ten minutes. I can do all the activities we talked about in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book to look at the Activity Log and see how the restore deployed a new database. I recommend customers try out various scenarios here to see expected recovery times for their workload. Just remember that higher RTO and RPO objective can be achieved with concepts like geo-replication and failover groups which will be discussed later in this chapter.

Note

Your recovery time will depend on what we need to restore to get you to the desired point in time. Learn more at [`https://learn.microsoft.com/azure/azure-sql/database/recovery-using-backups?view=azuresql&tabs=azure-portal#recovery-time`](https://learn.microsoft.com/azure/azure-sql/database/recovery-using-backups?view=azuresql&tabs=azure-portal#recovery-time), including limits on the number of concurrent restores.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig7_HTML.jpg)

Figure 8-7

Querying the dropped table from PITR

1.  **Verify the new database has the dropped table restored**

    I went back to my Azure VM, **bwsql2022**, where I had connected as an admin to the logical master database. If I navigate to this database and run the following T-SQL statement:

    ```
    SELECT COUNT(*) FROM SalesLT.OrderRating;
    GO
    ```

    I get back 125,000 rows as you can see in Figure [8-7](#496204_2_En_8_Chapter.xhtml#Fig7).

    I now have two choices:

*   Drop the current bwadw database and rename this new database, so my applications would lose any changes since the time for PITR.

*   Merge the data from this new database from SalesLT.OrderRating into my original database. I realize this may not be a simple operation if foreign keys exist.

### Geo-restore of Databases

Let’s say that your database from Azure SQL Database or Managed Instance is unavailable due to a data center outage. While these situations are rare, it would be nice to be able to restore a backup from a geo-redundant backup from another region that does not have an outage.

The process is called **geo-restore** and is outlined at [`https://learn.microsoft.com/azure/azure-sql/database/recovery-using-backups?view=azuresql&tabs=azure-portal#geo-restore`](https://learn.microsoft.com/azure/azure-sql/database/recovery-using-backups?view=azuresql&tabs=azure-portal#geo-restore). The concept is that you will deploy a new Azure SQL Database or create a new database for a Managed Instance based on a backup. When you select this option, you will be presented with all the known backups that exist for your existing Azure SQL deployments. If the data center is down where your backups are normally stored, we will retrieve the geo-redundant copy of the backup from a different region.

### Restore Backups from Deleted Databases

The built-in HADR options just keep coming with Azure SQL. Let’s say you *accidentally* deleted a database for an Azure SQL Database or Managed Instance.

Note

Deleting a database for Azure SQL Database or Managed Instance does more behind the scenes than a traditional SQL Server given we have all types of services and operations tied to the database. This is why you can delete a database through Azure interfaces or the DROP DATABASE T-SQL statement.

I realize this may not be something you almost never see, but one of the great stories for built-in HADR is that when you delete a database, you can restore from backups associated with the deleted database using PITR based on the retention period you configured. If your retention period is seven days and you delete a database, we can’t perform any more backups, but you can restore to a point in time from the earliest backup point to the time of database deletion.

Note

You cannot recover from a deleted logical server or Managed Instance. However, if you have configured LTR backup retention, you can create new databases based on these backups. You may be asking since LTR backups are not free, how do I ever remove LTR backups? You can use the PowerShell cmdlet **Remove-AzSqlDatabaseLongTermRetentionBackup** or **Remove-AzSqlInstanceDatabaseLongTermRetentionBackup** *even if the logical server or Managed Instance has been deleted*.

You can restore deleted databases through the Azure portal, az CLI, or PowerShell. Read more about how to do this at [`https://learn.microsoft.com/azure/azure-sql/database/recovery-using-backups?view=azuresql&tabs=azure-portal#deleted-database-restore`](https://learn.microsoft.com/azure/azure-sql/database/recovery-using-backups?view=azuresql&tabs=azure-portal#deleted-database-restore).

### Restore in Azure SQL Managed Instance

I’ve mentioned that it is possible to execute a BACKUP T-SQL statement for an Azure SQL Managed Instance earlier in this chapter. We have referred to this capability as a *native* database backup. The term native is used because you can perform a full database backup using the BACKUP T-SQL statement to disk storage. This disk storage must be an Azure Storage account and uses the “backup to URL” capability that SQL Server has supported for several releases.

You must use the WITH COPY_ONLY option to back up a database for a Managed Instance. SQL Server has supported a *copy-only* backup for several releases. A copy-only backup does not affect the backup sequence of full, differential, and log backups. Our team has posted a nice blog about how to set up a native backup with Managed Instance at [`https://techcommunity.microsoft.com/t5/azure-sql-database/native-database-backup-in-azure-sql-managed-instance/ba-p/386154`](https://techcommunity.microsoft.com/t5/azure-sql-database/native-database-backup-in-azure-sql-managed-instance/ba-p/386154).

Since you can perform a native backup with a Managed Instance, you can also restore these backups using the T-SQL RESTORE statement. You can only restore copy-only backups from a Managed Instance to an existing or new Managed Instance. Just remember it is a new database. For example, you will get an error (Mg 41901) if you try to use the WITH REPLACE syntax of RESTORE.

In addition, you can take any full database backup from any supported version of SQL Server and restore it to a Managed Instance. This is actually a method you use to perform a simple offline migration from SQL Server to a Managed Instance. Learn more at [`https://learn.microsoft.com/data-migration/sql-server/managed-instance/database-migration-service?toc=%2Fazure%2Fdms%2Ftoc.json&tabs=online-with-extension`](https://learn.microsoft.com/data-migration/sql-server/managed-instance/database-migration-service?toc=%252Fazure%252Fdms%252Ftoc.json&tabs=online-with-extension).

The following documentation page walks you through the process of performing a native restore with a Managed Instance: [`https://learn.microsoft.com/azure/azure-sql/managed-instance/restore-sample-database-quickstart`](https://learn.microsoft.com/azure/azure-sql/managed-instance/restore-sample-database-quickstart).

## Built-In Availability

SQL Server has a great tradition of providing the necessary capabilities and software to keep your database highly available. The tradition started with Always On Failover Cluster Instance (FCI) using shared storage and integrating with technologies such as Windows Server Failover Clustering (WSFC) for automated failover. In SQL Server 2012, we introduced Always On Availability Groups to allow a non-shared storage approach, read replicas, and still integrated with WSFC for automatic failover decisions. SQL Server Linux also supports these capabilities but is integrated with Linux technologies such as Pacemaker.

One of the aspects of Azure SQL the engineering team wanted to provide was a public commitment to a Service-Level Agreement (SLA), including availability. They also wanted the deployment process for a database or Managed Instance to “just do it” when it came to configuring and setting up availability. Finally, since Azure SQL was deployed with Azure Service Fabric (SF), we needed to integrate with SF for failover decisions.

The result is truly an amazing story. Every Azure SQL deployment option you choose has a built-in availability. Let’s take a look at each deployment option and the architecture of availability that makes it all happen. You can also use the following documentation page as a reference: [`https://learn.microsoft.com/en-us/azure/azure-sql/database/business-continuity-high-availability-disaster-recover-hadr-overview`](https://learn.microsoft.com/en-us/azure/azure-sql/database/business-continuity-high-availability-disaster-recover-hadr-overview). Even though the details of this chapter for General Purpose and Business Critical service tiers apply to both Azure SQL Managed Instance and Database, there are a few differences worth noting for Azure SQL Managed Instance as you can see at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/high-availability-sla-local-zone-redundancy`](https://learn.microsoft.com/azure/azure-sql/managed-instance/high-availability-sla-local-zone-redundancy).

### General Purpose Availability

I’ve described the overall concept of a General Purpose (GP) service tier several times in this book. Your databases are stored on Azure Storage, while tempdb is stored on local SSD storage. Let’s use a visual to describe more about the General Purpose architecture and how availability and failover work. (These figures are based on diagrams in the documentation at [`https://learn.microsoft.com/azure/azure-sql/database/high-availability-sla-local-zone-redundancy?view=azuresql&tabs=azure-powershell#basic-standard-and-general-purpose-service-tier-availability`](https://learn.microsoft.com/azure/azure-sql/database/high-availability-sla-local-zone-redundancy?view=azuresql&tabs=azure-powershell#basic-standard-and-general-purpose-service-tier-availability).)

Figure [8-8](#496204_2_En_8_Chapter.xhtml#Fig8) shows the architecture for GP availability.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig8_HTML.jpg)

Figure 8-8

General Purpose service tier availability

An application will connect to the primary replica (there is only one replica with General Purpose) by using gateways in the control ring in an Azure region. If you remember, we talked about connection types of proxy and redirect with gateways in Chapters [4](#496204_2_En_4_Chapter.xhtml) and [6](#496204_2_En_6_Chapter.xhtml) of the book. In either case, the gateways provide a connection abstraction for the application. Notice the local SSD storage for the deployment is where tempdb is stored.

The database and transaction log files are put on Azure Premium Storage using LRS (local redundant storage). This means your database and transaction log files are replicated three times within the physical location of the data center.

As I’ve described earlier in this chapter, your backup files are stored in a different storage location with redundancy options of LRS, ZRS, and GRS.

Note

Azure SQL Managed Instance also supports geo-zone-redundant backups for a GP service tier even if the deployment is not zone redundant.

The Azure SQL deployment is integrated with Service Fabric (SF) to detect problems (e.g., a node failure) and initiate a *failover* if necessary. If a failover is required, we will find a new node with spare capacity to host your deployment.

The local storage of the new node hosts SQL Server, including tempdb. The new SQL Server will be directed to your database and log files on Azure Storage. This type of architecture is very similar to how an FCI works for SQL Server. Your downtime is based on how fast we can find a new node with enough capacity to host your deployment choices (vCores, etc.). In addition, SQL Server that is hosting your database on the new node has just started up with a cold buffer and plan cache, so normal startup activities will affect your performance (recovery of the database will be extremely fast since we use Accelerated Database Recovery).

You may be wondering how does the application connect to the new node after a failover? The gateways are the answer. The application never changes any names to connect to the new node. The gateways take care of that logic. The application must simply retry a connection and is off and running.

### Business Critical Availability

A Business Critical (BC) deployment relies on local storage and a series of replicas, much like an Always On Availability Group (AG). Let’s look at the BC architecture as compared to General Purpose. Figure [8-9](#496204_2_En_8_Chapter.xhtml#Fig9) shows the architecture of a BC deployment with availability.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig9_HTML.jpg)

Figure 8-9

Business Critical service tier availability

You can see from this figure that gateways are still an important part of connectivity and that a primary replica exists. Local storage is used for tempdb but also for database and log files. In addition, like an Always On Availability Group, there are secondary replicas. For a BC deployment, we always keep four replicas up and running (one primary and three secondaries). From a transaction point of view, a commit cannot proceed on the primary replica until *at least one* of the secondary replicas has acknowledged the changes are hardened.

You can also see that backup files are stored in Azure Storage with redundancy options as I’ve described earlier in this book.

Note

Azure SQL Managed Instance also supports geo-zone-redundant backups for a BC service tier even if the deployment is not zone redundant.

If a failover is necessary, we simply need to choose a secondary replica that is synchronized and make the primary replica just like an AG. Downtime is significantly less than General Purpose since the new primary replica simply has to run undo recovery to become available. Since Accelerated Database Recovery (ADR) is enabled, undo recovery can be very fast. I’ll discuss more about the importance of ADR later in this chapter. If the old primary replica is not usable, we will need to spin up a new secondary replica (and synchronize it) to keep four available.

Just like General Purpose service tier, applications just need to reconnect and start running again due to the use of gateways. In addition, a BC deployment will allow you to use *one* of the secondary replicas as a *read-only replica* as part of your monthly free for using BC. Our gateways help provide the redirection logic. You simply just ensure application supplied the correct “read intent” option. You can learn more at [`https://learn.microsoft.com/azure/azure-sql/database/read-scale-out`](https://learn.microsoft.com/azure/azure-sql/database/read-scale-out).

Note

Read scale-out supports session-level consistency. It means, if the read-only session reconnects after a connection error caused by replica unavailability, it may be redirected to a replica that is not 100% up to date with the read-write replica. Likewise, if an application writes data using a read-write session and immediately reads it using a read-only session, it is possible that the latest updates are not immediately visible on the replica (milliseconds up to single-digit seconds). The latency is caused by an asynchronous transaction log redo operation.

Let’s look at an example where you can test out the built-in availability and a failover scenario for Azure SQL Database. For this example, I deployed a new Business Critical service tier database, **bwsqlbc**, using the same logical server I’ve been using in the book, **bwsqllogicalserver**. This database was deployed using the sample AdventureWorksLT and uses 2 vCores.

To try this example, you will also need the ostress.exe program I’ve used in previous chapters as described and in the prerequisites in the beginning of the chapter. You will use Azure PowerShell as part of this exercise, and I also have scripts you will use in the **ch8_availability** folder.

You can use any client you need. For me, I’ll use my own laptop because I want to authenticate with Azure using my Microsoft account with MFA:

1.  **Log in to Azure with PowerShell and set your subscription context.** I first had to log in to Azure with PowerShell using the following command:

```
Connect-AzAccount
```

This prompts me for MFA which is required at Microsoft. I also get a list of subscriptions to set as the context. Choose the right number and hit enter.

1.  **Edit the scripts to prepare for the test.**

    Edit the script **querybase_bc.cmd** to put in your server, database, login, and password. Edit the **failoverbase_bc.ps1** script to put in your resource group, server, and database. If you are unsure about your resource group, you can find it on the Overview screen of your logical server or database.

2.  In one command window, start the test by **executing querybase_bc.cmd** which should look similar to the following:

    ```
    ostress.exe -Sbwsqllogicalserver.database.windows.net -Q"SELECT COUNT(*) FROM SalesLT.Customer" -Usqladmin -dbwsqlbc -P -n1 -r50000 -T146
    ```

    This will display a repeating output of the count of rows in a table.

3.  In another command window, **execute the failoverbasedb.cmd script** which should look similar to the following:

    ```
    $resourceGroup = "bwsqldbrg"
    $server = "bwsqllogicalserver"
    $database = "bwsqlbc"
    Invoke-AzSqlDatabaseFailover -ResourceGroupName $resourceGroup -ServerName $server -DatabaseName $database
    ```

    When this command is successful, you are just brought back to the command window. Notice this command executes a failover *outside* of the context of the database and requires proper Azure RBAC permissions (e.g., Contributor level can perform this operation).

In other command window where you ran ostress, you should start to see connection failover errors. In seconds, you should see the ostress program reconnect and start to display row counts again.

You can now see how fast an application can reconnect when a failover occurs with your deployed Azure SQL Database.

A few comments are worth noting about this failover test:

*   A failover for a General Purpose service tier will take longer because it uses shared storage and requires a new node to spin up.

*   There is an equivalent command for Azure SQL Managed Instance called **Invoke-AzSqlInstanceFailover**.

*   You can only execute a manual failover every 15 minutes.

### Hyperscale Availability

I’ve described the unique characteristics of the Hyperscale service tier for Azure SQL Database in several chapters of the book so far. Let’s dive further into the pieces of the Hyperscale architecture, including the interesting way availability is handled.

Let’s look at a visual of the Hyperscale architecture in Figure [8-10](#496204_2_En_8_Chapter.xhtml#Fig10) and describe in more details the components when it comes to availability.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig10_HTML.jpg)

Figure 8-10

Hyperscale availability

The **primary compute node** is a primary replica for a Hyperscale deployment. Hyperscale has **zero to four secondary** **HA replicas** which are represented as **secondary compute nodes**. I’ll discuss more about how replicas work shortly. The primary compute node hosts a SQL Server for your database. This SQL Server has the standard components like a buffer pool to host database pages. Additionally on this primary compute node is your tempdb databases on local storage plus an RBEX *cache*. Caches in Hyperscale are all files on local SSD storage. RBEX stands for **resilient buffer pool extension**. It is similar but not exactly the same as the buffer pool extension (which you can read more about at [`https://learn.microsoft.com/sql/database-engine/configure-windows/buffer-pool-extension`](https://learn.microsoft.com/sql/database-engine/configure-windows/buffer-pool-extension)). The concept is that if a query needs a database page and it is not in buffer pool, it will first try to read that page from RBEX.

What if a page is not available in either the compute buffer pool or RBEX cache? We deploy a set of **page servers**. Page servers are nodes with SQL Servers that host database pages in memory and are covered by another RBEX cache. Page servers are *paired* for redundancy and high availability. Azure SQL determines the proper number of page servers to support the deployment and database size. Remember, for Hyperscale you don’t choose a maximum database size. We just keep growing and scaling the system through page server and Azure Storage to meet your database size.

Tip

We now support a way to shrink your Hyperscale database to save space and costs. See more at [`https://techcommunity.microsoft.com/t5/azure-sql-blog/public-preview-shrink-for-azure-sql-database-hyperscale/ba-p/4181976`](https://techcommunity.microsoft.com/t5/azure-sql-blog/public-preview-shrink-for-azure-sql-database-hyperscale/ba-p/4181976).

What if the database page the query needs is not on the compute node or a page server? Database files that back your database are stored in Azure standard storage. In order to maximize performance, this architecture works best when we rarely have to go to Azure Storage to retrieve a page for a database. When page servers first start up, they are seeded with pages from database files on Azure Storage. Page servers then will populate RBEX caches on the primary node (and secondaries if they exist). If a page is not on the primary compute buffer pool but we find it in the RBEX cache of the node, that is considered a *cache hit*. If the page is not in RBEX, we attempt to get the page from a page server (or its RBEX cache), but it is considered a *cache miss*.

Having database files on Azure Storage has one major advantage for automated backups and point-in-time restore for Hyperscale. Because the data files are infrequently accessed once the Hyperscale caching system is *warm*, we can back up the database using **snapshot backups**. This capability is very similar to using file-snapshot backups with Azure Virtual Machine as documented at [`https://learn.microsoft.com/sql/relational-databases/backup-restore/file-snapshot-backups-for-database-files-in-azure`](https://learn.microsoft.com/sql/relational-databases/backup-restore/file-snapshot-backups-for-database-files-in-azure). Snapshot backups are a huge advantage for Hyperscale because they don’t affect application operations and the restore of a database is extremely fast. Just like other Azure SQL options, backups are stored on redundant storage separate from the data and log files.

There is another piece I haven’t discussed yet from this model. What about the transaction log? Hyperscale redirects a transaction log I/O from the primary node to a **Log Service**. Compute nodes still have a log cache, but when an I/O is needed to flush log records, they are directed by the SQL Server engine to the Log Service. The Log Service runs on a different node. It has its own log cache (local SSD storage), log storage with Azure Premium Storage called the *landing zone*, and redundant storage with Azure Standard Storage called long-term log storage. We never have to back up the transaction log. A combination of snapshot backups and long-term log storage can be used to restore to a point in time.

The Log Service receives logged changes and then feeds these changes to update page servers and replicas (if they exist). This means that while Hyperscale uses logged changes to feed replicas, it doesn’t use the exact underlying technology of Always On Availability Groups to keep replicas in sync.

One interesting aspect to Hyperscale is database page I/O. For the compute node, dirty pages are not written to a database file. Hot pages on the compute node are written to RBEX cache, so they are readily available. Page servers are updated through logged changes with the Log Service. Therefore, any type of checkpoint I/O happens from a page server to database files on Azure Storage. This is very nice as it offloads any database file I/O from the compute node.

I mentioned earlier the concept that you can have zero to four secondary HA replicas. You may be asking how can this architecture support built-in high availability with zero replicas? Because the underlying database and transaction log files are on Azure Storage (and not on local storage), if we need to execute a failover, we simply provision a new node, and data will be synchronized from underlying page servers (which are backed by Azure Storage). Furthermore, like all of Azure SQL, Accelerated Database Recovery (ADR) is enabled, so when the new compute node comes online or a secondary replica becomes the primary, recovery to get to a consistent state is extremely fast. Every aspect of the Hyperscale distributed architecture is fault tolerant.

If we have secondary replicas provisioned, failover (which like other Azure SQL architectures is integrated with Azure Service Fabric) is of course much faster because we can just switch to one of those nodes to become the new primary node. In addition, if your application connects to the database with read intent, Azure SQL will load-balance across all available secondary replicas.

In addition to four HA replicas, you can have up to an additional 30 **named replicas**. HA replicas can be used for read workloads using an ApplicationIntent connection string for ReadOnly. Named replicas are specifically designed for read scale (but are not used for failover). They have their own unique database name and are strictly read-only, so do not need a special connection string attribute. You can read more about how to use named replicas at [`https://learn.microsoft.com/azure/azure-sql/database/service-tier-hyperscale-replicas?view=azuresql#named-replica`](https://learn.microsoft.com/azure/azure-sql/database/service-tier-hyperscale-replicas?view=azuresql#named-replica).

Your one-stop guide to Hyperscale can be found at [`https://aka.ms/hyperscale`](https://aka.ms/hyperscale).

Tip

Remember, because of the unique architecture of Hyperscale, you cannot change the service tier to General Purpose or Business Critical once you deploy. You would have to export out your data and import into the new tier. You can go back one time if you move from General Purpose to Hyperscale. This is a “size of data” operation, so the larger the database, the longer it takes to revert back.

## Extend HADR with Azure

While the built-in availability of Azure SQL is really a major advantage for you to consider moving to the cloud, there are ways to go even further and build in more availability and disaster recovery solutions with Azure.

This includes zone redundancy, geo-replication, failover groups, and disaster recovery with managed instance link. In addition, it is important to understand how the promise of the Azure SQL Service-Level Agreement meets your needs. This includes how we govern and limit certain activity and deploy innovative technologies like hot patching to maximize uptime.

### Zone Redundancy

I mentioned in Chapter [2](#496204_2_En_2_Chapter.xhtml) of the book features of the Azure ecosystem, including regions and data centers. One of the capabilities you can take advantage of to infuse ***high availability*** is to deploy with **Azure Availability Zones**. We call this capability for Azure SQL **zone redundancy**. Availability Zones are a collection of unique physical locations within a region. Each zone in the collection is made up of multiple data centers. Zone redundancy is available for General Purpose (with an extra charge) and Business Critical service tiers (included) for both Azure SQL Managed Instance and Azure SQL Database. Check on region availability for these options because they may not be available in all Azure regions. In addition, you can also enable zone redundancy for Hyperscale. Earlier in this chapter, I talked about backup redundancy, including zones. In this section, the focus is on redundancy of the storage of your database and log files and compute.

Let’s look at each service tier architecture and see how it looks after enabling zone redundancy. Figure [8-11](#496204_2_En_8_Chapter.xhtml#Fig11) shows the General Purpose service tier with zone redundancy.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig11_HTML.jpg)

Figure 8-11

General Purpose service tier with zone redundancy

First, notice that data and log files now use Zone Redundant Storage (ZRS) with Azure. This means these files are synchronized across all three zones (which means there can be a small performance degradation for this operation). At any given time, the primary replica or compute for SQL Server for the database is in a specific zone. Zone redundancy allows Azure to use a new node in a different zone should a failover occur. Notice at the top of the figure, gateways used to connect are also zone redundant routed through a traffic manager.

Business Critical service tier deployments now look like Figure [8-12](#496204_2_En_8_Chapter.xhtml#Fig12).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig12_HTML.jpg)

Figure 8-12

Business Critical service tier with zone redundancy

You can see in this figure the replicas for the BC tier are distributed across zones, so any failover can occur in a different zone should it be necessary. Storage is local to each replica. Like the GP tier, gateways are distributed across zones supported by a traffic manager.

Finally, Figure [8-13](#496204_2_En_8_Chapter.xhtml#Fig13) shows the Hyperscale architecture with zone redundancy.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig13_HTML.jpg)

Figure 8-13

Hyperscale with zone redundancy

You can see in Figure [8-13](#496204_2_En_8_Chapter.xhtml#Fig13) that replicas, page servers, the log service, and data files are all zone redundant. In order to use zone redundancy, you must have at least one HA replica. Figure [8-14](#496204_2_En_8_Chapter.xhtml#Fig14) shows how to enable zone redundancy for a Hyperscale database during deployment.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig14_HTML.jpg)

Figure 8-14

Enabling zone redundancy for Hyperscale

Note

You can only choose zone redundancy for Hyperscale at database creation time. Your backup redundancy must be zone redundant (or geo-zone-redundant) if you choose this option for Hyperscale.

### Geo-replication

Let’s say you want to go even further and make your database resilient *across* Azure regions. Azure SQL Database supports a concept called **geo-replication**. Geo-replication uses Always On Availability Group technology to *asynchronously* transmit log changes to another Azure SQL Database deployment on a different logical server. The secondary database can be in a different Azure region or the same Azure region. Secondary databases can be used for replica purposes.

Note

Using a geo-replicated database in the same region can give a General Purpose service tier a read replica or expand the number of replicas for a Business Critical service tier (which by default gets one).

Geo-replicated databases can be used for failover purposes which include unexpected Azure region events or to support an application upgrade with minimal downtime. However, failover is a manual process initiated by an administrator through Azure interfaces or through T-SQL (ALTER DATABASE).

Let’s see how to create a geo-replicated database in another region for a database I’ve deployed and used in the book called **bwadw**, which is a Hyperscale database.

I’ll navigate to my database in the Azure portal and select Replicas under Data Management like in Figure [8-15](#496204_2_En_8_Chapter.xhtml#Fig15).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig15_HTML.jpg)

Figure 8-15

Creating a geo-replica of a Hyperscale database

Notice at the top of the screen that, as mentioned earlier in this chapter, I can create a standby replica if I use the General Purpose or Business Critical service tier on my source database which can provide cost savings if the replica is passive.

I can on this screen select Create Replica and am presented with a screen very much like deploying a new database. You can see some of my choices in Figure [8-16](#496204_2_En_8_Chapter.xhtml#Fig16).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig16_HTML.jpg)

Figure 8-16

Deployment geo-replica details

I can use this same experience to create a named replica as I mentioned earlier in this chapter. Notice I’m choosing a different logical server in a different region as the deployment location, but the database name is the same. One of the nice features for geo-replication is that the database can be in the same region and even with the same logical server. This is an example of extending out a read scale-out scenario for General Purpose service tier deployments or giving you an additional read replica for Business Critical service tier deployments (or even a Hyperscale database with zero replicas).

The deployment creates a new database of the same name as the primary on the new logical server and then performs an initial synchronization called **seeding**. When this operation is complete, you can see the status of the new secondary like Figure [8-17](#496204_2_En_8_Chapter.xhtml#Fig17).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig17_HTML.jpg)

Figure 8-17

A geo-replicated secondary database after seeding

Any changes from my primary are asynchronously sent to the secondary (to avoid blocking primary workload transactions). In addition, I can now connect to [**bwadwserver.database.windows.net**](http://bwadwserver.database.windows.net) and perform read operations against the **bwadw** database. The database is read only, so any modification operations would fail. When I connect to the geo-replicated secondary database, I don’t need any special connection string option. Azure SQL supports up to four secondary databases configured with geo-replication.

Tip

You are allowed to create geo-secondary databases based on a geo-replicated secondary database, thus giving you even more read replica. This is called *chaining*, but know that these chained secondaries will likely have a lag of data synchronization the further you build the chain.

A geo-replicated database can also be used for failover purposes for disaster recovery. All failover operations are manual. Your data loss depends on whether data is synchronized to the replica and the availability of the source database. There are several methods to initiate a failover, including the Azure Portal, az CLI, PowerShell, and ALTER DATABASE. A “planned failover” or FAILOVER requires the source database to be available and a data synchronization to occur between the primary and secondary replicas. A FORCED FAILOVER does not wait for any synchronization and doesn’t require the source database to be available. The DMV **sys.dm_geo_replication_link_status** can be used to check the status of data synchronization between primary and secondary replicas.

Applications can use Azure technology like Azure Traffic Manager to set up an abstraction to connect to a primary even after failover. Learn more from our documentation about building applications for global Azure SQL Database deployments at [`https://learn.microsoft.com/azure/azure-sql/database/designing-cloud-solutions-for-disaster-recovery`](https://learn.microsoft.com/azure/azure-sql/database/designing-cloud-solutions-for-disaster-recovery). Our documentation at [`https://learn.microsoft.com/azure/azure-sql/database/active-geo-replication-overview`](https://learn.microsoft.com/azure/azure-sql/database/active-geo-replication-overview) has a nice visual showing what a fully deployed Azure SQL Database with geo-replication can look like as seen in Figure [8-18](#496204_2_En_8_Chapter.xhtml#Fig18).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig18_HTML.jpg)

Figure 8-18

A fully deployed Azure SQL Database using geo-replication

Here are some important points to consider when using geo-replication:

*   You can set up a geo-replicated database on another Azure subscription with a little work. Learn more at [`https://learn.microsoft.com/azure/azure-sql/database/active-geo-replication-overview?view=azuresql#cross-subscription-geo-replication`](https://learn.microsoft.com/azure/azure-sql/database/active-geo-replication-overview?view=azuresql#cross-subscription-geo-replication).

*   Server-level firewall rules on the primary are not replicated, so consider using database firewall rules or other methods to connect to the secondary.

*   Using contained database users (even with Microsoft Entra) has a huge advantage as they are replicated.

*   Geo-replication uses asynchronous replication of data. Therefore, if you fail over to a secondary, you might experience data loss (but not consistency). However, if you require a secondary to be completely in sync before a failover, you can use the **sp_wait_for_database_copy_sync** stored procedure. Learn more at [`https://learn.microsoft.com/azure/azure-sql/database/active-geo-replication-overview?view=azuresql#preventing-the-loss-of-critical-data`](https://learn.microsoft.com/azure/azure-sql/database/active-geo-replication-overview?view=azuresql#preventing-the-loss-of-critical-data).

*   The az CLI supports creating and configuring geo-replication (e.g., **az sql db replica**), and PowerShell cmdlets exist to support geo-replication (e.g., **New-AzSqlDatabaseSecondary**).

### Failover Group

Always On Availability Groups in SQL Server support the concept of a *group*, which is a collection of databases that are the unit of failover. **Failover groups** provide this same concept in Azure SQL both for Azure SQL Managed Instance and Azure SQL Database.

Figure [8-19](#496204_2_En_8_Chapter.xhtml#Fig19) shows the architecture of a failover group for Azure SQL Database (you can also see an architecture of failover groups for Azure SQL Managed Instance at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/failover-group-sql-mi?view=azuresql#failover-group-architecture`](https://learn.microsoft.com/azure/azure-sql/managed-instance/failover-group-sql-mi?view=azuresql#failover-group-architecture)).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig19_HTML.jpg)

Figure 8-19

Failover group for Azure SQL Database

Notice here that unlike geo-replication, failover groups operate at the logical server level (and then you place databases in the failover group).

Notice in this diagram that geo-replication is used to replicate any database in the group to the secondary. In addition, two listeners are provided to abstract the application from connecting to the primary replicas (read-write) and secondaries (read-only). And in this architecture, you can build your application as fault-tolerant and use Azure Traffic Manager to load-balance the workload (you would implement the Azure Traffic Manager solution).

Let’s go build a failover group in another region for the Azure SQL Database using the logical server I’ve been using in the book, **bwsqllogicalserver**. I’ll first navigate to my logical server in the Azure portal and select Failover groups from the Service Menu like in Figure [8-20](#496204_2_En_8_Chapter.xhtml#Fig20).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig20_HTML.jpg)

Figure 8-20

Creating a failover group for a logical server

Now I’ll select **Add group** to create the failover group. You can see in Figure [8-21](#496204_2_En_8_Chapter.xhtml#Fig21) my choices to create the new group.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig21_HTML.jpg)

Figure 8-21

Creating the details of a failover group

First, I need to provide the group a name. This becomes the logical name of the group to connect for both read-write and read-only (very much like a listener for an Always On Availability Group).

I also provided the definition of a new logical server in a new region (it could be an existing logical server but must be in another region).

Now I can choose which databases to add to the group by clicking Configure database. I now get a screen like Figure [8-22](#496204_2_En_8_Chapter.xhtml#Fig22).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig22_HTML.jpg)

Figure 8-22

Choosing databases and a failover group

I’ve chosen two databases I’ve already deployed to be in the group. Here is the cool part of failover groups. The databases do not have to be configured the same to be in the group. In this example, I’ve chosen a Business Critical service tier and a Hyperscale database. Notice I can choose to make these standby replicas, and when you make this choice, you can see your cost estimate decrease. I’ll leave this as No and click **Select**.

I selected **Create** which fires off a new deployment for the failover group. When the deployment finished (which includes seeding like geo-replication), Figure [8-23](#496204_2_En_8_Chapter.xhtml#Fig23) shows the updated screen for the status of the failover group.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig23_HTML.jpg)

Figure 8-23

Failover group after deployment

Notice the Read/Write failover policy says **Customer Managed**. There are two types of failover policies:

*   **Customer managed (recommended and default)** – Customers can perform a failover of a group when they notice an unexpected outage impacting one or more databases in the failover group. When using command-line tools such as PowerShell, the Azure CLI, or the REST API, the failover policy value for customer managed is manual.

*   **Microsoft managed** – In the event of a widespread outage that impacts a primary region, Microsoft initiates failover of all impacted failover groups that have their failover policy configured to be Microsoft managed. Microsoft-managed failover won’t be initiated for individual failover groups or a subset of failover groups in a region. When using command-line tools such as PowerShell, the Azure CLI, or the REST API, the failover policy value for Microsoft managed is automatic.

Note

This feature was known in the first edition of the book as auto-failover groups because the default policy was to automatically fail over. We have learned over the last few years that customers are best protected by managing the failover process, hence the name change and adoption of these new policy types.

You can read all the details of failover group policies at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/failover-group-sql-mi?view=azuresql#failover-policy`](https://learn.microsoft.com/azure/azure-sql/managed-instance/failover-group-sql-mi?view=azuresql#failover-policy).

If you click the failover group, you get a very nice global map visual with options to manage the failover group and connection information for both the primary server and read-only replicas like Figure [8-24](#496204_2_En_8_Chapter.xhtml#Fig24).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig24_HTML.jpg)

Figure 8-24

Configuration and managing a failover group in the Azure portal

You can see from this screen you get a visual map of the locations of the failover group logical server and details of both primary and secondary replicas. Below are the new endpoints (these become your new logical server names) to connect for read-write or read-only workloads.

At the top, you can add or remove databases from the group or initiate a planned or forced failover.

Here are a few other important points about failover groups for Azure SQL:

*   Failover groups can also be managed using the PowerShell (**Add-AzSqlDatabaseToFailoverGroup** and **Switch-AzSqlDatabaseFailoverGroup**).

*   Because failover groups use asynchronous replication data when a failover occurs automatically, *data loss could occur*. Therefore, if you require no data loss with failover groups, applications can call the **sp_wait_for_database_copy_sync** procedure after committing a transaction to ensure all data is synchronized. Learn more at [`https://learn.microsoft.com/azure/azure-sql/database/failover-group-sql-db?view=azuresql&tabs=azure-powershell#preventing-the-loss-of-critical-data`](https://learn.microsoft.com/azure/azure-sql/database/failover-group-sql-db?view=azuresql&tabs=azure-powershell#preventing-the-loss-of-critical-data).

*   One of the options you can configure for Microsoft-managed failover policy is called a *grace period* using the **GracePeriodWithDataLossHours** parameter (the default is one hour). This parameter defines the time we will wait to do an automatic failover should the primary be down, and we believe data loss may occur. If no data loss occurs, the automatic failover takes place immediately.

One important limitation for failover groups involves system databases. System databases are **not** replicated. Therefore, any instance-level data, such as SQL Server Agent jobs, must be manually created on the secondary instance.

### Disaster Recovery with Managed Instance Link

We announced with SQL Server 2022 a new way to use the cloud for disaster recovery. I call it ***managed*** disaster recovery. I remember in the fall of 2022 speaking with a customer in Dallas. They told me the sad tale of an employee that had left the company that managed their DR site. Unfortunately, they had no documentation or details on the site and how to manage it. After I explained our managed disaster recovery story, this customer said to me, “Sign me up!”

This feature is encompassed into a capability called disaster recovery with Managed Instance link. You learned earlier in the book how to use Managed Instance link for migration.

Figure [8-25](#496204_2_En_8_Chapter.xhtml#Fig25) shows how to use it for both offline and online disaster recovery.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig25_HTML.jpg)

Figure 8-25

Disaster recovery with Managed Instance link

Let’s look at both scenarios and how they work in more detail.

#### Offline Disaster Recovery

Taking advantage of a *versioned* Azure SQL Managed Instance, you can create a link between SQL Server 2022 and the instance. Behind the scenes, Availability Group technology is used (even if you don’t have an AG on SQL Server 2022, including Standard Edition) akin to a Distributed Availability Group (DAG). Creating the link seeds the database from SQL Server 2022 to the already deployed Managed Instance.

Changes are now automatically sent (asynchronously) from SQL Server to Managed Instance. If you want to fail over to the Managed Instance, you can choose a planned (you must ensure the data is synched) or forced failover. Once you fail over, the Managed Instance become the “read-write” primary. If you want to fall back, you can execute a COPY_ONLY backup on the Managed Instance to Azure Storage and restore it back to SQL Server 2022\. Now SQL Server can become the primary replica again. This technique is considered offline because the time it takes to get back online for SQL Server 2022 is the time it takes to restore a full database backup.

#### Online Disaster Recovery

The offline scenario is what we shipped as a public preview when SQL Server 2022 shipped and is now generally available. In preview now at the time of the writing of this book is a new method to set up SQL Server 2022 and Managed Instance as *failover partners*, much like a true Distributed Availability Group (DAG). In this scenario, you can fail over to the Managed Instance just as you did in the offline scenario. But a failover is more like “switching roles.” This way, any switch is an online operation since very few changes would have to be synchronized to the secondary before the switch operation takes place. Once you switch to Managed Instance, you can switch back to SQL Server 2022.

In both scenarios, if the primary is offline, you can always force a failover to the secondary with the knowledge that you might have data loss. Why did I start this section calling this feature managed disaster recovery? Because your disaster recovery site, which is an Azure SQL Managed Instance, is managed by Microsoft. You have an SLA for availability as well as automated backups. No one needs a disaster recovery site until well…a disaster happens, but wouldn’t it be nice to know it is always there and ready when needed?

Get started with disaster recovery with Managed Instance link at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/managed-instance-link-feature-overview?view=azuresql#disaster-recovery`](https://learn.microsoft.com/azure/azure-sql/managed-instance/managed-instance-link-feature-overview?view=azuresql#disaster-recovery).

### Azure SQL SLA

One of the advantages to deploy SQL Server with Azure SQL Managed Instance and Database is the promise of availability. Our architecture which I’ve described in this chapter helps us achieve this promise. The promise for you is in the form of a **Service-Level Agreement (SLA)**. You can view the official SLA for Azure SQL at [`https://azure.microsoft.com/support/legal/sla/sql-database`](https://azure.microsoft.com/support/legal/sla/sql-database).

The Azure SQL SLA means that Microsoft will ensure we maintain a **service level**, or you will be eligible for a credit for your account. Service levels are stated in terms of *nines*. Nines are a percentage of uptime or availability you are guaranteed for your deployment.

As an example, if you deploy an Azure SQL Database with the Business Critical or Premium service tier and use zone redundancy, your SLA is 99.995%. If you look at a table like [`https://en.wikipedia.org/wiki/High_availability`](https://en.wikipedia.org/wiki/High_availability), you can see that 99.995% is defined as “four and a half nines” and means you could experience at maximum 26.30 minutes per year or 2.19 minutes per month of *downtime*. Other deployment options have different service-level promises.

If you look at our SLA documentation, downtime is defined as “…the total accumulated Deployment Minutes across all Databases in a given Microsoft Azure subscription during which the Database is unavailable. A minute is considered unavailable for a given Database if all continuous attempts by Customer to establish a connection to the Database within the minute fail.”

There are certain aspects to Azure SQL which allow us to make these SLA promises, including but not limited to

*   Built-in availability and integration with the Azure Service Fabric

*   Enforcing resource limits such as log governance

Note

There are several reasons why log governance is needed to manage PaaS services. This includes database recoverability, high availability, disaster recovery, and predictable performance. Learn more at [`https://azure.microsoft.com/blog/resource-governance-in-azure-sql-database/`](https://azure.microsoft.com/blog/resource-governance-in-azure-sql-database/).

*   Enabling database options such as Accelerated Database Recovery.

One innovative technology we use in Azure SQL to maximize availability is **hot patching**. Hot patching allows us to patch the SQL Server engine code without restarting SQL Server. Read the amazing story of hot patching at [`https://azure.microsoft.com/blog/hot-patching-sql-server-engine-in-azure-sql-database/`](https://azure.microsoft.com/blog/hot-patching-sql-server-engine-in-azure-sql-database/).

## Database Availability and Consistency

For SQL Server, you may be familiar with features and tools to make or restrict the availability of your database or perform advanced recovery scenarios. In addition, SQL Server provides tools to ensure the database is consistent, both from a physical and logical perspective.

Azure SQL in general does not provide the same level of advanced capabilities in this area mainly because they are not needed given the high level of redundancy and availability built into the service.

Let’s examine a few of these areas, so your knowledge can be more complete when comparing to SQL Server.

### Database Availability

You may have needed with a SQL Server to change the database state with ALTER DATABASE to OFFLINE or EMERGENCY for advanced recovery scenarios. You don’t have access to use these options, but after reading about all of the built-in capabilities of Azure SQL and our SLA, you have to ask, does it matter? In my opinion (and believe me I’ve used these options to help customers over the years in support), the answer is no.

With Azure SQL Database and Managed Instance, while you cannot put the data in single-user mode, Azure SQL Database allows you to use the option RESTRICTED_USER. Learn more at [`https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql-set-options`](https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql-set-options).

In SQL Server 2005, my colleague Robert Dorr and I were in Microsoft support and convinced the engineering team to create a simple method to connect into a “hung server.” The result was a feature called Dedicated Admin Connection (DAC). DAC *is supported* for Azure SQL. Learn more at [`https://learn.microsoft.com/sql/database-engine/configure-windows/diagnostic-connection-for-database-administrators`](https://learn.microsoft.com/sql/database-engine/configure-windows/diagnostic-connection-for-database-administrators).

### Accelerated Database Recovery (ADR)

In my book, *SQL Server 2019 Revealed*, I covered the amazing story of Accelerated Database Recovery (ADR). No longer will a SQL Server administrator worry about long-running database recovery times or out-of-control transaction logs. ADR is not just a feature; it is part of the Azure SQL availability story! In fact, for Azure SQL it is just part of the engine and not really something you turn on or off. You can learn more about how ADR works in our documentation at [`https://learn.microsoft.com/azure/azure-sql/accelerated-database-recovery`](https://learn.microsoft.com/azure/azure-sql/accelerated-database-recovery) or in the whitepaper written by our engineering team at [`https://aka.ms/sqladr`](https://aka.ms/sqladr).

### Database Consistency

All Azure SQL databases are configured using the CHECKSUM option for database consistency. One of the benefits of using a PaaS service is that our engineering team has automation to check for any inconsistencies due to issues like a checksum problem and take correction action. For example, if you are deploying a Business Critical service tier, we can issue an online automatic page repair (learn more about how this works at [`https://learn.microsoft.com/sql/sql-server/failover-clusters/automatic-page-repair-availability-groups-database-mirroring`](https://learn.microsoft.com/sql/sql-server/failover-clusters/automatic-page-repair-availability-groups-database-mirroring)).

In addition, keep in mind these facts if you are concerned about any database inconsistency issue:

*   General Purpose and Hyperscale store database and log files on Azure Storage which is mirrored by default with three copies.

*   Business Critical tiers have three other replicas always available with their own storage.

*   Our engineering team has built-in data integrity and consistency alert monitoring in our service. If automation can’t solve the problem, we will directly notify a customer and take necessary steps to ensure data is restored and consistent. If we think we can repair a problem with no data loss, we might take this action, and you never have to be notified.

*   Azure SQL does support DBCC CHECKDB (but not repair option) for you to manually check your database consistency at any time.

*   We’ve added checks for databases for “lost write” and “stale read” detection which we have seen in some situations occur due to an underlying I/O system issue.

Peter Carlin, Vice President at Microsoft, who I mentioned as part of Azure SQL history in Chapter [1](#496204_2_En_1_Chapter.xhtml), has a very nice blog post outlining all the things we do to manage data integrity for your database in Azure. Read his post at [`https://azure.microsoft.com/blog/data-integrity-in-azure-sql-database`](https://azure.microsoft.com/blog/data-integrity-in-azure-sql-database/)[`/`](https://azure.microsoft.com/blog/data-integrity-in-azure-sql-database/).

## Monitoring Availability

As with any set of capabilities, you will no doubt want to monitor various aspects of availability for Azure SQL. This includes server and instance availability, database availability, backup/restore history, status of replicas, and failover reasons. In addition, since Azure SQL runs in the Azure ecosystem, knowing the status and health of Azure services in regions and data centers can also be important.

Azure SQL provides you similar interfaces to SQL Server to monitor availability, including catalog views, Dynamic Management Views (DMV), and Extended Events (XEvent). In addition, Azure interfaces such as the Azure portal, az CLI, PowerShell, Database Watcher, and Copilot provide additional capabilities to monitor the availability of your deployment.

Let’s dive into a few examples of using these interfaces and monitoring capabilities.

### Instance, Server, and Database Availability

Aside from Azure service impacting events, you can view the availability of your Azure SQL Managed Instance or Azure SQL Databases through the Azure portal. One of the primary methods to view a possible reason for a Managed Instance or Database to not be available is by examining **Resource Health** through the Azure portal. You will see an example of this scenario later in this major section of the chapter.

You can always use standard SQL Server tools such as SQL Server Management Studio to connect to a Managed Instance or Database server and check the status of these resources through the tool or T-SQL queries.

In addition, interfaces such as the az CLI can show the status of Azure SQL such as

*   **az sql mi list** – List the status of managed instances

*   **az sql db list** – List the status of Azure SQL Databases

PowerShell commands can also be used to find out the availability of an Azure SQL Database such as

*   **Get-AzSQLDatabase** – Get all the databases on a server and their details, including status

Note

For a SQL Server, I often look at past ERRORLOG files or the system_health XEvent session files for service availability and health. Azure SQL Managed Instance supports these tools. However, these files are not copied to replicas, so if a failover occurs, the history of these files are lost.

### Backup and Restore History

Azure SQL automatically backs up databases and transaction logs. For SQL Server, it is very common to use the msdb system tables to look at backup history. You can also do this in Azure SQL Managed Instance with a query like the following:

```
SELECT database_name, case when type = 'D' then 'Full' when type = 'I' then 'Differential' when type = 'L' then 'Log' end as type,
has_backup_checksums, backup_size, compressed_backup_size, backup_start_date, backup_finish_date
FROM msdb.dbo.backupset
ORDER BY backup_finish_date DESC;
GO
```

I’ve been running my instance for a while, so I scrolled down to see the older results, which looked like the one in Figure [8-26](#496204_2_En_8_Chapter.xhtml#Fig26).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig26_HTML.jpg)

Figure 8-26

Older results for automated backups for managed instance

These results show a user database I’ve created, but it will show any user database that has been created for the instance (it doesn’t show system database backups). Notice the first result is a full backup and then a series of log backups. Scrolling the top of the results, you can see other types of backups as in Figure [8-27](#496204_2_En_8_Chapter.xhtml#Fig27).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig27_HTML.jpg)

Figure 8-27

Recent results for automated backups for Managed Instance

Long-term backup retention history can be viewed through the Azure portal or CLI interfaces. Learn more at [`https://learn.microsoft.com/azure/azure-sql/database/long-term-backup-retention-configure`](https://learn.microsoft.com/azure/azure-sql/database/long-term-backup-retention-configure).

Additionally, Azure SQL Managed Instance supports using XEvents to track backup history. See a blog post describing how to do this at [`https://techcommunity.microsoft.com/t5/azure-database-support-blog/lesson-learned-128-how-to-track-the-automated-backup-for-an/ba-p/1442355`](https://techcommunity.microsoft.com/t5/azure-database-support-blog/lesson-learned-128-how-to-track-the-automated-backup-for-an/ba-p/1442355).

Any restore of a database using point-in-time restore results in the creation of a new database, so the history of restore can be viewed as looking at the creation of a new database. All operations to create a new database can be viewed through Azure Activity Logs.

Msdb backup history is not available for Azure SQL Database, but a DMV is available called **sys.dm_database_backups**. I ran this query against one of my Azure SQL Databases (backup size details are not available):

```
SELECT logical_database_name, case when backup_type = 'D' then 'Full' when backup_type = 'I' then 'Differential' when backup_type = 'L' then 'Log' end as backup_type_desc, backup_start_date, backup_finish_date, in_retention
FROM sys.dm_database_backups
ORDER BY backup_finish_date DESC;
GO
```

For my Business Critical service tier database, the older results looked like Figure [8-28](#496204_2_En_8_Chapter.xhtml#Fig28).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig28_HTML.jpg)

Figure 8-28

Older results for automated backups from Azure SQL Database

Backups retained and shown in the backup history view depend on configured backup retention. Backups older than the retention period (in_retention = 0) are also shown in the sys.dm_database_backups view. They’re needed to do point-in-time restore within the configured retention.

Scrolling up, I can see the latest results like in Figure [8-29](#496204_2_En_8_Chapter.xhtml#Fig29) which include log and differential backups.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig29_HTML.jpg)

Figure 8-29

Top results for automated backups for Azure SQL Database

Tip

Our new **Copilot** can also be used to find information about backups with a prompt like “**When was the most recent backup of my database created?**”

### Region, Data Center, and Service Availability

To get a global view of the status of Azure regions and data centers, use the **Azure status** dashboard which you can find at [`https://status.azure.com`](https://status.azure.com). Figure [8-30](#496204_2_En_8_Chapter.xhtml#Fig30) shows an example of the Azure status dashboard with a default view of Current Impact.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig30_HTML.jpg)

Figure 8-30

The Azure status dashboard for current impact

Azure status can show the status of all Azure services in any Azure region. The status shows all services independent of your use of a specific service. I selected Americas on this view, and you can see the results in Figure [8-31](#496204_2_En_8_Chapter.xhtml#Fig31).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig31_HTML.jpg)

Figure 8-31

Azure service status for Americas

This is a long table as you can scroll down and see the status for any service. Any service that has a “–” means that service is not available in that region.

To get notified of Azure status, you can use the RSS feed at the top of the page. In addition, you can see a complete history of Azure status through Azure status history at [`https://status.azure.com/status/history`](https://status.azure.com/status/history).

You can also get more information about the health of Azure services specific to your subscription through the Azure portal for a capability called **Azure Service Health**. Through Service Health, you can see current issues for Azure services, planned maintenance that could affect availability, and health history. Search **Service Health** at the top of your portal or find it on your dashboard. When I selected this and selected Health History, you can see my results in Figure [8-32](#496204_2_En_8_Chapter.xhtml#Fig32).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig32_HTML.jpg)

Figure 8-32

Azure Service Health history

The events here include advisories of possible future issues to resolve and planned maintenance operations. On the Service Menu, you can see options to look at things like current service issues or even set up alerts for topics like planned maintenance.

### Availability Metrics

Let’s say you would like to track at a more granular level the availability of your Azure SQL Database to see how this lines up with the promised SLA. Azure Metrics from the portal has a new way to do this as you can see in Figure [8-33](#496204_2_En_8_Chapter.xhtml#Fig33).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig33_HTML.jpg)

Figure 8-33

Availability metric in the Azure Portal

I changed the time interval for the last 30 days, but this value can also be retrieved for a longer history with logs. In addition, like other Azure Metrics, you can set up alerts to see if this value changes from 100%. Our calculation is based on a combination of averages from detailed connection metrics. Learn more at [`https://learn.microsoft.com/azure/azure-sql/database/monitoring-metrics-alerts?view=azuresql#availability-metric`](https://learn.microsoft.com/azure/azure-sql/database/monitoring-metrics-alerts?view=azuresql#availability-metric).

What if I didn’t know about this new metric and wanted to see if our new Copilot can help. I used the prompt “What is the availability of this database over the last 30 days?” and got a response as you can see in Figure [8-34](#496204_2_En_8_Chapter.xhtml#Fig34).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig34_HTML.jpg)

Figure 8-34

Copilot showing availability metrics for my database

### Replica Status

To monitor the state of replicas in Azure SQL, you can use the DMV **sys.dm_database_replica_states**. This DMV could be used, for example, to look at the status of replicas for a Business Critical service tier. You could also use this DMV to check the status for replicas for a deployment with geo-replication or failover groups.

For the database I deployed in this chapter called **bwsqlbc**, I can connect with SSMS to this database and run the following T-SQL statement:

```
SELECT is_primary_replica, synchronization_state_desc, synchronization_health_desc, last_received_time, last_redone_time
FROM sys.dm_database_replica_states;
GO
```

I get the results from this database as you can see in Figure [8-35](#496204_2_En_8_Chapter.xhtml#Fig35).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig35_HTML.jpg)

Figure 8-35

Replica status for an Azure SQL Database using Business Critical

For geo-replication and failover groups, there are additional DMVs to check on the status of replication between logical server and instances. This includes **sys.geo_replication_links** (run in the context of the logical master) and **sys.dm_geo_replication_link_status** (run in the context of the user database). One example of using these DMVs is seeding (the initial sync of the geo-secondary). sys.geo_replication_links can show the state of the seeding process as it progresses and completes.

In this chapter, I configured a failover group. Let’s connect to the failover group server and see what these DMVs look like:

I connected to the failover group server **bwazuresql.database.windows.net** in the context of one of the databases in the group and ran the following T-SQL statement:

```
SELECT partner_server, partner_database, replication_lag_sec, replication_state_desc, role_desc
FROM sys.dm_geo_replication_link_status;
GO
```

I got the results as you can see in Figure [8-36](#496204_2_En_8_Chapter.xhtml#Fig36).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig36_HTML.jpg)

Figure 8-36

Failover group replica status

A **replication_state_desc** = CATCHUP means the servers are synchronized.

You can see the full documentation of **sys.dm_geo_replication_link_status** at [`https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-geo-replication-link-status-azure-sql-database`](https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-geo-replication-link-status-azure-sql-database).

All of these DMVs to check replica status work for both Azure SQL Managed Instance and Database. The az CLI can be used to check on replica status (**az sql db replica**). PowerShell cmdlets exist to support checking replica status, for example, **Get-AzSqlDatabaseReplicationLink**.

Another way to see the status of a geo-replicated database is to use Database Watcher. I set up Database Watcher back in Chapter [7](#496204_2_En_7_Chapter.xhtml) to track the database **bwadw**. I also in this chapter set up a geo-replication for this database. I can use Database Watcher to get more details on the geo-replicated status as you can see in Figure [8-37](#496204_2_En_8_Chapter.xhtml#Fig37).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig37_HTML.jpg)

Figure 8-37

Using Database Watcher to see geo-replicated database status

### Failover Reasons

There are various reasons why a failover can occur, planned and unplanned, for Azure SQL Managed Instance and Database. Because the reasons for failover could vary, the best method to track if a failover occurred for your database is to use **Resource Health** for your Managed Instance or Database. Figure [8-38](#496204_2_En_8_Chapter.xhtml#Fig38) shows an example of Resource Health for one of my Azure SQL Database deployments and a health event which resulted in a failover.

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig38_HTML.jpg)

Figure 8-38

Resource Health for an Azure SQL Database

The **Unavailable** event lines up when I performed a manual failover of the database. Notice it says it was an Unplanned event. How could I know whether this was caused by a manual failover? I looked at the Activity Log for this database and found these details as you see in Figure [8-39](#496204_2_En_8_Chapter.xhtml#Fig39).

![](images/496204_2_En_8_Chapter/496204_2_En_8_Fig39_HTML.jpg)

Figure 8-39

Using the Activity Log to track a manual failover

Note

Resource Health is reported at the database level for both Azure SQL Managed Instance and Database deployments.

## Summary

In this chapter, you learned the amazing built-in availability capabilities for Azure SQL, including automatic backups, point-in-time restore, and built-in availability architectures of General Purpose, Business Critical, and Hyperscale.

You also learned how to use the power of Azure for further redundancy with zone redundancy, geo-replication, and failover groups. You learned how Azure provides built-in data integrity capabilities and processes and how to monitor the availability of your Azure SQL deployments.

There are some additional labs you can use (these are free – you just need your Azure subscription) to learn more about availability for Azure SQL at

[`https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/06_Manage_and_Optimize_AzureSQLMI`](https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/06_Manage_and_Optimize_AzureSQLMI)

[`https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/07_Deploy_Manage_Optimize_AzureSQLDB`](https://github.com/microsoft/cloudsqlworkshop/tree/main/cloudsqlworkshop/07_Deploy_Manage_Optimize_AzureSQLDB)

Availability is the last part of the meat and potatoes of Azure SQL. In the next chapter, you will extend your knowledge of Azure SQL to learn capabilities not specifically related to security, performance, or availability.

# 9. Extending Your Knowledge of Azure SQL

You have been through the journey of deploying and configuring Azure SQL Managed Instance and Databases. You have then seen all the capabilities and tasks you need to secure your deployments, make and keep Azure SQL fast, and ensure your data is highly available. In this chapter, we will extend your knowledge for Azure SQL by looking closer at a feature comparison with SQL Server, understanding options for job management, see methods you can use to help support your deployments, and review best practices for using Azure SQL.

This chapter will contain examples for you to try out and use as you read along. For you to try out any of the techniques, commands, or examples I use in this chapter, you will need

*   An Azure subscription.

*   A minimum of Contributor role access to the Azure subscription. You can read more about Azure built-in roles at [`https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles`](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles).

*   Access to the Azure Portal.

*   I will use several Azure SQL Databases and a Managed Instance I deployed earlier in this book.

*   To connect to Managed Instance, you will need a *jumpbox* or virtual machine in Azure to connect. I showed you how to do this in Chapter [4](#496204_2_En_4_Chapter.xhtml) of the book. One simple way to do this is to create a new Azure Virtual Machine and deploy it to the same virtual network as the Managed Instance (you will use a different subnet than the Managed Instance).

*   To connect to Azure SQL Database, I’m going to use the Azure VM I deployed in Chapter [3](#496204_2_En_3_Chapter.xhtml), called **bwsql2022**, and configured for a private endpoint in Chapter [6](#496204_2_En_6_Chapter.xhtml) (you could use another method as long as you can connect to the Azure SQL Database).

*   You will run some T-SQL in this chapter, so install a tool like SQL Server Management Studio (SSMS) at [`https://aka.ms/ssms`](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15).

## Surface Area of Azure SQL

Throughout this book, I have compared the features and capabilities of SQL Server with Azure SQL Managed Instance and Database. Let’s review a few capabilities I often get asked that are not specifically related to security, performance, and availability.

As you read through this section, consider these documentation references:

*   T-SQL differences between Managed Instance and SQL Server at [`https://aka.ms/azuresqlmitsqldiff`](https://aka.ms/azuresqlmitsqldiff).

*   T-SQL differences between Azure SQL Database and SQL Server at [`https://aka.ms/azuresqldbtsqldiff`](https://aka.ms/azuresqldbtsqldiff).

*   Feature comparison between Azure SQL Managed Instance and Azure SQL Database at [`https://aka.ms/azuresqlfeatures`](https://aka.ms/azuresqlfeatures).

### Linked Servers and Cross-Database Queries

SQL Server users can execute queries for objects (and joining them) that reside in other databases on the same SQL Server instance (cross-database query) or another instance (linked server query). Linked servers for SQL Server also allow queries against other data providers (through OLE-DB) such as Oracle. Cross-database queries and linked servers allow read queries (SELECT) but also (assuming the provider supports it) distributed transactions.

**Cross-database queries** are supported in Azure SQL Managed Instance but not with Azure SQL Database.

Note

Azure SQL Database does support the concept of elastic database query (in preview) which can allow joining data across databases. Learn more at [`https://learn.microsoft.com/azure/azure-sql/database/elastic-query-getting-started-vertical`](https://learn.microsoft.com/azure/azure-sql/database/elastic-query-getting-started-vertical).

**Linked Servers** are supported for Azure SQL Managed Instance but not with Azure SQL Database. There are limitations though for how you can use Linked Servers with Azure SQL Managed Instance. For example, only SQL Server–based providers are supported. See the full list of differences for Linked Servers on Managed Instance at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server?view=azuresql#linked-servers`](https://learn.microsoft.com/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server?view=azuresql#linked-servers). Linked Servers for Managed Instance support Microsoft Entra authentication.

Note

You can create a Linked Server from a SQL Server on-premises, in Azure Virtual Machine, or Managed Instance, where the *remote server* is an Azure Database Logical Server.

### External Tables

In SQL Server 2016, we introduced a capability called Polybase using a T-SQL object EXTERNAL TABLE to be directed to Hadoop file systems. In SQL Server 2019, we expanded this capability to include other data sources such as SQL Server, Oracle, MongoDB, Teradata, and other sources that support an ODBC Driver. We call this capability *data virtualization*, because the data access is remote, but it *feels* like the data is local through external tables.

In SQL Server 2022, we introduced a new method for Polybase by supporting external data sources for Azure Storage using REST APIs instead of the previous methods with ODBC. REST provides a more lightweight approach since it executes inside the SQL engine. In addition, external tables can be used to read or write data to file formats like Parquet, CSV, and Delta (read only).

The process to use Polybase with Azure SQL Managed Instance is like SQL Server 2022 as you see in Figure [9-1](#496204_2_En_9_Chapter.xhtml#Fig1).

![](images/496204_2_En_9_Chapter/496204_2_En_9_Fig1_HTML.jpg)

Figure 9-1

Data virtualization using REST API for SQL Server 2022

For Azure SQL Managed Instance, a more managed approach is available. While you still have external data sources, file formats, and external tables, here are the following differences from Figure [9-1](#496204_2_En_9_Chapter.xhtml#Fig1):

*   There is no need to install the Polybase feature as it is already “installed” for Managed Instance.

*   There is no need to configure Polybase using sp_configure as it is already enabled.

*   S3 is not supported at the time of the writing of this book, so only Azure Blob Storage (abs) and Azure Data Lake Storage (adls) data sources are supported.

You still need a database scoped credential for any private storage account, but one nice addition for Azure SQL Managed Instance from SQL Server 2022 is that managed identities are also supported.

The best way for you to get started using this feature is to connect to your Azure SQL Managed Instance and try the following query against a known public dataset:

```
SELECT *
FROM OPENROWSET(
BULK 'abs://nyctlc@azureopendatastorage.blob.core.windows.net/yellow/puYear=2001/puMonth=1',
FORMAT = 'parquet'
) AS taxidata;
GO
```

You can see how easily it is to get started with data virtualization *with no configuration* required for your Managed Instance.

Another powerful feature of data virtualization is the ability to create an external table and export data at the same time. We called this CREATE TABLE AS SELECT (CETAS). One of the benefits of this capability is to take the result of this data as Parquet files and create a “cold data” solution integrated with Microsoft Fabric. The scenario looks like Figure [9-2](#496204_2_En_9_Chapter.xhtml#Fig2).

![](images/496204_2_En_9_Chapter/496204_2_En_9_Fig2_HTML.jpg)

Figure 9-2

Using data virtualization for integration with Microsoft Fabric

The concept here is to switch a SQL partition that is considered “cold” to a separate table. Then use the CETAS concept to export this table to Azure Storage as Parquet. Since this is an external table, you can query the parquet data as a table. In addition, you can use the concept of a shortcut in Microsoft Fabric to easily integrate this data into Microsoft Fabric. You can see this in action on an episode of Data Exposed I was in hosted by Anna Hoffman at [`https://learn.microsoft.com/shows/data-exposed/sql-integration-with-microsoft-fabric-data-exposed`](https://learn.microsoft.com/shows/data-exposed/sql-integration-with-microsoft-fabric-data-exposed). You will see in the next chapter a more efficient method to synchronize your SQL data with the Microsoft Fabric with a concept called *mirroring*.

Learn more about data virtualization with Azure SQL Managed Instance at [`https://aka.ms/midatavirt`](https://aka.ms/midatavirt).

While Azure SQL Database does not support Polybase in this fashion, it does support external tables to use for elastic scale query. This means that you can technically set up an **external data source** to another Azure SQL Database logical server and database and define an external table with that data source. So, you can now query the remote database as an external table or even join a local table with the remote database table. You can review the T-SQL reference for external data source for Azure SQL Database at [`https://learn.microsoft.com/sql/t-sql/statements/create-external-data-source-transact-sql?view=azuresqldb-current&tabs=dedicated#overview-azure-sql-database`](https://learn.microsoft.com/sql/t-sql/statements/create-external-data-source-transact-sql?view=azuresqldb-current&tabs=dedicated#overview-azure-sql-database).

Take a look at a cool example of using this capability from my colleague Dimitri Furman where he uses external tables to look at Hyperscale read-scale monitoring. You can find the source at [`https://github.com/dimitri-furman/samples/tree/master/azure-sql-readscale-monitoring`](https://github.com/dimitri-furman/samples/tree/master/azure-sql-readscale-monitoring)

### Database Mail

SQL Server has supported a built-in *mail* capability since back in the days of SQL Server 4.21 for Windows NT. The latest version of this feature is called **Database Mail**. The concept is that you can use T-SQL to send mail messages based on the Simple Mail Transfer Protocol (SMTP). If you have never used Database Mail before, you can learn more at [`https://learn.microsoft.com/sql/relational-databases/database-mail/database-mail`](https://learn.microsoft.com/sql/relational-databases/database-mail/database-mail).

We introduced support for Database Mail as a feature for Azure SQL Managed Instance as this was a capability many customers who were trying to use Azure SQL Database desired as part of their migration. Database Mail is an *instance* feature which makes sense why it is part of Azure SQL Managed Instance.

One of the common scenarios for using Database Mail is for alerts for SQL Server Agent jobs. There are a few configuration differences to use Database Mail for SQL Server Agent jobs. Read more at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server?view=azuresql#sql-server-agent`](https://learn.microsoft.com/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server?view=azuresql#sql-server-agent). Just about everything else is the same to use Database Mail with Azure SQL Managed Instance except for these two differences as called out in the documentation:

*   sp_send_dbmail cannot send attachments using the @file_attachments parameter. Local file system and external shares or Azure Blob Storage are not accessible from this procedure.

*   The @query parameter in the sp_send_db_mail procedure doesn't work.

### Service Broker

Back in SQL Server 2005, we heard from our customers that they wanted to build service-oriented applications using asynchronous messaging techniques. We designed and built a system called Service Broker that uses the power of SQL Server tables, programming, and communication to implement a messaging system within the SQL Server engine. If you have never used Service Broker, you can get started at [`https://learn.microsoft.com/sql/database-engine/configure-windows/sql-server-service-broker`](https://learn.microsoft.com/sql/database-engine/configure-windows/sql-server-service-broker).

Service Broker is another example of an instance feature, and we do not support this capability with Azure SQL Database. CloudLifter to the rescue again. We support Service Broker applications with Azure SQL Managed Instance. One major exception is that we only support Service Broker between Managed Instances. See the full set of differences at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server?view=azuresql#service-broker`](https://learn.microsoft.com/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server?view=azuresql#service-broker).

### Full-Text Search

Full-text search has been an integrated component of the SQL Server engine for many releases. Full-text search provides searching capabilities through T-SQL against *text* data stored in the database. Even though full-text search is technically an instance-level feature for SQL Server, we support this for both Azure SQL Managed Instance and Databases. You can read about how to get started with full-text search at [`https://learn.microsoft.com/sql/relational-databases/search/full-text-search`](https://learn.microsoft.com/sql/relational-databases/search/full-text-search).

There are a few limitations with full-text when it comes to Azure SQL:

*   “Third-party filters” are not supported (examples are Office and .pdf filters).

*   You cannot manage aspects to the services that support full-text like fdhost.

*   Semantic search is not supported.

### Machine Learning Services

In SQL Server 2016, we introduced R Services for SQL Server. The concept is that we would use a new architecture to allow you to run R programs in a secure, isolated, and scalable manner *on the same computer* as SQL Server. In SQL Server 2017, we added Python support and rebranded this capability as **Machine Learning Services**.

We eventually enabled this capability in Azure SQL Managed Instance (it is not available in Azure SQL Database) with a few caveats:

*   You need to enable the feature using sp_configure with the “external scripts enabled” option.

*   Java was added as an extensibility language in SQL Server 2019 but is not available in Azure SQL Managed Instance.

*   There are R and Python packages installed, but if you need something different, you may need to use a tool called sqlmlutils. Read more at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/machine-learning-services-differences?view=azuresql#python-and-r-packages`](https://learn.microsoft.com/azure/azure-sql/managed-instance/machine-learning-services-differences?view=azuresql#python-and-r-packages).

*   You cannot use custom resource governance for the execution of R and Python.

Read the complete list of differences at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/machine-learning-services-differences`](https://learn.microsoft.com/azure/azure-sql/managed-instance/machine-learning-services-differences).

### Distributed Transactions (DTC)

DTC support was a new feature enabled for Azure SQL Managed Instance since the first edition of the release of this book. This feature was often seen as a blocker by customers to migrate from SQL Server, so the Managed Instance engineering team enabled this for the cloud.

It is important first to call out when to use the DTC feature for Managed Instance vs. supporting a distributed database transaction across Azure cloud services.

You can execute distributed transactions (with .Net libraries) across Azure SQL Database or across databases for Azure SQL Managed Instance with a concept called *native support* for distributed transactions. You cannot use this across Azure SQL Database and Azure SQL Managed Instance. Read more about native support at [`https://learn.microsoft.com/azure/azure-sql/database/elastic-transactions-overview`](https://learn.microsoft.com/azure/azure-sql/database/elastic-transactions-overview).

DTC support for Azure SQL Managed Instance provides the ability to run distributed transactions in *mixed* environments or Azure SQL Managed Instances (including with T-SQL). Mixed environment can be any system that supports distributed transaction protocols like XA, including on-premises SQL Server instances. In this scenario, Azure SQL Managed Instance manages all aspects of DTC like the DTC Service that is managed by Windows Server on-premises. For distributed transactions across Managed Instances, you will also want to take advantage of a security model called a *server trust group*. This provides authentication models across instances for scenarios like DTC, Service Broker, and linked servers using Microsoft Entra. Learn more about server trust groups at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/server-trust-group-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/server-trust-group-overview). There are network connectivity considerations to configure for DTC with Managed Instance. Learn more at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/distributed-transaction-coordinator-dtc`](https://learn.microsoft.com/azure/azure-sql/managed-instance/distributed-transaction-coordinator-dtc).

### xp_cmdshell

xp_cmdshell is a system-supported extended stored procedure (xproc) in SQL Server. This xproc runs as a DLL in the process space of SQL Server but creates a new process on the SQL Server computer to run the command you provide as a parameter. While a nifty feature of SQL Server for years, some administrators disable the use of this procedure because it is considered a security vulnerability. For that reason, we don’t support xp_cmdshell in Azure SQL.

There really isn’t anything equivalent for Azure SQL to run commands from a T-SQL procedure that is invoked by the engine. You would need to bring this code outside of your server-side programming logic into a script or application.

## Job Management

As part of managing a SQL Server environment, you will no doubt want to schedule *jobs* to execute various tasks related to your SQL Server. Let’s review what options you have to perform various aspects for job management with your Azure SQL deployment.

### SQL Server Agent

For years, users of SQL Server have used the built-in capabilities of SQL Server Agent. SQL Server Agent is a job scheduling system that is integrated with SQL Server. SQL Server Agent is another example of an instance-level feature we have not supported with Azure SQL Database but now support with Azure SQL Managed Instance.

Figure [9-3](#496204_2_En_9_Chapter.xhtml#Fig3) shows SSMS within my jumpbox VM I deployed in Chapter [4](#496204_2_En_4_Chapter.xhtml) connected to my Azure SQL Managed Instance.

![](images/496204_2_En_9_Chapter/496204_2_En_9_Fig3_HTML.jpg)

Figure 9-3

SQL Server Agent with Azure SQL Managed Instance

You will start using SQL Server Agent, and it will feel very much like job management with SQL Server. There are a few limitations, and we document these at [`https://learn.microsoft.com/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server?view=azuresql#sql-server-agent`](https://learn.microsoft.com/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server?view=azuresql#sql-server-agent).

Probably the biggest difference you will see is that we do not support CmdExec or PowerShell job step types. SQL Agent for Managed Instance is mostly used for T-SQL jobs, SSIS jobs, and jobs used to support Replication.

### Elastic Jobs

If Azure SQL Database is your preferred and best deployment option, you do have an alternative to schedule jobs that execute T-SQL statements using **Azure** **Elastic Jobs**. In the first edition of this book, Elastic Jobs was in preview and it stayed that way for a few years, leaving customers understandably wondering about its future. I was excited to see us refresh the preview and announce General Availability of Elastic Jobs in April 2024\. You can get started by reading at [`https://learn.microsoft.com/azure/azure-sql/database/elastic-jobs-overview`](https://learn.microsoft.com/azure/azure-sql/database/elastic-jobs-overview).

Some of the nice new features added since the original preview are as follows:

*   Microsoft Entra ID (formerly Azure Active Directory) support for central administration of authentication and permissions.

*   Service-managed private link support to securely connect to target databases.

*   Integration with Azure Alerts for job execution status notification.

*   Easily scale job agent’s tier to connect to hundreds of target databases concurrently across Azure.

*   Dynamic enumeration target databases in target servers and elastic pools.

*   Jobs can be composed of multiple steps to customize the execution sequence.

One of the cool features of Elastic Jobs is that I can run jobs in parallel, such as scheduling index maintenance across many Azure SQL Databases in parallel.

Elastic Jobs are essentially a *separate* Azure service in the form of a *job agent* that allows you to run T-SQL jobs against targets, which are Azure SQL Databases (including elastic pools). The Elastic Job service uses its own Azure SQL Database, known as the job database, to record scheduling information and job output.

Figure [9-4](#496204_2_En_9_Chapter.xhtml#Fig4) shows the overall architecture.

![](images/496204_2_En_9_Chapter/496204_2_En_9_Fig4_HTML.jpg)

Figure 9-4

The Elastic Jobs architecture

The Elastic Job agent is billed based on a service tier which defines a max concurrency for your jobs. In addition, standard billing applies to the compute and storage of the Azure SQL Database job database. Here are a few important points about Elastic Jobs:

*   Targets can be defined in groups of servers and/or databases.

*   You can use a very low-cost Azure SQL Database service tier for your job database, but the performance on the job agent could be impacted if you have scheduled a larger number of jobs concurrently against a large number of targets. There is no real metric to guide you here, so I recommend you start small or even use Serverless and monitor usage.

*   You can use private endpoints to connect the job agent with target Azure SQL Databases.

*   Elastic Jobs support dynamic enumeration. This is very cool. You could define a target at the logical server level. Every database in the server would now be part of the target for the job. The job agent would choose databases each time it runs, so as databases are removed and added, it can intelligently just pick any applicable database.

Use the tutorial at [`https://learn.microsoft.com/azure/azure-sql/database/elastic-jobs-tsql-create-manage`](https://learn.microsoft.com/azure/azure-sql/database/elastic-jobs-tsql-create-manage) to learn more about creating and using Elastic Jobs.

### Azure Automation

Azure automation is an Azure service that allows you to automate cloud management tasks which can include integration with Azure SQL. This includes executing tasks using languages like PowerShell with a concept called a *runbook*. Learn more about Azure automation at [`https://learn.microsoft.com/azure/automation/overview#process-automation`](https://learn.microsoft.com/azure/automation/overview#process-automation).

Check out our documentation for specific information on how to use Azure automation with Azure SQL Database at [`https://learn.microsoft.com/azure/azure-sql/database/automation-manage`](https://learn.microsoft.com/azure/azure-sql/database/automation-manage). Think of this as a more advanced way to automate tasks using PowerShell against Azure SQL Database than Elastic Jobs.

## Supporting Azure SQL

I have spent much of my early career at Microsoft in technical support, and many of the details of SQL Server I have compared in this book are based on knowledge I gained while in support.

There are a few topics regarding supporting your Azure SQL deployment that are different than SQL Server worth calling out in this chapter, including error handling, stack dumps, using resources in the Azure portal to assist in troubleshooting, using Copilot, and providing feedback with UserVoice.

### Handling Errors

Most errors that you can encounter with Azure SQL Managed Instance and Database will be common with SQL Server. Our documentation has a complete list of engine errors at [`https://learn.microsoft.com/sql/relational-databases/errors-events/database-engine-events-and-errors`](https://learn.microsoft.com/sql/relational-databases/errors-events/database-engine-events-and-errors).

For Azure SQL, you may encounter some new errors, and they usually center around *connectivity*, *resource governance*, and *support*. You can find a list of some of these scenarios in our documentation at [`https://learn.microsoft.com/azure/azure-sql/database/troubleshoot-common-errors-issues`](https://learn.microsoft.com/azure/azure-sql/database/troubleshoot-common-errors-issues).

For example, in Chapter [8](#496204_2_En_8_Chapter.xhtml) of the book, you saw a scenario when an application could not connect during a failover and encountered an error to retry a connection (using ostress.exe).

```
[SQL Server]Database '' on server '' is not currently available.  Please retry the connection later.  If the problem persists, contact customer support, and provide them the session tracing ID of '{CC39135B-D638-4A51-BB25-EABB8A5315A0}'.
```

This is **Msg 40613** and is an example of a connectivity error because a *valid* logical server is not available, but *your network connectivity is fine*. What this error effectively means is that your connectivity is fine to our gateways, but the resource we need to redirect you to (your logical server or instance) is not available.

Let’s say the problem is that a network error has occurred for your application. You would get a more traditional SQL Server error like

```
10053: A transport-level error has occurred when receiving results from the server. (Provider: TCP Provider, error: 0 - An established connection was aborted by the software in your host machine)
```

One unique error to Azure SQL due to a failed connection from not having a firewall rule configured correctly is

```
Error 40615:
Cannot open server '%.*ls' requested by the login. Client with IP address '%.*ls' is not allowed to access the server.  To enable access, use the Windows Azure Management Portal or run sp_set_firewall_rule on the master database to create a firewall rule for this IP address or address range.  It may take up to five minutes for this change to take effect.
```

Fortunately, there is good context here on the cause of the problem (lack of firewall rule setup for this client).

An example of a resource governance type of error is exceeding a resource limit as specified by your deployment option. For example, for Azure SQL Database if you exceeded the maximum number of workers, you might get an error like this:

```
Msg 10928
The request limit for the database is  and has been reached.
```

The limit you may encounter could be storage. The following is an example of an error when you have exceeded the maximum storage limit for a Managed Instance:

```
Msg 1133
The managed instance has reached its storage limit. The storage usage for the managed instance cannot exceed (%d) MBs.
```

Finally, you may encounter an error because you are trying to use a feature or a T-SQL statement that is not supported. For example, if you tried to change the “max server memory” option with sp_configure in Managed Instance, you would get an error like this:

```
Msg 5870
Changes to server configuration option max server memory (MB) are not supported in SQL Database Managed Instances
```

An error may not be so obvious as to the problem. If you tried to execute sp_configure when connected to Azure SQL Database, you would get an error like the following, indicating this system procedure is not supported for Azure SQL Database.

```
Msg 2812, Level 16, State 62, Line 1
Could not find stored procedure 'sp_configure'.
```

In some cases, you will encounter an error because you don’t have access to a feature due to your deployment option. For example, In-Memory OLTP is only available in Business Critical service tiers. Therefore, if you try to create a memory-optimized table in a General Purpose service tier, you will encounter the following error:

```
Msg 40536, Level 16, State 2, Line 1
'MEMORY_OPTIMIZED tables' is not supported in this service tier of the database. See Books Online for more details on feature support in different service tiers of Windows Azure SQL Database.
```

### Stack Dumps

In some cases, the SQL Server engine encounters a fatal error (e.g., ACCESS_VIOLATION) that results in a termination of the connection of the application and the creation of a *stack dump* on the server. If this type of problem is persistent, users of SQL Server are used to working with Microsoft technical support to examine these dump files to determine the cause of the problem. The following is a Microsoft article that describes an example of this type of problem: [`https://support.microsoft.com/help/4519796/fix-stack-dump-occurs-when-table-type-has-a-user-defined-constraint-in`](https://support.microsoft.com/help/4519796/fix-stack-dump-occurs-when-table-type-has-a-user-defined-constraint-in).

For Azure SQL, you don’t have access to the underlying file system to look at these files. The good news is that it doesn’t matter. Stack dumps are conditions that Azure SQL automatically handles. It is possible your application may encounter one of these types of problems, but our back-end systems have alerts to monitor these types of issues automatically. Our engineers get notified and an immediate investigation is initiated. If the problem is severe enough to require SQL Server to be restarted, we may initiate a failover. One of the benefits to this type of monitoring is that we may recognize a pattern to a problem like this that affects multiple users and initiate a code fix on the next *train* – another example of the benefits of a versionless SQL Server.

### Troubleshooting Resources in the Azure Portal

My former colleagues worked with our engineering team to build in tools to assist you in troubleshooting your own problems in the Azure portal.

I mentioned using the Resource Health option in the Service Menu to look for possible failovers that affected availability. Figure [9-5](#496204_2_En_9_Chapter.xhtml#Fig5) shows this screen for my Azure SQL Database with a reference to select a Troubleshooting tool (you can also access the Troubleshooting Tool from the **Support+Troubleshooting** option on the Services menu).

![](images/496204_2_En_9_Chapter/496204_2_En_9_Fig5_HTML.jpg)

Figure 9-5

Accessing the troubleshooting tool from Resource Health

If you select **Troubleshooting tool**, you will see a set of common problems based on the top issues Microsoft support sees from customer support requests like Figure [9-6](#496204_2_En_9_Chapter.xhtml#Fig6).

![](images/496204_2_En_9_Chapter/496204_2_En_9_Fig6_HTML.jpg)

Figure 9-6

Diagnosing and solving common problems for Azure SQL

I can either search for a common type of problem or choose a category and use a specific set of screens to troubleshoot an issue. Let’s say I can’t connect to my Azure SQL Database and am getting an error Msg 40613 I mentioned earlier in this section of the chapter.

I can select the Troubleshoot link from Connectivity: Troubleshoot DB Availability and Connection Error. I can choose my problem is Msg 40613 as a problem and scrolling down I can see more details to help me troubleshoot my specific problem like in Figure [9-7](#496204_2_En_9_Chapter.xhtml#Fig7).

![](images/496204_2_En_9_Chapter/496204_2_En_9_Fig7_HTML.jpg)

Figure 9-7

Troubleshooting connectivity error 40613

I can select a time frame for the problem and run diagnostics. The diagnostics may use built-in telemetry in Azure to help determine the cause. If this doesn’t help me, I can scroll down like in Figure [9-8](#496204_2_En_9_Chapter.xhtml#Fig8).

![](images/496204_2_En_9_Chapter/496204_2_En_9_Fig8_HTML.jpg)

Figure 9-8

More information to troubleshoot error 40613

This includes common solutions, web search results, and the ability to search community forums or even create new support requests. One huge advantage of using this method for a support request is the support engineers at Microsoft have the context of everything you have selected here regarding the problem, your database context, and diagnostic that have been run saving you time and effort.

Microsoft has various support plans to help you with your Azure deployments. All Azure customers for no charge get a fundamental set of support options. However, there are also options to pay for support to increase your level of availability for Microsoft Support to handle business-critical issues 24/7\. Learn about all the Azure support plans at [`https://azure.microsoft.com/support/plans/`](https://azure.microsoft.com/support/plans/).

What about **Copilot**? I mentioned this as a new tool to help you manage and configure your Azure SQL Database. Let’s say I can’t log in to Azure SQL Database, and I get error 40615, which indicates a firewall problem. Let’s say I don’t know the details of that message, and I find out that someone complains they cannot log in. I can use Copilot for a prompt like in Figure [9-9](#496204_2_En_9_Chapter.xhtml#Fig9) to help.

![](images/496204_2_En_9_Chapter/496204_2_En_9_Fig9_HTML.jpg)

Figure 9-9

Copilot helps with a login failure to Azure SQL Database

I like the context here. I know the time frame, the server, and the database, and I know it is a firewall issue. And I’m given prompts to help resolve the problem.

If I select “Can you help me add an IP to the firewall rules,” I get specific instructions on how to do this as you see in Figure [9-10](#496204_2_En_9_Chapter.xhtml#Fig10).

![](images/496204_2_En_9_Chapter/496204_2_En_9_Fig10_HTML.jpg)

Figure 9-10

Copilot solution to add a firewall rule

I still must find out what client is having the problem and set up the rule. If I had SQL Auditing turned on, I could easily get that information.

### UserVoice

You may be using Azure SQL and have a suggestion, not necessarily a problem that needs support. Microsoft provides a forum for your feedback for Azure SQL through a concept called *Azure Ideas*. You can access feedback for Azure SQL and SQL Server at [`https://aka.ms/sqlfeedback`](https://aka.ms/sqlfeedback).

I can assure you that as an engineering team, we do look at these requests and how customers vote on them, so find something you are passionate about and get others to vote for your idea.

## Azure SQL Best Practices

To wrap up this chapter, let’s take a look at resources that can help you with best practices for your Azure SQL deployment.

### Security Playbook

About a year ago, Jakub Szymaszek, a Principal Program Manager on our team who specializes in security (and one of the nicest people you will ever meet), approached me about an idea the team was working on. He said “Bob, you have worked with SQL Server for many years. Can you take a look at a *security playbook* we are working on?” I gave him some feedback and looked forward to how the project would progress.

The result of this effort has been infused into our documentation at [`https://learn.microsoft.com/azure/azure-sql/database/security-best-practice`](https://learn.microsoft.com/azure/azure-sql/database/security-best-practice). This document covers the collective knowledge of our team for best practices for you to secure Azure SQL. This includes authentication, access management, data protection, network security, monitoring and auditing, common security threads, and security aspects for availability. This is your “de facto” guide to read and learn as you deploy and secure your Azure SQL deployments. You will find many of the concepts you learned in Chapter [6](#496204_2_En_6_Chapter.xhtml) of the book line up with these best practices.

### Best Practices for Performance

In Chapter [7](#496204_2_En_7_Chapter.xhtml) of the book, you saw a comprehensive look at performance for Azure SQL. There are some best practices to follow as you saw in that chapter. We also have a very nice summary of performance best practices for you to read at [`https://learn.microsoft.com/azure/azure-sql/database/performance-guidance`](https://learn.microsoft.com/azure/azure-sql/database/performance-guidance). This includes guidance for monitoring, query design, and application development.

Another excellent resource comes from my longtime colleague Jack Li who now works in our engineering team. Learn more at [`https://learn.microsoft.com/azure/azure-sql/identify-query-performance-issues`](https://learn.microsoft.com/azure/azure-sql/identify-query-performance-issues).

### Azure Advisor

Given that in the Azure ecosystem there is a great deal of telemetry about your deployments (but not your data), why not build in some automation to give you advice? That is **Azure Advisor**. Azure Advisor is the collective knowledge of the Azure engineering team all up to give you advice about your deployments in the topics of cost, security, reliability, operational excellence, and performance. You can read more about Azure Advisor at [`https://learn.microsoft.com/azure/advisor/advisor-overview`](https://learn.microsoft.com/azure/advisor/advisor-overview). Figure [9-11](#496204_2_En_9_Chapter.xhtml#Fig11) shows an example of Azure Advisor for my subscription.

![](images/496204_2_En_9_Chapter/496204_2_En_9_Fig11_HTML.jpg)

Figure 9-11

Azure Advisor

Here is a nice blog post outlining what Azure Advisor can do for you: [`https://azure.microsoft.com/blog/your-single-source-for-azure-best-practices/`](https://azure.microsoft.com/blog/your-single-source-for-azure-best-practices/). You can see from my advisor overview, I have some work to go look at security!

Notice in Azure Advisor the area of *saving costs*. Our documentation calls out some of these cost saving ideas at [`https://learn.microsoft.com/azure/advisor/advisor-cost-recommendations`](https://learn.microsoft.com/azure/advisor/advisor-cost-recommendations). For me, savings costs are very important balanced with the needs of your application for security, performance, and availability. Keep these cost saving ideas in mind for Azure SQL:

*   Monitor your performance for Azure SQL Database and right-size your vCore and storage choices over time. You saw in Chapter [7](#496204_2_En_7_Chapter.xhtml) of the book how easy it is to scale databases.

*   Take a hard look at the Serverless compute tier. Serverless comes with auto-scaling, pay per second, and pause for compute when idle.

*   Use the start/stop feature of Azure SQL Managed Instance when there are periods where the instance is not being used.

*   Take advantage of using your existing licenses with Azure Hybrid Benefit.

*   Read my blog on how to maximize savings with SQL at [`https://www.microsoft.com/sql-server/blog/2024/02/15/how-sql-developers-can-maximize-savings`](https://www.microsoft.com/sql-server/blog/2024/02/15/how-sql-developers-can-maximize-savings).

*   If you only need a replica for disaster recovery, mark it as *passive* and saving on costs. This concept is called standby replicas in Azure SQL Database.

*   Shut down Azure Virtual Machines when you don’t need them.

*   Another way to analyze costs is to use the cost analysis option in the Azure portal. This provides a nice breakdown of costs and helps forecast future costs. Learn more at [`https://learn.microsoft.com/azure/cost-management-billing/costs/quick-acm-cost-analysis`](https://learn.microsoft.com/azure/cost-management-billing/costs/quick-acm-cost-analysis).

### Keep Up to Date with New Features

Our team creates resources for you to stay in touch with everything that is new about Azure SQL, including Azure SQL Managed Instance and Azure SQL Database for both preview and generally available features. I advise you bookmark these links:

[`https://aka.ms/whatsnewsqldb`](https://aka.ms/whatsnewsqldb)

[`https://aka.ms/whatsnewsqlmi`](https://aka.ms/whatsnewsqlmi)

### Stay in Touch with Our Team

I have some resources for you to use to keep track of various aspects of Azure SQL, gain more knowledge, and learn more best practices.

*   Follow the X handle @AzureSQL. This is the official handle for our Azure SQL engineering team. Also, we often use #azuresql to post interesting announcements, presentations, and facts about Azure SQL. My colleague Marisa Brasile does an amazing job of keeping the community and industry up to date with these resources. You can also follow my handle at @bobwardms.

*   I often post important news and announcements for Azure SQL on LinkedIn at [`linkedin.com/in/bobwardms`](https://www.linkedin.com/in/bobwardms?lipi=urn%253Ali%253Apage%253Ad_flagship3_profile_view_base_contact_details%3B6b7uErh1QDqWjl%252BELkGeMw%253D%253D).

*   There are several blogs our team engages in I think you will find helpful:

    Our **main SQL Server blog** at [`https://www.microsoft.com/sql-server/blog`](https://www.microsoft.com/sql-server/blog)

    The **Tech Community Blog** at [`https://techcommunity.microsoft.com/t5/azure-sql-database/bg-p/Azure-SQL-Database`](https://techcommunity.microsoft.com/t5/azure-sql-database/bg-p/Azure-SQL-Database)

    Our **Azure SQL Devs’ Corner** at [`https://devblogs.microsoft.com/azure-sql`](https://devblogs.microsoft.com/azure-sql)

*   My colleague Anna Hoffman and I built a series of training materials for self-paced study and hands-on learning. The content for these materials spans labs, videos, and open source. All of these are completely free! Check out these links:

    [**https://aka.ms/azuresqlfundamentals**](https://aka.ms/azuresqlfundamentals) – This is a self-paced course you can take on Microsoft Learn. The cool part of this course is that you don’t need an Azure subscription. A free sandbox is provided for you to try out Azure SQL!

    [**https://aka.ms/azuresql4beginners**](https://aka.ms/azuresql4beginners) – This is a series of short videos on YouTube (some 60+ videos) for you to learn Azure SQL at your own pace. The videos line up with the Azure SQL Fundamentals labs (but there is even more here).

    [**https://aka.ms/azuresqlworkshop**](https://aka.ms/azuresqlworkshop) – A great introductory workshop that we built to complement Azure SQL for Beginners and Azure SQL Fundamentals.

    [**https://aka.ms/cloudsqlworkshop**](https://aka.ms/cloudsqlworkshop) – This is a recent workshop I built to be used for an in-person day-long workshop on Azure SQL, including Azure VM, Managed Instance, and Azure SQL Database. It comes with self-paced labs and a full PowerPoint deck.

    Anna also hosts a video series called **Data Exposed** where members of the engineering team talk about various aspects of Azure SQL and SQL Server. Take a look at this series at [`https://aka.ms/dataexposed`](https://aka.ms/dataexposed).

## Summary

In this chapter, you extended your knowledge of Azure SQL by looking at features compared to SQL Server such as linked servers and Database Mail, understanding options for job management, exploring ways to support your deployments, and reviewing best practices for using Azure SQL.

So what else is there to learn about Azure SQL? If you recall, I mentioned in Chapters [1](#496204_2_En_1_Chapter.xhtml) and [2](#496204_2_En_2_Chapter.xhtml) new capabilities in Azure SQL that go beyond the traditional Relational Database Management System (RDBMS). Move on to the last chapter of the book to go *Beyond RDBMS*.

# 10. Beyond RDBMS

In the summer of 2023, I was trying to think of a way to describe some of the great capabilities we have in Azure SQL that is “in the box” but also features that are not normally in a traditional Relational Database Management System (RDBMS). The word “beyond” just came to me as a way to describe this. So, I started sketching down on paper (I’m old school) two columns:

*   What features are built inside the database that in many cases might require an add-on or other products? I listed those on the left side.

*   What capabilities are beyond what many think are part of a traditional RDBMS? Those were on the right side.

After sketching this out, I built a slide to describe all of this as you see in Figure [10-1](#496204_2_En_10_Chapter.xhtml#Fig1).

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig1_HTML.jpg)

Figure 10-1

Azure SQL is beyond RDBMS

This chapter describes in more detail what you see on this figure. Some of the features you see on the left side we have talked about in the book, but it will be a good summary to see each of these.

This chapter will contain examples for you to try out and use as you read along. For you to try out any of the techniques, commands, or examples I use in this chapter, you will need

*   An Azure subscription.

*   A minimum of Contributor role access to the Azure subscription. You can read more about Azure built-in roles at [`https://learn.microsoft.com/azure/role-based-access-control/built-in-roles`](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles).

*   Access to the Azure Portal.

*   In this chapter, I will use Azure SQL Database as many of these capabilities are enabled for that service, but many of them either are now or will be part of Azure SQL Managed Instance.

*   To connect to Azure SQL Database, I’m going to use the Azure VM I deployed in Chapter [3](#496204_2_En_3_Chapter.xhtml), called **bwsql2022**, and configured for a private endpoint in Chapter [6](#496204_2_En_6_Chapter.xhtml) (you could use another method as long as you can connect to the Azure SQL Database). You can use any method you like provided you can connect to an Azure SQL Database.

*   You will see some T-SQL in this chapter, so install a tool like SQL Server Management Studio (SSMS) at [`https://aka.ms/ssms`](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms%253Fview%253Dsql-server-ver15).

*   Some of the features I’ll show you here at the time of the writing of this book are in preview and not available publicly. But in the future, they will be so I’ve listed resources for you in the chapter to keep up to date with our progress, so you can access them.

## “It’s in the Box?”

The left side of Figure [10-1](#496204_2_En_10_Chapter.xhtml#Fig1) represents features that are built into the Azure SQL service and all of them integrated into the database engine and the T-SQL language.

Let’s go through each of the features on the left side of Figure [10-1](#496204_2_En_10_Chapter.xhtml#Fig1) with some examples.

### JSON

JSON is a very popular storage format for data for all types of applications. In the past, we have shipped a series of T-SQL functions to process and view JSON formatted data, including the following capabilities:

*   Parse JSON text and read or modify values.

*   Transform arrays of JSON objects into table format.

*   Run any Transact-SQL query on the converted JSON objects.

*   Format the results of Transact-SQL queries in JSON format.

As an example, you could use T-SQL to extract JSON like the following example:

```
DECLARE @jsonInfo NVARCHAR(MAX)
DECLARE @town NVARCHAR(32)
SET @jsonInfo=N'{"info":{"address":[{"town":"Paris"},{"town":"London"}]}}';
SET @town=JSON_VALUE(@jsonInfo,'$.info.address[0].town'); -- Paris
SET @town=JSON_VALUE(@jsonInfo,'$.info.address[1].town'); -- London
```

In these examples, the data type used is a character data type from a local variable, but it could have been a character-based column. We have a wide range of T-SQL functions to use with JSON data, which you can see the full list at [`https://aka.ms/jsonsqlfunctions`](https://aka.ms/jsonsqlfunctions).

What is new is a *native* JSON data type, which is in preview at the time of the writing of this book, now allows you to use the same set of JSON T-SQL functions but use a type that is more geared toward JSON documents.

The native JSON type, called **json**, is more efficient than using a character column because it provides

*   More efficient reads, as the document is already parsed

*   More efficient writes, as the query can update individual values without accessing the entire document

*   More efficient storage, optimized for compression

*   No change in compatibility with existing code

The JSON type internally stores data using UTF-8 encoding, Latin1_General_100_BIN2_UTF8\. This behavior matches the JSON specification.

An example used in a table looks like this. Let’s say you were storing general information about orders that are captured from your application in JSON format. Originally, your table definition might look like this (credits for this example from my colleague Umachandar Jayachandran (we call him UC)):

```
CREATE TABLE dbo.Orders(
order_id int NOT NULL IDENTITY,
order_info nvarchar(max) NOT NULL
)
```

Now it can look like this:

```
CREATE TABLE dbo.Orders(
order_id int NOT NULL IDENTITY,
order_info json NOT NULL
)
```

Now any JSON document describing the log can be stored natively right into the table like the following:

```
INSERT INTO dbo.Orders(order_info)
VALUES
(
'{
"OrderNumber": "S043659",
"Date": "2024-05-24T08:01:00",
"AccountNumber": "AW29825",
"Price": 59.99,
"Quantity": 1
}'
),
(
'{
"OrderNumber": "S043661",
"Date": "2024-05-20T12:20:00",
"AccountNumber": "AW7365",
"Price": 24.99,
"Quantity": 3
}'
)
```

If you query this data, you will see a result like the one in Figure [10-2](#496204_2_En_10_Chapter.xhtml#Fig2).

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig2_HTML.jpg)

Figure 10-2

Results from using the new JSON data type

Now you can use JSON functions to extract fields out of the JSON format in a relational result like the following query:

```
SELECT o.order_id, JSON_VALUE(o.order_info, '$.AccountNumber') AS account_number
FROM dbo.Orders o;
```

In addition to helping with the construction of JSON documents, we have two new T-SQL functions.

#### JSON_ARRAYAGG

Generate a JSON array based on the aggregation of values. So basically, take a number of rows of data and produce a single JSON array of values. For example, this query based on the above data:

```
SELECT JSON_ARRAYAGG(JSON_ARRAY(o.order_info))
FROM dbo.Orders o;
```

produces a single array like the following result:

```
[[{"OrderNumber":"S043659","Date":"2024-05-24T08:01:00","AccountNumber":"AW29825","Price":59.99,"Quantity":1}],[{"OrderNumber":"S043661","Date":"2024-05-20T12:20:00","AccountNumber":"AW7365","Price":24.99,"Quantity":3}]]
```

#### JSON_OBJECTAGG

This function is similar except it is very valuable to produce a JSON document of objects as key/value pairs. For example, this query based on the above data:

```
SELECT JSON_OBJECTAGG(o.order_id:JSON_OBJECT('AccountNumber':o.order_info))
FROM dbo.Orders o;
```

produces key/value pairs like the following:

```
{"1":{"AccountNumber":{"OrderNumber":"S043659","Date":"2024-05-24T08:01:00","AccountNumber":"AW29825","Price":59.99,"Quantity":1}},"2":{"AccountNumber":{"OrderNumber":"S043661","Date":"2024-05-20T12:20:00","AccountNumber":"AW7365","Price":24.99,"Quantity":3}}}
```

Try it out for yourself and learn more at [`https://aka.ms/jsonsql`](https://aka.ms/jsonsql).

### RegEx

SQL Server comes with T-SQL operators such as LIKE and a series of T-SQL “string” functions. But for years, our community has asked us to provide built into the T-SQL language Regular Expressions or RegEx.

Note

For a primer on RegEx, I suggest you read this documentation article: [`https://learn.microsoft.com/dotnet/standard/base-types/regular-expression-language-quick-reference`](https://learn.microsoft.com/dotnet/standard/base-types/regular-expression-language-quick-reference).

I was excited to see us announce this capability in the spring of 2024\. It is still under preview as of the writing of this book, so there are no documentation references except for the announcement we made in the following blog about this new functionality at [`https://devblogs.microsoft.com/azure-sql/introducing-regular-expression-regex-support-in-azure-sql-db`](https://devblogs.microsoft.com/azure-sql/introducing-regular-expression-regex-support-in-azure-sql-db).

Note

These examples only work if you are involved in the preview program at the time of the writing of this book.

Here is a good example you can see from the blog:

```
-- Create Employees table with some records and check constraints for Email and Phone_Number columns
DROP TABLE IF EXISTS Employees
CREATE TABLE Employees (
ID INT IDENTITY(101,1),
[Name] VARCHAR(150),
Email VARCHAR(320)
CHECK (REGEXP_LIKE(Email, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')),
Phone_Number VARCHAR(20)
CHECK (REGEXP_LIKE (Phone_Number, '^(\d{3})-(\d{3})-(\d{4})$'))
);
-- Insert some sample data
INSERT INTO Employees ([Name], Email, Phone_Number) VALUES
('John Doe', 'john@contoso.com', '123-456-7890'),
('Alice Smith', 'alice@fabrikam.com', '234-567-8901'),
('Bob Johnson', 'bob@fabrikam.net','345-678-9012'),
('Eve Jones', 'eve@contoso.com', '456-789-0123'),
('Charlie Brown', 'charlie@contoso.co.in', '567-890-1234');
-- find all the employees whose email addresses end with .com
SELECT [Name], Email
FROM Employees
WHERE REGEXP_LIKE(Email, '\.com$');
```

The **REGEXP_LIKE**() can replace the traditional LIKE operator to provide a richer pattern matching experience. The result from this query is the one from Figure [10-3](#496204_2_En_10_Chapter.xhtml#Fig3).

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig3_HTML.jpg)

Figure 10-3

Results from a Regular Expression search

Notice also in the CREATE TABLE statement the use of REGEXP_LIKE to enforce the format of a phone number and email.

### Spatial and Graph

**Spatial data types** have been in the SQL Server for many releases. But I often find many users don’t know its existence. Spatial data types are based internally on SQLCLR and support the following types of data:

*   The **geometry** type represents data in a Euclidean (flat) coordinate system.

*   The **geography** type represents data in a round-earth coordinate system.

So instead of having to use another product to integrate with SQL, these types have spatial intelligence and are built into the engine and T-SQL language.

These types are *objects*, which means that they contain values and methods to use against the values. So, for example, the geography type has methods like **STDistance** which can be used to find the shortest distance between a point in a **geography** instance and a point in another **geography** instance.

Learn more about all the details of spatial data types for SQL at [`https://aka.ms/sqlspatial`](https://aka.ms/sqlspatial).

Not every data fits nicely into a relational set of tables. In fact, one data pattern is a **graph**. Consider a network of data with people that like restaurants in certain cities. One easy way to represent this data and *traverse* the network is with a graph like in Figure [10-4](#496204_2_En_10_Chapter.xhtml#Fig4).

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig4_HTML.jpg)

Figure 10-4

A network of data represented as a graph

A graph database is a special purpose database to use data represented in this fashion. For Azure SQL, we have built graph capabilities into the engine and the T-SQL language.

T-SQL has been extended to define tables as **nodes** and **edges**. In Figure [10-4](#496204_2_En_10_Chapter.xhtml#Fig4), Person, City, and Restaurant are node tables. LivesIn, Likes, LocatedIn, and Friends are edge tables. With this data stored, you can now use a new syntax for searching using the **MATCH** keyword like the following example:

```
-- Find Restaurants that John likes
SELECT Restaurant.name
FROM Person, likes, Restaurant
WHERE MATCH (Person-(likes)->Restaurant)
AND Person.name = 'John';
```

Learn how to get started with graph data at [`https://aka.ms/sqlgraph`](https://aka.ms/sqlgraph).

### Columnstore and Memory Optimized

I mentioned **columnstore indexes** in Chapter [7](#496204_2_En_7_Chapter.xhtml) of the book, but it is absolutely worth repeating because I see this technology so underused. Columnstore indexes are nothing short of amazing. I continue to see customers who just do not take advantage of this capability. Columnstore index can accelerate read query performance by 100x for the right workload. Columnstore indexes are supported in every deployment option you choose with Azure SQL. One myth about columnstore is that it is *only* an in-memory technology. The truth is that columnstore indexes perform best when they fit in memory and use compression, so more will fit in your memory limits. However, a columnstore index does not have to all fit in memory. One proof point for you on how important this technology is. We use columnstore indexes in all of our TPC-H benchmarks ([`https://www.tpc.org/tpch/results/tpch_perf_results5.asp?resulttype=noncluster&version=3`](https://www.tpc.org/tpch/results/tpch_perf_results5.asp%253Fresulttype%253Dnoncluster%2526version%253D3)). To get started with columnstore indexes, see the documentation at [`https://aka.ms/sqlcolumnstore`](https://aka.ms/sqlcolumnstore).

I also mentioned memory-optimized capabilities with **In-Memory OLTP** in Chapter [7](#496204_2_En_7_Chapter.xhtml). This is also worth repeating since there are many use cases for this technology, and it is built into the database engine for Azure SQL. In-Memory OLTP is surfaced through **memory-optimized tables**. Memory-optimized tables are only available in Business Critical service tiers. However, Hyperscale for Azure SQL Database does allow memory-optimized table variables which can be useful for stored procedures. Learn more at [`https://aka.ms/sqlmemoryoptimized`](https://aka.ms/sqlmemoryoptimized).

### Ledger

I covered the topic of Ledger, officially known as SQL Ledger, in Chapter [6](#496204_2_En_6_Chapter.xhtml) of the book. SQL Ledger allows you to create a table using T-SQL with an additional syntax WITH LEDGER = ON, and now automatically we will track all changes to the table using the technology of temporal tables. We will build a history table of your changes and a view that allows you to see the current table along with the historical changes. This is different than temporal tables because we also include a database ledger and a digest providing you auditing of who made the changes and methods to verify the data has been tampered with through crypto hashes. Learn more at [`https://aka.ms/sqlledger`](https://aka.ms/sqlledger).

### Query Intelligence

Query intelligence encompasses features built into the database engine to make your applications faster called Intelligent Query Processing (IQP) and capabilities that *use* the engine like Automatic Tuning.

I covered this topic extensively in Chapter [7](#496204_2_En_7_Chapter.xhtml) of the book. Learn more about IQP at [`https://aka.ms/iqp`](https://aka.ms/iqp). Learn more about Automatic Tuning at [`https://aka.ms/sqlautotuning`](https://aka.ms/sqlautotuning).

One more topic not covered in the book so far was an enhancement since the first edition of this book called optimized locking. Azure SQL Database now has an improved lock management system called **optimizing locking**. Using features like Accelerated Database Recovery (versioning) and Read Committed Snapshot Isolation (RCSI), optimized locking reduces the number of locks required for large transactions reducing lock memory and lock escalations. Any developer that struggles with lock escalation will appreciate what is now built into the engine to eliminate them. Read more at [`https://learn.microsoft.com/sql/relational-databases/performance/optimized-locking`](https://learn.microsoft.com/sql/relational-databases/performance/optimized-locking).

## REST API Integration

Connecting your data in SQL *inside* the engine has many advantages:

*   You can use the security of the engine with all the standard security principles and access you know.

*   Like stored procedures, you are reducing application traffic and use a server-side programming model to make your data integration more efficient.

I feel like Polybase has that similar type of advantage. With Azure SQL, we have built a new feature to interface with any REST API endpoint. Codename Project Solaria, we now have a system stored procedure, called **sp_invoke_external_rest_endpoint**, to send a *payload* to a REST API endpoint and receive back a response (output). Here is a diagram in Figure [10-5](#496204_2_En_10_Chapter.xhtml#Fig5) my colleague Davide Mauri built in 2022 when we first released the public preview of this feature.

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig5_HTML.jpg)

Figure 10-5

The original Solaria project

The solution is generic in the sense that you can send JSON, XML, or TEXT (depending on the protocol of the endpoint) to any REST API for a specific purpose. The REST API can process the input and optionally respond back with an answer (or response/output) which is in the form of character data. Based on the protocol or the REST API endpoint, the character data may be JSON, XML, or just TEXT.

Here is a code fragment to retrieve data from a table in the database and send the data as a message to Azure Event Hub (which can then process the message to perform some type of action):

```
DECLARE @Id UNIQUEIDENTIFIER = NEWID();
DECLARE @payload NVARCHAR(MAX) = (
SELECT *
FROM (
VALUES (@Id, 'John', 'Doe')
) AS UserTable(UserId, FirstName, LastName)
FOR JSON AUTO,
WITHOUT_ARRAY_WRAPPER
)
DECLARE @url NVARCHAR(4000) = 'https://.servicebus.windows.net/from-sql/messages';
DECLARE @headers NVARCHAR(4000) = N'{"BrokerProperties": "' + STRING_ESCAPE('{"PartitionKey": "' + CAST(@Id AS NVARCHAR(36)) + '"}', 'json') + '"}'
DECLARE @ret INT, @response NVARCHAR(MAX);
EXEC @ret = sp_invoke_external_rest_endpoint @url = @url,
@headers = @headers,
@credential = [https://.servicebus.windows.net],
@payload = @payload,
@response = @response OUTPUT;
SELECT @ret AS ReturnCode, @response AS Response;
```

The input is optional as some REST API endpoints can be used to “get” data. Here is an example of retrieving data in an Azure Storage file:

```
DECLARE @ret INT, @response NVARCHAR(MAX);
EXEC @ret = sp_invoke_external_rest_endpoint
@url = N'https://blobby.blob.core.windows.net/datafiles/my_favorite_blobs.txt?sp=r&st=2023-07-28T19:56:07Z&se=2023-07-29T03:56:07Z&spr=https&sv=2022-11-02&sr=b&sig=XXXXXX1234XXXXXX6789XXXXX',
@headers = N'{"Accept":"application/xml"}',
@method = 'GET',
@response = @response OUTPUT;
SELECT @ret AS ReturnCode, @response AS Response;
```

We support a known set of endpoints, but the list is vast at [`https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql?view=azuresqldb-current&tabs=request-headers#allowed-endpoints`](https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql%253Fview%253Dazuresqldb-current%2526tabs%253Drequest-headers%2523allowed-endpoints).

When we first built this feature, we were not as focused on GenAI applications as we are today. However, you can imagine now a huge focus is on AI. Fortunately, many *AI hosting engines or* *inference endpoints* support REST. You will see later in this chapter in the section called “SQL and AI” how you can use this feature to build any AI application *inside* Azure SQL.

At the time of the writing of this book, sp_invoke_external_rest_endpoint is generally available for Azure SQL Database and is in preview for Azure SQL Managed Instance. My colleague Brian Spendolini is the lead PM for this feature, so follow his blogs to see more examples at [`https://devblogs.microsoft.com/azure-sql/author/bspendolini`](https://devblogs.microsoft.com/azure-sql/author/bspendolini).

## Data API Builder and GraphQL

Any developer that must build code to interact with a database knows that no matter what programming language you use, there is some overhead to connecting and querying a database. Using a database is usually all about getting *authenticated* and then running *queries* (or stored procedures). The code required to make all of that happen is sometimes referred to as *plumbing* code.

What if there was a simpler way to build an app and reduce or even eliminate the plumbing code? Along comes **Data API Builder** (DAB). DAB is cross-platform, open source, and independent of language, technology, and frameworks. It requires zero code and a single configuration file. Best of all, it's free (you can view the open source GitHub repo at [`https://github.com/Azure/data-api-builder`](https://github.com/Azure/data-api-builder)).

The best way to show how this works is with a diagram, and the documentation provides a very nice one in Figure [10-6](#496204_2_En_10_Chapter.xhtml#Fig6).

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig6_HTML.jpg)

Figure 10-6

DATA API Builder (DAB) architecture

While DAB can work with other databases, I’ll focus on how it works with Azure SQL.

DAB is a software runtime (either running as its own service or within an application like Azure Static Web Apps) that effectively translates your configuration of a connection and schema into endpoints that support GraphQL or REST. You may have heard of REST, but if you are not familiar with GraphQL, you can learn the basics at [`https://graphql.org`](https://graphql.org). You also have the ability to run DAB in a container.

As an application developer, you can now compose REST or GraphQL formatted text or commands which DAB will turn into the actual calls using a driver to the back-end database to connect and execute queries.

I like this example in the documentation to show some basics: [`https://learn.microsoft.com/azure/data-api-builder/database-objects#stored-procedures`](https://learn.microsoft.com/azure/data-api-builder/database-objects%2523stored-procedures).

In the example, there is a database with tables for books and authors (the full schema can be found at [`https://github.com/Azure/data-api-builder/tree/main/samples/getting-started/azure-sql-db`](https://github.com/Azure/data-api-builder/tree/main/samples/getting-started/azure-sql-db)). Let’s say you have a stored procedure that selects books written by an author. With DAB and REST, you can compose a string like the following:

```
GET http:///api/GetCowrittenBooksByAuthor?author=isaac%20asimov
```

This will call the stored procedure with a parameter of “Isaac Asimov” and return results to the calling application through the endpoint.

I consider DAB a game changer for developers. I believe it can save you time and improve the quality of your application because it is just less lines of code for you to write and maintain. Start your journey at [`https://aka.ms/dab`](https://aka.ms/dab). Also, watch this Data Exposed episode with my colleague Jerry Nixon, the lead PM for this feature, at [`https://learn.microsoft.com/shows/data-exposed/data-api-builder-is-now-generally-available-data-exposed`](https://learn.microsoft.com/shows/data-exposed/data-api-builder-is-now-generally-available-data-exposed).

## DevOps, GitHub, and Containers

Developers need to have the tools and capabilities to be efficient for data-driven applications. Azure SQL has three features that support developers in these areas.

### Database Projects

Developers need to be able to build database projects, create new databases, new data-tier applications (DACs), and update existing databases and data-tier applications. These projects use tools like sqlpackage.exe, SQL Server Data Tools in Visual Studio (SSDT), DACPAC, and BACPAC files. We have had these technologies for some time to assist developers to build out an application that includes database objects and data.

One advancement in this area we have created is **SDK-style SQL projects**. SDK-style SQL projects are especially advantageous for applications shipped through pipelines or built in cross-platform environments.

My colleague Drew Skwiers-Koballa explains SDK-style SQL project differences in this excellent blog post at [`https://techcommunity.microsoft.com/t5/azure-sql-blog/microsoft-build-sql-the-next-frontier-of-sql-projects/ba-p/3290628`](https://techcommunity.microsoft.com/t5/azure-sql-blog/microsoft-build-sql-the-next-frontier-of-sql-projects/ba-p/3290628). In summary, the advantages are

*   The “globbing” (wildcard) file selection for the project to choose .sql files. This greatly reduces the size of the definition of the project.

*   The project can use **dotnet build** which provides class-platform compatibility and a simpler experience, including in a CI/CD pipeline.

*   A seamless integration with GitHub Actions for DevOps scenarios.

The initial release of SDK-style projects comes with Visual Studio Code, but I believe for the future you will see this incorporated into Visual Studio.

### GitHub Actions

Any new DevOps project must have an efficient method through a CI/CD pipeline for code to be developed, tested, and released into production.

We support today this process with Azure DevOps and Azure Pipelines, which you can read about at [`https://learn.microsoft.com/azure/devops/pipelines/targets/azure-sqldb`](https://learn.microsoft.com/azure/devops/pipelines/targets/azure-sqldb).

GitHub has become such a standard for development that we felt we needed to support the same type of DevOps capability. Therefore, we built a feature that integrates with GitHub actions ([`https://docs.github.com/actions`](https://docs.github.com/actions)) called **sql-action**.

sql-action allows you to declare a set of steps through a YAML file (.yml) to build and test your database project using SDK-style SQL projects, containers, and sqlcmd to help run tests.

Note

sql-action uses go-sqlcmd for sqlcmd capabilities. go-sqlcmd (which is called sqlcmd – I know confusing) is a new version of sqlcmd built on the Go language and is completely open source. See the full story at [`https://github.com/microsoft/go-sqlcmd`](https://github.com/microsoft/go-sqlcmd).

I wanted to test out what is possible here so built an example you can try yourself at [`https://github.com/microsoft/bobsql/tree/master/demos/devops_sqlcontainers/githubactions`](https://github.com/microsoft/bobsql/tree/master/demos/devops_sqlcontainers/githubactions).

My concept here was to use GitHub actions to create the database and all the database objects in Azure SQL Database. But I used GitHub actions to first test this in a SQL Server 2002 container. And here is the amazing part: GitHub supports a concept called a *runner*. This allows me to test out my code and my SQL Database in a VM or container, for free! (If you need larger sizes, you can then upgrade to a paid version.)

In my scenario, I wanted my GitHub action to run a “smoke test” to ensure the performance of a specific query was acceptable and did not encounter any warnings, for example, the detection of an anti-pattern query.

Any change to database objects like a stored procedure is automatically detected when a commit is done to the GitHub repo. The action automatically kicks in and runs my “smoke test.” The action fails if it detects an issue and the commit fails. Drew has created a great set of documents on this at [`https://github.com/Azure/sql-action`](https://github.com/Azure/sql-action).

### Containers

Containers are simply one of the best innovations I’ve seen over the last several years for developers. SQL Server is fully supported as a container, and I covered this topic extensively in *Pro SQL Server Linux*, *SQL Server 2019 Revealed*, and *SQL Server 2022 Revealed* books. Containers just make it so easy to package a *solution*, including solutions that integrate with SQL Server.

For Azure SQL, we have recently released a concept called a **Dev Container templates for Azure SQL Database**, a project led by my colleague Carlos Robles. One of the major benefits I talked about SQL containers in the books listed above is *consistency*. I can deploy a SQL container with a consistent version and even include specific database scripts or whatever tools I want so that the environment is consistent to my requirements cross-platform (Linux, MacOS, Windows) or multiclouds.

Dev Container templates extend that concept to ensure I have a complete consistent development environment, including SQL Server, developer frameworks, and even developer tools that allow for SDK-style SQL projects. In addition, they provide a seamless integration with our GitHub sql-action.

Take a look at this blog by Carlos to introduce the topic at [`https://devblogs.microsoft.com/azure-sql/azure-sql-dev-containers`](https://devblogs.microsoft.com/azure-sql/azure-sql-dev-containers).

Note

When I last talked to Carlos, one of the ideas we are considering is a SQL container that *behaves* like Azure SQL Database. In other words, the container would be a database engine that is versionless and matches the capabilities and limits of an Azure SQL Database giving you a great compatibility story with the cloud.

## SQL and AI

I’ve told you the story in Chapter [7](#496204_2_En_7_Chapter.xhtml) of the book of how we designed and built the Microsoft Copilot skills in Azure SQL Database. At this same time in the summer of 2023, our team knew we needed to make investments in Azure SQL to support Generative AI (GenAI) applications. Our Copilot is an AI assistant and good example of a GenAI application. In fact, this application uses a Retrieval Augmented Generation (RAG) pattern. A RAG pattern is when an application *augments* a prompt to a language model.

We also knew that a common technique for GenAI applications was to use the concept of **embeddings** to execute the concept of a **vector search** (or sometimes called a similarity search). Embeddings are vectors generated by an AI model and are understood by AI models. They are extremely powerful to perform *smarter* searching on your data. Most AI models that are used for this purpose are known as language models in the form of size: Large Language Models (LLM) down to Small Language Models (SLM).

Figure [10-7](#496204_2_En_10_Chapter.xhtml#Fig7) is a diagram I often show how to put this story together.

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig7_HTML.jpg)

Figure 10-7

GenAI application fundamentals with databases

There are two main concepts to look at with this diagram:

*   You can build RAG pattern applications by using standard SQL searching techniques to get data context and augment a prompt with those data results to send to a language model.

*   Optionally, you can use a language model to generate embeddings based on text data in your database and store them as a vector in a column in a table. This makes SQL now a *vector store* supporting *native vectors*. Then you can take a prompt from an application to generate an embedding from that text. You can then use a vector search to compare the embedding from the prompt with embeddings in your database. Some models are specifically designed for this purpose and are called embedding models (many general-purpose language models can also generate embeddings, but embeddings models are generally more precise). Using SQL for this approach has advantages since you can combine your vector search with standard SQL searching techniques. I call this a **hybrid search**.

You could take this concept a step further and take the results from the hybrid search and augment a prompt to a language model to create an even more precise RAG pattern.

Let’s now look at specific ways you can build GenAI applications with SQL using these techniques. In addition, we will look at a different way to use a Copilot experience to generate SQL queries.

### GenAI and SQL

To get started with GenAI and SQL, let’s look at a simple example to interact with an AI model inside the SQL engine. This example uses the REST API support I’ve describe earlier in the chapter and comes from a blog post from my colleague Brian Spendolini at [`https://devblogs.microsoft.com/azure-sql/using-openai-rest-endpoints-with-azure-sql-database`](https://devblogs.microsoft.com/azure-sql/using-openai-rest-endpoints-with-azure-sql-database):

```
-- Declare a variable to store the API endpoint
declare @url nvarchar(4000)
= N'https://skynetbeta.openai.azure.com/openai/deployments/chattykathy/chat/completions?api-version=2023-07-01-preview';
declare @headers nvarchar(102) = N'{"api-key":"1001001sos1001001indistress"}'
-- Declare a variable to store the payload (request body) for the POST request.
declare @payload nvarchar(max) = N'{"messages":[[{"role":"system","content":"You are an AI assistant that helps people find information."},
{"role":"system","content":"Why is the sky blue?"}]}'
-- Declare variables for the return status of the procedure and --- the API response content
declare @ret int, @response nvarchar(max);
exec @ret = sp_invoke_external_rest_endpoint
@url = @url,
@method = 'POST',
@headers = @headers,
@payload = @payload,
@timeout = 230,
@response = @response output;
-- Display the return code of the procedure and the response from -- the API
select @ret as ReturnCode, @response as Response;
```

In this example, the URL for the AI model is fictional. You will see an example later in this section of an OpenAI model in Azure. One key point here is that you can send *prompts* to any AI model through REST endpoints. In this case, we are just asking a question to a ***language model***. Notice the input or payload includes a system message to provide instructions on behavior and a question.

Here is a fragment of the response which is in JSON format:

```
"choices": [
{
"index": 0,
"finish_reason": "stop",
"message": {
"role": "assistant",
"content": "The sky appears blue because when the sun's light enters Earth's atmosphere,
it is scattered by the gases and particles in the air. This scattering causes the shorter
blue wavelengths of light to be dispersed more than the other colors in the spectrum.
This is known as Rayleigh scattering. As a result, when we look up at the sky during
the day, we see it as blue."
},
```

This example probably doesn’t represent how you would use SQL to interact with a language model, so let’s look at three types of methods you can use for GenAI applications using Azure SQL, which you can see described in Figure [10-8](#496204_2_En_10_Chapter.xhtml#Fig8).

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig8_HTML.jpg)

Figure 10-8

AI scenarios for Azure SQL

Let’s look more at the scenarios in the figure and other concepts.

#### Use AI Services

This first scenario is common to build a “chat with your data” experience where the components live *outside* of SQL. Figure [10-9](#496204_2_En_10_Chapter.xhtml#Fig9) shows a visual of how these components work together.

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig9_HTML.jpg)

Figure 10-9

Using Azure AI Services with your data

Azure OpenAI Services are well integrated with Azure AI Search indexes. In this scenario, you can build an Azure AI search index based on an Azure SQL Database table. When you build the index, you can apply *skills* to the index. One of those skills is to generate embeddings in the index based on a column in your table. You can apply other skills, including content moderation, to strip out any Personally Identifiable Information (PII).

With this index built, you can now build a GenAI application using a vector search against the index with embeddings or a RAG pattern. A simple way to do this is with Azure AI Studio. This tool allows you to test your various types of prompts which are converted into vector and/or hybrid searches against the index. You can use the *chat playground* as you can see in Figure [10-10](#496204_2_En_10_Chapter.xhtml#Fig10).

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig10_HTML.jpg)

Figure 10-10

An example of a chat playground to test an Azure Search Index

Notice the prompt is an imprecise question using words like “tight budget” and “appealing.” There is a place where language models shine as they understand these words and can use embeddings to find the right results and format them in a very consumable fashion. You can even take your test in the playground and build a simple application or your own *Copilot* on your data.

Notice also in Figure [10-10](#496204_2_En_10_Chapter.xhtml#Fig10) that Azure AI Search indexes can be automatically updated from the underlying SQL data using features like change data capture (CDC) which is supported in Azure SQL.

See the end-to-end steps on how to try this yourself at [`https://aka.ms/sqlaisamples`](https://aka.ms/sqlaisamples).

#### Use T-SQL for Hybrid Search

Using the concepts I described in Figure [10-7](#496204_2_En_10_Chapter.xhtml#Fig7) and earlier in the chapter, you can perform a hybrid search *inside* SQL using new capabilities in T-SQL, including using REST API and new T-SQL functions to process embeddings.

My colleagues Pooja Kamath and Davide Mauri wrote a blog post announcing the private preview of **native vector support** which you can read at [`https://devblogs.microsoft.com/azure-sql/announcing-eap-native-vector-support-in-azure-sql-database`](https://devblogs.microsoft.com/azure-sql/announcing-eap-native-vector-support-in-azure-sql-database).

In this example, the concept is to

*   Use REST API to Azure OpenAI with an embedding model to generate embeddings based on text in the database (customer reviews of food). The embeddings are returned in JSON format.

*   Use new T-SQL support to convert the embeddings in JSON to a binary format and store this in a varbinary column.

*   Take a prompt from the application and use REST APIs to get an embedding for the prompt and convert this to binary.

*   Do a vector search comparison of the prompt and the reviews in the database using a concept called *vector distance* like the following:

    ```
    --Assuming you have a stored procedure to get embeddings for a given text
    DECLARE @e VARBINARY(8000);
    EXEC dbo.GET_EMBEDDINGS @model = '', @text = 'healthy options instead of coke', @embedding = @e OUTPUT;
    SELECT TOP(10) ProductId,
    Summary,
    Text,
    VECTOR_DISTANCE('cosine',@e, VectorBinary) AS Distance
    FROM dbo.FineFoodReviews
    ORDER BY Distance;
    ```

The lower the distance number, the more *similar* the search. In this example, we built a SQL procedure GET_EMBEDDINGS that behind the scenes use the REST API and language models like the following:

```
CREATE PROCEDURE [dbo].[GET_EMBEDDINGS]
(
@model VARCHAR(MAX),
@text NVARCHAR(MAX),
@embedding VARBINARY(8000) OUTPUT
)
AS
BEGIN
DECLARE @retval INT, @response NVARCHAR(MAX);
DECLARE @url VARCHAR(MAX);
DECLARE @payload NVARCHAR(MAX) = JSON_OBJECT('input': @text);
-- Set the @url variable with proper concatenation before the EXEC statement
SET @url = 'https://.openai.azure.com/openai/deployments/' + @model + '/embeddings?api-version=2023-03-15-preview';
EXEC dbo.sp_invoke_external_rest_endpoint
@url = @url,
@method = 'POST',
@payload = @payload,
@headers = '{"Content-Type":"application/json", "api-key":""}',
@response = @response OUTPUT;
-- Use JSON_QUERY to extract the embedding array directly
DECLARE @jsonArray NVARCHAR(MAX) = JSON_QUERY(@response, '$.result.data[0].embedding');
SET @embedding = JSON_ARRAY_TO_VECTOR(@jsonArray);
END
GO
```

Now that you can get vector distances, you could combine this with other SQL searching methods or hybrid search like the following:

```
-- Assuming you have a stored procedure to get embeddings for a given text
DECLARE @e VARBINARY(8000);
EXEC dbo.GET_EMBEDDINGS @model = '', @text = 'quick fix breakfast option for toddlers', @embedding = @e OUTPUT;
-- Comprehensive query with multiple filters.
SELECT TOP(10)
f.Id,
f.ProductId,
f.UserId,
f.Score,
f.Summary,
f.Text,
VECTOR_DISTANCE('cosine', @e, VectorBinary) AS Distance,
CASE
WHEN LEN(f.Text) > 100 THEN 'Detailed Review'
ELSE 'Short Review'
END AS ReviewLength,
CASE
WHEN f.Score >= 4 THEN 'High Score'
WHEN f.Score BETWEEN 2 AND 3 THEN 'Medium Score'
ELSE 'Low Score'
END AS ScoreCategory
FROM FineFoodReviews f
WHERE
f.UserId NOT LIKE 'Anonymous%' -- User-based filter to exclude anonymous users
AND f.Score >= 2 -- Score threshold filter
AND LEN(f.Text) > 50 -- Text length filter for detailed reviews
AND (f.Text LIKE '%gluten%' OR f.Text LIKE '%dairy%') -- Inclusion of specific words
ORDER BY
Distance,  -- Order by distance
f.Score DESC, -- Secondary order by review score
ReviewLength DESC; -- Tertiary order by review length
```

Imagine now combining the use of JSON, RegEx, spatial, inequality and equality operators, and vector distance to truly execute a smarter search using the power of SQL. That is beyond RDBMS!

Remember that you can additionally use the results of hybrid search *with a* *RAG pattern* as Davide Mauri shows in this blog post: [`https://devblogs.microsoft.com/azure-sql/rag-with-azure-sql`](https://devblogs.microsoft.com/azure-sql/rag-with-azure-sql).

Note

This story is just going to continue to get better. This includes support for a vector data type, index, and other T-SQL built-in features to make the entire hybrid search story simpler and streamlined with your application. I will leave that for the next book.

Using the SQL engine to power your GenAI application has many benefits, including

*   Familiarity of the T-SQL language

*   Integrated into the query processor and data types

*   Benefit from the scale of the engine, including Azure SQL Hyperscale

*   Use the security boundary you trust with authentication and access control in the engine

#### Use Frameworks

With the advantage of using the SQL engine, you will still need to use a driver or provider to connect and execute queries. One example that is available for you is for Entity-Framework core which you can see at [`https://github.com/Azure-Samples/azure-sql-db-vector-search/tree/main/EFCoreVectors`](https://github.com/Azure-Samples/azure-sql-db-vector-search/tree/main/EFCoreVectors).

Furthermore, it might be nice to have a set of libraries (framework) that can *orchestrate* the activities of using SQL for GenAI applications.

Two popular frameworks today are **LangChain** and **Semantic Kernel**, both open source libraries. LangChain and Semantic Kernel have built-in support for SQL and as we build out our vector story further will include SQL as a vector store. LangChain also has a library to help you generate SQL queries based on your schema from a natural language prompt.

You can learn more about Semantic Kernel support for SQL at [`https://github.com/microsoft/semantic-kernel/tree/main/dotnet/src/Connectors/Connectors.Memory.SqlServer`](https://github.com/microsoft/semantic-kernel/tree/main/dotnet/src/Connectors/Connectors.Memory.SqlServer).

You can learn more about LangChain support for SQL at [`https://python.langchain.com/v0.2/docs/integrations/tools/sql_database`](https://python.langchain.com/v0.2/docs/integrations/tools/sql_database).

#### What About Chat History?

Many GenAI applications today are chat experiences using prompts. We have seen most of these chat experiences require the need to store the history of the chat (with permissions of the user) in addition to feedback from the user about the chat experience. Databases seem like a natural fit to store this type of data, so Azure SQL could be a good candidate. There are not a lot of standards yet in this space, but I expect in the future there will be.

One example of a schema that could be used for chat history and feedback comes from the open source project **ChainLit** (used to build chatbots) at [`https://docs.chainlit.io/data-persistence/custom`](https://docs.chainlit.io/data-persistence/custom).

#### Where Do You Store Your Text Data?

All the GenAI examples provide smart searching on text data because that type of data is natural fit for language models (even though there are language models that support video, audio, images, etc.).

While SQL can be used in many situations to store text along with attributes of the text that are other columns, I’ve seen many customers store text data they want to use with GenAI in other storage systems, including file systems, Azure Storage, S3, and others. These could be JSON documents, text files, csv files, and more prominently PDF documents.

Therefore, as you look at GenAI, you should consider whether it will make sense to store this text data in Azure SQL and classify the text with other attributes in columns to prove hybrid searching with AI models.

One resource to consider helping you extract PDF into text to insert into SQL is to use a Python library such as [`https://pypi.org/project/pypdf`](https://pypi.org/project/pypdf). PDF files are so common I think this will be a future scenario to focus on.

#### Resources

Bookmark the following resources to keep up to date with all the latest we have on SQL and AI:

[`https://aka.ms/sqlai`](https://aka.ms/sqlai) – This is a one-stop hub for all topics related to SQL and AI in our documentation.

[`https://aka.ms/sqlaisamples`](https://aka.ms/sqlaisamples) – This is a GitHub repo with code examples.

[`https://aka.ms/sqlaiworkshop`](https://aka.ms/sqlaiworkshop) – This is a workshop my colleague Muazma Zahid, me, and others on our team have built to get an end-to-end look at SQL and AI.

Another good summary of our capabilities can be found in my blog post at [`https://www.microsoft.com/sql-server/blog/2024/06/26/getting-started-with-delivering-generative-ai-capabilities-in-sql-server-and-azure-sql`](https://www.microsoft.com/sql-server/blog/2024/06/26/getting-started-with-delivering-generative-ai-capabilities-in-sql-server-and-azure-sql). In this blog post, I also referenced a video I recorded with the famous Microsoft Mechanics team which you can see at `https://aka.ms/SQLAIMechanics`.

### Natural Language to SQL Query

You have seen throughout the book examples of Microsoft Copilot skills in Azure SQL Database to assist you in managing and troubleshooting Azure SQL. We have developed another AI-assisted experience called **natural language** **to SQL query**.

This experience is available today in the context of your database in the Azure Portal using the Query Editor. This feature uses a RAG pattern by using your schema (including table names, column names, and keys) to pass to a language model to generate a SQL query.

The best way to see how this works is to see it in action. I navigated to a database I’ve created in this book called **bwadw** which uses the sample AdventureWorksLT during deployment.

I selected the Query Editor from the Service menu and logged in. Now I have a new option in the Query Editor to **Open Copilot**. When I do this, I get a new edit box to type in a natural language statement that has context to my database like in Figure [10-11](#496204_2_En_10_Chapter.xhtml#Fig11).

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig11_HTML.jpg)

Figure 10-11

Using natural language to generate SQL for a multi-join scenario

You can see from the text I used a query would need to probably use multiple tables which likely requires several joins. The **Table selection** option allows me to advise the Copilot in the editor which tables to use to feed the RAG pattern for the model. This does not use your data, only your schema. The default is all tables.

If I select **Generate Query** and then select **Accept**, I’ll get the following query generated as you can see in Figure [10-12](#496204_2_En_10_Chapter.xhtml#Fig12).

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig12_HTML.jpg)

Figure 10-12

Generated query for multi-join scenario

This doesn’t show the full query, but I wanted to show you the comment generated which explains what was generated for the query. You can see in the code fragment the T-SQL required to join multiple tables. The full query looks like this:

```
SELECT
c.FirstName + ' ' + c.LastName AS CustomerName,
a.StateProvince AS State,
p.Name AS ProductName,
SUM(od.LineTotal) AS TotalSales
FROM
SalesLT.Customer AS c
JOIN SalesLT.CustomerAddress AS ca ON c.CustomerID = ca.CustomerID
JOIN SalesLT.Address AS a ON ca.AddressID = a.AddressID
JOIN SalesLT.SalesOrderHeader AS oh ON c.CustomerID = oh.CustomerID
JOIN SalesLT.SalesOrderDetail AS od ON oh.SalesOrderID = od.SalesOrderID
JOIN SalesLT.Product AS p ON od.ProductID = p.ProductID
JOIN SalesLT.ProductCategory AS pc ON p.ProductCategoryID = pc.ProductCategoryID
WHERE
pc.Name = 'Mountain Bikes'
GROUP BY
c.FirstName + ' ' + c.LastName,
a.StateProvince,
p.Name
ORDER BY
TotalSales DESC;
```

You can see all the joins and aggregates generated required to support natural language statements like “by state” and “ordered by….”

I can now take the query and run it by selecting Run or take the generated query and run it in my favorite SQL tool.

Another interesting example for me is generating a query that allows you to navigate a hierarchy within a relational table. So, I fed Copilot in the editor this prompt:

```
Find the heirarchy of product categories at any level
```

Note

I misspelled the word hierarchy on purpose just to see if it still worked.

Copilot in the editor generated the following query:

```
WITH RecursiveCTE AS (
SELECT
ProductCategoryID,
Name,
ParentProductCategoryID,
0 AS Level
FROM
SalesLT.ProductCategory
WHERE
ParentProductCategoryID IS NULL
UNION ALL
SELECT
pc.ProductCategoryID,
pc.Name,
pc.ParentProductCategoryID,
rc.Level + 1
FROM
SalesLT.ProductCategory pc
INNER JOIN
RecursiveCTE rc ON pc.ParentProductCategoryID = rc.ProductCategoryID
)
SELECT
ProductCategoryID,
Name,
ParentProductCategoryID,
Level
FROM
RecursiveCTE
ORDER BY
Level, ProductCategoryID;
```

So nice. The generated query is an example of a recursive Common Table Expression (CTE). You can learn more about this query technique at [`https://learn.microsoft.com/sql/t-sql/queries/with-common-table-expression-transact-sql?view=sql-server-ver16#guidelines-for-defining-and-using-recursive-common-table-expressions`](https://learn.microsoft.com/sql/t-sql/queries/with-common-table-expression-transact-sql%253Fview%253Dsql-server-ver16%2523guidelines-for-defining-and-using-recursive-common-table-expressions). I believe this capability will help so many to build SQL queries from the simplest to the most complex, saving you time and effort.

I was chatting about this feature with Joe Sack, who is the PM leader for this project and just happens to be the tech editor for this book. He said to me, “Bob did you know that this Copilot can generate queries for system catalog views and DMVs, right?” I was like “Uh no.” We know the schema of all our system catalog view and DMVs, so we can easily feed any language model this metadata. So, I gave it a spin with two examples:

**Show me which tables have consumed the most space in this database**

The resulting generated query was the following:

```
SELECT
t.NAME AS TableName,
s.Name AS SchemaName,
p.rows AS RowCounts,
SUM(a.total_pages) * 8 AS TotalSpaceKB,
SUM(a.used_pages) * 8 AS UsedSpaceKB,
(SUM(a.total_pages) - SUM(a.used_pages)) * 8 AS UnusedSpaceKB
FROM
sys.tables t
INNER JOIN
sys.indexes i ON t.OBJECT_ID = i.object_id
INNER JOIN
sys.partitions p ON i.object_id = p.OBJECT_ID AND i.index_id = p.index_id
INNER JOIN
sys.allocation_units a ON p.partition_id = a.container_id
LEFT OUTER JOIN
sys.schemas s ON t.schema_id = s.schema_id
WHERE
t.NAME NOT LIKE 'dt%'
AND t.is_ms_shipped = 0
AND i.OBJECT_ID > 255
GROUP BY
t.Name, s.Name, p.Rows
ORDER BY
TotalSpaceKB DESC;
```

That was a good test for catalog views.

What about DMVs? I tried this prompt:

**Show me the top preemptive waits for this database**

The generated query was exactly what I wanted:

```
SELECT TOP 10 wait_type, wait_time_ms, signal_wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type LIKE 'PREEMPTIVE%'
ORDER BY wait_time_ms DESC;
```

I know Joe and the team are testing a wide range of options for what is possible here, so I’m so excited to see the future.

## SQL and Microsoft Fabric

In the book *SQL Server 2022 Revealed*, I talked about a capability called **Synapse Link**. The concept is to sync data from Azure SQL or SQL Server to Azure Synapse, and then changes are captured and fed to Synapse for a near real-time analytics solution.

In 2023, we released a new data platform called Microsoft Fabric which includes all types of data solutions, including analytics. Figure [10-13](#496204_2_En_10_Chapter.xhtml#Fig13) shows a visual picture of Microsoft Fabric.

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig13_HTML.jpg)

Figure 10-13

Microsoft Fabric

You can see that Synapse is now part of the Fabric family. But more importantly, a concept called **OneLake** is a critical part of this story.

OneLake is defined in the documentation as “OneLake is a single, unified, logical data lake for your whole organization. A data lake processes large volumes of data from various sources.” What I like is the title of the main documentation page for OneLake which says **OneLake, the OneDrive for data** ([`https://learn.microsoft.com/fabric/onelake/onelake-overview`](https://learn.microsoft.com/fabric/onelake/onelake-overview)). I like this description because I think of my OneDrive as a place to store any file of any type anytime I want.

But a key difference for OneLake in my opinion is that a common data format is **Delta**. Delta is a file format based on Parquet that lends itself very easily to querying data like a table. This means the Fabric team can provide a **SQL endpoint** for you to query anything in OneLake that is in Delta format.

Now back to the story of Synapse Link. **Azure SQL with Synapse Link** uses parquet files to initially sync tables into a landing zone. Synapse then takes this data and creates Synapse pool tables. Azure SQL then uses a new change feed architecture based on the transaction log to send changes to the landing zone, where they are consumed by Synapse to submit changes to pool tables.

Fabric uses a concept called **mirroring**. Figure [10-14](#496204_2_En_10_Chapter.xhtml#Fig14) shows Fabric mirroring for databases.

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig14_HTML.jpg)

Figure 10-14

Fabric mirroring

**Fabric mirroring** **for SQL** is effectively the same capability as Synapse Link for SQL except the data is stored in Fabric OneLake as you see in Figure [10-15](#496204_2_En_10_Chapter.xhtml#Fig15).

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig15_HTML.jpg)

Figure 10-15

Fabric mirroring for SQL

Behind the scenes, our SQL code is pretty much the same as with Synapse Link for Azure SQL and SQL Server except the landing zone is in the Fabric. Fabric now replicates that data into OneLake automatically in Delta format.

Now that the data is in OneLake, the entire Fabric ecosystem is available to your SQL data. Figure [10-16](#496204_2_En_10_Chapter.xhtml#Fig16) shows an example query using the SQL Analytics endpoint against SQL mirrored data to run a SQL query.

![](images/496204_2_En_10_Chapter/496204_2_En_10_Fig16_HTML.jpg)

Figure 10-16

Using the SQL Analytics endpoint in Fabric

Imagine now getting to join this data with any other data in OneLake or using this data to build Power BI reports.

Synapse Link is still supported today, but our future in this area will be mirroring. Here are some notable differences:

*   Mirroring supports DDL changes like add/drop column.

*   Your data does not go to Synapse even in Fabric. It is stored in OneLake and any query against this data using serverless queries.

We already have a preview for Azure SQL Managed Instance for mirroring, and I expect mirroring for SQL Server in VM and on-premises to be available soon. Learn all the details to get started at [`https://aka.ms/sqldbmirroring`](https://aka.ms/sqldbmirroring). You can see an example of using SQL mirroring with AI in Fabric at [`https://aka.ms/sqlaisamples`](https://aka.ms/sqlaisamples).

## Azure Arc

One final topic for the chapter not in the Beyond RDBMS in Figure [10-1](#496204_2_En_10_Chapter.xhtml#Fig1) is **Azure Arc**. I’ve included Azure Arc in this chapter because I did briefly mention this in the first edition of the book, and it is worth calling it out again. I also covered this topic extensively in the book *SQL Server 2022 Revealed*.

Azure Arc is a simple but powerful concept. This is not deploying SQL in Azure but rather *connecting* your existing SQL Server on-premises or in other clouds ***to Azure***.

Connecting means running lightweight agents inside your VM or on the machine where SQL Server is installed. These agents connect you to Azure to give you additional capabilities and make it *feel like* you are running in Azure. We call this SQL Server enabled by Azure Arc.

One of the powerful aspects to Arc is that we can keep adding capabilities without a change to SQL Server but also take advantage of new features with a new major version. SQL Server enabled by Azure Arc is updated on a monthly basis. As of the time of the writing of this book, here is a list of features Arc provides you:

*   **Manage your SQL Server instances at scale** from the Azure portal

*   **Best practices assessment** to see any possible issues from your current configuration of your server and databases

*   **Microsoft Entra authentication** support for SQL Server 2022

*   **Microsoft Defender for Cloud** which we have covered extensively in Chapter [6](#496204_2_En_6_Chapter.xhtml) of this book

*   **Microsoft Purview** support for SQL Server 2022 which we covered in Chapter [6](#496204_2_En_6_Chapter.xhtml) of this book

*   **Pay-as-you-go licensing** for SQL Server to save licensing costs especially for infrequently used SQL Server instances

*   **Extended Security Updates (ESU)** for older versions of SQL Server

*   **Performance dashboards** to give you lightweight visualizations of your SQL Server performance

*   **Migration assessment** to continuously check your ability to migrate to Azure SQL options

Learn more about SQL Server enabled by Azure Arc at [`https://aka.ms/arcsqlserver`](https://aka.ms/arcsqlserver).

## Futures

The future is very bright in my opinion for Azure SQL especially in the areas that I described in this chapter. I have some thoughts here for the future. None of these are a promise from Microsoft or guarantee of a feature, but I thought it to be important to give you my insights. We will see over the next year or so whether my instincts were right.

*   **Use of** **Small Language Models (SLM)**

    I think many people today are *overprovisioning* AI applications with Large Language Models because no one really knows how to “right size” with a language model. I think the use of SLMs will become popular as they are less expensive and consume less resources. However, finding the right model and right size will become the next set of challenges in AI.

*   **NL2SQL frameworks**

    Vector search works fine for text data, and you can combine this with hybrid search. But consider developers who want the SQL to do all of this generated by AI. Therefore, I think you may see more frameworks are used to generate NL2SQL and that SQL could include hybrid searching.

*   **GenAI ground to cloud**

    If we follow suit on how we have taken on other big investments, I expect SQL Server to see features that will support GenAI applications. I have seen in 2024 companies like Dell, HPE, Nvidia, AMD, and Red Hat emphasize the need for AI applications not running in public clouds. Microsoft just happens to have one of the most popular databases in the planet in SQL Server so….

*   **More and enhanced Copilot experiences**

    Copilots are here to stay; it is just a matter of refining them and making sure they operate at the right speed and provide value. I think Microsoft will continue to invest in this space including our team that runs Azure SQL, SQL Server, and our tools.

*   **Enhancements for JSON and vectors**

    We have a good start with JSON and vectors in SQL, but we need more, and I expect you will see more investments from us.

*   **Keep investing in Microsoft Fabric**

    In 2024, Microsoft Fabric was on fire! It continues to be a popular data platform, and I expect that it will continue to grow. I expect to see new and deeper experiences with data with Microsoft Fabric in the future.

## Summary

I thought it would be fitting in the summary of the last chapter of this book to see if Copilot in Microsoft Word could build a good summary. I thought it was pretty good, so I put it here in quotes unedited.

> “Azure SQL offers advanced features that extend beyond traditional Relational Database Management Systems (RDBMS). These capabilities include extensive JSON support, allowing for efficient parsing, transforming, querying, and formatting of JSON data. Additionally, Azure SQL supports Regular Expressions (RegEx) for advanced pattern matching and validation, enhancing traditional T-SQL string functions. The platform also includes spatial and graph data types, enabling the modeling and querying of network-like data structures using nodes and edges.
> 
> Furthermore, Azure SQL provides columnstore indexing and memory optimization to significantly enhance query performance. The SQL Ledger feature uses temporal tables to automatically track changes, ensuring auditability and data integrity through cryptographic hashes. Intelligent Query Processing (IQP) and Automatic Tuning features optimize query performance and application efficiency. Azure SQL also integrates with REST APIs, allowing seamless data exchange and interaction with external services directly from the database engine.
> 
> Lastly, Azure SQL supports AI applications and DevOps practices, including GenAI and natural language to SQL query generation. The platform integrates with GitHub Actions and containers, facilitating modern development workflows. Data API Builder (DAB) simplifies app development by translating database connections and schemas into REST or GraphQL endpoints, reducing the need for extensive plumbing code. These advanced features make Azure SQL a powerful tool for modern data management and application development.”

Index A Accelerate and tune performance application latency CPU capacity I/O performance memory SQL Server worker limits Accelerated Database Recovery (ADR) Accelerated networking Access Control List (ACL) ACL SeeAccess Control List (ACL) ADF SeeAzure Data Factory (ADF) ADR SeeAccelerated Database Recovery (ADR) ADS SeeAdvanced Data Security (ADS) Advanced Data Security (ADS) Advanced Threat Protection (ATP) AGs SeeAvailability Group (AGs) AHB SeeAzure Hybrid Benefit (AHB) AI Search indexes AKS SeeAzure Kubernetes Service (AKS) Always Encrypted Amazon Elastic Compute Cloud (EC2) AMD processor APC SeeAutomatic Plan Correction (APC) API SeeApplication Programming Interface (API) Application Programming Interface (API) ARM SeeAzure Resource Manager (ARM) ARM architecture ARM templates ATP SeeAdvanced Threat Protection (ATP) Automated backup Automated backup system Automated patching Automatic indexing Automatic Plan Correction (APC) Automatic Tuning Auto-pause delay AutoPilot Availability Group (AGs) Availability Sets Availability Zones az CLI az sql vm az sql vm CLI Azure definition Azure account Azure Active Directory Azure Advisor Azure Arc Azure Arc migration assessment Azure automation Azure Availability Set Azure Backup Azure Bastion Azure Blob Storage Azure Boost Azure built-in roles Azure Cloud Shell Azure Confidential Ledger Azure data center Azure Data Factory (ADF) Azure documentation AzureDTExec Azure ecosystem account API ARM Azure Marketplace Azure Monitor Azure Portal CLI data centers identities Management Groups Pay As You Go Policy regions security concept SLAs subscription tenant trust, privacy, and compliance Azure File Shares Azure Government regions Azure Hybrid Benefit (AHB) Azure Ideas Azure interfaces Azure Key Vault Azure Key Vault integration Azure Kubernetes Service (AKS) Azure locks Azure logical server Azure Marketplace Azure Marketplace Terms Azure metrics Azure Migrate Server Migration Azure Monitor Azure Policy Azure Portal Azure Cloud Shell Copilot customize dashboard deployment of SQL Server 2022 deployment options features home page Microsoft Copilot skills in Azure SQL Database notifications preview Query Editor quickly create search SQL services and marketplace user interface Azure PowerShell Azure Pricing Calculator Azure Private Link Azure products Azure regions Azure Reserved VM Instances Azure Resource Manager (ARM) Activity Logs Azure documentation Azure locks benefits deployment and management interfaces RBAC resource groups templates tools and APIs Azure Security Center for SQL Azure Service Fabric Azure Services Platform cloud services FC IaaS IIS in 2008 PaaS PDC Red Dog Service Fabric Windows Fabric Azure Shared Disks Azure SQL aspects best practices deployments documentation references externa tables CREATE TABLE AS SELECT (CETAS) data sources/file formats data virtualization external data source file formats integration, Microsoft Fabric query failover groups features IaaS *vs.* PaaS interfaces lineup SeeAzure SQL lineup in Marketplace performance SLA Azure SQL Database ADS auditing authentication az CLI *vs.* Azure SQL managed instance Azure VM bwsql2022 back end branding Business Critical service tiers capabilities cloud-first approach CloudLifter collation configuration choices configuration options customers database Database as a Service database compatibility level data integrity decision flow deployed private endpoint deployment deployment options documentation reference DTU DTU *vs.* vCore editions eDTU elastic pool engineering model ERRORLOG files ERRORLOG value FILESTREAM fleet manager GA General Purpose Hyperscale Ibiza ILDC intelligent performance locked pages maintenance window Malmo manage Azure SQL Managed Instance manage security policies MDCS migrating networking options options options and limits PaaS environment performance monitoring performance tiers Pottermore Premium private and public preview private IP address purchasing model purchasing option reporting and customer feedback restrictions database storage Query Store space management SaaS applications self-service database restore server admin login serverless Serverless serverless database server proxy account service tiers single database Socrates SQL elastic pools SQL logins and map stale page detection startup parameters statement Sterling stop and start surface area syntax TFS time zone update policy user accounts vCore purchasing model Web and Business editions Azure SQL Database Hyperscale Azure SQL Database Managed Instance Azure SQL lineup Azure SQL Database managed instance PaaS threads Virtual Machines visual Azure SQL Managed Instance *vs.* Azure SQL Database benefits business critical capabilities customers and Database database engine instance general purpose PaaS service pools service tier *vs.* SQL Server on Azure Virtual Machines surface area tools tracks failed logins T-SQL differences T-SQL scripts versionless and versioned SQL Server Azure SQL security Azure Storage Azure subscription Azure tenant Azure Traffic Manager Azure Update Manager Azure Virtual Machine Azure Virtual Network Gateway Azure VM configuration CLI interfaces resizing Security (RBAC) Service Menu stopping *vs.* deallocating connecting with RDP deploying HADR navigating command bar essentials service menu Working Pane networking performance monitoring SQL Server IaaS Agent Extension storage performance Azure Blob Storage best practices Premium SSD v2 Azure VM bwsql2022 Azure VM resources az vm CLI B BACKUP and RESTORE functionality BACPAC file Basics BC SeeBusiness Critical (BC) Beyond RDBMS Bing Search Service Bison BKW Energie Blackbird Blades Boot diagnostics Bring Your Own Key (BYOK) for Azure SQL database with TDE Bring your own license (BYOL) Built-in availability Business Critical (BC) applications architecture backup files bwsqlbc failover failover test gateways local storage MFA ostress.exe program General Purpose (GP) Hyperscale architecture aspects caches database files database page HA replicas Log Service named replicas nodes page servers secondary replicas snapshot backups Bulk copy program (bcp) BULK INSERT Business Critical (BC) Business Critical service Business Critical service tiers bwhyperscale bwsql2022 bwsqllogicalserver BYOL SeeBring Your Own License (BYOL) C CampBrain Capacity reservations CDC SeeChange data capture (CDC) C&E Chaining Change data capture (CDC) Chat playground Classic virtual machines CLI SeeCommand-line interface (CLI) Cloud Microsoft Defender Cloud computing CloudDB Cloud-first approach CloudLifter Cloud services Cloud witness CMK SeeCustomer-managed key (CMK) Columnstore indexes Command-line interface (CLI) with ARM templates Common Table Expression (CTE) Community Technology Preview (CTP) Compute model Compute tiers Confidential virtual machine Configuration options database files and filegroups maintaining indexes maintaining statistics MAXDOP Resource Governor tempdb Configuring Azure SQL ALTER DATABASE reference compute and storage configuration options database database compatibility level General Purpose service instance I/O performance maintenance window master database settings and Maintenance Updates setup program SLA requirements sp_configure start and stop instance tempdb trace flags update policy virtual network Configuring networking Connecting to Your VM Containers Copilot architecture database context database context results skills Copilot experience Cost of Goods Sold (COGS) Cross-database queries CTE SeeCommon Table Expression (CTE) CTP SeeCommunity Technology Preview (CTP) Customer-managed key (CMK) D DAB SeeData API Builder (DAB) DAB architecture DAC SeeDedicated Admin Connection (DAC) DACs SeeData-tier applications (DACs) DAG SeeDistributed Availability Group (DAG) Data API Builder (DAB) Database as a Service Database availability Database compatibility Database consistency Database engine instance Database ledger Database Mail Database Migration Service (DMS) Database Transaction Unit (DTU) Database Tuning Advisor Database Tuning Advisor (DTA) Database Watcher Data centers Data classification overview Data disks Data end-to-end Data loading techniques Data nodes Data providers Data Security Dashboard Data-tier applications (DACs) Data virtualization Data warehouse services DDM SeeDynamic Data Masking (DDM) Dedicated Admin Connection (DAC) Dedicated host Deploy! Deploying accounts, port, and OS licensing availability options on Azure VM Bison capacity reservations CLI with ARM templates configuration advanced management monitoring networking OS disks SQL server settings tags create button dedicated host Deploy on Your Own download a template for automation EULA gallery images go to resource image migration to azure navigating notifications icon pricing Privacy policy region reserved instances resource group security type spot instance SQL server configuration containers installs Linux with Azure VM Terraform validation VM architecture VM name VM sizes Deploying Azure SQL application authentication choices Basics screen Basics screen choices Compute + Storage compute and storage options connection type consideration and tools Copilot customers database database platforms database server deployment deployment choices deployment options DMS firewall rule GP and BC infrastructure and virtualization logical server migration networking options performance and proximity POC pre-deployment planning pricing calculator process public endpoint region security choices SLO storage size switch database context virtual machine virtual network Deployment Deploy on Your Own Dev Container templates DevOps model Disaster recovery Managed Instance link offline online Distributed Availability Group (DAG) Distributed transactions coordinator (DTC) DMVs SeeDynamic Management Views (DMVs) DMS SeeDatabase Migration Service (DMS) DOP feedback DTU SeeDatabase Transaction Unit (DTU) Dynamic Data Masking (DDM) Dynamic Management Views (DMVs) Deep Dive Deep Troubleshooting Managed Instance SQL Database SQL Managed Instance XEvent E E4bds_v5 EC2 SeeAmazon Elastic Compute Cloud (EC2) *eDTU* EHA SeeExchange Hosted Archive (EHA) Elastic Jobs Elastic pools Embeddings Enable auto-shutdown Enabled guest-level monitoring Enable guest-level monitoring Encryption at Host End User Licensing Agreement (EULA) Entra admin ERRORLOG ERRORLOG files E-series EULA SeeEnd User Licensing Agreement (EULA) ESU SeeExtended Security Updates (ESU) Ev4/Ev5 Exchange Hosted Archive (EHA) Extended Events (XEvent) Extended Security Updates (ESU) Extensions External data source F Fabric controller (FC) Fabric mirroring Failover Cluster Instance (FCI) Failover groups architecture configuration/managing databases deployment details group name important points listeners locations logical server level policies Failover process Fast provisioning Fault domains FC SeeFabric controller (FC) FCI SeeFailover Cluster Instance (FCI) File snapshot backups Firewall Firewall rules Fleet manager Fleetspace FQDN SeeFully Qualified Domain Name (FQDN) Full SQL Server box Full-text search Fully Qualified Domain Name (FQDN) G GA SeeGeneral Availability (GA) Gallery images Gateway Server GDPR SeeGeneral Protection Data Regulation (GDPR) Gen5 GenAI applications benefits history General Availability (GA) General Protection Data Regulation (GDPR) General Purpose, Business Critical, and Hyperscale General Purpose (GP) General Purpose service Generation 2 (Gen2) VM Geographic information system (GIS) platform Geography Geo-redundant storage (GRS) Geo-replication Azure SQL Database considerations databases failover failover methods features Hyperscale database replicas details seeding Geo-restore process Geo-zone-redundant storage (GZRS) GitHub go-sqlcmd Government/national regions GP SeeGeneral Purpose (GP) Graph database Group GRS SeeGeo-redundant storage (GRS) Guest OS updates GZRS SeeGeo-zone-redundant storage (GZRS) H HADR SeeHigh Availability and Disaster Recovery (HADR) HADR capabilities automatic backups backup retention backup storage consumption costs GRS GZRS LRS schedule T-SQL statements ZRS built-in availability SeeBuilt-in availability database availability database consistency disaster recovery SeeDisaster recovery failover groups SeeFailover groups geo-replication SeeGeo-replication geo-restore Managed Instance monitoring availability metrics backup/restore history failover reasons interfaces PowerShell commands regions and data centers replica status Resource Health service availability passive replica PITR SeePoint-in-time restore (PITR) restore backups zone redundancy SeeZone redundancy Handling errors Hardware configuration Hardware security modules (HSMs) Helsinki High availability (HA) High Availability and Disaster Recovery (HADR) AG Azure availability Azure Backup Azure Storage FCI Linux high availability SQL Server Host group Hot patching HSMs SeeHardware security modules (HSMs) Hybrid search Hyperscale architecture service tier Hyper-V Hyper-V Manager Hyper-V VM I IaaS SeeInfrastructure as a Service (IaaS) Ibiza IFI SeeInstant File Initialization (IFI) ILDC SeeMicrosoft Israel Development Center (ILDC) Incubation Inference endpoints Infrastructure as a Service (IaaS) In-Memory OLTP Input/Output per Second (IOPS) IIS SeeInternet Information Server (IIS) Insights Instant File Initialization (IFI) Intelligent capabilities APC Automatic Tuning automation IQP query details recommendations steps to connect database Intelligent performance Intelligent Query Processing (IQP) Interfaces for Azure SQL az CLI Azure Portal PowerShell REST API SQL CLIs SSMS TDS T-SQL Visual Studio Tools Internet Information Server (IIS) Internet services The Internet Services Disruption I/O latency IOPS SeeInput/Output per Second (IOPS) IQP SeeIntelligent Query Processing (IQP) J JSON formatted data capabilities columnstore indexes functions geography instance graph database JSON_ARRAYAGG JSON_OBJECTAGG ledger native RegEx spatial and graph STDistance Just-in-time access K Kerberos authentication Komatsu Kubernetes Kusto Query Language (KQL) queries L Language model Large Language Models (LLM) Linked servers Linux high availability Live Site experiences LLM SeeLarge Language Models (LLM) Load Balancing options Locally redundant storage (LRS) Log Analytics workspace Logical database server Logical master database Log rate governance Long-term backup retention (LTR) LRS SeeLocally redundant storage (LRS) LTR SeeLong-term backup retention (LTR) LTR backups M Machine learning services Malmo Managed identities Managed Instance automated backups backup retention deleting database failover groups geo-zone-redundant backups jumpbox/virtual machine PITR RESTORE T-SQL statement SQL Server 2022 Managed Instance link Managed Instance pool Management Groups Marketplace Master database Master node MAXDOP SeeMax degree of parallelism (MAXDOP) Max degree of parallelism (MAXDOP) Maximizing Storage Performance Max server memory MDCS SeeMicrosoft Developer Center Serbia (MDCS) Memory optimized Memory-optimized tables Microsoft Copilot forum support plans Microsoft Copilot skills Microsoft Copilot skills in Azure SQL Database Microsoft defender Microsoft Defender Microsoft Defender for Cloud Microsoft Developer Center Serbia (MDCS) Microsoft Entra Microsoft Entra authentication Microsoft Entra logins Microsoft Entra principals Microsoft Fabric Microsoft Israel Development Center (ILDC) Microsoft Purview DevOps Migration Azure Arc migration assessment Azure Migrate Server Migration database restore Azure files Azure import/export service Azure infrastructure backup to URL copy and paste and DMS Migration assessment Mirroring Mixed mode security Monitoring tools Azure monitor Copilot Database Watcher DMV Extended Events Lightweight Query Profiling query plan debugging Query Store visualizations mssql Multiple VMs Multi-subnet virtual network Multi-tenant Multi-tenant databases N Native vectors Native vector support Natural language Networking Networking settings Network security group (NSG) NextGen General Purpose service NL2SQL frameworks NSG SeeNetwork security group (NSG) Noisy neighbor O OneLake OpenAI model Optimizing locking OS disks OS guest diagnostics Outbound traffic Overprovisioning P PaaS SeePlatform as a Service (PaaS) PaaS service PaaS SQL Partitions Passwordless authentication Pausing Pay As You Go PDC SeeProfessional Developers Conference (PDC) Performance capability indexes In-Memory OLTP intelligent performance max capacities partitions SQL Server 2022 enhancements IQP query plan Performance monitoring DMVS insights metrics operating system tools Performance tiers Performance troubleshooting Copilot PowerShell/Alerts running scenario CPU problem Database Watcher Query Store sys.dm_exec_procedure_stats sys.dm_exec_query_stats sys.dm_exec_requests running *vs.* waiting sys.dm_db_resource_stats waiting Database Watcher Query Store resources sys.dm_exec_requests sys.dm_os_waiting_tasks sys.dm_os_wait_stats Personally Identifiable Information (PII) Pink Poodle PITR SeePoint-in-time restore (PITR) Platform as a Service (PaaS) Plumbing code POC SeeProof of concept (POC) Point-in-time restore (PITR) automated backups create database database database name documentation dropped table drop the table key concept log backup Managed Instance recovery time restore database restore process Point-to-Site (P2S) Polybase method Portal Pottermore Power BI PowerShell commands Premium Premium file share Premium SSD Premium SSD v2 Pricing calculator Privacy policy Professional Developers Conference (PDC) Project Dearborn Proof of concept (POC) Proximity placement groups Proxy connection P2S SeePoint-to-Site (P2S) Purchasing option Purhasing model Q Query intelligence Query Store QuickStart Quota increase request R RCSI SeeRead Committed Snapshot Isolation (RCSI) RDFE SeeRed Dog Front End (RDFE) Read Committed Snapshot Isolation (RCSI) Recovery model Recovery Point Object (RPO) Recovery Time Object (RTO) Red Dog Red Dog Front End (RDFE) Relational Database Management System (RDBMS) Remote Desktop Client (RDP) Remote Desktop Connection (RDP) protocol Remote Desktop Protocol (RDP) Representational State Transfer (REST) Resizing Resource Governor Resource group Resource reservations REST API REST API endpoints Restore capabilities RESTORE DATABASE statement RESTORE statement Retrieval Augmented Generation (RAG) pattern Role-Based Access Control (RBAC) administrator contributor role SQL Server RPO SeeRecovery Point Objective (RPO) R Services (Advanced Analytics) RTO SeeRecovery Time Object (RTO) Runbook Runner Running scenario S SaaS SeeSoftware as a Service (SaaS) SaaS applications SaaS ISVs SAWA SeeSQL Azure on Windows Azure (SAWA) SAWAv2 Scratch Script files SDK-style SQL projects SDS SeeSQL Data Services (SDS) Securing Azure SQL capabilities deployment private endpoint private link public endpoint security security, performance, and availability virtual machine VNet peering Security management authentication Azure Activity Log Azure ecosystem Azure portal Security playbook Self-service database restore Serverless Serverless database hyperscale bwsqldbserverless Serverless Hyperscale database Server Tools and Business (STB) Service Broker Service Fabric (SF) Service-Level Agreements (SLAs) Service-Level Objective (SLO) Service tiers Service Update (SU) SF SeeService Fabric (SF) Silos Simple Mail Transfer Protocol (SMTP) Single database Site-to-Site (S2S) SLM SeeSmall Language Models (SLM) SLO SeeService-Level Objective (SLO) Small Language Models (SLM) SMTP SeeSimple Mail Transfer Protocol (SMTP) Snapshot backups Socrates Software as a Service (SaaS) Solaria project Spark Spatial data sp_invoke_external_rest_endpoint Spot instance sql-action SQL Audit SQL auditing SQL Audit records SQL Authentication SQL Authentication account SQL Azure Azure data center characteristics logo management portal multi-tenant databases PaaS relational database service portal SAWA SDS Silverlight administration and Windows SQL Azure on Windows Azure (SAWA) SQL Database auditing SQL Data Services (SDS) AutoPilot capacity planning clusters COGS database as a utility data nodes fabric processes Gateway Server hosting system live service master node partitions pay as you go service REST SQL services TDS and Windows Azure SQL DBAs SQL elastic pools SQL gallery image SQL HA deployment SQL injection attacks SQL Ledger SQL principal SQL searching methods SQL Server and Linux high availability SQL Server 2005 SQL Server 2019 SQL Server 2022 SQL Server 2022 Enterprise SQL Server Agent SQL Server Analysis Services (SSAS) SQL Server Audit SQL Server Command-Line Interfaces (CLIs) SQL Server Data Tools (SSDT) SQL Server endpoint SQL Server gallery image SQL Server IaaS Agent Extension Command Bar PowerShell registering security configure Microsoft Defender Service Menu settings backups best practices assessment configure high availability storage updates SQL virtual machine SQL Virtual Machine Resource Provider subscription SQL Server IaaS Agent Extension Settings SQL Server installations SQL Server instance features SQL Server Integration Services (SSIS) components SQL Server License SQL Server Management Studio (SSMS) SQL Server Replication SQL Server Resource Governor SQL server settings additional options automated backup automated patching Azure Key Vault integration database and transaction files instant file initialization lock pages in memory networking networking, security, and storage premium SSD disk types R Services (Advanced Analytics) SQL Authentication SQL Server instance settings SQL Server License storage configuration tempdb storage SQL Server Settings SQL Server tools SQL services SQL Threat Detection SQL Threat Protection SQL Virtual Machine Resource Provider SQL virtual machines SSAS SeeSQL Server Analysis Services (SSAS) SSIS SeeSQL Server Integration Services (SSIS) SSMS SeeSQL Server Management Studio (SSMS) SSDT SeeSQL Server Data Tools (SSDT) Stack dumps Stand-alone SQL Server Standby replica STB SeeServer Tools and Business (STB) Sterling Sterling (SAWAv2) architecture logical server noisy neighbor non-SSD drives Pottermore resource reservations SLO SQL Server Resource Governor TFS T-SQL surface area v12 Windows Azure admin portal Windows Fabric WinFab application SU SeeService Update (SU) Synapse Link Sysprep installed SQL Server System-assigned managed identity System_health XEvent session files T Tabular Data Stream (TDS) Tags TDS SeeTabular Data Stream (TDS) Team Foundation Services (TFS) Tempdb database Templates Temporary storage Tenant Tenants Terraform Text data TFS SeeTeam Foundation Services (TFS) Threat protection Tight budget TLS SeeTransport Layer Security (TLS) TLS connection encryption Transparent Data Encryption (TDE) enabled with SQL Server deployments Transport Layer Security (TLS) Troubleshooting common problems connectivity error diagnostics Resource Health support request Trusted launch virtual machines T-SQL T-SQL BULK INSERT T-SQL functions T-SQL surface area U Update domains User-assigned identity V v11 v12 Value-added services vCore vCore purchasing model Vector distance Vector store Verification queries DMV DMVs sanity check SQL Server Versionless SQL Server Virtual cluster Virtual Device Interface (VDI) APIs Virtual machines (VM) availability sets in Azure Blob Storage C&E classic concept data disks EC2 gallery images Gen2 Helsinki IaaS Opening up OS disk PAAS RDFE SLAs in SQL Azure SQL server SeeAzure VM STB Virtual network Virtual network peering Virtual network service endpoint Visual Studio Code Visual Studio Tools VM architecture VM bwsqlvmsrg VM role VM sizes VMware Vulnerability assessment W WFCS SeeWindows Failover Cluster Solution (WFCS) Windows Authentication Windows Azure Windows Fabric Windows Failover Cluster Solution (WFCS) “Windows in the cloud.” Windows Server Failover Clustering (WSFC) WinFab application Workbooks Worker limits Workload environment WSFC SeeWindows Server Failover Clustering (WSFC) X, Y xp_cmdshell Z Zone redundancy BC enabling GP Hyperscale Zone-redundant Storage (ZRS) ZRS SeeZone-redundant Storage (ZRS)