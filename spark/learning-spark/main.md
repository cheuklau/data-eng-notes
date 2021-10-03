# Learning Spark

## Table of Contents
* [Chapter 1: Introduction](#chapter-1-introduction)
* [Chapter 2: Getting Started](#chapter-2-getting-started)
* [Chapter 3: Structured APIs](#chapter-3-structured-apis)

## Chapter 1: Introduction

<details>
  <summary>Introduction</summary>

### Introduction

- Spark is designed for large-scale distributed data processing.
- Spark provides in-memory storage for intermediate computations.
- Spark incorporates libraries with APIs for machine learning (MLlib), SQL (Spark SQL), and stream processing.

</details>

<details>
  <summary>Spark's design philosophy</summary>

### Spark's design philosophy
- Speed
    + Takes advantage of multithreading and parallel processing.
    + Builds query computations as a DAG; DAG scheduler and query optimizer construct efficient computational graph that is highly parallelizable.
    + All intermediate results in memory, and limited disk I/O.
- Ease of use
    + Fundamental abstraction of a logical data structure as a Resilient Distributed Dataset (RDD).
    + Transformations and actions are operations that act on RDDs.
- Modularity
    + Support for multiple languages (Python, Java, Scala).
    + Well-documented APIs.
- Extensibility
    + Focuses on parallel computation engine rather than storage.
    + Spark can read data stored in a myriad of sources.

</details>

<details>
  <summary>Spark's distribution execution</summary>

### Spark's distribution execution

- Spark application consists of a driver that orchestrates parallel operations on the cluster (i.e., executors and cluster manager) through a `SparkSession`.
- Spark driver
    + Communicates with cluster manager, requesting resources (CPU, memory) for Spark's executors (JVMs).
    + Transforms all Spark operations into DAG computations, schedules them and distributes them as tasks across executors.
- SparkSession
    + Provides a single entry point to all of Spark's functionality.
    + Create using high-level API in different programming languages or created for you in a Spark shell.
- Cluster manager
    + Manages resources for cluster of nodes Spark runs on.
    + Currently supports standalone, Hadoop YARM, Mesos, Kubernetes
- Spark executor
    + Runs on each worker node
    + Communicates with driver and executes tasks on workers
- Distributed data and partitions
    + Physical data distributed across storage as partitions in HDFS or cloud storage e.g., S3
    + Spark treats each partition as a high-level logical data abstraction i.e., DataFrame in memory
    + Executors only process data closest to them (data locality)

</details>

## Chapter 2: Getting Started

<details>
  <summary>Downloading Spark</summary>

### Downloading Spark

- Use Spark shell to prototype Spark operations with small datasets
- Then write complex Spark application for large datasets
- Downloaded [Spark](https://www.apache.org/dyn/closer.lua/spark/spark-3.1.2/spark-3.1.2-bin-hadoop2.7.tgz)

</details>

<details>
  <summary>Spark's Directories and Files</summary>

### Spark Directories and Files

- Files in the tar:
    + `bin` contains scripts to interact with Spark e.g., Spark shells
    + `sbin` contains administrative scripts for starting/stopping Spark components in the cluster
    + `kubernetes` contains Dockerfiles for creating Docker images for Spark distribution on a Kubernetes cluster
    + `data` contains text files that are input for Spark's components e.g., MLLib, Structured Streaming and GraphX
- Start PySpark from `bin`
    + Every computation expressed in high-level API is decomposed into low-level optimized and generated RDD operations which are converted to Scala bytecode for executor's JVMs
- Key concepts of Spark application
    + Application
        * User program built on Spark using its APIs
        * Driver program and executors on cluster
    + SparkSession
        * Point of entry to interact with underlying Spark functionality and allows programming Spark with its APIs
        * In a Spark shell, Spark driver instantiates it for you
    + Job
        * Parallel computation made of multiple tasks that get created in response to a Spark action
    + Stage
        * Each job gets divided into smaller sets of tasks called stages
    + Task
        * Unit of work or execution that will be sent to oSpark executor

</details>

<details>
  <summary>Understanding Spark Application Concepts</summary>

### Understanding Spark Application Concepts

- Driver converts Spark application into one or more Spark jobs
- Each job is transformed into a DAG
- Each node in a DAG is a single or multiple Spark stages
- Stages created based on what operations can be performed serially or in parallel
- Each stage is made of Spark tasks which are federated across each Spark executor
- Each task maps to a single core and works on a single partition of data

</details>

<details>
  <summary>Transformation, Actions and Lazy Evaluations</summary>

### Transformation, Actions and Lazy Evaluations

- Spark operations classified as transformations and actions
- Transformations transform a dataframe into a new dataframe without altering original data
    + Example: `select()`, `filter()`
    + Transformations are not done immediately, instead they are performed lazily (recorded) until an action is invoked
- Actions include `show()`, `take()`, `count()`, `collect()`

</details>

<details>
  <summary>Narrow and Wide Transformations</summary>

### Narrow and Wide Transformations

- Any transformation where a single output partition is computed from a single input partition is a narrow transformation e.g., `filter()`, `contains()`
- Wide transformations require data from other partitions to be read in, combined and written to disk e.g., `groupBy()`, `orderBy()`

</details>

<details>
  <summary>Spark UI</summary>

### Spark UI

- Driver launches a Spark UI running on port 4040
- View scheduler stages and tasks
- Summary of RDD sizes and memory usage
- Information about environment and executors
- All Spark SQL queries

</details>

<details>
  <summary>Your First Standalone Application</summary>

### Your First Standalone Application

```python
# Import SparkSession and related functions from the PySpark module.
import sys
from pyspark.sql import SparkSession
from pyspark.sql.functions import count

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: mnmcount <file>", file=sys.stderr) sys.exit(-1)

# Build a SparkSession using the SparkSession APIs.
# If one does not exist, then create an instance.
# There can only be one SparkSession per JVM.
spark = (SparkSession
     .builder
     .appName("PythonMnMCount")
     .getOrCreate())

# Get the M&M data set filename from the command-line arguments
mnm_file = sys.argv[1]

# Read file into a Spark DataFrame using the CSV
# format by inferring the schema and specifying that the
# file contains a header, which provides column names.
mnm_df = (spark.read.format("csv")
     .option("header", "true")
     .option("inferSchema", "true")
     .load(mnm_file))

# We use the DataFrame high-level APIs. Note
# that we don't use RDDs at all. Because some of Spark's
# functions return the same object, we can chain function calls.
# 1. Select from the DataFrame the fields "State", "Color", and "Count"
# 2. Since we want to group each state and its M&M color count,
# we use groupBy()
# 3. Aggregate counts of all colors and groupBy() State and Color
# 4 orderBy() in descending order
count_mnm_df = (mnm_df
     .select("State", "Color", "Count")
     .groupBy("State", "Color")
     .agg(count("Count").alias("Total"))
     .orderBy("Total", ascending=False))

# Show the resulting aggregations for all the states and colors;
# a total count of each color per state.
# Note show() is an action, which will trigger the above
# query to be executed.
count_mnm_df.show(n=60, truncate=False)
print("Total Rows = %d" % (count_mnm_df.count()))

# While the above code aggregated and counted for all
# the states, what if we just want to see the data for
# a single state, e.g., CA?
# 1. Select from all rows in the DataFrame
# 2. Filter only CA state
# 3. groupBy() State and Color as we did above
# 4. Aggregate the counts for each color
# 5. orderBy() in descending order
# Find the aggregate count for California by filtering
ca_count_mnm_df = (mnm_df
     .select("State", "Color", "Count")
     .where(mnm_df.State == "CA")
     .groupBy("State", "Color")
     .agg(count("Count").alias("Total"))
     .orderBy("Total", ascending=False))

# Show the resulting aggregation for California.
# As above, show() is an action that will trigger the execution of the
# entire computation.
ca_count_mnm_df.show(n=10, truncate=False)

# Stop the SparkSession
spark.stop()
```
- To run the above example:
```bash
$SPARK_HOME/bin/spark-submit mmmcount.py data/mmm_dataset.csv
```

</details>

## Chapter 3: Structured APIs

<details>
  <summary>What's Underneath an RDD</summary>

### What's Underneath an RDD

- Three RDD characteristics:
1. Dependencies
    * Instructs Spark how an RDD is constructed with its inputs.
    * Allows Spark to recreate RDDs (resiliency).
2. Partitions
    * Spark can split the work to parallelize computation across executors.
3. Compute function
    * Produces an `Iterator[T]` for the data stored in the RDD.

</details>

<details>
  <summary>Structuring Spark</summary>

### Structuring Spark

- Key schemes for structuring Spark starting in 2.x:
1. Express computations with common data analysis patterns e.g., filtering, selecting, counting, etc.
2. Set of common operators in a DSL to tell Spark what to compute with your data, allowing Spark to construct an efficient query plan for execution.
3. Allow arrangement of data in a tabular format similar to SQL table or spreadsheet with supported structured data types

### Key Benefits

- Above structure allows Spark to increase expressivity and composability
- Compare low-level RDD API for the following:
```python
# Create an RDD of tuples (name, age)
dataRDD = sc.parallelize([("Brooke", 20), ("Denny", 31), ("Jules", 30), ("TD", 35), ("Brooke", 25)])
# Use map and reduceByKey transformations with their lambda
# expressions to aggregate and then compute average
agesRDD = (dataRDD.map(lambda x: (x[0], (x[1], 1)))
                  .reduceByKey(lambda x, y: (x[0] + y[0], x[1] + y[1]))
                  .map(lambda x: (x[0], x[1][0]/x[1][1])))
```
- To using DSL ooperators and the DataFrame API:
```python
# In Python
from pyspark.sql import SparkSession
from pyspark.sql.functions import avg
# Create a DataFrame using SparkSession
spark = (SparkSession.builder
                     .appName("AuthorsAges")
                     .getOrCreate())
# Create a DataFrame
data_df = spark.createDataFrame([("Brooke", 20), ("Denny", 31), ("Jules", 30), ("TD", 35), ("Brooke", 25)], ["name", "age"])
# Group the same names together, aggregate their ages, and compute an average
avg_df = data_df.groupBy("name").agg(avg("age"))
# Show the results of the final execution
avg_df.show()
```
- Note that you can switch back to the unstructured low-level RDD API if you need more control about how computations are performed.
- The simplicity and expressivity we observe is because of the Spark SQL engine that the high-level structured APIs are built on.

</details>

<details>
  <summary>The DataFrame API</summary>

### The DataFrame API

- Inspired by Pandas DataFrames.
- Distributed in-memory tables with named columns and schemas.
- Each column has a specific data type e.g., integer, string, array, map, etc.
- DataFrames are immutable and Spark keeps a lineage of all transformations

### Spark's Basic Data Types

- Spark supports basic internal data types of its supported programming languages.
- For example, Python `float` is Spark data type `FloatType`.

### Spark's Structured and Complex Data Types

- Data will often be complex e.g., maps, arrays, structs, dates, timestamps, etc.
- For example, Python `dict` is Spark data type `MapType` which can be instantiated with `MapType(keyType, valueType, [nullable])`.

</details>

<details>
  <summary>Schemas and Creating DataFrames</summary>

### Schemas and Creating DataFrames

- Schema in Spark defines the column names and associated data types for a DataFrame.
- Defining a schema up-front as opposed to a schema-on-read approach has benefits:
1. Relieves Spark from having to infer data types.
2. Prevents Spark from creating a separate job to read a large portion of data to ascertain the schema.
3. Detect errors early if data doesn't match the schema.

### Two Ways to Define a Schema

- First way is to define it programmatically. For example:
```python
from pyspark.sql.types import *
schema = StructType([StructField("author", StringType(), False),
      StructField("title", StringType(), False),
      StructField("pages", IntegerType(), False)])
```
- Second way is to employ a Data Definition Language (DDL) string. For example:
```python
schema = "author STRING, title STRING, pages INT"
```
- Example of creating a DataFrame for a given schema:
```python
from pyspark.sql import SparkSession

# Define schema for our data using DDL
schema = "`Id` INT, `First` STRING, `Last` STRING, `Url` STRING, `Published` STRING, `Hits` INT, `Campaigns` ARRAY<STRING>"

# Create our static data
data = [[1, "Jules", "Damji", "https://tinyurl.1", "1/4/2016", 4535, ["twitter", "LinkedIn"]],
        [2, "Brooke","Wenig", "https://tinyurl.2", "5/5/2018", 8908, ["twitter", "LinkedIn"]],
        [3, "Denny", "Lee", "https://tinyurl.3", "6/7/2019", 7659, ["web", "twitter", "FB", "LinkedIn"]],
        [4, "Tathagata", "Das", "https://tinyurl.4", "5/12/2018", 10568, ["twitter", "FB"]],
        [5, "Matei","Zaharia", "https://tinyurl.5", "5/14/2014", 40578, ["web", "twitter", "FB", "LinkedIn"]],
        [6, "Reynold", "Xin", "https://tinyurl.6", "3/2/2015", 25568, ["twitter", "LinkedIn"]]]

# Main program
if __name__ == "__main__":
    # Create a SparkSession
    spark = (SparkSession
         .builder
         .appName("Example-3_6")
         .getOrCreate())
    # Create a DataFrame using the schema defined above
    blogs_df = spark.createDataFrame(data, schema)
    # Show the DataFrame; it should reflect our table above
    blogs_df.show()
    # Print the schema used by Spark to process the DataFrame
    print(blogs_df.printSchema())
```

</details>
