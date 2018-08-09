# Project 3 Setup

Carlos Sancini<br>
Assignment 11<br>
W205 Introduction to Data Engineering<br>
MIDS Summer 2018<br>

## Executive Summary

A game development company has launched a new game for which there are two user events that are particularly interesting in tracking, such as buy a sword, buy a knife and join a guild (each of the them has metadata). To process these user actions, the mobile app makes API calls to a web-based API-server. The API server for the game will be instrumented in order to catch and analyze these two event types.

This project aims to establish a reference implementation of a speed layer of a Lambda architecture. The speed layer will be responsible for capturing and acting upon the mentioned events. A Python Flask module will be used to write a simple API server.

Basically, in this setting, three activities are performed to set up the stack:

- First, creating a queue in Kafka to produce messages;
- Second, setting up a Flask web server with a web API that receives requests and produces messages to Kafka regarding the user activity and also request headers;
- Third, send requests to the API and check messages in Kafka;
- Fourth, use PySpark to consume events, transform messages and write them to HDFS as parquet files.

## Big Data Lambda Architecture

The following schematic represents a conceptual view of the Lambda architecture. This project aims to build the speed and batch layer indicated with number 2 in the figure.

![Lambda Architecture](lambda.png)

The cluster consists of the following components, all of which are executed by Docker containers:

- Kafka<sup>1</sup>: a distributed streaming platform that has capabilities that provide means to publish/subscribe to record streams, similar to a message queue or corporate messaging system; to store streams of records in a fault-tolerant durable way; and, finally, for processing streams of records as they occur.

- Zookeeper<sup>2</sup>: ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. These services are dependencies of a Kafka distributed application.

- Spark <sup>3</sup>: Spark is a unified analytics engine for large-scale data processing, which is suitable to rapidly query, analyze, and transform data at scale. In this setting Spark will enable interactive queries across large and distributed data sets, as well the processing of streaming data from Kafka.

- Hadoop HDFS <sup>4</sup>: The Hadoop Distributed File System (HDFS) is a distributed file system designed and a highly fault-tolerant environment that is designed to be deployed on low-cost hardware. HDFS provides high throughput access to application data and is suitable for applications that have large data sets. In a Hadoop Distributed File System (HDFS), once a file is closed, it cannot be modified. Hence, immutability is a core concept of a Big Data architecture.

- Ubuntu: a general-purpose Linux environment (referenced as mids) for setting up Docker and running tools such as ```jq``` and ```vi```.

<sub>1) Definition from kafka.apache.org</sub><br>
<sub>2) Definition from zookeeper.apache.org</sub><br>
<sub>3) Definition from spark.apache.org</sub>
<sub>3) Definition from hadoop.apache.org</sub>

## Step by step Annotations

## 1. Setting up the Kafka environment

A new a docker-compose.yml file is created to set up the cluster.

```bash
sudo mkdir ~/w205/assignment-11-csancini/spark-from-files
cd ~/w205/assignment-11-csancini/spark-from-files
vi docker-compose.yml
```

```yml
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
      - "/home/science/w205:/w205"
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
      - "/home/science/w205:/w205"
    extra_hosts:
      - "moby:127.0.0.1"
```

The following bash command copies the python files associated with the flask web API that produces messages to Kafka.

```bash
cp ~/w205/course-content/11-Storing-Data-III/*.py .
```

## 2. Spinning up the cluster

Next, the cluster is started.

```bash
docker-compose up -d
```

To check whether Kafka and Hadoop HDFS are running, their log are opened in the terminal.

```bash
docker-compose logs -f cloudera
docker-compose logs -f kafka
```

HDFS directories are then checked.

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/
```

```bash
Found 2 items
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-07-29 22:32 /tmp/hive
```

## 3. Creating the Kafka topic

A topic called ```game_events``` is created with the first command and checked in Kafka with the second command.

```bash
docker-compose exec kafka \
    kafka-topics \
      --create \
      --topic game-events \
      --partitions 1 \
      --replication-factor 1 \
      --if-not-exists \
      --zookeeper zookeeper:32181

docker-compose exec kafka \
  kafka-topics \
    --describe \
    --topic game-events \
    --zookeeper zookeeper:32181
```

The following output is obtained:

```bash
Topic:game-events       PartitionCount:1        ReplicationFactor:1     Configs:
        Topic: game-events      Partition: 0    Leader: 1       Replicas: 1     Isr: 1
```

## 4. Creating a simple API server

The simple API server is created below with the Python Flask library.

```python
#!/usr/bin/env python
import json
from kafka import KafkaProducer
from flask import Flask, request

app = Flask(__name__)
producer = KafkaProducer(bootstrap_servers='kafka:29092')


def log_to_kafka(topic, event):
    event.update(request.headers)
    producer.send(topic, json.dumps(event).encode())


@app.route("/")
def default_response():
    default_event = {'event_type': 'default'}
    log_to_kafka('events', default_event)
    return "\nThis is the default response!\n"


@app.route("/purchase_a_sword")
def purchase_a_sword():
    purchase_sword_event = {'event_type': 'purchase_sword'}
    log_to_kafka('events', purchase_sword_event)
    return "\nSword Purchased!\n"


@app.route("/purchase_a_knife")
def purchase_a_knife():
    purchase_knife_event = {'event_type': 'purchase_knife'}
    log_to_kafka('events', purchase_knife_event)
    return "\nKnife Purchased!\n"


@app.route("/join_guild")
def join_guild():
    join_guild_event = {'event_type': 'join_guild'}
    log_to_kafka('events', join_guild_event)
    return "\nJoin Guild!\n"
```

## 5. Running the python script and calling the API

The python script is executed in the following command. It will hold the Linux terminal and the output from the python program will be shown as API calls are made.

```bash
docker-compose exec mids \
  env FLASK_APP=/w205/assignment-11-csancini/spark-from-files/game_api.py \
  flask run --host 0.0.0.0
```

```bash
 * Serving Flask app "game_api"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```

Calls to the API are then made with curl.

```bash
docker-compose exec mids curl http://localhost:5000/
docker-compose exec mids curl http://localhost:5000/purchase_a_sword
docker-compose exec mids curl http://localhost:5000/
docker-compose exec mids curl http://localhost:5000/join_guild
docker-compose exec mids curl http://localhost:5000/join_guild
docker-compose exec mids curl http://localhost:5000/purchase_a_knife
docker-compose exec mids curl http://localhost:5000/purchase_a_knife
docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```

The result log in Flask terminal is shown below.

```bash
127.0.0.1 - - [29/Jul/2018 22:16:21] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [29/Jul/2018 22:16:32] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [29/Jul/2018 22:16:38] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [29/Jul/2018 22:16:47] "GET /join_guild HTTP/1.1" 200 -
127.0.0.1 - - [29/Jul/2018 22:16:55] "GET /join_guild HTTP/1.1" 200 -
127.0.0.1 - - [29/Jul/2018 22:17:02] "GET /purchase_a_knife HTTP/1.1" 200 -
127.0.0.1 - - [29/Jul/2018 22:17:09] "GET /purchase_a_knife HTTP/1.1" 200 -
127.0.0.1 - - [29/Jul/2018 22:17:19] "GET /purchase_a_sword HTTP/1.1" 200 -
```

## 6. Consuming messages and writing then to HDFS with PySpark

First, the messages are checked using the kafkacat utility.

```bash
docker-compose exec mids kafkacat -C -b kafka:29092 -t game_events -o beginning -e
```

```bash
{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "join_guild", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "join_guild", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_knife", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_knife", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
% Reached end of topic events [0] at offset 8: exiting
```

Then, the following python code is executed to consume the kafka messages and to write then to HDFS.

```python
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
        .option("subscribe", "game_events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    events = raw_events.select(raw_events.value.cast('string'))
    extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF()

    extracted_events.write.parquet("/tmp/extracted_events")


if __name__ == "__main__":
    main()
```

The code is submitted to the spark cluster with the following command.

```bash
docker-compose exec spark spark-submit /w205/assignment-11-csancini/spark-from-files/extract_events.py
```

Next, HDFS is checked to confirm that the parquet file was written to the distributed storage.

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/
docker-compose exec cloudera hadoop fs -ls /tmp/extracted_events/
```

```bash
Found 3 items
drwxr-xr-x   - root   supergroup          0 2018-07-29 22:34 /tmp/extracted_events
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-07-29 22:32 /tmp/hive
```

```bash
Found 2 items
-rw-r--r--   1 root supergroup          0 2018-07-29 22:34 /tmp/extracted_events/_SUCCESS
-rw-r--r--   1 root supergroup       1189 2018-07-29 22:34 /tmp/extracted_events/part-00000-e4917c46-ca6a-4197-98b8-9efc26358161-c000.snappy.parquet
```

## 8.  Transforming messages

Using a Spark user defined function (udf), changes are made to the events before storing them in HDFS.

```python
#!/usr/bin/env python
"""Extract events from kafka, transform, and write to hdfs
"""
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('string')
def munge_event(event_as_json):
    event = json.loads(event_as_json)
    event['Host'] = "moe" # silly change to show it works
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

The output generate is shown below.

```python
+------+-------------+----+-----------+--------------+--------------------+  
|Accept|Cache-Control|Host| User-Agent|    event_type|           timestamp|  
+------+-------------+----+-----------+--------------+--------------------+  
|   */*|     no-cache| moe|curl/7.47.0|       default|2018-07-29 22:16:...|  
|   */*|     no-cache| moe|curl/7.47.0|purchase_sword|2018-07-29 22:16:...|  
|   */*|     no-cache| moe|curl/7.47.0|       default|2018-07-29 22:16:...|  
|   */*|     no-cache| moe|curl/7.47.0|    join_guild|2018-07-29 22:16:...|  
|   */*|     no-cache| moe|curl/7.47.0|    join_guild|2018-07-29 22:16:...|  
|   */*|     no-cache| moe|curl/7.47.0|purchase_knife|2018-07-29 22:17:...|  
|   */*|     no-cache| moe|curl/7.47.0|purchase_knife|2018-07-29 22:17:...|  
|   */*|     no-cache| moe|curl/7.47.0|purchase_sword|2018-07-29 22:17:...|  
+------+-------------+----+-----------+--------------+--------------------+  
```

## 9.  Separating events

The python code below is then used to separate the events in different HDFS directories.

```bash
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
    sword_purchases \
        .write \
        .mode("overwrite") \
        .parquet("/tmp/sword_purchases")

    default_hits = extracted_events \
        .filter(extracted_events.event_type == 'default')
    default_hits.show()
    default_hits \
        .write \
        .mode("overwrite") \
        .parquet("/tmp/default_hits")


if __name__ == "__main__":
    main()
```

By listing Hadoop directories the following output is shown.

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/
```

```bash
Found 5 items
drwxr-xr-x   - root   supergroup          0 2018-07-29 23:03 /tmp/default_hits
drwxr-xr-x   - root   supergroup          0 2018-07-29 22:44 /tmp/extracted_events
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-07-29 22:32 /tmp/hive
drwxr-xr-x   - root   supergroup          0 2018-07-29 23:03 /tmp/sword_purchases
```

## 11. Tearing the cluster down

Exiting flask.

```bash
control-C
```

Exiting PySpark.

```bash
exit()
```

The cluster is shut down and docker processes are checked.

```bash
docker-compose down
docker-compose ps
docker ps -a
```

## 12. Enhancements Appendix

The following enhancements were considered during this assignment:

- Executive Summary and Architecture description;
- More meaningful topic name: game_events;
- Additional API services: ```purchase_a_knife()``` and ```join_guild()```;
