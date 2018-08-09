# Assignment 6: Annotations

## 1. Setting up the kafka environment

A new directory and a docker-compose.yml file are create to set up the cluster.

```bash
mkdir w205/assignment-06-csancini/kafka
cd w205/assignment-06-csancini/kafka
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
    #ports:
      #- "32181:32181"
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
      KAFKA_MESSAGE_MAX_BYTES: 15000000
    volumes:
      - /home/science/w205:/w205
    expose:
      - "9092"
      - "29092"
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

To check whether kafka is running and to monitor its activity, the log is opened in the terminal.

```bash
docker-compose up -d
docker-compose logs -f kafka
```

## 3. Creating the kafka topic

A topic called ```video-books-assessments``` is created, with first command, and checked in kafka with the second command.

```bash
docker-compose exec kafka \
    kafka-topics \
      --create \
      --topic video-books-assessments \
      --partitions 1 \
      --replication-factor 1 \
      --if-not-exists \
      --zookeeper zookeeper:32181

docker-compose exec kafka \
  kafka-topics \
    --describe \
    --topic video-books-assessments \
    --zookeeper zookeeper:32181
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

The json is a large file, so message publishing will be split using jq in the next section. In addition, it was included in the docker-compose.yml a configuration parameter for kafka that increases the size of the message:  ```KAFKA_MESSAGE_MAX_BYTES = 15000000```.

## 5. Publishing first subset of messages using the kafkacat utility

Publishing 1000 messages at a time to kafka.

```bash
docker-compose exec mids bash -c "cat /w205/assignment-06-csancini/kafka/assessment-attempts-20180128-121051-nested.json | jq '.[0:1000]' -c | kafkacat -P -b kafka:29092 -t video-books-assessments && echo 'Produced 1000 messages.'"

docker-compose exec mids bash -c "cat /w205/assignment-06-csancini/kafka/assessment-attempts-20180128-121051-nested.json | jq '.[1000:2000]' -c | kafkacat -P -b kafka:29092 -t video-books-assessments && echo 'Produced 1000 messages.'"

docker-compose exec mids bash -c "cat /w205/assignment-06-csancini/kafka/assessment-attempts-20180128-121051-nested.json | jq '.[2000:3000]' -c | kafkacat -P -b kafka:29092 -t video-books-assessments && echo 'Produced 1000 messages.'"
```

The last 280 will be left for a second publishing step in which the consumer will not start from the beginning but consume incrementally the messages.

## 6. Consuming first subset of messages using kafka-console-consumer

The messages will be consumed from the beginning of time. The kafkcat utility was not used because it was slow to process the large size messages.

```bash
docker-compose exec kafka kafka-console-consumer --bootstrap-server kafka:29092 --topic video-books-assessments --from-beginning
```

## 7. Publishin and consuming the second subset of messages using the kafkacat utility

In this step, a new message is published but only the last kafka offset is consumed.

```bash
docker-compose exec mids bash -c "cat /w205/assignment-06-csancini/kafka/assessment-attempts-20180128-121051-nested.json | jq '.[3000:3280]' -c | kafkacat -P -b kafka:29092 -t video-books-assessments && echo 'Produced 280 messages.'"

docker-compose exec kafka kafka-console-consumer --bootstrap-server kafka:29092 --topic video-books-assessments --partition 0 --offset 3
```

## 8. Tearing the cluster down

The cluster is shut down and docker processed are checked.

```bash
docker-compose down
docker-compose ps
docker ps -a
```

## Enhancements Appendix

The following enhancements were considered during this assignment:
- More meaningful queue name;
- Publishing the json file in parts;
- Reading the queue from a given offset;
- JSON file analysis.