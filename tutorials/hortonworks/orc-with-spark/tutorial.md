---
layout: tutorial
title: Using Hive with ORC from Apache Spark
tutorial-id: 400
tutorial-series: Spark
tutorial-version: hdp-2.4.0
intro-page: true
components: [ spark, hive, orc ]
---


### Introduction

In this tutorial, we will explore how you can access and analyze data on Hive from Spark. In particular, you will learn:

*   How to interact with Apache Spark through an interactive Spark shell
*   How to read a text file from HDFS and create a RDD
*   How to interactively analyze a data set through a rich set of Spark API operations
*   How to create an Hive table in ORC File format
*   How to query an Hive table using Spark SQL
*   How to persist data in ORC file format

Spark SQL uses the Spark engine to execute SQL queries either on data sets persisted in HDFS or on existing RDDs. It allows you to manipulate data with SQL statements within a Spark program.

### Prerequisites

This tutorial is a part of series of hands-on tutorials to get you started with HDP using Hortonworks sandbox. Please ensure you complete the prerequisites before proceeding with this tutorial.

*   Downloaded and Installed latest [Hortonworks Sandbox](http://hortonworks.com/products/hortonworks-sandbox/#install)
*   [Learning the Ropes of the Hortonworks Sandbox](http://hortonworks.com/hadoop-tutorial/learning-the-ropes-of-the-hortonworks-sandbox/)

### Getting the dataset

To begin, login to Hortonworks Sandbox through SSH:

![](/assets/orc-with-spark/Screenshot_2015-04-13_07_58_43.png?dl=1)

The default password is `hadoop`.

Now let’s download the dataset with the command below:

~~~ bash
wget http://hortonassets.s3.amazonaws.com/tutorial/data/yahoo_stocks.csv
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2008.49.00.png?dl=1)

and copy the downloaded file to HDFS:

~~~ bash
hadoop fs -put ./yahoo_stocks.csv /tmp/
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2008.49.55.png?dl=1)

### Starting the Spark shell

Use the command below to launch the Scala REPL for Apache Spark:

~~~ bash
spark-shell --master yarn-client --driver-memory 512m --executor-memory 512m
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2008.53.08.png?dl=1)

Notice it is already starting with Hive integration as we have preconfigured it on the Hortonworks Sandbox.

Before we get started with the actual analytics let's import some of the libraries we are going to use below.

~~~ java
import org.apache.spark.sql.hive.orc._
import org.apache.spark.sql._
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-21%2011.43.56.png?dl=1)

### Creating HiveContext

HiveContext is an instance of the Spark SQL execution engine that integrates with data stored in Hive. The more basic SQLContext provides a subset of the Spark SQL support that does not depend on Hive. It reads the configuration for Hive from hive-site.xml on the classpath.

~~~ java
val hiveContext = new org.apache.spark.sql.hive.HiveContext(sc)
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-21%2011.47.33.png?dl=1)

### Creating ORC tables

ORC is a self-describing type-aware columnar file format designed for Hadoop workloads. It is optimized for large streaming reads and with integrated support for finding required rows fast. Storing data in a columnar format lets the reader read, decompress, and process only the values required for the current query. Because ORC files are type aware, the writer chooses the most appropriate encoding for the type and builds an internal index as the file is persisted.

Predicate pushdown uses those indexes to determine which stripes in a file need to be read for a particular query and the row indexes can narrow the search to a particular set of 10,000 rows. ORC supports the complete set of types in Hive, including the complex types: structs, lists, maps, and unions.

Specifying `as orc` at the end of the SQL statement below ensures that the Hive table is stored in the ORC format.

~~~ java
hiveContext.sql("create table yahoo_orc_table (date STRING, open_price FLOAT, high_price FLOAT, low_price FLOAT, close_price FLOAT, volume INT, adj_price FLOAT) stored as orc")
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2009.33.34.png?dl=1)

### Loading the file and creating a RDD

A **Resilient Distributed Dataset** (RDD), is an immutable collection of objects that is partitioned and distributed across multiple physical nodes of a YARN cluster and that can be operated in parallel.

Once an RDD is instantiated, you can apply a [series of operations](https://spark.apache.org/docs/1.2.0/programming-guide.html#rdd-operations). All operations fall into one of two types: [transformations](https://spark.apache.org/docs/1.2.0/programming-guide.html#transformations) or [actions](https://spark.apache.org/docs/1.2.0/programming-guide.html#actions). **Transformation** operations, as the name suggests, create new datasets from an existing RDD and build out the processing DAG that can then be applied on the partitioned dataset across the YARN cluster. An **Action** operation, on the other hand, executes DAG and returns a value.

Normally, we would have directly loaded the data in the ORC table we created above and then created an RDD from the same, but in this to cover a little more surface of Spark we will create an RDD directly from the CSV file on HDFS and then apply Schema on the RDD and write it back to the ORC table.

With the command below we instantiate an RDD:

~~~ java
val yahoo_stocks = sc.textFile("hdfs://sandbox.hortonworks.com:8020/tmp/yahoo_stocks.csv")
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-21%2012.08.16.png?dl=1)

To preview data in `yahoo_stocks` type:

~~~ java
yahoo_stocks.take(10)
~~~

Note that `take(10)` returns only ten records that are not in any particular order.

### Separating the header from the data

Let’s assign the first row of the RDD above to a new variable

~~~ java
val header = yahoo_stocks.first
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2010.14.21.png?dl=1)

Let’s dump this new RDD in the console to see what we have here:

~~~ java
header
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2010.22.10.png?dl=1)

Now we need to separate the data into a new RDD where we do not have the header above and :

~~~ java
val data = yahoo_stocks.filter(_ != header)
~~~

the first row to be seen is indeed only the data in the RDD

~~~ java
data.first
~~~

### Creating a schema

There’s two ways of doing this.

~~~ java
case class YahooStockPrice(date: String, open: Float, high: Float, low: Float, close: Float, volume: Integer, adjClose: Float)
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2011.54.06.png?dl=1)

### Attaching the schema to the parsed data

Create an RDD of Yahoo Stock Price objects and register it as a table.

~~~ java
val stockprice = data.map(_.split(",")).map(row => YahooStockPrice(row(0), row(1).trim.toFloat, row(2).trim.toFloat, row(3).trim.toFloat, row(4).trim.toFloat, row(5).trim.toInt, row(6).trim.toFloat)).toDF()
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2011.59.33.png?dl=1)

Let’s verify that the data has been correctly parsed by the statement above by dumping the first row of the RDD containing the parsed data:

~~~ java
stockprice.first
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2014.02.58.png?dl=1)

if we want to dump more all the rows, we can use

~~~ java
stockprice.show
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2014.08.33.png?dl=1)

To verify the schema, let’s dump the schema:

~~~ java
stockprice.printSchema
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2014.12.38.png?dl=1)

### Registering a temporary table

Now let’s give this RDD a name, so that we can use it in Spark SQL statements:

~~~ java
stockprice.registerTempTable("yahoo_stocks_temp")
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2014.19.30.png?dl=1)

### Querying against the table

Now that our schema’s RDD with data has a name, we can use Spark SQL commands to query it. Remember the table below is not a Hive table, it is just a RDD we are querying with SQL.

~~~ java
val results = sqlContext.sql("SELECT * FROM yahoo_stocks_temp")
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2016.24.14.png?dl=1)

The resultset returned from the Spark SQL query is now loaded in the `results` RDD. Let’s pretty print it out on the command line.

~~~ java
results.map(t => "Stock Entry: " + t.toString).collect().foreach(println)
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-21%2013.08.32.png?dl=1)

### Saving as an ORC file

Now let’s persist back the RDD into the Hive ORC table we created before.

~~~ java
results.write.format("orc").save("yahoo_stocks_orc")
~~~

To store results in a hive directory rather than user directory, use this path instead:

~~~ bash
/apps/hive/warehouse/yahoo_stocks_orc
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2016.52.44.png?dl=1)

### Reading the ORC file

Let’s now try to read back the ORC file, we just created back into an RDD. But before we do so, we need a `hiveContext`:

~~~ java
val hiveContext = new org.apache.spark.sql.hive.HiveContext(sc)
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2017.23.06.png?dl=1)

now we can try to read the ORC file with:

~~~ java
val yahoo_stocks_orc = hiveContext.read.format("orc").load("yahoo_stocks_orc")
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2017.24.05.png?dl=1)

Let’s register it as a temporary in-memory table mapped to the ORC table:

~~~ java
yahoo_stocks_orc.registerTempTable("orcTest")
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2017.24.53.png?dl=1)

Now we can verify whether we can query it back:

~~~ java
hiveContext.sql("SELECT * from orcTest").collect.foreach(println)
~~~

![](/assets/orc-with-spark/Screenshot%202015-05-28%2017.26.08.png?dl=1)

Voila! We just did a round trip of using Spark shell, reading data from HDFS, creating an Hive table in ORC format, querying the Hive Table, and persisting data using Spark SQL.

Hope this tutorial illustrated some of the ways you can integrate Hive and Spark.
