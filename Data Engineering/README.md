# Building a data pipeline by Steve Dille  
How to use Kafka and Spark continers to write messages to HDFS files for subsequent processing in analytics.

Apache Kafka is used for creating real-time data pipelines. In this example, we simply substitute a .json file for the messages that would be flowing in real-time.

Apache Spark is an open-source distributed general-purpose cluster-computing framework commonly used in data science and for scaling up big data analytics.

HDFS is the Hadoop Distributed File System (HDFS). It is the primary data storage system used by Hadoop applications. HDFS is a key part of the many Hadoop ecosystem technologies, as it provides a reliable means for managing pools of big data and supporting related big data analytics applications. Once data is stored in HDFS, it may be queried in SQL for analytics via Spark SQL, Presto etc.. or moved into other data analytics repositories as needed. I store the data in parquet files for high performance columnar access in analytics.

Docker is also used to spin up the containers needed in the cloud for this infrastructure.  Here is the docker .yml file for the containers:  


The example below will demonstrate the commands needed to build a data pipleine:   
https://github.com/steviebyte/Portfolio/blob/master/Data%20Engineering/Spark%20Kafka%20HDFS.md
      
