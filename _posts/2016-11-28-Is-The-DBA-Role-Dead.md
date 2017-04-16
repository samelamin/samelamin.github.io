---
layout: post
title: "Is The DBA Role Dead?"
description: ""
category:
tags: []
---

#### Warning!!

I just want to start off by saying this piece is not meant to be trolling despite the impertinent title. 
Our industry is filled with hundreds of thousands of DBAs that are integral to their business and their value cannot be understated. 
I just wish to explain my views on what I think we are heading as an industry in the big data world

#### What is a DBA?
First of all, lets define what we mean by DBA. 
According to the source off all knowledge that is good and true - [wikipiedia](https://en.wikipedia.org/wiki/Database_administrator) - Database administrators (DBAs) use specialized software to store and organize data
The role may include capacity planning, installation, configuration, database design, migration, performance monitoring, security, troubleshooting, as well as backup and data recovery.
In lamen terms, a DBA is someone you hire to ensure your database is healthy,available, querable in a timely (defined by you or your business) manner and ensures there are always backups in case of disaster

#### What is Big Data and what is the fuss all about?
The new millennium saw the rise of Web 2.0 applications, which revolved around user-generated content. 
The Internet went from hosting static content to content that was dynamic, with the end user in the driving seat. 
In a matter of months, social networks, photo sharing, media streaming, blogs, wikis, and their ilk became ubiquitous. This resulted in an explosion in the amount of data on the Internet. 
To even store this data, let alone process it, an entirely different new of computing, dubbed warehouse-scale computing was needed.


#### Ok then, shouldnt this be the best time to be a DBA?
While you would think that is true, I think its not. Heres why

Traditionally you would have a DBA or a team of them to ensure your data is always available and expect them to work alongside your engineers and analysts to ensure your system is always up and running. Also the DBAs were gate keepers of this database or databases, this means any change goes through their eyes because at the end of the day they are ultimatly responsible. I would also put my hand up and say as an engineer we have historically been bad at making a mess of the database. I am sure any engineer can recall a time where they were faced with a table with a column called "column 0" or "column 1" and no one knows what that is used for but cannot be deleted in case something critical breaks. A DBA is there to stop things like that happening.

While I think there are countless systems out there in a variety of industries that have and will continue to require a DBA, more and more companies are utilising the power of the cloud to offload it and pay for that expertise. 

The boom of NOSQL databases have made people favour [BASE over ACID](http://www.dataversity.net/acid-vs-base-the-shifting-ph-of-database-transaction-processing/),and store entire objects rather than having relational tables. 

I must stress that I do not think that having high performance is the end goal if you cant even trust your data. I certainly recommend watching the [MongoDb is Web Scale](https://www.youtube.com/watch?v=b2F-DItXtZs)

What I am seeing is companies paying to push those problems to the cloud. 

This is the closest parallel I can think of this is - not exactly a perfect fit but the closest I can think of anyway - the slow decline of network administrators or the operations role. We used to hire administrators to spin up machines and monitor them for us. They would work with engineers to ensure that thats machines were always healthy and serving traffic, they would be responsible for security patches and alerting. With the rise of Devops and immutable infrastructre the adaptable engineers had to learn new skills, how to script the infrastructure and learn new tools and technologies to help them do that. 

The ones that didnt adapt slowly find them selves far less employable than they once were. That is the nature of our business we have to constantly be learning to be at the top of our game

DBAs are no different, and like I said it isnt a perfect parallel but you can start seeing what I mean. 
 
Backups and monitoring? Cloud providers such as AWS, Google and Azure are giving us services where we can script a database and they take care of all the things like back up and monitoring that have been within the domain of a DBA. 

The query is running to slow? Well then you can pay per query using the [Azure Datawarehouse](https://azure.microsoft.com/en-gb/services/sql-data-warehouse/) and [Google Big Query](https://cloud.google.com/bigquery/) so no matter what the size of the data they garuntee that you will get a correct result within seconds

So we are basically paying for the convience of not having to worry about our database and its a price I see more and more companies willing to pay for, because at the end of the day its cheaper than having your own DBA and also you wont ever employ someone who knows about database administration more than the big powerhouses, its what they do every single day. It is the same for someone trying to set up their in house infrastructure, its cheaper and easier to just go with the cloud

Now engineers don’t have to worry about things like defragmentation, index rebuilds, and data file space, let alone infrastructure components like disk, RAID, Ubuntu kernel versions, etc. Whole classes of problems are transitioned to the cloud provider.

#### Wrapping up
In summary, DBAs might not be dead but the role certainly is evolving, and those who dont start expanding their skillset might be in danger

I think the days of hiring an expert to do your DB admin are on the decline. I will think we are still a way from the career being completly dead giving the multitude of companies that need it
But if someone comes to me and says "Hi Sam I am thinking of a career of a DBA" I would say there might be better options out there


