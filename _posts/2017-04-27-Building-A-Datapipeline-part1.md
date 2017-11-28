---
layout: post
title: "Building A Data Pipeline Using Apache Spark. Part 1"
description: ""
category:
tags: [Apache Spark, End to End Pipeline, Testing, Data, Pipeline, S3]
---

# Building A Scalable And Reliable Data Pipeline. Part 1
This post was inspired by a call I had with some of the Spark community user group on testing. If you havent watch it then you will be happy to know that it was recorded, you can watch it [here](https://www.youtube.com/watch?v=2q0uAldCQ8M), there are some amazing ideas and thoughts being shared.

While there are a multitude of tutorials on how to build Spark applications, in my humble opinion there are not enough out there for the major gotchas and pains you feel when building them and we are in a unique industry where we learn from our failures. No other industry pushes their workers to fail. You do not see a pilot who keeps crashing on take off stay in work for long! While in software our motto for a long time has been to fail fast! 

Additionally, a data pipeline is not just one or multiple spark application, its also workflow manager that handles scheduling, failures, retries and backfilling to name just a few.

Finally a data pipeline is also a data serving layer, for example Redshift, Cassandra, Presto or Hive.

This is why I am hoping to build a series of posts explaining how I am currently building data pipelines, the series aims to constract a data pipeline from scratch all the way to a productionalised pipeline. Including a workflow manager and a dataserving layer.

We will also explore the pains and frustrations of dealing with third party data providers and I hope the community will share their ideas and horror stories which will in turn give knowledge sharing and help us build better pipelines! 


You can find the entire code base [here](https://github.com/samelamin/spark-wordcount-bdd)

*I have been lucky enough to build this pipeline with the amazing [Marius Feteanu](https://github.com/mariusfeteanu) who you can contact [here](mailto:marius.feteanu@gmail.com) *

#### Please bare in mind these snippets are to give you a better understanding. Please do not use it as production code.



## Tests Tests Tests! 
As a strong advocate of test first development, I am starting the series on what inspired the orginal hangout call which is testing! 

There are a variety of testing tools for Spark. While you can always just create an in-memory spark Context, I am a lazy developer and laziness is a virtue for a developer! 

There are some frameworks to avoid writing boiler plate code, some of them are listed below (If I missed any please give me a shout and I will add them)
 
 Scala:
  - [Spark Test Base](https://github.com/holdenk/spark-testing-base)
  - [Spark Build Examples](https://github.com/datastax/SparkBuildExamples)
  - [Databricks Integration Tests](https://github.com/databricks/spark-integration-tests)
 
Pyspark:
  - [Spark Test Base](https://github.com/holdenk/spark-testing-base)
  - [Pyspark Testing](https://github.com/msukmanowsky/pyspark-testing)
  - [Pytest - Spark](https://github.com/malexer/pytest-spark)
  - [DummyRdd](https://github.com/wdm0006/DummyRDD)
 
 
 I am personally using Scala in my data pipeline and for that I am using Holdens Spark Test Base. I prefer writing my tests in a BDD manner. If you are in Pyspark world sadly Holden's test base wont work so I suggest you check out Pytest and [pytest-bdd](https://github.com/pytest-dev/pytest-bdd). 
 
 Now it was highlighted in the call that like myself a lot of engineers focuss on the code so below is an example of writing a simple word count test in Scala
 
 ``` scala
package com.bddsample

import com.holdenkarau.spark.testing.DataFrameSuiteBase
import org.scalatest.Matchers._
import org.scalatest.{FeatureSpec, GivenWhenThen}
import scala.util.Random

import org.apache.spark.rdd.RDD
import org.apache.spark.sql.types.{DateType, DecimalType, StructType}

class WordCountSpecs extends FeatureSpec with GivenWhenThen with DataFrameSuiteBase {
  feature("Word Count") {
    scenario("When dealing with rdds") {

      Given("A list of words")
      val input = List("hi", "hi holden", "bye")


      When("Parralising the RDD")
      val rdd = tokenize(sc.parallelize(input)).collect().toList

      Then("We should get the expected output")
      val expected = List(List("hi"), List("hi", "holden"), List("bye"))
      rdd should be (expected)

    }
    def tokenize(f: RDD[String]) = {
      f.map(_.split(" ").toList)
    }
  }
}
```


## Raw Data
So now that we have some tests what is the first step of implementing a data pipeline? Ingesting raw data ofcourse, practically every single ETL job I had to write had to either ingest CSV,TSV, or JSON. This can either be dropped in traditionally to an SFTP server but more recently into cloud storage like Amazon S3. I also had to ingest JSON data from an API endpoint


While the task itself isnt difficult, there are various scenarios that can make it so, e.g.:
 - What happens if the data never arrives? What do you display to the end users? An empty table or tell them no data arrived? 
 
 
 - The data arrives but it is in the incorrect format, either there are new columns/fields or missing fields
 
 
 - For CSVs, Sometimes the data comes with headers sometimes it does not


 - How do you deal with backfills, i.e. your data provider says "remember that data we sent you days or months ago, that's all rubbish data heres the correct one" (For now.......)
 
 
 - The data arrives again but fields have incorrect values, you expected an int but got a string or the ever more annoying the assortment of timestamps you receive. 
 
 
 To tackle these issues what we do is the very first step in our pipeline is to clean the data and standardize it so it is easier for us to manage. For example dealing with GBs of data in RAW CSV and JSON is extremly difficult so we need to transform it to a format that is more managable like [PARQUET](https://parquet.apache.org/) or [AVRO](https://avro.apache.org/docs/1.2.0/). We currently use PARQUET
 
 
 The biggest pain when dealing with enterprise data warehouses like Redshift is that compute and storage are tied together. 
 
 So you would have issues when a data scientist is trying to run a monster query that is hogging all the resources while your data analyst is also trying to do their regular queries, while you also have dashboards being used by you're business users AND your ETL process that are currently running throughout the day.
 
 We wanted to split compute from storage to fix this exact problem. So we implemented a data lake in Amazon S3. 
 Below are 2 diagrams of our architecture at a high level 
 
 ![PLATFORM](https://raw.githubusercontent.com/samelamin/samelamin.github.io/master/img/platform.png "PLATFORM")

 ![DATA LAKE](https://raw.githubusercontent.com/samelamin/samelamin.github.io/master/img/datalake.png "DATA LAKE")

 

 *I will be touching more on these as the series eveolve but for now we are going to focus only on the RAW ingestion.*

 We ship our code with the schema versions we expect. This is to ensure the processed data will be in the shape we expect, and allow us to then do transformation queries to produce the data models our customers (data analysts,scientists and end users) expect. 
 
 So lets say we have data coming in JSON format and below is an example of such data:
 
 
  ``` json
{
	"customers": [{
			"id": 1,
			"email": "abc1@abc.com"
		},
		{
			"id": 2,
			"email": "abc2@abc.com"
		},
		{
			"id": 3,
			"email": "abc3@abc.com"
		}
	]
}
 ``` 

we add a resource to the code to say that we expect the V1 schema to be 

  ```json
{
  "type" : "struct",
  "fields" : [ {
    "name" : "customers",
    "type" : {
      "type" : "array",
      "elementType" : {
        "type" : "struct",
        "fields" : [ {
          "name" : "email",
          "type" : "string",
          "nullable" : true,
          "metadata" : { }
        }, {
          "name" : "id",
          "type" : "long",
          "nullable" : true,
          "metadata" : { }
        } ]
      },
      "containsNull" : true
    },
    "nullable" : true,
    "metadata" : { }
  } ]
}

 ``` 


We then read in the JSON while applying the schema like so: 

```scala

val actualInputVersion = 1
val jobName = "SampleJSONRawExtract"
val json_schema_string = Source.fromInputStream(transform.getClass.getResourceAsStream(s"${jobName}/$actualInputVersion.json")).mkString
val schema = DataType.fromJson(json_schema_string) match {case s:StructType => s}

val df = spark.read
	.schema(schema)
	.json(pathToJSONFile)

df.write.parquet(pathToWriteParquetTo)

``` 
  
  
  
We then save our processed file back down to our data lake

And of course we have to ensure that any changes to the code do not miss any data so below is a sample test to ensure the number of rows stay the same after processing

  
```scala
package com.bddsample
import com.holdenkarau.spark.testing.DataFrameSuiteBase
import org.apache.spark.sql.SaveMode
import org.scalatest.Matchers._
import org.scalatest.{FeatureSpec, GivenWhenThen}
import org.apache.spark.sql.functions.{explode,col}

class RawExtractTests extends FeatureSpec with GivenWhenThen with DataFrameSuiteBase {
  feature("Raw Extract") {
    scenario("When reading raw data") {

      Given("Some Json Data")
      val sampleJson =
        """
          |{
          |	"customers": [{
          |			"id": 1,
          |			"email": "abc1@abc.com"
          |		},
          |		{
          |			"id": 2,
          |			"email": "abc2@abc.com"
          |		},
          |		{
          |			"id": 3,
          |			"email": "abc3@abc.com"
          |		}
          |	]
          |}
        """.stripMargin

      val df = sqlContext.read.json(sc.parallelize(Seq(sampleJson)))
        .withColumn("customers",explode(col("customers")))


      When("We Save To Parquet")
      val pathToWriteParquetTo = "output.parquet"
      df.write.mode(SaveMode.Overwrite).parquet(pathToWriteParquetTo)

      Then("We should clean and standardize the output to parquet")
      val expectedParquet = spark.read.parquet(pathToWriteParquetTo)

      Then("We should have the correct number of rows")
      val actualRowCount = expectedParquet.select("customers").count()
      actualRowCount should not be 0
      actualRowCount should be (3)
    }
  }
}
```   
  
  
 So to summarise the very first step we do is extract the raw data, ensure the data is in the right format by applying a schema over it and then save it back to parquet.
 
 
 To make it simple to manage the data we partition by day so as an example we save the file back to 
 
 ``output/rawdata/{YEAR}/{MONTH}/{DAY}/{VERSION}/{JOB.parquet}``
 

 That is it for part 1, if you have any feedback please do not hesitate to get in touch! 
