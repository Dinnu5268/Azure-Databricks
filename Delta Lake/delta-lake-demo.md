# Databricks Delta Lake - A Friendly Intro

This article introduces Databricks Delta Lake. A revolutionary storage layer that brings reliability and improve performance of data lakes using Apache Spark.

First, we'll go through the dry parts which explain what Apache Spark and data lakes are and it explains the issues faced with data lakes. Then it talks about Delta lake and how it solved these issues with a practical, easy-to-apply tutorial.

## Introduction to Apache Spark

If you don't know what Spark is, Apache Spark is a large-scale data processing and unified analytics engine for big data and machine learning. It was originally developed at UC Berkeley in 2009. Apache Spark is 100% open source, hosted at the vendor-independent Apache Software Foundation.

Apache Spark achieves high performance for both batch and streaming data, using a state-of-the-art DAG scheduler, a query optimizer, and a physical execution engine. Spark offers over 80 high-level operators that make it easy to build parallel apps, and you can use it interactively from the Scala, Python, R, and SQL shells.

Spark runs on Hadoop, Apache Mesos, Kubernetes, standalone, or in the cloud. It can access diverse data sources. You can run Spark using its standalone cluster mode, on EC2, on Hadoop YARN, on Mesos, or on Kubernetes. Access data in HDFS, Alluxio, Apache Cassandra, Apache HBase, Apache Hive, and hundreds of other data sources.

## Building data lakes

A data lake is a central location  that holds a large amount of data in its native, raw format, as well as a way to organize large volumes of highly diverse data. Compared to the hierarchical data warehouse which stores data in files or folders, a data lake uses a flat architecture to store the data.

A data lake holds big data from many sources in a raw format. It can store structured, semi-structured, or unstructured data, which data can be kept in a more flexible format so we can transform when used for analytics, data science & machine learning.

![Data-lake](https://user-images.githubusercontent.com/11095178/72041650-09489b00-32a4-11ea-9bfd-14feb2b508fe.jpg)

[Credits](https://databricks.com/glossary/data-lake)

## Problem with data lakes

Sadly, we don’t live in a perfect world. So, majority of data lake projects fail. What makes building data lakes a pain is, you guessed it, data.

Two problems face data engineers, machine learning engineers and data scientists when dealing with data: Reliability and Performance.
Why are these projects struggling with reliability and performance?

Unreliable, low-quality data leads to slow performance. Data in most cases is not ready for data science and machine learning, which is why data teams get busy building complex pipelines to process ingested data by partitioning, cleansing and wrangling to make it useful for model training and business analytics.

## Reliability issues

- **Failed jobs** leave data in corrupt state. This requires tedious data cleanup after failed jobs. Unfortunately, cloud storage solutions available don’t provide native support for atomic transactions which leads to incomplete and corrupt files on cloud can break queries and jobs reading from.

- **No schema enforcement** leads to data with inconsistent and low-quality structure. Mismatching data types between files or partitions cause transaction issues and going through workarounds to solve. Such workarounds are using string/varchar type for all fields, then to cast them to preferred data type when fetching data or applying OLAP (online analytical processing) transactions.

- **Lack of consistency** when mixing appends and reads or when both batching and streaming data to the same location. This is because cloud storage, unlike RDMS, is not ACID compliant.

## Performance issues

- **File size inconsistency** with either too small or too big files. Having too many files causes workers spending more time accessing, opening and closing files when reading which affects performance.

- **Partitioning,** while useful, can be a performance bottleneck when a query selects too many fields.

- **Slow read performance** of cloud storage compared to file system storage. Throughput for Cloud object/blob storage is between 20-50MB per second. Whereas local SSDs can reach 300MB per second.

Thus, comes Delta Lake, the next generation engine built on Apache Spark

## What is Delta Lake?

Delta Lake is an open source storage layer that brings reliability to data lakes. It provides ACID transactions, scalable metadata handling, and unifies streaming and batch data processing. Delta Lake is fully compatible with Apache Spark APIs. You can easily use it on top of your data lake with minimal changes, and yes, it’s open source! (Built on standard parquet)

Data lake brings both reliability and performance to data lakes.

## Performance key features

- **Compaction:** Delta Lake can improve the speed of read queries from a table by coalescing small files into larger ones. 
- **Data skipping:** When you write data into a Delta table, information is collected automatically. Delta Lake on Databricks takes advantage of this information (minimum and maximum values) to boost queries. You do not need to configure data skipping so the feature is activated (if applicable).
- **Caching:** Delta caching accelerates reads by creating copies of remote files in the nodes local storage using a fast intermediate data format. The data is cached automatically when a file is fetched from a remote source. Successive reads of the same data are performed locally, which results in significantly improved reading speed.

## Reliability key features

- **ACID transactions:** Finally! Serializable isolation levels ensure readers never see inconsistent data, same as RDMS. 
- **Schema enforcement:** Delta Lake automatically validates the data frame schema being written is compatible with table’s schema. Before writing from a data frame to a table, Delta Lake checks if the columns in the table exist in the data frame, columns’ data types match and column names cannot be different (even by case).
- **Data versioning:** The transaction log for a Delta table contains versioning information that supports Delta Lake evolution. Delta Lake tracks minimum reader and writer version separately.
- **Time travel:** Delta’s time travel capabilities simplify rollback and audit changes in data. Every operation to a Delta table or directory is automatically versioned. You may time travel using timestamp or version number.

Enough reading! Let’s see how Delta Lake works in practice.

We are going to use the notebook tutorial here provided by Databricks to exercise how can we use Delta Lake.we will create a standard table using Parquet format and run a quick query to observe its performance. Then, we create a Delta table, optimize it and run a second query using Databricks Delta version of the same table to see the performance difference. We will also look at the table history.

The data set used is for airline flights in 2008. It contains over 7 million records.

We will read the dataset which is originally of CSV format:

```
flights = spark.read.format("csv") \
  .option("header", "true") \
  .option("inferSchema", "true") \
  .load("/databricks-datasets/asa/airlines/2008.csv")
```

Then, we will create a table from sample data using Parquet:

```
# Write Parquet-based table from flights data
flights.write.format("parquet") \
  .mode("overwrite") \
  .partitionBy("Origin") \
  .save("/tmp/flights_parquet")
```

To test the performance of the parquet-based table, we will query the top 20 airlines with most flights in 2008 on Mondays by month.

```
from pyspark.sql.functions import count

flights_parquet = spark.read.format("parquet") \
  .load("/tmp/flights_parquet")

display(flights_parquet.filter("DayOfWeek = 1") \
        .groupBy("Month", "Origin") \
        .agg(count("*").alias("TotalFlights")) \
        .orderBy("TotalFlights", ascending=False) \
        .limit(20)
)
```

This query took me about 38.94 seconds with a cluster using `Standard_DS3_v2` machine type; 14GB memory with 4 cores, using 4-8 nodes.

Now, let's try Delta. We will create a Delta-based table using same dataset:

```
flights.write.format("delta") \
  .mode("append") \
  .partitionBy("Origin") \
  .save("/tmp/flights_delta")
  
# Create delta table
display(spark.sql("DROP TABLE IF EXISTS flights"))
display(spark.sql("CREATE TABLE flights USING DELTA LOCATION '/tmp/flights_delta'"))
```

Before we test the Delta table, we may optimize it using `ZORDER` by the column `DayofWeek`. This column is used to filter data when querying (Fetching all flights on Mondays):

```
display(spark.sql("OPTIMIZE flights ZORDER BY (DayofWeek)"))
```

Ok, now we can test the query's performance when using Databricks Delta:

```
flights_delta = spark.read \
  .format("delta") \
  .load("/tmp/flights_delta")

display(flights_delta.filter("DayOfWeek = 1").groupBy("Month","Origin").agg(count("*").alias("TotalFlights")).orderBy("TotalFlights", ascending=False).limit(20))
```

Running the query on Databricks Delta took 6.52 seconds only. That's about 5x faster!

## Taking a look at versioning in Delta

Using the `flights` table, we can browse all the changes to this table running the following:

```
display(spark.sql("DESCRIBE HISTORY flights"))
```

![databricks-delta-data-version](https://user-images.githubusercontent.com/11095178/72041487-9a6b4200-32a3-11ea-921f-ea2bd6795e1a.jpg)

You see two rows: The row with version `0` (lower row) shows the initial version when table is created. The row version `1` shows when the optimization step.

For fun, let’s try to use flights table version 0 which is prior to applying optimization on `DayOfWeek`. We’ll re-read the table’s data of version `0` and run the same query to test the performance:

```
flights_delta_version_0 = spark.read \
  .format("delta") \
  .option("versionAsOf", "0") \
  .load("/tmp/flights_delta")

display(flights_delta_version_0.filter("DayOfWeek = 1").groupBy("Month","Origin").agg(count("*").alias("TotalFlights")).orderBy("TotalFlights", ascending=False).limit(20))
```

The query took me 36.3 seconds to run using same cluster as before. This shows how optimizing Delta table is very crucial for performance.

Hope this article helps learning about Databricks Delta!

Sources:
- [Databricks Glossary - Delta Lake](https://databricks.com/glossary/data-lake)
- [Apache Spark](https://spark.apache.org/)
- [Databricks - Spark](https://databricks.com/spark/about)
- [Databricks - Delta](https://docs.databricks.com/delta/index.html)
- [Databricks - Optimization Tutorial](https://docs.databricks.com/delta/optimizations/optimization-examples.html#delta-lake-on-databricks-optimizations-python-notebook)
- [Databricks - Delta Lake Time Travel](https://databricks.com/blog/2019/02/04/introducing-delta-time-travel-for-large-scale-data-lakes.html)




