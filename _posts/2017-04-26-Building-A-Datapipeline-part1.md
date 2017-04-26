---
layout: post
title: "Building a data pipeline Part 1"
description: ""
category:
tags: [Apache Spark, Testing, Data, Pipeline, S3]
---

# Building A Scalable And Reliable Data Pipeline
This post was inspired by a call I had with some of the spark community user group on testing. If you havent watch it then you will be happy to know that it was recorded, you can watch it [here](https://www.youtube.com/watch?v=2q0uAldCQ8M), there are some amazing ideas and thoughts being shared.


While there are a multitude of tutorials on how to build Apache Spark applications, in my humble opinion there are not enough knowledge out there for the major gotchas and pains you feel when building them, and we are in a unique industry where we learn from our failures. No other industry pushes their workers to fail. You do not see a pilot who keeps crashing on take off stay in work for long! While in software our motto for a long time has been to fail fast! 

This is why I am hoping to build a series of posts explaining how I am currently building data pipelines, the pains and frustrations of dealing with third party data providers and I hope the community will share their ideas and horror stories which will in turn give knowledge sharing and help us build better pipelines! 


## Tests Tests Tests! 
As a strong advocate of test first development, I am starting the series on what inspired the orginal hangout call which is testing! 

There are a variety of testing tools for Spark. While you can always just create an inmemory Spark Context. But I am a lazy engineer and laziness is a virtue for a developer! There are some frameworks to help you build them, some of them are listed below (If I missed any please give me a shout and I will add them)
 
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

You can find the entire code base [here](https://github.com/samelamin/spark-wordcount-bdd)


## Raw Data
So now that we have some tests what is the first step of implementing a data pipeline? Ingesting raw data ofcourse, practically every single ETL job I had to write had to either ingest CSV,TSV, or JSON. This can either be dropped in traditionally to an SFTP server but more recently into cloud storage like Amazon S3. I also had to ingest JSON data from an API endpoint


While the task itself isnt difficult, there are various scenarios that can make it so, e.g.:
 - What happens if the data never arrives? What do you display to the end users? An empty table or tell them no data arrived? 
 
 
 - The data arrives but it is in the incorret format, either there are new columns/fields or missing fields
 
 
 - For CSVs, Sometimes the data comes with headers sometimes it does not


 - How do you deal with backfills, i.e. you're data provider says remember that data we sent you days or months ago, that's all rubbish data heres the correct one (For now.......)
 
 
 - How do you deal with backfills, i.e. you're data provider says remember that data we sent you days or months ago, that's all rubbish data heres the correct one (For now.......)
 
 
 - The data arrives again but fields have incorrect values, you expected an int but got a string or the ever more annoying the assortment of timestamps you receive. 
 
 
 To tackle these issues what we do is the very first step in our pipeline is to clean the data and standerdize it so it is easier for us to manage. Forexample dealing with Gbs of data in RAW CSV and JSON is extremly difficult so we need to transform it to a format that is more managable like [PARQUET](https://parquet.apache.org/) or [AVRO](https://avro.apache.org/docs/1.2.0/). We currently use PARQUET
 
 ### Please bare in mind this snippits are to to give you an understanding. Please do not use it as production code.
 
 The biggest pain when dealing with enterprise data warehouses like Redshift is that computing and storage are tied together. 
 
 So you would have issues when a data scientist is trying to run a monster query that is hogging all the resources while you're data analyst is also trying to do their regular queries, while you also have dashboards being used by you're business users AND your ETL process that are currently running throughout the day.
 
 We wanted to split computing from storage to fix this exact problem. So we implemented a data lake in Amazon S3. 
 Below os a high level architecture of what the data lake looks like
 
 
 ![DATA LAKE](https://github.com/samelamin/samelamin.github.io/img/datalake.png "DATA LAKE")

 

 We ship our code with the schema versions we expect. So lets say we have data coming in CSV format and below is an example of the data expect
 
 
  ``` csv
expected_column,another_one
abc,123
def,456
 ``` 

we add a resource to the code to say that we expect the V1 schema to be 

  ```csv
{
  "type" : "struct",
  "fields" : [ {
    "name" : "expected_column",
    "type" : "string",
    "nullable" : false,
    "metadata" : { }
  }, {
    "name" : "another_one",
    "type" : "int",
    "nullable" : false,
    "metadata" : { }
  }
  ]
}

 ``` 


We then read in the csv while applying the schema like so: 

  ```scala
  
   val actualInputVersion = 1
   val jobName = "SampleCSVRawExtract"
   val csvHeader = True
   val json_schema_string = Source.fromInputStream(transform.getClass.getResourceAsStream(s"${jobName}/$actualInputVersion.json")).mkString
   val schema = DataType.fromJson(json_schema_string) match {case s:StructType => s}
	
   val df = spark.read
        .option("header", csvHeader)
        .option("delimiter", csvDelimiter)
        .schema(schema)
        .csv(pathToCSVFile)
		
    df.write.parquet(pathToWriteParquetTo)
    
  }

  ``` 
  
  
  We then save our processed file back down to our data lake
  
  
  
  
 So to summarise the very first step we do is extract the raw data, ensure the data is in the right format by applying a schema over it and then save it back to parquet.
 
 
 
 That is it for part 1, if you have any feedback please do not hesitate to get in touch! 
 
 
