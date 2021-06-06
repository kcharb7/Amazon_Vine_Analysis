# Amazon_Vine_Analysis
## Overview
### *Purpose*
Big Market, a start-up that helps companies optimize their marketing efforts, was tasked with helping the company SellBy. SellBy sells a large catalogue of product online and wishes to determine how their customer reviews compare to those of their competitors for similar products. SellBy has asked for assistance in analyzing Amazon reviews written by members of the paid Amazon Vine Program to determine if there is any bias toward favourable reviews from Vine members. 

# Analysis
## Perform ETL on Amazon Product Reviews
To begin, I created a new database called “amazonreviews” with Amazon RDS. Once created, I created a new database in pgAdmin with my Amazon RDS server. In pgAdmin, I ran the following query to create tables for my new database:
```
CREATE TABLE review_id_table (
  review_id TEXT PRIMARY KEY NOT NULL,
  customer_id INTEGER,
  product_id TEXT,
  product_parent INTEGER,
  review_date DATE -- this should be in the formate yyyy-mm-dd
);

-- This table will contain only unique values
CREATE TABLE products_table (
  product_id TEXT PRIMARY KEY NOT NULL UNIQUE,
  product_title TEXT
);

-- Customer table for first data set
CREATE TABLE customers_table (
  customer_id INT PRIMARY KEY NOT NULL UNIQUE,
  customer_count INT
);

-- vine table
CREATE TABLE vine_table (
  review_id TEXT PRIMARY KEY,
  star_rating INTEGER,
  helpful_votes INTEGER,
  total_votes INTEGER,
  vine TEXT,
  verified_purchase TEXT
);
```
Next, I created a Google Colab Notebook file named “Amazon_Reviews_ETL” and installed Spark:
```
import os
spark_version = 'spark-3.1.2'
os.environ['SPARK_VERSION']=spark_version

# Install Spark and Java
!apt-get update
!apt-get install openjdk-11-jdk-headless -qq > /dev/null
!wget -q http://www-us.apache.org/dist/spark/$SPARK_VERSION/$SPARK_VERSION-bin-hadoop2.7.tgz
!tar xf $SPARK_VERSION-bin-hadoop2.7.tgz
!pip install -q findspark

# Set Environment Variables
import os
os.environ["JAVA_HOME"] = "/usr/lib/jvm/java-11-openjdk-amd64"
os.environ["SPARK_HOME"] = f"/content/{spark_version}-bin-hadoop2.7"

# Start a SparkSession
import findspark
findspark.init()
```
Next, I downloaded a Postgres driver that allowed Spark to interact with Postgres:
```
# Download the Postgres driver that will allow Spark to interact with Postgres.
!wget https://jdbc.postgresql.org/download/postgresql-42.2.16.jar
```
Once successfully downloaded, I created a Spark session:
```
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("BigData-Challenge").config("spark.driver.extraClassPath","/content/postgresql-42.2.16.jar").getOrCreate()
```
Then, I extracted the Amazon review dataset pertaining to watches and placed it in a new DataFrame:
```
from pyspark import SparkFiles
url = "https://s3.amazonaws.com/amazon-reviews-pds/tsv/amazon_reviews_us_Watches_v1_00.tsv.gz"
spark.sparkContext.addFile(url)
df = spark.read.option("encoding", "UTF-8").csv(SparkFiles.get(""), sep="\t", header=True, inferSchema=True)
df.show()
````
I created a DataFrame from the Amazon review dataset to match the schema of the customers_table in PgAdmin using the groupby() function on the customer_id column and the agg() function to get the count of all customer ids:
```
# Create the customers_table DataFrame
customers_df = df.groupby("customer_id").agg({"customer_id": "count"}).withColumnRenamed("count(customer_id)", "customer_count")
customers_df.show()
```
I then created the products_table DataFrame using the selection() function on the “product_id” and “product_title” columns in the dataset. I additionally used the drop_duplicates() function to retrieve only unique product ids:
```
# Create the products_table DataFrame and drop duplicates. 
products_df = df.select(["product_id", "product_title"]).drop_duplicates()
products_df.show()
```
Using the select() function, I selected all columns in the dataset that matched those in the review_id_table in pgAdmin and used the to_date() function to convert the review_date column to a date:
```
# Create the review_id_table DataFrame. 
# Convert the 'review_date' column to a date datatype with to_date("review_date", 'yyyy-MM-dd').alias("review_date")
review_id_df = df.select(["review_id","customer_id", "product_id", "product_parent", to_date("review_date", 'yyyy-MM-dd').alias("review_date")])
review_id_df.show()
```
Finally, I used the select() function to select the columns from the dataset that matched those in the vine_table in pgAdmin:
```
# Create the vine_table. DataFrame
vine_df = df.select(["review_id", "star_rating", "helpful_votes", "total_votes", "vine", "verified_purchase"])
vine_df.show()
```
Once all DataFrames were created, I created the connection to my AWS RDS instance using the following code:
```
# Configure settings for RDS
mode = "append"
jdbc_url="jdbc:postgresql://amazonreviews.cftomahzpbcg.us-east-2.rds.amazonaws.com:5432/postgres"
config = {"user":"postgres", 
          "password": "password", 
          "driver":"org.postgresql.Driver"}
```
Then, I loaded the DataFrames into their corresponding tables in pgAdmin:
```
# Write review_id_df to table in RDS
review_id_df.write.jdbc(url=jdbc_url, table='review_id_table', mode=mode, properties=config)

# Write products_df to table in RDS
products_df.write.jdbc(url=jdbc_url, table='products_table', mode=mode, properties=config)

# Write customers_df to table in RDS
customers_df.write.jdbc(url=jdbc_url, table='customers_table', mode=mode, properties=config)

# Write vine_df to table in RDS
vine_df.write.jdbc(url=jdbc_url, table='vine_table', mode=mode, properties=config)
```
In pg Admin, I ran a query to check that each table populated correctly:
```
-- Query database to check successful upload
SELECT * FROM customers_table;
SELECT * FROM products_table;
SELECT * FROM review_id_table;
SELECT * FROM vine_table;
```

## Determine Bias of Vine Reviews
I created a new Google Colab Notebook named “Vine_Review_Analysis” and extracted the same data as above and recreated the vine_df:
```
import os
spark_version = 'spark-3.1.2'
os.environ['SPARK_VERSION']=spark_version

# Install Spark and Java
!apt-get update
!apt-get install openjdk-11-jdk-headless -qq > /dev/null
!wget -q http://www-us.apache.org/dist/spark/$SPARK_VERSION/$SPARK_VERSION-bin-hadoop2.7.tgz
!tar xf $SPARK_VERSION-bin-hadoop2.7.tgz
!pip install -q findspark

# Set Environment Variables
import os
os.environ["JAVA_HOME"] = "/usr/lib/jvm/java-11-openjdk-amd64"
os.environ["SPARK_HOME"] = f"/content/{spark_version}-bin-hadoop2.7"

# Start a SparkSession
import findspark
findspark.init()

from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("BigData-Challenge").config("spark.driver.extraClassPath","/content/postgresql-42.2.16.jar").getOrCreate()

from pyspark import SparkFiles
url = "https://s3.amazonaws.com/amazon-reviews-pds/tsv/amazon_reviews_us_Watches_v1_00.tsv.gz"
spark.sparkContext.addFile(url)
df = spark.read.option("encoding", "UTF-8").csv(SparkFiles.get(""), sep="\t", header=True, inferSchema=True)
df.show()

# Create the vine_table. DataFrame
vine_df = df.select(["review_id", "star_rating", "helpful_votes", "total_votes", "vine", "verified_purchase"])
vine_df.show()
```
Then I filtered the vine_df to retrieve all the rows where the total_votes count is equal to or greater than 20 to retrieve reviews that are more likely to be helpful:
```
# Create a DataFrame where the total_votes is equal to or greater than 20
vote_count_df = vine_df.filter(vine_df["total_votes"] >= 20)
vote_count_df.show()
```
The vote_count_df was filtered to retrieve all rows where the number of helpful_votes divided by total_votes is equal to or greater than 50%:
```
# Create a DataFrame with all the rows from vote_count_df where the number of helpful_votes/total_votes is equal to or greater than 50%
helpful_votes_df = vote_count_df.filter((vote_count_df["helpful_votes"]/vote_count_df["total_votes"]) >= 0.5)
helpful_votes_df.show()
```
Two DataFrames were created by filtering the helpful_votes_df: one DataFrame included rows where a review was written as part of the Vine program (paid) and the other DataFrame included rows where a review was not written as part of the Vine program (unpaid):
```
# Create a DataFrame where all the rows from helpful_votes_df where vine == 'Y'
paid_df = helpful_votes_df.filter(helpful_votes_df["vine"] == 'Y')
paid_df.show()

# Create a DataFrame where all the rows from helpful_votes_df where vine == 'N'
unpaid_df = helpful_votes_df.filter(helpful_votes_df["vine"] == 'N')
unpaid_df.show()
```
Next, I determined the total number of paid and unpaid reviews:
```
# Determine the total number of paid reviews
total_paid_reviews = paid_df.count()
total_paid_reviews

# Determine the total number of unpaid reviews
total_unpaid_reviews = unpaid_df.count()
total_unpaid_reviews
```
Then, I determined the number of 5-star for paid and unpaid reviews:
```
# Determine the number of 5-star paid reviews
five_star_paid = paid_df.filter(paid_df["star_rating"] == 5).count()
five_star_paid

# Determine the number of 5-star unpaid reviews
five_star_unpaid = unpaid_df.filter(unpaid_df["star_rating"] == 5).count()
five_star_unpaid
```
Finally, I determined the percentage of 5-star reviews for paid and unpaid reviews:
```
# Determine the percentage of 5-star paid reviews
percentage_five_paid = (five_star_paid/total_paid_reviews) *100
percentage_five_paid

# Determine the percentage of 5-star unpaid reviews
percentage_five_unpaid = (five_star_unpaid/total_unpaid_reviews) *100
percentage_five_unpaid
```

### *Results*
-	How many Vine reviews and non-Vine reviews were there?
There was a total of 47 vine reviews and 8,362 non-vine reviews.

![total_reviews_paid.png]( https://github.com/kcharb7/Amazon_Vine_Analysis/blob/main/Images/total_paid_reviews.png)
![total_reviews_unpaid.png]( https://github.com/kcharb7/Amazon_Vine_Analysis/blob/main/Images/total_unpaid_reviews.png)

-	How many Vine reviews were 5 stars? How many non-Vine reviews were 5 stars?
15 vine reviews and 4,332 non-vine reviews were 5 stars, respectively. 

![five_star_paid.png]( https://github.com/kcharb7/Amazon_Vine_Analysis/blob/main/Images/five_star_paid.png)
![five_star_unpaid.png]( https://github.com/kcharb7/Amazon_Vine_Analysis/blob/main/Images/five_star_unpaid.png)

-	What percentage of Vine reviews were 5 stars? What percentage of non-Vine reviews were 5 stars?
31.9% of vine reviews and 51.8% of non-vine reviews were 5 stars, respectively. 

![percentage_five_paid.png]( https://github.com/kcharb7/Amazon_Vine_Analysis/blob/main/Images/percentage_five_paid.png)
![percentage_five_unpaid.png]( https://github.com/kcharb7/Amazon_Vine_Analysis/blob/main/Images/percentage_five_unpaid.png)

### *Summary*
In summary, with only 31.9% of vine reviewers versus 51.8% of non-vine reviewers having provided 5 stars, the results suggests that there is no bias for reviews in the Vine program. However, the number of Vine reviewers was only 47 compared to 8,363 non-Vine reviewers and thus a greater sample of Vine reviewers is needed to gather a better representation of Vine reviewers. Further analyses could be conducted that determines the average star rating for Vine reviewers and non-Vine reviews to see if Vine reviewers provide a higher average rating than non-Vine reviewers.
