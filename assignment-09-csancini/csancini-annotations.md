# Project 3 Setup

Carlos Sancini<br>
Assignment 9<br>
W205 Introduction to Data Engineering<br>
MIDS Summer 2018<br>

## Executive Summary

A game development company has launched a new game for which there are two user events that are particularly interesting in tracking, such as buy a sword, buy a knife and join a guild (each of the them has metadata). To process these user actions, the mobile app makes API calls to a web-based API-server. The API server for the game will be instrumented in order to catch and analyze these two event types.

This project aims to establish a reference implementation of a speed layer of a Lambda architecture. The speed layer will be responsible for capturing and acting upon the mentioned events. A Python Flask module will be used to write a simple API server.

Basically, in this setting, three activities are performed to set up the stack:

- First, creating a queue in Kafka to publish messages.
- Second, setting up a Flask web server with a web API that receives requests and publishes messages to Kafka.
- Third, send requests to the API and check messages in Kafka.

## Big Data Lambda Architecture

The following schematic represents a conceptual view of the Lambda architecture. This project aims to build the batch layer indicated with number 2 in the figure.

![Lambda Architecture](lambda.png)

The cluster for the Batch Layer consists of the following components, all of which are executed by Docker containers:

- Kafka<sup>1</sup>: a distributed streaming platform that has capabilities that provide means to publish/subscribe to record streams, similar to a message queue or corporate messaging system; to store streams of records in a fault-tolerant durable way; and, finally, for processing streams of records as they occur.

- Zookeeper<sup>2</sup>: ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. These services are dependencies of a Kafka distributed application.

- Ubuntu: a general-purpose Linux environment (referenced as mids) for setting up Docker, setting up Flask and running tools such as ```jq``` and ```vi```.

<sub>1) Definition from kafka.apache.org</sub><br>
<sub>2) Definition from zookeeper.apache.org</sub><br>

## Step by step Annotations

## 1. Setting up the Kafka environment

A new a docker-compose.yml file is created to set up the cluster.

```bash
sudo mkdir ~/w205/assignment-09-csancini/flask-with-kafka
cd ~/w205/assignment-09-csancini/flask-with-kafka
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

A topic called ```game_users_events``` is created with the first command and checked in Kafka with the second command.

```bash
docker-compose exec kafka \
    kafka-topics \
      --create \
      --topic game-users-events \
      --partitions 1 \
      --replication-factor 1 \
      --if-not-exists \
      --zookeeper zookeeper:32181

docker-compose exec kafka \
  kafka-topics \
    --describe \
    --topic game_users_events \
    --zookeeper zookeeper:32181
```

The following output is obtained:

```bash
Topic:game-users-events PartitionCount:1        ReplicationFactor:1     Configs:
        Topic: game-users-events        Partition: 0    Leader: 1       Replicas: 1     Isr: 1
```

## 4. Creating a simple API server

The simple API server is created below with the Python Flask library.

```python
#!/usr/bin/env python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def default_response():
    return "\nThis is the default response!\n"

@app.route("/purchase_a_sword")
def purchase_sword():
    # business logic to purchase sword
    return "\nSword Purchased!\n"

@app.route("/purchase_a_knife")
def purchase_knife():
    # business logic to purchase knife
    return "\nKnife Purchased!\n"

@app.route("/join_guild")
def join_guild():
    # business logic to join guild
    return "\nGuild Joined!\n"
```

## 5. Running the python script and calling the API

The python script is executed in the following command. It will hold the Linux terminal and the output from the python program will be shown as API calls are made.

```bash
docker-compose exec mids env FLASK_APP=/w205/assignment-09-csancini/flask-with-kafka/game_api.py flask run
```

```bash
 * Serving Flask app "game_api"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
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
127.0.0.1 - - [14/Jul/2018 18:01:16] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:01:27] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:01:34] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:01:41] "GET /join_guild HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:01:46] "GET /join_guild HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:01:53] "GET /purchase_a_knife HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:01:55] "GET /purchase_a_knife HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:02:09] "GET /purchase_a_sword HTTP/1.1" 200 -
```

## 6. Adding to the API a KafkaProducer to log user events to Kafka

Next, a ```KafkaProducer``` is created to log user events.

```python
#!/usr/bin/env python
from kafka import KafkaProducer
from flask import Flask
app = Flask(__name__)
event_logger = KafkaProducer(bootstrap_servers='kafka:29092')
events_topic = 'game_users_events'

@app.route("/")
def default_response():
    event_logger.send(events_topic, 'default'.encode())
    return "\nThis is the default response!\n"

@app.route("/purchase_a_sword")
def purchase_sword():
    # business logic to purchase sword
    # log event to kafka
    event_logger.send(events_topic, 'purchased_sword'.encode())
    return "\nSword Purchased!\n"

@app.route("/purchase_a_knife")
def purchase_knife():
    # business logic to purchase knife
    # log event to kafka
    event_logger.send(events_topic, 'purchased_knife'.encode())
    return "\nKnife Purchased!\n"

@app.route("/join_guild")
def join_guild():
    # business logic to join guild
    # log event to kafka
    event_logger.send(events_topic, 'joined_guild'.encode())
    return "\nGuild Joined!\n"
```

## 7. Publishing user events to Kafka

The same the python script is re-executed.

```bash
docker-compose exec mids env FLASK_APP=/w205/assignment-09-csancini/flask-with-kafka/game_api.py flask run
```

```bash
 * Serving Flask app "game_api"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

The same calls are made to the API, but now it will publish the events to Kafka.

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
 * Serving Flask app "game_api"
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
127.0.0.1 - - [14/Jul/2018 18:12:31] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:12:34] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:12:39] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:12:46] "GET /join_guild HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:12:49] "GET /join_guild HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:13:01] "GET /purchase_a_knife HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:13:03] "GET /purchase_a_knife HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2018 18:13:07] "GET /purchase_a_sword HTTP/1.1" 200 -
```

## 8. Consuming messages using the kafkacat utility

Lastly, the messages are consumed from the Kafka topic.

```bash
docker-compose exec mids bash -c "kafkacat -C -b kafka:29092 -t game_users_events -o beginning -e"
```

```bash
default
purchased_sword
default
joined_guild
joined_guild
purchased_knife
purchased_knife
purchased_sword
% Reached end of topic game_users_events [0] at offset 8: exiting
```

## 9. Tearing the cluster down

The cluster is shut down and docker processes are checked.

```bash
docker-compose down
docker-compose ps
docker ps -a
```

## 10. Enhancements Appendix

The following enhancements were considered during this assignment:

- Executive Summary and Architecture description;
- More meaningful topic name: game_users_events;
- Additional API services: ```purchase_a_knife()``` and ```join_guild()```;