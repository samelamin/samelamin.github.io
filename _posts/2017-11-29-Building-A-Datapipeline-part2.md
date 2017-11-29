---
layout: post
title: "Building A Data Pipeline Using Apache Spark. Part 2"
description: ""
category:
tags: [Apache Spark, End to End Pipeline, Testing, Data, Pipeline, S3]
---

# Building A Scalable And Reliable Data Pipeline. Part 2

This is a much belated second chapter on building a data pipeline using Apache Spark, while there are a multitude of tutorials on how to build Spark applications, in my humble opinion there are not enough out there for the major gotchas and pains you feel when building them and we are in a unique industry where we learn from our failures. No other industry pushes their workers to fail. You do not see a pilot who keeps crashing on take off stay in work for long! While in software our motto for a long time has been to fail fast! 

In the last [post](https://samelamin.github.io/2017/04/27/Building-A-Datapipeline-part1/) we looked at the first part of building the data pipeline which was the raw ingestion and saving to parquet a more managemeble format for storing and paritioning data. 


In this chapter we will be looking at the next stage of the pipeline process, which is transforming and shaping the data to a fact [table](https://en.wikipedia.org/wiki/Fact_table) with the right [grain](https://www.ibm.com/support/knowledgecenter/en/SS9UM9_8.5.0/com.ibm.datatools.dimensional.ui.doc/topics/c_dm_design_cycle_2_idgrain.html). 


## Transformations!
So assuming we now have the RAW data in a managable format, we now need to convert it to the domain of the business. For example say you have data coming from your payment provider which tells you about the number of orders you have made as well as any transaction references and order ids. 

You have no control over what the provider calls an Order, it could be called "Transaction_ref" or "order_id" while the customer id can be called "customer_ref" or "transaction_origin_id". These terms might not make sense to your end customer - the business analyst or data scientiest - your job as a data engineer is to transform that data and shape it in the terms that will understand. This becomes evidently more important when you reach the next stage of joining multiple transformed tables to create your fact table. For anyone doing the code, be it a data engineer or an analyst trying to explore the transformed tables, Its easier to do the joins when the columns actually relate to business terms that are used day by day

For those of you coming from a tranditional software engineering background, this is also refered to as the [ubiquitous language](https://martinfowler.com/bliki/UbiquitousLanguage.html)



So to help clarify these things, imagine we have these two schemas RAW on S3


The first is a table of customers, lets call this Dataframe 1 (DF1)


```
	{
		"type": "struct",
		"fields": [{
			"name": "email",
			"type": "string",
			"nullable": true,
			"metadata": {}
		}, {
			"name": "customer_id",
			"type": "long",
			"nullable": true,
			"metadata": {}
		}, {
			"name": "order_id",
			"type": "string",
			"nullable": true,
			"metadata": {}
		}]
	}
 ``` 
 
 And the second dataframe is data coming from our payment provider, as a list of payments made
 
 
 
```
	{
		"type": "struct",
		"fields": [{
			"name": "customer_ref",
			"type": "string",
			"nullable": false,
			"metadata": {}
		},{
			"name": "transaction_id",
			"type": "string",
			"nullable": false,
			"metadata": {}
		}, {
			"name": "transaction_ref",
			"type": "string",
			"nullable": false,
			"metadata": {}
		},{
			"name": "payment_method",
			"type": "string",
			"nullable": false,
			"metadata": {}
		},{
			"name": "transaction_timestamp",
			"type": "timestamp",
			"nullable": false,
			"metadata": {}
		}, {
			"name": "amount",
			"type": "long",
			"nullable": false,
			"metadata": {}
		}]
	}
 ``` 



In this simplistic example we can have an educated guess that "transaction_ref" is the order id and "customer_ref" is the email,we can see that we can create the fact table by simply joining these two

While this seems realativly straight forward, in the real world this gets far more complicated when you are given over 20+ fields and need to combine 7 or more data sources for the same fact table.




The way I personally structure my job is to make it easier to test. To do this I have a SparkJob for each class I want, then each class has a respective transformation class, this gets two or more dataframes then does whatever transformations it needs to do and saves it back to S3

By seperating the transformation from the actual class it makes it much easier to test because then in our tests we can pass in sample data in a certain shape then run our transformations and assert on the resulting shape. There are multiple levels of testing I usually implement, but this is the most basic and easiest to start with


so an example Transform class can be


```scala
object OrdersTransform  extends Transform {
  // name of table
  override val name: String = "orders"
  // schema version
  override val version: Int = 1

  def executeNew(orders: DataFrame, payments: DataFrame): DataFrame = {
    val factTable = orders
      .join(payments, col("transaction_ref") === col("order_id"), "left_outer") // Equi-join
      .select(
        col("order_id"),
        col("transaction_timestamp").alias("order_timestamp"),
        col("transaction_id").alias("payment_transaction_ref"),
        col("amount").alias("order_amount"),
        col("payment_method").alias("card_type"),
        col("customer_id")
      )
    .withColumn("dwh_insert_date", now_utc_udf()) // The first time I have seen this row
    .withColumn("dwh_update_date", now_utc_udf()) // The last time I have updated this row
    checkOutput(factTable)
  }
```

The above code is a simple join over the 2 input dataframes, we then check the output to ensure the number of columns produced is in the right schema, correct order of columns. We check column order because of weird issues we faced with Presto where new columns being added in different versions of schema caused strange abnormalities. For example returning the incorrect column during a select of a particular column

You can find more info [here](https://github.com/prestodb/presto/issues/8911). However there is a fix in a setting now so you shouldnt need to do it. However for our own peace of mind we left it there



The above code is easily testable, and we can get the actual class to call it via the below code

```scala
object OrdersJob extends SnapshotJob {
    override protected val transform = OrdersTransform

    override def loadNew(spark: SparkSession, appConfig: Config):DataFrame = {
        val ordersExtractPath = s"${appConfig.dataRoot.get}/$OUTPUT_DIR/${OrdersExtract.namePath}/${appConfig.runDatePath}"
        val orders = load(spark, ordersExtractPath, OrdersExtractTransform.version)

        val paymentsPath = s"${appConfig.dataRoot.get}/$OUTPUT_DIR/${PaymentsExtract.namePath}/${appConfig.runDatePath}"
        val payments = load(spark, paymentsPath, PaymentsExtractTransform.version)

        transform.executeNew(orders, payments)
    }
}

```
The classes for "OrdersExtract" and appConfig are just the RAW classes and config classes that are added to define the jobs run date and hence where the data is stored in S3

Essentially, all the class is doing is reading the RAW files from S3 and passing it to the transform class, so this logic should be as basic and simple as possible and leave the complicated business logic in the heavily testing transform class

I will discuss the overall code structure and the architecture in the next section in the hopes to clarify the difference between a Job and Transform Class. 

So to summarise the very second step we do is read in all the RAW parquet files we ingested and do whatever transformations we need to do then save it back as parquet to S3 partiioned by day
 
As an example we save the file back to 
 
 `output/dwh/{YEAR}/{MONTH}/{DAY}/{VERSION}/{JOB.parquet}`
 
That is it for part 2, if you have any feedback or ideas you want to share please do not hesitate to get in touch or comment below! 
