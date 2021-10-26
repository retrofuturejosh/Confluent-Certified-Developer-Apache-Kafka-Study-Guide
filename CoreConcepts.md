## Table of Contents
- [Kafka Core Concepts](#Kafka-Core-Concepts)
  - [Events](#events)
  - [Topics](#topics)
  - [Partitioning](#Partitioning)
  - [Brokering](#Brokering)
  - [Replication](#Replication)
  - [Producers](#Producers)
  - [Consumers](#consumers)

## Kafka Core Concepts

#### Events
- key and structured value
- immutable
- represents something that happened
- think of programming in terms of events > things

#### Topics
- ordered log of events
- new events appended to end of topic
- by default, messages aren't deleted until configured amount of time (even if read)
- logs (not queues), meaning they are durable, replicated, fault-tolerant records

#### Partitioning
- systematic way of breaking the one topic log file into many logs
- allows topics to be stored on separate servers
- messages with no key are sent round robin to all partitions
- messages with keys are hashed and sent to corresponding partition
  -  messages with the same key will be in the same partition in chronological order
  - e.g. if userId is key, all events for that userId will be in order in the same partition

#### Brokering
- computer, instance, or container running the kafka process
- manage partitions
- handle read/write requests
- manage replication of partitions
- purposefully simple
- don’t do any computation over messages or routing of messages between topics

#### Replication
- copies of data for fault tolerance
- one lead partition and N-1 followers (N is replication factor)
- writes/reads happen to the leader
- invisible process (for developers)
- tunable in producer

#### Producers
- external application that writes messages to a Kafka cluster
- communicateswith the cluster using Kafka’s network protocol
- existing libraries for major languages
  -  under the hood do the following:
    - put messages in topics
    - connection pooling
    - network buffering
    - partitioning

#### Consumers
- external application that reads messages from Kafka topics and does some work with them
- existing libraries for major languages
  - does the following
  - reads messages from topics
  - connection pooling
  - network protocol
  - horizontally and elastically scalable
  - maintains ordering within partitions at scale
- more complicated than producer applications
- reading does NOT delete a message
- single instance of consumer will receive ALL messages from each partition in order (only in order per partition, NOT across partitions)
- consumer groups (multiple instances of consumer application) will automatically reshuffle and balance reads (default behavior)
- maintains ordering-by-key guarantee across n number of consumers
