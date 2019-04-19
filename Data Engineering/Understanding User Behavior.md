### Assignment 11 - Steve Dille

Summary
This project simulates event tracking at a game development company. The latest mobile game has two events we are interested in tracking:  buy a sword & join guild... This assignment constructs the API server to catch these events using flask, kafka, python, spark and Apache zookeeper to maintain configuration information and provide distributed synchronization.

In this project, we build out a Docker cluster for an end to end system. A web API server receives API calls from a phone application, logs the API calls, generates events in json format, and publishes them to a kafka topic. A big data analytics framework using the lambda architecture is built out. For the speed layer, spark will be used to subscribe to the kafka topic and prepare the data for analytics. For the batch layer, hadoop hdfs files in parquet format will be used. For the serving layer, external tools will be used to query the data in the hadoop hdfs parquet files. 

We use a web browser (instead of apache bench) to generate web API calls. The spark is enhanced to write the data frame into hadoop hdfs in parquet format, a user defined function is introduced to munge the data frame, and a filter is used to separate events later in the project.



### Setting up to run Spark jobs

Create a directory, CD into it for flask, kafka and spark, get docker-compose
```
mkdir ~/w205/spark-from-files/
cd ~/w205/spark-from-files
cp ~/w205/course-content/11-Storing-Data-III/docker-compose.yml .
cp ~/w205/course-content/11-Storing-Data-III/*.py .
```

Here is the docker-compose.yml file for zookeeper, kafka, spark, cloudera and the MIDS image. 

```
YML
---
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 32181
      ZOOKEEPER_TICK_TIME: 2000
    expose:
      - "2181"
      - "2888"
      - "32181"
      - "3888"
    extra_hosts:
      - "moby:127.0.0.1"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:32181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    expose:
      - "9092"
      - "29092"
    extra_hosts:
      - "moby:127.0.0.1"

  cloudera:
    image: midsw205/cdh-minimal:latest
    expose:
      - "8020" # nn
      - "50070" # nn http
      - "8888" # hue
    #ports:
    #- "8888:8888"
    extra_hosts:
      - "moby:127.0.0.1"

  spark:
    image: midsw205/spark-python:0.0.5
    stdin_open: true
    tty: true
    expose:
      - "8888"
    ports:
      - "8888:8888"
    volumes:
      - "~/w205:/w205"
    command: bash
    depends_on:
      - cloudera
    environment:
      HADOOP_NAMENODE: cloudera
    extra_hosts:
      - "moby:127.0.0.1"

  mids:
    image: midsw205/base:latest
    stdin_open: true
    tty: true
    expose:
      - "5000"
    ports:
      - "5000:5000"
    volumes:
      - "~/w205:/w205"
    extra_hosts:
      - "moby:127.0.0.1"
```



Spin up the cluster
```
docker-compose up -d
```

Display the log output for the service cloudera in this window so we can see events
```
docker-compose logs -f cloudera
```

Execute cloudera hadoop
```
docker-compose exec cloudera hadoop fs -ls /tmp/
```

```
OUTPUT
Found 2 items
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2019-03-28 19:40 /tmp/hive
```



Create a topic called events on kafka to capture our events
```
docker-compose exec kafka kafka-topics --create --topic events --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
```


```
OUTPUT
Created topic "events".
```

Use the python flask library to write a simple API server that logs events in kafka named game_api.py. Flask is a free webserver employed for this task.  Here is the API server code:

```
#!/usr/bin/env python
import json
from kafka import KafkaProducer
from flask import Flask, request

app = Flask(__name__)
producer = KafkaProducer(bootstrap_servers='kafka:29092')

; Put string as json logged on the topic on kafka
def log_to_kafka(topic, event):
    event.update(request.headers)
    producer.send(topic, json.dumps(event).encode())

; This function returns the default response if nothing is purchased
@app.route("/")
def default_response():
    default_event = {'event_type': 'default'}
    log_to_kafka('events', default_event)
    return "This is the default response!\n"

; This function returns purchased a sword if one is purchased
@app.route("/purchase_a_sword")
def purchase_a_sword():
    purchase_sword_event = {'event_type': 'purchase_sword'}
    log_to_kafka('events', purchase_sword_event)
    return "Sword Purchased!\n"
```


Run the Flask ap
```
docker-compose exec mids env FLASK_APP=/w205/spark-from-files/game_api.py flask run --host 0.0.0.0
```
```
OUTPUT
* Serving Flask app "game_api"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
12.27.66.8 - - [28/Mar/2019 19:49:04] "GET / HTTP/1.1" 200 -
12.27.66.8 - - [28/Mar/2019 19:49:04] "GET /favicon.ico HTTP/1.1" 404 -
12.27.66.8 - - [28/Mar/2019 19:49:08] "GET / HTTP/1.1" 200 -
12.27.66.8 - - [28/Mar/2019 19:49:10] "GET / HTTP/1.1" 200 -
12.27.66.8 - - [28/Mar/2019 19:50:56] "GET /purchase_a_sword HTTP/1.1" 200 -
12.27.66.8 - - [28/Mar/2019 19:50:59] "GET /purchase_a_sword HTTP/1.1" 200 -
12.27.66.8 - - [28/Mar/2019 19:51:04] "GET /purchase_a_sword HTTP/1.1" 200 -
```

In a second terminal session, test it out for default and purchasing a sword where we send an event to the API server and those events are logged to kafka. I did 3 each 
```
http://xxxxx:5000/

http://xxxxx:5000/purchase_a_sword
```

Read from kafka - Use kafkacat to consume the events from the events topic
```
docker-compose exec mids kafkacat -C -b kafka:29092 -t events -o beginning -e
```

Here are the 6 events I generated – 3 defaults, 3 purchase swords
```
{"Accept-Language": "en-US,en;q=0.9", "event_type": "default", "Host": "157.230.138.133:5000", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3", "Upgrade-Insecure-Requests": "1", "Connection": "keep-alive", "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36", "Accept-Encoding": "gzip, deflate"}
{"Accept-Language": "en-US,en;q=0.9", "event_type": "default", "Host": "157.230.138.133:5000", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3", "Upgrade-Insecure-Requests": "1", "Connection": "keep-alive", "Cache-Control": "max-age=0", "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36", "Accept-Encoding": "gzip, deflate"}
{"Accept-Language": "en-US,en;q=0.9", "event_type": "default", "Host": "157.230.138.133:5000", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3", "Upgrade-Insecure-Requests": "1", "Connection": "keep-alive", "Cache-Control": "max-age=0", "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36", "Accept-Encoding": "gzip, deflate"}
{"Accept-Language": "en-US,en;q=0.9", "event_type": "purchase_sword", "Host": "157.230.138.133:5000", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3", "Upgrade-Insecure-Requests": "1", "Connection": "keep-alive", "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36", "Accept-Encoding": "gzip, deflate"}
{"Accept-Language": "en-US,en;q=0.9", "event_type": "purchase_sword", "Host": "157.230.138.133:5000", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3", "Upgrade-Insecure-Requests": "1", "Connection": "keep-alive", "Cache-Control": "max-age=0", "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36", "Accept-Encoding": "gzip, deflate"}
{"Accept-Language": "en-US,en;q=0.9", "event_type": "purchase_sword", "Host": "157.230.138.133:5000", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3", "Upgrade-Insecure-Requests": "1", "Connection": "keep-alive", "Cache-Control": "max-age=0", "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36", "Accept-Encoding": "gzip, deflate"}
% Reached end of topic events [0] at offset 6: exiting
```

Lets use pyspark code to extract events from kafka and write them to parquet files in hdfs.  Here is the code. 

```
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""

import json
from pyspark.sql import SparkSession


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    events = raw_events.select(raw_events.value.cast('string'))
    extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF()

    extracted_events \
        .write \
        .parquet("/tmp/extracted_events")


if __name__ == "__main__":
    main()
```


Run the extract code
```
docker-compose exec spark spark-submit /w205/spark-from-files/extract_events.py
```


Check out the results in hadoop
```
docker-compose exec cloudera hadoop fs -ls /tmp/

Found 3 items
drwxr-xr-x   - root   supergroup          0 2019-03-28 20:02 /tmp/extracted_events
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2019-03-28 19:40 /tmp/hive

docker-compose exec cloudera hadoop fs -ls /tmp/extracted_events/

Found 2 items
-rw-r--r--   1 root supergroup          0 2019-03-28 20:02 /tmp/extracted_events/_SUCCESS
-rw-r--r--   1 root supergroup       3398 2019-03-28 20:02 /tmp/extracted_events/part-00000-820d4405-d388-4b6e-a429-f281d5d3af24-c000.snappy.parquet

```


More Spark – Let’s Transform Events – Python Code
```
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('string')
def munge_event(event_as_json):
    event = json.loads(event_as_json)
    event['Host'] = "moe"
    event['Cache-Control'] = "no-cache"
    return json.dumps(event)


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    munged_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'),
                raw_events.timestamp.cast('string')) \
        .withColumn('munged', munge_event('raw'))
    munged_events.show()

    extracted_events = munged_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.munged))) \
        .toDF()
    extracted_events.show()

    extracted_events \
        .write \
        .mode("overwrite") \
        .parquet("/tmp/extracted_events")


if __name__ == "__main__":
    main()

```



Run the transform code

```
docker-compose exec spark spark-submit /w205/spark-from-files/transform_events.py
```

Relevant output showing the events
```
+--------------------+---------------+---------------+-------------+----------+----+-------------------------+--------------------+--------------+--------------------+
|              Accept|Accept-Encoding|Accept-Language|Cache-Control|Connection|Host|Upgrade-Insecure-Requests|          User-Agent|    event_type|           timestamp|
+--------------------+---------------+---------------+-------------+----------+----+-------------------------+--------------------+--------------+--------------------+
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|       default|2019-03-28 19:49:...|
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|       default|2019-03-28 19:49:...|
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|       default|2019-03-28 19:49:...|
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|purchase_sword|2019-03-28 19:50:...|
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|purchase_sword|2019-03-28 19:50:...|
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|purchase_sword|2019-03-28 19:51:...|
+--------------------+---------------+---------------+-------------+----------+----+-------------------------+--------------------+--------------+--------------------+
```



Let's look at separating events – Here is Python code
```
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('string')
def munge_event(event_as_json):
    event = json.loads(event_as_json)
    event['Host'] = "moe"
    event['Cache-Control'] = "no-cache"
    return json.dumps(event)


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    munged_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'),
                raw_events.timestamp.cast('string')) \
        .withColumn('munged', munge_event('raw'))

    extracted_events = munged_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.munged))) \
        .toDF()

    sword_purchases = extracted_events \
        .filter(extracted_events.event_type == 'purchase_sword')
    sword_purchases.show()
    # sword_purchases \
        # .write \
        # .mode("overwrite") \
        # .parquet("/tmp/sword_purchases")

    default_hits = extracted_events \
        .filter(extracted_events.event_type == 'default')
    default_hits.show()
    # default_hits \
        # .write \
        # .mode("overwrite") \
        # .parquet("/tmp/default_hits")


if __name__ == "__main__":
    main()

```

Run the code
```
docker-compose exec spark spark-submit /w205/spark-from-files/separate_events.py
```

Relevant output - here are the events
```

+--------------------+---------------+---------------+-------------+----------+----+-------------------------+--------------------+--------------+--------------------+
|              Accept|Accept-Encoding|Accept-Language|Cache-Control|Connection|Host|Upgrade-Insecure-Requests|          User-Agent|    event_type|           timestamp|
+--------------------+---------------+---------------+-------------+----------+----+-------------------------+--------------------+--------------+--------------------+
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|purchase_sword|2019-03-28 19:50:...|
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|purchase_sword|2019-03-28 19:50:...|
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|purchase_sword|2019-03-28 19:51:...|
+--------------------+---------------+---------------+-------------+----------+----+-------------------------+--------------------+--------------+--------------------+

………

+--------------------+---------------+---------------+-------------+----------+----+-------------------------+--------------------+----------+--------------------+
|              Accept|Accept-Encoding|Accept-Language|Cache-Control|Connection|Host|Upgrade-Insecure-Requests|          User-Agent|event_type|           timestamp|
+--------------------+---------------+---------------+-------------+----------+----+-------------------------+--------------------+----------+--------------------+
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|   default|2019-03-28 19:49:...|
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|   default|2019-03-28 19:49:...|
|text/html,applica...|  gzip, deflate| en-US,en;q=0.9|     no-cache|keep-alive| moe|                        1|Mozilla/5.0 (Maci...|   default|2019-03-28 19:49:...|
+--------------------+---------------+---------------+-------------+----------+----+-------------------------+--------------------+----------+--------------------+

…
```



Take the containers down
```
docker-compose down
```
