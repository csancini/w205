# Tracking User Activity project

Carlos Sancini<br>
Assignment 8<br>
W205 Introduction to Data Engineering<br>
MIDS Summer 2018<br>

# Executive Summary

A new service that provides video book reviews has been created, and now companies like Pearson want to publish their ratings for data scientist analysis.

This project aims to establish a reference implementation of a batch layer of a Lambda architecture. The batch layer will be responsible for landing and structuring data for queries.

Basically, in this setting, three activities are performed to set up the architecture:

- First, create a queue and publish messages with Kafka.
- Second, use Spark to transform the messages.
- Third, land transformed messages in HDFS.

# Big Data Lambda Architecture

The following schematic represents a conceptual view of the Lambda architecture. This project aims to build the batch layer indicated with number 2 in the figure.

![Lambda Architecture](lambda.png)

The cluster for the Batch Layer consists of the following components, all of which are executed by Docker containers:

- Kafka<sup>1</sup>: a distributed streaming platform that has capabilities that provide means to publish/subscribe to record streams, similar to a message queue or corporate messaging system; to store streams of records in a fault-tolerant durable way; and, finally, for processing streams of records as they occur.

- Zookeeper<sup>2</sup>: ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. These services are dependencies of a Kafka distributed application.

- Spark <sup>3</sup>: Spark is a unified analytics engine for large-scale data processing, which is suitable to rapidly query, analyze, and transform data at scale. In this setting Spark will enable interactive queries across large and distributed data sets, as well the processing of streaming data from Kafka.

- Hadoop HDFS <sup>4</sup>: The Hadoop Distributed File System (HDFS) is a distributed file system designed and a highly fault-tolerant environment that is designed to be deployed on low-cost hardware. HDFS provides high throughput access to application data and is suitable for applications that have large data sets. In a Hadoop Distributed File System (HDFS), once a file is closed, it cannot be modified. Hence,  immutability is a core concept of a Big Data architecture.

- Ubuntu: a general-purpose Linux environment (referenced as mids) for setting up Docker and running tools such as ```jq``` and ```vi```.

<sub>1) Definition from kafka.apache.org</sub><br>
<sub>2) Definition from zookeeper.apache.org</sub><br>
<sub>3) Definition from spark.apache.org</sub>
<sub>3) Definition from hadoop.apache.org</sub>

# Step by step Annotations

## 1. Setting up the Kafka and Spark environment

A new a docker-compose.yml file is created to set up the cluster.

```bash
sudo mkdir ~/w205/assignment-08-csancini/spark-with-kafka-and-hdfs
cd ~/w205/assignment-08-csancini/spark-with-kafka-and-hdfs
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
    expose:
      - "9092"
      - "29092"

  cloudera:
    image: midsw205/cdh-minimal:latest
    expose:
      - "8020" # nn
      - "50070" # nn http
      - "8888" # hue
    #ports:
    #- "8888:8888"

  spark:
    image: midsw205/spark-python:0.0.5
    stdin_open: true
    tty: true
    volumes:
      - /home/science/w205:/w205
    command: bash
    depends_on:
      - cloudera
    environment:
      HADOOP_NAMENODE: cloudera

  mids:
    image: midsw205/base:latest
    stdin_open: true
    tty: true
    volumes:
      - /home/science/w205:/w205
```

## 2. Spinning up the cluster

To check whether Kafka is running and to monitor its activity, the log is opened in the terminal.

```bash
docker-compose up -d
docker-compose logs -f kafka
```

## 3. Checking HDFS file system

To check whether Hadoop is running, the command below lists the file structure within HDFS. The Hadoop HDFS is a separate file system from the local Linux file system and the directories are different and have different paths. Files and directories updated in HDFS can take some time to show up due to replication tasks within the cluster. As a consequence, to provide a highly available distributed cluster, Big Data architectures are eventually consistent.

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/
```

```bash
Found 2 items
drwxrwxrwt   - mapred   mapred            0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root     supergroup        0 2018-07-07 20:30 /tmp/hive
```

## 4. Creating the Kafka topic

A topic called ```video-books-assessments``` is created with the first command and checked in Kafka with the second command.

```bash
docker-compose exec kafka \
    kafka-topics \
      --create \
      --topic video-books-assessments \
      --partitions 1 \
      --replication-factor 1 \
      --if-not-exists \
      --zookeeper zookeeper:32181 \
      --config max.message.bytes=5242880

docker-compose exec kafka \
  kafka-topics \
    --describe \
    --topic video-books-assessments \
    --zookeeper zookeeper:32181
```

The following output is obtained:

```bash
Topic:video-books-assessments   PartitionCount:1        ReplicationFactor:1     Configs:max.message.bytes=5242880
        Topic: video-books-assessments  Partition: 0    Leader: 1       Replicas: 1     Isr: 1
```

## 5. Getting and checking the data

The file with the input data is downloaded and analyzed.

```bash
docker-compose exec mids bash -c \
  "curl -L -o assignment-08-csancini/assessment-attempts-20180128-121051-nested.json \
  https://goo.gl/f5bRm4"

docker-compose exec mids bash -c \
  "ls -l --block-size=M \
  assignment-08-csancini/assessment-attempts-20180128-121051-nested.json"

docker-compose exec mids bash -c \
  "cat assignment-08-csancini/assessment-attempts-20180128-121051-nested.json | \
  jq '.[0]'"

docker-compose exec mids bash -c \
  "cat assignment-08-csancini/assessment-attempts-20180128-121051-nested.json | \
  jq '. | length'"
  
docker-compose exec mids bash -c \
  "cat assignment-08-csancini/assessment-attempts-20180128-121051-nested.json | \
  jq '.[].exam_name' | sort | uniq | sort -g"

docker-compose exec mids bash -c \
  "cat assignment-08-csancini/assessment-attempts-20180128-121051-nested.json | \
  jq  '.[].exam_name' | sort | uniq | wc -l"
```

The json file seems to refer to users' self-assessments about a variety of O'Reilly video books. There are 3280 assessment entries related to 103 different video books. The most requested ones with over 100 assessments are: Software Architecture Fundamentals Understanding the Basics, Introduction to Machine Learning, Learning to Program with R, Intermediate Python Programming, Introduction to Java 8, Introduction to Python, and Learning Git.

The json is a large file, so message publishing will be split using jq in the next section. In addition, it was included in the docker-compose.yml a configuration parameter for Kafka that increases the size of the message to 5MB:  ```KAFKA_MESSAGE_MAX_BYTES = 5242880```.

## 6. Publishing first subset of messages using the kafkacat utility

Publishing 1000 messages at a time to Kafka. The -c option in jq is used to remove the outer square bracket wrapper and transform the json to a compact format.

```bash

docker-compose exec mids \
  bash -c "cat /w205/assignment-08-csancini/assessment-attempts-20180128-121051-nested.json \
    | jq '.[0:1000][]' -c \
    | kafkacat -P -b kafka:29092 -t video-books-assessments commits && echo 'Produced 1000 messages.'"

docker-compose exec mids \
  bash -c "cat /w205/assignment-08-csancini/assessment-attempts-20180128-121051-nested.json \
  | jq '.[1000:2000][]' -c \
  | kafkacat -P -b kafka:29092 -t video-books-assessments commits && echo 'Produced 1000 messages.'"

docker-compose exec mids \
  bash -c "cat /w205/assignment-08-csancini/assessment-attempts-20180128-121051-nested.json \
  | jq '.[2000:3000][]' -c \
  | kafkacat -P -b kafka:29092 -t video-books-assessments commits && echo 'Produced 1000 messages.'"  

docker-compose exec mids \
  bash -c "cat /w205/assignment-08-csancini/assessment-attempts-20180128-121051-nested.json \
  | jq '.[3000:3279][]' -c \
  | kafkacat -P -b kafka:29092 -t video-books-assessments commits && echo 'Produced 279 messages.'"
```

The last json structure will be left for a second publishing step in which the consumer will not start from the beginning but consume the message incrementally.

## 7. Consuming the first subset of messages using PySpark

PySpark is initialized in a new terminal window with the following command:

```bash
docker-compose exec spark pyspark
```

The subset of messages is then consumed from ```video-books-assessments``` kafka topic in PySpark from the beginning of time. In addition, a configuration parameter ```kafka.max.partition.fetch.bytes``` was set to enable PySpark consume a large message.

```PySpark
raw_messages = spark \
  .read \
  .format("kafka") \
  .option("kafka.bootstrap.servers", "kafka:29092") \
  .option("subscribe","video-books-assessments") \
  .option("startingOffsets", "earliest") \
  .option("endingOffsets", "latest") \
  .option("kafka.max.partition.fetch.bytes","5242880")
  .load()
```

The next code load the DataFrame and avoids warning messages. This is due to Spark's use of "lazy evaluation" which is a characteristic of a Big Data architecture (along with immutability and eventual consistency).

```PySpark
raw_messages.cache()
DataFrame[key: binary, value: binary, topic: string, partition: int, offset: bigint, timestamp: timestamp, timestampType: int]
```

Here the DataFrame schema is shown. This reflects Kafka queue schema, but the message itself is contained in the ```value``` field and its content is in binary format. A count method is also called to confirm that the last message is still missing.

```PySpark
raw_messages.printSchema()
root
 |-- key: binary (nullable = true)
 |-- value: binary (nullable = true)
 |-- topic: string (nullable = true)
 |-- partition: integer (nullable = true)
 |-- offset: long (nullable = true)
 |-- timestamp: timestamp (nullable = true)
 |-- timestampType: integer (nullable = true)
```

```PySpark
raw_messages.count()
3279
```

```PySpark
raw_messages.show(5)  
+----+--------------------+--------------------+---------+------+--------------------+-------------+  
| key|               value|               topic|partition|offset|           timestamp|timestampType|  
+----+--------------------+--------------------+---------+------+--------------------+-------------+  
|null|[7B 22 6B 65 65 6...|video-books-asses...|        0|     0|1969-12-31 23:59:...|            0|  
|null|[7B 22 6B 65 65 6...|video-books-asses...|        0|     1|1969-12-31 23:59:...|            0|  
|null|[7B 22 6B 65 65 6...|video-books-asses...|        0|     2|1969-12-31 23:59:...|            0|  
|null|[7B 22 6B 65 65 6...|video-books-asses...|        0|     3|1969-12-31 23:59:...|            0|  
|null|[7B 22 6B 65 65 6...|video-books-asses...|        0|     4|1969-12-31 23:59:...|            0|  
+----+--------------------+--------------------+---------+------+--------------------+-------------+  
only showing top 5 rows
```

To change the message value from binary to string, a new data frame is derived from the original one.

```PySpark
messages = raw_messages.select(raw_messages.value.cast('string'))

messages.printSchema()

root  
 |-- value: string (nullable = true)  

messages.show(5)

+--------------------+  
|               value|  
+--------------------+  
|{"keen_timestamp"...|  
|{"keen_timestamp"...|  
|{"keen_timestamp"...|  
|{"keen_timestamp"...|  
|{"keen_timestamp"...|  
+--------------------+  
```

Then, the first json message content is inspected with pprint so that the complex schema can be more easily understood.

```PySpark
import json
from pprint import pprint as pp

sample_message = json.loads(messages.select('value').take(1)[0].value)
pp(sample_message)
```

```PySpark
{'base_exam_id': '37f0a30a-7464-11e6-aa92-a8667f27e5dc',
 'certification': 'false',
 'exam_name': 'Normal Forms and All That Jazz Master Class',
 'keen_created_at': '1516717442.735266',
 'keen_id': '5a6745820eb8ab00016be1f1',
 'keen_timestamp': '1516717442.735266',
 'max_attempts': '1.0',
 'sequences': {'attempt': 1,
               'counts': {'all_correct': False,
                          'correct': 2,
                          'incomplete': 1,
                          'incorrect': 1,
                          'submitted': 4,
                          'total': 4,
                          'unanswered': 0},
               'id': '5b28a462-7a3b-42e0-b508-09f3906d1703',
               'questions': [{'id': '7a2ed6d3-f492-49b3-b8aa-d080a8aad986',
                              'options': [{'at': '2018-01-23T14:23:24.670Z',
                                           'checked': True,
                                           'correct': True,
                                           'id': '49c574b4-5c82-4ffd-9bd1-c3358faf850d',
                                           'submitted': 1},
                                          {'at': '2018-01-23T14:23:25.914Z',
                                           'checked': True,
                                           'correct': True,
                                           'id': 'f2528210-35c3-4320-acf3-9056567ea19f',
                                           'submitted': 1},
                                          {'checked': False,
                                           'correct': True,
                                           'id': 'd1bf026f-554f-4543-bdd2-54dcf105b826'}],
                              'user_correct': False,
                              'user_incomplete': True,
                              'user_result': 'missed_some',
                              'user_submitted': True},
                             {'id': 'bbed4358-999d-4462-9596-bad5173a6ecb',
                              'options': [{'at': '2018-01-23T14:23:30.116Z',
                                           'checked': True,
                                           'id': 'a35d0e80-8c49-415d-b8cb-c21a02627e2b',
                                           'submitted': 1},
                                          {'checked': False,
                                           'correct': True,
                                           'id': 'bccd6e2e-2cef-4c72-8bfa-317db0ac48bb'},
                                          {'at': '2018-01-23T14:23:41.791Z',
                                           'checked': True,
                                           'correct': True,
                                           'id': '7e0b639a-2ef8-4604-b7eb-5018bd81a91b',
                                           'submitted': 1}],
                              'user_correct': False,
                              'user_incomplete': False,
                              'user_result': 'incorrect',
                              'user_submitted': True},
                             {'id': 'e6ad8644-96b1-4617-b37b-a263dded202c',
                              'options': [{'at': '2018-01-23T14:23:52.510Z',
                                           'checked': False,
                                           'id': 'a9333679-de9d-41ff-bb3d-b239d6b95732'},
                                          {'checked': False,
                                           'id': '85795acc-b4b1-4510-bd6e-41648a3553c9'},
                                          {'at': '2018-01-23T14:23:54.223Z',
                                           'checked': True,
                                           'correct': True,
                                           'id': 'c185ecdb-48fb-4edb-ae4e-0204ac7a0909',
                                           'submitted': 1},
                                          {'at': '2018-01-23T14:23:53.862Z',
                                           'checked': True,
                                           'correct': True,
                                           'id': '77a66c83-d001-45cd-9a5a-6bba8eb7389e',
                                           'submitted': 1}],
                              'user_correct': True,
                              'user_incomplete': False,
                              'user_result': 'correct',
                              'user_submitted': True},
                             {'id': '95194331-ac43-454e-83de-ea8913067055',
                              'options': [{'checked': False,
                                           'id': '59b9fc4b-f239-4850-b1f9-912d1fd3ca13'},
                                          {'checked': False,
                                           'id': '2c29e8e8-d4a8-406e-9cdf-de28ec5890fe'},
                                          {'checked': False,
                                           'id': '62feee6e-9b76-4123-bd9e-c0b35126b1f1'},
                                          {'at': '2018-01-23T14:24:00.807Z',
                                           'checked': True,
                                           'correct': True,
                                           'id': '7f13df9c-fcbe-4424-914f-2206f106765c',
                                           'submitted': 1}],
                              'user_correct': True,
                              'user_incomplete': False,
                              'user_result': 'correct',
                              'user_submitted': True}]},
 'started_at': '2018-01-23T14:23:19.082Z',
 'user_exam_id': '6d4089e4-bde5-4a22-b65f-18bce9ab79c8'}
```

## 8. Publishing another message and consuming the last offset in PySpark

In this step, a new message with the last json item is published.

```bash
docker-compose exec mids bash -c "cat /w205/assignment-08-csancini/assessment-attempts-20180128-121051-nested.json | jq '.[3279]'"
"Learning Linux System Administration"
```

```bash
docker-compose exec mids \
  bash -c "cat /w205/assignment-08-csancini/assessment-attempts-20180128-121051-nested.json \
  | jq '.[3279]' -c \
  | kafkacat -P -b kafka:29092 -t video-books-assessments && echo 'Produced 1 message.'"
```

The queue offset is then checked in Kafka.

```bash
docker-compose exec kafka \
  kafka-run-class kafka.tools.GetOffsetShell --broker-list localhost:29092 --topic video-books-assessments --time -1
```

```bash
video-books-assessments:0:3280
```

In the following code, only the last offset is consumed in PySpark from Kafka.

```PySpark
raw_last_message = spark \
  .read \
  .format("kafka") \
  .option("kafka.bootstrap.servers", "kafka:29092") \
  .option("subscribe","video-books-assessments") \
  .option("startingOffsets", """{"video-books-assessments":{"0":3279}}""") \
  .option("endingOffsets", "latest") \
  .option("kafka.max.partition.fetch.bytes","5242880")
  .load()
last_message = raw_last_message.select(raw_last_message.value.cast('string'))
```

To confirm that only the last json item is consumed, the end its content with the exam name is displayed to assert that the last kafka offset was consumed correctly.

```PySpark
last_message.select('value').take(1)[0].value

..."keen_id":"5a3a3df2448eb200012a2efd","exam_name":"Learning Linux System Administration"}'
```

## 9. Joining the data frames

Next, the data frames are joined.

```PySpark
last_message.count()
1
```

```PySpark
messages.count()
3279
```

```PySpark
messages = messages.union(last_message)
messages.count()
3280
```

## 10. Extracting JSON messages and saving them to HDFS

The batch layer is updated with parsed JSON messages that were consumed from Kafka.

```PySpark
from pyspark.sql import Row
extracted_messages = messages.rdd.map(lambda x: Row(**json.loads(x.value))).toDF()
```

Now the data frame reflects the schema of the JSON files published.

```PySpark
extracted_messages.printSchema()
root  
 |-- base_exam_id: string (nullable = true)  
 |-- certification: string (nullable = true)  
 |-- exam_name: string (nullable = true)  
 |-- keen_created_at: string (nullable = true)  
 |-- keen_id: string (nullable = true)  
 |-- keen_timestamp: string (nullable = true)  
 |-- max_attempts: string (nullable = true)  
 |-- sequences: map (nullable = true)  
 |    |-- key: string  
 |    |-- value: array (valueContainsNull = true)  
 |    |    |-- element: map (containsNull = true)  
 |    |    |    |-- key: string  
 |    |    |    |-- value: boolean (valueContainsNull = true)  
 |-- started_at: string (nullable = true)  
 |-- user_exam_id: string (nullable = true)  
```

Landing the JSON content in HDFS in parquet format. As the message content is unprocessed, the "raw" naming convention is used.

```PySpark
extracted_messages.write.parquet("/tmp/raw_assessments")
```

The HDFS directories are inspected to checked whether the data was landed correctly.

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/
```

```HDFS
Found 3 items
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-07-08 14:46 /tmp/hive
drwxr-xr-x   - root   supergroup          0 2018-07-08 15:45 /tmp/raw_assessments
```

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/raw_assessments
```

```HDFS
Found 3 items
-rw-r--r-- 1 root supergroup      0 2018-07-08 15:45 /tmp/raw_assessments/_SUCCESS
-rw-r--r-- 1 root supergroup 345279 2018-07-08 15:45 /tmp/raw_assessments/part-00000-0a3b8438-f0c9-4feb-b7f3-6774dc5f3d59-c000.snappy.parquet
-rw-r--r-- 1 root supergroup   3876 2018-07-08 15:45 /tmp/raw_assessments/part-00001-0a3b8438-f0c9-4feb-b7f3-6774dc5f3d59-c000.snappy.parquet
```

## 11. Using Spark SQL to create summarized views to be consumed from the Batch Layer

The ```assessment``` table is registered from the content in the data frame.

```PySpark
extracted_messages.registerTempTable('assessments')
```

Querying for content. As shown in the second query, the questions for each exam are a nested structure.

```PySpark
spark.sql("SELECT base_exam_id, exam_name, keen_id, user_exam_id FROM assessments").show(5, False)
```

```PySpark
+------------------------------------+-------------------------------------------+------------------------+------------------------------------+  
|base_exam_id                        |exam_name                                  |keen_id                 |user_exam_id                        |  
+------------------------------------+-------------------------------------------+------------------------+------------------------------------+  
|37f0a30a-7464-11e6-aa92-a8667f27e5dc|Normal Forms and All That Jazz Master Class|5a6745820eb8ab00016be1f1|6d4089e4-bde5-4a22-b65f-18bce9ab79c8|  
|37f0a30a-7464-11e6-aa92-a8667f27e5dc|Normal Forms and All That Jazz Master Class|5a674541ab6b0a0001c6e723|2fec1534-b41f-4419-b741-79d372f05cbe|  
|4beeac16-bb83-4d58-83e4-26cdc38f0481|The Principles of Microservices            |5a67999d3ed3e300016ef0f1|8edbc8a8-4d26-4292-a5af-ae3f246cb09f|  
|4beeac16-bb83-4d58-83e4-26cdc38f0481|The Principles of Microservices            |5a6799694fc7c70001034706|c0ee680e-8892-4e64-a7f2-bb576a665dc5|  
|6442707e-7488-11e6-831b-a8667f27e5dc|Introduction to Big Data                   |5a6791e824fccd00018c3ff9|e4525b79-7904-4050-a068-27969b01f6bd|  
+------------------------------------+-------------------------------------------+------------------------+------------------------------------+  
```

```PySpark
spark.sql("SELECT sequences FROM assessments LIMIT 5").show()
```

```PySpark
+--------------------+  
|           sequences|  
+--------------------+  
|Map(questions -> ...|  
|Map(questions -> ...|  
|Map(questions -> ...|  
|Map(questions -> ...|  
|Map(questions -> ...|  
+--------------------+  
```

As a final check, the top assessments are recalculated to ensure that they correspond to those calculated previously with JQ.

```PySpark
spark
  .sql("\
    SELECT DISTINCT exam_name, COUNT(base_exam_id) AS total \
    FROM assessments GROUP BY exam_name HAVING total > 100 ORDER BY total DESC")
  .show(10, False)
```

```PySpark
+-----------------------------------------------------------+-----+  
|exam_name                                                  |total|  
+-----------------------------------------------------------+-----+  
|Learning Git                                               |394  |  
|Introduction to Python                                     |162  |  
|Introduction to Java 8                                     |158  |  
|Intermediate Python Programming                            |158  |  
|Learning to Program with R                                 |128  |  
|Introduction to Machine Learning                           |119  |  
|Software Architecture Fundamentals Understanding the Basics|109  |  
+-----------------------------------------------------------+-----+  
```

To compute a summarized view that involves questions (nested json structures), it is necessary to unroll them with a lambda function and the flatMap in the RDD.

```PySpark
def lambda_unroll_questions(x):
    raw_dict = json.loads(x.value)
    my_list = []
    for l in raw_dict["sequences"]["questions"]:
        if "user_correct" in l:
            my_list.append({"keen_id": raw_dict["keen_id"], "question_id": l["id"], "user_correct": l["user_correct"]})
        else:
            my_list.append({"keen_id": raw_dict["keen_id"], "question_id": l["id"], "user_correct": "false"})
    return my_list

questions = messages.rdd.flatMap(lambda_unroll_questions).toDF()

questions.registerTempTable('questions')

spark.sql("SELECT * FROM questions").show(5, False)
```

A sample from the unrolled structure that originated the ```questions``` table is shown below.

```PySpark
+------------------------+------------------------------------+------------+  
|keen_id                 |question_id                         |user_correct|  
+------------------------+------------------------------------+------------+  
|5a6745820eb8ab00016be1f1|7a2ed6d3-f492-49b3-b8aa-d080a8aad986|false       |  
|5a6745820eb8ab00016be1f1|bbed4358-999d-4462-9596-bad5173a6ecb|false       |  
|5a6745820eb8ab00016be1f1|e6ad8644-96b1-4617-b37b-a263dded202c|true        |  
|5a6745820eb8ab00016be1f1|95194331-ac43-454e-83de-ea8913067055|true        |  
|5a674541ab6b0a0001c6e723|95194331-ac43-454e-83de-ea8913067055|true        |  
+------------------------+------------------------------------+------------+  
```

Lastly, a join between ```assessments``` and ```questions``` tables is made to create the final summarized view that is landed in HDFS.

```PySpark
assessments_summary = spark.sql("SELECT DISTINCT a.exam_name, count(a.base_exam_id) counts, b.question_id, b.user_correct FROM assessments a JOIN questions b ON a.keen_id = b.keen_id GROUP BY exam_name, question_id, user_correct ORDER BY exam_name, question_id")

assessments_summary.show(10, False)
```

```PySpark
+------------------------------------+------+------------------------------------+------------+
|exam_name                           |counts|question_id                         |user_correct|
+------------------------------------+------+------------------------------------+------------+
|A Practical Introduction to React.js|3     |7ff41d87-1f73-4068-9889-c4f82cd935b4|false       |
|A Practical Introduction to React.js|6     |7ff41d87-1f73-4068-9889-c4f82cd935b4|true        |
|A Practical Introduction to React.js|3     |80aad87e-c7b2-4e11-8687-c284c0234e0e|false       |
|A Practical Introduction to React.js|6     |80aad87e-c7b2-4e11-8687-c284c0234e0e|true        |
|A Practical Introduction to React.js|3     |a6effaf7-94ba-4586-aaa4-5c15eecb1ece|true        |
|A Practical Introduction to React.js|6     |a6effaf7-94ba-4586-aaa4-5c15eecb1ece|false       |
|A Practical Introduction to React.js|3     |dab47905-63c6-46bc-97cb-99b7f3eb5877|false       |
|A Practical Introduction to React.js|6     |dab47905-63c6-46bc-97cb-99b7f3eb5877|true        |
|Advanced Machine Learning           |60    |2a3e4766-6bd0-11e6-8d56-a8667f27e5dc|true        |
|Advanced Machine Learning           |7     |2a3e4766-6bd0-11e6-8d56-a8667f27e5dc|false       |
+------------------------------------+------+------------------------------------+------------+ 
```

```PySpark
assessments_summary.write.parquet("/tmp/summary")
```

The files are then check in HDFS.

```bash
docker-compose exec cloudera hadoop fs -ls /tmp/summary
```

```HDFS
Found 201 items
-rw-r--r--   1 root supergroup     0 2018-07-09 03:04 /tmp/summary/_SUCCESS
-rw-r--r--   1 root supergroup  1460 2018-07-09 03:04 /tmp/summary/part-00000-0b1cc1de-981d-4af0-8d33-8ba50a7e08b5-c000.snappy.parquet
-rw-r--r--   1 root supergroup  1413 2018-07-09 03:04 /tmp/summary/part-00001-0b1cc1de-981d-4af0-8d33-8ba50a7e08b5-c000.snappy.parquet
-rw-r--r--   1 root supergroup  1344 2018-07-09 03:04 /tmp/summary/part-00002-0b1cc1de-981d-4af0-8d33-8ba50a7e08b5-c000.snappy.parquet
-rw-r--r--   1 root supergroup  1439 2018-07-09 03:04 /tmp/summary/part-00003-0b1cc1de-981d-4af0-8d33-8ba50a7e08b5-c000.snappy.parquet
...
```

## 12. Tearing the cluster down

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
- Combining two DataFrames in PySpark;
- Unrolling nested json data structures in PySpark with RDD flatMap;
- Querying two data frames using the JOIN clause in SparkSQL.
