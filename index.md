# Kafka Introduction

## What is Kafka?

Many developers look at kafka and want to fit it into the Message Queue category but there are many key differences.

Firstly data fanout in Message Queues is awkward. Each consumer maintains a cache inside the Message queue. This cache is usually in memory and stopped consumers can cause major headaches for Message Queue administrators. This cache is often limited in size frequently to sizes like 1/2 Megabytes and if you go past that threshold your consumer can never catch up! This frequently results in crazy full reloads where all data is passed through the MQ pointlessly to satisfy the needs of a single consumer.

More importantly kafka was not built originally as a just a message passing mechanism. It was noted by the architects of Kafka that all conventional databases are actually split into 2 sections.

   * Redo logs
   * Materialised Views

The redo logs contain an immutable log of everything which happens to a database (sometimes also called a journal). The materialised view contains a checkpointed and thus hopefully reliable view of this journal deduplicated and queryable using SQL.

The key realisations in Kafka are that

   * Deduplicating messages can be important to prevent unbounded data growth (sprawl). This is the function that compacted topics forfills.
   * materialised views dont have to be in tables. They can be in any output such as elastic search, redis, local key/value stores, graph databases, etc, etc

Of all the output formats that Kafka can output to SQL is one of the formats which is the least friendly for horizontal scaling. It is thus rarely used for microservice architectures espectially when the incoming data is unbounded in nature.

Kafka is not just a mechanism for queueing data. It is also a mechanism for storing data without presumption about how it will be consumed.

## Kafka Producers/Consumers

You can read and write to Kafka using a number of mechanisms

   * Command line tools like kafka-console-consumer, kafkacat, kt, etc
   * Codified use using various kafka libraries such as the java API, Kafka Streams, librdkafka or samara
   * Tools designed for streaming such as kafka-connect, mirrormaker or Apache nifi

## What do we mean by api?

In many cases when we are using Kafka for codified use it is important to realise that changing the data that is being produced has the potential to break upstream consumers. This is an overhead which is probably managable when you have a small number of microservices but as the numbers grow (Netflix have over 1000) the chance of breaking your consumers becomes very high and this is why schemas become very important.

There are a number of prominent schema mechanisms with protobufs probably being the quickest and most efficient. Confluent however, the authors of kafka have chosen Avro as their schema of choice and the whole Confluent architecture along with Kafka Streams revolves around this.

The easiest way to think of Kafka is as a streaming API where the Schema is your API documentation. Avrodoc can even generate API like documentation http://avrodoc.herokuapp.com/#/

## Starting Kafka
Start up a Kafka/Zookeeper cluster using

```
docker-compose up
```

Download the confluent kafka distribution from https://www.confluent.io. In these examples it has been installed into /app/confluent

## Consumer group check

Check the consumer groups for the new kafka instance

```
/app/confluent/bin/kafka-consumer-groups --bootstrap-server localhost:9092 --list
```

## Simple producing and consuming

Create a simple topic with a single replica and a single partition

```
/app/confluent/bin/kafka-topics --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
```

Produce and consume data using single producer and single consumer

```
/app/confluent/bin/kafka-console-producer --broker-list localhost:9092 --topic test
/app/confluent/bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
```

Check the consumer groups again. We can see a consumer group has been automatically allocated.

## What happens if we set our own consumer group?
Lets try setting the consumer group

```
/app/confluent/bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning --group consumer-1
```
What happens if we add another consumer group using the same id

```
/app/confluent/bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning --group consumer-1
```

## Lets try again with a few more partitions
 
and create a new test topic

```
/app/confluent/bin/kafka-topics --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 100 --topic test
```

And try again

```
/app/confluent/bin/kafka-console-producer --broker-list localhost:9092 --topic test
/app/confluent/bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning --group consumer-1
/app/confluent/bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning --group consumer-1
```

We can see that the data is split between the consumers. Partitions are split across a consumer groups id but each partition can only be consumed by a single consumer in the same group. If you only have a single partition the data can only be consumed by a single consumer within that group. For this reason choosing the right number of partitions for a topic is fairly important as affects the scalability of the cluster.

## Allocation within topics

When we consume from Kafka we usually look at the values but Kafka is a key value store and the key can be used for a variety of things like filtering. By default the key is used as a deterministic seed for the partition allocation. When no key is specified kafka automatically allocates information used for sharding and should be close to an equal split over a large number of events.

If we manually set the key when we produce messages

```
/app/confluent/bin/kafka-console-producer --broker-list localhost:9092 --topic test2 --property "parse.key=true" --property "key.separator=:"
```

We can see that data is distributed predictably across partitions based on the key. The distribution allocation mechanism in kafka is extensible but this behaviour is the default. Custom allocation approaches are commonly used for advanced sharding techniques.

## Compacted Topics

Lets create a compacted topic

```
/app/confluent/bin/kafka-topics --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic compacted --config min.cleanable.dirty.ratio=0.01 --config cleanup.policy=compact --config segment.ms=100 --config delete.retention.ms=100
```

And try consuming using another tool based on librdkafka

```
kafkacat -b localhost:9092 -t compacted -e
```

And produce some more messages
```
/app/confluent/bin/kafka-console-producer --broker-list localhost:9092 --topic test2 --property "parse.key=true" --property "key.separator=:"
```

We can see that duplicate messages with the same key get cleaned up. This is the basis for data storage within Kafka. If we look at the command used for topic creation we can see that we set a very frequent segment creation and very low retention period. We also tell Kafka to compact frequently by changing the cleanable.dirty.ratio compaction usually runs far less frequently and we expect the target materialised view to be based on the same key as Kafka to ensure messages are overwritten.

## Listing Topics

We can list topics with

```
/app/confluent/bin/kafka-topics --bootstrap-server localhost:9092 --list 
```

## Replication in Kafka

When we create a topic we choose the replication factor.

```
/app/confluent/bin/kafka-topics --replication-factor x
```

Each replica works in the same way. However one replica will always be the leader. This is the one which is written to first.

For a simple partition we can see the partition allocation
```
/app/confluent/bin/kafka-topics --bootstrap-server localhost:9092 --describe test
```

A more complicated partition will look like this

```
Topic:account-event	PartitionCount:120	ReplicationFactor:3	Configs:
	Topic: account-event	Partition: 0	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: account-event	Partition: 1	Leader: 3	Replicas: 3,1,2	Isr: 2,3,1
	Topic: account-event	Partition: 2	Leader: 1	Replicas: 1,2,3	Isr: 2,3,1
	Topic: account-event	Partition: 3	Leader: 2	Replicas: 2,1,3	Isr: 2,3,1
~~~~~~~~~~~~~~~~ 120 of these.
```

When we configure our producer we can choose to 

   * Fire and forget
   * Wait for an ack from the leader
   * Wait for the producer to satisfy min.ISR

Depending on the option we choose will affect how quickly the producer returns to the client.


Min.isr can be set on a topic using an option on topic creation

```
/app/confluent/bin/kafka-topics --config min.insync.replicas=x
```

Generally you want enough replicas to ensure your data is safe but we can run into problems. If you have 3 Kafka Servers and you have 3 replicas and min.isr also set to 3 as soon as you lose a single Kafka server your topic will still be available for read but you will be unable to write to it. For this reason when you have a 3 node Kafka cluster you need min.isr set to 2. This is too little for a reliable cluster. My recommended minimum cluster size is 5 Kafka servers with a min.isr set to 3

# Offsets - where to consume to

Kafka is immutable. This means that when you produce to Kafka is will always produce to the end of the topic.

When consuming from Kafka it is important to understand the importance of offsets for reliable Kafka use.

The pattern is usually

  * Go to last successfully commited offset
  * Read the next message from Kafka
  * Ensure the action is performed based on that message 
  * Only submit the updated offset back to Kafka once the message has been passed on succesfully
  * Restart the pattern

This ensures that on failure we re-read the failed message and process again. This can make failure scenarios complicated. What if the action taken persistently fails. Usually consumption is permenantly stuck. The code needs to understand when it is and when it is not acceptible to acceptible to submit the updated offset back anyway. Most of the time its the action that needs to be fixed to ensure data consistency.

## So i have an offset per Topic?

Unfortunately its a lot more complicated than that. Every partition has an offset for each consumer group id so a topic with 600 partitions will have 600 different offsets which all need managing seperately (fortunately kafka does most of this for us). But it causes an interesting interesting situation when you want to monitor consumption because you have to monitor all 600 partitions. This is what we use burrow for and will be covered in another session. 

## Simple offset management

When we consumer from kafka by default we consume from the end. If we want to consume from the beginning we can add the flag

```
  --from-beginning
```

## Deleting topics

```
/app/confluent/bin/kafka-topics --bootstrap-server localhost:9092  --delete --topic test
```

## Other talking points

   * idempotency in Kafka - need special consumer and producer. Guarantees exactly once. Great for Kafka messages. Not quite so good for restoring potentially overlapping data.
   * Golden source of truth
   * Offset Checker/Updater
   * Topic operator


