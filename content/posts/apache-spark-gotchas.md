---
title: "Apache Spark Gotchas"
date: 2023-01-18T18:19:19+01:00
author: "Aymane BOUMAAZA"
draft: false
tags:
 - "apache"
 - "spark"
 - "gotchas"
 - "tips"
 - "big data"
 - "data"
---

Apache Spark is an open-source, distributed computing system that provides a wide range of tools for data processing, analytics, and machine learning. It's a popular choice for many organizations due to its ability to scale and its support for a wide range of programming languages. However, like any complex system, there are a few gotchas that users should be aware of when working with Spark. In this post, I will cover some of the most common gotchas that I learned while working with Spark, it will also be a sort of summary of some concepts I read about in the **Learning Spark: Lightning-Fast Big Data Analysis book** (**[link](https://pages.databricks.com/rs/094-YMS-629/images/LearningSpark2.0.pdf)**).

# Lazy Evaluation

Spark uses a technique called **lazy evaluation** to optimize the processing of RDDs. This means that [transformations](https://spark.apache.org/docs/latest/rdd-programming-guide.html#transformations) on RDDs are not actually executed until an [action](https://spark.apache.org/docs/latest/rdd-programming-guide.html#actions) (e.g., show, count, collect, take) is called on them. This can lead to confusion, as it is not always clear when a transformation will be executed. For example, when timing a transformation, it is important to note that the transformation will not be executed until an action is called, so there is no point in timing the transformation instruction itself.

```scala
val bigDf = ...
val transformed = bigDf.filter(...).map(...) // this transformation will not be executed yet (lazy evaluation)
transformed.count() // this action will trigger the execution of the transformation and will take more time than the transformation itself
```

# Joins


## Join strategies
There are two main strategies for performing joins in Spark: broadcast joins and shuffle joins. The strategy used depends (not only) on the size of the dataframes being joined, and Spark will automatically choose the best strategy. However, it is important to understand how each strategy works to avoid unexpected behavior.

### Broadcast joins

Broadcast joins are used when one of the dataframes being joined is small enough to fit in **memory** ([10MB by default](https://spark.apache.org/docs/latest/sql-performance-tuning.html#other-configuration-options:~:text=1.3.0-,spark.sql.autoBroadcastJoinThreshold,-10485760%20(10%20MB))). Spark will **send a copy of the small dataframe to all nodes in the cluster** (hence the name **broadast** join), and then perform the join locally on each node. This is generally faster than a shuffle join, but it can lead to out of memory errors if the dataframe is too large.
{{< figure src="/img/broadcast_join.png" align="center" alt="Broadcast Join" caption="Broadcast Join" border="#f8f4f0" >}}

### Shuffle joins

In shuffle joins, Spark will **shuffle the dataframes partitions between nodes**, to ensure that all data with the same key is on the same node. This is generally slower than a broadcast join because of all the shuffling that occurs, but it can be used to join dataframes of any size.
{{< figure src="/img/shuffle_join.png" align="center" alt="Shuffle Join" caption="Shulffle Join" border="#f8f4f0" >}}

> See: [Spark Join Strategies - How & What?](https://towardsdatascience.com/strategies-of-spark-join-c0e7b4572bcf) and [The art of joining in Spark](https://towardsdatascience.com/the-art-of-joining-in-spark-dcbd33d693c)


## Duplicate columns after joins

An issue that I faced multiple times when working with Spark is having duplicate columns after performing a join between two dataframes having the same join column name.

```scala
val left = Seq(("John", "Doe", 29),
            ("Aymane", "Boumaaza", 20),
            ("Jane", "Doe", 29)
        ).toDF("firstname", "lastname", "age")

// +---------+--------+---+
// |firstname|lastname|age|
// +---------+--------+---+
// |     John|     Doe| 29|
// |   Aymane|Boumaaza| 20|
// |     Jane|     Doe| 29|
// +---------+--------+---+

val right = Seq(("John"), ("Aymane"), ("Jane")).toDF("firstname")

// +---------+
// |firstname|
// +---------+
// |     John|
// |   Aymane|
// |     Jane|
// +---------+

val joined = left.join(right, left("firstname") === right("firstname"))

// +---------+--------+---+---------+
// |firstname|lastname|age|firstname|   // Notice the duplicate column (firstname)
// +---------+--------+---+---------+
// |     John|     Doe| 29|     John|
// |   Aymane|Boumaaza| 20|   Aymane|
// |     Jane|     Doe| 29|     Jane|
// +---------+--------+---+---------+
```

When performing a join in Spark, use a list/sequence as the `by` parameter to avoid duplicate columns to ensure that the resulting dataframe only has a single copy of each column.

```scala
val joined = left.join(right, Seq("firstname"))
// or val joined = left.join(right, "firstname") if using only one column to join

// +---------+--------+---+
// |firstname|lastname|age|
// +---------+--------+---+
// |     John|     Doe| 29|
// |   Aymane|Boumaaza| 20|
// |     Jane|     Doe| 29|
// +---------+--------+---+
```

> See: [Databricks - Prevent duplicated columns when joining two DataFrames](https://kb.databricks.com/en_US/data/join-two-dataframes-duplicated-columns)

# Writing to files

## Filenames

When writing to file in Spark, it is important to note that the output will be written to a **directory, not a single file**. This is because each partition is written to a separate file, and the files are combined into a single directory. It is important to keep this in mind when specifying the output path, as adding a `.csv` extension to the path will create a directory with the name `filename.csv` rather than a single file, **therefore there's no need to add the extension (since it's a directory)**.

```scala
val df = Seq(("John", "Doe", "29"),
            ("Aymane", "Boumaaza", "20"),
            ("Jane", "Doe", "29")
        ).toDF("firstname", "lastname", "age")

df.write.csv("/tmp/my_csv.csv")
df.write.parquet("/tmp/my_parquet.parquet")
```

`/tmp/my_csv.csv` and `/tmp/my_parquet.parquet` will be both **directories** containing the actual `csv` and `parquet` files.

```console
$ ls -l /tmp
drwxr-xr-x  2 root root   4096 Jan  4 20:20 my_csv.csv
drwxr-xr-x  2 root root   4096 Jan  4 20:20 my_parquet.parquet
```

## Number of partitions

When writing to file in Apache Spark, it's important to consider the number of partitions in your data. **The more partitions you have, the more files are created**. This can be beneficial if you are working with a large dataset and want to parallelize the writing process. However, if you have a small dataset, using too many partitions can lead to unnecessary overhead and slower performance, and vice versa when working with large dataframes and having small number of partitions will result in less files but bigger ones. It is important to carefully consider the size and structure of your dataset when deciding on the number of partitions to use.

### Small DataFrame

Minimize the number of partitions to avoid unnecessary overhead.

```scala
val smallDf = Seq(("John", "Doe", "29"),
            ("Aymane", "Boumaaza", "20"),
            ("Jane", "Doe", "29")
        ).toDF("firstname", "lastname", "age")

smallDf.rdd.partitions.size // Int = 3

smallDf.write.csv("/tmp/output")
// /tmp/output will be a directory containing 3 (number of partitions) csv files

smallDf.coalesce(1).write.csv("/tmp/output") // or repartition(1)
// /tmp/output will be a directory containing a single csv file (with other files)
```

### Big DataFrame

Use a reasonable number of partitions to have manageable output.

```scala
val bigDf = ...

bigDf.rdd.partitions.size // Int = 200

bigDf.write.csv("/tmp/output")
// /tmp/output.csv will be a directory containing 200 (number of partitions) csv files

bigDf.coalesce(10).write.csv("/tmp/output") // or repartition(10)
// /tmp/output.csv will be a directory containing 10 csv files which is more manageable than 200 files
```

## Write Options

When saving a dataframe to file in Spark, it is important to consider the various options available to you. For example, the `header` option controls whether the column names are included in the output `csv` file. By default, this option is set to `false`, which means that header containing the column names will not be included. It is important to carefully consider the options available to you when writing to file to ensure that the output is in the desired format.

```scala
val df = Seq(("John", "Doe", "29"),
            ("Aymane", "Boumaaza", "20"),
            ("Jane", "Doe", "29")
        ).toDF("firstname", "lastname", "age")

df.write.csv("/tmp/output") // header is false by default

df.write.option("header",true).option("delimiter","|").csv("/tmp/output")

```
> See [Spark SQL - Data Source](https://spark.apache.org/docs/latest/sql-data-sources.html)


# PySpark UDFs vs Pandas UDFs

When working with User-Defined Functions (UDFs) in **PySpark** (2.3+), it is often more efficient to use Pandas UDFs (also known as vectorized UDFs) instead of standard Spark UDFs. Because PySpark UDFs require data movement between the JVM and Python, which is expensive, Pandas UDFs allow you to process the data using the power of vectorization and Apache Arrow. This can significantly improve the performance of your Spark application.

```python
import pandas as pd
from pyspark.sql.types import StringType
from pyspark.sql.functions import pandas_udf, udf


df = spark.createDataFrame(
    data=[
        ("Aymane", "Boumaaza", 20),
        ("John", "Doe", 30),
        ("Jane", "Doe", 40),
    ],
    schema=["firstname", "lastname", "age"],
)

# Standard UDF
@udf(returnType=StringType())
def fullname(firstname, lastname):
    return f"{firstname.capitalize()} {lastname.upper()}"

df.withColumn("fullname", fullname("firstname", "lastname"))

# Pandas UDF
@pandas_udf(StringType())
def fullname_pd(firstname: pd.Series, lastname: pd.Series) -> pd.Series:
    # You can use any of the pandas APIs here
    return firstname.str.capitalize() + " " + lastname.str.upper()

df.withColumn("fullname", fullname_pd("firstname", "lastname"))
```

# `cache` vs `persist`

In Spark, you can use the `cache` and `persist` functions to store data in memory for faster access. However, there are some important differences between these two functions. The persist function allows you to specify the storage level (e.g., memory only, memory and disk, etc.), while the `cache` function stores data in the default storage level `MEMORY_AND_DISK`, first Spark tries to store the dataframe in memory, if there's any excess, it will be stored in disk. It is important to choose the appropriate function based on your use case. The choice of storage level is crucial as it can have a significant impact on the performance of your Spark application.

**Note from the [Learning Spark: Lightning-Fast Big Data Analysis](https://pages.databricks.com/rs/094-YMS-629/images/LearningSpark2.0.pdf) book:**

> When you use cache() or persist(), the DataFrame is not fully cached until you invoke an action that goes through every record (e.g., count()). If you use an action like take(1), only one partition will be cached because Catalyst realizes that you do not need to compute all the partitions just to retrieve one record.


# Conclusion
Most of the topics I covered in this article were introduced to me in the [Learning Spark: Lightning-Fast Big Data Analysis](https://pages.databricks.com/rs/094-YMS-629/images/LearningSpark2.0.pdf) book, that I highly recommend to anyone who wants to learn more about Spark, it's a great resource to know more about Spark. I hope you found this article useful, if you have any questions or comments, please [reach out](https://twitter.com/_Enamya). Thanks for reading!
