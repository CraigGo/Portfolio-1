# Building a data pipeline by Steve Dille  
How to use Kafka and Spark continers to write messages to HDFS files for subsequent processing in analytics.

Apache Kafka is used for creating real-time data pipelines. In this example, we simply substitute a .json file for the messages that would be flowing in real-time.

Apache Spark is an open-source distributed general-purpose cluster-computing framework commonly used in data science and for scaling up big data analytics.

HDFS is the Hadoop Distributed File System (HDFS). It is the primary data storage system used by Hadoop applications. HDFS is a key part of the many Hadoop ecosystem technologies, as it provides a reliable means for managing pools of big data and supporting related big data analytics applications. Once data is stored in HDFS, it may be queried in SQL for analytics via Spark SQL, Presto etc.. or moved into other data analytics repositories as needed. I store the data in parquet files for high performance columnar access in analytics.

Docker is also used to spin up the containers needed in the cloud for this infrastructure.  Here is the docker .yml file for the containers:  

https://github.com/steviebyte/Portfolio/blob/master/Data%20Engineering/docker-compose.yml

The example below will demonstrate the commands needed to build a data pipleine:   
https://github.com/steviebyte/Portfolio/blob/master/Data%20Engineering/Spark%20Kafka%20HDFS.md

This assignment demonstrates tracking user activity. I build out a Docker cluster for a prototype of the Lambda Architecture. The Lambda Architecture consists of 1) a speed layer, 2) a batch layer, and a 3) a serving layer. This Assignment focuses on the internally facing components of the batch layer and the speed layer, both built using Parquet files to create a scale out SQL columnar data store on top of the Hadoop Distributed File System (HDFS).

I also created a pyspark shell in the spark container and used the spark kafka API to read the messages from the kafka topic into a spark dataframe and explored the resulting schema and binary data. I also created a new spark dataframe with the key and value slots converted to string and explored the resulting schema and json data as strings.

Last, I wrote the spark dataframes out in parquet format to hadoop hdfs and used spark SQL to impose schema on the dataframes and perform examples of simple queries againt the dataframes using spark SQL.
      
