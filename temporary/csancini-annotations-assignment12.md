# Project 3

Carlos Sancini<br>
Assignment 12<br>
W205 Introduction to Data Engineering<br>
MIDS Summer 2018<br>

## Executive Summary

A game development company has launched a new game for which there are two user events that are particularly interesting in tracking, such as buy a sword, buy a knife and join a guild (each of the them has metadata). To process these user actions, the mobile app makes API calls to a web-based API-server. The API server for the game will be instrumented in order to catch and analyze these two event types.

This project aims to establish a reference implementation of a batch layer of a Lambda architecture. 

Basically, in this setting, three activities are performed to set up the stack:

- First, creating a queue in Kafka to produce messages;
- Second, setting up a Flask web server with a web API that receives requests and produces messages to Kafka regarding the user activity and also request headers;
- Third, send requests to the API and check messages in Kafka;
- Fourth, write python code to handle multiple schemas in the same topic.
- Fifth, query HDFS using SparkSQL in Jupiter Notebook.

## Big Data Lambda Architecture

The following schematic represents a conceptual view of the Lambda architecture. This project aims to build the speed and batch layer indicated with number 2 in the figure.

![Lambda Architecture](lambda.png)

The cluster consists of the following components, all of which are executed by Docker containers:

- Kafka<sup>1</sup>: a distributed streaming platform that has capabilities that provide means to publish/subscribe to record streams, similar to a message queue or corporate messaging system; to store streams of records in a fault-tolerant durable way; and, finally, for processing streams of records as they occur.

- Zookeeper<sup>2</sup>: ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. These services are dependencies of a Kafka distributed application.

- Spark <sup>3</sup>: Spark is a unified analytics engine for large-scale data processing, which is suitable to rapidly query, analyze, and transform data at scale. In this setting Spark will enable interactive queries across large and distributed data sets, as well the processing of streaming data from Kafka.

- Hadoop HDFS <sup>4</sup>: The Hadoop Distributed File System (HDFS) is a distributed file system designed and a highly fault-tolerant environment that is designed to be deployed on low-cost hardware. HDFS provides high throughput access to application data and is suitable for applications that have large data sets. In a Hadoop Distributed File System (HDFS), once a file is closed, it cannot be modified. Hence,  immutability is a core concept of a Big Data architecture.

- Ubuntu: a general-purpose Linux environment (referenced as mids) for setting up Docker and running tools such as ```jq``` and ```vi```.

<sub>1) Definition from kafka.apache.org</sub><br>
<sub>2) Definition from zookeeper.apache.org</sub><br>
<sub>3) Definition from spark.apache.org</sub>
<sub>3) Definition from hadoop.apache.org</sub>

## Step by step Annotations

## 1. Setting up the Kafka environment

A new a docker-compose.yml file is created to set up the cluster.

```bash
sudo mkdir ~/w205/assignment-12-csancini/fullstack
cd ~/w205/assignment-12-csancini/fullstack
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
    volumes:
      - /home/science/w205:/w205
    expose:
      - "8888"
    ports:
      - "8888:8888"
    depends_on:
      - cloudera
    environment:
      HADOOP_NAMENODE: cloudera
    extra_hosts:
      - "moby:127.0.0.1"
    command: bash

  mids:
    image: midsw205/base:0.1.9
    stdin_open: true
    tty: true
    volumes:
      - /home/science/w205:/w205
    expose:
      - "5000"
    ports:
      - "5000:5000"
    extra_hosts:
      - "moby:127.0.0.1"
```

The following bach command copies the python files associated with the flask web API that produces messages to Kafka.

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

HDFS are then checked.

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/

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

The simple API server is created below with the Python Flask library. The event purchase_a_knife and join_guild have a different schema, and these difference will be handled in PySpark in the following sections.

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
    log_to_kafka('game-events', default_event)
    return "\nThis is the default response!\n"


@app.route("/purchase_a_sword")
def purchase_a_sword():
    purchase_sword_event = {'event_type': 'purchase_sword'}
    log_to_kafka('game-events', purchase_sword_event)
    return "\nSword Purchased!\n"


@app.route("/purchase_a_knife")
def purchase_a_knife():
    purchase_knife_event = {'event_type': 'purchase_knife',
                            'description': 'very sharp knife'}
    log_to_kafka('game-events', purchase_knife_event)
    return "Knife Purchased!\n"


@app.route("/join_guild")
def join_guild():
    join_guild_event = {'event_type': 'join_guild'}
    log_to_kafka('game-events', join_guild_event)
    return "\nJoin Guild!\n"
```

## 5. Running the python script and calling the API with Apache Bench

First, the python script is executed in the following command. It will hold the Linux terminal and the output from the python program will be shown as API calls are made.

```bash
docker-compose exec mids \
    env FLASK_APP=/w205/assignment-12-csancini/fullstack/game_api.py flask \
    run --host 0.0.0.0
```

```bash
 * Serving Flask app "game_api"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```

Then, kafkacat is set up in continuous mode with the -C option, so we view messages coming in.

```bash
docker-compose exec mids kafkacat -C -b kafka:29092 -t game-events -o beginning
```

Next, calls to the API are then made with Apache Bench, which is a utility designed to stress test web servers using a high volume of data in a short amount of time.

```bash
docker-compose exec mids ab -n 10 -H "Host: user1.comcast.com" http://localhost:5000/
docker-compose exec mids ab -n 10 -H "Host: user1.comcast.com" http://localhost:5000/purchase_a_sword
docker-compose exec mids ab -n 10 -H "Host: user2.att.com" http://localhost:5000/
docker-compose exec mids ab -n 10 -H "Host: user2.att.com" http://localhost:5000/purchase_a_sword
```

A sample of the Flask result log is shown below.

```bash
127.0.0.1 - - [04/Aug/2018 18:27:12] "GET /purchase_a_sword HTTP/1.0" 200 -
127.0.0.1 - - [04/Aug/2018 18:27:12] "GET /purchase_a_sword HTTP/1.0" 200 -
...
```

A sample kafkacat result log is shown below.

```bash
...
{"Host": "user1.comcast.com", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
{"Host": "user1.comcast.com", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "ApacheBench/2.3"}
...
```

A sample Apache Bench result log is shown below.

```bash
This is ApacheBench, Version 2.3 <$Revision: 1706008 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient).....done


Server Software:        Werkzeug/0.14.1
Server Hostname:        localhost
Server Port:            5000

Document Path:          /purchase_a_sword
Document Length:        17 bytes

Concurrency Level:      1
Time taken for tests:   0.026 seconds
Complete requests:      10
Failed requests:        0
Total transferred:      1720 bytes
HTML transferred:       170 bytes
Requests per second:    385.21 [#/sec] (mean)
Time per request:       2.596 [ms] (mean)
Time per request:       2.596 [ms] (mean, across all concurrent requests)
Transfer rate:          64.70 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.0      0       0
Processing:     2    2   0.7      3       4
Waiting:        0    1   1.5      2       4
Total:          2    3   0.7      3       4
WARNING: The median and mean for the processing time are not within a normal deviation
        These results are probably not that reliable.

Percentage of the requests served within a certain time (ms)
  50%      3
  66%      3
  75%      3
  80%      3
  90%      4
  95%      4
  98%      4
  99%      4
 100%      4 (longest request)
 ```

## 6. Consuming messages and writting then to HDFS with PySpark

The following python code is executed to consume the kafka messages. Using a Spark user defined function (udf), changes are made to the events before storing them in HDFS. In this setting, the code can only handle 1 schema for game-events.

```python
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
        .option("subscribe", "game-events") \
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

The code is then submitted to the spark cluster using spark-submit.

```bash
 docker-compose exec spark spark-submit /w205/assignment-12-csancini/fullstack/separate_events.py
```

The console then shows PySpark output.

```bash
 +------+-------------+----+---------------+--------------+--------------------+
|Accept|Cache-Control|Host|     User-Agent|    event_type|           timestamp|
+------+-------------+----+---------------+--------------+--------------------+
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
+------+-------------+----+---------------+--------------+--------------------+
only showing top 20 rows
```

```bash
+------+-------------+----+---------------+----------+--------------------+
|Accept|Cache-Control|Host|     User-Agent|event_type|           timestamp|
+------+-------------+----+---------------+----------+--------------------+
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:35:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:35:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:35:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:35:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:35:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:35:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:35:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:35:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:35:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:35:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:36:...|
|   */*|     no-cache| moe|ApacheBench/2.3|   default|2018-08-04 19:36:...|
+------+-------------+----+---------------+----------+--------------------+
only showing top 20 rows
```

Next, HDFS is checked to confirm that the parquet files were written separately to the distributed storage.

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/
```

```bash
Found 4 items
drwxr-xr-x   - root   supergroup          0 2018-08-04 19:36 /tmp/default_hits
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-08-04 19:34 /tmp/hive
drwxr-xr-x   - root   supergroup          0 2018-08-04 19:36 /tmp/sword_purchases
```

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/default_hits/
```

```bash
Found 2 items
-rw-r--r--   1 root supergroup          0 2018-08-04 19:36 /tmp/default_hits/_SUCCESS
-rw-r--r--   1 root supergroup       1764 2018-08-04 19:36 /tmp/default_hits/part-00000-2cc6dea2-e74a-4867-880d-c7cf4583153b-c000.snappy.parquet
```

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/sword_purchases/
```

```bash
Found 2 items
-rw-r--r--   1 root supergroup          0 2018-08-04 19:36 /tmp/sword_purchases/_SUCCESS
-rw-r--r--   1 root supergroup       1868 2018-08-04 19:36 /tmp/sword_purchases/part-00000-42c6f64e-1c98-4de2-80f6-1837e13cefa6-c000.snappy.parquet
```

## 7.  Handling multiple schemas

New Apache Bench calls are made to the API, consequently different schemas will take place in game-events topic.

```bash
docker-compose exec mids ab -n 10 -H "Host: user2.att.com" http://localhost:5000/purchase_knife
docker-compose exec mids ab -n 10 -H "Host: user2.att.com" http://localhost:5000/join_guild
```

```bash
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "purchase_knife", "Accept": "*/*", "description": "very sharp knife"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "purchase_knife", "Accept": "*/*", "description": "very sharp knife"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "purchase_knife", "Accept": "*/*", "description": "very sharp knife"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "purchase_knife", "Accept": "*/*", "description": "very sharp knife"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "purchase_knife", "Accept": "*/*", "description": "very sharp knife"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "purchase_knife", "Accept": "*/*", "description": "very sharp knife"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "purchase_knife", "Accept": "*/*", "description": "very sharp knife"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "purchase_knife", "Accept": "*/*", "description": "very sharp knife"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "purchase_knife", "Accept": "*/*", "description": "very sharp knife"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "purchase_knife", "Accept": "*/*", "description": "very sharp knife"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "join_guild", "Accept": "*/*", "description": "nice guild"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "join_guild", "Accept": "*/*", "description": "nice guild"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "join_guild", "Accept": "*/*", "description": "nice guild"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "join_guild", "Accept": "*/*", "description": "nice guild"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "join_guild", "Accept": "*/*", "description": "nice guild"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "join_guild", "Accept": "*/*", "description": "nice guild"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "join_guild", "Accept": "*/*", "description": "nice guild"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "join_guild", "Accept": "*/*", "description": "nice guild"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "join_guild", "Accept": "*/*", "description": "nice guild"}
{"Host": "user1.comcast.com", "User-Agent": "ApacheBench/2.3", "event_type": "join_guild", "Accept": "*/*", "description": "nice guild"}
```

The following python codes addresses multiple schemas generated by purchase_a_knife and join_guild in the game-events topic. The code below filters just the purchase_a_sword events.

```python
#!/usr/bin/env python
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('boolean')
def is_purchase(event_as_json):
    event = json.loads(event_as_json)
    if event['event_type'] == 'purchase_sword':
        return True
    return False


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
        .option("subscribe", "game-events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    purchase_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'),
                raw_events.timestamp.cast('string')) \
        .filter(is_purchase('raw'))

    extracted_purchase_events = purchase_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.raw))) \
        .toDF()
    extracted_purchase_events.printSchema()
    extracted_purchase_events.show()


if __name__ == "__main__":
    main()
```

```bash
docker-compose exec spark spark-submit /w205/assignment-12-csancini/fullstack/just_filtering.py
```

```bash
+------+-----------------+---------------+--------------+--------------------+
|Accept|             Host|     User-Agent|    event_type|           timestamp|
+------+-----------------+---------------+--------------+--------------------+
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
+------+-----------------+---------------+--------------+--------------------+
only showing top 20 rows
```

Then the filtered events are stored in HDFS in overwrite mode.

```bash
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('boolean')
def is_purchase(event_as_json):
    event = json.loads(event_as_json)
    if event['event_type'] == 'purchase_sword':
        return True
    return False


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
        .option("subscribe", "game-events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    purchase_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'),
                raw_events.timestamp.cast('string')) \
        .filter(is_purchase('raw'))

    extracted_purchase_events = purchase_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.raw))) \
        .toDF()
    extracted_purchase_events.printSchema()
    extracted_purchase_events.show()

    extracted_purchase_events \
        .write \
        .mode('overwrite') \
        .parquet('/tmp/purchases')


if __name__ == "__main__":
    main()
```

```bash
docker-compose exec spark spark-submit /w205/assignment-12-csancini/fullstack/filtered_writes.py
```

```bash
+------+-----------------+---------------+--------------+--------------------+
|Accept|             Host|     User-Agent|    event_type|           timestamp|
+------+-----------------+---------------+--------------+--------------------+
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
|   */*|user1.comcast.com|ApacheBench/2.3|purchase_sword|2018-08-04 19:36:...|
+------+-----------------+---------------+--------------+--------------------+
only showing top 20 rows
```

Next, HDFS is checked to confirm that the new parquet files were written separately to the distributed storage.

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/
```

```bash
Found 5 items
drwxr-xr-x   - root   supergroup          0 2018-08-04 19:36 /tmp/default_hits
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-08-04 19:34 /tmp/hive
drwxr-xr-x   - root   supergroup          0 2018-08-04 19:52 /tmp/purchases
drwxr-xr-x   - root   supergroup          0 2018-08-04 19:36 /tmp/sword_purchases
```

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/purchases/
```

```bash
Found 2 items
-rw-r--r--   1 root supergroup          0 2018-08-04 19:52 /tmp/purchases/_SUCCESS
-rw-r--r--   1 root supergroup       1693 2018-08-04 19:52 /tmp/purchases/part-00000-ee589caa-3c8b-4aa4-8ce8-ef89741fe16f-c000.snappy.parquet
```

## 8.  Querying with SparkSQL in Jupiter Notebook

First, Jupiter Notebook is set up.

```bash
docker-compose exec spark bash
```

Second, a symbolic link to a mapped directory is create so the notebook can outlive the Docker cluster.

```bash
df -lk
```

```bash
Filesystem     1K-blocks     Used Available Use% Mounted on
overlay         81120924 33432204  47672336  42% /
tmpfs              65536        0     65536   0% /dev
tmpfs            2023208        0   2023208   0% /sys/fs/cgroup
/dev/vda1       81120924 33432204  47672336  42% /w205
shm                65536        0     65536   0% /dev/shm
tmpfs            2023208        0   2023208   0% /proc/scsi
tmpfs            2023208        0   2023208   0% /sys/firmware
```

```bash
ln -s /w205 w205
```

Finally, the following notebook is created.

![Assignment 12 Jupiter Notebook](assignment-12-csancini.ipynb)

## 9. Tearing the cluster down

Exiting flask.

```bash
control-C
```

Exiting kafkacat.

```bash
control-C
```

The cluster is shut down and docker processes are checked.

```bash
docker-compose down
docker-compose ps
docker ps -a
```

## 10. Enhancements Appendix

The following enhancements were considered during this assignment:

- Executive Summary and Architecture description;
- More meaningful topic name: game_events;
- Additional API services: ```purchase_a_knife()``` and ```join_guild()```;