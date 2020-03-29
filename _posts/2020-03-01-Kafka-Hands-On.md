---
layout: post
title: Getting Started With Apache Kafka - Quick Hands On
blog : true
published: true
date: March 01, 2020
comments: true
---

If you had a chance to go through my previous [post]({% post_url 2020-02-29-Kafka-Architecture %}), you should have developed a good understanding of Kafka Architecture. But, how good is understanding a technology without a hands-on? So, we take a step forward and practically see how these architectural components interact & work under the hood. The hands on will cover the following points 

1. *Cluster Setup*
2. *Topic Creation*
3. *Consumer*
4. *Producer*

At the end of this post, you will also be left with enough knowledge to setup your own working kafka cluster. So lets get started !!

> Note : Throughout this post we will be using *docker* to demonstrate the hands-on. Make sure that [docker](https://www.docker.com/get-started) is setup & running in your working environment. The code used for the hands on is available in [this](https://github.com/rohithsankepally/apache-kafka-blog) repository. Please make sure that you clone this repository in your working environment before you proceed further.

## Cluster Setup

<p align="center">
<img src="{{ site.baseurl }}/images/zookeeper-kafka.png" height="300" width="450">
</p>

To make the cluster setup easy, we use [`docker-compose`](https://docs.docker.com/compose/). This way, the cluster can be launched using a single command, provided we specify all the dependencies/requirements in a `docker-compose.yml` file. For this purpose, I created a [`docker-compose.yml`](https://github.com/rohithsankepally/apache-kafka-blog/blob/master/hands-on/docker-compose.yml) file with all the necessary configuration. Lets now go through the components used in the cluster setup.

### Kafka Brokers

We use the [bitnami kafka](https://hub.docker.com/r/bitnami/kafka) docker image for the setup. A kafka cluster is a collection of kafka brokers. In this working example, our kafka cluster contains two brokers `broker-0` & `broker-1` which can be seen in the [`docker-compose.yml`](https://github.com/rohithsankepally/apache-kafka-blog/blob/master/hands-on/docker-compose.yml) file.

While launching a broker, we need to provide a configuration file which determines the properties of the broker. The [bitnami kafka](https://hub.docker.com/r/bitnami/kafka) docker image uses `/opt/bitnami/kafka/conf/server.properties` file for setting up the broker. For the purpose of this hands on, we use custom configuration file for each broker. Here, we create two files [`broker-0.properties`](https://github.com/rohithsankepally/apache-kafka-blog/blob/master/hands-on/broker-0.properties) & [`broker-1.properties`](https://github.com/rohithsankepally/apache-kafka-blog/blob/master/hands-on/broker-1.properties) which serve as configuration for the brokers. Thanks to [docker volumes](https://docs.docker.com/storage/volumes/), which helps to map these files in such a way that the bitnami kafka image uses these files for setting up the brokers.

### Zookeeper

Kafka is built using zookeeper. [Zookeeper](https://zookeeper.apache.org/) is a distributed synchronization service used to manage a set of interconnected systems. Kafka uses zookeeper for coordination of brokers and also stores metadata of topics, partitions etc. The internal working of zookeeper is out of this blog's scope. Since kafka is dependent on zookeeper, before starting the kafka cluster, we ensure that zookeeper is up and running. We use the [bitnami zookeeper](https://hub.docker.com/r/bitnami/zookeeper) docker image to setup zookeeper. We can see from the [`docker-compose.yml`](https://github.com/rohithsankepally/apache-kafka-blog/blob/master/hands-on/docker-compose.yml) file that the kafka brokers `broker-0` & `broker-1` depend on `zookeeper`.

Now that we have defined all the dependencies, its time to launch our kafka cluster. All that you have to do is, navigate to the cloned repository and execute the commands below

```bash
$ cd hands-on
$ sh cluster-start.sh
```
This should start the kafka cluster which comprises of the zookeeper and two kafka brokers. This can be verified by checking the list of running docker containers running the command below in a new terminal window.

```bash
$ docker container ls
```

| CONTAINER ID|  IMAGE  |   COMMAND | CREATED |  STATUS | PORTS  | NAMES 
| ------------|---------|-----------|---------|---------|--------|-------
|1c26dfb264db | bitnami/kafka:latest| "/entrypoint.sh /run…" | 7 seconds ago |  Up 6 seconds | 9092/tcp, 0.0.0.0:9093->9093/tcp | broker-1 |
|aa0f4f7073b1  |    bitnami/kafka:latest   |    "/entrypoint.sh /run…" |  8 seconds ago    |   Up 7 seconds   | 0.0.0.0:9092->9092/tcp | broker-0 |
|96f169a92a19  |  bitnami/zookeeper:latest | "/entrypoint.sh /run…" |  9 seconds ago   |    Up 8 seconds    |    2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, 8080/tcp   |zookeeper

You should see three containers(one zookeeper & two kafka) up and running as shown above. With this we have finished setting up the kafka cluster.

## Topic Creation

Now that we have the cluster up and running we would like to see how it works. Before we start producing/consuming messages, we need a *topic*. A topic can be created in kafka by providing the topic name and some configuration details. You can find the list of topic configurations in apache kafka [documentation](https://kafka.apache.org/documentation/#topicconfigs). We create a topic by using only two configuration parameters namely ```--partitions``` & ```--replication-factor``` which indicate the number of partitions & number of replicas per partition respectively. The topic creation is carried out by execution of below command

```bash
$ docker run --network=hands-on_kafka-tier -ti bitnami/kafka:latest kafka-topics.sh --create --zookeeper zookeeper:2181 --replication-factor 2 --partitions 2 --topic cars

```	

This command results in the creation of a topic named *cars* with *two* partitions having *two* replicas each. The details of the above created topic can be seen by the using command below

```bash
$ docker run --network=hands-on_kafka-tier -ti bitnami/kafka:latest kafka-topics.sh  --zookeeper zookeeper:2181 --describe cars
```

`Topic: cars	PartitionCount: 2	ReplicationFactor: 2	Configs:`<br>
`Topic: cars	Partition: 0	Leader: 0	Replicas: 0,1	Isr: 0,1`<br>
`Topic: cars	Partition: 1	Leader: 1	Replicas: 1,0	Isr: 1,0`

This output can be mapped as follows

| Component   |Description                 |
| ------------|----------------------------|
| Partition: 0| 0th partition of topic |
| Partition: 1| 1st partition of topic |
| Leader: 0   | *broker-0* |
| Leader: 1   | *broker-1* |
| Replicas    | List of brokers(*broker-0*,*broker-1*) replicating this topic data |
| In Sync Replicas(Isr) | List of brokers(*broker-0*,*broker-1*) that in sync with the leader |

As mentioned in the previous [post]({% post_url 2020-02-29-Kafka-Architecture %}), each partition is assigned a leader. From the above result we can see that for `Partition: 0` the leader is `broker-0`. Similarily for `Partition: 1` the leader is `broker-1`. For the sake of simplicity let us call `Partition: 0` & `Partition: 1` of `cars` topic as `cars-0` & `cars-1` respectively.

> Note : Due to the randomness in leader election, it might occur that the leader for `Partition: 0` is *broker-1* and that of `Partition: 1` is *broker-0*. In that case the output & mapping will slightly differ.

## Consumer

We first setup consumers subscribed to the above generated topic followed by writing messages to the topic. This way we get to see how each message is being processed by the consumers. We will be launching two consumer instances belonging to a consumer group `car-group` & subscribe to the topic `cars`. In this working example, we set up each consumer as a separate docker container. Now, open a new terminal window and enter the command below which should launch the first consumer instance with name `consumer-0`.

```bash
$ docker run --network=hands-on_kafka-tier --name=consumer-0 -ti bitnami/kafka:latest kafka-console-consumer.sh --bootstrap-server broker-0:9092,broker-1:9093 --topic cars --consumer-property group.id=car-group
```

Inorder to launch the second consumer instance, open another terminal window and execute the same command with a different name say `consumer-1` as shown below.

```bash
$ docker run --network=hands-on_kafka-tier --name=consumer-1 -ti bitnami/kafka:latest kafka-console-consumer.sh --bootstrap-server broker-0:9092,broker-1:9093 --topic cars --consumer-property group.id=car-group
```

Now that we have launched two consumers ready to process messages from `cars`, lets check the status of these consumers through their consumer group.

```bash
$ docker run --network=hands-on_kafka-tier -ti bitnami/kafka:latest kafka-consumer-groups.sh  --bootstrap-server broker-0:9092,broker-1:9093 --describe --group car-group
```

| TOPIC      |PARTITION|CURRENT-OFFSET| LOG-END-OFFSET | LAG | CONSUMER-ID | HOST  | CLIENT-ID |
| -----------|---------|--------------| ---------------|-----|-------------|----------|
| cars| 1 | 0 | 0 | 0 | consumer-1-306edd7e-18b3-4502-9783-093b9b3597c9 | /172.27.0.5 | consumer-car-group-1
|cars|  0 | 0 | 0 | 0 |                consumer-1-0a67dcb0-8687-47af-9f3b-a6eb82a7bb4b | /172.27.0.6   |   consumer-car-group-1

As discussed in the previous [post]({% post_url 2020-02-29-Kafka-Architecture %}), the partitions of a topic are equally distributed across consumers of a consumer group to allow parallel processing of messages in a topic. From the above result we can clearly see that one consumer is assigned to *cars-1* and the other one is assigned to *cars-0*. We can also notice that both the consumers have their *CURRENT-OFFSET* as *0*, which indicates that they have not yet processed any of the messages in their respective partitions.

## Producer

Now that we have the topic setup and consumers subscribed to our topic, its time for some action. Let's produce !! Open a new terminal window and enter the command below.

```bash
$ docker run --network=hands-on_kafka-tier --name=producer -ti bitnami/kafka:latest kafka-console-producer.sh --broker-list broker-0:9092,broker-1:9093 --topic cars
```
This should result in a command prompt where we can write messages to our topic. Let's write our first message.

```bash
> Mercedes
```

Open the terminal windows where we launched the consumer instances. You can see this message is consumed by one of those consumers. Let us also verify this by checking the status of the consumers

```bash
$ docker run --network=hands-on_kafka-tier -ti bitnami/kafka:latest kafka-consumer-groups.sh  --bootstrap-server broker-0:9092,broker-1:9093 --describe --group car-group
```

GROUP  | TOPIC | PARTITION | CURRENT-OFFSET | LOG-END-OFFSET | LAG | CONSUMER-ID |HOST  | CLIENT-ID |
| ---------|---------|-----------|---------|--------|--------|--------|-------- | -------|
car-group  |   cars | 1  | 0   |0    |0  |         consumer-car-group-1-3cf14661-c5cc-43f6-8320-88ee3381e658 | /172.27.0.5 |     consumer-car-group-1
car-group  |     cars  |  0  |1   | 1  | 0 |consumer-car-group-1-32e408ce-6ba3-4506-92e9-5f4827d083fe | /172.27.0.6 |   consumer-car-group-1

We can see that the `CURRENT-OFFSET` of consumer reading from *cars-0* is 1. This indicates that it has processed the above message. Let's write another message by going to the terminal window which is running the producer instance.

```bash
> BMW
```
From the terminal windows running the consumer instances, we can see that this message is processed by the other consumer. Let us verify this by checking the status of the consumers again.

```bash
$ docker run --network=hands-on_kafka-tier -ti bitnami/kafka:latest kafka-consumer-groups.sh  --bootstrap-server broker-0:9092,broker-1:9093 --describe --group car-group
```

GROUP  | TOPIC | PARTITION | CURRENT-OFFSET | LOG-END-OFFSET | LAG | CONSUMER-ID |HOST  | CLIENT-ID |
| ---------|---------|-----------|---------|--------|--------|--------|-------- | -------|
car-group  |   cars | 1  |  1   |   1    |  0  |         consumer-car-group-1-3cf14661-c5cc-43f6-8320-88ee3381e658 | /172.22.0.5 |     consumer-car-group-1
car-group  |     cars  |  0  |  1   |  1  | 0 |consumer-car-group-1-32e408ce-6ba3-4506-92e9-5f4827d083fe | /172.22.0.5 |   consumer-car-group-1

The `CURRENT-OFFSET` of consumer reading from *cars-1* is also 1 which indicates that it has processed the message.

From the above results, we can clearly see that the messages are equally shared among the consumers each reading from their respective partitions. This also indicates that the messages of the topic are equally distributed across partitions.

## Chaos Engineering

### What if the broker goes down?
So far so good. But, what if one of our brokers goes down? Such situations are very common in production environments. Since kafka is known for its fault tolerant behaviour, let's take one of our brokers down & see how kafka handles the situation.

```bash
$ docker stop broker-0
```

This command stops `broker-0` which can be verified by `docker container ls` command. Now lets check the status of our `cars` topic.

```bash
$ docker run --network=hands-on_kafka-tier -ti bitnami/kafka:latest kafka-topics.sh  --zookeeper zookeeper:2181 --describe cars
```

`Topic: cars	PartitionCount: 2	ReplicationFactor: 2	Configs:` <br>
`Topic: cars	Partition: 0	Leader: 1	Replicas: 0,1	Isr: 1`<br>
`Topic: cars	Partition: 1	Leader: 1	Replicas: 1,0	Isr: 1`

By comparing this with the previous status of topic *cars*, we can clearly see that for *Partition: 0*, the leader has changed from `Leader: 0` to `Leader: 1` i.e *broker-0* to *broker-1*. Since, *broker-0* is down, it is not in sync with the new leader(*broker-1*) and therefore the `Isr` is changed from `0,1` to `1`. Thus the cluster stays unaffected even after one of the brokers is down, with the other broker acting as the sole leader for both the partitions. You can further check this by writing more messages and verify that they are getting processed by the consumers. 

### What if one of the consumers go down?

Well, we saw kafka cluster being unaffected after one of the brokers went down. Now, lets add more chaos to the system. We now take down one of the consumers and see whether kafka can handle this failure.

```bash
$ docker stop consumer-0
```
This command stops `consumer-0` which can be verified by `docker container ls` command. Now lets check the status of the consumers.

```bash
$ docker run --network=hands-on_kafka-tier -ti bitnami/kafka:latest kafka-consumer-groups.sh  --bootstrap-server broker-1:9093 --describe --group car-group
```

GROUP  | TOPIC | PARTITION | CURRENT-OFFSET | LOG-END-OFFSET | LAG | CONSUMER-ID |HOST  | CLIENT-ID |
| ---------|---------|-----------|---------|--------|--------|--------|-------- | -------|
car-group  |   cars |1  | 1   |1    |0  |         consumer-car-group-1-32e408ce-6ba3-4506-92e9-5f4827d083fe | /172.22.0.5 |     consumer-car-group-1
car-group  |     cars  |  0  |1   | 1  | 0 |consumer-car-group-1-32e408ce-6ba3-4506-92e9-5f4827d083fe | /172.22.0.5 |   consumer-car-group-1

From the above result we can see that both the partitons are assigned to only one consumer. This clearly shows that kafka rebalances the consumers across the partitions to ensure that the consumer group reads all the messages of the partition at any point of time. You can try and verify the working of the cluster by writing some more messages and see them getting processed by the working consumer.

Both the above demonstrations indicate the fault tolerant behaviour of kafka.


## Conclusion

This blog tried to give a practical explanation about interactions between architectural components within a Kafka cluster. In addition to this the hands-on also demonstrated the fault tolerant behaviour of kafka. Hopefully this gave you a good understanding on how things work under the hood. Also, you now have enough knowledge to setup your own kafka cluster. Please do share your feedback in the comments section below. In the upcoming blog post we’ll be going over the internals of Apache Kafka. Stay tuned & Happy Coding !!