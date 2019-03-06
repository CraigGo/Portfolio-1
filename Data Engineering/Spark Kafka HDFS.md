Kafka, Spark, HDFS W205 Data Engineering Project - UC Berkeley, MIDS Program
Tracking User Activity

In this project, you work at an ed tech firm. You've created a service that delivers assessments, and now lots of different customers (e.g., Pearson) want to publish their assessments on it. You need to get ready for data scientists who work for these customers to run queries on the data.

Through 3 different activites (6,7,8), you will spin up existing containers and prepare the infrastructure to land the data in the form and structure it needs to be to be queried.

6 - Publish and consume messages with kafka.
7 - Use spark to transform the messages.
8 - Use spark to transform the messages so that you can land them in hdfs.

Steve Dille, W205 Assignment 8 Annotations for Publishing and Consuming Messages with Kafka, Spark and HDFS
Copy the .YML file from course content repository and create a directory.
759 mkdir ~/w205/spark-with-kafka-and-hdfs
760 cd ~/w205/spark-with-kafka-and-hdfs
761 cp ~/w205/course-content/08-Querying-Data/docker-compose.yml .

Spin up the MIDS cluster, zookeeper and kafka.
762 docker-compose up -d

Checkout Hadoop before we write to it.
764 docker-compose exec cloudera hadoop fs -ls /tmp/

Create a topic in kafka called assessments.
765 docker-compose exec kafka kafka-topics --create --topic assessments --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181

Pull in the json file to test messaging with.
766 cd ~/w205
767 curl -L -o assessment-attempts-20180128-121051-nested.json https://goo.gl/ME6hjp
768 cd ~/w205/spark-with-kafka-and-hdfs

Publish some test messages to assessments with kafkacat.
770 docker-compose exec mids bash -c "cat /w205/assessment-attempts-20180128-121051-nested.json | jq '.[]' -c | kafkacat -P -b kafka:29092 -t assessments"

Run spark using the spark container.
771 docker-compose exec spark pyspark

Within spark, you can read messages from kafka with commands like.
raw_assess = spark.read.format("kafka").option("kafka.bootstrap.servers", "kafka:29092").option("subscribe","assessments").option("startingOffsets", "earliest").option("endingOffsets", "latest").load()

Cache the data.
raw_assess.cache()

See the schema & messages.
raw_assess.printSchema()

Cast the messages as strings.
assess = raw_assess.select(raw_assess.value.string('string'))

Write to HDFS.
assess.write.parquet("/tmp/assess")

Look at what was written.
assess.show()

Extract Data - Deal with unicode.
import sys sys.stdout = open(sys.stdout.fileno(), mode='w', encoding='utf8', buffering=1)

Translate the json into something more readable - take a look.
import json
assess.rdd.map(lambda x: json.loads(x.value)).toDF().show()

extracted_assess = assess.rdd.map(lambda x: json.loads(x.value)).toDF()

from pyspark.sql import Row
extracted_assess = assess.rdd.map(lambda x: Row(**json.loads(x.value))).toDF()

extracted_assess.show()

Save the extract.
extracted_assess.write.parquet("/tmp/extracted_assess")

Compare the results.
assess.show()
extracted_assess.show()
exit()

Take down the cluster...
771 docker-compose down
