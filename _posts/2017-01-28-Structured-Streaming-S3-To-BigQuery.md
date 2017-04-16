---
layout: post
title: "Spark Structured Streaming from S3 to BigQuery"
description: ""
category:
tags: [Streaming, Apache Spark 2, Structured Streaming, S3]
---

#### Apache Spark 2.0 Arrives!
Apache Spark 2.0 adds the first version of a new higher-level API, Structured Streaming, for building [continuous applications.](https://databricks.com/blog/2016/07/28/continuous-applications-evolving-streaming-in-apache-spark-2-0.html) The main goal is to make it easier to build end-to-end streaming applications, which integrate with storage, serving systems, and batch jobs in a consistent and fault-tolerant way.

There are a variety of streaming technologies out there, from [Apache Kafka](https://kafka.apache.org/) to [Kinesis](https://aws.amazon.com/kinesis/streams/) and even [Google Cloud Dataflow](https://cloud.google.com/dataflow/)

Spark aims to work alongside these technologies as an engine for large scale data processing.

#### Google BigQuery
I have spoken about Google BigQuery in a previous [post](http://samelamin.github.io/2016/11/28/Is-The-DBA-Role-Dead.html) and I am using it quite heavily in my current project and I have to say I am yet to find a massive issue with it. Every single query I ran - regardless of the dataset sizes (GB/TB of data) - returns in seconds

My one annoyance is that you cant partition by anything other than the day the data was inserted and that can become a problem because you are paying per query.

#### Amazon S3
Amazon S3 is a great storage tool, more and more companies are using it as a [data lake](https://aws.amazon.com/blogs/big-data/introducing-the-data-lake-solution-on-aws/). In the industry vernacular, a Data Lake is a massive storage and processing subsystem capable of absorbing large volumes of structured and unstructured data and processing a multitude of concurrent analysis jobs

#### Gluing Them All Together
As a data engineer my customers are the analysts, and my job is to empower them to get insight from a variety of data sources. Unfortunately most analysts are not comfortable writing python code, they are far more comfortable using SQL.

I wanted the ability to stream any new data coming into an S3 bucket to be transferred directly to BigQuery to enable analysts to query it via SQL regardless of the data structure and get results immediately

As a lazy engineer I do not want to spend ages writing code, whether to set up data extraction or identifying and defining schemas or even ensuring fault tolerance. Structured streaming seemed perfect for this as spark allows me to infer the schema from the provided data whether its JSON, CSV or Parquet if we are really lucky!

I would stream these datasets into BigQuery tables and step back and let the analysts do their magic!

####  Too Good To Be True

I was all ready to push data to Google BQ only to realize that because structured streaming is very new there are virtually no connectors out there.

So I did what any lazy engineer would do, I wrote [one](https://github.com/samelamin/spark-bigquery)!

#### Setting Up
I am still in the process of uploading my artifact to Sonatype so for now you will have to build it from source using:

```
sbt clean assembly
```

Once its built and referenced in your project you can easily read a stream, currently the only sources that Spark Structured Streaming support are S3 and HDFS. To create a stream off S3:

```
import com.samelamin.spark.bigquery._

val df = spark.readStream.json("s3a://bucket")
df.writeStream
      .option("checkpointLocation", "s3a://checkpoint/dir")
      .option("tableSpec","my-project:my_dataset.my_table")
      .format("com.samelamin.spark.bigquery")
      .start()
```

What we are doing here is creating a stream but also setting the checkpoint directory to somewhere in S3, this enables fault tolerance. In the event that the stream fails when you relaunch it, it will pick up where it left off

Now any new file that gets added to that bucket path gets appended to the BigQuery Table

Thats pretty much it folks, Feedback is of course always welcome and if you want to get involved feel free to send PRs!

Finally, a huge thank you to [Holden Karau](https://twitter.com/holdenkarau), whom without her support I wouldn't have been able to write this connector in the first place!

If you want to test your spark jobs, please do check out Holden's [Spark Test Base](https://github.com/holdenk/spark-testing-base). It was a life saver for me!
