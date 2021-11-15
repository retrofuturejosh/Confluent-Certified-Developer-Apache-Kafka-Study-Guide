## Table of Contents
- [Kafka Internals](#Kafka-Internals)
  - [Cluster Membership](#Cluster-Membership)
  - [Request Processing](#Request-processing)
  - [Physical Storage](#Physical-storage)
  - [Compaction](#Compaction)

## Cluster Membership
Zookeeper maintains a list of brokers. When a broker joins, it registers with unique ID by creating an ephemeral node.

Ephemeral node is automatically removed if broker loses connectivity, but the broker ID persists. A new broker with the same ID can take its place and be assigned the same partitions and topics

## The Controller
Controller is a Kafka broker that is responsible for electing partition leaders

First broker to join creates an ephemeral node called `/controller`. Subsequent brokers try to create this node, but get an error. Then they create a `Zookeeper watch` on the controller node to get notified of changes.

Cluster only has ONE controller

If controller broker goes down, the other nodes will attempt to create the `/controller` ephemeral node. First broker wins. Each time a new controller is elected the controller epoch number is incremented to keep track of the current controller

Controller watches for brokers that leave cluster and reassigns their leader role to another replica

Controller watches for brokers joining the cluser. If the ID has existing replicas, controller notifies brokers of the change

## Replication

Two types of replicas
- Leader replica: single replica. all produce and consume requests go through leader
- Follower replica: All non-leader replicas. Job is to stay up-to-date with leader

Followers stay in sync by sending the leader `Fetch` requests (same as consumers). Response includes messages and offset, always in order

If follower hasn't requested message in more than 10 seconds, it is considered out of sync. An out of sync replica can not become the new leader

Amount of time a follower can be inactive is controlled by `replica.lag.time.max.ms`

Each partition has a preferred leader: the replica that was leader when topic was created

`auto.leader.rebalance.enable` set to true will check in preferred leader is not current leader. If it's in-sync, leader election will make it the current leader


## Request Processing

Most of a brokers job is to process requests sent to partition leaders from clients, partition replicas, and the controller

Clients always initiate connections and send requests, broker process requests and responds

All requests have a header that includes:
- request type
- request version
- correlation ID (identifies the request and appears in response)
- client ID: identifes app that sent request

Main requests are: Produce requests and Fetch requests

Clients can also use metadata request (made to ANY broker), which includes a list of topics the client is interested in. Server responds with which partitions exist in the topics, the replicas for each partition, and which replica is the leader

## Physical Storage

Basic storage unit is partition replica

Partition cannot be split between multiple brokers or even multiple disks on one broker. Size is limited by space available on a signle mount point

### Partition Allocation

Handled to follow rules of replication (follower replicas cannot be on broker with leader). Assigned round robin.

Directory is assigned per partition based on which directoy has the fewest partitions (based on number of partitions, not size of partition)

### File Management

Partitions are broken into segments that are either 1GB or a week of data (whichever is smaller).

Writes are made to the active segment, which is never deleted (so data may be older than TTL)

### File Format

Stored ina single data file, which stores messages and offsets. Exact same format as the messages sent in order to use zero-copy optimization

Message contains key, value, offset, message size, checksum code, version of message format, compression codec, and timestamp (configurable to be from producer or broker)

Compressed messages are compressed together in a batch that is sent to consumer (broker does not decompress)

Broker has `DumpLogSegment` tool which allows you to look at a partition segment in the filesystem and examine contents (`--deep-iteration` param will show info about compressed messages)

### Indexes

allows Kafka to quickly locate messages. automatically generated. safe to delete.

### Compaction

Deletes old messages with the same key as a newer message

Log is split into two portions:
- Clean (previously compacted)
- dirty (written after last compaction)

Compaction is enabled with `log.cleaner.enabled` param at start

Brokers run compaction manager thread which chooses partition with highest ratio of dirty messages

Cleaner thread creates in-memory map to find dirty messages

### Deleted Events

In order to delete a key, set the value as null, which is called a tombstone message.

Consumers can read this null value and know to delete a user from a database, for example

### When are topics compacted?

Current segment is never compacted

Kafka starts compacting when 50% of topic contains dirty records




