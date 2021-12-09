## Table of Contents
- [Kafka Consumers](#Kafka-Consumers)
  - [Concepts](#Concepts)
  - [Creating a Consumer](#creating-a-consumer)
  - [Subscribing to Topics](#subscribing-to-topics)
  - [The Poll Loop](#the-poll-loop)
  - [Commits and Offsets](#commits-and-offsets)
  - [Standalone Consumer](#standalone-consumer)

## Concepts

### Consumers and Consumer Groups

Consumer groups are multiple instances of a consumer application. Each consumer in a consumer group will receive messages from a different subset of the partitions in a given topic

If you have more consumers than partitions, some consumers will sit idle

### Consumer Groups and Partition Rebalance

Moving partition ownership from one consumer to another is called rebalance. It casues a short window of unavailability for the entire consumer group

Consumers maintain membership in a group and ownership of a partition by sending `heartbeats` to a Kafka broker designated `group coordinator`. heartbeats are sent when the consumer polls and when it commits records it has consumed

In new versions of Kafka, you can set `max.poll.interval.ms` to handle longer delays for polling records

### Assigning Partitions

When a consumer joins a group, it sends a `JoinGroup` request. First consumer is the `leader`. The leader gets a list of all the consumers in the group and assigns a subset of partitions to each consumer. Implements `PartitionAssignor` to decide which partitions should be handled by which consumer. Leader sends assignments to `GroupCoordinator` which sends this info to all the consumers.

## Creating a Consumer

Create an instance of `KafkaConsumer` and pass `Properties`. Required properties are:
- `bootstrap.servers` - connection string for cluster
- `key.deserializer` - class that takes byte array and turns it into java object
- `value.derializer` - class that takes byte array and turns it into java object
- `group.id` - OPTIONAL, specifies the consumer group

## Subscribing to Topics

use subscribe method
```
consumer.subscribe(Collections.singletonList("customerCountries"));
```

You can also pass a regular expression to match any new topics and begin consuming those
```
consumer.subscribe("test.*");
```

## The Poll Loop

Consumer uses an infinite loop to handle all the details of coordination, partition rebalances, heartbeats, and data fetching

poll() method is a timeout interval determing how long consumer will wait for data, it returns a list of records containing the topic, partition, offset, key, and value

always close() the consumer before exiting, triggering a rebalance immediately rather than waiting for the group coordinator to discover the missing heartbeats

be sure that any processing that happens within the poll loop is fast and effecient

threading: must have a single thread per consumer

## Configuring Consumers

`fetch.min.bytes` - specify the min amount of data to receive from broker when fetching records

`fetch.max.wait.ms` - wait until there's enough data before responding to the consumer. default is 500ms.example: if you set `fetch.max.wait.ms` to 100 ms and `fetch.min.bytes` to 1MB, you will receive response when either data has hit 1MB or after 100 ms, whichever is first

`max.partition.fetch.bytes` - max number of bytes that will be returned per partition. must be larger than largest message size (`max.message.size` in broker config)

`session.timeout.ms` - amount of time consumer can be out of contact with broker while still considered alive. default is 3 seconds. must be higher than `heartbeat.interval.ms`

`auto.offset.reset` - controls behavior of reading from offset if committed offset for consumer is invalid. default is 'latest', alternative is 'earliest' (read from beginning)

`enable.auto.commit` - default is true, set to false if you prefer to control offset commits. this can help minimize diplicates and avoid missing data. can also be used with `auto.commit.interval.ms`

`partition.assignment.strategy` - 2 provided strategies
    1. Range - DEFAULT. assigns each consumer a consecutive subset of partitions from each topic individually. Can create an unbalanced distribution
    2. Round Robin - takes all partitions from all topics and subscribes sequentially, one by one. Creates a more equal distribution of partitions
    3. Implement your own assignment class

`max.poll.records` - controls max number of records a single poll() call will return

`receive.buffer.bytes` and `sennd.buffer.bytes` - size of TCP send and receive buffers used by sockets when reading/writing data. If set to -1, OS defaults are used. increase if consumers and brokers are in a different datacenter due to higher latency and lower bandwidth

## Commits and Offsets

When a consumer is ready to commit, it sends a message to a special `_consumer_offsets` topic

If a consumer cashes or new consumer joins before a commit, some messages may be processed twice

### Automatic Commit

Easiest way to commit. Every 5 seconds consumer will commit the largest offset your client received from poll(). can be adjusted with `auto.commit.interval.ms`. A call to poll() or close() will also commit offset automatically. NOT good option for avoiding duplicate messages

### Commit Current Offset

control offset committing by setting `auto.commit.offset` to false. `commitSync()` allows you to commit with blocking code. Only call commitSync() AFTER you have processed records

### Async Commit

`consumer.commitAsync()` will allow application to continue running while committing. The drawback is that it will not retry. Takes a callback. If you choose to retry, must be aware that commits could happen in the wrong order and cause duplicate processing

Safely retrying async commits: create an increasing number every time you commit. If sequence number is higher than the retry, a newer commit has been made

### Combining Sync and Async Commits

Use commitAsync and use commitSync in finally block just before shutdown

### Commit Specified Offset

call commitSync() or commitAsync() with a map of partitions and offsets to commit in order to have more granular control

### Rebalance Listeners

run code when partitions are added or removed from consumer by passing `ConsumerRebalanceListener` when calling `subscribe()` method. There are two methods to implement:
1. `onPartitionsRevoked` - called before rebalancing starts and after consumer stopped consuming messages. You can commit offsets so next consumer will know exactly where to start
2. `onPartitionsAssigned` - callsed after partitions have been assigned, before consuming messages

### Consuming Records with Specific Offsets

use `seek()` method to find a specific offset. can be used when storing offset in external db or data source. When adding a new consumer, fetch current offset from DB and use seek() to find the exact place to begin processing

### Safely Exiting

Call `consumer.wakeup()` method from another thread to force a `WakeupException` in the polling thread. Use a finally block to call `consumer.close()`

## Deserializers

Using AvroSerializer and Schema repository is best option to make sure serializers/deserializers match

## Standalone Consumer

Instead of subscribing to a topic, assign all partitions to consumer using `consumer.partitionsFor("topic")` method to get all partitions and `consumer.assign(partitions)` method to assign
