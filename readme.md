# Confluent Certified Developer for Apache Kafka - Study Guide

## Table of Contents
- [Kafka Core Concepts](#Kafka-Core-Concepts)
  - [Events](#events)
  - [Topics](#topics)
  - [Partitioning](#Partitioning)
  - [Brokering](#Brokering)
  - [Replication](#Replication)
  - [Producers](#Producers)
  - [Consumers](#consumers)
- [Kafka Ecosystem](#Kafka-Ecosystem)
  - [Schema Registry](#Schema-Registry)
  - [Kafka Streams](#Kafka-Streams)
  - [ksqlDB](#ksqlDB)
  - [Kafka APIs](#Kafka-APIs)

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

## Kafka Ecosystem
General notes
- do NOT write your own framework
- use existing infrastructure from the kafka community
- only write applications that have domain-specific, business value

#### Kafka Connect
- system for connecting non-Kafka systems to Kafka
- external client process, NOT on broker
- declarative
- scalable
- fault-tolerant
- removes need to write code
- configured with some JSON
- Connectors
  - pluggable software component
  - interfaces with external system and kafka
  - also exist as runtime entities
  - Source connectors act as Producers
  - Sink connectors act as Consumers

#### Schema Registry
- Problem:
  - new consumers will emerge for topics
  - schemas evolve with business
- Schema Registry solves this:
  - server process external to Kafka brokers
  - maintains a database of schemas
  - high availability option available
  - Consumer/Producer API component (check schema compatability)
  - defines schema compatability rules per topic
  - Producer API prevents incompatible messages from being produced
  - Consumer API prevents incompatible messages from being consumed
- Supported formats
  - JSON schema
  - Avro
  - Protocol buffers

#### Kafka Streams
- Functional Java API
- filtering, grouping, aggregating, joining
- scalable, fault-tolerant state management
- scalable computation based on Consumer groups
- integrates within your services as Library
- runs in context of your application
- does not require special infra

#### ksqlDB
- database optimized for stream processing
- runs on its own scalable, fault-tolerant cluster adjacent to kafka cluster
- stream processing programs written in SQL
- command line interface
- REST API for application integration
- Java library
- Kafka Connect integration

#### Kafka APIs
- Admin API
  - manage and inspect topics, brokers, and Kafka objects
- Producer API
  - write a stream of events to one or more topics
- Consumer API
  - read and process one or more topics' events
- Kafka Streams API
  - implement stream processing applications and microservice
  - performs transformations, stateful operations (e.g. aggregations and joins, windowing, processing based on event-time)
  - read from one or more topics, generate output to one or more topics, transforming input streams to output streams
- Kafka Connect API
  - data import/export connectors that read/write streams of events from/to external systems
