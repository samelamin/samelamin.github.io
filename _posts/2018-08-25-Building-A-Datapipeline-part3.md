---
layout: post
title: "Building A Data Pipeline Using Apache Spark. Part 3"
description: ""
category:
tags: [Apache Spark, End to End Pipeline, Testing, Data, Pipeline, S3]
---

# Building A Scalable And Reliable Data Pipeline. Part 3
This is the long overdue third chapter on building a data pipeline using Apache Spark.

While there are a multitude of tutorials on how to build Spark applications, in my humble opinion there are not enough out there for the major gotchas and pains you feel while building them!

## Why the long break?
The reason being in the past 10 months so much has happened to me on a personal and professional front. For starters I moved jobs, moved house and most importantly I welcomed the birth of my first child! It has been a hectic few months and I only am now getting back to contributing to the community


## Where we left off?
In the last [post](https://samelamin.github.io/2017/04/27/Building-A-Datapipeline-part1/) we looked at the second part of building the data pipeline which was the transformations  and why naming is important


In this chapter we will be looking at how I like to split my pipeline, code structure and the underlying foundation of every single ETL job


## The Foundations

Although creating a data pipeline comes with a variety of complexity, when  it boils down to it you really are running a procedure and you want to know when it fails(alerting), why(logs) and what action (e.g retry) to take.

During my career I have seen a lot of bad practices and anti-patterns that try and solve all these issues in one code base. From rerunning in a for loop, to abusing the [multi-processing](https://docs.python.org/3.4/library/multiprocessing.html?highlight=process) package even implementing dependency checks in a convoluted manner as a form of a [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph)

This makes things very difficult to manage for even one engineer, but imagine if you had a team of 6 trying to to maintain it. That is why I always push for a separation of concerns We move the scheduling, retrying, alerting to a different tool/technology such as Airflow and let spark do what id does best which is handle data processing. That is at its core what the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single_responsibility_principle) advocates.

This is the architecture I am trying to implement here, our pipeline is essentially a collection of classes that all inherit from class `job` and each job has a corresponding `transform` associated with it

These jobs will be run in parallel in Spark clusters using simple Spark Submit commands submitted via a scheduler. You can use a variety of schedulers the simplest being a cron job to the more complex schedulers like Airflow and Luigi, my preference of course is Airflow


### Why do we need a transform

Some of you might think what's the point of the transform class, we can just call the job and do whatever transformations you are doing there.

While you certainly will make your code less complex you do compromise on one very important aspect which is testability. The transformation in ETL is the key part to the business and the most complex, for example how you calculate sales is crucial to show insight how well a company is doing, so inserting a bug in here can be very costly, especially if business decisions are then taken off them (this point is important and we will come back to it later in the series)



How I make my spark code testable is a technique that engineers have been using for decades. It is simply to move the transformation into its own method. We pass in all the dependencies in the job to the transformation class. So say we need 4 sources to generate one table. Our transformation class will take in these 4 data frames and return the one df.

This allows us to generate sample data off the first 4 data frames, pass it to the transformation and assert on the result. We can then ensure all the key transformations we made such as sum of sales or number of active accounts, churn or any other key metrics

A note to remember is that this tests only the transformation, so you know the logic you ran is correct, but in the big data world this does not guarantee the table in production will be correct. Any number of issues can arise such as a missing dependency or the data within the dependency is incorrect

We will cover the three levels of testing in the next chapter

## How it fits Together
Now that we have a collection of jobs and respective transforms we need to connect all of them together, by separating the scheduling of the job with the processing of the job we are free to choose what tools we want to do that as well as alerting, back-offs and retries


Now all we need to do is package the code up in a jar or an egg if you are in the python world and spark submit that class, this enables us to create our DAG the way we want and its very very simple to know exactly what the code is doing


In summary, following the SRP makes the code more testable and more importantly managable which leads to fewer bugs and makes all involved a whole lot happier!

Remember that the very best code is not complex or convoluted but simple enough for anyone regardless of their technical ability to follow


That is it for part 3, in the next we will cover testing in the big data world and how we go about ensuring every single deploy is risk free ( within reason )


 As always, if you have any feedback or ideas you want to share please do not hesitate to get in touch!
