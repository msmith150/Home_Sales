# Home Sales Analysis Using SparkSQL

In this project, I analyzed home sales data using Apache Spark and SparkSQL. I performed various data analysis tasks, including loading the data, running SQL queries, caching, partitioning the data, and using the Parquet file format to optimize storage and performance. My goal was to extract key metrics from the home sales dataset, such as average prices, based on various criteria like the number of bedrooms, bathrooms, and other features.

## Prerequisites
To follow along with this project, you'll need:

Python 3.x
Apache Spark 3.x
Jupyter Notebook or Google Colab
An internet connection to download the necessary files

## Setup and Installation
To set up the environment and run the code, follow these steps:

### 1. Set Spark Version

I started by specifying the version of Spark I wanted to use (in this case, spark-3.5.3):

python
Copy code
import os
spark_version = 'spark-3.5.3'
os.environ['SPARK_VERSION'] = spark_version

### 2. Install Spark and Java

Next, I installed Spark and the required Java version (openjdk-11), which is needed to run Spark.

bash
Copy code
!apt-get update
!apt-get install openjdk-11-jdk-headless -qq > /dev/null
!wget -q http://www.apache.org/dist/spark/$SPARK_VERSION/$SPARK_VERSION-bin-hadoop3.tgz
!tar xf $SPARK_VERSION-bin-hadoop3.tgz
!pip install -q findspark

### 3. Set Environment Variables
I set the JAVA_HOME and SPARK_HOME environment variables to point to the appropriate directories:

python
Copy code
os.environ["JAVA_HOME"] = "/usr/lib/jvm/java-11-openjdk-amd64"
os.environ["SPARK_HOME"] = f"/content/{spark_version}-bin-hadoop3"

### 4. Start SparkSession

Once the environment was set up, I started the SparkSession to interact with Spark:

python
Copy code
import findspark
findspark.init()
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("SparkSQL").getOrCreate()

## Steps in the Code

### 1. Read Data from AWS S3 into a DataFrame

I downloaded the home sales data from an AWS S3 bucket and read it into a DataFrame.

python
Copy code
from pyspark import SparkFiles
url = "https://2u-data-curriculum-team.s3.amazonaws.com/dataviz-classroom/v1.2/22-big-data/home_sales_revised.csv"
spark.sparkContext.addFile(url)
home_sales_df = spark.read.csv(SparkFiles.get("home_sales_revised.csv"), sep=",", header=True)
home_sales_df.show()

### 2. Create a Temporary Table for SQL Queries


I then created a temporary view of the home sales data. This allowed me to query the data using SQL.

python
Copy code
home_sales_df.createOrReplaceTempView("home_sales")

### 3. Calculate the Average Price of a 4-Bedroom House Per Year
I ran a SQL query to calculate the average price of 4-bedroom houses sold each year, rounding the results to two decimal places:

python
Copy code
query = """
SELECT
  YEAR(date) AS year,
  ROUND(AVG(price), 2) AS average_price
FROM
  home_sales
WHERE
  bedrooms = 4
GROUP BY
  YEAR(date)
ORDER BY
  year
"""
average_price_df = spark.sql(query)
average_price_df.show()

### 4. Average Price by Year Built (3 Bedrooms, 3 Bathrooms)


Next, I calculated the average price of homes with 3 bedrooms and 3 bathrooms, grouped by the year they were built.

python
Copy code
query = """
SELECT
  date_built AS year_built,
  ROUND(AVG(price), 2) AS average_price
FROM
  home_sales
WHERE
  bedrooms = 3 AND bathrooms = 3
GROUP BY
  date_built
ORDER BY
  year_built
"""
average_price_by_year_built_df = spark.sql(query)
average_price_by_year_built_df.show()

### 5. Average Price with Additional Filters (Floor Count, Square Footage)


I extended the previous query by adding additional conditions for the number of floors and square footage, to get a more specific analysis.

python
Copy code
query = """
SELECT
  date_built AS year_built,
  ROUND(AVG(price), 2) AS average_price
FROM
  home_sales
WHERE
  bedrooms = 3
  AND bathrooms = 3
  AND floors = 2
  AND sqft_living >= 2000
GROUP BY
  date_built
ORDER BY
  year_built
"""
average_price_by_year_built_df = spark.sql(query)
average_price_by_year_built_df.show()

### 6. Average Price Per View Rating (With Conditions)


I wanted to know the average price of homes by view rating, where the average price was greater than or equal to $350,000. I also measured the runtime of this query for performance evaluation.

python
Copy code
start_time = time.time()

query = """
SELECT
  view,
  ROUND(AVG(price), 2) AS average_price
FROM
  home_sales
GROUP BY
  view
HAVING
  AVG(price) >= 350000
ORDER BY
  view
"""
average_price_by_view_df = spark.sql(query)
average_price_by_view_df.show()

print("--- %s seconds ---" % (time.time() - start_time))

### 7. Cache the Home Sales Table


I cached the home_sales table to speed up subsequent queries.

python
Copy code
spark.sql("cache table home_sales")

### 8. Check if the Table is Cached


I checked if the home_sales table was successfully cached.

python
Copy code
spark.catalog.isCached('home_sales')

### 9. Run Query on Cached Data


With the data cached, I ran the same query again to see the performance improvement.

python
Copy code
start_time = time.time()

query = """
SELECT
  view,
  ROUND(AVG(price), 2) AS average_price
FROM
  home_sales
GROUP BY
  view
HAVING
  AVG(price) >= 350000
ORDER BY
  view
"""
average_price_by_view_df = spark.sql(query)

print("--- %s seconds ---" % (time.time() - start_time))

### 10. Partition Data by "date_built"

I partitioned the data by the date_built field to optimize storage and performance.

python
Copy code
home_sales_df.write.partitionBy("date_built").mode("overwrite").parquet("homes_sales_partitioned")

### 11. Read the Partitioned Parquet Data

I read the partitioned Parquet data back into a DataFrame.

python
Copy code
p_home_sales_df_p = spark.read.parquet('homes_sales_partitioned')

### 12. Create a Temporary Table for the Partitioned Data

I created a temporary table for the partitioned Parquet data to run SQL queries on it.

python
Copy code
p_home_sales_df_p.createOrReplaceTempView("home_sales_temp_table")

### 13. Run Query on Partitioned Data

I ran the same query as before (average price per view rating) on the partitioned data to compare performance.

python
Copy code
start_time = time.time()

query = """
SELECT
  view,
  ROUND(AVG(price), 2) AS average_price
FROM
  home_sales
GROUP BY
  view
HAVING
  AVG(price) >= 350000
ORDER BY
  view
"""
average_price_by_view_df = spark.sql(query)

print("--- %s seconds ---" % (time.time() - start_time))

### 14. Uncache the Temporary Table

I uncached the home_sales table to free up memory.

python
Copy code
spark.sql("uncache table home_sales")

### 15. Check if the Table is Uncached

Finally, I verified that the table was no longer cached.

python
Copy code
if spark.catalog.isCached("home_sales"):
  print("table is still cached")
else:
  print("all clear")

## Conclusion
Through this project, I demonstrated how to use Apache Spark and SparkSQL to perform data analysis tasks such as calculating average home prices, creating temporary tables, partitioning data, caching, and running optimized queries on large datasets. These techniques are crucial for handling large-scale data in a distributed environment like Spark.
