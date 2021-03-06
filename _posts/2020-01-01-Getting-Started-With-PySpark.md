# PySpark Tutorial

Short list of how to do basic things like read from a Hive database table to create a PySpark DataFrame, manipulate the data, and write your outputs to HDFS.

These methods should be the same whether you have 10 rows or 10 million rows in the DataFrame.
See [Spark Programming Guide](https://spark.apache.org/docs/2.0.2/sql-programming-guide.html) for more.

## Getting Started - Consoles and Scripts
Open the Pyspark Console. Every other tutorial in the world will use Jupyter Notebooks, but I *hate* them so I'm sticking with the command line.
```
pyspark
```

If you want, you can set the Spark Python REPL to IPython using environmental variables. However I have found this to be quite buggy, especially when using UDFs -
I've often hit errors telling me that the driver and worker nodes are using different versions of Python.
```
export PYSPARK_PYTHON=/path/to/bin/python
export PYSPARK_DRIVER_PYTHON=/path/to/bin/ipython
pyspark
```

Instead, I use a really stupid hack and start IPython from within a normal PySpark console:
```
pyspark
```

```python
from IPython import start_ipython; start_ipython()
```

This seems to be much more stable, but the downside is that you can't actually close it properly.
Rather just typing `exit`, you need to suspend the console with `CTRL+Z`, and then `kill` the job __twice__ from the stopped jobs list.
```
jobs
[1]+  Stopped                 python

kill %1
[1]+  Stopped                 python
kill %1
[1]+  Terminated              python
```

Of course, you could just use a Jupyter notebook, but again I really really hate them.

Opening a PySpark console will create a `SparkSession` for you, accessible as the Python variable `spark`.
If you are writing a script, you need to create this yourself:
```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()
```
You can add extra stuff to this `SparkSession` if you discover that you need to, such as extra settings to read and write from partitioned Hive tables. Eg:
```python
    spark = SparkSession \
        .builder \
        .enableHiveSupport() \
        .config('hive.exec.dynamic.partition', 'true') \
        .config('hive.exec.dynamic.partition.mode', 'nonstrict') \
        .config('spark.sql.sources.partitionOverwriteMode', 'dynamic') \
        .getOrCreate()
```

Now that we have a console or script open, import some basic stuff
```{python}
from pyspark.sql import functions as F
from pyspark.sql.types import StringType, DoubleType, FloatType
import math
from functools import reduce
```

If you want, set the Spark loggin level a level other than `INFO`.
Otherwise you could be spammed with INFO messages that make it impossible to see what's going on
```{python}
sc.setLogLevel("ERROR")
```

Read data from a Hive table:
```{sql}
df = sqlContext.sql("SELECT * FROM database_name.spark_tutorial_view")
df.show()
```
Alternatively, read it directly from a file using `spark.read.{{file format}}`:
```python
# obviously you'd only want to use one of these
df = spark.read.parquet('/path/to/parquet/data')
df = spark.read.orc('/path/to/orc/data')
df = spark.read.csv('/path/to/csv/data')
```

We can also create some data locally (or read it into a Pandas DataFrame) and them make a Spark DataFrame out of it:
```{python}
headers = ("id", "some_text")
data=[
        (1, "all_lower_case_1"),
        (2, "asljkwq3j4flqkje342"),
        (3, "it was the best of times, it was the worst of times"),
        (4, "May the fourth be with you"),
        (5, "The fifth element"),
        (6, "all_lower_case_1"),
     ]
df = spark.createDataFrame(data, headers)

# Print the first few lines
df.show()
```

## Straightforward summaries, aggregation, filtering etc

```{python}

# Simple count
df.count()

# Group by and aggregate in some way
df.groupby('some_text').agg(F.max('some_text')).show()

df.groupby('some_text').count().show()

# Count distinct
df.select('some_text').distinct().count()

# Filter according to some rule: show only the even-numbered rows
df.filter(df.id % 2 == 0).show()

# You can also use SQL-style syntax with filter
df.filter('id = 2').show()
```

## Standard UDFs
You can't just apply a function to a DataFrame column like you can
in R or pandas. Instead you need to declare it as a UDF, and give it a return type

 * Step 1: Declare a function
```{python}
def add_one(x):
    # some complicated analysis or manipulation of the data
    y = x + 1.0
    return y
```

 * Step 2: Turn this local function into a UDF, with a return type (float, double, string etc) so that it can be used on the distributed DataFrame
```{python}
udf_add_one = F.udf(add_one, FloatType())
```

The function you pass to udf() can be a lambda instead of a "full" function, but it still needs a return type 

```{python}
udf_upper = F.udf(lambda x: x.upper(), StringType())
udf_log = F.udf(lambda x: math.log(x), FloatType())
```

## Adding New Columns to DataFrames
Add a new column "id_plus_one" to the DataFrame df, using the UDF we defined.


We use `withColumn("some_column_name", some_value)` to add or overwrite a column on a DataFrame. If a column named `some_column_name` does not already exist in the DataFrame, we add this column to the DataFrame. Otherwise, the values in this column overwritten with `some_value`.


The end result in both cases is that the DataFrame has a column called `some_column_name`, and the value of the nth row of `some_column_name` is the nth value returned by `some_value`. You need to wrap the column name you are accessing in the UDF in col().

```{python}
new_df = df.withColumn("id_plus_one", udf_add_one(F.col("id")))
new_df = new_df.withColumn("id_plus_two", udf_add_one(F.col("id_plus_one")))
new_df.show()
```


## Manipulate a few columns in the one go.
This is more of a Python trick than an "Intro to PySpark" topic. Nevertheless, it might be handy
```{python}
names = ["id_plus_one", "id_plus_two"]

new_df = reduce(lambda new_df, idx: new_df \
    .withColumn("log_" + names[idx], udf_log(F.col(names[idx]))), 
    range(len(names)), 
    new_df)

new_df.show()
```

## String Manipulation.

We are using a UDF to turn all the text in the column `some_text` to uppercase.
We'll do it in-place too, for the craic. You could do more sophisticated stuff here.
The process is the same:

```{python}
new_df = new_df.withColumn("some_text", udf_upper(F.col("some_text")))
new_df.show()
```

## Pandas UDFs

Pandas UDFs are available in the newer versions of Spark.
They tend to be much faster and slightly easier to write than normal PySpark UDFs.

They basically treat each partition of a Spark DataFrame as a Pandas dataframe,
and apply a Python function (not a Spark UDF) to each independently.
This makes Pandas UDFs particularly good for
[embarrassingly parallel tasks](https://en.wikipedia.org/wiki/Embarrassingly_parallel),
where you don't have to compare any row of the dataframe to any other row.

Be careful about this, because a Pandas UDF would give you the right answer for some
non-embarrassingly parallel problems if your DataFrame happens to have
exactly one partition. 

A Pandas UDF would work for the following (even though you don't need a UDF for 
most of them):
 * squaring a number
 * getting the log of a number
 * applying the predict() method of a pre-fitted ML model
 * capping a value at <= 100.0

A Pandas UDF is NOT appropriate for tasks like
 * min-max scaling a column
 * getting the percentile rank of each column entry


Create a Pandas UDF by wrapping a normal function in the `@pandas_udf` decorator,
and giving it a return type.
Pandas UDFs can take return types as strings, or else the usual types imported
from `pyspark.sql.types`.

For example, you can either write `@pandas_udf('string')` or `@pandas_udf(StringType())`.

```python
from pyspark.sql.functions import pandas_udf

@pandas_udf('double')  # equivalently @pandas_udf(DoubleType())
def pandas_add_one(x):
    return x + 1.0

new_df = df.withColumn("id_plus_one", pandas_add_one(col("id")))

new_df.show()
```

You'll find more information about them in [Databricks' Spark documentation](https://docs.databricks.com/spark/latest/spark-sql/udf-python-pandas.html).

## Extracting Data From DataFrames For Non-Distributed Work

### The `toPandas` method

The easiest way to do this is is to call the `toPandas()` method on the Spark DataFrame.
This will create a Pandas DataFrame containing the same data. The Pandas DataFrame will
exist only on the driver node, unlike the Spark one, so make sure you have enough
RAM on the driver to hold it.

```python
new_pandas_df = new_df.toPandas()

new_pandas_df.head()
```

### The `collect` method

The other (older and more awkward) way to do it is to call the `collect()` method
on the Spark DataFrame.
This creates a list of `Row` objects, that you can index into to access what you want.

```{python}
collected = new_df.collect()

print(collected)
```

## SQL-style Joins

### Inner Join

```{python}
even_df = df.filter('id % 2 == 0')

new_even = even_df.join(new_df, "id", "inner")
new_even.show()
```

### Left-Anti Join

Make a DataFrame containing the ids that appear in one DataFrame, but not the other.
Specifically, in `new_df` only, not in `even_df`. (That is, the odd-numbered `ids`).

In SQL you would do this with `WHERE NOT EXISTS(SELECT id FROM ...)`,
or `LEFT OUTER JOIN former_list WHERE former_list.id IS NULL`.

```{python}
only_in_new_df = new_df.join(new_even, "id", "leftanti")
only_in_new_df.show()
```

## Write DataFrame to File.

This works same regardless of whether you're working with HDFS,
AWS S3 (or the GCP or Azure equivalents), or on a local non-distributed filesystem.

Similar to how you read data of a given format, the syntax to write to a given
location in a given format is
`{{DATAFRAME_NAME}}.write.{{DATA_FORMAT}}('/path/to/save/location')`

By default, overwriting and appending are not allowed - you will get an error if you
try to write to a location that already has data in it.
Add `mode='overwrite'` or `mode='append'` to overwrite or add to data in a
given location.

```python
# {{DATAFRAME_NAME}}.write.{{DATA_FORMAT}}('/path/to/save/location')
# Will only work if no data exists here
only_in_new_df.write.parquet("/path/to/my/hdfs/location/pyspark_tutorial/only_in_new_df")

# Write data as CSV to some location that may or may not already have data there
only_in_new_df.write.csv("/path/to/my/hdfs/location/pyspark_tutorial/another_location", mode="overwrite")
```

