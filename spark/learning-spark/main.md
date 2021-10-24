# Learning Spark

## Table of Contents
* [Chapter 1: Introduction](#chapter-1-introduction)
* [Chapter 2: Getting Started](#chapter-2-getting-started)
* [Chapter 3: Structured APIs](#chapter-3-structured-apis)
* [Chapter 4: Spark SQL and DataFrames](#chapter-4-spark-sql-and-dataframes)

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

### Columns and Expressions

- In Spark's supported languages, columns are objects with public methods.
- Column objects can't exist in isolation; each column is part of a row in a record, and all the rows together constitute a DataFrame.

### Rows

- A row in Spark is a Row object containing one or more columns.
- Row is an ordered collection of fields which can be accessed by index starting at 0.

</details>

<details>
  <summary>Common DataFrame Operations</summary>

### Common DataFrame Operations

- DataFrameReader allows you to read data into a DataFrame from different sources e.g., JSON, CSV, Parequet, Avro, etc.
- DataFrameWriter allows you to write a DataGrame back to a data source.
- Example:
```python
from pyspark.sql.types import *

# Programmatic way to define a schema
fire_schema = StructType([StructField('CallNumber', IntegerType(), True),
                StructField('UnitID', StringType(), True),
                StructField('IncidentNumber', IntegerType(), True),
                StructField('CallType', StringType(), True),
                StructField('CallDate', StringType(), True),
                StructField('WatchDate', StringType(), True),
                StructField('CallFinalDisposition', StringType(), True),
                StructField('AvailableDtTm', StringType(), True),
                StructField('Address', StringType(), True),
                StructField('City', StringType(), True),
                StructField('Zipcode', IntegerType(), True),
                StructField('Battalion', StringType(), True),
                StructField('StationArea', StringType(), True),
                StructField('Box', StringType(), True),
                StructField('OriginalPriority', StringType(), True),
                StructField('Priority', StringType(), True),
                StructField('FinalPriority', IntegerType(), True),
                StructField('ALSUnit', BooleanType(), True),
                StructField('CallTypeGroup', StringType(), True),
                StructField('NumAlarms', IntegerType(), True),
                StructField('UnitType', StringType(), True),
                StructField('UnitSequenceInCallDispatch', IntegerType(), True),
                StructField('FirePreventionDistrict', StringType(), True),
                StructField('SupervisorDistrict', StringType(), True),
                StructField('Neighborhood', StringType(), True),
                StructField('Location', StringType(), True),
                StructField('RowID', StringType(), True),
                StructField('Delay', FloatType(), True)])

# Use the DataFrameReader interface to read a CSV file
sf_fire_file = "/databricks-datasets/learning-spark-v2/sf-fire/sf-fire-calls.csv"
fire_df = spark.read.csv(sf_fire_file, header=True, schema=fire_schema)
```
- `spark.read.csv()` reads in CSV file and returns a DataFrame of rows and named columns with the types dictated in the schema.
- Parquet is the default format for `DataFrameWriter`.

### Saving a DataFrame as a Parquet File or SQL Table

- Example of persisting a DataFrame into a Parquet file or SQL table:
```python
// Save as a Parquet file
parquet_path = ...
fire_df.write.format("parquet").save(parquet_path)

parquet_table = ... # name of the table
fire_df.write.format("parquet").saveAsTable(parquet_table)
```

### Transformations and Actions

- Once DataFrame is read into memory we can perform transformations and actions on it.

### Projection and Filters

- Projection in relational parlance returns only rows matching a certain relational condition by filters.
    * `select()` used for projections.
    * `filter()` or `where()` used for filters.
- Example:
```python
few_fire_df = (fire_df
      .select("IncidentNumber", "AvailableDtTm", "CallType")
      .where(col("CallType") != "Medical Incident"))
few_fire_df.show(5, truncate=False)
```

### Renaming, Adding and Dropping Columns

- Example renaming columns:
```python
new_fire_df = fire_df.withColumnRenamed("Delay", "ResponseDelayedinMins")
(new_fire_df
    .select("ResponseDelayedinMins")
    .where(col("ResponseDelayedinMins") > 5)
    .show(5, False))
```
- We can transform column types with methods such as `to_timestamp()`, `to_date()`, etc.

### Aggregations

- Handful of transformations aggregate by column names and then aggregate counts across them e.g., `groupBy()`, `orderBy()`, `count()`, etc.

### Other Common DataFrame Operations

- DataFrame API provids statisical methods e.g., `min()`, `max()`, `sum()`, `avg()`, etc.

</details>

<details>
  <summary>DataFrames Versus Datasets</summary>

### DataFrames Versus Datasets

- If you require relational transformations similar to SQL-like queries, use DataFrames.
- Use Datasets if you want to take advantage of Tungsten's efficient serialization with Encoders.
- Use DataFrames if you want unification, code optimization and simplification of APIs across Spark components.
- For space and efficiency use DataFrames.

### When to Use RDDs

- When using a third-party package written using RDDs.
- Want to precisely instruct Spark how to do a query.

</details>

<details>
  <summary>Spark SQL and the Underlying Engine</summary>

### Spark SQL and the Underlying Engine

- Spark SQL allows developers to issue SQL compatible queries on structured data with a schema.
- Additionally, Spark SQL engine:
    * Unifies Spark components and permits abstractions to DataFrame/Datasets in multiple languages.
    * Reads and writes structured data with a specific schema from structured file formats (JSON, CSV, Text, Avro, Parquet), and converts data into temporary tables.
    * Interactive Spark SQL shell for quick data exploration.
    * Connects to external tools via standard database JDBC/ODBC connectors.
    * Generates optimized query plans for the JVM.

### The Catalyst Optimizer

- Takes a computational query and converts it into an execution plan in four phases:
    1. Analysis
        * Spark SQL engine generates an abstract syntax tree for the DataFrame query.
    2. Logical optimization
        * Uses a rules based optimization approach to construct a set of plans.
    3. Physical planning
        * Generates an optimal physical plan for the above selected logical plan.
    4. Code generation
        * Generates efficient Java bytecode to run on each machine.
        * Project Tungsten facilitates whole-stage code generation which coollapses the query into a single function, eliminating virtual function calls and employing CPU registers for intermediate data.
        * This improves CPU efficiency and performance.
- To view the execution plan for a Python DataFrame: `dataframe.explain(True)`

</details>

## Chapter 4: Spark SQL and DataFrames

<details>
  <summary>Using Spark SQL in Spark Applications</summary>

### Using Spark SQL in Spark Applications

- Use `SparkSession` to access Spark functionality.
- Use `sql()` method to issue any SQL query which will return a DataFrame.
- Example of using a schema to read data into a DataFrame and register DataFrame as a temporary view to query it with SQL.
```python
from pyspark.sql import SparkSession # Create a SparkSession
spark = (SparkSession
      .builder
      .appName("SparkSQLExampleApp")
      .getOrCreate())
# Path to data set
csv_file = "/databricks-datasets/learning-spark-v2/flights/departuredelays.csv"
# Read and create a temporary view
# Infer schema (note that for larger files you
# may want to specify the schema)
df = (spark.read.format("csv")
      .option("inferSchema", "true")
      .option("header", "true")
      .load(csv_file))
df.createOrReplaceTempView("us_delay_flights_tbl")

spark.sql("""SELECT distance, origin, destination
    FROM us_delay_flights_tbl WHERE distance > 1000
    ORDER BY distance DESC""").show(10)
```

</details>

<details>
  <summary>SQL Tables and Views</summary>

### SQL Tables and Views

- Each table in Spark has metadata e.g., schema, description, name, column names, partitions, etc.
- All this is stored in a central metastore.
- Spark by default uses Hive metastore in `/usr/hive/warehouse`

</details>

<details>
  <summary>Managed Versus Unmanaged Tables</summary>

### Managed Versus Unmanaged Tables

- For managed tables, Spark manages metadata and data in the file store e.g., local filesystem, HDSFS, S3.
- For unmanaged tables, Spark only manages the metadata and you manage the data in external data sources e.g., Cassandra.
    * If you do a `DROP TABLE`, only the table is deleted from metadata.

</details>

<details>
  <summary>Creating SQL Databases and Tables</summary>

### Creating SQL Databases and Tables

- Spark creates tables in `default` database by default.
- Create a database:
```python
spark.sql("CREATE DATABASE learn_spark_db")
spark.sql("USE learn_spark_db")
```
- Create a managed table:
```python
spark.sql("CREATE TABLE managed_us_Delay_flight_tbl (date STRING, delay INT, distance INT, origin STRING, destination STRING)")
```
- Create an unmanaged table from a data source:
```python
spark.sql("""CREATE TABLE us_delay_flights_tbl(date STRING, delay INT,
      distance INT, origin STRING, destination STRING)
      USING csv OPTIONS (PATH
      '/databricks-datasets/learning-spark-v2/flights/departuredelays.csv')""")
```

</details>

<details>
  <summary>Creating Views</summary>

### Creating Views

- Spark can create views on top of tables.
- They can be global or session-scoped, and are temporary (gone after Spark application terminates).
- Example:
```python
df_sfo = spark.sql("SELECT date, delay, origin, destination FROM us_delay_flights_tbl WHERE origin = 'SFO'")
df_jfk = spark.sql("SELECT date, delay, origin, destination FROM us_delay_flights_tbl WHERE origin = 'JFK'")
# Create a temporary and global temporary view
df_sfo.createOrReplaceGlobalTempView("us_origin_airport_SFO_global_tmp_view")
df_jfk.createOrReplaceTempView("us_origin_airport_JFK_tmp_view")
# Perform queries
spark.sql("SELECT * FROM us_origin_airport_JFK_tmp_view")
# Drop views
spark.catalog.dropGlobalTempView("us_origin_airport_SFO_global_tmp_view")
spark.catalog.dropTempView("us_origin_airport_JFK_tmp_view")
```
- Spark manages metadata of managed and unmanaged tables, captured in the Catalog.
- Catalog is a high-level abstraction in Spark SQL for storing metadata.
- Example:
```python
spark.catalog.listDatabases()
spark.catalog.listTables()
spark.catalog.listColumns("us_delay_flights_tbl")
```

</details>

<details>
  <summary>Data Sources for DataFrames and SQL Tables</summary>

### Data Sources for DataFrames and SQL Tables

#### DataFrameReader

- `DataFrameReader` is the core construct for reading data from a data source into DataFrame.
- Usage:
```
DataFrameReader.format(args).option("key", "value").schema(args).load()
```
- To get an instance handle of `DataFrameReader`, use `SparkSession.read` or `SparkSession.readStream`.
- `format()`
    * Arguments include `parquet`, `csv`, `txt`, `json`, `jdbc`, `orc`, `avro`, etc.
- `option()`
    * Arguments include `mode`, `inferSchema`, `path`.
- `schema()`
    * Argument is a DDL string or struct e.g., `A INT, B STRING`.
- `load`
    * Argument is `/path/to/data/source`
- Parquet is the default and preferred data source for Spark because it is efficient, uses columnar storage, and employs a fast compression algorithm.

#### DataFrameWriter

- `DataFrameWriter` writes data to a specified built-in data source.
- Usages:
```
DataFrameWriter.format(args)
      .option(args)
      .bucketBy(args)
      .partitionBy(args)
      .save(path)

DataFrameWriter.format(args).option(args).sortBy(args).saveAsTable(table)
```
- To get an instance handle: `DataFrame.write` or `DataFrane.writeStream`.
- `format()`
    * Arguments include `parquet`, `csv`, `txt`, `json`, `jdbc`, `orc`, `avro`, etc.
- `option()`
    * Arguments include `mode`, `path`.
- `bucketBy()`
    * Arguments include `numBuckets`, and names of columns to bucket by.
- `save()`
    * Arguments include `/path/to/data/source`.
- `saveAsTable()`
    * Arguments include `table_name`.

#### Parquet

- Default data source in Spark.
- Open-soource columnar file format offering many I/O optimizations.
- Recommend to save DataFrames in Parquet for downstream applicatioons.
- Parquet files stored in directory structure containing data files, metadata, compressed files and status files.
- To read Parquet files into a DataFrame:
```python
file = """/databricks-datasets/learning-spark-v2/flights/summary-data/parquet/
      2010-summary.parquet/"""
df = spark.read.format("parquet").load(file)
```
- Note no need to supply schema since Parquet saves it as part of metadata.
- Can also create a Spark SQL unmanaged table/view using SQL:
```sql
CREATE OR REPLACE TEMPORARY VIEW us_delay_flights_tbl USING parquet
OPTIONS (
path "/databricks-datasets/learning-spark-v2/flights/summary-data/parquet/ 2010-summary.parquet/" )
```
- Then read data into a DataFrame using SQL:
```python
spark.sql("SELECT * FROM us_delay_flights_tbl").show()
```
- To write a DataFrame in Parquet:
```python
(df.write.format("parquet")
      .mode("overwrite")
      .option("compression", "snappy")
      .save("/tmp/data/parquet/df_parquet"))
```

#### JSON

- Single-line and multiline mode both supported in Spark.
- In single-line mode, each line is a JSON object.
- In multiline mode, the entire multiline oobject is a JSON object.
- To read JSON into a DataFrame:
```python
file = "/databricks-datasets/learning-spark-v2/flights/summary-data/json/*"
df = spark.read.format("json").load(file)
```
- To save a DataFrane as a JSON file:
```python
(df.write.format("json")
      .mode("overwrite")
      .option("compression", "snappy")
      .save("/tmp/data/json/df_json")))
```

#### Other data source types

- Other data source types covered include CSV, Avro, ORC, images (for deep learning frameworks),

</detail>

## Chapter 5: Interacting with External Data Souorces