---
layout: post
title: Deep Dive Into Apache Kafka | Storage Internals
blog : true
published: true
date: March 12, 2020
comments: true
---

In my previous [post]({% post_url 2020-03-01-Kafka-Hands-On %}), we discussed about how different Kafka components work together through a hands on. But if you are looking for in depth understanding of Kafka, knowledge about its storage is very important. So, what do I mean by storage? Well, when I was doing a hands on in my previous post, I was curious to find out how Kafka stores & retrieves messages. Additionally, it was interesting to figure out how Kafka stores consumer offsets for all the partition(s). So in this blog post, I tried to the explain the storage internals of Apache Kafka in a simple and practical way. Specifically, we will be discussing about the storage and retrieval of 

1. Topic Messages
2. Consumer Offsets

At the end of this blog post, you will have a good understanding of storage architecture of Apache Kafka. In addition to this, there are a few pointers around how to tune Kafka storage based on your requirements. Without further ado, lets get started.

> Note : Before you start reading this post, I would strongly recommend you read & try the hands on in my previous [post]({% post_url 2020-03-01-Kafka-Hands-On %}).

## Topic Storage

If you had a chance to try the hands on in my previous [post]({% post_url 2020-03-01-Kafka-Hands-On %}), you will notice that the brokers were started using configuration files namely `broker-0.properties` & `broker-1.properties`. These configuration files contain important parameters that control different components of our Kafka cluster. You can find the complete list of the broker parameters in the Kafka's official [documentation](https://kafka.apache.org/documentation/#brokerconfigs). Trying to understand each and every parameter in this file can be a bit overwhelming at the moment. For now, let us try to focus only on the most important parameters defined in these configuration files.

| -----------+----------------------------------------------------+-----------+----------+
| Parameter  |             Description                            |  *broker-0*       | *broker-1* |
| -----------+----------------------------------------------------+-----------+----------+
|   broker.id| A unique identifier for each broker | 0 | 1
|    port    | Broker runs on this port | 9092 | 9093
| zookeeper.connect | IP and port of running zookeeper instance | localhost:2181 |localhost:2181
| log.dirs   | Directory where the messages of a topic are stored | `/tmp/logs/kafka-logs/broker-0` | `/tmp/logs/kafka-logs/broker-1`

Among these parameters, our focus will be mainly on understanding the directory structure specified by the `log.dirs` parameter. The `log.dirs` parameter indicates the location of the folder/directory where the topic messages are stored. This directory has a beautiful structure which captures the storage architecture of Kafka.

### Generating data

Before we look at the log directory, let us generate some data by writing some messages to our topic. To do so, we first make sure that we setup the working environment setup as discussed in the previous [post]({% post_url 2020-03-01-Kafka-Hands-On %}). The code used in this blog post on is available in this [repository](https://github.com/rohithsankepally/apache-kafka-blog). Please make sure that you clone this repository in your working environment before you proceed further. To make the setup simple, I wrote a bash script [`setup.sh`](https://github.com/rohithsankepally/apache-kafka-blog/blob/master/hands-on/setup.sh) with all the required commands. Navigate to the `hands-on` directory & execute this command

```bash
cd hands-on/
sh setup.sh
```	

This should do the cluster setup & launch the consumer instances. We now want to store topic data by producing some messages. To avoid the manual generation of messages, I have written another bash script [`produce.sh`](https://github.com/rohithsankepally/apache-kafka-blog/blob/master/hands-on/produce.sh) which writes data from the [cars.txt](https://github.com/rohithsankepally/apache-kafka-blog/blob/master/hands-on/data/cars.txt) to `cars` topic. Now execute the command given below.

```bash
sh produce.sh
```

This might take a few seconds to complete. After successful execution of the above command, there should be enough data stored in the Kafka cluster for us to proceed further. Let's now go through the directory structure and practically see how Kafka manages data internally.

### Segments

According to the Kafka [architecture]({% post_url 2020-02-29-Kafka-Architecture %}), topics are divided into partitions which are stored in brokers. This might lead us to think that partition is the standard unit of a topic storage in Kafka. But however, this doesn't seem to be true. Partitions are further divided into smaller elements called **segments** which store the topic data. 

Let us see how segments are represented in Kafka's storage structure. From the configuration of `broker-0` we know that the data of `cars-0`(0th partition of topic `cars`) is stored at`/tmp/logs/kafka-logs/broker-0/cars-0`. So lets do a `ls` to see whats in there. 

```bash
docker exec -it broker-0 /bin/bash -c 'ls /tmp/logs/kafka-logs/broker-0/cars-0'
```
```
*00000000000000000000.log*
*00000000000000000000.index*
*00000000000000000000.timeindex*
*00000000000000000035.log*
*00000000000000000035.index*
*00000000000000000035.timeindex* 
.
.
.
```

We can see groups of files(with same name) with each group having a `.log`, `.index` & `.timeindex` files. Each of above groups of files represents a `segment`.

But, you might wonder, when and how does Kafka know that it has to create a new segment. For this, Kafka relies on a parameter named `log.segment.bytes` which indicates the maximum size(in bytes) of a segment.

<p align="center">
<img src="https://media.giphy.com/media/RMePLVcnCRW1NAvIPG/giphy.gif" height="250" width="400">
</p>

Whenever messages are written to a partition, Kafka writes these messages to a segment until it reaches the limit specified by `log.segment.bytes`. Once this limit is reached, Kafka creates a new segment within the partition & starts writing messages to the new segment. The suffix in the segment's file name, indicates the *base offset* (offset of the first messsage) of the segment. For example,

`*00000000000000000000.log* - Contains messages where 0 <= offset < 35.` <br>
`*00000000000000000035.log* - Contains messages where offset >= 35.`<br>

If you try to open and read these files(`.log`, `.index`, `.timeindex`), you will find that the content in these files is not readable. To get a practical understanding, we will go through each one of these files separately and explore their contents.

### The '.log' file

As the name suggests, every message written to a partition is logged in the *.log* file. In order to read the contents of a *.log* file, we use the command shown below

```bash
docker exec -it broker-0 kafka-run-class.sh kafka.tools.DumpLogSegments --deep-iteration --print-data-log --files /tmp/logs/kafka-logs/broker-0/cars-0/00000000000000000000.log
```

```
Dumping /tmp/logs/kafka-logs/broker-0/cars-0/00000000000000000000.log
Starting offset: 0
baseOffset: 0 lastOffset: 6 count: 7 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false position: 0 CreateTime: 1586329540137 size: 173 magic: 2 compresscodec: NONE crc: 386807681 isvalid: true
| offset: 0 CreateTime: 1586329540133 keysize: 1 valuesize: 3 sequence: -1 headerKeys: [] key: 2 payload: BMW
| offset: 1 CreateTime: 1586329540135 keysize: 1 valuesize: 9 sequence: -1 headerKeys: [] key: 5 payload: Chevrolet
| offset: 2 CreateTime: 1586329540135 keysize: 1 valuesize: 7 sequence: -1 headerKeys: [] key: 6 payload: Porsche
| offset: 3 CreateTime: 1586329540136 keysize: 2 valuesize: 6 sequence: -1 headerKeys: [] key: 10 payload: Jaguar
| offset: 4 CreateTime: 1586329540136 keysize: 2 valuesize: 5 sequence: -1 headerKeys: [] key: 11 payload: Volvo
| offset: 5 CreateTime: 1586329540136 keysize: 2 valuesize: 10 sequence: -1 headerKeys: [] key: 12 payload: Land Rover
| offset: 6 CreateTime: 1586329540137 keysize: 2 valuesize: 12 sequence: -1 headerKeys: [] key: 15 payload: Aston Martin
baseOffset: 7 lastOffset: 13 count: 7 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false position: 173 CreateTime: 1586329548845 size: 173 magic: 2 compresscodec: NONE crc: 3669906974 isvalid: true
| offset: 7 CreateTime: 1586329548841 keysize: 1 valuesize: 3 sequence: -1 headerKeys: [] key: 2 payload: BMW
| offset: 8 CreateTime: 1586329548842 keysize: 1 valuesize: 9 sequence: -1 headerKeys: [] key: 5 payload: Chevrolet
| offset: 9 CreateTime: 1586329548842 keysize: 1 valuesize: 7 sequence: -1 headerKeys: [] key: 6 payload: Porsche
| offset: 10 CreateTime: 1586329548844 keysize: 2 valuesize: 6 sequence: -1 headerKeys: [] key: 10 payload: Jaguar
| offset: 11 CreateTime: 1586329548844 keysize: 2 valuesize: 5 sequence: -1 headerKeys: [] key: 11 payload: Volvo
| offset: 12 CreateTime: 1586329548844 keysize: 2 valuesize: 10 sequence: -1 headerKeys: [] key: 12 payload: Land Rover
| offset: 13 CreateTime: 1586329548845 keysize: 2 valuesize: 12 sequence: -1 headerKeys: [] key: 15 payload: Aston Martin
.
.
.
baseOffset: 28 lastOffset: 34 count: 7 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false position: 692 CreateTime: 1586329575827 size: 173 magic: 2 compresscodec: NONE crc: 3347769538 isvalid: true
| offset: 28 CreateTime: 1586329575821 keysize: 1 valuesize: 3 sequence: -1 headerKeys: [] key: 2 payload: BMW
| offset: 29 CreateTime: 1586329575823 keysize: 1 valuesize: 9 sequence: -1 headerKeys: [] key: 5 payload: Chevrolet
| offset: 30 CreateTime: 1586329575823 keysize: 1 valuesize: 7 sequence: -1 headerKeys: [] key: 6 payload: Porsche
| offset: 31 CreateTime: 1586329575824 keysize: 2 valuesize: 6 sequence: -1 headerKeys: [] key: 10 payload: Jaguar
| offset: 32 CreateTime: 1586329575825 keysize: 2 valuesize: 5 sequence: -1 headerKeys: [] key: 11 payload: Volvo
| offset: 33 CreateTime: 1586329575825 keysize: 2 valuesize: 10 sequence: -1 headerKeys: [] key: 12 payload: Land Rover
| offset: 34 CreateTime: 1586329575827 keysize: 2 valuesize: 12 sequence: -1 headerKeys: [] key: 15 payload: Aston Martin
```

Before we understand the result, I should mention that producer batches the messages before it writes to the partition to improve throughput. This is evident from the above result, where we can see messages grouped together. For the ease of understanding, the first batch of messages in the above `.log` file can be represented as follows


|   baseOffset  |  lastOffset  | count | position | CreatedTime | size | Messages |
| --------------|--------------|-------|----------|-------------|------|----------|
|   **0**       |     **6**        | 7 | 0 | **1586329540137** | 173  | {::nomarkdown}<table>  <thead>  <tr>  <th>offset</th>  <th>CreatedTime</th>  <th>payload</th>  </tr>  </thead>  <tbody>  <tr>  <td><b>0</b></td>  <td>1586234166066</td>  <td>BMW</td>  </tr> <tr>  <td>1</td>  <td>1586234166067</td>  <td>Chevrolet</td>  </tr> <tr>  <td>...</td>  <td>...</td>  <td>...</td>  </tr> <tr>  <td><b>6</b></td>  <td><b>1586329540137</b></td>  <td>Aston Martin</td>  </tr> </tbody>  </table> {:/}|


A quick summary of the above representation

- **baseOffset** : Offset of first message in the batch
- **lastOffset** : Offset of last message in the batch
- **count** : Number of messages in the batch
- **position** : Position of the batch in the file
- **CreatedTime** :Created time of last message in the batch
- **size** : Size of the batch(in bytes)
- **Messages** : List of messages(& its details) in the batch
<!--
|   baseOffset  |  lastOffset  | count | position | CreatedTime | size | Messages |
| --------------|--------------|-------|----------|-------------|------|----------|
| ... | ... | ... | ... | ... | ... | ... |
|   **14**       |     **20**        | 7 | **346** | **1586329557553** | 173  | {::nomarkdown}<table>  <thead>  <tr>  <th>offset</th>  <th>CreatedTime</th>  <th>payload</th>  </tr>  </thead>  <tbody>  <tr>  <td><b>14</b></td>  <td>1586329557545</td>  <td>BMW</td>  </tr>  <td>...</td>  <td>...</td>  <td>...</td>  </tr> <tr>  <td><b>20</b></td>  <td><b>1586329557553</b></td>  <td>Jaguar</td>  </tr> </tbody>  </table> {:/}|
| ... | ... | ... | ... | ... | ... | ... |	
|   **28**       |     **34**        | 7 | **692** | **1586329575827** | 173  | {::nomarkdown}<table>  <thead>  <tr>  <th>offset</th>  <th>CreatedTime</th>  <th>payload</th>  </tr>  </thead>  <tbody>  <tr>  <td><b>28</b></td>  <td>1586329575821</td>  <td>BMW</td>  </tr>  <tr>  <td>...</td>  <td>...</td>  <td>...</td>  </tr> <tr>  <td><b>34</b></td>  <td><b>1586329575827</b></td>  <td>Jaguar</td>  </tr> </tbody>  </table> {:/}|
| ... | ... | ... | ... | ... | ... | ... |	-->

### The '.index' file

Before we start discussing about the `.index` file, lets take a small detour to understand the need for `.index` file. As we know, every consumer has an associated numeric offset for its partition(s). This offset indicates the offset of the last processed message of a partition. Since the consumers process messages continuously, **extracting a message given an offset** should be a very frequent operation.

Lets say the consumer needs to find a message with offset **k** in a partiton. This involves two steps

`Step 1: Find the appropriate segment for offset k.`<br>
`Step 2: Extract the message with offset k from this segment.` 

From our previous discussion on segments, we know that the file name's suffix indicates the offset of the first message in that segment. Given this information, by doing a simple [binary search](https://www.geeksforgeeks.org/binary-search/) on the files names within the partition, we can quickly figure out the segment which contains the message for the given offset **k**. So, *Step 1* is sorted.

Now that we have the segment which contains the message, how do we extract that message? From the previous section, we know that every message along with its offset is being stored in the `.log` file. One possible(& naive) way to implement this would be to iterate over the contents of the `.log` file and extract out the message with the given offset. But this is not efficient as the size of the `.log` file may grow over time, and processing the entire file would be difficult. So how do you think Kafka handles this? 

This is where the index file (*.index*) comes into picture. The index file stores a mapping of *relative offset*(4 bytes) to a *numeric value*(4 bytes). This numeric value indicates the position in the `.log` file where the message with *offset* = (*base offset* + *relative offset*) is located. Let us check out the contents of the index file by using the command below.

```bash
docker exec -it broker-0 kafka-run-class.sh kafka.tools.DumpLogSegments --deep-iteration --print-data-log --files /tmp/logs/kafka-logs/broker-0/cars-0/00000000000000000000.index
```

```
Dumping /tmp/logs/kafka-logs/broker-0/cars-0/00000000000000000000.index
offset: 20 position: 346
offset: 34 position: 692
```

The result indicates that the message with offset 20 ( = 0 + 20) is located at position 346 in `00000000000000000000.log` file. 


<p align="center">
<img src="{{ site.baseurl }}/images/log-index-mapping.png">
</p>

Given such a mapping, we can easily extract a message in the segment given an offset **k**. Therefore, *Step 2* is also sorted.

But wait, does kafka store this mapping for every single offset? No. From the above result we can see the mapping only for offsets 20 and 34.

Having that said, how does Kafka know when to add a entry in the index file? Kafka uses a parameter named `log.index.interval.bytes`. This parameter indicates how frequently (after how many bytes) an index entry would be added. By tuning this parameter, we can control the number of entries in the `.index` file and thereby the size of the `.index` file. You can play around with this parameter and see how the mapping in `.index` file changes.

### The '.timeindex' file

The `.timeindex` file stores the timestamp to offset mapping (**T**, **offset**). A time index entry (**T**, **offset**) indicates that in a given segment, all those messages whose timestamp > **T** have their offset > **offset**. This is primarily used to search offsets by timestamp. You can find more details about `.timeindex` [here](https://cwiki.apache.org/confluence/display/KAFKA/KIP-33+-+Add+a+time+based+log+index).
Let us try to checkout the contents of the `.timeindex` file in our working example by using the following commad. 

```bash
docker exec -it broker-0 kafka-run-class.sh kafka.tools.DumpLogSegments --deep-iteration --print-data-log --files /tmp/logs/kafka-logs/broker-0/cars-0/00000000000000000000.timeindex
```
```
Dumping /tmp/logs/kafka-logs/broker-0/cars-0/00000000000000000000.timeindex
timestamp: 1586329557553 offset: 20
timestamp: 1586329575827 offset: 34
```

We can clearly see the mapping between timestamp([Unix time](https://en.wikipedia.org/wiki/Unix_time)) and the offsets. You will notice that these offsets are exactly the same offsets seen in the `.index` file(discussed in the previous section). This is because, Kafka uses the same `log.index.interval.bytes` parameter to decide when to add a time index entry.

## Consumer Offset Storage

As discussed before, every consumer needs to store a numeric offset for partition(s) it is consuming. But, where does kafka store this numeric offset?

Before Kafka 0.9, Zookeeper supported the storage & retrieval of consumer offsets. Due to scalability issues with zookeeper, Kafka now started storing this information in a topic named `__consumer_offsets`. The number of paritions and replication factor of this topic can be tuned by using `offsets.topic.num.partitions` & `offsets.topic.replication.factor` respectively.

Let us now explore the content of messages in this topic. To do so, we start a consumer which consumes messages from this topic using a formatter. 

```bash
docker run --network=hands-on_kafka-tier -ti bitnami/kafka:latest kafka-console-consumer.sh --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" --bootstrap-server broker-0:9092,broker-1:9093 --topic __consumer_offsets
```
Upon execution, you will see a output as shown below.

```
[car-group,cars,1]::OffsetAndMetadata(offset=255, leaderEpoch=Optional[0], metadata=, commitTimestamp=1586224403170, expireTimestamp=None)
[car-group,cars,0]::OffsetAndMetadata(offset=210, leaderEpoch=Optional[0], metadata=, commitTimestamp=1586224403182, expireTimestamp=None)
```

Each message in this topic can be represented in the following format

```
[consumer-group, topic-name, partition-number]::OffsetAndMetadata(offset=consumer-offset, leaderEpoch=Optional[0], metadata=, commitTimestamp=1585019671259, expireTimestamp=None)
```

By looking at the first entry, it is clear that Kafka is storing the numeric offset(`=210`) for consumer that belongs to a consumer group(=`car-group`), consuming data from `0th` partition(`cars-0`) of topic `cars`.


## Conclusion

Let's quickly summarize our learnings from this post

* `log.dirs` indicates the directory where Kafka stores topic data.
* Partitions are divided into segments.
* `log.segment.bytes` indicates the maximum size(in bytes) of a segment.
* `.log` file contains all the messages.
* `.index` file contains the mapping of message offset to its position in `.log` file.
* `log.index.interval.bytes` indicates the memory size(in bytes) after which an entry in index/timeindex file is added.
* Kafka uses `.index` & `.log` file to quickly extract the message for a given offset.
* `.timeindex` file contains the mapping of timestamp to message offset.
* `.timeindex` is used to search messages by timestamp.
* Consumer offsets are stored in a internal topic(`__consumer_offsets`) managed by Kafka.
* `offsets.topic.num.partitions` indicates the number of partitions in `__consumer_offsets`.
* `offsets.topic.replication.factor` indicates the replication factor of `__consumer_offsets`.

Hope this post gave you a practical understanding of how Kafka stores and manages internally. Additionally, you should now have an idea about how to tune your storage based on the requirements. Please do share your feedback in the comments section below. In the upcoming blog post we will explore Kafka API(s) and learn how to use them. Stay Tuned & Happy coding !!



