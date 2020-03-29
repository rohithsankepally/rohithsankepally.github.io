---
layout: post
title: Getting Started With Apache Kafka - Internals
blog : true
published: false
date: March 12, 2020
comments: true
---

In my previous [post]({% post_url 2020-03-01-Kafka-Hands-On %}), we discussed how different Kafka components work together through a hands on. But if you want an in depth understanding of Kafka, knowledge about the Kafka storage is very important. In continuation to the previous post, in this blog post, we will deal with the following

1. Storage & retrieval of topic messages
2. Storage & retrieval consumer offsets

> Note : Before you start reading this post, I would strongly recommend you read and try the hands on in my previous [post]({% post_url 2020-03-01-Kafka-Hands-On %}).

## Exploring the Storage

### Server Properties

In my previous [post]({% post_url 2020-03-01-Kafka-Hands-On %}), we started our brokers using configuration files namely `broker-0.properties` & `broker-1.properties`. These configuration files contain important parameters that control different components of our kafka cluster. You can find the list of the broker parameters in the Kafka's official [documentation](https://kafka.apache.org/documentation/#brokerconfigs). Trying to understand each and every parameter in this file can be a bit overwhelming at the moment. For now, let us try to focus only on the most important parameters defined in our configuration files.

| -----------+----------------------------------------------------+-----------+----------+
| Parameter  |             Description                            |  *broker-0*       | *broker-1* |
| -----------+----------------------------------------------------+-----------+----------+
|   broker.id| A unique identifier for each broker | 0 | 1
|    port    | Broker runs on this port | 9092 | 9093
| zookeeper.connect | IP and port of running zookeeper instance | localhost:2181 |localhost:2181
| log.dirs   | Directory where the messages of a topic are stored | `logs/kafka-logs/broker0` | `logs/kafka-logs/broker1`

Among these our focus will be mainly on understanding the directory structure specified by the `log.dirs` parameter.

### Topic Storage

The `log.dirs` properties determines the location where the topic messages are stored. Additionally, this directory contains other files which help us gain insights into the internal storage structure of kafka. In this section we will go through this directory structure and understand how kafka manages data internally.

#### Partitions & Segments

According to the [architecture](({% post_url 2020-02-29-Kafka-Architecture %})) of kafka, topics are divided into partitions which are stored in brokers. This might lead us to think that partition is the standard storage unit of a topic in kafka. But however this is not true. Partitions are further divided into smaller units called `segments` which inturn contain the topic data.

Before we look at the log directories, let us generate some data by write some messages to our `cars` topic. To do so make sure that you have the working environment setup(discussed in the previous [post]({% post_url 2020-03-01-Kafka-Hands-On %})) upto a point where you can start producing messages. Now we write the messages from `cars.txt` to the `cars ` topic.





* directory structure*

Now lets take a look at the log directory and see how the above written messages are being stored. As discussed in the previous post, replicas of both the partitions(`cars-0` & `cars-1`) of the topic `cars` are stored in *broker-0*. We can see a directory corresponding to each these replicas. In addition to this, we also see some other files which will not be discussed in this post. Without further ado, let us now quickly look into the directory structure of `cars-0` stored in *broker-0*. The directory contains 6 files namely

* *00000000000000000000.log*
* *00000000000000000000.index*
* *00000000000000000000.timeindex*
* *00000000000000000000.log*
* *00000000000000000000.index*
* *00000000000000000000.timeindex*

We can see that the files are grouped based on their names.  


The content in these files is not readable as such. Let us try to understand each one of them separately.

#### The '.log' File

The log file (\*.log) is the file where the contents of a message are stored. There are tools which can convert this file into a human readable format. For instance, to read the contents of the *00000000000000000000.log* file, execute the command below

```bash
kafka-run-class kafka.tools.DumpLogSegments --deep-iteration --print-data-log --files logs/kafka-logs/broker-0/cars-1/00000000000000000000.log
```

`Dumping logs/kafka-logs/broker-0/cars-1/00000000000000000000.log`<br>
`Starting offset: 0`<br>
`baseOffset: 0 lastOffset: 1 count: 2 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false position: 0 CreateTime: 1584931132392 size: 83 magic: 2 compresscodec: NONE crc: 2968353941 isvalid: true
| offset: 0 CreateTime: 1584931132372 keysize: -1 valuesize: 8 sequence: -1 headerKeys: [] payload: mercedes`

We can see some messages along with their offsets are found in this file. The message details of every message that is written to a partition gets appended to this file. 

#### The '.index' file

As discussed [before](), every consumer has an associated numeric offset for every partition that it is reading from. This offset indicates the offset of the last processed message in a partition. So, for any consumer, extracting a message given an offset would be a very frequent operation. We know that every message along with its offset is being stored in the `.log` file. One possible way to achieve this would be to iterate over the contents of the `.log` file and filter out the message with the given offset. But this is not efficient as the size of the `.log` file may become very large over time, and processing the entire file would be a tedious task. To overcome this, kafka uses the index file (*.index*). The index file stores a mapping of offset and position in the file where the message corresponding to this offset is present. Let us check out the contents of the index file.

```bash
kafka-run-class kafka.tools.DumpLogSegments --deep-iteration --print-data-log --files logs/kafka-logs/broker-0/cars-1/00000000000000000000.index
```
`Dumping logs/kafka-logs/broker-0/cars-1/00000000000000000000.index
offset: 0 position: 0`

Given such mapping, we can easily figure out the position in file given an offset using binary search. But wait, what if we do store this mapping for every offset? This would lead to increase in the size of `.index` file. Also the time taken to get the position in the file given an offset would also increase. Keeping this in mind, kafka introduced a parameter named `index.interval.bytes`. This parameter depicts the memory interval(in bytes) after which an index entry would be added. By tuning this parameter, we can control the number of entries in the `.index` file and thereby the size of the `.index` file.


### Consumer Offset Storage

As discussed before, every consumer needs to store a numeric offset indicating the last processed message of partition(s) it is consuming. But, where does kafka store this numeric offset?

Before Kafka 0.9, zookeeper supported the storage & retrieval of offsets associated to each consumer within the consumer group. Due to scalability issues with zookeeper, kafka now started storing this information in a topic named `__consumer_offsets`. The number of paritions and replication factor of this topic can be controlled by `offsets.topic.num.partitions` & `offsets.topic.replication.factor` respectively. In this working example, we set the values of these parameters as follows

`offsets.topic.num.partitions=1` <br>
`offsets.topic.replication.factor=2`

Let us now see the content of messages in this topic. To do so, we start a consumer which consumes messages from this topic using a formatter. 

```bash
docker run --network=hands-on_kafka-tier -ti bitnami/kafka:latest kafka-console-consumer.sh --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" --bootstrap-server broker-0:9092,broker-1:9093 --topic __consumer_offsets
```
By executing the above mentioned command, you will see an output as shown below.

`
[car-group,cars,0]::OffsetAndMetadata(offset=2, leaderEpoch=Optional[0], metadata=, commitTimestamp=1585019671259, expireTimestamp=None)` <br>
`[car-group,cars,1]::OffsetAndMetadata(offset=1, leaderEpoch=Optional[0], metadata=, commitTimestamp=1585019671476, expireTimestamp=None)`

Each message in the topic is of the following format

`[consumer-group, topic-name, partition-number]::OffsetAndMetadata(offset=consumer-offset, leaderEpoch=Optional[0], metadata=, commitTimestamp=1585019671259, expireTimestamp=None)
`

By looking at the first entry, it is clear that kafka is storing the numeric offset(=2) for consumer that belongs to a consumer group(=car-group), consuming data from its assigned partition(=0). 


## Conclusion

In this blog post we focused only on the most important parameters of the broker configuration file. 


