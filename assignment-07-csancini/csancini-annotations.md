# Tracking User Activity project

Carlos Sancini<br>
Assignment 7<br>
W205 Introduction to Data Engineering<br>
MIDS Summer 2018<br>

# Executive Summary

A new service that provides video book reviews has been created, and now companies like Pearson want to publish their ratings for data scientist analysis.

This project aims to establish a reference implementation of a batch layer of a Lambda architecture. The batch layer will be responsible for landing and structuring data for queries.

Basically, in this setting, two activities are performed to set up the architecture:

- First, create a queue and publish messages with Kafka.
- Second, use Spark to transform the messages.

In a later setting, Spark will be used to transform messages so that they can be landed in HDFS.

# Big Data Lambda Architecture

The following schematic represents a conceptual view of the Lambda architecture. This project aims to build the batch layer indicated with number 2 in the figure.

![Lambda Architecture](lambda.png)

The cluster for the Batch Layer consists of the following components, all of which are executed by Docker containers:

- Kafka<sup>1</sup>: a distributed streaming platform that has capabilities that provide means to publish/subscribe to record streams, similar to a message queue or corporate messaging system; to store streams of records in a fault-tolerant durable way; and, finally, for processing streams of records as they occur.
- Zookeeper<sup>2</sup>: ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. These services are dependencies of a Kakfa distributed application.
- Spark <sup>3</sup>: Spark is a unified analytics engine for large-scale data processing, which is suitable to rapidly query, analyze, and transform data at scale. In this setting Spark will enable interactive queries across large and distributed data sets, as well the processing of streaming data from Kafka.
- Ubuntu: a general purpose Linux environment (referenced as mids) for setting up Docker and running tools such as ```jq``` and ```vi```.

<sub>1) Definitions from kafka.apache.org</sub><br>
<sub>2) Definitions from zookeeper.apache.org</sub><br>
<sub>3) Definitions from spark.apache.org</sub>


# Step by step Annotations

## 1. Setting up the Kafka and Spark environment

A new a docker-compose.yml file is create to set up the cluster.

```bash
cd w205/assignment-07-csancini
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
      KAFKA_MESSAGE_MAX_BYTES: 5242880
    volumes:
      - /home/science/w205:/w205
    expose:
      - "9092"
      - "29092"
    extra_hosts:
      - "moby:127.0.0.1"

  spark:
    image: midsw205/spark-python:0.0.5
    stdin_open: true
    tty: true
    volumes:
      - /home/science/w205:/w205
    command: bash
    extra_hosts:
      - "moby:127.0.0.1"

  mids:
    image: midsw205/base:latest
    stdin_open: true
    tty: true
    volumes:
      - /home/science/w205:/w205
    extra_hosts:
      - "moby:127.0.0.1"
```

## 2. Spinning up the cluster

To check whether Kafka is running and to monitor its activity, the log is opened in the terminal.

```bash
docker-compose up -d
docker-compose logs -f kafka
```

## 3. Creating the Kafka topic

A topic called ```video-books-assessments``` is created with the first command and checked in Kafka with the second command.

```bash
docker-compose exec kafka \
    kafka-topics \
      --create \
      --topic video-books-assessments \
      --partitions 1 \
      --replication-factor 1 \
      --if-not-exists \
      --zookeeper zookeeper:32181
      --config max.message.bytes=5242880

docker-compose exec kafka \
  kafka-topics \
    --describe \
    --topic video-books-assessments \
    --zookeeper zookeeper:32181
```
The following output is obtained:
```
Topic:video-books-assessments   PartitionCount:1        ReplicationFactor:1     Configs:
        Topic: video-books-assessments  Partition: 0    Leader: 1       Replicas: 1     Isr: 1
```

## 4. Getting and checking the data

The file with the input data is downloaded and analyzed.

```bash
curl -L -o assessment-attempts-20180128-121051-nested.json https://goo.gl/f5bRm4
ls -l --block-size=M assessment-attempts-20180128-121051-nested.json
cat assessment-attempts-20180128-121051-nested.json | jq '.[0]'
cat assessment-attempts-20180128-121051-nested.json | jq '. | length'
cat assessment-attempts-20180128-121051-nested.json | jq '.[].exam_name' | sort | uniq | sort -g
cat assessment-attempts-20180128-121051-nested.json | jq '.[].exam_name' | sort | uniq | wc -l
```

The json file seems to refer to users' self-assessments about a variety of O'Reilly video books. There are 3280 assessment entries related to 103 different video books. The most requested ones with over 100 assessments are: Software Architecture Fundamentals Understanding the Basics, Introduction to Machine Learning, Learning to Program with R, Intermediate Python Programming, Introduction to Java 8, Introduction to Python, and Learning Git.

The json is a large file, so message publishing will be split using jq in the next section. In addition, it was included in the docker-compose.yml a configuration parameter for Kafka that increases the size of the message to 5MB:  ```KAFKA_MESSAGE_MAX_BYTES = 5242880```.

## 5. Publishing first subset of messages using the kafkacat utility

Publishing 1000 messages at a time to Kafka. The -c option in jq is used to remove the outer square bracket wrapper and transform the json to a compact format.

```bash
docker-compose exec mids \
  bash -c "cat /w205/assignment-07-csancini/assessment-attempts-20180128-121051-nested.json \
    | jq '.[0:1000]' -c \
    | kafkacat -P -b kafka:29092 -t video-books-assessments && echo 'Produced 1000 messages.'"

docker-compose exec mids \
  bash -c "cat /w205/assignment-07-csancini/assessment-attempts-20180128-121051-nested.json \
  | jq '.[1000:2000]' -c \
  | kafkacat -P -b kafka:29092 -t video-books-assessments && echo 'Produced 1000 messages.'"

docker-compose exec mids \
  bash -c "cat /w205/assignment-07-csancini/assessment-attempts-20180128-121051-nested.json \
  | jq '.[2000:3000]' -c \
  | kafkacat -P -b kafka:29092 -t video-books-assessments && echo 'Produced 1000 messages.'"  

docker-compose exec mids \
  bash -c "cat /w205/assignment-07-csancini/assessment-attempts-20180128-121051-nested.json \
  | jq '.[3000:3279]' -c \
  | kafkacat -P -b kafka:29092 -t video-books-assessments && echo 'Produced 279 messages.'"
```

The last json structure will be left for a second publishing step in which the consumer will not start from the beginning but consume the message incrementally.

## 6. Consuming first subset of messages using PySpark

PySpark is initialized in a new terminal window with the following command:

```bash
docker-compose exec spark pyspark
```

The subset of messages is then consumed from ```video-books-assessments``` kafka topic in PySpark from the beginning of time. In addition, a configuration parameter ```kafka.max.partition.fetch.bytes``` was set to enable PySpark consume a large message.

```python
messages = spark \
  .read \
  .format("kafka") \
  .option("kafka.bootstrap.servers", "kafka:29092") \
  .option("subscribe","video-books-assessments") \
  .option("startingOffsets", "earliest") \
  .option("endingOffsets", "latest") \
  .option("kafka.max.partition.fetch.bytes","5242880")
  .load()
```

Here the DataFrame schema is shown. This reflects Kafka queue schema, but the message itself is contained in the ```value``` field and its content is in binary format.

```python
messages.printSchema()
```

```
root  
  |-- key: binary (nullable = true)  
  |-- value: binary (nullable = true)  
  |-- topic: string (nullable = true)  
  |-- partition: integer (nullable = true)  
  |-- offset: long (nullable = true)  
  |-- timestamp: timestamp (nullable = true)  
  |-- timestampType: integer (nullable = true)  
```

```python
messages.show()
```

```
+----+--------------------+--------------------+---------+------+--------------------+-------------+  
| key|               value|               topic|partition|offset|           timestamp|timestampType|  
+----+--------------------+--------------------+---------+------+--------------------+-------------+  
|null|[5B 7B 22 6B 65 6...|video-books-asses...|        0|     0|1969-12-31 23:59:...|            0|  
|null|[5B 7B 22 6B 65 6...|video-books-asses...|        0|     1|1969-12-31 23:59:...|            0|  
|null|[5B 7B 22 6B 65 6...|video-books-asses...|        0|     2|1969-12-31 23:59:...|            0|  
|null|[5B 7B 22 6B 65 6...|video-books-asses...|        0|     3|1969-12-31 23:59:...|            0|  
+----+--------------------+--------------------+---------+------+--------------------+-------------+  
```

For this reason, the value is translated in a human readable string. A new data frame is derived from the original one.

```python
messages_as_strings = messages.selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
messages_as_strings.printSchema()
```

```
root  
  |-- key: string (nullable = true)  
  |-- value: string (nullable = true)  
```

```python
messages_as_strings.show()
```

```
+----+--------------------+  
| key|               value|  
+----+--------------------+  
|null|[{"keen_timestamp...|   
|null|[{"keen_timestamp...|  
|null|[{"keen_timestamp...|  
|null|[{"keen_timestamp...|  
+----+--------------------+  
```

Then, the first json message content is inspected.

```python
messages_as_strings.select('value').take(1)[0]
```

## 7. Publishing another message and consuming the last offset in PySpark

In this step, a new message with the last json item published.

```bash
docker-compose exec mids \
  bash -c "cat /w205/assignment-07-csancini/assessment-attempts-20180128-121051-nested.json \
  | jq '.[3279]' -c \
  | kafkacat -P -b kafka:29092 -t video-books-assessments && echo 'Produced 1 message.'"
```

Then, only the last offset is consumed in PySpark from Kafka.

```python
messages = spark \
  .read \
  .format("kafka") \
  .option("kafka.bootstrap.servers", "kafka:29092") \
  .option("subscribe","video-books-assessments") \
  .option("startingOffsets", """{"video-books-assessments":{"0":4}}""") \
  .option("endingOffsets", "latest") \
  .option("kafka.max.partition.fetch.bytes","5242880")
  .load()
messages_as_strings = messages.selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
```

To confirm that only the last json item is consumed, its content is displayed.

```python
messages_as_strings.select('value').take(1)[0].value
```

## 8. Parsing the message content in json format

The python json library is imported and used to load the message content.

```python
json_message = json.loads(messages_as_strings.select('value').take(1)[0].value)
json_message
{'keen_timestamp': '1513766386.5845051', 'max_attempts': '1.0', 'started_at': '2017-12-20T10:38:26.490Z'...
```

Next, specific json atributtes are selected.

```python
print(json_message['sequences']['questions'][0])

{'user_incomplete': False, 'user_correct': True, 'options': [
        {'checked': True, 'at': '2017-12-20T10: 38: 29.540Z', 'id': 'a8b3951b-d1e1-4fa7-94cb-0a0415be1573', 'submitted': 1, 'correct': True
        },
        {'checked': False, 'id': 'f026267d-1997-48a3-9021-5c27fb743f3b'
        },
        {'checked': False, 'id': '72f87cf2-b723-4a02-b5b6-538278349eab'
        },
        {'checked': False, 'id': 'b64ddac5-5ac6-431a-b73d-c7388c760b85'
        }
    ], 'user_submitted': True, 'id': 'ff4fd2c3-d00d-4fee-b783-f1a6091f61f9', 'user_result': 'correct'
}
```

## 9. Tearing the cluster down

The cluster is shut down and docker processed are checked.

```bash
docker-compose down
docker-compose ps
docker ps -a
```

# Enhancements Appendix

The following enhancements were considered during this assignment:

- More meaningful queue name;
- JSON file analysis.
- Additional configuration in Kafka to enable larger messages;
- Additional configuration in PySpark to enable larger messages;
- Publishing the json file in parts;
- Reading the queue from a given offset in PySpark;
