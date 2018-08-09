# Project 3 Setup

Carlos Sancini<br>
Assignment 10<br>
W205 Introduction to Data Engineering<br>
MIDS Summer 2018<br>

## Executive Summary

A game development company has launched a new game for which there are two user events that are particularly interesting in tracking, such as buy a sword, buy a knife and join a guild (each of the them has metadata). To process these user actions, the mobile app makes API calls to a web-based API-server. The API server for the game will be instrumented in order to catch and analyze these two event types.

This project aims to establish a reference implementation of a speed layer of a Lambda architecture. The speed layer will be responsible for capturing and acting upon the mentioned events. A Python Flask module will be used to write a simple API server.

Basically, in this setting, three activities are performed to set up the stack:

- First, creating a queue in Kafka to produce messages;
- Second, setting up a Flask web server with a web API that receives requests and produces messages to Kafka;
- Third, send requests to the API and check messages in Kafka;
- Fourth, consume events in PySpark.

## Big Data Lambda Architecture

The following schematic represents a conceptual view of the Lambda architecture. This project aims to build the batch layer indicated with number 2 in the figure.

![Lambda Architecture](lambda.png)

The cluster for the Batch Layer consists of the following components, all of which are executed by Docker containers:

- Kafka<sup>1</sup>: a distributed streaming platform that has capabilities that provide means to produce/consume to record streams, similar to a message queue or corporate messaging system; to store streams of records in a fault-tolerant durable way; and, finally, for processing streams of records as they occur.

- Zookeeper<sup>2</sup>: ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. These services are dependencies of a Kafka distributed application.

- Ubuntu: a general-purpose Linux environment (referenced as mids) for setting up Docker, setting up Flask and running tools such as ```jq``` and ```vi```.

<sub>1) Definition from kafka.apache.org</sub><br>
<sub>2) Definition from zookeeper.apache.org</sub><br>

## Step by step Annotations

## 1. Setting up the Kafka environment

A new a docker-compose.yml file is created to set up the cluster.

```bash
sudo mkdir ~/w205/assignment-10-csancini/flask-with-kafka-and-spark
cd ~/w205/assignment-10-csancini/flask-with-kafka-and-spark
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
    extra_hosts:
      - "moby:127.0.0.1"
    command: bash

  mids:
    image: midsw205/base:0.1.8
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

## 2. Spinning up the cluster

To check whether Kafka is running and to monitor its activity, the log is opened in the terminal.

```bash
docker-compose up -d
docker-compose logs -f kafka
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
from flask import Flask

app = Flask(__name__)
producer = KafkaProducer(bootstrap_servers='kafka:29092')
events_topic = 'game_events'


def log_to_kafka(topic, event):
  producer.send(topic, json.dumps(event).encode())


@app.route("/")
def default_response():
  default_event = {'event_type': 'default'}
  log_to_kafka(events_topic, default_event)
  return "\nThis is the default response!\n"


@app.route("/purchase_a_sword")
def purchase_a_sword():
  purchase_sword_event = {'event_type': 'purchase_sword'}
  log_to_kafka(events_topic, purchase_sword_event)
  return "\nSword Purchased!\n"


@app.route("/purchase_a_knife")
def purchase_a_knife():
  purchase_knife_event = {'event_type': 'purchase_knife'}
  log_to_kafka(events_topic, purchase_knife_event)
  return "\nKnife Purchased!\n"


@app.route("/join_guild")
def join_guild():
  join_guild_event = {'event_type': 'join_guild'}
  log_to_kafka(events_topic, join_guild_event)
  return "\nGuild Joined!\n"
```

## 5. Running the python script and calling the API

The python script is executed in the following command. It will hold the Linux terminal and the output from the python program will be shown as API calls are made.

```bash
docker-compose exec mids \
  env FLASK_APP=/w205/assignment-10-csancini/flask-with-kafka-and-spark/game_api_with_json_events.py \
  flask run --host 0.0.0.0

```

```bash
 * Serving Flask app "game_api_with_json_events"
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
127.0.0.1 - - [22/Jul/2018 13:31:11] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [22/Jul/2018 13:31:21] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [22/Jul/2018 13:31:32] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [22/Jul/2018 13:31:37] "GET /join_guild HTTP/1.1" 200 -
127.0.0.1 - - [22/Jul/2018 13:31:39] "GET /join_guild HTTP/1.1" 200 -
127.0.0.1 - - [22/Jul/2018 13:31:46] "GET /purchase_a_knife HTTP/1.1" 200 -
127.0.0.1 - - [22/Jul/2018 13:31:48] "GET /purchase_a_knife HTTP/1.1" 200 -
127.0.0.1 - - [22/Jul/2018 13:31:55] "GET /purchase_a_sword HTTP/1.1" 200 -
```

## 6. Consuming messages using the kafkacat utility

Lastly, the messages are consumed from the Kafka topic.

```bash
docker-compose exec mids kafkacat -C -b kafka:29092 -t game-events -o beginning -e
```

```bash
{"event_type": "default"}
{"event_type": "purchase_sword"}
{"event_type": "default"}
{"event_type": "join_guild"}
{"event_type": "join_guild"}
{"event_type": "purchase_knife"}
{"event_type": "purchase_knife"}
{"event_type": "purchase_sword"}
% Reached end of topic game_events [0] at offset 8: exiting
```

## 8.  Adding more key value attributes to the json objects

Using the request module, header attributes are added to the message to be produced.

```python
#!/usr/bin/env python
import json
from kafka import KafkaProducer
from flask import Flask, request

app = Flask(__name__)
producer = KafkaProducer(bootstrap_servers='kafka:29092')
events_topic = 'game_events'


def log_to_kafka(topic, event):
  event.update(request.headers)
  producer.send(topic, json.dumps(event).encode())


@app.route("/")
def default_response():
  default_event = {'event_type': 'default'}
  log_to_kafka(events_topic, default_event)
  return "\nThis is the default response!\n"


@app.route("/purchase_a_sword")
def purchase_a_sword():
  purchase_sword_event = {'event_type': 'purchase_sword'}
  log_to_kafka(events_topic, purchase_sword_event)
  return "\nSword Purchased!\n"


@app.route("/purchase_a_knife")
def purchase_a_knife():
  purchase_knife_event = {'event_type': 'purchase_knife'}
  log_to_kafka(events_topic, purchase_knife_event)
  return "\nKnife Purchased!\n"


@app.route("/join_guild")
def join_guild():
  join_guild_event = {'event_type': 'join_guild'}
  log_to_kafka(events_topic, join_guild_event)
  return "\nGuild Joined!\n"
```

## 9.  Adding more key value attributes to the json objects

After running the flask server again and making new API calls, the consumed messages with kafkacat can be seen below.

```bash
{"event_type": "default"}
{"event_type": "purchase_sword"}
{"event_type": "default"}
{"event_type": "join_guild"}
{"event_type": "join_guild"}
{"event_type": "purchase_knife"}
{"event_type": "purchase_knife"}
{"event_type": "purchase_sword"}
{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_knife", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "join_guild", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
% Reached end of topic game_events [0] at offset 12: exiting
```

## 10. Consuming messages with PySpark

Running a PySpark shell.

```python
docker-compose exec spark pyspark
```

Using PySpark, consume the kafka topic.

```python
raw_events = spark \
  .read \
  .format("kafka") \
  .option("kafka.bootstrap.servers", "kafka:29092") \
  .option("subscribe","game-events") \
  .option("startingOffsets", "earliest") \
  .option("endingOffsets", "latest") \
  .load()
```

Caching raw events and extracting the json message.

```python
import json
raw_events.cache()
events = raw_events.select(raw_events.value.cast('string'))
extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF()
```

Showing extracted events.

```python
extracted_events.show()
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